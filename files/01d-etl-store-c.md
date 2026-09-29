# 01d-etl-store-c.md

File in questo blocco:

- `src/etl-store/reduce.ts`
- `src/etl-store/serialize.ts`
- `src/etl-store/state.ts`
- `src/etl-store/store.ts`
- `src/etl-store/types.ts`

---

### `src/etl-store/reduce.ts`

826 righe

```ts
/**
 * Comandi: `reduce(state, command) → { state, result }`, funzione pura.
 * Un comando rifiutato restituisce lo stato invariato e il motivo. Ogni
 * ramo riporta il gestore del prototipo corrispondente
 * (docs/prototype/isa-fusion-prototype.html); il dettaglio è nel README.
 */
import {
  META,
  boxCapacity,
  cardById,
  connect as coreConnect,
  defaultParams,
  deleteLink as coreDeleteLink,
  deleteNodes as coreDeleteNodes,
  deleteStep as coreDeleteStep,
  detachStep as coreDetachStep,
  duplicateNodes,
  inputsOf,
  insertOnLink as coreInsertOnLink,
  insertable,
  mergeBoxes,
  relation,
  reorderSteps as coreReorderSteps,
  withParamAt,
} from "../etl-core";
import type {
  Card,
  ComponentId,
  DatasetParams,
  Graph,
  IdGenerator,
  Link,
  Params,
} from "../etl-core";
import {
  CARD,
  GRID,
  anyOverlap,
  assignSlots,
  autoLayout as layoutAuto,
  clampPoint,
  clearSlots,
  computeSlots,
  detachPositionFn,
  dropFree,
  dropInSlot,
  firstFreeSlot,
  freeSpot,
  insertPosition,
  moveNode,
  outputPositionFn,
  placeInSlots,
  relocateAfterMerge,
  resolveOverlaps,
  setSlot,
  settleNewNode,
  withPositions,
} from "../etl-layout";
import type { LayoutMode, Point } from "../etl-layout";
import { ZOOM_MAX, ZOOM_MIN } from "./state";
import type {
  Command,
  CommandResult,
  Counters,
  DropTarget,
  EtlState,
  Inspector,
  LibraryItem,
  PanelKey,
  Panels,
  ReduceOutcome,
} from "./types";

// --- Supporto -----------------------------------------------------------------

const OK: CommandResult = { ok: true };

function refuse(state: EtlState, reason: string): ReduceOutcome {
  return { state, result: { ok: false, reason } };
}

function done(state: EtlState): ReduceOutcome {
  return { state, result: OK };
}

/** Generatore di id a partire dai contatori dello stato (prototipo: `uidCounter++`). */
class Ids {
  uid: number;
  constructor(counters: Counters) {
    this.uid = counters.uid;
  }
  next(prefix: "ds" | "op" | "out"): string {
    const id = prefix + "-" + this.uid;
    this.uid += 1;
    return id;
  }
  gen(prefix: "ds" | "op" | "out"): IdGenerator {
    return () => this.next(prefix);
  }
}

function isComponent(type: string): type is ComponentId {
  return Object.prototype.hasOwnProperty.call(META, type);
}

function nodeExists(graph: Graph, id: string): boolean {
  return !!graph.cards[id];
}

function sameLink(a: Link, b: Link): boolean {
  return a.from === b.from && a.to === b.to;
}

/**
 * I nodi creati da un'operazione (presenti in `after` e non in `before`),
 * nell'ordine di creazione, prendono posto come nel prototipo: in
 * Organizzato una postazione, in Libero gli altri si scansano (etl-layout,
 * `settleNewNode`; prototipo `spawnOutput`, riga 1766, e simili).
 */
function settleCreated(before: Graph, after: Graph, mode: LayoutMode): Graph {
  let next = after;
  for (const id of Object.keys(after.cards)) {
    if (!before.cards[id]) next = settleNewNode(next, id, { mode });
  }
  return next;
}

/** Stato con un nuovo grafo e contatori; selezione e inspector ripuliti dai nodi spariti. */
function withGraph(
  state: EtlState,
  graph: Graph,
  ids: Ids,
  extra: Partial<EtlState> = {},
): EtlState {
  const selection = (extra.selection ?? state.selection).filter((id) => !!graph.cards[id]);
  const insp = extra.inspector ?? state.inspector;
  const inspector: Inspector =
    insp.nodeId && !graph.cards[insp.nodeId] ? { nodeId: null, step: 0 } : insp;
  return {
    ...state,
    ...extra,
    graph,
    selection,
    inspector,
    counters: { ...state.counters, ...(extra.counters ?? {}), uid: ids.uid },
  };
}

/** Collega e sistema l'eventuale output appena generato. */
function connectSettled(
  graph: Graph,
  from: string,
  to: string,
  ids: Ids,
  mode: LayoutMode,
): { ok: true; graph: Graph } | { ok: false; reason: string } {
  const r = coreConnect(graph, from, to, ids.gen("out"), outputPositionFn({ mode }));
  if (!r.ok) return r;
  return { ok: true, graph: settleCreated(graph, r.graph, mode) };
}

/** Prototipo, righe 1774-1840 (`performMerge`) e 1826-1834 (riposizionamento). */
function mergeSettled(
  graph: Graph,
  dragged: string,
  target: string,
  ids: Ids,
  mode: LayoutMode,
): { ok: true; graph: Graph } | { ok: false; reason: string } {
  const r = mergeBoxes(graph, dragged, target, ids.gen("out"), outputPositionFn({ mode }));
  if (!r.ok) return r;
  const relocated = relocateAfterMerge(r.graph, target, { mode });
  return { ok: true, graph: settleCreated(graph, relocated, mode) };
}

/**
 * Prototipo, righe 1853-1873 (`insertOnLink`): la lavorazione va a metà
 * del cavo, poi il flusso si riallaccia attraverso di lei.
 */
function insertSettled(
  graph: Graph,
  node: string,
  link: Link,
  ids: Ids,
  mode: LayoutMode,
): { ok: true; graph: Graph } | { ok: false; reason: string } {
  if (!graph.links.some((l) => sameLink(l, link)))
    return { ok: false, reason: "Il collegamento non esiste" };
  if (!insertable(graph, link, node))
    return {
      ok: false,
      reason:
        "Si può inserire solo una lavorazione ancora slegata, in un collegamento da un dataset a una lavorazione",
    };
  const mid = insertPosition(graph, link) as Point;
  let next = moveNode(graph, node, mid);
  if (mode === "grid") {
    const slots = computeSlots();
    next = setSlot(next, node, firstFreeSlot(next, slots, mid.x, mid.y, node));
  }
  const inserted = coreInsertOnLink(next, link, node, ids.gen("out"), outputPositionFn({ mode }));
  if (!inserted) return { ok: false, reason: "Inserimento non riuscito" };
  let settled = settleCreated(next, inserted, mode);
  settled = mode === "grid" ? placeInSlots(settled, computeSlots()) : dropFree(settled, node);
  return { ok: true, graph: settled };
}

/** Prototipo, righe 4710-4723 (`paletteRelation`): cosa succede rilasciando un nuovo elemento su un nodo. */
function paletteRelation(
  graph: Graph,
  kind: Card["kind"],
  targetId: string,
): "merge" | "link" | "link-reverse" | null {
  const t = graph.cards[targetId];
  if (!t) return null;
  if (kind === "op" && t.kind === "op") return "merge";
  if (kind === "dataset" && t.kind === "op") {
    if (inputsOf(graph, targetId).length >= boxCapacity(t)) return null;
    return "link";
  }
  if (kind === "op" && t.kind === "dataset") {
    if ((t.capacity ?? 0) > 1 && (t.filled ?? 0) < (t.capacity ?? 0)) return null;
    return "link-reverse";
  }
  return null;
}

function snap(v: number): number {
  return Math.round(v / GRID) * GRID;
}

// --- Comandi --------------------------------------------------------------------

/** Prototipo, righe 4939-5069 (rilascio dalla cassetta degli strumenti o dalla libreria). */
function addNode(
  state: EtlState,
  p: Extract<Command, { type: "addNode" }>["payload"],
): ReduceOutcome {
  if (!isComponent(p.component)) return refuse(state, `Tipo di nodo sconosciuto: ${p.component}`);
  const kind: Card["kind"] = p.component === "dataset" ? "dataset" : "op";
  let lib: LibraryItem | undefined;
  if (p.libraryId !== undefined) {
    lib = state.library.find((x) => x.id === p.libraryId);
    if (!lib) return refuse(state, "Il dataset non è nella libreria");
    if (kind !== "dataset") return refuse(state, "Dalla libreria si trascinano solo dataset");
  }
  const ids = new Ids(state.counters);
  const id = ids.next(kind === "dataset" ? "ds" : "op");
  let ds = state.counters.ds;
  let name = META[p.component].label;
  if (p.component === "dataset") {
    if (lib) name = lib.name;
    else {
      ds += 1;
      name = "Dataset " + ds;
    }
  }
  let params: Params = defaultParams(p.component);
  if (lib) {
    const dp: DatasetParams = {
      ...(params as DatasetParams),
      path: lib.path,
      columns: [...lib.columns],
    };
    params = dp as Params;
  }
  const make = (x: number, y: number): Card => ({
    id,
    kind,
    components: [p.component],
    params: [params],
    name,
    x,
    y,
  });
  const graph = state.graph;
  const mode = state.mode;
  const counters = { ds };

  // rilasciata su un cavo: nasce già inserita nel flusso (righe 5000-5016)
  if (p.target && "link" in p.target) {
    if (kind !== "op")
      return refuse(state, "Solo una lavorazione si può inserire in un collegamento");
    const created = {
      ...graph,
      cards: { ...graph.cards, [id]: make(p.point.x - CARD / 2, p.point.y - CARD / 2) },
    };
    const r = insertSettled(created, id, p.target.link, ids, mode);
    if (!r.ok) return refuse(state, r.reason);
    return done(withGraph(state, r.graph, ids, { counters: { ...state.counters, ...counters } }));
  }

  // rilasciata su un nodo: fusione o collegamento (righe 5019-5046)
  if (p.target && "node" in p.target) {
    const tId = p.target.node;
    const t = graph.cards[tId];
    if (!t) return refuse(state, "Il nodo di destinazione non esiste");
    const rel = paletteRelation(graph, kind, tId);
    if (rel) {
      const start =
        rel === "merge"
          ? { x: t.x, y: t.y }
          : freeSpot(graph, snap(t.x + (rel === "link" ? -150 : 150)), t.y);
      let next: Graph = { ...graph, cards: { ...graph.cards, [id]: make(start.x, start.y) } };
      if (rel === "merge") {
        const r = mergeSettled(next, id, tId, ids, mode);
        if (!r.ok) return refuse(state, r.reason);
        return done(
          withGraph(state, r.graph, ids, { counters: { ...state.counters, ...counters } }),
        );
      }
      next = settleNewNode(next, id, { mode });
      const r =
        rel === "link"
          ? connectSettled(next, id, tId, ids, mode)
          : connectSettled(next, tId, id, ids, mode);
      if (!r.ok) return refuse(state, r.reason);
      return done(withGraph(state, r.graph, ids, { counters: { ...state.counters, ...counters } }));
    }
    // nessuna relazione possibile: il prototipo lo rilascia come nel vuoto
  }

  // rilasciata nel vuoto (righe 5048-5065)
  const raw = { x: p.point.x - CARD / 2, y: p.point.y - CARD / 2 };
  const pos = freeSpot(graph, snap(raw.x), snap(raw.y));
  const next = settleNewNode(
    { ...graph, cards: { ...graph.cards, [id]: make(pos.x, pos.y) } },
    id,
    { mode },
  );
  return done(withGraph(state, next, ids, { counters: { ...state.counters, ...counters } }));
}

/** Prototipo, righe 4619-4627 (`nudgeSelection`). */
function moveNodes(
  state: EtlState,
  p: Extract<Command, { type: "moveNodes" }>["payload"],
): ReduceOutcome {
  if (state.mode === "grid")
    return refuse(state, "In modalità Organizzato le postazioni sono fisse");
  const ids = p.ids.filter((id) => nodeExists(state.graph, id));
  if (!ids.length) return refuse(state, "Nessun nodo da spostare");
  const pos = new Map<string, Point>();
  for (const id of ids) {
    const c = state.graph.cards[id] as Card;
    pos.set(id, clampPoint({ x: c.x + p.dx, y: c.y + p.dy }));
  }
  return done({ ...state, graph: withPositions(state.graph, pos) });
}

/**
 * Rilascio dopo un trascinamento (prototipo, righe 2058-2105, `onUp` del
 * trascinamento di un nodo), applicato allo stato in cui i nodi sono già
 * nella posizione di rilascio. `origin` sono le posizioni di partenza (per
 * la fusione e il collegamento il nodo torna al suo posto, righe 2087-2088).
 */
export function dropAt(
  state: EtlState,
  ids: readonly string[],
  origin: ReadonlyMap<string, Point>,
  target: DropTarget | undefined,
): ReduceOutcome {
  const mode = state.mode;
  const idsObj = new Ids(state.counters);
  // gruppo: si separa o si riassegna, nessun bersaglio (righe 2076-2081)
  if (ids.length > 1) {
    const graph =
      mode === "grid"
        ? assignSlots(state.graph)
        : anyOverlap(state.graph)
          ? resolveOverlaps(state.graph, { snap: true })
          : state.graph;
    return done(withGraph(state, graph, idsObj));
  }
  const id = ids[0] as string;
  const c = state.graph.cards[id] as Card;

  if (target && "node" in target && target.node !== id) {
    const rel = relation(state.graph, id, target.node).relation;
    if (rel === "merge" || rel === "link" || rel === "link-reverse") {
      const o = origin.get(id) ?? { x: c.x, y: c.y };
      const back = moveNode(state.graph, id, o);
      const r =
        rel === "merge"
          ? mergeSettled(back, id, target.node, idsObj, mode)
          : rel === "link"
            ? connectSettled(back, id, target.node, idsObj, mode)
            : connectSettled(back, target.node, id, idsObj, mode);
      if (!r.ok) return refuse(state, r.reason);
      const inspector: Inspector | undefined =
        rel === "merge" && state.inspector.nodeId === id
          ? { nodeId: target.node, step: 0 }
          : undefined;
      return done(withGraph(state, r.graph, idsObj, inspector ? { inspector } : {}));
    }
  }
  if (target && "link" in target) {
    const canInsert =
      c.kind === "op" && !state.graph.links.some((l) => l.from === id || l.to === id);
    if (canInsert && insertable(state.graph, target.link, id)) {
      const r = insertSettled(state.graph, id, target.link, idsObj, mode);
      if (!r.ok) return refuse(state, r.reason);
      return done(withGraph(state, r.graph, idsObj));
    }
  }
  // nel vuoto (o su un bersaglio non valido): Organizzato = postazione, Libero = separa e riallinea
  const graph = mode === "grid" ? dropInSlot(state.graph, id, c.x, c.y) : dropFree(state.graph, id);
  return done(withGraph(state, graph, idsObj));
}

function dropNodes(
  state: EtlState,
  p: Extract<Command, { type: "dropNodes" }>["payload"],
): ReduceOutcome {
  const ids = p.ids.filter((id) => nodeExists(state.graph, id));
  if (!ids.length) return refuse(state, "Nessun nodo da spostare");
  const origin = new Map<string, Point>();
  const pos = new Map<string, Point>();
  for (const id of ids) {
    const c = state.graph.cards[id] as Card;
    origin.set(id, { x: c.x, y: c.y });
    pos.set(id, { x: c.x + p.dx, y: c.y + p.dy });
  }
  const moved = { ...state, graph: withPositions(state.graph, pos) };
  const out = dropAt(moved, ids, origin, p.target);
  return out.result.ok ? out : refuse(state, out.result.reason);
}

/** Prototipo, righe 3974-4024 (collegamento dalle porte): si collega soltanto, mai fusione. */
function connectCmd(
  state: EtlState,
  p: Extract<Command, { type: "connect" }>["payload"],
): ReduceOutcome {
  if (!nodeExists(state.graph, p.from) || !nodeExists(state.graph, p.to))
    return refuse(state, "Il nodo non esiste");
  if (p.from === p.to) return refuse(state, "Un nodo non si collega a sé stesso");
  const rel = relation(state.graph, p.from, p.to);
  if (rel.relation === "displace")
    return refuse(state, rel.displaceReason ?? "Questi due nodi non si possono collegare");
  if (rel.relation !== "link" && rel.relation !== "link-reverse")
    return refuse(state, "Questi due nodi non si possono collegare");
  const ids = new Ids(state.counters);
  const r =
    rel.relation === "link"
      ? connectSettled(state.graph, p.from, p.to, ids, state.mode)
      : connectSettled(state.graph, p.to, p.from, ids, state.mode);
  if (!r.ok) return refuse(state, r.reason);
  return done(withGraph(state, r.graph, ids));
}

function mergeCmd(
  state: EtlState,
  p: Extract<Command, { type: "merge" }>["payload"],
): ReduceOutcome {
  if (p.dragged === p.target) return refuse(state, "Un box non si fonde con sé stesso");
  if (!nodeExists(state.graph, p.dragged) || !nodeExists(state.graph, p.target))
    return refuse(state, "Il nodo non esiste");
  const ids = new Ids(state.counters);
  const r = mergeSettled(state.graph, p.dragged, p.target, ids, state.mode);
  if (!r.ok) return refuse(state, r.reason);
  const inspector: Inspector | undefined =
    state.inspector.nodeId === p.dragged ? { nodeId: p.target, step: 0 } : undefined;
  return done(withGraph(state, r.graph, ids, inspector ? { inspector } : {}));
}

function insertCmd(
  state: EtlState,
  p: Extract<Command, { type: "insertOnLink" }>["payload"],
): ReduceOutcome {
  if (!nodeExists(state.graph, p.node)) return refuse(state, "Il nodo non esiste");
  const ids = new Ids(state.counters);
  const r = insertSettled(state.graph, p.node, p.link, ids, state.mode);
  if (!r.ok) return refuse(state, r.reason);
  return done(withGraph(state, r.graph, ids));
}

function stepIndexError(graph: Graph, box: string, index: number): string | null {
  const c = cardById(graph, box);
  if (!c) return "Il box non esiste";
  if (c.kind !== "op") return "Un dataset non ha passaggi";
  if (!Number.isInteger(index) || index < 0 || index >= c.components.length)
    return "Il passaggio non esiste";
  return null;
}

/** Prototipo, righe 2137-2203 (`detachStep`). */
function detachCmd(
  state: EtlState,
  p: Extract<Command, { type: "detachStep" }>["payload"],
): ReduceOutcome {
  const err = stepIndexError(state.graph, p.box, p.index);
  if (err) return refuse(state, err);
  const ids = new Ids(state.counters);
  const r = coreDetachStep(
    state.graph,
    p.box,
    p.index,
    ids.gen("op"),
    detachPositionFn({ mode: state.mode, ...(p.dropPoint ? { dropPoint: p.dropPoint } : {}) }),
  );
  if (!r) return refuse(state, "Si sgancia un passaggio solo da un box combinato");
  return done(withGraph(state, settleCreated(state.graph, r.graph, state.mode), ids));
}

/** Prototipo, righe 2205-2248 (`deleteStep`). */
function deleteStepCmd(
  state: EtlState,
  p: Extract<Command, { type: "deleteStep" }>["payload"],
): ReduceOutcome {
  const err = stepIndexError(state.graph, p.box, p.index);
  if (err) return refuse(state, err);
  if ((state.graph.cards[p.box] as Card).components.length < 2)
    return refuse(state, "Si elimina un passaggio solo da un box combinato");
  const ids = new Ids(state.counters);
  const next = coreDeleteStep(
    state.graph,
    p.box,
    p.index,
    ids.gen("out"),
    outputPositionFn({ mode: state.mode }),
  );
  const inspector: Inspector | undefined =
    state.inspector.nodeId === p.box ? { nodeId: p.box, step: 0 } : undefined;
  return done(
    withGraph(
      state,
      settleCreated(state.graph, next, state.mode),
      ids,
      inspector ? { inspector } : {},
    ),
  );
}

/** Prototipo, righe 2330-2338 (vista espansa) e 3834-3843 (inspector). */
function reorderCmd(
  state: EtlState,
  p: Extract<Command, { type: "reorderSteps" }>["payload"],
): ReduceOutcome {
  const err =
    stepIndexError(state.graph, p.box, p.from) ?? stepIndexError(state.graph, p.box, p.to);
  if (err) return refuse(state, err);
  if (p.from === p.to) return done(state);
  const next = coreReorderSteps(state.graph, p.box, p.from, p.to);
  const inspector: Inspector | undefined =
    state.inspector.nodeId === p.box ? { nodeId: p.box, step: p.to } : undefined;
  return done(withGraph(state, next, new Ids(state.counters), inspector ? { inspector } : {}));
}

/** Prototipo, righe 4480-4506 (`commitDelete`) e 4407 (`renderAll` ricrea gli output). */
function deleteNodesCmd(
  state: EtlState,
  p: Extract<Command, { type: "deleteNodes" }>["payload"],
): ReduceOutcome {
  const targets = p.ids.filter((id) => nodeExists(state.graph, id));
  if (!targets.length) return refuse(state, "Nessun nodo da eliminare");
  const ids = new Ids(state.counters);
  const next = coreDeleteNodes(
    state.graph,
    targets,
    ids.gen("out"),
    outputPositionFn({ mode: state.mode }),
  );
  return done(withGraph(state, settleCreated(state.graph, next, state.mode), ids));
}

/** Prototipo, righe 4565-4573 (`deleteLink`). */
function deleteLinkCmd(
  state: EtlState,
  p: Extract<Command, { type: "deleteLink" }>["payload"],
): ReduceOutcome {
  if (!state.graph.links.some((l) => sameLink(l, p.link)))
    return refuse(state, "Il collegamento non esiste");
  const ids = new Ids(state.counters);
  const next = coreDeleteLink(
    state.graph,
    p.link,
    ids.gen("out"),
    outputPositionFn({ mode: state.mode }),
  );
  return done(withGraph(state, settleCreated(state.graph, next, state.mode), ids));
}

/** Prototipo, righe 4594-4618 (`duplicateSelection`): gli output non si duplicano. */
function duplicateCmd(
  state: EtlState,
  p: Extract<Command, { type: "duplicate" }>["payload"],
): ReduceOutcome {
  const src = p.ids.filter((id) => {
    const c = state.graph.cards[id];
    return !!c && !c.isOutput;
  });
  if (!src.length) return refuse(state, "Niente da duplicare: gli output non si duplicano");
  const ids = new Ids(state.counters);
  const prefixes = src.map(
    (id) => ((state.graph.cards[id] as Card).kind === "dataset" ? "ds" : "op") as "ds" | "op",
  );
  let k = 0;
  const { graph, createdIds } = duplicateNodes(
    state.graph,
    src,
    () => ids.next(prefixes[k++] ?? "op"),
    {
      x: GRID * 2,
      y: GRID * 2,
    },
  );
  let next = graph;
  if (state.mode === "grid") {
    const slots = computeSlots();
    for (const id of createdIds) {
      const c = next.cards[id] as Card;
      next = setSlot(next, id, firstFreeSlot(next, slots, c.x, c.y, id));
    }
    next = placeInSlots(next, slots);
  } else if (anyOverlap(next)) next = resolveOverlaps(next, { snap: true });
  return done(withGraph(state, next, ids, { selection: createdIds }));
}

/** Parametri di un passaggio (prototipo: modifica diretta nell'inspector, righe 3515-3900). */
function setParamsCmd(
  state: EtlState,
  p: Extract<Command, { type: "setParams" }>["payload"],
): ReduceOutcome {
  const c = state.graph.cards[p.node];
  if (!c) return refuse(state, "Il nodo non esiste");
  if (!Number.isInteger(p.index) || p.index < 0 || p.index >= c.components.length)
    return refuse(state, "Il passaggio non esiste");
  const card = withParamAt(c, p.index, p.params);
  return done({
    ...state,
    graph: { ...state.graph, cards: { ...state.graph.cards, [p.node]: card } },
  });
}

/** Nome del nodo (prototipo, righe 3860-3870: al blur; un nome vuoto lasciava il precedente). */
function renameCmd(
  state: EtlState,
  p: Extract<Command, { type: "renameNode" }>["payload"],
): ReduceOutcome {
  const c = state.graph.cards[p.node];
  if (!c) return refuse(state, "Il nodo non esiste");
  const name = p.name.trim();
  if (!name) return refuse(state, "Il nome non può essere vuoto");
  if (name === c.name) return done(state);
  return done({
    ...state,
    graph: { ...state.graph, cards: { ...state.graph.cards, [p.node]: { ...c, name } } },
  });
}

/** Prototipo, righe 3952-3966 (`setMode`). */
function setModeCmd(
  state: EtlState,
  p: Extract<Command, { type: "setMode" }>["payload"],
): ReduceOutcome {
  if (p.mode !== "free" && p.mode !== "grid") return refuse(state, "Modalità sconosciuta");
  if (p.mode === state.mode) return done(state);
  const graph = p.mode === "grid" ? assignSlots(state.graph) : clearSlots(state.graph);
  return done({ ...state, mode: p.mode, graph });
}

/** Prototipo, righe 4220-4327 (`autoLayout`). */
function autoLayoutCmd(
  state: EtlState,
  p: Extract<Command, { type: "autoLayout" }>["payload"],
): ReduceOutcome {
  if (!(p.viewport.w > 0 && p.viewport.h > 0)) return refuse(state, "Area visibile non valida");
  if (!Object.keys(state.graph.cards).length) return done(state);
  return done({
    ...state,
    graph: layoutAuto(state.graph, { viewport: p.viewport, mode: state.mode }),
  });
}

/** Prototipo, righe 2693-2727 (selezione). */
function selectCmd(
  state: EtlState,
  p: Extract<Command, { type: "select" }>["payload"],
): ReduceOutcome {
  const missing = p.ids.find((id) => !nodeExists(state.graph, id));
  if (missing) return refuse(state, `Il nodo ${missing} non esiste`);
  return done({ ...state, selection: Array.from(new Set(p.ids)) });
}

/** Prototipo, righe 2713-2727 (`selectCard`, `deselect`) e 3830 (passaggio selezionato). */
function inspectCmd(
  state: EtlState,
  p: Extract<Command, { type: "inspect" }>["payload"],
): ReduceOutcome {
  if (p.node === null) return done({ ...state, inspector: { nodeId: null, step: 0 } });
  const c = state.graph.cards[p.node];
  if (!c) return refuse(state, "Il nodo non esiste");
  const step = p.step ?? 0;
  if (!Number.isInteger(step) || step < 0 || step >= c.components.length)
    return refuse(state, "Il passaggio non esiste");
  return done({ ...state, inspector: { nodeId: p.node, step } });
}

/** Prototipo, righe 4921-4937 (caricamento di un CSV nella libreria). */
function loadDatasetCmd(
  state: EtlState,
  p: Extract<Command, { type: "loadDataset" }>["payload"],
): ReduceOutcome {
  if (!p.columns.length) return refuse(state, "Il file non contiene colonne leggibili");
  if (!p.name.trim()) return refuse(state, "Il dataset deve avere un nome");
  const lib = state.counters.lib + 1;
  const item: LibraryItem = {
    id: "lib-" + lib,
    name: p.name,
    path: p.path,
    columns: p.columns,
    rows: p.rows,
  };
  return done({
    ...state,
    library: [...state.library, item],
    counters: { ...state.counters, lib },
  });
}

/** Prototipo, righe 4826-4833 (`applyOpen`), 4835-4842 (`setPanelOpen`), 4865-4877 (`setSide`). */
function setPanelCmd(
  state: EtlState,
  p: Extract<Command, { type: "setPanel" }>["payload"],
): ReduceOutcome {
  if (p.panel !== "tools" && p.panel !== "insp") return refuse(state, "Pannello sconosciuto");
  const sides = ["left", "right", "top", "bottom"];
  if (p.side !== undefined && !sides.includes(p.side)) return refuse(state, "Lato sconosciuto");
  const other: PanelKey = p.panel === "tools" ? "insp" : "tools";
  let panels: Panels = state.panels;
  let open = p.open;
  if (p.side !== undefined && p.side !== panels[p.panel].side) {
    // cambiare lato chiude il pannello, lo sposta e lo riapre (righe 4867-4876)
    panels = { ...panels, [p.panel]: { side: p.side, open: false } };
    open = open ?? true;
  }
  if (open === true) {
    // sullo stesso bordo si apre un pannello alla volta
    if (panels[other].side === panels[p.panel].side)
      panels = { ...panels, [other]: { ...panels[other], open: false } };
    panels = { ...panels, [p.panel]: { ...panels[p.panel], open: true } };
  } else if (open === false) {
    panels = { ...panels, [p.panel]: { ...panels[p.panel], open: false } };
  }
  return done({ ...state, panels });
}

/** Prototipo, righe 937-946 (`view`), 4104-4110 (`zoomAt`: zoom tra 0,35 e 2). */
function setViewCmd(
  state: EtlState,
  p: Extract<Command, { type: "setView" }>["payload"],
): ReduceOutcome {
  const v = { ...state.view, ...p };
  if (![v.x, v.y, v.zoom].every((n) => Number.isFinite(n)))
    return refuse(state, "Vista non valida");
  return done({
    ...state,
    view: { x: v.x, y: v.y, zoom: Math.max(ZOOM_MIN, Math.min(ZOOM_MAX, v.zoom)) },
  });
}

function setOptionsCmd(
  state: EtlState,
  p: Extract<Command, { type: "setOptions" }>["payload"],
): ReduceOutcome {
  if (p.maxBends !== undefined && !(Number.isInteger(p.maxBends) && p.maxBends >= 0))
    return refuse(state, "Il numero di snodi deve essere un intero non negativo");
  if (p.flowOnlyIfValid !== undefined && typeof p.flowOnlyIfValid !== "boolean")
    return refuse(state, "Valore non valido per 'flusso solo se valido'");
  return done({ ...state, options: { ...state.options, ...p } });
}

/** Applica un comando. Un comando rifiutato non modifica lo stato. */
export function reduce(state: EtlState, command: Command): ReduceOutcome {
  switch (command.type) {
    case "addNode":
      return addNode(state, command.payload);
    case "moveNodes":
      return moveNodes(state, command.payload);
    case "dropNodes":
      return dropNodes(state, command.payload);
    case "connect":
      return connectCmd(state, command.payload);
    case "merge":
      return mergeCmd(state, command.payload);
    case "insertOnLink":
      return insertCmd(state, command.payload);
    case "detachStep":
      return detachCmd(state, command.payload);
    case "deleteStep":
      return deleteStepCmd(state, command.payload);
    case "reorderSteps":
      return reorderCmd(state, command.payload);
    case "deleteNodes":
      return deleteNodesCmd(state, command.payload);
    case "deleteLink":
      return deleteLinkCmd(state, command.payload);
    case "duplicate":
      return duplicateCmd(state, command.payload);
    case "setParams":
      return setParamsCmd(state, command.payload);
    case "renameNode":
      return renameCmd(state, command.payload);
    case "setMode":
      return setModeCmd(state, command.payload);
    case "autoLayout":
      return autoLayoutCmd(state, command.payload);
    case "select":
      return selectCmd(state, command.payload);
    case "inspect":
      return inspectCmd(state, command.payload);
    case "loadDataset":
      return loadDatasetCmd(state, command.payload);
    case "setPanel":
      return setPanelCmd(state, command.payload);
    case "setView":
      return setViewCmd(state, command.payload);
    case "setOptions":
      return setOptionsCmd(state, command.payload);
    default: {
      const unknownType: string = (command as { type: string }).type;
      return refuse(state, `Comando sconosciuto: ${unknownType}`);
    }
  }
}
```

