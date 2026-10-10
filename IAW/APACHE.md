# 🔴 APACHE — Documentación completa en Ubuntu

> **Asignatura:** Implantación de Aplicaciones Web · 2º ASIR
> **Entorno:** Ubuntu Server · todo desde la terminal
> **Temas:** servidor web · estructura de archivos · configuración · módulos · virtual hosts

---

## 📑 Índice

1. ¿Qué es Apache y para qué sirve?
2. Cómo funciona por dentro (arquitectura y MPM)
3. Apache vs Nginx: el contraste
4. Instalación rápida
5. Mapa de archivos y qué vamos a crear
6. Los archivos de configuración, uno por uno
7. Virtual hosts en detalle
8. Extras que conviene saber
9. Tabla comparativa Nginx vs Apache

---

## 1. 🧭 ¿Qué es Apache y para qué sirve?

**Apache HTTP Server** (en Ubuntu, el paquete y el servicio se llaman **`apache2`**) es uno de los servidores web más veteranos y extendidos. Nació en 1995 a partir del servidor NCSA HTTPd y hoy lo mantiene la **Apache Software Foundation** como software libre.

Recibe peticiones HTTP/HTTPS de los clientes (navegadores, `curl`, aplicaciones) y devuelve una respuesta: una página, un archivo o el resultado de ejecutar una aplicación.

### 🛠️ Para qué se usa

| Uso | Qué significa |
|---|---|
| 🌐 **Servidor web** | Sirve páginas y archivos estáticos (HTML, CSS, JS, imágenes) |
| 🐘 **Contenido dinámico** | Ejecuta aplicaciones (PHP y otras) mediante módulos o procesos externos |
| 🔀 **Proxy inverso** | Reenvía peticiones a otra aplicación (con `mod_proxy`) |
| 🔒 **HTTPS** | Gestiona el cifrado TLS (con `mod_ssl`) |
| 🧩 **Plataforma modular** | Se amplía activando módulos: reescritura de URLs, cabeceras, autenticación... |

> 💡 **Idea clave:** el punto fuerte de Apache es su **modularidad** y su **configuración flexible**, incluida la configuración por directorio mediante `.htaccess`. Es el servidor clásico de la pila **LAMP** (Linux + Apache + MySQL + PHP).

---

## 2. ⚙️ Cómo funciona por dentro (arquitectura y MPM)

Apache usa un **proceso padre** y varios **procesos hijo** (y, según el modelo, hilos dentro de ellos) para atender las conexiones.

```
                 ┌─────────────────────────────┐
                 │      PROCESO PADRE          │
                 │  lee la configuración,      │
                 │  abre los puertos y crea    │
                 │  los hijos (corre como root)│
                 └──────────────┬──────────────┘
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
   │   HIJO 1    │       │   HIJO 2    │       │   HIJO N    │
   │ atiende     │       │ atiende     │       │ atiende     │
   │ peticiones  │       │ peticiones  │       │ peticiones  │
   │ (www-data)  │       │ (www-data)  │       │ (www-data)  │
   └─────────────┘       └─────────────┘       └─────────────┘
```

Lo puedes comprobar tú mismo:

```bash
ps aux | grep apache2
```

Debes ver un proceso `apache2` de **root** (el padre) y varios de **www-data** (los hijos).

### 🧬 Los MPM: el "motor" de Apache

Apache no tiene un único modelo de funcionamiento. Se elige con un módulo llamado **MPM** (*Multi-Processing Module*):

| MPM | Cómo trabaja | Cuándo se usa |
|---|---|---|
| `prefork` | Un **proceso** por conexión, sin hilos | Compatibilidad con módulos no seguros para hilos (típicamente `mod_php`) |
| `worker` | Varios procesos, cada uno con **varios hilos** | Mayor eficiencia que `prefork` |
| `event` | Como `worker`, pero gestiona mejor las conexiones inactivas (*keep-alive*) | **El más moderno y el habitual por defecto** |

Para ver cuál estás usando:

```bash
apache2ctl -V | grep -i mpm
```

> ⚠️ **Matiz importante:** si instalas `libapache2-mod-php`, Ubuntu cambia el MPM a `prefork`. Si usas PHP-FPM, puedes mantener `event`. Es una decisión de arquitectura que conviene saber explicar.

---

## 3. 🥊 Apache vs Nginx: el contraste

Siguiendo el símil del restaurante:

- **Apache:** un camarero (proceso o hilo) asignado a cada cliente. Muy flexible, cada camarero puede hacer de todo, pero consume más con mucha gente.
- **Nginx:** pocos camareros que atienden muchas mesas a la vez sin quedarse esperando.

