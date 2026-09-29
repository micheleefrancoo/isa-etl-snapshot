# VALIDATION REPORT — Fase 3: stato dell'applicazione (`src/etl-store/`)

Generato: 2026-09-29T09:28:26Z (UTC) — branch `feat/etl-store`

## 1. Esiti

Installazione pulita prima della validazione: `rm -rf node_modules && npm ci`
(eseguito dall'utente, perché il sistema di permessi della sessione lo
blocca: 441 pacchetti, 0 vulnerabilità).

```
npx tsc --noEmit  → PASS, 0 errori
npm run lint      → 0 errori, 14 warning (tutti pre-esistenti, react-refresh/only-export-components;
                    nessuno in src/etl-store/)
npm test          → 275/275 test passati, 20 file
npm run build     → riuscita
```

Grep `from "react"|window|document|localStorage` nei `.ts` di
`src/etl-store/` esclusi `react.ts` e `persistence.ts`: **nessun
risultato** (verificato anche da un test, `persistence.test.ts` › "nucleo
puro").

### Test per file

| File                                                  |    Test |
| ----------------------------------------------------- | ------: |
| src/canvas/\_\_tests\_\_/panelPositioning.test.ts     |       9 |
| src/canvas/layout/\_\_tests\_\_/canvasBounds.test.ts  |      13 |
| src/canvas/layout/\_\_tests\_\_/dropZones.test.ts     |       9 |
| src/canvas/layout/\_\_tests\_\_/panelRegistry.test.ts |      10 |
| src/etl-core/\_\_tests\_\_/csv.test.ts                |       9 |
| src/etl-core/\_\_tests\_\_/expressions.test.ts        |       7 |
| src/etl-core/\_\_tests\_\_/fase11-requisiti.test.ts   |      30 |
| src/etl-core/\_\_tests\_\_/fase11.test.ts             |      11 |
| src/etl-core/\_\_tests\_\_/mutations.test.ts          |       9 |
| src/etl-core/\_\_tests\_\_/params.test.ts             |      11 |
| src/etl-core/\_\_tests\_\_/relations.test.ts          |      11 |
| src/etl-core/\_\_tests\_\_/schema.test.ts             |       5 |
| src/etl-core/\_\_tests\_\_/state.test.ts              |      26 |
| src/etl-layout/\_\_tests\_\_/golden.test.ts           |      17 |
| src/etl-layout/\_\_tests\_\_/properties.test.ts       |      11 |
| src/etl-layout/\_\_tests\_\_/unit.test.ts             |      22 |
| **src/etl-store/\_\_tests\_\_/persistence.test.ts**   |   **9** |
| **src/etl-store/\_\_tests\_\_/react.test.ts**         |   **3** |
| **src/etl-store/\_\_tests\_\_/reduce.test.ts**        |  **40** |
| **src/etl-store/\_\_tests\_\_/store.test.ts**         |  **13** |
| **Totale**                                            | **275** |

Rispetto alla Fase 2.1 (210): +65, tutti in etl-store.

## 2. Passo 0 — un solo gestore di pacchetti

Rimossi `bun.lock` e `bunfig.toml`. Resta solo npm (`package-lock.json`).

## 3. Comandi e righe del prototipo

| Comando                                                           | Prototipo (righe)                      | Passo di cronologia      |
| ----------------------------------------------------------------- | -------------------------------------- | ------------------------ |
| `addNode` (cassetta, libreria; nel vuoto, su un nodo, su un cavo) | 4939-5069, `paletteRelation` 4710-4723 | sì (4996)                |
| `moveNodes` (tastiera)                                            | 4619-4627                              | sì (4623)                |
| `dropNodes` / gesto `beginGesture`… `commitGesture`               | 1959-2105                              | sì, uno per gesto (1983) |
| `connect` (dalle porte)                                           | 3974-4024                              | sì (4017)                |
| `merge`                                                           | 1774-1840                              | sì                       |
| `insertOnLink`                                                    | 1853-1873                              | sì                       |
| `detachStep`                                                      | 2137-2203                              | sì (2140)                |
| `deleteStep`                                                      | 2205-2248                              | sì (2208)                |
| `reorderSteps`                                                    | 2330-2338, 3834-3843                   | sì (2331, 3838)          |
| `deleteNodes`                                                     | 4480-4506                              | sì (4481)                |
| `deleteLink`                                                      | 4565-4573                              | sì (4568)                |
| `duplicate`                                                       | 4594-4618                              | sì (4597)                |
| `setParams`                                                       | inspector 3515-3900                    | sì (aggiunta)            |
| `renameNode`                                                      | 3860-3870                              | sì (aggiunta)            |
| `setMode`                                                         | 3952-3966                              | sì (3954)                |
| `autoLayout`                                                      | 4220-4329                              | sì (4223)                |
| `select`                                                          | 2693-2712                              | no                       |
| `inspect`                                                         | 2713-2727, 3830                        | no                       |
| `loadDataset` (e `store.loadCsv`)                                 | 4921-4937                              | no                       |
| `setPanel`                                                        | 4826-4877                              | no                       |
| `setView`                                                         | 937-946, 4104-4110                     | no                       |
| `setOptions`                                                      | 918                                    | no                       |
| `undo` / `redo`                                                   | 4392-4404                              | —                        |

Il comportamento riportato riga per riga è nella tabella "Comandi" di
`src/etl-store/README.md`. Ogni comando ha almeno un test riuscito e uno
rifiutato in `reduce.test.ts`.

## 4. Contenuto dei punti di cronologia

Prototipo (`snapshot()`, righe 4336-4338):
`{ cards, linksArr, comboCounter, outCounter, uidCounter }`; profondità
`HIST_MAX = 50` (riga 4335).

Qui (`HistoryEntry`), profondità 50:

| Prototipo                    | etl-store                     |
| ---------------------------- | ----------------------------- |
| `cards`, `linksArr`          | `graph`                       |
| `uidCounter`                 | `counters.uid`                |
| `comboCounter`, `outCounter` | derivati dal grafo (etl-core) |
| —                            | `counters.ds` (aggiunta)      |
| —                            | `mode` (aggiunta)             |

Aggiunte motivate (dettaglio nel README):

- `mode`: nel prototipo annullare `setMode` lasciava la modalità
  Organizzato con i nodi senza postazione.
- `counters.ds`: il ripristino dà esattamente gli stessi nomi.
- `setParams`, `renameNode` creano un passo: nel prototipo le modifiche
  nell'inspector non chiamavano `pushHistory` e venivano annullate insieme
  all'azione precedente.

Esiti dei test di cronologia (`store.test.ts`): trascinamento di 30
aggiornamenti = 1 passo; annulla riporta la posizione iniziale,
ripristina la finale; limite di 50 passi; un comando dopo un annullamento
cancella i passi da ripristinare; selezione, inspector, vista, pannelli,
opzioni e libreria non creano passi; scenario completo (CSV → dataset →
filtro → parametri → join con un secondo dataset: 8 passi, annullati fino
al canvas vuoto e ripristinati con grafo e contatori identici).

## 5. Registro e salvataggio

- Registro: comandi rifiutati registrati con il motivo; un gesto = una
  voce con posizioni iniziali e finali; esportazione JSON valida; il
  contenuto dei CSV non entra nel registro (solo nome, percorso, colonne,
  righe).
- Salvataggio: chiave `isa.etl.v2.<solutionId>`, formato versione 1;
  andata e ritorno senza perdite; dati corrotti, incoerenti o di versione
  sconosciuta ignorati; chiavi precedenti né lette né cancellate;
  scrittura differita di 400 ms (una sola scrittura per una raffica di
  modifiche; selezione e vista non scrivono); nel server/Node canvas
  vuoto senza errori.

## 6. Residui di Lovable (per una pulizia separata)

Non toccati in questa fase:

| Residuo                                       | Dove                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| cartella `.lovable/`                          | `.lovable/project.json`, `.lovable/plan/*.md`                                                                |
| pacchetto `@lovable.dev/vite-tanstack-config` | `package.json` (devDependencies), `package-lock.json`, importato da `vite.config.ts`                         |
| `src/lib/lovable-error-reporting.ts`          | importato da `src/routes/__root.tsx` (`reportLovableError`)                                                  |
| riferimenti a Lovable nei documenti           | `README.md` (righe 17, 21: link all'editor), `AGENTS.md` (avviso sulla cronologia pubblicata)                |
| riferimenti a Lovable negli script            | `scripts/generate-snapshot.mjs` (file `.lovable/` inclusi nello snapshot; `bun.lock` fra i nomi di lockfile) |

## 7. Note

- Lo script dello snapshot non ha un'area dedicata a `src/etl-store/`, che
  finisce in "13-misc": aggiungerla richiede una modifica fuori da
  `src/etl-store/`, esclusa da questa fase.
- Differenze rispetto al prototipo, tutte nel README: canvas e libreria
  iniziali vuoti; aggiunte ai punti di cronologia; comandi espliciti con
  bersagli impossibili rifiutati con motivo; nome vuoto rifiutato invece
  di mantenere il precedente.
