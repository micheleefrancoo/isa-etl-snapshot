# Validazione — Fase 6b.1: Inspector, contenuto, forma, tendine ISA, selettori multipli

Branch `feat/inspector-core`, da `main` (d560fdd, con la 6b.0). Commit piccoli e frequenti: elenco delle parti del prototipo, token di forma e semantici, `menu.ts`, contenuto dell'Inspector, pulsanti sul nodo e pannello espanso, test, e2e, documentazione.

## Esiti

| Controllo                                | Esito                                                                                                                                        |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `npx tsc --noEmit`                       | nessun errore                                                                                                                                |
| `npm run lint`                           | 0 errori (14 avvisi preesistenti di `react-refresh`)                                                                                         |
| `npm test`                               | 53 file, 1059 test, tutti superati                                                                                                           |
| `npm run build`                          | riuscita                                                                                                                                     |
| `node scripts/check-tokens.mjs` (esteso) | 0 violazioni (debito preesistente invariato: 26); ora controlla anche `src/etl-canvas/inspector/**`                                          |
| `scripts/e2e-fase5.mjs`                  | 34 prove su 34                                                                                                                               |
| `scripts/e2e-fase6a.mjs`                 | 92 prove su 92                                                                                                                               |
| `scripts/e2e-fase6b1.mjs` (nuovo)        | 164 prove su 164, eventi veri, tre ripetizioni consecutive senza instabilità                                                                 |
| Canvas a riposo                          | identico: `v2-*` e `fase4b/v2-*` a 0 pixel di differenza; il resto entro il rumore già noto (0–106 px, anche nei ritagli del solo prototipo) |

Test per file (nuovi o toccati): `inspector` 23, `inspector-rules` 12, `inspector-logic` 15, `menu` 8, `controlbar` 9, `overlay-layout` 7, `panels-layout` 22 (+2: tetto d'altezza), `panels-actions` 18, `toolbox` 14, `view` 14, `check-tokens` 7 (+3: scala tipografica e degli spazi).

Il codice dell'Inspector è provato senza DOM finto (non c'è jsdom nel repository): la logica dei selettori è pura (`logic.ts`, `menu.ts`, `params.ts`), la struttura si prova con il rendering lato server e il comportamento con tastiera, puntatore e focus nel browser vero (`e2e-fase6b1.mjs`).

## Che cosa c'è

- **Contenuto per tipo di nodo**: lavorazione senza ingresso → stato bloccato; con ingresso → intestazione (famiglia, nome in linea), campi e liste del catalogo, colonne e valori dallo schema in ingresso (`schemaOf`); dataset e output → nome, origine, colonne in sola lettura col tipo; box combinato → elenco dei passaggi riordinabile (puntatore e Alt+↑/↓) con sgancio ed eliminazione, tabella di riferimento prima di un join e nota «tabella unica» dopo. Filtro e join mostrano solo «Le condizioni arrivano nella prossima fase».
- **StyledSelect** (scelta singola con ricerca e «oppure scrivi» in corsivo), **ColumnPicker** (ordine di scelta, riordino con trascinamento e Alt+←/→, Tutte/Nessuna sulle visibili, conteggio annunciato), **ValuePicker** (etichette, ricerca, «+ Aggiungi», incolla di più valori, grafia dei dati, avviso fuori dominio senza azzeramento), **MultiList** (righe comprimibili, riassunto dal vivo, globali, note). Riempi vuoti: elenco con una colonna, testo con più; Raggruppa: nome del risultato disattivato con l'anteprima di `measureNames` se la misura ha più colonne.
- **Pulsanti sul nodo**: × (al passaggio e al focus, area ≥ 32 px, stessa anteprima e conferma di `nodesRemovedBy`) ed espansione dei box; **pannello espanso** con menu per passaggio e sgancio trascinando fuori dal pannello.
- **Forma**: nessun elemento nativo; scala tipografica e degli spazi in `src/theme/layout-tokens.css`; tendine nel portale `#ei-portal`, posizionate da `placeMenu` (8 px dal campo, 16 px dai bordi della finestra, lato con più spazio, scorrimento interno); testi in `copy.ts`; tutti i colori da token, con nuovi token semantici in entrambi i temi e modi e nuovi controlli di contrasto.
- **Tetto d'altezza** (`cappedPanelHeight`, 45%) e **colonne CSS** sui bordi alto e basso.
- **Focus e cronologia**: scrivere non perde mai focus né cursore (20 caratteri nel nome, 20 cifre in un campo numerico, nessuna perdita); 20 caratteri sono UNA voce di registro e UN annullamento li ripristina; Esc nel pannello riporta il focus al canvas senza deselezionare; l'apertura con un clic non sposta il focus.

## Tabella: funzione di dominio o comando per ogni azione dell'Inspector

