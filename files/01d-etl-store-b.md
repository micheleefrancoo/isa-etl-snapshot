# 01d-etl-store-b.md

File in questo blocco:

- `src/etl-store/__tests__/reduce.test.ts`
- `src/etl-store/__tests__/save-versions.test.ts`
- `src/etl-store/__tests__/store.test.ts`
- `src/etl-store/derived.ts`
- `src/etl-store/index.ts`
- `src/etl-store/persistence.ts`
- `src/etl-store/react.ts`

---

### `src/etl-store/__tests__/reduce.test.ts`

487 righe

```ts
import { describe, expect, it } from "vitest";
import { outputOf, inputsOf } from "../../etl-core";
import type { Card, FilterParams, Params } from "../../etl-core";
import { CARD, GRID, computeSlots } from "../../etl-layout";
import { initialState, reduce } from "..";
import type { EtlState } from "..";
import { COLUMNS, card, createdIds, ok, refused, withDatasetAndFilter } from "./helpers";

/** Dataset collegato al filtro: A -> F -> output. */
function connected(): { state: EtlState; ds: string; filter: string; out: string } {
  const { state, ds, filter } = withDatasetAndFilter();
  const s = ok(state, { type: "connect", payload: { from: ds, to: filter } });
  return { state: s, ds, filter, out: outputOf(s.graph, filter) as string };
}

/** Un box combinato filtro + ordina, alimentato da un dataset. */
function combined(): { state: EtlState; box: string } {
  const { state, ds, filter } = connected();
  let s = ok(state, { type: "addNode", payload: { component: "sort", point: { x: 700, y: 700 } } });
  const sort = createdIds(state, s)[0] as string;
  s = ok(s, { type: "merge", payload: { dragged: sort, target: filter } });
  void ds;
  return { state: s, box: filter };
}

describe("addNode (cassetta e libreria)", () => {
  it("riesce: dataset dalla cassetta, allineato alla griglia, con nome 'Dataset 1'", () => {
    const s0 = initialState();
    const s = ok(s0, {
      type: "addNode",
      payload: { component: "dataset", point: { x: 300, y: 300 } },
    });
    const id = createdIds(s0, s)[0] as string;
    const c = card(s.graph, id);
    expect(c).toMatchObject({ kind: "dataset", name: "Dataset 1", x: 260, y: 260 });
    expect(c.x % GRID).toBe(0);
  });

  it("riesce: dalla libreria il dataset porta percorso e colonne", () => {
    const { state, ds } = withDatasetAndFilter();
    expect(card(state.graph, ds).params[0]).toMatchObject({
      path: "vendite.csv",
      columns: COLUMNS,
    });
    expect(card(state.graph, ds).name).toBe("vendite");
  });

  it("riesce: rilasciato su un box, un dataset si collega e nasce l'output", () => {
    const { state, filter } = withDatasetAndFilter();
    const s = ok(state, {
      type: "addNode",
      payload: {
        component: "dataset",
        libraryId: "lib-1",
        point: { x: 0, y: 0 },
        target: { node: filter },
      },
    });
    expect(inputsOf(s.graph, filter)).toHaveLength(1);
    expect(outputOf(s.graph, filter)).not.toBeNull();
  });

  it("riesce: una lavorazione rilasciata su un box si fonde", () => {
    const { state, filter } = withDatasetAndFilter();
    const s = ok(state, {
      type: "addNode",
      payload: { component: "sort", point: { x: 0, y: 0 }, target: { node: filter } },
    });
    expect(card(s.graph, filter).components).toEqual(["filter", "sort"]);
    expect(Object.keys(s.graph.cards)).toHaveLength(2);
  });

  it("riesce: una lavorazione rilasciata su un cavo viene inserita", () => {
    const { state, ds, filter } = connected();
    const s = ok(state, {
      type: "addNode",
      payload: {
        component: "sort",
        point: { x: 450, y: 320 },
        target: { link: { from: ds, to: filter } },
      },
    });
    const sort = createdIds(state, s).find((id) => card(s.graph, id).kind === "op") as string;
    expect(s.graph.links).toContainEqual({ from: ds, to: sort });
    expect(inputsOf(s.graph, filter)[0]?.from).toBe(outputOf(s.graph, sort));
  });

  it("rifiuta: tipo sconosciuto, dataset non in libreria, dataset su un cavo", () => {
    const { state, ds, filter } = connected();
    expect(
      refused(state, {
        type: "addNode",
        payload: { component: "nope" as "filter", point: { x: 0, y: 0 } },
      }),
    ).toMatch(/sconosciuto/);
    expect(
      refused(state, {
        type: "addNode",
        payload: { component: "dataset", libraryId: "lib-9", point: { x: 0, y: 0 } },
      }),
    ).toMatch(/libreria/);
    expect(
      refused(state, {
        type: "addNode",
        payload: {
          component: "dataset",
          point: { x: 0, y: 0 },
          target: { link: { from: ds, to: filter } },
        },
      }),
    ).toMatch(/lavorazione/);
  });
});

describe("moveNodes (spostamento da tastiera)", () => {
  it("riesce: sposta e limita al mondo", () => {
    const { state, ds } = withDatasetAndFilter();
    const s = ok(state, { type: "moveNodes", payload: { ids: [ds], dx: 2, dy: -9999 } });
    expect(card(s.graph, ds)).toMatchObject({ x: card(state.graph, ds).x + 2, y: 6 });
  });

  it("rifiuta: in Organizzato le postazioni sono fisse", () => {
    const { state, ds } = withDatasetAndFilter();
    const g = ok(state, { type: "setMode", payload: { mode: "grid" } });
    expect(refused(g, { type: "moveNodes", payload: { ids: [ds], dx: 2, dy: 0 } })).toMatch(
      /Organizzato/,
    );
  });
});

describe("dropNodes (spostamento e rilascio)", () => {
  it("riesce: rilasciato nel vuoto, gli altri si scansano e si riallineano", () => {
    const { state, ds } = withDatasetAndFilter();
    const s = ok(state, { type: "dropNodes", payload: { ids: [ds], dx: 100, dy: 50 } });
    expect(card(s.graph, ds)).toMatchObject({
      x: card(state.graph, ds).x + 100,
      y: card(state.graph, ds).y + 50,
    });
  });

  it("riesce: rilasciato su un box si collega, e il dataset torna al suo posto", () => {
    const { state, ds, filter } = withDatasetAndFilter();
    const s = ok(state, {
      type: "dropNodes",
      payload: { ids: [ds], dx: 500, dy: 0, target: { node: filter } },
    });
    expect(inputsOf(s.graph, filter)[0]?.from).toBe(ds);
    expect(card(s.graph, ds)).toMatchObject({
      x: card(state.graph, ds).x,
      y: card(state.graph, ds).y,
    });
  });

  it("rifiuta: nessun nodo esistente", () => {
    expect(
      refused(initialState(), { type: "dropNodes", payload: { ids: ["x"], dx: 1, dy: 1 } }),
    ).toMatch(/Nessun nodo/);
  });
});

describe("connect", () => {
  it("riesce: in entrambi i versi (dalla lavorazione verso il dataset si collega al contrario)", () => {
    const { state, ds, filter } = withDatasetAndFilter();
    const s = ok(state, { type: "connect", payload: { from: filter, to: ds } });
    expect(s.graph.links).toContainEqual({ from: ds, to: filter });
  });

  it("rifiuta con il motivo di etl-core: due dataset, cicli, lavorazione con lavorazione", () => {
    const { state, ds, filter, out } = connected();
    expect(refused(state, { type: "connect", payload: { from: ds, to: out } })).toBe(
      "Due dataset non si fondono: serve una lavorazione, ad esempio un Join",
    );
    expect(refused(state, { type: "connect", payload: { from: out, to: filter } })).toMatch(
      /ciclo/,
    );
    const s = ok(state, {
      type: "addNode",
      payload: { component: "sort", point: { x: 100, y: 900 } },
    });
    const sort = createdIds(state, s)[0] as string;
    expect(refused(s, { type: "connect", payload: { from: sort, to: filter } })).toMatch(
      /non si possono collegare/,
    );
  });
});

describe("merge", () => {
  it("riesce: il box risultante ha i passaggi di entrambi", () => {
    const { state, box } = combined();
    expect(card(state.graph, box).components).toEqual(["filter", "sort"]);
  });

  it("rifiuta: un dataset non si fonde", () => {
    const { state, ds, filter } = withDatasetAndFilter();
    expect(refused(state, { type: "merge", payload: { dragged: ds, target: filter } })).toMatch(
      /due lavorazioni/,
    );
  });
});

describe("insertOnLink", () => {
  it("riesce: A -> X -> output di X -> F", () => {
    const { state, ds, filter } = connected();
    let s = ok(state, {
      type: "addNode",
      payload: { component: "sort", point: { x: 100, y: 1000 } },
    });
    const x = createdIds(state, s)[0] as string;
    s = ok(s, { type: "insertOnLink", payload: { node: x, link: { from: ds, to: filter } } });
    expect(s.graph.links).toContainEqual({ from: ds, to: x });
  });

  it("rifiuta: collegamento box -> output", () => {
    const { state, filter, out } = connected();
    const s = ok(state, {
      type: "addNode",
      payload: { component: "sort", point: { x: 100, y: 1000 } },
    });
    const x = createdIds(state, s)[0] as string;
    expect(
      refused(s, { type: "insertOnLink", payload: { node: x, link: { from: filter, to: out } } }),
    ).toMatch(/Si può inserire/);
  });
});

describe("detachStep, deleteStep, reorderSteps", () => {
  it("detachStep riesce: il passaggio diventa un nodo sotto il box", () => {
    const { state, box } = combined();
    const s = ok(state, { type: "detachStep", payload: { box, index: 1 } });
    const d = createdIds(state, s)[0] as string;
    expect(card(s.graph, d).components).toEqual(["sort"]);
    expect(card(s.graph, box).components).toEqual(["filter"]);
  });

  it("detachStep rifiuta: box non combinato o passaggio inesistente", () => {
    const { state, filter } = withDatasetAndFilter();
    expect(refused(state, { type: "detachStep", payload: { box: filter, index: 0 } })).toMatch(
      /combinato/,
    );
    expect(refused(state, { type: "detachStep", payload: { box: filter, index: 3 } })).toMatch(
      /non esiste/,
    );
  });

  it("deleteStep riesce e rifiuta su un box semplice", () => {
    const { state, box } = combined();
    const s = ok(state, { type: "deleteStep", payload: { box, index: 0 } });
    expect(card(s.graph, box).components).toEqual(["sort"]);
    expect(refused(s, { type: "deleteStep", payload: { box, index: 0 } })).toMatch(/combinato/);
  });

  it("reorderSteps riesce (componenti e parametri insieme) e rifiuta un indice fuori range", () => {
    const { state, box } = combined();
    const s = ok(state, { type: "reorderSteps", payload: { box, from: 0, to: 1 } });
    expect(card(s.graph, box).components).toEqual(["sort", "filter"]);
    expect((card(s.graph, box).params[1] as unknown as FilterParams).conditions).toBeDefined();
    expect(refused(state, { type: "reorderSteps", payload: { box, from: 0, to: 5 } })).toMatch(
      /non esiste/,
    );
  });
});

describe("deleteNodes, deleteLink", () => {
  it("deleteNodes riesce: il box sparisce con il suo output", () => {
    const { state, filter, out } = connected();
    const s = ok(state, { type: "deleteNodes", payload: { ids: [filter] } });
    expect(s.graph.cards[filter]).toBeUndefined();
    expect(s.graph.cards[out]).toBeUndefined();
  });

  it("deleteNodes rifiuta: nessun nodo esistente", () => {
    expect(refused(initialState(), { type: "deleteNodes", payload: { ids: ["x"] } })).toMatch(
      /Nessun nodo/,
    );
  });

  it("deleteLink riesce: l'output senza ingressi sparisce; rifiuta un collegamento inesistente", () => {
    const { state, ds, filter, out } = connected();
    const s = ok(state, { type: "deleteLink", payload: { link: { from: ds, to: filter } } });
    expect(s.graph.cards[out]).toBeUndefined();
    expect(refused(s, { type: "deleteLink", payload: { link: { from: ds, to: filter } } })).toMatch(
      /non esiste/,
    );
  });
});

describe("duplicate", () => {
  it("riesce: copie spostate di GRID*2, selezionate; gli output esclusi", () => {
    const { state, ds, out } = connected();
    const s = ok(state, { type: "duplicate", payload: { ids: [ds, out] } });
    const created = createdIds(state, s);
    expect(created).toHaveLength(1);
    expect(card(s.graph, created[0] as string).name).toBe("vendite copia");
    expect(s.selection).toEqual(created);
  });

  it("rifiuta: solo output", () => {
    const { state, out } = connected();
    expect(refused(state, { type: "duplicate", payload: { ids: [out] } })).toMatch(/output/);
  });
});

describe("setParams, renameNode", () => {
  it("setParams riesce: il filtro compilato diventa completo", () => {
    const { state, filter } = connected();
    const params: FilterParams = {
      conditions: [
        { column: "regione", op: "=", mode: "list", values: ["Nord"], text: "", sep: "," },
      ],
    };
    const s = ok(state, {
      type: "setParams",
      payload: { node: filter, index: 0, params: params as unknown as Params },
    });
    expect(card(s.graph, filter).params[0]).toEqual(params);
  });

  it("setParams rifiuta: passaggio inesistente", () => {
    const { state, filter } = connected();
    expect(
      refused(state, { type: "setParams", payload: { node: filter, index: 2, params: {} } }),
    ).toMatch(/non esiste/);
  });

  it("renameNode riesce e rifiuta un nome vuoto", () => {
    const { state, filter } = connected();
    const s = ok(state, { type: "renameNode", payload: { node: filter, name: "  Solo Nord " } });
    expect(card(s.graph, filter).name).toBe("Solo Nord");
    expect(refused(s, { type: "renameNode", payload: { node: filter, name: "  " } })).toMatch(
      /vuoto/,
    );
  });
});

describe("setMode, autoLayout", () => {
  it("setMode riesce: in Organizzato ogni nodo ha una postazione; in Libero le perde", () => {
    const { state } = connected();
    const g = ok(state, { type: "setMode", payload: { mode: "grid" } });
    const slots = computeSlots();
    for (const c of Object.values(g.graph.cards))
      expect(slots[c.slot as number]).toEqual({ x: c.x, y: c.y });
    const f = ok(g, { type: "setMode", payload: { mode: "free" } });
    for (const c of Object.values(f.graph.cards)) expect(c.slot).toBeUndefined();
  });

  it("setMode rifiuta una modalità sconosciuta", () => {
    expect(refused(initialState(), { type: "setMode", payload: { mode: "x" as "free" } })).toMatch(
      /sconosciuta/,
    );
  });

  it("autoLayout riesce: il flusso va da sinistra a destra; rifiuta un'area non valida", () => {
    const { state, ds, filter, out } = connected();
    const s = ok(state, { type: "autoLayout", payload: { viewport: { w: 712, h: 520 } } });
    const x = (id: string): number => (s.graph.cards[id] as Card).x;
    expect(x(ds)).toBeLessThan(x(filter));
    expect(x(filter)).toBeLessThan(x(out));
    expect(refused(state, { type: "autoLayout", payload: { viewport: { w: 0, h: 520 } } })).toMatch(
      /non valida/,
    );
  });
});

describe("select, inspect", () => {
  it("select riesce e rifiuta un nodo inesistente", () => {
    const { state, ds, filter } = withDatasetAndFilter();
    expect(ok(state, { type: "select", payload: { ids: [ds, filter, ds] } }).selection).toEqual([
      ds,
      filter,
    ]);
    expect(refused(state, { type: "select", payload: { ids: ["x"] } })).toMatch(/non esiste/);
  });

  it("inspect riesce e rifiuta un passaggio inesistente", () => {
    const { state, filter } = withDatasetAndFilter();
    expect(ok(state, { type: "inspect", payload: { node: filter } }).inspector).toEqual({
      nodeId: filter,
      step: 0,
    });
    expect(refused(state, { type: "inspect", payload: { node: filter, step: 1 } })).toMatch(
      /non esiste/,
    );
  });
});

describe("loadDataset", () => {
  it("riesce: id lib-N; rifiuta un file senza colonne", () => {
    const s = ok(initialState(), {
      type: "loadDataset",
      payload: { name: "a", path: "a.csv", columns: COLUMNS, rows: 3 },
    });
    expect(s.library.map((l) => l.id)).toEqual(["lib-1"]);
    expect(
      refused(s, {
        type: "loadDataset",
        payload: { name: "b", path: "b.csv", columns: [], rows: 0 },
      }),
    ).toBe("Il file non contiene colonne leggibili");
  });
});

describe("setPanel, setView, setOptions", () => {
  it("setPanel: aprire un pannello chiude l'altro sullo stesso bordo; cambiare lato lo riapre", () => {
    let s = ok(initialState(), { type: "setPanel", payload: { panel: "insp", side: "left" } });
    expect(s.panels).toEqual({
      tools: { side: "left", open: false },
      insp: { side: "left", open: true },
    });
    s = ok(s, { type: "setPanel", payload: { panel: "tools", open: true } });
    expect(s.panels.insp.open).toBe(false);
    expect(
      refused(s, { type: "setPanel", payload: { panel: "x" as "tools", open: true } }),
    ).toMatch(/sconosciuto/);
  });

  it("setPanel: un solo pannello aperto alla volta, su qualunque bordo, nello stesso aggiornamento", () => {
    const sides = ["left", "right", "top", "bottom"] as const;
    for (const a of sides) {
      for (const b of sides) {
        let s = ok(initialState(), { type: "setPanel", payload: { panel: "tools", side: a } });
        s = ok(s, { type: "setPanel", payload: { panel: "insp", side: b, open: true } });
        // aprire l'Inspector chiude la cassetta, anche se stanno su bordi diversi
        expect(s.panels.insp.open).toBe(true);
        expect(s.panels.tools.open).toBe(false);
        s = ok(s, { type: "setPanel", payload: { panel: "tools", open: true } });
        expect(s.panels.tools.open).toBe(true);
        expect(s.panels.insp.open).toBe(false);
        s = ok(s, { type: "setPanel", payload: { panel: "insp", open: true } });
        expect([s.panels.tools.open, s.panels.insp.open]).toEqual([false, true]);
        // chiudere non apre l'altro
        s = ok(s, { type: "setPanel", payload: { panel: "insp", open: false } });
        expect([s.panels.tools.open, s.panels.insp.open]).toEqual([false, false]);
      }
    }
  });

  it("setView: zoom limitato tra 0,35 e 2; rifiuta valori non finiti", () => {
    const s = ok(initialState(), { type: "setView", payload: { x: 10, zoom: 9 } });
    expect(s.view).toEqual({ x: 10, y: 0, zoom: 2 });
    expect(refused(s, { type: "setView", payload: { y: Number.NaN } })).toMatch(/non valida/);
  });

  it("setOptions: flusso solo se valido; rifiuta snodi negativi", () => {
    const s = ok(initialState(), { type: "setOptions", payload: { flowOnlyIfValid: true } });
    expect(s.options.flowOnlyIfValid).toBe(true);
    expect(refused(s, { type: "setOptions", payload: { maxBends: -1 } })).toMatch(/intero/);
  });
});

describe("comando sconosciuto e purezza", () => {
  it("un comando sconosciuto viene rifiutato", () => {
    const s = initialState();
    const out = reduce(s, { type: "boh", payload: {} } as unknown as Parameters<typeof reduce>[1]);
    expect(out.result).toEqual({ ok: false, reason: "Comando sconosciuto: boh" });
    expect(out.state).toBe(s);
  });

  it("reduce non modifica lo stato che riceve", () => {
    const { state, ds, filter } = withDatasetAndFilter();
    const snapshot = JSON.stringify(state);
    reduce(state, { type: "connect", payload: { from: ds, to: filter } });
    reduce(state, { type: "setMode", payload: { mode: "grid" } });
    reduce(state, { type: "duplicate", payload: { ids: [ds] } });
    expect(JSON.stringify(state)).toBe(snapshot);
    void CARD;
  });
});

describe("clearAll", () => {
  it("elimina nodi e collegamenti, tiene libreria e modalità; rifiuta su un canvas vuoto", () => {
    const { state } = connected();
    let withLib = ok(state, {
      type: "loadDataset",
      payload: { name: "a", path: "a.csv", columns: COLUMNS, rows: 3 },
    });
    withLib = ok(withLib, { type: "setMode", payload: { mode: "grid" } });
    const s = ok(withLib, { type: "clearAll", payload: {} });
    expect(Object.keys(s.graph.cards)).toEqual([]);
    expect(s.graph.links).toEqual([]);
    expect(s.selection).toEqual([]);
    expect(s.inspector).toEqual({ nodeId: null, step: 0 });
    expect(s.library).toEqual(withLib.library);
    expect(s.mode).toBe("grid");
    expect(refused(s, { type: "clearAll", payload: {} })).toBe("Il canvas è già vuoto");
  });
});
```

