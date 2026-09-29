# 13-misc-a.md

File in questo blocco:

- `src/etl-store/README.md`
- `src/etl-store/__tests__/helpers.ts`
- `src/etl-store/__tests__/persistence.test.ts`
- `src/etl-store/__tests__/react.test.ts`
- `src/etl-store/__tests__/reduce.test.ts`

---

### `src/etl-store/README.md`

195 righe

```md
# etl-store — Fase 3: stato, cronologia, registro, salvataggio

Un archivio unico per il canvas ETL: stato, comandi, cronologia,
registro delle attività e salvataggio, più il collegamento a React.
Nessun componente visivo (Fase 4). Il riferimento è il prototipo
`docs/prototype/isa-fusion-prototype.html`.

## Moduli

```
types.ts        stato, comandi, risultati, punti di cronologia, voci di registro
state.ts        stato iniziale (canvas vuoto) e valori predefiniti
reduce.ts       reduce(state, command) → { state, result }: tutti i comandi, puro
store.ts        createEtlStore: cronologia, gesti, registro, dispatch, percorsi derivati
derived.ts      derivati: stato e schema dei nodi, flusso, percorsi con memoria
serialize.ts    formato di salvataggio e validazione, puro
persistence.ts  UNICO modulo che tocca localStorage (con controlli per il server)
react.ts        UNICO modulo che importa React (useSyncExternalStore)
index.ts        il nucleo (senza react.ts e persistence.ts)
```

Il nucleo (tutto tranne `react.ts` e `persistence.ts`) non importa React e
non usa le API del browser: gira identico in Node. Nessuna nuova
dipendenza.

## Stato

| Campo       | Contenuto                                                                   | Cronologia      | Salvato           |
| ----------- | --------------------------------------------------------------------------- | --------------- | ----------------- |
| `graph`     | grafo di etl-core (nodi, collegamenti, parametri, posizioni, postazioni)    | sì              | sì                |
| `mode`      | `'free'` (Libero) o `'grid'` (Organizzato)                                  | sì (vedi sotto) | sì                |
| `library`   | dataset caricati: `id`, `name`, `path`, `columns`, `rows` (numero di righe) | no              | sì                |
| `selection` | identificativi selezionati                                                  | no              | no                |
| `inspector` | nodo e passaggio aperti nell'inspector                                      | no              | no                |
| `panels`    | strumenti e inspector: lato e aperto/chiuso                                 | no              | sì                |
| `view`      | `x`, `y`, `zoom` (0,35–2)                                                   | no              | no                |
| `options`   | `flowOnlyIfValid` (spento), `maxBends` (1)                                  | no              | sì                |
| `counters`  | `uid`, `ds` (nei punti di cronologia), `lib`                                | `uid`, `ds`     | ricavati dagli id |

Derivati, mai memorizzati né salvati (`derived.ts`, `store.getRoutes()`):
percorsi dei cavi (`settleLinks` a partire dai percorsi precedenti), stato
di ogni nodo (`nodeState`), schema di ogni nodo (`schemaOf`), flusso di un
collegamento (`linkLive`).

Il pannello "Funzionalità" del prototipo (righe 914-935, `FEATURES`) era
uno strumento di progettazione e non è portato: tutte le funzionalità sono
sempre attive. Resta solo l'opzione "flusso solo se valido" (`flowGate`,
riga 918), spenta per impostazione predefinita.

## Comandi

Ogni comando è `{ type, payload }`; `reduce(state, command)` restituisce
`{ state, result }` con `result` = `{ ok: true }` oppure
`{ ok: false, reason }` (il motivo testuale di etl-core quando esiste). Un
comando rifiutato non modifica lo stato. I comandi che creano nodi usano
le `PositionFn` e `settleNewNode` di etl-layout, secondo la tabella del
README di etl-layout.

| Comando             | Prototipo                                                      | Comportamento riportato                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Passo              |
| ------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| `addNode`           | 4939-5069 (rilascio dalla cassetta)                            | `pushHistory` (4996); id `ds-N`/`op-N` da `uidCounter` (4998); nome: libreria → nome del file, dataset → `Dataset N` (`dsCounter`, 5001), lavorazione → etichetta di `META`; parametri predefiniti, dalla libreria `path` e `columns` (5003-5006). Su un cavo (5009-5016): nasce in `wp - CARD/2`, poi `insertOnLink`. Su un nodo (5019-5046), secondo `paletteRelation` (4710-4723): fusione → nasce sul bersaglio e `performMerge`; collegamento → `freeSpot(round((t.x ∓ 150)/GRID)·GRID, t.y)`, `connect`, `resolveOverlaps(uid, uid, true)`. Nessuna relazione possibile: come nel vuoto. Nel vuoto (5048-5065): `freeSpot` sul punto allineato alla griglia, postazione in Organizzato, `resolveOverlaps(uid, uid, true)` | sì                 |
| `moveNodes`         | 4619-4627 (`nudgeSelection`)                                   | rifiutato in Organizzato; `pushHistory`; sposta e `clampCard`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | sì                 |
| `dropNodes` e gesto | 1959-2105 (trascinamento di un nodo)                           | `pushHistory` alla partenza (1983); durante: in Libero le coppie incompatibili si scansano (`resolveOverlaps(uid, uid, false, true)`, 2009) e un nodo incompatibile sotto il puntatore viene spinto via (`displace`, 2029-2033); al rilascio (2058-2105): gruppo → postazioni o separazione (2076-2081); su un cavo, se la lavorazione è slegata → `insertOnLink` (2083); su un nodo compatibile → il nodo torna al suo posto e fusione/collegamento (2084-2091); in Organizzato → postazione più vicina, con scambio (2093-2100); in Libero → `resolveOverlaps(uid, null, true)` (2102)                                                                                                                                        | sì (uno per gesto) |
| `connect`           | 3974-4024 (collegamento dalle porte)                           | `relation`; fusione e spinta escluse ("dalle porte si collega soltanto", 3995); `pushHistory` (4017); `connect` nel verso giusto (4019-4020); l'output nasce (1884, `refreshOutput`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | sì                 |
| `merge`             | 1774-1840 (`performMerge`)                                     | fusione (etl-core `mergeBoxes`), poi `relocateAfter` del box e dei suoi output (1826-1834)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | sì                 |
| `insertOnLink`      | 1853-1873                                                      | nodo a metà del cavo, postazione in Organizzato, riallacciamento, `resolveOverlaps(uid, null, true)`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | sì                 |
| `detachStep`        | 2137-2203                                                      | `pushHistory` (2140); nuovo nodo al punto di rilascio o sotto il box (etl-layout, correzione C5); `enforceCapacity`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | sì                 |
| `deleteStep`        | 2205-2248                                                      | `pushHistory` (2208); passaggio eliminato, `enforceCapacity`, output aggiornato                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | sì                 |
| `reorderSteps`      | 2330-2338 (vista espansa), 3834-3843 (inspector)               | `pushHistory` (2331, 3838); componenti e parametri spostati insieme; il passaggio selezionato segue                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | sì                 |
| `deleteNodes`       | 4480-4506 (`commitDelete`)                                     | `pushHistory` (4481); nodi e output rimasti senza produttore; gli output di box che hanno ancora ingressi rinascono (`renderAll`, 4407)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | sì                 |
| `deleteLink`        | 4565-4573                                                      | `pushHistory` (4568); `pruneOutputs`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | sì                 |
| `duplicate`         | 4594-4618 (`duplicateSelection`)                               | output esclusi; `pushHistory` (4597); copie a `+GRID·2`, nome `… copia` (4604-4605); postazioni in Organizzato, altrimenti separazione se si sovrappongono (4616); le copie diventano la selezione (4617)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | sì                 |
| `setParams`         | inspector (3515-3900): modifica diretta di `cards[uid].params` | sostituisce i parametri di un passaggio                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | sì (aggiunta)      |
| `renameNode`        | 3860-3870 (nome nell'inspector, al blur)                       | spazi tolti; un nome vuoto viene rifiutato (nel prototipo restava il nome precedente)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | sì (aggiunta)      |
| `setMode`           | 3952-3966                                                      | stessa modalità → nulla; `pushHistory` (3954); Organizzato → `assignSlots`; Libero → postazioni dimenticate                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | sì                 |
| `autoLayout`        | 4220-4329                                                      | canvas vuoto → nulla; `pushHistory` (4223); riordino (etl-layout) sull'area visibile passata nel comando                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | sì                 |
| `select`            | 2693-2712 (`toggleInSelection`, `setSelection`)                | identificativi esistenti, senza duplicati                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | no                 |
| `inspect`           | 2713-2727 (`selectCard`, `deselect`), 3830                     | nodo e passaggio dell'inspector                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | no                 |
| `loadDataset`       | 4921-4937 (caricamento di un CSV)                              | senza colonne → "Il file non contiene colonne leggibili"; id `lib-N` (`libCounter`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | no                 |
| `setPanel`          | 4826-4877 (`applyOpen`, `setPanelOpen`, `setSide`)             | aprire un pannello chiude l'altro sullo stesso bordo (4837-4840); cambiare lato chiude, sposta e riapre (4867-4876)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | no                 |
| `setView`           | 937-946, 4104-4110 (`zoomAt`)                                  | zoom limitato tra 0,35 e 2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | no                 |
| `setOptions`        | 918 (`flowGate`)                                               | `flowOnlyIfValid`, `maxBends` (intero ≥ 0)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | no                 |

`store.loadCsv(text, fileName)` legge il CSV (etl-core `parseCSV`, come il
prototipo a 4926) e invia `loadDataset` con i SOLI metadati: nome,
percorso, colonne (con i valori distinti, come nel prototipo, per i
selettori), numero di righe. Il testo del file non entra mai nello stato
né nel registro.

## Cronologia

Nel prototipo (righe 4333-4404):

- profondità `HIST_MAX = 50` (riga 4335): oltre, il passo più vecchio si
  scarta;
- un punto di cronologia è `snapshot()` (riga 4336-4338):
  `JSON.stringify({ cards, linksArr, comboCounter, outCounter, uidCounter })`,
  cioè tutti i nodi (con parametri, posizioni, postazioni), tutti i
  collegamenti e i contatori di nomi e id;
- `pushHistory()` (4343-4348) salva lo stato PRIMA dell'azione e svuota i
  passi da ripristinare;
- `undo()` (4392-4397) mette lo stato attuale fra i passi da ripristinare
  e ripristina l'ultimo punto; `redo()` (4398-4404) fa il contrario, con lo
  stesso limite.

Qui un punto di cronologia (`HistoryEntry`) contiene:

| Prototipo                                     | Qui                                                                |
| --------------------------------------------- | ------------------------------------------------------------------ |
| `cards`, `linksArr`                           | `graph` (nodi e collegamenti, immutabili)                          |
| `uidCounter`                                  | `counters.uid`                                                     |
| `comboCounter`, `outCounter`                  | derivati dal grafo in etl-core (nomi "Combined Box N", "Output N") |
| — (`dsCounter` non è nel punto del prototipo) | `counters.ds`                                                      |
| —                                             | `mode`                                                             |

Aggiunte rispetto al prototipo, e perché:

- **`mode`**: `setMode` crea un passo (riga 3954) ma il prototipo non
  salva la modalità, quindi annullarlo lasciava la modalità Organizzato
  con i nodi privi di postazione. Qui annullare riporta anche la
  modalità.
- **`counters.ds`**: senza, annullare e ripetere l'aggiunta di un dataset
  darebbe "Dataset 2" invece di "Dataset 1"; con esso il ripristino è
  identico.
- **`setParams` e `renameNode` creano un passo**: nel prototipo la
  modifica dei parametri nell'inspector non chiama `pushHistory`, quindi
  "annulla" la cancellava insieme all'azione precedente. Qui ogni modifica
  è un passo (per la digitazione, l'interfaccia può raggruppare più
  modifiche in un gesto).

Regole:

- Un gesto è un solo passo: `beginGesture` / `updateGesture` /
  `commitGesture` / `cancelGesture`. Gli aggiornamenti sono transitori:
  nessun passo, nessuna voce di registro. Solo il rilascio crea un passo,
  con lo stato di prima del gesto. `cancelGesture` torna allo stato di
  partenza.
- Un nuovo comando dopo un annullamento cancella i passi da ripristinare.
- Selezione, inspector, vista, pannelli, opzioni e libreria non entrano
  nella cronologia.
- Un comando rifiutato, o riuscito ma senza effetto sul contenuto dei
  punti (es. `setMode` sulla modalità attuale), non crea un passo.
- Annullare e ripristinare tolgono dalla selezione e dall'inspector i nodi
  che non esistono più.
- Un comando, un annullamento o un nuovo gesto durante un gesto in corso
  lo annullano prima.

## Registro delle attività

Elenco in sola aggiunta (`store.getLog()`, `store.exportLog()` → JSON) di
tutti i comandi, compresi quelli rifiutati, di annulla/ripristina e dei
gesti conclusi: `{ id, time, type, payload, result }`. Un gesto si
registra una sola volta, al rilascio, come
`{ type: "gesture", payload: { kind: "move", ids, from, to, target } }` con
le posizioni iniziali e finali; gli aggiornamenti transitori e i gesti
annullati non si registrano. Il payload è una copia JSON. Nessun invio a
server in questa fase.

## Salvataggio

- Chiave: `isa.etl.v2.<solutionId>`. I dati delle chiavi precedenti non si
  leggono né si cancellano.
- Contenuto: `{ version: 1, graph, mode, library, panels, options }`. Non
  si salvano cronologia, selezione, inspector, vista, percorsi. I
  contatori si ricavano dagli id al caricamento, così i nuovi id non
  collidono.
- Scrittura differita di 400 ms dopo l'ultima modifica di ciò che si
  salva (`persist(store, solutionId)`), mai durante un gesto.
- Caricamento con validazione (`fromSaved`): un dato non valido, incoerente
  o di versione sconosciuta viene ignorato senza errori, partendo da un
  canvas vuoto. Sul server (nessun localStorage) si parte da un canvas
  vuoto e non si scrive nulla.

## React

```ts
import { EtlStoreProvider, useEtlState, usePersistentEtlStore } from "@/etl-store/react";

