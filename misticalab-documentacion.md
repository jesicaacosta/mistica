# Místicalab — Documentación técnica

Sitio de una sola página (`misticalab-tienda.html`), sin build ni dependencias de servidor. Todo el HTML/CSS/JS vive en un único archivo autocontenido: se abre directo en un navegador o con Live Server, no requiere `npm install` ni backend.

## 1. Arquitectura

- **Un solo archivo.** `<style>` en el `<head>` (todo el CSS), `<script>` al final de `<body>` (todo el JS). No hay módulos ni imports.
- **Render por JS, no HTML estático.** Las secciones de catálogo, carrusel, filtros y redes están vacías en el HTML (`<main id="catalog"></main>`, etc.) y se llenan con `innerHTML` a partir de arrays de datos, en el arranque del script.
- **Sin backend.** No hay servidor, base de datos ni API propia. La única persistencia es `localStorage` del navegador (carrito), por diseño y por seguridad (ver §5, limitación de Mercado Pago).
- **Un solo estado global mutable:** el objeto `cart`. Todo lo demás (`PRODUCTS`, `READY_PRODUCTS`, `PROMOS`, `SOCIAL_LINKS`, `GARMENT_TYPES`) es de solo lectura en tiempo de ejecución; se edita a mano en el código, no desde la UI.

## 2. Mapa del archivo (buscar por estos comentarios/anclas)

| Bloque | Qué contiene |
|---|---|
| `:root{...}` | Variables CSS de color/tema. Único lugar para cambiar la paleta. |
| `header`, `nav#site-nav`, `#menu-toggle` | Header sticky + menú responsive (hamburguesa <780px, horizontal ≥780px). |
| `.carousel`, `#carousel-track` | Carrusel de promos (CSS + estructura). |
| `#nosotros` | Sección estática "Quiénes somos" (texto fijo en el HTML, no en JS). |
| `#productos-tools`, `#filter-row`, `#search-input` | Barra de búsqueda + chips de categoría. |
| `.card`, `#product-modal` | Estilos de tarjeta de producto y modal de detalle. |
| `#cart-drawer`, `#overlay` | Panel lateral del carrito; el overlay es compartido con el modal. |
| `#stock-head`, `#ready-catalog` | Sección "Stock ya hecho" (piezas con stock real y limitado). |
| `#gracias` | Pantalla de agradecimiento post-compra (oculta, se activa por `#gracias` en la URL). |
| `ZONA DE CONFIGURACIÓN` (dentro del `<script>`) | **Todo lo editable sin tocar lógica**: ver §3. |
| Funciones JS | Ver §4, tabla de referencia. |
| `ARRANQUE` (final del `<script>`) | Llama a todos los `render*()` una vez al cargar la página. |

## 3. Zona de configuración (`<script>`, primeras ~150 líneas)

### 3.1 Constantes sueltas

| Constante | Tipo | Uso |
|---|---|---|
| `WHATSAPP_NUMBER` | string | Número con código de país, sin `+` ni espacios. Usado en checkout multi-línea y en `SOCIAL_LINKS`. |
| `MP_LINK_MONTO_LIBRE` | string (URL) | Link de pago de Mercado Pago con monto editable por el comprador. Se abre cuando el carrito tiene más de una línea. |
| `SIZES` | string[] | Talles por defecto (S–7XL). Fallback si el producto ni su `type` definen talles propios. |
| `COLORS` | string[] | Colores por defecto: Blanco, Negro, Gris, Rojo, Azul marino. |
| `CUSTOM_NOTE` | string | Texto fijo "Prenda personalizada… 10 a 15 días hábiles". Se inyecta en toda tarjeta/modal del catálogo a pedido (no en Stock ya hecho). |
| `GARMENT_TYPES` | objeto | Ver §3.2. |

### 3.2 `GARMENT_TYPES` — catálogo de tipos de prenda

Objeto `{ clave: { label, material, sizes?, note? } }`. `sizes` y `note` son **opcionales**: si faltan, se usa `SIZES` (S–7XL) y no se muestra nota.

```js
remera:              { label, material }                         // S–7XL
remera_corte_mujer:  { label, material, sizes:["S","M","L","XL"] }
remera_oversize:     { label, material }                         // S–7XL
remera_modal:        { label, material }                         // S–7XL, tela modal
top_tiritas:         { label, material, sizes:["Único"], note }
crop_top:            { label, material, sizes:["Único"], note }
buzo_canguro:        { label, material }                         // S–7XL
buzo_cuello_redondo: { label, material }                         // S–7XL
campera_canguro:     { label, material }                         // S–7XL
```

**Regla de resolución de talles** (misma lógica en `renderCatalog` y `openProductModal`):
`p.sizes || GARMENT_TYPES[p.type]?.sizes || SIZES` — el producto individual manda sobre el tipo, y el tipo manda sobre el default global.

### 3.3 `PRODUCTS` — catálogo a pedido (sin stock, siempre disponible)

