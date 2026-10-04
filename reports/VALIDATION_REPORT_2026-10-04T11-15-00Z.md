# Validazione — Fase 6b.0: dominio, colonne multiple nelle voci

Branch `feat/multi-columns`, da `main` (8db8087). Solo dominio (`etl-core`) più la versione del formato in `etl-store`: nessuna interfaccia.

## Esiti

| Controllo                       | Esito                                                |
| ------------------------------- | ---------------------------------------------------- |
| `npx tsc --noEmit`              | nessun errore                                        |
| `npm run lint`                  | 0 errori (14 avvisi preesistenti di `react-refresh`) |
| `npm test`                      | 49 file, 928 test, tutti superati                    |
| `npm run build`                 | riuscita                                             |
| `node scripts/check-tokens.mjs` | 0 violazioni (debito preesistente invariato: 26)     |
| `e2e-fase5`, `e2e-fase6a`       | 34 e 92 prove superate (l'app carica e salva in v2)  |

Test per file (`etl-core`, `etl-store`): `multi-columns` 71 (nuovo), `save-versions` 6 (nuovo), `fase11-requisiti` 30, `state` 26, `reduce` 42, `store` 16 (+1), `fase11` 11, `params` 11, `relations` 11, `grouping` 11, `csv` 9, `mutations` 9, `persistence` 9, `expressions` 7, `schema` 5, `react` 3.

## Cosa cambia

- **Tipo di campo `"columns"`** in `MULTI_DEFS`; le righe hanno `columns: string[]` (l'ordine conta) al posto di `column`. Convertono: Converti tipo, Arrotonda, Normalizza, Pulisci testo, Riempi vuoti, Sostituisci valori, Rimuovi duplicati, Seleziona colonne, Ordina, Raggruppa (chiavi e misure). Restano a colonna singola Rinomina, Dividi colonna, Calcola colonna, le colonne del filtro e i lati del join (provato).
- **Migrazione in `ensureMulti`**, idempotente: `column: "x"` → `columns: ["x"]`, stringa vuota → `[]`, `column` rimosso; un `columns` già presente e non vuoto prevale; vale anche per il formato a voce singola. Provata per ciascuna operazione (11 liste): migrazione, stringa vuota, idempotenza, precedenza.
- **`stepMissing`/`nodeState`**: riga incompleta se `columns` è vuoto o manca un altro campo obbligatorio (l'avviso resta «Da configurare: Converti tipo», ecc.).
- **Riassunti**: `columnsText(columns, max = 3)` («a, b, c» / «a, b +2»); ogni operazione usa l'elenco, per esempio «importo, quantita → intero».
- **`measureNames(row)`**: più colonne → sempre `${funzione}_${colonna}`, alias ignorato; una colonna → alias oppure `${funzione}_${colonna}`.
- **`flattenRows(type, params)`**: righe espanse (campo `column`), una lista per lista dell'operazione, nell'ordine elencato; scarta le righe incomplete; con più colonne l'alias delle misure si azzera.
- **`columnsDomain(schema, columns)`** (unione nell'ordine delle colonne, senza duplicati, tetto 500) e **`valuesOutsideDomain(values, domain)`**. Cambiare le colonne non azzera i valori (provato; documentato in `NOTE_DIVERGENZE.md`, § 5).
- **Salvataggio**: versione 1 → 2. `fromSaved` legge v1 e v2; un v1 passa da `ensureParams` (filtro, join, voci multiple) su ogni card, un v2 non cambia (provato con un v2 che contiene volutamente righe nel vecchio formato). Fixture v1 con tutte le operazioni coinvolte più Rinomina e un filtro con il vecchio `logic`.
- **Registro**: il payload di `setParams` riporta `columns` come elenco (provato); nessuna compatibilità per le voci già registrate.
- **Immutabilità**: `ensureMulti`, `ensureParams`, `flattenRows`, `stepMissing`, `measureNames`, `columnsText`, `columnsDomain`, `valuesOutsideDomain` provate su dati congelati in profondità; `fromSaved` non modifica il dato ricevuto.

## Letture di `row.column` altrove

Nessuna da correggere. Fuori da `etl-core` solo la vecchia rotta `?canvas=v1` (`src/lib/etl-node-config.ts`, pannelli in `src/components/isa/etl/`) usa un proprio `column`, senza importare `etl-core`. `etl-canvas` ed `etl-store` non leggono `row.column`; la scena del prototipo (`seed.ts`) usa `defaultParams`, quindi nasce già nel formato nuovo.

## Scelte da confermare (la specifica lasciava margine)

1. **Ordine di `columnsDomain`**: «unione ordinata» è stata letta come ordine delle colonne elencate e, in ciascuna, quello del suo dominio (già ordinato da `parseCSV`), non un ordinamento alfabetico globale.
2. **`flattenRows` e righe con un campo obbligatorio vuoto**: oltre a quelle senza colonne, scarta anche quelle con un altro campo obbligatorio vuoto, coerentemente con `stepMissing`.
3. **`ensureParams` al caricamento di un v1** applica anche le migrazioni di filtro (`migrateFilterLogic`) e join (`ensureKeys`), non solo quella delle colonne: sono idempotenti e finora avvenivano al primo uso.
4. **Righe espanse**: usano il campo `column` (singolare) e non hanno `columns`.
