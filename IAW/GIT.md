# 🟠 GIT — Cómo inicializarlo (guía rápida)

> **Asignatura:** Implantación de Aplicaciones Web · 2º ASIR
> **Entorno:** Ubuntu · todo desde la terminal
> **Objetivo:** instalar Git, configurarlo, crear el primer repositorio y conectarlo con GitHub

---

## 📑 Índice

1. ¿Qué es Git? (en 1 minuto)
2. Instalación
3. Configuración inicial
4. Crear un repositorio (`init` o `clone`)
5. Tu primer commit
6. Conectar con GitHub
7. El archivo `.gitignore`
8. Chuleta de comandos

---

## 1. 🧭 ¿Qué es Git? (en 1 minuto)

**Git** es un sistema de **control de versiones**: guarda el historial de cambios de un proyecto para que puedas volver atrás, ver quién cambió qué y trabajar en equipo sin pisarte.

**GitHub** es un servicio web que **aloja repositorios Git** en la nube. Git es la herramienta; GitHub es el sitio donde subes tu trabajo.

### Las 4 zonas de Git

```
 ┌─────────────┐   git add   ┌─────────────┐  git commit  ┌─────────────┐  git push  ┌─────────────┐
 │  WORKSPACE  │ ──────────► │   STAGING   │ ───────────► │    LOCAL    │ ─────────► │   REMOTO    │
 │ tus archivos│             │    AREA     │              │ REPOSITORIO │            │  (GitHub)   │
 └─────────────┘             └─────────────┘              └─────────────┘            └─────────────┘
```

| Zona | Qué es | Analogía |
|---|---|---|
| **Workspace** | Tus archivos tal como los editas | Tu escritorio |
| **Staging area** | Lo que has marcado para guardar | La caja donde preparas el envío |
| **Repositorio local** | El historial de commits en tu máquina | Fotos guardadas del proyecto |
| **Repositorio remoto** | La copia en GitHub | La nube |

### Estados de un archivo

| Estado | Significado |
|---|---|
| `untracked` | Git todavía no lo conoce |
| `modified` | Está controlado, pero lo has cambiado |
| `staged` | Marcado para entrar en el próximo commit |
| `committed` | Guardado en el historial |

---

## 2. 📦 Instalación

```bash
sudo apt update
sudo apt install git -y
```

Comprobar que está instalado:

```bash
git --version
```

---

## 3. ⚙️ Configuración inicial

Solo se hace **una vez por equipo**. Git guardará este nombre y correo en cada commit que hagas.

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tucorreo@ejemplo.com"
git config --global init.defaultBranch main
```

| Comando | Qué hace |
|---|---|
| `user.name` | Nombre que aparecerá como autor de los commits |
| `user.email` | Correo asociado a los commits |
| `init.defaultBranch main` | Hace que la rama inicial de los repositorios nuevos se llame `main` |

Comprobar la configuración:

```bash
git config --list
```

> 💡 **Consejo:** usa en `user.email` el mismo correo de tu cuenta de GitHub, para que tus commits queden vinculados a tu perfil.

---

## 4. 🗂️ Crear un repositorio

Hay dos puntos de partida. Elige según el caso:

| Caso | Comando |
|---|---|
| Empiezo un proyecto **desde cero** en mi máquina | `git init` |
| Ya existe en GitHub y quiero **descargarlo** | `git clone <url>` |

### Opción A — Repositorio nuevo en local

```bash
mkdir mi-proyecto
cd mi-proyecto
git init
```

`git init` crea una carpeta oculta llamada **`.git`** dentro del proyecto. Ahí vive todo el historial y la configuración del repositorio.

```bash
ls -a        # verás la carpeta .git
```

> ⚠️ **Nunca borres ni edites a mano la carpeta `.git`**: si la pierdes, pierdes el historial del proyecto.

### Opción B — Clonar un repositorio existente

```bash
git clone https://github.com/usuario/repositorio.git
```

Crea una carpeta con el nombre del repositorio, con todo su contenido y su historial, y ya conectada al remoto (`origin`).

---

## 5. ✅ Tu primer commit

Dentro del repositorio, el ciclo básico es siempre el mismo:

```bash
echo "# Mi proyecto" > README.md      # creamos un archivo

