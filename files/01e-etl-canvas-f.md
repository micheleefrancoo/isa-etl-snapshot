# 01e-etl-canvas-f.md

File in questo blocco:

- `src/etl-canvas/panels/layout.ts`
- `src/etl-canvas/panels/panels.css`
- `src/etl-canvas/panels/ui-icons.tsx`
- `src/etl-canvas/seed.ts`
- `src/etl-canvas/tokens.css`
- `src/etl-canvas/transitions.ts`
- `src/etl-canvas/view.ts`

---

### `src/etl-canvas/panels/layout.ts`

152 righe

```ts
/**
 * Geometria dei pannelli agganciabili: funzioni pure sullo stato dei pannelli
 * di etl-store (`Panels`: lato e aperto/chiuso di ciascuno). Misure del
 * prototipo (docs/prototype/isa-fusion-prototype.html, righe 4756-4900), salvo
 * l'altezza dell'area di lavoro (vedi `viewCompensation`).
 *
 * Nessuna logica di dominio: misure e compensazione della vista sono
 * geometria dell'interfaccia.
 */
import type { PanelKey, Panels, Side, View } from "../../etl-store";

/** I due pannelli, nell'ordine delle schede (prototipo, riga 4797). */
export const PANEL_KEYS: readonly PanelKey[] = ["tools", "insp"];

export const PANEL_LABEL: Readonly<Record<PanelKey, string>> = {
  tools: "Strumenti",
  insp: "Inspector",
};

/** Nome del pannello nei suggerimenti (riga 4759-4760). */
export const PANEL_NAME: Readonly<Record<PanelKey, string>> = {
  tools: "gli strumenti",
  insp: "l’inspector",
};

export const SIDE_NAME: Readonly<Record<Side, string>> = {
  left: "sinistro",
  right: "destro",
  top: "superiore",
  bottom: "inferiore",
};

export const SIDES: readonly Side[] = ["left", "right", "top", "bottom"];

/** Misure proprie di ciascun pannello: larghezza (bordi verticali) e altezza (orizzontali). Righe 4759-4760. */
export const PANEL_SIZE: Readonly<Record<PanelKey, { readonly w: number; readonly h: number }>> = {
  tools: { w: 264, h: 206 },
  insp: { w: 308, h: 300 },
};

/** Margine verso il canvas, parte della misura del pannello (CSS `.panel`, riga 227). */
export const EXTENT_PAD = 16;

/** Distanza tra due tacche sullo stesso bordo (riga 4850). */
export const NOTCH_SPREAD = 40;

export const isVertical = (side: Side): boolean => side === "left" || side === "right";

/** I due pannelli stanno sullo stesso bordo: diventano schede di un unico pannello. */
export function isGrouped(panels: Panels): boolean {
  return panels.tools.side === panels.insp.side;
}

/** Misura effettiva: in gruppo entrambi prendono la maggiore, così cambiare scheda non sposta il canvas (riga 4772). */
export function panelSize(panels: Panels, key: PanelKey): { w: number; h: number } {
  if (!isGrouped(panels)) return { ...PANEL_SIZE[key] };
  return {
    w: Math.max(PANEL_SIZE.tools.w, PANEL_SIZE.insp.w),
    h: Math.max(PANEL_SIZE.tools.h, PANEL_SIZE.insp.h),
  };
}

/** Quanto spazio sottrae al canvas un pannello aperto (riga 4763). */
export function panelExtent(panels: Panels, key: PanelKey): number {
  const s = panelSize(panels, key);
  return (isVertical(panels[key].side) ? s.w : s.h) + EXTENT_PAD;
}

/** Spazio sottratto al canvas dai pannelli aperti sul bordo `side`. */
export function openExtent(panels: Panels, side: Side): number {
  return PANEL_KEYS.filter((k) => panels[k].open && panels[k].side === side).reduce(
    (sum, k) => sum + panelExtent(panels, k),
    0,
  );
}

/**
 * Spostamento da dare alla vista perché i nodi restino fermi sullo schermo
 * quando i pannelli passano da `before` a `after`.
 *
 * Solo i bordi SINISTRO e ALTO spostano l'origine del canvas: lì il canvas
 * cede spazio dal suo lato d'origine, quindi `view.x` (sinistra) e `view.y`
 * (alto) si compensano. A destra e in basso il canvas cede dal lato opposto e
 * la vista non cambia. Vale anche quando cambia la misura di un pannello
 * aperto (due pannelli che diventano schede).
 *
 * Nel prototipo (righe 4780-4784, 4815-4824) i pannelli orizzontali non
 * toglievano altezza al canvas ma la aggiungevano all'area di lavoro, perché
 * la pagina scorre; nell'app il contenitore ha l'altezza della finestra, quindi
 * anche i pannelli orizzontali sottraggono spazio al canvas (vedi
 * NOTE_DIVERGENZE.md).
 */
export function viewCompensation(
  before: Panels,
  after: Panels,
): { readonly dx: number; readonly dy: number } {
  return {
    dx: openExtent(before, "left") - openExtent(after, "left"),
    dy: openExtent(before, "top") - openExtent(after, "top"),
  };
}

/** Applica la compensazione a una vista. */
export function compensate(view: View, before: Panels, after: Panels): View {
  const { dx, dy } = viewCompensation(before, after);
  return dx === 0 && dy === 0 ? view : { ...view, x: view.x + dx, y: view.y + dy };
}

/**
 * Altezza minima del canvas, la riga centrale dello spazio di lavoro. Se i
 * pannelli in alto e in basso lasciano meno spazio, scorre il contenitore
 * dello spazio di lavoro, non la pagina.
 */
export const MIN_CANVAS_HEIGHT = 200;

/** Il bordo del canvas più vicino a un punto (riga 4851-4856). */
export function nearestSide(
  point: { readonly x: number; readonly y: number },
  rect: {
    readonly left: number;
    readonly right: number;
    readonly top: number;
    readonly bottom: number;
  },
): Side {
  const d: Record<Side, number> = {
    left: Math.abs(point.x - rect.left),
    right: Math.abs(rect.right - point.x),
    top: Math.abs(point.y - rect.top),
    bottom: Math.abs(rect.bottom - point.y),
  };
  return SIDES.slice().sort((a, b) => d[a] - d[b])[0] as Side;
}

/** Scostamento della tacca lungo il bordo: due tacche sullo stesso bordo si affiancano (riga 4850). */
export function notchOffset(panels: Panels, key: PanelKey): number {
  const mates = PANEL_KEYS.filter((k) => panels[k].side === panels[key].side);
  if (mates.length < 2) return 0;
  return mates.indexOf(key) === 0 ? -NOTCH_SPREAD : NOTCH_SPREAD;
}

/** La tacca si nasconde se il pannello è aperto o se si raggiunge dalla scheda dell'altro, aperto sullo stesso bordo (righe 4856-4858). */
export function notchHidden(panels: Panels, key: PanelKey): boolean {
  const other = PANEL_KEYS.find((k) => k !== key) as PanelKey;
  return panels[key].open || (panels[other].side === panels[key].side && panels[other].open);
}

/** Il pannello aperto sul bordo `side` (la scheda attiva), se c'è. */
export function activeTab(panels: Panels, side: Side): PanelKey | null {
  return PANEL_KEYS.find((k) => panels[k].side === side && panels[k].open) ?? null;
}
```