| Aspecto | 🔴 Apache | 🟢 Nginx |
|---|---|---|
| **Filosofía** | Procesos/hilos configurables (MPM) | Eventos asíncronos |
| **Configuración por directorio** | ✅ `.htaccess` | ❌ No existe |
| **Módulos** | Se activan/desactivan en caliente con `a2enmod` / `a2dismod` | Mayoría integrados o dinámicos, cargados al arrancar |
| **Contenido dinámico** | `mod_php` o PHP-FPM | Solo PHP-FPM (FastCGI) |
| **Contenido estático con mucha carga** | Bueno | Muy eficiente |
| **Uso típico** | Alojamiento compartido, aplicaciones que dependen de `.htaccess` | Proxy inverso, balanceo, webs con mucho tráfico |

> ⚠️ **Matiz para no decir un tópico:** con el MPM `event`, Apache ya trabaja de forma mucho más eficiente y la diferencia de rendimiento no es tan grande como se suele contar. La diferencia real está en el **modelo de configuración** (`.htaccess`, módulos) y en el papel de cada uno como proxy inverso.

---

## 4. 📦 Instalación rápida

```bash
sudo apt update
sudo apt install apache2 -y
```

- `apt update` actualiza la lista de paquetes disponibles.
- `apt install apache2` instala el servidor, crea el usuario `www-data` y lo deja arrancado y habilitado.

### ✅ Comprobar que funciona

```bash
apache2 -v                       # versión instalada
systemctl status apache2         # estado del servicio
curl -I http://localhost         # cabeceras de respuesta
sudo ss -tlnp | grep apache2     # puertos en los que escucha
```

> ✅ **Resultado esperado:** `active (running)` en el estado y `HTTP/1.1 200 OK` en el `curl`. En el navegador verás la página *"Apache2 Ubuntu Default Page"*.

### ⚠️ Aviso típico: *"Could not reliably determine the server's fully qualified domain name"*

Es un aviso (no un error) que aparece porque Apache no tiene un `ServerName` global. Se soluciona así:

```bash
echo "ServerName localhost" | sudo tee /etc/apache2/conf-available/servername.conf
sudo a2enconf servername
sudo systemctl reload apache2
```

### 🚧 Si ya tienes Nginx instalado en la misma máquina

Los dos intentarán usar el **puerto 80** y el segundo en arrancar fallará con `Address already in use`. Para trabajar con Apache:

```bash
sudo systemctl stop nginx          # para Nginx mientras pruebas Apache
sudo systemctl start apache2
```

> 💡 Si quieres que convivan, uno de ellos debe usar otro puerto.

### 🔥 Firewall (solo si usas UFW)

```bash
sudo ufw app list                  # perfiles: Apache, Apache Full, Apache Secure
sudo ufw allow 'Apache'            # abre el puerto 80
sudo ufw allow 8080/tcp            # abre el 8080 (lo necesitaremos en la práctica)
```

| Perfil | Puertos |
|---|---|
| `Apache` | 80 (HTTP) |
| `Apache Secure` | 443 (HTTPS) |
| `Apache Full` | 80 y 443 |

---

## 5. 🗺️ Mapa de archivos y qué vamos a crear

### Estructura que deja la instalación

```
/etc/apache2/
├── apache2.conf            ← configuración PRINCIPAL
├── ports.conf              ← puertos en los que escucha
├── envvars                 ← variables de entorno (usuario, logs, PID...)
├── sites-available/        ← TODOS los sitios definidos
│   ├── 000-default.conf
│   └── default-ssl.conf
├── sites-enabled/          ← solo los ACTIVOS (enlaces simbólicos)
│   └── 000-default.conf -> ../sites-available/000-default.conf
├── mods-available/         ← TODOS los módulos disponibles (.load y .conf)
├── mods-enabled/           ← solo los módulos ACTIVOS (enlaces)
├── conf-available/         ← fragmentos de configuración disponibles
└── conf-enabled/           ← fragmentos activos (enlaces)

/var/www/html/              ← carpeta web por defecto
/var/log/apache2/           ← registros (access.log y error.log)
```

> 💡 **Patrón a recordar:** Apache repite el esquema *available / enabled* en **tres** sitios (sitios, módulos y configuraciones). Cada uno tiene su pareja de comandos para activar y desactivar.

| Qué se gestiona | Activar | Desactivar |
|---|---|---|
| Sitios (virtual hosts) | `a2ensite` | `a2dissite` |
| Módulos | `a2enmod` | `a2dismod` |
| Configuraciones | `a2enconf` | `a2disconf` |

### 📝 Lo que vamos a crear nosotros (adelanto)

| Qué | Dónde | Para qué |
|---|---|---|
| Carpeta web del sitio | `/var/www/dominioa/` | Guardar el contenido de la web |
| Página de inicio | `/var/www/dominioa/index.html` | Lo que se verá al entrar |
| Archivo del sitio | `/etc/apache2/sites-available/dominioa.conf` | Definir el virtual host |
| Puerto extra | `/etc/apache2/ports.conf` (`Listen 8080`) | Abrir el puerto 8080 |
| Activación | `sudo a2ensite dominioa.conf` | Crea el enlace en `sites-enabled` |

