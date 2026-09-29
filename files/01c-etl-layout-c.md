# 01c-etl-layout-c.md

File in questo blocco:

- `src/etl-layout/free.ts`
- `src/etl-layout/hitTest.ts`
- `src/etl-layout/index.ts`
- `src/etl-layout/links.ts`
- `src/etl-layout/nodes.ts`
- `src/etl-layout/path.ts`
- `src/etl-layout/placement.ts`
- `src/etl-layout/routing.ts`
- `src/etl-layout/slots.ts`
- `src/etl-layout/types.ts`

---

### `src/etl-layout/free.ts`

246 righe

```ts
/**
 * Modalità Libero: limiti del mondo, aggancio alla griglia, separazione
 * dei nodi sovrapposti. Prototipo, righe 1537-1609, 1939-1956, 2630-2638.
 */
import { compatiblePair } from "../etl-core";
import type { Card, Graph } from "../etl-core";
import {
  CARD,
  DISPLACE_BOTTOM,
  DISPLACE_PUSH,
  FREE_SPOT_RADIUS,
  FREE_SPOT_STEP_X,
  FREE_SPOT_STEP_Y,
  GRID,
  LABEL_H,
  OVERLAP_ITERATIONS,
  OVERLAP_MARGIN,
  OVERLAP_PAD,
  WORLD_H,
  WORLD_MARGIN,
  WORLD_W,
} from "./constants";
import type { Point, Size } from "./types";

export const DEFAULT_WORLD: Size = { w: WORLD_W, h: WORLD_H };

/** Riga 1537: il nodo resta dentro il mondo. */
export function clampPoint(p: Point, world: Size = DEFAULT_WORLD): Point {
  return {
    x: Math.max(WORLD_MARGIN, Math.min(world.w - CARD - WORLD_MARGIN, p.x)),
    y: Math.max(WORLD_MARGIN, Math.min(world.h - CARD - LABEL_H - WORLD_MARGIN, p.y)),
  };
}

/** Riga 1541 (uso nelle righe 1582-1583): allineamento alla griglia. */
export function snapToGrid(p: Point): Point {
  return { x: Math.round(p.x / GRID) * GRID, y: Math.round(p.y / GRID) * GRID };
}

/** Sostituisce le posizioni indicate, senza toccare il resto del grafo. */
export function withPositions(graph: Graph, pos: ReadonlyMap<string, Point>): Graph {
  let changed = false;
  const cards: Record<string, Card> = {};
  for (const [id, c] of Object.entries(graph.cards)) {
    const p = pos.get(id);
    if (p && (p.x !== c.x || p.y !== c.y)) {
      cards[id] = { ...c, x: p.x, y: p.y };
      changed = true;
    } else cards[id] = c;
  }
  return changed ? { ...graph, cards } : graph;
}

export interface ResolveOverlapsOptions {
  /** Il nodo che resta fermo: gli altri si scansano. */
  readonly fixedId?: string | null;
  /** Allinea alla griglia tutti tranne `fixedId`. */
  readonly snap?: boolean;
  /** Separa solo le coppie incompatibili (durante il trascinamento). */
  readonly onlyIncompatible?: boolean;
  readonly world?: Size;
}

/**
 * Riga 1551 (solo il ramo della modalità Libero; in Organizzato il
 * prototipo chiama `placeInSlots`, vedi `slots.ts`). Separa le coppie
 * troppo vicine lungo l'asse di compenetrazione minore, in proporzione.
 */
export function resolveOverlaps(graph: Graph, opts: ResolveOverlapsOptions = {}): Graph {
  const world = opts.world ?? DEFAULT_WORLD;
  const fixedId = opts.fixedId ?? null;
  const minDX = CARD + OVERLAP_PAD;
  const minDY = CARD + LABEL_H + OVERLAP_PAD;
  const ids = Object.keys(graph.cards);
  const pos = new Map<string, { x: number; y: number }>();
  for (const id of ids) {
    const c = graph.cards[id] as Card;
    pos.set(id, { x: c.x, y: c.y });
  }
  const clampIn = (p: { x: number; y: number }): void => {
    const q = clampPoint(p, world);
    p.x = q.x;
    p.y = q.y;
  };
  for (let iter = 0; iter < OVERLAP_ITERATIONS; iter++) {
    let moved = false;
    for (let i = 0; i < ids.length; i++) {
      for (let j = i + 1; j < ids.length; j++) {
        const idA = ids[i] as string;
        const idB = ids[j] as string;
        const A = pos.get(idA) as { x: number; y: number };
        const B = pos.get(idB) as { x: number; y: number };
        if (opts.onlyIncompatible && compatiblePair(graph, idA, idB)) continue;
        const dx = B.x - A.x;
        const dy = B.y - A.y;
        const ox = minDX - Math.abs(dx);
        const oy = minDY - Math.abs(dy);
        if (ox <= 0 || oy <= 0) continue;
        let sx = 0;
        let sy = 0;
        if (ox / minDX < oy / minDY) sx = ((dx >= 0 ? 1 : -1) * ox) / 2 + 0.5;
        else sy = ((dy >= 0 ? 1 : -1) * oy) / 2 + 0.5;
        const aFixed = idA === fixedId;
        const bFixed = idB === fixedId;
        if (aFixed) {
          B.x += sx * 2;
          B.y += sy * 2;
        } else if (bFixed) {
          A.x -= sx * 2;
          A.y -= sy * 2;
        } else {
          A.x -= sx;
          A.y -= sy;
          B.x += sx;
          B.y += sy;
        }
        clampIn(A);
        clampIn(B);
        moved = true;
      }
    }
    if (!moved) break;
  }
  for (const id of ids) {
    const p = pos.get(id) as { x: number; y: number };
    if (opts.snap && id !== fixedId) {
      const s = snapToGrid(p);
      p.x = s.x;
      p.y = s.y;
    }
    clampIn(p);
  }
  return withPositions(graph, pos);
}

/**
 * Riga 2009 (dentro il trascinamento): durante lo spostamento di un nodo
 * solo le coppie incompatibili si scansano; il nodo in mano resta fermo
 * e nulla viene allineato alla griglia.
 */
export function separateWhileDragging(graph: Graph, draggedId: string, world?: Size): Graph {
  return resolveOverlaps(graph, {
    fixedId: draggedId,
    snap: false,
    onlyIncompatible: true,
    ...(world ? { world } : {}),
  });
}

/** Riga 2102: rilasciato nel vuoto, separa e riallinea alla griglia. */
export function dropFree(graph: Graph, droppedId: string, world?: Size): Graph {
  return resolveOverlaps(graph, { fixedId: droppedId, snap: true, ...(world ? { world } : {}) });
}

/** Riga 1591: un nodo in (x, y) toccherebbe un altro nodo? */
export function overlapsAny(graph: Graph, x: number, y: number, ignoreId?: string | null): boolean {
  return Object.keys(graph.cards).some((id) => {
    const c = graph.cards[id];
    return (
      id !== ignoreId &&
      !!c &&
      Math.abs(c.x - x) < CARD + OVERLAP_MARGIN &&
      Math.abs(c.y - y) < CARD + LABEL_H + OVERLAP_MARGIN
    );
  });
}

/** Riga 1595: il punto libero più vicino, cercando a spirale in otto direzioni. */
export function freeSpot(
  graph: Graph,
  x: number,
  y: number,
  ignoreId?: string | null,
  world: Size = DEFAULT_WORLD,
): Point {
  const maxX = world.w - CARD - WORLD_MARGIN;
  const maxY = world.h - CARD - LABEL_H - WORLD_MARGIN;
  const cx = Math.max(WORLD_MARGIN, Math.min(maxX, x));
  const cy = Math.max(WORLD_MARGIN, Math.min(maxY, y));
  if (!overlapsAny(graph, cx, cy, ignoreId)) return { x: cx, y: cy };
  const dirs: readonly (readonly [number, number])[] = [
    [1, 0],
    [1, 1],
    [0, 1],
    [1, -1],
    [0, -1],
    [-1, 1],
    [-1, 0],
    [-1, -1],
  ];
  for (let r = 1; r <= FREE_SPOT_RADIUS; r++) {
    for (const [ox, oy] of dirs) {
      const nx = Math.max(WORLD_MARGIN, Math.min(maxX, cx + ox * r * FREE_SPOT_STEP_X));
      const ny = Math.max(WORLD_MARGIN, Math.min(maxY, cy + oy * r * FREE_SPOT_STEP_Y));
      if (!overlapsAny(graph, nx, ny, ignoreId)) return { x: nx, y: ny };
    }
  }
  return { x: cx, y: cy };
}

/** Riga 2630: almeno due nodi troppo vicini. */
export function anyOverlap(graph: Graph): boolean {
  const cards = Object.values(graph.cards);
  for (let i = 0; i < cards.length; i++)
    for (let j = i + 1; j < cards.length; j++) {
      const A = cards[i] as Card;
      const B = cards[j] as Card;
      if (
        Math.abs(A.x - B.x) < CARD + OVERLAP_MARGIN &&
        Math.abs(A.y - B.y) < CARD + LABEL_H + OVERLAP_MARGIN
      )
        return true;
    }
  return false;
}

/**
 * Riga 1939: un nodo incompatibile sotto quello trascinato viene spinto
 * via lungo la congiungente dei centri. Il prototipo, a fine animazione
 * (riga 1955), separa poi con `resolveOverlaps(dragged, null, true)`:
 * qui è incluso, restituendo lo stato finale.
 */
export function displace(
  graph: Graph,
  draggedId: string,
  targetId: string,
  world: Size = DEFAULT_WORLD,
): Graph {
  const d = graph.cards[draggedId];
  const t = graph.cards[targetId];
  if (!d || !t) return graph;
  let vx = t.x + CARD / 2 - (d.x + CARD / 2);
  let vy = t.y + CARD / 2 - (d.y + CARD / 2);
  const len = Math.hypot(vx, vy) || 1;
  if (len < 1) {
    vx = 0;
    vy = 1;
  }
  let nx = t.x + (vx / len) * DISPLACE_PUSH;
  let ny = t.y + (vy / len) * DISPLACE_PUSH;
  nx = Math.max(WORLD_MARGIN, Math.min(world.w - CARD - WORLD_MARGIN, nx));
  ny = Math.max(WORLD_MARGIN, Math.min(world.h - DISPLACE_BOTTOM, ny));
  const pushed = withPositions(graph, new Map([[targetId, { x: nx, y: ny }]]));
  return resolveOverlaps(pushed, { fixedId: draggedId, snap: true, world });
}
```