### `src/etl-canvas/panels/panels.css`

638 righe

```css
/*
 * Pannelli agganciabili e cassetta degli strumenti (Fase 6a). Misure e
 * transizioni del prototipo (docs/prototype/isa-fusion-prototype.html, righe
 * indicate); colori, raggi e ombre solo dai token di ../tokens.css
 * (`--ec-*`, che leggono i semantici `--isa-*` dei temi). L'orientamento
 * (colonna sui bordi verticali, fascia sugli orizzontali) è la classe
 * `ec-horiz`, scelta dal bordo.
 */

/* spazio di lavoro: il canvas al centro, quattro approdi attorno — righe 211-220 */
.ec-workspace-host {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 0;
  /* se i pannelli in alto e in basso lasciano al canvas meno del minimo, scorre questo contenitore, non la pagina */
  overflow-x: hidden;
  overflow-y: auto;
}
.ec-workspace {
  display: grid;
  width: 100%;
  /* l'altezza è quella del contenitore: sopra e sotto i pannelli tolgono altezza al canvas (riga centrale) */
  height: 100%;
  grid-template-columns: auto minmax(0, 1fr) auto;
  grid-template-rows: auto minmax(var(--ec-canvas-min-h), 1fr) auto;
  color: var(--ec-ink);
  font-family: var(--ec-font);
}
.ec-workspace * {
  box-sizing: border-box;
}
.ec-dock-top {
  grid-column: 1 / 4;
  grid-row: 1;
  display: flex;
  flex-direction: column;
}
.ec-dock-bottom {
  grid-column: 1 / 4;
  grid-row: 3;
  display: flex;
  flex-direction: column;
}
.ec-dock-left {
  grid-column: 1;
  grid-row: 2;
  display: flex;
}
.ec-dock-right {
  grid-column: 3;
  grid-row: 2;
  display: flex;
}
.ec-center {
  grid-column: 2;
  grid-row: 2;
  position: relative;
  min-width: 0;
  min-height: 0;
}
/* dentro lo spazio di lavoro l'altezza minima del canvas è quella della riga centrale (MIN_CANVAS_HEIGHT), non i 520px del canvas isolato */
.ec-workspace .ec-center .etl-canvas {
  min-height: 0;
}

/* un pannello cede spazio aprendosi; il margine verso il canvas fa parte della sua misura — righe 224-240 */
.ec-panel {
  overflow: hidden;
  position: relative;
  flex: 0 0 auto;
  transition:
    width 0.3s cubic-bezier(0.32, 0.72, 0, 1),
    height 0.3s cubic-bezier(0.32, 0.72, 0, 1);
}
.ec-panel.ec-side-left,
.ec-panel.ec-side-right {
  width: 0;
  height: 100%;
}
.ec-panel.ec-side-left.ec-open,
.ec-panel.ec-side-right.ec-open {
  width: calc(var(--pw) + 16px);
}
.ec-panel.ec-side-top,
.ec-panel.ec-side-bottom {
  height: 0;
  width: 100%;
}
.ec-panel.ec-side-top.ec-open,
.ec-panel.ec-side-bottom.ec-open {
  height: calc(var(--ph) + 16px);
}
.ec-panel-body {
  position: absolute;
}
.ec-panel.ec-side-left > .ec-panel-body {
  left: 0;
  top: 0;
  width: var(--pw);
  height: 100%;
}
.ec-panel.ec-side-right > .ec-panel-body {
  right: 0;
  top: 0;
  width: var(--pw);
  height: 100%;
}
.ec-panel.ec-side-top > .ec-panel-body {
  left: 0;
  top: 0;
  height: var(--ph);
  width: 100%;
}
.ec-panel.ec-side-bottom > .ec-panel-body {
  left: 0;
  bottom: 0;
  height: var(--ph);
  width: 100%;
}
.ec-panel.ec-instant {
  transition: none !important;
}

/* due pannelli sullo stesso bordo diventano schede di un unico pannello — righe 305-328 */
.ec-dock-tabs {
  position: absolute;
  z-index: 3;
  display: flex;
  gap: 4px;
  padding: 4px;
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  border-radius: var(--ec-r-minimap);
}
.ec-dock-tab {
  all: unset;
  cursor: pointer;
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  padding: 7px 10px;
  border-radius: var(--ec-r-sm);
  font-family: inherit;
  font-size: 12px;
  font-weight: 700;
  color: var(--ec-empty-ink);
  transition:
    background 0.18s,
    color 0.18s;
}
.ec-dock-tab svg {
  width: 14px;
  height: 14px;
  flex-shrink: 0;
}
.ec-dock-tab:hover {
  color: var(--ec-ink);
}
.ec-dock-tab:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-dock-tab.ec-on {
  background: var(--ec-accent);
  color: var(--ec-node-fill-ink);
}
.ec-panel.ec-grouped.ec-side-left > .ec-dock-tabs {
  left: 0;
  top: 0;
  width: var(--pw);
  height: 42px;
}
.ec-panel.ec-grouped.ec-side-right > .ec-dock-tabs {
  right: 0;
  top: 0;
  width: var(--pw);
  height: 42px;
}
.ec-panel.ec-grouped.ec-side-left > .ec-panel-body,
.ec-panel.ec-grouped.ec-side-right > .ec-panel-body {
  top: 50px;
  height: calc(100% - 50px);
}
.ec-panel.ec-grouped.ec-horiz > .ec-dock-tabs {
  left: 0;
  width: 48px;
  height: var(--ph);
  flex-direction: column;
}
.ec-panel.ec-grouped.ec-side-top > .ec-dock-tabs {
  top: 0;
}
.ec-panel.ec-grouped.ec-side-bottom > .ec-dock-tabs {
  bottom: 0;
}
.ec-panel.ec-grouped.ec-horiz > .ec-panel-body {
  left: 56px;
  width: calc(100% - 56px);
}
.ec-panel.ec-grouped.ec-horiz .ec-dock-tab span {
  display: none;
}
@keyframes ec-tab-in {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}
.ec-panel.ec-tab-in > .ec-panel-body {
  animation: ec-tab-in 0.26s cubic-bezier(0.32, 0.72, 0, 1);
}

/* tacca: il pannello chiuso resta raggiungibile, e si trascina su un altro bordo — righe 270-296 */
.ec-notch {
  all: unset;
  cursor: grab;
  position: absolute;
  z-index: 16;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--ec-surface-strong);
  color: var(--ec-accent-text);
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  transition:
    opacity 0.2s,
    transform 0.3s cubic-bezier(0.32, 0.72, 0, 1);
  touch-action: none;
}
.ec-notch:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 2px;
}
.ec-notch svg {
  width: 14px;
  height: 14px;
}
.ec-notch-left {
  left: 0;
  width: 24px;
  height: 66px;
  border-left: 0;
  border-top-right-radius: var(--ec-r-minimap);
  border-bottom-right-radius: var(--ec-r-minimap);
}
.ec-notch-right {
  right: 0;
  width: 24px;
  height: 66px;
  border-right: 0;
  border-top-left-radius: var(--ec-r-minimap);
  border-bottom-left-radius: var(--ec-r-minimap);
}
.ec-notch-top {
  top: 0;
  height: 24px;
  width: 66px;
  border-top: 0;
  border-bottom-left-radius: var(--ec-r-minimap);
  border-bottom-right-radius: var(--ec-r-minimap);
}
.ec-notch-bottom {
  bottom: 0;
  height: 24px;
  width: 66px;
  border-bottom: 0;
  border-top-left-radius: var(--ec-r-minimap);
  border-top-right-radius: var(--ec-r-minimap);
}
.ec-notch.ec-hidden {
  opacity: 0;
  pointer-events: none;
}
.ec-notch.ec-dragging {
  position: fixed;
  right: auto;
  bottom: auto;
  transition: none;
  cursor: grabbing;
  border-radius: var(--ec-r-minimap);
  border: 1px solid var(--ec-accent);
  width: 34px;
  height: 34px;
  box-shadow: var(--ec-drag-shadow);
}
.ec-edge-hint {
  position: absolute;
  z-index: 15;
  background: var(--ec-accent);
  opacity: 0;
  pointer-events: none;
  border-radius: var(--ec-r-pill);
  transition: opacity 0.15s;
}
.ec-edge-hint.ec-on {
  opacity: 0.55;
}
.ec-edge-hint.ec-e-left {
  left: 4px;
  top: 12%;
  width: 4px;
  height: 76%;
}
.ec-edge-hint.ec-e-right {
  right: 4px;
  top: 12%;
  width: 4px;
  height: 76%;
}
.ec-edge-hint.ec-e-top {
  top: 4px;
  left: 12%;
  height: 4px;
  width: 76%;
}
.ec-edge-hint.ec-e-bottom {
  bottom: 4px;
  left: 12%;
  height: 4px;
  width: 76%;
}
.ec-workspace .ec-center .ec-hint {
  position: absolute;
  left: 50%;
  bottom: 14px;
  transform: translateX(-50%);
  z-index: 16;
  max-width: 70%;
  padding: 7px 14px;
  border-radius: var(--ec-r-pill);
  background: var(--ec-surface-strong);
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  color: var(--ec-ink);
  font-size: 12px;
  font-weight: 600;
  pointer-events: none;
}

/* contenuto del pannello (cassetta e Inspector) — righe 235-268 */
.ec-tb-inner {
  box-sizing: border-box;
  width: 100%;
  height: 100%;
  overflow-y: auto;
  overscroll-behavior: contain;
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  border-radius: var(--ec-r-fill);
  padding: 18px 16px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.ec-tb-inner > * {
  flex: 0 0 auto;
}
.ec-tb-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;
}
.ec-tb-title {
  font-weight: 800;
  font-size: 15px;
}
.ec-close-btn {
  all: unset;
  cursor: pointer;
  display: grid;
  place-items: center;
  width: 26px;
  height: 26px;
  border-radius: var(--ec-r-pill);
  color: var(--ec-empty-ink);
}
.ec-close-btn:hover {
  color: var(--ec-ink);
  background: var(--ec-accent-soft);
}
.ec-close-btn:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-close-arrow {
  display: inline-flex;
  transition: transform 0.2s;
}
.ec-tb-sec {
  border-top: 1px solid var(--ec-panel-border);
  padding-top: 8px;
}
.ec-tb-sec:first-of-type {
  border-top: 0;
}
.ec-tb-sec-head {
  all: unset;
  cursor: pointer;
  width: 100%;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 4px 2px 8px;
}
.ec-tb-sec-head:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-chev {
  display: inline-flex;
  transition: transform 0.2s;
}
.ec-chev svg {
  width: 11px;
  height: 11px;
}
.ec-tb-sec.ec-open .ec-chev {
  transform: rotate(90deg);
  color: var(--ec-accent-text);
}
.ec-tb-sec-name {
  font-size: 10px;
  font-weight: 800;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--ec-empty-ink);
}
.ec-tb-sec-body {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px 4px;
  max-height: 0;
  overflow: hidden;
  opacity: 0;
  transition:
    max-height 0.3s cubic-bezier(0.32, 0.72, 0, 1),
    opacity 0.2s,
    padding 0.3s;
}
.ec-tb-sec.ec-open .ec-tb-sec-body {
  max-height: 640px;
  opacity: 1;
  padding-bottom: 10px;
}

/* voci — righe 40-53 */
.ec-pal-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  width: auto;
  cursor: grab;
  touch-action: none;
}
.ec-pal-item:active {
  cursor: grabbing;
}
.ec-pal-chip {
  width: 44px;
  height: 44px;
  border-radius: var(--ec-r-sm);
  background: var(--ec-node-op);
  color: var(--ec-node-op-ink);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.18s cubic-bezier(0.34, 1.56, 0.64, 1);
}
.ec-pal-item:hover .ec-pal-chip {
  transform: translateY(-2px) scale(1.05);
}
.ec-pal-item.ec-source .ec-pal-chip {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
}
.ec-pal-chip svg {
  width: 20px;
  height: 20px;
}
.ec-pal-label {
  font-size: 9.5px;
  font-weight: 700;
  color: var(--ec-empty-ink);
  text-align: center;
  line-height: 1.2;
}
.ec-lib-meta {
  font-size: 8.5px;
  color: var(--ec-empty-ink);
  font-weight: 600;
  margin-top: -3px;
  text-align: center;
}
.ec-tb-upload {
  all: unset;
  cursor: pointer;
  grid-column: 1 / -1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  padding: 9px 12px;
  border-radius: var(--ec-r-sm);
  border: 1.5px dashed var(--ec-accent-soft-2);
  font-size: 12px;
  font-weight: 700;
  color: var(--ec-accent-text);
  background: var(--ec-mm-view-bg);
}
.ec-tb-upload:hover {
  background: var(--ec-accent-soft);
}
.ec-tb-upload:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-tb-upload svg {
  width: 14px;
  height: 14px;
}
.ec-tb-empty {
  grid-column: 1 / -1;
  font-size: 11px;
  color: var(--ec-empty-ink);
  font-weight: 600;
  text-align: center;
  padding: 2px 0 4px;
}
.ec-tb-status {
  font-size: 11px;
  font-weight: 600;
  color: var(--ec-empty-ink);
  line-height: 1.4;
}
.ec-insp-name {
  font-size: 13px;
  font-weight: 700;
  color: var(--ec-ink);
  overflow-wrap: anywhere;
}

/* anteprima del trascinamento dalla cassetta — righe 55-63 */
.ec-ghost {
  position: fixed;
  z-index: 60;
  pointer-events: none;
  width: 88px;
  height: 88px;
  border-radius: var(--ec-r-op);
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--ec-node-op);
  color: var(--ec-node-op-ink);
  opacity: 0.9;
  box-shadow: var(--ec-drag-shadow);
}
.ec-ghost.ec-source {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
  border-radius: var(--ec-r-fill);
}
.ec-ghost svg {
  width: 26px;
  height: 26px;
}

/* orientamento orizzontale (bordi alto e basso): le sezioni scorrono in fila — righe 329-338 */
.ec-panel.ec-horiz .ec-tb-inner {
  flex-direction: row;
  overflow-x: auto;
  overflow-y: hidden;
  gap: 14px;
  align-items: stretch;
}
.ec-panel.ec-horiz .ec-tb-head {
  flex-direction: column;
  align-items: flex-start;
  justify-content: flex-start;
  gap: 8px;
  margin: 0;
}
.ec-panel.ec-horiz .ec-tb-sec {
  border-top: 0;
  border-left: 1px solid var(--ec-panel-border);
  padding: 0 0 0 12px;
}
.ec-panel.ec-horiz .ec-tb-sec-head {
  pointer-events: none;
}
.ec-panel.ec-horiz .ec-chev {
  display: none;
}
.ec-panel.ec-horiz .ec-tb-sec-body {
  max-height: none;
  opacity: 1;
  padding: 0;
  grid-template-columns: none;
  grid-template-rows: repeat(2, auto);
  grid-auto-flow: column;
  grid-auto-columns: 66px;
}
.ec-panel.ec-horiz .ec-tb-upload {
  grid-column: auto;
  grid-row: span 2;
  flex-direction: column;
  width: 66px;
  padding: 8px 4px;
  text-align: center;
}

@media (prefers-reduced-motion: reduce) {
  .ec-workspace,
  .ec-panel,
  .ec-notch,
  .ec-tb-sec-body,
  .ec-pal-chip {
    transition: none;
  }
  .ec-panel.ec-tab-in > .ec-panel-body {
    animation: none;
  }
}
```

