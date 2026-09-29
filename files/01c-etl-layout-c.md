# 01c-etl-layout-c.md

File in questo blocco:

- `src/etl-layout/__tests__/unit.test.ts`
- `src/etl-layout/autoLayout.ts`
- `src/etl-layout/constants.ts`
- `src/etl-layout/free.ts`
- `src/etl-layout/hitTest.ts`
- `src/etl-layout/index.ts`
- `src/etl-layout/links.ts`
- `src/etl-layout/nodes.ts`
- `src/etl-layout/path.ts`

---

### `src/etl-layout/__tests__/unit.test.ts`

317 righe

```ts
import { describe, expect, it } from "vitest";
import { defaultParams, detachStep, insertOnLink } from "../../etl-core";
import type { Card, ComponentId, Graph } from "../../etl-core";
import {
  CARD,
  GRID,
  LABEL_H,
  PORTS,
  WORLD_H,
  WORLD_W,
  anyOverlap,
  assignSlots,
  borderPoint,
  clampPoint,
  computeSlots,
  detachPositionFn,
  displace,
  displaceTarget,
  distanceToRoute,
  dropFree,
  dropInSlot,
  freeSpot,
  insertPosition,
  linkAt,
  moveNode,
  nodeAt,
  nodePorts,
  outputPositionFn,
  relocateAfterMerge,
  resolveOverlaps,
  roundedPath,
  separateWhileDragging,
  settleLinks,
  settleNewNode,
  snapToGrid,
} from "..";

function card(
  id: string,
  comp: ComponentId,
  x: number,
  y: number,
  extra: Partial<Card> = {},
): Card {
  return {
    id,
    kind: comp === "dataset" ? "dataset" : "op",
    components: [comp],
    params: [defaultParams(comp)],
    name: id,
    x,
    y,
    ...extra,
  };
}

function graphOf(cards: Card[], links: { from: string; to: string }[] = []): Graph {
  const rec: Record<string, Card> = {};
  for (const c of cards) rec[c.id] = c;
  return { cards: rec, links };
}

describe("porte", () => {
  it("quattro porte al centro dei lati: destra, sotto, sinistra, sopra", () => {
    const ports = nodePorts({ x: 100, y: 200 });
    expect(ports.map((p) => ({ x: Math.round(p.anchor.x), y: Math.round(p.anchor.y) }))).toEqual([
      { x: 188, y: 244 },
      { x: 144, y: 288 },
      { x: 100, y: 244 },
      { x: 144, y: 200 },
    ]);
    expect(ports.map((p) => p.axis)).toEqual([
      { x: 1, y: 0 },
      { x: 0, y: 1 },
      { x: -1, y: 0 },
      { x: 0, y: -1 },
    ]);
  });

  it("borderPoint proietta sul bordo del quadrato anche in diagonale", () => {
    const p = borderPoint(0, 0, 44, Math.PI / 4);
    expect(p.x).toBeCloseTo(44, 9);
    expect(p.y).toBeCloseTo(44, 9);
  });
});

describe("percorso SVG", () => {
  it("due punti: solo M e L", () => {
    expect(
      roundedPath(
        [
          { x: 0, y: 0 },
          { x: 10, y: 0 },
        ],
        11,
      ),
    ).toBe("M 0.00 0.00 L 10.00 0.00");
  });

  it("uno snodo: raccordo Q con raggio limitato a metà del tratto più corto", () => {
    expect(
      roundedPath(
        [
          { x: 0, y: 0 },
          { x: 100, y: 0 },
          { x: 100, y: 10 },
        ],
        11,
      ),
    ).toBe("M 0.00 0.00 L 95.00 0.00 Q 100.00 0.00 100.00 5.00 L 100.00 10.00");
  });
});

describe("individuazione", () => {
  const g = graphOf([card("A", "dataset", 100, 100), card("B", "filter", 140, 120)]);

  it("nodeAt: vince il nodo creato dopo; l'etichetta conta; ignoreId salta il nodo in mano", () => {
    expect(nodeAt(g, { x: 150, y: 150 })).toBe("B");
    expect(nodeAt(g, { x: 150, y: 150 }, "B")).toBe("A");
    expect(nodeAt(g, { x: 102, y: 105 })).toBe("A");
    expect(nodeAt(g, { x: 184, y: 120 + CARD + 12 })).toBe("B");
    expect(nodeAt(g, { x: 20, y: 20 })).toBeNull();
    // etichetta: 8 px sotto il quadrato, larga 96 px centrata (da x - 4 a x + 92)
    const solo = graphOf([card("S", "dataset", 500, 500)]);
    expect(nodeAt(solo, { x: 544, y: 500 + CARD + 7.5 })).toBeNull();
    expect(nodeAt(solo, { x: 544, y: 500 + CARD + 8.5 })).toBe("S");
    expect(nodeAt(solo, { x: 500 - 3.75, y: 600 })).toBe("S");
    expect(nodeAt(solo, { x: 500 - 4.25, y: 600 })).toBeNull();
  });

  it("linkAt: entro 8 px dal percorso, anche sulla curva del raccordo", () => {
    const two = graphOf(
      [card("A", "dataset", 100, 300), card("B", "filter", 400, 300)],
      [{ from: "A", to: "B" }],
    );
    const { routes } = settleLinks(two);
    expect(linkAt(routes, { x: 300, y: 344 + 7 })).toBe("A|B");
    expect(linkAt(routes, { x: 300, y: 344 + 8.25 })).toBeNull();
    const l = settleLinks(
      graphOf(
        [card("A", "dataset", 104, 312), card("B", "filter", 416, 442)],
        [{ from: "A", to: "B" }],
      ),
    ).routes;
    const route = l["A|B"];
    expect(route?.shape.kind).toBe("L");
    // vertice dello snodo a L (460, 356): la curva passa a ~3,2 px dal vertice
    expect(distanceToRoute({ x: 460, y: 356 }, route as NonNullable<typeof route>)).toBeLessThan(4);
  });
});

describe("modalità Libero", () => {
  it("limiti del mondo 2600 × 1600 con margine 6 (valori letterali)", () => {
    expect(clampPoint({ x: 99999, y: 99999 })).toEqual({ x: 2506, y: 1484 });
    expect(clampPoint({ x: -1, y: -1 })).toEqual({ x: 6, y: 6 });
  });

  it("clampPoint e snapToGrid", () => {
    expect(clampPoint({ x: -50, y: 99999 })).toEqual({ x: 6, y: WORLD_H - CARD - LABEL_H - 6 });
    expect(clampPoint({ x: 99999, y: 0 })).toEqual({ x: WORLD_W - CARD - 6, y: 6 });
    expect(snapToGrid({ x: 40, y: 38 })).toEqual({ x: 2 * GRID, y: GRID });
  });

  it("resolveOverlaps separa due nodi sovrapposti; il nodo fisso non si muove", () => {
    const g = graphOf([card("A", "dataset", 300, 300), card("B", "dataset", 320, 310)]);
    const out = resolveOverlaps(g, { fixedId: "A" });
    expect(out.cards["A"]).toMatchObject({ x: 300, y: 300 });
    expect(anyOverlap(out)).toBe(false);
  });

  it("durante il trascinamento si scansano solo le coppie incompatibili", () => {
    const g = graphOf([
      card("D", "dataset", 300, 300),
      card("E", "dataset", 330, 300),
      card("F", "filter", 280, 310),
    ]);
    const out = separateWhileDragging(g, "D");
    expect(out.cards["D"]).toMatchObject({ x: 300, y: 300 });
    expect(out.cards["E"]?.x).not.toBe(330);
    expect(out.cards["F"]).toMatchObject({ x: 280, y: 310 });
  });

  it("dropFree riallinea alla griglia tutti tranne il nodo rilasciato", () => {
    const g = graphOf([card("A", "dataset", 301, 299), card("B", "dataset", 600, 601)]);
    const out = dropFree(g, "A");
    expect(out.cards["A"]).toMatchObject({ x: 301, y: 299 });
    expect(out.cards["B"]).toMatchObject({ x: 598, y: 598 });
  });

  it("freeSpot cerca a spirale un punto libero", () => {
    const g = graphOf([card("A", "dataset", 300, 300)]);
    expect(freeSpot(g, 800, 800)).toEqual({ x: 800, y: 800 });
    expect(freeSpot(g, 300, 300)).toEqual({ x: 300 + CARD + 26, y: 300 });
  });

  it("displaceTarget: centri coincidenti -> spinta verso il basso (correzione Fase 2.1)", () => {
    const g = graphOf([card("D", "dataset", 300, 300), card("E", "dataset", 300, 300)]);
    expect(displaceTarget(g, "D", "E")).toEqual({ x: 300, y: 300 + CARD + 36 });
  });

  it("displaceTarget: stesso limite inferiore di clampCard (correzione Fase 2.1)", () => {
    const g = graphOf([card("D", "dataset", 300, 1400), card("E", "dataset", 300, 1450)]);
    // 1600 - 88 - 22 - 6 = 1484 (nel prototipo 1600 - 88 - 26 = 1486)
    expect(displaceTarget(g, "D", "E")).toEqual({ x: 300, y: 1484 });
  });

  it("displace spinge via il nodo sotto quello trascinato", () => {
    const g = graphOf([card("D", "dataset", 300, 300), card("E", "dataset", 340, 300)]);
    const out = displace(g, "D", "E");
    expect(out.cards["D"]).toMatchObject({ x: 300, y: 300 });
    expect((out.cards["E"] as Card).x).toBeGreaterThanOrEqual(300 + CARD + 28);
  });
});

describe("modalità Organizzato", () => {
  it("assegnazione: dall'alto a sinistra, la postazione libera più vicina", () => {
    const g = graphOf([card("A", "dataset", 40, 30), card("B", "filter", 60, 40)]);
    const out = assignSlots(g);
    const slots = computeSlots();
    expect(out.cards["A"]?.slot).toBe(0);
    expect(out.cards["B"]?.slot).toBe(1);
    expect(out.cards["B"]).toMatchObject(slots[1] as object);
  });

  it("scambio di posto: chi occupa la postazione prende quella lasciata libera", () => {
    const g = assignSlots(graphOf([card("A", "dataset", 40, 30), card("B", "filter", 170, 40)]));
    const slots = computeSlots();
    const target = slots[1] as { x: number; y: number };
    const out = dropInSlot(g, "A", target.x + 5, target.y + 5);
    expect(out.cards["A"]?.slot).toBe(1);
    expect(out.cards["B"]?.slot).toBe(0);
    expect(out.cards["B"]).toMatchObject(slots[0] as object);
  });
});

describe("posizionamento dei nodi generati", () => {
  it("passaggio sganciato: al punto di rilascio, oppure vicino al box", () => {
    const g = graphOf([card("B", "filter", 300, 300)]);
    expect(detachPositionFn({ dropPoint: { x: 700, y: 500 } })(g, "B")).toEqual({
      x: 700 - CARD / 2,
      y: 500 - CARD / 2,
    });
    // correzione Fase 2.1: sotto il box, a CARD + LABEL_H + 18 = 128 px, senza toccarlo
    expect(detachPositionFn()(g, "B")).toEqual({ x: 300, y: 300 + 128 });
    // punto di rilascio in fondo al mondo: limite 1600 - 88 - 26 (riga 2176)
    expect(detachPositionFn({ dropPoint: { x: 700, y: 99999 } })(g, "B")).toEqual({
      x: 656,
      y: 1486,
    });
  });

  it("detachStep di etl-core accetta detachPositionFn", () => {
    const g = graphOf([
      {
        ...card("B", "filter", 300, 300),
        components: ["filter", "sort"],
        params: [defaultParams("filter"), defaultParams("sort")],
      },
    ]);
    const r = detachStep(g, "B", 1, () => "S", detachPositionFn());
    expect(r?.graph.cards["S"]).toMatchObject({ x: 300, y: 300 + 128 });
  });

  it("nodo inserito su un cavo: a metà strada, poi etl-core crea il suo output a destra", () => {
    let g = graphOf(
      [card("A", "dataset", 100, 300), card("B", "filter", 700, 300), card("X", "sort", 50, 800)],
      [{ from: "A", to: "B" }],
    );
    const link = g.links[0] as { from: string; to: string };
    const mid = insertPosition(g, link);
    expect(mid).toEqual({ x: 400, y: 300 });
    g = moveNode(g, "X", mid as { x: number; y: number });
    let n = 0;
    const ids = (): string => "O" + ++n;
    const next = insertOnLink(g, link, "X", ids, outputPositionFn());
    // l'output di X (O1): a destra di 200 px, 598 sulla griglia, cadrebbe su B: primo punto libero
    expect(next?.cards["O1"]).toMatchObject(freeSpot(g, Math.round(600 / GRID) * GRID, 300));
    expect(next?.links).toContainEqual({ from: "X", to: "O1" });
    expect(next?.links).toContainEqual({ from: "O1", to: "B" });
    const settled = settleNewNode(next as Graph, "X");
    expect(anyOverlap(settled)).toBe(false);
  });

  it("dopo una fusione il box va a valle del suo primo ingresso", () => {
    const g = graphOf(
      [card("A", "dataset", 500, 300), card("B", "filter", 100, 300)],
      [{ from: "A", to: "B" }],
    );
    const out = relocateAfterMerge(g, "B");
    expect(out.cards["B"]).toMatchObject({ x: 500 + CARD + 78, y: 300 });
  });
});

describe("purezza", () => {
  it("nessuna funzione modifica il grafo che riceve", () => {
    const g = graphOf(
      [
        card("A", "dataset", 300, 300),
        card("B", "filter", 320, 310),
        card("C", "sort", 700, 300, { slot: 3 }),
      ],
      [{ from: "A", to: "B" }],
    );
    const snapshot = JSON.stringify(g);
    settleLinks(g);
    resolveOverlaps(g, { snap: true });
    separateWhileDragging(g, "A");
    displace(g, "A", "B");
    assignSlots(g);
    dropInSlot(g, "A", 0, 0);
    settleNewNode(g, "B", { mode: "grid" });
    relocateAfterMerge(g, "B");
    outputPositionFn()(g, "B");
    expect(JSON.stringify(g)).toBe(snapshot);
  });
});
```