> ⚠️ **Detalle importante:** en Apache/Ubuntu los archivos de sitio **deben terminar en `.conf`**. Si no, `a2ensite` no los reconoce y Apache no los carga. En Nginx no hacía falta.

---

## 6. 🔍 Los archivos de configuración, uno por uno

> 💡 **Truco para estudiar:** para ver un archivo sin comentarios ni líneas vacías:
> ```bash
> grep -v '^\s*#' /etc/apache2/apache2.conf | grep -v '^\s*$'
> ```

### 6.1 📄 `/etc/apache2/apache2.conf` — configuración principal

Define el comportamiento **global** y se encarga de **incluir** todo lo demás.

> ℹ️ Ejemplo orientativo sin comentarios: el contenido exacto puede variar según la versión. Compáralo con el tuyo usando `cat`.

```apache
DefaultRuntimeDir ${APACHE_RUN_DIR}
PidFile ${APACHE_PID_FILE}
Timeout 300
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5

User ${APACHE_RUN_USER}
Group ${APACHE_RUN_GROUP}

HostnameLookups Off
ErrorLog ${APACHE_LOG_DIR}/error.log
LogLevel warn

IncludeOptional mods-enabled/*.load
IncludeOptional mods-enabled/*.conf
Include ports.conf

<Directory />
    Options FollowSymLinks
    AllowOverride None
    Require all denied
</Directory>

<Directory /usr/share>
    AllowOverride None
    Require all granted
</Directory>

<Directory /var/www/>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

AccessFileName .htaccess

<FilesMatch "^\.ht">
    Require all denied
</FilesMatch>

LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined

IncludeOptional conf-enabled/*.conf
IncludeOptional sites-enabled/*.conf
```

#### 🧩 Qué hace cada bloque

| Directiva | Qué hace |
|---|---|
| `DefaultRuntimeDir` / `PidFile` | Dónde guarda Apache los archivos de ejecución y el PID del proceso padre |
| `Timeout 300` | Segundos máximos esperando una operación de red |
| `KeepAlive On` | Permite reutilizar una misma conexión para varias peticiones |
| `MaxKeepAliveRequests 100` | Peticiones máximas por conexión reutilizada |
| `KeepAliveTimeout 5` | Segundos que se espera la siguiente petición antes de cerrar |
| `User` / `Group` | Usuario y grupo con los que corren los procesos hijo (`www-data`, definido en `envvars`) |
| `HostnameLookups Off` | No resuelve nombres DNS de los clientes en los logs (más rápido) |
| `ErrorLog` / `LogLevel` | Archivo de errores y nivel de detalle (`warn`) |
| `IncludeOptional mods-enabled/*.load` y `*.conf` | **Carga los módulos activos** |
| `Include ports.conf` | Incluye los puertos de escucha |
| `<Directory />` | Por defecto, **prohíbe el acceso a todo el sistema de archivos** |
| `<Directory /var/www/>` | Permite el acceso a `/var/www/` |
| `AccessFileName .htaccess` | Nombre del archivo de configuración por directorio |
| `<FilesMatch "^\.ht">` | Impide que se descarguen los archivos `.htaccess` y `.htpasswd` |
| `LogFormat` | Define el formato de los registros de acceso |
| `IncludeOptional conf-enabled/*.conf` | Carga las configuraciones activas |
| `IncludeOptional sites-enabled/*.conf` | **Carga los sitios activos** (aquí se conectan los virtual hosts) |

> 🔐 **Política de seguridad que conviene entender:** Apache parte de "**todo prohibido**" (`Require all denied` en `/`) y va **concediendo** acceso solo a las carpetas necesarias. Por eso, si sirves una web desde una carpeta fuera de `/var/www/`, tendrás que añadir un bloque `<Directory>` que lo permita, o verás un **403**.

#### 🌳 Jerarquía de la configuración

A diferencia de Nginx (que usa `{ }`), Apache usa **etiquetas tipo XML** para agrupar directivas:

```
Configuración global (apache2.conf)
├── <Directory /var/www/>          ← reglas para una carpeta
├── <VirtualHost *:80>             ← un virtual host
│   ├── <Directory /var/www/sitio> ← reglas de una carpeta dentro del sitio
│   ├── <Location /admin>          ← reglas para una URL
│   └── <Files "secreto.txt">      ← reglas para un archivo
└── <VirtualHost *:8080>           ← otro virtual host

.htaccess  →  reglas por directorio, leídas en cada petición
```

> ⚠️ **Sintaxis:** las directivas **no** llevan `;` al final. Los bloques se abren con `<Etiqueta>` y se cierran con `</Etiqueta>`. Los comentarios empiezan por `#`.

