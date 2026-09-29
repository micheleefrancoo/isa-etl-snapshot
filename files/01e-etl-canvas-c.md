# 01e-etl-canvas-c.md

File in questo blocco:

- `src/etl-canvas/loop.ts`
- `src/etl-canvas/model.ts`
- `src/etl-canvas/motion.tsx`
- `src/etl-canvas/seed.ts`
- `src/etl-canvas/tokens.css`
- `src/etl-canvas/transitions.ts`
- `src/etl-canvas/view.ts`

---

### `src/etl-canvas/loop.ts`

118 righe

```ts
/**
 * Un solo ciclo requestAnimationFrame per tutto il canvas. Si ferma
 * quando la scheda è nascosta e quando nessun compito ha nulla da
 * animare; con `prefers-reduced-motion` non parte mai (i compiti
 * mostrano lo stato finale con `settle`). Tutto ciò che tocca il browser
 * passa da `LoopEnv`, quindi nei test si sostituisce.
 */

export interface LoopEnv {
  raf(cb: () => void): number;
  caf(id: number): void;
  /** Orologio in millisecondi (nel browser `performance.now`). */
  now(): number;
  hidden(): boolean;
  reducedMotion(): boolean;
  onVisibilityChange(cb: () => void): () => void;
  onReducedMotionChange(cb: () => void): () => void;
}

export interface Task {
  /** Disegna il frame a `now`; `true` se ha ancora qualcosa da animare. */
  frame(now: number): boolean;
  /** Mostra lo stato finale, senza movimento (movimento ridotto). */
  settle(): void;
}

export interface Loop {
  add(task: Task): () => void;
  /** C'è (forse) lavoro nuovo: se serve, il ciclo riparte. */
  wake(): void;
  running(): boolean;
  dispose(): void;
}

export function createLoop(env: LoopEnv): Loop {
  const tasks = new Set<Task>();
  let id: number | null = null;

  const cancel = (): void => {
    if (id !== null) {
      env.caf(id);
      id = null;
    }
  };

  const settleAll = (): void => {
    for (const t of [...tasks]) t.settle();
  };

  const tick = (): void => {
    id = null;
    const now = env.now();
    let busy = false;
    for (const t of [...tasks]) if (t.frame(now)) busy = true;
    if (busy && !env.hidden() && !env.reducedMotion()) id = env.raf(tick);
  };

  const schedule = (): void => {
    if (id !== null || tasks.size === 0) return;
    if (env.hidden()) return;
    if (env.reducedMotion()) {
      settleAll();
      return;
    }
    id = env.raf(tick);
  };

  const offVisibility = env.onVisibilityChange(() => {
    if (env.hidden()) cancel();
    else schedule();
  });
  const offMotion = env.onReducedMotionChange(() => {
    if (env.reducedMotion()) {
      cancel();
      settleAll();
    } else schedule();
  });

  return {
    add(task) {
      tasks.add(task);
      schedule();
      return () => {
        tasks.delete(task);
        if (tasks.size === 0) cancel();
      };
    },
    wake: schedule,
    running: () => id !== null,
    dispose() {
      cancel();
      tasks.clear();
      offVisibility();
      offMotion();
    },
  };
}

/** Ambiente reale. Va creato solo nel browser (in un effetto), mai durante il rendering. */
export function browserEnv(): LoopEnv {
  const mq = window.matchMedia("(prefers-reduced-motion: reduce)");
  return {
    raf: (cb) => requestAnimationFrame(() => cb()),
    caf: (id) => cancelAnimationFrame(id),
    now: () => performance.now(),
    hidden: () => document.hidden,
    reducedMotion: () => mq.matches,
    onVisibilityChange(cb) {
      document.addEventListener("visibilitychange", cb);
      return () => document.removeEventListener("visibilitychange", cb);
    },
    onReducedMotionChange(cb) {
      mq.addEventListener("change", cb);
      return () => mq.removeEventListener("change", cb);
    },
  };
}
```

### `src/etl-canvas/model.ts`

76 righe

```ts
/**
 * Dal grafo di etl-core a ciò che il canvas disegna: classi e icone di ogni
 * nodo, fette di un output parziale. Funzioni pure.
 */
import type { Card, ComponentId } from "../etl-core";

/** Classe di conteggio per la disposizione delle icone (prototipo `countClass`, righe 982-985). */
export function countClass(n: number): string {
  if (n <= 1) return "count-1";
  if (n === 2) return "count-2";
  if (n <= 3) return "count-3";
  if (n <= 6) return "count-6";
  return "count-many";
}

/** Un output è "parziale" se attende altre tabelle (riga 1620). */
export function isPartial(card: Card): boolean {
  return (
    card.kind === "dataset" &&
    card.isOutput === true &&
    card.capacity !== undefined &&
    card.capacity > 1 &&
    (card.filled ?? 0) < card.capacity
  );
}

export interface Slice {
  readonly full: boolean;
}

/** Le fette di un output parziale, riempite da sinistra (righe 1622-1627). */
export function slicesOf(card: Card): Slice[] {
  const n = card.capacity ?? 0;
  const f = card.filled ?? 0;
  return Array.from({ length: n }, (_, i) => ({ full: i < f }));
}

export interface NodeView {
  readonly id: string;
  readonly card: Card;
  /** Classi del contenitore del nodo. */
  readonly className: string;
  readonly iconClass: string;
  readonly partial: boolean;
  readonly slices: readonly Slice[];
  readonly icons: readonly ComponentId[];
  readonly warn: string | null;
  readonly selected: boolean;
}

/** Classi del nodo (prototipo `createCardEl`, righe 1003-1004, più `partial`, `warn`, `selected`). */
export function nodeView(card: Card, warn: string | null, selected: boolean): NodeView {
  const combined = card.kind === "op" && card.components.length > 1;
  const partial = isPartial(card);
  const classes = ["ec-card"];
  if (card.kind === "dataset") classes.push("ec-dataset");
  if (card.kind === "dataset" && card.isOutput) classes.push("ec-output");
  if (combined) classes.push("ec-combined");
  if (partial) classes.push("ec-partial");
  if (warn) classes.push("ec-warn");
  if (selected) classes.push("ec-selected");
  const icons: ComponentId[] =
    card.kind === "dataset" && card.isOutput ? ["dataset"] : [...card.components];
  return {
    id: card.id,
    card,
    className: classes.join(" "),
    iconClass: partial ? "ec-icon-wrap ec-split" : `ec-icon-wrap ec-${countClass(icons.length)}`,
    partial,
    slices: partial ? slicesOf(card) : [],
    icons,
    warn,
    selected,
  };
}
```