### `src/etl-canvas/panels/ui-icons.tsx`

73 righe

```tsx
/** Icone dell'interfaccia dei pannelli (prototipo, righe 4790-4793, 813-818, 893): tracciati statici, nessun dato dell'utente. */
import type { Side } from "../../etl-store";

function Svg(props: { children: React.ReactNode; size?: number }) {
  return (
    <svg
      viewBox="0 0 24 24"
      width={props.size}
      height={props.size}
      fill="none"
      stroke="currentColor"
      strokeWidth={2.2}
      strokeLinecap="round"
      strokeLinejoin="round"
      aria-hidden="true"
    >
      {props.children}
    </svg>
  );
}

export function ToolsIcon() {
  return (
    <Svg>
      <rect x="4" y="4" width="6.5" height="6.5" rx="1.5" />
      <rect x="13.5" y="4" width="6.5" height="6.5" rx="1.5" />
      <rect x="4" y="13.5" width="6.5" height="6.5" rx="1.5" />
      <rect x="13.5" y="13.5" width="6.5" height="6.5" rx="1.5" />
    </Svg>
  );
}

export function InspectorIcon() {
  return (
    <Svg>
      <line x1="4" y1="7" x2="20" y2="7" />
      <line x1="4" y1="17" x2="20" y2="17" />
      <circle cx="9" cy="7" r="2.4" />
      <circle cx="15" cy="17" r="2.4" />
    </Svg>
  );
}

export function UploadIcon() {
  return (
    <Svg>
      <path d="M12 16V4" />
      <path d="M7 9l5-5 5 5" />
      <path d="M5 20h14" />
    </Svg>
  );
}

export function ChevronIcon() {
  return (
    <Svg>
      <polyline points="9 18 15 12 9 6" />
    </Svg>
  );
}

/** Freccia di chiusura: indica sempre il bordo verso cui il pannello rientra (riga 285-289). */
export function CloseArrow(props: { side: Side }) {
  const turn = { left: 0, right: 180, top: 90, bottom: -90 }[props.side];
  return (
    <span className="ec-close-arrow" style={{ transform: `rotate(${turn}deg)` }}>
      <Svg size={13}>
        <polyline points="15 6 9 12 15 18" />
      </Svg>
    </span>
  );
}
```

