# Rumbo — landing page

Landing de una sola página para **Rumbo**, una agenda física de desarrollo profesional (metas de carrera, hábitos, calendario). Todo el sitio vive en **un solo archivo HTML** (`index.html`): sin build step, sin dependencias de npm, sin framework. Se abre directo en el navegador o se sirve con cualquier servidor estático.

El sitio usa **Three.js** (cargado por CDN vía import map) para varias escenas 3D a lo largo del scroll, y un selector de pedido por WhatsApp con cantidades por color.

## Cómo correrlo localmente

No hace falta instalar nada. Solo sirve el archivo con cualquier servidor estático, por ejemplo:

```bash
python3 -m http.server 8743
```

y abre `http://localhost:8743`.

> Evita abrir `index.html` directo con `file://` — los `import` de módulos de Three.js y la carga de imágenes vía `fetch`/`texture loader` pueden fallar por CORS en `file://`. Siempre usa un servidor local.

## Estructura del proyecto

```
index.html     ← todo: HTML + CSS + JS, una sola página
assets/        ← fotos de producto (jpg comprimidos con sips)
```

No hay `package.json`, no hay bundler, no hay carpeta `src/`. Todo el CSS está en un `<style>` al inicio del `<head>`, y el JS está repartido en:

- Varios `<script type="module">` — cada uno es una escena Three.js independiente (Hero, "Qué hay adentro", Ediciones, "¿Es para ti?").
- Un `<script>` normal al final — lógica de UI que no depende de Three.js (scroll suave, selector de edición/cantidad, links de WhatsApp, reveal-on-scroll, nav).

## Secciones de la página (de arriba a abajo)

1. **Hero** (`header.hero`) — titular, CTA, foto de producto con escena 3D sutil (`#hero-stage`).
2. **"Qué hay adentro"** (`#inside`) — recorrido de cámara en 3D guiado por scroll, muestra las páginas interiores de la agenda.
3. **Ediciones** (`#editions`) — selector de color/edición con preview 3D con crossfade entre tapas, más el selector de cantidad y pedido por WhatsApp (ver abajo).
4. **Testimonios** (`#testimonials`) — placeholder hasta que haya reseñas reales.
5. **"¿Es para ti?"** (`#fit`) — comparación con puntos ambiente en 3D de fondo.
6. **Footer** y **barra fija de WhatsApp en mobile** (`.sticky-wa`).

## Cómo funciona cada pieza clave

### Escenas 3D (Three.js)

Todas comparten el mismo patrón, pensado para que el sitio nunca se rompa si WebGL falla o el usuario pide menos movimiento:

1. Al inicio de cada módulo: si `prefers-reduced-motion: reduce` está activo, o si el elemento del DOM que necesita no existe, el módulo **no hace nada** y se sale.
2. El `await import('three')` y todo el setup de Three.js van en un `try/catch`. Si algo falla (WebGL no disponible, CDN caído, etc.), se añade la clase `no-3d` al contenedor de esa escena.
3. El CSS mapea `.no-3d` a mostrar un `<img>`/contenido plano en vez del `<canvas>` — así que siempre hay un fallback visual, nunca una sección rota o vacía.
4. Cada render loop solo corre mientras su sección está en el viewport (`IntersectionObserver`) y la pestaña está visible (Page Visibility API) — para no gastar GPU de más cuando no se ve.

Para agregar una nueva escena 3D en otra sección, copia este patrón de uno de los módulos existentes (el de Hero es el más simple) en vez de empezar desde cero.

**Nota técnica:** siempre usa `renderer.setClearColor(color, 1)` además de `scene.background = new THREE.Color(...)` — solo `scene.background` deja el color de fondo real del WebGL en negro aunque la escena "se vea" con el color correcto en el código.

### Selector de edición + cantidad + pedido por WhatsApp

En `#editions`, cada color (Medianoche/Arena/Terracota) tiene:

