# VALIDATION REPORT — Fase 3.1: raggruppamento della cronologia e registro (`src/etl-store/`)

Generato: 2026-09-29T09:39:17Z (UTC) — branch `feat/etl-store-fix`

## 1. Esiti

```
npx tsc --noEmit  → PASS, 0 errori
npm run lint      → 0 errori, 14 warning (tutti pre-esistenti, react-refresh/only-export-components;
                    nessuno in src/etl-store/)
npm test          → 286/286 test passati, 21 file
npm run build     → riuscita
```

Grep React/API del browser nel nucleo di `src/etl-store/` (esclusi
`react.ts` e `persistence.ts`): nessun risultato.

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
| **src/etl-store/\_\_tests\_\_/grouping.test.ts**      |  **11** |
| src/etl-store/\_\_tests\_\_/persistence.test.ts       |       9 |
| src/etl-store/\_\_tests\_\_/react.test.ts             |       3 |
| src/etl-store/\_\_tests\_\_/reduce.test.ts            |      40 |
| src/etl-store/\_\_tests\_\_/store.test.ts             |      13 |
| **Totale**                                            | **286** |

Rispetto alla Fase 3 (275): +11 (`grouping.test.ts`). Il test del limite
di 50 passi in `store.test.ts` ora distanzia i comandi di 1500 ms: con il
raggruppamento, 60 `moveNodes` consecutivi sarebbero correttamente un solo
passo.

## 2. Modifiche

| #   | Modifica                                                                                                                                                                                                                                                                                                                                                       | Dove                                                                                                    |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| 1   | Raggruppamento: chiave `setParams` → nodo + passaggio, `renameNode` → nodo, `moveNodes` → id ordinati; stessa chiave entro 1000 ms dal comando precedente → nessun nuovo passo, voce di registro aggiornata (`time` del primo, `until`, `count`); interruzione con qualunque altro comando (anche rifiutato), annulla, ripristina, gesto o pausa oltre 1000 ms | `store.ts` (`groupKey`, `GROUP_WINDOW_MS`, `dispatch`), `types.ts` (`LogEntry.until`, `LogEntry.count`) |
| 2   | `setView` non entra nel registro                                                                                                                                                                                                                                                                                                                               | `store.ts` (`UNLOGGED_COMMANDS`)                                                                        |
| 3   | In `restore`, indice dell'inspector portato all'ultimo passaggio esistente                                                                                                                                                                                                                                                                                     | `store.ts` (`restore`)                                                                                  |
| 4   | Area dello snapshot `01d-etl-store`, tra etl-layout e i componenti                                                                                                                                                                                                                                                                                             | `scripts/generate-snapshot.mjs`                                                                         |

Scelta documentata: un comando rifiutato con la stessa chiave non si unisce
al gruppo, si registra a parte e interrompe il gruppo (la voce raggruppata
descrive solo comandi riusciti).

## 3. Test richiesti

| Requisito                                                                                                | Test (`grouping.test.ts`)                       | Esito |
| -------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | ----- |
| 30 `setParams` sullo stesso passaggio a 50 ms: 1 passo, 1 voce con count 30; un annulla torna all'inizio | "30 setParams sullo stesso passaggio…"          | ✅    |
| due gruppi separati da 1500 ms: due passi                                                                | "due gruppi sullo stesso passaggio…"            | ✅    |
| `setParams` alternati su due passaggi: nessun raggruppamento                                             | "setParams su due passaggi diversi, alternati…" | ✅    |
| 20 `moveNodes` della stessa selezione: 1 passo; cambiando selezione a metà: 2                            | "20 moveNodes della stessa selezione…"          | ✅    |
| un annulla in mezzo interrompe il gruppo                                                                 | "un annulla in mezzo interrompe il gruppo…"     | ✅    |
| 100 `setView`: nessuna voce                                                                              | "100 setView: nessuna voce di registro…"        | ✅    |
| annullando una fusione con l'inspector sull'ultimo passaggio, indice valido                              | "annullando una fusione…"                       | ✅    |

Test aggiuntivi: la finestra di 1000 ms si misura dal comando precedente
(10 comandi a 900 ms = 1 passo); l'ordine degli id non cambia la chiave;
qualunque altro comando e un gesto interrompono il gruppo.
