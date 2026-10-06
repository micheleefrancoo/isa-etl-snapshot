# 01e-etl-canvas-b.md

File in questo blocco:

- `src/etl-canvas/README.md`
- `src/etl-canvas/__tests__/animator.test.ts`
- `src/etl-canvas/__tests__/autofit.test.ts`

---

### `src/etl-canvas/README.md`

280 righe

```md
# etl-canvas — Fasi 4a, 4b, 5, 6a e 6b.1: il canvas, le sue animazioni, i gesti, i pannelli e l'Inspector

Resa visiva del canvas ETL in React, fedele al prototipo
`docs/prototype/isa-fusion-prototype.html`. Solo **vista**: token, nodi,
cavi, pan, zoom, controlli di zoom, minimappa (4a) e animazioni: flusso nei
cavi, attesa delle fette vuote, transizione dei percorsi (4b), e i gesti (Fase 5): trascinamento, fusione,
collegamento, porte, selezione, tastiera, e i pannelli (Fase 6a): cassetta degli
strumenti e Inspector, agganciabili ai quattro bordi; il contenuto
dell'Inspector (Fase 6b.1): campi, selettori di colonne e di valori, voci
multiple, passaggi dei box combinati. Le condizioni di filtro e join e il
layout a tre colonne dei bordi alto e basso sono della Fase 6b.2.

Importa da `etl-core`, `etl-layout` ed `etl-store`; nessuno di questi importa
da qui. Non usa il vecchio stato (`src/lib/etl-workflow.tsx`): legge e
scrive solo attraverso `etl-store`.

## Moduli

```
tokens.css       token --ec-* (livello 3) con ambito .etl-canvas: leggono solo i token semantici --isa-* dei temi (src/theme/); tema predefinito chiaro = prototipo, scuro progettato
canvas.css       aspetto di nodi, cavi, controlli, minimappa; importa tokens.css
EtlCanvas.tsx    EtlCanvas (solo browser, misura l'area) e CanvasSurface (la resa, rendibile anche in Node)
Node.tsx         un nodo: chip, icone, fette, etichetta, indicatore ambra
Links.tsx        i cavi da store.getRoutes()
Minimap.tsx      minimappa e clic/trascinamento per spostare la vista
icons.tsx        icone del catalogo di etl-core
model.ts         grafo → classi e fette di ogni nodo (puro)
view.ts          zoom, Adatta, minimappa (puro, numeri del prototipo)
actions.ts       fit/zoomIn/zoomOut/zoomReset/zoomAtPoint: applicano la vista con setView
seed.ts          scena iniziale del prototipo (solo sviluppo)
contrast.ts      contrasto WCAG tra i token (test e report)
flow.ts          4b, puro: finestra e contorno del tubo del flusso, opacità dell'attesa
transitions.ts   4b, puro: interpolazione dei punti, dissolvenza incrociata, piano della transizione
loop.ts          4b: UN ciclo requestAnimationFrame condiviso (+ ambiente del browser)
engine.ts        4b: livello sottile che applica lo stato visivo agli attributi SVG
motion.tsx       4b: contesto con cui cavi e nodi registrano i propri elementi
interaction.ts   5: controller dei gesti (puntatore, tastiera) → comandi di etl-store. Puro, senza DOM
drop.ts          5: handleCanvasDrop / previewCanvasDrop, il rilascio di un nuovo elemento (per la Fase 6)
```

## Uso

```tsx
const store = usePersistentEtlStore(solutionId); // etl-store/react
<EtlCanvas store={store} />;
```

Il contenitore deve avere un'altezza (minimo 520 px). Nella rotta
`solutions.$solutionId.etl.tsx` il nuovo canvas è quello **predefinito**; il
vecchio (codice invariato) si raggiunge solo con `?canvas=v1`. Solo in
sviluppo, `?seed=prototype` carica la scena del prototipo se il canvas è
vuoto (in produzione `seed` è ignorato) e `window.__etlStore` espone lo
store alla console e allo script delle schermate.

## Fase 5 — Gesti

Questo livello **non contiene logica di dominio**: `interaction.ts` traduce
eventi già classificati (coordinate dell'area, bersaglio del gesto) in
chiamate a funzioni che esistevano. `EtlCanvas.tsx` è l'unico punto che
legge il DOM (`classify`) e registra gli ascoltatori (Pointer Events su
`window` durante il gesto, `keydown` sul documento). Il controller si prova
con eventi simulati (`__tests__/interaction.test.ts`, `keyboard.test.ts`,
`drop.test.ts`); `scripts/e2e-fase5.mjs` prova gli stessi gesti con
Pointer Events veri in Chromium.

| Gesto                                                                   | Prototipo (righe)                                                                                 | Funzione chiamata oggi                                                                                                                                       |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Trascinare un nodo                                                      | `stage` `pointerdown` 1959-2107 (`onMove` 1978-2056, `onUp` 2058-2105)                            | `store.beginGesture` / `updateGesture` / `commitGesture` / `cancelGesture` (Fase 3); il rilascio è `dropAt` di etl-store (un passo di cronologia)            |
| Esito sopra un altro nodo (fusione, collegamento, inverso, spostamento) | 2034-2056                                                                                         | etl-core `relation` (1917-1936); `performMerge`/`connect` al rilascio: `dropAt` → `mergeBoxes`/`connect`; lo spostamento è `displace` dentro `updateGesture` |
| Trascinare su un cavo (inserimento)                                     | 2015-2030, 2092                                                                                   | etl-layout `linkAt`; etl-core `insertable`; al rilascio `dropAt` → `insertOnLink`                                                                            |
| Tirare un cavo da una porta                                             | 3974-4024                                                                                         | etl-layout `nodePorts` (punto di partenza); comando `connect`                                                                                                |
| Rilasciare dalla cassetta o dalla libreria                              | `paletteEl` `pointerdown` 4939-5067                                                               | comando `addNode` (con `paletteRelation` per l'anteprima): `handleCanvasDrop` / `previewCanvasDrop` in `drop.ts`                                             |
| Click, Maiusc+click                                                     | 2063-2071, `selectCard` 2713, `toggleInSelection` 2693                                            | comandi `select` + `inspect`                                                                                                                                 |
| Riquadro di selezione                                                   | 4058-4089                                                                                         | etl-layout `nodeRect`; comandi `select` + `inspect`                                                                                                          |
| Trascinare il gruppo                                                    | 1969-2001, 2076-2081                                                                              | gesto di etl-store con più `ids` (`dropAt` ricompone i sovrapposti)                                                                                          |
| Click sul vuoto                                                         | 4584                                                                                              | comandi `select` + `inspect` (vuoti)                                                                                                                         |
| Click su un cavo                                                        | `linkHits` `click` 4575-4580, `deleteLink` 4565                                                   | comando `deleteLink` (senza conferma, come nel prototipo; un trascinamento che parte dal cavo non elimina)                                                   |
| Canc / Backspace                                                        | `deleteMany` 4538, `deleteCard` 4558, `commitDelete` 4480, `nodesRemovedBy` 4432; tasto 4630-4634 | `nodesRemovedBy` (anteprima) e comando `deleteNodes`                                                                                                         |
| Frecce (2 px, Maiusc = `GRID`)                                          | 4619-4627; tasto 4642-4644                                                                        | comando `moveNodes` (tenere premuto = un solo passo, Fase 3.1)                                                                                               |
| Cmd/Ctrl+D                                                              | 4594-4618, 4645                                                                                   | comando `duplicate`                                                                                                                                          |
| Cmd/Ctrl+A, Esc                                                         | 4646; Esc 4636                                                                                    | comandi `select` + `inspect`                                                                                                                                 |
| Cmd/Ctrl+Z, +Maiusc+Z, Ctrl+Y                                           | `undo` 4389, `redo` 4395; tasti 4650-4653                                                         | `store.undo()` / `store.redo()`                                                                                                                              |
| Pan: spazio o tasto centrale; zoom Cmd/Ctrl+rotella                     | 4036-4054, 4166-4180, 4092-4102                                                                   | già collegati nella 4a (`setView`); la 5 verifica che convivano con la selezione                                                                             |

**Esiti mostrati durante il trascinamento** (`InteractionUi.drop`), ciascuno
con colore e stile di contorno propri, tutti da token semantici
(`--isa-drop-*`): `merge` (anello pieno), `link` (anello pieno), `link-reverse`
(tratteggio), `displace` (punteggiato, col motivo di etl-core nel
suggerimento), `reject` (continuo). Il cavo in cui si inserirebbe la
lavorazione prende `ec-link-hot`; i nodi che l'eliminazione porterebbe via
(`nodesRemovedBy`: scelti + output a valle) prendono `ec-doomed` mentre la
conferma è aperta.

**Scelte e differenze dal prototipo**

- _Soglia di avvio_: `DRAG_THRESHOLD_PX` = 5 px, in `etl-layout/constants.ts`
  (prototipo, riga 1982). Sotto la soglia è un click. Il riquadro di
  selezione usa 4 px (prototipo, riga 4064).
- _Inserimento su cavo_: il nodo in mano è un ostacolo e fa scansare i cavi;
  il puntatore resta quindi spesso lontano dal cavo disegnato. Il test sul
  cavo si fa sui percorsi attuali **e** su quelli di prima del gesto
  (`startRoutes`).
- _Selezione_: `select` e `inspect` insieme; i pannelli (`setPanel`) non si
  toccano (l'apertura dell'Inspector è della Fase 6). Il riquadro non
  scrive nello store mentre si trascina (il registro delle attività si
  riempirebbe): lo mostra con `InteractionUi.marquee` e seleziona al rilascio.
- _Esc_ durante un trascinamento lo annulla (`cancelGesture`); poi chiude la
  conferma; poi deseleziona.
- _Scorciatoie di annulla/ripristina_ non agiscono con il fuoco in un campo di
  testo (il prototipo le applicava sempre): lì vale l'annulla del campo.
- _Non collegati_ (fuori dall'elenco della fase): i pulsanti di eliminazione ed
  espansione sul nodo (prototipo 4584, Fase 6).

## Fase 6a — Pannelli e cassetta

Codice in `panels/` (`Dock.tsx`, `Toolbox.tsx`, `InspectorShell.tsx`,
`EtlWorkspace.tsx`; logica pura in `layout.ts`, `actions.ts`, `csv.ts`,
`families.ts`). Lo stato (lato, aperto/chiuso, scheda attiva) è in `etl-store`
(`panels`) e si salva con il resto. Prototipo:
`docs/prototype/isa-fusion-prototype.html`.

| Elemento                                                                            | Prototipo (righe)                                                                                                | Qui                                                                                                                            |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Struttura, griglia dei quattro bordi, colonna o fascia                              | CSS 211-346 (`#dock-*` 216-219), HTML 808-838                                                                    | `Dock.tsx`, `panels.css`, `layout.ts`                                                                                          |
| Apertura e chiusura, tacca visibile solo da chiuso                                  | `setPanelOpen` 4835-4860, `layoutNotches` 4847-4858, CSS `.notch` 272-295                                        | comando `setPanel`; `layout.ts` (la tacca si nasconde se il pannello è aperto o raggiungibile da una scheda)                   |
| Trascinare la tacca su un altro bordo (soglia, un clic apre)                        | 4881-4905 (soglia 4884, «un click apre soltanto» 4899)                                                           | `Dock.tsx` (gesto della tacca), `actions.ts` (`setSide`: chiude, sposta, riapre)                                               |
| Due pannelli sullo stesso bordo: schede, contenuto sul posto                        | `dock-tabs` CSS 306-327, `switchTab` 4787-4806; nessuna animazione di apertura                                   | `Dock.tsx`, `layout.ts` (`grouped`, scheda attiva); la larghezza non cambia al cambio di scheda                                |
| Posizione della vista dopo un cambio dei pannelli                                   | 4780-4784, 4815-4824 (nel prototipo: compensazione a sinistra; qui spinta senza sovrapposizioni, § 9 delle note) | `layout.ts` (`keepVisible`), applicata da `Dock.tsx` dopo ogni cambio di area                                                  |
| Posizione di minimappa, zoom, suggerimento e tacche                                 | 138-161, 272-295 (CSS; qui decisa dal bordo del pannello aperto, § 10 delle note)                                | `overlayLayout.ts` (funzione pura), usata da `Dock.tsx` e `EtlCanvas.tsx`                                                      |
| Barra dei controlli (Libero/Organizzato, Riordina, Annulla, Ripristina, Svuota)     | barra del prototipo senza «Funzionalità» né «Reimposta» (§ 11 delle note)                                        | `ControlBar.tsx`; comando `clearAll` di etl-store, conferma nel controller (`requestClearAll`)                                 |
| Sezioni della cassetta (Dataset, Filtra e ordina, Trasforma, Merge e union, Output) | `buildPalette` 4727-4752, `SECTIONS`, `palItem` 4727-4733                                                        | `Toolbox.tsx`, `families.ts`: sezioni e voci derivano da `etl-core/catalog/operations.ts`, non da una lista a mano             |
| Sezione comprimibile                                                                | `.tb-sec-head` 246-250, clic 4914-4919 (senza ridisegno)                                                         | stato locale di `Toolbox.tsx` per sezione                                                                                      |
| Caricamento CSV e deduzione dei tipi                                                | `parseCSV` 4672-4725 (tipo: integer, numerico, data, stringa: righe 4696-4699), `change` 4921-4937               | `panels/csv.ts` → `store.loadCsv` (`parseCSV` di etl-core, comando `loadDataset`); voce trascinabile nella sezione Dataset     |
| Trascinare una voce dalla cassetta al canvas                                        | `paletteEl` `pointerdown` 4939-5067, `paletteRelation` 4711                                                      | `EtlWorkspace.tsx` → `handleCanvasDrop` / `previewCanvasDrop` (`drop.ts`)                                                      |
| Inspector (guscio): si apre al clic su un nodo, si chiude senza                     | `selectCard` / `deselect` 2692-2727, `openInspector` 2692                                                        | `InspectorShell.tsx` (solo il nome del nodo), `actions.ts` (`followInspector`: apertura solo al clic, memoria di sostituzione) |

Non portati: i pulsanti «Funzionalità» e «Reimposta» (tutte le funzionalità sono sempre
attive, Fase T). Prova nel browser: `scripts/e2e-fase6a.mjs` (92 prove,
schermate in `docs/visual/fase6a/`).

## Fase 6b.1 — Inspector

Codice in `inspector/` (montato da `panels/InspectorShell.tsx`). Prototipo:
`docs/prototype/isa-fusion-prototype.html`. Si porta la versione FINALE: per i
valori il selettore unico (`pickerHtml`), non le modalità elenco/manuale
superate (`mode`, `text`, `sep` restano solo come formato di transito, già
migrato dal dominio).

| Elemento del prototipo                                                                  | Righe                                                                 | Qui                                                                                                                              |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `renderInspector`, `renderInspectorInner`, stato vuoto, selezione multipla              | 3528-3548, 3636-3664, 3679-3683                                       | `Inspector.tsx`; la multiselezione non apre nulla (regola della 6a.2)                                                            |
| Intestazione: famiglia (`kind`), nome modificabile, chiudi                              | 3677-3694, 3862-3873                                                  | `Header.tsx` (comando `renameNode`)                                                                                              |
| Dataset e output: nota del risultato, capienza                                          | 3697-3706                                                             | `Inspector.tsx` (sola lettura delle colonne con il tipo)                                                                         |
| Stato bloccato (lucchetto, tabelle richieste dal join)                                  | 3708-3719 (`.insp-lock` CSS 498-503)                                  | `BlockedNotice.tsx`                                                                                                              |
| Elenco dei passaggi del box combinato: sequenza, riordino, elimina                      | 3721-3745, 3752-3838 (`bindStepList`)                                 | `StepList.tsx` (comandi `reorderSteps`, `deleteStep`, `detachStep`)                                                              |
| Tabelle di un join: sinistra, destra, di riferimento, nota «tabella unica»              | 3747-3774                                                             | `Inspector.tsx` (campi `table`, `leftTable`, `rightTable` dei parametri; `setParams`)                                            |
| Campo: `fieldHtml` (select, colonna, testo)                                             | 2728-2740                                                             | `Field.tsx`, `StyledSelect.tsx`                                                                                                  |
| Tendine: `selectHtml`, `columnSelect`, `valueSelect`, `closeSelects`, `bindSelects`     | 2742-2833, 2956-2966                                                  | `StyledSelect.tsx`, `menu.ts` (posizione: il prototipo stacca il menu in `body` e lo dimensiona sulla finestra, righe 2793-2810) |
| Colonne: `columnSelect` (una sola); qui il selettore multiplo                           | 2751-2766                                                             | `ColumnPicker.tsx` (campi `columns`, Fase 6b.0)                                                                                  |
| Selettore di valori: `pickerHtml`, `pickerInner`, `filterPicker`, `addTokens`, tastiera | 2840-2954, `splitTokens` 2839                                         | `ValuePicker.tsx` (`splitTokens` di etl-core)                                                                                    |
| Voci multiple: `mlField`, `renderMulti`, `multiSelectChange`, `bindMulti`               | 2967-3095 (nel prototipo cambiare colonna azzera i valori, 3014-3021) | `MultiList.tsx` (qui i valori NON si azzerano: `NOTE_DIVERGENZE.md` di etl-core § 5)                                             |
| Passaggio selezionato e parametri (`ensureParams`, `selectedStep`)                      | 3764-3800, 2613-2617                                                  | `Inspector.tsx`                                                                                                                  |
| Pulsante elimina sul nodo (`del-btn`, visibile al passaggio) e conferma                 | 89-102, 989-997, 4536-4590                                            | `Node.tsx`; `controller.requestDeleteNodes` (stessa anteprima e conferma di `nodesRemovedBy`, Fase 5)                            |
| Pulsante di espansione dei box combinati (`expand-btn`)                                 | 685-694, 995-997, 2258-2262                                           | `Node.tsx`                                                                                                                       |
| Pannello espanso: passaggi, riordino, menu «Configura parametri / Sgancia / Elimina»    | 710-730, 855-866, 2117-2135, 2250-2400 (`closeAllDropdowns` 2111)     | `StepList.tsx` e il pannello espanso (menu come componente nostro)                                                               |
| Trascinare un passaggio fuori dal pannello lo sgancia                                   | 2290-2335, 2137-2204 (`detachStep`)                                   | comando `detachStep` (con punto di rilascio nel mondo)                                                                           |

### Struttura di `inspector/` (Fase 6b.1)

| File                                                  | Compito                                                                                                                   |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `Inspector.tsx`                                       | il contenuto, per tipo di nodo; scrive solo con `setParams`, `renameNode`, `inspect` e i comandi dei passaggi             |
| `Header.tsx`, `NameInput.tsx`, `family.ts`            | famiglia del nodo e nome modificabile in linea (un carattere = un `renameNode`, raggruppato in un solo passo)             |
| `BlockedNotice.tsx`                                   | stato bloccato di una lavorazione senza ingresso                                                                          |
| `StepList.tsx`, `ActionMenu.tsx`, `ExpandedPanel.tsx` | passaggi del box combinato (riordino col puntatore e con Alt+↑/↓), menu dei passaggi e pannello espanso                   |
| `Field.tsx`, `StyledSelect.tsx`                       | campo e scelta singola con ricerca (combobox ARIA)                                                                        |
| `ColumnPicker.tsx`                                    | colonne multiple, nell'ordine di scelta, riordinabili; `Tutte`/`Nessuna` sulle visibili; conteggio annunciato             |
| `ValuePicker.tsx`                                     | valori come etichette, ricerca, «+ Aggiungi», incolla di più valori, dominio = `columnsDomain`, avviso fuori dominio      |
| `MultiList.tsx`                                       | liste a voci multiple del catalogo (`MULTI_DEFS`): righe comprimibili, riassunto dal vivo, globali e note                 |
| `Menu.tsx`, `menu.ts`                                 | portale dei menu e posizionamento puro (`placeMenu`): 8 px dal campo, 16 px dai bordi della finestra, lato con più spazio |
| `logic.ts`, `params.ts`                               | logica pura dei selettori e lettura/scrittura immutabile dei parametri (solo `columns`, mai `flattenRows`)                |
| `copy.ts`, `inspector.css`                            | tutti i testi; tutti gli stili, solo a token                                                                              |

Forma: nessun elemento nativo (`<select>`, `<option>`, `<datalist>`, input numerici,
di data, colore, intervallo), nessuna tendina del sistema; ogni menu è un
`[role=listbox]` (o `menu`) nel portale `#ei-portal`. Scala tipografica e degli
spazi in `src/theme/layout-tokens.css`, controllata da `scripts/check-tokens.mjs`
in `src/etl-canvas/inspector/**`. Prove: `__tests__/inspector*.test.ts(x)`,
`menu.test.ts`, `inspector-logic.test.ts` e `scripts/e2e-fase6b1.mjs`.

## Rendering lato server

`EtlCanvas` produce, sul server e nel primo rendering di idratazione, lo
stesso contenitore vuoto (`useSyncExternalStore` con snapshot server
`false`); dopo l'idratazione misura l'area con un `ResizeObserver` e monta
`CanvasSurface`. Nessun accesso a `window`/`document` durante il rendering.

## Carattere e temi

- Manrope (`@fontsource-variable/manrope`) è il carattere di tutta l'app,
  caricato da `src/styles.css`; `--ec-font` deriva da `--font-sans`. Nessuna
  richiesta a server esterni: i file sono serviti dall'app, e il browser
  scarica un sottoinsieme solo se il testo lo usa.
- Raggi dello stage e della minimappa derivati da `--radius` del tema (nel
  tema predefinito, stessi 20 e 14 px del prototipo). Gli altri token restano del canvas: vedi il
  report della fase di fondazione per i ruoli che l'app definisce con valori
  diversi.
- Modo e tema arrivano da `<html>` (classe `.dark` e `data-theme`, vedi
  `src/theme/README.md`); `tokens.css` non ha blocchi per modo: i token
  semantici variano da soli. Le famiglie di operazioni (`data-family` sul
  nodo) leggono `--isa-op-*`; nel tema predefinito coincidono tutte con la
  tinta unica del prototipo.
- Tema chiaro = valori del prototipo, con la riga di provenienza accanto a
  ogni token (un test verifica che i colori chiari compaiano nel prototipo).
  Tema scuro = derivato dal tema scuro dell'app, con la stessa tinta
  d'accento; i contrasti sono verificati da `__tests__/tokens.test.ts`
  (testo ≥ 4,5:1, elementi non testuali ≥ 3:1).

## Vista

`store.getState().view` (`x`, `y`, `zoom`) è l'unica fonte: pan, zoom e
minimappa la modificano con `setView`, che non entra né nel registro né
nella cronologia. Pan: barra spaziatrice + trascinamento, tasto centrale,
rotella (come nel prototipo, senza modificatore sposta la vista);
Cmd/Ctrl + rotella = zoom attorno al puntatore. Zoom tra 0,35 e 2; Adatta
usa margine 48 e zoom al più 1,25 (come il prototipo), quindi con una scena
più grande di ciò che lo zoom minimo può contenere si ferma a 0,35.

## Verifica visiva

`node scripts/visual-fase4.mjs` avvia `vite dev`, apre prototipo e nuovo
canvas a 1440 × 900 e salva in `docs/visual/fase4/`: `prototipo.png`,
`v2-chiaro.png`, `v2-scuro.png`, un ritaglio per tipo di nodo e tema, e
`misure.json` (posizioni, colori, misure lette dal DOM).

## Animazioni (Fase 4b)

Le animazioni sono un effetto visivo sopra percorsi **già calcolati**: il
motore legge `route.pts` e `route.d` di `getRoutes()` e non li scrive né li
ricalcola. Non chiama mai `settleLinks` né altre funzioni di etl-layout che
instradino (`__tests__/no-reroute.test.ts` lo verifica con uno spy); per
disegnare i punti interpolati usa solo `roundedPath`, che arrotonda punti
dati. Nessuna modifica a etl-core, etl-layout, etl-store.

### Routine del prototipo portate

| Cosa                                                                                                                                                        | Prototipo (righe)      | Qui                              |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | -------------------------------- |
| Costanti del tubo: `BASE_W` 2,1 · `SPEED` 0,16 px/ms · `BALL` 4,4 · `FRONT` 7,5 · `BACK` 19                                                                 | 1415-1418              | `flow.ts`                        |
| Profilo del tubo, gaussiana asimmetrica: `sg = u ≥ 0 ? FRONT : BACK`, `exp(-u²/sg²)`                                                                        | 1419-1422              | `tubeProfile`                    |
| `smooth01(x) = x²(3-2x)` sul tratto `min(s, len-s)/22` (il tubo emerge dalla porta e vi rientra)                                                            | 1478, 1510             | `smooth01`, `EDGE_FADE`          |
| Ciclo `len + BACK·3,2`; punto che avanza `sb = ((now-t0)·SPEED) % ciclo − BACK·1,1`; tratto `[max(0, sb−BACK·3), min(len, sb+FRONT·3,2)]`; niente se < 2 px | 1488-1497              | `flowWindow`                     |
| Contorno: un campione ogni 1,6 px (min. 10), normale alla tangente, mezzo spessore `BASE_W/2 + BALL·profilo·bordo`, riempimento `rgba(108,99,255,0.6)`      | 1499-1517              | `tubeOutline`, token `--ec-flow` |
| Flusso solo sui collegamenti attivi (`linkLive`)                                                                                                            | 1474-1477, 1489        | `linkLive` di etl-store          |
| Flusso fermo durante lo spostamento di un nodo (`flowPaused`)                                                                                               | 1423, 1485, 1985, 2072 | `engine.setGesturing`            |
| `t0` del cavo = istante del primo disegno                                                                                                                   | 1313                   | `t0` di ogni cavo nel motore     |
| Attesa: `animation: waiting 1.9s ease-in-out infinite`; `0%,100% {opacity:.45}`, `50% {opacity:.95}`; a riposo `.85`                                        | 669-670                | `waitingOpacity`                 |
| Ciclo di disegno: un solo `requestAnimationFrame` per tutto il canvas                                                                                       | 1480-1521              | `loop.ts`                        |
| Durata `.38s` della transizione della vista (`.world.easing`)                                                                                               | 136                    | `TRANSITION_MS`                  |

Valori derivati da un calcolo, non copiati a occhio: il ciclo (`len +
BACK·3,2`), la posizione (`sb`), gli estremi del tratto e il numero di
campioni si calcolano con le formule sopra; l'opacità dell'attesa è la
funzione `cubic-bezier(.42,0,.58,1)` di CSS applicata a ciascuna metà del
periodo. Le differenze e le scelte nuove sono in `NOTE_DIVERGENZE.md`.

### Risparmio energetico

- Un solo ciclo condiviso (`loop.ts`), non un timer per cavo.
- Si ferma da solo quando nessun compito ha nulla da animare (nessun cavo
  attivo, nessuna fetta vuota, nessuna transizione in corso) e riparte
  quando arriva lavoro nuovo (`wake`).
- Si ferma con `document.hidden` e riparte con `visibilitychange`.
- Con `prefers-reduced-motion: reduce` il ciclo non parte mai: il flusso è un
  tubo fermo a metà cavo, le transizioni sono istantanee, l'attesa resta a
  riposo (0,85). Se la preferenza cambia a canvas aperto, il ciclo si ferma o
  riparte.
- Nel rendering non si toccano `requestAnimationFrame`, `matchMedia`,
  `document`: il motore si avvia in un effetto, con l'ambiente del browser.
```

### `src/etl-canvas/__tests__/animator.test.ts`

267 righe

```ts
import { readFileSync, readdirSync } from "node:fs";
import { resolve } from "node:path";
import { describe, expect, it } from "vitest";
import type { Card } from "../../etl-core";
import { CARD, LABEL_H } from "../../etl-layout";
import type { Panels, View } from "../../etl-store";
import { createLoop } from "../loop";
import { createDockAnimator } from "../panels/animator";
import type { PanelEl } from "../panels/animator";
import { AUTOFIT_MS, SAFE_MARGIN, requiredNodes } from "../panels/autoFit";
import { canvasSize, restArea, slotExtent, slotsOf } from "../panels/dockArea";
import type { WorkspaceMetrics } from "../panels/dockArea";
import { fakeEnv } from "./fake-env";
import { storeWith } from "./helpers";

const WS: WorkspaceMetrics = { w: 1280, h: 660, barH: 44 };
const dense: Card[] = Array.from(
  { length: 35 },
  (_, i) => ({ id: `n${i}`, x: 40 + (i % 7) * 100, y: 40 + Math.floor(i / 7) * 110 }) as Card,
);

function setup(opts: { ws?: WorkspaceMetrics; reduced?: boolean } = {}) {
  const ws = opts.ws ?? WS;
  const store = storeWith();
  const s = store.getState();
  store.replaceState({
    ...s,
    graph: { ...s.graph, cards: Object.fromEntries(dense.map((c) => [c.id, c])) },
    panels: { tools: { side: "left", open: false }, insp: { side: "right", open: false } },
    view: { x: 0, y: 0, zoom: 1 },
  });
  const f = fakeEnv();
  if (opts.reduced) f.setReduced(true);
  const loop = createLoop(f.env);
  const vars: Record<string, string> = {};
  const el = (id: string): PanelEl => ({
    style: { setProperty: (_n, v) => void (vars[id] = v) },
  });
  const animator = createDockAnimator({ store, loop, metrics: () => ws });
  animator.register("tools", el("tools"));
  animator.register("insp", el("insp"));
  animator.start();
  const panels = (): Panels => store.getState().panels;
  return { store, f, loop, animator, vars, ws, panels };
}

type Sample = {
  view: View;
  amounts: Record<"tools" | "insp", number>;
  origin: { x: number; y: number };
  size: { w: number; h: number };
};

/** Fa girare i frame di 16 ms uno alla volta e campiona vista e pannelli a ciascuno. */
function run(t: ReturnType<typeof setup>, max = 200): Sample[] {
  const out: Sample[] = [];
  const sample = (): void => {
    const a = t.animator.debug().amounts;
    const slots = slotsOf(t.panels(), a);
    out.push({
      view: t.store.getState().view,
      amounts: { ...a },
      size: canvasSize(t.ws, slots),
      origin: {
        x: slots.filter((s) => s.side === "left").reduce((n, s) => n + slotExtent(s, t.ws), 0),
        y:
          t.ws.barH +
          slots.filter((s) => s.side === "top").reduce((n, s) => n + slotExtent(s, t.ws), 0),
      },
    });
  };
  sample();
  for (let i = 0; i < max && (t.f.pending() > 0 || t.animator.debug().animating); i++) {
    t.f.step(16);
    sample();
  }
  return out;
}

const boxOf = (c: Card, v: View, o: { x: number; y: number }) => ({
  x1: o.x + v.x + c.x * v.zoom,
  y1: o.y + v.y + c.y * v.zoom,
  x2: o.x + v.x + (c.x + CARD) * v.zoom,
  y2: o.y + v.y + (c.y + CARD + LABEL_H) * v.zoom,
});

describe("animatore: un solo orologio per pannello e vista", () => {
  it("apre l'Inspector in basso: R1–R4 a ogni frame", () => {
    const t = setup();
    const v0 = t.store.getState().view;
    const req = requiredNodes(dense, v0, restArea(t.panels(), t.ws));
    expect(req.length).toBeGreaterThanOrEqual(20);
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    const frames = run(t);
    expect(frames.length).toBeGreaterThanOrEqual(10);
    expect(frames.length).toBeLessThanOrEqual(Math.ceil(AUTOFIT_MS / 16) + 3);
    // R2: a ogni frame i nodi richiesti sono dentro il canvas di quel frame (almeno il margine)
    for (const [i, fr] of frames.entries()) {
      for (const c of req) {
        const b = boxOf(c, fr.view, { x: 0, y: 0 });
        expect(b.x1, `frame ${i}`).toBeGreaterThanOrEqual(SAFE_MARGIN - 0.5);
        expect(b.y1, `frame ${i}`).toBeGreaterThanOrEqual(SAFE_MARGIN - 0.5);
        expect(b.x2, `frame ${i}`).toBeLessThanOrEqual(fr.size.w - SAFE_MARGIN + 0.5);
        expect(b.y2, `frame ${i}`).toBeLessThanOrEqual(fr.size.h - SAFE_MARGIN + 0.5);
      }
    }
    // R3: zoom monotono, dal valore iniziale al finale
    const zooms = frames.map((fr) => fr.view.zoom);
    expect(zooms[0]).toBe(1);
    expect(zooms.at(-1)!).toBeLessThan(1);
    for (let i = 1; i < zooms.length; i++)
      expect(zooms[i]!).toBeLessThanOrEqual(zooms[i - 1]! + 1e-12);
    // R4: nessun salto (spostamento sullo schermo tra due frame ≤ 2,5 volte la media)
    for (const c of [req[0]!, req[req.length - 1]!, req[Math.floor(req.length / 2)]!]) {
      const pos = frames.map((fr) => boxOf(c, fr.view, fr.origin));
      const steps = pos.slice(1).map((p, i) => Math.hypot(p.x1 - pos[i]!.x1, p.y1 - pos[i]!.y1));
      const mean = steps.reduce((a, b) => a + b, 0) / steps.length;
      expect(Math.max(...steps)).toBeLessThanOrEqual(2.5 * mean);
    }
    // il pannello e la vista partono e finiscono insieme, con lo stesso progresso
    expect(frames[0]!.amounts.insp).toBe(0);
    expect(frames.at(-1)!.amounts.insp).toBe(1);
    expect(t.vars["insp"]).toBe("1");
  });

  it("R6: apri → chiudi senza altre azioni, la vista finale è quella iniziale", () => {
    const t = setup();
    const v0 = t.store.getState().view;
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    run(t);
    expect(t.store.getState().view.zoom).toBeLessThan(1);
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", open: false } });
    run(t);
    const v = t.store.getState().view;
    expect(v.x).toBeCloseTo(v0.x, 9);
    expect(v.y).toBeCloseTo(v0.y, 9);
    expect(v.zoom).toBeCloseTo(v0.zoom, 9);
  });

  it("un'azione dell'utente sulla vista toglie il ripristino e fa diventare suo l'intento", () => {
    const t = setup();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    run(t);
    t.store.dispatch({ type: "setView", payload: { zoom: 0.5 } });
    expect(t.animator.debug().intent).toBe(0.5);
    expect(t.animator.debug().restore).toBeNull();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", open: false } });
    run(t);
    expect(t.store.getState().view.zoom).toBeCloseTo(0.5, 9); // intento 0,5: non risale a 1
  });

  it("una rotella a metà transizione ferma la vista dov'è; il pannello finisce", () => {
    const t = setup();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    for (let i = 0; i < 8; i++) t.f.step(16);
    const before = t.store.getState().view;
    expect(before.zoom).toBeLessThan(1);
    expect(before.zoom).toBeGreaterThan(t.animator.debug().intent * 0 + 0.2);
    t.store.dispatch({ type: "setView", payload: { x: before.x - 30, y: before.y } });
    const user = t.store.getState().view;
    const frames = run(t);
    for (const fr of frames) expect(fr.view).toEqual(user);
    expect(frames.at(-1)!.amounts.insp).toBe(1);
    expect(t.animator.debug().intent).toBe(user.zoom);
  });

  it("due cambi ravvicinati: il nuovo bersaglio parte dalla vista corrente, senza salti", () => {
    const t = setup();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    for (let i = 0; i < 6; i++) t.f.step(16);
    const mid = t.store.getState().view;
    const amountMid = t.animator.debug().amounts.insp;
    expect(amountMid).toBeGreaterThan(0);
    expect(amountMid).toBeLessThan(1);
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "top", open: true } });
    expect(t.store.getState().view).toEqual(mid); // nessun salto nel momento del cambio
    expect(t.animator.debug().amounts.insp).toBe(0); // l'Inspector si riapre sul nuovo bordo
    t.f.step(16);
    expect(t.store.getState().view).toEqual(mid); // il primo frame parte da lì
    run(t);
    expect(t.animator.debug().amounts.insp).toBe(1);
  });

  it("il pannello spostato su un altro bordo lascia un guscio che si richiude e poi sparisce", () => {
    const t = setup();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "right", open: true } });
    run(t);
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom" } });
    expect(t.animator.getSnapshot().ghosts).toHaveLength(1);
    expect(t.animator.getSnapshot().ghosts[0]!.side).toBe("right");
    run(t);
    expect(t.animator.getSnapshot().ghosts).toHaveLength(0);
  });

  it("il cambio di scheda è immediato (snapNext)", () => {
    const t = setup();
    t.store.dispatch({ type: "setPanel", payload: { panel: "tools", side: "left", open: true } });
    run(t);
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "left" } });
    run(t);
    t.animator.snapNext();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", open: true } });
    expect(t.animator.debug().animating).toBe(false);
    expect(t.animator.debug().amounts).toEqual({ tools: 0, insp: 1 });
  });

  it("ridimensionamento a riposo: la vista si ricalcola subito, senza animazione", () => {
    const t = setup();
    t.animator.onMeasure(restArea(t.panels(), t.ws));
    const small = { size: { w: 500, h: 300 }, insets: { top: 0, right: 0, bottom: 0, left: 0 } };
    t.animator.onMeasure(small);
    const v = t.store.getState().view;
    expect(v.zoom).toBeLessThan(1);
    expect(t.animator.debug().animating).toBe(false);
    // la finestra torna com'era: la vista torna esattamente quella di prima
    t.animator.onMeasure(restArea(t.panels(), t.ws));
    expect(t.store.getState().view).toEqual({ x: 0, y: 0, zoom: 1 });
  });
});

