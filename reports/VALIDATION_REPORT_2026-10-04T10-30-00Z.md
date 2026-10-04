# Validazione — Fase 6a.2: un pannello alla volta, spinta senza sovrapposizioni, barra dei controlli

Branch `feat/panels-exclusive`, da `main` (cf6e294). Tre commit: il codice, la baseline visiva rigenerata (solo PNG, separata) e questo report.

## La barra dei controlli esisteva già?

No. In `src/etl-canvas/` non c'era nessuna barra (modo Libero/Organizzato, Riordina, Annulla, Ripristina, «Reimposta»): gli equivalenti esistevano solo nella vecchia rotta `?canvas=v1`, fuori da `etl-canvas`. È stata costruita (`panels/ControlBar.tsx`), senza «Funzionalità» né «Reimposta».

## Esiti

| Controllo                       | Esito                                                                                                                                         |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `npx tsc --noEmit`              | nessun errore                                                                                                                                 |
| `npm run lint`                  | 0 errori (14 avvisi preesistenti di `react-refresh`)                                                                                          |
| `npm test`                      | 47 file, 850 test, tutti superati                                                                                                             |
| `npm run build`                 | riuscita                                                                                                                                      |
| `node scripts/check-tokens.mjs` | 0 violazioni (debito preesistente invariato: 26)                                                                                              |
| `scripts/e2e-fase5.mjs`         | 34 prove su 34 (il selettore del pulsante «Annulla» della conferma ora è circoscritto alla finestra, perché la barra ha un «Annulla» proprio) |
| `scripts/e2e-fase6a.mjs`        | 92 prove su 92                                                                                                                                |
| Colori letterali nei file nuovi | nessuno                                                                                                                                       |