### `src/etl-store/serialize.ts`

184 righe

```ts
/**
 * Formato di salvataggio, puro (nessun accesso al browser: vedi
 * persistence.ts). Si salvano grafo, modalità, libreria, pannelli e
 * opzioni, con un numero di versione. Non si salvano cronologia, selezione,
 * inspector, vista, percorsi.
 */
import { META } from "../etl-core";
import type { Card, ColumnDef, Graph, Link } from "../etl-core";
import { DEFAULT_OPTIONS, initialState } from "./state";
import type { EtlState, LibraryItem, Options, PanelState, Panels } from "./types";

/** Versione del formato. Un dato di versione diversa viene ignorato. */
export const SAVE_VERSION = 1;

export interface SavedState {
  readonly version: typeof SAVE_VERSION;
  readonly graph: Graph;
  readonly mode: EtlState["mode"];
  readonly library: readonly LibraryItem[];
  readonly panels: Panels;
  readonly options: Options;
}

export function toSaved(state: EtlState): SavedState {
  return {
    version: SAVE_VERSION,
    graph: state.graph,
    mode: state.mode,
    library: state.library,
    panels: state.panels,
    options: state.options,
  };
}

// --- Validazione -------------------------------------------------------------

type Obj = Record<string, unknown>;

function isObj(v: unknown): v is Obj {
  return typeof v === "object" && v !== null && !Array.isArray(v);
}
function isNum(v: unknown): v is number {
  return typeof v === "number" && Number.isFinite(v);
}
function isStr(v: unknown): v is string {
  return typeof v === "string";
}

function validColumn(v: unknown): v is ColumnDef {
  return (
    isObj(v) &&
    isStr(v["name"]) &&
    isStr(v["type"]) &&
    Array.isArray(v["values"]) &&
    v["values"].every(isStr)
  );
}

function validCard(id: string, v: unknown): v is Card {
  if (!isObj(v)) return false;
  if (v["id"] !== id || !isStr(v["name"])) return false;
  if (v["kind"] !== "dataset" && v["kind"] !== "op") return false;
  if (!isNum(v["x"]) || !isNum(v["y"])) return false;
  const comps = v["components"];
  const params = v["params"];
  if (!Array.isArray(comps) || !comps.length) return false;
  if (!comps.every((c) => isStr(c) && Object.prototype.hasOwnProperty.call(META, c))) return false;
  if ((v["kind"] === "dataset") !== (comps.length === 1 && comps[0] === "dataset")) return false;
  if (!Array.isArray(params) || !params.every(isObj)) return false;
  for (const k of ["slot", "capacity", "filled"])
    if (v[k] !== undefined && !isNum(v[k])) return false;
  if (v["isOutput"] !== undefined && typeof v["isOutput"] !== "boolean") return false;
  return true;
}

function validGraph(v: unknown): v is Graph {
  if (!isObj(v) || !isObj(v["cards"]) || !Array.isArray(v["links"])) return false;
  const cards = v["cards"];
  for (const [id, c] of Object.entries(cards)) if (!validCard(id, c)) return false;
  return v["links"].every(
    (l: unknown) =>
      isObj(l) && isStr(l["from"]) && isStr(l["to"]) && !!cards[l["from"]] && !!cards[l["to"]],
  );
}

function validLibraryItem(v: unknown): v is LibraryItem {
  return (
    isObj(v) &&
    isStr(v["id"]) &&
    isStr(v["name"]) &&
    isStr(v["path"]) &&
    isNum(v["rows"]) &&
    Array.isArray(v["columns"]) &&
    v["columns"].every(validColumn)
  );
}

function validPanel(v: unknown): v is PanelState {
  return (
    isObj(v) &&
    typeof v["open"] === "boolean" &&
    (v["side"] === "left" || v["side"] === "right" || v["side"] === "top" || v["side"] === "bottom")
  );
}

function validOptions(v: unknown): v is Options {
  return (
    isObj(v) &&
    typeof v["flowOnlyIfValid"] === "boolean" &&
    isNum(v["maxBends"]) &&
    Number.isInteger(v["maxBends"]) &&
    v["maxBends"] >= 0
  );
}

/** Il più grande N fra gli identificativi "prefisso-N" (per non riusare id dopo il caricamento). */
function maxSuffix(ids: readonly string[], re: RegExp): number {
  let max = -1;
  for (const id of ids) {
    const m = re.exec(id);
    if (m) max = Math.max(max, Number(m[1]));
  }
  return max;
}

/**
 * Ricostruisce lo stato da un dato salvato. Un dato non valido o di
 * versione sconosciuta restituisce `null` (il chiamante parte da un canvas
 * vuoto), senza eccezioni.
 */
export function fromSaved(raw: unknown): EtlState | null {
  try {
    if (!isObj(raw) || raw["version"] !== SAVE_VERSION) return null;
    const { graph, mode, library, panels, options } = raw;
    if (!validGraph(graph)) return null;
    if (mode !== "free" && mode !== "grid") return null;
    if (!Array.isArray(library) || !library.every(validLibraryItem)) return null;
    if (!isObj(panels) || !validPanel(panels["tools"]) || !validPanel(panels["insp"])) return null;
    if (!validOptions(options)) return null;
    const cards = Object.values(graph.cards);
    const base = initialState();
    return {
      ...base,
      graph: {
        cards: { ...graph.cards },
        links: graph.links.map((l: Link) => ({ from: l.from, to: l.to })),
      },
      mode,
      library: [...library],
      panels: { tools: { ...panels["tools"] }, insp: { ...panels["insp"] } } as Panels,
      options: { ...DEFAULT_OPTIONS, ...options },
      counters: {
        uid: maxSuffix(Object.keys(graph.cards), /^(?:ds|op|out)-(\d+)$/) + 1,
        ds: Math.max(
          0,
          maxSuffix(
            cards.map((c) => c.name),
            /^Dataset (\d+)$/,
          ),
        ),
        lib: Math.max(
          0,
          maxSuffix(
            library.map((l: LibraryItem) => l.id),
            /^lib-(\d+)$/,
          ),
        ),
      },
    };
  } catch {
    return null;
  }
}

/** Da testo JSON (quello scritto dal salvataggio) allo stato, o `null`. */
export function parseSaved(text: string | null): EtlState | null {
  if (!text) return null;
  try {
    return fromSaved(JSON.parse(text) as unknown);
  } catch {
    return null;
  }
}
```

