# 01d-etl-store-a.md

File in questo blocco:

- `src/etl-store/README.md`
- `src/etl-store/__tests__/grouping.test.ts`
- `src/etl-store/__tests__/helpers.ts`
- `src/etl-store/__tests__/persistence.test.ts`
- `src/etl-store/__tests__/react.test.ts`

---

### `src/etl-store/README.md`

227 righe

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
| `clearAll`          | (assente nel prototipo: barra dei controlli, Fase 6a.2)        | elimina nodi e collegamenti; libreria, contatori e modalità restano                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | sì                 |
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
  che non esistono più; se il nodo dell'inspector esiste ma ha meno
  passaggi (es. annullando una fusione), l'indice va all'ultimo passaggio
  esistente (Fase 3.1).
- Un comando, un annullamento o un nuovo gesto durante un gesto in corso
  lo annullano prima.

### Raggruppamento dei comandi consecutivi (Fase 3.1)

Scrivere in un campo o tenere premuta una freccia non produce un passo per
ogni carattere o pressione.

- Chiave di raggruppamento (`groupKey`): `setParams` → nodo + indice del
  passaggio; `renameNode` → nodo; `moveNodes` → insieme degli
  identificativi, ordinato. Gli altri comandi non si raggruppano.
- Un comando riuscito con la stessa chiave del comando precedente, arrivato
  entro 1000 ms (`GROUP_WINDOW_MS`) da quello, si unisce al passo
  precedente: la cronologia non aggiunge un passo (resta lo stato prima del
  primo comando del gruppo) e il registro aggiorna l'ultima voce invece di
  aggiungerne una (payload e risultato dell'ultimo comando, `time` del
  primo, `until` con l'istante dell'ultimo, `count` con il numero di
  comandi uniti).
- Il gruppo si interrompe con qualunque altro comando (anche un comando
  rifiutato, che si registra a parte), annulla, ripristina o gesto, e dopo
  più di 1000 ms di pausa.
- Un solo annulla riporta allo stato prima del primo comando del gruppo.
- L'orologio è quello iniettabile dello store (`now`): i test sono
  deterministici.

## Registro delle attività

Elenco in sola aggiunta (`store.getLog()`, `store.exportLog()` → JSON) di
tutti i comandi, compresi quelli rifiutati, di annulla/ripristina e dei
gesti conclusi: `{ id, time, type, payload, result }`, più `until` e
`count` per le voci che raggruppano più comandi (vedi sopra: è l'unico caso
in cui una voce si aggiorna invece di aggiungerne una). `setView` non entra
nel registro (Fase 3.1): cambia decine di volte al secondo e non descrive il
lavoro dell'utente. Un gesto si
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
- Versione del formato: 2 (Fase 6b.0, colonne multiple). Un salvataggio v1 si
  carica eseguendo su ogni card le migrazioni dei parametri (`ensureParams`
  di etl-core); un v2 si carica com'è.

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

### `src/etl-store/__tests__/grouping.test.ts`

198 righe

