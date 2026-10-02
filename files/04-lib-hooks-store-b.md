# 04-lib-hooks-store-b.md

File in questo blocco:

- `src/lib/etl-workflow.tsx`
- `src/lib/modules.ts`
- `src/lib/solutions-store.tsx`
- `src/lib/theme.test.tsx`
- `src/lib/theme.tsx`
- `src/lib/utils.ts`

---

### `src/lib/etl-workflow.tsx`

476 righe

```tsx
import { useCallback, useSyncExternalStore } from "react";
import { nodeDef } from "./etl-catalog";
import type { NodeSize } from "./etl-node-size";
import { NODE_H, NODE_W, ROUTE_GAP, estimateNodeSize } from "./etl-node-size";

export type NodeStatus = "ready" | "running" | "succeeded" | "error";

export type EtlNode = {
  id: string;
  type: string;
  title: string;
  x: number;
  y: number;
  config: Record<string, string>;
  status: NodeStatus;
  /** Nodi "transform" combinati insieme condividono lo stesso groupId. Opzionale e retrocompatibile. */
  groupId?: string | undefined;
};

export type EtlEdge = {
  id: string;
  fromNode: string;
  fromPort: string;
  toNode: string;
  toPort: string;
};

export type LayoutMode = "auto" | "manual";

export type EtlWorkflow = {
  nodes: EtlNode[];
  edges: EtlEdge[];
  layout: LayoutMode;
};

type Entry = {
  present: EtlWorkflow;
  past: EtlWorkflow[];
  future: EtlWorkflow[];
  savedAt: number;
};

const EMPTY: EtlWorkflow = { nodes: [], edges: [], layout: "auto" };
const EMPTY_ENTRY: Entry = { present: EMPTY, past: [], future: [], savedAt: 0 };

/** Origine della griglia di auto-layout. */
export const LAYOUT_ORIGIN = 32;

// Stato client-side dell'authoring del workflow, per soluzione.
// Nessuna esecuzione reale: il motore arriverà nella fase dedicata.
const store = new Map<string, Entry>();
const listeners = new Set<() => void>();
const emit = () => listeners.forEach((l) => l());

const uid = (p: string) => `${p}-${Math.random().toString(36).slice(2, 8)}`;

const node = (
  type: string,
  title: string,
  x: number,
  y: number,
  config: Record<string, string>,
): EtlNode => ({ id: uid("node"), type, title, x, y, config, status: "ready" });

/** Un gruppo con un solo membro non è più un gruppo: gli toglie il groupId. */
function dissolveSingletonGroups(nodes: EtlNode[]): EtlNode[] {
  const counts = new Map<string, number>();
  for (const n of nodes) {
    if (n.groupId) counts.set(n.groupId, (counts.get(n.groupId) ?? 0) + 1);
  }
  return nodes.map((n) =>
    n.groupId && (counts.get(n.groupId) ?? 0) < 2 ? { ...n, groupId: undefined } : n,
  );
}

/**
 * Disposizione automatica a colonne, seguendo la direzione del flusso
 * dati. Passo di griglia ADATTIVO (PARTE C del redesign): usa la
 * dimensione REALE di ogni card (stessa `estimateNodeSize` del
 * rendering, condivisa via lib/etl-node-size.ts per non disallineare le
 * due logiche) invece di un passo fisso — che con card di dimensioni
 * molto diverse (una card sources può essere più larga o più stretta di
 * una transform quadrata) causava sovrapposizioni o spazi vuoti
 * eccessivi.
 *
 * `estimateNodeSize` qui non riceve i `display` settings dell'utente
 * (stato locale del componente canvas, non del workflow persistito):
 * usa la sua baseline di default, coerente in ogni run di autoLayout
 * indipendentemente da chi/quando lo invoca.
 *
 * Un nodo appena aggiunto e ancora privo di collegamenti ha profondità
 * 0: finisce quindi nella prima colonna, all'ultima riga (comportamento
 * voluto, PARTE B — niente eccezioni per rispettare la posizione di
 * drop/doppio click quando il layout è "automatico").
 */
export function autoLayout(workflow: EtlWorkflow): EtlNode[] {
  const depth = new Map<string, number>();
  const order = pipelineOrder(workflow);
  for (const n of order) {
    const parents = workflow.edges.filter((e) => e.toNode === n.id);
    const d = parents.length
      ? Math.max(...parents.map((e) => (depth.get(e.fromNode) ?? 0) + 1))
      : 0;
    depth.set(n.id, d);
  }

  const sizes = new Map<string, NodeSize>();
  for (const n of workflow.nodes) {
    sizes.set(n.id, estimateNodeSize(n, workflow));
  }

  const columns = new Map<number, EtlNode[]>();
  for (const n of order) {
    const d = depth.get(n.id) ?? 0;
    const list = columns.get(d);
    if (list) {
      list.push(n);
    } else {
      columns.set(d, [n]);
    }
  }

  const maxDepth = Math.max(0, ...Array.from(columns.keys()));

  /*
   * Larghezza di ogni colonna = card più larga che contiene, così le
   * card della colonna successiva non si sovrappongono mai a quelle
   * larghe di questa. Righe allineate in ALTO nella colonna (non
   * centrate): impilate una sotto l'altra con l'altezza reale di
   * ciascuna, non un passo fisso.
   */
  const columnX: number[] = [];
  let x = LAYOUT_ORIGIN;
  for (let d = 0; d <= maxDepth; d++) {
    columnX.push(x);
    const widest = Math.max(
      NODE_W,
      ...(columns.get(d) ?? []).map((n) => sizes.get(n.id)?.width ?? NODE_W),
    );
    x += widest + ROUTE_GAP;
  }

  const positions = new Map<string, { x: number; y: number }>();
  for (let d = 0; d <= maxDepth; d++) {
    let y = LAYOUT_ORIGIN;
    for (const n of columns.get(d) ?? []) {
      positions.set(n.id, { x: columnX[d]!, y });
      y += (sizes.get(n.id)?.height ?? NODE_H) + ROUTE_GAP;
    }
  }

  return workflow.nodes.map((n) => ({ ...n, ...(positions.get(n.id) ?? {}) }));
}

const arranged = (w: EtlWorkflow): EtlWorkflow =>
  w.layout === "auto" ? { ...w, nodes: autoLayout(w) } : w;

const storageKey = (solutionId: string) => `isa.etl.workflow.${solutionId}`;

function load(solutionId: string): EtlWorkflow | null {
  if (typeof window === "undefined") return null;
  try {
    const raw = window.localStorage.getItem(storageKey(solutionId));
    if (!raw) return null;
    const parsed = JSON.parse(raw) as Partial<EtlWorkflow>;
    if (!Array.isArray(parsed.nodes) || !Array.isArray(parsed.edges)) return null;
    return {
      nodes: parsed.nodes,
      edges: parsed.edges,
      layout: parsed.layout === "manual" ? "manual" : "auto",
    };
  } catch {
    return null;
  }
}

function persist(solutionId: string, workflow: EtlWorkflow) {
  if (typeof window === "undefined") return;
  try {
    window.localStorage.setItem(storageKey(solutionId), JSON.stringify(workflow));
  } catch {
    /* storage non disponibile: lo stato resta in memoria */
  }
}

function entryOf(solutionId: string): Entry {
  const existing = store.get(solutionId);
  if (existing) return existing;
  const restored = load(solutionId);
  const created: Entry = {
    present: restored ? arranged(restored) : EMPTY,
    past: [],
    future: [],
    savedAt: Date.now(),
  };
  store.set(solutionId, created);
  return created;
}

const commit = (solutionId: string, next: EtlWorkflow, history = true) => {
  const entry = entryOf(solutionId);
  const present = arranged(next);
  store.set(solutionId, {
    present,
    past: history ? [...entry.past.slice(-40), entry.present] : entry.past,
    future: history ? [] : entry.future,
    savedAt: Date.now(),
  });
  persist(solutionId, present);
  emit();
};

export function useEtlWorkflow(solutionId: string) {
  const entry = useSyncExternalStore(
    (cb) => {
      listeners.add(cb);
      return () => listeners.delete(cb);
    },
    () => entryOf(solutionId),
    () => EMPTY_ENTRY,
  );
  const workflow = entry.present;

  const addNode = useCallback(
    (
      type: string,
      x: number,
      y: number,
      config?: Record<string, string>,
      title?: string,
      // Bug 1.3: un nodo aggiunto via drag-and-drop esplicito è
      // un'intenzione di posizionamento manuale, esattamente come
      // moveNode — se il workflow resta in layout "auto", `commit()`
      // (via `arranged()`) ricalcolerebbe subito tutte le posizioni con
      // autoLayout(), facendo "sparire" la card dal punto di rilascio.
      // Il doppio click dalla palette NON passa questo flag: per quel
      // percorso lo schema a colonne dell'auto-layout vince sempre,
      // comportamento invariato.
      manual = false,
    ) => {
      const def = nodeDef(type);
      if (!def) return undefined;
      const base: Record<string, string> = {};
      for (const f of def.fields) base[f.key] = f.defaultValue ?? "";
      const created = node(type, title ?? def.label, x, y, { ...base, ...config });
      const current = entryOf(solutionId).present;
      commit(solutionId, {
        ...current,
        ...(manual ? { layout: "manual" as const } : {}),
        nodes: [...current.nodes, created],
      });
      return created.id;
    },
    [solutionId],
  );

  const moveNode = useCallback(
    (id: string, x: number, y: number, history = false) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          // trascinare un nodo passa automaticamente al posizionamento manuale
          layout: "manual",
          nodes: current.nodes.map((n) => (n.id === id ? { ...n, x, y } : n)),
        },
        history,
      );
    },
    [solutionId],
  );

  const setLayout = useCallback(
    (layout: LayoutMode) => {
      const current = entryOf(solutionId).present;
      commit(solutionId, { ...current, layout });
    },
    [solutionId],
  );

  const updateNode = useCallback(
    (
      id: string,
      patch: { title?: string; config?: Record<string, string>; status?: NodeStatus },
      history = true,
    ) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          nodes: current.nodes.map((n) =>
            n.id === id
              ? {
                  ...n,
                  ...(patch.title !== undefined ? { title: patch.title } : {}),
                  ...(patch.status !== undefined ? { status: patch.status } : {}),
                  config: { ...n.config, ...patch.config },
                }
              : n,
          ),
        },
        history,
      );
    },
    [solutionId],
  );

  const removeNode = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      const nodes = dissolveSingletonGroups(current.nodes.filter((n) => n.id !== id));
      commit(solutionId, {
        ...current,
        nodes,
        edges: current.edges.filter((e) => e.fromNode !== id && e.toNode !== id),
      });
    },
    [solutionId],
  );

  const groupNodes = useCallback(
    (ids: string[]) => {
      if (ids.length < 2) return;
      const current = entryOf(solutionId).present;
      // Riunisce anche i membri di eventuali gruppi già esistenti a cui
      // appartengono gli id passati, così due gruppi che vengono uniti
      // finiscono davvero sotto un unico groupId, senza lasciare membri
      // orfani nel gruppo vecchio.
      const involvedGroupIds = new Set(
        current.nodes.filter((n) => ids.includes(n.id) && n.groupId).map((n) => n.groupId!),
      );
      const fullIds = new Set(ids);
      for (const n of current.nodes) {
        if (n.groupId && involvedGroupIds.has(n.groupId)) fullIds.add(n.id);
      }
      const groupId = involvedGroupIds.values().next().value ?? uid("group");
      commit(solutionId, {
        ...current,
        nodes: current.nodes.map((n) => (fullIds.has(n.id) ? { ...n, groupId } : n)),
      });
    },
    [solutionId],
  );

  const ungroupNode = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      const node = current.nodes.find((n) => n.id === id);
      if (!node?.groupId) return;
      const cleared = current.nodes.map((n) => (n.id === id ? { ...n, groupId: undefined } : n));
      commit(solutionId, { ...current, nodes: dissolveSingletonGroups(cleared) });
    },
    [solutionId],
  );

  const connect = useCallback(
    (fromNode: string, fromPort: string, toNode: string, toPort: string) => {
      if (fromNode === toNode) return;
      const current = entryOf(solutionId).present;
      if (
        current.edges.some(
          (e) =>
            e.fromNode === fromNode &&
            e.fromPort === fromPort &&
            e.toNode === toNode &&
            e.toPort === toPort,
        )
      )
        return;
      // una porta di input accetta una sola connessione
      const edges = current.edges.filter((e) => !(e.toNode === toNode && e.toPort === toPort));
      commit(solutionId, {
        ...current,
        edges: [...edges, { id: uid("edge"), fromNode, fromPort, toNode, toPort }],
      });
    },
    [solutionId],
  );

  const removeEdge = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      commit(solutionId, { ...current, edges: current.edges.filter((e) => e.id !== id) });
    },
    [solutionId],
  );

  const setStatuses = useCallback(
    (updates: Record<string, NodeStatus>) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          nodes: current.nodes.map((n) => (updates[n.id] ? { ...n, status: updates[n.id]! } : n)),
        },
        false,
      );
    },
    [solutionId],
  );

  const undo = useCallback(() => {
    const e = entryOf(solutionId);
    const previous = e.past[e.past.length - 1];
    if (!previous) return;
    store.set(solutionId, {
      present: previous,
      past: e.past.slice(0, -1),
      future: [e.present, ...e.future].slice(0, 40),
      savedAt: Date.now(),
    });
    persist(solutionId, previous);
    emit();
  }, [solutionId]);

  const redo = useCallback(() => {
    const e = entryOf(solutionId);
    const next = e.future[0];
    if (!next) return;
    store.set(solutionId, {
      present: next,
      past: [...e.past, e.present],
      future: e.future.slice(1),
      savedAt: Date.now(),
    });
    persist(solutionId, next);
    emit();
  }, [solutionId]);

  return {
    workflow,
    canUndo: entry.past.length > 0,
    canRedo: entry.future.length > 0,
    addNode,
    moveNode,
    setLayout,
    updateNode,
    removeNode,
    groupNodes,
    ungroupNode,
    connect,
    removeEdge,
    setStatuses,
    undo,
    redo,
  };
}

/** Ordine topologico di esecuzione. */
export function pipelineOrder(workflow: EtlWorkflow): EtlNode[] {
  const indeg = new Map(workflow.nodes.map((n) => [n.id, 0]));
  for (const e of workflow.edges) indeg.set(e.toNode, (indeg.get(e.toNode) ?? 0) + 1);
  const queue = workflow.nodes.filter((n) => (indeg.get(n.id) ?? 0) === 0);
  const out: EtlNode[] = [];
  const seen = new Set<string>();
  while (queue.length) {
    const n = queue.shift()!;
    if (seen.has(n.id)) continue;
    seen.add(n.id);
    out.push(n);
    for (const e of workflow.edges.filter((x) => x.fromNode === n.id)) {
      const left = (indeg.get(e.toNode) ?? 0) - 1;
      indeg.set(e.toNode, left);
      if (left <= 0) {
        const target = workflow.nodes.find((x) => x.id === e.toNode);
        if (target) queue.push(target);
      }
    }
  }
  for (const n of workflow.nodes) if (!seen.has(n.id)) out.push(n);
  return out;
}
```

