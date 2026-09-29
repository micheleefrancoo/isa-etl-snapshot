# 13-misc-b.md

File in questo blocco:

- `src/etl-store/__tests__/store.test.ts`
- `src/etl-store/derived.ts`
- `src/etl-store/index.ts`
- `src/etl-store/persistence.ts`
- `src/etl-store/react.ts`
- `src/etl-store/reduce.ts`
- `src/etl-store/serialize.ts`
- `src/etl-store/state.ts`

---

### `src/etl-store/__tests__/store.test.ts`

230 righe

```ts
import { describe, expect, it } from "vitest";
import { outputOf } from "../../etl-core";
import type { Card, FilterParams, Params } from "../../etl-core";
import { HISTORY_LIMIT, createEtlStore, initialState } from "..";
import type { EtlStore } from "..";
import { COLUMNS, withDatasetAndFilter } from "./helpers";

function storeWith(): { store: EtlStore; ds: string; filter: string } {
  const { state, ds, filter } = withDatasetAndFilter();
  let t = 0;
  return { store: createEtlStore({ initial: state, now: () => ++t }), ds, filter };
}

function pos(store: EtlStore, id: string): { x: number; y: number } {
  const c = store.getState().graph.cards[id] as Card;
  return { x: c.x, y: c.y };
}

describe("cronologia", () => {
  it("un trascinamento di 30 aggiornamenti crea un solo passo; annulla e ripristina", () => {
    const { store, ds } = storeWith();
    const start = pos(store, ds);
    expect(store.beginGesture({ ids: [ds] })).toEqual({ ok: true });
    for (let i = 1; i <= 30; i++) store.updateGesture({ dx: i * 10, dy: i * 4 });
    expect(store.historySize().past).toBe(0);
    expect(store.commitGesture()).toEqual({ ok: true });
    expect(store.historySize()).toEqual({ past: 1, future: 0 });
    const end = pos(store, ds);
    expect(end).not.toEqual(start);
    store.undo();
    expect(pos(store, ds)).toEqual(start);
    store.redo();
    expect(pos(store, ds)).toEqual(end);
  });

  it("annullare un gesto riporta lo stato di partenza senza passi", () => {
    const { store, ds } = storeWith();
    const before = store.getState().graph;
    store.beginGesture({ ids: [ds] });
    store.updateGesture({ dx: 300, dy: 0 });
    store.cancelGesture();
    expect(store.getState().graph).toBe(before);
    expect(store.historySize().past).toBe(0);
  });

  it(`limite di ${HISTORY_LIMIT} passi (prototipo HIST_MAX)`, () => {
    const { store, ds } = storeWith();
    for (let i = 0; i < HISTORY_LIMIT + 10; i++)
      store.dispatch({ type: "moveNodes", payload: { ids: [ds], dx: 2, dy: 0 } });
    expect(HISTORY_LIMIT).toBe(50);
    expect(store.historySize().past).toBe(50);
    let n = 0;
    while (store.canUndo()) {
      store.undo();
      n++;
    }
    expect(n).toBe(50);
  });

  it("un comando dopo un annullamento cancella i passi da ripristinare", () => {
    const { store, ds } = storeWith();
    store.dispatch({ type: "moveNodes", payload: { ids: [ds], dx: 2, dy: 0 } });
    store.dispatch({ type: "moveNodes", payload: { ids: [ds], dx: 2, dy: 0 } });
    store.undo();
    expect(store.canRedo()).toBe(true);
    store.dispatch({ type: "moveNodes", payload: { ids: [ds], dx: 0, dy: 2 } });
    expect(store.canRedo()).toBe(false);
  });

  it("selezione, inspector, vista, pannelli, opzioni e libreria non creano passi", () => {
    const { store, ds } = storeWith();
    store.dispatch({ type: "select", payload: { ids: [ds] } });
    store.dispatch({ type: "inspect", payload: { node: ds } });
    store.dispatch({ type: "setView", payload: { x: 40, zoom: 1.5 } });
    store.dispatch({ type: "setPanel", payload: { panel: "insp", open: true } });
    store.dispatch({ type: "setOptions", payload: { flowOnlyIfValid: true } });
    store.dispatch({
      type: "loadDataset",
      payload: { name: "b", path: "b.csv", columns: COLUMNS, rows: 1 },
    });
    expect(store.historySize()).toEqual({ past: 0, future: 0 });
  });

  it("un comando rifiutato o senza effetto non crea passi", () => {
    const { store } = storeWith();
    store.dispatch({ type: "deleteNodes", payload: { ids: ["nessuno"] } });
    store.dispatch({ type: "setMode", payload: { mode: "free" } });
    expect(store.historySize().past).toBe(0);
  });

  it("annullare ripristina anche la modalità e i contatori", () => {
    const { store } = storeWith();
    const before = store.getState();
    store.dispatch({ type: "setMode", payload: { mode: "grid" } });
    store.dispatch({ type: "addNode", payload: { component: "sort", point: { x: 900, y: 900 } } });
    store.undo();
    store.undo();
    expect(store.getState().mode).toBe("free");
    expect(store.getState().graph).toBe(before.graph);
    expect(store.getState().counters).toEqual(before.counters);
  });

  it("dopo un annullamento la selezione perde i nodi spariti", () => {
    const { store } = storeWith();
    store.dispatch({ type: "addNode", payload: { component: "sort", point: { x: 900, y: 900 } } });
    const id = Object.keys(store.getState().graph.cards).pop() as string;
    store.dispatch({ type: "select", payload: { ids: [id] } });
    store.dispatch({ type: "inspect", payload: { node: id } });
    store.undo();
    expect(store.getState().selection).toEqual([]);
    expect(store.getState().inspector.nodeId).toBeNull();
  });
});

describe("registro delle attività", () => {
  it("i comandi rifiutati sono registrati con il motivo", () => {
    const { store, ds } = storeWith();
    store.dispatch({ type: "connect", payload: { from: ds, to: ds } });
    const last = store.getLog().at(-1);
    expect(last).toMatchObject({
      type: "connect",
      payload: { from: ds, to: ds },
      result: { ok: false, reason: "Un nodo non si collega a sé stesso" },
    });
  });

  it("un gesto produce una sola voce, con posizione iniziale e finale", () => {
    const { store, ds } = storeWith();
    const start = pos(store, ds);
    const n0 = store.getLog().length;
    store.beginGesture({ ids: [ds] });
    for (let i = 1; i <= 30; i++) store.updateGesture({ dx: i * 10, dy: 0 });
    store.commitGesture();
    expect(store.getLog().length).toBe(n0 + 1);
    const entry = store.getLog().at(-1);
    expect(entry?.type).toBe("gesture");
    expect(entry?.payload).toMatchObject({
      kind: "move",
      ids: [ds],
      from: { [ds]: start },
      to: { [ds]: pos(store, ds) },
    });
  });

  it("ogni voce ha id crescente, istante, tipo, payload e risultato; l'esportazione è JSON valido", () => {
    const { store, ds, filter } = storeWith();
    store.dispatch({ type: "connect", payload: { from: ds, to: filter } });
    store.undo();
    store.redo();
    const parsed = JSON.parse(store.exportLog()) as { id: number; time: number; type: string }[];
    expect(parsed.map((e) => e.type)).toEqual(["connect", "undo", "redo"]);
    expect(parsed.map((e) => e.id)).toEqual([1, 2, 3]);
    expect(parsed.every((e) => typeof e.time === "number")).toBe(true);
  });

  it("caricando un CSV il registro contiene solo i metadati, non il contenuto del file", () => {
    const store = createEtlStore();
    const csv = "cliente;importo\nSEGRETO-1;10\nSEGRETO-2;20\n";
    expect(store.loadCsv(csv, "clienti.csv")).toEqual({ ok: true });
    const text = store.exportLog();
    const entry = store.getLog()[0];
    expect(entry?.payload).toMatchObject({ name: "clienti", path: "clienti.csv", rows: 2 });
    expect(Object.keys(entry?.payload as object).sort()).toEqual([
      "columns",
      "name",
      "path",
      "rows",
    ]);
    expect(text).not.toContain("cliente;importo\n");
  });
});

describe("scenario completo", () => {
  it("CSV -> dataset -> filtro -> parametri -> join con un secondo dataset; annulla tutto e ripristina tutto", () => {
    const store = createEtlStore();
    expect(store.loadCsv("id,regione,importo\n1,Nord,10\n2,Sud,20\n", "vendite.csv")).toEqual({
      ok: true,
    });
    expect(store.loadCsv("id,cliente\n1,Rossi\n2,Bianchi\n", "clienti.csv")).toEqual({ ok: true });
    const d = (cmd: Parameters<EtlStore["dispatch"]>[0]): void => {
      expect(store.dispatch(cmd), cmd.type).toEqual({ ok: true });
    };
    const newest = (): string => Object.keys(store.getState().graph.cards).at(-1) as string;

    d({
      type: "addNode",
      payload: { component: "dataset", libraryId: "lib-1", point: { x: 150, y: 300 } },
    });
    const vendite = newest();
    d({ type: "addNode", payload: { component: "filter", point: { x: 500, y: 300 } } });
    const filter = newest();
    d({ type: "connect", payload: { from: vendite, to: filter } });
    const params: FilterParams = {
      conditions: [
        { column: "regione", op: "=", mode: "list", values: ["Nord"], text: "", sep: "," },
      ],
    };
    d({
      type: "setParams",
      payload: { node: filter, index: 0, params: params as unknown as Params },
    });
    d({ type: "addNode", payload: { component: "join", point: { x: 1000, y: 400 } } });
    const join = newest();
    d({
      type: "connect",
      payload: { from: outputOf(store.getState().graph, filter) as string, to: join },
    });
    d({
      type: "addNode",
      payload: { component: "dataset", libraryId: "lib-2", point: { x: 600, y: 800 } },
    });
    const clienti = newest();
    d({ type: "connect", payload: { from: clienti, to: join } });

    const final = store.getState();
    expect(final.graph.links.filter((l) => l.to === join)).toHaveLength(2);
    const steps = store.historySize().past;
    expect(steps).toBe(8);

    while (store.canUndo()) store.undo();
    expect(store.getState().graph).toEqual(initialState().graph);
    expect(store.getState().library).toHaveLength(2);

    while (store.canRedo()) store.redo();
    expect(store.getState().graph).toEqual(final.graph);
    expect(store.getState().counters).toEqual(final.counters);
    expect(store.historySize()).toEqual({ past: steps, future: 0 });
  });
});
```

