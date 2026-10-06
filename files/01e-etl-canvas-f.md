# 01e-etl-canvas-f.md

File in questo blocco:

- `src/etl-canvas/__tests__/view.test.ts`
- `src/etl-canvas/actions.ts`
- `src/etl-canvas/canvas.css`
- `src/etl-canvas/contrast.ts`
- `src/etl-canvas/drop.ts`
- `src/etl-canvas/engine.ts`
- `src/etl-canvas/flow.ts`
- `src/etl-canvas/icons.tsx`
- `src/etl-canvas/index.ts`
- `src/etl-canvas/inspector/ActionMenu.tsx`
- `src/etl-canvas/inspector/BlockedNotice.tsx`

---

### `src/etl-canvas/__tests__/view.test.ts`

194 righe

```ts
import { describe, expect, it } from "vitest";
import { CARD, LABEL_H } from "../../etl-layout";
import { ZOOM_MAX, ZOOM_MIN } from "../../etl-store";
import { fit, zoomAtPoint, zoomIn, zoomOut, zoomReset } from "../actions";
import { prototypeScene } from "../seed";
import { bounds, fitView, minimapFrame, toWorld, viewFromMinimap, wheelPan, zoomAt } from "../view";
import { SIZE, storeWith } from "./helpers";

function allInside(store: ReturnType<typeof storeWith>, size = SIZE): void {
  const { view, graph } = store.getState();
  for (const c of Object.values(graph.cards)) {
    const x1 = c.x * view.zoom + view.x;
    const y1 = c.y * view.zoom + view.y;
    const x2 = (c.x + CARD) * view.zoom + view.x;
    const y2 = (c.y + CARD + LABEL_H) * view.zoom + view.y;
    expect(x1, c.id).toBeGreaterThanOrEqual(0);
    expect(y1, c.id).toBeGreaterThanOrEqual(0);
    expect(x2, c.id).toBeLessThanOrEqual(size.w);
    expect(y2, c.id).toBeLessThanOrEqual(size.h);
  }
}

describe("Adatta", () => {
  it("dopo la chiamata tutti i nodi rientrano nell'area visibile", () => {
    const store = storeWith();
    store.dispatch({ type: "setView", payload: { x: -900, y: 400, zoom: 2 } });
    fit(store, SIZE);
    allInside(store);
  });

  it("vale per finestre piccole e grandi", () => {
    for (const size of [
      { w: 320, h: 240 },
      { w: 1440, h: 900 },
      { w: 600, h: 1200 },
    ]) {
      const store = storeWith();
      fit(store, size);
      allInside(store, size);
    }
  });

  it("vale anche per una scena sparsa su tutto il mondo", () => {
    const size = { w: 1200, h: 800 };
    const store = storeWith();
    const s = prototypeScene();
    const far = { ...s.graph.cards["op-export"]!, x: 2400, y: 1400 };
    store.replaceState({
      ...s,
      graph: { ...s.graph, cards: { ...s.graph.cards, "op-export": far } },
    });
    fit(store, size);
    allInside(store, size);
  });

  it("oltre il limite di zoom (0,35) Adatta si ferma al limite, come nel prototipo", () => {
    const store = storeWith();
    const s = prototypeScene();
    const far = { ...s.graph.cards["op-export"]!, x: 2400, y: 1400 };
    store.replaceState({
      ...s,
      graph: { ...s.graph, cards: { ...s.graph.cards, "op-export": far } },
    });
    fit(store, { w: 320, h: 240 });
    expect(store.getState().view.zoom).toBe(ZOOM_MIN);
  });

  it("con il margine del prototipo (48) e zoom al più 1,25", () => {
    const one = [{ x: 100, y: 100 }];
    const v = fitView(one, { w: 2000, h: 2000 });
    expect(v.zoom).toBe(1.25);
    const b = bounds(one)!;
    // centrato
    expect(v.x + b.x1 * v.zoom).toBeCloseTo((2000 - (b.x2 - b.x1) * v.zoom) / 2, 6);
  });

  it("canvas vuoto: vista di partenza", () => {
    expect(fitView([], SIZE)).toEqual({ x: 0, y: 0, zoom: 1 });
  });
});

describe("zoom", () => {
  it("limiti del prototipo (0,35 – 2)", () => {
    const store = storeWith();
    for (let i = 0; i < 40; i++) zoomIn(store, SIZE);
    expect(store.getState().view.zoom).toBe(ZOOM_MAX);
    for (let i = 0; i < 80; i++) zoomOut(store, SIZE);
    expect(store.getState().view.zoom).toBe(ZOOM_MIN);
    zoomReset(store, SIZE);
    expect(store.getState().view.zoom).toBe(1);
  });

  it("attorno al puntatore il punto del mondo sotto il puntatore non si muove", () => {
    const store = storeWith();
    store.dispatch({ type: "setView", payload: { x: 30, y: 50, zoom: 0.8 } });
    const before = toWorld(store.getState().view, 400, 300);
    zoomAtPoint(store, 400, 300, 1.7);
    const after = toWorld(store.getState().view, 400, 300);
    expect(after.x).toBeCloseTo(before.x, 6);
    expect(after.y).toBeCloseTo(before.y, 6);
    expect(store.getState().view.zoom).toBeCloseTo(1.7, 6);
  });

  it("zoomAt è puro e rispetta i limiti", () => {
    const v = { x: 0, y: 0, zoom: 1 };
    expect(zoomAt(v, 10, 10, 99).zoom).toBe(ZOOM_MAX);
    expect(v).toEqual({ x: 0, y: 0, zoom: 1 });
  });

  it("la vista passa da etl-store: la modifica notifica gli ascoltatori", () => {
    const store = storeWith();
    let n = 0;
    store.subscribe(() => n++);
    zoomIn(store, SIZE);
    expect(n).toBe(1);
    // e non entra nel registro né nella cronologia
    expect(store.getLog().some((e) => e.type === "setView")).toBe(false);
    expect(store.canUndo()).toBe(false);
  });
});

describe("minimappa", () => {
  it("contiene tutti i nodi e la porzione visibile nel riquadro 168 × 104", () => {
    const store = storeWith();
    const cards = Object.values(store.getState().graph.cards);
    const frame = minimapFrame(cards, store.getState().view, SIZE);
    for (const c of cards) {
      const l = frame.ox + (c.x - frame.x1) * frame.k;
      const t = frame.oy + (c.y - frame.y1) * frame.k;
      expect(l).toBeGreaterThanOrEqual(0);
      expect(t).toBeGreaterThanOrEqual(0);
      expect(l + CARD * frame.k).toBeLessThanOrEqual(168 + 1e-9);
      expect(t + CARD * frame.k).toBeLessThanOrEqual(104 + 1e-9);
    }
  });

  it("un clic sulla minimappa porta quel punto al centro dell'area", () => {
    const store = storeWith();
    const cards = Object.values(store.getState().graph.cards);
    const view = store.getState().view;
    const frame = minimapFrame(cards, view, SIZE);
    const next = viewFromMinimap(frame, view, SIZE, 84, 52);
    const center = toWorld(next, SIZE.w / 2, SIZE.h / 2);
    expect(center.x).toBeCloseTo(frame.x1 + (84 - frame.ox) / frame.k, 6);
    expect(center.y).toBeCloseTo(frame.y1 + (52 - frame.oy) / frame.k, 6);
  });
});

describe("«Adatta» con i margini dei widget e rotella", () => {
  it("con i margini l'insieme sta nell'area meno i widget", () => {
    const cards = [
      { x: 0, y: 0 },
      { x: 600, y: 300 },
    ];
    const size = { w: 1000, h: 600 };
    const insets = { top: 0, right: 0, bottom: 120, left: 200 };
    const v = fitView(cards, size, insets);
    const b = bounds(cards)!;
    expect(b.x1 * v.zoom + v.x).toBeGreaterThanOrEqual(insets.left - 0.001);
    expect(b.x2 * v.zoom + v.x).toBeLessThanOrEqual(size.w - insets.right + 0.001);
    expect(b.y1 * v.zoom + v.y).toBeGreaterThanOrEqual(insets.top - 0.001);
    expect(b.y2 * v.zoom + v.y).toBeLessThanOrEqual(size.h - insets.bottom + 0.001);
    // senza margini il risultato è quello di sempre
    expect(fitView(cards, size)).toEqual(
      fitView(cards, size, { top: 0, right: 0, bottom: 0, left: 0 }),
    );
  });

  it("rotella semplice: pan in verticale (e in orizzontale col trackpad); Maiusc: orizzontale; zoom resta a Cmd/Ctrl", () => {
    const view = { x: 10, y: 20, zoom: 1.5 };
    expect(wheelPan(view, { deltaX: 0, deltaY: 30, shiftKey: false })).toEqual({
      x: 10,
      y: -10,
      zoom: 1.5,
    });
    expect(wheelPan(view, { deltaX: 8, deltaY: 30, shiftKey: false })).toEqual({
      x: 2,
      y: -10,
      zoom: 1.5,
    });
    expect(wheelPan(view, { deltaX: 0, deltaY: 30, shiftKey: true })).toEqual({
      x: -20,
      y: 20,
      zoom: 1.5,
    });
    // Maiusc con un trackpad che già invia deltaX: si usa deltaX
    expect(wheelPan(view, { deltaX: 12, deltaY: 5, shiftKey: true })).toEqual({
      x: -2,
      y: 15,
      zoom: 1.5,
    });
  });
});
```

