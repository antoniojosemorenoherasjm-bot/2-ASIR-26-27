<div align="center">

# 🌐 Nginx con 2 Virtual Hosts

### Dos webs distintas en una misma máquina Ubuntu 26.04

![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_26.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Puertos](https://img.shields.io/badge/Puertos-80_%7C_8080-blue?style=for-the-badge)

</div>

---

## 🎯 Objetivo

Configurar **Nginx** para servir **dos páginas web diferentes** desde la **misma máquina**, usando **dos puertos distintos**.

| 🔗 URL | 🔌 Puerto | 📄 Web mostrada |
|:---|:---:|:---|
| `http://172.16.5.20` | **80** | Dominio A |
| `http://172.16.5.20:8080` | **8080** | Dominio B |

```text
                    ┌───────────────────────────────┐
                    │      Ubuntu 26.04 · Nginx     │
                    │         172.16.5.20           │
                    │                               │
 Cliente ──:80───►  │  dominioa ─► /var/www/dominioa│
         ──:8080──► │  dominiob ─► /var/www/dominiob│
                    └───────────────────────────────┘
```

---

## 📋 Índice

1. [Instalar Nginx](#1--instalar-nginx)
2. [Crear las webs](#2--crear-las-webs)
3. [Crear los virtual hosts](#3--crear-los-virtual-hosts)
4. [Activar los virtual hosts](#4--activar-los-virtual-hosts)
5. [Comprobar y aplicar la configuración](#5--comprobar-y-aplicar-la-configuración)
6. [Configurar el firewall](#6--configurar-el-firewall)
7. [Probar las páginas](#7--probar-las-páginas)
8. [Solución de problemas](#-si-algo-falla)
9. [Resumen de la estructura](#-resumen-de-la-estructura)

---

## 1 · 📦 Instalar Nginx

Actualizamos la lista de paquetes e instalamos Nginx:

```bash
sudo apt update && sudo apt install -y nginx
```

---

## 2 · 🗂️ Crear las webs

Creamos **dos carpetas** dentro de `/var/www/`. Cada una guardará una de las páginas.

```bash
sudo mkdir -p /var/www/dominioa /var/www/dominiob
```

### 🅰️ Web del puerto 80

Creamos el `index.html` del **dominio A**:

```bash
sudo tee /var/www/dominioa/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Dominio A - Puerto 80</title>
</head>
<body>
    <h1>Bienvenid@ a Dominio A: Puerto 80</h1>
</body>
</html>
EOF
```

> 👉 Se mostrará al entrar en **`http://172.16.5.20`**

### 🅱️ Web del puerto 8080

Creamos el `index.html` del **dominio B**:

```bash
sudo tee /var/www/dominiob/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Dominio B - Puerto 8080</title>
</head>
<body>
    <h1>Bienvenid@ a Dominio B: Puerto 8080</h1>
</body>
</html>
EOF
```

> 👉 Se mostrará al entrar en **`http://172.16.5.20:8080`**

<details>
<summary>💡 <b>Alternativa rápida:</b> copiar el dominio A y modificarlo</summary>

```bash
sudo cp -r /var/www/dominioa/. /var/www/dominiob/
sudo sed -i -e 's/Dominio A/Dominio B/g' -e 's/Puerto 80/Puerto 8080/g' /var/www/dominiob/index.html
```

> [!NOTE]
> Se cambian **dos** textos: el nombre del dominio (`Dominio A` → `Dominio B`) y el puerto (`Puerto 80` → `Puerto 8080`). Si solo cambiamos el puerto, la web B diría "Dominio A: Puerto 8080".

</details>

---

## 3 · ⚙️ Crear los virtual hosts

Ahora le indicamos a Nginx **qué web mostrar según el puerto** utilizado.

> [!TIP]
> En `server_name` va la IP o el dominio que corresponda. En esta práctica usamos la IP **172.16.5.20**.

### 🅰️ Virtual host del puerto 80

```bash
sudo tee /etc/nginx/sites-available/dominioa > /dev/null <<'EOF'
server {
    listen 80;
    server_name 172.16.5.211;

    root /var/www/dominioa;
    index index.html;
}
EOF
```

| Directiva | Qué hace |
|:---|:---|
| `listen 80` | Nginx escucha en el **puerto 80** |
| `server_name 172.16.5.20` | IP utilizada |
| `root /var/www/dominioa` | Carpeta donde están los archivos de la web |
| `index index.html` | Archivo principal |

### 🅱️ Virtual host del puerto 8080

```bash
sudo tee /etc/nginx/sites-available/dominiob > /dev/null <<'EOF'
server {
    listen 8080;
    server_name 172.16.5.211;

    root /var/www/dominiob;
    index index.html;
}
EOF
```

Aquí Nginx escucha en el **puerto 8080** y usa los archivos de `/var/www/dominiob`.

---

## 4 · 🔗 Activar los virtual hosts

Los archivos de configuración están en `/etc/nginx/sites-available/`.
Para activarlos creamos **enlaces simbólicos** en `/etc/nginx/sites-enabled/`:

```bash
sudo ln -s /etc/nginx/sites-available/dominioa /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/dominiob /etc/nginx/sites-enabled/
```

Y quitamos el sitio que viene **por defecto** con Nginx:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

> [!IMPORTANT]
> Esto evita que aparezca la página **"Welcome to nginx"** en lugar de nuestra web.

```text
sites-available/  (configuraciones disponibles)
   ├── dominioa ──────┐
   └── dominiob ───┐  │      enlaces simbólicos
                   │  │
sites-enabled/     ▼  ▼      (configuraciones activas)
   ├── dominioa
   └── dominiob
```

---

## 5 · ✅ Comprobar y aplicar la configuración

**Primero comprobamos** que no hay errores:

```bash
sudo nginx -t
```

Si todo está bien, debe aparecer:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**Después recargamos** Nginx para aplicar los cambios:

```bash
sudo systemctl reload nginx
```

---

## 6 · 🔥 Configurar el firewall

> [!NOTE]
> Este paso **solo es necesario si UFW está activo**.

Comprobamos su estado:

```bash
sudo ufw status
```

Si está activo, permitimos los puertos 80 y 8080:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 8080/tcp
```

---

## 7 · 🧪 Probar las páginas

### Desde la terminal con `curl`

```bash
curl http://172.16.5.20        # → Dominio A · Puerto 80
curl http://172.16.5.20:8080   # → Dominio B · Puerto 8080
```

Si el sistema no reconoce `curl`:

```bash
sudo apt install curl
```

### Desde el navegador del equipo anfitrión

| Entramos en | Debe aparecer |
|:---|:---|
| `http://172.16.5.20` | Página del **Puerto 80** (Dominio A) |
| `http://172.16.5.20:8080` | Página del **Puerto 8080** (Dominio B) |

---

## 🛠️ Si algo falla

| ❌ Problema | 🔍 Posible causa |
|:---|:---|
| `nginx -t` da error | Falta un `;`, una llave `{ }` o hay texto suelto en el archivo de configuración |
| Sale *Welcome to nginx* | No se ha eliminado el sitio `default` (paso 4) |
| Funciona en la VM pero no desde fuera | Firewall o red de VirtualBox (adaptador puente o red interna) |
| Error **403** o **404** | La ruta de `root` está mal escrita o falta el `index.html` |

---

## 🧾 Resumen de la estructura

Al terminar tendremos **dos webs**:

```text
/var/www/dominioa/
└── index.html      →  Puerto 80

/var/www/dominiob/
└── index.html      →  Puerto 8080
```

Y **dos configuraciones de Nginx**, activadas mediante enlaces en `/etc/nginx/sites-enabled/`:

```text
/etc/nginx/sites-available/dominioa
/etc/nginx/sites-available/dominiob
```

### 🏁 Resultado final

Con la **misma IP** accedemos a **dos páginas diferentes** según el puerto:

```text
http://172.16.5.20        →  Dominio A  →  Puerto 80
http://172.16.5.20:8080   →  Dominio B  →  Puerto 8080
```

---

<div align="center">

**✨ Fin de la práctica ✨**

</div>
