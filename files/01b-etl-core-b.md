# 01b-etl-core-b.md

File in questo blocco:

- `src/etl-core/__tests__/multi-columns.test.ts`
- `src/etl-core/__tests__/mutations.test.ts`
- `src/etl-core/__tests__/params.test.ts`
- `src/etl-core/__tests__/relations.test.ts`
- `src/etl-core/__tests__/schema.test.ts`
- `src/etl-core/__tests__/state.test.ts`
- `src/etl-core/catalog/icons.ts`
- `src/etl-core/catalog/operations.ts`

---

### `src/etl-core/__tests__/multi-columns.test.ts`

446 righe

```ts
import { describe, expect, it } from "vitest";
import {
  MULTI_DEFS,
  columnsDomain,
  columnsText,
  createValuesField,
  defaultParams,
  ensureMulti,
  ensureParams,
  flattenRows,
  measureNames,
  stepMissing,
  nodeState,
  valuesOutsideDomain,
} from "..";
import type { ColumnDef, MultiRow, OperationType, Params, ValuesField } from "..";
import { addLink } from "../model/graph";
import { buildGraph, dataset, op } from "./helpers";

/** Le dieci operazioni che ora scelgono più colonne per riga, con una riga nel vecchio formato. */
const MULTI_COLUMN_OPS: { type: OperationType; list: string; legacyRow: MultiRow }[] = [
  { type: "cast", list: "items", legacyRow: { column: "importo", to: "intero" } },
  { type: "round", list: "items", legacyRow: { column: "importo", decimals: "2" } },
  { type: "scale", list: "items", legacyRow: { column: "importo", method: "min-max" } },
  { type: "textClean", list: "items", legacyRow: { column: "nome", action: "maiuscole" } },
  { type: "fillNa", list: "items", legacyRow: { column: "nome", value: "n/d" } },
  {
    type: "replaceVal",
    list: "items",
    legacyRow: {
      column: "regione",
      match: "è uguale a",
      find: { ...createValuesField(), values: ["Nord"] },
      with: "N",
    },
  },
  { type: "dedup", list: "items", legacyRow: { column: "id", cmp: "esatto" } },
  { type: "selectCols", list: "items", legacyRow: { column: "id" } },
  { type: "sort", list: "items", legacyRow: { column: "id", dir: "crescente" } },
  { type: "aggregate", list: "groupBy", legacyRow: { column: "regione" } },
  { type: "aggregate", list: "measures", legacyRow: { column: "importo", fn: "somma", alias: "" } },
];

/** Operazioni che restano a colonna singola. */
const SINGLE_COLUMN_OPS: OperationType[] = ["rename", "splitCol", "compute"];

function legacyParams(type: OperationType, list: string, row: MultiRow): Params {
  const base = ensureMulti(type, {}) as Params;
  return { ...base, [list]: [row] };
}

function deepFreeze<T>(o: T): T {
  if (o && typeof o === "object") {
    Object.freeze(o);
    for (const v of Object.values(o as Record<string, unknown>)) deepFreeze(v);
  }
  return o;
}

describe("migrazione column → columns", () => {
  for (const { type, list, legacyRow } of MULTI_COLUMN_OPS) {
    it(`${type}/${list}: una riga con column diventa columns, column sparisce`, () => {
      const migrated = ensureMulti(type, legacyParams(type, list, legacyRow));
      const row = (migrated[list] as MultiRow[])[0] as MultiRow;
      expect(row["columns"]).toEqual([legacyRow["column"]]);
      expect("column" in row).toBe(false);
      // gli altri campi restano
      for (const [k, v] of Object.entries(legacyRow)) {
        if (k !== "column") expect(row[k]).toEqual(v);
      }
    });

    it(`${type}/${list}: stringa vuota → lista vuota`, () => {
      const migrated = ensureMulti(type, legacyParams(type, list, { ...legacyRow, column: "" }));
      expect((migrated[list] as MultiRow[])[0]?.["columns"]).toEqual([]);
    });

    it(`${type}/${list}: è idempotente`, () => {
      const once = ensureMulti(type, legacyParams(type, list, legacyRow));
      const twice = ensureMulti(type, once as Params);
      expect(twice).toEqual(once);
    });

    it(`${type}/${list}: con columns già presente prevale su column`, () => {
      const migrated = ensureMulti(
        type,
        legacyParams(type, list, { ...legacyRow, columns: ["a", "b"] }),
      );
      expect((migrated[list] as MultiRow[])[0]?.["columns"]).toEqual(["a", "b"]);
    });
  }

  it("il vecchio formato a voce singola (campi al primo livello) diventa una riga con una colonna", () => {
    const migrated = ensureMulti("round", { column: "importo", decimals: "3" });
    expect(migrated["items"]).toEqual([{ columns: ["importo"], decimals: "3" }]);
    const sort = ensureMulti("sort", { column: "id", dir: "decrescente" });
    expect((sort["items"] as MultiRow[])[0]).toMatchObject({
      columns: ["id"],
      dir: "decrescente",
    });
  });

  it("senza dati le righe nuove hanno columns vuoto", () => {
    for (const { type, list } of MULTI_COLUMN_OPS) {
      const rows = defaultParams(type)[list] as MultiRow[];
      expect(rows[0]?.["columns"]).toEqual([]);
    }
  });

  it("le operazioni a colonna singola non cambiano", () => {
    for (const type of SINGLE_COLUMN_OPS) {
      const md = MULTI_DEFS[type];
      if (!md) continue; // splitCol non ha voci multiple
      for (const list of md.lists) {
        expect(list.fields.some((f) => f.type === "columns")).toBe(false);
      }
    }
    const migrated = ensureMulti("rename", { items: [{ column: "a", newName: "b" }] });
    expect(migrated["items"]).toEqual([{ column: "a", newName: "b" }]);
  });

  it("ensureParams applica la migrazione e lascia stare un componente senza voci multiple", () => {
    const m = ensureParams("cast", { items: [{ column: "x", to: "testo" }] });
    expect((m["items"] as MultiRow[])[0]?.["columns"]).toEqual(["x"]);
    const lim = { n: "5", from: "inizio" };
    expect(ensureParams("limit", lim)).toBe(lim);
  });
});

describe("flattenRows", () => {
  const params = {
    items: [
      { columns: ["a", "b", "c"], to: "intero" },
      { columns: [], to: "testo" },
      { columns: ["d"], to: "data" },
    ],
  };

  it("una riga con N colonne equivale a N righe, nell'ordine elencato; le righe vuote si scartano", () => {
    const flat = flattenRows("cast", params);
    expect(flat["items"]?.map((r) => `${r.column}:${String(r["to"])}`)).toEqual([
      "a:intero",
      "b:intero",
      "c:intero",
      "d:data",
    ]);
    // nessun campo `columns` nelle righe espanse
    expect(flat["items"]?.every((r) => !("columns" in r))).toBe(true);
  });

  it("l'ordine delle colonne è significativo", () => {
    const f1 = flattenRows("sort", { items: [{ columns: ["a", "b"], dir: "crescente" }] });
    const f2 = flattenRows("sort", { items: [{ columns: ["b", "a"], dir: "crescente" }] });
    expect(f1["items"]?.map((r) => r.column)).toEqual(["a", "b"]);
    expect(f2["items"]?.map((r) => r.column)).toEqual(["b", "a"]);
  });

  it("una riga con un campo obbligatorio vuoto è incompleta e si scarta", () => {
    const flat = flattenRows("round", {
      items: [
        { columns: ["a"], decimals: "" },
        { columns: ["b"], decimals: "2" },
      ],
    });
    expect(flat["items"]?.map((r) => r.column)).toEqual(["b"]);
  });

  it("raggruppa: chiavi e misure in due liste; l'alias non si applica con più colonne", () => {
    const flat = flattenRows("aggregate", {
      groupBy: [{ columns: ["regione", "area"] }],
      measures: [
        { columns: ["importo"], fn: "somma", alias: "tot" },
        { columns: ["a", "b"], fn: "media", alias: "ignorato" },
      ],
    });
    expect(flat["groupBy"]?.map((r) => r.column)).toEqual(["regione", "area"]);
    expect(flat["measures"]?.map((r) => `${r.column}/${String(r["alias"])}`)).toEqual([
      "importo/tot",
      "a/",
      "b/",
    ]);
  });

  it("accetta anche il vecchio formato e un tipo senza voci multiple", () => {
    expect(flattenRows("round", { items: [{ column: "x", decimals: "1" }] })["items"]).toEqual([
      { decimals: "1", column: "x" },
    ]);
    expect(flattenRows("limit", { n: "1" })).toEqual({});
  });
});

describe("completezza: stepMissing e nodeState", () => {
  it("colonne vuote → incompleto; una o più colonne → completo", () => {
    expect(stepMissing("cast", { items: [{ columns: [], to: "testo" }] })).toBe(true);
    expect(stepMissing("cast", { items: [{ columns: ["a"], to: "testo" }] })).toBe(false);
    expect(stepMissing("cast", { items: [{ columns: ["a", "b"], to: "testo" }] })).toBe(false);
  });

  it("un altro campo obbligatorio vuoto rende la riga incompleta", () => {
    expect(stepMissing("round", { items: [{ columns: ["a"], decimals: "" }] })).toBe(true);
    expect(stepMissing("fillNa", { items: [{ columns: ["a", "b"], value: "" }] })).toBe(true);
  });

  it("basta una riga incompleta; aggregate richiede chiavi e misure", () => {
    expect(
      stepMissing("sort", {
        items: [
          { columns: ["a"], dir: "crescente" },
          { columns: [], dir: "crescente" },
        ],
      }),
    ).toBe(true);
    expect(
      stepMissing("aggregate", {
        groupBy: [{ columns: ["a"] }],
        measures: [{ columns: [], fn: "somma", alias: "" }],
      }),
    ).toBe(true);
    expect(
      stepMissing("aggregate", {
        groupBy: [{ columns: ["a", "b"] }],
        measures: [{ columns: ["c", "d"], fn: "somma", alias: "" }],
      }),
    ).toBe(false);
  });

  it("anche nel vecchio formato", () => {
    expect(stepMissing("cast", { items: [{ column: "a", to: "testo" }] })).toBe(false);
    expect(stepMissing("cast", { items: [{ column: "", to: "testo" }] })).toBe(true);
  });

  it("nodeState avvisa con colonne vuote e tace quando ce ne sono", () => {
    const cols: ColumnDef[] = [{ name: "a", type: "stringa", values: ["x"] }];
    const ds = dataset("ds-1", {
      params0: { source: "CSV", path: "a.csv", header: "Sì", columns: cols } as Params,
    });
    const make = (columns: string[]) => {
      const card = op("op-1", ["cast"], {
        params: [{ items: [{ columns, to: "testo" }] }],
      });
      return addLink(buildGraph([ds, card]), { from: "ds-1", to: "op-1" });
    };
    expect(nodeState(make([]), "op-1")).toBe("Da configurare: Converti tipo");
    expect(nodeState(make(["a"]), "op-1")).toBeNull();
    // «b» non è nei dati in ingresso: la riga è incompleta (Fase 6b.2, Passo 0)
    expect(nodeState(make(["a", "b"]), "op-1")).toBe("Da configurare: Converti tipo");
  });
});

describe("colonne assenti dallo schema in ingresso (Fase 6b.2, Passo 0)", () => {
  const cols: ColumnDef[] = [
    { name: "a", type: "stringa", values: ["x"] },
    { name: "b", type: "stringa", values: ["y"] },
  ];
  const par = (columns: string[]): Params => ({ items: [{ columns, to: "testo" }] });

  it("stepMissing: con lo schema una colonna assente rende la riga incompleta", () => {
    expect(stepMissing("cast", par(["a", "zz"]), cols)).toBe(true);
    expect(stepMissing("cast", par(["a", "b"]), cols)).toBe(false);
  });

  it("stepMissing: senza schema, o con schema vuoto, non cambia nulla", () => {
    expect(stepMissing("cast", par(["zz"]))).toBe(false);
    expect(stepMissing("cast", par(["zz"]), [])).toBe(false);
    expect(stepMissing("cast", par(["zz"]), null)).toBe(false);
  });

  it("nodeState: puntino ambra finché la colonna manca; tolta, il nodo torna pronto", () => {
    const ds = dataset("ds-1", {
      params0: { source: "CSV", path: "a.csv", header: "Sì", columns: cols } as Params,
    });
    const make = (columns: string[]) =>
      addLink(buildGraph([ds, op("op-1", ["cast"], { params: [par(columns)] })]), {
        from: "ds-1",
        to: "op-1",
      });
    expect(nodeState(make(["a", "zz"]), "op-1")).toBe("Da configurare: Converti tipo");
    expect(nodeState(make(["a"]), "op-1")).toBeNull();
  });

  it("nodeState: se la sorgente non ha colonne caricate non c'è avviso", () => {
    const ds = dataset("ds-1", { params0: { source: "CSV", path: "a.csv" } as Params });
    const g = addLink(buildGraph([ds, op("op-1", ["cast"], { params: [par(["zz"])] })]), {
      from: "ds-1",
      to: "op-1",
    });
    expect(nodeState(g, "op-1")).toBeNull();
  });
});

describe("riassunti", () => {
  it("columnsText: 0, 1, 3 e 4 colonne", () => {
    expect(columnsText([])).toBe("");
    expect(columnsText(["a"])).toBe("a");
    expect(columnsText(["a", "b", "c"])).toBe("a, b, c");
    expect(columnsText(["a", "b", "c", "d"])).toBe("a, b +2");
    expect(columnsText(["a", "b", "c", "d", "e"])).toBe("a, b +3");
    expect(columnsText(["a", "b", "c", "d"], 5)).toBe("a, b, c, d");
  });

  const sum = (type: OperationType, list: string, row: MultiRow) =>
    MULTI_DEFS[type]?.lists.find((l) => l.key === list)?.sum(row);

  it("ogni operazione riassume l'elenco di colonne; senza colonne nessun riassunto", () => {
    expect(sum("cast", "items", { columns: ["importo", "quantita"], to: "intero" })).toBe(
      "importo, quantita → intero",
    );
    expect(sum("round", "items", { columns: ["a", "b"], decimals: "2" })).toBe("a, b · 2 decimali");
    expect(sum("scale", "items", { columns: ["a"], method: "z-score" })).toBe("a · z-score");
    expect(sum("textClean", "items", { columns: ["a", "b"], action: "minuscole" })).toBe(
      "a, b · minuscole",
    );
    expect(sum("fillNa", "items", { columns: ["a", "b"], value: "0" })).toBe("a, b = 0");
    expect(sum("selectCols", "items", { columns: ["a", "b", "c", "d"] })).toBe("a, b +2");
    expect(sum("dedup", "items", { columns: ["a", "b"], cmp: "ignora spazi" })).toBe(
      "a, b · ignora spazi",
    );
    expect(sum("dedup", "items", { columns: ["a"], cmp: "esatto" })).toBe("a");
    expect(sum("sort", "items", { columns: ["a", "b"], dir: "decrescente" })).toBe("a, b ↓");
    expect(sum("aggregate", "groupBy", { columns: ["a", "b"] })).toBe("a, b");
    expect(
      sum("replaceVal", "items", {
        columns: ["a", "b"],
        match: "contiene",
        find: { ...createValuesField(), values: ["x"] } as ValuesField,
        with: "y",
      }),
    ).toBe("a, b contiene x → y");
    for (const { type, list } of MULTI_COLUMN_OPS) {
      expect(sum(type, list, { columns: [] }), `${type}/${list}`).toBeNull();
    }
  });

  it("le misure: con una colonna l'alias si vede, con più no", () => {
    expect(sum("aggregate", "measures", { columns: ["a"], fn: "somma", alias: "tot" })).toBe(
      "somma(a) → tot",
    );
    expect(sum("aggregate", "measures", { columns: ["a", "b"], fn: "media", alias: "tot" })).toBe(
      "media(a, b)",
    );
  });
});

describe("measureNames", () => {
  it("una colonna: alias oppure funzione_colonna", () => {
    expect(measureNames({ columns: ["importo"], fn: "somma", alias: "totale" })).toEqual([
      "totale",
    ]);
    expect(measureNames({ columns: ["importo"], fn: "somma", alias: "" })).toEqual([
      "somma_importo",
    ]);
  });

  it("più colonne: sempre funzione_colonna, alias ignorato", () => {
    expect(measureNames({ columns: ["a", "b"], fn: "media", alias: "x" })).toEqual([
      "media_a",
      "media_b",
    ]);
  });

  it("nessuna colonna: nessun nome", () => {
    expect(measureNames({ columns: [], fn: "somma", alias: "x" })).toEqual([]);
  });
});

describe("domini di valori", () => {
  const schema: ColumnDef[] = [
    { name: "regione", type: "stringa", values: ["Nord", "Sud", "Centro"] },
    { name: "zona", type: "stringa", values: ["Sud", "Isole", "Nord"] },
    { name: "vuota", type: "stringa", values: [] },
  ];

  it("l'unione nell'ordine delle colonne elencate, senza duplicati", () => {
    expect(columnsDomain(schema, ["regione", "zona"])).toEqual(["Nord", "Sud", "Centro", "Isole"]);
    expect(columnsDomain(schema, ["zona", "regione"])).toEqual(["Sud", "Isole", "Nord", "Centro"]);
    expect(columnsDomain(schema, ["regione", "regione"])).toEqual(["Nord", "Sud", "Centro"]);
  });

  it("colonne sconosciute o senza valori si saltano; nessuna colonna → vuoto", () => {
    expect(columnsDomain(schema, ["x", "vuota"])).toEqual([]);
    expect(columnsDomain(schema, [])).toEqual([]);
  });

  it("tetto di 500 valori", () => {
    const big: ColumnDef[] = [
      { name: "a", type: "stringa", values: Array.from({ length: 400 }, (_, i) => `a${i}`) },
      { name: "b", type: "stringa", values: Array.from({ length: 400 }, (_, i) => `b${i}`) },
    ];
    const d = columnsDomain(big, ["a", "b"]);
    expect(d).toHaveLength(500);
    expect(d[399]).toBe("a399");
    expect(d[400]).toBe("b0");
  });

  it("valuesOutsideDomain: i valori scelti non presenti", () => {
    expect(valuesOutsideDomain(["Nord", "Mare", "Sud", "Cielo"], ["Nord", "Sud"])).toEqual([
      "Mare",
      "Cielo",
    ]);
    expect(valuesOutsideDomain([], ["Nord"])).toEqual([]);
    expect(valuesOutsideDomain(["x"], [])).toEqual(["x"]);
  });

  it("cambiare le colonne di una riga NON azzera i valori già scelti", () => {
    const find = { ...createValuesField(), values: ["Nord", "Mare"] };
    const before = ensureMulti("replaceVal", {
      items: [{ columns: ["regione"], match: "è uguale a", find, with: "N" }],
    });
    const row = (before["items"] as MultiRow[])[0] as MultiRow;
    const changed = ensureMulti("replaceVal", { items: [{ ...row, columns: ["zona"] }] });
    const kept = ((changed["items"] as MultiRow[])[0] as MultiRow)["find"] as ValuesField;
    expect(kept.values).toEqual(["Nord", "Mare"]);
    expect(valuesOutsideDomain(kept.values, columnsDomain(schema, ["zona"]))).toEqual(["Mare"]);
  });
});

describe("Riempi vuoti con più colonne", () => {
  it("il valore è unico per la riga (un solo campo value)", () => {
    const md = MULTI_DEFS.fillNa?.lists[0];
    expect(md?.fields.filter((f) => f.k === "value")).toHaveLength(1);
    const flat = flattenRows("fillNa", { items: [{ columns: ["a", "b"], value: "0" }] });
    expect(flat["items"]?.map((r) => String(r["value"]))).toEqual(["0", "0"]);
  });
});

describe("nessuna funzione modifica gli input", () => {
  it("ensureMulti, ensureParams, flattenRows, stepMissing, measureNames sui dati congelati", () => {
    for (const { type, list, legacyRow } of MULTI_COLUMN_OPS) {
      const params = deepFreeze(legacyParams(type, list, legacyRow));
      const copy = JSON.stringify(params);
      expect(() => ensureMulti(type, params)).not.toThrow();
      expect(() => ensureParams(type, params)).not.toThrow();
      expect(() => flattenRows(type, params)).not.toThrow();
      expect(() => stepMissing(type, params)).not.toThrow();
      expect(JSON.stringify(params)).toBe(copy);
    }
    const row = deepFreeze({ columns: ["a", "b"], fn: "somma", alias: "x" });
    expect(() => measureNames(row)).not.toThrow();
    const cols = deepFreeze(["a", "b"]);
    expect(() => columnsText(cols)).not.toThrow();
    const schema = deepFreeze([{ name: "a", type: "stringa", values: ["x"] }] as ColumnDef[]);
    expect(() => columnsDomain(schema, cols)).not.toThrow();
    expect(() => valuesOutsideDomain(deepFreeze(["x"]), deepFreeze(["y"]))).not.toThrow();
  });
});
```

