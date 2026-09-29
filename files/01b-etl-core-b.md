# 01b-etl-core-b.md

File in questo blocco:

- `src/etl-core/__tests__/params.test.ts`
- `src/etl-core/__tests__/relations.test.ts`
- `src/etl-core/__tests__/schema.test.ts`
- `src/etl-core/__tests__/state.test.ts`
- `src/etl-core/catalog/icons.ts`
- `src/etl-core/catalog/operations.ts`
- `src/etl-core/catalog/params.ts`
- `src/etl-core/data/csv.ts`
- `src/etl-core/index.ts`

---

### `src/etl-core/__tests__/params.test.ts`

124 righe

```ts
import { describe, expect, it } from "vitest";
import {
  ensureKeys,
  ensureMulti,
  migrateFilterLogic,
  hasEquiJoinCondition,
  createValuesField,
} from "../catalog/params";
import type { FilterParams, JoinParams, MultiRow } from "../model/types";

describe("migrazioni (scenario 9)", () => {
  it("il selettore globale E/O del filtro diventa il connettore di ogni condizione", () => {
    const par: FilterParams = {
      logic: "O",
      conditions: [
        { column: "regione", op: "=", mode: "list", values: ["Nord"], text: "", sep: "," },
        { column: "importo", op: ">", mode: "manual", values: [], text: "100", sep: "," },
        { column: "stato", op: "=", mode: "list", values: ["Chiuso"], text: "", sep: "," },
      ],
    };
    const migrated = migrateFilterLogic(par);
    expect(migrated.logic).toBeUndefined();
    expect(migrated.conditions[0]?.conn).toBeUndefined();
    expect(migrated.conditions[1]?.conn).toBe("OR");
    expect(migrated.conditions[2]?.conn).toBe("OR");
  });

  it("una condizione con conn gia impostato non viene sovrascritta dalla migrazione", () => {
    const par: FilterParams = {
      logic: "O",
      conditions: [
        { column: "a", op: "=", mode: "list", values: [], text: "", sep: "," },
        { column: "b", op: "=", mode: "list", values: [], text: "", sep: ",", conn: "AND" },
      ],
    };
    const migrated = migrateFilterLogic(par);
    expect(migrated.conditions[1]?.conn).toBe("AND");
  });

  it("chiavi di join leftKey/rightKey diventano una lista `keys`", () => {
    const par = { type: "inner", leftKey: "id", rightKey: "customer_id" } as unknown as JoinParams;
    const keys = ensureKeys(par);
    expect(keys).toEqual([
      expect.objectContaining({
        left: "id",
        right: "customer_id",
        op: "=",
        lmode: "col",
        rmode: "col",
      }),
    ]);
  });

  it("campi semplici (vecchio formato) diventano la prima voce della lista", () => {
    const legacy = { column: "importo", decimals: "3" };
    const migrated = ensureMulti("round", legacy);
    const rows = migrated["items"] as MultiRow[];
    expect(rows).toHaveLength(1);
    expect(rows[0]?.["column"]).toBe("importo");
    expect(rows[0]?.["decimals"]).toBe("3");
  });

  it("un testo (vecchio formato) viene migrato in una lista di valori", () => {
    const legacy = { items: [{ column: "regione", find: "Nord, Centro" }] };
    const migrated = ensureMulti("replaceVal", legacy);
    const rows = migrated["items"] as MultiRow[];
    const find = rows[0]?.["find"];
    expect(find).toEqual({ mode: "list", values: ["Nord", "Centro"], text: "", sep: "," });
  });

  it("un valore `values` gia in forma di oggetto non viene toccato dalla migrazione", () => {
    const already = createValuesField();
    already.mode = "list";
    already.values = ["A", "B"];
    const legacy = { items: [{ column: "x", find: already }] };
    const migrated = ensureMulti("replaceVal", legacy);
    const rows = migrated["items"] as MultiRow[];
    expect(rows[0]?.["find"]).toBe(already);
  });

  it("l'operatore mancante nelle condizioni di join diventa '='", () => {
    const par: JoinParams = { type: "inner", keys: [{ left: "a", right: "b" }] };
    const [key] = ensureKeys(par);
    expect(key?.op).toBe("=");
  });

  it("lmode, rmode e rlist hanno valori predefiniti", () => {
    const par: JoinParams = { type: "inner", keys: [{ left: "a", right: "b" }] };
    const [key] = ensureKeys(par);
    expect(key?.lmode).toBe("col");
    expect(key?.rmode).toBe("col");
    expect(key?.rlist).toEqual({ mode: "list", values: [], text: "", sep: "," });
  });
});

describe("avviso di prestazioni sul Join (scenario 12): hasEquiJoinCondition", () => {
  it("false (quindi l'avviso va mostrato) con sole disuguaglianze", () => {
    const keys = ensureKeys({
      type: "inner",
      keys: [{ left: "a", right: "b", op: ">" }],
    });
    expect(hasEquiJoinCondition(keys)).toBe(false);
  });

  it("false (quindi l'avviso va mostrato) con uguaglianze colonna = valore", () => {
    const keys = ensureKeys({
      type: "inner",
      keys: [{ left: "a", right: "", op: "=", lmode: "col", rmode: "val", rval: "Nord" }],
    });
    expect(hasEquiJoinCondition(keys)).toBe(false);
  });

  it("true (quindi l'avviso NON va mostrato) con almeno una condizione colonna = colonna", () => {
    const keys = ensureKeys({
      type: "inner",
      keys: [
        { left: "a", right: "", op: ">", lmode: "col", rmode: "val", rval: "10" },
        { left: "id", right: "customer_id", op: "=" },
      ],
    });
    expect(hasEquiJoinCondition(keys)).toBe(true);
  });
});
```

### `src/etl-core/__tests__/relations.test.ts`

167 righe

