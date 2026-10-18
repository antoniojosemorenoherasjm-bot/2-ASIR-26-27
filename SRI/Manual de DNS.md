# Servicio DNS con BIND9

Apuntes de **Servicios de Red e Internet** (2º ASIR). Explicación básica de cómo se configura un servidor DNS con BIND9 en Ubuntu, usando mi laboratorio como ejemplo.

> Documento en construcción: se irá ampliando a medida que avance el tema en clase.

---

## Índice

1. [Qué es DNS](#que-es-dns)
2. [Mi laboratorio](#laboratorio)
3. [Los ficheros de BIND9](#ficheros)
4. [named.conf.options: comportamiento del servidor](#options)
5. [named.conf.local: las zonas](#local)
6. [El fichero de zona directa](#zona-directa)
7. [La zona inversa](#zona-inversa)
8. [Primario, esclavo y número de serie](#primario-esclavo)
9. [Consultas recursivas y forwarders](#recursion)
10. [Comprobar con dig](#dig)
11. [Errores típicos](#errores)
12. [Chuleta de comandos](#chuleta)

---

<a id="que-es-dns"></a>
## 1. Qué es DNS

DNS (*Domain Name System*) traduce **nombres** a **direcciones IP**, y al revés.

- **Resolución directa:** nombre a IP. `www.asir.test` → `172.16.6.210`
- **Resolución inversa:** IP a nombre. `172.16.6.210` → `ns1.asir.test`

Usa el **puerto 53** (UDP para consultas normales, TCP para respuestas grandes y transferencias de zona).

Algunos conceptos que aparecen en todo el documento:

| Concepto | Qué es |
|---|---|
| **Zona** | La parte del espacio de nombres de la que se encarga un servidor. Ejemplo: `asir.test` |
| **Registro** | Cada línea de información dentro de una zona. Ejemplo: "www tiene la IP 172.16.6.210" |
| **Servidor autoritativo** | El que es "dueño" de una zona y da la respuesta oficial |
| **Resolver / caché** | Servidor que busca respuestas de otras zonas y las guarda un tiempo |

[⬆ Volver al índice](#índice)

---

<a id="laboratorio"></a>
## 2. Mi laboratorio

Dos máquinas virtuales Ubuntu en VirtualBox, en la red `172.16.6.0/24`:

```
                     Red 172.16.6.0/24
   +-------------------------------------------------------+
   |                                                       |
   |  +---------------------+       +---------------------+ |
   |  |  ns1 (PRIMARIO)     |       |  ns2 (ESCLAVO)      | |
   |  |  172.16.6.210       |------>|  172.16.6.212       | |
   |  |  BIND9              | copia |  BIND9              | |
   |  |  Tiene el original  | zonas |  Copia del primario | |
   |  +---------------------+       +---------------------+ |
   |            |                                          |
   +------------|------------------------------------------+
                |
                v  (lo que no sabe, lo pregunta fuera)
        +----------------------+
        |  Forwarders          |
        |  8.8.8.8 / 1.1.1.1   |
        +----------------------+
```

| Dato | Valor |
|---|---|
| Dominio | `asir.test` |
| Servidor primario | `ns1.asir.test` → `172.16.6.210` |
| Servidor esclavo | `ns2.asir.test` → `172.16.6.212` |
| Web | `www.asir.test` (registro A) |
| FTP | `ftp.asir.test` (alias de www) |
| Zona inversa | `6.16.172.in-addr.arpa` |

[⬆ Volver al índice](#índice)

---

<a id="ficheros"></a>
## 3. Los ficheros de BIND9

Todo está en `/etc/bind/`. El fichero principal `named.conf` no se suele tocar: solo incluye a los demás.

```
/etc/bind/named.conf
      |
      +-- include "named.conf.options"        -> como se comporta el servidor
      +-- include "named.conf.local"          -> que zonas sirve
      +-- include "named.conf.default-zones"  -> zonas internas (no se toca)

/etc/bind/db.asir.test     -> fichero de zona directa  (nombres -> IP)
/etc/bind/db.172.16.6      -> fichero de zona inversa  (IP -> nombres)
```

Hay **dos tipos de fichero con sintaxis distinta**:

| Tipo | Ficheros | Comentarios | Detalle importante |
|---|---|---|---|
| Configuración | `named.conf.*` | `//`, `#` o `/* */` | Cada línea y cada bloque `{ }` termina en `;` |
| Zona | `db.*` | Solo `;` | El punto final de los nombres importa |

[⬆ Volver al índice](#índice)

---

<a id="options"></a>
## 4. named.conf.options: comportamiento del servidor

```
acl "red-asir" {
    172.16.6.0/24;
    localhost;
};

options {
    directory "/var/cache/bind";

    listen-on    { 127.0.0.1; 172.16.6.210; };
    listen-on-v6 { none; };

    recursion yes;
    allow-query     { red-asir; };
    allow-recursion { red-asir; };

    forwarders {
        8.8.8.8;
        1.1.1.1;
    };

    dnssec-validation auto;
};
```

| Línea | Qué hace |
|---|---|
| `acl "red-asir"` | Crea una lista de direcciones con nombre para no repetir IPs. Va **fuera** de `options` |
| `directory` | Carpeta de trabajo de BIND |
| `listen-on` | En qué IPs **del servidor** escucha BIND |
| `listen-on-v6 { none; }` | No escuchar por IPv6 |
| `recursion yes` | Permite que el servidor busque nombres que no son suyos (google.com, etc.) |
| `allow-query` | **Quién** puede hacer consultas |
| `allow-recursion` | **Quién** puede pedir nombres de fuera |
| `forwarders` | A qué servidores reenvía lo que no sabe |
| `dnssec-validation auto` | Comprueba las firmas de seguridad de las respuestas (DNSSEC) |

**No confundir:** `listen-on` es *por dónde escucho*; `allow-query` es *a quién contesto*.

[⬆ Volver al índice](#índice)

---

<a id="local"></a>
## 5. named.conf.local: las zonas

Aquí se declara **qué zonas tiene el servidor** y si es primario o esclavo de cada una.

### En el primario (172.16.6.210)

```
zone "asir.test" {
    type master;
    file "/etc/bind/db.asir.test";
    allow-transfer { 172.16.6.212; };
};

zone "6.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.172.16.6";
    allow-transfer { 172.16.6.212; };
};
```

### En el esclavo (172.16.6.212)

```
zone "asir.test" {
    type slave;
    file "/var/cache/bind/db.asir.test";
    masters { 172.16.6.210; };
};

zone "6.16.172.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.172.16.6";
    masters { 172.16.6.210; };
};
```

| Línea | Qué hace |
|---|---|
| `zone "asir.test"` | Nombre de la zona. Aquí **sin** punto final |
| `type master` | Este servidor tiene el original (también se escribe `primary`) |
| `type slave` | Este servidor copia la zona de otro (también se escribe `secondary`) |
| `file` | En el primario: fichero que escribo yo. En el esclavo: donde guarda la copia que recibe |
| `allow-transfer` | Quién puede copiar la zona entera. Solo el esclavo |
| `masters` | De qué servidor copia el esclavo (también se escribe `primaries`) |

En el esclavo, la ruta va en `/var/cache/bind/` porque BIND **no tiene permiso para escribir** en `/etc/bind/`.

[⬆ Volver al índice](#índice)

---

<a id="zona-directa"></a>
## 6. El fichero de zona directa

`/etc/bind/db.asir.test`:

```
$TTL    86400
$ORIGIN asir.test.
@       IN      SOA     ns1.asir.test. admin.asir.test. (
                        2026101801      ; Serial
                        3600            ; Refresh
                        900             ; Retry
                        1209600         ; Expire
                        86400 )         ; Negative Cache TTL
;
@       IN      NS      ns1.asir.test.
@       IN      NS      ns2.asir.test.

ns1     IN      A       172.16.6.210
ns2     IN      A       172.16.6.212
www     IN      A       172.16.6.210
ftp     IN      CNAME   www
```

### Símbolos

| Símbolo | Significado |
|---|---|
| `;` | Comentario |
| `@` | Equivale al nombre de la zona: `asir.test.` |
| `( )` | Permiten escribir un registro en varias líneas (se usa en el SOA) |
| `.` al final | El nombre está completo y no se le añade nada |
| `IN` | Clase *Internet*. Siempre es esta |

### El punto final

Un nombre **sin punto final** se completa añadiéndole el dominio. Un nombre **con punto final** se deja tal cual.

| Escribo | BIND entiende | |
|---|---|---|
| `www` | `www.asir.test.` | Correcto |
| `www.asir.test.` | `www.asir.test.` | Correcto |
| `www.asir.test` | `www.asir.test.asir.test.` | **Error** |

Regla: o el nombre corto, o el completo **con** punto.

### Las directivas del principio

| Directiva | Qué hace |
|---|---|
| `$TTL 86400` | Tiempo (en segundos) que otros servidores pueden guardar en caché los registros. 86400 s = 1 día |
| `$ORIGIN asir.test.` | Dominio que se añade a los nombres cortos. Si no se pone, se usa el nombre de la zona |

### El registro SOA

Es el primer registro de la zona y dice quién manda en ella.

```
@    IN    SOA    ns1.asir.test.     admin.asir.test.    ( ...numeros... )
                       |                    |
               servidor primario    email del responsable
```

| Campo | Valor | Significado |
|---|---|---|
| Servidor primario | `ns1.asir.test.` | Quién tiene el original |
| Email | `admin.asir.test.` | Significa `admin@asir.test`. El primer punto hace de `@` |
| Serial | `2026101801` | Versión de la zona. **Hay que subirlo en cada cambio**. Formato: año, mes, día y número de cambio del día |
| Refresh | `3600` | Cada cuánto el esclavo comprueba si hay cambios (1 hora) |
| Retry | `900` | Si no contacta con el primario, cada cuánto reintenta (15 min) |
| Expire | `1209600` | Si pasa este tiempo sin contactar, el esclavo deja de responder (14 días) |
| Negative TTL | `86400` | Cuánto tiempo se guarda en caché una respuesta de "no existe" |

### Tipos de registro

Cada línea sigue el formato `nombre  clase  tipo  dato`:

| Tipo | Para qué sirve | Ejemplo |
|---|---|---|
| `NS` | Indica los servidores DNS de la zona | `@ IN NS ns1.asir.test.` |
| `A` | Nombre → IPv4 | `www IN A 172.16.6.210` |
| `AAAA` | Nombre → IPv6 | `www IN AAAA fd00::10` |
| `CNAME` | Alias: un nombre que apunta a otro nombre | `ftp IN CNAME www` |
| `MX` | Servidor de correo. El número es la prioridad (menor = preferente) | `@ IN MX 10 mail.asir.test.` |
| `TXT` | Texto libre (verificaciones, SPF...) | `@ IN TXT "texto"` |
| `PTR` | IP → nombre. Solo en la zona inversa | `210 IN PTR ns1.asir.test.` |

Reglas básicas:

- Cada servidor que aparece en un `NS` necesita su registro `A`.
- Un nombre con `CNAME` no puede tener ningún otro registro.
- `MX` y `NS` deben apuntar a un nombre con registro `A`, nunca a un `CNAME`.

[⬆ Volver al índice](#índice)

---

<a id="zona-inversa"></a>
## 7. La zona inversa

Sirve para preguntar "¿de quién es esta IP?".

### Cómo se forma el nombre

Se toma la parte de red de la IP, **se le da la vuelta** y se añade `in-addr.arpa`:

```
  Red:             172 . 16 . 6 . 0/24
  Parte de red:    172 . 16 . 6
  Al reves:          6 . 16 . 172
  Nombre zona:       6 . 16 . 172 . in-addr.arpa
```

Otro ejemplo: la red `192.168.10.0/24` tendría la zona `10.168.192.in-addr.arpa`.

### El fichero

`/etc/bind/db.172.16.6`:

```
$TTL    86400
@       IN      SOA     ns1.asir.test. admin.asir.test. (
                        2026101801      ; Serial
                        3600            ; Refresh
                        900             ; Retry
                        1209600         ; Expire
                        86400 )         ; Negative Cache TTL
;
@       IN      NS      ns1.asir.test.
@       IN      NS      ns2.asir.test.

210     IN      PTR     ns1.asir.test.
212     IN      PTR     ns2.asir.test.
```

- El SOA y los NS son iguales que en la zona directa.
- `210` es el último octeto de la IP. Como no lleva punto, se completa: `210.6.16.172.in-addr.arpa.`
- El nombre al que apunta el `PTR` **siempre lleva punto final**. Si no, BIND añadiría `6.16.172.in-addr.arpa.` detrás.

[⬆ Volver al índice](#índice)

---

<a id="primario-esclavo"></a>
## 8. Primario, esclavo y número de serie

El esclavo tiene una copia de las zonas. Si el primario cae, el esclavo sigue respondiendo.

El esclavo **solo copia la zona si el serial del primario es mayor que el suyo**. Por eso, si cambio la zona y no subo el serial, el esclavo se queda con la versión antigua.

```
  PRIMARIO 172.16.6.210                       ESCLAVO 172.16.6.212
  ---------------------                       --------------------
  1. Edito db.asir.test
  2. Subo el serial
  3. rndc reload
  4. Aviso (NOTIFY) -------------------------> "ha cambiado algo"
                                               5. Pido el SOA
                       <-----------------------
  6. Envio el SOA ---------------------------->
                                               7. Serial mayor que el mio?
                                                  Si -> pido la zona
                       <-----------------------
  8. Envio la zona --------------------------->
                                               9. La guardo
```

Para comprobar que los dos tienen la misma versión, se compara el serial (tercer número):

```
dig +short asir.test SOA @172.16.6.210
dig +short asir.test SOA @172.16.6.212
```

[⬆ Volver al índice](#índice)

---

<a id="recursion"></a>
## 9. Consultas recursivas y forwarders

| Tipo | Qué significa |
|---|---|
| **Recursiva** | "Dame la respuesta final, búscala tú". La hace el cliente a su servidor DNS |
| **Iterativa** | "Dime lo que sepas o a quién preguntar". La hacen los servidores entre ellos |

Qué pasa cuando pregunto a mi servidor por `www.google.com`:

```
  Cliente              Mi BIND               8.8.8.8          Raiz / .com / google
     |                    |                      |                      |
     |-- 1. pregunta ---->|                      |                      |
     |                    | 2. Es mio? No.       |                      |
     |                    |    En cache? No.     |                      |
     |                    |-- 3. reenvia ------->|                      |
     |                    |                      |-- 4. busca --------->|
     |                    |                      |<---------------------|
     |                    |<-- 5. respuesta -----|                      |
     |                    | 6. La guarda en cache|                      |
     |<-- 7. respuesta ---|                      |                      |
```

- Los **forwarders** (8.8.8.8 y 1.1.1.1) hacen la búsqueda por mi servidor.
- La respuesta se guarda en **caché**: la segunda vez que se pregunta, responde al momento.
- Se limita `allow-recursion` a mi red para no tener un **resolver abierto**, que cualquiera de Internet podría usar para hacer ataques.

[⬆ Volver al índice](#índice)

---

<a id="dig"></a>
## 10. Comprobar con dig

`dig` hace **una consulta DNS** y muestra la respuesta completa del servidor.

```
dig  [@servidor]  nombre  [tipo]
```

| Parte | Qué es | Si no lo pongo |
|---|---|---|
| `@172.16.6.210` | A qué servidor pregunto | Usa el DNS del sistema |
| `www.asir.test` | Qué nombre busco | Obligatorio |
| `A`, `MX`, `NS`, `SOA`... | Qué tipo de registro quiero | Pide `A` |
| `-x 172.16.6.210` | Consulta inversa | |

### Ejemplo de salida

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41231
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;www.asir.test.                 IN      A

;; ANSWER SECTION:
www.asir.test.          86400   IN      A       172.16.6.210

;; Query time: 1 msec
;; SERVER: 172.16.6.210#53(172.16.6.210) (UDP)
```

### Qué mirar

**1. El `status`:**

| Status | Significado |
|---|---|
| `NOERROR` | Todo correcto |
| `NXDOMAIN` | El nombre no existe |
| `REFUSED` | El servidor no me quiere contestar (no estoy en `allow-query`) |
| `SERVFAIL` | El servidor ha fallado (zona con errores, sin conexión a los forwarders...) |

**2. Los `flags`:**

| Flag | Significado |
|---|---|
| `aa` | Respuesta autoritativa: el servidor es dueño de la zona |
| `rd` | He pedido recursión |
| `ra` | El servidor me permite recursión |

**3. La `ANSWER SECTION`:** la respuesta, con el mismo formato que el fichero de zona:

```
www.asir.test.     86400     IN      A       172.16.6.210
    nombre          TTL     clase   tipo         dato
```

**4. El `SERVER`:** qué servidor ha contestado de verdad.

[⬆ Volver al índice](#índice)

---

<a id="errores"></a>
## 11. Errores típicos

| Síntoma | Causa | Solución |
|---|---|---|
| `named-checkconf` da error | Falta un `;` o una `}` | Revisar la línea que indica el error |
| Un nombre aparece como `www.asir.test.asir.test.` | Falta el punto final | Poner el nombre corto o terminarlo en `.` |
| El esclavo no ve los cambios | No se ha subido el serial | Subir el serial y hacer `rndc reload` |
| El esclavo no recibe la zona | `allow-transfer` mal o ruta en `/etc/bind` | Revisar `allow-transfer` y usar `/var/cache/bind/` |
| `status: REFUSED` | La IP del cliente no está permitida | Revisar `allow-query` / `allow-recursion` |
| `status: SERVFAIL` en mi zona | La zona no cargó por un error | `named-checkzone` y revisar `journalctl -u named` |
| La inversa no funciona | Nombre de zona mal invertido o sin `in-addr.arpa` | Revisar el nombre en `named.conf.local` |
| Cambio un registro y no se nota | No he recargado BIND | `sudo rndc reload` |

[⬆ Volver al índice](#índice)

---

<a id="chuleta"></a>
## 12. Chuleta de comandos

### Instalación

```bash
sudo apt install bind9 bind9-utils dnsutils
```

### Comprobar la configuración

```bash
sudo named-checkconf
sudo named-checkzone asir.test /etc/bind/db.asir.test
sudo named-checkzone 6.16.172.in-addr.arpa /etc/bind/db.172.16.6
```

### Gestionar el servicio

```bash
sudo systemctl restart named
sudo systemctl status named
sudo rndc reload                 # recargar todas las zonas
sudo rndc reload asir.test       # recargar solo una zona
sudo journalctl -u named -f      # ver el log en directo
```

### Consultas con dig

```bash
dig www.asir.test @172.16.6.210            # registro A
dig ftp.asir.test @172.16.6.210            # CNAME
dig asir.test NS @172.16.6.210             # servidores DNS de la zona
dig asir.test SOA @172.16.6.210            # SOA (ver el serial)
dig -x 172.16.6.210 @172.16.6.210          # consulta inversa
dig www.google.com @172.16.6.210           # recursiva (usa los forwarders)
dig +short www.asir.test @172.16.6.210     # solo la respuesta
dig www.asir.test @172.16.6.212            # comprobar el esclavo
```

[⬆ Volver al índice](#índice)