### `src/etl-canvas/seed.ts`

68 righe

```ts
/**
 * Scena iniziale del prototipo (`init`, righe 5070-5083 di
 * docs/prototype/isa-fusion-prototype.html): Vendite 2026, Filtra Righe,
 * Unisci, Ordina, Esporta, con posizioni identiche e nessun cavo. Serve
 * solo in sviluppo (`?seed=prototype`).
 */
import { META, defaultParams } from "../etl-core";
import type { Card, ColumnDef, ComponentId } from "../etl-core";
import { initialState } from "../etl-store";
import type { EtlState } from "../etl-store";

/** `SCHEMA` del prototipo (righe 2405-2414). Il tipo "object" della colonna `categoria` è "stringa" in etl-core. */
const SCHEMA: ColumnDef[] = [
  { name: "id", type: "integer", values: Array.from({ length: 30 }, (_, i) => String(i + 1)) },
  { name: "cliente", type: "stringa", values: ["Acme", "Borealis", "Cedro", "Delta", "Eureka"] },
  { name: "regione", type: "stringa", values: ["Nord", "Centro", "Sud", "Isole"] },
  { name: "categoria", type: "stringa", values: ["Hardware", "Software", "Servizi", "Consulenza"] },
  { name: "stato", type: "stringa", values: ["Aperto", "In corso", "Chiuso", "Annullato"] },
  {
    name: "quantita",
    type: "integer",
    values: ["1", "2", "3", "5", "8", "10", "12", "20", "25", "50"],
  },
  { name: "importo", type: "numerico", values: ["45.2", "80", "120.5", "300", "512.9", "1049"] },
  {
    name: "data",
    type: "data",
    values: ["2026-01-03", "2026-01-04", "2026-01-05", "2026-01-06", "2026-02-01"],
  },
];

function op(id: string, type: ComponentId, x: number, y: number): Card {
  return {
    id,
    kind: "op",
    components: [type],
    params: [defaultParams(type)],
    name: META[type].label,
    x,
    y,
  };
}

export function prototypeScene(): EtlState {
  const ds: Card = {
    id: "ds1",
    kind: "dataset",
    components: ["dataset"],
    params: [{ ...defaultParams("dataset"), path: "vendite_2026.csv", columns: SCHEMA }],
    name: META.dataset.label,
    x: 26,
    y: 182,
  };
  const cards = [
    ds,
    op("op-filter", "filter", 260, 52),
    op("op-join", "join", 260, 182),
    op("op-sort", "sort", 260, 338),
    op("op-export", "exportOp", 442, 338),
  ];
  const base = initialState();
  return {
    ...base,
    graph: { cards: Object.fromEntries(cards.map((c) => [c.id, c])), links: [] },
    counters: { ...base.counters, ds: 1 },
  };
}
```