### `src/etl-store/__tests__/save-versions.test.ts`

179 righe

```ts
import { describe, expect, it } from "vitest";
import type { Card, MultiRow } from "../../etl-core";
import { SAVE_VERSION, fromSaved, parseSaved, toSaved } from "..";

/** Un salvataggio della versione 1, come lo scriveva l'app prima delle colonne multiple. */
const V1_FIXTURE = {
  version: 1,
  mode: "free",
  library: [],
  panels: {
    tools: { side: "left", open: true },
    insp: { side: "right", open: false },
  },
  options: { flowOnlyIfValid: false, maxBends: 2 },
  graph: {
    links: [],
    cards: {
      "ds-1": card(
        "ds-1",
        "dataset",
        ["dataset"],
        [{ source: "CSV", path: "a.csv", header: "Sì" }],
      ),
      "op-1": card("op-1", "op", ["cast"], [{ items: [{ column: "importo", to: "intero" }] }]),
      "op-2": card("op-2", "op", ["round"], [{ column: "importo", decimals: "3" }]),
      "op-3": card(
        "op-3",
        "op",
        ["scale"],
        [{ items: [{ column: "importo", method: "z-score" }] }],
      ),
      "op-4": card(
        "op-4",
        "op",
        ["textClean"],
        [{ items: [{ column: "nome", action: "maiuscole" }] }],
      ),
      "op-5": card("op-5", "op", ["fillNa"], [{ items: [{ column: "nome", value: "n/d" }] }]),
      "op-6": card(
        "op-6",
        "op",
        ["replaceVal"],
        [
          {
            items: [
              {
                column: "regione",
                match: "è uguale a",
                find: { mode: "manual", values: [], text: "Nord, Sud", sep: "," },
                with: "N",
              },
            ],
          },
        ],
      ),
      "op-7": card(
        "op-7",
        "op",
        ["dedup"],
        [{ keep: "la prima", items: [{ column: "id", cmp: "esatto" }] }],
      ),
      "op-8": card(
        "op-8",
        "op",
        ["selectCols"],
        [{ mode: "tieni", items: [{ column: "id" }, { column: "" }] }],
      ),
      "op-9": card("op-9", "op", ["sort"], [{ items: [{ column: "id", dir: "decrescente" }] }]),
      "op-10": card(
        "op-10",
        "op",
        ["aggregate"],
        [
          {
            groupBy: [{ column: "regione" }],
            measures: [{ column: "importo", fn: "somma", alias: "tot" }],
          },
        ],
      ),
      "op-11": card("op-11", "op", ["rename"], [{ items: [{ column: "a", newName: "b" }] }]),
      "op-12": card(
        "op-12",
        "op",
        ["filter"],
        [
          {
            logic: "O",
            conditions: [
              { column: "a", op: "=", mode: "list", values: ["x"], text: "", sep: "," },
              { column: "b", op: "=", mode: "list", values: ["y"], text: "", sep: "," },
            ],
          },
        ],
      ),
    },
  },
};

function card(
  id: string,
  kind: "dataset" | "op",
  components: string[],
  params: unknown[],
): Record<string, unknown> {
  return { id, kind, components, params, name: id, x: 10, y: 10 };
}

const rows = (c: Card | undefined, key = "items"): MultiRow[] =>
  ((c?.params[0] as Record<string, unknown>)[key] ?? []) as MultiRow[];

describe("versione del formato: 1 → 2", () => {
  it("la versione corrente è la 2 e toSaved la scrive", () => {
    expect(SAVE_VERSION).toBe(2);
    const s = fromSaved(V1_FIXTURE);
    expect(s).not.toBeNull();
    expect(toSaved(s!).version).toBe(2);
  });

  it("un salvataggio v1 con tutte le operazioni coinvolte si carica migrato", () => {
    const g = fromSaved(JSON.parse(JSON.stringify(V1_FIXTURE)))!.graph;
    const c = (id: string) => g.cards[id];
    expect(rows(c("op-1"))[0]?.["columns"]).toEqual(["importo"]);
    expect(rows(c("op-2"))[0]).toMatchObject({ columns: ["importo"], decimals: "3" }); // voce singola
    expect(rows(c("op-3"))[0]?.["columns"]).toEqual(["importo"]);
    expect(rows(c("op-4"))[0]?.["columns"]).toEqual(["nome"]);
    expect(rows(c("op-5"))[0]?.["columns"]).toEqual(["nome"]);
    expect(rows(c("op-6"))[0]?.["columns"]).toEqual(["regione"]);
    // anche la migrazione dei valori scritti a mano passa da ensureParams
    expect((rows(c("op-6"))[0]?.["find"] as { values: string[] }).values).toEqual(["Nord", "Sud"]);
    expect(rows(c("op-7"))[0]?.["columns"]).toEqual(["id"]);
    expect(rows(c("op-8")).map((r) => r["columns"])).toEqual([["id"], []]);
    expect(rows(c("op-9"))[0]?.["columns"]).toEqual(["id"]);
    expect(rows(c("op-10"), "groupBy")[0]?.["columns"]).toEqual(["regione"]);
    expect(rows(c("op-10"), "measures")[0]).toMatchObject({ columns: ["importo"], alias: "tot" });
    for (const id of ["op-1", "op-3", "op-4", "op-5", "op-6", "op-7", "op-8", "op-9"]) {
      expect(
        rows(c(id)).every((r) => !("column" in r)),
        id,
      ).toBe(true);
    }
    // restano a colonna singola
    expect(rows(c("op-11"))[0]).toMatchObject({ column: "a", newName: "b" });
    expect("columns" in (rows(c("op-11"))[0] as MultiRow)).toBe(false);
    // il filtro passa dalla sua migrazione (connettore dalla seconda condizione)
    const conds = (c("op-12")?.params[0] as { conditions: { conn?: string }[] }).conditions;
    expect(conds[1]?.conn).toBe("OR");
  });

  it("un salvataggio v1 caricato e risalvato è v2 e si ricarica uguale", () => {
    const first = fromSaved(JSON.parse(JSON.stringify(V1_FIXTURE)))!;
    const saved = toSaved(first);
    const again = parseSaved(JSON.stringify(saved))!;
    expect(again.graph).toEqual(first.graph);
    expect(toSaved(again)).toEqual(saved);
  });

  it("un salvataggio v2 non cambia: nessuna migrazione, nemmeno su una riga nel vecchio formato", () => {
    const raw = JSON.parse(JSON.stringify({ ...V1_FIXTURE, version: 2 }));
    const loaded = fromSaved(raw)!;
    expect(loaded.graph.cards["op-1"]?.params[0]).toEqual({
      items: [{ column: "importo", to: "intero" }],
    });
    expect(loaded.graph.cards["op-2"]?.params[0]).toEqual({ column: "importo", decimals: "3" });
    expect(loaded.graph.cards["op-12"]?.params[0]).toEqual(raw.graph.cards["op-12"].params[0]);
  });

  it("le versioni sconosciute si ignorano", () => {
    expect(fromSaved({ ...V1_FIXTURE, version: 3 })).toBeNull();
    expect(fromSaved({ ...V1_FIXTURE, version: 0 })).toBeNull();
  });

  it("il caricamento non modifica il dato ricevuto", () => {
    const raw = JSON.parse(JSON.stringify(V1_FIXTURE));
    const copy = JSON.stringify(raw);
    fromSaved(raw);
    expect(JSON.stringify(raw)).toBe(copy);
  });
});
```

