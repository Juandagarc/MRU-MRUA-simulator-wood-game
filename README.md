# Simulador Cinemática — Juego Didáctico de Madera

Réplica web del juego didáctico de madera para enseñanza de cinemática (MRU / MRUA). Los estudiantes arrastran piezas de madera grabadas con curvas cinemáticas sobre una gráfica de Posición vs. Tiempo, formando funciones a trozos con continuidad C⁰ validada en tiempo real.

**Demo**: abre `index.html` directamente en el navegador — sin servidor, sin dependencias.

---

## Cómo jugar

| Acción | Resultado |
|---|---|
| Arrastrar pieza del inventario → gráfica | Colocar pieza en el tablero |
| Arrastrar pieza del tablero | Reubicar o devolver al inventario |
| Soltar fuera de la gráfica | Devuelve la pieza al inventario |
| Doble clic fuera de la gráfica | Reinicia todo el tablero |

Las piezas se encajan (snap) a la cuadrícula `t = múltiplo de 10`, `X = múltiplo de 50`. Si el extremo de una pieza ya colocada está cerca, el imán de continuidad actúa: la nueva pieza se alinea automáticamente para formar una función continua.

Un punto verde en la unión indica continuidad C⁰. Un punto rojo indica discontinuidad.

---

## Arquitectura

El juego corre íntegramente en un único archivo `index.html` (HTML + JS vanilla). La arquitectura sigue el patrón MVC con cuatro clases independientes:

```
┌──────────────┐  estado inmutable  ┌──────────────────┐
│ StateManager ├───────────────────▶│  CanvasRenderer  │
└──────┬───────┘                    └──────────────────┘
       ▲                                      ▲
       │ mutaciones                           │ pinta @ 60fps
┌──────┴───────┐                    ┌─────────┴──────────┐
│ InputHandler │                    │     MathEngine     │
└──────────────┘                    └────────────────────┘
```

| Clase | Responsabilidad |
|---|---|
| `MathEngine` | Cálculo puro: `delta(tpl, dt)`, `samples()`, `continuity()`. Sin estado. |
| `StateManager` | Fuente de verdad: `inventory[]`, `board[]`, `drag`. Solo muta estado, nunca dibuja. |
| `CanvasRenderer` | Dibuja todo sobre `<canvas>`. Nunca muta estado. |
| `InputHandler` | Traduce eventos de mouse/touch a llamadas de `StateManager`. |
| `Game` | Orquesta el loop RAF con dirty-flag. |

---

## Prácticas clave — síguelas al crear un juego nuevo

### 1. Escala física única

Existe una sola fuente de verdad para la conversión píxel ↔ unidad:

```js
SCALE: { PX_PER_SEC: 8, PX_PER_METER: 1.2 }
```

Esta escala se usa **tanto** para dimensionar los mosaicos del inventario **como** para renderizar las curvas en la gráfica. Si usas escalas distintas, las figuras del inventario y del tablero tendrán formas diferentes — el error más común al extender este proyecto.

### 2. `posSpan` define la altura física del mosaico

Cada plantilla declara `posSpan`: el desplazamiento neto en metros que produce la curva. El renderer lo convierte a píxeles para fijar la altura del bloque:

```js
innerH = Math.abs(tpl.posSpan) * PX_PER_METER
```

Para que las piezas lineales sean visualmente iguales a las cuadráticas de mayor span, usa una pendiente que produzca el mismo `posSpan`:

```js
// X = 2t² durante 10s → posSpan 200m
// X = 20t durante 10s → posSpan 200m  ✓ misma altura
L_POS: { shape:"linear", a:20, dur:10, posSpan:200 }
```

### 3. El ancla siempre es el punto de inicio de la curva

`tileAnchorOffset(tpl)` devuelve `{dx, dy}` desde la esquina superior izquierda del mosaico hasta el punto de inicio de la curva:

- Curva **sube** (`posSpan > 0`): el ancla está en la esquina inferior izquierda del área interior.
- Curva **baja** (`posSpan < 0`): el ancla está en la esquina superior izquierda.
- Curva **horizontal** (`posSpan = 0`): el ancla está centrada verticalmente.

Usa este offset de forma consistente al calcular `boardTileRect`, la posición del ghost de arrastre y la preview de snap. Inconsistencias aquí causan que la curva no coincida con la posición matemática al soltar la pieza.