### `src/etl-core/__tests__/mutations.test.ts`

222 righe

```ts
import { describe, expect, it } from "vitest";
import { dataset, op, buildGraph, testIdGenerator } from "./helpers";
import {
  connect,
  refreshOutput,
  mergeBoxes,
  deleteNodes,
  nodesRemovedBy,
  insertable,
  insertOnLink,
  duplicateNodes,
} from "../rules/mutations";
import { cardById, outputOf, inputsOf } from "../model/graph";
import type { Graph } from "../model/types";

function expectOk(result: { ok: boolean }): asserts result is { ok: true; graph: Graph } {
  expect(result.ok).toBe(true);
}

describe("output parziale (scenario 3)", () => {
  it("dopo la prima tabella su un join capacity=2 filled=1; completo dopo la seconda", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), dataset("B"), op("box", ["join"])]);

    const r1 = connect(graph, "A", "box", nextId);
    expectOk(r1);
    graph = refreshOutput(r1.graph, "box", nextId);
    const outId = outputOf(graph, "box") as string;
    let out = cardById(graph, outId);
    expect(out?.capacity).toBe(2);
    expect(out?.filled).toBe(1);
    expect(out && (out.capacity ?? 0) > 1 && (out.filled ?? 0) < (out.capacity ?? 0)).toBe(true);

    const r2 = connect(graph, "B", "box", nextId);
    expectOk(r2);
    graph = refreshOutput(r2.graph, "box", nextId);
    out = cardById(graph, outId);
    expect(out?.capacity).toBe(2);
    expect(out?.filled).toBe(2);
  });
});

describe("mergeBoxes (scenario 4)", () => {
  it("il box risultante ha entrambi gli output, nessun collegamento da un proprio output verso se stesso", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([
      dataset("A"),
      dataset("B"),
      op("box1", ["filter"]),
      op("box2", ["sort"]),
    ]);

    const c1 = connect(graph, "A", "box1", nextId);
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box1", nextId);
    const c2 = connect(graph, "B", "box2", nextId);
    expectOk(c2);
    graph = refreshOutput(c2.graph, "box2", nextId);

    const out1Before = outputOf(graph, "box1") as string;
    const out2Before = outputOf(graph, "box2") as string;

    const merged = mergeBoxes(graph, "box2", "box1", nextId);
    expectOk(merged);
    graph = merged.graph;

    expect(cardById(graph, "box1")?.components).toEqual(["filter", "sort"]);
    expect(cardById(graph, "box2")).toBeUndefined();

    const producedByBox1 = graph.links.filter((l) => l.from === "box1").map((l) => l.to);
    expect(new Set(producedByBox1)).toEqual(new Set([out1Before, out2Before]));
    expect(graph.links.some((l) => l.from === "box1" && l.to === "box1")).toBe(false);
    // Nessun collegamento da un proprio output verso il box stesso.
    expect(graph.links.some((l) => l.from === out1Before && l.to === "box1")).toBe(false);
    expect(graph.links.some((l) => l.from === out2Before && l.to === "box1")).toBe(false);
  });

  it("il nome diventa 'Combined Box' alla prima fusione tra due box semplici", () => {
    const nextId = testIdGenerator("n");
    const graph = buildGraph([op("a", ["filter"]), op("b", ["sort"])]);
    const merged = mergeBoxes(graph, "b", "a", nextId);
    expectOk(merged);
    expect(cardById(merged.graph, "a")?.name).toBe("Combined Box");
  });
});

describe("eliminazione di un box (scenario 6)", () => {
  it("il suo output e tutto cio che dipendeva solo da lui sparisce; nodesRemovedBy lo prevede", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box1", ["filter"]), op("box2", ["sort"])]);
    const c1 = connect(graph, "A", "box1", nextId);
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box1", nextId);
    const out1 = outputOf(graph, "box1") as string;
    const c2 = connect(graph, out1, "box2", nextId);
    expectOk(c2);
    graph = refreshOutput(c2.graph, "box2", nextId);
    const out2 = outputOf(graph, "box2") as string;

    // nodesRemovedBy segue solo la cascata degli OUTPUT (prototipo, righe
    // 4432-4453: il ciclo filtra `c[id].isOutput`): `box2` non è un
    // output, quindi resta — orfano, senza ingressi — anche se il suo
    // unico input (`out1`) sparisce con `box1`.
    const removed = nodesRemovedBy(graph, "box1");
    expect(removed).toEqual(new Set(["box1", out1, out2]));

    graph = deleteNodes(graph, "box1", nextId);
    expect(cardById(graph, "box1")).toBeUndefined();
    expect(cardById(graph, out1)).toBeUndefined();
    expect(cardById(graph, out2)).toBeUndefined();
    expect(cardById(graph, "box2")).toBeDefined();
    expect(cardById(graph, "A")).toBeDefined();
    expect(graph.links).toHaveLength(0);
  });
});

describe("insertOnLink (scenario 7)", () => {
  it("dataset -> box diventa dataset -> X -> output di X -> box", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box", ["sort"]), op("X", ["filter"])]);
    const c1 = connect(graph, "A", "box", nextId);
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box", nextId);
    const linkAtoBox = graph.links.find((l) => l.from === "A" && l.to === "box");
    expect(linkAtoBox).toBeDefined();

    const next = insertOnLink(graph, linkAtoBox!, "X", nextId);
    expect(next).not.toBeNull();
    graph = next as typeof graph;

    expect(graph.links.some((l) => l.from === "A" && l.to === "X")).toBe(true);
    const outX = outputOf(graph, "X") as string;
    expect(outX).toBeTruthy();
    expect(graph.links.some((l) => l.from === outX && l.to === "box")).toBe(true);
    expect(graph.links.some((l) => l.from === "A" && l.to === "box")).toBe(false);
  });

  it("e rifiutato su un collegamento box -> output", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box", ["sort"]), op("X", ["filter"])]);
    const c1 = connect(graph, "A", "box", nextId);
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box", nextId);
    const outId = outputOf(graph, "box") as string;
    const linkBoxToOut = graph.links.find((l) => l.from === "box" && l.to === outId)!;

    expect(insertable(graph, linkBoxToOut, "X")).toBe(false);
    expect(insertOnLink(graph, linkBoxToOut, "X", nextId)).toBeNull();
  });

  it("e rifiutato quando X ha gia collegamenti", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([
      dataset("A"),
      dataset("D2"),
      op("box", ["sort"]),
      op("X", ["filter"]),
      op("Y", ["sort"]),
    ]);
    const c1 = connect(graph, "A", "box", nextId);
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box", nextId);
    const c2 = connect(graph, "D2", "Y", nextId);
    expectOk(c2);
    graph = c2.graph;
    // X non ha collegamenti: inseribile.
    const linkAtoBox = graph.links.find((l) => l.from === "A" && l.to === "box")!;
    expect(insertable(graph, linkAtoBox, "X")).toBe(true);
    // Y ha gia un collegamento (D2 -> Y): non inseribile.
    expect(insertable(graph, linkAtoBox, "Y")).toBe(false);
  });
});

describe("purezza delle funzioni di rules/mutations.ts (scenario 15)", () => {
  it("nessuna funzione modifica il grafo che riceve", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([
      dataset("A"),
      dataset("B"),
      op("box1", ["filter"]),
      op("box2", ["join"]),
      op("box3", ["sort"]),
    ]);
    const c1 = connect(graph, "A", "box1", nextId);
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box1", nextId);
    const c2 = connect(graph, "B", "box2", nextId);
    expectOk(c2);
    graph = c2.graph;

    const snapshot = JSON.parse(JSON.stringify(graph));

    connect(graph, outputOf(graph, "box1") as string, "box2", nextId);
    refreshOutput(graph, "box2", nextId);
    mergeBoxes(graph, "box3", "box2", nextId);
    deleteNodes(graph, "box1", nextId);
    nodesRemovedBy(graph, "box1");
    insertOnLink(graph, graph.links[0]!, "box3", nextId);
    duplicateNodes(graph, ["A", "B"], nextId);

    expect(JSON.parse(JSON.stringify(graph))).toEqual(snapshot);
  });
});

describe("duplicateNodes", () => {
  it("duplica senza collegamenti, escludendo gli output", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box", ["filter"])]);
    const c1 = connect(graph, "A", "box", nextId);
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box", nextId);
    const outId = outputOf(graph, "box") as string;

    const { graph: next, createdIds } = duplicateNodes(graph, ["A", "box", outId], nextId);
    expect(createdIds).toHaveLength(2); // l'output e escluso
    for (const id of createdIds) {
      expect(inputsOf(next, id)).toHaveLength(0);
      expect(next.links.some((l) => l.from === id)).toBe(false);
    }
  });
});
```