### `src/etl-store/__tests__/store.test.ts`

283 righe

```ts
import { describe, expect, it } from "vitest";
import { outputOf } from "../../etl-core";
import type { Card, FilterParams, Params } from "../../etl-core";
import { HISTORY_LIMIT, createEtlStore, initialState } from "..";
import type { EtlStore } from "..";
import { COLUMNS, withDatasetAndFilter } from "./helpers";

/** `tick` = millisecondi fra un comando e il successivo (1500: nessun raggruppamento). */
function storeWith(tick = 1): { store: EtlStore; ds: string; filter: string } {
  const { state, ds, filter } = withDatasetAndFilter();
  let t = 0;
  return { store: createEtlStore({ initial: state, now: () => (t += tick) }), ds, filter };
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
    // comandi distanziati di 1500 ms: ognuno è un passo (nessun raggruppamento)
    const { store, ds } = storeWith(1500);
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

describe("clearAll", () => {
  it("è un solo passo di cronologia e una voce di registro; annullare ripristina tutto", () => {
    const { store } = storeWith(1500);
    const before = store.getState().graph;
    const logBefore = store.getLog().length;
    expect(store.dispatch({ type: "clearAll", payload: {} })).toEqual({ ok: true });
    expect(store.historySize().past).toBe(1);
    expect(store.getLog().length).toBe(logBefore + 1);
    expect(store.getLog().at(-1)?.type).toBe("clearAll");
    expect(Object.keys(store.getState().graph.cards)).toEqual([]);
    store.undo();
    expect(store.getState().graph).toEqual(before);
    store.redo();
    expect(Object.keys(store.getState().graph.cards)).toEqual([]);
  });

  it("la libreria dei dataset caricati non viene toccata", () => {
    const { store } = storeWith();
    store.dispatch({
      type: "loadDataset",
      payload: { name: "b", path: "b.csv", columns: COLUMNS, rows: 1 },
    });
    const library = store.getState().library;
    expect(library.length).toBeGreaterThan(0);
    store.dispatch({ type: "clearAll", payload: {} });
    expect(store.getState().library).toEqual(library);
    store.undo();
    expect(store.getState().library).toEqual(library);
  });
});

describe("registro con colonne multiple", () => {
  it("il payload di setParams riporta le colonne come elenco, nell'ordine", () => {
    const { store, filter } = storeWith();
    const r = store.dispatch({
      type: "setParams",
      payload: {
        node: filter,
        index: 0,
        params: { items: [{ columns: ["b", "a"], to: "intero" }] },
      },
    });
    expect(r).toEqual({ ok: true });
    const last = store.getLog().at(-1) as { type: string; payload: unknown };
    expect(last.type).toBe("setParams");
    expect(
      (last.payload as { params: { items: { columns: string[] }[] } }).params.items[0]?.columns,
    ).toEqual(["b", "a"]);
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

36 righe

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
export { reduce, dropAt, paletteRelation } from "./reduce";
export {
  createEtlStore,
  HISTORY_LIMIT,
  HISTORY_COMMANDS,
  GROUP_WINDOW_MS,
  UNLOGGED_COMMANDS,
  groupKey,
} from "./store";
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

