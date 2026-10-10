# 🟢 NGINX — Documentación completa en Ubuntu

> **Asignatura:** Implantación de Aplicaciones Web · 2º ASIR
> **Entorno:** Ubuntu Server · todo desde la terminal
> **Temas:** servidor web · estructura de archivos · configuración · virtual hosts

---

## 📑 Índice

1. ¿Qué es Nginx y para qué sirve?
2. Cómo funciona por dentro (arquitectura)
3. Nginx vs Apache: el contraste
4. Instalación rápida
5. Mapa de archivos y qué vamos a crear
6. Los archivos de configuración, uno por uno
7. Virtual hosts en detalle
8. Extras que conviene saber
9. Tabla comparativa Nginx vs Apache

---

## 1. 🧭 ¿Qué es Nginx y para qué sirve?

**Nginx** (se pronuncia *"engine-x"*) es un software de servidor que recibe peticiones HTTP/HTTPS de los clientes (navegadores, `curl`, aplicaciones) y les devuelve una respuesta: una página, una imagen, un archivo, o el resultado de otra aplicación.

Lo creó Igor Sysoev para resolver un problema concreto: atender **miles de conexiones simultáneas** gastando muy pocos recursos.

### 🛠️ Para qué se usa

| Uso | Qué significa |
|---|---|
| 🌐 **Servidor web** | Sirve páginas y archivos estáticos (HTML, CSS, JS, imágenes) |
| 🔀 **Proxy inverso** | Se coloca delante de otra aplicación y le reenvía las peticiones |
| ⚖️ **Balanceador de carga** | Reparte el tráfico entre varios servidores backend |
| 🗄️ **Caché HTTP** | Guarda respuestas para no recalcularlas cada vez |
| 🔒 **Terminación TLS/HTTPS** | Gestiona el cifrado y pasa tráfico simple al backend |

> 💡 **Idea clave:** Nginx es muy bueno sirviendo contenido **estático** y como **intermediario**. Para contenido dinámico (PHP, Python...) no lo ejecuta él mismo: se lo pasa a otro programa (por ejemplo, PHP-FPM).

---

## 2. ⚙️ Cómo funciona por dentro (arquitectura)

Nginx usa un modelo **asíncrono y orientado a eventos**: un solo *worker* puede atender muchísimas conexiones a la vez, sin quedarse bloqueado esperando a ninguna.

```
                 ┌─────────────────────────────┐
                 │       PROCESO MASTER        │
                 │  lee la configuración,      │
                 │  crea y controla workers    │
                 │  (corre como root)          │
                 └──────────────┬──────────────┘
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
   │  WORKER 1   │       │  WORKER 2   │       │  WORKER N   │
   │ atiende     │       │ atiende     │       │ atiende     │
   │ conexiones  │       │ conexiones  │       │ conexiones  │
   │ (www-data)  │       │ (www-data)  │       │ (www-data)  │
   └─────────────┘       └─────────────┘       └─────────────┘
```

- **Master:** lee la configuración, abre los puertos y gestiona a los workers. No atiende peticiones.
- **Workers:** hacen el trabajo real. Por defecto hay uno por núcleo de CPU (`worker_processes auto;`).

Lo puedes comprobar tú mismo:

```bash
ps aux | grep nginx
```

Debes ver un `master process` (usuario root) y varios `worker process` (usuario www-data).

---

## 3. 🥊 Nginx vs Apache: el contraste

Piensa en un restaurante:

- **Apache (modelo clásico):** un camarero asignado a cada mesa. Si hay mucha gente, necesitas muchos camareros y se consume más.
- **Nginx:** pocos camareros que atienden muchas mesas a la vez, sin quedarse parados esperando a que un cliente decida.

