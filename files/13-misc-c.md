# 13-misc-c.md

File in questo blocco:

- `src/etl-store/store.ts`
- `src/etl-store/types.ts`

---

### `src/etl-store/store.ts`

355 righe

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

/** Ripristina un punto di cronologia; selezione e inspector perdono i nodi spariti. */
function restore(state: EtlState, e: HistoryEntry): EtlState {
  const exists = (id: string): boolean => !!e.graph.cards[id];
  return {
    ...state,
    graph: e.graph,
    mode: e.mode,
    counters: { ...state.counters, uid: e.counters.uid, ds: e.counters.ds },
    selection: state.selection.filter(exists),
    inspector:
      state.inspector.nodeId && !exists(state.inspector.nodeId)
        ? { nodeId: null, step: 0 }
        : state.inspector,
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
  const log: LogEntry[] = [];
  const listeners = new Set<Listener>();
  const routes = createRoutesCache();

  const emit = (): void => {
    for (const l of [...listeners]) l();
  };
  const record = (type: string, payload: unknown, result: CommandResult): void => {
    log.push({ id: log.length + 1, time: now(), type, payload: plain(payload), result });
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
      const before = state;
      const { state: next, result } = reduce(state, command);
      record(command.type, command.payload, result);
      if (!result.ok) return result;
      if (HISTORY_COMMANDS.has(command.type) && !sameEntry(before, next)) pushStep(before);
      setState(next);
      return result;
    },

    /** Prototipo, righe 4392-4397 (`undo`). */
    undo() {
      if (gesture) dropGesture();
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
      past = [];
      future = [];
      setState(next);
    },
  };
  return store;
}
```

### `src/etl-store/types.ts`

189 righe

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
}
```