### `src/etl-store/derived.ts`

52 righe

```ts
/**
 * Valori derivati dallo stato: calcolati, mai memorizzati nella cronologia
 * né salvati.
 */
import { nodeState, schemaOf } from "../etl-core";
import type { ColumnDef, Graph, Link } from "../etl-core";
import { settleLinks } from "../etl-layout";
import type { LinkRoutes } from "../etl-layout";
import type { EtlState } from "./types";

/** Stato di ogni nodo (prototipo `nodeState`, righe 1447-1459): `null` = pronto, altrimenti il motivo. */
export function nodeStates(graph: Graph): Record<string, string | null> {
  const out: Record<string, string | null> = {};
  for (const id of Object.keys(graph.cards)) out[id] = nodeState(graph, id);
  return out;
}

/** Schema di ogni nodo (prototipo `schemaOf`, righe 2415-2430). */
export function schemas(graph: Graph): Record<string, ColumnDef[] | null> {
  const out: Record<string, ColumnDef[] | null> = {};
  for (const id of Object.keys(graph.cards)) out[id] = schemaOf(graph, id);
  return out;
}

/**
 * Un collegamento trasporta dati? Prototipo `linkLive` (righe 1474-1477):
 * sempre, salvo con "flusso solo se valido", che richiede entrambi i capi
 * pronti.
 */
export function linkLive(state: EtlState, l: Link): boolean {
  if (!state.options.flowOnlyIfValid) return true;
  return !nodeState(state.graph, l.from) && !nodeState(state.graph, l.to);
}

/**
 * Percorsi dei cavi con memoria: ogni calcolo parte dai percorsi
 * precedenti (stabilità di etl-layout) e si ricalcola solo quando cambiano
 * il grafo o il limite di snodi.
 */
export function createRoutesCache(): (state: EtlState) => LinkRoutes {
  let lastGraph: Graph | null = null;
  let lastBends = -1;
  let routes: LinkRoutes = {};
  return (state) => {
    if (state.graph === lastGraph && state.options.maxBends === lastBends) return routes;
    routes = settleLinks(state.graph, routes, { maxBends: state.options.maxBends }).routes;
    lastGraph = state.graph;
    lastBends = state.options.maxBends;
    return routes;
  };
}
```