| Aspecto | 🟢 Nginx | 🔴 Apache |
|---|---|---|
| **Filosofía** | Eventos asíncronos | Procesos/hilos (según el MPM) |
| **Contenido estático** | Muy rápido, poco consumo | Bueno, pero consume más con mucha carga |
| **Contenido dinámico** | Lo delega (p. ej. PHP-FPM) | Puede ejecutarlo con módulos (`mod_php`) o con PHP-FPM |
| **Configuración por directorio** | ❌ No existe `.htaccess` | ✅ `.htaccess` permite configurar sin tocar el archivo principal |
| **Módulos** | Se cargan al arrancar (muchos van integrados) | Se activan y desactivan fácilmente (`a2enmod`) |
| **Uso típico hoy** | Proxy inverso, balanceo, webs con mucho tráfico | Alojamiento compartido, aplicaciones que dependen de `.htaccess` |

> ⚠️ **Matiz importante (para no decir un tópico):** el Apache moderno con el MPM *event* también trabaja por eventos, así que la diferencia de rendimiento ya no es tan abismal como se cuenta. Lo que sí sigue siendo distinto es el **modelo de configuración** (sin `.htaccess`) y el **papel como proxy inverso**.

---

## 4. 📦 Instalación rápida

```bash
sudo apt update
sudo apt install nginx -y
```

- `apt update` actualiza la lista de paquetes disponibles.
- `apt install nginx` descarga e instala el servidor, crea el usuario `www-data` y lo deja arrancado.

### ✅ Comprobar que funciona

```bash
nginx -v                         # versión instalada
systemctl status nginx           # estado del servicio
curl -I http://localhost         # cabeceras de respuesta (debe dar 200 OK)
sudo ss -tlnp | grep nginx       # puertos en los que escucha
```

> ✅ **Resultado esperado:** `active (running)` en el estado y `HTTP/1.1 200 OK` en el `curl`.

### 🔥 Firewall (solo si usas UFW)

```bash
sudo ufw status                  # ¿está activo?
sudo ufw allow 'Nginx HTTP'      # abre el puerto 80
sudo ufw allow 8080/tcp          # abre el 8080 (lo necesitaremos en la práctica)
```

> 💡 Si UFW está inactivo, no hace falta tocar nada.

---

## 5. 🗺️ Mapa de archivos y qué vamos a crear

### Estructura que deja la instalación

```
/etc/nginx/
├── nginx.conf              ← configuración PRINCIPAL
├── mime.types              ← tipos de archivo (.html, .css, .png...)
├── conf.d/                 ← configuraciones extra (*.conf)
├── snippets/               ← trozos de configuración reutilizables
├── modules-enabled/        ← módulos activados
├── sites-available/        ← TODOS los sitios definidos
│   └── default
└── sites-enabled/          ← solo los sitios ACTIVOS (enlaces simbólicos)
    └── default -> ../sites-available/default

/var/www/html/              ← carpeta web por defecto
/var/log/nginx/             ← registros (access.log y error.log)
```

### 📝 Lo que vamos a crear nosotros (adelanto)

| Qué | Dónde | Para qué |
|---|---|---|
| Carpeta web del sitio | `/var/www/dominioa/` | Guardar el contenido de la web |
| Página de inicio | `/var/www/dominioa/index.html` | Lo que se verá al entrar |
| Archivo de configuración | `/etc/nginx/sites-available/dominioa` | Definir el sitio (el *server block*) |
| Enlace simbólico | `/etc/nginx/sites-enabled/dominioa` | Activar el sitio |

> Todo esto se explica en detalle en los apartados 6 y 7.

---

## 6. 🔍 Los archivos de configuración, uno por uno

> 💡 **Truco para estudiar:** para ver un archivo sin comentarios ni líneas vacías:
> ```bash
> grep -v '^\s*#' /etc/nginx/nginx.conf | grep -v '^\s*$'
> ```

### 6.1 📄 `/etc/nginx/nginx.conf` — configuración principal

Es el primer archivo que lee Nginx. Define el comportamiento **global** y se encarga de **incluir** los demás archivos.

> ℹ️ Ejemplo orientativo: el contenido exacto puede variar según la versión. Compáralo con el tuyo usando `cat`.

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;

    gzip on;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

