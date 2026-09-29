# 01b-etl-core-a.md

File in questo blocco:

- `src/etl-core/NOTE_DIVERGENZE.md`
- `src/etl-core/README.md`
- `src/etl-core/__tests__/csv.test.ts`
- `src/etl-core/__tests__/expressions.test.ts`
- `src/etl-core/__tests__/fase11-requisiti.test.ts`
- `src/etl-core/__tests__/fase11.test.ts`
- `src/etl-core/__tests__/helpers.ts`
- `src/etl-core/__tests__/mutations.test.ts`

---

### `src/etl-core/NOTE_DIVERGENZE.md`

85 righe

```md
# Note di divergenza

Comportamenti di `src/etl-core/` che differiscono dal prototipo
(`docs/prototype/isa-fusion-prototype.html`): correzioni intenzionali
introdotte nella Fase 1.1 e chiarimenti architetturali.

## Correzioni intenzionali (Fase 1.1)

Le prime due note della Fase 1 descrivevano comportamenti replicati
fedelmente; nella Fase 1.1 sono stati corretti di proposito, insieme ad
altri punti che nascevano dallo stesso problema.

### 1. Messaggi di capienza generali

`linkRefusal` (prototipo, riga 1913; `rules/relations.ts`). Il prototipo
diceva sempre "Il join ha gia le sue due tabelle" quando `cap > 1`, anche
con capienza 3 o senza alcun join, e "al join manca una tabella" per un
output incompleto. Ora:

- `Il box ha già tutte le sue N tabelle` (N = `boxCapacity(box)`);
- `Questo output non è ancora completo: manca ancora una tabella in ingresso`.

### 2. Una sola fonte di verità per i valori

Nel prototipo un campo a più valori (`ValuesField`, condizioni del filtro,
`rlist` del join) aveva due rappresentazioni, `values` e `text`, e diverse
funzioni guardavano l'una o l'altra a seconda di `mode`. Un `text`
residuo, non più mostrato dall'interfaccia, poteva far risultare
"completa" una condizione con `values` vuoto (prototipo, righe 1430-1431
e 2534-2537). Ora:

- `values` è l'unica fonte di verità; `text` è solo un formato di
  transito.
- `normalizeValuesField` (`catalog/params.ts`) è la migrazione unica
  testo → valori, che nel prototipo era sparsa dentro `pickerHtml`
  (righe 2840-2848). Viene applicata da `ensureMulti`, `ensureKeys`
  (`rlist`) e `migrateFilterLogic`. Il testo viene diviso su virgola,
  punto e virgola, a capo e sul `sep` dichiarato (la migrazione del
  prototipo usava solo `sep`, ma `splitTokens`, riga 2839, già accettava
  questi separatori), senza duplicati.
- `fieldFilled` e `stepMissing('filter', ...)` contano solo `values` per
  gli operatori in `MULTI_OPS`; gli altri operatori richiedono `text`.
- `summarizeCond` usa sempre `values` per `MULTI_OPS` e non riceve più il
  parametro `columnHasValues`.

### 3. `connect` garantisce l'output

Nel prototipo l'output veniva creato dall'interazione UI dopo `connect`.
Ora `connect(graph, source, target, nextId, positionFn?)` chiama
`refreshOutput` sul box di destinazione. Invariante: dopo `connect`, ogni
lavorazione con almeno un ingresso ha un output con `capacity`/`filled`
aggiornati. Anche `mergeBoxes` accetta ora `positionFn`.

L'invariante vale per ogni operazione esportata di `rules/mutations.ts`.
Nel prototipo `renderAll` (riga 4407) chiamava `refreshOutput` su ogni
lavorazione dopo un'eliminazione; qui lo fanno direttamente
`deleteNodes(graph, ids, nextId, positionFn?)` e
`deleteLink(graph, link, nextId, positionFn?)`, che ricevono ora `nextId`:
eliminare l'output di un box che ha ancora ingressi (o il collegamento
box → output) lo ricrea.

### 4. Parametri predefiniti nel formato attuale

`defaultParams('filter')` non genera più il campo superato `logic`, e
`defaultParams('join')` produce chiavi già complete (come dopo
`ensureKeys`), invece di generare il vecchio formato solo per migrarlo.

## Non è una divergenza (chiarimento architetturale)

`connect` nel prototipo (riga 1875-1885) **non** controlla i cicli: quel
controllo vive solo in `linkRefusal`, invocata dall'interazione utente
_prima_ di chiamare `connect`. Il compito chiede esplicitamente che
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

164 righe

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
  dall'interazione UI _prima_ di chiamare `connect` (che da solo
  controllava solo duplicati e capienza). Qui `connect` è l'unico punto
  d'ingresso e usa `linkRefusal` internamente, come richiesto dal
  compito. Non è una correzione spontanea — vedi `NOTE_DIVERGENZE.md`.
  Dalla Fase 1.1 `connect` riceve anche `nextId` e crea/aggiorna l'output
  del box di destinazione.

Vedi `NOTE_DIVERGENZE.md` per le correzioni intenzionali della Fase 1.1
rispetto al prototipo (messaggi di capienza, una sola fonte di verità per
i valori, output garantito da `connect`, parametri predefiniti).

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
NOTE_DIVERGENZE.md      differenze intenzionali rispetto al prototipo (Fase 1.1)
```

