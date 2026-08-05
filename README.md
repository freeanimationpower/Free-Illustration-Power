# Free Illustration Power — Programa de Ilustracion Digital por Capas

> **Accede directamente**: [freeanimationpower.org/tools/illustration/](https://freeanimationpower.org/tools/illustration/) — Ilustra con 94 pinceles sin instalar nada.

<img width="2172" height="724" alt="free vector images 1 linea" src="https://github.com/user-attachments/assets/6ffb24c5-5a43-4c3a-a06a-b6ac7051f0c0" />


## 1. Presentación del Proyecto

**FAP Draw** es una aplicación web de ilustración digital que funciona completamente en el navegador, sin dependencias externas, sin instalación, y sin backend. Es una evolución directa de **FAP (Free Animation Power)**, adaptando su motor de dibujo de 60 pinceles y su arquitectura vanilla JS hacia un flujo de trabajo de ilustración por capas, similar a Photoshop, Krita o Procreate.

### Motivación

FAP original (`VERSION WEB PC` y `VERSION WEB MOVIL`) es un programa de animación cuadro por cuadro. Funciona con un timeline horizontal donde cada frame es un lienzo independiente. Para ilustración estática, este modelo es limitado: necesitas trabajar con **capas** (layers) que se apilan verticalmente, cada una con su propio contenido, opacidad, visibilidad y bloqueo independientes.

**FAP Draw** reemplaza el timeline de animación por un **panel de capas** tipo Photoshop, manteniendo todo el poder del motor de pinceles original (ahora ampliado a 94 pinceles).

### Idea central

> "La misma libertad de FAP para dibujar, pero con la organización y control de un sistema de capas profesional."

---

## 2. Repositorio de Referencia

El proyecto se basa directamente en:

- **FAP Desktop (VERSION WEB PC)** — `D:\FAP WEB\VERSION WEB PC\index.html`
  - Aplicación single-file vanilla JS para animación frame-by-frame
  - Canvas 1920x1080, 60 pinceles, pointer events con presión de stylus
  - Export GIF/Video, Save/Load `.fap`
  - Repositorio GitHub: [github.com/freeanimationpower](https://github.com/freeanimationpower)
  - Sitio web: [freeanimationpower.org](https://freeanimationpower.org)

- **FAP Mobile (VERSION WEB MOVIL)** — `D:\FAP WEB\VERSION WEB MOVIL\index.html`
  - Versión adaptada para pantallas táctiles móviles
  - Mismo motor de pinceles, interfaz optimizada para móvil

**FAP Draw** nace como un fork conceptual de FAP Desktop, reemplazando el sistema de frames/timeline por un sistema de capas, y expandiendo el set de pinceles de 60 a 94.

---

## 3. Qué se Hizo y Cómo — Detalle Técnico

### 3.1 Arquitectura General

| Característica | Implementación |
|---|---|
| **Lenguaje** | JavaScript ES6+ vanilla (sin TypeScript, sin transpilador) |
| **Interfaz** | HTML5 + CSS3 (Flexbox, CSS Custom Properties) |
| **Renderizado** | Canvas 2D API (1920×1080 resolución interna) |
| **Dependencias** | **Cero** — sin npm, sin CDN, sin frameworks |
| **Build** | Innecesario — abrir `index.html` en el navegador |
| **Entrada** | Pointer Events API con `setPointerCapture`, `getCoalescedEvents`, presión y tilt |
| **Archivo** | Single-file: 1 archivo `index.html` (~1760 líneas) |

#### ¿Por qué single-file y sin dependencias?

1. **Portabilidad máxima**: funciona en cualquier navegador moderno sin servidor
2. **Sin configuración**: no requiere Node.js, Webpack, Vite ni nada similar
3. **Legado de FAP**: FAP original ya usaba esta arquitectura, lo que permite reutilizar el 100% del motor de pinceles
4. **Offline-first**: toda la lógica es local, no hay llamadas a servidores
5. **Distribución trivial**: un solo archivo que pesa menos de 100 KB sin comprimir

#### ¿Por qué Canvas 2D y no WebGL/WebGPU?

Canvas 2D permite:
- Manipulación directa de píxeles con `getImageData`/`putImageData` (esencial para flood fill, undo/redo)
- `globalCompositeOperation` para efectos de mezcla (destination-out para sal, alcohol, lifting)
- API simple y universal (funciona en cualquier navegador, incluyendo móviles)
- No requiere shaders ni GPU programming

La limitación es que no permite fluid simulation real (como Adobe Fresco con GPU), pero se compensa con técnicas de partículas procedurales, multi-capa, y texturizado por ruido determinista.

---

### 3.2 Sistema de Capas (Layer System)

#### Estructura de datos

```javascript
layers: [
  {
    id: 'layer_1',
    name: 'Capa 1',
    canvas: <offscreenCanvas>,  // 1920×1080
    visible: true,
    locked: false,
    opacity: 1.0,               // 0.0 a 1.0
    undoStack: [ImageData, ...], // máx 30 estados
    redoStack: [ImageData, ...]
  },
  ...
]
activeLayerIndex: 0
```

#### ¿Por qué cada capa tiene su propio canvas offscreen?

1. **Aislamiento**: dibujar en una capa no afecta a las demás
2. **Persistencia**: el contenido de cada capa se guarda como imagen (no como comandos)
3. **Composición rápida**: `renderComposite()` solo hace `drawImage()` de cada canvas de capa sobre el canvas de display, en orden bottom-up
4. **Undo/redo por capa**: guardar/restaurar `ImageData` es directo sin afectar otras capas
5. **Exportación**: cada capa se puede exportar independientemente a PNG (data URL)

#### Flujo de renderizado

```
renderComposite():
  1. ctx.clearRect() + fill blanco
  2. Para cada capa (i=0 → n-1, bottom-up):
     if visible && opacity > 0:
       ctx.globalAlpha = layer.opacity
       ctx.drawImage(layer.canvas, 0, 0)
       ctx.globalAlpha = 1
  3. drawGrid() (si está activado)
```

#### Flujo de dibujo sobre capa activa

```
startDraw():
  1. syncLayerBuffer()       — copia layer.canvas → layerBuffer
  2. saveUndoState()          — guarda ImageData en undoStack
  3. drawDot() sobre ctx (display) + lbCtx (buffer)

moveDraw():
  1. Para cada evento coalescido:
     - Interpola puntos entre lastPos y currentPos
     - Aplica presión multi-parámetro
     - Renderiza brush segment en ctx + lbCtx

endDraw():
  1. flushLayerBuffer()       — copia layerBuffer → layer.canvas
  2. renderComposite()        — redibuja todo el composite
```

#### ¿Por qué usar layerBuffer intermedio?

El buffer intermedio (`layerBuffer`) permite:
- Dibujar el trazo completo antes de guardarlo permanentemente en la capa
- Si el usuario cancela (ej. pierde el foco), el trazo parcial no se guarda
- Mantiene sincronizados el canvas de display (`ctx`) y el buffer durante el trazo
- Facilita el undo: solo se guarda el estado al INICIO del trazo, no en cada punto

---

### 3.3 Motor de Pinceles — 94 Pinceles

#### Organización

| # | Categoría | Cantidad | Descripción |
|---|---|---|---|
| 1 | **Clásicos** | 20 | round, square, pencil, soft, calligraphy, marker, ink, chalk, glow, pixel, splatter, watercolor, charcoal, crayon, hardEraser, dots, hatch, neon, fur, drip |
| 2 | **Texturas** | 20 | airbrush, sand, sponge, pastel, gravel, rake, chain, zigzag, wave, lace, oil, dryBrush, smudge, acrylic, gouache, sparkle, comet, graffiti, spiderweb, smoke |
| 3 | **Efectos** | 20 | confetti, rain, bubbles, vines, scales, stitch, grid, embers, crackle, diamond, feather, wire, pebble, brick, scribble, drizzle, tribal, frost, marble, holographic |
| 4 | **Acuarela Húmeda** | 5 | Aguada húmeda redonda, plana, floración, detalle, mezcla |
| 5 | **Acuarela Seca** | 5 | Textura seca redonda, granulación, áspero, perfilador, sal |
| 6 | **Aguadas/Lavados** | 4 | Degradado, variegado, plano, húmedo sobre húmedo |
| 7 | **Bordes** | 4 | Suave, duro, sangrado, plumoso |
| 8 | **Textura Papel** | 3 | Prensado frío, prensado caliente, rugoso |
| 9 | **Salpicaduras** | 3 | Fina, goteo, floración |
| 10 | **Tinta/Gouache** | 3 | Tinta china, gouache opaco, gouache semi-transparente |
| 11 | **Efectos Acuarela** | 3 | Sal, alcohol, levantado/secado |
| 12 | **Óleo** | 4 | Espátula, cerda, empaste, difumino |

**Total: 94 pinceles**

#### ¿Por qué 94 pinceles y no solo los 30 de acuarela?

Los 60 pinceles originales de FAP representan técnicas de dibujo tradicional (lápiz, tiza, carboncillo, marcador, tinta) y efectos especiales (neón, graffiti, telaraña, fuego, etc.). Eliminarlos habría reducido la versatilidad del programa. En lugar de eso, se **sumaron** 34 pinceles nuevos especializados en acuarela y óleo sobre los 60 existentes.

#### Anatomía de un pincel

Cada pincel se define con:
```javascript
{
  cat: 'classic'|'water'|'oil',   // categoría para curvas de presión
  cap: 'round'|'square'|'butt',   // forma del extremo de línea
  join: 'round'|'miter',           // forma de unión entre segmentos
  scale: 1.0,                      // multiplicador de tamaño
  alpha: 1.0,                      // multiplicador de opacidad base
  category: 'classic'|'watercolor'|'oil',  // curvas de presión
  // Flags específicos (ej: wetWash:true, saltEffect:true, etc.)
}
```

#### Pipeline de renderizado de un pincel

```
renderBrushSegment(ctx, brush, color, alpha, lineWidth, x, y, dx, dy, dist, pressureParams):
  ├── ¿Tiene flag de acuarela/óleo nuevos?
  │   └── Sí → renderizado especializado (30+30 bloques únicos)
  ├── ¿Tiene flag de pincel FAP original?
  │   └── Sí → drawOriginalBrush() (50+ bloques de renderizado)
  └── No → stroke simple (lineTo entre puntos)
```

Los pinceles de acuarela nuevos usan técnicas como:
- **Multi-capa translúcida**: 3-8 capas de círculos/rectángulos con alpha decreciente
- **Bloom/Borde**: anillos oscuros en los bordes con centro claro (efecto de secado)
- **Difusión por partículas**: 15-60 partículas aleatorias con distribución controlada por presión
- **Textura de papel procedural**: ruido determinista basado en `sin/cos` de la posición que omite píxeles
- **Operaciones de composición**: `destination-out` para efectos de sal, alcohol, y lifting
- **Mezcla de color**: `hueShift()` para pinceles variegados que varían el tono dentro del trazo

Los pinceles de óleo nuevos usan:
- **Espátula**: rectángulo rotado con líneas paralelas de textura
- **Cerda**: múltiples líneas paralelas con alpha variable por cerda
- **Empaste**: highlights y sombras que crean ilusión de relieve 3D
- **Difumino**: muestrea color del canvas (`getImageData`) y lo arrastra con alpha bajo

---

### 3.4 Sistema de Sensibilidad Multi-Parámetro

#### ¿Por qué multi-parámetro en vez de solo tamaño?

En software profesional como Fresco, la presión del lápiz controla múltiples aspectos simultáneamente para simular el comportamiento real de la pintura:
- Más presión = más pintura depositada = más opacidad Y más tamaño Y menos dispersión
- Menos presión = el pincel apenas roza = menos opacidad Y menos tamaño Y más dispersión

#### Curvas de respuesta por categoría

```
PRESSURE_CURVES = {
  classic: {
    size:    {min:0.20, max:1.00, curve:'linear'}   // 20%→100%
    opacity: {min:0.30, max:1.00, curve:'linear'}   // 30%→100%
    scatter: {min:0.80, max:0.20, curve:'soft'}     // inverso
    flow:    {min:0.50, max:1.00, curve:'linear'}   // 50%→100%
    grain:   {min:0.30, max:0.00, curve:'soft'}     // inverso
  },
  watercolor: {
    size:    {min:0.15, max:1.00, curve:'soft'}     // 15%→100%, curva suave
    opacity: {min:0.08, max:0.90, curve:'soft'}     // 8%→90%, más rango bajo
    scatter: {min:1.50, max:0.25, curve:'soft'}     // más dispersión en baja presión
    flow:    {min:0.15, max:1.00, curve:'soft'}
    grain:   {min:1.00, max:0.20, curve:'soft'}     // más grano visible
    bloom:   {min:0.60, max:0.10, curve:'soft'}     // bloom más intenso con poca presión
  },
  oil: {
    size:    {min:0.20, max:1.00, curve:'linear'}
    opacity: {min:0.30, max:1.00, curve:'hard'}     // respuesta más abrupta
    scatter: {min:0.80, max:0.15, curve:'soft'}
    flow:    {min:0.40, max:1.00, curve:'linear'}
    grain:   {min:0.50, max:0.05, curve:'hard'}
  }
}
```

#### Funciones de easing

- **linear**: `t` (respuesta directa)
- **soft**: `t < 0.5 ? 2t² : 1 - (-2t+2)²/2` (ease-in-out, más control en presiones medias)
- **hard**: `t²` (respuesta cuadrática, más control en presiones bajas)

La curva `soft` es ideal para acuarela porque da más rango de control en presiones bajas (donde ocurre la mayor parte del trabajo con acuarela). La curva `hard` es mejor para óleo porque la pintura espesa responde más abruptamente a la presión.

#### Sliders de control manual

Además de la presión del lápiz, el usuario puede ajustar manualmente:
- **S (Size)**: Tamaño base del pincel (1-80px)
- **O (Opacity)**: Opacidad base (5-100%)
- **F (Flow)**: Tasa de flujo de partículas (10-100%)
- **D (Scatter)**: Cantidad de dispersión base (0-100%)

Estos sliders actúan como multiplicadores sobre los valores derivados de la presión.

---

### 3.5 Panel de Capas UI

#### Diseño

El panel de capas ocupa 260px en el lado derecho (estilo Photoshop). Contiene:

- **Header**: título "CAPAS" + contador
- **Lista scrollable**: capas en orden inverso (la superior primero, como Photoshop)
- **Footer**: slider de opacidad + botones de acción

#### Cada fila de capa muestra

| Elemento | Interacción |
|---|---|
| Miniatura (36×24px) | Click selecciona capa |
| Nombre | Doble click para renombrar |
| Botón visibilidad (ojo) | Click toggle visible/oculto |
| Botón bloqueo (candado) | Click toggle bloqueado/desbloqueado |

#### ¿Por qué no drag & drop para reordenar?

El drag & drop en vanilla JS sobre una lista de capas requiere manejo complejo de eventos (dragstart, dragover, drop, reordenamiento visual en tiempo real). Para la primera versión, se optó por botones Subir/Bajar que son más simples, predecibles, y funcionan igual en desktop y mobile. El drag & drop se puede añadir en una versión futura.

#### Operaciones de capa

| Acción | Botón | Atajo |
|---|---|---|
| Nueva capa | `[+]` | `Ctrl+N` |
| Duplicar capa | `Dup` | `Ctrl+J` |
| Eliminar capa | `Del` | `Ctrl+Delete` |
| Subir capa | `↑` | `Ctrl+Up` |
| Bajar capa | `↓` | `Ctrl+Down` |

---

### 3.6 Undo/Redo por Capa

Cada capa mantiene su propio historial independiente (máx 30 estados):

```
undoStack: [ImageData_t0, ImageData_t1, ...]  ← estados anteriores
redoStack: [ImageData_r0, ImageData_r1, ...]  ← estados rehechos
```

**¿Por qué undo por capa y no global?**

En un sistema de capas, el usuario trabaja en una capa a la vez. Si undo fuera global, deshacer un trazo en la capa 3 podría afectar cambios en la capa 1. El undo por capa aísla las operaciones y es el comportamiento estándar en software de ilustración (Photoshop, Krita, GIMP).

**¿Por qué guardar ImageData completo en vez de diff?**

Un diff (solo los píxeles cambiados) sería más eficiente en memoria, pero:
1. Más complejo de implementar (hay que trackear la bounding box del trazo)
2. ImageData completo es más robusto (no hay edge cases con diffs parciales)
3. 30 estados × ~8MB = ~240MB máximo, aceptable para navegadores modernos
4. El garbage collector libera memoria de estados viejos

---

### 3.7 Zoom y Pan

| Operación | Entrada |
|---|---|
| Zoom in | Rueda mouse arriba, tecla `+` |
| Zoom out | Rueda mouse abajo, tecla `-` |
| Zoom hacia cursor | Rueda mouse (el zoom se ancla en la posición del cursor) |
| Reset zoom | `Ctrl+0` |
| Ajustar canvas | `Ctrl+1` |
| Pan | Herramienta mano (H) o Space+arrastrar |
| Rango zoom | 0.15× a 64× |

El viewport se aplica mediante CSS `transform: scale(zoom) translate(panX, panY)` sobre el elemento `<canvas>`. Esto permite que el canvas mantenga su resolución interna de 1920×1080 mientras se escala visualmente.

---

### 3.8 Herramientas

| Herramienta | Tecla | Comportamiento |
|---|---|---|
| **Pincel** | B | Dibuja con el pincel y color activos |
| **Borrador** | E | Pinta blanco (#FFFFFF) en la capa activa |
| **Relleno** | G | Flood fill en la capa activa (tolerancia 20) |
| **Mano** | H | Pan/arrastre del lienzo |
| **Cuentagotas** | I | Muestrea color del composite visible |

**¿Por qué el borrador pinta blanco y no usa transparencia?**

Las capas tienen fondo blanco (no transparente). Borrar a transparente requeriría soporte de canal alpha en el compositing de capas, lo cual añade complejidad. Borrar a blanco es más simple y funciona para el 90% de los casos de uso. En una versión futura se puede añadir soporte de transparencia con fondo checkerboard.

---

### 3.9 Exportación y Formato de Archivo

#### Export PNG

Renderiza el composite de todas las capas visibles (aplanado) a 1920×1080 y descarga como PNG.

#### Formato `.fapd`

```json
{
  "v": 2,
  "type": "fapd",
  "layers": [
    {
      "name": "Fondo",
      "visible": true,
      "locked": false,
      "opacity": 100,
      "data": "data:image/png;base64,iVBORw0KG..."
    },
    ...
  ]
}
```

Cada capa se serializa como PNG base64. Ventajas:
- Sin pérdida de calidad (PNG es lossless)
- Compatible con cualquier visor de imágenes
- Fácil de inspeccionar/modificar manualmente
- El archivo se puede arrastrar y soltar sobre la ventana para abrirlo

---

## 4. Bugs Corregidos

### Bug #1: Canvas se volvía gris al dibujar

**Causa**: Los pinceles de acuarela usan `globalAlpha` muy bajo (0.02-0.06). Al terminar el trazo, `renderComposite()` no reseteaba `globalAlpha` a 1. Como resultado, `clearRect()` y `fillRect()` operaban con alpha ~0.04, limpiando/dibujando solo al 4%, dejando residuos grises.

**Solución**: Agregar `ctx.globalAlpha = 1; ctx.globalCompositeOperation = 'source-over';` al inicio de `renderComposite()`.

### Bug #2: Herramienta Mano (H) bloqueaba el dibujo

**Causa**: La variable `spaceDidPan` se activaba al hacer pan con Space+arrastrar, pero **nunca se reseteaba**. La función `startDraw()` verificaba `state.activeTool === 'hand' || spaceDidPan` para entrar en modo pan. Al quedar `spaceDidPan = true` permanentemente, cualquier intento de dibujar era interceptado como pan.

**Solución**: Resetear `spaceDidPan = false` y `preSpaceTool = null` en `setTool()` cada vez que se cambia explícitamente de herramienta.

### Bug #3: Pinceles originales desaparecidos

**Causa**: En la primera implementación solo se incluyeron 9 pinceles "clásicos" + 30 acuarela + 4 óleo = 43. Los 51 pinceles restantes del FAP original no se portaron.

**Solución**: Se agregaron las definiciones de los 51 pinceles faltantes al objeto `BRUSHES`, y se implementó la función `drawOriginalBrush()` que contiene el renderizado completo de todos los pinceles FAP originales (jitter, splat, water, charcoal, crayon, dots, hatch, neon, fur, drip, airbrush, sand, sponge, pastel, gravel, rake, chain, zigzag, wave, lace, oil, dryBrush, smudge, acrylic, gouache, sparkle, comet, graffiti, spiderweb, smoke, confetti, rain, bubbles, vines, scales, stitch, grid, embers, crackle, diamond, feather, wire, pebble, brick, scribble, drizzle, tribal, frost, marble, holographic).

---

## 5. Estructura del Proyecto

```
D:\FAP WEB\programa de dibujo\
├── index.html          (1761 líneas, single-file app)
├── README.md           (este archivo)
└── icons\              (118 archivos de íconos)
    ├── logo.png, LUPA.png
    ├── brushes\        (60 íconos PNG de pinceles)
    ├── UI icons PNG    (undo, redo, brush, eraser, etc.)
    └── UI icons SVG    (eye, lock, new_layer, delete, etc.)
```

---

## 6. Atajos de Teclado

| Tecla | Acción |
|---|---|
| `B` / `E` / `G` / `H` / `I` | Herramientas: Pincel, Borrador, Relleno, Mano, Cuentagotas |
| `[` / `]` | Tamaño pincel -1 / +1 |
| `Shift+[` / `]` | Opacidad pincel -5% / +5% |
| `1-9` | Pinceles rápidos (primeros 9 de la tira) |
| `Ctrl+Z` | Undo (capa activa) |
| `Ctrl+Shift+Z` | Redo (capa activa) |
| `Ctrl+N` | Nueva capa |
| `Ctrl+J` | Duplicar capa |
| `Ctrl+Delete` | Eliminar capa |
| `Ctrl+Up/Down` | Mover capa arriba/abajo |
| `Ctrl+S` | Guardar proyecto (.fapd) |
| `Ctrl+O` | Abrir proyecto (.fapd) |
| `Ctrl+Shift+P` | Exportar PNG |
| `Ctrl+0` | Reset zoom |
| `Ctrl+1` | Ajustar canvas a la ventana |
| `+` / `-` | Zoom in / out |
| `Space + drag` | Pan temporal |
| `#` | Toggle cuadrícula |
| `?` | Mostrar/ocultar panel de atajos |

---

## 7. Limitaciones Conocidas y Trabajo Futuro

1. **Sin transparencia real**: Las capas tienen fondo blanco, no soportan canal alpha. Implementar fondo checkerboard y soporte de transparencia real.
2. **Sin modos de fusión (blend modes)**: Las capas solo usan composición normal (alpha blending). Añadir multiply, screen, overlay, etc.
3. **Sin máscaras de capa**: No hay máscaras para ocultar parcialmente capas.
4. **Sin transformación de capas**: No se puede escalar, rotar o mover capas individualmente.
5. **Sin capas vectoriales**: Solo capas raster (píxeles).
6. **Undo no跨-capa**: Si cambias de capa, el historial de la capa anterior se mantiene pero no hay forma de deshacer el cambio de capa en sí.
7. **Sin selector de color avanzado**: Solo la paleta de 66 colores + cuentagotas. No hay rueda de color ni sliders HSL/RGB.
8. **Sin soporte de texto**: No hay herramienta de texto.
9. **Rendimiento de undo**: Guardar ImageData de 1920×1080 (8MB) en cada trazo puede ser pesado en dispositivos de baja memoria.

---

## Documentacion Tecnica

- [Informe Tecnico Free Illustration Power](informes_pdf/08_Free_Illustration_Power.pdf) — Documento completo de arquitectura, sistema de capas, motor de 94 pinceles, sensibilidad multi-parametro y especificaciones tecnicas.

---

## 8. Créditos

- **Autor**: Eduardo Fierro Duque, Santiago de Chile
- **Basado en**: FAP — Free Animation Power ([freeanimationpower.org](https://freeanimationpower.org))
- **Repositorio**: [github.com/freeanimationpower/Free-Illustration-Power](https://github.com/freeanimationpower/Free-Illustration-Power)
- **Año**: 2026

---

## 9. Changelog de Mejoras (Julio 2026)

### 9.1 Correcciones de Pinceles

| Bug | Descripcion | Solucion |
|---|---|---|
| Caligrafia rota | `drawOriginalBrush` no renderizaba elipses rotadas | Handler `br.slant` con elipses rotadas interpoladas |
| Brillo (Glow) roto | Perdia el doble trazo (nucleo brillante interior) | Handler `br.doubleStroke` con outer glow + inner bright core |
| Efectos Ac. invisibles | w28/w29/w30 usaban alpha extremadamente bajo (1-5%) | Ajuste de alpha base: 0.12→0.25, 0.10→0.22, 0.04→0.18 |
| Pinceles con dispersion | `Math.random()` diferente entre ctx y lbCtx → particulas saltaban al soltar | PRNG deterministico Mulberry32 con `withSameSeed()` |
| Punto inicial gigante | `startDraw` no escalaba por presion como `moveDraw` | `dotSize = lw * (0.2+pressure*0.8) * (pp.size||1)` |

### 9.2 Capas Transparentes y Borrador Real

| Cambio | Descripcion |
|---|---|
| Capas transparentes | `createLayerCanvas()` no rellena con blanco. `renderComposite()` provee fondo blanco visual |
| Fondo sin blanco solido | La capa "Fondo" ya no tiene relleno blanco opaco (causaba que al reordenar capas el Fondo tapara todo) |
| Borrador transparente | Usa `globalCompositeOperation = 'destination-out'` en vez de pintar blanco |
| Canvas state limpio | `syncLayerBuffer()` y `flushLayerBuffer()` resetean `globalAlpha`, `compositeOperation`, `shadowBlur`, `filter` antes de copiar |

### 9.3 UI y Experiencia de Usuario

| Cambio | Descripcion |
|---|---|
| Iconos de pinceles | 34 iconos nuevos para acuarela y oleo. Fallback a texto si no hay PNG |
| Categorias con fondo | Bloques de pinceles con fondos alternados (`bg-tertiary`/`bg-secondary`) |
| Scrollbars ampliados | Brush strip, color strip y layers panel: 5-6px → 14px para tablet |
| Tiras mas espaciosas | Altura 30→44px, colores 22→28px, gaps ampliados |
| Botones de capa | Rediseno en 4 bloques: Opacidad, Nueva+Imagen, Duplicar+Eliminar, Subir+Bajar |
| Nombres completos | Botones con nombres legibles (Duplicar, Eliminar, Subir, Bajar, Imagen) |

### 9.4 Importacion y Exportacion

| Funcionalidad | Descripcion |
|---|---|
| Importar Imagen | Boton "Imagen" + `Ctrl+I`. PNG/JPG/WebP como nueva capa, escalada y centrada |
| Export PNG | `Ctrl+Shift+P` |
| Export JPG | `Ctrl+Shift+J` (calidad 92%) |
| Export EPS | `Ctrl+Shift+E` (PostScript nivel 2 con JPEG embebido via DCTDecode) |
| Formato `.fapd` | Guarda/Abre proyecto editable con todas las capas y opacidad |
| Archivos export | `FIP_01.png`, `FIP_02.jpg`, `FIP_03.eps` |

### 9.5 Sistema de Debug

| Componente | Descripcion |
|---|---|
| Panel DEBUG | `Ctrl+Shift+D` activa panel flotante + logs en consola |
| Trazado de pinceles | Muestra ruta de renderizado (water/oil → original → default) |
| Verificacion de canvas | Detecta `globalAlpha`/`compositeOperation`/`shadowBlur` sucios |
| Actualizacion optimizada | Solo 1 DOM update por trazo (al final), no por segmento |

### 9.6 Herramienta Mover (eliminada)

Se implemento y elimino una herramienta de desplazamiento de objetos (tecla V) con:
- Seleccion por rectangulo (marquee)
- Varita magica (flood-fill BFS)
- Escaneo lineal de capa completa

Se elimino por complejizar el sistema y causar inestabilidad. Se retomara en version futura.

### 9.7 Renombrado

| Antes | Ahora |
|---|---|
| FAP Draw | **Free Illustration Power** |
| `FAP_DRAW_01.png` | `FIP_01.png` |