#### 🧩 Qué hace cada línea

| Directiva | Qué hace |
|---|---|
| `user www-data;` | Usuario con el que se ejecutan los **workers** (sin privilegios de root, por seguridad) |
| `worker_processes auto;` | Número de workers: `auto` = uno por núcleo de CPU |
| `pid /run/nginx.pid;` | Archivo donde se guarda el PID del proceso master |
| `error_log ...;` | Archivo global de errores |
| `include modules-enabled/*.conf;` | Carga los módulos dinámicos activados |
| `events { worker_connections 768; }` | Conexiones simultáneas que puede manejar **cada worker** |
| `http { ... }` | Bloque con **todo lo relacionado con HTTP** |
| `sendfile on;` | Envía archivos directamente desde el kernel: más eficiente |
| `tcp_nopush on;` | Optimiza el envío de paquetes junto con `sendfile` |
| `types_hash_max_size` | Tamaño de la tabla interna de tipos MIME |
| `include mime.types;` | Carga la correspondencia extensión → tipo de contenido |
| `default_type ...;` | Tipo que se usa si no se reconoce la extensión |
| `access_log ...;` | Registro de todas las peticiones |
| `gzip on;` | Comprime las respuestas para ahorrar ancho de banda |
| `include conf.d/*.conf;` | Carga configuraciones extra |
| `include sites-enabled/*;` | **Carga los sitios activos** (aquí se conectan los virtual hosts) |

> 🧮 **Capacidad aproximada:** `worker_processes × worker_connections` = conexiones simultáneas máximas.

#### 🌳 Jerarquía de contextos

Nginx organiza la configuración en **bloques anidados** llamados *contextos*. Lo que defines en un nivel superior lo heredan los inferiores.

```
main (global)
├── events { }
└── http { }
    ├── server { }               ← un virtual host
    │   ├── location / { }
    │   └── location /img/ { }
    └── server { }               ← otro virtual host
```

> ⚠️ **Sintaxis:** cada directiva termina en `;` y los bloques usan `{ }`. Olvidar un `;` es el error más común. Por eso siempre se valida con `nginx -t`.

---

### 6.2 📄 `/etc/nginx/sites-available/default` — el sitio por defecto

Contiene un bloque `server` (un **virtual host**). Sin comentarios, es básicamente esto:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### 🧩 Directiva por directiva

| Directiva | Qué hace |
|---|---|
| `server { }` | Define un sitio web (un virtual host) |
| `listen 80 default_server;` | Puerto de escucha (IPv4). `default_server` = atiende lo que no coincida con ningún otro sitio |
| `listen [::]:80 ...;` | Lo mismo para IPv6 |
| `root /var/www/html;` | Carpeta raíz de donde se sirven los archivos |
| `index ...;` | Lista de archivos que se prueban, en orden, al pedir un directorio |
| `server_name _;` | Nombre(s) del sitio. `_` es un nombre "comodín" que no coincide con nada real |
| `location / { }` | Reglas para las URLs que empiezan por `/` (todas) |
| `try_files $uri $uri/ =404;` | Busca el archivo exacto, luego un directorio, y si no existe devuelve **404** |

#### 🎯 Cómo elige Nginx el `location`

| Tipo | Ejemplo | Significado |
|---|---|---|
| Exacto | `location = /login` | Solo coincide con esa URL exacta (máxima prioridad) |
| Prefijo preferente | `location ^~ /img/` | Prefijo que gana frente a las expresiones regulares |
| Expresión regular | `location ~ \.php$` | Coincide por patrón (distingue mayúsculas); `~*` no las distingue |
| Prefijo normal | `location /docs/` | Coincide por el comienzo de la URL (gana el más largo) |

---

### 6.3 🔗 `/etc/nginx/sites-enabled/` — los sitios activos

No contiene archivos reales: contiene **enlaces simbólicos** (accesos directos) a los archivos de `sites-available`.