## Corrispondenza con il prototipo

Ogni funzione esportata, il file che la contiene, la funzione
corrispondente nel prototipo e la riga in cui si trova — per verificare
la parità una funzione alla volta. "—" significa che non esiste un
corrispondente diretto (helper nuovo, richiesto dall'architettura a
funzioni pure).

| Funzione esportata                                                                                        | File                    | Corrispondente nel prototipo                    | Riga                 |
| --------------------------------------------------------------------------------------------------------- | ----------------------- | ----------------------------------------------- | -------------------- |
| `createGraph`                                                                                             | `model/graph.ts`        | — (`cards = {}, linksArr = []`)                 | 976                  |
| `cardById`                                                                                                | `model/graph.ts`        | `cards[uid]`                                    | 1001                 |
| `inputsOf`                                                                                                | `model/graph.ts`        | `inputsOf`                                      | 1613                 |
| `outputOf`                                                                                                | `model/graph.ts`        | `outputOf`                                      | 1614                 |
| `setCard`, `removeCard`, `removeCards`, `addLink`, `filterLinks`, `withLinks`, `patchCard`, `withParamAt` | `model/graph.ts`        | — (helper immutabili)                           | —                    |
| `createSequentialIdGenerator`                                                                             | `model/graph.ts`        | — (`uidCounter` globale)                        | 976                  |
| `ICONS`                                                                                                   | `catalog/icons.ts`      | `ICONS`                                         | 871-894              |
| `EMPTY_SLOT_ICON`                                                                                         | `catalog/icons.ts`      | `ICONS.empty`                                   | 877                  |
| `META`                                                                                                    | `catalog/operations.ts` | `META`                                          | 895-904              |
| `SECTIONS`                                                                                                | `catalog/operations.ts` | `SECTIONS`                                      | 4657-4663            |
| `MERGE_OPS`                                                                                               | `catalog/operations.ts` | `MERGE_OPS`                                     | 1611                 |
| `sectionOf`                                                                                               | `catalog/operations.ts` | — (derivato da `SECTIONS`)                      | —                    |
| `MULTI_OPS`                                                                                               | `catalog/params.ts`     | `MULTI_OPS`                                     | 2433                 |
| `NO_VALUE_OPS`                                                                                            | `catalog/params.ts`     | `NO_VALUE_OPS`                                  | 2434                 |
| `FILTER_OPS`                                                                                              | `catalog/params.ts`     | `FILTER_OPS`                                    | 2435                 |
| `SEPARATORS`                                                                                              | `catalog/params.ts`     | `SEPARATORS`                                    | 2436-2439            |
| `LIST_OPS`                                                                                                | `catalog/params.ts`     | `LIST_OPS`                                      | 3142                 |
| `JOIN_OPS`                                                                                                | `catalog/params.ts`     | `JOIN_OPS`                                      | 3158                 |
| `JOIN_OP_NAME`                                                                                            | `catalog/params.ts`     | `JOIN_OP_NAME`                                  | 3159                 |
| `LOGIC_OPS`                                                                                               | `catalog/params.ts`     | `LOGIC_OPS`                                     | 3299                 |
| `LOGIC_HELP`                                                                                              | `catalog/params.ts`     | `LOGIC_HELP`                                    | 3300-3303            |
| `newCondition`                                                                                            | `catalog/params.ts`     | `newCondition`                                  | 2440-2442            |
| `createValuesField`                                                                                       | `catalog/params.ts`     | `VALUES_DEF`                                    | 2529                 |
| `valuesText`                                                                                              | `catalog/params.ts`     | `valuesText`                                    | 2530-2533            |
| `fieldFilled`                                                                                             | `catalog/params.ts`     | `fieldFilled`                                   | 2534-2537            |
| `PARAM_DEFS`                                                                                              | `catalog/params.ts`     | `PARAM_DEFS`                                    | 2445-2525            |
| `MULTI_DEFS`                                                                                              | `catalog/params.ts`     | `MULTI_DEFS`                                    | 2538-2582            |
| `ensureMulti`                                                                                             | `catalog/params.ts`     | `ensureMulti`                                   | 2585-2602            |
| `defaultParams`                                                                                           | `catalog/params.ts`     | `defaultParams`                                 | 2604-2612            |
| `ensureParamsFor`                                                                                         | `catalog/params.ts`     | `ensureParams`                                  | 2613-2617            |
| `ensureKeys`                                                                                              | `catalog/params.ts`     | `ensureKeys`                                    | 3124-3140            |
| `migrateFilterLogic`                                                                                      | `catalog/params.ts`     | dentro `renderFilter`                           | 3396-3400            |
| `sideText`                                                                                                | `catalog/params.ts`     | `sideText`                                      | 3143-3147            |
| `keyComplete`                                                                                             | `catalog/params.ts`     | `keyComplete`                                   | 3149-3153            |
| `summarizeKey`                                                                                            | `catalog/params.ts`     | `summarizeKey`                                  | 3190-3194            |
| `summarizeCond`                                                                                           | `catalog/params.ts`     | `summarizeCond`                                 | 3109-3121            |
| `hasEquiJoinCondition`                                                                                    | `catalog/params.ts`     | `equiKey` dentro `renderJoinKeys`               | 3220-3221            |
| `boxCapacity`                                                                                             | `rules/relations.ts`    | `boxCapacity`                                   | 1612                 |
| `reaches`                                                                                                 | `rules/relations.ts`    | `reaches`                                       | 1887-1899            |
| `linkRefusal`                                                                                             | `rules/relations.ts`    | `linkRefusal`                                   | 1901-1915            |
| `relation`                                                                                                | `rules/relations.ts`    | `relation`                                      | 1917-1936            |
| `compatiblePair`                                                                                          | `rules/relations.ts`    | `compatiblePair`                                | 1543-1550            |
| `connect`                                                                                                 | `rules/mutations.ts`    | `connect` + `linkRefusal` (unificate)           | 1875-1885, 1901-1915 |
| `spawnOutput`                                                                                             | `rules/mutations.ts`    | `spawnOutput` (senza animazione)                | 1730-1772            |
| `refreshOutput`                                                                                           | `rules/mutations.ts`    | `refreshOutput`                                 | 1659-1672            |
| `pruneOutputs`                                                                                            | `rules/mutations.ts`    | `pruneOutputs`                                  | 4414-4430            |
| `enforceCapacity`                                                                                         | `rules/mutations.ts`    | `enforceCapacity`                               | 1638-1648            |
| `nodesRemovedBy`                                                                                          | `rules/mutations.ts`    | `nodesRemovedBy`                                | 4432-4453            |
| `deleteNodes`                                                                                             | `rules/mutations.ts`    | `commitDelete` (solo dominio)                   | 4480-4506            |
| `deleteLink`                                                                                              | `rules/mutations.ts`    | `deleteLink`                                    | 4565-4570            |
| `mergeBoxes`                                                                                              | `rules/mutations.ts`    | `performMerge` (senza animazioni/DOM)           | 1774-1840            |
| `insertable`                                                                                              | `rules/mutations.ts`    | `insertable`                                    | 1846-1852            |
| `insertOnLink`                                                                                            | `rules/mutations.ts`    | `insertOnLink` (senza posizionamento)           | 1853-1873            |
| `detachStep`                                                                                              | `rules/mutations.ts`    | `detachStep` (senza animazioni/DOM)             | 2137-2203            |
| `deleteStep`                                                                                              | `rules/mutations.ts`    | `deleteStep` (senza animazioni/DOM)             | 2205-2247            |
| `reorderSteps`                                                                                            | `rules/mutations.ts`    | riordino dentro il gestore di drag dei passaggi | 2330-2338            |
| `duplicateNodes`                                                                                          | `rules/mutations.ts`    | `duplicateSelection` (solo dominio)             | 4594-4618            |
| `isPartialOutput`                                                                                         | `rules/mutations.ts`    | condizione dentro `renderOutputIcon`            | 1620-1621            |
| `defaultPositionFn`                                                                                       | `rules/mutations.ts`    | fallback di posizionamento dentro `spawnOutput` | 1745                 |
| `stepMissing`                                                                                             | `rules/state.ts`        | `stepMissing`                                   | 1426-1446            |
| `nodeState`                                                                                               | `rules/state.ts`        | `nodeState`                                     | 1447-1459            |
| `groupRuns`                                                                                               | `logic/expressions.ts`  | `groupRuns`                                     | 3312-3323            |
| `normalizeGroups`                                                                                         | `logic/expressions.ts`  | `normalizeGroups`                               | 3325-3328            |
| `groupPair`                                                                                               | `logic/expressions.ts`  | `groupPair`                                     | 3330-3336            |
| `splitAt`                                                                                                 | `logic/expressions.ts`  | `splitAt`                                       | 3338-3343            |
| `ungroup`                                                                                                 | `logic/expressions.ts`  | ramo `ung` nel gestore click dell'inspector     | 3614                 |
| `addToGroup`                                                                                              | `logic/expressions.ts`  | ramo `addin` nel gestore click dell'inspector   | 3615-3621            |
| `leftAssoc`                                                                                               | `logic/expressions.ts`  | `leftAssoc`                                     | 3366-3369            |
| `groupedPreview`                                                                                          | `logic/expressions.ts`  | `groupedPreview` (stringa pura, senza HTML)     | 3371-3383            |
| `schemaOf`                                                                                                | `schema/schema.ts`      | `schemaOf`                                      | 2415-2430            |
| `parseCSV`                                                                                                | `data/csv.ts`           | `parseCSV` (su stringa, non `File`)             | 4672-4708            |