### `src/etl-layout/hitTest.ts`

85 righe

```ts
/**
 * Individuazione del nodo e del cavo sotto un punto. Nel prototipo è il
 * DOM a farlo (`elementFromPoint`, righe 2013, 3992, 4960, e i tracciati
 * `.hit` larghi 16 px, riga 1402); qui la stessa geometria, senza DOM.
 *
 * Ordine di sovrapposizione: come nel DOM del prototipo, il nodo (o il
 * cavo) creato dopo sta sopra, quindi vince l'ultimo che contiene il punto.
 */
import type { Card, Graph } from "../etl-core";
import { ELBOW_R, LINK_HIT_WIDTH } from "./constants";
import { labelRect, nodeRect } from "./nodes";
import { roundedPieces } from "./path";
import type { LinkRoute, LinkRoutes, Point, Rect } from "./types";

function inRect(p: Point, r: Rect): boolean {
  return p.x >= r.x && p.x <= r.x + r.w && p.y >= r.y && p.y <= r.y + r.h;
}

/** Il nodo sotto `p` (quadrato o etichetta), o `null`. `ignoreId` è il nodo in mano (riga 2012). */
export function nodeAt(graph: Graph, p: Point, ignoreId?: string | null): string | null {
  const ids = Object.keys(graph.cards);
  for (let i = ids.length - 1; i >= 0; i--) {
    const id = ids[i] as string;
    if (id === ignoreId) continue;
    const c = graph.cards[id] as Card;
    if (inRect(p, nodeRect(c)) || inRect(p, labelRect(c))) return id;
  }
  return null;
}

function distToSegment(p: Point, a: Point, b: Point): number {
  const vx = b.x - a.x;
  const vy = b.y - a.y;
  const len2 = vx * vx + vy * vy;
  const t = len2 ? Math.max(0, Math.min(1, ((p.x - a.x) * vx + (p.y - a.y) * vy) / len2)) : 0;
  return Math.hypot(p.x - (a.x + t * vx), p.y - (a.y + t * vy));
}

/** Suddivisioni di una curva di raccordo per misurare la distanza. */
const QUAD_SAMPLES = 16;

function quadPoint(a: Point, c: Point, b: Point, t: number): Point {
  const u = 1 - t;
  return {
    x: u * u * a.x + 2 * u * t * c.x + t * t * b.x,
    y: u * u * a.y + 2 * u * t * c.y + t * t * b.y,
  };
}

/** Distanza di `p` dal percorso arrotondato di un cavo. */
export function distanceToRoute(p: Point, route: Pick<LinkRoute, "pts">): number {
  let best = Infinity;
  for (const piece of roundedPieces(route.pts, ELBOW_R)) {
    if (piece.kind === "line") {
      best = Math.min(best, distToSegment(p, piece.from, piece.to));
    } else {
      let prev = piece.from;
      for (let k = 1; k <= QUAD_SAMPLES; k++) {
        const q = quadPoint(piece.from, piece.ctrl, piece.to, k / QUAD_SAMPLES);
        best = Math.min(best, distToSegment(p, prev, q));
        prev = q;
      }
    }
  }
  return best;
}

/**
 * Il cavo sotto `p` entro la tolleranza del prototipo (metà di
 * `stroke-width="16"`, cioè 8 px), o `null`. Restituisce la chiave `from|to`.
 * `routes` va passato nell'ordine dei collegamenti (come `layoutLinks`).
 */
export function linkAt(
  routes: LinkRoutes,
  p: Point,
  tolerance: number = LINK_HIT_WIDTH / 2,
): string | null {
  const keys = Object.keys(routes);
  for (let i = keys.length - 1; i >= 0; i--) {
    const key = keys[i] as string;
    if (distanceToRoute(p, routes[key] as LinkRoute) <= tolerance) return key;
  }
  return null;
}
```

### `src/etl-layout/index.ts`

79 righe

```ts
/**
 * etl-layout — Fase 2: geometria del canvas in TypeScript puro.
 * Può importare da etl-core, mai viceversa. Vedi README.md.
 */
export * from "./constants";
export * from "./types";
export {
  nodeRect,
  labelRect,
  footprint,
  obstacleRect,
  nodeCenter,
  borderPoint,
  axisOf,
  nodePorts,
} from "./nodes";
export {
  segHitsRect,
  routeCost,
  buildRoute,
  countBends,
  segCross,
  countCrossings,
  overshoot,
  routeLength,
  slide,
  orthogonalize,
  removeReversals,
  buildFromShape,
  backtrack,
  shapeCandidates,
  routeCandidates,
  chooseRoute,
} from "./routing";
export type { ChooseRouteOptions, RouteCandidate } from "./routing";
export { roundedPath, roundedPieces, cleanPoints } from "./path";
export type { PathPiece } from "./path";
export { linkKey, layoutLinks, settleLinks, SETTLE_MAX_PASSES } from "./links";
export type { LayoutLinksOptions } from "./links";
export {
  DEFAULT_WORLD,
  clampPoint,
  snapToGrid,
  withPositions,
  resolveOverlaps,
  separateWhileDragging,
  dropFree,
  overlapsAny,
  freeSpot,
  anyOverlap,
  displace,
} from "./free";
export type { ResolveOverlapsOptions } from "./free";
export {
  computeSlots,
  slotTaken,
  nearestSlot,
  firstFreeSlot,
  setSlot,
  placeInSlots,
  assignSlots,
  clearSlots,
  dropInSlot,
  outputSlotFor,
} from "./slots";
export { autoLayout } from "./autoLayout";
export type { AutoLayoutOptions } from "./autoLayout";
export {
  outputPositionFn,
  detachPositionFn,
  insertPosition,
  moveNode,
  settleNewNode,
  relocateAfter,
  relocateAfterMerge,
} from "./placement";
export type { PlacementOptions } from "./placement";
export { nodeAt, linkAt, distanceToRoute } from "./hitTest";
```

### `src/etl-layout/links.ts`

225 righe