```ts
import { describe, expect, it } from "vitest";
import { dataset, op, buildGraph, testIdGenerator } from "./helpers";
import { connect, enforceCapacity, refreshOutput, deleteStep } from "../rules/mutations";
import { relation, boxCapacity } from "../rules/relations";
import { cardById, outputOf, inputsOf } from "../model/graph";
import type { Graph } from "../model/types";

function expectOk(result: { ok: boolean }): asserts result is { ok: true; graph: Graph } {
  expect(result.ok).toBe(true);
}

describe("connect: ciclo a distanza (scenario 1)", () => {
  it("A -> box1 -> out1 -> box2 -> out2; collegare out2 a box1 e rifiutato per ciclo", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box1", ["filter"]), op("box2", ["filter"])]);

    const r1 = connect(graph, "A", "box1", nextId);
    expectOk(r1);
    graph = refreshOutput(r1.graph, "box1", nextId);
    const out1 = outputOf(graph, "box1");
    expect(out1).not.toBeNull();

    const r2 = connect(graph, out1 as string, "box2", nextId);
    expectOk(r2);
    graph = refreshOutput(r2.graph, "box2", nextId);
    const out2 = outputOf(graph, "box2");
    expect(out2).not.toBeNull();

    const rejected = connect(graph, out2 as string, "box1", nextId);
    expect(rejected.ok).toBe(false);
    if (!rejected.ok) {
      expect(rejected.reason).toBe(
        "Un box non può agganciarsi a ciò che produce: sarebbe un ciclo infinito",
      );
    }
  });
});

describe("boxCapacity e connect: capienza (scenario 2)", () => {
  it("un join accetta 2 tabelle, join+union 3", () => {
    const joinBox = op("joinBox", ["join"]);
    const joinUnionBox = op("comboBox", ["join", "union"]);
    expect(boxCapacity(joinBox)).toBe(2);
    expect(boxCapacity(joinUnionBox)).toBe(3);
  });

  it("una terza tabella su un box con un solo join e rifiutata con la capienza reale", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), dataset("B"), dataset("C"), op("box", ["join"])]);
    const r1 = connect(graph, "A", "box", nextId);
    expectOk(r1);
    graph = r1.graph;
    const r2 = connect(graph, "B", "box", nextId);
    expectOk(r2);
    graph = r2.graph;

    const r3 = connect(graph, "C", "box", nextId);
    expect(r3.ok).toBe(false);
    if (!r3.ok) expect(r3.reason).toBe("Il box ha già tutte le sue 2 tabelle");
  });

  it("un box senza join (capacita 1) rifiuta la seconda tabella con il motivo generico", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), dataset("B"), op("box", ["sort"])]);
    const r1 = connect(graph, "A", "box", nextId);
    expectOk(r1);
    graph = r1.graph;
    const r2 = connect(graph, "B", "box", nextId);
    expect(r2.ok).toBe(false);
    if (!r2.ok) expect(r2.reason).toBe("Il box accetta una sola tabella in ingresso");
  });
});

describe("enforceCapacity: rimozione del join da un box con due ingressi (scenario 5)", () => {
  it("resta il collegamento piu vecchio e l'output torna a capacity 1", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), dataset("B"), op("box", ["join", "sort"])]);
    const r1 = connect(graph, "A", "box", nextId);
    expectOk(r1);
    graph = r1.graph;
    const r2 = connect(graph, "B", "box", nextId);
    expectOk(r2);
    graph = r2.graph;
    graph = refreshOutput(graph, "box", nextId);

    const outBefore = cardById(graph, outputOf(graph, "box") as string);
    expect(outBefore?.capacity).toBe(2);
    expect(outBefore?.filled).toBe(2);
    expect(inputsOf(graph, "box").map((l) => l.from)).toEqual(["A", "B"]);

    // Rimuove il passaggio 'join' (indice 0): il box resta con solo 'sort'.
    graph = deleteStep(graph, "box", 0, nextId);

    expect(cardById(graph, "box")?.components).toEqual(["sort"]);
    expect(inputsOf(graph, "box").map((l) => l.from)).toEqual(["A"]);
    const outAfter = cardById(graph, outputOf(graph, "box") as string);
    expect(outAfter?.capacity).toBe(1);
    expect(outAfter?.filled).toBe(1);
  });

  it("enforceCapacity da solo mantiene i collegamenti piu vecchi entro la nuova capienza", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([
      dataset("A"),
      dataset("B"),
      dataset("C"),
      op("box", ["join", "union"]),
    ]);
    for (const id of ["A", "B", "C"]) {
      const r = connect(graph, id, "box", nextId);
      expectOk(r);
      graph = r.graph;
    }
    expect(inputsOf(graph, "box")).toHaveLength(3);
    graph = {
      ...graph,
      cards: {
        ...graph.cards,
        box: { ...(cardById(graph, "box") as ReturnType<typeof op>), components: ["join"] },
      },
    };
    graph = enforceCapacity(graph, "box");
    expect(inputsOf(graph, "box").map((l) => l.from)).toEqual(["A", "B"]);
  });
});

describe("relation: matrice (scenario 8)", () => {
  it("lavorazione su lavorazione -> merge", () => {
    const graph = buildGraph([op("a", ["filter"]), op("b", ["sort"])]);
    expect(relation(graph, "a", "b").relation).toBe("merge");
  });

  it("dataset su lavorazione -> link (se valido)", () => {
    const graph = buildGraph([dataset("d"), op("b", ["filter"])]);
    expect(relation(graph, "d", "b").relation).toBe("link");
  });

  it("lavorazione su dataset -> link-reverse (se valido)", () => {
    const graph = buildGraph([op("b", ["filter"]), dataset("d")]);
    expect(relation(graph, "b", "d").relation).toBe("link-reverse");
  });

  it("box sul proprio output -> displace, motivo ciclo", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box", ["filter"])]);
    const r1 = connect(graph, "A", "box", nextId);
    expectOk(r1);
    graph = refreshOutput(r1.graph, "box", nextId);
    const outId = outputOf(graph, "box") as string;

    const res = relation(graph, "box", outId);
    expect(res.relation).toBe("displace");
    expect(res.displaceReason).toBe(
      "Un box non può agganciarsi a ciò che produce: sarebbe un ciclo infinito",
    );
  });

  it("dataset su dataset -> displace, motivo 'serve una lavorazione'", () => {
    const graph = buildGraph([dataset("x"), dataset("y")]);
    const res = relation(graph, "x", "y");
    expect(res.relation).toBe("displace");
    expect(res.displaceReason).toBe(
      "Due dataset non si fondono: serve una lavorazione, ad esempio un Join",
    );
  });
});
```

### `src/etl-core/__tests__/schema.test.ts`

75 righe

```ts
import { describe, expect, it } from "vitest";
import { dataset, op, buildGraph, testIdGenerator } from "./helpers";
import { connect, refreshOutput } from "../rules/mutations";
import { outputOf } from "../model/graph";
import { schemaOf } from "../schema/schema";
import type { ColumnDef, DatasetParams } from "../model/types";

const COLS_A: ColumnDef[] = [
  { name: "id", type: "integer", values: ["1", "2"] },
  { name: "regione", type: "stringa", values: ["Nord", "Sud"] },
];
const COLS_B: ColumnDef[] = [
  { name: "id", type: "integer", values: ["1", "2"] },
  { name: "importo", type: "numerico", values: ["10", "20"] },
];

function withColumns(id: string, columns: ColumnDef[]) {
  const params0: DatasetParams = { columns };
  return dataset(id, { params0: params0 as unknown as Record<string, unknown> });
}

describe("schemaOf (scenario 14)", () => {
  it("le colonne di una sorgente vengono dai suoi parametri", () => {
    const graph = buildGraph([withColumns("A", COLS_A)]);
    expect(schemaOf(graph, "A")).toEqual(COLS_A);
  });

  it("attraverso due livelli di output (sorgente -> box1 -> box2)", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([withColumns("A", COLS_A), op("box1", ["sort"]), op("box2", ["limit"])]);
    const c1 = connect(graph, "A", "box1", nextId);
    if (!c1.ok) throw new Error("unexpected refusal");
    graph = refreshOutput(c1.graph, "box1", nextId);
    const out1 = outputOf(graph, "box1") as string;
    const c2 = connect(graph, out1, "box2", nextId);
    if (!c2.ok) throw new Error("unexpected refusal");
    graph = refreshOutput(c2.graph, "box2", nextId);
    const out2 = outputOf(graph, "box2") as string;

    expect(schemaOf(graph, out1)).toEqual(COLS_A);
    expect(schemaOf(graph, out2)).toEqual(COLS_A);
  });

  it("una lavorazione con due ingressi vede l'unione delle colonne, senza duplicati", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([
      withColumns("A", COLS_A),
      withColumns("B", COLS_B),
      op("box", ["join"]),
    ]);
    const c1 = connect(graph, "A", "box", nextId);
    if (!c1.ok) throw new Error("unexpected refusal");
    graph = c1.graph;
    const c2 = connect(graph, "B", "box", nextId);
    if (!c2.ok) throw new Error("unexpected refusal");
    graph = c2.graph;

    const cols = schemaOf(graph, "box");
    expect(cols?.map((c) => c.name)).toEqual(["id", "regione", "importo"]);
  });

  it("null se non determinabile (sorgente senza colonne caricate)", () => {
    const graph = buildGraph([dataset("A")]);
    expect(schemaOf(graph, "A")).toBeNull();
  });

  it("null oltre la profondita massima (guardia anti-ciclo)", () => {
    // Non costruibile con un ciclo reale (il dominio lo impedisce): verifica
    // solo che la guardia esista e non generi un loop infinito su un id
    // inesistente passato con profondita elevata.
    const graph = buildGraph([]);
    expect(schemaOf(graph, "assente", 30)).toBeNull();
  });
});
```