const store = usePersistentEtlStore(solutionId); // carica, salva con scrittura differita
<EtlStoreProvider store={store}>…</EtlStoreProvider>
const mode = useEtlState((s) => s.mode);         // useSyncExternalStore
```

## Differenze rispetto al prototipo

- Canvas iniziale vuoto e libreria vuota (il prototipo partiva con un
  flusso dimostrativo e il dataset "Vendite 2026", righe 4669, 5077-5082);
  il primo dataset dalla cassetta si chiama quindi "Dataset 1".
- Punti di cronologia: aggiunte descritte sopra (`mode`, `counters.ds`,
  passi per `setParams` e `renameNode`).
- Un comando esplicito con un bersaglio impossibile (cavo non inseribile,
  nodo inesistente) viene rifiutato con il motivo; l'interfaccia del
  prototipo non poteva inviarlo. Rilasciando su un nodo senza relazione
  possibile, come nel prototipo, si rilascia nel vuoto.
```

### `src/etl-store/__tests__/helpers.ts`

57 righe

```ts
import { expect } from "vitest";
import type { Card, ColumnDef, Graph } from "../../etl-core";
import { initialState, reduce } from "..";
import type { Command, EtlState } from "..";

/** Applica un comando che DEVE riuscire. */
export function ok(state: EtlState, command: Command): EtlState {
  const out = reduce(state, command);
  expect(out.result, `${command.type}: ${JSON.stringify(out.result)}`).toEqual({ ok: true });
  return out.state;
}

/** Applica un comando che DEVE essere rifiutato: restituisce il motivo e verifica che lo stato non cambi. */
export function refused(state: EtlState, command: Command): string {
  const out = reduce(state, command);
  expect(out.result.ok, command.type).toBe(false);
  expect(out.state).toBe(state);
  return out.result.ok ? "" : out.result.reason;
}

export const COLUMNS: ColumnDef[] = [
  { name: "id", type: "integer", values: ["1", "2", "3"] },
  { name: "regione", type: "stringa", values: ["Nord", "Sud"] },
  { name: "importo", type: "numerico", values: ["10", "20.5"] },
];

export function card(graph: Graph, id: string): Card {
  const c = graph.cards[id];
  expect(c, `nodo ${id}`).toBeDefined();
  return c as Card;
}

/** Id dei nodi creati da un comando. */
export function createdIds(before: EtlState, after: EtlState): string[] {
  return Object.keys(after.graph.cards).filter((id) => !before.graph.cards[id]);
}

/** Uno stato con un dataset A (colonne note) e un filtro F, scollegati. */
export function withDatasetAndFilter(): { state: EtlState; ds: string; filter: string } {
  let s = initialState();
  s = ok(s, {
    type: "loadDataset",
    payload: { name: "vendite", path: "vendite.csv", columns: COLUMNS, rows: 3 },
  });
  const s1 = ok(s, {
    type: "addNode",
    payload: { component: "dataset", libraryId: "lib-1", point: { x: 200, y: 300 } },
  });
  const ds = createdIds(s, s1)[0] as string;
  const s2 = ok(s1, {
    type: "addNode",
    payload: { component: "filter", point: { x: 700, y: 300 } },
  });
  const filter = createdIds(s1, s2)[0] as string;
  return { state: s2, ds, filter };
}
```