```ts
/**
 * Percorsi di tutti i cavi del grafo. Prototipo, righe 1300-1411
 * (`drawLinks`), nello stato A REGIME: senza le animazioni di
 * `easeAngle` (righe 1339-1340) e dello snodo (riga 1341), cioè con
 * l'angolo di aggancio uguale alla porta scelta e lo snodo uguale al suo
 * valore obiettivo. Le animazioni sono la Fase 5.
 */
import type { Graph, Link } from "../etl-core";
import { CARD, ELBOW_R, LANE_GAP, LANE_NEAR, MAX_BENDS, PORT_SPREAD } from "./constants";
import { axisOf, borderPoint, nodeCenter, obstacleRect } from "./nodes";
import { roundedPath } from "./path";
import { buildFromShape, chooseRoute, slide } from "./routing";
import type { Axis, LinkRoute, LinkRoutes, Point, Rect, Shape } from "./types";

export function linkKey(l: Link): string {
  return l.from + "|" + l.to;
}

export interface LayoutLinksOptions {
  /** Snodi ammessi per cavo (predefinito: `MAX_BENDS` del prototipo, 1). */
  readonly maxBends?: number;
  /** Nodo in trascinamento: non è un ostacolo per i cavi altrui (righe 1317, 1844). */
  readonly draggingId?: string | null;
}

interface Chosen {
  readonly link: Link;
  readonly index: number;
  readonly portA: number;
  readonly portB: number;
  readonly horiz: boolean;
  readonly shape: Shape;
  readonly a: Point;
  readonly b: Point;
}

function shift(pt: Point, axis: Axis, off: number): Point {
  if (!off) return pt;
  return axis.x !== 0 ? { x: pt.x, y: pt.y + off } : { x: pt.x + off, y: pt.y };
}

/**
 * Una valutazione completa di tutti i cavi (una chiamata di `drawLinks`
 * con `nextEval` scaduto per tutti). `prev` è il risultato precedente: da
 * lì vengono le porte da conservare (stabilità) e i percorsi degli altri
 * cavi per contare gli incroci — come nel prototipo, dove ogni cavo vede
 * i percorsi del fotogramma precedente.
 */
export function layoutLinks(
  graph: Graph,
  prev: LinkRoutes = {},
  opts: LayoutLinksOptions = {},
): Record<string, LinkRoute> {
  const half = CARD / 2;
  const maxBends = opts.maxBends ?? MAX_BENDS;
  const ids = Object.keys(graph.cards);

  // primo passaggio: porte e forme (righe 1306-1344)
  const items: Chosen[] = [];
  graph.links.forEach((l, index) => {
    const ca = graph.cards[l.from];
    const cb = graph.cards[l.to];
    if (!ca || !cb) return;
    const a = nodeCenter(ca);
    const b = nodeCenter(cb);
    const st = prev[linkKey(l)];
    const obstacles: Rect[] = ids
      .filter((id) => id !== l.from && id !== l.to && id !== opts.draggingId)
      .map((id) => obstacleRect(graph.cards[id] as { x: number; y: number }));
    const others = graph.links
      .filter((o) => o !== l)
      .map((o) => prev[linkKey(o)])
      .filter((os): os is LinkRoute => !!os)
      .map((os) => os.pts);
    const best = chooseRoute(a, b, half, obstacles, {
      prevPortA: st?.portA,
      prevPortB: st?.portB,
      others,
      maxBends,
    });
    if (!best) return;
    items.push({
      link: l,
      index,
      portA: best.portA,
      portB: best.portB,
      horiz: best.horiz,
      shape: best.shape,
      a,
      b,
    });
  });

  // più cavi sulla stessa porta: si distanziano lungo il bordo (righe 1346-1358)
  const groups = new Map<string, { index: number; end: "a" | "b" }[]>();
  const push = (key: string, entry: { index: number; end: "a" | "b" }): void => {
    const arr = groups.get(key);
    if (arr) arr.push(entry);
    else groups.set(key, [entry]);
  };
  for (const it of items) {
    push(it.link.from + "#" + it.portA, { index: it.index, end: "a" });
    push(it.link.to + "#" + it.portB, { index: it.index, end: "b" });
  }
  const spread = new Map<string, number>();
  for (const arr of groups.values()) {
    arr.forEach((entry, idx) => {
      const off = arr.length > 1 ? (idx - (arr.length - 1) / 2) * PORT_SPREAD : 0;
      spread.set(entry.index + entry.end, off);
    });
  }

  // cavi nello stesso corridoio: ognuno riceve una corsia propria (righe 1364-1381)
  const lanes: { horizSeg: boolean; knob: number; lo: number; hi: number; lane: number }[] = [];
  const laneOff = new Map<number, number>();
  for (const it of items) {
    if (it.shape.kind !== "Z") continue;
    const knob = it.shape.knob;
    const da0 = axisOf(Math.cos(it.portA), Math.sin(it.portA));
    const pa0 = borderPoint(it.a.x, it.a.y, half, it.portA);
    const pb0 = borderPoint(it.b.x, it.b.y, half, it.portB);
    const horizSeg = da0.x === 0;
    const lo = horizSeg ? Math.min(pa0.x, pb0.x) : Math.min(pa0.y, pb0.y);
    const hi = horizSeg ? Math.max(pa0.x, pb0.x) : Math.max(pa0.y, pb0.y);
    const dir = horizSeg ? da0.y || 1 : da0.x || 1;
    let lane = 0;
    while (
      lanes.some(
        (o) =>
          o.horizSeg === horizSeg &&
          o.lane === lane &&
          Math.abs(o.knob - knob) < LANE_NEAR &&
          o.lo < hi &&
          lo < o.hi,
      )
    )
      lane++;
    lanes.push({ horizSeg, knob, lo, hi, lane });
    laneOff.set(it.index, lane * LANE_GAP * dir);
  }

  // percorsi finali (righe 1384-1401)
  const out: Record<string, LinkRoute> = {};
  for (const it of items) {
    const da = axisOf(Math.cos(it.portA), Math.sin(it.portA));
    const db = axisOf(Math.cos(it.portB), Math.sin(it.portB));
    const pa0 = borderPoint(it.a.x, it.a.y, half, it.portA);
    const pb0 = borderPoint(it.b.x, it.b.y, half, it.portB);
    const oa = it.shape.kind === "straight" ? (it.shape.oa ?? 0) : 0;
    const ob = it.shape.kind === "straight" ? (it.shape.ob ?? 0) : 0;
    const sprA = spread.get(it.index + "a") ?? 0;
    const sprB = spread.get(it.index + "b") ?? 0;
    // su un tratto rettilineo i due capi devono scostarsi insieme (riga 1395)
    const common = it.shape.kind === "straight" ? (sprA + sprB) / 2 : null;
    const pa = shift(slide(pa0, da, oa), da, common !== null ? common : sprA);
    const pb = shift(slide(pb0, db, ob), db, common !== null ? common : sprB);
    // la forma finale non riapplica oa/ob, già applicati sopra (riga 1399)
    const finalShape: Shape =
      it.shape.kind === "Z"
        ? { kind: "Z", knob: it.shape.knob + (laneOff.get(it.index) ?? 0) }
        : it.shape.kind === "L"
          ? it.shape
          : { kind: "straight" };
    const pts = buildFromShape(pa, da, pb, db, finalShape);
    out[linkKey(it.link)] = {
      from: it.link.from,
      to: it.link.to,
      portA: it.portA,
      portB: it.portB,
      horiz: it.horiz,
      shape: it.shape,
      pts,
      d: roundedPath(pts, ELBOW_R),
      pa: { x: pa.x, y: pa.y },
      pb: { x: pb.x, y: pb.y },
    };
  }
  return out;
}

function samePts(a: readonly Point[], b: readonly Point[]): boolean {
  if (a.length !== b.length) return false;
  return a.every((p, i) => {
    const q = b[i] as Point;
    return Math.abs(p.x - q.x) < 1e-9 && Math.abs(p.y - q.y) < 1e-9;
  });
}

function sameRoutes(a: LinkRoutes, b: LinkRoutes): boolean {
  const ka = Object.keys(a);
  if (ka.length !== Object.keys(b).length) return false;
  return ka.every((k) => {
    const ra = a[k] as LinkRoute;
    const rb = b[k];
    return !!rb && ra.portA === rb.portA && ra.portB === rb.portB && samePts(ra.pts, rb.pts);
  });
}

/** Passate massime di `settleLinks`. */
export const SETTLE_MAX_PASSES = 8;

/**
 * Ripete `layoutLinks` finché i percorsi non cambiano più (al massimo
 * `maxPasses` volte): è lo stato a cui il prototipo arriva dopo qualche
 * fotogramma, visto che ogni cavo conta gli incroci con i percorsi del
 * fotogramma precedente.
 */
export function settleLinks(
  graph: Graph,
  prev: LinkRoutes = {},
  opts: LayoutLinksOptions & { readonly maxPasses?: number } = {},
): { routes: Record<string, LinkRoute>; passes: number } {
  const maxPasses = opts.maxPasses ?? SETTLE_MAX_PASSES;
  let current: LinkRoutes = prev;
  let passes = 0;
  while (passes < maxPasses) {
    const next = layoutLinks(graph, current, opts);
    passes++;
    const stable = sameRoutes(next, current);
    current = next;
    if (stable) break;
  }
  return { routes: current as Record<string, LinkRoute>, passes };
}
```

