# 🔐 SSH (Secure Shell) desde cero

> Apuntes de **Servicios de Red e Internet** – 2º ASIR
> Laboratorio usado en los ejemplos: dos máquinas virtuales Ubuntu en VirtualBox
> - **Cliente:** `172.16.5.211`
> - **Servidor:** `172.16.5.210`
> - **Usuario:** `antonio` en ambas
>
> En los ejemplos con puerto personalizado se usa el **2222**. Sustitúyelo por el tuyo.

---

## Índice

1. [Qué es SSH](#1-qué-es-ssh)
2. [SSH frente a Telnet](#2-ssh-frente-a-telnet)
3. [Modelo cliente-servidor](#3-modelo-cliente-servidor)
4. [Cómo se establece y cifra la conexión](#4-cómo-se-establece-y-cifra-la-conexión)
5. [La clave de host y known_hosts](#5-la-clave-de-host-y-known_hosts)
6. [El ataque Man-in-the-Middle](#6-el-ataque-man-in-the-middle)
7. [Métodos de autenticación](#7-métodos-de-autenticación)
   - [7.1 Usuario y contraseña](#71-usuario-y-contraseña)
   - [7.2 Clave pública y privada](#72-clave-pública-y-privada)
   - [7.3 Otros métodos](#73-otros-métodos)
8. [Ficheros de configuración](#8-ficheros-de-configuración)
   - [8.1 Mapa de ficheros](#81-mapa-de-ficheros)
   - [8.2 Configuración del servidor: sshd_config](#82-configuración-del-servidor-sshd_config)
   - [8.3 Configuración del cliente: ssh_config y ~/.ssh/config](#83-configuración-del-cliente-ssh_config-y-sshconfig)
   - [8.4 authorized_keys](#84-authorized_keys)
   - [8.5 Permisos obligatorios](#85-permisos-obligatorios)
9. [Transferir archivos con scp](#9-transferir-archivos-con-scp)
   - [9.1 Sintaxis básica](#91-sintaxis-básica)
   - [9.2 Ejemplos prácticos](#92-ejemplos-prácticos)
   - [9.3 Opciones más útiles](#93-opciones-más-útiles)
   - [9.4 Errores típicos con scp](#94-errores-típicos-con-scp)
   - [9.5 Alternativas: sftp y rsync](#95-alternativas-sftp-y-rsync)
10. [Otros usos de SSH](#10-otros-usos-de-ssh)
11. [Medidas de seguridad](#11-medidas-de-seguridad)
12. [Diagnóstico de problemas](#12-diagnóstico-de-problemas)
13. [Chuleta de comandos](#13-chuleta-de-comandos)
14. [Antes de conectarte en un entorno real](#14-antes-de-conectarte-en-un-entorno-real)

---

## 1. Qué es SSH

**SSH (Secure Shell)** es un protocolo de red que permite **conectarse y administrar otra máquina de forma remota a través de un canal cifrado**. Funciona por defecto sobre **TCP, puerto 22**.

Con SSH puedes:

- Abrir una terminal en otra máquina como si estuvieras sentado delante de ella.
- Ejecutar comandos remotos sueltos.
- Copiar archivos entre máquinas (`scp`, `sftp`, `rsync`).
- Crear túneles cifrados para otros servicios.

**Un poco de historia:** SSH lo creó Tatu Ylönen en 1995, después de que en su universidad (Finlandia) se produjera un ataque en el que se capturaron contraseñas que viajaban en claro por la red. La versión actual del protocolo es **SSH-2**, que es la que se usa hoy. **SSH-1 es obsoleta e insegura**.

La implementación más usada es **OpenSSH**, que viene en Linux, macOS e incluso en Windows 10/11.

SSH garantiza tres cosas:

| Propiedad | Qué significa |
|---|---|
| **Confidencialidad** | Nadie que capture el tráfico puede leerlo (cifrado). |
| **Integridad** | Nadie puede modificar los datos por el camino sin que se detecte (MAC). |
| **Autenticación** | El cliente comprueba que el servidor es quien dice ser, y el servidor comprueba quién es el usuario. |

[⬆ Volver al índice](#índice)

---

## 2. SSH frente a Telnet

Antes de SSH se usaba **Telnet** (puerto 23) para administrar máquinas en remoto. El problema es que **Telnet no cifra nada**: el usuario, la contraseña y cada comando viajan en texto plano.

Si alguien captura el tráfico con Wireshark ("Seguir flujo TCP"), esto es lo que vería:

**Conexión Telnet:**
```
Ubuntu 24.04 LTS
servidor login: antonio
Password: MiContraseña123
antonio@servidor:~$ sudo cat /etc/shadow
```

👉 La contraseña aparece **tal cual**. Cualquiera en la misma red puede robarla.

**Conexión SSH:**
```
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13
....^.f..W..3.k...L.].....z.c..A....9..j.x..Q..r...
.p...e.%..}..8.....H...t..Y,..^.....Ww.....Z.......
```

👉 Solo se ve en claro el **saludo inicial con la versión**. Todo lo demás, incluida la contraseña, es ilegible.

### Comparativa

| Característica | Telnet | SSH |
|---|---|---|
| Puerto por defecto | 23/TCP | 22/TCP |
| Cifrado del tráfico | ❌ No | ✅ Sí |
| Contraseña protegida | ❌ Viaja en claro | ✅ Viaja dentro del canal cifrado |
| Verifica la identidad del servidor | ❌ No | ✅ Sí (clave de host) |
| Integridad de los datos | ❌ No | ✅ Sí |
| Autenticación con claves | ❌ No | ✅ Sí |
| Transferencia de archivos | ❌ No | ✅ `scp`, `sftp` |
| Túneles | ❌ No | ✅ Sí |

**Conclusión:** Telnet no debe usarse nunca para administrar sistemas. Hoy solo tiene sentido como herramienta de diagnóstico (por ejemplo, `telnet IP puerto` para comprobar si un puerto está abierto, aunque para eso es mejor `nc -zv IP puerto`).

[⬆ Volver al índice](#índice)

---

## 3. Modelo cliente-servidor

SSH sigue el modelo **cliente-servidor**:

- **Servidor:** la máquina a la que te conectas. Ejecuta el demonio **`sshd`**, que está siempre escuchando en el puerto 22 esperando conexiones. Paquete: `openssh-server`.
- **Cliente:** la máquina desde la que te conectas. Usa el programa **`ssh`** (y `scp`, `sftp`...). Paquete: `openssh-client`, normalmente ya instalado.

```
      CLIENTE (172.16.5.211)                          SERVIDOR (172.16.5.210)
 ┌──────────────────────────┐                    ┌─────────────────────────────┐
 │  Usuario: antonio        │                    │  sshd (demonio)             │
 │          │               │   canal cifrado    │  escucha en el puerto 22    │
 │          ▼               │ =================> │             │               │
 │  ssh / scp / sftp        │ <================= │             ▼               │
 │  (puerto origen          │      TCP / 22      │  bash del usuario antonio   │
 │   aleatorio)             │                    │                             │
 └──────────────────────────┘                    └─────────────────────────────┘
```

### Instalación y puesta en marcha

En el **servidor**:
```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh      # arrancar y activar al inicio
sudo systemctl status ssh            # comprobar que está activo
ss -tlnp | grep ssh                  # comprobar en qué puerto escucha
```

> 📝 En Ubuntu/Debian el servicio se llama **`ssh`**. En Red Hat, Rocky o Fedora se llama **`sshd`**.

En el **cliente**, la conexión más básica:
```bash
ssh antonio@172.16.5.210             # puerto 22 por defecto
ssh -p 2222 antonio@172.16.5.210     # puerto personalizado
```

Para salir de la sesión: `exit` o `Ctrl + D`.

[⬆ Volver al índice](#índice)

---

## 4. Cómo se establece y cifra la conexión

Esta es la parte más importante para **entender** SSH y no solo usarlo.

### Las tres claves que intervienen

Un error muy común es pensar que tu par de claves (`id_ed25519` / `id_ed25519.pub`) es lo que cifra la conversación. **No es así.** En una conexión SSH intervienen tres claves distintas, cada una con un papel:

| Clave | Dónde está | Tipo | Para qué sirve |
|---|---|---|---|
| **Clave de host** | Servidor: `/etc/ssh/ssh_host_*` | Asimétrica (par) | El **servidor** demuestra quién es. |
| **Par de claves del usuario** | Privada en el cliente (`~/.ssh/id_ed25519`), pública en el servidor (`~/.ssh/authorized_keys`) | Asimétrica (par) | El **usuario** demuestra quién es. |
| **Clave de sesión** | Solo en memoria, en ambos extremos | **Simétrica** | **Cifra todo el tráfico** de la sesión. |

### Cifrar no es lo mismo que firmar

Con criptografía asimétrica se pueden hacer dos cosas distintas:

| Operación | Se usa la clave... | Se comprueba/descifra con... | Para qué |
|---|---|---|---|
| **Cifrar** | **pública** del destinatario | **privada** del destinatario | Que solo él pueda leerlo (confidencialidad) |
| **Firmar** | **privada** propia | **pública** propia | Demostrar que lo has hecho tú (identidad) |

En SSH, las claves asimétricas (de host y de usuario) se usan para **firmar**, es decir, para demostrar identidad. El cifrado del tráfico lo hace la clave de sesión simétrica.

### Las 4 fases de una conexión

```
         CLIENTE (ssh)                                SERVIDOR (sshd)
              |                                              |
              |---- Conexión TCP al puerto 22 -------------->|
              |<--- Versión: SSH-2.0-OpenSSH ----------------|
              |---- Versión: SSH-2.0-OpenSSH --------------->|
              |                                              |
 ---------------- FASE 1: Negociación de algoritmos -----------------
              |---- Algoritmos que soporto ----------------->|
              |<--- Algoritmos que soporto ------------------|
              |                                              |
 ---------------- FASE 2: Intercambio de claves (DH) ----------------
              |---- Mi parte pública del intercambio ------->|
              |<--- Su parte pública + clave de host --------|
              |     + FIRMA con su clave privada de host     |
              |                                              |
   El cliente verifica la firma y busca la clave en known_hosts
              |                                              |
   Ambos calculan la MISMA clave de sesión.
   >>> A partir de aquí TODO va cifrado <<<
              |                                              |
 ---------------- FASE 3: Autenticación del usuario -----------------
              |---- Contraseña o firma con clave privada --->|
              |<--- Acceso concedido ------------------------|
              |                                              |
 ---------------- FASE 4: Sesión cifrada ----------------------------
              |<=========== shell, scp, túneles ============>|
```

**Fase 1 – Negociación.** Cliente y servidor se dicen qué algoritmos soportan y eligen los mejores que tengan en común: algoritmo de intercambio de claves, de cifrado (AES, ChaCha20...), de integridad (MAC) y de clave de host (ed25519, rsa...).

**Fase 2 – Intercambio de claves (Diffie-Hellman).** Ambas partes calculan **la misma clave de sesión sin enviarla nunca por la red**. Además, el servidor **firma** el intercambio con su clave de host privada. El cliente verifica esa firma y comprueba en `known_hosts` que la clave del servidor es la esperada (ver [apartado 5](#5-la-clave-de-host-y-known_hosts)).

**Fase 3 – Autenticación del usuario.** Ocurre **ya dentro del canal cifrado**. Por eso es seguro enviar la contraseña: viaja cifrada con la clave de sesión.

**Fase 4 – Sesión.** Todo lo que escribes y recibes se cifra con la clave de sesión simétrica.

### Diffie-Hellman explicado con pintura

¿Cómo pueden dos máquinas acordar un secreto común hablando por una red que todo el mundo puede escuchar? La analogía clásica:

1. Cliente y servidor acuerdan en público un color base, por ejemplo **amarillo**.
2. Cada uno elige en secreto otro color: el cliente **rojo** y el servidor **azul**.
3. Cada uno mezcla el amarillo con su color secreto y envía la mezcla al otro:
   - Cliente envía *amarillo + rojo* = naranja.
   - Servidor envía *amarillo + azul* = verde.
4. Cada uno añade su color secreto a la mezcla que ha recibido:
   - Cliente: verde + rojo = **amarillo + azul + rojo**.
   - Servidor: naranja + azul = **amarillo + rojo + azul**.
5. ¡Los dos tienen **el mismo color final**!

Un espía ha visto el amarillo, el naranja y el verde, pero **no puede "separar" la pintura** para obtener los colores secretos. En matemáticas, esa "pintura imposible de separar" son operaciones como el logaritmo discreto o las curvas elípticas (X25519).

> 💡 Las versiones recientes de OpenSSH usan por defecto algoritmos **híbridos post-cuánticos** (como ML-KEM combinado con X25519), pensados para resistir futuros ordenadores cuánticos.

### ¿Por qué usar una clave simétrica para el tráfico?

- **Velocidad:** el cifrado simétrico es muchísimo más rápido que el asimétrico.
- **Perfect Forward Secrecy (PFS):** la clave de sesión es temporal y diferente en cada conexión. Aunque mañana alguien robara tu clave privada o la del servidor, **no podría descifrar las sesiones que hubiera grabado hoy**.

[⬆ Volver al índice](#índice)

---

## 5. La clave de host y known_hosts

### El aviso de la primera conexión

La primera vez que te conectas a un servidor aparece esto:

```
The authenticity of host '172.16.5.210 (172.16.5.210)' can't be established.
ED25519 key fingerprint is SHA256:Xk3fP9...q2Rw.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

**Qué te está preguntando:** el servidor te ha enseñado su **clave pública de host**, y SSH te muestra su **huella** (*fingerprint*), un resumen corto de esa clave. Como es la primera vez, tu cliente no tiene con qué compararla, así que te pregunta: *"¿Confías en que esta huella pertenece de verdad al servidor al que quieres conectarte?"*.

**Qué pasa al responder `yes`:** la clave pública del servidor se guarda en tu fichero **`~/.ssh/known_hosts`**. En las siguientes conexiones SSH la compara automáticamente:

- ✅ Si coincide, conecta sin preguntar.
- 🚨 Si **no** coincide, bloquea la conexión con un aviso grave (ver más abajo).

Este modelo se llama **TOFU (Trust On First Use)**: se confía la primera vez y luego se vigila que no cambie.

### Cómo verificar la huella correctamente

Lo correcto es comparar la huella del aviso con la real **antes** de escribir `yes`. Se puede ver en el propio servidor (desde la consola de VirtualBox, no por SSH):

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# 256 SHA256:Xk3fP9...q2Rw root@servidor (ED25519)
```

Si las dos huellas coinciden, puedes aceptar con seguridad.

### El fichero known_hosts

En Ubuntu verás que las entradas parecen texto sin sentido:

```
|1|Fz8kq...=|9Wd2s...= ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
```

Es porque Ubuntu activa `HashKnownHosts yes`: los nombres e IPs se guardan **con hash**. Así, si alguien roba el fichero, no puede saber a qué máquinas te conectas.

Comandos útiles:
```bash
ssh-keygen -F 172.16.5.210                  # buscar la entrada de un host
ssh-keygen -R 172.16.5.210                  # borrar la entrada de un host
ssh-keygen -R "[172.16.5.210]:2222"         # borrar si usa un puerto distinto del 22
```

### Cuando la clave del servidor cambia

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
...
Host key verification failed.
```

Hay dos explicaciones posibles:

1. **Motivo legítimo:** has reinstalado el servidor, has regenerado sus claves o has reutilizado la IP para otra máquina.
2. **Ataque Man-in-the-Middle:** alguien se está haciendo pasar por el servidor (ver [apartado 6](#6-el-ataque-man-in-the-middle)).

👉 **Solo** si sabes con certeza que es el caso 1, borra la entrada antigua con `ssh-keygen -R IP` y vuelve a conectar. No borres la línea "por costumbre" sin saber por qué ha cambiado.

### ⚠️ Cuidado al clonar máquinas virtuales

Si clonas una VM en VirtualBox, **el clon tiene exactamente las mismas claves de host** que la original. Tendrías dos servidores distintos con la misma identidad, y eso es un fallo de seguridad. En el clon hay que regenerarlas:

```bash
sudo rm /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo systemctl restart ssh
```

### La opción StrictHostKeyChecking

Controla qué hace el cliente con hosts desconocidos o cambiados:

| Valor | Comportamiento |
|---|---|
| `ask` (por defecto) | Pregunta si el host es nuevo. Bloquea si ha cambiado. |
| `accept-new` | Acepta hosts nuevos sin preguntar, pero bloquea si han cambiado. |
| `yes` | Solo conecta a hosts que ya están en `known_hosts`. |
| `no` | Acepta todo, incluso claves cambiadas. **❌ Nunca en entornos reales.** |

[⬆ Volver al índice](#índice)

---

## 6. El ataque Man-in-the-Middle

### Qué es

Un ataque **Man-in-the-Middle (MITM)**, u "hombre en el medio", es aquel en el que un atacante se coloca **entre el cliente y el servidor**. Los dos creen que hablan directamente entre sí, pero en realidad todo pasa por el atacante, que puede **leer y modificar** la comunicación.

```
  Lo que creen el cliente y el servidor:

     ┌───────────┐                                         ┌───────────┐
     │  CLIENTE  │ <-------------------------------------> │ SERVIDOR  │
     └───────────┘                                         └───────────┘

  Lo que ocurre en realidad:

     ┌───────────┐                ┌────────────┐           ┌───────────┐
     │  CLIENTE  │ <------------> │  ATACANTE  │ <-------> │ SERVIDOR  │
     └───────────┘   intercepta   └────────────┘  reenvía  └───────────┘
```

En una red local, el atacante puede colocarse en medio con técnicas como el **envenenamiento ARP** (*ARP spoofing*), una red Wi-Fi falsa o la manipulación de respuestas DNS.

### Contra Telnet: el ataque es trivial

Como Telnet no cifra ni verifica la identidad del servidor, el atacante solo tiene que reenviar los paquetes mientras lee usuario, contraseña y comandos.

### Contra SSH: el atacante necesita la clave de host

Para engañarte con SSH, el atacante tiene que montar **dos conexiones separadas**:

```
 Cliente  ══(conexión SSH 1)══>  Atacante  ══(conexión SSH 2)══>  Servidor real
```

En la conexión 1 el atacante debe presentarte una **clave de host** y **firmar** el intercambio con su parte privada. Pero **no tiene la clave privada de host del servidor real** (está en `/etc/ssh/` del servidor). Así que solo puede presentarte **su propia clave**, que tendrá **otra huella**. Entonces:

| Situación | Qué ocurre |
|---|---|
| Ya te habías conectado antes | La clave no coincide con `known_hosts` → aparece **`REMOTE HOST IDENTIFICATION HAS CHANGED`** y SSH bloquea la conexión. **El ataque falla.** |
| Es tu primera conexión | Ves el aviso normal de host desconocido con una huella. Si escribes `yes` sin verificarla, **caes en la trampa**. Es el punto débil del modelo TOFU. |

### Qué consigue el atacante si caes

Depende del método de autenticación:

- **Con contraseña:** la contraseña viaja cifrada... **pero hasta el atacante**, que la descifra en su extremo. **Te roba la contraseña** y puede usarla para entrar al servidor real.
- **Con clave pública:** tu firma incluye el **identificador único de la sesión**, que se deriva del intercambio de claves. Como la conexión 1 (contigo) y la conexión 2 (con el servidor) tienen identificadores distintos, el atacante **no puede reutilizar tu firma** para entrar al servidor real. Tu clave privada nunca sale de tu máquina.

👉 Esta es otra razón de peso para preferir la autenticación por clave pública.

### Cómo protegerse

1. **Verificar la huella** en la primera conexión ([apartado 5](#5-la-clave-de-host-y-known_hosts)).
2. **No ignorar nunca** el aviso de identificación cambiada sin saber el motivo.
3. Usar **autenticación por clave pública** en lugar de contraseña.
4. En entornos grandes: distribuir las claves de host de forma centralizada, con **certificados SSH** o registros DNS **SSHFP**, para no depender de TOFU.

[⬆ Volver al índice](#índice)

---

## 7. Métodos de autenticación

Una vez establecido el canal cifrado, el usuario tiene que demostrar quién es. Los dos métodos que más se usan son la contraseña y la clave pública.

### 7.1 Usuario y contraseña

Es el método **por defecto**. El servidor pide la contraseña del usuario del sistema (la de `/etc/shadow`) y la comprueba.

```bash
ssh antonio@172.16.5.210
antonio@172.16.5.210's password:
```

| ✅ Ventajas | ❌ Inconvenientes |
|---|---|
| No requiere configuración. | Vulnerable a **ataques de fuerza bruta** y diccionario. |
| Fácil de usar desde cualquier equipo. | Vulnerable al robo en un ataque MITM ([apartado 6](#6-el-ataque-man-in-the-middle)). |
| | La seguridad depende de que la contraseña sea robusta. |
| | Hay que escribirla en cada conexión. |

### 7.2 Clave pública y privada

El usuario genera un **par de claves**:

- **Clave privada** (`~/.ssh/id_ed25519`): se queda **solo** en el cliente. Es secreta y **nunca se comparte**.
- **Clave pública** (`~/.ssh/id_ed25519.pub`): se copia al servidor, en `~/.ssh/authorized_keys` del usuario. Puede compartirse sin riesgo.

#### Cómo funciona

```
          CLIENTE                                         SERVIDOR
  (tiene la clave privada)                  (tiene la pública en authorized_keys)
             |                                                |
             |-- 1. Quiero entrar como antonio -------------->|
             |      con esta clave pública                    |
             |                                                |
             |               2. ¿Está en ~antonio/.ssh/authorized_keys?
             |                                                |
             |<- 3. Sí. Demuestra que tienes la privada ------|
             |                                                |
   4. Firma datos de la sesión                                |
      con su clave privada                                    |
             |                                                |
             |-- 5. Envía la firma -------------------------->|
             |                                                |
             |               6. Verifica la firma con la clave pública
             |                                                |
             |<- 7. Acceso concedido -------------------------|
```

La clave privada **nunca viaja por la red**: solo viaja una firma que demuestra que la tienes.

#### Paso 1: generar el par de claves (en el cliente)

```bash
ssh-keygen -t ed25519 -C "antonio@cliente-asir"
```

- `-t ed25519`: tipo de clave. **Ed25519 es la opción recomendada**: moderna, rápida, segura y con claves cortas.
- `-C`: comentario para identificar la clave (aparece al final de la pública).
- Si necesitas compatibilidad con sistemas muy antiguos: `ssh-keygen -t rsa -b 4096`.
- **DSA** está obsoleto y OpenSSH ya no lo admite.

El programa preguntará:
```
Enter file in which to save the key (/home/antonio/.ssh/id_ed25519):   ← Enter
Enter passphrase (empty for no passphrase):                            ← passphrase
```

#### ¿Qué es la passphrase?

Es una contraseña que **cifra la clave privada en el disco**. Si alguien te roba el fichero `id_ed25519`, no podrá usarlo sin la passphrase.

> ⚠️ No confundir: la passphrase **no** se envía al servidor. Solo sirve para "desbloquear" tu clave privada en tu propia máquina.

#### Paso 2: copiar la clave pública al servidor

Forma recomendada:
```bash
ssh-copy-id antonio@172.16.5.210
ssh-copy-id -p 2222 antonio@172.16.5.210     # con puerto personalizado
```

`ssh-copy-id` se conecta (con contraseña esta última vez), añade tu clave pública al final de `~/.ssh/authorized_keys` del servidor y ajusta los permisos.

Forma manual (para entender qué hace):
```bash
cat ~/.ssh/id_ed25519.pub | ssh antonio@172.16.5.210 \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

#### Paso 3: conectar

```bash
ssh antonio@172.16.5.210
Enter passphrase for key '/home/antonio/.ssh/id_ed25519':
```

Ahora te pide la **passphrase de tu clave**, no la contraseña del usuario del servidor.

#### ssh-agent: no escribir la passphrase cada vez

`ssh-agent` guarda en memoria tu clave ya desbloqueada durante la sesión:

```bash
eval "$(ssh-agent -s)"      # arrancar el agente (en escritorio suele estar ya activo)
ssh-add ~/.ssh/id_ed25519   # añadir la clave (pide la passphrase una vez)
ssh-add -l                  # listar claves cargadas
ssh-add -D                  # quitar todas las claves del agente
```

#### Contraseña vs clave pública

| Aspecto | Contraseña | Clave pública |
|---|---|---|
| Resistencia a fuerza bruta | Depende de la contraseña | Prácticamente imposible |
| Si hay un MITM | Te roban la contraseña | No pueden reutilizar tu firma |
| Comodidad | Escribirla siempre | Con `ssh-agent`, automático |
| Automatización (scripts, backups) | Mala | Ideal |
| Riesgo principal | Contraseña débil o reutilizada | Robo de la clave privada (mitigado con passphrase) |

### 7.3 Otros métodos

Existen otros métodos menos habituales que conviene conocer:

| Método | En qué consiste |
|---|---|
| **Host-based** | El servidor confía en el **usuario porque viene de una máquina concreta** de confianza (autentica la máquina, no la persona). Configuración compleja y poco usada hoy. |
| **Keyboard-interactive / 2FA** | Permite pedir varios factores, por ejemplo clave pública + código **TOTP** de una app como Google Authenticator (módulo PAM `libpam-google-authenticator`). |
| **Llaves de seguridad FIDO2** | La clave privada vive en una llave física (YubiKey...). Se generan con `ssh-keygen -t ed25519-sk`. Hay que tocar la llave para conectar. |
| **Certificados SSH** | Una autoridad certificadora (CA) firma las claves de usuarios y hosts. Evita copiar claves a cada servidor y el problema de TOFU. Muy usado en empresas grandes. |
| **GSSAPI / Kerberos** | Inicio de sesión único en dominios corporativos (por ejemplo, Active Directory). |

[⬆ Volver al índice](#índice)

---

## 8. Ficheros de configuración

### 8.1 Mapa de ficheros

```
SERVIDOR                                        CLIENTE
/etc/ssh/                                       /etc/ssh/
├── sshd_config          ← config del servidor  └── ssh_config        ← config global del cliente
├── sshd_config.d/       ← config adicional
├── ssh_host_ed25519_key      ← clave de host privada   ~/.ssh/
├── ssh_host_ed25519_key.pub  ← clave de host pública   ├── config             ← config personal del cliente
├── ssh_host_rsa_key(.pub)                              ├── id_ed25519         ← TU clave privada
└── ...                                                 ├── id_ed25519.pub     ← TU clave pública
                                                        └── known_hosts        ← huellas de servidores conocidos
~/.ssh/ (del usuario en el servidor)
└── authorized_keys      ← claves públicas autorizadas a entrar
```

> 💡 **Truco para no confundirse:** `sshd_config` lleva **d** de *daemon* (demonio), así que es el del **servidor**. `ssh_config` sin **d** es el del **cliente**.

| Fichero | Máquina | Función |
|---|---|---|
| `/etc/ssh/sshd_config` | Servidor | Configuración del servicio SSH |
| `/etc/ssh/sshd_config.d/*.conf` | Servidor | Configuración extra que se carga desde `sshd_config` |
| `/etc/ssh/ssh_host_*` | Servidor | Claves de host (identidad del servidor) |
| `~/.ssh/authorized_keys` | Servidor | Claves públicas que pueden entrar como ese usuario |
| `/etc/ssh/ssh_config` | Cliente | Configuración del cliente para todos los usuarios |
| `~/.ssh/config` | Cliente | Configuración personal del cliente (alias, puertos...) |
| `~/.ssh/id_*` / `id_*.pub` | Cliente | Par de claves del usuario |
| `~/.ssh/known_hosts` | Cliente | Claves de host de servidores conocidos |

### 8.2 Configuración del servidor: sshd_config

Fichero: `/etc/ssh/sshd_config`. Las líneas que empiezan por `#` son comentarios y muestran el **valor por defecto**.

Directivas más importantes:

| Directiva | Ejemplo | Qué hace |
|---|---|---|
| `Port` | `Port 2222` | Puerto en el que escucha el servidor. |
| `ListenAddress` | `ListenAddress 172.16.5.210` | IP en la que escucha (por defecto, todas). |
| `PermitRootLogin` | `PermitRootLogin no` | Permitir o no entrar directamente como root. |
| `PasswordAuthentication` | `PasswordAuthentication no` | Permitir o no el acceso con contraseña. |
| `PubkeyAuthentication` | `PubkeyAuthentication yes` | Permitir el acceso con clave pública. |
| `KbdInteractiveAuthentication` | `KbdInteractiveAuthentication no` | Autenticación interactiva (necesaria para 2FA). |
| `PermitEmptyPasswords` | `PermitEmptyPasswords no` | Permitir usuarios sin contraseña (nunca). |
| `AuthorizedKeysFile` | `AuthorizedKeysFile .ssh/authorized_keys` | Dónde buscar las claves públicas autorizadas. |
| `AllowUsers` | `AllowUsers antonio` | Lista blanca de usuarios que pueden entrar. |
| `AllowGroups` | `AllowGroups sshusers` | Lista blanca por grupo. |
| `DenyUsers` | `DenyUsers invitado` | Usuarios a los que se les niega el acceso. |
| `MaxAuthTries` | `MaxAuthTries 3` | Intentos de autenticación por conexión. |
| `LoginGraceTime` | `LoginGraceTime 30` | Segundos para autenticarse antes de cortar. |
| `ClientAliveInterval` | `ClientAliveInterval 300` | Cada cuántos segundos comprobar si el cliente sigue vivo. |
| `ClientAliveCountMax` | `ClientAliveCountMax 2` | Comprobaciones fallidas antes de cerrar la sesión inactiva. |
| `X11Forwarding` | `X11Forwarding no` | Permitir abrir aplicaciones gráficas remotas. |
| `Banner` | `Banner /etc/issue.net` | Mensaje que se muestra antes de iniciar sesión. |

#### ⚠️ Trampa importante: la carpeta sshd_config.d

En Ubuntu, la primera línea útil de `sshd_config` es:
```
Include /etc/ssh/sshd_config.d/*.conf
```

Y en `sshd_config` **gana el primer valor que se lee** para cada directiva. Como los ficheros de `sshd_config.d/` se cargan **antes** que el resto de `sshd_config`, pueden anular lo que escribas en el fichero principal.

Ejemplo real: en algunas instalaciones existe `/etc/ssh/sshd_config.d/50-cloud-init.conf` con `PasswordAuthentication yes`. Aunque pongas `no` en `sshd_config`, **seguirá permitiendo contraseñas**.

Por eso, después de cambiar algo, comprueba siempre la **configuración efectiva**:

```bash
sudo sshd -T | grep -i passwordauthentication
```

#### Aplicar cambios de forma segura

```bash
sudo sshd -t                     # 1. Comprobar la sintaxis (si no dice nada, está bien)
sudo systemctl restart ssh       # 2. Reiniciar el servicio
```

> 📝 **Cambio de puerto en Ubuntu moderno (24.04 y posteriores):** SSH se activa por **socket de systemd**, así que después de cambiar `Port` o `ListenAddress` hay que ejecutar:
> ```bash
> sudo systemctl daemon-reload
> sudo systemctl restart ssh.socket
> ss -tlnp | grep ssh             # comprobar el nuevo puerto
> ```

> 🛟 **Regla de oro:** cuando cambies la configuración de SSH, **no cierres la sesión actual**. Abre **otra terminal** y prueba a conectar. Si algo falla, todavía tienes la primera sesión para arreglarlo. En una VM tienes la consola de VirtualBox, pero en un servidor remoto real podrías quedarte fuera.

### 8.3 Configuración del cliente: ssh_config y ~/.ssh/config

- `/etc/ssh/ssh_config`: afecta a **todos los usuarios** del cliente.
- `~/.ssh/config`: configuración **personal**. Es el más útil.

Permite crear **alias** para no tener que recordar IPs, puertos ni usuarios:

```
# ~/.ssh/config

Host servidor
    HostName 172.16.5.210
    User antonio
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host cliente
    HostName 172.16.5.211
    User antonio

# Opciones para todos los hosts
Host *
    ServerAliveInterval 60
    AddKeysToAgent yes
```

Ahora, en lugar de:
```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 antonio@172.16.5.210
```
basta con:
```bash
ssh servidor
scp informe.txt servidor:~/       # scp también usa los alias
```

| Opción | Qué hace |
|---|---|
| `Host` | Nombre del alias (o patrón, como `*`). |
| `HostName` | IP o nombre real de la máquina. |
| `User` | Usuario remoto. |
| `Port` | Puerto remoto. |
| `IdentityFile` | Clave privada a usar. |
| `ServerAliveInterval` | Envía señales para que la conexión no se corte por inactividad. |
| `AddKeysToAgent` | Añade la clave al `ssh-agent` automáticamente al usarla. |
| `ProxyJump` | Saltar a través de otra máquina (ver [apartado 10](#10-otros-usos-de-ssh)). |

Este fichero debe tener permisos `600`: `chmod 600 ~/.ssh/config`.

### 8.4 authorized_keys

Fichero `~/.ssh/authorized_keys` **en el servidor**, dentro del home del usuario al que se quiere entrar. Contiene **una clave pública por línea**:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK3x... antonio@cliente-asir
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIP9q... antonio@macbook
```

Para **revocar** el acceso a un equipo, basta con borrar su línea.

También se pueden añadir **restricciones** delante de una clave:
```
from="172.16.5.211" ssh-ed25519 AAAAC3Nza... antonio@cliente-asir
```
Así, esa clave **solo funciona si la conexión viene de 172.16.5.211**.

### 8.5 Permisos obligatorios

SSH es muy estricto con los permisos (opción `StrictModes yes`). Si son demasiado abiertos, **ignora las claves** y no te deja entrar con ellas.

| Ruta | Permisos | Comando |
|---|---|---|
| `~/.ssh/` | `700` (drwx------) | `chmod 700 ~/.ssh` |
| `~/.ssh/authorized_keys` | `600` (-rw-------) | `chmod 600 ~/.ssh/authorized_keys` |
| `~/.ssh/id_ed25519` (privada) | `600` (-rw-------) | `chmod 600 ~/.ssh/id_ed25519` |
| `~/.ssh/id_ed25519.pub` (pública) | `644` (-rw-r--r--) | `chmod 644 ~/.ssh/id_ed25519.pub` |
| `~/.ssh/config` | `600` (-rw-------) | `chmod 600 ~/.ssh/config` |
| Home del usuario | Sin escritura para grupo/otros | `chmod go-w ~` |

Además, todo debe pertenecer al usuario: `chown -R antonio:antonio ~/.ssh`.

[⬆ Volver al índice](#índice)

---

## 9. Transferir archivos con scp

**`scp` (Secure Copy)** copia archivos entre máquinas **a través de SSH**. Usa la misma autenticación (contraseña o clave), el mismo cifrado y el mismo puerto que SSH. No necesita instalar ni configurar nada más.

### 9.1 Sintaxis básica

```bash
scp [opciones] ORIGEN DESTINO
```

Una ruta **remota** se escribe así:
```
usuario@host:ruta
```

👉 Los **dos puntos `:`** son los que indican que la ruta es remota. Sin ellos, `scp` trabaja en local.

| Forma | Significado |
|---|---|
| `antonio@172.16.5.210:/tmp/` | Carpeta `/tmp/` del servidor (ruta absoluta) |
| `antonio@172.16.5.210:docs/` | Carpeta `docs/` dentro del home de antonio (ruta relativa) |
| `antonio@172.16.5.210:` | El home de antonio (solo los dos puntos) |
| `servidor:~/` | Usando un alias de `~/.ssh/config` |

```
  SUBIR (local → remoto):    scp  archivo_local           usuario@host:destino
  BAJAR (remoto → local):    scp  usuario@host:archivo    destino_local
```

> 💡 Regla mnemotécnica: igual que `cp`, **primero origen, después destino**. Lo que lleva `usuario@host:` delante está en la otra máquina.

### 9.2 Ejemplos prácticos

**Subir un archivo** del cliente al home del servidor:
```bash
scp informe.txt antonio@172.16.5.210:
```

**Subir un archivo a una ruta concreta**:
```bash
scp informe.txt antonio@172.16.5.210:/home/antonio/practicas/
```

**Subir y renombrar** a la vez:
```bash
scp informe.txt antonio@172.16.5.210:/home/antonio/informe_final.txt
```

**Bajar un archivo** del servidor a la carpeta actual del cliente (el `.` es "aquí"):
```bash
scp antonio@172.16.5.210:/etc/ssh/sshd_config .
```

**Bajar a una carpeta concreta**:
```bash
scp antonio@172.16.5.210:/var/log/syslog ~/logs/syslog_servidor.txt
```

**Copiar varios archivos** a la vez:
```bash
scp archivo1.txt archivo2.txt script.sh antonio@172.16.5.210:/tmp/
```

**Copiar una carpeta entera** (recursivo, `-r`):
```bash
scp -r practicas/ antonio@172.16.5.210:/home/antonio/
scp -r antonio@172.16.5.210:/etc/nginx/ ./copia_nginx/
```

**Con puerto personalizado** (¡**`-P` mayúscula**!):
```bash
scp -P 2222 informe.txt antonio@172.16.5.210:
```

**Con una clave concreta**:
```bash
scp -i ~/.ssh/id_ed25519 informe.txt antonio@172.16.5.210:
```

**Usar comodines en remoto** (entre comillas para que no los interprete tu shell local):
```bash
scp 'antonio@172.16.5.210:/var/log/*.log' ./logs/
```

**Archivos con espacios** en el nombre:
```bash
scp "mi informe.txt" antonio@172.16.5.210:
scp 'antonio@172.16.5.210:"/home/antonio/mi informe.txt"' .
```

**Entre dos servidores remotos** desde tu máquina (`-3` hace que los datos pasen por ti):
```bash
scp -3 antonio@172.16.5.210:/tmp/datos.txt antonio@172.16.5.212:/tmp/
```

**Con alias** de `~/.ssh/config` (ya incluye usuario, IP, puerto y clave):
```bash
scp informe.txt servidor:~/practicas/
scp servidor:/etc/hostname .
```

### 9.3 Opciones más útiles

| Opción | Qué hace |
|---|---|
| `-r` | Copia carpetas de forma recursiva. |
| `-P puerto` | Puerto del servidor. **Mayúscula**, al contrario que en `ssh` (`-p`). |
| `-i clave` | Clave privada a usar. |
| `-p` | **Preserva** fechas de modificación y permisos del original. |
| `-C` | Comprime los datos durante la transferencia (útil en redes lentas). |
| `-l límite` | Limita el ancho de banda en Kbit/s (`-l 8000` ≈ 1 MB/s). |
| `-v` | Modo detallado, para diagnosticar fallos. |
| `-q` | Modo silencioso (sin barra de progreso). |
| `-3` | Copia entre dos remotos pasando por la máquina local. |
| `-o opción` | Pasa una opción de SSH, por ejemplo `-o StrictHostKeyChecking=accept-new`. |

> ⚠️ **La trampa del `-P`:**
> - `ssh -p 2222 ...` → **minúscula**
> - `scp -P 2222 ...` → **mayúscula** (en `scp`, `-p` minúscula es "preservar")
>
> Usar alias en `~/.ssh/config` evita este lío.

> 📝 Desde OpenSSH 9.0, `scp` usa por debajo el protocolo **SFTP** en lugar del antiguo protocolo SCP, que tenía fallos de diseño. Para ti no cambia nada. Si conectas con un servidor muy antiguo que no lo soporta, existe la opción `-O` para usar el protocolo clásico.

### 9.4 Errores típicos con scp

| Problema | Causa | Solución |
|---|---|---|
| Se crea en local un archivo llamado `antonio@172.16.5.210` | Te has olvidado de los **dos puntos `:`** | Escribe `antonio@172.16.5.210:` |
| `Permission denied` al escribir en `/etc/`, `/var/www/`... | Tu usuario no tiene permiso en esa carpeta (`scp` no usa `sudo`) | Copia a tu home y luego en el servidor: `sudo mv archivo /destino/` |
| `ssh: connect to host ... port 22: Connection refused` | El servidor usa otro puerto | Añade `-P 2222` |
| `not a regular file` | Intentas copiar una carpeta sin `-r` | Añade `-r` |
| `No such file or directory` | Ruta mal escrita o relativa a otro sitio | Recuerda: las rutas relativas remotas parten del **home** |
| Se ha machacado un archivo | `scp` **sobrescribe sin preguntar** | Revisa el destino antes, o usa `rsync` |

### 9.5 Alternativas: sftp y rsync

#### sftp: sesión interactiva

`sftp` abre una sesión interactiva tipo FTP, pero sobre SSH. Es útil para navegar por el servidor antes de decidir qué copiar.

```bash
sftp antonio@172.16.5.210
sftp -P 2222 antonio@172.16.5.210      # también con -P mayúscula
```

| Comando | Qué hace |
|---|---|
| `ls` / `lls` | Listar remoto / listar local |
| `cd` / `lcd` | Cambiar de carpeta remota / local |
| `pwd` / `lpwd` | Ver carpeta actual remota / local |
| `get archivo` | **Descargar** (remoto → local) |
| `put archivo` | **Subir** (local → remoto) |
| `get -r carpeta` / `put -r carpeta` | Descargar / subir carpetas |
| `mkdir`, `rm`, `rename` | Gestionar archivos remotos |
| `bye` o `exit` | Salir |

> 💡 La **`l`** delante de un comando significa *local*.

#### rsync: sincronización inteligente

`rsync` solo copia **lo que ha cambiado**, puede reanudar transferencias y es ideal para copias de seguridad. Tiene que estar instalado en las dos máquinas.

```bash
rsync -avz --progress practicas/ antonio@172.16.5.210:/home/antonio/practicas/
rsync -avz -e "ssh -p 2222" practicas/ antonio@172.16.5.210:/home/antonio/practicas/
```

- `-a`: modo archivo (recursivo, conserva permisos, fechas, enlaces...).
- `-v`: detallado. `-z`: comprime. `--progress`: muestra el progreso.
- **La barra final importa:** `practicas/` copia el **contenido** de la carpeta; `practicas` copia la **carpeta en sí**.
- ⚠️ `--delete` borra en el destino lo que no existe en el origen. Pruébalo antes con `--dry-run`.

#### ¿Cuál usar?

| Herramienta | Mejor para |
|---|---|
| `scp` | Copias rápidas y puntuales de uno o pocos archivos. |
| `sftp` | Explorar el servidor y copiar de forma interactiva. |
| `rsync` | Carpetas grandes, copias repetidas, backups y sincronización. |

[⬆ Volver al índice](#índice)

---

## 10. Otros usos de SSH

**Ejecutar un comando remoto** sin abrir sesión:
```bash
ssh servidor 'df -h'
ssh servidor 'sudo systemctl status nginx'
```

**Túnel local (`-L`)**: acceder desde el cliente a un servicio del servidor como si fuera local. Ejemplo: ver la web (puerto 80) del servidor en `http://localhost:8080` del cliente.
```bash
ssh -L 8080:localhost:80 servidor
```

**Túnel remoto (`-R`)**: lo contrario, publicar en el servidor un puerto de tu máquina.
```bash
ssh -R 9000:localhost:3000 servidor
```

**Túnel dinámico (`-D`)**: crea un proxy SOCKS para navegar a través del servidor.
```bash
ssh -D 1080 servidor
```

**Saltar por una máquina intermedia (`-J`, *ProxyJump*)**: llegar a una máquina interna pasando por un "bastión".
```bash
ssh -J antonio@bastion antonio@maquina-interna
```

**Aplicaciones gráficas remotas (`-X`)**: ejecutar un programa gráfico del servidor y verlo en el cliente (requiere `X11Forwarding yes`).
```bash
ssh -X servidor
```

[⬆ Volver al índice](#índice)

---

## 11. Medidas de seguridad

Un servidor SSH expuesto a Internet recibe **miles de intentos de acceso automáticos al día**. Conviene distinguir entre medidas que **protegen de verdad** y medidas que solo **reducen ruido**.

### Medidas reales

| Medida | Configuración | Qué protege |
|---|---|---|
| **Desactivar el acceso por contraseña** | `PasswordAuthentication no` | Elimina por completo la fuerza bruta de contraseñas. **La más importante.** |
| **Usar claves con passphrase** | `ssh-keygen -t ed25519` | Si roban la clave, no pueden usarla sin la passphrase. |
| **Prohibir el login de root** | `PermitRootLogin no` | Root existe en todas las máquinas: es el primer objetivo. Entra con tu usuario y usa `sudo`. |
| **Lista blanca de usuarios** | `AllowUsers antonio` | Solo esos usuarios pueden entrar, aunque haya otros en el sistema. |
| **Limitar intentos y tiempo** | `MaxAuthTries 3`, `LoginGraceTime 30` | Ralentiza ataques y corta conexiones colgadas. |
| **Fail2ban** | Ver abajo | Bloquea en el firewall las IPs que fallan repetidamente. |
| **Firewall** | `ufw` | Permite SSH solo desde las IPs o redes necesarias. |
| **Mantener OpenSSH actualizado** | `sudo apt upgrade` | Corrige vulnerabilidades conocidas. |
| **Cerrar sesiones inactivas** | `ClientAliveInterval 300`, `ClientAliveCountMax 2` | Evita sesiones abiertas olvidadas. |
| **Desactivar lo que no se usa** | `X11Forwarding no`, `PermitEmptyPasswords no` | Menos funciones, menos superficie de ataque. |
| **2FA** | Clave + TOTP | Aunque roben la clave, falta el segundo factor. |
| **Verificar las huellas de host** | [Apartado 5](#5-la-clave-de-host-y-known_hosts) | Protege contra Man-in-the-Middle. |

### Cambiar el puerto 22: ¿medida de seguridad?

**Tiene algo de utilidad, pero no es una protección real.** Es lo que se llama **seguridad por oscuridad**.

✅ **Lo que sí consigue:**
- Evita la mayoría de bots automáticos que solo prueban el puerto 22.
- Los logs quedan mucho más limpios, y es más fácil detectar ataques reales.

❌ **Lo que no consigue:**
- Ocultar el servicio a alguien que te ataque de forma específica. Un escaneo de todos los puertos lo encuentra en segundos, porque SSH **se identifica solo** al conectar:
  ```bash
  nmap -sV -p- 172.16.5.210
  # 2222/tcp open  ssh  OpenSSH 9.6p1 Ubuntu
  ```

👉 **Conclusión:** cambiar el puerto es un complemento aceptable, pero **nunca sustituye** a desactivar las contraseñas, usar claves y tener fail2ban o un firewall. Un servidor con contraseñas débiles en el puerto 2222 sigue siendo vulnerable.

### Ejemplo de configuración bastionada

Se recomienda ponerla en un fichero propio dentro de `sshd_config.d/`, con un número bajo para que se cargue primero (recuerda que gana el primer valor leído):

```bash
sudo nano /etc/ssh/sshd_config.d/10-bastionado.conf
```

```
# /etc/ssh/sshd_config.d/10-bastionado.conf
Port 2222
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
AllowUsers antonio
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
```

### Orden seguro para aplicarla (para no quedarte fuera)

1. Comprueba que **ya puedes entrar con clave pública** (`ssh-copy-id` hecho y probado).
2. Crea el fichero de configuración.
3. Comprueba la sintaxis: `sudo sshd -t`.
4. Comprueba la configuración efectiva: `sudo sshd -T | grep -Ei 'port|passwordauth|permitroot'`.
5. Si has cambiado el puerto, ábrelo en el firewall **antes** de reiniciar.
6. Reinicia: `sudo systemctl restart ssh` (y, si cambiaste el puerto en Ubuntu 24.04+, `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`).
7. **Sin cerrar la sesión actual**, prueba desde otra terminal: `ssh -p 2222 antonio@172.16.5.210`.
8. Comprueba que la contraseña ya no funciona: `ssh -o PubkeyAuthentication=no -p 2222 antonio@172.16.5.210` debe responder `Permission denied (publickey)`.

### Firewall con ufw

```bash
sudo ufw allow from 172.16.5.0/24 to any port 2222 proto tcp   # solo desde la red del laboratorio
sudo ufw enable
sudo ufw status numbered
```

O, para limitar conexiones repetidas desde una misma IP:
```bash
sudo ufw limit 2222/tcp
```

### Fail2ban

```bash
sudo apt install fail2ban
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled  = true
port     = 2222
maxretry = 3
findtime = 10m
bantime  = 1h
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd            # ver IPs bloqueadas
sudo fail2ban-client set sshd unbanip IP    # desbloquear una IP
```

> 📝 Si tu sistema no tiene `/var/log/auth.log`, añade `backend = systemd` en la sección `[sshd]`.

[⬆ Volver al índice](#índice)

---

## 12. Diagnóstico de problemas

### Herramientas

**En el cliente**, el modo detallado muestra cada paso de la conexión:
```bash
ssh -v antonio@172.16.5.210       # detallado
ssh -vvv antonio@172.16.5.210     # máximo detalle
```

**En el servidor:**
```bash
sudo systemctl status ssh                 # ¿está activo?
ss -tlnp | grep ssh                       # ¿en qué puerto escucha?
sudo journalctl -u ssh -f                 # logs del servicio en tiempo real
sudo tail -f /var/log/auth.log            # intentos de inicio de sesión
sudo sshd -t                              # ¿la configuración es válida?
sudo sshd -T                              # configuración efectiva completa
```

### Errores frecuentes

| Error | Causa probable | Solución |
|---|---|---|
| `Connection refused` | El servicio no está activo o escucha en otro puerto | `systemctl status ssh`, `ss -tlnp`, revisar `-p` |
| `Connection timed out` | Firewall bloqueando, IP incorrecta o red de VirtualBox mal configurada | Comprobar `ping`, `ufw status` y el adaptador de red de la VM |
| `No route to host` | Problema de red o firewall que rechaza | Revisar IPs, máscara y firewall |
| `Permission denied (publickey)` | Clave no autorizada, permisos incorrectos o usuario equivocado | Revisar `authorized_keys`, los [permisos](#85-permisos-obligatorios) y `ssh -v` |
| `Permission denied (publickey,password)` | Contraseña incorrecta o usuario fuera de `AllowUsers` | Revisar usuario y `/var/log/auth.log` |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | La clave de host cambió (reinstalación, clon de VM... o MITM) | Averiguar el motivo; si es legítimo, `ssh-keygen -R IP` |
| `UNPROTECTED PRIVATE KEY FILE!` | La clave privada tiene permisos demasiado abiertos | `chmod 600 ~/.ssh/id_ed25519` |
| `Too many authentication failures` | El agente ofrece demasiadas claves antes de la correcta | `ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 ...` |
| La sesión se congela tras inactividad | Un router o NAT corta la conexión | `ServerAliveInterval 60` en `~/.ssh/config` |

[⬆ Volver al índice](#índice)

---

## 13. Chuleta de comandos

```bash
# ── Servidor ─────────────────────────────────────────────
sudo apt install openssh-server                 # instalar
sudo systemctl enable --now ssh                 # arrancar y activar
sudo systemctl restart ssh                      # reiniciar tras cambios
sudo sshd -t                                    # validar configuración
sudo sshd -T                                    # ver configuración efectiva
ss -tlnp | grep ssh                             # ver puerto de escucha
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # huella de la clave de host

# ── Conexión ─────────────────────────────────────────────
ssh antonio@172.16.5.210                        # conectar
ssh -p 2222 antonio@172.16.5.210                # puerto personalizado (p minúscula)
ssh servidor                                    # usando alias de ~/.ssh/config
ssh servidor 'comando'                          # ejecutar comando remoto
ssh -v antonio@172.16.5.210                     # modo diagnóstico

# ── Claves ───────────────────────────────────────────────
ssh-keygen -t ed25519 -C "comentario"           # generar par de claves
ssh-copy-id -p 2222 antonio@172.16.5.210        # copiar clave pública al servidor
eval "$(ssh-agent -s)" && ssh-add               # cargar clave en el agente
ssh-keygen -R 172.16.5.210                      # borrar host de known_hosts

# ── Transferencia de archivos ────────────────────────────
scp archivo antonio@172.16.5.210:ruta/          # subir
scp antonio@172.16.5.210:ruta/archivo .         # bajar
scp -r carpeta/ antonio@172.16.5.210:ruta/      # subir carpeta
scp -P 2222 archivo antonio@172.16.5.210:       # puerto personalizado (P MAYÚSCULA)
sftp -P 2222 antonio@172.16.5.210               # sesión interactiva
rsync -avz -e "ssh -p 2222" carpeta/ antonio@172.16.5.210:ruta/   # sincronizar
```

[⬆ Volver al índice](#índice)

---

## 14. Antes de conectarte en un entorno real

Si en una empresa te piden "conéctate por SSH al servidor", aclara esto **antes** de hacerlo:

| # | Pregunta | Por qué importa |
|---|---|---|
| 1 | **¿Qué servidor exactamente?** (IP o nombre) | Conectarse a la máquina equivocada, sobre todo si es de producción, es un error clásico. |
| 2 | **¿Con qué usuario?** | Lo normal es un usuario nominal propio, nunca `root` ni cuentas compartidas. |
| 3 | **¿Cómo me autentico?** | Si es por clave, se envía la **clave pública**, nunca la privada. Si es por contraseña, que llegue por un canal seguro. |
| 4 | **¿Qué puerto y cómo llego?** | Puede usar un puerto distinto del 22 o requerir VPN o un bastión (`ProxyJump`). |
| 5 | **¿Cuál es la huella (fingerprint) del servidor?** | Para verificar la clave de host en la primera conexión y evitar un [Man-in-the-Middle](#6-el-ataque-man-in-the-middle). |
| 6 | **¿Para qué entro y con qué permisos?** | Saber qué tarea hacer, si tendrás `sudo` y si hay horario o ventana para hacer cambios. |

👉 **La más importante es la 6:** entrar sin saber qué puedes tocar es como se rompen cosas en producción.

[⬆ Volver al índice](#índice)