### `src/etl-canvas/actions.ts`

42 righe

```ts
/** Azioni sulla vista, applicate attraverso etl-store (`setView`). */
import type { Size } from "../etl-layout";
import type { EtlStore } from "../etl-store";
import { fitView, zoomAt, zoomCentered, ZOOM_STEP } from "./view";
import type { Insets } from "./view";

/** "Adatta": inquadra tutti i nodi, evitando lo spazio dei widget in sovrimpressione. */
export function fit(store: EtlStore, size: Size, insets?: Insets): void {
  store.dispatch({
    type: "setView",
    payload: fitView(Object.values(store.getState().graph.cards), size, insets),
  });
}

export function zoomBy(store: EtlStore, size: Size, factor: number): void {
  const view = store.getState().view;
  store.dispatch({ type: "setView", payload: zoomCentered(view, size, view.zoom * factor) });
}

export function zoomIn(store: EtlStore, size: Size): void {
  zoomBy(store, size, ZOOM_STEP);
}

export function zoomOut(store: EtlStore, size: Size): void {
  zoomBy(store, size, 1 / ZOOM_STEP);
}

export function zoomReset(store: EtlStore, size: Size): void {
  store.dispatch({
    type: "setView",
    payload: zoomCentered(store.getState().view, size, 1),
  });
}

/** Zoom attorno al puntatore (`px`, `py` relativi all'area). */
export function zoomAtPoint(store: EtlStore, px: number, py: number, zoom: number): void {
  store.dispatch({
    type: "setView",
    payload: zoomAt(store.getState().view, px, py, zoom),
  });
}
```

### `src/etl-canvas/canvas.css`

658 righe