---

### 6.2 📄 `/etc/apache2/ports.conf` — puertos de escucha

```apache
Listen 80

<IfModule ssl_module>
    Listen 443
</IfModule>

<IfModule mod_gnutls.c>
    Listen 443
</IfModule>
```

| Directiva | Qué hace |
|---|---|
| `Listen 80` | Apache abre el puerto 80 y espera conexiones |
| `<IfModule ssl_module>` | Solo abre el 443 **si** el módulo SSL está activo |

> 🎯 **Diferencia clave con Nginx:** en Nginx, `listen` está dentro de cada `server`. En Apache, **`Listen` (en `ports.conf`) abre el puerto** y **`<VirtualHost *:puerto>` decide qué sitio responde** en él. Si en un virtual host usas el puerto 8080 pero olvidas `Listen 8080`, **el sitio no funcionará**.

---

### 6.3 📄 `/etc/apache2/envvars` — variables de entorno

Define valores que luego se usan en el resto de archivos con la sintaxis `${VARIABLE}`.

| Variable | Valor habitual | Para qué |
|---|---|---|
| `APACHE_RUN_USER` | `www-data` | Usuario de los procesos hijo |
| `APACHE_RUN_GROUP` | `www-data` | Grupo de los procesos hijo |
| `APACHE_PID_FILE` | `/var/run/apache2/apache2.pid` | Archivo del PID |
| `APACHE_LOG_DIR` | `/var/log/apache2` | Carpeta de logs |

> 💡 Por eso en `apache2.conf` y en los sitios aparece `${APACHE_LOG_DIR}/error.log` en lugar de una ruta fija.

---

### 6.4 📄 `/etc/apache2/sites-available/000-default.conf` — el sitio por defecto

Contiene un bloque `<VirtualHost>`. Sin comentarios, es básicamente esto:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

#### 🧩 Directiva por directiva

| Directiva | Qué hace |
|---|---|
| `<VirtualHost *:80>` | Define un sitio que atiende peticiones en **cualquier IP (`*`) por el puerto 80** |
| `ServerAdmin` | Correo del administrador (se muestra en algunas páginas de error) |
| `DocumentRoot` | Carpeta raíz de donde se sirven los archivos (equivale a `root` en Nginx) |
| `ErrorLog` | Archivo donde se guardan los errores de este sitio |
| `CustomLog ... combined` | Registro de accesos con el formato `combined` |
| `ServerName` (comentada por defecto) | Nombre con el que se identifica el sitio |

> 💡 **¿Por qué se llama `000-default.conf`?** Porque Apache carga los archivos de `sites-enabled/` **por orden alfabético**, y el **primer virtual host** de cada IP:puerto es el que atiende lo que no coincide con ningún otro. El `000` garantiza que sea el primero.

---

### 6.5 🔗 `sites-enabled/` — los sitios activos

Igual que en Nginx: no contiene archivos reales, sino **enlaces simbólicos** a `sites-available/`. La diferencia es que en Apache **no se crean a mano**, los gestionan comandos:

```bash
sudo a2ensite dominioa.conf      # activa: crea el enlace
sudo a2dissite dominioa.conf     # desactiva: elimina el enlace
ls -l /etc/apache2/sites-enabled
```

| Carpeta | Función |
|---|---|
| `sites-available/` | El "catálogo": todos los sitios escritos, activos o no |
| `sites-enabled/` | Solo los que **están funcionando** |

> 💡 Tras `a2ensite` o `a2dissite`, Apache te recuerda que hay que **recargar** el servicio para que el cambio surta efecto.

---

### 6.6 🧩 `mods-available/` y `mods-enabled/` — los módulos

Apache es modular: casi todo lo que hace lo hace un **módulo**. Cada módulo tiene normalmente dos archivos:

| Archivo | Contenido |
|---|---|
| `nombre.load` | La orden que **carga** el módulo |
| `nombre.conf` | Su **configuración** (opcional) |

```bash
ls /etc/apache2/mods-enabled           # módulos activos
apache2ctl -M                          # módulos cargados en este momento
sudo a2enmod rewrite                   # activar un módulo
sudo a2dismod rewrite                  # desactivarlo
sudo systemctl restart apache2         # los módulos requieren reinicio
```

> ⚠️ Al activar o desactivar un **módulo**, es más seguro hacer `restart` que `reload`.

#### Módulos que conviene conocer

| Módulo | Para qué sirve |
|---|---|
| `rewrite` | Reescribir y redirigir URLs (muy usado con `.htaccess`) |
| `ssl` | Habilitar HTTPS |
| `headers` | Añadir o modificar cabeceras HTTP |
| `proxy` + `proxy_http` | Funcionar como proxy inverso |
| `php*` (`libapache2-mod-php`) | Ejecutar PHP dentro de Apache |
| `status` | Página de estado del servidor |
| `deflate` | Comprimir respuestas |

