# 01b-etl-core-a.md

File in questo blocco:

- `src/etl-core/NOTE_DIVERGENZE.md`
- `src/etl-core/README.md`
- `src/etl-core/__tests__/csv.test.ts`
- `src/etl-core/__tests__/expressions.test.ts`
- `src/etl-core/__tests__/helpers.ts`
- `src/etl-core/__tests__/mutations.test.ts`
- `src/etl-core/__tests__/params.test.ts`
- `src/etl-core/__tests__/relations.test.ts`
- `src/etl-core/__tests__/schema.test.ts`
- `src/etl-core/__tests__/state.test.ts`
- `src/etl-core/catalog/icons.ts`
- `src/etl-core/catalog/operations.ts`

---

### `src/etl-core/NOTE_DIVERGENZE.md`

60 righe

```md
# Note di divergenza

Comportamenti del prototipo (`docs/prototype/isa-fusion-prototype.html`) che
sembrano un errore o sono ambigui, replicati fedelmente senza correggerli,
come richiesto dal compito.

## 1. Il messaggio di capienza superata dice sempre "due tabelle"

`linkRefusal` (prototipo, riga 1913; porting in `rules/relations.ts`,
funzione `linkRefusal`):

```js
return cap > 1 ? 'Il join ha gia le sue due tabelle' : 'Il box accetta una sola tabella in ingresso';
```

Il testo è cablato su "due tabelle" (e menziona sempre "il join") ogni
volta che `cap > 1`, ma `cap = boxCapacity(box) = 1 + numero di componenti
in MERGE_OPS`. Un box con più di un'operazione di merge (per esempio
`join` + `union`, capienza 3) mostrerebbe lo stesso messaggio "ha già le
sue due tabelle" anche quando la capienza reale è 3 e anche se il box
contiene solo `union` senza alcun `join`. Replicato esattamente
(`rules/relations.ts`, `linkRefusal`), incluso il commento nel codice che
rimanda a questa nota.

## 2. `stepMissing('filter', ...)` ignora la modalità del valore

`stepMissing` (prototipo, riga 1430-1431; porting in `rules/state.ts`):

```js
return !cs.length || cs.some(c => !c.column || (!NO_VALUE_OPS.includes(c.op) &&
  !(c.text && c.text.trim()) && !(c.values && c.values.length)));
```

Una condizione è considerata "completa" se **`text` oppure `values`** non
sono vuoti, indipendentemente da `c.mode` (che decide quale dei due campi
l'interfaccia mostra e usa davvero). Se un utente passa dalla modalità
"lista" a "manuale" e ritorna a "lista" lasciando un vecchio valore in
`text`, la condizione risulta "completa" anche se `values` è vuoto e la
modalità attiva è `list` — un valore residuo, non più mostrato
nell'interfaccia, maschera uno stato realmente incompleto. Replicato
esattamente in `rules/state.ts`, `stepMissing`.

## Non è una divergenza (chiarimento architetturale)

`connect` nel prototipo (riga 1875-1885) **non** controlla i cicli: quel
controllo vive solo in `linkRefusal`, invocata dall'interazione utente
*prima* di chiamare `connect`. Il compito chiede esplicitamente che
`connect` "rifiuti cicli, duplicati e superamento della capienza", quindi
`rules/mutations.ts`'s `connect` unifica le due funzioni — non è una
correzione spontanea di un comportamento ambiguo, ma un requisito
esplicito della specifica (vedi il commento nel codice e la sezione
corrispondente di README.md).

Allo stesso modo, la numerazione di "Output N" e "Combined Box N" nel
prototipo usa contatori globali mutabili che un dominio a funzioni pure
non può avere: qui il numero è derivato dal grafo corrente (quanti
output/box combinati esistono già). Conseguenza necessaria del vincolo
"funzione pura" richiesto dal compito, non un comportamento diverso
deliberato — vedi il commento in testa a `rules/mutations.ts`.
```

### `src/etl-core/README.md`

162 righe