- Un botón "swatch" que cambia la foto/descripción en preview (variable `currentEdition` en el script final).
- Un stepper de cantidad (`data-qty-for="NombreColor"`) que suma/resta sobre el objeto `cart` en JS.

La función `renderCart()` decide qué mensaje de WhatsApp armar:

- **Carrito vacío** (todas las cantidades en 0): se usa el flujo simple de siempre — un solo color, cantidad implícita de 1, mensaje tipo *"Hola, quiero pedir mi Rumbo, Edición Medianoche ($42)."*
- **Carrito con algo** (cualquier color con cantidad > 0): se muestra una caja de resumen (`#order-summary`) con el detalle línea por línea y el total, y el botón cambia a *"Pedir por WhatsApp — N unidades"* con un mensaje compuesto tipo *"Hola, quiero pedir: 2x Medianoche, 1x Terracota (3 unidades, $126 total)."*

Para agregar una edición nueva: duplica un `.edition-row` en el HTML (botón swatch + stepper) y agrega el nombre al array `EDITIONS` en el JS — el resto (resumen, total, mensaje) se recalcula solo.

El precio unitario vive en una sola variable: `UNIT_PRICE` (actualmente `42`).

### Número de WhatsApp — **pendiente de reemplazar**

```js
var WHATSAPP_NUMBER = '51999999999'; // reemplazar por el numero real del negocio
```

Esto es un **placeholder**. Antes de lanzar el sitio hay que cambiarlo por el número real (formato internacional sin `+` ni espacios, ej. `51987654321`).

### Scroll suave

El sitio usa un scroll inercial hecho a mano (no una librería) para que la navegación se sienta como una "película" en vez de scroll nativo brusco. Está escrito dentro del `<script>` final, activo solo en desktop (`pointer: fine`) y respeta `prefers-reduced-motion`. Si necesitas tocarlo, busca el comentario que explica por qué no se usó Lenis (se probó dos veces y falló en este entorno).

## Haciendo cambios comunes

| Quiero... | Dónde tocar |
|---|---|
| Cambiar el precio | variable `UNIT_PRICE` en el script final |
| Cambiar el número de WhatsApp | variable `WHATSAPP_NUMBER` |
| Agregar/quitar una edición de color | `.edition-row` en el HTML de `#editions` + array `EDITIONS` en JS |
| Cambiar textos de cualquier sección | directo en el HTML, son textos planos |
| Agregar una nueva escena 3D | copiar el patrón de un `<script type="module">` existente (ver sección de arriba) |
| Cambiar colores/tipografía | variables CSS en `:root` al inicio del `<style>` (`--ink`, `--accent`, `--surface`, etc., con variantes para dark mode) |
| Agregar/quitar una sección completa | buscar el `<section>` correspondiente por su `id` |

## Flujo de trabajo (GitFlow)

- `master` → siempre desplegable, es lo que está publicado.
- `develop` → rama de integración, base para el siguiente trabajo.
- `feature/<nombre-descriptivo>` → una rama por cambio, creada desde `develop`, mergeada de vuelta por PR.

Commits en formato [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `build:`, etc.

## Publicar cambios

No hay build. Cualquier cambio a `index.html` o `assets/` ya está "compilado" — solo hace falta commitear y pushear.

Este sitio se publica con **GitHub Pages** desde la rama `master`, carpeta raíz (`/`). Un push a `master` actualiza el sitio en vivo automáticamente (puede tardar uno o dos minutos en propagarse).

## Para quien retome el mantenimiento

Ver [`CONTRIBUTING.md`](./CONTRIBUTING.md) — tiene las reglas de implementación que costaron tiempo de debugging (Three.js, scroll, etc.) y no son obvias leyendo el código una sola vez. Es un documento plano, sin dependencia de ninguna herramienta en particular: sirve igual si trabajas con Claude, ChatGPT, Copilot o a mano.
