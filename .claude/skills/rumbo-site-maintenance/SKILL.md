---
name: rumbo-site-maintenance
description: Mantener y extender la landing page de Rumbo (agenda física de desarrollo profesional) — un único index.html con escenas Three.js, scroll suave a mano y pedido por WhatsApp con carrito de cantidades por color. Usar esta skill siempre que se pida modificar, corregir o agregar algo al sitio de Rumbo.
---

# Mantenimiento del sitio Rumbo

Este sitio es **un solo archivo** (`index.html`) sin build step: HTML + CSS + JS vanilla, Three.js cargado por CDN. No hay `npm install`, no hay bundler, no hay `src/`. Edita `index.html` directo.

Lee primero `README.md` en la raíz del repo — tiene el mapa completo de secciones, variables clave (`UNIT_PRICE`, `WHATSAPP_NUMBER`, `EDITIONS`) y la tabla de "cambios comunes". Esta skill complementa ese README con las reglas de diseño/implementación que no son obvias leyendo el código una sola vez.

## Cómo probar cambios

1. Sirve el archivo con un servidor estático real, nunca `file://`:
   ```bash
   python3 -m http.server 8743
   ```
2. Abre `http://localhost:8743` y **siempre cachebustea** la URL con un query param que cambie en cada prueba (`?cb=12345`) — el navegador sirve JS viejo en caché agresivamente y da falsos negativos.
3. Si estás probando algo que depende de scroll o `requestAnimationFrame` (las escenas 3D, el scroll suave), mantené la pestaña/panel de preview **visible y en foco** — un panel oculto limita el framerate del navegador y da síntomas falsos de "se congeló" o "no responde".
4. Probá tanto desktop como un viewport mobile (375×812) — el sitio tiene comportamientos específicos para mobile (barra de WhatsApp fija, layout del selector de ediciones).

## Reglas de las escenas Three.js

Cada escena 3D del sitio (Hero, "Qué hay adentro", Ediciones, "¿Es para ti?") sigue el mismo patrón defensivo. Si agregás una escena nueva, copiá este esqueleto:

```js
<script type="module">
(async () => {
  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;
  const stage = document.getElementById('mi-stage');
  if (!stage) return;
  try {
    const THREE = await import('three');
    // ...setup...
    renderer.setClearColor(0xfaf6ee, 1); // IMPORTANTE, ver nota abajo
    // ...render loop gateado por IntersectionObserver + Page Visibility...
  } catch (e) {
    stage.classList.add('no-3d'); // el CSS ya sabe mostrar el fallback <img>
  }
})();
</script>
```

Puntos que ya costaron tiempo de debugging en este proyecto, no los repitas:

- **`renderer.setClearColor(color, 1)` es obligatorio** además de `scene.background = new THREE.Color(...)`. Solo `scene.background` deja el clear color real del WebGL en negro — se nota recién al inspeccionar con `renderer.getClearColor()`, visualmente puede parecer que "ya está bien".
- **Stacking context**: un elemento `position:absolute`/`fixed` (como el `<canvas>` de una escena ambiente) pinta SIEMPRE encima de hermanos `position:static`, sin importar el orden en el DOM. Si el contenido de texto queda tapado por un canvas de fondo, el fix es `position:relative; z-index:1` en el contenido y `z-index:0` (o nada) en el canvas — no reordenar el DOM, eso no alcanza.
- **Colapso a 0×0**: si reemplazás un `<img>` por un `<canvas>`/`<div>` dentro de un contenedor flex con `align-items:center`, el canvas no tiene el fallback de tamaño intrínseco que sí tiene una imagen — puede colapsar a 0×0. Poné `width:100%` explícito en el wrapper y en el stage.
- **Render loops solo cuando son visibles**: usa `IntersectionObserver` sobre la sección y `document.visibilitychange`/`document.hidden` para pausar el `requestAnimationFrame` cuando la sección no está en pantalla o la pestaña está oculta. Sin esto el sitio gasta GPU de más en escenas que nadie está viendo.
- **"Cover" de texturas**: para que una imagen rellene un plano sin deformarse (como `object-fit: cover` pero en Three.js), calculá el frustum de la cámara en el plano del mesh (`2*tan(fov/2)*distance`), escalá el plano a ese tamaño, y ajustá `texture.repeat`/`texture.offset` comparando el aspect ratio del frustum contra el de la imagen.

## Scroll suave (inercial, hecho a mano)

El sitio NO usa Lenis ni ninguna librería de smooth-scroll — se probó Lenis dos veces en este proyecto y las dos veces su estado interno de scroll nunca llegó a mover la barra real (`window.scrollY` quedaba congelado pese a que el `ticker` corría). En su lugar hay un listener de `wheel` no-pasivo que acumula un `target` y un loop de `requestAnimationFrame` que interpola hacia él (factor `0.12`), más un listener de `scroll` normal que resincroniza el estado cuando el movimiento no vino de la rueda (teclado, scrollbar, anchor links). Solo corre en desktop (`pointer: fine`) y se desactiva completo bajo `prefers-reduced-motion`.

Si necesitás tocar la velocidad o el "peso" del scroll, el único número que importa es ese factor de interpolación (`0.12` — más alto = más rápido/menos inercia, más bajo = más lento/más inercia). No reintroduzcas Lenis sin verificar primero, en este mismo entorno de pruebas, que `lenis.on('scroll', cb)` realmente dispara y que `window.scrollY` se mueve.

## Carrito de pedido (cantidad por color)

`#editions` tiene un selector de color (swatch, cambia el preview) separado de un stepper de cantidad por color (`data-qty-for`). El estado vive en el objeto `cart` del script final. La función `renderCart()` es la única fuente de verdad del mensaje de WhatsApp y el texto del botón:

- Carrito vacío → vuelve al flujo simple de un solo ítem (compatibilidad con el comportamiento original).
- Carrito con algo → muestra `#order-summary` con el detalle y cambia el CTA a "Pedir por WhatsApp — N unidades".

Si agregás una edición nueva, agregala a la constante `EDITIONS` y duplicá un `.edition-row` — no hace falta tocar `renderCart()`.

## Qué NO hacer

- No agregues un bundler, framework o `package.json` — el punto del proyecto es que sea un solo archivo servible desde cualquier lado, incluido GitHub Pages sin build.
- No metas credenciales, tokens ni precios de prueba reales en el HTML — el número de WhatsApp (`WHATSAPP_NUMBER`) sigue siendo un placeholder (`51999999999`) hasta que el negocio confirme el real.
- No agregues 3D "decorativo" sin relación con el contenido de esa sección — cada escena existente tiene un propósito narrativo (mostrar el producto, mostrar el interior, mostrar las ediciones). Ver la skill `3d-website-architect` para el criterio completo de cuándo vale la pena usar 3D.