---

### 6.7 🧱 `conf-available/` y `conf-enabled/` — configuraciones sueltas

Fragmentos de configuración global que se pueden activar y desactivar como los sitios:

```bash
sudo a2enconf servername
sudo a2disconf servername
```

Archivos que trae por defecto, por ejemplo:

| Archivo | Para qué |
|---|---|
| `security.conf` | Ajustes de seguridad (`ServerTokens`, `ServerSignature`...) |
| `charset.conf` | Codificación de caracteres por defecto |
| `other-vhosts-access-log.conf` | Registro de accesos de los virtual hosts |
| `serve-cgi-bin.conf` | Soporte para scripts CGI |

---

### 6.8 🧷 `.htaccess` — configuración por directorio

Es **la gran diferencia con Nginx**. Un archivo `.htaccess` colocado dentro de una carpeta modifica el comportamiento de Apache **para esa carpeta y sus subcarpetas**, sin tocar la configuración principal y sin reiniciar nada.

Ejemplo:

```apache
# Impedir que se liste el contenido de la carpeta
Options -Indexes

# Página de error personalizada
ErrorDocument 404 /404.html
```

#### Cuándo se lee

Solo se aplica si la directiva `AllowOverride` del bloque `<Directory>` correspondiente lo permite:

| Valor | Efecto |
|---|---|
| `AllowOverride None` | **Se ignora** el `.htaccess` (es el valor por defecto en `/var/www/`) |
| `AllowOverride All` | Se permite que el `.htaccess` modifique casi cualquier cosa |

Para activarlo en un sitio, dentro de su `<Directory>`:

```apache
<Directory /var/www/dominioa>
    AllowOverride All
</Directory>
```

#### ⚖️ Ventajas e inconvenientes

| ✅ Ventajas | ❌ Inconvenientes |
|---|---|
| Se cambia la configuración sin acceso a la configuración principal | Apache lo lee **en cada petición** y en cada carpeta: más lento |
| Muy usado por aplicaciones web (WordPress, etc.) | Más difícil de auditar y de mantener |
| No requiere reiniciar el servicio | Un error de sintaxis provoca un **500 Internal Server Error** |

> 🎯 **Recomendación profesional:** si tienes acceso a la configuración del servidor, es mejor poner las reglas en el `<VirtualHost>` o en un `<Directory>` y dejar `AllowOverride None`. Es más rápido y más seguro. `.htaccess` es útil cuando **no** tienes ese acceso.

---

### 6.9 🧱 Contenedores: `<Directory>`, `<Location>` y `<Files>`

Sirven para aplicar reglas a una parte concreta:

| Contenedor | Se aplica a... | Ejemplo |
|---|---|---|
| `<Directory ruta>` | Una carpeta del **sistema de archivos** | `<Directory /var/www/dominioa>` |
| `<Location url>` | Una **URL** de la web | `<Location /admin>` |
| `<Files nombre>` | Un **archivo** concreto | `<Files "config.php">` |

Directivas habituales dentro de un `<Directory>`:

| Directiva | Qué hace |
|---|---|
| `Options Indexes` | Si no hay `index`, **lista el contenido** de la carpeta |
| `Options -Indexes` | Lo **impide** (más seguro) |
| `Options FollowSymLinks` | Permite seguir enlaces simbólicos |
| `AllowOverride None/All` | Controla si se lee `.htaccess` |
| `Require all granted` | **Permite** el acceso a todos |
| `Require all denied` | **Deniega** el acceso a todos |
| `Require ip 172.16.5.0/24` | Permite solo a una red concreta |

> ℹ️ `Require` es la sintaxis de **Apache 2.4**. Las antiguas `Order`, `Allow` y `Deny` pertenecen a versiones anteriores y no deben usarse.

---

### 6.10 📜 `/var/log/apache2/` — los registros

| Archivo | Qué guarda |
|---|---|
| `access.log` | Cada petición recibida |
| `error.log` | Errores y avisos del servidor |
| `other_vhosts_access.log` | Accesos de virtual hosts que no definan su propio log |

Una línea típica de `access.log` (formato `combined`):

```
172.16.5.1 - - [10/Oct/2026:10:15:32 +0000] "GET / HTTP/1.1" 200 3380 "-" "Mozilla/5.0 ..."
```

| Campo | Significado |
|---|---|
| `172.16.5.1` | IP del cliente |
| `[10/Oct/2026:10:15:32 +0000]` | Fecha y hora |
| `"GET / HTTP/1.1"` | Petición: método, URL y versión |
| `200` | Código de respuesta (200 = OK) |
| `3380` | Bytes enviados |
| `"Mozilla/5.0 ..."` | Navegador o programa del cliente |