### `src/etl-layout/nodes.ts`

73 righe

```ts
/**
 * Rettangoli dei nodi e porte. Prototipo, righe 1018-1052 (porte) e CSS
 * righe 626-682 (dimensioni della card e dell'etichetta).
 */
import type { Card } from "../etl-core";
import { CARD, LABEL_GAP, LABEL_H, LABEL_MAX_W, OBST_PAD, PORTS } from "./constants";
import type { Anchor, Axis, Point, Rect } from "./types";

type Positioned = Pick<Card, "x" | "y">;

/** Il quadrato del nodo (CSS `.icon-wrap`, riga 630). */
export function nodeRect(c: Positioned): Rect {
  return { x: c.x, y: c.y, w: CARD, h: CARD };
}

/**
 * L'etichetta sotto il nodo: centrata, larga al più `LABEL_MAX_W`, alta
 * `LABEL_H - LABEL_GAP` (CSS righe 626, 681-682; il prototipo riserva
 * `LABEL_H` sotto la card, riga 1524).
 */
export function labelRect(c: Positioned): Rect {
  return {
    x: c.x + CARD / 2 - LABEL_MAX_W / 2,
    y: c.y + CARD + LABEL_GAP,
    w: LABEL_MAX_W,
    h: LABEL_H - LABEL_GAP,
  };
}

/** Ingombro del nodo con l'etichetta: `CARD × (CARD + LABEL_H)`. */
export function footprint(c: Positioned): Rect {
  return { x: c.x, y: c.y, w: CARD, h: CARD + LABEL_H };
}

/** Ostacolo per l'instradamento dei cavi (righe 1318-1319). */
export function obstacleRect(c: Positioned): Rect {
  return {
    x: c.x - OBST_PAD,
    y: c.y - OBST_PAD,
    w: CARD + OBST_PAD * 2,
    h: CARD + LABEL_H + OBST_PAD * 2,
  };
}

export function nodeCenter(c: Positioned): Point {
  return { x: c.x + CARD / 2, y: c.y + CARD / 2 };
}

/** Riga 1031: punto sul bordo del quadrato di semilato `half` nella direzione `angle`. */
export function borderPoint(cx: number, cy: number, half: number, angle: number): Anchor {
  const dx = Math.cos(angle);
  const dy = Math.sin(angle);
  const m = Math.max(Math.abs(dx), Math.abs(dy)) || 1;
  return { x: cx + (dx / m) * half, y: cy + (dy / m) * half, nx: dx, ny: dy };
}

/** Riga 1050: asse dominante dell'ancoraggio. */
export function axisOf(nx: number, ny: number): Axis {
  return Math.abs(nx) >= Math.abs(ny)
    ? { x: Math.sign(nx) || 1, y: 0 }
    : { x: 0, y: Math.sign(ny) || 1 };
}

/** Le quattro porte di un nodo, nell'ordine di `PORTS`: destra, sotto, sinistra, sopra. */
export function nodePorts(c: Positioned): { angle: number; anchor: Anchor; axis: Axis }[] {
  const center = nodeCenter(c);
  return PORTS.map((angle) => ({
    angle,
    anchor: borderPoint(center.x, center.y, CARD / 2, angle),
    axis: axisOf(Math.cos(angle), Math.sin(angle)),
  }));
}
```

### `src/etl-layout/path.ts`

103 righe

```ts
/** Percorso SVG con raccordi arrotondati. Prototipo, righe 1281-1298. */
import { EPS } from "./constants";
import type { Point } from "./types";

function f(n: number): string {
  return n.toFixed(2);
}

/** Toglie i punti che coincidono con il precedente (riga 1282). */
export function cleanPoints(pts: readonly Point[]): Point[] {
  return pts.filter((p, i) => {
    if (i === 0) return true;
    const q = pts[i - 1] as Point;
    return Math.hypot(p.x - q.x, p.y - q.y) > EPS;
  });
}

/**
 * Riga 1281: `M`, poi per ogni snodo un tratto `L` fino all'inizio del
 * raccordo e una curva `Q` con il vertice come punto di controllo. Il
 * raggio è limitato alla metà dei due tratti adiacenti.
 */
export function roundedPath(pts: readonly Point[], r: number): string {
  const clean = cleanPoints(pts);
  if (clean.length < 3) return "M " + clean.map((p) => f(p.x) + " " + f(p.y)).join(" L ");
  const first = clean[0] as Point;
  let d = "M " + f(first.x) + " " + f(first.y);
  for (let i = 1; i < clean.length - 1; i++) {
    const prev = clean[i - 1] as Point;
    const cur = clean[i] as Point;
    const next = clean[i + 1] as Point;
    const l1 = Math.hypot(cur.x - prev.x, cur.y - prev.y);
    const l2 = Math.hypot(next.x - cur.x, next.y - cur.y);
    const rr = Math.min(r, l1 / 2, l2 / 2);
    const a = {
      x: cur.x + ((prev.x - cur.x) / (l1 || 1)) * rr,
      y: cur.y + ((prev.y - cur.y) / (l1 || 1)) * rr,
    };
    const b = {
      x: cur.x + ((next.x - cur.x) / (l2 || 1)) * rr,
      y: cur.y + ((next.y - cur.y) / (l2 || 1)) * rr,
    };
    d +=
      " L " +
      f(a.x) +
      " " +
      f(a.y) +
      " Q " +
      f(cur.x) +
      " " +
      f(cur.y) +
      " " +
      f(b.x) +
      " " +
      f(b.y);
  }
  const last = clean[clean.length - 1] as Point;
  d += " L " + f(last.x) + " " + f(last.y);
  return d;
}

/** Un tratto del percorso arrotondato: rettilineo o curva quadratica. */
export type PathPiece =
  | { readonly kind: "line"; readonly from: Point; readonly to: Point }
  | { readonly kind: "quad"; readonly from: Point; readonly ctrl: Point; readonly to: Point };

/**
 * Gli stessi tratti che `roundedPath` scrive nella stringa, come
 * geometria (per l'individuazione del cavo sotto un punto).
 */
export function roundedPieces(pts: readonly Point[], r: number): PathPiece[] {
  const clean = cleanPoints(pts);
  const pieces: PathPiece[] = [];
  if (clean.length < 2) return pieces;
  if (clean.length < 3) {
    for (let i = 0; i < clean.length - 1; i++)
      pieces.push({ kind: "line", from: clean[i] as Point, to: clean[i + 1] as Point });
    return pieces;
  }
  let pen = clean[0] as Point;
  for (let i = 1; i < clean.length - 1; i++) {
    const prev = clean[i - 1] as Point;
    const cur = clean[i] as Point;
    const next = clean[i + 1] as Point;
    const l1 = Math.hypot(cur.x - prev.x, cur.y - prev.y);
    const l2 = Math.hypot(next.x - cur.x, next.y - cur.y);
    const rr = Math.min(r, l1 / 2, l2 / 2);
    const a = {
      x: cur.x + ((prev.x - cur.x) / (l1 || 1)) * rr,
      y: cur.y + ((prev.y - cur.y) / (l1 || 1)) * rr,
    };
    const b = {
      x: cur.x + ((next.x - cur.x) / (l2 || 1)) * rr,
      y: cur.y + ((next.y - cur.y) / (l2 || 1)) * rr,
    };
    pieces.push({ kind: "line", from: pen, to: a });
    pieces.push({ kind: "quad", from: a, ctrl: cur, to: b });
    pen = b;
  }
  pieces.push({ kind: "line", from: pen, to: clean[clean.length - 1] as Point });
  return pieces;
}
```

### `src/etl-layout/placement.ts`

200 righe