```css
/*
 * Aspetto del canvas ETL (Fase 4a): nodi, cavi, controlli, minimappa.
 * Misure e classi del prototipo (docs/prototype/isa-fusion-prototype.html,
 * righe indicate); colori solo dai token di tokens.css. Tutte le regole
 * sono limitate a `.etl-canvas`. Il carattere (Manrope) è quello di tutta l'app,
 * caricato da src/styles.css.
 */
@import "./tokens.css";

.etl-canvas {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 520px;
  box-sizing: border-box;
  padding: 0;
  background: var(--ec-bg);
  color: var(--ec-ink);
  font-family: var(--ec-font);
  border-radius: var(--ec-r-stage);
}
.etl-canvas *,
.etl-canvas *::before,
.etl-canvas *::after {
  box-sizing: border-box;
  font-family: inherit;
}

/* riga 621-624 */
.etl-canvas .ec-stage {
  position: absolute;
  inset: 0;
  user-select: none;
  touch-action: none;
  border-radius: var(--ec-r-stage);
  background: var(--ec-stage);
  overflow: hidden;
}
.etl-canvas .ec-stage.ec-pannable {
  cursor: grab;
}
.etl-canvas .ec-stage.ec-panning {
  cursor: grabbing;
}

/* righe 3-4 di .world e .links (righe 205-206) */
.etl-canvas .ec-world {
  position: absolute;
  left: 0;
  top: 0;
  transform-origin: 0 0;
}
.etl-canvas .ec-links {
  position: absolute;
  inset: 0;
  overflow: visible;
  pointer-events: none;
}

/* nodo — riga 626 */
.etl-canvas .ec-card {
  position: absolute;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  width: 88px;
  z-index: 2;
}
.etl-canvas .ec-icon-wrap {
  position: relative;
  width: 88px;
  height: 88px;
  border-radius: var(--ec-r-op);
  background: var(--ec-node-op);
  color: var(--ec-node-op-ink);
  box-shadow: var(--ec-node-op-outline);
  display: grid;
  place-content: center;
  justify-items: center;
  align-items: center;
  gap: 6px;
  padding: 7px;
}
.etl-canvas .ec-dataset .ec-icon-wrap {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
  border-radius: var(--ec-r-fill);
  box-shadow: none;
}
.etl-canvas .ec-output .ec-icon-wrap {
  opacity: var(--ec-output-opacity);
}
.etl-canvas .ec-selected .ec-icon-wrap {
  box-shadow: var(--ec-select-ring);
}
.etl-canvas .ec-icon-wrap svg {
  display: block;
  flex-shrink: 0;
}
.etl-canvas .ec-count-1 {
  grid-template-columns: repeat(1, auto);
}
.etl-canvas .ec-count-2 {
  grid-template-columns: repeat(2, auto);
}
.etl-canvas .ec-count-3,
.etl-canvas .ec-count-6,
.etl-canvas .ec-count-many {
  grid-template-columns: repeat(3, auto);
}
.etl-canvas .ec-count-1 svg {
  width: 26px;
  height: 26px;
}
.etl-canvas .ec-count-2 svg {
  width: 20px;
  height: 20px;
}
.etl-canvas .ec-count-3 svg {
  width: 17px;
  height: 17px;
}
.etl-canvas .ec-count-6 svg {
  width: 15px;
  height: 15px;
}
.etl-canvas .ec-count-many svg {
  width: 12px;
  height: 12px;
}

/* output parziale: una fetta per tabella attesa — righe 656-669 */
.etl-canvas .ec-dataset .ec-icon-wrap.ec-split {
  padding: 0;
  display: flex;
  overflow: hidden;
  background: var(--ec-split-bg);
  gap: 0;
}
.etl-canvas .ec-split .ec-slice {
  flex: 1 1 0;
  min-width: 0;
  height: 100%;
  padding: 2px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.etl-canvas .ec-split .ec-slice.ec-slice-full {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
}
.etl-canvas .ec-split .ec-slice.ec-slice-empty {
  background: var(--ec-split-empty);
  color: var(--ec-split-empty-ink);
}
.etl-canvas .ec-split .ec-slice + .ec-slice {
  border-left: 1.5px dashed var(--ec-split-line);
}
.etl-canvas .ec-split .ec-slice svg {
  width: 100%;
  height: auto;
  max-width: 22px;
  max-height: 100%;
}
.etl-canvas .ec-split .ec-slice.ec-slice-empty svg {
  opacity: 0.85;
}

/* etichetta — righe 680-683 */
.etl-canvas .ec-label {
  font-size: 10.5px;
  font-weight: 700;
  text-align: center;
  line-height: 1.25;
  color: var(--ec-ink);
  max-width: 96px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  padding: 0 2px;
}
.etl-canvas .ec-dataset .ec-label {
  color: var(--ec-accent-text);
}
.etl-canvas .ec-partial .ec-label {
  color: var(--ec-muted);
  font-style: italic;
}

/* indicatore ambra — righe 183-187 */
.etl-canvas .ec-state-dot {
  position: absolute;
  right: -4px;
  bottom: -4px;
  width: 13px;
  height: 13px;
  border-radius: var(--ec-r-pill);
  background: var(--ec-warn);
  border: 2.5px solid var(--ec-warn-ring);
  z-index: 4;
}

/* cavi — righe 1404-1408 */
.etl-canvas .ec-link {
  fill: none;
  stroke: var(--ec-link);
  stroke-width: 2.1;
  stroke-linecap: round;
  stroke-linejoin: round;
}
.etl-canvas .ec-link-dot-ds {
  fill: var(--ec-link-dot-fill);
}
.etl-canvas .ec-link-dot-op {
  fill: var(--ec-link-dot-op);
}

/* flusso nei cavi (riga 1517: fill rgba(108,99,255,0.6)) e tracciato uscente di una dissolvenza */
.etl-canvas .ec-flow {
  fill: var(--ec-flow);
  pointer-events: none;
}
.etl-canvas .ec-link-ghost {
  fill: none;
  stroke: var(--ec-link);
  stroke-width: 2.1;
  stroke-linecap: round;
  stroke-linejoin: round;
  pointer-events: none;
}

/* controlli di zoom — righe 138-150 */
.etl-canvas .ec-zoom {
  position: absolute;
  /* posizione e misura: inline, da panels/overlayLayout.ts */
  box-sizing: border-box;
  z-index: 15;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 2px;
  padding: 4px;
  border-radius: var(--ec-r-pill);
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
}
.etl-canvas .ec-zoom button {
  all: unset;
  cursor: pointer;
  min-width: 28px;
  height: 28px;
  padding: 0 8px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--ec-r-pill);
  font-family: var(--ec-font);
  font-size: 12.5px;
  font-weight: 700;
  color: var(--ec-ink);
}
.etl-canvas .ec-zoom button:hover {
  background: var(--ec-accent-soft);
  color: var(--ec-accent-text);
}
.etl-canvas .ec-zoom button:focus-visible {
  outline: 2px solid var(--ec-accent-text);
  outline-offset: 1px;
}
.etl-canvas .ec-zoom .ec-fit {
  color: var(--ec-accent-text);
}

/* minimappa — righe 152-161 */
.etl-canvas .ec-minimap {
  position: absolute;
  /* posizione e misura: inline, da panels/overlayLayout.ts */
  box-sizing: border-box;
  z-index: 15;
  border-radius: var(--ec-r-minimap);
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  overflow: hidden;
  cursor: pointer;
}
.etl-canvas .ec-mm-node {
  position: absolute;
  border-radius: var(--ec-r-xs);
  background: var(--ec-mm-node);
}
.etl-canvas .ec-mm-node.ec-ds {
  background: var(--ec-mm-node-ds);
}
.etl-canvas .ec-mm-view {
  position: absolute;
  border: 1.5px solid var(--ec-mm-view-line);
  border-radius: var(--ec-r-sm);
  background: var(--ec-mm-view-bg);
  pointer-events: none;
}

/* stato vuoto */
.etl-canvas .ec-empty {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  text-align: center;
  pointer-events: none;
  z-index: 1;
}
.etl-canvas .ec-empty-title {
  font-size: 15px;
  font-weight: 800;
  color: var(--ec-ink);
}
.etl-canvas .ec-empty-text {
  font-size: 12.5px;
  font-weight: 600;
  color: var(--ec-empty-ink);
}

/* ── Gesti (Fase 5): a riposo non cambia nulla, tutto compare solo durante il gesto ── */

/* nodo in movimento — righe 627, 639 */
.etl-canvas .ec-card.ec-dragging {
  cursor: grabbing;
  z-index: 20;
}
.etl-canvas .ec-card.ec-dragging .ec-icon-wrap {
  box-shadow: var(--ec-drag-shadow);
  transform: scale(1.05);
}

/* esiti del rilascio: ciascuno ha colore e stile di contorno propri (righe 640-642 per fusione e collegamento) */
.etl-canvas .ec-card.ec-drop-merge .ec-icon-wrap {
  box-shadow: var(--ec-drop-merge-ring);
  transform: scale(1.09);
}
.etl-canvas .ec-card.ec-drop-merge .ec-icon-wrap svg {
  transform: scale(0.88);
}
.etl-canvas .ec-card.ec-drop-link .ec-icon-wrap {
  box-shadow: var(--ec-drop-link-ring);
  transform: scale(1.04);
}
.etl-canvas .ec-card.ec-drop-link-reverse .ec-icon-wrap {
  outline: 3px dashed var(--ec-drop-link-reverse);
  outline-offset: 3px;
  transform: scale(1.04);
}
.etl-canvas .ec-card.ec-drop-displace .ec-icon-wrap {
  outline: 3px dotted var(--ec-drop-displace);
  outline-offset: 3px;
}
.etl-canvas .ec-card.ec-drop-reject .ec-icon-wrap {
  outline: 3px solid var(--ec-drop-reject);
  outline-offset: 3px;
}

/* nodi che sparirebbero se si confermasse l'eliminazione */
.etl-canvas .ec-card.ec-doomed .ec-icon-wrap {
  outline: 3px dashed var(--ec-doomed);
  outline-offset: 3px;
  opacity: 0.55;
}

/* cavo in cui si inserirebbe la lavorazione (riga 1403, `hot`) */
.etl-canvas .ec-link.ec-link-hot {
  stroke: var(--ec-link-insert);
  stroke-width: 3.4;
}

/* porte — righe 168-181: compaiono al passaggio del puntatore */
.etl-canvas .ec-port {
  position: absolute;
  width: 11px;
  height: 11px;
  border-radius: var(--ec-r-pill);
  box-sizing: border-box;
  background: var(--ec-port-fill);
  border: 2px solid var(--ec-port-line);
  z-index: 5;
  cursor: crosshair;
  opacity: 0;
  transform: scale(0.5);
  transition:
    opacity 0.14s,
    transform 0.14s cubic-bezier(0.34, 1.56, 0.64, 1);
}
.etl-canvas .ec-card:hover .ec-port {
  opacity: 1;
  transform: scale(1);
}
.etl-canvas .ec-card.ec-dragging .ec-port {
  opacity: 0;
}
.etl-canvas .ec-port:hover {
  background: var(--ec-port-line);
}
.etl-canvas .ec-port-t {
  left: calc(44px - 5.5px);
  top: -5.5px;
}
.etl-canvas .ec-port-b {
  left: calc(44px - 5.5px);
  top: calc(88px - 5.5px);
}
.etl-canvas .ec-port-l {
  top: calc(44px - 5.5px);
  left: -5.5px;
}
.etl-canvas .ec-port-r {
  top: calc(44px - 5.5px);
  left: calc(88px - 5.5px);
}

/* cavo provvisorio tirato da una porta — riga 3995 */
.etl-canvas .ec-temp-link {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  overflow: visible;
  pointer-events: none;
  z-index: 19;
}
.etl-canvas .ec-temp-link path {
  fill: none;
  stroke: var(--ec-temp-link-muted);
  stroke-width: 2.2;
  stroke-dasharray: 6 5;
  stroke-linecap: round;
}
.etl-canvas .ec-temp-link circle {
  fill: var(--ec-temp-link-muted);
}
.etl-canvas .ec-temp-link.ec-valid path {
  stroke: var(--ec-temp-link);
}
.etl-canvas .ec-temp-link.ec-valid circle {
  fill: var(--ec-temp-link);
}

/* riquadro di selezione — riga 163 */
.etl-canvas .ec-marquee {
  position: absolute;
  z-index: 14;
  pointer-events: none;
  border: 1.5px dashed var(--ec-marquee-line);
  background: var(--ec-marquee-fill);
  border-radius: var(--ec-r-sm);
}

/* suggerimento durante un gesto (prototipo: `hint`) */
.etl-canvas .ec-hint {
  position: absolute;
  /* posizione e misura: inline, da panels/overlayLayout.ts */
  box-sizing: border-box;
  z-index: 16;
  padding: 0 14px;
  line-height: 28px;
  text-align: center;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  border-radius: var(--ec-r-pill);
  background: var(--ec-surface-strong);
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  color: var(--ec-ink);
  font-size: 12px;
  font-weight: 600;
  pointer-events: none;
}

/* conferma di eliminazione — righe 112-132 */
.etl-canvas .ec-confirm {
  position: absolute;
  z-index: 30;
  width: 246px;
  padding: 22px;
  border-radius: var(--ec-r-fill);
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-overlay-shadow);
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.etl-canvas .ec-confirm-title {
  font-weight: 800;
  font-size: 15px;
  color: var(--ec-ink);
}
.etl-canvas .ec-confirm-text {
  font-size: 11.5px;
  font-weight: 600;
  line-height: 1.5;
  color: var(--ec-empty-ink);
  margin-top: 3px;
}
.etl-canvas .ec-confirm-actions {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
}
.etl-canvas .ec-confirm-actions button {
  all: unset;
  cursor: pointer;
  font-family: inherit;
  font-size: 12.5px;
  font-weight: 700;
  padding: 9px 15px;
  border-radius: var(--ec-r-pill);
}
.etl-canvas .ec-confirm-actions button:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 2px;
}
.etl-canvas .ec-confirm-actions .ec-confirm-cancel {
  color: var(--ec-empty-ink);
  background: var(--ec-accent-soft);
}
.etl-canvas .ec-confirm-actions .ec-confirm-cancel:hover {
  color: var(--ec-ink);
}
.etl-canvas .ec-confirm-actions .ec-confirm-ok {
  background: var(--ec-danger);
  color: var(--ec-text-on-danger);
}

/* minimappa ridotta a un pulsante, quando l'area è troppo piccola per ospitarla (Fase 6a.2) */
.etl-canvas .ec-minimap-toggle {
  all: unset;
  cursor: pointer;
  position: absolute;
  box-sizing: border-box;
  z-index: 15;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--ec-r-minimap);
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  color: var(--ec-accent-text);
}
.etl-canvas .ec-minimap-toggle svg {
  width: 20px;
  height: 20px;
}
.etl-canvas .ec-minimap-toggle:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 2px;
}
.etl-canvas .ec-mm-close {
  all: unset;
  cursor: pointer;
  position: absolute;
  top: 2px;
  right: 4px;
  z-index: 1;
  font-size: 14px;
  line-height: 1;
  padding: 2px 4px;
  color: var(--ec-empty-ink);
}
.etl-canvas .ec-mm-close:focus-visible {
  outline: 2px solid var(--ec-select);
}

/* pulsanti sul nodo (Fase 6b.1): elimina (×) ed espansione dei box combinati — righe 89-102, 685-694 */
.etl-canvas .ec-node-btn {
  all: unset;
  position: absolute;
  z-index: 4;
  display: grid;
  place-items: center;
  width: 19px;
  height: 19px;
  border-radius: var(--ec-r-pill);
  cursor: pointer;
  opacity: 0;
  pointer-events: none;
  transform: scale(0.7);
  box-shadow: var(--ec-glass-shadow);
  transition:
    opacity 0.16s,
    transform 0.16s cubic-bezier(0.34, 1.56, 0.64, 1),
    background 0.16s;
}
/* l'area che si può premere è di almeno 32 px, anche se il disegno è più piccolo */
.etl-canvas .ec-node-btn::after {
  content: "";
  position: absolute;
  inset: -7px;
}
.etl-canvas .ec-node-btn svg {
  width: 9px;
  height: 9px;
}
.etl-canvas .ec-del-btn {
  top: -5px;
  left: -5px;
  background: var(--isa-text);
  color: var(--isa-surface-overlay);
}
.etl-canvas .ec-del-btn:hover {
  background: var(--isa-danger);
  color: var(--isa-text-on-danger);
}
.etl-canvas .ec-expand-btn {
  top: -7px;
  right: -7px;
  width: 24px;
  height: 24px;
  background: var(--isa-accent);
  color: var(--isa-text-on-accent);
}
.etl-canvas .ec-expand-btn svg {
  width: 12px;
  height: 12px;
}
.etl-canvas .ec-card:hover .ec-node-btn,
.etl-canvas .ec-card:focus-within .ec-node-btn {
  opacity: 1;
  pointer-events: auto;
  transform: scale(1);
}
.etl-canvas .ec-card.ec-dragging .ec-node-btn,
.etl-canvas .ec-card.ec-drop-target .ec-node-btn {
  opacity: 0;
  pointer-events: none;
}
.etl-canvas .ec-node-btn:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 2px;
}
@media (prefers-reduced-motion: reduce) {
  .etl-canvas .ec-node-btn {
    transition: none;
  }
}
```