```ts
import { describe, expect, it } from "vitest";
import type { FilterParams, Params } from "../../etl-core";
import { GROUP_WINDOW_MS, createEtlStore } from "..";
import type { Command, EtlState, EtlStore } from "..";
import { createdIds, ok, withDatasetAndFilter } from "./helpers";

/** Store con un orologio manuale: `advance(ms)` sposta l'istante del prossimo comando. */
function clockStore(initial: EtlState): { store: EtlStore; advance: (ms: number) => void } {
  let t = 1_000_000;
  const store = createEtlStore({ initial, now: () => t });
  return { store, advance: (ms) => (t += ms) };
}

function filterParams(value: string): Params {
  const p: FilterParams = {
    conditions: [{ column: "regione", op: "=", mode: "list", values: [value], text: "", sep: "," }],
  };
  return p as unknown as Params;
}

function setParams(node: string, index: number, value: string): Command {
  return { type: "setParams", payload: { node, index, params: filterParams(value) } };
}

describe("raggruppamento dei comandi consecutivi (Fase 3.1)", () => {
  it("30 setParams sullo stesso passaggio a 50 ms: un passo, una voce con count 30; un annulla torna all'inizio", () => {
    const { state, filter } = withDatasetAndFilter();
    const { store, advance } = clockStore(state);
    const start = store.getState().graph;
    const t0 = 1_000_000;
    for (let i = 1; i <= 30; i++) {
      if (i > 1) advance(50);
      expect(store.dispatch(setParams(filter, 0, "N".repeat(i)))).toEqual({ ok: true });
    }
    expect(store.historySize()).toEqual({ past: 1, future: 0 });
    expect(store.getLog()).toHaveLength(1);
    const entry = store.getLog()[0];
    expect(entry).toMatchObject({ type: "setParams", count: 30, time: t0, until: t0 + 29 * 50 });
    expect(entry?.payload).toMatchObject({ node: filter, index: 0 });
    expect(JSON.stringify(entry?.payload)).toContain("N".repeat(30));
    store.undo();
    expect(store.getState().graph).toBe(start);
  });

  it("due gruppi sullo stesso passaggio separati da 1500 ms: due passi", () => {
    const { state, filter } = withDatasetAndFilter();
    const { store, advance } = clockStore(state);
    for (let i = 0; i < 5; i++) {
      store.dispatch(setParams(filter, 0, "A" + i));
      advance(50);
    }
    advance(1500);
    for (let i = 0; i < 5; i++) {
      store.dispatch(setParams(filter, 0, "B" + i));
      advance(50);
    }
    expect(store.historySize().past).toBe(2);
    expect(store.getLog().map((e) => e.count)).toEqual([5, 5]);
  });

  it(`il limite è ${GROUP_WINDOW_MS} ms dal comando precedente, non dal primo`, () => {
    const { state, filter } = withDatasetAndFilter();
    const { store, advance } = clockStore(state);
    for (let i = 0; i < 10; i++) {
      store.dispatch(setParams(filter, 0, "x" + i));
      advance(900);
    }
    expect(store.historySize().past).toBe(1);
    advance(GROUP_WINDOW_MS);
    store.dispatch(setParams(filter, 0, "dopo la pausa"));
    expect(store.historySize().past).toBe(2);
  });

  it("setParams su due passaggi diversi, alternati: nessun raggruppamento", () => {
    const { state, filter } = withDatasetAndFilter();
    let s = ok(state, {
      type: "addNode",
      payload: { component: "filter", point: { x: 900, y: 900 } },
    });
    const other = createdIds(state, s)[0] as string;
    s = ok(s, { type: "merge", payload: { dragged: other, target: filter } });
    const { store, advance } = clockStore(s);
    for (let i = 0; i < 6; i++) {
      store.dispatch(setParams(filter, i % 2, "v" + i));
      advance(50);
    }
    expect(store.historySize().past).toBe(6);
    expect(store.getLog()).toHaveLength(6);
    expect(store.getLog().every((e) => e.count === undefined)).toBe(true);
  });

  it("20 moveNodes della stessa selezione: un passo; cambiando selezione a metà: due passi", () => {
    const { state, ds, filter } = withDatasetAndFilter();
    const a = clockStore(state);
    for (let i = 0; i < 20; i++) {
      a.store.dispatch({ type: "moveNodes", payload: { ids: [filter, ds], dx: 2, dy: 0 } });
      a.advance(30);
    }
    expect(a.store.historySize().past).toBe(1);
    expect(a.store.getLog()[0]?.count).toBe(20);

    const b = clockStore(state);
    for (let i = 0; i < 20; i++) {
      const ids = i < 10 ? [ds] : [ds, filter];
      b.store.dispatch({ type: "moveNodes", payload: { ids, dx: 0, dy: 2 } });
      b.advance(30);
    }
    expect(b.store.historySize().past).toBe(2);
  });

  it("l'ordine degli identificativi non conta per la chiave", () => {
    const { state, ds, filter } = withDatasetAndFilter();
    const { store, advance } = clockStore(state);
    store.dispatch({ type: "moveNodes", payload: { ids: [ds, filter], dx: 2, dy: 0 } });
    advance(30);
    store.dispatch({ type: "moveNodes", payload: { ids: [filter, ds], dx: 2, dy: 0 } });
    expect(store.historySize().past).toBe(1);
  });

  it("un annulla in mezzo interrompe il gruppo: il comando successivo crea un nuovo passo", () => {
    const { state, filter } = withDatasetAndFilter();
    const { store, advance } = clockStore(state);
    store.dispatch({ type: "renameNode", payload: { node: filter, name: "A" } });
    advance(50);
    store.dispatch({ type: "renameNode", payload: { node: filter, name: "AB" } });
    advance(50);
    store.undo();
    advance(50);
    store.dispatch({ type: "renameNode", payload: { node: filter, name: "ABC" } });
    expect(store.historySize()).toEqual({ past: 1, future: 0 });
    expect(store.getLog().map((e) => [e.type, e.count ?? 1])).toEqual([
      ["renameNode", 2],
      ["undo", 1],
      ["renameNode", 1],
    ]);
  });

  it("qualunque altro comando interrompe il gruppo", () => {
    const { state, filter } = withDatasetAndFilter();
    const { store, advance } = clockStore(state);
    store.dispatch(setParams(filter, 0, "a"));
    advance(50);
    store.dispatch({ type: "select", payload: { ids: [filter] } });
    advance(50);
    store.dispatch(setParams(filter, 0, "b"));
    expect(store.historySize().past).toBe(2);
  });

  it("un gesto interrompe il gruppo", () => {
    const { state, ds } = withDatasetAndFilter();
    const { store, advance } = clockStore(state);
    store.dispatch({ type: "moveNodes", payload: { ids: [ds], dx: 2, dy: 0 } });
    advance(50);
    store.beginGesture({ ids: [ds] });
    store.updateGesture({ dx: 200, dy: 0 });
    store.commitGesture();
    advance(50);
    store.dispatch({ type: "moveNodes", payload: { ids: [ds], dx: 2, dy: 0 } });
    expect(store.historySize().past).toBe(3);
  });
});

describe("registro senza vista (Fase 3.1)", () => {
  it("100 setView: nessuna voce di registro; gli altri comandi restano registrati", () => {
    const { store, advance } = clockStore(withDatasetAndFilter().state);
    for (let i = 0; i < 100; i++) {
      expect(store.dispatch({ type: "setView", payload: { x: i, zoom: 1 + i / 200 } })).toEqual({
        ok: true,
      });
      advance(16);
    }
    expect(store.getLog()).toHaveLength(0);
    expect(store.getState().view.x).toBe(99);
    store.dispatch({ type: "setPanel", payload: { panel: "insp", open: true } });
    expect(store.getLog().map((e) => e.type)).toEqual(["setPanel"]);
  });
});

describe("inspector dopo annulla e ripristina (Fase 3.1)", () => {
  it("annullando una fusione con l'inspector sull'ultimo passaggio, l'indice va a un passaggio esistente", () => {
    const { state, filter } = withDatasetAndFilter();
    const s = ok(state, {
      type: "addNode",
      payload: { component: "sort", point: { x: 900, y: 900 } },
    });
    const sort = createdIds(state, s)[0] as string;
    const { store } = clockStore(s);
    store.dispatch({ type: "merge", payload: { dragged: sort, target: filter } });
    store.dispatch({ type: "inspect", payload: { node: filter, step: 1 } });
    expect(store.getState().inspector).toEqual({ nodeId: filter, step: 1 });
    store.undo();
    expect(store.getState().graph.cards[filter]?.components).toEqual(["filter"]);
    expect(store.getState().inspector).toEqual({ nodeId: filter, step: 0 });
    store.redo();
    expect(store.getState().inspector).toEqual({ nodeId: filter, step: 0 });
  });
});
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

177 righe

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
    // entrambi i pannelli aperti: si normalizza lasciando aperta solo la cassetta
    const both = fromSaved({
      ...good,
      panels: { tools: { side: "left", open: true }, insp: { side: "right", open: true } },
    });
    expect(both?.panels.tools.open).toBe(true);
    expect(both?.panels.insp.open).toBe(false);
    const onlyInsp = fromSaved({
      ...good,
      panels: { tools: { side: "left", open: false }, insp: { side: "right", open: true } },
    });
    expect(onlyInsp?.panels.insp.open).toBe(true);
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