```ts
/**
 * Posizionamento dei nodi generati, come `PositionFn` compatibili con
 * etl-core (`spawnOutput`, `refreshOutput`, `connect`, `detachStep`, ...).
 * Prototipo, righe 1675-1707 (`relocateAfter`), 1730-1772
 * (`spawnOutput`), 1853-1869 (`insertOnLink`), 2175-2181 (`detachStep`),
 * 1826-1834 (riposizionamento dopo `performMerge`).
 *
 * Una `PositionFn` sceglie solo il punto del nuovo nodo; nel prototipo
 * subito dopo gli altri nodi si scansano (o, in Organizzato, il nuovo
 * nodo prende la sua postazione). Quel secondo passo è `settleNewNode`.
 */
import type { Card, Graph, Link, PositionFn } from "../etl-core";
import {
  CARD,
  DETACH_OFFSET_Y,
  DISPLACE_BOTTOM,
  GRID,
  LABEL_H,
  OUTPUT_EDGE_MARGIN,
  OUTPUT_OFFSET_X,
  RELOCATE_STEP_X,
  RELOCATE_STEP_Y,
  RELOCATE_TRIES,
  SLOT_H,
  SLOT_W,
  WORLD_MARGIN,
} from "./constants";
import { DEFAULT_WORLD, freeSpot, overlapsAny, resolveOverlaps, withPositions } from "./free";
import {
  computeSlots,
  firstFreeSlot,
  nearestSlot,
  outputSlotFor,
  placeInSlots,
  setSlot,
  slotTaken,
} from "./slots";
import type { LayoutMode, Point, Size } from "./types";

export interface PlacementOptions {
  readonly mode?: LayoutMode;
  readonly world?: Size;
}

/**
 * Riga 1738-1746: dove nasce l'output del box `anchorId`. In Libero: a
 * destra di 200 (entro il mondo), allineato alla griglia, poi il primo
 * punto libero. In Organizzato: la postazione di `outputSlotFor`.
 */
export function outputPositionFn(opts: PlacementOptions = {}): PositionFn {
  const world = opts.world ?? DEFAULT_WORLD;
  return (graph, anchorId) => {
    const box = graph.cards[anchorId];
    if (!box) return { x: 0, y: 0 };
    const rightX = Math.min(box.x + OUTPUT_OFFSET_X, world.w - CARD - OUTPUT_EDGE_MARGIN);
    if (opts.mode === "grid") {
      const slots = computeSlots(world);
      const slot = outputSlotFor(graph, slots, box);
      const sp = slot >= 0 ? slots[slot] : undefined;
      return sp ? { x: sp.x, y: sp.y } : freeSpot(graph, rightX, box.y, null, world);
    }
    return freeSpot(graph, Math.round(rightX / GRID) * GRID, box.y, null, world);
  };
}

/**
 * Righe 2175-2179: dove finisce un passaggio sganciato dal box
 * `anchorId`. Con un punto di rilascio: centrato lì, entro il mondo;
 * senza: il primo punto libero sotto il box.
 */
export function detachPositionFn(
  opts: PlacementOptions & { readonly dropPoint?: Point } = {},
): PositionFn {
  const world = opts.world ?? DEFAULT_WORLD;
  return (graph, anchorId) => {
    const drop = opts.dropPoint;
    if (drop) {
      return {
        x: Math.max(WORLD_MARGIN, Math.min(world.w - CARD - WORLD_MARGIN, drop.x - CARD / 2)),
        y: Math.max(WORLD_MARGIN, Math.min(world.h - DISPLACE_BOTTOM, drop.y - CARD / 2)),
      };
    }
    const box = graph.cards[anchorId];
    if (!box) return { x: 0, y: 0 };
    return freeSpot(
      graph,
      Math.max(WORLD_MARGIN, Math.min(box.x, world.w - CARD - WORLD_MARGIN)),
      box.y + DETACH_OFFSET_Y,
      null,
      world,
    );
  };
}

/**
 * Riga 1859: una lavorazione inserita su un cavo si colloca a metà strada
 * tra i due estremi. Va applicata al nodo PRIMA di `insertOnLink` di
 * etl-core (a cui si passa `outputPositionFn` per il suo output).
 */
export function insertPosition(graph: Graph, link: Link): Point | null {
  const a = graph.cards[link.from];
  const b = graph.cards[link.to];
  if (!a || !b) return null;
  return { x: Math.round((a.x + b.x) / 2), y: Math.round((a.y + b.y) / 2) };
}

/** Sposta un nodo in `p` (per applicare `insertPosition`). */
export function moveNode(graph: Graph, id: string, p: Point): Graph {
  return withPositions(graph, new Map([[id, p]]));
}

/**
 * Dopo la creazione di un nodo in posizione scelta (righe 1749 e 1766,
 * 1860 e 1868-1869, 2181 e 2193): in Organizzato prende la postazione
 * libera più vicina e tutti si allineano alle postazioni; in Libero gli
 * altri si scansano e si riallineano alla griglia, mentre lui resta dov'è.
 */
export function settleNewNode(graph: Graph, id: string, opts: PlacementOptions = {}): Graph {
  const world = opts.world ?? DEFAULT_WORLD;
  const c = graph.cards[id];
  if (!c) return graph;
  if (opts.mode === "grid") {
    const slots = computeSlots(world);
    const withSlotAssigned = setSlot(graph, id, firstFreeSlot(graph, slots, c.x, c.y, id));
    return placeInSlots(withSlotAssigned, slots);
  }
  return resolveOverlaps(graph, { fixedId: id, snap: true, world });
}

/**
 * Riga 1675: porta `id` a valle di `anchorId` (a destra, poi sopra, poi
 * sotto, fino a 5 passi). Restituisce `null` se nessun posto va bene.
 */
export function relocateAfter(
  graph: Graph,
  anchorId: string,
  id: string,
  opts: PlacementOptions = {},
): Graph | null {
  const world = opts.world ?? DEFAULT_WORLD;
  const a = graph.cards[anchorId];
  const c = graph.cards[id];
  if (!a || !c) return null;
  const grid = opts.mode === "grid";
  const stepX = grid ? SLOT_W : RELOCATE_STEP_X;
  const stepY = grid ? SLOT_H : RELOCATE_STEP_Y;
  const slots = grid ? computeSlots(world) : [];
  for (let k = 1; k <= RELOCATE_TRIES; k++) {
    const cands: Point[] = [
      { x: a.x + stepX * k, y: a.y },
      { x: a.x + stepX * k, y: a.y - stepY },
      { x: a.x + stepX * k, y: a.y + stepY },
    ];
    for (const t of cands) {
      if (grid) {
        const idx = nearestSlot(graph, slots, t.x, t.y, id, false);
        if (idx < 0) continue;
        const sp = slots[idx] as Point;
        if (Math.abs(sp.x - t.x) > SLOT_W / 2 || Math.abs(sp.y - t.y) > SLOT_H / 2) continue;
        if (slotTaken(graph, idx, id)) continue;
        return moveNode(setSlot(graph, id, idx), id, sp);
      }
      if (
        t.x < WORLD_MARGIN ||
        t.y < WORLD_MARGIN ||
        t.x > world.w - CARD - WORLD_MARGIN ||
        t.y > world.h - CARD - LABEL_H - WORLD_MARGIN
      )
        continue;
      if (overlapsAny(graph, t.x, t.y, id)) continue;
      return moveNode(graph, id, t);
    }
  }
  return null;
}

/**
 * Righe 1826-1834: dopo una fusione, il box risultante va a valle del suo
 * primo ingresso; i suoi output restano a valle e si spostano solo se ora
 * starebbero alle spalle del box o sopra altri nodi.
 */
export function relocateAfterMerge(
  graph: Graph,
  targetId: string,
  opts: PlacementOptions = {},
): Graph {
  const ins = graph.links.filter((l) => l.to === targetId);
  const first = ins[0];
  if (!first) return graph;
  let next = relocateAfter(graph, first.from, targetId, opts) ?? graph;
  for (const l of next.links.filter((x) => x.from === targetId)) {
    const o = next.cards[l.to];
    const b = next.cards[targetId] as Card;
    if (o && (o.x <= b.x || overlapsAny(next, o.x, o.y, l.to))) {
      next = relocateAfter(next, targetId, l.to, opts) ?? next;
    }
  }
  return next;
}
```

### `src/etl-layout/routing.ts`

408 righe