### `src/etl-canvas/tokens.css`

144 righe

```css
/*
 * Token del canvas ETL (livello 3 — di componente). Ambito: SOLO il
 * contenitore `.etl-canvas` e lo spazio di lavoro `.ec-workspace` che lo
 * contiene (pannelli e cassetta, Fase 6a).
 *
 * Usano SOLO token semantici (`--isa-*`, definiti per tema e per modo in
 * src/theme/themes/*.css), mai colori o misure scritte a mano: il modo
 * chiaro/scuro e il tema (`data-theme`) arrivano da `<html>` per eredità.
 * Nel tema predefinito, chiaro: valori IDENTICI al prototipo
 * (docs/prototype/isa-fusion-prototype.html); ogni token riporta la riga da
 * cui viene. Scuro: progettato (il prototipo non lo definisce) a partire dal
 * tema scuro dell'app, con la stessa tinta d'accento #6C63FF; l'accento resta
 * sui riempimenti, mentre testi ed elementi sottili usano una tinta più chiara.
 */
.etl-canvas,
.ec-workspace {
  /* sfondo della pagina dietro al canvas — riga 10 */
  --ec-bg: var(--isa-surface-base);
  /* superficie del canvas (stage) — riga 623 */
  --ec-stage: var(--isa-stage);
  /* vetro dei controlli e della minimappa — riga 11 */
  --ec-surface-strong: var(--isa-surface-raised);
  /* bordo del vetro — riga 12 */
  --ec-panel-border: var(--isa-border);
  /* testo — riga 13 */
  --ec-ink: var(--isa-text);
  /* testo secondario — riga 14 */
  --ec-muted: var(--isa-text-muted);
  /* stato vuoto (elemento nuovo, assente nel prototipo): testo con contrasto ≥ 4,5:1 */
  --ec-empty-ink: var(--isa-text-secondary);
  /* accento — riga 15 */
  --ec-accent: var(--isa-accent);
  /* accento come colore di testo (etichette, "Adatta") — righe 15, 149, 649: coincide con l'accento */
  --ec-accent-text: var(--isa-accent-text);
  /* accento tenue — riga 16 */
  --ec-accent-soft: var(--isa-accent-soft);
  /* accento medio — riga 17 */
  --ec-accent-soft-2: var(--isa-accent-soft-2);
  /* nodo lavorazione (chip tinto) — riga 631 */
  --ec-node-op: var(--isa-tint);
  /* icona del nodo lavorazione — riga 631 */
  --ec-node-op-ink: var(--isa-tint-ink);
  /* bordo del nodo lavorazione: il prototipo non ne ha (trasparente) */
  --ec-node-op-border: var(--isa-tint-border);
  /* nodo dataset e output (chip pieno) — riga 634 */
  --ec-node-fill: var(--isa-dataset-fill);
  /* icona sul chip pieno — riga 634 */
  --ec-node-fill-ink: var(--isa-text-on-accent);
  /* opacità dell'output — riga 637 */
  --ec-output-opacity: 0.92;
  /* output parziale: fondo del nodo — riga 658 */
  --ec-split-bg: var(--isa-split-bg);
  /* fetta vuota: fondo e icona — riga 665 */
  --ec-split-empty: var(--isa-split-empty);
  --ec-split-empty-ink: var(--isa-split-empty-ink);
  /* separatore tra le fette — riga 666 */
  --ec-split-line: var(--isa-split-line);
  /* indicatore ambra e suo bordo — riga 185 */
  --ec-warn: var(--isa-warning);
  --ec-warn-ring: var(--isa-warning-ring);
  /* contorno di selezione — riga 508 */
  --ec-select: var(--isa-select);
  --ec-select-ring: var(--isa-ring-select);
  /* contorno interno del nodo lavorazione (colore: --ec-node-op-border) */
  --ec-node-op-outline: var(--isa-outline-node-op);
  /* cavo e suoi capi — righe 1404, 1047 */
  --ec-link: var(--isa-link);
  --ec-link-dot-fill: var(--isa-accent);
  --ec-link-dot-op: var(--isa-link-dot-tint);
  /* flusso nei cavi — riga 1517 */
  --ec-flow: var(--isa-flow);
  /* minimappa — righe 159-161 */
  --ec-mm-node: var(--isa-mm-node);
  --ec-mm-node-ds: var(--isa-mm-node-ds);
  --ec-mm-view-line: var(--isa-mm-view-line);
  --ec-mm-view-bg: var(--isa-mm-view-bg);
  /* raggi — righe 631 (nodo op), 634 (nodo pieno), 623 (stage), 154 (minimappa), 140 (controlli) */
  --ec-r-op: var(--isa-radius-node-op);
  --ec-r-fill: var(--isa-radius-node-fill);
  /* derivati dal raggio del tema (`--radius`): nel tema predefinito 20 e 14 px, come nel prototipo */
  --ec-r-stage: var(--isa-radius-panel);
  --ec-r-minimap: var(--isa-radius-control);
  --ec-r-pill: var(--isa-radius-pill);
  /* minimappa: nodo (2 px) e riquadro visibile (4 px) */
  --ec-r-xs: var(--isa-radius-xs);
  --ec-r-sm: var(--isa-radius-sm);
  /* ombra del vetro — righe 141, 155 */
  --ec-glass-shadow: var(--isa-shadow-glass);
  /* sfocatura del vetro — righe 140, 154 */
  --ec-glass-blur: var(--isa-blur-glass);
  /* carattere — riga 6 (link) e 21 (body): quello dell'app (`--font-sans`, src/styles.css) */
  --ec-font: var(--font-sans);
}

/* gesti (Fase 5): esiti del rilascio, cavo da inserire, riquadro, cavo provvisorio, porte, conferma */
.etl-canvas,
.ec-workspace {
  --ec-drop-merge: var(--isa-drop-merge);
  --ec-drop-merge-ring: var(--isa-ring-drop-merge);
  --ec-drop-link-ring: var(--isa-ring-drop-link);
  --ec-drop-link: var(--isa-drop-link);
  --ec-drop-link-reverse: var(--isa-drop-link-reverse);
  --ec-drop-displace: var(--isa-drop-displace);
  --ec-drop-reject: var(--isa-drop-reject);
  --ec-link-insert: var(--isa-drop-insert);
  --ec-doomed: var(--isa-doomed);
  --ec-marquee-line: var(--isa-marquee-line);
  --ec-marquee-fill: var(--isa-marquee-fill);
  --ec-temp-link: var(--isa-temp-link);
  --ec-temp-link-muted: var(--isa-temp-link-muted);
  --ec-port-fill: var(--isa-port-fill);
  --ec-port-line: var(--isa-port-line);
  --ec-danger: var(--isa-danger);
  --ec-text-on-danger: var(--isa-text-on-danger);
  --ec-drag-shadow: var(--isa-shadow-drag);
  --ec-overlay-shadow: var(--isa-shadow-overlay);
}

/*
 * Famiglie di operazioni: il nodo lavorazione prende il colore della propria
 * famiglia (filtra-ordina, trasforma, merge-union, output). Nel tema
 * predefinito le quattro famiglie coincidono con la tinta unica del prototipo.
 */
.etl-canvas [data-family="filter"],
.ec-workspace [data-family="filter"] {
  --ec-node-op: var(--isa-op-filter-soft);
  --ec-node-op-ink: var(--isa-op-filter);
}
.etl-canvas [data-family="transform"],
.ec-workspace [data-family="transform"] {
  --ec-node-op: var(--isa-op-transform-soft);
  --ec-node-op-ink: var(--isa-op-transform);
}
.etl-canvas [data-family="merge"],
.ec-workspace [data-family="merge"] {
  --ec-node-op: var(--isa-op-merge-soft);
  --ec-node-op-ink: var(--isa-op-merge);
}
.etl-canvas [data-family="output"],
.ec-workspace [data-family="output"] {
  --ec-node-op: var(--isa-op-output-soft);
  --ec-node-op-ink: var(--isa-op-output);
}
```