**Non portata**: `logicPreview` (prototipo, righe 3386-3392) — superseduta
da `groupedPreview`, non più usata nel prototipo stesso.

## Test

77 test (Vitest) sui 15 scenari richiesti, organizzati per area — vedi
l'intestazione di ogni `describe` per il riferimento al numero di
scenario del compito:

| File                            | Scenari        | Test |
| ------------------------------- | -------------- | ---- |
| `__tests__/relations.test.ts`   | 1, 2, 5, 8     | 11   |
| `__tests__/mutations.test.ts`   | 3, 4, 6, 7, 15 | 9    |
| `__tests__/params.test.ts`      | 9, 12          | 11   |
| `__tests__/state.test.ts`       | 10             | 25   |
| `__tests__/expressions.test.ts` | 11             | 7    |
| `__tests__/csv.test.ts`         | 13             | 9    |
| `__tests__/schema.test.ts`      | 14             | 5    |

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

### `src/etl-core/__tests__/fase11-requisiti.test.ts`

257 righe

```ts
import { describe, expect, it } from "vitest";
import { dataset, op, buildGraph, testIdGenerator } from "./helpers";
import {
  connect,
  deleteLink,
  deleteNodes,
  deleteStep,
  detachStep,
  duplicateNodes,
  enforceCapacity,
  insertOnLink,
  mergeBoxes,
  pruneOutputs,
  refreshOutput,
  reorderSteps,
  spawnOutput,
} from "../rules/mutations";
import { stepMissing } from "../rules/state";
import { cardById, inputsOf, outputOf } from "../model/graph";
import { ensureKeys, ensureMulti } from "../catalog/params";
import type { FilterParams, Graph, JoinParams, MultiRow, Params, PositionFn } from "../model/types";

/** Requisiti espliciti della Fase 1.1 (vedi NOTE_DIVERGENZE.md). */

function expectOk(result: { ok: boolean }): asserts result is { ok: true; graph: Graph } {
  expect(result.ok).toBe(true);
}

/** Ogni lavorazione con almeno un ingresso ha un output. */
function boxesWithoutOutput(graph: Graph): string[] {
  return Object.values(graph.cards)
    .filter((c) => c.kind === "op" && inputsOf(graph, c.id).length > 0 && !outputOf(graph, c.id))
    .map((c) => c.id);
}

describe("invariante: dopo ogni operazione di rules/mutations.ts ogni lavorazione con ingressi ha un output", () => {
  const nextId = testIdGenerator("n");

  /**
   * A, B, C -> box1 [join, union] -> out1 -> box2 [filter] -> out2.
   * box3 [sort] e X [filter] sono isolati.
   */
  function scenario(): { graph: Graph; out1: string; out2: string } {
    let graph = buildGraph([
      dataset("A"),
      dataset("B"),
      dataset("C"),
      op("box1", ["join", "union"]),
      op("box2", ["filter"]),
      op("box3", ["sort"]),
      op("X", ["filter"]),
    ]);
    for (const id of ["A", "B", "C"]) {
      const r = connect(graph, id, "box1", nextId);
      expectOk(r);
      graph = r.graph;
    }
    const out1 = outputOf(graph, "box1") as string;
    const r = connect(graph, out1, "box2", nextId);
    expectOk(r);
    graph = r.graph;
    return { graph, out1, out2: outputOf(graph, "box2") as string };
  }

  const cases: Array<[string, (s: ReturnType<typeof scenario>) => Graph | null]> = [
    [
      "connect",
      ({ graph }) => {
        const r = connect(graph, "A", "X", nextId);
        expectOk(r);
        return r.graph;
      },
    ],
    ["spawnOutput", ({ graph }) => spawnOutput(graph, "box3", nextId)?.graph ?? null],
    ["refreshOutput", ({ graph }) => refreshOutput(graph, "box2", nextId)],
    ["pruneOutputs", ({ graph }) => pruneOutputs(graph)],
    ["enforceCapacity", ({ graph }) => enforceCapacity(graph, "box1")],
    ["deleteNodes (un output)", ({ graph, out1 }) => deleteNodes(graph, out1, nextId)],
    ["deleteNodes (un dataset)", ({ graph }) => deleteNodes(graph, "A", nextId)],
    ["deleteNodes (un box)", ({ graph }) => deleteNodes(graph, "box1", nextId)],
    [
      "deleteLink (box -> output)",
      ({ graph, out1 }) => deleteLink(graph, { from: "box1", to: out1 }, nextId),
    ],
    [
      "deleteLink (dataset -> box)",
      ({ graph }) => deleteLink(graph, { from: "A", to: "box1" }, nextId),
    ],
    [
      "mergeBoxes",
      ({ graph }) => {
        const r = mergeBoxes(graph, "box2", "box1", nextId);
        expectOk(r);
        return r.graph;
      },
    ],
    [
      "mergeBoxes (box isolato)",
      ({ graph }) => {
        const r = mergeBoxes(graph, "box3", "box2", nextId);
        expectOk(r);
        return r.graph;
      },
    ],
    ["insertOnLink", ({ graph }) => insertOnLink(graph, { from: "A", to: "box1" }, "X", nextId)],
    ["detachStep", ({ graph }) => detachStep(graph, "box1", 0, nextId)?.graph ?? null],
    ["deleteStep", ({ graph }) => deleteStep(graph, "box1", 1, nextId)],
    ["reorderSteps", ({ graph }) => reorderSteps(graph, "box1", 0, 1)],
    ["duplicateNodes", ({ graph }) => duplicateNodes(graph, ["box1", "A"], nextId).graph],
  ];

  it("lo scenario di partenza rispetta l'invariante", () => {
    expect(boxesWithoutOutput(scenario().graph)).toEqual([]);
  });

  it.each(cases)("%s", (_name, run) => {
    const result = run(scenario());
    expect(result).not.toBeNull();
    expect(boxesWithoutOutput(result as Graph)).toEqual([]);
  });

  it("deleteNodes su un output lo ricrea per il box che ha ancora ingressi", () => {
    const { graph, out1 } = scenario();
    const next = deleteNodes(graph, out1, nextId);
    const recreated = outputOf(next, "box1");
    expect(recreated).not.toBeNull();
    expect(recreated).not.toBe(out1);
    expect(cardById(next, recreated as string)).toMatchObject({ capacity: 3, filled: 3 });
  });
});

describe("conversione testo -> valori (virgola, punto e virgola, a capo, senza duplicati)", () => {
  it("ensureMulti: campo 'find' di Sostituisci valori", () => {
    const legacy = { items: [{ column: "regione", find: "Nord, Sud;Centro\nNord\n" }] };
    const rows = ensureMulti("replaceVal", legacy)["items"] as MultiRow[];
    expect(rows[0]?.["find"]).toEqual({
      mode: "list",
      values: ["Nord", "Sud", "Centro"],
      text: "",
      sep: ",",
    });
  });

  it("ensureMulti: un campo manuale in forma di oggetto viene unito ai valori esistenti", () => {
    const legacy = {
      items: [
        {
          column: "regione",
          find: { mode: "manual", values: ["Nord"], text: "Sud;Nord", sep: "," },
        },
      ],
    };
    const rows = ensureMulti("replaceVal", legacy)["items"] as MultiRow[];
    expect(rows[0]?.["find"]).toMatchObject({ mode: "list", values: ["Nord", "Sud"], text: "" });
  });

  it("ensureKeys: rlist del join", () => {
    const par: JoinParams = {
      type: "inner",
      keys: [
        {
          left: "a",
          right: "",
          op: "è uno di",
          rmode: "list",
          rlist: { mode: "manual", values: [], text: "x;y\nz,x", sep: "," },
        },
      ],
    };
    const [key] = ensureKeys(par);
    expect(key?.rlist).toEqual({ mode: "list", values: ["x", "y", "z"], text: "", sep: "," });
  });
});

describe("stepMissing del filtro con operatore a valore singolo", () => {
  it("'>' con values pieno e text vuoto -> incompleta", () => {
    const par: FilterParams = {
      conditions: [
        { column: "importo", op: ">", mode: "list", values: ["10"], text: "", sep: "," },
      ],
    };
    expect(stepMissing("filter", par as unknown as Params)).toBe(true);
  });
});

describe("messaggi di capienza", () => {
  it("box join + union: la quarta tabella è rifiutata con la capienza reale (3)", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([
      dataset("A"),
      dataset("B"),
      dataset("C"),
      dataset("D"),
      op("box", ["join", "union"]),
    ]);
    for (const id of ["A", "B", "C"]) {
      const r = connect(graph, id, "box", nextId);
      expectOk(r);
      graph = r.graph;
    }
    const r = connect(graph, "D", "box", nextId);
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.reason).toBe("Il box ha già tutte le sue 3 tabelle");
  });

  it("box con sola union: la terza tabella è rifiutata senza nominare il join", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), dataset("B"), dataset("C"), op("box", ["union"])]);
    for (const id of ["A", "B"]) {
      const r = connect(graph, id, "box", nextId);
      expectOk(r);
      graph = r.graph;
    }
    const r = connect(graph, "C", "box", nextId);
    expect(r.ok).toBe(false);
    if (!r.ok) {
      expect(r.reason).toBe("Il box ha già tutte le sue 2 tabelle");
      expect(r.reason).not.toMatch(/join/);
    }
  });

  it("output incompleto di una union: messaggio generale", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box", ["union"]), op("next")]);
    const r1 = connect(graph, "A", "box", nextId);
    expectOk(r1);
    graph = r1.graph;
    const r2 = connect(graph, outputOf(graph, "box") as string, "next", nextId);
    expect(r2.ok).toBe(false);
    if (!r2.ok) {
      expect(r2.reason).toBe(
        "Questo output non è ancora completo: manca ancora una tabella in ingresso",
      );
    }
  });
});

describe("mergeBoxes accetta positionFn", () => {
  it("usa positionFn per l'output creato dalla fusione", () => {
    // box2 ha un ingresso ma nessun output (grafo costruito a mano): la
    // fusione in box1 deve crearne uno, posizionato da positionFn.
    const base = buildGraph([dataset("A"), op("box1", ["filter"]), op("box2", ["sort"])]);
    const graph: Graph = { ...base, links: [{ from: "A", to: "box2" }] };
    const calls: string[] = [];
    const positionFn: PositionFn = (_g, anchorId) => {
      calls.push(anchorId);
      return { x: 777, y: 333 };
    };
    const r = mergeBoxes(graph, "box2", "box1", testIdGenerator("n"), positionFn);
    expectOk(r);
    const out = outputOf(r.graph, "box1");
    expect(out).not.toBeNull();
    expect(calls).toEqual(["box1"]);
    expect(cardById(r.graph, out as string)).toMatchObject({ x: 777, y: 333 });
  });
});
```