### `src/etl-core/__tests__/state.test.ts`

115 righe

```ts
import { describe, expect, it } from "vitest";
import { stepMissing } from "../rules/state";
import { defaultParams } from "../catalog/params";
import type { ComponentId, FilterParams, JoinParams, Params } from "../model/types";

/**
 * Scenario 10: per ogni tipo di operazione, parametri vuoti -> incompleto;
 * compilati -> completo. Copre i tre rami di stepMissing (filter, join,
 * MULTI_DEFS, exportOp, campo semplice) su ogni tipo del catalogo.
 */
describe("stepMissing per ogni tipo di operazione (scenario 10)", () => {
  it("filter: vuoto -> incompleto", () => {
    const par = defaultParams("filter");
    expect(stepMissing("filter", par)).toBe(true);
  });

  it("filter: con colonna e valore -> completo", () => {
    const par: FilterParams = {
      conditions: [
        { column: "regione", op: "=", mode: "list", values: ["Nord"], text: "", sep: "," },
      ],
    };
    expect(stepMissing("filter", par as unknown as Params)).toBe(false);
  });

  it("filter: operatore a più valori con solo un testo residuo -> incompleto", () => {
    const par: FilterParams = {
      conditions: [
        { column: "regione", op: "=", mode: "list", values: [], text: "Nord", sep: "," },
      ],
    };
    expect(stepMissing("filter", par as unknown as Params)).toBe(true);
  });

  it("filter: operatore 'e vuoto' non richiede valore -> completo con sola colonna", () => {
    const par: FilterParams = {
      conditions: [
        { column: "regione", op: "è vuoto", mode: "list", values: [], text: "", sep: "," },
      ],
    };
    expect(stepMissing("filter", par as unknown as Params)).toBe(false);
  });

  it("join: vuoto (chiavi senza colonne) -> incompleto", () => {
    const par = defaultParams("join");
    expect(stepMissing("join", par)).toBe(true);
  });

  it("join: con entrambe le colonne -> completo", () => {
    const par: JoinParams = { type: "inner", keys: [{ left: "id", right: "customer_id" }] };
    expect(stepMissing("join", par as unknown as Params)).toBe(false);
  });

  const multiTypes: ComponentId[] = [
    "cast",
    "rename",
    "fillNa",
    "replaceVal",
    "round",
    "scale",
    "textClean",
    "compute",
    "selectCols",
    "dedup",
    "sort",
    "aggregate",
  ];
  for (const type of multiTypes) {
    it(`${type}: parametri predefiniti (riga vuota) -> incompleto`, () => {
      const par = defaultParams(type);
      expect(stepMissing(type, par)).toBe(true);
    });
  }

  it("cast: colonna e tipo compilati -> completo", () => {
    const par = { items: [{ column: "importo", to: "intero" }] } as unknown as ReturnType<
      typeof defaultParams
    >;
    expect(stepMissing("cast", par)).toBe(false);
  });

  it("aggregate: chiave e misura compilate -> completo", () => {
    const par = {
      groupBy: [{ column: "regione" }],
      measures: [{ column: "importo", fn: "somma", alias: "" }],
    } as unknown as ReturnType<typeof defaultParams>;
    expect(stepMissing("aggregate", par)).toBe(false);
  });

  it("exportOp: senza destinazione -> incompleto", () => {
    const par = defaultParams("exportOp");
    expect(stepMissing("exportOp", par)).toBe(true);
  });

  it("exportOp: con destinazione -> completo", () => {
    const par = { format: "CSV", dest: "output.csv" };
    expect(stepMissing("exportOp", par)).toBe(false);
  });

  const simpleRequiredTypes: ComponentId[] = ["sort", "limit", "sample"];
  for (const type of simpleRequiredTypes) {
    it(`${type}: parametri predefiniti -> completo o incompleto secondo i campi richiesti`, () => {
      const par = defaultParams(type);
      // sort ha 'column' (type:'column', sempre richiesto) vuoto -> incompleto;
      // limit/sample hanno 'n'/'pct' con default non vuoto e req:true -> completo.
      const expected = type === "sort";
      expect(stepMissing(type, par)).toBe(expected);
    });
  }

  it("undefined -> sempre incompleto", () => {
    expect(stepMissing("filter", undefined)).toBe(true);
  });
});
```

### `src/etl-core/catalog/icons.ts`

43 righe

```ts
/**
 * Tracciati SVG (frammenti `<path>`/`<circle>`/...) come stringhe, senza
 * JSX: chi disegna l'icona li avvolge nel proprio `<svg>` (prototipo:
 * `svgTag`, righe 979-981 di docs/prototype/isa-fusion-prototype.html).
 *
 * Porting letterale di ICONS (righe 871-894).
 */
import type { ComponentId } from "../model/types";

export const ICONS: Readonly<Record<ComponentId, string>> = {
  filter: '<path d="M4 4h16l-6 8v6l-4 2v-8z"/>',
  dataset:
    '<ellipse cx="12" cy="5" rx="8" ry="3"/><path d="M4 5v6c0 1.7 3.6 3 8 3s8-1.3 8-3V5"/><path d="M4 11v6c0 1.7 3.6 3 8 3s8-1.3 8-3v-6"/>',
  join: '<circle cx="9" cy="12" r="6.5"/><circle cx="15" cy="12" r="6.5"/>',
  sort: '<path d="M8 9l4-4 4 4"/><path d="M16 15l-4 4-4-4"/>',
  exportOp: '<path d="M14 3h7v7"/><path d="M21 3l-9 9"/><path d="M5 12v7a2 2 0 0 0 2 2h7"/>',
  dedup:
    '<rect x="4" y="4" width="11" height="11" rx="2"/><rect x="9" y="9" width="11" height="11" rx="2"/>',
  limit:
    '<line x1="4" y1="6" x2="20" y2="6"/><line x1="4" y1="11" x2="20" y2="11"/><line x1="4" y1="16" x2="11" y2="16"/><path d="M15 14l3 3 3-3"/>',
  sample:
    '<circle cx="6" cy="6" r="1.8"/><circle cx="12" cy="12" r="1.8"/><circle cx="18" cy="8" r="1.8"/><circle cx="8" cy="18" r="1.8"/><circle cx="17" cy="17" r="1.8"/>',
  selectCols:
    '<rect x="4" y="4" width="4" height="16" rx="1"/><rect x="10" y="4" width="4" height="16" rx="1"/><rect x="16" y="4" width="4" height="16" rx="1"/>',
  compute: '<path d="M9 20c2 0 2-4 3-8s1-8 3-8"/><line x1="7" y1="11" x2="15" y2="11"/>',
  cast: '<path d="M5 8h13l-3-3"/><path d="M19 16H6l3 3"/>',
  round: '<circle cx="12" cy="12" r="8"/><circle cx="12" cy="12" r="2"/>',
  scale:
    '<line x1="6" y1="20" x2="6" y2="14"/><line x1="12" y1="20" x2="12" y2="9"/><line x1="18" y1="20" x2="18" y2="4"/>',
  aggregate: '<path d="M18 5H7l6 7-6 7h11"/>',
  textClean: '<path d="M5 6h14"/><path d="M12 6v13"/>',
  replaceVal:
    '<path d="M4 7h11"/><path d="M12 4l3 3-3 3"/><path d="M20 17H9"/><path d="M12 14l-3 3 3 3"/>',
  splitCol: '<path d="M12 4v16"/><path d="M4 8l4 4-4 4"/><path d="M20 8l-4 4 4 4"/>',
  rename: '<path d="M4 20h4L19 9l-4-4L4 16z"/>',
  fillNa: '<rect x="4" y="4" width="16" height="16" rx="3"/><path d="M8 12h8"/><path d="M12 8v8"/>',
  union:
    '<rect x="5" y="4" width="14" height="6" rx="1.5"/><rect x="5" y="14" width="14" height="6" rx="1.5"/>',
};

/** Icona per una "fetta" vuota di un output parziale (prototipo: `ICONS.empty`, riga 877). */
export const EMPTY_SLOT_ICON = '<path d="M9 7l-5 5 5 5"/><path d="M15 7l5 5-5 5"/>';
```