### `src/etl-store/state.ts`

43 righe

```ts
/** Stato iniziale e costanti dello stato. */
import { createGraph } from "../etl-core";
import { MAX_BENDS } from "../etl-layout";
import type { EtlState, Options, Panels, View } from "./types";

/**
 * Pannelli iniziali (prototipo, righe 4758-4760): strumenti a sinistra e
 * aperti, inspector a destra e chiuso.
 */
export const DEFAULT_PANELS: Panels = {
  tools: { side: "left", open: true },
  insp: { side: "right", open: false },
};

export const DEFAULT_VIEW: View = { x: 0, y: 0, zoom: 1 };

/** Zoom minimo e massimo (prototipo, `zoomAt`, riga 4105). */
export const ZOOM_MIN = 0.35;
export const ZOOM_MAX = 2;

/**
 * Opzioni dell'utente. Il pannello "Funzionalità" del prototipo (righe
 * 914-935) era uno strumento di progettazione e non è portato: tutte le
 * funzionalità sono sempre attive. Resta solo "flusso solo se valido"
 * (`flowGate`, spento, riga 918).
 */
export const DEFAULT_OPTIONS: Options = { flowOnlyIfValid: false, maxBends: MAX_BENDS };

/** Canvas vuoto: nessun nodo, libreria vuota, modalità Libero. */
export function initialState(): EtlState {
  return {
    graph: createGraph(),
    mode: "free",
    library: [],
    selection: [],
    inspector: { nodeId: null, step: 0 },
    panels: DEFAULT_PANELS,
    view: DEFAULT_VIEW,
    options: DEFAULT_OPTIONS,
    counters: { uid: 0, ds: 0, lib: 0 },
  };
}
```