### `src/etl-core/__tests__/fase11.test.ts`

149 righe

```ts
import { describe, expect, it } from "vitest";
import { dataset, op, buildGraph, testIdGenerator } from "./helpers";
import { connect } from "../rules/mutations";
import { cardById, outputOf } from "../model/graph";
import {
  createValuesField,
  defaultParams,
  fieldFilled,
  migrateFilterLogic,
  normalizeValuesField,
  summarizeCond,
} from "../catalog/params";
import type { FilterCondition, FilterParams, JoinParams, ValuesField } from "../model/types";

/**
 * Correzioni intenzionali della Fase 1.1 (vedi NOTE_DIVERGENZE.md):
 * una sola fonte di verità per i valori, output garantito da `connect`,
 * messaggi di capienza generali, parametri predefiniti già nel formato attuale.
 */
describe("normalizeValuesField", () => {
  it("un testo manuale diventa una lista, senza duplicati e senza token vuoti", () => {
    const v: ValuesField = {
      mode: "manual",
      values: ["Nord"],
      text: "Nord; Sud;;Centro",
      sep: ";",
    };
    expect(normalizeValuesField(v)).toEqual({
      mode: "list",
      values: ["Nord", "Sud", "Centro"],
      text: "",
      sep: ";",
    });
  });

  it("un campo già in lista, o con testo vuoto, torna identico", () => {
    const list: ValuesField = { mode: "list", values: ["A"], text: "residuo", sep: "," };
    expect(normalizeValuesField(list)).toBe(list);
    const empty: ValuesField = { mode: "manual", values: [], text: "  ", sep: "," };
    expect(normalizeValuesField(empty)).toBe(empty);
  });

  it("non modifica il campo che riceve", () => {
    const v: ValuesField = { mode: "manual", values: [], text: "a,b", sep: "," };
    const snapshot = JSON.parse(JSON.stringify(v));
    normalizeValuesField(v);
    expect(v).toEqual(snapshot);
  });
});

describe("fieldFilled conta solo values", () => {
  it("un testo residuo non basta a rendere compilato un campo a più valori", () => {
    const v: ValuesField = { ...createValuesField(), text: "Nord" };
    expect(fieldFilled({ type: "values" }, v)).toBe(false);
    expect(fieldFilled({ type: "values" }, { ...createValuesField(), values: ["Nord"] })).toBe(
      true,
    );
  });
});

describe("migrateFilterLogic normalizza anche i valori", () => {
  it("logic globale -> connettori, e testo manuale -> lista", () => {
    const par: FilterParams = {
      logic: "O",
      conditions: [
        { column: "a", op: "=", mode: "manual", values: [], text: "x, y", sep: "," },
        { column: "b", op: "=", mode: "list", values: ["z"], text: "", sep: "," },
      ],
    };
    const migrated = migrateFilterLogic(par);
    expect(migrated.logic).toBeUndefined();
    expect(migrated.conditions[0]).toMatchObject({ mode: "list", values: ["x", "y"], text: "" });
    expect(migrated.conditions[1]?.conn).toBe("OR");
  });
});

describe("summarizeCond usa values per gli operatori a più valori", () => {
  const base: FilterCondition = {
    column: "regione",
    op: "=",
    mode: "list",
    values: [],
    text: "",
    sep: ",",
  };

  it("ignora il testo residuo", () => {
    expect(summarizeCond({ ...base, text: "vecchio" })).toBe("regione = …");
    expect(summarizeCond({ ...base, values: ["Nord", "Sud"] })).toBe("regione = Nord, Sud");
  });

  it("gli altri operatori usano text", () => {
    expect(summarizeCond({ ...base, op: ">", text: "10" })).toBe("regione > 10");
  });
});

describe("defaultParams nel formato attuale", () => {
  it("il filtro non ha il campo logic", () => {
    expect("logic" in (defaultParams("filter") as unknown as FilterParams)).toBe(false);
  });

  it("il join ha le chiavi già complete", () => {
    const par = defaultParams("join") as unknown as JoinParams;
    expect(par.keys[0]).toEqual({
      left: "",
      right: "",
      op: "=",
      lmode: "col",
      rmode: "col",
      rlist: createValuesField(),
      lval: "",
      rval: "",
    });
  });
});

describe("connect garantisce l'output", () => {
  it("dopo il primo ingresso il box ha un output, aggiornato al secondo", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), dataset("B"), op("box", ["join"])]);
    const r1 = connect(graph, "A", "box", nextId);
    if (!r1.ok) throw new Error("unexpected refusal");
    graph = r1.graph;
    const out = outputOf(graph, "box");
    expect(out).toBe("n-1");
    expect(cardById(graph, out as string)).toMatchObject({ capacity: 2, filled: 1 });

    const r2 = connect(graph, "B", "box", nextId);
    if (!r2.ok) throw new Error("unexpected refusal");
    expect(outputOf(r2.graph, "box")).toBe(out);
    expect(cardById(r2.graph, out as string)).toMatchObject({ capacity: 2, filled: 2 });
  });

  it("un output incompleto viene rifiutato col messaggio generale", () => {
    const nextId = testIdGenerator("n");
    let graph = buildGraph([dataset("A"), op("box", ["join"]), op("next")]);
    const r1 = connect(graph, "A", "box", nextId);
    if (!r1.ok) throw new Error("unexpected refusal");
    graph = r1.graph;
    const r2 = connect(graph, outputOf(graph, "box") as string, "next", nextId);
    expect(r2.ok).toBe(false);
    if (!r2.ok) {
      expect(r2.reason).toBe(
        "Questo output non è ancora completo: manca ancora una tabella in ingresso",
      );
    }
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