### `src/etl-canvas/motion.tsx`

10 righe

```tsx
import { createContext, useContext } from "react";
import type { MotionEngine } from "./engine";

/** Il motore delle animazioni, raggiungibile da cavi e nodi per registrare i propri elementi. */
export const MotionContext = createContext<MotionEngine | null>(null);

export function useMotion(): MotionEngine | null {
  return useContext(MotionContext);
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

127 righe

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
  --ec-bg: var(--isa-bg);
  /* superficie del canvas (stage) — riga 623 */
  --ec-stage: var(--isa-stage);
  /* vetro dei controlli e della minimappa — riga 11 */
  --ec-surface-strong: var(--isa-surface-strong);
  /* bordo del vetro — riga 12 */
  --ec-panel-border: var(--isa-panel-border);
  /* testo — riga 13 */
  --ec-ink: var(--isa-ink);
  /* testo secondario — riga 14 */
  --ec-muted: var(--isa-muted);
  /* stato vuoto (elemento nuovo, assente nel prototipo): testo con contrasto ≥ 4,5:1 */
  --ec-empty-ink: var(--isa-empty-ink);
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
  --ec-node-fill: var(--isa-accent);
  /* icona sul chip pieno — riga 634 */
  --ec-node-fill-ink: var(--isa-on-accent);
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
  --ec-warn: var(--isa-amber);
  --ec-warn-ring: var(--isa-amber-ring);
  /* contorno di selezione — riga 508 */
  --ec-select: var(--isa-select);
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
  --ec-r-op: var(--isa-r-node-op);
  --ec-r-fill: var(--isa-r-node-fill);
  /* derivati dal raggio dell'app (`--radius`, src/styles.css): stessi valori del prototipo, 20 e 14 px */
  --ec-r-stage: calc(var(--radius) + 4px);
  --ec-r-minimap: calc(var(--radius) - 2px);
  --ec-r-pill: 999px;
  /* ombra del vetro — righe 141, 155 */
  --ec-glass-shadow: var(--isa-glass-shadow);
  /* sfocatura del vetro — righe 140, 154 */
  --ec-glass-blur: var(--isa-glass-blur);
  /* carattere — riga 6 (link) e 21 (body): quello dell'app (`--font-sans`, src/styles.css) */
  --ec-font: var(--font-sans);
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
  --ec-bg: var(--isa-bg);
  --ec-stage: var(--isa-stage);
  --ec-surface-strong: var(--isa-surface-strong);
  --ec-panel-border: var(--isa-panel-border);
  --ec-ink: var(--isa-ink);
  --ec-muted: var(--isa-muted);
  --ec-empty-ink: var(--isa-empty-ink);
  --ec-accent: var(--isa-accent);
  --ec-accent-text: var(--isa-accent-text);
  --ec-accent-soft: var(--isa-accent-soft);
  --ec-accent-soft-2: var(--isa-accent-soft-2);
  --ec-node-op: var(--isa-tint);
  --ec-node-op-ink: var(--isa-tint-ink);
  --ec-node-op-border: var(--isa-tint-border);
  --ec-node-fill: var(--isa-accent);
  --ec-node-fill-ink: var(--isa-on-accent);
  --ec-output-opacity: 0.92;
  --ec-split-bg: var(--isa-split-bg);
  --ec-split-empty: var(--isa-split-empty);
  --ec-split-empty-ink: var(--isa-split-empty-ink);
  --ec-split-line: var(--isa-split-line);
  --ec-warn: var(--isa-amber);
  --ec-warn-ring: var(--isa-amber-ring);
  --ec-select: var(--isa-select);
  --ec-link: var(--isa-link);
  --ec-link-dot-fill: var(--isa-accent);
  --ec-link-dot-op: var(--isa-link-dot-tint);
  --ec-flow: var(--isa-flow);
  --ec-mm-node: var(--isa-mm-node);
  --ec-mm-node-ds: var(--isa-mm-node-ds);
  --ec-mm-view-line: var(--isa-mm-view-line);
  --ec-mm-view-bg: var(--isa-mm-view-bg);
  --ec-glass-shadow: var(--isa-glass-shadow);
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