### `src/etl-canvas/transitions.ts`

109 righe

```ts
/**
 * Transizione morbida quando il percorso di un cavo cambia in modo
 * discreto: calcolo puro. Non ricalcola percorsi: interpola o dissolve
 * percorsi già calcolati da etl-layout.
 */

export interface Pt {
  readonly x: number;
  readonly y: number;
}

/**
 * Durata della transizione: 380 ms, la stessa di `.world.easing`
 * (prototipo, riga 136). Andamento lineare nel tempo, per avere
 * interpolazioni esatte e verificabili.
 */
export const TRANSITION_MS = 380;
/** Due percorsi con punti a meno di questa distanza sono lo stesso percorso. */
export const SAME_EPS = 0.01;

export function samePoints(a: readonly Pt[], b: readonly Pt[], eps = SAME_EPS): boolean {
  if (a.length !== b.length) return false;
  for (let i = 0; i < a.length; i++) {
    const p = a[i] as Pt;
    const q = b[i] as Pt;
    if (Math.abs(p.x - q.x) > eps || Math.abs(p.y - q.y) > eps) return false;
  }
  return true;
}

/** Avanzamento lineare 0..1. Con durata 0 la transizione è già finita. */
export function progress(elapsedMs: number, durationMs = TRANSITION_MS): number {
  if (!(durationMs > 0)) return 1;
  return Math.max(0, Math.min(1, elapsedMs / durationMs));
}

/** Punto per punto: `from + (to - from) * t`. I due percorsi devono avere lo stesso numero di punti. */
export function interpolatePoints(from: readonly Pt[], to: readonly Pt[], t: number): Pt[] {
  if (from.length !== to.length) {
    throw new Error(`interpolatePoints: ${from.length} punti contro ${to.length}`);
  }
  const k = Math.max(0, Math.min(1, t));
  if (k === 1) return to.map((p) => ({ x: p.x, y: p.y }));
  return from.map((p, i) => {
    const q = to[i] as Pt;
    return { x: p.x + (q.x - p.x) * k, y: p.y + (q.y - p.y) * k };
  });
}

/** Dissolvenza incrociata lineare: le due opacità sommano sempre a 1. */
export function crossfade(t: number): { readonly old: number; readonly next: number } {
  const k = Math.max(0, Math.min(1, t));
  return { old: 1 - k, next: k };
}

export type Plan =
  | { readonly kind: "none" }
  | { readonly kind: "morph"; readonly from: readonly Pt[]; readonly to: readonly Pt[] }
  | { readonly kind: "fade"; readonly from: readonly Pt[]; readonly to: readonly Pt[] };

export interface PlanInput {
  /** Percorso attualmente mostrato, o null se il cavo è nuovo. */
  readonly prev: readonly Pt[] | null;
  readonly next: readonly Pt[];
  /** Un gesto di trascinamento è in corso: già continuo, niente transizione. */
  readonly gesturing: boolean;
  /** Preferenza di movimento ridotto: transizioni istantanee. */
  readonly reduced: boolean;
}

/**
 * Cosa fare quando il percorso di un cavo passa da `prev` a `next`:
 * niente (cavo nuovo, identico, durante un gesto o con movimento ridotto),
 * interpolare (stesso numero di punti) o dissolvere (numero diverso).
 */
export function planTransition(input: PlanInput): Plan {
  const { prev, next, gesturing, reduced } = input;
  if (!prev || gesturing || reduced) return { kind: "none" };
  if (samePoints(prev, next)) return { kind: "none" };
  return prev.length === next.length
    ? { kind: "morph", from: prev, to: next }
    : { kind: "fade", from: prev, to: next };
}

/** Stato visivo di una transizione a `elapsedMs` dal suo inizio. */
export type Visual =
  | { readonly kind: "points"; readonly pts: readonly Pt[]; readonly done: boolean }
  | {
      readonly kind: "fade";
      readonly old: number;
      readonly next: number;
      readonly done: boolean;
    };

export function sampleTransition(
  plan: Plan,
  elapsedMs: number,
  durationMs = TRANSITION_MS,
): Visual | null {
  if (plan.kind === "none") return null;
  const t = progress(elapsedMs, durationMs);
  const done = t >= 1;
  if (plan.kind === "morph") {
    return { kind: "points", pts: interpolatePoints(plan.from, plan.to, t), done };
  }
  const f = crossfade(t);
  return { kind: "fade", old: f.old, next: f.next, done };
}
```