### `src/lib/modules.ts`

38 righe

```ts
import { ArrowLeftRight, LayoutDashboard, Sigma } from "lucide-react";
import type { ModuleKey } from "./solutions-store";

export type ModuleMeta = {
  key: ModuleKey;
  label: string;
  description: string;
  Icon: typeof ArrowLeftRight;
  to:
    | "/solutions/$solutionId/etl"
    | "/solutions/$solutionId/model"
    | "/solutions/$solutionId/dashboard";
};

export const MODULES: ModuleMeta[] = [
  {
    key: "etl",
    label: "ETL",
    description: "Data pipeline & ingestion",
    Icon: ArrowLeftRight,
    to: "/solutions/$solutionId/etl",
  },
  {
    key: "model",
    label: "Model & Algorithm",
    description: "Formule matematiche, logica e parametri di calcolo",
    Icon: Sigma,
    to: "/solutions/$solutionId/model",
  },
  {
    key: "dashboard",
    label: "Dashboard",
    description: "Grafici, indicatori e reportistica",
    Icon: LayoutDashboard,
    to: "/solutions/$solutionId/dashboard",
  },
];
```

### `src/lib/solutions-store.tsx`

332 righe

```tsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState } from "react";

export type SolutionStatus = "draft" | "live";
export type ChartKind = "bar" | "line";
export type ModuleKey = "etl" | "model" | "dashboard";

export type Parameter = {
  id: string;
  label: string;
  unit: string;
  value: number;
  min: number;
  max: number;
  step: number;
};

export type Solution = {
  id: string;
  name: string;
  description: string;
  status: SolutionStatus;
  version: string;
  chart: ChartKind;
  series: number[];
  updatedAt: string;
  owner: string;
  parameters: Parameter[];
  modules: Partial<Record<ModuleKey, SolutionStatus>>;
  shares: SolutionShare[];
};

export type SharePermission = "view" | "edit";

export type SolutionShare = {
  id: string;
  email: string;
  permission: SharePermission;
};

export type BatchJob = {
  id: string;
  solutionName: string;
  progress: number;
  state: "queued" | "running" | "done";
};

export type ActivityItem = {
  id: string;
  who: string;
  what: string;
  when: string;
};

const p = (
  id: string,
  label: string,
  unit: string,
  value: number,
  min: number,
  max: number,
  step = 1,
): Parameter => ({ id, label, unit, value, min, max, step });

const seed: Solution[] = [];

const initialJobs: BatchJob[] = [];

const initialActivity: ActivityItem[] = [];

type Store = {
  solutions: Solution[];
  jobs: BatchJob[];
  activity: ActivityItem[];
  createSolution: (input: {
    name: string;
    description: string;
    version?: string;
    modules?: ModuleKey[];
  }) => string;
  deleteSolution: (id: string) => void;
  updateSolution: (
    id: string,
    patch: Partial<Pick<Solution, "name" | "version" | "status">>,
  ) => void;
  duplicateSolution: (id: string) => string | undefined;
  addShare: (id: string, email: string, permission: SharePermission) => void;
  updateShare: (id: string, shareId: string, permission: SharePermission) => void;
  removeShare: (id: string, shareId: string) => void;

  updateParameter: (solutionId: string, paramId: string, value: number) => void;
  runJob: (solutionId: string) => void;
};

// Keep a single context instance across hot reloads, otherwise the provider and
// the consumer can end up using two different contexts after an HMR update.
const globalStore = globalThis as unknown as {
  __isaStoreContext?: React.Context<Store | null>;
};
const StoreContext =
  globalStore.__isaStoreContext ??
  (globalStore.__isaStoreContext = createContext<Store | null>(null));

const slug = (s: string) =>
  s
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-|-$/g, "") || `solution-${Date.now()}`;

const randomSeries = () =>
  Array.from({ length: 9 }, (_, i) => Math.round(18 + i * 4 + Math.random() * 22));

const SOLUTIONS_KEY = "isa.solutions";

export function SolutionsProvider({ children }: { children: React.ReactNode }) {
  const [solutions, setSolutions] = useState<Solution[]>(seed);
  const [hydrated, setHydrated] = useState(false);

  // Le soluzioni restano disponibili anche uscendo e rientrando dall'app.
  useEffect(() => {
    try {
      const raw = window.localStorage.getItem(SOLUTIONS_KEY);
      if (raw) {
        const parsed = JSON.parse(raw) as Solution[];
        if (Array.isArray(parsed)) setSolutions(parsed);
      }
    } catch {
      /* storage non disponibile */
    }
    setHydrated(true);
  }, []);

  useEffect(() => {
    if (!hydrated) return;
    try {
      window.localStorage.setItem(SOLUTIONS_KEY, JSON.stringify(solutions));
    } catch {
      /* storage non disponibile */
    }
  }, [solutions, hydrated]);
  const [jobs, setJobs] = useState<BatchJob[]>(initialJobs);
  const [activity, setActivity] = useState<ActivityItem[]>(initialActivity);

  const createSolution: Store["createSolution"] = useCallback((input) => {
    const id = `${slug(input.name)}-${Math.random().toString(36).slice(2, 6)}`;
    const solution: Solution = {
      id,
      name: input.name,
      description: input.description || "Nuova soluzione di calcolo.",
      status: "draft",
      version: input.version?.trim() || "v1.0.0",
      chart: "bar",
      series: randomSeries(),
      updatedAt: new Date().toISOString().slice(0, 10),
      owner: "Tu",
      parameters: [
        p("input-1", "Input principale", "u", 100, 0, 1000, 1),
        p("input-2", "Coefficiente", "x", 1.2, 0, 10, 0.1),
      ],
      modules: Object.fromEntries(
        (input.modules?.length ? input.modules : (["etl"] as ModuleKey[])).map((key) => [
          key,
          "draft" as SolutionStatus,
        ]),
      ),
      shares: [],
    };
    setSolutions((prev) => [solution, ...prev]);
    setActivity((prev) => [
      { id: `a-${id}`, who: "Tu", what: `hai creato ${input.name}`, when: "ora" },
      ...prev,
    ]);
    return id;
  }, []);

  const deleteSolution: Store["deleteSolution"] = useCallback((id) => {
    setSolutions((prev) => prev.filter((s) => s.id !== id));
  }, []);

  const today = () => new Date().toISOString().slice(0, 10);

  const updateSolution: Store["updateSolution"] = useCallback((id, patch) => {
    setSolutions((prev) =>
      prev.map((s) => (s.id === id ? { ...s, ...patch, updatedAt: today() } : s)),
    );
  }, []);

  const duplicateSolution: Store["duplicateSolution"] = useCallback((id) => {
    let newId: string | undefined;
    setSolutions((prev) => {
      const source = prev.find((s) => s.id === id);
      if (!source) return prev;
      newId = `${slug(source.name)}-copy-${Math.random().toString(36).slice(2, 6)}`;
      const copy: Solution = {
        ...source,
        id: newId,
        name: `${source.name} (copia)`,
        status: "draft",
        updatedAt: today(),
        parameters: source.parameters.map((param) => ({ ...param })),
        modules: { ...source.modules },
        shares: [],
      };
      return [copy, ...prev];
    });
    return newId;
  }, []);

  const addShare: Store["addShare"] = useCallback((id, email, permission) => {
    const trimmed = email.trim();
    if (!trimmed) return;
    setSolutions((prev) =>
      prev.map((s) =>
        s.id !== id
          ? s
          : {
              ...s,
              shares: [
                ...s.shares,
                {
                  id: `share-${Math.random().toString(36).slice(2, 8)}`,
                  email: trimmed,
                  permission,
                },
              ],
            },
      ),
    );
  }, []);

  const updateShare: Store["updateShare"] = useCallback((id, shareId, permission) => {
    setSolutions((prev) =>
      prev.map((s) =>
        s.id !== id
          ? s
          : {
              ...s,
              shares: s.shares.map((sh) => (sh.id === shareId ? { ...sh, permission } : sh)),
            },
      ),
    );
  }, []);

  const removeShare: Store["removeShare"] = useCallback((id, shareId) => {
    setSolutions((prev) =>
      prev.map((s) =>
        s.id !== id ? s : { ...s, shares: s.shares.filter((sh) => sh.id !== shareId) },
      ),
    );
  }, []);

  const updateParameter: Store["updateParameter"] = useCallback((solutionId, paramId, value) => {
    setSolutions((prev) =>
      prev.map((s) =>
        s.id !== solutionId
          ? s
          : {
              ...s,
              parameters: s.parameters.map((param) =>
                param.id === paramId ? { ...param, value } : param,
              ),
            },
      ),
    );
  }, []);

  const runJob: Store["runJob"] = useCallback(
    (solutionId) => {
      const solution = solutions.find((s) => s.id === solutionId);
      const jobId = `job-${Math.random().toString(36).slice(2, 7)}`;
      setJobs((prev) => [
        {
          id: jobId,
          solutionName: solution?.name ?? "Soluzione",
          progress: 4,
          state: "running",
        },
        ...prev,
      ]);
      const timer = setInterval(() => {
        setJobs((prev) =>
          prev.map((j) => {
            if (j.id !== jobId) return j;
            const next = Math.min(100, j.progress + 9 + Math.random() * 12);
            return { ...j, progress: next, state: next >= 100 ? "done" : "running" };
          }),
        );
      }, 700);
      setTimeout(() => clearInterval(timer), 12000);
    },
    [solutions],
  );

  const value = useMemo(
    () => ({
      solutions,
      jobs,
      activity,
      createSolution,
      deleteSolution,
      updateSolution,
      duplicateSolution,
      addShare,
      updateShare,
      removeShare,
      updateParameter,
      runJob,
    }),
    [
      solutions,
      jobs,
      activity,
      createSolution,
      deleteSolution,
      updateSolution,
      duplicateSolution,
      addShare,
      updateShare,
      removeShare,
      updateParameter,
      runJob,
    ],
  );

  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}

export function useSolutions() {
  const ctx = useContext(StoreContext);
  if (!ctx) throw new Error("useSolutions must be used inside SolutionsProvider");
  return ctx;
}
```