```md
# etl-core — Fase 1: dominio ETL in TypeScript puro

Porting del modello dati e del catalogo delle operazioni del prototipo
[`docs/prototype/isa-fusion-prototype.html`](../../docs/prototype/isa-fusion-prototype.html),
che sostituiranno quelli attuali (`src/lib/etl-workflow.tsx`,
`src/lib/etl-catalog.ts`) in una fase successiva.

Questa fase costruisce **solo il dominio**: tipi, catalogo, regole del
grafo, parametri, espressioni logiche, lettura CSV. Nessuna interfaccia,
nessuna geometria (instradamento e layout sono la Fase 2), nessuno store
React (Fase 3).

## Principio: funzioni pure

Nel prototipo lo stato (`cards`, `linksArr`) è globale e viene modificato
sul posto. Qui ogni operazione riceve un `Graph` e restituisce un nuovo
`Graph` (o un risultato), **senza mai modificare l'input**. Due
conseguenze dirette:

- **Generatore di id iniettabile** (`IdGenerator = () => string`): ogni
  funzione che crea un nodo lo riceve come parametro, per test
  deterministici. Vedi `model/graph.ts`, `createSequentialIdGenerator`.
- **Numerazione derivata dal grafo**: il prototipo usa contatori globali
  mutabili (`outCounter`, `comboCounter`) per nomi come "Output 3" o
  "Combined Box 2". Un dominio a funzioni pure non ha un posto per questo
  stato: il numero è invece calcolato contando gli output/box combinati
  già presenti nel grafo corrente. Nell'uso normale (senza eliminare e
  ricreare più volte gli stessi nodi) il risultato coincide con il
  prototipo. Vedi il commento in testa a `rules/mutations.ts`.
- **`connect` unifica cicli, duplicati e capienza**: nel prototipo il
  controllo dei cicli viveva solo in `linkRefusal`, invocata
  dall'interazione UI *prima* di chiamare `connect` (che da solo
  controllava solo duplicati e capienza). Qui `connect` è l'unico punto
  d'ingresso e usa `linkRefusal` internamente, come richiesto dal
  compito. Non è una correzione spontanea — vedi `NOTE_DIVERGENZE.md`.

Vedi `NOTE_DIVERGENZE.md` per i comportamenti del prototipo che sembrano
un errore o sono ambigui e che sono stati replicati fedelmente senza
correggerli.

## Struttura

```
model/types.ts       tipi: Card, Link, Graph, parametri, ecc.
model/graph.ts        creazione e lettura (cardById, inputsOf, outputOf) + helper immutabili
catalog/operations.ts operazioni, etichette, sezioni (META, SECTIONS, MERGE_OPS)
catalog/icons.ts       tracciati SVG come stringhe, senza JSX
catalog/params.ts      definizioni dei parametri, valori predefiniti, migrazioni
rules/relations.ts     capienza, cicli, compatibilità, motivi di rifiuto
rules/mutations.ts     operazioni sul grafo
rules/state.ts         completezza dei parametri e stato dei nodi
logic/expressions.ts   connettori, gruppi, anteprima
schema/schema.ts       propagazione delle colonne
data/csv.ts            lettura CSV e deduzione dei tipi
__tests__/             41 scenari di test (vedi tabella sotto)
NOTE_DIVERGENZE.md      comportamenti ambigui del prototipo, replicati senza correggerli
```

## Corrispondenza con il prototipo

Ogni funzione esportata, il file che la contiene, la funzione
corrispondente nel prototipo e la riga in cui si trova — per verificare
la parità una funzione alla volta. "—" significa che non esiste un
corrispondente diretto (helper nuovo, richiesto dall'architettura a
funzioni pure).

| Funzione esportata | File | Corrispondente nel prototipo | Riga |
|---|---|---|---|
| `createGraph` | `model/graph.ts` | — (`cards = {}, linksArr = []`) | 976 |
| `cardById` | `model/graph.ts` | `cards[uid]` | 1001 |
| `inputsOf` | `model/graph.ts` | `inputsOf` | 1613 |
| `outputOf` | `model/graph.ts` | `outputOf` | 1614 |
| `setCard`, `removeCard`, `removeCards`, `addLink`, `filterLinks`, `withLinks`, `patchCard`, `withParamAt` | `model/graph.ts` | — (helper immutabili) | — |
| `createSequentialIdGenerator` | `model/graph.ts` | — (`uidCounter` globale) | 976 |
| `ICONS` | `catalog/icons.ts` | `ICONS` | 871-894 |
| `EMPTY_SLOT_ICON` | `catalog/icons.ts` | `ICONS.empty` | 877 |
| `META` | `catalog/operations.ts` | `META` | 895-904 |
| `SECTIONS` | `catalog/operations.ts` | `SECTIONS` | 4657-4663 |
| `MERGE_OPS` | `catalog/operations.ts` | `MERGE_OPS` | 1611 |
| `sectionOf` | `catalog/operations.ts` | — (derivato da `SECTIONS`) | — |
| `MULTI_OPS` | `catalog/params.ts` | `MULTI_OPS` | 2433 |
| `NO_VALUE_OPS` | `catalog/params.ts` | `NO_VALUE_OPS` | 2434 |
| `FILTER_OPS` | `catalog/params.ts` | `FILTER_OPS` | 2435 |
| `SEPARATORS` | `catalog/params.ts` | `SEPARATORS` | 2436-2439 |
| `LIST_OPS` | `catalog/params.ts` | `LIST_OPS` | 3142 |
| `JOIN_OPS` | `catalog/params.ts` | `JOIN_OPS` | 3158 |
| `JOIN_OP_NAME` | `catalog/params.ts` | `JOIN_OP_NAME` | 3159 |
| `LOGIC_OPS` | `catalog/params.ts` | `LOGIC_OPS` | 3299 |
| `LOGIC_HELP` | `catalog/params.ts` | `LOGIC_HELP` | 3300-3303 |
| `newCondition` | `catalog/params.ts` | `newCondition` | 2440-2442 |
| `createValuesField` | `catalog/params.ts` | `VALUES_DEF` | 2529 |
| `valuesText` | `catalog/params.ts` | `valuesText` | 2530-2533 |
| `fieldFilled` | `catalog/params.ts` | `fieldFilled` | 2534-2537 |
| `PARAM_DEFS` | `catalog/params.ts` | `PARAM_DEFS` | 2445-2525 |
| `MULTI_DEFS` | `catalog/params.ts` | `MULTI_DEFS` | 2538-2582 |
| `ensureMulti` | `catalog/params.ts` | `ensureMulti` | 2585-2602 |
| `defaultParams` | `catalog/params.ts` | `defaultParams` | 2604-2612 |
| `ensureParamsFor` | `catalog/params.ts` | `ensureParams` | 2613-2617 |
| `ensureKeys` | `catalog/params.ts` | `ensureKeys` | 3124-3140 |
| `migrateFilterLogic` | `catalog/params.ts` | dentro `renderFilter` | 3396-3400 |
| `sideText` | `catalog/params.ts` | `sideText` | 3143-3147 |
| `keyComplete` | `catalog/params.ts` | `keyComplete` | 3149-3153 |
| `summarizeKey` | `catalog/params.ts` | `summarizeKey` | 3190-3194 |
| `summarizeCond` | `catalog/params.ts` | `summarizeCond` | 3109-3121 |
| `hasEquiJoinCondition` | `catalog/params.ts` | `equiKey` dentro `renderJoinKeys` | 3220-3221 |
| `boxCapacity` | `rules/relations.ts` | `boxCapacity` | 1612 |
| `reaches` | `rules/relations.ts` | `reaches` | 1887-1899 |
| `linkRefusal` | `rules/relations.ts` | `linkRefusal` | 1901-1915 |
| `relation` | `rules/relations.ts` | `relation` | 1917-1936 |
| `compatiblePair` | `rules/relations.ts` | `compatiblePair` | 1543-1550 |
| `connect` | `rules/mutations.ts` | `connect` + `linkRefusal` (unificate) | 1875-1885, 1901-1915 |
| `spawnOutput` | `rules/mutations.ts` | `spawnOutput` (senza animazione) | 1730-1772 |
| `refreshOutput` | `rules/mutations.ts` | `refreshOutput` | 1659-1672 |
| `pruneOutputs` | `rules/mutations.ts` | `pruneOutputs` | 4414-4430 |
| `enforceCapacity` | `rules/mutations.ts` | `enforceCapacity` | 1638-1648 |
| `nodesRemovedBy` | `rules/mutations.ts` | `nodesRemovedBy` | 4432-4453 |
| `deleteNodes` | `rules/mutations.ts` | `commitDelete` (solo dominio) | 4480-4506 |
| `deleteLink` | `rules/mutations.ts` | `deleteLink` | 4565-4570 |
| `mergeBoxes` | `rules/mutations.ts` | `performMerge` (senza animazioni/DOM) | 1774-1840 |
| `insertable` | `rules/mutations.ts` | `insertable` | 1846-1852 |
| `insertOnLink` | `rules/mutations.ts` | `insertOnLink` (senza posizionamento) | 1853-1873 |
| `detachStep` | `rules/mutations.ts` | `detachStep` (senza animazioni/DOM) | 2137-2203 |
| `deleteStep` | `rules/mutations.ts` | `deleteStep` (senza animazioni/DOM) | 2205-2247 |
| `reorderSteps` | `rules/mutations.ts` | riordino dentro il gestore di drag dei passaggi | 2330-2338 |
| `duplicateNodes` | `rules/mutations.ts` | `duplicateSelection` (solo dominio) | 4594-4618 |
| `isPartialOutput` | `rules/mutations.ts` | condizione dentro `renderOutputIcon` | 1620-1621 |
| `defaultPositionFn` | `rules/mutations.ts` | fallback di posizionamento dentro `spawnOutput` | 1745 |
| `stepMissing` | `rules/state.ts` | `stepMissing` | 1426-1446 |
| `nodeState` | `rules/state.ts` | `nodeState` | 1447-1459 |
| `groupRuns` | `logic/expressions.ts` | `groupRuns` | 3312-3323 |
| `normalizeGroups` | `logic/expressions.ts` | `normalizeGroups` | 3325-3328 |
| `groupPair` | `logic/expressions.ts` | `groupPair` | 3330-3336 |
| `splitAt` | `logic/expressions.ts` | `splitAt` | 3338-3343 |
| `ungroup` | `logic/expressions.ts` | ramo `ung` nel gestore click dell'inspector | 3614 |
| `addToGroup` | `logic/expressions.ts` | ramo `addin` nel gestore click dell'inspector | 3615-3621 |
| `leftAssoc` | `logic/expressions.ts` | `leftAssoc` | 3366-3369 |
| `groupedPreview` | `logic/expressions.ts` | `groupedPreview` (stringa pura, senza HTML) | 3371-3383 |
| `schemaOf` | `schema/schema.ts` | `schemaOf` | 2415-2430 |
| `parseCSV` | `data/csv.ts` | `parseCSV` (su stringa, non `File`) | 4672-4708 |

**Non portata**: `logicPreview` (prototipo, righe 3386-3392) — superseduta
da `groupedPreview`, non più usata nel prototipo stesso.

## Test

77 test (Vitest) sui 15 scenari richiesti, organizzati per area — vedi
l'intestazione di ogni `describe` per il riferimento al numero di
scenario del compito:

| File | Scenari | Test |
|---|---|---|
| `__tests__/relations.test.ts` | 1, 2, 5, 8 | 11 |
| `__tests__/mutations.test.ts` | 3, 4, 6, 7, 15 | 9 |
| `__tests__/params.test.ts` | 9, 12 | 11 |
| `__tests__/state.test.ts` | 10 | 25 |
| `__tests__/expressions.test.ts` | 11 | 7 |
| `__tests__/csv.test.ts` | 13 | 9 |
| `__tests__/schema.test.ts` | 14 | 5 |

`__tests__/helpers.ts` non è un file di test: fornisce i costruttori
`dataset`, `op`, `buildGraph`, `testIdGenerator` usati dagli altri file.
```

