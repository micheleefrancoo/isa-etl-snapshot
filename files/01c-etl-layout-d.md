# 01c-etl-layout-d.md

File in questo blocco:

- `src/etl-layout/placement.ts`
- `src/etl-layout/routing.ts`
- `src/etl-layout/slots.ts`
- `src/etl-layout/types.ts`

---

### `src/etl-layout/placement.ts`

202 righe

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
  DROP_BOTTOM,
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
 * senza: il primo punto libero sotto il box, a `DETACH_OFFSET_Y` (correzione
 * intenzionale della Fase 2.1: nel prototipo il punto toccava sempre il box
 * e finiva di lato).
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
        y: Math.max(WORLD_MARGIN, Math.min(world.h - DROP_BOTTOM, drop.y - CARD / 2)),
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

432 righe

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
  // Correzione intenzionale (Fase 2.1, NOTE_DIVERGENZE.md): anche sotto
  // STRAIGHT_EPS gli agganci scorrono di metà ciascuno, così il cavo è
  // perfettamente dritto (nel prototipo restava uno scalino).
  if (da.x && db.x && da.x === -db.x) {
    const dy = pb.y - pa.y;
    if (dy === 0) out.push({ kind: "straight" });
    else if (Math.abs(dy) < STRAIGHT_EPS || Math.abs(dy) <= slack * 2)
      out.push({ kind: "straight", oa: dy / 2, ob: -dy / 2 });
  }
  if (da.y && db.y && da.y === -db.y) {
    const dx = pb.x - pa.x;
    if (dx === 0) out.push({ kind: "straight" });
    else if (Math.abs(dx) < STRAIGHT_EPS || Math.abs(dx) <= slack * 2)
      out.push({ kind: "straight", oa: dx / 2, ob: -dx / 2 });
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

function selfObstacles(a: Point, b: Point, half: number, obstacles: readonly Rect[]): Rect[] {
  const inset = SELF_OBSTACLE_INSET;
  const side = half * 2 - inset * 2;
  return obstacles.concat([
    { x: a.x - half + inset, y: a.y - half + inset, w: side, h: side },
    { x: b.x - half + inset, y: b.y - half + inset, w: side, h: side },
  ]);
}

function scoreOf(
  pA: number,
  pB: number,
  shape: Shape,
  a: Point,
  b: Point,
  half: number,
  selfObs: readonly Rect[],
  opts: ChooseRouteOptions,
): RouteCandidate {
  const maxBends = opts.maxBends ?? MAX_BENDS;
  const others = opts.others ?? [];
  const pa: Anchor = borderPoint(a.x, a.y, half, pA);
  const da = axisOf(Math.cos(pA), Math.sin(pA));
  const pb: Anchor = borderPoint(b.x, b.y, half, pB);
  const db = axisOf(Math.cos(pB), Math.sin(pB));
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
  return { score, portA: pA, portB: pB, shape, horiz: da.x !== 0, cost, bends, back, cr, pts };
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
  const selfObs = selfObstacles(a, b, half, obstacles);
  const out: RouteCandidate[] = [];
  for (const pA of PORTS) {
    const pa = borderPoint(a.x, a.y, half, pA);
    const da = axisOf(Math.cos(pA), Math.sin(pA));
    for (const pB of PORTS) {
      const pb = borderPoint(b.x, b.y, half, pB);
      const db = axisOf(Math.cos(pB), Math.sin(pB));
      for (const shape of shapeCandidates(pa, da, pb, db, obstacles, half - PORT_SLACK_INSET)) {
        out.push(scoreOf(pA, pB, shape, a, b, half, selfObs, opts));
      }
    }
  }
  return out;
}

/**
 * Valuta un percorso dato (porte e forma) con lo stesso costo di
 * `chooseRoute`: serve a confrontare il percorso attuale di un cavo con
 * l'alternativa (Fase 2.1, convergenza).
 */
export function evaluateRoute(
  a: Point,
  b: Point,
  half: number,
  obstacles: readonly Rect[],
  portA: number,
  portB: number,
  shape: Shape,
  opts: ChooseRouteOptions = {},
): RouteCandidate {
  return scoreOf(portA, portB, shape, a, b, half, selfObstacles(a, b, half, obstacles), opts);
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

187 righe

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
 *
 * Correzione intenzionale (Fase 2.1, NOTE_DIVERGENZE.md): se il nodo
 * rilasciato non aveva una postazione, chi occupava quella di arrivo va
 * nella postazione libera più vicina (nel prototipo restava senza). Più in
 * generale, al termine ogni nodo senza postazione riceve la libera più
 * vicina: ogni nodo ha una postazione e nessuna postazione ha due nodi.
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
  for (const other of Object.keys(next.cards)) {
    const o = next.cards[other] as Card;
    if (o.slot !== undefined) continue;
    next = setSlot(next, other, firstFreeSlot(next, slots, o.x, o.y, other));
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

82 righe

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
  /**
   * Punti del percorso di base, prima di scostamenti e corsie: dipendono
   * solo dalla scelta di questo cavo. Su questi si contano gli incroci
   * (Fase 2.1, convergenza).
   */
  readonly basePts: readonly Point[];
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