### `src/etl-layout/autoLayout.ts`

171 righe

```ts
/**
 * Riordino automatico. Prototipo, righe 4220-4327 (`autoLayout`):
 * colonne per profondità del flusso, ordine per baricentro dei vicini,
 * simmetria verticale, colonna di parcheggio per i nodi isolati.
 */
import type { Card, Graph } from "../etl-core";
import {
  AUTO_AVAIL_MARGIN,
  AUTO_BARY_PASSES,
  AUTO_COL,
  AUTO_ROW_MAX,
  AUTO_ROW_MIN,
  AUTO_SOURCE_PASSES,
  AUTO_SYMMETRY_PASSES,
  AUTO_X0_MIN,
  AUTO_Y0_MIN,
  CARD,
  LABEL_H,
} from "./constants";
import { DEFAULT_WORLD, anyOverlap, clampPoint, resolveOverlaps, withPositions } from "./free";
import { assignSlots } from "./slots";
import type { LayoutMode, Size } from "./types";

export interface AutoLayoutOptions {
  /**
   * Area visibile del canvas (`stage.clientWidth/clientHeight` nel
   * prototipo): colonne e righe vengono centrate lì.
   */
  readonly viewport: Size;
  readonly world?: Size;
  readonly mode?: LayoutMode;
}

export function autoLayout(graph: Graph, opts: AutoLayoutOptions): Graph {
  const world = opts.world ?? DEFAULT_WORLD;
  const stageW = opts.viewport.w;
  const stageH = opts.viewport.h;
  const links = graph.links;
  const all = Object.keys(graph.cards);
  if (!all.length) return graph;
  const card = (id: string): Card => graph.cards[id] as Card;

  // un nodo senza collegamenti non appartiene al flusso
  const isolated = all.filter((id) => !links.some((l) => l.from === id || l.to === id));
  const ids = all.filter((id) => !isolated.includes(id));

  // 1. profondità = percorso più lungo dalle sorgenti (righe 4228-4239)
  const depth: Record<string, number> = {};
  for (const id of ids) depth[id] = 0;
  let changed = true;
  let guard = 0;
  while (changed && guard++ < ids.length + 6) {
    changed = false;
    for (const l of links) {
      if (
        graph.cards[l.from] &&
        graph.cards[l.to] &&
        (depth[l.to] as number) < (depth[l.from] as number) + 1
      ) {
        depth[l.to] = (depth[l.from] as number) + 1;
        changed = true;
      }
    }
  }
  // una sorgente si avvicina al suo consumatore più vicino (righe 4240-4250)
  for (let pass = 0; pass < AUTO_SOURCE_PASSES; pass++) {
    for (const id of ids) {
      const hasInput = links.some((l) => l.to === id);
      if (hasInput) continue;
      const succ = links.filter((l) => l.from === id).map((l) => depth[l.to] as number);
      if (!succ.length) continue;
      depth[id] = Math.max(0, Math.min(...succ) - 1);
    }
  }
  const minD = Math.min(...ids.map((id) => depth[id] as number));
  for (const id of ids) depth[id] = (depth[id] as number) - minD;

  const layers: (string[] | undefined)[] = [];
  for (const id of ids) {
    const d = depth[id] as number;
    (layers[d] = layers[d] ?? []).push(id);
  }
  const cols: string[][] = layers.filter((l): l is string[] => !!l);
  if (isolated.length) cols.push(isolated);
  if (!cols.length) return graph;

  // posizioni di lavoro, modificate sul posto come nel prototipo
  const pos = new Map<string, { x: number; y: number }>();
  for (const id of all) pos.set(id, { x: card(id).x, y: card(id).y });
  const P = (id: string): { x: number; y: number } => pos.get(id) as { x: number; y: number };

  // 2. ordine dentro la colonna: baricentro dei vicini (righe 4256-4278)
  const posIn: Record<string, number> = {};
  const reindex = (): void => {
    for (const layer of cols) layer.forEach((id, i) => (posIn[id] = i));
  };
  for (const layer of cols) layer.sort((a, b) => P(a).y - P(b).y);
  reindex();
  for (let it = 0; it < AUTO_BARY_PASSES; it++) {
    for (const layer of cols) {
      const bary: Record<string, number> = {};
      for (const id of layer) {
        const nb: number[] = [];
        for (const l of links) {
          if (l.to === id && posIn[l.from] !== undefined) nb.push(posIn[l.from] as number);
          if (l.from === id && posIn[l.to] !== undefined) nb.push(posIn[l.to] as number);
        }
        bary[id] = nb.length ? nb.reduce((a, b) => a + b, 0) / nb.length : (posIn[id] as number);
      }
      layer.sort((a, b) => (bary[a] as number) - (bary[b] as number));
      reindex();
    }
  }

  // 3. griglia di partenza: colonne equidistanti, ogni colonna centrata (righe 4280-4295)
  const maxCount = Math.max(...cols.map((c) => c.length));
  const avail = stageH - AUTO_AVAIL_MARGIN - (CARD + LABEL_H);
  const ROW =
    maxCount > 1
      ? Math.max(AUTO_ROW_MIN, Math.min(AUTO_ROW_MAX, avail / (maxCount - 1)))
      : AUTO_ROW_MAX;
  const totalW = (cols.length - 1) * AUTO_COL + CARD;
  const x0 = Math.max(AUTO_X0_MIN, (stageW - totalW) / 2);
  cols.forEach((layer, li) => {
    const h = (layer.length - 1) * ROW + CARD + LABEL_H;
    const y0 = Math.max(AUTO_Y0_MIN, (stageH - h) / 2);
    layer.forEach((id, i) => {
      P(id).x = x0 + li * AUTO_COL;
      P(id).y = y0 + i * ROW;
    });
  });

  // 4. simmetria: ogni nodo tende al baricentro dei vicini (righe 4297-4316)
  for (let pass = 0; pass < AUTO_SYMMETRY_PASSES; pass++) {
    for (const layer of cols) {
      for (const id of layer) {
        const nb: number[] = [];
        for (const l of links) {
          if (l.to === id && graph.cards[l.from]) nb.push(P(l.from).y);
          if (l.from === id && graph.cards[l.to]) nb.push(P(l.to).y);
        }
        if (nb.length) P(id).y = (P(id).y + nb.reduce((a, b) => a + b, 0) / nb.length) / 2;
      }
      layer.sort((a, b) => P(a).y - P(b).y);
      for (let i = 1; i < layer.length; i++) {
        const prev = P(layer[i - 1] as string);
        const cur = P(layer[i] as string);
        if (cur.y - prev.y < ROW) cur.y = prev.y + ROW;
      }
      const first = P(layer[0] as string).y;
      const last = P(layer[layer.length - 1] as string).y;
      const off = (stageH - (last - first + CARD + LABEL_H)) / 2 - first;
      for (const id of layer) P(id).y += off;
    }
  }

  // arrotondamento e limiti del mondo (riga 4319)
  for (const id of all) {
    const p = P(id);
    const q = clampPoint({ x: Math.round(p.x), y: Math.round(p.y) }, world);
    p.x = q.x;
    p.y = q.y;
  }
  const placed = withPositions(graph, pos);

  // righe 4321-4323
  if (opts.mode === "grid") return assignSlots(placed, world);
  if (anyOverlap(placed)) return resolveOverlaps(placed, { snap: false, world });
  return placed;
}
```