### `src/etl-core/catalog/operations.ts`

78 righe

```ts
/**
 * Operazioni, etichette e sezioni della cassetta degli strumenti.
 * Porting letterale di META (righe 895-904) e SECTIONS (righe 4657-4663)
 * di docs/prototype/isa-fusion-prototype.html.
 */
import type { ComponentId, OperationType } from "../model/types";

export interface OperationMeta {
  readonly label: string;
}

/** Prototipo: META. Etichetta per ogni componente, incluso 'dataset'. */
export const META: Readonly<Record<ComponentId, OperationMeta>> = {
  filter: { label: "Filtra Righe" },
  dataset: { label: "Vendite 2026" },
  join: { label: "Unisci (Join)" },
  sort: { label: "Ordina" },
  exportOp: { label: "Esporta" },
  dedup: { label: "Rimuovi duplicati" },
  limit: { label: "Limita righe" },
  sample: { label: "Campiona" },
  selectCols: { label: "Seleziona colonne" },
  compute: { label: "Calcola colonna" },
  cast: { label: "Converti tipo" },
  round: { label: "Arrotonda" },
  scale: { label: "Normalizza" },
  aggregate: { label: "Raggruppa" },
  textClean: { label: "Pulisci testo" },
  replaceVal: { label: "Sostituisci valori" },
  splitCol: { label: "Dividi colonna" },
  rename: { label: "Rinomina" },
  fillNa: { label: "Riempi vuoti" },
  union: { label: "Accoda (Union)" },
};

export interface SectionDef {
  readonly id: string;
  readonly name: string;
  readonly items?: readonly OperationType[];
}

/** Prototipo: SECTIONS. La sezione "data" non ha `items`: contiene il dataset. */
export const SECTIONS: readonly SectionDef[] = [
  { id: "data", name: "Dataset" },
  {
    id: "rows",
    name: "Filtra e ordina",
    items: ["filter", "sort", "dedup", "limit", "sample", "selectCols"],
  },
  {
    id: "xform",
    name: "Trasforma dati",
    items: [
      "compute",
      "cast",
      "round",
      "scale",
      "aggregate",
      "textClean",
      "replaceVal",
      "splitCol",
      "rename",
      "fillNa",
    ],
  },
  { id: "merge", name: "Merge e union", items: ["join", "union"] },
  { id: "out", name: "Output", items: ["exportOp"] },
];

/** Operazioni che richiedono più di una tabella in ingresso (prototipo: MERGE_OPS, riga 1611). */
export const MERGE_OPS: readonly OperationType[] = ["join", "union"];

/** Sezione che contiene un dato tipo di operazione, se esiste (derivato da SECTIONS). */
export function sectionOf(type: ComponentId): SectionDef | null {
  if (type === "dataset") return SECTIONS.find((s) => s.id === "data") ?? null;
  return SECTIONS.find((s) => s.items?.includes(type)) ?? null;
}
```

### `src/etl-core/catalog/params.ts`

846 righe