| Azione dell'Inspector                                        | Funzione o comando                                                                   | Righe del prototipo           |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------ | ----------------------------- |
| Mostrare lo stato bloccato / la capienza del join            | `inputsOf`, `boxCapacity`                                                            | 3708-3719                     |
| Schema in ingresso (colonne e tipi)                          | `schemaOf`                                                                           | 2431-2432                     |
| Rinominare il nodo                                           | comando `renameNode` (raggruppato per nodo)                                          | 3862-3873                     |
| Cambiare un campo (testo, scelta, colonna singola)           | comando `setParams` (parametri nuovi; per le voci multiple passano da `ensureMulti`) | 2728-2740, 3880-3893          |
| Scegliere, togliere, riordinare le colonne                   | `setParams` sul campo `columns`; `columnsOf`, `ensureMulti`                          | 2751-2766 (qui multiple)      |
| Proporre i valori di una riga                                | `columnsDomain`                                                                      | 2956-2966, 2840-2954          |
| Segnalare i valori fuori dominio e rimuoverli                | `valuesOutsideDomain`, `setParams`                                                   | 3014-3021 (qui non si azzera) |
| Dividere i valori incollati                                  | `splitTokens`                                                                        | 2839                          |
| Riassunto di riga                                            | `L.sum` di `MULTI_DEFS`, `columnsText`                                               | 2984-3027                     |
| Nome del risultato di una misura                             | `measureNames`                                                                       | 2967-2983                     |
| Aggiungere e rimuovere una riga                              | `setParams`; riga nuova da `defaultParams`                                           | 3036-3052                     |
| Scegliere il passaggio da configurare                        | comando `inspect` (`node`, `step`)                                                   | 3764-3800                     |
| Riordinare i passaggi (puntatore, Alt+↑/↓)                   | comando `reorderSteps`                                                               | 3752-3838                     |
| Eliminare un passaggio                                       | comando `deleteStep`                                                                 | 3792                          |
| Sganciare un passaggio (pulsante, menu, trascinamento fuori) | comando `detachStep` (con `dropPoint` nel mondo)                                     | 2137-2204, 2290-2335          |
| Tabelle di un join (riferimento, sinistra, destra)           | `MERGE_OPS`, `inputsOf`, `setParams` (`table`, `leftTable`, `rightTable`)            | 3747-3774                     |
| Pulsante × sul nodo                                          | comando `deleteNodes` dopo la conferma; anteprima `nodesRemovedBy`                   | 4536-4590                     |
| «Configura parametri» dal pannello espanso                   | comandi `select`, `inspect`, `setPanel`                                              | 2354-2370                     |
| Esc nel pannello                                             | solo interfaccia (focus al canvas)                                                   | —                             |

Nessuna funzione o comando mancava: non è stato aggiunto nulla al dominio né a `etl-store`.

## Schermate (`docs/visual/fase6b1/`)

`inspector-bloccato`, `converti-tipo-due-colonne`, `sostituisci-valori-due-colonne`, `tendina-colonne-aperta`, `tendina-valori-aperta`, `tendina-in-alto`, `tendina-vicino-al-bordo-1280x720`, `box-combinato-passaggi`, `nodo-con-pulsanti`, `pannello-espanso`, `inspector-in-basso-1440`, `inspector-in-basso-1280x720`: ciascuna `-chiaro` e `-scuro`; in più due in tema notte (`tendina-colonne-aperta-notte-chiaro`, `converti-tipo-due-colonne-notte-scuro`).

## Che cosa ho dovuto lasciare o decidere

- **Condizioni di filtro e join** (connettori, gruppi, anteprima), il **tipo di join** e il **layout a tre colonne** dei bordi alto e basso: Fase 6b.2. Ora filtro e join mostrano solo la nota; i bordi alto e basso usano colonne CSS.
- **Riordino dei criteri di Ordina**: non in questa fase (come da specifica).
- **Caselle di spunta**: nelle voci dei menu ho usato la semantica ARIA `listbox`/`option` con `aria-selected` e una casella disegnata, non un `input` nascosto per ogni voce; l'Inspector non ha altre caselle né radio. Se si vuole l'`input` reale nascosto anche lì, va aggiunto.
- **Tabelle di riferimento del join**: il valore proposto di default non viene scritto nei parametri finché l'utente non sceglie (nel prototipo si scriveva subito); l'Engine futuro dovrà quindi trattare l'assenza come «la prima tabella» (la seconda per la destra).
- **«Sostituisci con»** (campo `value` di Sostituisci valori): segue la stessa regola di Riempi vuoti (elenco con una colonna, testo con più), non specificata per questa operazione.
- **Colonne non presenti nei dati**: oltre a «oppure scrivi» nelle scelte singole, anche il ColumnPicker accetta «+ Aggiungi “x”» (in corsivo), perché un dataset senza colonne note non permetterebbe altrimenti di configurare nulla.
- **Test dei componenti con DOM reale**: non c'è `jsdom`; la copertura è logica pura + rendering lato server + e2e nel browser. Non ho aggiunto dipendenze.
- **Persistenza di un passaggio scelto**: il passaggio mostrato è `inspector.step` di `etl-store`, quindi non si salva (come già per l'Inspector).