### `src/etl-layout/constants.ts`

153 righe

```ts
/**
 * Costanti numeriche della geometria, IDENTICHE al prototipo
 * (docs/prototype/isa-fusion-prototype.html). Ogni costante riporta la
 * riga da cui proviene.
 */

// --- Nodi --------------------------------------------------------------------

/** Lato del quadrato di un nodo (riga 909; CSS `.icon-wrap`, riga 630). */
export const CARD = 88;
/** Spazio occupato dall'etichetta sotto il nodo (riga 1524). */
export const LABEL_H = 22;
/** Distanza tra il quadrato e l'etichetta (CSS `.card { gap:8px }`, riga 626). */
export const LABEL_GAP = 8;
/** Larghezza massima dell'etichetta (CSS `.label { max-width:96px }`, riga 682). */
export const LABEL_MAX_W = 96;

// --- Mondo -------------------------------------------------------------------

/** Dimensioni del mondo con pan/zoom attivo (riga 936; `FEATURES.panZoom` è `true`, riga 919). */
export const WORLD_W = 2600;
export const WORLD_H = 1600;
/** Margine dal bordo del mondo per `clampCard` (righe 1538-1539). */
export const WORLD_MARGIN = 6;

// --- Porte e cavi ------------------------------------------------------------

/** Angoli delle quattro porte: destra, sotto, sinistra, sopra (riga 1018). */
export const PORTS: readonly number[] = [0, Math.PI / 2, Math.PI, -Math.PI / 2];
/** Tratto rettilineo in uscita da una porta prima dello snodo (riga 1020). */
export const STUB = 32;
/** Raggio dei raccordi arrotondati (riga 1020). */
export const ELBOW_R = 11;
/** Snodi ammessi per collegamento (riga 1021). */
export const MAX_BENDS = 1;
/** Margine attorno a ogni nodo trattato come ostacolo (riga 1055). */
export const OBST_PAD = 9;
/** Rientro del rettangolo dei due nodi collegati usati come ostacolo (riga 1245). */
export const SELF_OBSTACLE_INSET = 2;
/** Scorrimento massimo del punto di aggancio: `half - 14` (riga 1257). */
export const PORT_SLACK_INSET = 14;
/** Sotto questa differenza i due agganci sono già allineati (righe 1218, 1223). */
export const STRAIGHT_EPS = 1.5;
/** Distanza degli snodi candidati dai bordi di un ostacolo (righe 1234-1235). */
export const KNOB_OBSTACLE_MARGIN = 14;
/** Passo e numero degli snodi candidati attorno a quello centrale (riga 1237). */
export const KNOB_STEP = 24;
export const KNOB_STEPS = 8;
/** Tolleranza sui confronti tra coordinate (righe 1057, 1090, 1102, 1180). */
export const EPS = 0.5;
/** `segCross`: margine sulle estremità (riga 1101). */
export const CROSS_MARGIN = 3;
/** `segCross`: due tratti paralleli sono sullo stesso binario sotto questa distanza (righe 1109, 1114). */
export const CROSS_SAME_TRACK = 4;
/** `segCross`: sovrapposizione minima per contare due tratti paralleli (righe 1112, 1117). */
export const CROSS_MIN_OVERLAP = 8;

/** Pesi della funzione di costo di `chooseRoute` (righe 1267-1268). */
export const SCORE = {
  /** Ogni attraversamento di un nodo. */
  obstacle: 1e6,
  /** Ogni snodo oltre il limite. */
  overBends: 6000,
  /** Ogni incrocio con un altro cavo. */
  crossing: 3500,
  /** Ogni ripiegamento rispetto alla porta. */
  backtrack: 3000,
  /** Uscita e rientro dallo stesso lato (forma a U). */
  sameSide: 1800,
  /** Ogni pixel fuori dal rettangolo che unisce gli agganci. */
  overshoot: 14,
  /** Ogni pixel di lunghezza. */
  length: 1,
  /** Ogni porta diversa da quella precedente. */
  portChange: 0.8,
} as const;

/** Distanza tra cavi che condividono la stessa porta (riga 1355). */
export const PORT_SPREAD = 15;
/** Distanza tra corsie parallele (riga 1380). */
export const LANE_GAP = 14;
/** Due snodi più vicini di così condividono il corridoio (riga 1378). */
export const LANE_NEAR = 12;
/** Spessore dell'area sensibile di un cavo: `stroke-width="16"` (riga 1402). Tolleranza = metà. */
export const LINK_HIT_WIDTH = 16;

// --- Modalità Libero ----------------------------------------------------------

/** Passo della griglia di aggancio (riga 1541). */
export const GRID = 26;
/** Distanza minima tra due nodi in `resolveOverlaps` (riga 1553). */
export const OVERLAP_PAD = 28;
/** Iterazioni massime di `resolveOverlaps` (riga 1556). */
export const OVERLAP_ITERATIONS = 80;
/** Margine di `overlapsAny` e `anyOverlap` (righe 1593, 2635). */
export const OVERLAP_MARGIN = 18;
/** Passi di `freeSpot` (righe 1600-1603). */
export const FREE_SPOT_RADIUS = 8;
export const FREE_SPOT_STEP_X = CARD + 26;
export const FREE_SPOT_STEP_Y = CARD + LABEL_H + 22;
/** Spinta di `displace` (riga 1946). */
export const DISPLACE_PUSH = CARD + 36;
/**
 * Limite inferiore del punto di rilascio di un passaggio sganciato:
 * `worldH() - CARD - 26` (riga 2176). Il prototipo usava lo stesso valore
 * anche in `displace` (riga 1950); dalla Fase 2.1 `displace` usa il limite
 * di `clampCard` (NOTE_DIVERGENZE.md).
 */
export const DROP_BOTTOM = CARD + 26;

// --- Modalità Organizzato -----------------------------------------------------

/** Postazioni: larghezza, altezza, margine (riga 3908). */
export const SLOT_W = 128;
export const SLOT_H = 142;
export const SLOT_M = 14;

// --- Riordino automatico ------------------------------------------------------

/** Distanza tra le colonne (riga 4282). */
export const AUTO_COL = CARD + 78;
/** Margine verticale sottratto all'altezza disponibile (riga 4284). */
export const AUTO_AVAIL_MARGIN = 16;
/** Distanza tra le righe: minima e massima (righe 4285-4287). */
export const AUTO_ROW_MIN = CARD + LABEL_H + 18;
export const AUTO_ROW_MAX = CARD + LABEL_H + 28;
/** Margini minimi a sinistra e in alto (righe 4289, 4292). */
export const AUTO_X0_MIN = 12;
export const AUTO_Y0_MIN = 8;
/** Passate: sorgenti avvicinate, baricentro, simmetria (righe 4242, 4265, 4297). */
export const AUTO_SOURCE_PASSES = 3;
export const AUTO_BARY_PASSES = 6;
export const AUTO_SYMMETRY_PASSES = 5;

// --- Nodi generati ------------------------------------------------------------

/** Output: a destra del box di 200, entro il mondo meno 12 (righe 1743, 1745). */
export const OUTPUT_OFFSET_X = 200;
export const OUTPUT_EDGE_MARGIN = 12;
/**
 * Passaggio sganciato senza punto di rilascio: sotto il box, alla distanza
 * minima che evita la sovrapposizione (`CARD + LABEL_H + 18`, la soglia di
 * `overlapsAny`). Correzione intenzionale (Fase 2.1): il prototipo usava
 * `CARD + 34` (riga 2178), che toccava sempre il box.
 */
export const DETACH_OFFSET_Y = CARD + LABEL_H + 18;
/** `relocateAfter`: passi orizzontali/verticali e tentativi (righe 1678-1681). */
export const RELOCATE_STEP_X = CARD + 78;
export const RELOCATE_STEP_Y = CARD + LABEL_H + 30;
export const RELOCATE_TRIES = 5;
/** `outputSlotFor`: peso dello scarto verticale (riga 1715). */
export const OUTPUT_SLOT_DY_WEIGHT = 3;
```