### `src/etl-core/__tests__/params.test.ts`

125 righe

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
    expect(rows[0]?.["columns"]).toEqual(["importo"]);
    expect(rows[0]?.["column"]).toBeUndefined();
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

101 righe

```ts
import { describe, expect, it } from "vitest";
import { dataset, op, buildGraph, testIdGenerator } from "./helpers";
import { connect, refreshOutput } from "../rules/mutations";
import { outputOf } from "../model/graph";
import { columnsOutsideSchema, schemaOf } from "../schema/schema";
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

describe("columnsOutsideSchema (Fase 6b.2, Passo 0)", () => {
  it("le colonne assenti dallo schema, nell'ordine elencato e senza ripetizioni", () => {
    expect(columnsOutsideSchema(["id", "x", "regione", "y", "x"], COLS_A)).toEqual(["x", "y"]);
  });

  it("nessuna se tutte sono nello schema", () => {
    expect(columnsOutsideSchema(["regione", "id"], COLS_A)).toEqual([]);
  });

  it("schema vuoto o sconosciuto: non si sa cosa manchi, nessun avviso", () => {
    expect(columnsOutsideSchema(["x"], [])).toEqual([]);
    expect(columnsOutsideSchema(["x"], null)).toEqual([]);
    expect(columnsOutsideSchema(["x"], undefined)).toEqual([]);
  });

  it("distingue le maiuscole come lo schema", () => {
    expect(columnsOutsideSchema(["ID"], COLS_A)).toEqual(["ID"]);
  });

  it("non modifica gli argomenti", () => {
    const cols = ["x", "id"];
    columnsOutsideSchema(cols, COLS_A);
    expect(cols).toEqual(["x", "id"]);
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