### `src/etl-core/__tests__/csv.test.ts`

66 righe

```ts
import { describe, expect, it } from "vitest";
import { parseCSV } from "../data/csv";

describe("parseCSV (scenario 13)", () => {
  it("separatore virgola", () => {
    const res = parseCSV("id,nome\n1,Anna\n2,Bruno\n");
    expect(res?.columns.map((c) => c.name)).toEqual(["id", "nome"]);
    expect(res?.rows).toBe(2);
  });

  it("separatore punto e virgola", () => {
    const res = parseCSV("id;nome\n1;Anna\n2;Bruno\n");
    expect(res?.columns.map((c) => c.name)).toEqual(["id", "nome"]);
  });

  it("separatore tabulazione", () => {
    const res = parseCSV("id\tnome\n1\tAnna\n2\tBruno\n");
    expect(res?.columns.map((c) => c.name)).toEqual(["id", "nome"]);
  });

  it("virgolette che contengono il separatore", () => {
    const res = parseCSV('id,nome\n1,"Rossi, Anna"\n2,Bruno\n');
    const nomeIdx = res?.columns.findIndex((c) => c.name === "nome") ?? -1;
    const nomeValues = res?.columns[nomeIdx]?.values;
    expect(nomeValues).toContain("Rossi, Anna");
  });

  it('le virgolette raddoppiate `""` diventano una virgoletta letterale', () => {
    const res = parseCSV('id,nota\n1,"Ciao ""mondo""!"\n');
    const notaIdx = res?.columns.findIndex((c) => c.name === "nota") ?? -1;
    expect(res?.columns[notaIdx]?.values).toContain('Ciao "mondo"!');
  });

  it("tipi dedotti su un campione misto (integer, numerico, data, stringa)", () => {
    const res = parseCSV(
      [
        "id,importo,data,cliente",
        "1,45.2,2026-01-03,Acme",
        "2,80,2026-01-04,Borealis",
        "3,120.5,2026-01-05,Cedro",
      ].join("\n"),
    );
    const byName = new Map(res?.columns.map((c) => [c.name, c]));
    expect(byName.get("id")?.type).toBe("integer");
    expect(byName.get("importo")?.type).toBe("numerico");
    expect(byName.get("data")?.type).toBe("data");
    expect(byName.get("cliente")?.type).toBe("stringa");
  });

  it("i valori numerici distinti sono ordinati numericamente, non lessicograficamente", () => {
    const res = parseCSV("n\n10\n2\n1\n");
    const col = res?.columns[0];
    expect(col?.values).toEqual(["1", "2", "10"]);
  });

  it("testo vuoto o senza righe restituisce null", () => {
    expect(parseCSV("")).toBeNull();
    expect(parseCSV("\n\n   \n")).toBeNull();
  });

  it("una colonna senza intestazione prende un nome generato", () => {
    const res = parseCSV("id,\n1,x\n2,y\n");
    expect(res?.columns[1]?.name).toBe("colonna_2");
  });
});
```

