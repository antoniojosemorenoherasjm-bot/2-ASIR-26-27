# Servicio DHCP

Apuntes de **Servicios de Red e Internet** (2º ASIR). Explicación desde cero del protocolo DHCP a partir de mis prácticas: servidor Linux con `isc-dhcp-server`, su sustituto Kea, cliente Ubuntu Desktop y servidor Windows Server 2022.

> [!IMPORTANT]
> Los valores marcados como **(ejemplo)** no salen de mi práctica real: el rango de Linux, la MAC del cliente, la puerta de enlace, la IP del Windows Server... Hay que sustituirlos por los reales antes de usar estos apuntes como referencia de la práctica.

## Índice

1. [Qué es DHCP](#1-qué-es-dhcp)
2. [Escenario de mis prácticas](#2-escenario-de-mis-prácticas)
3. [Funcionamiento del proceso DORA](#3-funcionamiento-del-proceso-dora)
4. [Concesión y renovación](#4-concesión-y-renovación)
5. [Servidor Linux con isc-dhcp-server](#5-servidor-linux-con-isc-dhcp-server)
   - [5.1 Instalación](#51-instalación)
   - [5.2 El fichero dhcpd.conf línea a línea](#52-el-fichero-dhcpdconf-línea-a-línea)
   - [5.3 Reservas por MAC](#53-reservas-por-mac)
   - [5.4 Interfaz de escucha](#54-interfaz-de-escucha)
   - [5.5 El fichero de concesiones dhcpd.leases](#55-el-fichero-de-concesiones-dhcpdleases)
   - [5.6 Arrancar y comprobar](#56-arrancar-y-comprobar)
6. [Kea, el sustituto de isc-dhcp-server](#6-kea-el-sustituto-de-isc-dhcp-server)
7. [Cliente DHCP en Linux](#7-cliente-dhcp-en-linux)
   - [7.1 Quién pide la IP en mi Ubuntu Desktop](#71-quién-pide-la-ip-en-mi-ubuntu-desktop)
   - [7.2 dhclient.conf línea a línea](#72-dhclientconf-línea-a-línea)
   - [7.3 dhcpcd.conf línea a línea](#73-dhcpcdconf-línea-a-línea)
   - [7.4 Netplan con dhcp4](#74-netplan-con-dhcp4)
   - [7.5 Liberar, renovar y depurar](#75-liberar-renovar-y-depurar)
8. [Servidor DHCP en Windows Server 2022](#8-servidor-dhcp-en-windows-server-2022)
9. [Agente de retransmisión (DHCP relay)](#9-agente-de-retransmisión-dhcp-relay)
10. [Conmutación por error (failover)](#10-conmutación-por-error-failover)
11. [Seguridad en DHCP](#11-seguridad-en-dhcp)
12. [Errores típicos y soluciones](#12-errores-típicos-y-soluciones)
13. [Chuleta de comandos](#13-chuleta-de-comandos)

---

## 1. Qué es DHCP

**DHCP** (*Dynamic Host Configuration Protocol*, RFC 2131) es el protocolo que configura automáticamente la red de un equipo cuando se conecta. Sin DHCP habría que escribir a mano en cada equipo todos estos datos:

| Dato que recibe el cliente | Ejemplo en mi práctica Linux | Opción DHCP |
|---|---|---|
| Dirección IP | `172.16.5.100` (ejemplo) | No es una opción: va en el campo `yiaddr` del mensaje |
| Máscara de subred | `255.255.255.0` | 1 |
| Puerta de enlace | `172.16.5.1` (ejemplo) | 3 |
| Servidores DNS | `8.8.8.8`, `8.8.4.4` | 6 |
| Nombre de dominio | `asir.test` | 15 |
| Duración de la concesión | `600` segundos | 51 |

Ideas clave:

- Funciona como **cliente-servidor sobre UDP**: el servidor escucha en el puerto **67** y el cliente en el **68**. En `/etc/services` aparecen como `bootps` y `bootpc`, porque DHCP es la evolución de un protocolo anterior llamado BOOTP.
- La IP **no se regala, se presta** durante un tiempo llamado **concesión** (*lease*). El cliente tiene que renovarla antes de que caduque.
- Ventajas: evita IPs duplicadas por errores manuales, centraliza los cambios (si cambia el DNS, se cambia en un solo sitio) y reaprovecha las IPs de equipos que ya no están.

El RFC define tres formas de asignar direcciones:

| Tipo | Cómo funciona | Uso típico |
|---|---|---|
| Dinámica | IP libre del rango, prestada durante un tiempo | Clientes normales (lo que hacen mis prácticas) |
| Automática | IP del rango asignada de forma permanente | Poco habitual hoy |
| Manual (reserva) | El administrador ata una IP concreta a una MAC concreta | Impresoras, servidores internos, equipos que deben tener siempre la misma IP |

[⬆ Volver al índice](#índice)

---

## 2. Escenario de mis prácticas

### Práctica 1: Linux en VirtualBox

Dos máquinas virtuales. El profesor pide que el servidor tenga **una sola tarjeta de red**, así que la cambio de modo según lo que esté haciendo.

```
            MODO LABORATORIO (VirtualBox, Red interna "intnet")

 +-----------------------------+          +-----------------------------+
 | UbuntuSRI                   |          | UbuntuSRI Cliente           |
 | Servidor isc-dhcp-server    |          | Ubuntu Desktop              |
 | enp0s3: 172.16.5.210/24     |          | enp0s3: IP por DHCP         |
 | (IP fija)                   |          | (NetworkManager)            |
 +--------------+--------------+          +--------------+--------------+
                |                                        |
                +------------------ intnet --------------+
                        sin router y sin Internet
```

| Modo del adaptador del servidor | IP del servidor | Para qué lo uso | Servicio DHCP |
|---|---|---|---|
| **Puente** | `172.16.5.210` | Instalar paquetes, tener Internet, entrar por SSH | **Parado**. Si estuviera activo, repartiría IPs a los equipos del aula (ver [Seguridad](#11-seguridad-en-dhcp)) |
| **Red interna** (`intnet`) | `172.16.5.210` | La prueba de DHCP con el cliente | Arrancado |

- El servidor **conserva la IP `172.16.5.210` en los dos modos** porque necesita tener una IP dentro de la subred que reparte (`172.16.5.0/24`). La razón se explica en el [apartado 5.4](#54-interfaz-de-escucha).
- El cliente está **solo** en Red interna, así que la única forma que tiene de conseguir IP es mi servidor.
- Una versión anterior funcionó con dos adaptadores (el cliente recibió `192.168.10.100`). La desinstalé para rehacerla desde cero con un solo adaptador.

### Práctica 2: Windows Server 2022

```
 +-----------------------------+          +-----------------------------+
 | Windows Server 2022 (GUI)   |          | Cliente Ubuntu              |
 | Rol Servidor DHCP           |          | IP por DHCP                 |
 | Ámbito "LAN-Laboratorio"    |          | Recibió 192.168.10.213      |
 | Red 192.168.10.0/24         |          |                             |
 +--------------+--------------+          +--------------+--------------+
                |                                        |
                +--------------- Red interna ------------+
```

- Rango del ámbito: `192.168.10.210` - `192.168.10.219`.
- Exclusión: `192.168.10.210` - `192.168.10.212`.
- El cliente recibió `192.168.10.213`: la **primera dirección libre** después de las excluidas. Detalles en el [apartado 8](#8-servidor-dhcp-en-windows-server-2022).

### Práctica 3: el fichero de configuración del cliente

El profesor pide explicar entero el fichero de configuración del cliente DHCP. En mi VM cliente existen **dos**: `/etc/dhcp/dhclient.conf` y `/etc/dhcpcd.conf`. Por qué hay dos, y cuál se usa de verdad, está en el [apartado 7](#7-cliente-dhcp-en-linux).

[⬆ Volver al índice](#índice)

---

## 3. Funcionamiento del proceso DORA

Cuando un cliente se conecta sin IP, consigue su configuración con cuatro mensajes. Las iniciales forman la palabra **DORA**: *Discover, Offer, Request, Ack*.

```
 CLIENTE                                         SERVIDOR
 UbuntuSRI Cliente (sin IP)                      UbuntuSRI (172.16.5.210)
 UDP 68                                          UDP 67
   |                                                 |
   | 1. DHCPDISCOVER                    (broadcast)  |
   |    0.0.0.0:68 -> 255.255.255.255:67             |
   |    "Hay algun servidor DHCP?"                   |
   |------------------------------------------------>|
   |                                                 |
   | 2. DHCPOFFER                                    |
   |    172.16.5.210:67 -> cliente:68                |
   |    "Te ofrezco 172.16.5.100, mascara, gw, DNS"  |
   |<------------------------------------------------|
   |                                                 |
   | 3. DHCPREQUEST                     (broadcast)  |
   |    0.0.0.0:68 -> 255.255.255.255:67             |
   |    "Acepto la oferta de 172.16.5.210"           |
   |------------------------------------------------>|
   |                                                 |
   | 4. DHCPACK                                      |
   |    172.16.5.210:67 -> cliente:68                |
   |    "Confirmado: 172.16.5.100 durante 600 s"     |
   |<------------------------------------------------|
   |                                                 |
   | El cliente comprueba por ARP que nadie usa      |
   | esa IP y configura su interfaz.                 |
```

| Paso | Mensaje | Origen -> destino | Envío | Contenido importante |
|---|---|---|---|---|
| D | `DHCPDISCOVER` | `0.0.0.0:68` -> `255.255.255.255:67` | Broadcast | MAC del cliente y lista de opciones que pide |
| O | `DHCPOFFER` | `172.16.5.210:67` -> cliente`:68` | Broadcast o unicast (*) | IP ofrecida, máscara, puerta de enlace, DNS, duración, identificador del servidor (opción 54) |
| R | `DHCPREQUEST` | `0.0.0.0:68` -> `255.255.255.255:67` | Broadcast | IP que acepta (opción 50) y servidor elegido (opción 54) |
| A | `DHCPACK` | `172.16.5.210:67` -> cliente`:68` | Broadcast o unicast (*) | Confirmación y parámetros definitivos |

(*) Depende del bit *broadcast flag* que pone el cliente en su Discover: si lo activa, el servidor contesta por broadcast; si no, se lo envía directamente a su MAC.

### Por qué se usa broadcast

1. **El cliente no tiene IP**, así que el origen es `0.0.0.0` ("yo, sin dirección").
2. **No sabe quién es el servidor**, así que el destino es `255.255.255.255` en IP y `ff:ff:ff:ff:ff:ff` en MAC: se lo dice a toda la red local.
3. **El Request también va en broadcast, y es a propósito**: dentro lleva la opción 54 con la IP del servidor elegido. Así **todos** los servidores se enteran de la decisión: el elegido confirma con el ACK y los demás devuelven a su rango la IP que habían ofrecido. Si fuera unicast, los descartados no se enterarían.

> [!NOTE]
> **Consecuencia:** el broadcast no atraviesa routers. Un servidor DHCP solo atiende a los clientes de su propia red, salvo que un **agente de retransmisión** lleve los mensajes de una red a otra ([apartado 9](#9-agente-de-retransmisión-dhcp-relay)).

### Varios servidores en la misma red

El Discover lo reciben todos los servidores a la vez y cada uno puede hacer su oferta. Quien decide es el **cliente**, que normalmente acepta **la primera oferta que le llega**. La carrera la gana el servidor que **responde** antes. Esto es lo que aprovecha un servidor DHCP falso ([apartado 11](#11-seguridad-en-dhcp)).

### Comprobación final

Después del ACK, el cliente pregunta por ARP si alguien usa ya esa IP. Si alguien responde, la IP está duplicada: el cliente envía un `DHCPDECLINE` y vuelve a empezar.

[⬆ Volver al índice](#índice)

---

## 4. Concesión y renovación

La concesión (*lease*) es el tiempo durante el que el cliente puede usar la IP. Para no quedarse sin ella, el cliente intenta renovarla en dos momentos llamados **T1** y **T2**.

```
 Concesión de 600 s (default-lease-time 600)

 0 s                  300 s                525 s                600 s
 |--------------------|--------------------|--------------------|
 ACK recibido         T1 (50 %)            T2 (87,5 %)          Caduca
 BOUND                RENEWING             REBINDING            Pierde la IP
 Usa la IP            Request UNICAST      Request BROADCAST    y vuelve a
                      a su servidor        a cualquier servidor DISCOVER
```

| Fase | Momento | Qué hace el cliente | A quién se lo pide |
|---|---|---|---|
| **BOUND** | De 0 a T1 | Usa la IP sin preguntar nada | A nadie |
| **RENEWING** | T1 = 50 % (300 s) | Envía un `DHCPREQUEST` para renovar y lo reintenta periódicamente | **Unicast** a su servidor: ya tiene IP y sabe quién se la dio |
| **REBINDING** | T2 = 87,5 % (525 s) | Su servidor no contesta; pregunta a cualquiera | **Broadcast** a cualquier servidor |
| **Caducada** | 100 % (600 s) | Deja de usar la IP y empieza de cero | Broadcast (`DHCPDISCOVER`) |

- Si el servidor responde con un ACK en T1, **el contador vuelve a cero**. Es lo normal: el cliente nunca llega a quedarse sin IP.
- T1 y T2 se pueden fijar con las opciones 58 y 59. Si el servidor no las envía, se usan el 50 % y el 87,5 %.

### Ejemplo: el servidor se apaga

El servidor `UbuntuSRI` se apaga cuando el cliente lleva 400 s de una concesión de 600 s:

- **El cliente no pierde la conexión al apagarse el servidor.** El servidor DHCP no está en el camino del tráfico, solo presta direcciones. La IP sigue siendo válida.
- Desde el segundo 300 (T1) ya estaba intentando renovar por unicast, sin respuesta.
- En el segundo 525 (T2) pregunta por broadcast a cualquier servidor. En mi laboratorio no hay otro, así que nadie contesta.
- En el segundo 600 la concesión caduca: el cliente pierde la IP y vuelve al Discover.

Por eso una caída del servidor DHCP pasa desapercibida durante un rato y luego, de repente, los equipos se quedan sin red.

### Ejemplo: cambio la duración en el servidor

Si cambio `default-lease-time` de 600 a 60 y reinicio el servicio, **el cliente no se entera en ese momento**. En DHCP siempre es el cliente quien inicia la conversación: el servidor no puede avisarle. El cliente recibirá la nueva duración en su siguiente renovación (en T1, como muy tarde a los 300 s) o si fuerzo una renovación a mano.

### Otros mensajes DHCP

| Mensaje | Quién lo envía | Cuándo |
|---|---|---|
| `DHCPNAK` | Servidor | Rechaza un Request porque la IP pedida no es válida en esa red. El cliente descarta la IP y vuelve a empezar con un Discover. |
| `DHCPRELEASE` | Cliente | Devuelve la IP voluntariamente (al liberarla a mano o al apagar). Va en **unicast**: el cliente ya tiene IP y conoce al servidor. |
| `DHCPDECLINE` | Cliente | La comprobación ARP detecta que la IP ofrecida ya está en uso. |
| `DHCPINFORM` | Cliente | Ya tiene una IP puesta a mano y solo pide opciones (por ejemplo, el DNS). |

### Reconexión con una IP recordada (INIT-REBOOT)

Un cliente que ya tuvo una IP no empieza por el Discover: envía directamente un **Request en broadcast** pidiendo "su" IP de antes (opción 50). Ejemplo: muevo un portátil de mi red `172.16.5.0/24` a la red `192.168.10.0/24` del Windows Server sin apagarlo.

```
 Portatil (recuerda 172.16.5.100)               Windows Server (192.168.10.0/24)
   |                                                 |
   | DHCPREQUEST (broadcast) "quiero 172.16.5.100"   |
   |------------------------------------------------>|
   |                                                 |
   | DHCPNAK "esa IP no pertenece a esta red"        |
   |<------------------------------------------------|
   |                                                 |
   | DHCPDISCOVER (empieza de cero)                  |
   |------------------------------------------------>|
   |                                                 |
   | OFFER / REQUEST / ACK -> IP de 192.168.10.0/24  |
   |<----------------------------------------------->|
```

> [!NOTE]
> El NAK **solo lo envía un servidor autoritativo**, es decir, uno que se considera el responsable de esa red. Uno no autoritativo se calla y el portátil espera hasta agotar el tiempo antes de rendirse. En `isc-dhcp-server` esto lo controla la directiva `authoritative;` ([apartado 5.2](#52-el-fichero-dhcpdconf-línea-a-línea)).

[⬆ Volver al índice](#índice)

---

## 5. Servidor Linux con isc-dhcp-server

> [!WARNING]
> `isc-dhcp-server` está **obsoleto**: ISC dejó de mantenerlo a finales de 2022 y recomienda **Kea** como sustituto ([apartado 6](#6-kea-el-sustituto-de-isc-dhcp-server)). Sigue en los repositorios de Ubuntu y casi toda la documentación lo usa, así que es buena herramienta para aprender, pero en una instalación nueva hoy se usaría Kea.

### 5.1 Instalación

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

> [!NOTE]
> Justo después de instalarlo, el servicio intenta arrancar y **falla**. Es normal: todavía no tiene subred configurada ni interfaz de escucha.

| Fichero | Para qué sirve |
|---|---|
| `/etc/dhcp/dhcpd.conf` | Configuración principal: subredes, rangos, opciones, reservas |
| `/etc/default/isc-dhcp-server` | Por qué interfaz de red escucha el servidor |
| `/var/lib/dhcp/dhcpd.leases` | Registro de las concesiones que ha dado |
| `journalctl -u isc-dhcp-server` | Logs del servicio (no es un fichero, es el comando para verlos) |

El `dhcpd.conf` que trae el paquete es muy largo y casi todo son ejemplos comentados. Lo práctico es guardar una copia y escribir uno limpio:

```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.orig
sudo nano /etc/dhcp/dhcpd.conf
```

[⬆ Volver al índice](#índice)

### 5.2 El fichero dhcpd.conf línea a línea

```conf
# /etc/dhcp/dhcpd.conf  -  servidor UbuntuSRI (172.16.5.210)

# ---------- Ámbito GLOBAL (vale para todas las subredes) ----------
option domain-name "asir.test";
option domain-name-servers 8.8.8.8, 8.8.4.4;
default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;
authoritative;

# ---------- Ámbito de SUBRED ----------
subnet 172.16.5.0 netmask 255.255.255.0 {
    range 172.16.5.100 172.16.5.150;          # (ejemplo) ajustar a mi rango real
    option routers 172.16.5.1;                # (ejemplo) en intnet no existe
    option subnet-mask 255.255.255.0;
    option broadcast-address 172.16.5.255;
}

# ---------- RESERVA por MAC ----------
host cliente-sri {
    hardware ethernet 08:00:27:aa:bb:cc;      # (ejemplo) MAC real del cliente
    fixed-address 172.16.5.50;
}
```

**Reglas de sintaxis:**

- Cada sentencia termina en `;`.
- Los bloques van entre `{ }` y **no** llevan `;` después de la llave de cierre.
- Los comentarios empiezan por `#`.

En el fichero hay tres tipos de líneas: **declaraciones** (describen la red), **parámetros** (controlan cómo se comporta el servidor) y **opciones** (datos que se envían al cliente).

#### Declaraciones

| Declaración | Qué hace | En mi práctica |
|---|---|---|
| `subnet <red> netmask <máscara> { }` | Declara una red que el servidor conoce. **Debe coincidir con la IP de la interfaz** por la que escucha. | `172.16.5.0/24`, porque el servidor es `172.16.5.210` |
| `range <primera> <última>;` | IPs que se reparten dinámicamente. | `.100` a `.150` (ejemplo). No incluye ni la `.210` del servidor ni la `.1` del router |
| `host <nombre> { }` | Reserva para un equipo concreto ([apartado 5.3](#53-reservas-por-mac)). | `cliente-sri` |
| `hardware ethernet <MAC>;` | MAC del equipo de la reserva. | Va dentro de `host` |
| `fixed-address <IP>;` | IP que recibirá siempre ese equipo. | Va dentro de `host` |

#### Parámetros

| Parámetro | Qué hace | En mi práctica |
|---|---|---|
| `default-lease-time` | Duración de la concesión, en segundos, cuando el cliente **no pide** una concreta. | 600 s, que da T1 = 300 s y T2 = 525 s |
| `max-lease-time` | **Tope**: si el cliente pide una concesión más larga, recibe como mucho esta. | 7200 s (2 horas) |
| `authoritative` | El servidor se considera responsable de la red: responde con **NAK** a peticiones de IPs que no le corresponden ([apartado 4](#reconexión-con-una-ip-recordada-init-reboot)). | Activado: es el único servidor de su red |
| `ddns-update-style none` | No actualiza el DNS con los nombres de los clientes. | No lo uso. Podría enlazarse con mi servidor BIND9 de `asir.test` |

#### Opciones

| Opción | Código | Qué recibe el cliente | En mi práctica |
|---|---|---|---|
| `option subnet-mask` | 1 | Máscara de red | `255.255.255.0` |
| `option routers` | 3 | Puerta de enlace | `172.16.5.1` (ejemplo) |
| `option domain-name-servers` | 6 | Servidores DNS, separados por comas | `8.8.8.8, 8.8.4.4` |
| `option domain-name` | 15 | Sufijo DNS | `asir.test` |
| `option broadcast-address` | 28 | Dirección de broadcast | `172.16.5.255` |

> [!NOTE]
> **La puerta de enlace en mi Red interna.** En `intnet` solo hay dos VMs: no existe ningún `172.16.5.1`. El cliente recibe IP y puede comunicarse con el servidor, pero **no tiene Internet**. Para la práctica no importa, porque el objetivo es comprobar que recibe la configuración. Para darle Internet, el servidor tendría que hacer de router (reenvío de paquetes + NAT) y tener salida él mismo, y en modo Red interna no la tiene.

#### Ámbitos: qué valor gana

```
 +--------------------------------------------------------+
 | GLOBAL                                                 |
 | option domain-name-servers 8.8.8.8, 8.8.4.4;           |
 |                                                        |
 |   +------------------------------------------------+   |
 |   | SUBNET 172.16.5.0/24                           |   |
 |   | option routers 172.16.5.1;                     |   |
 |   |                                                |   |
 |   |   +----------------------------------------+   |   |
 |   |   | HOST cliente-sri                       |   |   |
 |   |   | fixed-address 172.16.5.50;             |   |   |
 |   |   +----------------------------------------+   |   |
 |   +------------------------------------------------+   |
 +--------------------------------------------------------+
   Si una opción aparece en varios niveles, gana la del
   nivel más interno (el más concreto).
```

Lo que se pone fuera de cualquier bloque es global. Lo que va dentro de `subnet` sobrescribe lo global solo para esa red, y lo que va dentro de `host` sobrescribe todo lo anterior solo para ese equipo. Es la misma lógica que las opciones de servidor, de ámbito y de reserva en Windows ([apartado 8](#8-servidor-dhcp-en-windows-server-2022)).

> [!TIP]
> Antes de reiniciar el servicio, comprueba siempre la sintaxis. Un `;` olvidado hace que el servicio no arranque y `systemctl` no dice en qué línea está el fallo; este comando sí:
> ```bash
> sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
> ```

[⬆ Volver al índice](#índice)

### 5.3 Reservas por MAC

Una reserva hace que un equipo reciba **siempre la misma IP**, pero gestionada desde el servidor DHCP.

```conf
host cliente-sri {
    hardware ethernet 08:00:27:aa:bb:cc;      # (ejemplo)
    fixed-address 172.16.5.50;
}
```

| Línea | Explicación |
|---|---|
| `host cliente-sri {` | Abre la reserva. `cliente-sri` es solo una etiqueta: no tiene que coincidir con el nombre del equipo, pero no puede repetirse. |
| `hardware ethernet 08:00:27:aa:bb:cc;` | MAC del cliente. En VirtualBox empiezan por `08:00:27`. |
| `fixed-address 172.16.5.50;` | IP que recibirá siempre. |

Para saber la MAC del cliente:

```bash
ip link show enp0s3          # en el cliente: línea "link/ether"
```

También aparece en los logs del servidor y en `dhcpd.leases` después de que el cliente pida IP por primera vez.

**Buenas prácticas:**

- **Pon la IP reservada fuera del `range`.** Si estuviera dentro (por ejemplo `172.16.5.120`), el servidor podría prestársela dinámicamente a otro cliente antes de que aparezca el equipo reservado, y acabarían dos equipos con la misma IP.
- Dentro de `host` se pueden poner opciones solo para ese equipo (otro DNS, otro nombre...).
- Para que se aplique: reinicia el servicio y, en el cliente, libera y renueva la IP. Si el cliente ya tenía una concesión, la conservará hasta renovar.
- Las reservas **no aparecen en `dhcpd.leases`**: esas IPs no se prestan, están fijadas en el `.conf`.

| | Exclusión | Reserva |
|---|---|---|
| Qué hace | Saca IPs del reparto automático | Ata una IP a una MAC concreta |
| Para qué | IPs puestas a mano (el propio servidor, el router, una impresora con IP fija) | Equipos que necesitan siempre la misma IP |
| ¿El equipo usa DHCP? | No, se configura manualmente | Sí |
| En isc-dhcp-server | No existe como tal: se consigue dejando esas IPs fuera del `range` | `host { hardware ethernet; fixed-address; }` |

Ventaja de la reserva frente a poner la IP a mano: si cambia el DNS o la puerta de enlace, el equipo recibe el cambio solo, en su siguiente renovación.

[⬆ Volver al índice](#índice)

### 5.4 Interfaz de escucha

El fichero `/etc/default/isc-dhcp-server` indica **por qué interfaz** escucha el servidor:

```bash
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
INTERFACESv4="enp0s3"
INTERFACESv6=""
```

| Variable | Qué hace |
|---|---|
| `DHCPDv4_CONF` / `DHCPDv6_CONF` | Ruta del fichero de configuración, si no es la de por defecto. Comentadas = ruta por defecto. |
| `DHCPDv4_PID` / `DHCPDv6_PID` | Fichero donde se guarda el número de proceso. |
| `OPTIONS` | Parámetros extra para arrancar `dhcpd`. |
| `INTERFACESv4` | **Interfaces por las que se sirve DHCP en IPv4.** Varias, separadas por espacios: `"enp0s3 enp0s8"`. |
| `INTERFACESv6` | Lo mismo para IPv6. Vacía = no se arranca el servidor IPv6. |

El nombre real de la interfaz se ve con `ip a`. En VirtualBox, el primer adaptador suele ser `enp0s3`.

**Por qué el servidor necesita una IP dentro de la subred que reparte:** los clientes no necesitan conocer al servidor, porque lo buscan por broadcast. Quien lo necesita es el propio `dhcpd`: cuando le llega un mensaje por una interfaz, mira la IP de esa interfaz y busca el bloque `subnet` que la contiene para saber de qué rango repartir.

```
 Llega un DHCPDISCOVER por enp0s3
                |
                v
 enp0s3 esta en INTERFACESv4? ---- no ----> dhcpd no escucha ahi
                | si
                v
 IP de enp0s3 = 172.16.5.210
                |
                v
 Hay un "subnet" que contenga 172.16.5.210? ---- no ----> el servicio
                | si                                       no arranca
                v
 Reparte del "range" de 172.16.5.0/24
```

Además, la IP del servidor viaja en las ofertas como **identificador del servidor** (opción 54), y es la dirección a la que el cliente pide las renovaciones en T1.

[⬆ Volver al índice](#índice)

### 5.5 El fichero de concesiones dhcpd.leases

`/var/lib/dhcp/dhcpd.leases` es la "memoria" del servidor: qué IP ha prestado, a qué MAC y hasta cuándo. Ejemplo de una entrada:

```
lease 172.16.5.100 {
  starts 6 2026/10/17 13:10:02;
  ends 6 2026/10/17 13:20:02;
  cltt 6 2026/10/17 13:10:02;
  binding state active;
  next binding state free;
  hardware ethernet 08:00:27:11:22:33;
  client-hostname "ubuntusri-cliente";
}
```

| Campo | Significado |
|---|---|
| `lease 172.16.5.100` | IP prestada |
| `starts` / `ends` | Inicio y fin de la concesión. El primer número es el día de la semana (0 = domingo, 6 = sábado) |
| `cltt` | Última vez que el cliente habló con el servidor (*client last transaction time*) |
| `binding state` | Estado actual: `active` (en uso), `free` (libre), `backup` (reservada para el otro servidor en failover) |
| `next binding state` | Estado al que pasará cuando caduque |
| `hardware ethernet` | MAC del cliente |
| `client-hostname` | Nombre que envió el cliente |

- **Las horas están en UTC**, no en hora española. En verano hay 2 horas de diferencia y en invierno 1.
- El fichero **solo crece**: cada renovación añade una entrada nueva para la misma IP. **La que vale es la última.** El servidor lo limpia de vez en cuando.
- Para verlo en forma de tabla:

```bash
dhcp-lease-list
```

- Para vaciarlo en el laboratorio: para el servicio, vacía el fichero (no lo borres, el servicio necesita que exista) y arranca de nuevo.

```bash
sudo systemctl stop isc-dhcp-server
sudo truncate -s 0 /var/lib/dhcp/dhcpd.leases
sudo systemctl start isc-dhcp-server
```

[⬆ Volver al índice](#índice)

### 5.6 Arrancar y comprobar

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf     # 1. comprobar la sintaxis
sudo systemctl restart isc-dhcp-server     # 2. aplicar los cambios
sudo systemctl status isc-dhcp-server      # 3. ver si está activo
sudo ss -ulpn | grep ':67'                 # 4. comprobar que escucha en UDP 67
journalctl -u isc-dhcp-server -f           # 5. ver los mensajes en directo
```

Cuando el cliente pide IP, en los logs se ve el proceso DORA completo:

```
DHCPDISCOVER from 08:00:27:11:22:33 via enp0s3
DHCPOFFER on 172.16.5.100 to 08:00:27:11:22:33 (ubuntusri-cliente) via enp0s3
DHCPREQUEST for 172.16.5.100 (172.16.5.210) from 08:00:27:11:22:33 (ubuntusri-cliente) via enp0s3
DHCPACK on 172.16.5.100 to 08:00:27:11:22:33 (ubuntusri-cliente) via enp0s3
```

En la línea del `DHCPREQUEST`, la IP entre paréntesis (`172.16.5.210`) es el servidor que el cliente ha elegido: la opción 54.

> [!WARNING]
> **Con un solo adaptador, este es el paso más importante.** Si paso el adaptador a modo Puente con el servicio activo, mi VM se convierte en un servidor DHCP falso en la red del aula.
> ```bash
> sudo systemctl stop isc-dhcp-server      # antes de pasar a modo Puente
> sudo systemctl disable isc-dhcp-server   # para que no arranque solo al encender
> sudo systemctl start isc-dhcp-server     # al volver a Red interna
> ```

[⬆ Volver al índice](#índice)

---

## 6. Kea, el sustituto de isc-dhcp-server

ISC (la organización que creó `isc-dhcp-server`) anunció su fin de vida a finales de 2022. Su sustituto es **Kea**, del mismo fabricante. Cambios principales:

- **Modular:** servicios separados para DHCPv4 (`kea-dhcp4-server`), DHCPv6 (`kea-dhcp6-server`), actualización de DNS (`kea-dhcp-ddns-server`) y una API REST (`kea-ctrl-agent`).
- **Configuración en JSON.**
- Las concesiones se guardan en un fichero CSV o en una **base de datos** (MySQL o PostgreSQL).
- Se puede **recargar la configuración sin reiniciar** el servicio, a través de la API.
- Funciones extra mediante módulos llamados *hooks*, como la alta disponibilidad ([apartado 10](#10-conmutación-por-error-failover)).

### Instalación

```bash
sudo apt install kea                   # instala dhcp4, dhcp6, ddns y ctrl-agent
# o solo el servidor IPv4:
sudo apt install kea-dhcp4-server
```

### Mi configuración traducida a Kea

El mismo servidor del [apartado 5.2](#52-el-fichero-dhcpdconf-línea-a-línea), en `/etc/kea/kea-dhcp4.conf`:

```jsonc
{
  "Dhcp4": {
    // Interfaz de escucha: ya no hay un fichero aparte
    "interfaces-config": {
      "interfaces": [ "enp0s3" ]
    },

    "authoritative": true,
    "valid-lifetime": 600,
    "max-valid-lifetime": 7200,

    // Dónde se guardan las concesiones
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv"
    },

    // Opciones globales
    "option-data": [
      { "name": "domain-name-servers", "data": "8.8.8.8, 8.8.4.4" },
      { "name": "domain-name", "data": "asir.test" }
    ],

    "subnet4": [
      {
        "id": 1,
        "subnet": "172.16.5.0/24",
        "pools": [ { "pool": "172.16.5.100 - 172.16.5.150" } ],
        "option-data": [
          { "name": "routers", "data": "172.16.5.1" }
        ],
        "reservations": [
          { "hw-address": "08:00:27:aa:bb:cc", "ip-address": "172.16.5.50" }
        ]
      }
    ]
  }
}
```

Kea acepta comentarios con `//` y `#` aunque el JSON estándar no los permite.

### Equivalencias

| Concepto | isc-dhcp-server | Kea |
|---|---|---|
| Paquete / servicio | `isc-dhcp-server` | `kea-dhcp4-server` |
| Fichero de configuración | `/etc/dhcp/dhcpd.conf` | `/etc/kea/kea-dhcp4.conf` |
| Formato | Sintaxis propia con `{ }` y `;` | JSON |
| Interfaz de escucha | `/etc/default/isc-dhcp-server` | `interfaces-config`, dentro del mismo JSON |
| Subred | `subnet 172.16.5.0 netmask 255.255.255.0 { }` | `"subnet4": [ { "subnet": "172.16.5.0/24" } ]` |
| Rango | `range a b;` | `"pools": [ { "pool": "a - b" } ]` |
| Opciones | `option routers 172.16.5.1;` | `"option-data": [ { "name": "routers", "data": "..." } ]` |
| Duración de la concesión | `default-lease-time` / `max-lease-time` | `valid-lifetime` / `max-valid-lifetime` |
| Reserva | `host { hardware ethernet; fixed-address; }` | `"reservations": [ { "hw-address", "ip-address" } ]` |
| Concesiones | `/var/lib/dhcp/dhcpd.leases` | `/var/lib/kea/kea-leases4.csv` o base de datos |
| Comprobar sintaxis | `dhcpd -t -cf /etc/dhcp/dhcpd.conf` | `kea-dhcp4 -t /etc/kea/kea-dhcp4.conf` |
| Alta disponibilidad | `failover peer` | Hook *High Availability* |

### Comandos

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf     # comprobar la sintaxis
sudo systemctl restart kea-dhcp4-server       # aplicar los cambios
sudo systemctl status kea-dhcp4-server
journalctl -u kea-dhcp4-server -f             # logs
sudo cat /var/lib/kea/kea-leases4.csv         # concesiones
```

> [!WARNING]
> No pueden estar `isc-dhcp-server` y `kea-dhcp4-server` activos a la vez en la misma interfaz: los dos quieren el puerto UDP 67.

[⬆ Volver al índice](#índice)

---

## 7. Cliente DHCP en Linux

### 7.1 Quién pide la IP en mi Ubuntu Desktop

En un Linux actual hay varios programas que pueden hacer de cliente DHCP. Que exista un fichero de configuración **no significa que ese programa se esté usando**.

| Cliente DHCP | De dónde viene | Se configura en | ¿Lo usa mi Ubuntu Desktop? |
|---|---|---|---|
| Cliente interno de **NetworkManager** | NetworkManager, el gestor de red de Ubuntu Desktop | Perfiles de conexión (`nmcli`, configuración gráfica, netplan) | **Sí, es el que se usa por defecto** |
| **systemd-networkd** | El gestor de red de Ubuntu Server | Netplan | No (es el de Ubuntu Server) |
| **dhcpcd** | Paquete `dhcpcd-base`, sustituto oficial de `dhclient` en Ubuntu desde la 24.04 | `/etc/dhcpcd.conf` | No, salvo que lo ejecute a mano |
| **dhclient** | Paquete `isc-dhcp-client`, del mismo proyecto obsoleto que `isc-dhcp-server` | `/etc/dhcp/dhclient.conf` | No; es un resto de versiones anteriores |

```
 /etc/netplan/*.yaml
   renderer: NetworkManager
            |
            v
 NetworkManager ---> cliente DHCP interno ---> DORA con el servidor
            |
            +--- NO lee /etc/dhcp/dhclient.conf
            +--- NO lee /etc/dhcpcd.conf
```

**Por qué hay dos ficheros:**

- `dhcpcd.conf` viene con el paquete `dhcpcd-base`, que Ubuntu instala porque `dhcpcd` sustituye a `dhclient` como cliente de línea de comandos. Está instalado, pero ningún servicio lo usa en el arranque normal de un escritorio.
- `dhclient.conf` es del antiguo `isc-dhcp-client`: queda de versiones anteriores o lo ha instalado alguna dependencia.

**Cómo comprobarlo en lugar de suponerlo:**

```bash
cat /etc/netplan/*.yaml                          # debe aparecer "renderer: NetworkManager"
ps aux | grep -E 'dhclient|dhcpcd' | grep -v grep # sin resultados = no se están ejecutando
nmcli -f DHCP4 device show enp0s3                # lo que NetworkManager recibió por DHCP
ls /var/lib/NetworkManager/*.lease               # ficheros de concesión del cliente interno
journalctl -u NetworkManager | grep -i dhcp      # el proceso DHCP en los logs
```

> [!IMPORTANT]
> **Error de concepto frecuente:** ningún fichero del cliente sirve para indicar **qué servidor** da las IPs. El cliente descubre al servidor por broadcast. El fichero del cliente configura cómo se comporta el propio cliente: qué nombre envía, qué opciones pide, cuánto espera... En `dhcpcd.conf` se puede poner una IP fija (`static`), pero eso es justo lo contrario de usar DHCP.

> [!TIP]
> **Para demostrar `dhcpcd` sin tocar la configuración de red**, el modo de prueba hace la petición y muestra lo que ofrece el servidor sin aplicarlo a la interfaz:
> ```bash
> sudo dhcpcd -T enp0s3
> ```

[⬆ Volver al índice](#índice)

### 7.2 dhclient.conf línea a línea

Contenido típico en Ubuntu (compáralo con el tuyo con `cat /etc/dhcp/dhclient.conf`; si cambia alguna línea, la explicación de cada directiva sirve igual):

```conf
option rfc3442-classless-static-routes code 121 = array of unsigned integer 8;

send host-name = gethostname();
#send dhcp-client-identifier 1:0:a0:24:ab:fb:9c;
#send dhcp-lease-time 3600;
#supersede domain-name "fugue.com home.vix.com";
#prepend domain-name-servers 127.0.0.1;

request subnet-mask, broadcast-address, time-offset, routers,
        domain-name, domain-name-servers, domain-search, host-name,
        dhcp6.name-servers, dhcp6.domain-search, dhcp6.fqdn, dhcp6.sntp-servers,
        netbios-name-servers, netbios-scope, interface-mtu,
        rfc3442-classless-static-routes, ntp-servers;

#require subnet-mask, domain-name-servers;
timeout 300;
#retry 60;
#reboot 10;
#select-timeout 5;
#initial-interval 2;
#script "/sbin/dhclient-script";
#media "-link0 -link1 -link2", "link0 link1";
#reject 192.33.137.209;

#alias { ... }
#lease { ... }
```

Las líneas que empiezan por `#` están desactivadas: son ejemplos de lo que se puede configurar.

| Línea | Qué hace |
|---|---|
| `option rfc3442-classless-static-routes code 121 = array of unsigned integer 8;` | **Define** la opción 121 (rutas estáticas sin clase) para que `dhclient` sepa interpretarla. No la pide: solo enseña su formato. |
| `send host-name = gethostname();` | Envía al servidor el nombre del equipo (opción 12). Por eso aparece `client-hostname` en `dhcpd.leases` y el nombre entre paréntesis en los logs. |
| `#send dhcp-client-identifier ...;` | Enviaría un identificador de cliente propio (opción 61). Por defecto se identifica con la MAC. |
| `#send dhcp-lease-time 3600;` | Pediría una concesión de 3600 s. El servidor la limita con `max-lease-time`. |
| `#supersede domain-name "...";` | Ignora el valor que mande el servidor y usa este. |
| `#prepend domain-name-servers 127.0.0.1;` | Añade este DNS **delante** de los que mande el servidor. |
| `request ...;` | Lista de opciones que el cliente **pide** (opción 55, *Parameter Request List*). El servidor solo envía las que tiene configuradas. |
| `#require subnet-mask, domain-name-servers;` | Rechaza las ofertas que no incluyan estas opciones. |
| `timeout 300;` | Si en 300 s no consigue una concesión, deja de intentarlo. |
| `#retry 60;` | Tiempo de espera antes de volver a intentarlo después de rendirse. |
| `#reboot 10;` | Segundos que intenta recuperar su IP anterior (INIT-REBOOT) antes de empezar con un Discover. |
| `#select-timeout 5;` | Segundos que espera más ofertas antes de elegir. Con 0 se queda con la primera. |
| `#initial-interval 2;` | Intervalo entre los primeros reintentos; se va duplicando. |
| `#script "/sbin/dhclient-script";` | Script que aplica a la interfaz la configuración recibida (IP, rutas, DNS). |
| `#media ...;` | Tipos de medio físico. Obsoleto. |
| `#reject 192.33.137.209;` | Ignora las ofertas de ese servidor. Es una defensa básica contra un servidor falso conocido. |
| `#alias { }` | IP fija adicional en la interfaz. |
| `#lease { }` | Concesión predefinida que se usaría si no responde ningún servidor. |

Modificadores para cambiar lo que manda el servidor:

| Modificador | Efecto | Ejemplo |
|---|---|---|
| `default` | Usa este valor solo si el servidor no envía ninguno | `default domain-name "asir.test";` |
| `supersede` | Sustituye siempre el valor del servidor | `supersede domain-name-servers 172.16.6.210;` (mi servidor DNS) |
| `prepend` | Añade el valor delante de los del servidor | `prepend domain-name-servers 127.0.0.1;` |
| `append` | Añade el valor detrás de los del servidor | `append domain-name-servers 1.1.1.1;` |

Comandos de `dhclient` (solo si está instalado):

```bash
sudo dhclient -r enp0s3        # liberar la IP (envía DHCPRELEASE)
sudo dhclient -v enp0s3        # pedir IP mostrando el proceso DORA
cat /var/lib/dhcp/dhclient*.leases
```

[⬆ Volver al índice](#índice)

### 7.3 dhcpcd.conf línea a línea

El `dhcpcd.conf` que trae el paquete es una configuración de ejemplo con estas directivas. En tu fichero, las que empiezan por `#` están desactivadas.

| Directiva | Qué hace | Relación con lo visto |
|---|---|---|
| `controlgroup wheel` | Grupo de usuarios que puede controlar `dhcpcd` sin ser root. | Administración |
| `hostname` | Envía el nombre del equipo al servidor (opción 12). | Igual que `send host-name` en `dhclient` |
| `clientid` | Se identifica ante el servidor con la **MAC** de la interfaz (opción 61). | Las reservas por MAC funcionan sin sorpresas |
| `duid` | Se identifica con un **DUID**, un identificador ligado al equipo y no a la tarjeta. Es la alternativa a `clientid`. | Ver el error de las VMs clonadas en el [apartado 12](#12-errores-típicos-y-soluciones) |
| `persistent` | Al cerrarse `dhcpcd`, deja la IP configurada en la interfaz. | |
| `vendorclassid` | Opción 60. Vacía para no enviar la versión de `dhcpcd` ni el sistema operativo. | Privacidad: dificulta identificar el equipo |
| `option domain_name_servers, domain_name, domain_search` | Pide DNS, dominio y dominios de búsqueda. | Lista de opciones pedidas (opción 55) |
| `option classless_static_routes` | Pide rutas estáticas sin clase (opción 121). | |
| `option interface_mtu` | Pide el MTU de la red y lo aplica. | |
| `option host_name` | Pide un nombre de equipo al servidor. | |
| `option ntp_servers` | Pide servidores de hora. | |
| `option rapid_commit` | Permite DHCP en **dos mensajes** (Discover y Ack directamente) si el servidor también lo admite. | Variante abreviada de DORA |
| `require dhcp_server_identifier` | Descarta los mensajes que no traigan la opción 54. | El identificador del servidor de DORA |
| `slaac private` / `slaac hwaddr` | Cómo genera direcciones IPv6 automáticas. | No es DHCPv4 |

Directivas útiles que no vienen en el fichero de ejemplo:

```conf
# IP fija en vez de DHCP (lo que se hace, por ejemplo, en una Raspberry Pi)
interface enp0s3
static ip_address=172.16.5.60/24
static routers=172.16.5.1
static domain_name_servers=8.8.8.8

# No gestionar una interfaz
denyinterfaces enp0s8
```

Comandos de `dhcpcd`:

```bash
sudo dhcpcd -T enp0s3      # prueba: muestra la oferta sin aplicarla
sudo dhcpcd -4 enp0s3      # pedir IP (solo IPv4)
sudo dhcpcd -n enp0s3      # volver a pedir y renovar
sudo dhcpcd -k enp0s3      # liberar la IP (envía DHCPRELEASE)
dhcpcd -U enp0s3           # mostrar la concesión actual
```

> [!WARNING]
> No ejecutes `dhcpcd` de forma normal (sin `-T`) en una interfaz que gestiona NetworkManager: tendrías dos programas peleando por la misma interfaz.

[⬆ Volver al índice](#índice)

### 7.4 Netplan con dhcp4

Netplan es la capa de Ubuntu que lee ficheros YAML en `/etc/netplan/` y se los pasa al gestor de red que corresponda (*renderer*).

**Ubuntu Desktop** (mi cliente): netplan solo delega en NetworkManager.

```yaml
# /etc/netplan/01-network-manager-all.yaml
network:
  version: 2
  renderer: NetworkManager
```

**Ubuntu Server con DHCP**, gestionado por systemd-networkd:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp-identifier: mac
```

| Línea | Qué hace |
|---|---|
| `version: 2` | Versión del formato de netplan. Siempre 2. |
| `renderer: networkd` | Quién aplica la configuración: `networkd` (servidor) o `NetworkManager` (escritorio). |
| `ethernets:` / `enp0s3:` | Interfaz que se configura. |
| `dhcp4: true` | Pide IPv4 por DHCP. |
| `dhcp-identifier: mac` | Se identifica ante el servidor con la MAC. Por defecto systemd-networkd usa un identificador basado en `/etc/machine-id`, y dos VMs clonadas recibirían la misma IP. |

Otras opciones útiles dentro de la interfaz:

```yaml
      dhcp4-overrides:
        use-dns: false        # ignora el DNS que mande el servidor
        use-routes: false     # ignora la puerta de enlace que mande el servidor
```

**El servidor DHCP necesita IP fija:** no puede pedirse una IP a sí mismo. Así sería en `UbuntuSRI` si lo gestiona netplan:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [172.16.5.210/24]
      routes:
        - to: default
          via: 172.16.5.1                 # (ejemplo) puerta de enlace del aula
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan try       # aplica y deshace en 120 s si no confirmas (útil por SSH)
sudo netplan apply     # aplica definitivamente
```

> [!NOTE]
> Netplan avisa si los ficheros YAML tienen permisos demasiado abiertos. Lo correcto es `sudo chmod 600 /etc/netplan/*.yaml`. Además, en YAML la sangría es con **espacios**, nunca con tabuladores.

[⬆ Volver al índice](#índice)

### 7.5 Liberar, renovar y depurar

| Acción | NetworkManager (mi cliente) | dhcpcd | dhclient | systemd-networkd |
|---|---|---|---|---|
| Ver la concesión | `nmcli -f DHCP4 device show enp0s3` | `dhcpcd -U enp0s3` | `cat /var/lib/dhcp/dhclient*.leases` | `networkctl status enp0s3` |
| Soltar la IP | `sudo nmcli device disconnect enp0s3` | `sudo dhcpcd -k enp0s3` | `sudo dhclient -r enp0s3` | - |
| Pedir o renovar | `sudo nmcli device connect enp0s3` | `sudo dhcpcd -n enp0s3` | `sudo dhclient -v enp0s3` | `sudo networkctl renew enp0s3` |

**Ver la cuenta atrás de la concesión:**

```bash
ip a show enp0s3
```

Una IP obtenida por DHCP aparece marcada como `dynamic`, con `valid_lft 587sec`: los segundos que le quedan a la concesión. Si lo repites, verás que baja y que vuelve a subir al renovar en T1.

**Capturar el tráfico DHCP** (en el servidor o en el cliente):

```bash
sudo tcpdump -i enp0s3 -n -e port 67 or port 68     # mensajes con MAC de origen y destino
sudo tcpdump -i enp0s3 -n -vv port 67 or port 68    # además, las opciones (53, 54, 50...)
```

Salida típica de un Discover y su respuesta:

```
IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from 08:00:27:11:22:33, length 300
IP 172.16.5.210.67 > 172.16.5.100.68: BOOTP/DHCP, Reply, length 300
```

En el modo `-vv`, la opción 53 (*DHCP-Message*) indica el tipo de mensaje: Discover, Offer, Request, ACK... En Wireshark, el filtro es `dhcp`.

[⬆ Volver al índice](#índice)

---

## 8. Servidor DHCP en Windows Server 2022

### Instalación del rol

- **Gráfico:** Administrador del servidor > Administrar > Agregar roles y características > **Servidor DHCP**. Al terminar, aparece un aviso para **completar la configuración de DHCP**.
- **PowerShell:**

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

- La consola de administración es `dhcpmgmt.msc` (Herramientas > DHCP).
- **El servidor debe tener IP fija** antes de instalar el rol.
- **En un dominio**, el servidor hay que **autorizarlo en Active Directory**. Si no está autorizado, no reparte IPs. Es una protección contra servidores DHCP de Windows no autorizados. En un grupo de trabajo, como mi práctica, no hace falta.

### Ámbito

Un **ámbito** es el conjunto de IPs que se reparten en una subred, junto con su configuración. Es el equivalente del bloque `subnet` de Linux.

| Campo del asistente | Valor en mi práctica |
|---|---|
| Nombre | `LAN-Laboratorio` |
| Dirección IP inicial | `192.168.10.210` |
| Dirección IP final | `192.168.10.219` |
| Longitud / máscara | `24` / `255.255.255.0` |
| Exclusión | `192.168.10.210` - `192.168.10.212` |
| Duración de la concesión | 8 días, el valor por defecto (ejemplo) |
| Enrutador, opción 003 | (ejemplo) |
| Servidores DNS, opción 006 | (ejemplo) |
| Activar el ámbito | Sí. **Un ámbito sin activar no reparte nada.** |

```
 Ámbito LAN-Laboratorio: 192.168.10.210 - 192.168.10.219

 .210  .211  .212 | .213  .214  .215  .216  .217  .218  .219
 [   EXCLUIDAS   ]|[       REPARTIBLES (7 direcciones)        ]
                     ^
                     primera libre: el cliente recibió .213
```

- El servidor suele entregar la **primera dirección libre**: por eso el cliente recibió `.213` y no otra.
- Quedan **7 direcciones** repartibles. Si se conecta un octavo cliente, no recibe IP; un Windows se autoasignaría una `169.254.x.x` (APIPA).
- Lo lógico es que el propio Windows Server tenga una de las IPs excluidas, por ejemplo `192.168.10.210` (ejemplo: comprobarlo en la práctica). Si su IP estuviera dentro del rango sin excluir, podría ofrecérsela a un cliente.

### Exclusiones y reservas

| | Exclusión | Reserva |
|---|---|---|
| Qué hace | Saca IPs del reparto automático | Ata una IP a una MAC concreta |
| Para qué | IPs puestas a mano: el servidor, el router, una impresora con IP fija | Equipos que necesitan siempre la misma IP |
| ¿El equipo usa DHCP? | No | Sí |
| Dónde se crea | Ámbito > Conjunto de direcciones > Nuevo intervalo de exclusión | Ámbito > Reservas > Nueva reserva |

**Crear una reserva:** Ámbito > Reservas > clic derecho > Nueva reserva.

| Campo | Ejemplo |
|---|---|
| Nombre de reserva | `cliente-ubuntu` |
| Dirección IP | `192.168.10.215` (ejemplo) |
| Dirección MAC | `080027AABBCC` (ejemplo; sin separadores o con guiones) |
| Tipos compatibles | DHCP |

En Windows, la IP reservada va dentro del rango del ámbito: el propio servidor se encarga de no dársela a otro equipo. Para que se aplique, el cliente tiene que liberar y renovar su IP.

### Opciones

Las opciones se pueden poner en varios niveles. Si una opción está en varios, **gana la más concreta**:

```
 Opciones de servidor      (todos los ámbitos)
        |
        v
 Opciones de ámbito        (solo LAN-Laboratorio)
        |
        v
 Opciones de reserva       (solo un equipo)      <-- la que gana
```

| Código | Opción | Equivalente en isc-dhcp-server |
|---|---|---|
| 003 | Enrutador (puerta de enlace) | `option routers` |
| 006 | Servidores DNS | `option domain-name-servers` |
| 015 | Nombre de dominio DNS | `option domain-name` |
| 044 | Servidores WINS | `option netbios-name-servers` |
| 066 / 067 | Servidor y fichero de arranque por red (PXE) | `next-server` u `option tftp-server-name` / `filename` |

La duración de la concesión no se pone como opción: se cambia en las propiedades del ámbito.

> [!TIP]
> En Propiedades de IPv4 > Opciones avanzadas está **Intentos de detección de conflictos**. Con un valor de 1 o 2, el servidor hace ping a la IP antes de ofrecerla y no la da si alguien responde.

### Lo mismo con PowerShell

```powershell
# Ámbito
Add-DhcpServerv4Scope -Name "LAN-Laboratorio" -StartRange 192.168.10.210 `
    -EndRange 192.168.10.219 -SubnetMask 255.255.255.0 -State Active

# Exclusión
Add-DhcpServerv4ExclusionRange -ScopeId 192.168.10.0 `
    -StartRange 192.168.10.210 -EndRange 192.168.10.212

# Reserva (ejemplo)
Add-DhcpServerv4Reservation -ScopeId 192.168.10.0 -IPAddress 192.168.10.215 `
    -ClientId "08-00-27-AA-BB-CC" -Name "cliente-ubuntu"

# Opciones de ámbito (ejemplo). -Force evita que valide que el DNS responde,
# algo que en una red interna sin Internet fallaría.
Set-DhcpServerv4OptionValue -ScopeId 192.168.10.0 -Router 192.168.10.1 `
    -DnsServer 8.8.8.8 -DnsDomain "asir.test" -Force

# Comprobar
Get-DhcpServerv4Scope
Get-DhcpServerv4Lease -ScopeId 192.168.10.0
```

En la consola gráfica, las IPs prestadas se ven en Ámbito > **Concesiones de direcciones**.

[⬆ Volver al índice](#índice)

---

## 9. Agente de retransmisión (DHCP relay)

**El problema:** el Discover es un broadcast y los routers no reenvían broadcasts. Con varias subredes habría que poner un servidor DHCP en cada una. La solución es un **agente de retransmisión** (*relay agent*): un equipo en la red del cliente que recoge sus broadcasts DHCP y se los reenvía en **unicast** al servidor.

En mis prácticas solo hay una subred. Este ejemplo añade una **subred B imaginaria** (`192.168.20.0/24`) para ver cómo funcionaría:

```
     Subred A: 172.16.5.0/24                       Subred B: 192.168.20.0/24 (ejemplo)

 +--------------------+       +----------------------------+       +--------------------+
 | UbuntuSRI          |       | Router con agente relay    |       | Cliente subred B   |
 | Servidor DHCP      |-------| eth0: 172.16.5.1           |-------| sin IP             |
 | 172.16.5.210       |       | eth1: 192.168.20.1         |       |                    |
 +--------------------+       +----------------------------+       +--------------------+

 1. Cliente   -> DISCOVER en broadcast. Por sí solo no pasaría del router.
 2. Relay     -> lo recibe por eth1 y lo reenvía en UNICAST a 172.16.5.210,
                 escribiendo en el campo giaddr su IP de esa red: 192.168.20.1
 3. Servidor  -> lee giaddr, busca el "subnet" que contiene 192.168.20.1
                 y ofrece una IP de 192.168.20.0/24
 4. Servidor  -> responde en unicast al relay (192.168.20.1)
 5. Relay     -> entrega la respuesta al cliente en la subred B
```

**Conceptos clave:**

- **`giaddr`** (*gateway IP address*): campo del mensaje DHCP donde el relay escribe la IP de la interfaz por la que le llegó el broadcast. Así el servidor sabe **de qué red viene el cliente**. Sin relay, el servidor usa la IP de su propia interfaz para elegir la subred ([apartado 5.4](#54-interfaz-de-escucha)); con relay, usa `giaddr`.
- **Opción 82** (*Relay Agent Information*): el relay puede añadir información extra, como el puerto del switch por el que llegó el cliente. Se usa para asignar IPs según el puerto y la usa DHCP snooping ([apartado 11](#11-seguridad-en-dhcp)).
- El servidor necesita una **ruta de vuelta** hacia la subred B, normalmente su puerta de enlace, que es el propio router.

### Configuración del servidor

El servidor necesita un bloque `subnet` para **cada** red a la que da servicio, aunque no tenga interfaz en ella:

```conf
# Red propia del servidor (ya existía)
subnet 172.16.5.0 netmask 255.255.255.0 {
    range 172.16.5.100 172.16.5.150;
    option routers 172.16.5.1;
}

# Red remota, atendida a través del relay (ejemplo)
subnet 192.168.20.0 netmask 255.255.255.0 {
    range 192.168.20.100 192.168.20.150;
    option routers 192.168.20.1;
}
```

### Configuración del relay

**Linux** (`isc-dhcp-relay`, del mismo proyecto obsoleto que `isc-dhcp-server`), en `/etc/default/isc-dhcp-relay`:

```bash
SERVERS="172.16.5.210"          # a quién reenviar
INTERFACES="eth0 eth1"          # las dos: la del cliente y la del servidor
OPTIONS=""
```

**Router Cisco:** en la interfaz que da a la red de los clientes.

```
interface GigabitEthernet0/1
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 172.16.5.210
```

**Windows Server:** con el rol de Enrutamiento y acceso remoto (RRAS), en IPv4 > **Agente de retransmisión DHCP**.

En redes reales, el relay casi siempre lo hace el router o el switch de capa 3 que ya separa las subredes.

[⬆ Volver al índice](#índice)

---

## 10. Conmutación por error (failover)

**El problema:** con un solo servidor DHCP, si se cae, los clientes pierden la IP cuando caducan sus concesiones ([apartado 4](#4-concesión-y-renovación)). La solución es tener **dos servidores**. Esto funciona gracias a la fase **REBINDING**: en T2 el cliente pregunta por broadcast a cualquier servidor, así que el segundo puede renovarle la concesión.

```
                       +----------------------+
                       |       Clientes       |
                       +----------+-----------+
                                  |
                +-----------------+-----------------+
                |                                   |
 +--------------+-------------+       +-------------+--------------+
 | Servidor principal         |<----->| Servidor secundario        |
 | 172.16.5.210               |  TCP  | 172.16.5.220 (ejemplo)     |
 | atiende el 50 %            |  647  | atiende el 50 %            |
 +----------------------------+       +----------------------------+

  Se sincronizan las concesiones. Si uno cae, el otro atiende a todos.
```

| Solución | Cómo funciona | A favor | En contra |
|---|---|---|---|
| **Ámbito dividido 80/20** | Dos servidores independientes con rangos que no se solapan: uno tiene el 80 % de las IPs y el otro el 20 % | Sencillo; vale cualquier software | No comparten concesiones; si cae el del 80 %, el otro solo tiene el 20 % de las IPs |
| **Failover de isc-dhcp-server** | Dos servidores (`primary` y `secondary`) se sincronizan por TCP | Comparten el estado de las concesiones | Obsoleto y delicado de configurar |
| **Failover de Windows Server** | Dos servidores Windows en modo equilibrio de carga o espera activa | Asistente gráfico sencillo | Solo entre servidores Windows |
| **Alta disponibilidad de Kea** | Hook *High Availability* con modos de equilibrio de carga, espera activa y copia pasiva | Moderno y mantenido | Más complejo de montar |

### Failover en isc-dhcp-server

En el servidor principal (`172.16.5.210`):

```conf
failover peer "failover-sri" {
    primary;
    address 172.16.5.210;
    port 647;
    peer address 172.16.5.220;          # (ejemplo) servidor secundario
    peer port 647;
    max-response-delay 60;
    max-unacked-updates 10;
    mclt 3600;
    split 128;
    load balance max seconds 3;
}

subnet 172.16.5.0 netmask 255.255.255.0 {
    option routers 172.16.5.1;
    pool {
        failover peer "failover-sri";
        range 172.16.5.100 172.16.5.150;
    }
}
```

| Directiva | Qué hace |
|---|---|
| `failover peer "nombre"` | Define la pareja de servidores. El nombre debe ser igual en los dos. |
| `primary;` / `secondary;` | Papel de cada servidor. |
| `address` / `peer address` | IP propia y del compañero. |
| `port` / `peer port` | Puerto TCP de la sincronización (647 es el registrado para esto). |
| `max-response-delay` | Segundos sin respuesta del compañero antes de darlo por caído. |
| `max-unacked-updates` | Actualizaciones que se pueden enviar sin confirmación. |
| `mclt` | *Maximum Client Lead Time*: cuánto puede alargar un servidor una concesión sin que el otro lo sepa. **Solo en el primario.** |
| `split 128` | Reparto de la carga entre 0 y 256: 128 es la mitad para cada uno. **Solo en el primario.** |
| `pool { failover peer ...; range ...; }` | El rango compartido tiene que ir dentro de un `pool` asociado a la pareja. |

El secundario lleva la misma configuración con `secondary;`, las IPs intercambiadas y **sin** `mclt` ni `split`. Los relojes de los dos servidores tienen que estar sincronizados (NTP).

### Failover en Windows Server

Clic derecho en el ámbito > **Configurar conmutación por error**:

| Modo | Cómo funciona | Valor por defecto |
|---|---|---|
| **Equilibrio de carga** | Los dos servidores reparten a la vez | 50 % cada uno |
| **Espera activa** | Uno reparte y el otro espera; se le reserva un porcentaje de IPs para cuando tome el relevo | 5 % reservado al de espera |

También se configura el MCLT y un **secreto compartido** para autenticar a los dos servidores entre sí.

[⬆ Volver al índice](#índice)

---

## 11. Seguridad en DHCP

DHCP **no tiene autenticación**: el cliente se fía del primero que le responde. De ahí salen los dos ataques clásicos.

> [!CAUTION]
> Estos ataques solo se prueban en una red aislada, como la Red interna de VirtualBox. Nunca en la red del centro ni en ninguna red ajena.

### Servidor DHCP falso (rogue DHCP)

Un equipo no autorizado responde a los Discover antes que el servidor legítimo.

```
 +-------------+   1. DISCOVER (broadcast)   +----------------------------+
 |             |---------------------------->| Servidor legítimo          |
 |             |                             | gw y DNS correctos         |
 |  Cliente    |                             +----------------------------+
 |             |
 |             |   2. OFFER (llega antes)    +----------------------------+
 |             |<----------------------------| Servidor falso             |
 +-------------+                             | gw = IP del atacante       |
                                             | DNS = IP del atacante      |
                                             +----------------------------+

 Resultado: el tráfico del cliente pasa por el atacante (man-in-the-middle)
```

| Lo que manda el servidor falso | Consecuencia |
|---|---|
| Puerta de enlace = el atacante | Todo el tráfico pasa por él: puede leerlo o modificarlo (*man-in-the-middle*) |
| DNS = el atacante | Puede redirigir a webs falsas (*phishing*) |
| Puerta de enlace inexistente | Los clientes se quedan sin red (denegación de servicio) |

**También puede ser accidental:** es exactamente lo que pasaría con mi VM `UbuntuSRI` en modo Puente con el servicio activo.

**Cómo detectarlo:**

```bash
sudo nmap --script broadcast-dhcp-discover -e enp0s3   # lista los servidores DHCP que responden
```

En una captura de Wireshark también se ve: llegan varios Offer con distinto identificador de servidor (opción 54).

### Agotamiento (DHCP starvation)

El atacante envía miles de Discover, cada uno con una **MAC falsa distinta**. El servidor presta una IP a cada "equipo" hasta que se queda sin direcciones y los clientes reales no reciben nada. Suele ser el primer paso de un rogue DHCP: se agota el servidor legítimo y luego responde el falso.

**Cómo detectarlo:**

- `dhcpd.leases` o `dhcp-lease-list` llenos de MACs desconocidas.
- En los logs de `isc-dhcp-server`: `no free leases`.
- Los clientes reales sin IP (en Windows, con una `169.254.x.x`).

### Defensa: DHCP snooping

Es una función de los **switches gestionables**. Divide los puertos en dos tipos:

```
                 +------------------------------------+
                 |          SWITCH (snooping)         |
                 |                                    |
                 |  Gi0/1: TRUSTED   Gi0/2-24:        |
                 |  (hacia el        UNTRUSTED        |
                 |   servidor real)  (usuarios)       |
                 +------+---------------------+-------+
                        |                     |
             +----------+---------+   +-------+------------+
             | Servidor DHCP      |   | PCs y posible      |
             | legítimo           |   | atacante           |
             +--------------------+   +--------------------+

 Por un puerto UNTRUSTED solo pasan mensajes de cliente
 (Discover, Request...). Un Offer o un Ack que entre por ahí
 se descarta: el servidor falso no puede responder a nadie.
```

| Función | Qué hace | Contra qué protege |
|---|---|---|
| Puertos *trusted* / *untrusted* | Solo los puertos de confianza pueden enviar respuestas de servidor (Offer, Ack, NAK) | Rogue DHCP |
| Límite de velocidad | Máximo de paquetes DHCP por segundo en cada puerto de usuario | Starvation |
| Comprobación de MAC | Descarta peticiones en las que la MAC del mensaje DHCP no coincide con la MAC real de la trama | Starvation con MACs falsas |
| Tabla de asignaciones (*binding table*) | Guarda qué IP, MAC, VLAN y puerto tiene cada cliente | Base para otras defensas |

Ejemplo en un switch Cisco:

```
ip dhcp snooping
ip dhcp snooping vlan 10
!
interface GigabitEthernet0/1
 description Hacia el servidor DHCP
 ip dhcp snooping trust
!
interface range GigabitEthernet0/2 - 24
 ip dhcp snooping limit rate 10
!
show ip dhcp snooping
show ip dhcp snooping binding
```

> [!NOTE]
> Si al activar DHCP snooping los clientes dejan de recibir IP, la causa típica es que el switch añade la opción 82 y el servidor descarta esos mensajes. Se soluciona con `no ip dhcp snooping information option`.

### Otras defensas

| Medida | Qué hace |
|---|---|
| **Port security** | Limita cuántas MACs puede haber en cada puerto del switch: frena el agotamiento |
| **Dynamic ARP Inspection (DAI)** | Usa la tabla de snooping para descartar respuestas ARP falsas |
| **IP Source Guard** | Usa la tabla de snooping para bloquear tráfico con IPs que no se asignaron por DHCP |
| **Autorización en Active Directory** | Impide que repartan IPs servidores DHCP de Windows no autorizados (no frena a un Linux) |
| **802.1X** | Solo se conectan a la red los equipos autenticados |
| **Segmentar en VLANs** | Un ataque solo afecta a su VLAN |
| **`reject` en `dhclient.conf`** | El cliente ignora a un servidor falso concreto ([apartado 7.2](#72-dhclientconf-línea-a-línea)) |

[⬆ Volver al índice](#índice)

---

## 12. Errores típicos y soluciones

| Síntoma | Causa | Solución |
|---|---|---|
| `isc-dhcp-server` falla justo después de instalarlo | Aún no tiene subred ni interfaz configuradas | Normal. Configurar `dhcpd.conf` e `INTERFACESv4` |
| Log: `No subnet declaration for enp0s3 (172.16.5.210)` | Ningún `subnet` contiene la IP de la interfaz | Añadir el `subnet` correcto o corregir la IP del servidor |
| Log: `Not configured to listen on any interfaces!` | `INTERFACESv4` vacío o con un nombre de interfaz que no existe | Mirar el nombre con `ip a` y corregirlo |
| El servicio no arranca y no se ve por qué | Error de sintaxis: falta un `;` o una llave | `sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf` indica la línea |
| El cliente no recibe IP | Las VMs no están en la misma Red interna (el nombre distingue mayúsculas), "cable conectado" desmarcado en VirtualBox, servidor en modo Puente o servicio parado | Revisar la configuración de red de las dos VMs. Con `tcpdump` en el servidor se ve si llega el Discover |
| El cliente recibe una IP `10.0.2.x` | Su adaptador está en modo NAT y responde el DHCP interno de VirtualBox | Poner el adaptador en Red interna |
| Los compañeros del aula reciben IPs raras | Mi servidor está activo en modo Puente | `sudo systemctl stop isc-dhcp-server` y `disable` antes de pasar a Puente |
| Dos VMs clonadas reciben la misma IP | Al clonar se copió la MAC o el `/etc/machine-id` (del que sale el DUID) | Generar MACs nuevas al clonar en VirtualBox; regenerar el machine-id; `dhcp-identifier: mac` en netplan |
| La reserva no se aplica | MAC mal escrita, servicio sin reiniciar o el cliente conserva su concesión anterior | Comprobar la MAC con `ip link`, reiniciar el servicio, liberar y renovar en el cliente |
| Dos equipos con la misma IP y una reserva | La IP reservada está dentro del `range` | Sacar la IP reservada del rango |
| El cliente tiene IP pero no Internet | `option routers` apunta a una puerta de enlace que no existe en `intnet` | Esperado en el laboratorio. Para tener Internet, el servidor tendría que hacer de router con NAT |
| Cambio `default-lease-time` y el cliente no se entera | El cambio llega en la siguiente renovación | Esperar a T1 o renovar a mano |
| Las horas de `dhcpd.leases` no cuadran | Están en UTC | Normal: sumar 2 horas en verano y 1 en invierno |
| Edito `dhclient.conf` o `dhcpcd.conf` y no cambia nada | NetworkManager no usa esos ficheros | Configurar con `nmcli` o desde la configuración gráfica |
| El servidor no puede usar el puerto 67 | Hay otro servidor DHCP activo (Kea, dnsmasq u otra instancia) | `sudo ss -ulpn \| grep ':67'` para ver quién lo usa |
| Windows: los clientes no reciben IP | El ámbito no está activado, o el servidor no está autorizado en el dominio | Activar el ámbito / autorizar en AD |
| Windows: error al poner el DNS en las opciones | Windows comprueba que el DNS responde y en la red interna no hay salida | Aceptar el aviso en la GUI o usar `-Force` en PowerShell |
| Clientes sin IP con el ámbito lleno | Rango pequeño (mi ámbito de Windows solo tiene 7 IPs) o agotamiento | Ampliar el rango, acortar la concesión o revisar si hay un ataque |

[⬆ Volver al índice](#índice)

---

## 13. Chuleta de comandos

### Servidor isc-dhcp-server

```bash
sudo apt install isc-dhcp-server                    # instalar
sudo nano /etc/dhcp/dhcpd.conf                      # configuración principal
sudo nano /etc/default/isc-dhcp-server              # interfaz de escucha (INTERFACESv4)
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf              # comprobar la sintaxis
sudo systemctl restart isc-dhcp-server              # aplicar los cambios
sudo systemctl status isc-dhcp-server               # estado
sudo systemctl stop isc-dhcp-server                 # parar (antes de pasar a modo Puente)
sudo systemctl disable isc-dhcp-server              # que no arranque solo al encender
journalctl -u isc-dhcp-server -f                    # ver DORA en directo
cat /var/lib/dhcp/dhcpd.leases                      # concesiones (horas en UTC)
dhcp-lease-list                                     # concesiones en forma de tabla
sudo ss -ulpn | grep ':67'                          # quién escucha en UDP 67
```

### Kea

```bash
sudo apt install kea-dhcp4-server                   # instalar
sudo nano /etc/kea/kea-dhcp4.conf                   # configuración (JSON)
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf           # comprobar la sintaxis
sudo systemctl restart kea-dhcp4-server             # aplicar los cambios
journalctl -u kea-dhcp4-server -f                   # logs
sudo cat /var/lib/kea/kea-leases4.csv               # concesiones
```

### Cliente Ubuntu Desktop (NetworkManager)

```bash
cat /etc/netplan/*.yaml                             # ver quién gestiona la red
nmcli device status                                 # interfaces y su estado
nmcli -f DHCP4 device show enp0s3                   # lo recibido por DHCP
sudo nmcli device disconnect enp0s3                 # soltar la IP
sudo nmcli device connect enp0s3                    # pedir IP de nuevo
ip a show enp0s3                                    # IP "dynamic" y valid_lft (cuenta atrás)
ip route                                            # puerta de enlace recibida
resolvectl status enp0s3                            # DNS recibidos
journalctl -u NetworkManager | grep -i dhcp         # el proceso DHCP en los logs
```

### Otros clientes Linux

```bash
sudo dhcpcd -T enp0s3          # dhcpcd: prueba sin aplicar nada
sudo dhcpcd -k enp0s3          # dhcpcd: liberar
sudo dhcpcd -n enp0s3          # dhcpcd: renovar
sudo dhclient -r enp0s3        # dhclient: liberar
sudo dhclient -v enp0s3        # dhclient: pedir IP mostrando el proceso
sudo networkctl renew enp0s3   # systemd-networkd: renovar
sudo netplan try               # probar cambios de netplan (vuelve atrás en 120 s)
sudo netplan apply             # aplicar cambios de netplan
```

### Diagnóstico

```bash
sudo tcpdump -i enp0s3 -n -e port 67 or port 68      # ver los mensajes DHCP con sus MACs
sudo tcpdump -i enp0s3 -n -vv port 67 or port 68     # ver además las opciones
sudo nmap --script broadcast-dhcp-discover -e enp0s3 # buscar servidores DHCP en la red
ip link show enp0s3                                  # MAC de la interfaz (para reservas)
```

### Windows Server (PowerShell)

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools          # instalar el rol
Get-DhcpServerv4Scope                                        # ver ámbitos
Get-DhcpServerv4Lease -ScopeId 192.168.10.0                  # ver concesiones
Get-DhcpServerv4ExclusionRange -ScopeId 192.168.10.0         # ver exclusiones
Get-DhcpServerv4Reservation -ScopeId 192.168.10.0            # ver reservas
Get-DhcpServerv4OptionValue -ScopeId 192.168.10.0            # ver opciones del ámbito
Restart-Service DHCPServer                                   # reiniciar el servicio
```

### Cliente Windows

```bat
ipconfig /all          :: IP, MAC, servidor DHCP y fechas de la concesión
ipconfig /release      :: liberar la IP
ipconfig /renew        :: pedir IP de nuevo
```

[⬆ Volver al índice](#índice)