### 4. `INVENTORY_SPEC` describe el layout exacto del juego físico

El inventario no es una cuadrícula uniforme. Cada entrada define columna (`c`), mitad (`h`) y plantilla (`t`):

```js
{ c:0, h:"top", t:"Q_NEG_2" }   // columna 0, mitad superior
{ c:3, h:"top-up", t:"Q_POS_1" } // columna 3, celda superior de la mitad superior
```

`_buildInventorySlots()` traduce cada `h` a coordenadas Y concretas. Si agregas un nuevo tipo de pieza con altura diferente, agrega un nuevo caso en ese switch y recalcula el centrado vertical de su grupo.

### 5. El pre-render del fondo es obligatorio para el rendimiento

La textura de madera (veta, nudos, biselado, tornillos) se dibuja una sola vez en un canvas offscreen `_buildBackgroundLayer()` y se copia con `drawImage` en cada frame. No redibujes elementos estáticos en el loop principal.

### 6. Dirty-flag en el loop

El loop RAF solo llama a `render()` cuando `state.dirty === true`. Cada mutación de estado llama a `state.markDirty()`. Esto mantiene la CPU libre cuando el usuario no interactúa.

```js
_loop() {
  if (this.state.dirty) {
    this.renderer.render();
    this.state.dirty = false;
  }
  requestAnimationFrame(this._loop);
}
```

### 7. Separación dura: el renderer no muta estado

`CanvasRenderer` recibe el estado como referencia de solo lectura. Nunca llama a métodos que muten `StateManager`. `InputHandler` es el único que muta. Esta separación permite portar el renderer a React/Canvas sin reescribir la lógica de juego.

### 8. Validación de snap en `_computeSnap`

Al soltar una pieza, la posición es válida (`snap.ok = true`) solo si:
1. La pieza entra completamente dentro del rango `[T_MIN, T_MAX]` y `[X_MIN, X_MAX]`.
2. No se superpone en tiempo con ninguna pieza ya colocada.

Si amplías los rangos de la gráfica o los tipos de pieza, revisa estas condiciones.

---

## Agregar un nuevo tipo de pieza

1. Agrega la plantilla en `TEMPLATES`:
   ```js
   MI_PIEZA: { id:"MI_PIEZA", shape:"quadratic", a:3, dur:10, label:"X=3t²", posSpan:300 }
   ```

2. Agrega una o más entradas en `INVENTORY_SPEC`:
   ```js
   { c:2, h:"top", t:"MI_PIEZA" }
   ```

3. Si la nueva pieza necesita un subtipo de layout que no existe, agrega un `case` en `_buildInventorySlots()`.

4. `MathEngine.delta` ya soporta `shape:"quadratic"`, `"linear"` y `"horizontal"`. Para una nueva forma (p. ej. sinusoidal), agrega el caso allí y en `_drawCurveInTile`.

---

## Adaptar a otro juego físico

Si tienes un juego didáctico diferente (p. ej. velocidad vs. tiempo, aceleración vs. tiempo):

1. Cambia `CFG.GRAPH` para reflejar los ejes del nuevo juego.
2. Redefine `tToPx / xToPy / pxToT / pyToX` según las nuevas unidades.
3. Ajusta `MathEngine.delta` para la ecuación cinemática relevante.
4. Mantén la misma escala física en inventario y tablero.
5. Actualiza `MathEngine.continuity` si la condición de continuidad cambia (p. ej. C¹ en lugar de C⁰).

---

## Estructura del archivo

```
index.html
  <style>          — layout CSS mínimo (grid centrado, canvas responsivo)
  <canvas #game>   — superficie de dibujo 1280×800
  <script>
    CFG            — toda la configuración (sin números mágicos)
    TEMPLATES      — plantillas de piezas
    INVENTORY_SPEC — layout del inventario
    Utils          — roundRectPath, pointInRect, makeSeededRng
    MathEngine     — lógica matemática pura
    StateManager   — estado del juego
    CanvasRenderer — rendering
    InputHandler   — eventos de entrada
    Game           — loop principal
```

---

## Tecnología

- HTML5 Canvas API — sin librerías externas
- JavaScript ES6+ (clases, `Object.freeze`, `requestAnimationFrame`)
- Compatible con Chrome, Firefox, Safari, Edge
- Funciona en móvil (touch events)
- Diseñado para portar a Next.js: cada clase es un módulo independiente