```ts
/**
 * Definizioni dei parametri, valori predefiniti e migrazioni.
 * Porting letterale di PARAM_DEFS, MULTI_DEFS e delle relative costanti
 * (righe 2433-2617 di docs/prototype/isa-fusion-prototype.html), più le
 * migrazioni sparse nelle funzioni `render*` (renderFilter riga 3394,
 * ensureKeys riga 3124).
 */
import type {
  ComponentId,
  FilterCondition,
  FilterParams,
  JoinKey,
  JoinOp,
  JoinParams,
  LogicOp,
  MultiFieldDef,
  MultiListDef,
  MultiOperationDef,
  MultiParams,
  MultiRow,
  OperationType,
  Params,
  SimpleFieldDef,
  ValuesField,
} from "../model/types";

// --- Vocabolari (prototipo: righe 2433-2439, 3142, 3158-3159, 3299-3303) ---

export const MULTI_OPS: readonly string[] = ["=", "≠", "è uno di", "non è uno di", "contiene"];
export const NO_VALUE_OPS: readonly string[] = ["è vuoto", "non è vuoto"];
export const FILTER_OPS: readonly string[] = [
  "=",
  "≠",
  "è uno di",
  "non è uno di",
  "contiene",
  ">",
  "<",
  "≥",
  "≤",
  "è vuoto",
  "non è vuoto",
];

export interface Separator {
  readonly label: string;
  readonly ch: string;
}

export const SEPARATORS: readonly Separator[] = [
  { label: "virgola", ch: "," },
  { label: "punto e virgola", ch: ";" },
  { label: "barra verticale", ch: "|" },
  { label: "a capo", ch: "\n" },
];

export const LIST_OPS: readonly string[] = ["è uno di", "non è uno di"];
export const JOIN_OPS: readonly JoinOp[] = ["=", "≠", "<", "≤", ">", "≥"];
export const JOIN_OP_NAME: readonly string[] = [
  "uguale a",
  "diverso da",
  "minore di",
  "minore o uguale a",
  "maggiore di",
  "maggiore o uguale a",
];

export const LOGIC_OPS: readonly LogicOp[] = ["AND", "OR", "XOR", "NAND", "NOR", "XNOR"];
export const LOGIC_HELP: Readonly<Record<LogicOp, string>> = {
  AND: "entrambe vere",
  OR: "almeno una vera",
  XOR: "una sola delle due vera",
  NAND: "non entrambe vere",
  NOR: "nessuna delle due vera",
  XNOR: "entrambe vere o entrambe false",
};

// --- Costruttori di valore predefinito (prototipo: newCondition, VALUES_DEF) ---

/** Prototipo, riga 2440-2442. */
export function newCondition(): FilterCondition {
  return { column: "", op: "=", mode: "list", values: [], text: "", sep: "," };
}

/** Prototipo, riga 2529 (`VALUES_DEF`). */
export function createValuesField(): ValuesField {
  return { mode: "list", values: [], text: "", sep: "," };
}

/** Prototipo, righe 2530-2533. */
export function valuesText(v: string | ValuesField | undefined): string {
  if (v === undefined) return "";
  if (typeof v === "string") return v;
  return v.mode === "list" ? v.values.join(", ") : v.text;
}

/**
 * Correzione intenzionale rispetto al prototipo (Fase 1.1, vedi
 * src/etl-core/NOTE_DIVERGENZE.md — "una sola fonte di verità per i
 * valori"): un campo a più valori conta solo `values`; `text` è solo un
 * formato di transito verso `values` (vedi `normalizeValuesField`), non
 * un secondo modo di essere "compilato". Nel prototipo (righe 2534-2537)
 * `fieldFilled` considerava compilato anche un `text` non vuoto rimasto
 * dalla modalità manuale.
 */
export function fieldFilled(
  f: { readonly type: string },
  v: string | ValuesField | undefined,
): boolean {
  if (f.type === "values") {
    const vf = v as ValuesField | undefined;
    return !!(vf && vf.values && vf.values.length > 0);
  }
  return !!(v && String(v).trim().length > 0);
}

/**
 * Correzione intenzionale rispetto al prototipo (Fase 1.1): migrazione
 * unica per ogni campo a più valori, oggi sparsa dentro `pickerHtml`
 * (prototipo, righe 2840-2848). Se `mode` è `'manual'` e `text` non è
 * vuoto, `text` viene diviso SOLO sul separatore registrato `sep` (o `,`
 * se assente), come la migrazione del prototipo: il testo era stato
 * scritto con quel separatore esplicito, quindi con `sep` `;` un valore
 * come "Rossi, Mario" resta intero. I token non vuoti vengono aggiunti a
 * `values` senza duplicati, poi `text` diventa `''` e `mode` diventa
 * `'list'`. Altrimenti il campo torna inalterato (mai mutato: restituisce
 * un nuovo oggetto solo se c'è qualcosa da migrare).
 *
 * La divisione su più separatori insieme riguarda solo l'inserimento dal
 * vivo nel selettore di valori: vedi `splitTokens`.
 */
export function normalizeValuesField(v: ValuesField): ValuesField {
  if (v.mode === "list" || !v.text || !v.text.trim()) return v;
  const tokens = v.text
    .split(v.sep || ",")
    .map((t) => t.trim())
    .filter((t) => t.length > 0);
  const values = v.values.slice();
  for (const t of tokens) {
    if (!values.includes(t)) values.push(t);
  }
  return { mode: "list", values, text: "", sep: v.sep };
}

/**
 * Prototipo, riga 2839 (`splitTokens`): divide un testo incollato o
 * scritto dal vivo nel selettore di valori su `,` `;` `|` e a capo, con
 * trim, senza token vuoti e senza duplicati. Solo per l'interfaccia: la
 * migrazione dei testi salvati usa `normalizeValuesField`, che divide
 * solo sul separatore registrato.
 */
export function splitTokens(text: string | null | undefined): string[] {
  const tokens = String(text ?? "")
    .split(/[,;|\n]/)
    .map((t) => t.trim())
    .filter((t) => t.length > 0);
  return Array.from(new Set(tokens));
}

function strField(row: MultiRow, key: string): string {
  const v = row[key];
  return typeof v === "string" ? v : "";
}

// --- PARAM_DEFS (prototipo, righe 2445-2525) --------------------------------

const COLUMN_FIELD = (label = "Colonna"): SimpleFieldDef => ({
  k: "column",
  label,
  type: "column",
  def: "",
});

/**
 * Definizioni a campo semplice, una per tipo di operazione (più `dataset`).
 * `filter` è `'custom'`: i suoi parametri (`FilterParams`) non seguono
 * questo schema generico, esattamente come nel prototipo.
 */
export const PARAM_DEFS: Readonly<Record<ComponentId, readonly SimpleFieldDef[] | "custom">> = {
  dataset: [
    {
      k: "source",
      label: "Origine",
      type: "select",
      opts: ["CSV", "Database", "API", "Foglio di calcolo"],
      def: "CSV",
    },
    { k: "path", label: "Percorso o tabella", type: "text", def: "" },
    {
      k: "header",
      label: "Prima riga di intestazione",
      type: "select",
      opts: ["Sì", "No"],
      def: "Sì",
    },
  ],
  filter: "custom",
  join: [
    {
      k: "type",
      label: "Tipo di join",
      type: "select",
      opts: ["inner", "left", "right", "full"],
      def: "inner",
    },
  ],
  sort: [
    COLUMN_FIELD(),
    {
      k: "dir",
      label: "Direzione",
      type: "select",
      opts: ["crescente", "decrescente"],
      def: "crescente",
    },
  ],
  exportOp: [
    {
      k: "format",
      label: "Formato",
      type: "select",
      opts: ["CSV", "XLSX", "Parquet", "Tabella DB"],
      def: "CSV",
    },
    { k: "dest", label: "Destinazione", type: "text", def: "", req: true },
  ],
  dedup: [
    COLUMN_FIELD("Colonna chiave"),
    {
      k: "keep",
      label: "Mantieni",
      type: "select",
      opts: ["la prima", "l’ultima"],
      def: "la prima",
    },
  ],
  limit: [
    { k: "n", label: "Numero di righe", type: "text", def: "100", req: true },
    { k: "from", label: "Dall’", type: "select", opts: ["inizio", "fine"], def: "inizio" },
  ],
  sample: [
    { k: "pct", label: "Percentuale", type: "text", def: "10", req: true },
    { k: "seed", label: "Seme casuale", type: "text", def: "" },
  ],
  selectCols: [
    COLUMN_FIELD("Colonna da tenere"),
    { k: "mode", label: "Modo", type: "select", opts: ["tieni", "escludi"], def: "tieni" },
  ],
  compute: [
    { k: "name", label: "Nuova colonna", type: "text", def: "", req: true },
    { k: "formula", label: "Formula", type: "text", def: "", req: true },
  ],
  cast: [
    COLUMN_FIELD(),
    {
      k: "to",
      label: "Nuovo tipo",
      type: "select",
      opts: ["intero", "decimale", "testo", "data", "booleano"],
      def: "decimale",
    },
  ],
  round: [COLUMN_FIELD(), { k: "decimals", label: "Decimali", type: "text", def: "2", req: true }],
  scale: [
    COLUMN_FIELD(),
    {
      k: "method",
      label: "Metodo",
      type: "select",
      opts: ["min-max", "z-score", "percentuale"],
      def: "min-max",
    },
  ],
  aggregate: [
    { k: "groupBy", label: "Raggruppa per", type: "column", def: "" },
    { k: "measure", label: "Misura", type: "column", def: "" },
    {
      k: "fn",
      label: "Funzione",
      type: "select",
      opts: ["somma", "media", "conteggio", "minimo", "massimo"],
      def: "somma",
    },
  ],
  textClean: [
    COLUMN_FIELD(),
    {
      k: "action",
      label: "Operazione",
      type: "select",
      opts: ["rimuovi spazi", "maiuscole", "minuscole", "iniziali maiuscole"],
      def: "rimuovi spazi",
    },
  ],
  replaceVal: [
    COLUMN_FIELD(),
    { k: "find", label: "Cerca", type: "text", def: "", req: true },
    { k: "with", label: "Sostituisci con", type: "text", def: "" },
  ],
  splitCol: [
    COLUMN_FIELD(),
    {
      k: "sep",
      label: "Separatore",
      type: "select",
      opts: [",", ";", "|", "spazio", "-"],
      def: ",",
    },
  ],
  rename: [COLUMN_FIELD(), { k: "newName", label: "Nuovo nome", type: "text", def: "", req: true }],
  fillNa: [
    COLUMN_FIELD(),
    { k: "value", label: "Valore di riempimento", type: "text", def: "", req: true },
  ],
  union: [
    { k: "mode", label: "Righe", type: "select", opts: ["tutte", "senza duplicati"], def: "tutte" },
    {
      k: "align",
      label: "Allineamento colonne",
      type: "select",
      opts: ["per nome", "per posizione"],
      def: "per nome",
    },
  ],
};

// --- MULTI_DEFS (prototipo, righe 2538-2582) --------------------------------

const COLF = (label?: string): MultiFieldDef => ({
  k: "column",
  label: label ?? "Colonna",
  type: "column",
  def: "",
});
const VALUES_FIELD_DEF = (label: string, req = true): MultiFieldDef => ({
  k: "find",
  label,
  type: "values",
  def: createValuesField,
  req,
});

export const MULTI_DEFS: Readonly<Partial<Record<OperationType, MultiOperationDef>>> = {
  cast: {
    lists: [
      {
        key: "items",
        label: "Colonne da convertire",
        noun: "Conversione",
        add: "Aggiungi conversione",
        fields: [
          COLF(),
          {
            k: "to",
            label: "Nuovo tipo",
            type: "select",
            opts: ["intero", "decimale", "testo", "data", "booleano"],
            def: "decimale",
          },
        ],
        sum: (r) =>
          strField(r, "column") ? `${strField(r, "column")} → ${strField(r, "to")}` : null,
      },
    ],
  },
  rename: {
    lists: [
      {
        key: "items",
        label: "Colonne da rinominare",
        noun: "Rinomina",
        add: "Aggiungi colonna",
        fields: [COLF(), { k: "newName", label: "Nuovo nome", type: "text", def: "", req: true }],
        sum: (r) =>
          strField(r, "column")
            ? `${strField(r, "column")} → ${strField(r, "newName") || "…"}`
            : null,
      },
    ],
  },
  fillNa: {
    lists: [
      {
        key: "items",
        label: "Colonne da riempire",
        noun: "Riempimento",
        add: "Aggiungi colonna",
        fields: [
          COLF(),
          { k: "value", label: "Valore di riempimento", type: "value", def: "", req: true },
        ],
        sum: (r) =>
          strField(r, "column")
            ? `${strField(r, "column")} = ${strField(r, "value") || "…"}`
            : null,
      },
    ],
  },
  replaceVal: {
    lists: [
      {
        key: "items",
        label: "Sostituzioni",
        noun: "Sostituzione",
        add: "Aggiungi sostituzione",
        fields: [
          COLF(),
          {
            k: "match",
            label: "Quando il valore",
            type: "select",
            opts: [
              "è uguale a",
              "è diverso da",
              "contiene",
              "inizia con",
              "finisce con",
              "corrisponde all’espressione",
            ],
            def: "è uguale a",
          },
          VALUES_FIELD_DEF("Valori da cercare"),
          { k: "with", label: "Sostituisci con", type: "value", def: "" },
        ],
        sum: (r) => {
          const column = strField(r, "column");
          if (!column) return null;
          const match = strField(r, "match") || "è uguale a";
          const find = valuesText(r["find"] as string | ValuesField | undefined) || "…";
          const withVal = strField(r, "with") || "∅";
          return `${column} ${match} ${find} → ${withVal}`;
        },
      },
    ],
  },
  round: {
    lists: [
      {
        key: "items",
        label: "Colonne da arrotondare",
        noun: "Arrotondamento",
        add: "Aggiungi colonna",
        fields: [COLF(), { k: "decimals", label: "Decimali", type: "text", def: "2", req: true }],
        sum: (r) =>
          strField(r, "column")
            ? `${strField(r, "column")} · ${strField(r, "decimals")} decimali`
            : null,
      },
    ],
  },
  scale: {
    lists: [
      {
        key: "items",
        label: "Colonne da normalizzare",
        noun: "Normalizzazione",
        add: "Aggiungi colonna",
        fields: [
          COLF(),
          {
            k: "method",
            label: "Metodo",
            type: "select",
            opts: ["min-max", "z-score", "percentuale"],
            def: "min-max",
          },
        ],
        sum: (r) =>
          strField(r, "column") ? `${strField(r, "column")} · ${strField(r, "method")}` : null,
      },
    ],
  },
  textClean: {
    lists: [
      {
        key: "items",
        label: "Colonne da pulire",
        noun: "Pulizia",
        add: "Aggiungi colonna",
        fields: [
          COLF(),
          {
            k: "action",
            label: "Operazione",
            type: "select",
            opts: ["rimuovi spazi", "maiuscole", "minuscole", "iniziali maiuscole"],
            def: "rimuovi spazi",
          },
        ],
        sum: (r) =>
          strField(r, "column") ? `${strField(r, "column")} · ${strField(r, "action")}` : null,
      },
    ],
  },
  compute: {
    lists: [
      {
        key: "items",
        label: "Colonne calcolate",
        noun: "Colonna",
        add: "Aggiungi colonna calcolata",
        fields: [
          { k: "name", label: "Nuova colonna", type: "text", def: "", req: true },
          { k: "formula", label: "Formula", type: "text", def: "", req: true },
        ],
        sum: (r) =>
          strField(r, "name") ? `${strField(r, "name")} = ${strField(r, "formula") || "…"}` : null,
      },
    ],
  },
  selectCols: {
    globals: [
      { k: "mode", label: "Modo", type: "select", opts: ["tieni", "escludi"], def: "tieni" },
    ],
    lists: [
      {
        key: "items",
        label: "Colonne",
        noun: "Colonna",
        add: "Aggiungi colonna",
        fields: [COLF()],
        sum: (r) => strField(r, "column") || null,
      },
    ],
  },
  dedup: {
    globals: [
      {
        k: "keep",
        label: "Mantieni",
        type: "select",
        opts: ["la prima", "l’ultima"],
        def: "la prima",
      },
    ],
    lists: [
      {
        key: "items",
        label: "Colonne chiave",
        noun: "Chiave",
        add: "Aggiungi chiave",
        fields: [
          COLF(),
          {
            k: "cmp",
            label: "Confronto",
            type: "select",
            opts: ["esatto", "ignora maiuscole", "ignora spazi", "ignora maiuscole e spazi"],
            def: "esatto",
          },
        ],
        sum: (r) => {
          const column = strField(r, "column");
          if (!column) return null;
          const cmp = strField(r, "cmp");
          return cmp && cmp !== "esatto" ? `${column} · ${cmp}` : column;
        },
        note: "Due righe sono duplicate quando coincidono su tutte le chiavi.",
      },
    ],
  },
  sort: {
    lists: [
      {
        key: "items",
        label: "Criteri di ordinamento",
        noun: "Criterio",
        add: "Aggiungi criterio",
        fields: [
          COLF(),
          {
            k: "dir",
            label: "Direzione",
            type: "select",
            opts: ["crescente", "decrescente"],
            def: "crescente",
          },
        ],
        sum: (r) => {
          const column = strField(r, "column");
          if (!column) return null;
          return column + (strField(r, "dir") === "crescente" ? " ↑" : " ↓");
        },
        note: "Il primo criterio è il principale; i successivi decidono a parità del precedente.",
      },
    ],
  },
  aggregate: {
    lists: [
      {
        key: "groupBy",
        label: "Raggruppa per",
        noun: "Chiave",
        add: "Aggiungi chiave",
        fields: [COLF()],
        sum: (r) => strField(r, "column") || null,
      },
      {
        key: "measures",
        label: "Misure",
        noun: "Misura",
        add: "Aggiungi misura",
        fields: [
          COLF("Colonna"),
          {
            k: "fn",
            label: "Funzione",
            type: "select",
            opts: ["somma", "media", "conteggio", "minimo", "massimo"],
            def: "somma",
          },
          { k: "alias", label: "Nome del risultato", type: "text", def: "" },
        ],
        sum: (r) => {
          const column = strField(r, "column");
          if (!column) return null;
          const fn = strField(r, "fn") || "somma";
          const alias = strField(r, "alias");
          return `${fn}(${column})${alias ? ` → ${alias}` : ""}`;
        },
      },
    ],
  },
};

// --- Valori predefiniti e migrazioni (prototipo, righe 2584-2617, 3124-3140, 3394-3400) ---

function blankRow(list: MultiListDef): MultiRow {
  const row: MultiRow = {};
  for (const f of list.fields) {
    row[f.k] = typeof f.def === "function" ? f.def() : f.def;
  }
  return row;
}

/** Prototipo, righe 2585-2602: migra il vecchio formato a voce singola in una lista di una riga. */
export function ensureMulti(type: OperationType, par: Params): MultiParams {
  const md = MULTI_DEFS[type];
  if (!md) return par as MultiParams;
  const next: MultiParams = { ...(par as MultiParams) };
  for (const f of md.globals ?? []) {
    if (next[f.k] === undefined) next[f.k] = f.def;
  }
  for (const list of md.lists) {
    const existing = next[list.key];
    if (!Array.isArray(existing)) {
      const row = blankRow(list);
      for (const f of list.fields) {
        const legacy = next[f.k];
        if (legacy !== undefined && legacy !== "") row[f.k] = legacy as string;
      }
      next[list.key] = [row];
    } else {
      next[list.key] = existing.map((row) => {
        const migrated: MultiRow = { ...row };
        for (const f of list.fields) {
          if (f.type === "values") {
            const v = migrated[f.k];
            const asField: ValuesField =
              v === undefined || typeof v !== "object"
                ? v
                  ? { ...createValuesField(), mode: "manual", text: String(v) }
                  : createValuesField()
                : v;
            migrated[f.k] = normalizeValuesField(asField);
          }
        }
        return migrated;
      });
    }
  }
  return next;
}

/**
 * Prototipo, righe 2604-2612. Correzione intenzionale rispetto al
 * prototipo (Fase 1.1): produce direttamente il formato attuale — il
 * filtro senza il campo `logic` (superato, mai stato lì fin dall'inizio
 * in questo dominio: non ha senso generarlo solo per poi migrarlo), il
 * join con le chiavi già passate da `ensureKeys` (op `'='`, `lmode`/
 * `rmode` `'col'`, `rlist` vuota, `lval`/`rval` vuoti).
 */
export function defaultParams(type: ComponentId): Params {
  if (type === "filter") {
    const params: FilterParams = { conditions: [newCondition()] };
    return params as unknown as Params;
  }
  if (type === "join") {
    const params: JoinParams = {
      type: "inner",
      keys: ensureKeys({ type: "inner", keys: [{ left: "", right: "" }] }),
    };
    return params as unknown as Params;
  }
  if (type !== "dataset" && MULTI_DEFS[type]) return ensureMulti(type, {});
  const defs = PARAM_DEFS[type];
  const out: Record<string, string> = {};
  if (Array.isArray(defs)) {
    for (const f of defs) out[f.k] = f.def;
  }
  return out;
}

/** Prototipo, righe 2613-2617: assicura che `card.params` abbia una voce per componente. */
export function ensureParamsFor(
  components: readonly ComponentId[],
  params: readonly Params[],
): Params[] {
  const next = params.slice();
  while (next.length < components.length) {
    next.push(defaultParams(components[next.length] as ComponentId));
  }
  return next;
}

/**
 * Prototipo, righe 3124-3140. Normalizza anche `rlist` (Fase 1.1: una
 * sola fonte di verità per i valori, vedi `normalizeValuesField`).
 */
export function ensureKeys(par: JoinParams): JoinKey[] {
  let keys = par.keys;
  if (!keys) {
    keys =
      par.leftKey || par.rightKey
        ? [{ left: par.leftKey ?? "", right: par.rightKey ?? "" }]
        : [{ left: "", right: "" }];
  }
  return keys.map((k) => ({
    ...k,
    op: k.op ?? "=",
    lmode: k.lmode ?? "col",
    rmode: k.rmode ?? "col",
    rlist: normalizeValuesField(
      k.rlist && typeof k.rlist === "object" ? k.rlist : createValuesField(),
    ),
    lval: k.lval ?? "",
    rval: k.rval ?? "",
  }));
}

/**
 * Prototipo, righe 3396-3400 (dentro `renderFilter`): il vecchio selettore
 * globale E/O diventa il connettore di ogni condizione dalla seconda in
 * poi. Applica anche (Fase 1.1, "una sola fonte di verità per i valori")
 * la migrazione testo → valori di `normalizeValuesField` a ogni
 * condizione, oggi sparsa dentro `pickerHtml` (prototipo, righe 2840-2848).
 */
export function migrateFilterLogic(par: FilterParams): FilterParams {
  const baseConditions = par.conditions ?? [newCondition()];
  const withLogic = par.logic
    ? baseConditions.map((c, i) =>
        i > 0 && !c.conn
          ? { ...c, conn: par.logic === "O" ? ("OR" as const) : ("AND" as const) }
          : c,
      )
    : baseConditions;
  const conditions = withLogic.map((c) => {
    const normalized = normalizeValuesField(c);
    return normalized === c
      ? c
      : {
          ...c,
          mode: normalized.mode,
          values: normalized.values,
          text: normalized.text,
          sep: normalized.sep,
        };
  });
  const { logic, ...rest } = par;
  void logic;
  return { ...rest, conditions };
}

// --- Riassunti (prototipo, righe 3109-3121, 3143-3194) ----------------------

/** Prototipo, righe 3143-3147: il testo di un lato di una chiave di join. */
export function sideText(
  k: JoinKey,
  side: "l" | "r",
  columnDef: (name: string) => { values: readonly string[] } | null,
): string {
  if (side === "l") return k.lmode === "val" ? (k.lval ? `“${k.lval}”` : "") : k.left;
  if (k.rmode === "val") return k.rval ? `“${k.rval}”` : "";
  if (k.rmode === "list") {
    const t = valuesText(k.rlist);
    return t ? `(${t})` : "";
  }
  return k.right;
  // `columnDef` è accettato per parità di firma con il prototipo (usato dal
  // chiamante per calcolare il dominio proposto), non serve qui.
  void columnDef;
}

/** Prototipo, righe 3149-3153. */
export function keyComplete(k: JoinKey): boolean {
  const l = k.lmode === "val" ? !!(k.lval && String(k.lval).trim()) : !!k.left;
  const r =
    k.rmode === "val"
      ? !!(k.rval && String(k.rval).trim())
      : k.rmode === "list"
        ? fieldFilled({ type: "values" }, k.rlist)
        : !!k.right;
  return l && r;
}

/** Prototipo, righe 3190-3194. */
export function summarizeKey(k: JoinKey): string | null {
  const l = k.lmode === "val" ? (k.lval ? `“${k.lval}”` : "") : k.left;
  const r =
    k.rmode === "val"
      ? k.rval
        ? `“${k.rval}”`
        : ""
      : k.rmode === "list"
        ? valuesText(k.rlist)
          ? `(${valuesText(k.rlist)})`
          : ""
        : k.right;
  if (!l && !r) return null;
  return `${l || "…"} ${k.op ?? "="} ${r || "…"}`;
}

/**
 * Prototipo, righe 3109-3121. Correzione intenzionale rispetto al
 * prototipo (Fase 1.1, vedi src/etl-core/NOTE_DIVERGENZE.md — "una sola
 * fonte di verità per i valori"): per gli operatori in `MULTI_OPS` il
 * riassunto usa sempre `values` (non `text`, e non serve più sapere se
 * la colonna ha un dominio noto: il parametro `columnHasValues` del
 * prototipo è stato rimosso). Per gli altri operatori usa `text`.
 */
export function summarizeCond(c: FilterCondition): string | null {
  if (!c.column) return null;
  if (NO_VALUE_OPS.includes(c.op)) return `${c.column} ${c.op}`;
  const v = MULTI_OPS.includes(c.op) ? c.values.join(", ") : c.text;
  if (!v) return `${c.column} ${c.op} …`;
  return `${c.column} ${c.op} ${v}`;
}

/**
 * Prototipo, righe 3219-3222: nessuna condizione di uguaglianza colonna =
 * colonna significa un confronto incrociato, potenzialmente molto lento.
 */
export function hasEquiJoinCondition(keys: readonly JoinKey[]): boolean {
  return keys.some((k) => k.lmode === "col" && k.rmode === "col" && (k.op ?? "=") === "=");
}
```