```bash
sudo tail -f /var/log/apache2/access.log     # ver en vivo
sudo tail -n 20 /var/log/apache2/error.log   # últimas 20 líneas de errores
```

---

### 6.11 🧰 Comandos esenciales de gestión

| Comando | Qué hace |
|---|---|
| `sudo apache2ctl configtest` | **Comprueba la sintaxis** de la configuración |
| `sudo apache2ctl -S` | Muestra los **virtual hosts** cargados y cuál es el predeterminado |
| `sudo apache2ctl -M` | Lista los **módulos** cargados |
| `apache2 -v` | Versión |
| `sudo systemctl start apache2` | Arrancar |
| `sudo systemctl stop apache2` | Parar |
| `sudo systemctl restart apache2` | Parar y volver a arrancar |
| `sudo systemctl reload apache2` | Recargar la configuración **sin cortar conexiones** |
| `sudo systemctl enable apache2` | Arrancar automáticamente al iniciar el sistema |
| `systemctl status apache2` | Ver el estado |
| `journalctl -u apache2` | Ver los logs del servicio |

#### 🔄 `reload` vs `restart`

| | `reload` | `restart` |
|---|---|---|
| Qué hace | Recarga la configuración de forma "graciosa": los procesos terminan lo que estaban haciendo | Para todo y vuelve a arrancar |
| Conexiones activas | ✅ Se respetan | ❌ Se cortan |
| Cuándo usarlo | Cambios en sitios y configuración | Al activar/desactivar **módulos**, o si algo no se aplica con `reload` |

#### ✅ Flujo de trabajo correcto

```
 editar archivo  ──►  sudo apache2ctl configtest  ──►  sudo systemctl reload apache2
                              │
                              └─ si da error: corregir ANTES de recargar
```

> ✅ **Resultado esperado de `configtest`:** `Syntax OK`.

---

## 7. 🌍 Virtual hosts en detalle

### 7.1 ¿Qué es un virtual host?

Es la capacidad de **alojar varios sitios web en un mismo servidor** (una sola instalación y, normalmente, una sola IP). En Apache, cada sitio se define con un bloque `<VirtualHost>`.

### 7.2 Tipos de virtual host

| Tipo | Cómo se distinguen los sitios | Ejemplo |
|---|---|---|
| 🔌 **Por puerto** | Cada sitio escucha en un puerto distinto | `dominioa` en el 80 y `dominiob` en el 8080 |
| 🏷️ **Por nombre** | Mismo puerto, distinto nombre de dominio (cabecera `Host`) | `dominioa` y `dominiob` ambos en el 80 |
| 🌐 **Por IP** | Cada sitio usa una IP distinta del servidor | `172.16.5.20` y `172.16.5.21` |

### 7.3 Cómo decide Apache qué sitio atiende una petición

```
Petición entrante
       │
       ▼
1️⃣ ¿Qué IP:puerto? ──► selecciona los <VirtualHost> que coinciden
       │
       ▼
2️⃣ ¿Qué nombre trae (cabecera Host)? ──► compara con ServerName y ServerAlias
       │
       ▼
3️⃣ ¿Ninguno coincide? ──► atiende el PRIMER <VirtualHost> cargado
                          para esa IP:puerto (orden alfabético de sites-enabled)
```

> 🎯 **Diferencia con Nginx:** Apache **no tiene** `default_server`. El sitio por defecto es, simplemente, **el primero que se carga**. Por eso importa el orden alfabético de los archivos (`000-default.conf`).

### 7.4 🧪 Ejemplo práctico: virtual hosts por puerto

**Escenario:** servidor con IP `172.16.5.20`.

| Sitio | Puerto | Carpeta |
|---|---|---|
| `dominioa` | 80 | `/var/www/dominioa` |
| `dominiob` | 8080 | `/var/www/dominiob` |

> ℹ️ Usamos los mismos nombres que en Nginx para poder compararlos. Adapta nombres e IP a los de tu práctica. Si Nginx está instalado en la misma máquina, páralo antes (`sudo systemctl stop nginx`).

#### Paso 1 — Crear las carpetas y las páginas

```bash
sudo mkdir -p /var/www/dominioa /var/www/dominiob

echo "<h1>Bienvenido a dominioa (Apache, puerto 80)</h1>"   | sudo tee /var/www/dominioa/index.html
echo "<h1>Bienvenido a dominiob (Apache, puerto 8080)</h1>" | sudo tee /var/www/dominiob/index.html
```

#### Paso 2 — Ajustar propietario y permisos

```bash
sudo chown -R www-data:www-data /var/www/dominioa /var/www/dominiob
sudo chmod -R 755 /var/www/dominioa /var/www/dominiob
```

#### Paso 3 — Abrir el puerto 8080 en `ports.conf`

```bash
sudo nano /etc/apache2/ports.conf
```

