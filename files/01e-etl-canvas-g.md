# 01e-etl-canvas-g.md

File in questo blocco:

- `src/etl-canvas/tokens.css`
- `src/etl-canvas/transitions.ts`
- `src/etl-canvas/view.ts`

---

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

172 righe

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
/** Spazio dell'area visibile occupato dai widget in sovrimpressione (minimappa, zoom): i nodi lo evitano. */
export interface Insets {
  readonly top: number;
  readonly right: number;
  readonly bottom: number;
  readonly left: number;
}
export const NO_INSETS: Insets = { top: 0, right: 0, bottom: 0, left: 0 };

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

/**
 * "Adatta": inquadra tutti i nodi con il margine del prototipo (righe
 * 4111-4131), dentro l'area meno lo spazio dei widget in sovrimpressione
 * (`insets`: nessuno per impostazione predefinita).
 */
export function fitView(
  cards: readonly Pick<Card, "x" | "y">[],
  size: Size,
  insets: Insets = NO_INSETS,
): View {
  const b = bounds(cards);
  if (!b) return { x: 0, y: 0, zoom: 1 };
  const w = Math.max(1, size.w - insets.left - insets.right);
  const h = Math.max(1, size.h - insets.top - insets.bottom);
  const zoom = Math.max(
    ZOOM_MIN,
    Math.min(
      FIT_ZOOM_MAX,
      Math.min(w / (b.x2 - b.x1 + FIT_PAD * 2), h / (b.y2 - b.y1 + FIT_PAD * 2)),
    ),
  );
  return {
    zoom,
    x: insets.left + (w - (b.x2 - b.x1) * zoom) / 2 - b.x1 * zoom,
    y: insets.top + (h - (b.y2 - b.y1) * zoom) / 2 - b.y1 * zoom,
  };
}

/**
 * Spostamento della vista per la rotella senza Cmd/Ctrl (righe 4092-4102):
 * scorre in verticale (e in orizzontale col trackpad); con Maiusc la rotella
 * verticale scorre in orizzontale.
 */
export function wheelPan(
  view: View,
  e: { readonly deltaX: number; readonly deltaY: number; readonly shiftKey: boolean },
): View {
  if (e.shiftKey && e.deltaX === 0) return { ...view, x: view.x - e.deltaY };
  return { ...view, x: view.x - e.deltaX, y: view.y - e.deltaY };
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