```bash
ls -l /etc/nginx/sites-enabled
# default -> /etc/nginx/sites-available/default
```

**¿Por qué existen dos carpetas?**

| Carpeta | Función |
|---|---|
| `sites-available/` | El "catálogo": todos los sitios que has escrito, activos o no |
| `sites-enabled/` | Solo los que **están funcionando** |

Así puedes activar o desactivar un sitio sin borrar su configuración:

```bash
# Activar
sudo ln -s /etc/nginx/sites-available/dominioa /etc/nginx/sites-enabled/

# Desactivar (borra solo el enlace, NO el archivo original)
sudo rm /etc/nginx/sites-enabled/dominioa
```

> ⚠️ **Detalle de examen:** el esquema `sites-available` / `sites-enabled` es una **convención de Debian/Ubuntu**, no de Nginx en general. En otras distribuciones se usa `conf.d/`.

---

### 6.4 📁 `conf.d/`, `snippets/` y otros

| Ruta | Para qué sirve |
|---|---|
| `/etc/nginx/conf.d/` | Archivos `*.conf` que se incluyen dentro de `http { }` |
| `/etc/nginx/snippets/` | Fragmentos reutilizables (por ejemplo, la configuración de PHP o de SSL) que se incluyen con `include` |
| `/etc/nginx/mime.types` | Tabla que asocia extensiones con tipos de contenido (`html` → `text/html`) |
| `/etc/nginx/modules-enabled/` | Módulos dinámicos activados |

```bash
ls -l /etc/nginx/conf.d /etc/nginx/snippets
```

---

### 6.5 📂 `/var/www/html/` — la carpeta web por defecto

Aquí está la página de bienvenida (`index.nginx-debian.html`). Es el `root` del sitio por defecto. Para nuestros propios sitios crearemos carpetas aparte dentro de `/var/www/`.

---

### 6.6 📜 `/var/log/nginx/` — los registros

| Archivo | Qué guarda |
|---|---|
| `access.log` | Cada petición recibida |
| `error.log` | Errores y avisos del servidor |

Una línea típica de `access.log`:

```
172.16.5.1 - - [10/Oct/2026:10:15:32 +0000] "GET / HTTP/1.1" 200 615 "-" "Mozilla/5.0 ..."
```

| Campo | Significado |
|---|---|
| `172.16.5.1` | IP del cliente |
| `[10/Oct/2026:10:15:32 +0000]` | Fecha y hora |
| `"GET / HTTP/1.1"` | Petición: método, URL y versión |
| `200` | Código de respuesta (200 = OK) |
| `615` | Bytes enviados |
| `"Mozilla/5.0 ..."` | Navegador o programa del cliente |

```bash
sudo tail -f /var/log/nginx/access.log     # ver en vivo
sudo tail -n 20 /var/log/nginx/error.log   # últimas 20 líneas de errores
```

---

### 6.7 🧰 Comandos esenciales de gestión

| Comando | Qué hace |
|---|---|
| `sudo nginx -t` | **Comprueba la sintaxis** de la configuración |
| `sudo nginx -T` | Comprueba y **muestra toda la configuración** ya ensamblada |
| `nginx -v` | Versión |
| `sudo systemctl start nginx` | Arrancar |
| `sudo systemctl stop nginx` | Parar |
| `sudo systemctl restart nginx` | Parar y volver a arrancar |
| `sudo systemctl reload nginx` | Recargar la configuración **sin cortar conexiones** |
| `sudo systemctl enable nginx` | Arrancar automáticamente al iniciar el sistema |
| `systemctl status nginx` | Ver el estado |
| `journalctl -u nginx` | Ver los logs del servicio |

#### 🔄 `reload` vs `restart`

| | `reload` | `restart` |
|---|---|---|
| Qué hace | El master lee la nueva configuración y crea workers nuevos; los antiguos terminan lo que estaban haciendo | Para todo y vuelve a arrancar |
| Conexiones activas | ✅ Se respetan | ❌ Se cortan |
| Cuándo usarlo | Cambios de configuración | Cambios que `reload` no aplica, o si el servicio está bloqueado |

