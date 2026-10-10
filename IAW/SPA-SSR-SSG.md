# 🟣 SPA vs SSR vs SSG — Cómo se genera una web

> **Tema:** formas de generar y entregar una página web
> **Pregunta clave:** ¿**quién** construye el HTML y **cuándo**?

---

## 📑 Índice

1. La idea clave
2. Glosario rápido
3. SPA — Single Page Application
4. SSR — Server-Side Rendering
5. SSG — Static Site Generator
6. Comparativa
7. ¿Cuándo usar cada una?
8. El enfoque híbrido
9. Matices a tener en cuenta
10. Relación con Nginx y Apache
11. Resumen en 3 líneas

---

## 1. 🧭 La idea clave

Toda web acaba siendo HTML en el navegador del usuario. Lo que cambia entre estos tres enfoques es **quién genera ese HTML y en qué momento**.

### 🍕 La analogía de la pizza

| Enfoque | En la analogía |
|---|---|
| **SSG** | Pizzas **precocinadas** en la estantería: ya están hechas, solo las coges |
| **SSR** | La pizzería la **hace al momento** cuando la pides |
| **SPA** | Te dan los **ingredientes** y la montas tú en casa (en el navegador) |

---

## 2. 📖 Glosario rápido

| Término | Significado |
|---|---|
| **Renderizar** | Generar el HTML final que ve el usuario |
| **Build (tiempo de construcción)** | Momento en que el desarrollador "compila" el sitio **antes** de publicarlo |
| **CDN** | Red de servidores repartidos por el mundo que guardan copias del sitio para servirlas desde el más cercano al usuario |
| **CMS headless** | Gestor de contenidos sin parte visual: solo almacena el contenido y lo entrega por API |
| **SEO** | Posicionamiento en buscadores: que Google entienda y encuentre tu página |

---

## 3. ⚡ SPA — Single Page Application

### ¿Cómo funciona?

El servidor entrega **un único HTML casi vacío** y un conjunto de archivos JavaScript. Es el **navegador** quien construye la página y quien pide los datos al servidor mediante API.

```
Navegador ──pide la página──►  Servidor
Navegador ◄── HTML casi vacío + JavaScript ──  Servidor
    │
    └─► El navegador ejecuta el JS, pide datos a la API y construye la página
```

### ✅ Ventajas

- Una vez cargada, **navegar es instantáneo**: no hay recargas entre secciones.
- Muy buena para experiencias **interactivas y personalizadas**.
- Mucho control sobre la arquitectura y uso de frameworks modernos.

### ❌ Inconvenientes

- La **primera carga** puede ser lenta, y empeora a medida que la aplicación crece.
- El **SEO es más difícil**, porque el HTML inicial no trae contenido.
- Los archivos JavaScript grandes se vuelven difíciles de mantener.

### 🛠️ Ejemplos de herramientas

React, Vue.js, Angular, Svelte.

---

## 4. 🖥️ SSR — Server-Side Rendering

### ¿Cómo funciona?

El servidor **genera la página completa en cada petición**: consulta la base de datos o el CMS, construye el HTML y lo envía ya terminado.

```
Navegador ──pide la página──►  Servidor
                                  │
                                  ├─ consulta la base de datos / CMS
                                  └─ construye el HTML completo
Navegador ◄── HTML completo ──  Servidor
```

### ✅ Ventajas

- El contenido está **siempre actualizado**: los cambios se ven al instante.
- Permite **personalizar** el contenido por usuario sin apaños.
- **Mejor SEO** que una SPA, porque el HTML llega ya con contenido.
- Funciona igual en cualquier dispositivo, sin depender de la potencia del cliente.

### ❌ Inconvenientes

- Suele requerir **más peticiones al servidor**.
- Por defecto suele ser **más lento** que SPA y SSG.
- Necesita una infraestructura que **aguante y escale** cuando crece el tráfico, porque el servidor trabaja en cada visita.

### 🛠️ Ejemplos

Next.js, Nuxt, SvelteKit y también las webs tradicionales con **PHP** (el servidor genera el HTML en cada petición).

---

## 5. 📄 SSG — Static Site Generator

### ¿Cómo funciona?

Las páginas se generan **una sola vez, al construir el sitio**. El resultado son archivos HTML estáticos ya hechos, que se publican y se sirven tal cual.

```
 Desarrollador / CMS
        │
        ▼
 ┌──────────────┐   build   ┌────────────────┐   publica   ┌──────┐
 │ Contenido +  │ ────────► │ Archivos HTML  │ ──────────► │ CDN  │ ──► Navegador
 │ plantillas   │           │ estáticos      │             └──────┘
 └──────────────┘           └────────────────┘
```

Suele combinarse con un **CMS headless**, un hosting estático y una **CDN**. Cuando cambia el contenido, un *webhook* avisa al generador, que reconstruye el sitio y lo vuelve a publicar.

### ✅ Ventajas