Añade debajo de `Listen 80`:

```apache
Listen 80
Listen 8080
```

> ⚠️ **Este paso es el que más se olvida.** En Apache, el `<VirtualHost *:8080>` por sí solo **no abre** el puerto. Hay que declararlo con `Listen`.

#### Paso 4 — Crear los archivos de los sitios

```bash
sudo nano /etc/apache2/sites-available/dominioa.conf
```

```apache
<VirtualHost *:80>
    ServerName dominioa
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/dominioa

    <Directory /var/www/dominioa>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/dominioa.error.log
    CustomLog ${APACHE_LOG_DIR}/dominioa.access.log combined
</VirtualHost>
```

```bash
sudo nano /etc/apache2/sites-available/dominiob.conf
```

```apache
<VirtualHost *:8080>
    ServerName dominiob
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/dominiob

    <Directory /var/www/dominiob>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/dominiob.error.log
    CustomLog ${APACHE_LOG_DIR}/dominiob.access.log combined
</VirtualHost>
```

#### 🧩 Qué cambia respecto al sitio por defecto

| Línea | Explicación |
|---|---|
| `<VirtualHost *:8080>` | Es lo que **diferencia** al segundo sitio: otro puerto |
| `ServerName dominiob` | Nombre con el que se identifica el sitio |
| `DocumentRoot` | Cada sitio tiene su propia carpeta |
| `<Directory ...>` | Permite explícitamente el acceso a esa carpeta (`Require all granted`) y desactiva el listado (`-Indexes`) |
| `ErrorLog` / `CustomLog` | Logs separados por sitio, mucho más fácil de diagnosticar |

#### Paso 5 — Activar los sitios

```bash
sudo a2ensite dominioa.conf
sudo a2ensite dominiob.conf
```

(Opcional) Desactivar el sitio por defecto para que no interfiera:

```bash
sudo a2dissite 000-default.conf
```

> 💡 Esto solo elimina el **enlace**. El archivo original en `sites-available` sigue intacto.

#### Paso 6 — Validar y aplicar

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

> ⚠️ Si tras el `reload` el puerto 8080 no aparece escuchando, haz un `sudo systemctl restart apache2`. Los cambios en `Listen` se aplican con más seguridad con un reinicio.

#### Paso 7 — Resolución de nombres (para poder usar `dominioa` / `dominiob`)

Un cliente necesita saber que esos nombres apuntan a `172.16.5.20`. Dos opciones:

**Opción A — Archivo `/etc/hosts` del cliente**

```bash
sudo nano /etc/hosts
```
```
172.16.5.20   dominioa dominiob
```

**Opción B — Un servidor DNS** (por ejemplo, BIND9) con registros A para ambos nombres.

> 🔗 **Documentación relacionada:** la instalación y configuración de BIND9 se explica en la documentación de la asignatura *Servicios en Red e Internet*: [📘 Documentación de BIND9 (SRI)](./bind9-documentacion.md)

#### Paso 8 — Probar

```bash
curl http://dominioa
curl http://dominiob:8080
sudo ss -tlnp | grep apache2        # debe aparecer escuchando en el 80 y en el 8080
sudo apache2ctl -S                  # muestra los virtual hosts cargados
```

`apache2ctl -S` es especialmente útil: te enseña qué `VirtualHost` hay en cada puerto, cuál es el predeterminado y en qué archivo y línea está definido cada uno.

Si aún no tienes resolución de nombres, puedes probar indicando el nombre a mano:

```bash
curl -H "Host: dominioa" http://172.16.5.20
```

---

### 7.5 🏷️ Variante: virtual hosts por nombre (mismo puerto)

Si ambos sitios escuchan en el 80, lo único que los distingue es `ServerName`:

```apache
# dominioa.conf
<VirtualHost *:80>
    ServerName dominioa
    ServerAlias www.dominioa
    DocumentRoot /var/www/dominioa
</VirtualHost>

# dominiob.conf
<VirtualHost *:80>
    ServerName dominiob
    ServerAlias www.dominiob
    DocumentRoot /var/www/dominiob
</VirtualHost>
```

| Directiva | Qué hace |
|---|---|
| `ServerName` | Nombre principal del sitio |
| `ServerAlias` | Nombres alternativos que responden al mismo sitio |

Aquí Apache lee la cabecera `Host` de la petición para decidir. Si entras solo por IP (sin nombre), te atenderá **el primer virtual host cargado**.

> 🎯 **Diferencia clave:** en los virtual hosts **por puerto**, el que decide es el puerto (`Listen` + `<VirtualHost *:puerto>`). En los **por nombre**, el que decide es `ServerName`.

---

## 8. ➕ Extras que conviene saber

### 8.1 👤 Usuario `www-data` y permisos