### `src/etl-core/data/csv.ts`

82 righe

```ts
/**
 * Lettura CSV e deduzione dei tipi. Porting letterale delle righe
 * 4672-4708 di docs/prototype/isa-fusion-prototype.html. Lavora su una
 * stringa già letta, non su `File`/`FileReader` (che non esistono in
 * Node).
 */
import type { ColumnDef, ColumnType } from "../model/types";

export interface ParsedCsv {
  readonly columns: readonly ColumnDef[];
  readonly rows: number;
}

const CANDIDATE_DELIMITERS = [",", ";", "\t", "|"];

function parseLine(line: string, delim: string): string[] {
  const out: string[] = [];
  let cur = "";
  let quoted = false;
  for (let i = 0; i < line.length; i += 1) {
    const ch = line[i];
    if (quoted) {
      if (ch === '"') {
        if (line[i + 1] === '"') {
          cur += '"';
          i += 1;
        } else {
          quoted = false;
        }
      } else {
        cur += ch;
      }
    } else if (ch === '"') {
      quoted = true;
    } else if (ch === delim) {
      out.push(cur);
      cur = "";
    } else {
      cur += ch;
    }
  }
  out.push(cur);
  return out.map((v) => v.trim());
}

/** Prototipo, righe 4672-4708. `null` se il testo non contiene righe non vuote. */
export function parseCSV(text: string): ParsedCsv | null {
  const lines = text
    .replace(/\r/g, "")
    .split("\n")
    .filter((l) => l.trim().length > 0);
  if (lines.length === 0) return null;
  const head = lines[0];
  if (head === undefined) return null;
  const delim =
    CANDIDATE_DELIMITERS.slice().sort((a, b) => head.split(b).length - head.split(a).length)[0] ??
    ",";

  const header = parseLine(head, delim);
  const rows = lines.slice(1, 1001).map((l) => parseLine(l, delim));

  const columns: ColumnDef[] = header.map((name, ci) => {
    const vals = rows.map((r) => r[ci]).filter((v): v is string => v !== undefined && v !== "");
    const isInt = vals.length > 0 && vals.every((v) => /^-?\d+$/.test(v));
    const isNum = vals.length > 0 && vals.every((v) => /^-?\d+([.,]\d+)?$/.test(v));
    const isDate =
      vals.length > 0 &&
      vals.every((v) => /^\d{4}-\d{2}-\d{2}/.test(v) || /^\d{1,2}\/\d{1,2}\/\d{2,4}$/.test(v));
    const type: ColumnType = isInt ? "integer" : isNum ? "numerico" : isDate ? "data" : "stringa";
    const distinct = Array.from(new Set(vals));
    const values =
      type === "integer" || type === "numerico"
        ? distinct
            .slice()
            .sort((a, b) => parseFloat(a.replace(",", ".")) - parseFloat(b.replace(",", ".")))
        : distinct;
    return { name: name || `colonna_${ci + 1}`, type, values: values.slice(0, 500) };
  });

  return { columns, rows: lines.length - 1 };
}
```