### `src/etl-store/store.ts`

434 righe

```ts
/**
 * L'archivio unico: stato, comandi, cronologia, gesti e registro delle
 * attività. Nessun React né API del browser: gira in Node.
 */
import { parseCSV } from "../etl-core";
import type { Card, Graph } from "../etl-core";
import { displace, separateWhileDragging, withPositions } from "../etl-layout";
import type { LinkRoutes, Point } from "../etl-layout";
import { relation } from "../etl-core";
import { createRoutesCache } from "./derived";
import { dropAt, reduce } from "./reduce";
import { initialState } from "./state";
import type {
  Command,
  CommandResult,
  CommandType,
  DropTarget,
  EtlState,
  HistoryEntry,
  LogEntry,
} from "./types";

/** Profondità della cronologia (prototipo, `HIST_MAX = 50`, riga 4335). */
export const HISTORY_LIMIT = 50;

/**
 * Comandi che creano un passo di cronologia, cioè quelli per cui il
 * prototipo chiama `pushHistory()` (righe 1983, 2140, 2208, 2331, 3838,
 * 3954, 4017, 4223, 4481, 4568, 4597, 4623, 4996), più `setParams` e
 * `renameNode` (aggiunte, vedi README). Un comando di questo elenco crea
 * un passo solo se riesce e cambia davvero grafo, contatori o modalità.
 */
export const HISTORY_COMMANDS: ReadonlySet<CommandType> = new Set<CommandType>([
  "addNode",
  "moveNodes",
  "dropNodes",
  "connect",
  "merge",
  "insertOnLink",
  "detachStep",
  "deleteStep",
  "reorderSteps",
  "deleteNodes",
  "deleteLink",
  "duplicate",
  "setParams",
  "renameNode",
  "setMode",
  "autoLayout",
]);

/**
 * Comandi consecutivi con la stessa chiave, arrivati entro questo
 * intervallo dal precedente, formano un solo passo di cronologia e una sola
 * voce di registro (Fase 3.1: scrivere in un campo, tenere premuta una
 * freccia).
 */
export const GROUP_WINDOW_MS = 1000;

/** Comandi che non entrano nel registro: la vista cambia decine di volte al secondo (Fase 3.1). */
export const UNLOGGED_COMMANDS: ReadonlySet<CommandType> = new Set<CommandType>(["setView"]);

/**
 * Chiave di raggruppamento: `setParams` → nodo + passaggio; `renameNode` →
 * nodo; `moveNodes` → insieme degli identificativi (ordinato). `null` per
 * tutti gli altri comandi, che non si raggruppano.
 */
export function groupKey(command: Command): string | null {
  switch (command.type) {
    case "setParams":
      return `setParams|${command.payload.node}|${command.payload.index}`;
    case "renameNode":
      return `renameNode|${command.payload.node}`;
    case "moveNodes":
      return `moveNodes|${JSON.stringify([...new Set(command.payload.ids)].sort())}`;
    default:
      return null;
  }
}

export interface GestureStart {
  readonly ids: readonly string[];
}

export interface GestureUpdate {
  /** Spostamento dall'inizio del gesto, in coordinate del mondo. */
  readonly dx: number;
  readonly dy: number;
  /** Nodo sotto il puntatore, se c'è (per spingere via i nodi incompatibili). */
  readonly over?: string | null;
}

export interface GestureEnd {
  readonly target?: DropTarget;
}

export type Listener = () => void;

export interface EtlStore {
  getState(): EtlState;
  subscribe(listener: Listener): () => void;
  dispatch(command: Command): CommandResult;
  undo(): CommandResult;
  redo(): CommandResult;
  canUndo(): boolean;
  canRedo(): boolean;
  /** Numero di passi annullabili e ripristinabili. */
  historySize(): { past: number; future: number };
  beginGesture(start: GestureStart): CommandResult;
  updateGesture(update: GestureUpdate): CommandResult;
  commitGesture(end?: GestureEnd): CommandResult;
  cancelGesture(): CommandResult;
  isGesturing(): boolean;
  /** Registro delle attività (sola aggiunta). */
  getLog(): readonly LogEntry[];
  /** Il registro come JSON. */
  exportLog(): string;
  /** Legge un CSV e carica nella libreria SOLO i metadati (nome, percorso, colonne, righe). */
  loadCsv(text: string, fileName: string): CommandResult;
  /** Percorsi dei cavi, derivati e con memoria. */
  getRoutes(): LinkRoutes;
  /** Sostituisce lo stato (caricamento salvato): azzera cronologia e gesto, non il registro. */
  replaceState(state: EtlState): void;
}

export interface StoreOptions {
  readonly initial?: EtlState;
  /** Orologio per il registro (predefinito: Date.now). */
  readonly now?: () => number;
  readonly historyLimit?: number;
}

interface Gesture {
  readonly ids: readonly string[];
  readonly start: EtlState;
  readonly origin: ReadonlyMap<string, Point>;
  lastDisplaced: string | null;
}

function entryOf(state: EtlState): HistoryEntry {
  return {
    graph: state.graph,
    counters: { uid: state.counters.uid, ds: state.counters.ds },
    mode: state.mode,
  };
}

function sameEntry(a: EtlState, b: EtlState): boolean {
  return (
    a.graph === b.graph &&
    a.mode === b.mode &&
    a.counters.uid === b.counters.uid &&
    a.counters.ds === b.counters.ds
  );
}

/**
 * Ripristina un punto di cronologia; selezione e inspector perdono i nodi
 * spariti. Se il nodo dell'inspector esiste ma ha meno passaggi di prima
 * (es. annullando una fusione), l'indice va all'ultimo passaggio esistente
 * (Fase 3.1).
 */
function restore(state: EtlState, e: HistoryEntry): EtlState {
  const exists = (id: string): boolean => !!e.graph.cards[id];
  const insp = state.inspector;
  const node = insp.nodeId ? e.graph.cards[insp.nodeId] : undefined;
  const inspector =
    insp.nodeId && !node
      ? { nodeId: null, step: 0 }
      : node && (insp.step >= node.components.length || insp.step < 0)
        ? { nodeId: insp.nodeId, step: Math.max(0, node.components.length - 1) }
        : insp;
  return {
    ...state,
    graph: e.graph,
    mode: e.mode,
    counters: { ...state.counters, uid: e.counters.uid, ds: e.counters.ds },
    selection: state.selection.filter(exists),
    inspector,
  };
}

function positionsOf(graph: Graph, ids: readonly string[]): Record<string, Point> {
  const out: Record<string, Point> = {};
  for (const id of ids) {
    const c = graph.cards[id];
    if (c) out[id] = { x: c.x, y: c.y };
  }
  return out;
}

/** Copia serializzabile in JSON (il registro non conserva riferimenti vivi). */
function plain(v: unknown): unknown {
  return v === undefined ? null : (JSON.parse(JSON.stringify(v)) as unknown);
}

export function createEtlStore(opts: StoreOptions = {}): EtlStore {
  const now = opts.now ?? (() => Date.now());
  const limit = opts.historyLimit ?? HISTORY_LIMIT;
  let state: EtlState = opts.initial ?? initialState();
  let past: HistoryEntry[] = [];
  let future: HistoryEntry[] = [];
  let gesture: Gesture | null = null;
  /** Gruppo di comandi in corso: chiave, istante dell'ultimo comando, voce di registro, passo creato. */
  let group: { key: string; last: number; logIndex: number; stepped: boolean } | null = null;
  const log: LogEntry[] = [];
  const listeners = new Set<Listener>();
  const routes = createRoutesCache();

  const emit = (): void => {
    for (const l of [...listeners]) l();
  };
  const record = (
    type: string,
    payload: unknown,
    result: CommandResult,
    time: number = now(),
  ): void => {
    log.push({ id: log.length + 1, time, type, payload: plain(payload), result });
  };
  /** Nuovo passo: `before` va nella cronologia, i passi da ripristinare si cancellano (riga 4343). */
  const pushStep = (before: EtlState): void => {
    past.push(entryOf(before));
    if (past.length > limit) past.shift();
    future = [];
  };
  const setState = (next: EtlState): void => {
    if (next === state) return;
    state = next;
    emit();
  };
  const dropGesture = (): void => {
    if (!gesture) return;
    const start = gesture.start;
    gesture = null;
    setState({ ...state, graph: start.graph });
  };

  const store: EtlStore = {
    getState: () => state,
    subscribe(listener) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },

    dispatch(command) {
      if (gesture) dropGesture();
      const t = now();
      const key = groupKey(command);
      const g = group;
      const joinable = !!g && key !== null && key === g.key && t - g.last <= GROUP_WINDOW_MS;
      const before = state;
      const { state: next, result } = reduce(state, command);
      const historic = HISTORY_COMMANDS.has(command.type) && !sameEntry(before, next);

      if (joinable && g && result.ok) {
        // stesso gruppo: nessun nuovo passo; l'ultima voce del registro si aggiorna
        const prev = log[g.logIndex] as LogEntry;
        log[g.logIndex] = {
          ...prev,
          payload: plain(command.payload),
          result,
          until: t,
          count: (prev.count ?? 1) + 1,
        };
        if (historic && !g.stepped) {
          pushStep(before);
          g.stepped = true;
        }
        g.last = t;
        setState(next);
        return result;
      }

      // qualunque altro comando (o un rifiuto) interrompe il gruppo
      group = null;
      if (!UNLOGGED_COMMANDS.has(command.type)) record(command.type, command.payload, result, t);
      if (!result.ok) return result;
      if (historic) pushStep(before);
      if (key !== null) group = { key, last: t, logIndex: log.length - 1, stepped: historic };
      setState(next);
      return result;
    },

    /** Prototipo, righe 4392-4397 (`undo`). */
    undo() {
      if (gesture) dropGesture();
      group = null;
      const e = past.pop();
      const result: CommandResult = e ? { ok: true } : { ok: false, reason: "Niente da annullare" };
      record("undo", {}, result);
      if (!e) return result;
      future.push(entryOf(state));
      setState(restore(state, e));
      return result;
    },

    /** Prototipo, righe 4398-4404 (`redo`). */
    redo() {
      if (gesture) dropGesture();
      group = null;
      const e = future.pop();
      const result: CommandResult = e
        ? { ok: true }
        : { ok: false, reason: "Niente da ripristinare" };
      record("redo", {}, result);
      if (!e) return result;
      past.push(entryOf(state));
      if (past.length > limit) past.shift();
      setState(restore(state, e));
      return result;
    },

    canUndo: () => past.length > 0,
    canRedo: () => future.length > 0,
    historySize: () => ({ past: past.length, future: future.length }),

    /**
     * Inizio di un trascinamento (prototipo, riga 1983: il passo si prepara
     * qui, con lo stato prima del gesto). Nessuna voce nel registro finché
     * il gesto non si conclude.
     */
    beginGesture(start) {
      if (gesture) dropGesture();
      group = null;
      const ids = start.ids.filter((id) => !!state.graph.cards[id]);
      if (!ids.length) return { ok: false, reason: "Nessun nodo da trascinare" };
      const origin = new Map<string, Point>();
      for (const id of ids) {
        const c = state.graph.cards[id] as Card;
        origin.set(id, { x: c.x, y: c.y });
      }
      gesture = { ids, start: state, origin, lastDisplaced: null };
      return { ok: true };
    },

    /**
     * Aggiornamento transitorio (prototipo, righe 1979-2056, `onMove`): i
     * nodi seguono il puntatore; in Libero, con un solo nodo, le coppie
     * incompatibili si scansano e un nodo incompatibile sotto il puntatore
     * viene spinto via. Né cronologia né registro.
     */
    updateGesture(update) {
      const g = gesture;
      if (!g) return { ok: false, reason: "Nessun gesto in corso" };
      if (!Number.isFinite(update.dx) || !Number.isFinite(update.dy))
        return { ok: false, reason: "Spostamento non valido" };
      const pos = new Map<string, Point>();
      for (const [id, o] of g.origin) pos.set(id, { x: o.x + update.dx, y: o.y + update.dy });
      let graph = withPositions(state.graph, pos);
      if (g.ids.length === 1 && state.mode === "free") {
        const id = g.ids[0] as string;
        graph = separateWhileDragging(graph, id);
        const over = update.over ?? null;
        if (over && over !== id && graph.cards[over]) {
          if (relation(graph, id, over).relation === "displace" && over !== g.lastDisplaced) {
            graph = displace(graph, id, over);
            g.lastDisplaced = over;
          }
        } else g.lastDisplaced = null;
      }
      setState({ ...state, graph });
      return { ok: true };
    },

    /**
     * Rilascio (prototipo, righe 2058-2105): un solo passo di cronologia e
     * una sola voce di registro, con le posizioni iniziali e finali.
     */
    commitGesture(end = {}) {
      group = null;
      const g = gesture;
      if (!g) return { ok: false, reason: "Nessun gesto in corso" };
      gesture = null;
      const out = dropAt(state, g.ids, g.origin, end.target);
      const from: Record<string, Point> = {};
      for (const [id, o] of g.origin) from[id] = o;
      if (!out.result.ok) {
        record(
          "gesture",
          { kind: "move", ids: g.ids, from, to: from, target: end.target ?? null },
          out.result,
        );
        setState({ ...state, graph: g.start.graph });
        return out.result;
      }
      const to = positionsOf(out.state.graph, g.ids);
      record(
        "gesture",
        { kind: "move", ids: g.ids, from, to, target: end.target ?? null },
        out.result,
      );
      if (!sameEntry(g.start, out.state)) pushStep(g.start);
      setState(out.state);
      return out.result;
    },

    /** Gesto annullato: si torna allo stato di partenza, senza passi né voci. */
    cancelGesture() {
      if (!gesture) return { ok: false, reason: "Nessun gesto in corso" };
      dropGesture();
      return { ok: true };
    },

    isGesturing: () => gesture !== null,
    getLog: () => log,
    exportLog: () => JSON.stringify(log, null, 2),

    loadCsv(text, fileName) {
      const parsed = parseCSV(text);
      return store.dispatch({
        type: "loadDataset",
        payload: {
          name: fileName.replace(/\.[^.]+$/, ""),
          path: fileName,
          columns: parsed ? parsed.columns : [],
          rows: parsed ? parsed.rows : 0,
        },
      });
    },

    getRoutes: () => routes(state),

    replaceState(next) {
      gesture = null;
      group = null;
      past = [];
      future = [];
      setState(next);
    },
  };
  return store;
}
```