### `src/etl-store/__tests__/persistence.test.ts`

165 righe

```ts
import { readFileSync, readdirSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { SAVE_VERSION, createEtlStore, fromSaved, initialState, parseSaved, toSaved } from "..";
import {
  SAVE_DELAY_MS,
  loadState,
  persist,
  saveState,
  storageKey,
  browserStorage,
} from "../persistence";
import type { StorageLike } from "../persistence";
import { ok, withDatasetAndFilter } from "./helpers";

function memoryStorage(initial: Record<string, string> = {}): StorageLike & {
  data: Record<string, string>;
  writes: number;
} {
  const data: Record<string, string> = { ...initial };
  return {
    data,
    writes: 0,
    getItem(k) {
      return k in data ? (data[k] as string) : null;
    },
    setItem(k, v) {
      data[k] = v;
      this.writes++;
    },
  };
}

function richState() {
  const { state, ds, filter } = withDatasetAndFilter();
  let s = ok(state, { type: "connect", payload: { from: ds, to: filter } });
  s = ok(s, { type: "setMode", payload: { mode: "grid" } });
  s = ok(s, { type: "setPanel", payload: { panel: "insp", side: "top" } });
  s = ok(s, { type: "setOptions", payload: { flowOnlyIfValid: true, maxBends: 2 } });
  s = ok(s, { type: "select", payload: { ids: [ds] } });
  return s;
}

describe("salvataggio", () => {
  it("andata e ritorno senza perdite (grafo, modalità, libreria, pannelli, opzioni)", () => {
    const s = richState();
    const back = parseSaved(JSON.stringify(toSaved(s)));
    expect(back).not.toBeNull();
    expect(back?.graph).toEqual(s.graph);
    expect(back?.mode).toBe(s.mode);
    expect(back?.library).toEqual(s.library);
    expect(back?.panels).toEqual(s.panels);
    expect(back?.options).toEqual(s.options);
    // non si salvano: selezione, vista, inspector
    expect(back?.selection).toEqual([]);
    expect(Object.keys(toSaved(s)).sort()).toEqual([
      "graph",
      "library",
      "mode",
      "options",
      "panels",
      "version",
    ]);
  });

  it("dopo il caricamento gli id nuovi non collidono con quelli salvati", () => {
    const s = richState();
    const back = parseSaved(JSON.stringify(toSaved(s))) ?? initialState();
    expect(back.counters.uid).toBe(s.counters.uid);
    expect(back.counters.lib).toBe(s.counters.lib);
    const next = ok(back, {
      type: "addNode",
      payload: { component: "sort", point: { x: 1500, y: 1200 } },
    });
    expect(Object.keys(next.graph.cards)).toHaveLength(Object.keys(s.graph.cards).length + 1);
  });

  it("dato corrotto, versione sconosciuta, grafo incoerente: ignorati senza errori", () => {
    const good = toSaved(richState());
    expect(parseSaved("{non è json")).toBeNull();
    expect(parseSaved(null)).toBeNull();
    expect(fromSaved({ ...good, version: SAVE_VERSION + 1 })).toBeNull();
    expect(fromSaved({ ...good, mode: "altro" })).toBeNull();
    expect(
      fromSaved({ ...good, graph: { cards: {}, links: [{ from: "a", to: "b" }] } }),
    ).toBeNull();
    expect(fromSaved({ ...good, graph: { cards: { x: { id: "y" } }, links: [] } })).toBeNull();
    expect(fromSaved(42)).toBeNull();
  });

  it("chiave nuova per soluzione; i dati delle chiavi precedenti non si leggono né si cancellano", () => {
    const old = { "isa.etl.sol-1": JSON.stringify(toSaved(richState())), "isa-etl-canvas": "{}" };
    const storage = memoryStorage(old);
    expect(storageKey("sol-1")).toBe("isa.etl.v2.sol-1");
    expect(loadState("sol-1", storage)).toEqual(initialState());
    saveState("sol-1", richState(), storage);
    expect(storage.data["isa.etl.sol-1"]).toBe(old["isa.etl.sol-1"]);
    expect(storage.data["isa-etl-canvas"]).toBe("{}");
    expect(loadState("sol-1", storage).mode).toBe("grid");
    expect(loadState("sol-2", storage)).toEqual(initialState());
  });

  it("dato non valido in localStorage: canvas vuoto", () => {
    const storage = memoryStorage({ "isa.etl.v2.s": '{"version":99}' });
    expect(loadState("s", storage)).toEqual(initialState());
  });

  it("in Node (senza localStorage) il caricamento parte da un canvas vuoto e il salvataggio non fa nulla", () => {
    expect(browserStorage()).toBeNull();
    expect(loadState("s")).toEqual(initialState());
    expect(saveState("s", initialState())).toBe(false);
  });
});

describe("scrittura differita", () => {
  beforeEach(() => vi.useFakeTimers());
  afterEach(() => vi.useRealTimers());

  it(`scrive una sola volta, ${SAVE_DELAY_MS} ms dopo l'ultima modifica`, () => {
    const { state, ds } = withDatasetAndFilter();
    const store = createEtlStore({ initial: state });
    const storage = memoryStorage();
    const handle = persist(store, "sol", storage);
    for (let i = 0; i < 5; i++) {
      store.dispatch({ type: "moveNodes", payload: { ids: [ds], dx: 2, dy: 0 } });
      vi.advanceTimersByTime(300);
    }
    expect(storage.writes).toBe(0);
    vi.advanceTimersByTime(SAVE_DELAY_MS - 300 - 1);
    expect(storage.writes).toBe(0);
    vi.advanceTimersByTime(1);
    expect(storage.writes).toBe(1);
    expect(loadState("sol", storage).graph).toEqual(store.getState().graph);
    // selezione e vista non provocano scritture
    store.dispatch({ type: "select", payload: { ids: [ds] } });
    store.dispatch({ type: "setView", payload: { x: 5 } });
    vi.advanceTimersByTime(SAVE_DELAY_MS * 2);
    expect(storage.writes).toBe(1);
    handle.stop();
  });
});