#### ✅ Flujo de trabajo correcto

```
 editar archivo  ──►  sudo nginx -t  ──►  sudo systemctl reload nginx
                          │
                          └─ si da error: corregir ANTES de recargar
```

> ✅ **Resultado esperado de `nginx -t`:** `syntax is ok` y `test is successful`.

---

## 7. 🌍 Virtual hosts en detalle

### 7.1 ¿Qué es un virtual host?

Es la capacidad de **alojar varios sitios web en un mismo servidor** (una sola IP y una sola instalación de Nginx). En Nginx, cada sitio se define con un bloque `server { }`.

### 7.2 Tipos de virtual host

| Tipo | Cómo se distinguen los sitios | Ejemplo |
|---|---|---|
| 🔌 **Por puerto** | Cada sitio escucha en un puerto distinto | `dominioa` en el 80 y `dominiob` en el 8080 |
| 🏷️ **Por nombre** | Mismo puerto, distinto nombre de dominio (cabecera `Host`) | `dominioa` y `dominiob` ambos en el 80 |
| 🌐 **Por IP** | Cada sitio usa una IP distinta del servidor | `172.16.5.20` y `172.16.5.21` |

### 7.3 Cómo decide Nginx qué sitio atiende una petición

```
Petición entrante
       │
       ▼
1️⃣ ¿Qué IP:puerto? ──► elige los server{} con ese listen
       │
       ▼
2️⃣ ¿Qué nombre trae (cabecera Host)? ──► compara con server_name
       │
       ▼
3️⃣ ¿Ninguno coincide? ──► usa el default_server de ese puerto
                          (o el primero definido si no hay ninguno marcado)
```

### 7.4 🧪 Ejemplo práctico: virtual hosts por puerto

**Escenario:** servidor con IP `172.16.5.20`.

| Sitio | Puerto | Carpeta |
|---|---|---|
| `dominioa` | 80 | `/var/www/dominioa` |
| `dominiob` | 8080 | `/var/www/dominiob` |

> ℹ️ Adapta los nombres y la IP a los de tu práctica.

#### Paso 1 — Crear las carpetas y las páginas

```bash
sudo mkdir -p /var/www/dominioa /var/www/dominiob

echo "<h1>Bienvenido a dominioa (puerto 80)</h1>"  | sudo tee /var/www/dominioa/index.html
echo "<h1>Bienvenido a dominiob (puerto 8080)</h1>" | sudo tee /var/www/dominiob/index.html
```

#### Paso 2 — Ajustar propietario y permisos

```bash
sudo chown -R www-data:www-data /var/www/dominioa /var/www/dominiob
sudo chmod -R 755 /var/www/dominioa /var/www/dominiob
```

#### Paso 3 — Crear los archivos de configuración

```bash
sudo nano /etc/nginx/sites-available/dominioa
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name dominioa;

    root /var/www/dominioa;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/dominioa.access.log;
    error_log  /var/log/nginx/dominioa.error.log;
}
```

```bash
sudo nano /etc/nginx/sites-available/dominiob
```

```nginx
server {
    listen 8080;
    listen [::]:8080;

    server_name dominiob;

    root /var/www/dominiob;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/dominiob.access.log;
    error_log  /var/log/nginx/dominiob.error.log;
}
```

#### 🧩 Qué cambia respecto al sitio por defecto

| Línea | Explicación |
|---|---|
| `listen 8080;` | Es lo que **diferencia** al segundo sitio: otro puerto |
| `server_name dominiob;` | Nombre con el que se identifica el sitio |
| `root /var/www/dominiob;` | Cada sitio tiene su propia carpeta |
| `access_log` / `error_log` | Logs separados por sitio, mucho más fácil de diagnosticar |

#### Paso 4 — Activar los sitios (enlaces simbólicos)