```ts
/**
 * Instradamento di un cavo. Prototipo, righe 1054-1279: forme candidate
 * (dritta, a L, a Z), costruzione dei punti senza segmenti obliqui né
 * inversioni, costo e scelta con stabilità.
 */
import {
  CROSS_MARGIN,
  CROSS_MIN_OVERLAP,
  CROSS_SAME_TRACK,
  EPS,
  KNOB_OBSTACLE_MARGIN,
  KNOB_STEP,
  KNOB_STEPS,
  MAX_BENDS,
  PORT_SLACK_INSET,
  PORTS,
  SCORE,
  SELF_OBSTACLE_INSET,
  STRAIGHT_EPS,
  STUB,
} from "./constants";
import { axisOf, borderPoint } from "./nodes";
import type { Anchor, Axis, Point, Rect, RouteChoice, Shape } from "./types";

/** Riga 1056: un tratto orizzontale o verticale attraversa l'interno di `r`. */
export function segHitsRect(x1: number, y1: number, x2: number, y2: number, r: Rect): boolean {
  if (Math.abs(y1 - y2) < EPS) {
    const lo = Math.min(x1, x2);
    const hi = Math.max(x1, x2);
    return y1 > r.y && y1 < r.y + r.h && hi > r.x && lo < r.x + r.w;
  }
  if (Math.abs(x1 - x2) < EPS) {
    const lo = Math.min(y1, y2);
    const hi = Math.max(y1, y2);
    return x1 > r.x && x1 < r.x + r.w && hi > r.y && lo < r.y + r.h;
  }
  return false;
}

/** Riga 1067: quante coppie (tratto, ostacolo) si intersecano. */
export function routeCost(pts: readonly Point[], obstacles: readonly Rect[]): number {
  let hits = 0;
  for (let i = 0; i < pts.length - 1; i++) {
    const p = pts[i] as Point;
    const q = pts[i + 1] as Point;
    for (const r of obstacles) if (segHitsRect(p.x, p.y, q.x, q.y, r)) hits++;
  }
  return hits;
}

/** Riga 1075: percorso a Z con lo snodo intermedio in `knob`. */
export function buildRoute(pa: Point, da: Axis, pb: Point, db: Axis, knob: number): Point[] {
  const p1 = { x: pa.x + da.x * STUB, y: pa.y + da.y * STUB };
  const p2 = { x: pb.x + db.x * STUB, y: pb.y + db.y * STUB };
  const pts: Point[] = [{ x: pa.x, y: pa.y }, p1];
  if (da.x !== 0) pts.push({ x: knob, y: p1.y }, { x: knob, y: p2.y });
  else pts.push({ x: p1.x, y: knob }, { x: p2.x, y: knob });
  pts.push(p2, { x: pb.x, y: pb.y });
  return pts;
}

function dedupe(pts: readonly Point[]): Point[] {
  return pts.filter((q, i) => {
    if (i === 0) return true;
    const p = pts[i - 1] as Point;
    return Math.hypot(q.x - p.x, q.y - p.y) > EPS;
  });
}

/** Riga 1089. */
export function countBends(pts: readonly Point[]): number {
  const p = dedupe(pts);
  let bends = 0;
  for (let i = 1; i < p.length - 1; i++) {
    const a = p[i - 1] as Point;
    const b = p[i] as Point;
    const c = p[i + 1] as Point;
    const h1 = Math.abs(b.y - a.y) < EPS;
    const h2 = Math.abs(c.y - b.y) < EPS;
    if (h1 !== h2) bends++;
  }
  return bends;
}

/** Riga 1100: incrocio o sovrapposizione sullo stesso binario tra due tratti. */
export function segCross(a: Point, b: Point, c: Point, d: Point): boolean {
  const m = CROSS_MARGIN;
  const sH = Math.abs(a.y - b.y) < EPS;
  const tH = Math.abs(c.y - d.y) < EPS;
  if (sH && !tH) {
    const x = c.x;
    const y = a.y;
    return (
      x > Math.min(a.x, b.x) + m &&
      x < Math.max(a.x, b.x) - m &&
      y > Math.min(c.y, d.y) + m &&
      y < Math.max(c.y, d.y) - m
    );
  }
  if (!sH && tH) return segCross(c, d, a, b);
  if (sH && tH && Math.abs(a.y - c.y) < CROSS_SAME_TRACK) {
    const lo = Math.max(Math.min(a.x, b.x), Math.min(c.x, d.x));
    const hi = Math.min(Math.max(a.x, b.x), Math.max(c.x, d.x));
    return hi - lo > CROSS_MIN_OVERLAP;
  }
  if (!sH && !tH && Math.abs(a.x - c.x) < CROSS_SAME_TRACK) {
    const lo = Math.max(Math.min(a.y, b.y), Math.min(c.y, d.y));
    const hi = Math.min(Math.max(a.y, b.y), Math.max(c.y, d.y));
    return hi - lo > CROSS_MIN_OVERLAP;
  }
  return false;
}

/** Riga 1121. */
export function countCrossings(
  pts: readonly Point[],
  others: readonly (readonly Point[])[],
): number {
  let n = 0;
  for (let i = 0; i < pts.length - 1; i++)
    for (const o of others)
      for (let j = 0; j < o.length - 1; j++)
        if (segCross(pts[i] as Point, pts[i + 1] as Point, o[j] as Point, o[j + 1] as Point)) n++;
  return n;
}

/** Riga 1131: quanto il tracciato esce dal rettangolo che unisce i due agganci. */
export function overshoot(pts: readonly Point[]): number {
  const qa = pts[0] as Point;
  const qb = pts[pts.length - 1] as Point;
  const minX = Math.min(qa.x, qb.x);
  const maxX = Math.max(qa.x, qb.x);
  const minY = Math.min(qa.y, qb.y);
  const maxY = Math.max(qa.y, qb.y);
  let o = 0;
  for (const p of pts) {
    o +=
      Math.max(0, minX - p.x) +
      Math.max(0, p.x - maxX) +
      Math.max(0, minY - p.y) +
      Math.max(0, p.y - maxY);
  }
  return o;
}

/** Riga 1142: lunghezza in metrica di Manhattan. */
export function routeLength(pts: readonly Point[]): number {
  let L = 0;
  for (let i = 0; i < pts.length - 1; i++) {
    const p = pts[i] as Point;
    const q = pts[i + 1] as Point;
    L += Math.abs(q.x - p.x) + Math.abs(q.y - p.y);
  }
  return L;
}

/** Riga 1148: scorre l'aggancio lungo il lato (perpendicolare all'asse di uscita). */
export function slide<T extends Point>(pt: T, axis: Axis, off: number): T {
  if (!off) return { ...pt };
  return axis.x !== 0 ? { ...pt, y: pt.y + off } : { ...pt, x: pt.x + off };
}

/** Riga 1157: nessun segmento obliquo. */
export function orthogonalize(pts: readonly Point[], da: Axis): Point[] {
  const out: Point[] = [pts[0] as Point];
  let horizFirst = da.x !== 0;
  for (let i = 1; i < pts.length; i++) {
    const p = out[out.length - 1] as Point;
    const q = pts[i] as Point;
    const dx = Math.abs(q.x - p.x);
    const dy = Math.abs(q.y - p.y);
    if (dx > EPS && dy > EPS) {
      out.push(horizFirst ? { x: q.x, y: p.y } : { x: p.x, y: q.y });
    }
    out.push(q);
    const last = out[out.length - 1] as Point;
    const prev = out[out.length - 2] as Point;
    horizFirst = Math.abs(last.y - prev.y) < EPS;
  }
  return out;
}

/** Riga 1173: elimina inversioni e punti superflui sulla stessa retta. */
export function removeReversals(pts: readonly Point[]): Point[] {
  const p = pts.slice();
  let changed = true;
  while (changed && p.length > 2) {
    changed = false;
    for (let i = 1; i < p.length - 1; i++) {
      const a = p[i - 1] as Point;
      const b = p[i] as Point;
      const c = p[i + 1] as Point;
      const sameX = Math.abs(a.x - b.x) < EPS && Math.abs(b.x - c.x) < EPS;
      const sameY = Math.abs(a.y - b.y) < EPS && Math.abs(b.y - c.y) < EPS;
      if (!sameX && !sameY) continue;
      const dot = (b.x - a.x) * (c.x - b.x) + (b.y - a.y) * (c.y - b.y);
      if (dot <= 0 || sameX || sameY) {
        p.splice(i, 1);
        changed = true;
        break;
      }
    }
  }
  return p;
}

/** Riga 1190. */
export function buildFromShape(pa: Point, da: Axis, pb: Point, db: Axis, shape: Shape): Point[] {
  const oa = shape.kind === "straight" ? (shape.oa ?? 0) : 0;
  const ob = shape.kind === "straight" ? (shape.ob ?? 0) : 0;
  const qa = slide(pa, da, oa);
  const qb = slide(pb, db, ob);
  let pts: Point[];
  if (shape.kind === "straight") {
    pts = [
      { x: qa.x, y: qa.y },
      { x: qb.x, y: qb.y },
    ];
  } else if (shape.kind === "L") {
    const corner = shape.first === "h" ? { x: qb.x, y: qa.y } : { x: qa.x, y: qb.y };
    pts = [{ x: qa.x, y: qa.y }, corner, { x: qb.x, y: qb.y }];
  } else {
    pts = buildRoute(qa, da, qb, db, shape.knob);
  }
  return removeReversals(orthogonalize(pts, da));
}

function dirSign(p: Point, q: Point): Axis {
  return { x: Math.sign(q.x - p.x), y: Math.sign(q.y - p.y) };
}

/** Riga 1203: penalizza i tracciati che tornano indietro rispetto alle porte. */
export function backtrack(pts: readonly Point[], da: Axis, db: Axis): number {
  if (pts.length < 2) return 0;
  let pen = 0;
  const d0 = dirSign(pts[0] as Point, pts[1] as Point);
  if ((da.x && d0.x && d0.x !== da.x) || (da.y && d0.y && d0.y !== da.y)) pen++;
  const n = pts.length;
  const dn = dirSign(pts[n - 2] as Point, pts[n - 1] as Point);
  if ((db.x && dn.x && dn.x === db.x) || (db.y && dn.y && dn.y === db.y)) pen++;
  return pen;
}

/** Riga 1214: forme candidate per una coppia di agganci. */
export function shapeCandidates(
  pa: Point,
  da: Axis,
  pb: Point,
  db: Axis,
  obstacles: readonly Rect[],
  slack: number,
): Shape[] {
  const out: Shape[] = [];
  if (da.x && db.x && da.x === -db.x) {
    const dy = pb.y - pa.y;
    if (Math.abs(dy) < STRAIGHT_EPS) out.push({ kind: "straight" });
    else if (Math.abs(dy) <= slack * 2) out.push({ kind: "straight", oa: dy / 2, ob: -dy / 2 });
  }
  if (da.y && db.y && da.y === -db.y) {
    const dx = pb.x - pa.x;
    if (Math.abs(dx) < STRAIGHT_EPS) out.push({ kind: "straight" });
    else if (Math.abs(dx) <= slack * 2) out.push({ kind: "straight", oa: dx / 2, ob: -dx / 2 });
  }
  if (da.x && db.y) out.push({ kind: "L", first: "h" });
  if (da.y && db.x) out.push({ kind: "L", first: "v" });
  const p1 = { x: pa.x + da.x * STUB, y: pa.y + da.y * STUB };
  const p2 = { x: pb.x + db.x * STUB, y: pb.y + db.y * STUB };
  const base = da.x !== 0 ? (p1.x + p2.x) / 2 : (p1.y + p2.y) / 2;
  const horiz = da.x !== 0;
  const knobs = [base];
  for (const r of obstacles) {
    if (horiz) knobs.push(r.x - KNOB_OBSTACLE_MARGIN, r.x + r.w + KNOB_OBSTACLE_MARGIN);
    else knobs.push(r.y - KNOB_OBSTACLE_MARGIN, r.y + r.h + KNOB_OBSTACLE_MARGIN);
  }
  for (let k = 1; k <= KNOB_STEPS; k++) knobs.push(base + k * KNOB_STEP, base - k * KNOB_STEP);
  for (const knob of knobs) out.push({ kind: "Z", knob });
  return out;
}

export interface ChooseRouteOptions {
  /** Porte del percorso precedente: il cavo le conserva finché non peggiora. */
  readonly prevPortA?: number | undefined;
  readonly prevPortB?: number | undefined;
  /** Percorsi degli altri cavi, per contare gli incroci. */
  readonly others?: readonly (readonly Point[])[];
  /** Snodi ammessi (predefinito: `MAX_BENDS` del prototipo). */
  readonly maxBends?: number;
}

/** Ogni candidato valutato da `chooseRoute`, con i suoi punti. */
export interface RouteCandidate extends RouteChoice {
  readonly pts: readonly Point[];
}

/**
 * Tutti i candidati di `chooseRoute` nell'ordine di valutazione del
 * prototipo (righe 1251-1274), con il loro punteggio.
 */
export function routeCandidates(
  a: Point,
  b: Point,
  half: number,
  obstacles: readonly Rect[],
  opts: ChooseRouteOptions = {},
): RouteCandidate[] {
  const maxBends = opts.maxBends ?? MAX_BENDS;
  const others = opts.others ?? [];
  const inset = SELF_OBSTACLE_INSET;
  const selfObs: Rect[] = obstacles.concat([
    {
      x: a.x - half + inset,
      y: a.y - half + inset,
      w: half * 2 - inset * 2,
      h: half * 2 - inset * 2,
    },
    {
      x: b.x - half + inset,
      y: b.y - half + inset,
      w: half * 2 - inset * 2,
      h: half * 2 - inset * 2,
    },
  ]);
  const out: RouteCandidate[] = [];
  for (const pA of PORTS) {
    const pa: Anchor = borderPoint(a.x, a.y, half, pA);
    const da = axisOf(Math.cos(pA), Math.sin(pA));
    for (const pB of PORTS) {
      const pb: Anchor = borderPoint(b.x, b.y, half, pB);
      const db = axisOf(Math.cos(pB), Math.sin(pB));
      for (const shape of shapeCandidates(pa, da, pb, db, obstacles, half - PORT_SLACK_INSET)) {
        const pts = buildFromShape(pa, da, pb, db, shape);
        const cost = routeCost(pts, selfObs);
        const bends = countBends(pts);
        const overBends = Math.max(0, bends - maxBends);
        const back = backtrack(pts, da, db);
        const changePen = (pA !== opts.prevPortA ? 1 : 0) + (pB !== opts.prevPortB ? 1 : 0);
        const sameSide = da.x === db.x && da.y === db.y ? 1 : 0;
        const cr = others.length ? countCrossings(pts, others) : 0;
        const score =
          cost * SCORE.obstacle +
          overBends * SCORE.overBends +
          cr * SCORE.crossing +
          back * SCORE.backtrack +
          sameSide * SCORE.sameSide +
          overshoot(pts) * SCORE.overshoot +
          routeLength(pts) * SCORE.length +
          changePen * SCORE.portChange;
        out.push({
          score,
          portA: pA,
          portB: pB,
          shape,
          horiz: da.x !== 0,
          cost,
          bends,
          back,
          cr,
          pts,
        });
      }
    }
  }
  return out;
}

/**
 * Riga 1243: sceglie porte e forma. `a`, `b` sono i centri dei due nodi,
 * `half` il semilato. Gerarchia delle penalità (dai pesi di `SCORE`):
 * nessun nodo attraversato, limite di snodi, nessun incrocio, nessun
 * ripiegamento, nessuna U, poi fuoriuscita e lunghezza minima.
 *
 * Stabilità (righe 1275-1277): il percorso con le porte precedenti vince
 * finché non attraversa nulla, non ripiega e non costa più snodi o
 * incroci dell'alternativa migliore.
 */
export function chooseRoute(
  a: Point,
  b: Point,
  half: number,
  obstacles: readonly Rect[],
  opts: ChooseRouteOptions = {},
): RouteChoice | null {
  let best: RouteCandidate | null = null;
  let keep: RouteCandidate | null = null;
  for (const cand of routeCandidates(a, b, half, obstacles, opts)) {
    if (!best || cand.score < best.score) best = cand;
    if (
      cand.portA === opts.prevPortA &&
      cand.portB === opts.prevPortB &&
      (!keep || cand.score < keep.score)
    )
      keep = cand;
  }
  const pick =
    keep &&
    best &&
    keep.cost === 0 &&
    keep.back === 0 &&
    keep.bends <= best.bends &&
    keep.cr <= best.cr
      ? keep
      : best;
  if (!pick) return null;
  const { pts: _pts, ...choice } = pick;
  void _pts;
  return choice;
}
```

