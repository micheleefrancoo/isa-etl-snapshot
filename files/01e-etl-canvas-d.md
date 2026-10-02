# 01e-etl-canvas-d.md

File in questo blocco:

- `src/etl-canvas/flow.ts`
- `src/etl-canvas/icons.tsx`
- `src/etl-canvas/index.ts`
- `src/etl-canvas/interaction.ts`
- `src/etl-canvas/loop.ts`
- `src/etl-canvas/model.ts`
- `src/etl-canvas/motion.tsx`
- `src/etl-canvas/seed.ts`
- `src/etl-canvas/tokens.css`
- `src/etl-canvas/transitions.ts`
- `src/etl-canvas/view.ts`

---

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
export { createInteractionController, DRAG_THRESHOLD } from "./interaction";
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

### `src/etl-canvas/interaction.ts`

573 righe

```ts
/**
 * Il livello dei gesti del canvas: dal puntatore e dalla tastiera ai comandi
 * di etl-store. TypeScript puro, senza DOM né React: gli eventi arrivano già
 * tradotti (coordinate relative all'area, bersaglio classificato), così si
 * può provare con eventi simulati.
 *
 * Questo livello NON contiene logica di dominio. Ogni gesto chiama una
 * funzione che esiste già:
 *   trascinamento     → store.beginGesture / updateGesture / commitGesture / cancelGesture
 *   esito del rilascio → etl-core `relation`, `insertable`; etl-layout `nodeAt`, `linkAt`
 *   porte             → comando `connect`; geometria `nodePorts`
 *   selezione         → comandi `select` e `inspect`; geometria `nodeRect`
 *   tastiera          → comandi `deleteNodes`, `duplicate`, `moveNodes`, `select`; `undo`/`redo`
 *   anteprima eliminazione → etl-core `nodesRemovedBy`
 * Vedi la tabella nel README.
 */
import { boxCapacity, inputsOf, insertable, nodesRemovedBy, relation } from "../etl-core";
import type { Graph, Link } from "../etl-core";
import { GRID, linkAt, linkKey, nodeAt, nodePorts, nodeRect } from "../etl-layout";
import type { LinkRoutes, Point, Rect } from "../etl-layout";
import type { CommandResult, EtlStore } from "../etl-store";
import { handleCanvasDrop, previewCanvasDrop } from "./drop";
import type { CanvasDropPayload, DropPreview } from "./drop";
import { toWorld } from "./view";

/**
 * Soglia di avvio del trascinamento, in pixel dello schermo. Il prototipo usa
 * 5 (riga 2000) ma la costante non è in etl-layout/constants.ts: si usa la
 * soglia già in uso nell'app (`RESOURCE_DRAG_THRESHOLD` della cassetta
 * attuale, 4 px), uguale a quella del riquadro di selezione del prototipo
 * (riga 4064). Sotto la soglia è un click.
 */
export const DRAG_THRESHOLD = 4;
/** Soglia del riquadro di selezione (prototipo, riga 4064). */
export const MARQUEE_THRESHOLD = 4;
/** Passo singolo delle frecce (prototipo, riga 4638: `GRID` con Maiusc, altrimenti 2). */
export const NUDGE_STEP = 2;

/** Esito mostrato su un nodo durante un trascinamento. */
export type DragOutcome = "merge" | "link" | "link-reverse" | "displace" | "reject";

/** Dove è iniziato il gesto, classificato dal livello che legge il DOM. */
export type DownTarget =
  | { readonly kind: "node"; readonly id: string }
  | { readonly kind: "port"; readonly id: string; readonly side: PortSide }
  | { readonly kind: "background" }
  | { readonly kind: "ignore" };

export type PortSide = "r" | "b" | "l" | "t";
/** Ordine di `PORTS` (etl-layout): destra, sotto, sinistra, sopra. */
const PORT_INDEX: Readonly<Record<PortSide, number>> = { r: 0, b: 1, l: 2, t: 3 };

/** Puntatore in coordinate dell'area (pixel dello schermo relativi in alto a sinistra dell'area). */
export interface PointerInput {
  readonly x: number;
  readonly y: number;
  readonly shiftKey?: boolean;
  readonly button?: number;
}

export interface KeyInput {
  readonly key: string;
  readonly metaKey?: boolean;
  readonly ctrlKey?: boolean;
  readonly shiftKey?: boolean;
  /** Il fuoco è in un campo di testo: le scorciatoie non si applicano. */
  readonly typing?: boolean;
}

export interface ConfirmState {
  readonly ids: readonly string[];
  /** Tutto ciò che sparirebbe: i nodi scelti e gli output a valle (etl-core `nodesRemovedBy`). */
  readonly removed: readonly string[];
  readonly title: string;
  readonly text: string;
}

export interface InteractionUi {
  /** Nodi in movimento. */
  readonly dragging: readonly string[];
  /** Nodo sotto il puntatore e cosa succederebbe al rilascio. */
  readonly drop: { readonly id: string; readonly outcome: DragOutcome } | null;
  /** Chiave (`da|a`) del cavo in cui si inserirebbe la lavorazione. */
  readonly insertLink: string | null;
  /** Cavo provvisorio tirato da una porta (coordinate del mondo). */
  readonly tempLink: { readonly from: Point; readonly to: Point; readonly valid: boolean } | null;
  /** Riquadro di selezione (coordinate dell'area) e nodi che comprende. */
  readonly marquee: { readonly rect: Rect; readonly ids: readonly string[] } | null;
  readonly confirm: ConfirmState | null;
  readonly hint: string | null;
}

export const IDLE_UI: InteractionUi = {
  dragging: [],
  drop: null,
  insertLink: null,
  tempLink: null,
  marquee: null,
  confirm: null,
  hint: null,
};

type Active =
  | {
      readonly kind: "node";
      readonly id: string;
      readonly start: Point;
      readonly shift: boolean;
      readonly group: readonly string[] | null;
      readonly canInsert: boolean;
      moved: boolean;
      /** Esito mostrato nell'ultimo aggiornamento: è quello applicato al rilascio. */
      drop: { id: string; outcome: DragOutcome } | null;
      insertLink: Link | null;
      /** Percorsi dei cavi prima del gesto (vedi `moveNode`). */
      startRoutes: LinkRoutes | null;
    }
  | {
      readonly kind: "port";
      readonly id: string;
      readonly from: Point;
      rel: "link" | "link-reverse" | null;
      target: string | null;
    }
  | {
      readonly kind: "marquee";
      readonly start: Point;
      readonly base: readonly string[];
      moved: boolean;
      ids: readonly string[];
    };

export interface InteractionController {
  getUi(): InteractionUi;
  subscribe(listener: () => void): () => void;
  /** Un gesto è in corso (il livello DOM ascolta i movimenti finché è vero). */
  isActive(): boolean;
  /** Barra spaziatrice premuta: navigazione, nessun gesto di selezione. */
  setSpace(down: boolean): void;
  down(target: DownTarget, input: PointerInput): boolean;
  move(input: PointerInput): void;
  up(input: PointerInput): void;
  /** Interrompe il gesto (pointercancel, Esc). */
  cancel(): void;
  key(input: KeyInput): boolean;
  confirmDelete(): CommandResult | null;
  cancelConfirm(): void;
  /** Punto dell'area → coordinate del mondo (per chi rilascia dalla cassetta). */
  toWorld(x: number, y: number): Point;
  /** Rilascio di un nuovo elemento: vedi drop.ts. */
  handleCanvasDrop(payload: CanvasDropPayload, point: Point): CommandResult;
  previewCanvasDrop(payload: CanvasDropPayload, point: Point): DropPreview;
}

function sameIds(a: readonly string[], b: readonly string[]): boolean {
  return a.length === b.length && a.every((id, i) => id === b[i]);
}

/** Testi della conferma (prototipo, righe 4509-4552). */
function confirmCopy(
  ids: readonly string[],
  removedCount: number,
): { title: string; text: string } {
  const downstream = removedCount - ids.length;
  if (ids.length === 1) {
    const n = downstream;
    return {
      title: "Eliminare il nodo?",
      text:
        n > 0
          ? `Fa parte del flusso. Verranno rimossi i suoi collegamenti e ${n}${n > 1 ? " risultati a valle." : " risultato a valle."}`
          : "Fa parte del flusso: i suoi collegamenti verranno rimossi.",
    };
  }
  return {
    title: `Eliminare ${ids.length} nodi?`,
    text:
      "Alcuni fanno parte del flusso: verranno rimossi i loro collegamenti" +
      (downstream > 0
        ? ` e ${downstream} ${downstream > 1 ? "risultati" : "risultato"} a valle.`
        : "."),
  };
}

export function createInteractionController(store: EtlStore): InteractionController {
  let ui: InteractionUi = IDLE_UI;
  let active: Active | null = null;
  let space = false;
  const listeners = new Set<() => void>();

  const setUi = (patch: Partial<InteractionUi>): void => {
    const next = { ...ui, ...patch };
    const keys = Object.keys(next) as (keyof InteractionUi)[];
    if (keys.every((k) => next[k] === ui[k])) return;
    ui = next;
    for (const l of [...listeners]) l();
  };
  const graph = (): Graph => store.getState().graph;
  const worldOf = (p: Point): Point => toWorld(store.getState().view, p.x, p.y);
  const linkOfKey = (key: string): Link | null =>
    graph().links.find((l) => linkKey(l) === key) ?? null;

  /** `select` + `inspect`: l'Inspector segue la selezione (nessun pannello, solo stato). */
  const applySelection = (ids: readonly string[]): void => {
    const s = store.getState();
    const unique = Array.from(new Set(ids));
    if (!sameIds(s.selection, unique)) store.dispatch({ type: "select", payload: { ids: unique } });
    const keep = s.inspector.nodeId && unique.includes(s.inspector.nodeId);
    const node = unique.length ? (keep ? s.inspector.nodeId : (unique[0] as string)) : null;
    if (node !== s.inspector.nodeId || (node !== null && s.inspector.step !== 0 && !keep)) {
      store.dispatch({ type: "inspect", payload: { node, step: 0 } });
    }
  };

  const hintForLink = (g: Graph, dragged: string, over: string): string => {
    const boxId = relation(g, dragged, over).relation === "link" ? over : dragged;
    const box = g.cards[boxId];
    const need = box ? boxCapacity(box) - inputsOf(g, boxId).length : 1;
    return need > 1
      ? "Rilascia: sarà la tabella di sinistra, poi servirà la seconda"
      : "Rilascia per collegare e generare l’output";
  };

  // --- trascinamento di un nodo ---------------------------------------------------

  const moveNode = (a: Extract<Active, { kind: "node" }>, input: PointerInput): void => {
    const view = store.getState().view;
    const sx = input.x - a.start.x;
    const sy = input.y - a.start.y;
    if (!a.moved) {
      if (Math.hypot(sx, sy) < DRAG_THRESHOLD) return;
      const ids = a.group ?? [a.id];
      if (a.canInsert) a.startRoutes = store.getRoutes();
      const r = store.beginGesture({ ids });
      if (!r.ok) {
        active = null;
        return;
      }
      a.moved = true;
      setUi({ dragging: ids });
    }
    const dx = sx / view.zoom;
    const dy = sy / view.zoom;
    if (a.group) {
      store.updateGesture({ dx, dy });
      return;
    }
    const p = worldOf(input);
    const over = nodeAt(graph(), p, a.id);
    // l'esito si calcola sul grafo PRIMA dell'aggiornamento: l'aggiornamento può spingere via l'altro nodo
    const rel = over ? relation(graph(), a.id, over) : null;
    store.updateGesture({ dx, dy, over });

    if (over && rel) {
      const outcome: DragOutcome = rel.relation ?? "reject";
      a.drop = { id: over, outcome };
      a.insertLink = null;
      setUi({
        drop: a.drop,
        insertLink: null,
        hint:
          outcome === "merge"
            ? "Rilascia per fondere le lavorazioni"
            : outcome === "link" || outcome === "link-reverse"
              ? hintForLink(graph(), a.id, over)
              : (rel.displaceReason ?? null),
      });
      return;
    }
    a.drop = null;
    // sopra un cavo, senza un nodo sotto: una lavorazione slegata si inserisce nel collegamento
    let insert: Link | null = null;
    if (a.canInsert) {
      // il nodo in mano è un ostacolo e fa scansare i cavi: si prova anche sui percorsi di prima del gesto
      const key = linkAt(store.getRoutes(), p) ?? (a.startRoutes ? linkAt(a.startRoutes, p) : null);
      const link = key ? linkOfKey(key) : null;
      if (link && insertable(graph(), link, a.id)) insert = link;
    }
    a.insertLink = insert;
    setUi({
      drop: null,
      insertLink: insert ? linkKey(insert) : null,
      hint: insert ? "Rilascia per inserire la lavorazione nel collegamento" : null,
    });
  };

  const releaseNode = (a: Extract<Active, { kind: "node" }>, input: PointerInput): void => {
    if (!a.moved) {
      // un click: Maiusc aggiunge o toglie; altrimenti seleziona solo questo (riga 2069)
      if (a.shift || input.shiftKey) {
        const sel = store.getState().selection;
        applySelection(sel.includes(a.id) ? sel.filter((x) => x !== a.id) : [...sel, a.id]);
      } else applySelection([a.id]);
      return;
    }
    const outcome = a.drop?.outcome;
    const target =
      a.drop && (outcome === "merge" || outcome === "link" || outcome === "link-reverse")
        ? { node: a.drop.id }
        : a.insertLink
          ? { link: a.insertLink }
          : undefined;
    store.commitGesture(target ? { target } : {});
  };

  // --- porte ----------------------------------------------------------------------

  const movePort = (a: Extract<Active, { kind: "port" }>, input: PointerInput): void => {
    const p = worldOf(input);
    const g = graph();
    const over = nodeAt(g, p, a.id);
    const r = over ? relation(g, a.id, over) : null;
    // dalle porte si collega soltanto: fusione e spostamento diventano rifiuto (riga 3990)
    const rel = r && (r.relation === "link" || r.relation === "link-reverse") ? r.relation : null;
    a.rel = rel;
    a.target = rel ? over : null;
    setUi({
      tempLink: { from: a.from, to: p, valid: !!rel },
      drop: over ? { id: over, outcome: rel ?? "reject" } : null,
      hint: rel
        ? "Rilascia per collegare"
        : over
          ? (r?.displaceReason ?? "Questi due nodi non si possono collegare")
          : "Trascina fino al nodo da collegare",
    });
  };

  const releasePort = (a: Extract<Active, { kind: "port" }>): void => {
    if (a.rel && a.target) {
      if (a.rel === "link")
        store.dispatch({ type: "connect", payload: { from: a.id, to: a.target } });
      else store.dispatch({ type: "connect", payload: { from: a.target, to: a.id } });
    }
  };

  // --- riquadro di selezione ------------------------------------------------------

  const marqueeRect = (a: Extract<Active, { kind: "marquee" }>, input: PointerInput): Rect => ({
    x: Math.min(a.start.x, input.x),
    y: Math.min(a.start.y, input.y),
    w: Math.abs(input.x - a.start.x),
    h: Math.abs(input.y - a.start.y),
  });

  /** Nodi il cui quadrato tocca il riquadro (prototipo, righe 4071-4076: il riquadro è in coordinate dello schermo). */
  const idsInRect = (rect: Rect): string[] => {
    const a = worldOf({ x: rect.x, y: rect.y });
    const b = worldOf({ x: rect.x + rect.w, y: rect.y + rect.h });
    const out: string[] = [];
    for (const c of Object.values(graph().cards)) {
      const r = nodeRect(c);
      if (r.x + r.w > a.x && r.x < b.x && r.y + r.h > a.y && r.y < b.y) out.push(c.id);
    }
    return out;
  };

  // --- eliminazione ---------------------------------------------------------------

  const requestDelete = (): void => {
    const ids = store.getState().selection.filter((id) => !!graph().cards[id]);
    if (!ids.length) return;
    const attached = graph().links.some((l) => ids.includes(l.from) || ids.includes(l.to));
    if (!attached) {
      store.dispatch({ type: "deleteNodes", payload: { ids } });
      return;
    }
    const removed = [...nodesRemovedBy(graph(), ids)];
    setUi({ confirm: { ids, removed, ...confirmCopy(ids, removed.length) } });
  };

  const nudge = (dx: number, dy: number): void => {
    const ids = store.getState().selection;
    if (ids.length) store.dispatch({ type: "moveNodes", payload: { ids, dx, dy } });
  };

  const finish = (): void => {
    active = null;
    setUi({
      dragging: [],
      drop: null,
      insertLink: null,
      tempLink: null,
      marquee: null,
      hint: null,
    });
  };

  const controller: InteractionController = {
    getUi: () => ui,
    subscribe(l) {
      listeners.add(l);
      return () => listeners.delete(l);
    },
    isActive: () => active !== null,
    setSpace(down) {
      space = down;
    },

    down(target, input) {
      if (active) return false;
      if (input.button !== undefined && input.button !== 0) return false;
      if (target.kind === "ignore") return false;
      if (ui.confirm) setUi({ confirm: null });
      if (space) return false;
      const start = { x: input.x, y: input.y };

      if (target.kind === "port") {
        const card = graph().cards[target.id];
        if (!card) return false;
        const port = nodePorts(card)[PORT_INDEX[target.side]];
        if (!port) return false;
        const from = { x: port.anchor.x, y: port.anchor.y };
        active = { kind: "port", id: target.id, from, rel: null, target: null };
        setUi({
          dragging: [target.id],
          tempLink: { from, to: worldOf(start), valid: false },
          hint: "Trascina fino al nodo da collegare",
        });
        return true;
      }

      if (target.kind === "node") {
        const card = graph().cards[target.id];
        if (!card) return false;
        const sel = store.getState().selection;
        const group =
          sel.includes(target.id) && sel.length > 1
            ? sel.filter((id) => !!graph().cards[id])
            : null;
        const canInsert =
          !group &&
          card.kind === "op" &&
          !graph().links.some((l) => l.from === target.id || l.to === target.id);
        active = {
          kind: "node",
          id: target.id,
          start,
          shift: !!input.shiftKey,
          group,
          canInsert,
          moved: false,
          drop: null,
          insertLink: null,
          startRoutes: null,
        };
        return true;
      }

      // sfondo: riquadro di selezione (Maiusc = si aggiunge alla selezione)
      const base = input.shiftKey ? [...store.getState().selection] : [];
      active = { kind: "marquee", start, base, moved: false, ids: base };
      return true;
    },

    move(input) {
      const a = active;
      if (!a) return;
      if (a.kind === "node") moveNode(a, input);
      else if (a.kind === "port") movePort(a, input);
      else {
        if (!a.moved && Math.hypot(input.x - a.start.x, input.y - a.start.y) < MARQUEE_THRESHOLD)
          return;
        a.moved = true;
        const rect = marqueeRect(a, input);
        const ids = Array.from(new Set([...a.base, ...idsInRect(rect)]));
        a.ids = ids;
        setUi({ marquee: { rect, ids } });
      }
    },

    up(input) {
      const a = active;
      if (!a) return;
      if (a.kind === "node") {
        releaseNode(a, input);
      } else if (a.kind === "port") {
        releasePort(a);
      } else if (a.moved) {
        applySelection(a.ids);
      } else if (store.getState().selection.length || store.getState().inspector.nodeId) {
        // un click sul vuoto deseleziona (riga 4584)
        applySelection([]);
      }
      finish();
    },

    cancel() {
      const a = active;
      if (!a) return;
      if (a.kind === "node" && a.moved) store.cancelGesture();
      finish();
    },

    key(e) {
      if (e.typing) return false;
      const mod = !!(e.metaKey || e.ctrlKey);
      const k = e.key.length === 1 ? e.key.toLowerCase() : e.key;
      if (!mod) {
        if (k === "Delete" || k === "Backspace") {
          if (!store.getState().selection.length) return false;
          requestDelete();
          return true;
        }
        if (k === "Escape") {
          if (active) {
            controller.cancel();
            return true;
          }
          if (ui.confirm) {
            setUi({ confirm: null });
            return true;
          }
          applySelection([]);
          return true;
        }
        const step = e.shiftKey ? GRID : NUDGE_STEP;
        const arrows: Record<string, [number, number]> = {
          ArrowLeft: [-step, 0],
          ArrowRight: [step, 0],
          ArrowUp: [0, -step],
          ArrowDown: [0, step],
        };
        const d = arrows[k];
        if (d && store.getState().selection.length) {
          nudge(d[0], d[1]);
          return true;
        }
        return false;
      }
      if (k === "d") {
        const r = store.dispatch({
          type: "duplicate",
          payload: { ids: store.getState().selection },
        });
        if (r.ok) applySelection(store.getState().selection);
        return true;
      }
      if (k === "a") {
        applySelection(Object.keys(graph().cards));
        return true;
      }
      if (k === "z" && e.shiftKey) {
        store.redo();
        return true;
      }
      if (k === "z") {
        store.undo();
        return true;
      }
      if (k === "y") {
        store.redo();
        return true;
      }
      return false;
    },

    confirmDelete() {
      const c = ui.confirm;
      if (!c) return null;
      setUi({ confirm: null });
      return store.dispatch({ type: "deleteNodes", payload: { ids: c.ids } });
    },
    cancelConfirm() {
      setUi({ confirm: null });
    },

    toWorld: (x, y) => toWorld(store.getState().view, x, y),
    handleCanvasDrop: (payload, point) => handleCanvasDrop(store, payload, point),
    previewCanvasDrop: (payload, point) => previewCanvasDrop(store, payload, point),
  };
  return controller;
}
```

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