```bash
sudo ln -s /etc/nginx/sites-available/dominioa /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/dominiob /etc/nginx/sites-enabled/
```

(Opcional) Desactivar el sitio por defecto para que no interfiera:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

> 💡 Esto solo elimina el **enlace**. El archivo original en `sites-available` sigue intacto.

#### Paso 5 — Validar y recargar

```bash
sudo nginx -t
sudo systemctl reload nginx
```

#### Paso 6 — Resolución de nombres (para poder usar `dominioa` / `dominiob`)

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

#### Paso 7 — Probar

```bash
curl http://dominioa
curl http://dominiob:8080
sudo ss -tlnp | grep nginx         # debe aparecer escuchando en el 80 y en el 8080
```

Si aún no tienes resolución de nombres, puedes probar igualmente indicando el nombre a mano:

```bash
curl -H "Host: dominioa" http://172.16.5.20
```

---

### 7.5 🏷️ Variante: virtual hosts por nombre (mismo puerto)

Si ambos sitios escuchan en el 80, lo único que los distingue es `server_name`:

```nginx
# dominioa
server {
    listen 80;
    server_name dominioa;
    root /var/www/dominioa;
}

# dominiob
server {
    listen 80;
    server_name dominiob;
    root /var/www/dominiob;
}
```

Aquí Nginx lee la cabecera `Host` de la petición para decidir. Por eso, sin resolución de nombres, entrar solo por IP te llevaría al `default_server`.

> 🎯 **Diferencia clave:** en los virtual hosts **por puerto**, el que decide es `listen`. En los **por nombre**, el que decide es `server_name`.

---

## 8. ➕ Extras que conviene saber

### 8.1 👤 Usuario `www-data` y permisos

Los workers corren como `www-data`. Si ese usuario no puede **leer** la carpeta o el archivo, el cliente verá un error **403 Forbidden**. Los directorios necesitan permiso de ejecución (`755`) para poder entrar en ellos.

### 8.2 🩺 Errores típicos y cómo diagnosticarlos

| Síntoma | Causa probable | Dónde mirar / solución |
|---|---|---|
| **403 Forbidden** | Permisos incorrectos o falta el `index` | `ls -l` de la carpeta, `chmod`/`chown`, directiva `index` |
| **404 Not Found** | `root` mal escrito o el archivo no existe | Revisar la ruta en `root` |
| **502 Bad Gateway** | El backend (PHP-FPM, app) está caído | `systemctl status` del backend |
| `Address already in use` | Otro programa ocupa ese puerto | `sudo ss -tlnp` para ver quién lo usa |
| `unknown directive` / `unexpected "}"` | Error de sintaxis, normalmente falta un `;` | `sudo nginx -t` indica archivo y línea |
| Sigue saliendo la página por defecto | El sitio no está enlazado en `sites-enabled` o `server_name` no coincide | `ls -l sites-enabled`, revisar el nombre |
| Los cambios no se ven | No recargaste, o caché del navegador | `reload` y probar con `curl` |

> 🔎 **Regla de oro:** ante cualquier fallo, mira primero `error.log` y ejecuta `nginx -t`.

### 8.3 🔐 Seguridad mínima

- Ocultar la versión de Nginx en las respuestas: añadir `server_tokens off;` dentro de `http { }`.
- Abrir en el firewall **solo** los puertos necesarios.
- No ejecutar los workers como root (ya viene configurado con `www-data`).
- Servir HTTPS en cualquier sitio real (certificados TLS).

### 8.4 🔀 Nginx como proxy inverso (para saber que existe)

Su uso más extendido hoy. Nginx recibe la petición y la reenvía a otra aplicación:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
}
```

El cliente solo habla con Nginx; la aplicación queda detrás, protegida.

---

## 9. 📊 Tabla comparativa Nginx vs Apache

> 📋 **Esta tabla se copiará tal cual al documento de Apache.**

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

<sub>📘 Documentación de Nginx · Implantación de Aplicaciones Web · 2º ASIR</sub>