### `src/etl-canvas/view.ts`

140 righe

```ts
/**
 * Geometria della vista (pan, zoom, Adatta, minimappa): funzioni pure, con
 * i numeri del prototipo (docs/prototype/isa-fusion-prototype.html).
 */
import type { Card } from "../etl-core";
import { CARD, LABEL_H } from "../etl-layout";
import type { Point, Size } from "../etl-layout";
import { ZOOM_MAX, ZOOM_MIN } from "../etl-store";
import type { View } from "../etl-store";

/** Fattore dei pulsanti + e − (prototipo, righe 4113-4114). */
export const ZOOM_STEP = 1.2;
/** Sensibilità della rotella con Cmd/Ctrl (riga 4093). */
export const WHEEL_ZOOM_RATE = 0.0022;
/** Margine di "Adatta" (riga 4127) e zoom massimo che può raggiungere (riga 4130). */
export const FIT_PAD = 48;
export const FIT_ZOOM_MAX = 1.25;
/** Minimappa: dimensioni e margine (righe 152-156, 4144-4145). */
export const MM_W = 168;
export const MM_H = 104;
export const MM_PAD = 20;
/** Lato minimo di un nodo nella minimappa (riga 4148). */
export const MM_NODE_MIN = 3;

export function clampZoom(z: number): number {
  return Math.max(ZOOM_MIN, Math.min(ZOOM_MAX, z));
}

/** Zoom a `z` mantenendo fermo il punto dello schermo (`px`, `py`). Prototipo `zoomAt`, righe 4104-4110. */
export function zoomAt(view: View, px: number, py: number, z: number): View {
  const zoom = clampZoom(z);
  return {
    x: px - (px - view.x) * (zoom / view.zoom),
    y: py - (py - view.y) * (zoom / view.zoom),
    zoom,
  };
}

/** Zoom attorno al centro dell'area visibile. */
export function zoomCentered(view: View, size: Size, z: number): View {
  return zoomAt(view, size.w / 2, size.h / 2, z);
}

/** Punto dello schermo (relativo all'area) → coordinate del mondo. Riga 943. */
export function toWorld(view: View, sx: number, sy: number): Point {
  return { x: (sx - view.x) / view.zoom, y: (sy - view.y) / view.zoom };
}

/** Ingombro di un insieme di nodi (quadrato + etichetta), o null se vuoto. */
export function bounds(
  cards: readonly Pick<Card, "x" | "y">[],
): { x1: number; y1: number; x2: number; y2: number } | null {
  if (cards.length === 0) return null;
  let x1 = Infinity;
  let y1 = Infinity;
  let x2 = -Infinity;
  let y2 = -Infinity;
  for (const c of cards) {
    x1 = Math.min(x1, c.x);
    y1 = Math.min(y1, c.y);
    x2 = Math.max(x2, c.x + CARD);
    y2 = Math.max(y2, c.y + CARD + LABEL_H);
  }
  return { x1, y1, x2, y2 };
}

/** "Adatta": inquadra tutti i nodi con il margine del prototipo (righe 4111-4131). */
export function fitView(cards: readonly Pick<Card, "x" | "y">[], size: Size): View {
  const b = bounds(cards);
  if (!b) return { x: 0, y: 0, zoom: 1 };
  const zoom = Math.max(
    ZOOM_MIN,
    Math.min(
      FIT_ZOOM_MAX,
      Math.min(size.w / (b.x2 - b.x1 + FIT_PAD * 2), size.h / (b.y2 - b.y1 + FIT_PAD * 2)),
    ),
  );
  return {
    zoom,
    x: (size.w - (b.x2 - b.x1) * zoom) / 2 - b.x1 * zoom,
    y: (size.h - (b.y2 - b.y1) * zoom) / 2 - b.y1 * zoom,
  };
}

/** Rettangolo del mondo visibile nell'area di dimensioni `size`. */
export function visibleWorld(
  view: View,
  size: Size,
): { x1: number; y1: number; x2: number; y2: number } {
  const x1 = -view.x / view.zoom;
  const y1 = -view.y / view.zoom;
  return { x1, y1, x2: x1 + size.w / view.zoom, y2: y1 + size.h / view.zoom };
}

export interface MinimapFrame {
  readonly x1: number;
  readonly y1: number;
  readonly k: number;
  readonly ox: number;
  readonly oy: number;
}

/** Riquadro della minimappa: la scala e l'origine (prototipo `renderMinimap`, righe 4133-4151). */
export function minimapFrame(
  cards: readonly Pick<Card, "x" | "y">[],
  view: View,
  size: Size,
): MinimapFrame {
  const v = visibleWorld(view, size);
  let x1 = v.x1;
  let y1 = v.y1;
  let x2 = v.x2;
  let y2 = v.y2;
  for (const c of cards) {
    x1 = Math.min(x1, c.x);
    y1 = Math.min(y1, c.y);
    x2 = Math.max(x2, c.x + CARD);
    y2 = Math.max(y2, c.y + CARD);
  }
  x1 -= MM_PAD;
  y1 -= MM_PAD;
  x2 += MM_PAD;
  y2 += MM_PAD;
  const k = Math.min(MM_W / (x2 - x1), MM_H / (y2 - y1));
  return { x1, y1, k, ox: (MM_W - (x2 - x1) * k) / 2, oy: (MM_H - (y2 - y1) * k) / 2 };
}

/** Vista che porta al centro dell'area il punto (`mx`, `my`) della minimappa (righe 4152-4165). */
export function viewFromMinimap(
  frame: MinimapFrame,
  view: View,
  size: Size,
  mx: number,
  my: number,
): View {
  const wx = frame.x1 + (mx - frame.ox) / frame.k;
  const wy = frame.y1 + (my - frame.oy) / frame.k;
  return { zoom: view.zoom, x: size.w / 2 - wx * view.zoom, y: size.h / 2 - wy * view.zoom };
}
```

