# VALIDATION REPORT — Fase 2.1: correzioni alla geometria (`src/etl-layout/`)

Generato: 2026-09-29T08:57:43Z (UTC) — branch `feat/etl-layout-fix`

## 1. Esiti

```
npx tsc --noEmit  → PASS, 0 errori
npm run lint      → 0 errori, 14 warning (tutti pre-esistenti, react-refresh/only-export-components;
                    nessuno in src/etl-layout/)
npm test          → 210/210 test passati, 16 file
npm run build     → riuscita
```

Grep React/DOM/localStorage nei `.ts` di `src/etl-layout/`: **nessun risultato**.

**Reinstallazione pulita: eseguita e validazione ripetuta da zero.** Il
comando `rm -rf node_modules && npm ci` era stato negato dal sistema di
permessi della sessione (non aggirato); è stato poi eseguito a mano
dall'utente (441 pacchetti, 0 vulnerabilità). La validazione ripetuta
dopo l'installazione pulita dà esiti identici: `tsc` 0 errori, lint 0
errori e 14 warning pre-esistenti, 210/210 test in 16 file, build
riuscita, grep vuoto.

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
| **Totale**                                            | **210** |

Rispetto alla Fase 2 (203): +7 (2 sui golden corretti, 3 proprietà nuove,
2 unità su `displaceTarget`); il test di proprietà "dritto: 2 punti o
scalino" è sostituito da "dritto: sempre 2 punti".

## 2. Correzioni

| #   | Correzione                                                                                                                       | Dove                                                                               |
| --- | -------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| C1  | Convergenza: aggiornamento sequenziale dei cavi, incroci contati sui percorsi di base, cambio solo con costo strettamente minore | `links.ts` (`layoutLinks`), `routing.ts` (`evaluateRoute`), `types.ts` (`basePts`) |
| C2  | Sotto 1,5 px gli agganci si allineano: un dritto ha sempre 2 punti                                                               | `routing.ts` (`shapeCandidates`)                                                   |
| C3  | `displace`: centri coincidenti spinti verso il basso                                                                             | `free.ts` (`displaceTarget`)                                                       |
| C4  | `displace`: limite inferiore di `clampCard`                                                                                      | `free.ts` (`displaceTarget`)                                                       |
| C5  | Passaggio sganciato: sotto il box a `CARD + LABEL_H + 18`                                                                        | `constants.ts` (`DETACH_OFFSET_Y`), `placement.ts`                                 |
| C6  | Scambio con un nodo senza postazione: l'occupante va nella libera più vicina; ogni nodo ha una postazione                        | `slots.ts` (`dropInSlot`)                                                          |

`NOTE_DIVERGENZE.md`: nuova sezione "Correzioni intenzionali" (C1-C6);
i comportamenti replicati 2, 3, 8 diventano R1-R3 con la nota
"Comportamento accettato: fa parte della resa approvata del prototipo";
R1 descrive le priorità come pesi sommati, voluti.

## 3. Passate prima e dopo, per scenario golden

"Prima" = prototipo (file golden); "dopo" = implementazione corretta.

| Scenario                                          | Passate prima      | Passate dopo | Risultato             |
| ------------------------------------------------- | ------------------ | ------------ | --------------------- |
| 01-dritto-allineati                               | 2                  | 2            | identico al prototipo |
| 02-dritto-scorrimento                             | 2                  | 2            | identico              |
| 03-oltre-scorrimento                              | 2                  | 2            | identico              |
| 04-ostacolo                                       | 2                  | 2            | identico              |
| 05-incrocio                                       | 2                  | 2            | identico              |
| **06-corsie**                                     | **8 (limite)**     | **2**        | **corretto**          |
| 07-join-output-parziale                           | 2                  | 2            | identico              |
| 08-spostamento (iniziale / piccolo / grande)      | 2 / 2 / 2          | 2 / 2 / 2    | identico              |
| **09-catena-riordino (prima / dopo il riordino)** | **8 (limite) / 2** | **3 / 2**    | **corretto**          |
| 10-riordino-isolati                               | 2 / 2              | 2 / 2        | identico              |
| 10b-riordino-colonna-fitta                        | 2 / 2              | 2 / 2        | identico              |
| 11-organizzato                                    | —                  | —            | identico              |
| 12-output-generato                                | —                  | —            | identico              |
| 13-output-organizzato                             | —                  | —            | identico              |

12 golden su 14 restano identici al prototipo (passate comprese) e passano.

## 4. Golden corretti (`__tests__/golden/corretti/`)

Incroci = coppie di tratti che si incrociano fra cavi diversi; snodi e
lunghezza sono sommati su tutti i cavi.

| Scenario           | Risultato          | Passate | Incroci   | Snodi       | Lunghezza       |
| ------------------ | ------------------ | ------- | --------- | ----------- | --------------- |
| 06-corsie          | iniziale           | 8 → 2   | 1 → **0** | 6 → 6       | 2366 → **2232** |
| 09-catena-riordino | prima del riordino | 8 → 3   | 4 → 4     | 13 → **11** | 6552 → **6310** |
| 09-catena-riordino | dopo il riordino   | 2 → 2   | 0 → 0     | 3 → 3       | 831 → 831       |

In `09`, dopo il riordino, un solo cavo (OF → J) cambia: una L
speculare, stessa lunghezza e stessi snodi. È un effetto della stabilità:
il percorso di partenza (il risultato "prima del riordino") è diverso. Le
posizioni del riordino sono identiche al prototipo.

`golden.test.ts` verifica anche che ogni golden corretto differisca davvero
dal prototipo e che non peggiori gli incroci.

## 5. Test di proprietà

| Proprietà                                                                                                                      | Scenari                                 | Esito |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------- | ----- |
| `settleLinks` termina prima del limite e il risultato è un punto fisso (`layoutLinks` sul risultato non cambia alcun percorso) | 200 grafi × maxBends 1 e 2 (seme fisso) | ✅    |
| ogni segmento orizzontale o verticale                                                                                          | 40 × maxBends 0, 1, 2                   | ✅    |
| nessuna inversione né punto superfluo                                                                                          | 40                                      | ✅    |
| un dritto ha sempre esattamente 2 punti; sotto 1,5 px è perfettamente dritto                                                   | 40 + 4 casi                             | ✅    |
| nessun attraversamento di nodi quando esiste un'alternativa                                                                    | 40                                      | ✅    |
| limite di snodi rispettato quando possibile                                                                                    | 40 × maxBends 0, 1, 2                   | ✅    |
| dopo qualunque `dropInSlot` ogni nodo ha una postazione e nessuna postazione ha due nodi                                       | 40 grafi × 6 rilasci                    | ✅    |
| riordino deterministico e puro; flusso in colonne crescenti                                                                    | 40                                      | ✅    |

## 6. Gestore di pacchetti (punto 4)

Nel repository nulla usa bun: né gli script di `package.json`, né
`.devcontainer`, né configurazioni di CI (non ce ne sono). Però
`bun.lock` e `bunfig.toml` vengono dal template di Lovable (commit
`5081cd0`), e `bunfig.toml` contiene esclusioni specifiche per i pacchetti
`@lovable.dev/*`: è probabile che Lovable installi le dipendenze con bun.
Come deciso, **non sono stati rimossi**. Da chiarire con Lovable prima di
toglierli.