### `src/etl-core/__tests__/expressions.test.ts`

102 righe

```ts
import { describe, expect, it } from "vitest";
import { groupPair, groupRuns, groupedPreview, splitAt, ungroup } from "../logic/expressions";
import type { FilterCondition } from "../model/types";

function cond(partial: Partial<FilterCondition> & { column: string }): FilterCondition {
  return { op: "=", mode: "manual", values: [], text: "", sep: ",", ...partial };
}

function summarize(c: FilterCondition): string {
  if (c.op === "è uno di") return `${c.column} è uno di ${c.values.join(", ")}`;
  return `${c.column} ${c.op} ${c.text}`;
}

let seq = 0;
function newGroupId(): string {
  seq += 1;
  return `g${seq}`;
}

describe("groupedPreview (scenario 11): anteprima come stringa pura", () => {
  it('"regione = Nord AND (importo > 100 OR stato = Chiuso)"', () => {
    const list: FilterCondition[] = [
      cond({ column: "regione", op: "=", text: "Nord" }),
      cond({ column: "importo", op: ">", text: "100", conn: "AND", g: "g1" }),
      cond({ column: "stato", op: "=", text: "Chiuso", conn: "OR", g: "g1" }),
    ];
    expect(groupedPreview(list, summarize)).toBe(
      "regione = Nord AND (importo > 100 OR stato = Chiuso)",
    );
  });

  it('"(regione è uno di Nord, Centro OR importo > 100) XOR stato = Chiuso"', () => {
    const list: FilterCondition[] = [
      cond({ column: "regione", op: "è uno di", values: ["Nord", "Centro"], g: "g1" }),
      cond({ column: "importo", op: ">", text: "100", conn: "OR", g: "g1" }),
      cond({ column: "stato", op: "=", text: "Chiuso", conn: "XOR" }),
    ];
    expect(groupedPreview(list, summarize)).toBe(
      "(regione è uno di Nord, Centro OR importo > 100) XOR stato = Chiuso",
    );
  });

  it("restituisce '' con meno di due condizioni", () => {
    expect(groupedPreview([cond({ column: "a", text: "1" })], summarize)).toBe("");
    expect(groupedPreview([], summarize)).toBe("");
  });

  it("tre condizioni non raggruppate si combinano a sinistra con parentesi dalla terza in poi", () => {
    const list: FilterCondition[] = [
      cond({ column: "a", text: "1" }),
      cond({ column: "b", text: "2", conn: "AND" }),
      cond({ column: "c", text: "3", conn: "OR" }),
    ];
    expect(groupedPreview(list, summarize)).toBe("(a = 1 AND b = 2) OR c = 3");
  });
});

describe("gruppi: groupPair, splitAt, ungroup (scenario 11)", () => {
  it("un gruppo che parte dalla prima condizione", () => {
    const list: FilterCondition[] = [
      cond({ column: "a", text: "1" }),
      cond({ column: "b", text: "2", conn: "AND" }),
      cond({ column: "c", text: "3", conn: "OR" }),
    ];
    // Raggruppa la prima e la seconda condizione (index=1 -> list[0] e list[1]).
    const grouped = groupPair(list, 1, newGroupId);
    const runs = groupRuns(grouped);
    expect(runs[0]?.g).not.toBeNull();
    expect(runs[0]).toMatchObject({ s: 0, e: 1 });
    expect(runs[1]).toMatchObject({ s: 2, e: 2, g: null });
    expect(groupedPreview(grouped, summarize)).toBe("(a = 1 AND b = 2) OR c = 3");
  });

  it("scioglimento di un gruppo: le condizioni tornano indipendenti", () => {
    const list: FilterCondition[] = [
      cond({ column: "a", text: "1" }),
      cond({ column: "b", text: "2", conn: "AND", g: "g1" }),
      cond({ column: "c", text: "3", conn: "AND", g: "g1" }),
    ];
    const dissolved = ungroup(list, "g1");
    expect(dissolved.every((c) => c.g === undefined)).toBe(true);
    const runs = groupRuns(dissolved);
    expect(runs).toEqual([
      { s: 0, e: 0, g: null },
      { s: 1, e: 1, g: null },
      { s: 2, e: 2, g: null },
    ]);
  });

  it("splitAt divide un gruppo in due a partire dall'indice indicato", () => {
    const list: FilterCondition[] = [
      cond({ column: "a", text: "1" }),
      cond({ column: "b", text: "2", conn: "AND", g: "g1" }),
      cond({ column: "c", text: "3", conn: "AND", g: "g1" }),
    ];
    const split = splitAt(list, 2, newGroupId);
    expect(split[1]?.g).toBe("g1");
    expect(split[2]?.g).not.toBe("g1");
    expect(split[2]?.g).toBeDefined();
  });
});
```