describe("movimento ridotto", () => {
  it("stato finale immediato, nessun frame di animazione, R1 vale", () => {
    const t = setup({ reduced: true });
    const req = requiredNodes(dense, t.store.getState().view, restArea(t.panels(), t.ws));
    const base = t.f.rafCalls();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    expect(t.f.rafCalls()).toBe(base);
    expect(t.animator.debug().animating).toBe(false);
    expect(t.animator.debug().amounts.insp).toBe(1);
    const a1 = restArea(t.panels(), t.ws);
    const v = t.store.getState().view;
    for (const c of req) {
      const b = boxOf(c, v, { x: 0, y: 0 });
      expect(b.x2).toBeLessThanOrEqual(a1.size.w - SAFE_MARGIN + 0.5);
      expect(b.y2).toBeLessThanOrEqual(a1.size.h - a1.insets.bottom + 0.5);
    }
  });
});

describe("un solo requestAnimationFrame", () => {
  it("l'animatore usa solo il ciclo condiviso: un rAF alla volta, nessuno prima del cambio", () => {
    const t = setup();
    t.f.step(16); // aggiungere il compito fa partire un frame a vuoto, che ferma il ciclo
    expect(t.f.pending()).toBe(0);
    const base = t.f.rafCalls();
    t.store.dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    expect(t.f.rafCalls()).toBe(base + 1);
    expect(t.f.pending()).toBe(1);
    let guard = 0;
    while (t.f.pending() > 0 && guard++ < 100) {
      t.f.step(16);
      expect(t.f.pending()).toBeLessThanOrEqual(1);
    }
    expect(t.f.pending()).toBe(0); // il ciclo si ferma a transizione finita
  });

  it("nessun file dei pannelli né autoFit chiama requestAnimationFrame", () => {
    const dir = resolve(__dirname, "../panels");
    const files = readdirSync(dir).filter((f) => /\.(ts|tsx)$/.test(f));
    expect(files).toContain("animator.ts");
    for (const f of files) {
      const code = readFileSync(resolve(dir, f), "utf8").replace(/\/\*[\s\S]*?\*\/|\/\/.*$/gm, "");
      expect(code, f).not.toMatch(/requestAnimationFrame|\braf\(/);
    }
  });
});
```

### `src/etl-canvas/__tests__/autofit.test.ts`

413 righe

```ts
import { describe, expect, it } from "vitest";
import type { Card } from "../../etl-core";
import type { Panels, View } from "../../etl-store";
import { ZOOM_MAX } from "../../etl-store";
import { CARD, LABEL_H } from "../../etl-layout";
import {
  AUTOFIT_MS,
  SAFE_MARGIN,
  easing,
  fitTarget,
  interpolateArea,
  interpolateView,
  nodeInside,
  planChange,
  requiredNodes,
  safeBox,
} from "../panels/autoFit";
import type { Area } from "../panels/autoFit";
import { restArea } from "../panels/dockArea";
import type { WorkspaceMetrics } from "../panels/dockArea";
import { MIN_ZOOM, fitView } from "../view";

const card = (id: string, x: number, y: number): Card => ({ id, x, y }) as Card;
const NOINS = { top: 0, right: 0, bottom: 0, left: 0 };
const area = (w: number, h: number, insets = NOINS): Area => ({ size: { w, h }, insets });
const V1: View = { x: 0, y: 0, zoom: 1 };
const inside = (cards: readonly Card[], v: View, a: Area, tol = 1e-6) =>
  cards.every((c) => nodeInside(c, v, a, tol));

/** Scena rada: sei nodi sparsi. Scena densa: una griglia 7 × 5. */
const sparse: Card[] = [
  card("a", 40, 40),
  card("b", 400, 40),
  card("c", 700, 40),
  card("d", 40, 300),
  card("e", 700, 300),
  card("m", 400, 170),
];
const dense: Card[] = Array.from({ length: 35 }, (_, i) =>
  card(`n${i}`, 40 + (i % 7) * 150, 40 + Math.floor(i / 7) * 130),
);

const WINDOWS = [
  { name: "1440×900", w: 1440, h: 900 },
  { name: "1280×720", w: 1280, h: 720 },
  { name: "1280×600", w: 1280, h: 600 },
] as const;
/** Lo spazio di lavoro: la finestra meno l'intestazione della pagina; la barra dei controlli sta sopra. */
const ws = (w: { w: number; h: number }): WorkspaceMetrics => ({ w: w.w, h: w.h - 60, barH: 44 });
const SIDES = ["left", "right", "top", "bottom"] as const;
const closed: Panels = {
  tools: { side: "left", open: false },
  insp: { side: "right", open: false },
};
const opened = (panel: "tools" | "insp", side: (typeof SIDES)[number]): Panels => ({
  tools: { side: panel === "tools" ? side : "left", open: panel === "tools" },
  insp: { side: panel === "insp" ? side : "right", open: panel === "insp" },
});

describe("requiredNodes", () => {
  it("sono i nodi interamente dentro l'area sicura; un nodo già tagliato non è richiesto", () => {
    const cards = [card("in", 100, 100), card("cut", 950, 100), card("edge", 10, 100)];
    const req = requiredNodes(cards, V1, area(1000, 600));
    expect(req.map((c) => c.id)).toEqual(["in"]); // «cut» sporge a destra, «edge» sta nel margine di 24 px
  });

  it("l'ingombro di un widget restringe l'area sicura", () => {
    const c = [card("n", 100, 460)]; // y2 = 460 + 88 + 26 = 574
    expect(requiredNodes(c, V1, area(1000, 600))).toHaveLength(1);
    expect(requiredNodes(c, V1, area(1000, 600, { ...NOINS, bottom: 100 }))).toHaveLength(0);
    expect(safeBox(area(1000, 600))).toEqual({
      x1: SAFE_MARGIN,
      y1: SAFE_MARGIN,
      x2: 1000 - SAFE_MARGIN,
      y2: 600 - SAFE_MARGIN,
    });
  });
});

describe("fitTarget: i tre esiti", () => {
  it("1: i nodi entrano allo zoom attuale, solo scorrimento minimo, zoom e posizioni invariati", () => {
    const prev = area(1000, 600);
    const next = area(700, 600);
    const few = [card("p", 40, 40), card("q", 440, 40)]; // da 40 a 528
    const r = fitTarget({ cards: few, view: V1, intent: 1, prev, next });
    expect(r.outcome).toBe(1);
    expect(r.reason).toBe("entrano");
    expect(r.view).toEqual({ x: 0, y: 0, zoom: 1 }); // 528 ≤ 700 − 24: non serve scorrere
    const tight = fitTarget({ cards: few, view: V1, intent: 1, prev, next: area(540, 600) });
    expect(tight.outcome).toBe(1);
    expect(tight.view).toEqual({ x: 540 - 24 - 528, y: 0, zoom: 1 });
    expect(inside(few, tight.view, area(540, 600))).toBe(true);
  });

  it("2: non entrano: lo zoom scende e il riquadro dei nodi richiesti sta centrato nell'area", () => {
    const prev = area(1000, 600);
    const next = area(500, 600);
    const r = fitTarget({ cards: sparse, view: V1, intent: 1, prev, next });
    expect(r.outcome).toBe(2);
    expect(r.reason).toBe("zoom-ridotto");
    expect(r.view.zoom).toBeLessThan(1);
    expect(r.view.zoom).toBeGreaterThanOrEqual(MIN_ZOOM);
    expect(inside(sparse, r.view, next)).toBe(true);
    // centrato: avanza quanto resta a sinistra e a destra del riquadro
    const left = r.view.x + 40 * r.view.zoom;
    const right = 500 - (r.view.x + (700 + CARD) * r.view.zoom);
    expect(left).toBeCloseTo(right, 6);
  });

  it("3: nemmeno lo zoom minimo basta: zoom minimo, allineato in alto a sinistra, e il motivo lo dice", () => {
    const wide = [card("a", 40, 40), card("b", 6000, 40), card("c", 40, 3000)];
    const prev = area(8000, 4000);
    const next = area(400, 300);
    const r = fitTarget({ cards: wide, view: V1, intent: 1, prev, next });
    expect(r.outcome).toBe(3);
    expect(r.reason).toBe("zoom-minimo");
    expect(r.view.zoom).toBe(MIN_ZOOM);
    expect(r.view.x + 40 * MIN_ZOOM).toBeCloseTo(SAFE_MARGIN, 6);
    expect(r.view.y + 40 * MIN_ZOOM).toBeCloseTo(SAFE_MARGIN, 6);
  });

  it("nessun nodo richiesto: la vista non cambia (stessa istanza)", () => {
    const r = fitTarget({
      cards: [card("x", 5000, 5000)],
      view: V1,
      intent: 1,
      prev: area(800, 600),
      next: area(300, 300),
    });
    expect(r.outcome).toBe(1);
    expect(r.reason).toBe("nessun-nodo");
    expect(r.view).toBe(V1);
  });

  it("un nodo già tagliato prima non fa scendere lo zoom", () => {
    const cards = [card("p", 40, 40), card("cut", 940, 40)];
    const r = fitTarget({
      cards,
      view: V1,
      intent: 1,
      prev: area(1000, 600),
      next: area(500, 600),
    });
    expect(r.required).toBe(1);
    expect(r.view.zoom).toBe(1);
  });

  it("lo zoom di intento è il tetto: un'area grande non lo supera", () => {
    const small: View = { x: 0, y: 0, zoom: 0.5 };
    const r = fitTarget({
      cards: sparse,
      view: small,
      intent: 0.8,
      prev: area(1000, 600),
      next: area(3000, 2000),
    });
    expect(r.outcome).toBe(2);
    expect(r.reason).toBe("zoom-ripreso");
    expect(r.view.zoom).toBeCloseTo(0.8, 9);
  });

  it("l'intento sale dopo uno zoom manuale: la discesa parte da lì e non supera quello scelto", () => {
    const manual: View = { x: 10, y: 10, zoom: 1.5 };
    const few = [card("p", 40, 40), card("q", 300, 40)];
    const prev = area(1400, 800);
    const next = area(500, 800);
    const r = fitTarget({ cards: few, view: manual, intent: 1.5, prev, next });
    expect(r.view.zoom).toBeLessThanOrEqual(1.5);
    expect(r.view.zoom).toBeLessThan(1.5); // 340 × 1,5 > 452: serve scendere
    expect(inside(few, r.view, next)).toBe(true);
    // con più spazio risale fino all'intento, mai oltre
    const back = fitTarget({
      cards: few,
      view: r.view,
      intent: 1.5,
      prev: next,
      next: area(2000, 900),
    });
    expect(back.view.zoom).toBeCloseTo(1.5, 9);
    expect(ZOOM_MAX).toBeGreaterThanOrEqual(1.5);
  });
});

describe("planChange: punto di ripristino", () => {
  const prev = area(1000, 600);
  const next = area(500, 600);

  it("apri → chiudi senza altre azioni: la vista torna ESATTAMENTE quella di prima", () => {
    const v0: View = { x: 12.345, y: -6.789, zoom: 1 };
    const open = planChange({ cards: sparse, view: v0, intent: 1, prev, next, restore: null });
    expect(open.fit?.outcome).toBe(2);
    expect(open.restore).toEqual({ view: v0, area: prev });
    const close = planChange({
      cards: sparse,
      view: open.view,
      intent: 1,
      prev: next,
      next: area(1000, 600),
      restore: open.restore,
    });
    expect(close.restored).toBe(true);
    expect(close.view).toEqual(v0);
    expect(close.restore).toBeNull();
  });

  it("senza punto di ripristino (azione dell'utente) vale la regola generale con lo zoom di intento", () => {
    const v0: View = { x: 0, y: 0, zoom: 1 };
    const open = planChange({ cards: sparse, view: v0, intent: 1, prev, next, restore: null });
    const close = planChange({
      cards: sparse,
      view: open.view,
      intent: 1,
      prev: next,
      next: prev,
      restore: null,
    });
    expect(close.restored).toBe(false);
    expect(close.fit?.reason).toBe("zoom-ripreso");
    expect(close.view.zoom).toBeCloseTo(1, 9);
  });

  it("il primo adattamento salva la vista di prima; i successivi non la sostituiscono", () => {
    const v0: View = { x: 0, y: 0, zoom: 1 };
    const one = planChange({ cards: sparse, view: v0, intent: 1, prev, next, restore: null });
    const two = planChange({
      cards: sparse,
      view: one.view,
      intent: 1,
      prev: next,
      next: area(400, 600),
      restore: one.restore,
    });
    expect(two.restore).toBe(one.restore);
  });

  it("un cambio che non tocca la vista non crea un punto di ripristino", () => {
    const r = planChange({
      cards: sparse,
      view: V1,
      intent: 1,
      prev,
      next: area(1200, 800),
      restore: null,
    });
    expect(r.restore).toBeNull();
  });
});

describe("tabella pannelli × bordi × finestre × scene", () => {
  let cases = 0;
  const outcomes = { 1: 0, 2: 0, 3: 0 };
  for (const win of WINDOWS) {
    for (const [sceneName, scene] of [
      ["rada", sparse],
      ["densa", dense],
    ] as const) {
      for (const panel of ["tools", "insp"] as const) {
        for (const side of SIDES) {
          it(`${win.name}, scena ${sceneName}, ${panel} su ${side}: R1 e ripristino`, () => {
            const m = ws(win);
            const a0 = restArea(closed, m);
            const a1 = restArea(opened(panel, side), m);
            const v0: View = { x: 0, y: 0, zoom: 1 };
            const open = planChange({
              cards: scene,
              view: v0,
              intent: 1,
              prev: a0,
              next: a1,
              restore: null,
            });
            const fit = open.fit!;
            outcomes[fit.outcome]++;
            cases++;
            const req = requiredNodes(scene, v0, a0);
            if (fit.outcome !== 3) expect(inside(req, open.view, a1)).toBe(true);
            expect(open.view.zoom).toBeLessThanOrEqual(1);
            expect(open.view.zoom).toBeGreaterThanOrEqual(MIN_ZOOM);
            // a ogni istante della transizione i nodi richiesti restano dentro (convessità)
            if (fit.outcome !== 3) {
              for (let s = 0; s <= 1.0001; s += 0.05) {
                expect(
                  inside(req, interpolateView(v0, open.view, s), interpolateArea(a0, a1, s)),
                ).toBe(true);
              }
            }
            const close = planChange({
              cards: scene,
              view: open.view,
              intent: 1,
              prev: a1,
              next: a0,
              restore: open.restore,
            });
            if (open.restore) expect(close.view).toEqual(v0);
            else if (close.fit && close.fit.outcome !== 3)
              expect(inside(requiredNodes(scene, open.view, a1), close.view, a0)).toBe(true);
          });
        }
      }
    }
  }
  it("la tabella tocca tutti e tre gli esiti sul canvas con i pannelli aperti, o almeno 1 e 2", () => {
    expect(cases).toBe(3 * 2 * 2 * 4);
    expect(outcomes[1]).toBeGreaterThan(0);
    expect(outcomes[2]).toBeGreaterThan(0);
  });
});

describe("interpolateView", () => {
  const a: View = { x: 10, y: -20, zoom: 1 };
  const b: View = { x: 110, y: 80, zoom: 0.4 };
  it("estremi esatti e interpolazione lineare in x, y e zoom", () => {
    expect(interpolateView(a, b, 0)).toBe(a);
    expect(interpolateView(a, b, 1)).toBe(b);
    const m = interpolateView(a, b, 0.25);
    expect(m.x).toBeCloseTo(35, 12);
    expect(m.y).toBeCloseTo(5, 12);
    expect(m.zoom).toBeCloseTo(0.85, 12); // lineare, non logaritmica
  });
  it("lo zoom è monotono", () => {
    let prev = a.zoom;
    for (let i = 1; i <= 100; i++) {
      const z = interpolateView(a, b, i / 100).zoom;
      expect(z).toBeLessThanOrEqual(prev);
      prev = z;
    }
  });
});

describe("easing", () => {
  it("valori noti: estremi esatti, simmetrica, pendenza al più π/2", () => {
    expect(easing(0)).toBe(0);
    expect(easing(1)).toBe(1);
    expect(easing(-1)).toBe(0);
    expect(easing(2)).toBe(1);
    expect(easing(0.5)).toBeCloseTo(0.5, 12);
    expect(easing(0.25)).toBeCloseTo((1 - Math.SQRT1_2) / 2, 12);
    expect(easing(0.75)).toBeCloseTo(1 - easing(0.25), 12);
    let maxSlope = 0;
    for (let i = 0; i < 1000; i++)
      maxSlope = Math.max(maxSlope, (easing((i + 1) / 1000) - easing(i / 1000)) * 1000);
    expect(maxSlope).toBeLessThanOrEqual(Math.PI / 2 + 1e-3);
  });
  it("monotona crescente", () => {
    let prev = 0;
    for (let i = 1; i <= 1000; i++) {
      const e = easing(i / 1000);
      expect(e).toBeGreaterThanOrEqual(prev);
      prev = e;
    }
  });
  it("la durata copre almeno 10 frame a 60 fps", () => {
    expect(AUTOFIT_MS / (1000 / 60)).toBeGreaterThanOrEqual(10);
  });
});

describe("convessità: dentro all'inizio e alla fine → dentro a ogni s (200 casi, seme fisso)", () => {
  function rng(seed: number) {
    let t = seed;
    return () => {
      t = (t + 0x6d2b79f5) | 0;
      let r = Math.imul(t ^ (t >>> 15), 1 | t);
      r = (r + Math.imul(r ^ (r >>> 7), 61 | r)) ^ r;
      return ((r ^ (r >>> 14)) >>> 0) / 4294967296;
    };
  }
  it("vale per tutti i casi generati", () => {
    const rand = rng(20261005);
    let checked = 0;
    while (checked < 200) {
      const n = 2 + Math.floor(rand() * 10);
      const cards = Array.from({ length: n }, (_, i) => card(`k${i}`, rand() * 900, rand() * 600));
      const a0 = area(600 + rand() * 900, 300 + rand() * 500, {
        top: rand() * 60,
        right: rand() * 200,
        bottom: rand() * 130,
        left: rand() * 60,
      });
      const a1 = area(300 + rand() * 1200, 200 + rand() * 600, {
        top: rand() * 60,
        right: rand() * 200,
        bottom: rand() * 130,
        left: rand() * 60,
      });
      const v0: View = { x: rand() * 120 - 30, y: rand() * 120 - 30, zoom: 0.5 + rand() * 1.2 };
      const req = requiredNodes(cards, v0, a0);
      if (req.length === 0) continue;
      const fit = fitTarget({ cards, view: v0, intent: v0.zoom, prev: a0, next: a1 });
      if (fit.outcome === 3) continue;
      checked++;
      expect(inside(req, fit.view, a1)).toBe(true);
      for (let i = 0; i <= 40; i++) {
        const s = i / 40;
        expect(inside(req, interpolateView(v0, fit.view, s), interpolateArea(a0, a1, s))).toBe(
          true,
        );
      }
    }
    expect(checked).toBe(200);
  });
});

describe("fitView riusato", () => {
  it("senza opzioni è «Adatta» del prototipo (margine 48, zoom al più 1,25)", () => {
    expect(fitView([{ x: 100, y: 100 }], { w: 2000, h: 2000 }).zoom).toBe(1.25);
  });
  it("con margine 0 e tetto: il riquadro riempie l'area sicura", () => {
    const v = fitView([{ x: 0, y: 0 }], { w: 400, h: 400 }, NOINS, { pad: 0, maxZoom: 5 });
    expect(v.zoom).toBeCloseTo(Math.min(400 / CARD, 400 / (CARD + LABEL_H)), 9);
  });
});
```