### `src/etl-core/index.ts`

101 righe

```ts
/**
 * Esportazioni pubbliche del dominio ETL (Fase 1). Vedi README.md per la
 * tabella di corrispondenza con le funzioni del prototipo
 * docs/prototype/isa-fusion-prototype.html.
 */

// --- Modello ------------------------------------------------------------
export * from "./model/types";
export {
  createGraph,
  cardById,
  inputsOf,
  outputOf,
  setCard,
  removeCard,
  removeCards,
  addLink,
  filterLinks,
  withLinks,
  patchCard,
  withParamAt,
  createSequentialIdGenerator,
} from "./model/graph";

// --- Catalogo -------------------------------------------------------------
export { ICONS, EMPTY_SLOT_ICON } from "./catalog/icons";
export { META, SECTIONS, MERGE_OPS, sectionOf } from "./catalog/operations";
export type { OperationMeta, SectionDef } from "./catalog/operations";
export {
  PARAM_DEFS,
  MULTI_DEFS,
  MULTI_OPS,
  NO_VALUE_OPS,
  FILTER_OPS,
  SEPARATORS,
  LIST_OPS,
  JOIN_OPS,
  JOIN_OP_NAME,
  LOGIC_OPS,
  LOGIC_HELP,
  newCondition,
  createValuesField,
  normalizeValuesField,
  splitTokens,
  valuesText,
  fieldFilled,
  ensureMulti,
  defaultParams,
  ensureParamsFor,
  ensureKeys,
  migrateFilterLogic,
  sideText,
  keyComplete,
  summarizeKey,
  summarizeCond,
  hasEquiJoinCondition,
} from "./catalog/params";
export type { Separator } from "./catalog/params";

// --- Regole ---------------------------------------------------------------
export { boxCapacity, reaches, linkRefusal, relation, compatiblePair } from "./rules/relations";
export type { Relation, RelationResult } from "./rules/relations";
export {
  connect,
  spawnOutput,
  refreshOutput,
  pruneOutputs,
  enforceCapacity,
  nodesRemovedBy,
  deleteNodes,
  deleteLink,
  mergeBoxes,
  insertable,
  insertOnLink,
  detachStep,
  deleteStep,
  reorderSteps,
  duplicateNodes,
  isPartialOutput,
  defaultPositionFn,
} from "./rules/mutations";
export { stepMissing, nodeState } from "./rules/state";

// --- Logica -----------------------------------------------------------
export {
  groupRuns,
  normalizeGroups,
  groupPair,
  splitAt,
  ungroup,
  addToGroup,
  leftAssoc,
  groupedPreview,
} from "./logic/expressions";
export type { Groupable, GroupRun } from "./logic/expressions";

// --- Schema e CSV -----------------------------------------------------
export { schemaOf } from "./schema/schema";
export { parseCSV } from "./data/csv";
export type { ParsedCsv } from "./data/csv";
```