Los procesos hijo corren como `www-data`. Si ese usuario no puede **leer** la carpeta o el archivo, el cliente verá un error **403 Forbidden**. Los directorios necesitan permiso de ejecución (`755`) para poder entrar en ellos.

### 8.2 🩺 Errores típicos y cómo diagnosticarlos

| Síntoma | Causa probable | Dónde mirar / solución |
|---|---|---|
| **403 Forbidden** | Permisos incorrectos, falta `Require all granted`, carpeta fuera de `/var/www/` sin bloque `<Directory>`, o no hay `index` y está `-Indexes` | `ls -l`, bloque `<Directory>`, `error.log` |
| **404 Not Found** | `DocumentRoot` mal escrito o el archivo no existe | Revisar la ruta |
| **500 Internal Server Error** | Error de sintaxis en un `.htaccess` o aplicación rota | `error.log` |
| `Invalid command 'RewriteEngine'` | Falta activar el módulo | `sudo a2enmod rewrite` y reiniciar |
| `Address already in use` | Otro programa (¿Nginx?) ocupa ese puerto | `sudo ss -tlnp` para ver quién lo usa |
| El sitio con puerto 8080 no responde | Falta `Listen 8080` en `ports.conf` | Revisar `ports.conf` |
| Sigue saliendo la página por defecto | El sitio no está activado, el archivo no termina en `.conf` o `ServerName` no coincide | `ls -l sites-enabled`, `apache2ctl -S` |
| Aviso `AH00558 ... fully qualified domain name` | Falta `ServerName` global | Ver apartado 4 |
| Los cambios no se ven | No recargaste, o caché del navegador | `reload` y probar con `curl` |

> 🔎 **Regla de oro:** ante cualquier fallo, mira primero `error.log` y ejecuta `apache2ctl configtest`.

### 8.3 🔐 Seguridad mínima

En `/etc/apache2/conf-available/security.conf`:

```apache
ServerTokens Prod
ServerSignature Off
```

- `ServerTokens Prod`: las respuestas solo dicen "Apache", sin versión ni sistema operativo.
- `ServerSignature Off`: no muestra la firma del servidor en las páginas de error.

Además:
- Usar `Options -Indexes` para que no se listen carpetas.
- Dejar `AllowOverride None` donde no se necesite `.htaccess`.
- Abrir en el firewall **solo** los puertos necesarios.
- Servir HTTPS en cualquier sitio real (módulo `ssl`).

### 8.4 🔀 Apache como proxy inverso (para saber que existe)

Con los módulos `proxy` y `proxy_http` activados:

```apache
<VirtualHost *:80>
    ServerName app.local
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/
</VirtualHost>
```

El cliente solo habla con Apache; la aplicación queda detrás.

---

## 9. 📊 Tabla comparativa Nginx vs Apache

| Característica | 🟢 Nginx | 🔴 Apache |
|---|---|---|
| **Paquete / servicio (Ubuntu)** | `nginx` | `apache2` |
| **Modelo de funcionamiento** | Asíncrono, orientado a eventos (master + workers) | Procesos o hilos según el MPM (`prefork`, `worker`, `event`) |
| **Configuración principal** | `/etc/nginx/nginx.conf` | `/etc/apache2/apache2.conf` |
| **Definición de puertos** | Directiva `listen` en cada `server` | `ports.conf` + directiva `Listen` |
| **Sitios disponibles / activos** | `sites-available/` y `sites-enabled/` | `sites-available/` y `sites-enabled/` |
| **Activar un sitio** | `ln -s` manual | `sudo a2ensite sitio` |
| **Desactivar un sitio** | `rm` del enlace | `sudo a2dissite sitio` |
| **Bloque de un sitio** | `server { }` | `<VirtualHost *:80> ... </VirtualHost>` |
| **Carpeta web por defecto** | `/var/www/html` | `/var/www/html` |
| **Usuario de ejecución** | `www-data` | `www-data` |
| **Módulos** | Integrados o dinámicos, se cargan al arrancar | `a2enmod` / `a2dismod` |
| **Configuración por directorio** | ❌ No (no existe `.htaccess`) | ✅ Sí, con `.htaccess` |
| **Contenido dinámico (PHP)** | Mediante PHP-FPM (FastCGI) | `mod_php` o PHP-FPM |
| **Contenido estático** | Muy eficiente | Bueno |
| **Proxy inverso / balanceo** | Uno de sus puntos fuertes | Posible con módulos (`mod_proxy`) |
| **Comprobar la sintaxis** | `sudo nginx -t` | `sudo apache2ctl configtest` |
| **Recargar sin cortar conexiones** | `sudo systemctl reload nginx` | `sudo systemctl reload apache2` |
| **Logs** | `/var/log/nginx/` | `/var/log/apache2/` |

---

<sub>📘 Documentación de Apache · Implantación de Aplicaciones Web · 2º ASIR</sub>