```ts
{
  id: string,        // único, sin espacios, usado como clave de <select> y de carrito
  name: string,
  tag: string,        // categoría libre → alimenta los chips de filtro (se derivan con new Set)
  type: keyof GARMENT_TYPES,
  price: number,      // en pesos, sin formato
  img: string,        // URL directa a la imagen; "" → placeholder SVG generado en runtime
  mpLink: string,     // link de pago individual de Mercado Pago; "" → botón avisa que falta cargarlo
  desc: string,       // texto largo, solo se ve en el modal
  sizes?: string[],   // override puntual del tipo de prenda
  colors?: string[],  // override puntual de COLORS
  stock?: false       // si se define en false, oculta el producto como agotado (deshabilita botones + selects). Omitido = siempre disponible.
}
```

### 3.4 `READY_PRODUCTS` — stock real y limitado

Mismo shape que `PRODUCTS` pero con **talle y color fijos** (no hay `<select>`, son texto) y `qty` en vez de `stock`:

```ts
{ id, name, tag, type, price, size: string, color: string, qty: number, img, mpLink, desc }
```

`qty <= 0` → se muestra el badge "Agotado" y se deshabilitan los botones. No hay decremento automático de `qty` al comprar (no hay backend que confirme el pago): hay que bajarlo a mano cuando se vende.

### 3.5 `PROMOS` y `SOCIAL_LINKS`

```ts
PROMOS: { tag, title, text, img }[]           // slides del carrusel, en orden de aparición
SOCIAL_LINKS: { label, icon, url, whatsapp? }[] // whatsapp:true → botón sólido destacado
```

## 4. Referencia de funciones

### 4.1 Catálogo, búsqueda y filtro

| Función | Firma | Qué hace |
|---|---|---|
| `getVisibleProducts()` | `() → Product[]` | Filtra `PRODUCTS` por `activeCategory` (variable global) y `searchTerm` (variable global, substring case-insensitive sobre `name`). |
| `onSearchInput(value)` | inputs del `<input>` | Setea `searchTerm` y vuelve a llamar `renderCatalog()`. |
| `setCategory(tag)` | click de un chip | Setea `activeCategory`, re-renderiza catálogo y chips (para marcar el activo). |
| `renderFilters()` | — | Reconstruye los chips a partir de `["Todos", ...new Set(PRODUCTS.map(p => p.tag))]`. Se debe volver a llamar si `PRODUCTS` cambia en runtime (no aplica si solo se edita el array en el código antes de cargar). |
| `renderCatalog()` | — | Pinta `#catalog` con las cards visibles: imagen, tag, nombre, línea de material (`GARMENT_TYPES`), precio, `CUSTOM_NOTE`, selects de talle/color, botones. |
| `renderReadyCatalog()` | — | Igual que arriba pero sobre `READY_PRODUCTS`, sin selects (talle/color en texto) y agotado por `qty`. |
| `placeholderImg(name)` | `string → data:URI` | SVG generado en runtime con el nombre del producto, para cuando `img` está vacío. No depende de red. |
| `formatPrice(n)` | `number → string` | `n.toLocaleString("es-AR")` con `$` adelante. |

### 4.2 Carrito

**Modelo de datos:** `cart` es un objeto `{ [clave]: cantidad }`. La clave compuesta es:

```
`${id}__${size}__${color}`
```

Esto permite que el mismo producto exista varias veces en el carrito con combinaciones distintas de talle/color, cada una como línea independiente.

| Función | Firma | Qué hace |
|---|---|---|
| `parseCartKey(key)` | `string → {id, size, color}` | `key.split("__")`. Único punto de la app que interpreta el formato de clave — si se cambia el separador `__`, se cambia solo acá. |
| `saveCart()` / carga inicial | — | `localStorage.setItem/getItem("misticalab_cart", JSON...)`, ambos en `try/catch` (silencioso si falla). |
| `getSelectedSize(id, prefix)` / `getSelectedColor(id, prefix)` | `(string, string?) → string` | Lee el `<select>` con id `prefix+id` (`"size-"`/`"color-"` en tarjetas, `"modal-size-"`/`"modal-color-"` en el modal). |
| `addToCart(id, sizeFromModal?, colorFromModal?)` | — | Si no recibe talle/color por parámetro, los lee del `<select>` de la tarjeta. Sirve tanto para catálogo normal (llamado sin argumentos extra) como para Stock ya hecho y el modal (llamado con talle/color literales). |
| `changeQty(key, delta)` | `(string, 1\|-1)` | Suma/resta 1; si llega a 0, borra la clave. |
| `removeFromCart(key)` | — | Borra la línea completa. |
| `renderCart()` | — | Recalcula cantidad total (badge del header) y total en pesos; redibuja `#cart-items`. |
| `openCart()` / `closeCart()` | — | Togglean las clases `.open` de `#cart-drawer` y `#overlay`. |

### 4.3 Checkout / Mercado Pago

⚠️ **Limitación de diseño, no un bug:** esta web no tiene servidor propio, así que no puede crear un cobro dinámico por el total exacto del carrito sin exponer una clave secreta de Mercado Pago en el cliente. Por eso la lógica es:

| Función | Comportamiento |
|---|---|
| `buyNow(id, size?, color?)` | Compra directa de 1 producto: abre `p.mpLink` en pestaña nueva. Si `mpLink` está vacío, `alert()` avisando que falta configurarlo. Si viene de la tarjeta, lee talle/color de los `<select>`; si viene del modal o de Stock ya hecho, recibe los valores como parámetro. |
| `checkout()` | Si el carrito tiene **una sola línea**, se comporta como `buyNow` (va directo a `mpLink` de ese producto). Si tiene **dos o más líneas**, abre `MP_LINK_MONTO_LIBRE` (para que el comprador cargue el total a mano) **y** abre WhatsApp (`wa.me`) con el detalle del pedido armado en texto, para que la dueña confirme manualmente. |

Para reemplazar esto por un cobro automático de carrito completo, hace falta un backend (aunque sea una función serverless) que llame a la API de Mercado Pago con el access token guardado del lado servidor — no es algo que se pueda resolver solo en este archivo.

### 4.4 Modal de detalle de producto

`openProductModal(id, fromReady?)`: `fromReady` (booleano) decide si busca el `id` en `PRODUCTS` o en `READY_PRODUCTS`, y arma el HTML interno (`pickers`, `addBtn`, `buyBtn`) de forma condicional: selects editables para catálogo normal, texto fijo para Stock ya hecho. `closeProductModal()` limpia las clases `.open` (comparte overlay con el carrito).

### 4.5 Carrusel, menú y "gracias"

| Función | Qué hace |
|---|---|
| `renderCarousel()` | Pinta slides y puntitos desde `PROMOS`. |
| `moveSlide(delta)` / `goToSlide(i)` | Cambian `currentSlide` (con wraparound por `%`) y llaman `updateCarouselPosition()`, que aplica `translateX`. |
| Autoplay | `setInterval(() => moveSlide(1), 6000)`, corre siempre, no se pausa al interactuar. |
| `toggleMenu()` / `closeMenu()` | Togglean `.open` en `#site-nav` (solo visible en mobile por CSS). |
| `showThanksIfNeeded()` | Se ejecuta en el arranque y en `window.addEventListener("hashchange", ...)`. Muestra `#gracias` si `location.hash === "#gracias"`. Pensado para configurar como URL de retorno en Mercado Pago. |
| `closeThanks()` | Limpia el hash y oculta la sección. |

## 5. Publicación / hosting

El archivo es autocontenido (fuentes vía Google Fonts, sin JS externo). Se puede:
- Abrir directo con doble click o Live Server para desarrollo.
- Subir tal cual a cualquier hosting estático (Netlify, GitHub Pages, Vercel, etc.) — es un solo `.html`, no necesita build.
- Publicar como Claude Artifact (como se hizo en esta conversación), que le da una URL propia.

**Cosas a las que hay que prestar atención si se autoalojan** (fuera de Claude): el CSP de Claude Artifacts bloquea imágenes externas salvo unos pocos hosts; en un hosting propio esa restricción no existe, así que ahí sí se pueden usar imágenes de cualquier CDN.

## 6. Guía rápida de extensión

| Quiero... | Dónde tocar |
|---|---|
| Agregar un producto a pedido | Copiar un bloque de `PRODUCTS`, pegar antes del `]`. |
| Agregar una pieza con stock real | Copiar un bloque de `READY_PRODUCTS`, con `size`/`color`/`qty` fijos. |
| Agregar un tipo de prenda nuevo | Sumar una clave a `GARMENT_TYPES` con `label`/`material` (y `sizes`/`note` si aplica). |
| Agregar una categoría de filtro | No hay array separado: simplemente usar un `tag` nuevo en algún producto: `renderFilters()` lo detecta solo. |
| Cambiar colores/tipografías de la marca | Solo en las variables `:root{ --bg, --flame, ... }`. |
| Cambiar el texto de demora de producción | Una sola constante: `CUSTOM_NOTE`. |
| Sumar una red social al footer de contacto | Un objeto más en `SOCIAL_LINKS`. |

## 7. Limitaciones conocidas / posibles próximos pasos

- **Sin backend real:** carrito y "stock" de `READY_PRODUCTS` no se sincronizan entre visitantes ni se decrementan solos al pagar; todo depende de que la dueña actualice el código a mano.
- **Sin persistencia entre dispositivos:** el carrito vive en `localStorage` del navegador de cada visitante, no en un servidor.
- **Cobro combinado manual:** pedidos con más de un producto dependen de WhatsApp + link de monto libre, no de un cobro automático por el total exacto.
- **Sin analítica:** no hay tracking de qué se agrega más al carrito ni de conversión.
- Cualquiera de estos tres puntos requeriría sumar un backend (aunque sea mínimo, tipo función serverless + una base de datos chica); hoy la arquitectura es 100% estática a propósito, para que se pueda alojar gratis y sin mantenimiento de servidor.