### `src/etl-store/index.ts`

29 righe

```ts
/**
 * etl-store — Fase 3: stato dell'applicazione, cronologia, registro delle
 * attività e salvataggio. Il nucleo esportato qui non importa React né usa
 * le API del browser; il collegamento a React è in `./react`, il
 * salvataggio nel browser in `./persistence`.
 */
export * from "./types";
export {
  initialState,
  DEFAULT_PANELS,
  DEFAULT_VIEW,
  DEFAULT_OPTIONS,
  ZOOM_MIN,
  ZOOM_MAX,
} from "./state";
export { reduce, dropAt } from "./reduce";
export { createEtlStore, HISTORY_LIMIT, HISTORY_COMMANDS } from "./store";
export type {
  EtlStore,
  StoreOptions,
  GestureStart,
  GestureUpdate,
  GestureEnd,
  Listener,
} from "./store";
export { nodeStates, schemas, linkLive, createRoutesCache } from "./derived";
export { SAVE_VERSION, toSaved, fromSaved, parseSaved } from "./serialize";
export type { SavedState } from "./serialize";
```

### `src/etl-store/persistence.ts`

117 righe

```ts
/**
 * Salvataggio in localStorage. È l'UNICO modulo di etl-store che tocca
 * localStorage, sempre dopo aver verificato che esista (rendering lato
 * server: in Node non c'è).
 *
 * - Chiave nuova, per soluzione: `isa.etl.v2.<solutionId>`. I dati delle
 *   chiavi precedenti non si leggono né si cancellano.
 * - Scrittura differita di 400 ms dopo l'ultima modifica.
 * - Caricamento con validazione (serialize.ts): un dato non valido o di
 *   versione sconosciuta viene ignorato, partendo da un canvas vuoto.
 */
import { initialState } from "./state";
import { parseSaved, toSaved } from "./serialize";
import type { EtlStore } from "./store";
import type { EtlState } from "./types";

export const STORAGE_PREFIX = "isa.etl.v2.";
export const SAVE_DELAY_MS = 400;

export function storageKey(solutionId: string): string {
  return STORAGE_PREFIX + solutionId;
}

/** Il sottoinsieme di Storage che serve (iniettabile nei test). */
export interface StorageLike {
  getItem(key: string): string | null;
  setItem(key: string, value: string): void;
}

/** localStorage se disponibile (browser), altrimenti `null` (server, Node, accesso negato). */
export function browserStorage(): StorageLike | null {
  try {
    if (typeof globalThis === "undefined" || !("localStorage" in globalThis)) return null;
    const ls = (globalThis as { localStorage?: StorageLike }).localStorage;
    return ls ?? null;
  } catch {
    return null;
  }
}

/** Stato salvato per la soluzione, o un canvas vuoto. Mai eccezioni. */
export function loadState(
  solutionId: string,
  storage: StorageLike | null = browserStorage(),
): EtlState {
  if (!storage) return initialState();
  try {
    return parseSaved(storage.getItem(storageKey(solutionId))) ?? initialState();
  } catch {
    return initialState();
  }
}

export function saveState(
  solutionId: string,
  state: EtlState,
  storage: StorageLike | null = browserStorage(),
): boolean {
  if (!storage) return false;
  try {
    storage.setItem(storageKey(solutionId), JSON.stringify(toSaved(state)));
    return true;
  } catch {
    return false;
  }
}

export interface PersistenceHandle {
  /** Scrive subito l'eventuale salvataggio in attesa. */
  flush(): void;
  /** Smette di osservare lo store (scrive prima quanto in attesa). */
  stop(): void;
}

/**
 * Collega uno store al salvataggio: dopo ogni modifica di ciò che si
 * salva, scrive dopo `SAVE_DELAY_MS` dall'ultima.
 */
export function persist(
  store: EtlStore,
  solutionId: string,
  storage: StorageLike | null = browserStorage(),
  delay: number = SAVE_DELAY_MS,
): PersistenceHandle {
  let timer: ReturnType<typeof setTimeout> | null = null;
  let last = store.getState();
  const write = (): void => {
    timer = null;
    saveState(solutionId, store.getState(), storage);
  };
  const unsubscribe = store.subscribe(() => {
    const s = store.getState();
    const changed =
      s.graph !== last.graph ||
      s.mode !== last.mode ||
      s.library !== last.library ||
      s.panels !== last.panels ||
      s.options !== last.options;
    last = s;
    if (!changed || !storage || store.isGesturing()) return;
    if (timer) clearTimeout(timer);
    timer = setTimeout(write, delay);
  });
  return {
    flush() {
      if (timer) {
        clearTimeout(timer);
        write();
      }
    },
    stop() {
      unsubscribe();
      this.flush();
    },
  };
}
```