### `src/etl-layout/slots.ts`

176 righe

```ts
/**
 * Modalità Organizzato: postazioni fisse, occupazione, scambio di posto.
 * Prototipo, righe 1709-1728 (`outputSlotFor`), 2679-2685
 * (`assignSlotsKeepingOrder`), 2093-2100 (rilascio con scambio),
 * 3907-3951 (postazioni).
 */
import type { Card, Graph } from "../etl-core";
import { CARD, OUTPUT_SLOT_DY_WEIGHT, SLOT_H, SLOT_M, SLOT_W } from "./constants";
import { DEFAULT_WORLD } from "./free";
import type { Point, Size } from "./types";

/** Riga 3910: la griglia delle postazioni che entrano nel mondo, riga per riga. */
export function computeSlots(world: Size = DEFAULT_WORLD): Point[] {
  const cols = Math.max(1, Math.floor((world.w - SLOT_M * 2) / SLOT_W));
  const rows = Math.max(1, Math.floor((world.h - SLOT_M * 2) / SLOT_H));
  const slots: Point[] = [];
  for (let r = 0; r < rows; r++)
    for (let c = 0; c < cols; c++)
      slots.push({ x: SLOT_M + c * SLOT_W + (SLOT_W - CARD) / 2, y: SLOT_M + r * SLOT_H });
  return slots;
}

/** Riga 3918: la postazione `idx` è occupata da un nodo diverso da `exceptId`? */
export function slotTaken(graph: Graph, idx: number, exceptId?: string | null): boolean {
  return Object.keys(graph.cards).some(
    (id) => id !== exceptId && (graph.cards[id] as Card).slot === idx,
  );
}

/** Riga 3921. Restituisce -1 se non c'è nessuna postazione (libera, con `freeOnly`). */
export function nearestSlot(
  graph: Graph,
  slots: readonly Point[],
  x: number,
  y: number,
  exceptId: string | null | undefined,
  freeOnly: boolean,
): number {
  let best = -1;
  let bestD = Infinity;
  slots.forEach((sp, i) => {
    if (freeOnly && slotTaken(graph, i, exceptId)) return;
    const d = Math.hypot(sp.x - x, sp.y - y);
    if (d < bestD) {
      bestD = d;
      best = i;
    }
  });
  return best;
}

/** Riga 3930: la libera più vicina, altrimenti la più vicina in assoluto. */
export function firstFreeSlot(
  graph: Graph,
  slots: readonly Point[],
  nearX: number,
  nearY: number,
  exceptId?: string | null,
): number {
  const i = nearestSlot(graph, slots, nearX, nearY, exceptId, true);
  return i >= 0 ? i : nearestSlot(graph, slots, nearX, nearY, exceptId, false);
}

function withSlot(card: Card, slot: number | undefined): Card {
  const { slot: _old, ...rest } = card;
  void _old;
  return slot === undefined ? rest : { ...rest, slot };
}

/** Assegna (o toglie, con `undefined`) la postazione di un nodo. */
export function setSlot(graph: Graph, id: string, slot: number | undefined): Graph {
  const c = graph.cards[id];
  if (!c) return graph;
  return { ...graph, cards: { ...graph.cards, [id]: withSlot(c, slot) } };
}

/** Riga 3934: ogni nodo con una postazione valida si sposta lì. */
export function placeInSlots(graph: Graph, slots: readonly Point[]): Graph {
  let changed = false;
  const cards: Record<string, Card> = {};
  for (const [id, c] of Object.entries(graph.cards)) {
    const sp = c.slot === undefined ? undefined : slots[c.slot];
    if (sp && (sp.x !== c.x || sp.y !== c.y)) {
      cards[id] = { ...c, x: sp.x, y: sp.y };
      changed = true;
    } else cards[id] = c;
  }
  return changed ? { ...graph, cards } : graph;
}

/**
 * Righe 3943-3951 (`assignSlots`) e 2679-2685 (`assignSlotsKeepingOrder`,
 * identica): chi sta più in alto a sinistra sceglie per primo la
 * postazione libera più vicina; poi ognuno si sposta nella sua.
 */
export function assignSlots(graph: Graph, world: Size = DEFAULT_WORLD): Graph {
  const slots = computeSlots(world);
  let next: Graph = graph;
  for (const id of Object.keys(graph.cards)) next = setSlot(next, id, undefined);
  const order = Object.keys(graph.cards).sort((A, B) => {
    const a = graph.cards[A] as Card;
    const b = graph.cards[B] as Card;
    return a.y - b.y || a.x - b.x;
  });
  for (const id of order) {
    const c = next.cards[id] as Card;
    next = setSlot(next, id, firstFreeSlot(next, slots, c.x, c.y, id));
  }
  return placeInSlots(next, slots);
}

/** Riga 3962: tornando in modalità Libero le postazioni si dimenticano. */
export function clearSlots(graph: Graph): Graph {
  let next = graph;
  for (const id of Object.keys(graph.cards)) next = setSlot(next, id, undefined);
  return next;
}

/**
 * Righe 2093-2100: un nodo rilasciato in (x, y) occupa la postazione più
 * vicina; se è occupata, chi la occupa prende la postazione lasciata
 * libera (scambio di posto).
 */
export function dropInSlot(
  graph: Graph,
  id: string,
  x: number,
  y: number,
  world: Size = DEFAULT_WORLD,
): Graph {
  const c = graph.cards[id];
  if (!c) return graph;
  const slots = computeSlots(world);
  let next = graph;
  const idx = nearestSlot(graph, slots, x, y, id, false);
  if (idx >= 0) {
    const occupant = Object.keys(graph.cards).find(
      (o) => o !== id && (graph.cards[o] as Card).slot === idx,
    );
    if (occupant) next = setSlot(next, occupant, c.slot);
    next = setSlot(next, id, idx);
  }
  return placeInSlots(next, slots);
}

/**
 * Riga 1709: prima postazione libera nel verso del flusso, a destra del
 * box e sulla stessa riga se possibile; altrimenti la libera più vicina.
 */
export function outputSlotFor(graph: Graph, slots: readonly Point[], box: Card): number {
  let best = -1;
  let bestScore = Infinity;
  slots.forEach((sp, i) => {
    if (slotTaken(graph, i)) return;
    const dx = sp.x - box.x;
    const dy = Math.abs(sp.y - box.y);
    if (dx <= 0) return;
    const score = dy * OUTPUT_SLOT_DY_WEIGHT + Math.abs(dx - SLOT_W);
    if (score < bestScore) {
      bestScore = score;
      best = i;
    }
  });
  if (best === -1) {
    slots.forEach((sp, i) => {
      if (slotTaken(graph, i)) return;
      const d = Math.hypot(sp.x - box.x, sp.y - box.y);
      if (d < bestScore) {
        bestScore = d;
        best = i;
      }
    });
  }
  return best;
}
```

