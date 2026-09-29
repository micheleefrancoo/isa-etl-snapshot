# 01e-etl-canvas-b.md

File in questo blocco:

- `src/etl-canvas/tokens.css`
- `src/etl-canvas/view.ts`

---

### `src/etl-canvas/tokens.css`

123 righe

```css
/*
 * Token del canvas ETL (Fase 4a). Ambito: SOLO il contenitore `.etl-canvas`.
 *
 * Tema chiaro: valori IDENTICI al prototipo
 * (docs/prototype/isa-fusion-prototype.html); ogni token riporta la riga da
 * cui viene. Tema scuro: progettato (il prototipo non lo definisce) a
 * partire dal tema scuro dell'app (src/styles.css, `.dark`), con la stessa
 * tinta d'accento #6C63FF. Il tema segue la classe `.dark` sull'elemento
 * radice, la stessa che imposta src/lib/theme.tsx.
 */
.etl-canvas {
  /* sfondo della pagina dietro al canvas — riga 10 */
  --ec-bg: #f5f3ee;
  /* superficie del canvas (stage) — riga 623 */
  --ec-stage: rgba(255, 255, 255, 0.32);
  /* vetro dei controlli e della minimappa — riga 11 */
  --ec-surface-strong: rgba(255, 255, 255, 0.92);
  /* bordo del vetro — riga 12 */
  --ec-panel-border: rgba(38, 36, 32, 0.06);
  /* testo — riga 13 */
  --ec-ink: #262420;
  /* testo secondario — riga 14 */
  --ec-muted: #847e74;
  /* stato vuoto (elemento nuovo, assente nel prototipo): testo con contrasto ≥ 4,5:1 */
  --ec-empty-ink: #6a645a;
  /* accento — riga 15 */
  --ec-accent: #6c63ff;
  /* accento come colore di testo (etichette, "Adatta") — righe 15, 149, 649: coincide con l'accento */
  --ec-accent-text: #6c63ff;
  /* accento tenue — riga 16 */
  --ec-accent-soft: rgba(108, 99, 255, 0.16);
  /* accento medio — riga 17 */
  --ec-accent-soft-2: rgba(108, 99, 255, 0.34);
  /* nodo lavorazione (chip tinto) — riga 631 */
  --ec-node-op: #e1dcf0;
  /* icona del nodo lavorazione — riga 631 */
  --ec-node-op-ink: #6c63ff;
  /* bordo del nodo lavorazione: il prototipo non ne ha (trasparente) */
  --ec-node-op-border: rgba(0, 0, 0, 0);
  /* nodo dataset e output (chip pieno) — riga 634 */
  --ec-node-fill: #6c63ff;
  /* icona sul chip pieno — riga 634 */
  --ec-node-fill-ink: #ffffff;
  /* opacità dell'output — riga 637 */
  --ec-output-opacity: 0.92;
  /* output parziale: fondo del nodo — riga 658 */
  --ec-split-bg: #efedf7;
  /* fetta vuota: fondo e icona — riga 665 */
  --ec-split-empty: #e6e3f5;
  --ec-split-empty-ink: #8f88c7;
  /* separatore tra le fette — riga 666 */
  --ec-split-line: rgba(108, 99, 255, 0.45);
  /* indicatore ambra e suo bordo — riga 185 */
  --ec-warn: #e0a23b;
  --ec-warn-ring: #f7f5f1;
  /* contorno di selezione — riga 508 */
  --ec-select: #6c63ff;
  /* cavo e suoi capi — righe 1404, 1047 */
  --ec-link: rgba(108, 99, 255, 0.34);
  --ec-link-dot-fill: #6c63ff;
  --ec-link-dot-op: #e1dcf0;
  /* minimappa — righe 159-161 */
  --ec-mm-node: #cfc9ef;
  --ec-mm-node-ds: #6c63ff;
  --ec-mm-view-line: #6c63ff;
  --ec-mm-view-bg: rgba(108, 99, 255, 0.08);
  /* raggi — righe 631 (nodo op), 634 (nodo pieno), 623 (stage), 154 (minimappa), 140 (controlli) */
  --ec-r-op: 22px;
  --ec-r-fill: 26px;
  --ec-r-stage: 20px;
  --ec-r-minimap: 14px;
  --ec-r-pill: 999px;
  /* ombra del vetro — righe 141, 155 */
  --ec-glass-shadow: 0 10px 24px -14px rgba(38, 36, 32, 0.4);
  /* sfocatura del vetro — righe 140, 154 */
  --ec-glass-blur: 16px;
  /* carattere — riga 6 (link) e 21 (body) */
  --ec-font: "Manrope Variable", "Manrope", system-ui, sans-serif;
}

/*
 * Tema scuro. Derivazione dai ruoli dell'app (`.dark` in src/styles.css):
 * sfondo oklch(0.19 0.008 260) ≈ #17181d; testo oklch(0.96 0.004 250) ≈
 * #f1f2f5; testo secondario oklch(0.75 0.008 260) ≈ #a9abb3; bordi
 * oklch(1 0 0 / 11%); ambra `--warning` oklch(0.8 0.12 80) ≈ #e8b34f.
 * L'accento resta #6c63ff sui riempimenti; per i testi e per gli elementi
 * sottili su fondo scuro si usa una tinta più chiara della stessa famiglia.
 */
.dark .etl-canvas {
  --ec-bg: #17181d;
  --ec-stage: rgba(255, 255, 255, 0.04);
  --ec-surface-strong: rgba(36, 37, 45, 0.92);
  --ec-panel-border: rgba(255, 255, 255, 0.11);
  --ec-ink: #f1f2f5;
  --ec-muted: #a9abb3;
  --ec-empty-ink: #a9abb3;
  --ec-accent: #6c63ff;
  --ec-accent-text: #a8a3ff;
  --ec-accent-soft: rgba(108, 99, 255, 0.28);
  --ec-accent-soft-2: rgba(108, 99, 255, 0.5);
  --ec-node-op: #3a3670;
  --ec-node-op-ink: #d0ccff;
  --ec-node-op-border: #7f78e6;
  --ec-node-fill: #6c63ff;
  --ec-node-fill-ink: #ffffff;
  --ec-output-opacity: 0.92;
  --ec-split-bg: #2a2843;
  --ec-split-empty: #2e2c4d;
  --ec-split-empty-ink: #a8a3e6;
  --ec-split-line: rgba(168, 163, 255, 0.55);
  --ec-warn: #e8b34f;
  --ec-warn-ring: #17181d;
  --ec-select: #a8a3ff;
  --ec-link: #7f78e6;
  --ec-link-dot-fill: #6c63ff;
  --ec-link-dot-op: #a8a3ff;
  --ec-mm-node: #7b74d9;
  --ec-mm-node-ds: #a8a3ff;
  --ec-mm-view-line: #a8a3ff;
  --ec-mm-view-bg: rgba(168, 163, 255, 0.12);
  --ec-glass-shadow: 0 10px 24px -14px rgba(0, 0, 0, 0.6);
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