### `src/lib/theme.test.tsx`

21 righe

```tsx
import { renderToString } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { ThemeProvider, useTheme } from "./theme";

function Probe() {
  const { theme, themeName, accentHue } = useTheme();
  return <span>{`${theme}|${themeName}|${accentHue}`}</span>;
}

describe("ThemeProvider", () => {
  it("renderizza sul server con il predefinito (scuro, prototipo, nessuna tinta) senza toccare localStorage", () => {
    expect(typeof window).toBe("undefined");
    const html = renderToString(
      <ThemeProvider>
        <Probe />
      </ThemeProvider>,
    );
    expect(html).toContain("dark|prototipo|null");
  });
});
```

### `src/lib/theme.tsx`

55 righe

```tsx
import { createContext, useContext, useEffect, useMemo, useSyncExternalStore } from "react";
import { DEFAULT_PREFERENCE, themeStore } from "@/theme/runtime";
import type { Mode, ThemeName } from "@/theme/runtime";

interface ThemeContextValue {
  /** Modo chiaro/scuro (nome storico, usato da header e pagina ETL). */
  theme: Mode;
  toggle: () => void;
  /** Tema scelto (`data-theme`) e tinta dell'accento (null = quella del tema). */
  themeName: ThemeName;
  accentHue: number | null;
  setThemeName: (name: ThemeName) => void;
  setAccentHue: (hue: number | null) => void;
}

const ThemeContext = createContext<ThemeContextValue>({
  theme: DEFAULT_PREFERENCE.mode,
  toggle: () => {},
  themeName: DEFAULT_PREFERENCE.theme,
  accentHue: DEFAULT_PREFERENCE.accentHue,
  setThemeName: () => {},
  setAccentHue: () => {},
});

/**
 * Espone la preferenza di tema ai componenti. La preferenza vive in
 * `themeStore` (src/theme/runtime.ts), già applicata a <html> dallo script di
 * avvio prima del primo disegno: qui non si scrive nulla al montaggio, si
 * legge soltanto. Il primo render sul server e in idratazione usa il
 * predefinito; subito dopo React allinea il valore salvato (nessun avviso).
 */
export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const pref = useSyncExternalStore(themeStore.subscribe, themeStore.get, () => DEFAULT_PREFERENCE);

  useEffect(() => {
    themeStore.sync();
  }, []);

  const value = useMemo<ThemeContextValue>(
    () => ({
      theme: pref.mode,
      toggle: themeStore.toggleMode,
      themeName: pref.theme,
      accentHue: pref.accentHue,
      setThemeName: themeStore.setTheme,
      setAccentHue: themeStore.setAccentHue,
    }),
    [pref],
  );

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export const useTheme = () => useContext(ThemeContext);
```

### `src/lib/utils.ts`

7 righe

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