### `src/etl-core/__tests__/helpers.ts`

53 righe

```ts
import { defaultParams } from "../catalog/params";
import { setCard } from "../model/graph";
import type { Card, ComponentId, Graph, Params } from "../model/types";

/** Crea una card dataset con parametri minimi (facoltativamente con colonne). */
export function dataset(id: string, overrides: Partial<Card> & { params0?: Params } = {}): Card {
  const { params0, ...rest } = overrides;
  return {
    id,
    kind: "dataset",
    components: ["dataset"],
    params: [params0 ?? defaultParams("dataset")],
    name: id,
    x: 0,
    y: 0,
    ...rest,
  };
}

/** Crea una card lavorazione con uno o più componenti (default: un solo tipo, parametri predefiniti). */
export function op(
  id: string,
  components: ComponentId[] = ["filter"],
  overrides: Partial<Card> = {},
): Card {
  return {
    id,
    kind: "op",
    components,
    params: components.map((c) => defaultParams(c)),
    name: id,
    x: 0,
    y: 0,
    ...overrides,
  };
}

/** Costruisce un grafo a partire da un elenco di card, senza collegamenti. */
export function buildGraph(cards: readonly Card[]): Graph {
  let graph: Graph = { cards: {}, links: [] };
  for (const c of cards) graph = setCard(graph, c);
  return graph;
}

/** Generatore di id deterministico per i test: 'id-1', 'id-2', ... */
export function testIdGenerator(prefix = "id"): () => string {
  let n = 0;
  return () => {
    n += 1;
    return `${prefix}-${n}`;
  };
}
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

    const r1 = connect(graph, "A", "box");
    expectOk(r1);
    graph = refreshOutput(r1.graph, "box", nextId);
    const outId = outputOf(graph, "box") as string;
    let out = cardById(graph, outId);
    expect(out?.capacity).toBe(2);
    expect(out?.filled).toBe(1);
    expect(out && (out.capacity ?? 0) > 1 && (out.filled ?? 0) < (out.capacity ?? 0)).toBe(true);

    const r2 = connect(graph, "B", "box");
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

    const c1 = connect(graph, "A", "box1");
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box1", nextId);
    const c2 = connect(graph, "B", "box2");
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
    const c1 = connect(graph, "A", "box1");
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box1", nextId);
    const out1 = outputOf(graph, "box1") as string;
    const c2 = connect(graph, out1, "box2");
    expectOk(c2);
    graph = refreshOutput(c2.graph, "box2", nextId);
    const out2 = outputOf(graph, "box2") as string;

    // nodesRemovedBy segue solo la cascata degli OUTPUT (prototipo, righe
    // 4432-4453: il ciclo filtra `c[id].isOutput`): `box2` non è un
    // output, quindi resta — orfano, senza ingressi — anche se il suo
    // unico input (`out1`) sparisce con `box1`.
    const removed = nodesRemovedBy(graph, "box1");
    expect(removed).toEqual(new Set(["box1", out1, out2]));

    graph = deleteNodes(graph, "box1");
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
    const c1 = connect(graph, "A", "box");
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
    const c1 = connect(graph, "A", "box");
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
    const c1 = connect(graph, "A", "box");
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box", nextId);
    const c2 = connect(graph, "D2", "Y");
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
    const c1 = connect(graph, "A", "box1");
    expectOk(c1);
    graph = refreshOutput(c1.graph, "box1", nextId);
    const c2 = connect(graph, "B", "box2");
    expectOk(c2);
    graph = c2.graph;

    const snapshot = JSON.parse(JSON.stringify(graph));

    connect(graph, outputOf(graph, "box1") as string, "box2");
    refreshOutput(graph, "box2", nextId);
    mergeBoxes(graph, "box3", "box2", nextId);
    deleteNodes(graph, "box1");
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
    const c1 = connect(graph, "A", "box");
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

  it("un testo manuale con separatore (vecchio formato) diventa una scelta manuale", () => {
    const legacy = { items: [{ column: "regione", find: "Nord;Centro" }] };
    const migrated = ensureMulti("replaceVal", legacy);
    const rows = migrated["items"] as MultiRow[];
    const find = rows[0]?.["find"];
    expect(find).toEqual(expect.objectContaining({ mode: "manual", text: "Nord;Centro" }));
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

164 righe

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

    const r1 = connect(graph, "A", "box1");
    expectOk(r1);
    graph = refreshOutput(r1.graph, "box1", nextId);
    const out1 = outputOf(graph, "box1");
    expect(out1).not.toBeNull();

    const r2 = connect(graph, out1 as string, "box2");
    expectOk(r2);
    graph = refreshOutput(r2.graph, "box2", nextId);
    const out2 = outputOf(graph, "box2");
    expect(out2).not.toBeNull();

    const rejected = connect(graph, out2 as string, "box1");
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

  it("una terza tabella su un box con un solo join e rifiutata col motivo del prototipo", () => {
    let graph = buildGraph([dataset("A"), dataset("B"), dataset("C"), op("box", ["join"])]);
    const r1 = connect(graph, "A", "box");
    expectOk(r1);
    graph = r1.graph;
    const r2 = connect(graph, "B", "box");
    expectOk(r2);
    graph = r2.graph;

    const r3 = connect(graph, "C", "box");
    expect(r3.ok).toBe(false);
    if (!r3.ok) expect(r3.reason).toBe("Il join ha già le sue due tabelle");
  });

  it("un box senza join (capacita 1) rifiuta la seconda tabella con il motivo generico", () => {
    let graph = buildGraph([dataset("A"), dataset("B"), op("box", ["sort"])]);
    const r1 = connect(graph, "A", "box");
    expectOk(r1);
    graph = r1.graph;
    const r2 = connect(graph, "B", "box");
    expect(r2.ok).toBe(false);
    if (!r2.ok) expect(r2.reason).toBe("Il box accetta una sola tabella in ingresso");
  });
});

describe("enforceCapacity: rimozione del join da un box con due ingressi (scenario 5)", () => {
  it("resta il collegamento piu vecchio e l'output torna a capacity 1", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), dataset("B"), op("box", ["join", "sort"])]);
    const r1 = connect(graph, "A", "box");
    expectOk(r1);
    graph = r1.graph;
    const r2 = connect(graph, "B", "box");
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
    let graph = buildGraph([
      dataset("A"),
      dataset("B"),
      dataset("C"),
      op("box", ["join", "union"]),
    ]);
    for (const id of ["A", "B", "C"]) {
      const r = connect(graph, id, "box");
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
    const r1 = connect(graph, "A", "box");
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

74 righe

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
    const c1 = connect(graph, "A", "box1");
    if (!c1.ok) throw new Error("unexpected refusal");
    graph = refreshOutput(c1.graph, "box1", nextId);
    const out1 = outputOf(graph, "box1") as string;
    const c2 = connect(graph, out1, "box2");
    if (!c2.ok) throw new Error("unexpected refusal");
    graph = refreshOutput(c2.graph, "box2", nextId);
    const out2 = outputOf(graph, "box2") as string;

    expect(schemaOf(graph, out1)).toEqual(COLS_A);
    expect(schemaOf(graph, out2)).toEqual(COLS_A);
  });

  it("una lavorazione con due ingressi vede l'unione delle colonne, senza duplicati", () => {
    let graph = buildGraph([
      withColumns("A", COLS_A),
      withColumns("B", COLS_B),
      op("box", ["join"]),
    ]);
    const c1 = connect(graph, "A", "box");
    if (!c1.ok) throw new Error("unexpected refusal");
    graph = c1.graph;
    const c2 = connect(graph, "B", "box");
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

106 righe

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
        { column: "regione", op: "=", mode: "manual", values: [], text: "Nord", sep: "," },
      ],
    };
    expect(stepMissing("filter", par as unknown as Params)).toBe(false);
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