112 righe

```ts
/**
 * Dal grafo di etl-core a ciò che il canvas disegna: classi e icone di ogni
 * nodo, fette di un output parziale. Funzioni pure.
 */
import { sectionOf } from "../etl-core";
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

/** Famiglia di operazioni (colore del nodo): una per sezione della cassetta, tranne i dataset. */
export type OpFamily = "filter" | "transform" | "merge" | "output";

const FAMILY_OF_SECTION: Readonly<Record<string, OpFamily>> = {
  rows: "filter",
  xform: "transform",
  merge: "merge",
  out: "output",
};

/** La famiglia di un nodo lavorazione (quella della prima operazione); i dataset non ne hanno. */
export function familyOf(card: Card): OpFamily | undefined {
  if (card.kind !== "op") return undefined;
  const first = card.components[0];
  const section = first === undefined ? null : sectionOf(first);
  return section ? FAMILY_OF_SECTION[section.id] : undefined;
}

export interface NodeView {
  readonly id: string;
  readonly card: Card;
  /** Classi del contenitore del nodo. */
  readonly className: string;
  readonly iconClass: string;
  readonly partial: boolean;
  readonly slices: readonly Slice[];
  readonly family: OpFamily | undefined;
  readonly icons: readonly ComponentId[];
  readonly warn: string | null;
  readonly selected: boolean;
}

/** Classi del nodo (prototipo `createCardEl`, righe 1003-1004, più `partial`, `warn`, `selected`). */
/** Stato dei gesti che cambia l'aspetto di un nodo (classi `ec-dragging`, `ec-drop-*`, `ec-doomed`). */
export interface NodeGestureState {
  readonly dragging?: boolean;
  readonly drop?: "merge" | "link" | "link-reverse" | "displace" | "reject" | null;
  readonly doomed?: boolean;
}

export function nodeView(
  card: Card,
  warn: string | null,
  selected: boolean,
  gesture: NodeGestureState = {},
): NodeView {
  const combined = card.kind === "op" && card.components.length > 1;
  const partial = isPartial(card);
  const classes = ["ec-card"];
  if (card.kind === "dataset") classes.push("ec-dataset");
  if (card.kind === "dataset" && card.isOutput) classes.push("ec-output");
  if (combined) classes.push("ec-combined");
  if (partial) classes.push("ec-partial");
  if (warn) classes.push("ec-warn");
  if (selected) classes.push("ec-selected");
  if (gesture.dragging) classes.push("ec-dragging");
  if (gesture.drop) classes.push(`ec-drop-${gesture.drop}`);
  if (gesture.doomed) classes.push("ec-doomed");
  const icons: ComponentId[] =
    card.kind === "dataset" && card.isOutput ? ["dataset"] : [...card.components];
  return {
    id: card.id,
    card,
    className: classes.join(" "),
    iconClass: partial ? "ec-icon-wrap ec-split" : `ec-icon-wrap ec-${countClass(icons.length)}`,
    partial,
    slices: partial ? slicesOf(card) : [],
    family: familyOf(card),
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

137 righe

```css
/*
 * Token del canvas ETL (livello 3 — di componente). Ambito: SOLO il
 * contenitore `.etl-canvas`.
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
.etl-canvas {
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
.etl-canvas {
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
.etl-canvas [data-family="filter"] {
  --ec-node-op: var(--isa-op-filter-soft);
  --ec-node-op-ink: var(--isa-op-filter);
}
.etl-canvas [data-family="transform"] {
  --ec-node-op: var(--isa-op-transform-soft);
  --ec-node-op-ink: var(--isa-op-transform);
}
.etl-canvas [data-family="merge"] {
  --ec-node-op: var(--isa-op-merge-soft);
  --ec-node-op-ink: var(--isa-op-merge);
}
.etl-canvas [data-family="output"] {
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