- **Muy rápido**: las páginas ya están hechas.
- **Muy buen SEO**.
- Infraestructura sencilla y **fácil de escalar**.
- Permite combinar contenido de varias fuentes.

### ❌ Inconvenientes

- Cada cambio de contenido exige **reconstruir el sitio**.
- La **personalización** y el contenido dinámico requieren servicios adicionales o soluciones más complejas.

### 🛠️ Ejemplos de herramientas

Astro, Hugo, Jekyll, Eleventy, Gatsby (y Next.js en modo estático).

---

## 6. 📊 Comparativa

| | ⚡ SPA | 🖥️ SSR | 📄 SSG |
|---|---|---|---|
| **¿Quién genera el HTML?** | El navegador | El servidor | Se genera al construir |
| **¿Cuándo?** | Al abrir la página | En cada petición | Una vez, antes de publicar |
| **Primera carga** | Lenta | Media | Muy rápida |
| **SEO** | Más difícil | Bueno | Muy bueno |
| **Contenido actualizado al instante** | Sí (vía API) | ✅ Sí | ❌ No, requiere rebuild |
| **Personalización** | Muy fácil | Fácil | Difícil |
| **Carga en el servidor** | Mínima | Alta | Casi nula |
| **Coste de infraestructura** | Bajo | Mayor | Muy bajo |
| **Navegación posterior** | Instantánea | Cada página se pide al servidor | Muy rápida |

---

## 7. 🎯 ¿Cuándo usar cada una?

| Si tu proyecto es... | Elige | Por qué |
|---|---|---|
| Una aplicación muy interactiva (panel, editor, herramienta) | ⚡ **SPA** | La experiencia tipo app y la fluidez importan más que el SEO |
| Una tienda online o contenido que cambia constantemente | 🖥️ **SSR** | Contenido siempre actualizado y personalizado, con buen SEO |
| Un blog, documentación o web corporativa | 📄 **SSG** | Rapidez, SEO y coste mínimo; el contenido cambia poco |

> 💡 **Regla práctica:** cuanto más **personalizado y cambiante** es el contenido, más te acercas a SSR o SPA. Cuanto más **estable**, más te conviene SSG.

---

## 8. 🔀 El enfoque híbrido

No hay que elegir una sola. Frameworks como **Next.js** permiten usar **SSG para unas páginas y SSR para otras** dentro del mismo proyecto:

| Parte del sitio | Enfoque | Motivo |
|---|---|---|
| Página de inicio, "Sobre nosotros", blog | SSG | Contenido estable, máxima velocidad |
| Carrito, perfil de usuario, stock en vivo | SSR | Datos personalizados y al día |

> ⚠️ Combinar enfoques requiere **planificar bien**, o la complejidad y los cuellos de botella pueden acabar compensando las ventajas.

**Conclusión:** no existe "el mejor". Depende del tipo de contenido, del público, del equipo y del presupuesto.

---

## 9. 🔎 Matices a tener en cuenta

1. **El artículo original tiene una fuente interesada.** Está escrito por Hygraph, una empresa de CMS headless, por lo que presenta muy favorablemente el modelo "SSG + CMS + CDN". Es útil, pero no es una fuente neutral.
2. **"El SEO de una SPA es casi imposible" es una exageración.** Los buscadores modernos ejecutan JavaScript. Lo correcto es que el SEO es **más difícil y menos fiable**, y se puede mitigar con prerenderizado o SSR.
3. **"SSR es más lento" es una tendencia, no una regla.** Con caché y una infraestructura adecuada, se puede compensar en gran medida.
4. **La frontera es difusa.** Un sitio estático (SSG) puede comportarse como una SPA una vez cargado en el navegador, por ejemplo si carga contenido con JavaScript.

---

## 10. 🌐 Relación con Nginx y Apache

| Enfoque | Qué papel juegan Nginx o Apache |
|---|---|
| 📄 **SSG** | Sirven directamente los **archivos HTML estáticos**. Es el caso más sencillo, igual que al servir una carpeta con un virtual host |
| ⚡ **SPA** | Sirven los archivos estáticos (HTML + JS) y la lógica de datos va en una API aparte |
| 🖥️ **SSR** | La aplicación corre en **otro proceso** (Node.js, PHP-FPM...) y Nginx o Apache actúan como **proxy inverso** delante de ella |

> 💡 Por eso Nginx y Apache siguen siendo relevantes aunque la web sea moderna: son la "puerta de entrada" que sirve los archivos o reenvía las peticiones a la aplicación.

---

## 11. 🧠 Resumen en 3 líneas

> **SPA** = lo monta el **navegador**.
> **SSR** = lo monta el **servidor** en cada visita.
> **SSG** = se monta **una vez**, antes de publicar.

---

<sub>📘 Documento de repaso · Basado en el artículo *"What is the Difference Between SPAs, SSGs, and SSR?"* de Hygraph (https://hygraph.com/blog/difference-spa-ssg-ssr), reexplicado y ampliado con matices propios.</sub>