git status                            # 1. ¿en qué estado está todo?
git add README.md                     # 2. lo pasamos al staging
git commit -m "Primer commit"         # 3. lo guardamos en el historial
```

| Paso | Comando | Qué hace |
|---|---|---|
| 1 | `git status` | Muestra el estado de los archivos. **Úsalo constantemente** |
| 2 | `git add <archivo>` | Pasa el archivo al staging (`git add -A` añade todo) |
| 3 | `git commit -m "mensaje"` | Guarda lo que hay en staging en el repositorio local |

Ver el historial:

```bash
git log                 # historial completo
git log --oneline       # una línea por commit
```

> 💡 **Buenos mensajes de commit:** breves y que expliquen **qué** has cambiado. Mejor "Añade página de contacto" que "cambios" o "arreglo".

---

## 6. 🔗 Conectar con GitHub

### 6.1 Crear el repositorio remoto

1. Entra en GitHub y crea un **nuevo repositorio**.
2. Si ya tienes un proyecto local, créalo **vacío** (sin README, sin `.gitignore` ni licencia) para evitar conflictos con tu historial.
3. Copia la URL del repositorio.

### 6.2 Autenticación (importante)

GitHub **ya no acepta tu contraseña de cuenta** para hacer `push` por HTTPS. Hay dos opciones:

| Opción | Cómo funciona | Recomendada |
|---|---|---|
| 🔑 **Clave SSH** | Generas un par de claves y subes la pública a GitHub | ✅ Sí |
| 🎫 **Token de acceso personal** | Lo generas en GitHub y lo usas como contraseña en HTTPS | Válida |

#### Configurar SSH (resumen)

```bash
ssh-keygen -t ed25519 -C "tucorreo@ejemplo.com"     # genera el par de claves
cat ~/.ssh/id_ed25519.pub                            # muestra la clave PÚBLICA
```

1. Copia el contenido que muestra el último comando.
2. En GitHub: **Settings → SSH and GPG keys → New SSH key** y pégala.
3. Comprueba la conexión:

```bash
ssh -T git@github.com
```

> ⚠️ **Solo se comparte la clave pública** (`.pub`). La clave privada (`id_ed25519`) **nunca** se sube ni se comparte.

Con SSH, la URL del repositorio tiene este formato: `git@github.com:usuario/repositorio.git`

### 6.3 Enlazar y subir un proyecto local

```bash
git remote add origin git@github.com:usuario/repositorio.git
git branch -M main
git push -u origin main
```

| Comando | Qué hace |
|---|---|
| `git remote add origin <url>` | Registra el remoto con el alias `origin` (el nombre habitual) |
| `git branch -M main` | Renombra la rama actual a `main` |
| `git push -u origin main` | Sube los commits y deja enlazada la rama con el remoto |

> 💡 El `-u` solo hace falta la **primera vez**. Después basta con `git push`.

Comprobar los remotos configurados:

```bash
git remote -v
```

Verás dos líneas: `(fetch)` es de donde **recibes** cambios y `(push)` es adonde los **envías**.

### 6.4 Día a día con el remoto

```bash
git push        # envía tus commits a GitHub
git pull        # trae los cambios de GitHub a tu máquina
```

> ℹ️ `git pull` equivale a `git fetch` (descargar) + `git merge` (fusionar). **Haz `pull` antes de empezar a trabajar** para partir de la última versión.

---

## 7. 🙈 El archivo `.gitignore`

Indica qué archivos **no** debe controlar Git (logs, contraseñas, archivos temporales...). Se crea en la raíz del proyecto:

```bash
nano .gitignore
```

Ejemplo:

```
*.log
*.class
.env
node_modules/
```

| Patrón | Ignora |
|---|---|
| `*.log` | Todos los archivos que terminen en `.log` |
| `.env` | El archivo `.env` (suele contener contraseñas o claves) |
| `node_modules/` | Toda esa carpeta |

> ⚠️ **Hazlo antes del primer commit.** Si un archivo ya se subió, ignorarlo después no lo borra del historial. Y **nunca** subas contraseñas, tokens ni claves privadas a un repositorio.

---

## 8. 🧰 Chuleta de comandos

| Comando | Para qué sirve |
|---|---|
| `git --version` | Ver la versión instalada |
| `git config --global user.name "..."` | Configurar tu nombre |
| `git config --global user.email "..."` | Configurar tu correo |
| `git config --list` | Ver la configuración |
| `git init` | Crear un repositorio nuevo |
| `git clone <url>` | Descargar un repositorio existente |
| `git status` | Ver el estado de los archivos |
| `git add <archivo>` / `git add -A` | Pasar archivos al staging |
| `git commit -m "mensaje"` | Guardar los cambios en el historial |
| `git log` / `git log --oneline` | Ver el historial |
| `git remote add origin <url>` | Conectar con un repositorio remoto |
| `git remote -v` | Ver los remotos configurados |
| `git push` | Enviar commits al remoto |
| `git pull` | Traer cambios del remoto |

### ↩️ Deshacer (comandos modernos)

| Quiero... | Comando |
|---|---|
| Sacar un archivo del staging | `git restore --staged <archivo>` |
| Descartar cambios sin guardar de un archivo | `git restore <archivo>` |
| Cambiar el mensaje del último commit | `git commit --amend -m "nuevo mensaje"` |

> ⚠️ `git restore <archivo>` **borra tus cambios sin guardar** y no se puede deshacer. Y no uses `--amend` sobre un commit que ya hayas subido con `push`.

---

<sub>📘 Guía rápida de Git · Implantación de Aplicaciones Web · 2º ASIR<br>Basada en el *Taller de introducción a git y GitHub* de José Juan Sánchez, licencia CC BY-SA 4.0, con correcciones y ampliaciones.</sub>