Nuovi test: `overlay-layout` (matrice di 11 aree × 3 stati × 16 combinazioni di bordi, nessuna intersezione e tutto dentro l'area), `controlbar` (barra, «Svuota» con conferma, assenza di «Reimposta» e «Funzionalità» nei file dei pannelli), `panels-layout` (`keepVisible`: per ogni bordo, scorrimento minimo, insieme troppo largo, margini dei widget, grafo e zoom invariati in Libero e Organizzato), `panels-actions` (apertura solo al clic, memoria di sostituzione e suo azzeramento), `view` (Adatta con margini, rotella), store (`clearAll`, esclusività per tutte le combinazioni di bordi, stato salvato con entrambi aperti).

## Cosa cambia

1. **Un pannello alla volta.** `setPanel` chiude l'altro nello stesso aggiornamento, su qualunque bordo; un salvataggio con entrambi aperti si normalizza lasciando aperta la cassetta. Sul bordo condiviso le schede restano l'interruttore, con la misura maggiore dei due.
2. **Inspector solo al clic.** L'apertura automatica parte dal nuovo evento `subscribeClick` del controller (rilascio senza trascinamento che lascia selezionato solo quel nodo, senza Maiusc). Non parte alla pressione, durante un trascinamento, con un riquadro, con una selezione multipla né dopo un rilascio dalla cassetta. Se sostituisce la cassetta, la deselezione la riapre; ogni azione esplicita sui pannelli azzera la memoria.
3. **Spinta senza sovrapposizioni** (`keepVisible`, sostituisce `viewCompensation`/`compensate`). Posizioni nel mondo e zoom non cambiano mai; i nodi si spostano con il bordo del canvas; dopo ogni cambio dell'area (apertura, chiusura, scheda, tacca, finestra) i nodi che erano interamente visibili restano visibili con lo scorrimento minimo, o si allinea al bordo di partenza se l'insieme non entra. Margine di sicurezza dei widget anche per «Adatta».
4. **Widget** (`overlayLayout`, funzione pura). Minimappa in basso a sinistra, in alto a sinistra con il pannello in basso; zoom in basso a destra; se l'area è piccola la minimappa passa all'angolo opposto e poi diventa un pulsante compatto che si espande al clic. Il suggerimento e le tacche sono nello stesso calcolo.
5. **Barra dei controlli:** Libero/Organizzato, Riordina, Annulla, Ripristina, Svuota (`clearAll`: un passo di cronologia, una voce nel registro, libreria intatta; conferma sempre, con il focus su «Annulla» e Esc che annulla).
6. **Rotella:** semplice = pan, Maiusc = orizzontale, Cmd/Ctrl = zoom. Il pan con rotella semplice c'era già (Fase 4a); si è aggiunto Maiusc. Nessun conflitto con la barra spaziatrice e il tasto centrale (provati nel browser).

## Posizione dei nodi sullo schermo, prima e dopo (1440 × 900, `x,y` in px)

Cassetta chiusa → aperta sul bordo indicato. Le posizioni nel mondo sono identiche prima e dopo in ogni caso (verificato). A sinistra i nodi avanzano con il bordo del canvas (280 px); in alto avanzano con il bordo (222 px) e poi `keepVisible` riporta in vista i più bassi (−40 px); a destra e in basso restano dove sono.

| Bordo  | ds1              | op-filter         | op-join           | op-sort           | op-export         |
| ------ | ---------------- | ----------------- | ----------------- | ----------------- | ----------------- |
| right  | 42,378 → 42,378  | 276,248 → 276,248 | 276,378 → 276,378 | 276,534 → 276,534 | 458,534 → 458,534 |
| top    | 42,378 → 42,560  | 276,248 → 276,430 | 276,378 → 276,560 | 276,534 → 276,716 | 458,534 → 458,716 |
| bottom | 42,338 → 42,338  | 276,208 → 276,208 | 276,338 → 276,338 | 276,494 → 276,494 | 458,494 → 458,494 |
| left   | 42,338 → 322,338 | 276,208 → 556,208 | 276,338 → 556,338 | 276,494 → 556,494 | 458,494 → 738,494 |

I dati completi sono in `docs/visual/fase6a/posizioni-nodi.json`. In tutti i casi i nodi prima interamente visibili restano interamente visibili e non intersecano il pannello aperto (misurato sui rettangoli del DOM).

## Widget nel browser

Per ciascun bordo (sinistra, destra, alto, basso, pannelli chiusi) e per le finestre 1440×900, 1280×720 e 1280×600: nessun widget (minimappa, zoom, tacche visibili) tocca un altro, la barra o esce dall'area del canvas; il pannello sta nella finestra e la pagina non scorre. Con il pannello in basso la minimappa è in alto a sinistra. A 1280×520 il canvas resta ≥ 160 px e scorre il contenitore, non la pagina.

## Baseline di fase4 e fase4b (commit separato)

La riga della barra riduce l'altezza del canvas, quindi le immagini a pagina intera cambiano. Differenze per regione rispetto alla baseline precedente (pixel con scarto > 8):

| Regione                                   | Cosa cambia                                                                              | Pixel (chiaro / scuro)                           |
| ----------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Intestazione (y < 146)                    | niente                                                                                   | 0 / 0                                            |
| Riga della barra (y 146–196)              | nuova                                                                                    | 34 597 / 56 595                                  |
| Area del canvas (y ≥ 196)                 | nodi e cavi traslati di 50 px verso il basso; minimappa e zoom restano ancorati al fondo | 49 440–62 730 su `v2-*`, `cavi-*`, `fase4b/v2-*` |
| Ritagli `crop-*` e immagini del prototipo | invariati entro il rumore tra due esecuzioni                                             | 0–68                                             |

I PNG sono stati rigenerati e committati a parte; webm e `misure.json` non sono stati toccati.

## Schermate (`docs/visual/fase6a/`)

Rigenerate con gli stessi nomi; nuove: `barra-controlli.png`, `svuota-conferma.png`, `minimappa-con-pannello-in-basso.png`, `nodi-spinti-pannello-in-alto.png`. `cassetta-alto-e-basso.png` è stata tolta: con un pannello alla volta lo stato che mostrava (uno in alto e uno in basso aperti insieme) non esiste più.

## Comportamenti non chiari rimandati

- Il contenuto dell'Inspector (Fase 6b) e i pulsanti di eliminazione ed espansione sul nodo restano rimandati.
- Il minimo del canvas (`MIN_CANVAS_HEIGHT` = 160) è stato abbassato da 200 per far stare cassetta in basso e barra in una finestra di 600 px; da riesaminare con il contenuto reale dell'Inspector.
- Un'area minuscola (sotto circa 260 × 200 px con le due tacche visibili) non può ospitare tutti i widget senza toccarsi: la matrice di prova parte da 260 × 200.
