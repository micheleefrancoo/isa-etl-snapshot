# VALIDATION REPORT — Fase 1.1: correzioni intenzionali al dominio ETL (`src/etl-core/`)

Generato: 2026-09-29T07:20:03Z (UTC)

## 1. Esiti

```
npx tsc --noEmit  → PASS, 0 errori
npm run lint      → 0 errori, 14 warning (tutti pre-esistenti, react-refresh/only-export-components,
                    nessuno in src/etl-core/)
npm test          → 157/157 test passati, 13 file
npm run build     → riuscita
```

Verifica React/DOM/localStorage (`grep -rn 'from "react"\|window\.\|document\.\|localStorage' src/etl-core/`):
**nessun risultato**.

## 2. Conteggio dei test

| Momento                      | Totale | Dettaglio                                           |
| ---------------------------- | -----: | --------------------------------------------------- |
| Dopo la Fase 1               |    118 | 41 pre-esistenti + 77 di `src/etl-core/`            |
| + `state.test.ts`            |    119 | +1 (operatore a più valori con solo `text` residuo) |
| + `fase11.test.ts`           |    130 | +11                                                 |
| + `fase11-requisiti.test.ts` |    157 | +27 (19 sull'invariante + 8 requisiti puntuali)     |

Nessun test è stato rimosso o accorpato. Tre test esistenti sono stati
aggiornati alle nuove regole (stessa copertura, attese diverse):

- `relations.test.ts` › "una terza tabella su un box con un solo join e
  rifiutata con la capienza reale" (prima: "...col motivo del prototipo").
- `params.test.ts` › "un testo (vecchio formato) viene migrato in una lista
  di valori" (prima: "...diventa una scelta manuale").
- `state.test.ts` › "filter: con colonna e valore -> completo" (ora il
  valore di `=` sta in `values`, non in `text`).

Le altre modifiche ai test esistenti aggiungono solo l'argomento `nextId`
a `connect` e `deleteNodes`.

## 3. Requisito → test

| Requisito                                                                                                                | File                                 | Test                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Invariante: dopo ogni operazione esportata di `rules/mutations.ts`, ogni lavorazione con almeno un ingresso ha un output | `__tests__/fase11-requisiti.test.ts` | "invariante: dopo ogni operazione di rules/mutations.ts ..." — un caso per `connect`, `spawnOutput`, `refreshOutput`, `pruneOutputs`, `enforceCapacity`, `deleteNodes` (output / dataset / box), `deleteLink` (box → output / dataset → box), `mergeBoxes` (due varianti), `insertOnLink`, `detachStep`, `deleteStep`, `reorderSteps`, `duplicateNodes`; più "deleteNodes su un output lo ricrea per il box che ha ancora ingressi" |
|                                                                                                                          | `__tests__/fase11.test.ts`           | "connect garantisce l'output › dopo il primo ingresso il box ha un output, aggiornato al secondo"                                                                                                                                                                                                                                                                                                                                   |
| Conversione testo → valori in `ensureMulti` (Sostituisci valori), separatori `,` `;` a capo, senza duplicati             | `__tests__/fase11-requisiti.test.ts` | "conversione testo -> valori ... › ensureMulti: campo 'find' di Sostituisci valori"; "› ensureMulti: un campo manuale in forma di oggetto viene unito ai valori esistenti"                                                                                                                                                                                                                                                          |
| Conversione testo → valori in `ensureKeys` (`rlist`), stessi separatori, senza duplicati                                 | `__tests__/fase11-requisiti.test.ts` | "conversione testo -> valori ... › ensureKeys: rlist del join"                                                                                                                                                                                                                                                                                                                                                                      |
| `stepMissing` del filtro, operatore `>`, `values` pieno e `text` vuoto → incompleta                                      | `__tests__/fase11-requisiti.test.ts` | "stepMissing del filtro con operatore a valore singolo › '>' con values pieno e text vuoto -> incompleta"                                                                                                                                                                                                                                                                                                                           |
| Messaggio di capienza su un box join + union                                                                             | `__tests__/fase11-requisiti.test.ts` | "messaggi di capienza › box join + union: la quarta tabella è rifiutata con la capienza reale (3)"                                                                                                                                                                                                                                                                                                                                  |
| Messaggio di capienza su un box composto solo da una union                                                               | `__tests__/fase11-requisiti.test.ts` | "messaggi di capienza › box con sola union: la terza tabella è rifiutata senza nominare il join"; "› output incompleto di una union: messaggio generale"                                                                                                                                                                                                                                                                            |
| `mergeBoxes` accetta `positionFn` e lo usa per l'output                                                                  | `__tests__/fase11-requisiti.test.ts` | "mergeBoxes accetta positionFn › usa positionFn per l'output creato dalla fusione"                                                                                                                                                                                                                                                                                                                                                  |

## 4. Correzioni emerse dalla verifica

Scrivere i test dei requisiti ha fatto emergere due lacune, corrette in
questo stesso intervento:

1. **Invariante violata da `deleteNodes` e `deleteLink`.** Eliminare
   l'output di un box che ha ancora ingressi (o il collegamento
   box → output) lasciava il box senza output. Nel prototipo lo ricreava
   `renderAll` (riga 4407). Ora entrambe ricevono `nextId` (e
   `positionFn` facoltativo) e chiamano `refreshOutput` su ogni
   lavorazione. Verificato che senza la correzione 3 test falliscono.
2. **Separatori.** `normalizeValuesField` divideva solo su `sep`; ora
   divide su virgola, punto e virgola, a capo e `sep` (come `splitTokens`
   del prototipo, riga 2839).

Entrambe documentate in `src/etl-core/NOTE_DIVERGENZE.md`.