### `src/etl-layout/types.ts`

76 righe

```ts
/** Tipi della geometria del canvas. */

export interface Point {
  readonly x: number;
  readonly y: number;
}

export interface Rect {
  readonly x: number;
  readonly y: number;
  readonly w: number;
  readonly h: number;
}

export interface Size {
  readonly w: number;
  readonly h: number;
}

/** Direzione di uscita da una porta: uno dei due componenti è 0, l'altro ±1. */
export interface Axis {
  readonly x: number;
  readonly y: number;
}

/** Punto di aggancio sul bordo di un nodo, con la direzione dell'angolo di porta. */
export interface Anchor extends Point {
  readonly nx: number;
  readonly ny: number;
}

/** Forme di tracciato (prototipo, righe 1147-1240): dritto (0 snodi), a L (1), a Z (2). */
export type Shape =
  | { readonly kind: "straight"; readonly oa?: number; readonly ob?: number }
  | { readonly kind: "L"; readonly first: "h" | "v" }
  | { readonly kind: "Z"; readonly knob: number };

/** Esito di `chooseRoute` (prototipo, riga 1269). */
export interface RouteChoice {
  readonly score: number;
  readonly portA: number;
  readonly portB: number;
  readonly shape: Shape;
  readonly horiz: boolean;
  /** Attraversamenti di nodi. */
  readonly cost: number;
  readonly bends: number;
  /** Ripiegamenti rispetto alle porte. */
  readonly back: number;
  /** Incroci con altri cavi. */
  readonly cr: number;
}

/** Percorso a regime di un cavo: l'equivalente statico di `linkState[key]` del prototipo. */
export interface LinkRoute {
  readonly from: string;
  readonly to: string;
  readonly portA: number;
  readonly portB: number;
  readonly horiz: boolean;
  readonly shape: Shape;
  /** Punti del percorso (dopo scostamenti e corsie). */
  readonly pts: readonly Point[];
  /** Percorso SVG con i raccordi arrotondati. */
  readonly d: string;
  /** Agganci effettivi alle due estremità. */
  readonly pa: Point;
  readonly pb: Point;
}

/** Percorsi di tutti i cavi, per chiave `from|to`. */
export type LinkRoutes = Readonly<Record<string, LinkRoute>>;

/** 'free' = Libero, 'grid' = Organizzato (prototipo, riga 3907). */
export type LayoutMode = "free" | "grid";
```