### `src/etl-store/types.ts`

196 righe

```ts
/** Tipi dello stato dell'applicazione (Fase 3). */
import type { ColumnDef, ComponentId, Graph, Link, Params } from "../etl-core";
import type { LayoutMode, Point, Size } from "../etl-layout";

export type PanelKey = "tools" | "insp";
export type Side = "left" | "right" | "top" | "bottom";

export interface PanelState {
  readonly side: Side;
  readonly open: boolean;
}

export type Panels = Readonly<Record<PanelKey, PanelState>>;

/** Un dataset caricato nella libreria: solo metadati, mai il contenuto del file. */
export interface LibraryItem {
  readonly id: string;
  readonly name: string;
  readonly path: string;
  readonly columns: readonly ColumnDef[];
  /** Numero di righe del file (senza intestazione). */
  readonly rows: number;
}

export interface Options {
  /** "Flusso solo se valido" (prototipo, `FEATURES.flowGate`, riga 918): spento per impostazione predefinita. */
  readonly flowOnlyIfValid: boolean;
  /** Snodi ammessi per cavo (etl-layout). */
  readonly maxBends: number;
}

export interface View {
  readonly x: number;
  readonly y: number;
  readonly zoom: number;
}

export interface Inspector {
  readonly nodeId: string | null;
  readonly step: number;
}

/**
 * Contatori per nomi e identificativi, come `uidCounter`/`dsCounter`/
 * `libCounter` del prototipo (righe 976, 4670). `uid` e `ds` fanno parte
 * dei punti di cronologia (riga 4337); `lib` no, come la libreria.
 */
export interface Counters {
  readonly uid: number;
  readonly ds: number;
  readonly lib: number;
}

export interface EtlState {
  readonly graph: Graph;
  readonly mode: LayoutMode;
  readonly library: readonly LibraryItem[];
  readonly selection: readonly string[];
  readonly inspector: Inspector;
  readonly panels: Panels;
  readonly view: View;
  readonly options: Options;
  readonly counters: Counters;
}

/** Dove viene rilasciato qualcosa: su un nodo o su un cavo. */
export type DropTarget = { readonly node: string } | { readonly link: Link };

export type Command =
  /** Nuovo nodo dalla cassetta (`type`) o dalla libreria (`libraryId`), rilasciato in `point` (coordinate del mondo). */
  | {
      readonly type: "addNode";
      readonly payload: {
        readonly component: ComponentId;
        readonly libraryId?: string;
        readonly point: Point;
        readonly target?: DropTarget;
      };
    }
  /** Spostamento da tastiera (prototipo `nudgeSelection`). */
  | {
      readonly type: "moveNodes";
      readonly payload: {
        readonly ids: readonly string[];
        readonly dx: number;
        readonly dy: number;
      };
    }
  /** Trascinamento concluso in un solo comando: spostamento di (dx, dy) e rilascio. */
  | {
      readonly type: "dropNodes";
      readonly payload: {
        readonly ids: readonly string[];
        readonly dx: number;
        readonly dy: number;
        readonly target?: DropTarget;
      };
    }
  | { readonly type: "connect"; readonly payload: { readonly from: string; readonly to: string } }
  | {
      readonly type: "merge";
      readonly payload: { readonly dragged: string; readonly target: string };
    }
  | {
      readonly type: "insertOnLink";
      readonly payload: { readonly node: string; readonly link: Link };
    }
  | {
      readonly type: "detachStep";
      readonly payload: {
        readonly box: string;
        readonly index: number;
        readonly dropPoint?: Point;
      };
    }
  | {
      readonly type: "deleteStep";
      readonly payload: { readonly box: string; readonly index: number };
    }
  | {
      readonly type: "reorderSteps";
      readonly payload: { readonly box: string; readonly from: number; readonly to: number };
    }
  | { readonly type: "deleteNodes"; readonly payload: { readonly ids: readonly string[] } }
  | { readonly type: "deleteLink"; readonly payload: { readonly link: Link } }
  | { readonly type: "duplicate"; readonly payload: { readonly ids: readonly string[] } }
  | {
      readonly type: "setParams";
      readonly payload: { readonly node: string; readonly index: number; readonly params: Params };
    }
  | {
      readonly type: "renameNode";
      readonly payload: { readonly node: string; readonly name: string };
    }
  | { readonly type: "setMode"; readonly payload: { readonly mode: LayoutMode } }
  | { readonly type: "autoLayout"; readonly payload: { readonly viewport: Size } }
  | { readonly type: "select"; readonly payload: { readonly ids: readonly string[] } }
  | {
      readonly type: "inspect";
      readonly payload: { readonly node: string | null; readonly step?: number };
    }
  | {
      readonly type: "loadDataset";
      readonly payload: {
        readonly name: string;
        readonly path: string;
        readonly columns: readonly ColumnDef[];
        readonly rows: number;
      };
    }
  | {
      readonly type: "setPanel";
      readonly payload: { readonly panel: PanelKey; readonly open?: boolean; readonly side?: Side };
    }
  | { readonly type: "setView"; readonly payload: Partial<View> }
  | { readonly type: "setOptions"; readonly payload: Partial<Options> };

export type CommandType = Command["type"];

export type CommandResult = { readonly ok: true } | { readonly ok: false; readonly reason: string };

export interface ReduceOutcome {
  readonly state: EtlState;
  readonly result: CommandResult;
}

/**
 * Contenuto di un punto di cronologia. Nel prototipo (riga 4337):
 * `{ cards, linksArr, comboCounter, outCounter, uidCounter }`. Qui: il
 * grafo (cards + links; i contatori di nomi "Combined Box N"/"Output N"
 * sono derivati dal grafo in etl-core), i contatori di id e nomi, e la
 * modalità (aggiunta: vedi README, "Cronologia").
 */
export interface HistoryEntry {
  readonly graph: Graph;
  readonly counters: Pick<Counters, "uid" | "ds">;
  readonly mode: LayoutMode;
}

/** Voce del registro delle attività. */
export interface LogEntry {
  readonly id: number;
  /** Istante in millisecondi (Date.now o l'orologio iniettato). */
  readonly time: number;
  readonly type: string;
  readonly payload: unknown;
  readonly result: CommandResult;
  /**
   * Solo per le voci che raggruppano più comandi consecutivi (Fase 3.1):
   * istante dell'ultimo comando unito (`time` resta quello del primo) e
   * numero di comandi uniti.
   */
  readonly until?: number;
  readonly count?: number;
}
```