describe("nucleo puro", () => {
  const dir = resolve(dirname(fileURLToPath(import.meta.url)), "..");

  it("il nucleo si importa in Node senza errori", async () => {
    const mod = await import("..");
    expect(typeof mod.createEtlStore).toBe("function");
    expect(typeof mod.reduce).toBe("function");
    expect(typeof globalThis.document).toBe("undefined");
  });

  it("fuori da react.ts e persistence.ts nessun uso di react, window, document, localStorage", () => {
    const core = readdirSync(dir).filter(
      (f) => f.endsWith(".ts") && f !== "react.ts" && f !== "persistence.ts",
    );
    expect(core.length).toBeGreaterThanOrEqual(6);
    for (const f of core) {
      const src = readFileSync(resolve(dir, f), "utf8");
      expect(src, f).not.toMatch(/from "react"|\bwindow\b|\bdocument\b|localStorage/);
    }
  });
});
```

### `src/etl-store/__tests__/react.test.ts`

32 righe

```ts
import { createElement } from "react";
import { renderToString } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { createEtlStore } from "..";
import { EtlStoreProvider, useEtlState, usePersistentEtlStore } from "../react";

function Mode(): string {
  return "modalità " + useEtlState((s) => s.mode);
}

function Persistent(): string {
  const store = usePersistentEtlStore("ssr");
  return "nodi " + Object.keys(useEtlState((s) => s.graph, store).cards).length;
}

describe("collegamento a React", () => {
  it("useEtlState legge lo stato del provider (rendering lato server)", () => {
    const store = createEtlStore();
    store.dispatch({ type: "setMode", payload: { mode: "grid" } });
    const html = renderToString(createElement(EtlStoreProvider, { store }, createElement(Mode)));
    expect(html).toContain("modalità grid");
  });

  it("usePersistentEtlStore sul server parte da un canvas vuoto, senza errori", () => {
    expect(renderToString(createElement(Persistent))).toContain("nodi 0");
  });

  it("fuori dal provider l'errore è esplicito", () => {
    expect(() => renderToString(createElement(Mode))).toThrow(/EtlStoreProvider/);
  });
});
```

### `src/etl-store/__tests__/reduce.test.ts`

447 righe

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
```

