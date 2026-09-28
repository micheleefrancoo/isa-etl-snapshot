# VALIDATION REPORT — Fase 1: dominio ETL puro (`src/etl-core/`)

Generato: 2026-09-28T21:06:48Z (UTC)

## 1. Esiti

```
npx tsc --noEmit  → PASS, 0 errori
npm run lint      → 0 errori, 14 warning (tutti pre-esistenti, react-refresh/only-export-components,
                    nessuno in src/etl-core/)
npm test          → 118/118 test passati, 11 file (41 pre-esistenti + 77 nuovi)
npm run build     → riuscita
```

Verifica React/DOM/localStorage (`grep -rn 'from "react"\|window\.\|document\.\|localStorage' src/etl-core/`):
**nessun risultato** — il modulo non importa React e non usa `window`,
`document` o `localStorage`.

## 2. File creati

```
src/etl-core/
  index.ts
  README.md
  NOTE_DIVERGENZE.md
  model/types.ts
  model/graph.ts
  catalog/operations.ts
  catalog/icons.ts
  catalog/params.ts
  rules/relations.ts
  rules/mutations.ts
  rules/state.ts
  logic/expressions.ts
  schema/schema.ts
  data/csv.ts
  __tests__/helpers.ts               (non un file di test: costruttori condivisi)
  __tests__/relations.test.ts
  __tests__/mutations.test.ts
  __tests__/params.test.ts
  __tests__/state.test.ts
  __tests__/expressions.test.ts
  __tests__/csv.test.ts
  __tests__/schema.test.ts
```

Nessuna modifica a file esistenti fuori da `src/etl-core/`, salvo
`scripts/generate-snapshot.mjs` (aggiunta della voce `01b-etl-core` in
`areaFor`, così che lo snapshot includa la nuova cartella — esplicitamente
consentito dal compito). `vitest.config.ts` non è stato toccato: il
pattern di default di Vitest raccoglie già `src/etl-core/__tests__/*.test.ts`
senza bisogno di configurazione aggiuntiva (verificato: i 77 nuovi test
sono stati eseguiti senza alcuna modifica a `vitest.config.ts`).

## 3. Numero di test per file

| File | Test | Scenari coperti |
|---|---|---|
| `__tests__/relations.test.ts` | 11 | 1, 2, 5, 8 |
| `__tests__/mutations.test.ts` | 9 | 3, 4, 6, 7, 15 |
| `__tests__/params.test.ts` | 11 | 9, 12 |
| `__tests__/state.test.ts` | 25 | 10 |
| `__tests__/expressions.test.ts` | 7 | 11 |
| `__tests__/csv.test.ts` | 9 | 13 |
| `__tests__/schema.test.ts` | 5 | 14 |
| **Totale** | **77** | tutti i 15 scenari richiesti |

Verificato con esecuzione separata per file (`npx vitest run <file>`):
tutti i conteggi coincidono con quanto dichiarato in `README.md`.

## 4. Contenuto di NOTE_DIVERGENZE.md

Due comportamenti del prototipo, ambigui o apparentemente errati,
replicati fedelmente senza correggerli (testo integrale in
`src/etl-core/NOTE_DIVERGENZE.md`):

1. **Messaggio di capienza superata cablato su "due tabelle"**
   (`linkRefusal`, prototipo riga 1913): il testo dice sempre "ha già le
   sue due tabelle" quando `cap > 1`, anche per un box con capienza 3
   (es. `join` + `union`) o composto solo da `union` senza alcun `join`.
2. **`stepMissing('filter', ...)` ignora la modalità del valore**
   (prototipo, righe 1430-1431): una condizione è "completa" se `text`
   **oppure** `values` non sono vuoti, indipendentemente da quale dei due
   campi la modalità attiva stia effettivamente usando — un valore
   residuo nell'altra modalità può mascherare uno stato incompleto.

Il documento chiarisce inoltre due adattamenti **non** considerati
divergenze (richiesti esplicitamente dal compito, non correzioni
spontanee): l'unificazione di `connect` con `linkRefusal` (controllo dei
cicli incluso), e la numerazione di "Output N"/"Combined Box N" derivata
dal grafo invece che da contatori globali mutabili (necessaria per il
vincolo di funzione pura).

## 5. Pubblicazione

Commit su `feat/etl-core`, push su `origin/feat/etl-core`. **Nessun
merge su `main`**, come richiesto. `./scripts/sync-snapshot.sh` eseguito
dal branch `feat/etl-core` dopo il commit, così che `src/etl-core/` sia
incluso nello snapshot — vedi messaggio finale in chat per SHA del
commit, numero di blocchi e link raw di INDEX.md.