### `src/etl-layout/free.ts`

265 righe

```ts
/**
 * Modalità Libero: limiti del mondo, aggancio alla griglia, separazione
 * dei nodi sovrapposti. Prototipo, righe 1537-1609, 1939-1956, 2630-2638.
 */
import { compatiblePair } from "../etl-core";
import type { Card, Graph } from "../etl-core";
import {
  CARD,
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
 * Riga 1939 (con le correzioni della Fase 2.1, NOTE_DIVERGENZE.md): dove
 * viene spinto il nodo `targetId`, incompatibile, finito sotto quello
 * trascinato: lungo la congiungente dei centri, di `DISPLACE_PUSH`, entro
 * i limiti di `clampCard`. Con centri coincidenti (o quasi) la spinta va
 * verso il basso.
 */
export function displaceTarget(
  graph: Graph,
  draggedId: string,
  targetId: string,
  world: Size = DEFAULT_WORLD,
): Point | null {
  const d = graph.cards[draggedId];
  const t = graph.cards[targetId];
  if (!d || !t) return null;
  let vx = t.x + CARD / 2 - (d.x + CARD / 2);
  let vy = t.y + CARD / 2 - (d.y + CARD / 2);
  // correzione: nel prototipo `hypot || 1` rendeva irraggiungibile questo ramo
  let len = Math.hypot(vx, vy);
  if (len < 1) {
    vx = 0;
    vy = 1;
    len = 1;
  }
  // correzione: stesso limite di `clampCard` (nel prototipo `worldH() - CARD - 26`, riga 1950)
  return clampPoint(
    { x: t.x + (vx / len) * DISPLACE_PUSH, y: t.y + (vy / len) * DISPLACE_PUSH },
    world,
  );
}

/**
 * Riga 1939: spinge via il nodo `targetId` (`displaceTarget`); il
 * prototipo, a fine animazione (riga 1955), separa poi con
 * `resolveOverlaps(dragged, null, true)`: qui è incluso, restituendo lo
 * stato finale.
 */
export function displace(
  graph: Graph,
  draggedId: string,
  targetId: string,
  world: Size = DEFAULT_WORLD,
): Graph {
  const p = displaceTarget(graph, draggedId, targetId, world);
  if (!p) return graph;
  const pushed = withPositions(graph, new Map([[targetId, p]]));
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

80 righe

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
  displaceTarget,
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

261 righe

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
import { buildFromShape, chooseRoute, evaluateRoute, slide } from "./routing";
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
  readonly basePts: readonly Point[];
  readonly a: Point;
  readonly b: Point;
}

function shift(pt: Point, axis: Axis, off: number): Point {
  if (!off) return pt;
  return axis.x !== 0 ? { x: pt.x, y: pt.y + off } : { x: pt.x + off, y: pt.y };
}

function sameShape(x: Shape, y: Shape): boolean {
  if (x.kind !== y.kind) return false;
  if (x.kind === "Z" && y.kind === "Z") return x.knob === y.knob;
  if (x.kind === "L" && y.kind === "L") return x.first === y.first;
  if (x.kind === "straight" && y.kind === "straight") return x.oa === y.oa && x.ob === y.ob;
  return false;
}

/**
 * Una valutazione completa di tutti i cavi. `prev` è il risultato
 * precedente: da lì vengono il percorso attuale di ogni cavo (stabilità)
 * e i percorsi degli altri per contare gli incroci.
 *
 * Correzione intenzionale (Fase 2.1, NOTE_DIVERGENZE.md): aggiornamento
 * SEQUENZIALE. I cavi si valutano nell'ordine dei collegamenti; ciascuno
 * conta gli incroci con i percorsi già aggiornati in questa passata per i
 * cavi che lo precedono e con quelli della passata precedente per i
 * successivi (nel prototipo tutti vedevano la passata precedente, e due
 * cavi potevano inseguirsi senza fermarsi). Gli incroci si contano sui
 * percorsi di base (`basePts`, prima di scostamenti e corsie), che
 * dipendono solo dalla scelta del singolo cavo. Un cavo cambia percorso
 * solo se l'alternativa scelta con le regole del prototipo costa
 * strettamente meno del percorso attuale. Poiché un incrocio costa uguale
 * ai due cavi coinvolti, ogni cambio fa scendere il costo totale: le
 * passate terminano sempre.
 */
export function layoutLinks(
  graph: Graph,
  prev: LinkRoutes = {},
  opts: LayoutLinksOptions = {},
): Record<string, LinkRoute> {
  const half = CARD / 2;
  const maxBends = opts.maxBends ?? MAX_BENDS;
  const ids = Object.keys(graph.cards);

  // percorsi di base correnti: all'inizio quelli della passata precedente
  const base = new Map<string, readonly Point[]>();
  for (const l of graph.links) {
    const r = prev[linkKey(l)];
    if (r) base.set(linkKey(l), r.basePts ?? r.pts);
  }

  // primo passaggio: porte e forme, in sequenza (righe 1306-1344)
  const items: Chosen[] = [];
  graph.links.forEach((l, index) => {
    const ca = graph.cards[l.from];
    const cb = graph.cards[l.to];
    if (!ca || !cb) return;
    const key = linkKey(l);
    const a = nodeCenter(ca);
    const b = nodeCenter(cb);
    const st = prev[key];
    const obstacles: Rect[] = ids
      .filter((id) => id !== l.from && id !== l.to && id !== opts.draggingId)
      .map((id) => obstacleRect(graph.cards[id] as { x: number; y: number }));
    const others = graph.links
      .filter((o) => o !== l)
      .map((o) => base.get(linkKey(o)))
      .filter((pts): pts is readonly Point[] => !!pts);
    const routeOpts = { prevPortA: st?.portA, prevPortB: st?.portB, others, maxBends };
    let pick = chooseRoute(a, b, half, obstacles, routeOpts);
    if (!pick) return;
    if (
      st &&
      (pick.portA !== st.portA || pick.portB !== st.portB || !sameShape(pick.shape, st.shape))
    ) {
      const current = evaluateRoute(a, b, half, obstacles, st.portA, st.portB, st.shape, routeOpts);
      if (!(pick.score < current.score)) pick = current;
    }
    const chosenBase = evaluateRoute(a, b, half, obstacles, pick.portA, pick.portB, pick.shape, {
      maxBends,
    }).pts;
    base.set(key, chosenBase);
    items.push({
      link: l,
      index,
      portA: pick.portA,
      portB: pick.portB,
      horiz: pick.horiz,
      shape: pick.shape,
      basePts: chosenBase,
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
      basePts: it.basePts,
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