### `src/etl-canvas/contrast.ts`

93 righe

```ts
/**
 * Contrasto WCAG tra i token di `tokens.css`. Usato dai test. I colori e il
 * calcolo del contrasto sono in src/theme/color.ts.
 */
import { contrast, over, parseColor } from "../theme/color";
import type { Rgba } from "../theme/color";

export { contrast, luminance, over, parseColor } from "../theme/color";
export type { Rgba } from "../theme/color";

function readBlock(css: string, selector: string, prefix: string): Record<string, string> {
  // il selettore può essere seguito da altri in lista (`.etl-canvas,\n.ec-workspace {`)
  const escaped = selector.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
  const found = new RegExp(`\\n${escaped}(?:,[^{}]*)?\\s*\\{`).exec(css);
  if (!found) throw new Error(`blocco non trovato: ${selector}`);
  const start = found.index;
  const end = css.indexOf("\n}", start);
  const out: Record<string, string> = {};
  for (const m of css
    .slice(start, end)
    .matchAll(new RegExp(`(${prefix}[a-z0-9-]+):\\s*([^;]+);`, "g"))) {
    out[m[1] as string] = (m[2] as string).trim();
  }
  return out;
}

/**
 * Legge i token del canvas (`--ec-*`) da tokens.css e li risolve con i token
 * semantici (`--isa-*`) di un tema e di un modo, già risolti in valori (vedi
 * src/theme/__tests__/support.ts: `resolveTokens`).
 */
export function readTokens(css: string, semantic: Record<string, string>): Record<string, string> {
  const tokens = readBlock(css, ".etl-canvas", "--ec-");
  const out: Record<string, string> = {};
  for (const [name, value] of Object.entries(tokens)) {
    out[name] = value.replace(/var\((--isa-[a-z0-9-]+)\)/g, (_, ref: string) => {
      const resolved = semantic[ref];
      if (resolved === undefined) throw new Error(`token semantico non trovato: ${ref}`);
      return resolved;
    });
  }
  return out;
}

export interface ContrastPair {
  readonly role: string;
  /** Token in primo piano e token di sfondo. */
  readonly fg: string;
  readonly bg: string;
  /** 4.5 per il testo, 3 per gli elementi non testuali. */
  readonly min: number;
}

/**
 * Coppie da verificare. Il fondo del canvas è `--ec-bg` con sopra
 * `--ec-stage`; il vetro è `--ec-surface-strong` sopra il canvas.
 */
export const PAIRS: readonly ContrastPair[] = [
  { role: "Etichetta dataset/output", fg: "--ec-accent-text", bg: "canvas", min: 4.5 },
  { role: "Etichetta lavorazione", fg: "--ec-ink", bg: "canvas", min: 4.5 },
  { role: "Etichetta output parziale", fg: "--ec-muted", bg: "canvas", min: 4.5 },
  { role: "Icona su nodo pieno", fg: "--ec-node-fill-ink", bg: "--ec-node-fill", min: 3 },
  { role: "Icona su nodo lavorazione", fg: "--ec-node-op-ink", bg: "--ec-node-op", min: 3 },
  { role: "Nodo pieno su canvas", fg: "--ec-node-fill", bg: "canvas", min: 3 },
  { role: "Bordo del nodo lavorazione", fg: "--ec-node-op-border", bg: "canvas", min: 3 },
  { role: "Icona fetta vuota", fg: "--ec-split-empty-ink", bg: "--ec-split-empty", min: 3 },
  { role: "Cavo", fg: "--ec-link", bg: "canvas", min: 3 },
  { role: "Indicatore ambra", fg: "--ec-warn", bg: "canvas", min: 3 },
  { role: "Contorno di selezione", fg: "--ec-select", bg: "canvas", min: 3 },
  { role: "Testo dei controlli di zoom", fg: "--ec-ink", bg: "glass", min: 4.5 },
  { role: "«Adatta»", fg: "--ec-accent-text", bg: "glass", min: 4.5 },
  { role: "Flusso nei cavi", fg: "--ec-flow", bg: "canvas", min: 3 },
  { role: "Nodo nella minimappa", fg: "--ec-mm-node", bg: "glass", min: 3 },
  { role: "Nodo dataset nella minimappa", fg: "--ec-mm-node-ds", bg: "glass", min: 3 },
  { role: "Riquadro visibile (minimappa)", fg: "--ec-mm-view-line", bg: "glass", min: 3 },
  { role: "Testo dello stato vuoto", fg: "--ec-empty-ink", bg: "canvas", min: 4.5 },
  { role: "Titolo dello stato vuoto", fg: "--ec-ink", bg: "canvas", min: 4.5 },
];

export function measure(tokens: Record<string, string>, pair: ContrastPair): number {
  const get = (name: string): Rgba => {
    const value = tokens[name];
    if (value === undefined) throw new Error(`token mancante: ${name}`);
    return parseColor(value);
  };
  const canvas = over(get("--ec-stage"), over(get("--ec-bg"), [255, 255, 255, 1]));
  const glass = over(get("--ec-surface-strong"), canvas);
  const bg =
    pair.bg === "canvas" ? canvas : pair.bg === "glass" ? glass : over(get(pair.bg), canvas);
  const fg = over(get(pair.fg), bg);
  return contrast(fg, bg);
}
```

### `src/etl-canvas/drop.ts`

94 righe

```ts
/**
 * Rilascio di un nuovo elemento sul canvas (prototipo `paletteEl` →
 * `onUp`, righe 4939-5069). La cassetta degli strumenti arriva nella Fase 6:
 * questa funzione contiene già tutta la logica di rilascio, così la Fase 6
 * deve solo chiamarla con il payload e il punto (coordinate del MONDO; da un
 * punto dello schermo si passa con `toWorld` di view.ts).
 *
 * Nessuna logica di dominio nuova: il bersaglio si individua con la
 * geometria di etl-layout (`nodeAt`, `linkAt`) e le regole sono quelle di
 * etl-core (`insertable`) e del comando `addNode` di etl-store, che decide
 * fusione, collegamento, inserimento o rilascio nel vuoto.
 */
import { insertable } from "../etl-core";
import type { Card, ComponentId, Graph, Link } from "../etl-core";
import { linkAt, nodeAt } from "../etl-layout";
import type { Point } from "../etl-layout";
import { paletteRelation } from "../etl-store";
import type { CommandResult, DropTarget, EtlStore } from "../etl-store";

/** Ciò che si rilascia: un componente della cassetta o un dataset della libreria. */
export interface CanvasDropPayload {
  readonly component: ComponentId;
  readonly libraryId?: string;
}

export type DropOutcome = "merge" | "link" | "link-reverse" | "insert";

export interface DropPreview {
  /** Bersaglio da passare ad `addNode`, se ce n'è uno valido. */
  readonly target: DropTarget | null;
  /** Cosa succederebbe al rilascio, o `null` (nel vuoto). */
  readonly outcome: DropOutcome | null;
  readonly nodeId?: string;
  readonly linkKey?: string;
}

const NEW_NODE = "__nuovo__";

function kindOf(component: ComponentId): Card["kind"] {
  return component === "dataset" ? "dataset" : "op";
}

function linkByKey(graph: Graph, key: string): Link | undefined {
  return graph.links.find((l) => l.from + "|" + l.to === key);
}

/** Cosa troverebbe un rilascio in `point` (coordinate del mondo), senza modificare nulla. */
export function previewCanvasDrop(
  store: EtlStore,
  payload: CanvasDropPayload,
  point: Point,
): DropPreview {
  const graph = store.getState().graph;
  const kind = kindOf(payload.component);
  const nodeId = nodeAt(graph, point);
  if (nodeId) {
    const rel = paletteRelation(graph, kind, nodeId);
    // senza una relazione possibile `addNode` rilascia come nel vuoto (prototipo, riga 5046)
    return rel
      ? { target: { node: nodeId }, outcome: rel, nodeId }
      : { target: null, outcome: null };
  }
  if (kind === "op") {
    const key = linkAt(store.getRoutes(), point);
    const link = key ? linkByKey(graph, key) : undefined;
    if (key && link && insertable(graph, link, NEW_NODE)) {
      return { target: { link }, outcome: "insert", linkKey: key };
    }
  }
  return { target: null, outcome: null };
}

/**
 * Rilascia `payload` in `point` (coordinate del mondo): crea il nodo e, se il
 * punto è su un nodo o su un cavo, lo fonde, lo collega o lo inserisce.
 * Un solo comando `addNode`: un solo passo di cronologia.
 */
export function handleCanvasDrop(
  store: EtlStore,
  payload: CanvasDropPayload,
  point: Point,
): CommandResult {
  const { target } = previewCanvasDrop(store, payload, point);
  return store.dispatch({
    type: "addNode",
    payload: {
      component: payload.component,
      ...(payload.libraryId !== undefined ? { libraryId: payload.libraryId } : {}),
      point,
      ...(target ? { target } : {}),
    },
  });
}
```

### `src/etl-canvas/engine.ts`

338 righe

```ts
/**
 * Il livello sottile tra le funzioni pure (flow.ts, transitions.ts) e il
 * DOM: ad ogni frame del ciclo condiviso calcola lo stato visivo e ne
 * aggiorna solo gli attributi SVG che cambiano (il tracciato del flusso,
 * l'opacità dell'attesa, `d` durante una transizione). Non ridisegna
 * l'albero React.
 *
 * VINCOLO: qui si legge `route.pts` e `route.d` di `getRoutes()` e non si
 * scrive né si ricalcola mai un percorso. Nessuna chiamata a `settleLinks`
 * o a funzioni di etl-layout che instradano (i test lo verificano). Per
 * disegnare i punti interpolati si usa solo `roundedPath`, che arrotonda
 * punti dati e non sceglie nulla.
 */
import { ELBOW_R, roundedPath } from "../etl-layout";
import { flowWindowFor, tubeOutline, waitingOpacityFor } from "./flow";
import type { Sampler } from "./flow";
import { createLoop } from "./loop";
import type { Loop, LoopEnv } from "./loop";
import { planTransition, sampleTransition } from "./transitions";
import type { Plan, Pt, Visual } from "./transitions";

/** Il minimo di un elemento SVG che serve qui (un vero elemento, o un finto nei test). */
export interface AttrEl {
  setAttribute(name: string, value: string): void;
  style: { opacity: string };
}
export interface PathEl extends AttrEl {
  getTotalLength(): number;
  getPointAtLength(s: number): { x: number; y: number };
}
export interface GroupLike {
  querySelector(selector: string): unknown;
}

export interface LinkInput {
  readonly key: string;
  /** `linkLive` di etl-store: solo i cavi attivi hanno il flusso. */
  readonly live: boolean;
  /** `route.pts` e `route.d` di getRoutes, in sola lettura. */
  readonly pts: readonly Pt[];
  readonly d: string;
  readonly pa: Pt;
  readonly pb: Pt;
}

export interface UpdateInput {
  readonly links: readonly LinkInput[];
  /** Un gesto di trascinamento è in corso (`store.isGesturing()`). */
  readonly gesturing: boolean;
}

interface Handles {
  readonly path: PathEl;
  readonly ghost: AttrEl;
  readonly flow: AttrEl;
  readonly dotA: AttrEl;
  readonly dotB: AttrEl;
}

interface Target {
  readonly pts: readonly Pt[];
  readonly d: string;
  readonly pa: Pt;
  readonly pb: Pt;
}

interface Rec {
  group: GroupLike | null;
  handles: Handles | null;
  live: boolean;
  t0: number | null;
  /** Punti mostrati ora (a metà transizione, quelli interpolati). */
  shown: readonly Pt[] | null;
  target: Target | null;
  tr: { plan: Plan; start: number; oldD: string } | null;
  /** Abbiamo scritto `d`/opacità a mano: a fine transizione si ripristinano. */
  dirty: boolean;
}

export interface MotionEngine {
  registerLink(key: string, group: GroupLike | null): void;
  registerSlice(id: string, el: AttrEl | null): void;
  /** Con `shared` il motore si aggiunge al ciclo di un altro (un solo requestAnimationFrame per tutto il canvas) e non lo chiude. */
  start(env: LoopEnv, shared?: Loop): void;
  stop(): void;
  update(input: UpdateInput): void;
  setGesturing(gesturing: boolean): void;
  /** Solo per i test. */
  debug(): { links: number; slices: number; running: boolean };
}

function handlesOf(g: GroupLike): Handles | null {
  const path = g.querySelector(".ec-link") as PathEl | null;
  const ghost = g.querySelector(".ec-link-ghost") as AttrEl | null;
  const flow = g.querySelector(".ec-flow") as AttrEl | null;
  const dotA = g.querySelector('[data-dot="a"]') as AttrEl | null;
  const dotB = g.querySelector('[data-dot="b"]') as AttrEl | null;
  if (!path || !ghost || !flow || !dotA || !dotB) return null;
  return { path, ghost, flow, dotA, dotB };
}

function setDot(el: AttrEl, p: Pt): void {
  el.setAttribute("cx", String(p.x));
  el.setAttribute("cy", String(p.y));
}

export function createMotionEngine(): MotionEngine {
  const links = new Map<string, Rec>();
  const slices = new Map<string, { el: AttrEl | null; t0: number | null }>();
  let env: LoopEnv | null = null;
  let loop: Loop | null = null;
  let ownsLoop = true;
  let off: (() => void) | null = null;
  let gesturing = false;
  let last: UpdateInput | null = null;

  const reduced = (): boolean => (env ? env.reducedMotion() : false);

  function clearFlow(rec: Rec): void {
    rec.handles?.flow.setAttribute("d", "");
  }

  function restore(rec: Rec): void {
    const h = rec.handles;
    const t = rec.target;
    if (!h || !t || !rec.dirty) return;
    h.path.setAttribute("d", t.d);
    h.path.style.opacity = "";
    h.ghost.setAttribute("d", "");
    h.ghost.style.opacity = "0";
    setDot(h.dotA, t.pa);
    setDot(h.dotB, t.pb);
    rec.dirty = false;
  }

  function apply(rec: Rec, v: Visual): void {
    const h = rec.handles;
    const t = rec.target;
    const tr = rec.tr;
    if (!h || !t || !tr) return;
    rec.dirty = true;
    if (v.kind === "points") {
      rec.shown = v.pts;
      h.path.setAttribute("d", roundedPath(v.pts, ELBOW_R));
      const a = v.pts[0];
      const b = v.pts[v.pts.length - 1];
      if (a) setDot(h.dotA, a);
      if (b) setDot(h.dotB, b);
    } else {
      // dissolvenza incrociata: il vecchio tracciato svanisce, il nuovo compare
      h.ghost.setAttribute("d", tr.oldD);
      h.ghost.style.opacity = String(v.old);
      h.path.style.opacity = String(v.next);
    }
  }

  function finish(rec: Rec): void {
    rec.tr = null;
    rec.shown = rec.target ? rec.target.pts : null;
    restore(rec);
  }

  function renderFlow(rec: Rec, now: number): boolean {
    const h = rec.handles;
    if (!h) return false;
    if (!rec.live || gesturing) {
      clearFlow(rec);
      return false;
    }
    let len = 0;
    try {
      len = h.path.getTotalLength();
    } catch {
      len = 0;
    }
    const win = flowWindowFor(len, now - (rec.t0 ?? now), reduced());
    if (!win) {
      clearFlow(rec);
      return true;
    }
    const sample: Sampler = (s) => h.path.getPointAtLength(s);
    h.flow.setAttribute("d", tubeOutline(sample, len, win));
    return true;
  }

  function renderSlices(now: number): void {
    for (const s of slices.values()) {
      s.t0 ??= now;
      if (s.el) s.el.style.opacity = String(waitingOpacityFor(now - s.t0, reduced()));
    }
  }

  function frame(now: number): boolean {
    let busy = false;
    for (const rec of links.values()) {
      if (rec.tr) {
        const v = sampleTransition(rec.tr.plan, now - rec.tr.start);
        if (v) apply(rec, v);
        if (!v || v.done) finish(rec);
        else busy = true;
      }
      if (renderFlow(rec, now)) busy = true;
    }
    if ([...slices.values()].some((s) => s.el)) {
      renderSlices(now);
      busy = true;
    }
    return busy;
  }

  /** Stato finale, senza movimento: transizioni concluse, flusso fermo a metà cavo, attesa a riposo. */
  function settle(): void {
    const now = env ? env.now() : 0;
    for (const rec of links.values()) {
      if (rec.tr) finish(rec);
      renderFlow(rec, now);
    }
    for (const s of slices.values()) if (s.el) s.el.style.opacity = "";
  }

  function applyUpdate(input: UpdateInput): void {
    if (!env) return;
    const now = env.now();
    gesturing = input.gesturing;
    const seen = new Set<string>();
    for (const l of input.links) {
      seen.add(l.key);
      const rec = links.get(l.key) ?? newRec();
      links.set(l.key, rec);
      rec.live = l.live;
      rec.t0 ??= now;
      const prev = rec.shown ?? rec.target?.pts ?? null;
      const plan = planTransition({
        prev,
        next: l.pts,
        gesturing: input.gesturing,
        reduced: env.reducedMotion(),
      });
      const oldD = rec.target?.d ?? l.d;
      rec.target = { pts: l.pts, d: l.d, pa: l.pa, pb: l.pb };
      if (plan.kind === "none") {
        rec.tr = null;
        rec.shown = l.pts;
        restore(rec);
      } else {
        rec.tr = { plan, start: now, oldD };
        const v = sampleTransition(plan, 0);
        if (v) apply(rec, v);
      }
    }
    for (const key of [...links.keys()]) if (!seen.has(key)) links.delete(key);
    // le fette smontate (React azzera il riferimento a ogni rendering): ora si dimenticano davvero
    for (const [id, s] of [...slices]) if (!s.el) slices.delete(id);
    if (env.reducedMotion()) settle();
    loop?.wake();
  }

  function newRec(): Rec {
    return {
      group: null,
      handles: null,
      live: false,
      t0: null,
      shown: null,
      target: null,
      tr: null,
      dirty: false,
    };
  }

  return {
    registerLink(key, group) {
      if (!group) {
        // React smonta il gruppo (o ne cambia il riferimento): si dimentica finché non torna
        const rec = links.get(key);
        if (rec) {
          rec.group = null;
          rec.handles = null;
        }
        return;
      }
      const rec = links.get(key) ?? newRec();
      links.set(key, rec);
      rec.group = group;
      rec.handles = handlesOf(group);
      rec.dirty = false;
    },

    registerSlice(id, el) {
      // React chiama il vecchio riferimento con null e il nuovo con l'elemento a ogni
      // rendering del nodo: la fase dell'attesa si conserva per identificativo
      const known = slices.get(id);
      if (!el) {
        if (known) known.el = null;
        return;
      }
      slices.set(id, { el, t0: known?.t0 ?? (env ? env.now() : null) });
      loop?.wake();
    },

    start(e, shared) {
      this.stop();
      env = e;
      ownsLoop = !shared;
      loop = shared ?? createLoop(e);
      off = loop.add({ frame, settle });
      if (last) applyUpdate(last);
    },

    stop() {
      off?.();
      off = null;
      if (ownsLoop) loop?.dispose();
      loop = null;
      env = null;
    },

    update(input) {
      last = input;
      applyUpdate(input);
    },

    setGesturing(g) {
      if (g === gesturing) return;
      gesturing = g;
      if (last) last = { ...last, gesturing: g };
      for (const rec of links.values()) if (g) clearFlow(rec);
      loop?.wake();
    },

    debug: () => ({
      links: links.size,
      slices: [...slices.values()].filter((s) => s.el).length,
      running: loop?.running() ?? false,
    }),
  };
}
```

### `src/etl-canvas/flow.ts`

180 righe

```ts
/**
 * Flusso nei cavi ("tubo elastico") e attesa delle fette vuote: calcolo puro.
 * Nessun React, nessun timer, nessun DOM: il tempo trascorso è un argomento.
 * Semantica del prototipo (docs/prototype/isa-fusion-prototype.html):
 * righe 1415-1424 (costanti e profilo), 1478 (smooth01), 1480-1521
 * (`animateBubbles`), 669-670 (attesa).
 */

/** Spessore del cavo, coincide con il tubo a riposo (riga 1415). */
export const BASE_W = 2.1;
/** px al millisecondo: stessa andatura su cavi lunghi e corti (riga 1416). */
export const SPEED = 0.16;
/** Rigonfiamento massimo per lato (riga 1417). */
export const BALL = 4.4;
/** Apertura rapida davanti, richiusura più lenta dietro (riga 1418). */
export const FRONT = 7.5;
export const BACK = 19;
/** Il tubo emerge dalla porta e vi rientra su questo tratto (riga 1510: `/ 22`). */
export const EDGE_FADE = 22;
/** Sotto questa lunghezza il tratto non si disegna (riga 1497: `s1 - s0 < 2`). */
export const MIN_VISIBLE = 2;
/** Campionamento del contorno: un punto ogni 1,6 px, almeno 10 (riga 1500). */
export const SAMPLE_STEP = 1.6;
export const MIN_SAMPLES = 10;

/** Riga 1478. */
export function smooth01(x: number): number {
  const c = Math.max(0, Math.min(1, x));
  return c * c * (3 - 2 * c);
}

/** Profilo gaussiano asimmetrico attorno al punto che avanza (righe 1419-1422). */
export function tubeProfile(u: number): number {
  const sg = u >= 0 ? FRONT : BACK;
  return Math.exp(-(u * u) / (sg * sg));
}

/** Tratto del cavo occupato dal tubo: `sb` è il punto che avanza (riga 1489-1495). */
export interface FlowWindow {
  readonly cycle: number;
  readonly sb: number;
  readonly s0: number;
  readonly s1: number;
}

/**
 * Finestra del tubo a `elapsedMs` dall'inizio del cavo (`now - st.t0`), su
 * un cavo lungo `len`. `null` se non c'è nulla da disegnare (righe 1488,
 * 1497). Il ciclo è `len + BACK * 3.2`: la pallina entra dalla porta di
 * uscita, sparisce oltre quella di ingresso e riparte.
 */
export function flowWindow(len: number, elapsedMs: number): FlowWindow | null {
  if (!(len > 0)) return null;
  const cycle = len + BACK * 3.2;
  const sb = ((elapsedMs * SPEED) % cycle) - BACK * 1.1;
  const s0 = Math.max(0, sb - BACK * 3);
  const s1 = Math.min(len, sb + FRONT * 3.2);
  if (s1 - s0 < MIN_VISIBLE) return null;
  return { cycle, sb, s0, s1 };
}

/**
 * Indicazione statica per chi preferisce meno movimento: lo stesso tubo,
 * fermo a metà cavo. Non esiste nel prototipo (NOTE_DIVERGENZE.md).
 */
export function staticFlowWindow(len: number): FlowWindow | null {
  if (!(len > 0)) return null;
  const sb = len / 2;
  return {
    cycle: len + BACK * 3.2,
    sb,
    s0: Math.max(0, sb - BACK * 3),
    s1: Math.min(len, sb + FRONT * 3.2),
  };
}

/** Finestra da mostrare: quella animata, o quella statica con movimento ridotto. */
export function flowWindowFor(len: number, elapsedMs: number, reduced: boolean): FlowWindow | null {
  return reduced ? staticFlowWindow(len) : flowWindow(len, elapsedMs);
}

export type Sampler = (s: number) => { readonly x: number; readonly y: number };

/**
 * Contorno chiuso del tubo (righe 1500-1516): campiona il percorso `d`
 * già calcolato con `sample` (in un browser `path.getPointAtLength`) e
 * allarga il tratto secondo `tubeProfile`. Non tocca mai il percorso.
 */
export function tubeOutline(sample: Sampler, len: number, win: FlowWindow): string {
  const n = Math.max(MIN_SAMPLES, Math.ceil((win.s1 - win.s0) / SAMPLE_STEP));
  const pts: { x: number; y: number; s: number }[] = [];
  for (let k = 0; k <= n; k++) {
    const s = win.s0 + ((win.s1 - win.s0) * k) / n;
    const q = sample(s);
    pts.push({ x: q.x, y: q.y, s });
  }
  const left: string[] = [];
  const right: string[] = [];
  for (let k = 0; k <= n; k++) {
    const a = pts[Math.max(0, k - 1)] as { x: number; y: number; s: number };
    const b = pts[Math.min(n, k + 1)] as { x: number; y: number; s: number };
    const p = pts[k] as { x: number; y: number; s: number };
    let tx = b.x - a.x;
    let ty = b.y - a.y;
    const tl = Math.hypot(tx, ty) || 1;
    tx /= tl;
    ty /= tl;
    const edge = smooth01(Math.min(p.s, len - p.s) / EDGE_FADE);
    const w = BASE_W / 2 + BALL * tubeProfile(p.s - win.sb) * edge;
    left.push((p.x - ty * w).toFixed(2) + " " + (p.y + tx * w).toFixed(2));
    right.push((p.x + ty * w).toFixed(2) + " " + (p.y - tx * w).toFixed(2));
  }
  return "M " + left.concat(right.reverse()).join(" L ") + " Z";
}

// --- Attesa delle fette vuote --------------------------------------------------

/** `animation: waiting 1.9s ease-in-out infinite` (riga 669). */
export const WAIT_PERIOD = 1900;
/** `@keyframes waiting { 0%,100% {opacity:.45} 50% {opacity:.95} }` (riga 670). */
export const WAIT_LOW = 0.45;
export const WAIT_HIGH = 0.95;
/** Opacità a riposo del simbolo (riga 669: `opacity:.85`), anche con movimento ridotto. */
export const WAIT_REST = 0.85;

/** Funzione di temporizzazione CSS `cubic-bezier(x1, y1, x2, y2)`. */
export function cubicBezier(x1: number, y1: number, x2: number, y2: number): (x: number) => number {
  const cx = 3 * x1;
  const bx = 3 * (x2 - x1) - cx;
  const ax = 1 - cx - bx;
  const cy = 3 * y1;
  const by = 3 * (y2 - y1) - cy;
  const ay = 1 - cy - by;
  const bx_ = (t: number) => ((ax * t + bx) * t + cx) * t;
  const by_ = (t: number) => ((ay * t + by) * t + cy) * t;
  const dx_ = (t: number) => (3 * ax * t + 2 * bx) * t + cx;
  return (x) => {
    if (x <= 0) return 0;
    if (x >= 1) return 1;
    let t = x;
    for (let i = 0; i < 8; i++) {
      const err = bx_(t) - x;
      if (Math.abs(err) < 1e-7) return by_(t);
      const d = dx_(t);
      if (Math.abs(d) < 1e-6) break;
      t -= err / d;
    }
    let lo = 0;
    let hi = 1;
    t = x;
    for (let i = 0; i < 40; i++) {
      const v = bx_(t);
      if (Math.abs(v - x) < 1e-7) break;
      if (v < x) lo = t;
      else hi = t;
      t = (lo + hi) / 2;
    }
    return by_(t);
  };
}

/** `ease-in-out` di CSS. */
export const easeInOut = cubicBezier(0.42, 0, 0.58, 1);

/**
 * Opacità del simbolo `<>` di una fetta vuota a `elapsedMs`: dal basso
 * (0,45) all'alto (0,95) a metà periodo e ritorno, con `ease-in-out` su
 * ciascuna metà, come i `@keyframes` del prototipo.
 */
export function waitingOpacity(elapsedMs: number): number {
  const phase = (((elapsedMs % WAIT_PERIOD) + WAIT_PERIOD) % WAIT_PERIOD) / WAIT_PERIOD;
  const u = phase < 0.5 ? phase * 2 : (1 - phase) * 2;
  return WAIT_LOW + (WAIT_HIGH - WAIT_LOW) * easeInOut(u);
}

/** Opacità da mostrare: animata, o ferma a riposo con movimento ridotto. */
export function waitingOpacityFor(elapsedMs: number, reduced: boolean): number {
  return reduced ? WAIT_REST : waitingOpacity(elapsedMs);
}
```

### `src/etl-canvas/icons.tsx`

24 righe

```tsx
import { EMPTY_SLOT_ICON, ICONS } from "../etl-core";
import type { ComponentId } from "../etl-core";

/** Icona del catalogo di etl-core (prototipo `svgTag`, righe 979-981). I tracciati sono costanti del catalogo, mai dati dell'utente. */
export function Icon(props: {
  id: ComponentId | "empty";
  svgRef?: (el: SVGSVGElement | null) => void;
}) {
  const inner = props.id === "empty" ? EMPTY_SLOT_ICON : ICONS[props.id];
  return (
    <svg
      ref={props.svgRef}
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth={2}
      strokeLinecap="round"
      strokeLinejoin="round"
      aria-hidden="true"
      dangerouslySetInnerHTML={{ __html: inner }}
    />
  );
}
```

### `src/etl-canvas/index.ts`

21 righe

```ts
/**
 * etl-canvas — Fase 4a: resa visiva del canvas ETL. Importa da etl-core,
 * etl-layout ed etl-store; nessuno di questi importa da qui.
 */
export { EtlCanvas, CanvasSurface } from "./EtlCanvas";
export type { CanvasSurfaceProps } from "./EtlCanvas";
export { prototypeScene } from "./seed";
export { fit, zoomIn, zoomOut, zoomReset, zoomAtPoint } from "./actions";
export { fitView, zoomAt, minimapFrame, bounds } from "./view";
export { nodeView, countClass, isPartial, slicesOf } from "./model";
export { createInteractionController } from "./interaction";
export type {
  InteractionController,
  InteractionUi,
  DownTarget,
  PointerInput,
  KeyInput,
} from "./interaction";
export { handleCanvasDrop, previewCanvasDrop } from "./drop";
export type { CanvasDropPayload, DropPreview } from "./drop";
```

### `src/etl-canvas/inspector/ActionMenu.tsx`

72 righe

```tsx
/**
 * Menu di azioni (ruolo menu) in un portale: usato dal pannello espanso per i
 * passaggi. Frecce, Home, Fine, Invio, Esc; il focus parte dalla prima voce e,
 * alla chiusura, torna al pulsante che l'ha aperto.
 */
import { useEffect, useRef, useState } from "react";
import type { KeyboardEvent, RefObject } from "react";
import { copy } from "./copy";
import { Menu } from "./Menu";

export interface ActionItem {
  readonly id: string;
  readonly label: string;
  readonly danger?: boolean;
  readonly onSelect: () => void;
}

export function ActionMenu(props: {
  readonly anchor: RefObject<HTMLElement | null>;
  readonly items: readonly ActionItem[];
  readonly onClose: (returnFocus: boolean) => void;
  readonly ariaLabel?: string;
}) {
  const { items } = props;
  const [active, setActive] = useState(0);
  const refs = useRef<(HTMLButtonElement | null)[]>([]);
  useEffect(() => refs.current[active]?.focus(), [active]);
  const onKey = (e: KeyboardEvent) => {
    const n = items.length;
    if (e.key === "ArrowDown") setActive((a) => (a + 1) % n);
    else if (e.key === "ArrowUp") setActive((a) => (a - 1 + n) % n);
    else if (e.key === "Home") setActive(0);
    else if (e.key === "End") setActive(n - 1);
    else if (e.key === "Tab") {
      props.onClose(false);
      return;
    } else return;
    e.preventDefault();
  };
  return (
    <Menu
      anchor={props.anchor}
      role="menu"
      ariaLabel={props.ariaLabel ?? copy.stepsMenuLabel}
      onClose={() => props.onClose(false)}
      onEscape={() => props.onClose(true)}
    >
      <div className="ei-menu-scroll" data-scroll="" onKeyDown={onKey}>
        {items.map((it, i) => (
          <button
            key={it.id}
            ref={(el) => {
              refs.current[i] = el;
            }}
            type="button"
            role="menuitem"
            tabIndex={i === active ? 0 : -1}
            className={"ei-option ei-menuitem" + (it.danger ? " ei-danger" : "")}
            onPointerMove={() => setActive(i)}
            onClick={() => {
              it.onSelect();
              props.onClose(false);
            }}
          >
            <span className="ei-option-label">{it.label}</span>
          </button>
        ))}
      </div>
    </Menu>
  );
}
```

### `src/etl-canvas/inspector/BlockedNotice.tsx`

16 righe

```tsx
/** Stato bloccato (prototipo, righe 3708-3719): una lavorazione senza tabella in ingresso non ha nulla da configurare. */
import { copy } from "./copy";
import { LockIcon } from "./icons";

export function BlockedNotice(props: { readonly capacity: number }) {
  return (
    <>
      <div className="ei-lock" role="status" data-testid="ei-blocked">
        <LockIcon />
        <div>{copy.lockedText}</div>
      </div>
      {props.capacity > 1 ? <div className="ei-help">{copy.lockedJoin(props.capacity)}</div> : null}
    </>
  );
}
```