### `src/etl-store/react.ts`

60 righe

```ts
/**
 * Collegamento a React: l'UNICO modulo di etl-store che importa React.
 * Usa `useSyncExternalStore` (nessuna nuova dipendenza).
 */
import {
  createContext,
  createElement,
  useContext,
  useEffect,
  useState,
  useSyncExternalStore,
} from "react";
import type { ReactNode } from "react";
import { persist, loadState } from "./persistence";
import { createEtlStore } from "./store";
import type { EtlStore } from "./store";
import type { EtlState } from "./types";

const EtlStoreContext = createContext<EtlStore | null>(null);

export function EtlStoreProvider(props: { store: EtlStore; children?: ReactNode }): ReactNode {
  return createElement(EtlStoreContext.Provider, { value: props.store }, props.children);
}

/** Lo store del provider più vicino. */
export function useEtlStoreInstance(): EtlStore {
  const store = useContext(EtlStoreContext);
  if (!store) throw new Error("useEtlStore va usato dentro <EtlStoreProvider>");
  return store;
}

/**
 * Una parte dello stato, aggiornata a ogni modifica. Il selettore deve
 * restituire valori stabili (parti dello stato o primitivi), non oggetti
 * nuovi a ogni chiamata.
 */
export function useEtlState<T>(selector: (state: EtlState) => T, store?: EtlStore): T {
  const ctx = useContext(EtlStoreContext);
  const s = store ?? ctx;
  if (!s) throw new Error("useEtlState va usato dentro <EtlStoreProvider> o con uno store");
  return useSyncExternalStore(
    s.subscribe,
    () => selector(s.getState()),
    () => selector(s.getState()),
  );
}

/**
 * Crea uno store per la soluzione `solutionId`, caricato da localStorage
 * (lato server: canvas vuoto) e salvato con scrittura differita.
 */
export function usePersistentEtlStore(solutionId: string): EtlStore {
  const [store] = useState(() => createEtlStore({ initial: loadState(solutionId) }));
  useEffect(() => {
    const handle = persist(store, solutionId);
    return () => handle.stop();
  }, [store, solutionId]);
  return store;
}
```

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

