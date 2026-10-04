# 01e-etl-canvas-b.md

File in questo blocco:

- `src/etl-canvas/README.md`
- `src/etl-canvas/__tests__/controlbar.test.tsx`
- `src/etl-canvas/__tests__/drop.test.ts`
- `src/etl-canvas/__tests__/engine.test.ts`
- `src/etl-canvas/__tests__/fake-env.ts`

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

### `src/etl-canvas/__tests__/controlbar.test.tsx`

145 righe

```tsx
import { readFileSync, readdirSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { createEtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
import { ControlBar } from "../panels/ControlBar";
import { storeWith } from "./helpers";

const root = resolve(dirname(fileURLToPath(import.meta.url)), "..");

function bar(store: ReturnType<typeof createEtlStore>): string {
  return renderToStaticMarkup(
    createElement(ControlBar, {
      store,
      controller: createInteractionController(store),
      area: { w: 800, h: 500 },
    }),
  );
}

describe("barra dei controlli", () => {
  it("ha Libero/Organizzato, Riordina, Annulla, Ripristina e Svuota", () => {
    const markup = bar(storeWith());
    for (const label of ["Libero", "Organizzato", "Riordina", "Annulla", "Ripristina", "Svuota"]) {
      expect(markup).toContain(label);
    }
    expect(markup).toContain('role="toolbar"');
    // i suggerimenti riportano le scorciatoie
    expect(markup).toContain("Cmd/Ctrl+Z");
    expect(markup).toContain("Cmd/Ctrl+Maiusc+Z");
  });

  it("«Reimposta» e «Funzionalità» non ci sono, né in barra né nei file dei pannelli", () => {
    expect(bar(storeWith())).not.toMatch(/Reimposta|Funzionalit/i);
    const dir = resolve(root, "panels");
    for (const f of readdirSync(dir).filter((n) => /\.(tsx?|css)$/.test(n))) {
      const text = readFileSync(resolve(dir, f), "utf8");
      // i commenti possono citare il pulsante del prototipo che non si porta
      const code = text.replace(/\/\*[\s\S]*?\*\//g, "").replace(/\/\/.*$/gm, "");
      expect(code, f).not.toMatch(/Reimposta|Funzionalit/i);
    }
  });

  it("Annulla e Ripristina sono disabilitati senza cronologia; Svuota e Riordina senza nodi", () => {
    const empty = bar(createEtlStore());
    expect(empty).toMatch(/aria-label="Annulla"[^>]*disabled/);
    expect(empty).toMatch(/aria-label="Ripristina"[^>]*disabled/);
    expect(empty).toMatch(/aria-label="Svuota il canvas"[^>]*disabled/);
    const store = storeWith();
    store.dispatch({ type: "moveNodes", payload: { ids: ["op-join"], dx: 4, dy: 0 } });
    const full = bar(store);
    expect(full).not.toMatch(/aria-label="Annulla"[^>]*disabled/);
    expect(full).toMatch(/aria-label="Ripristina"[^>]*disabled/);
    expect(full).not.toMatch(/aria-label="Svuota il canvas"[^>]*disabled/);
  });

  it("la modalità attiva è indicata con aria-pressed", () => {
    const store = storeWith();
    store.dispatch({ type: "setMode", payload: { mode: "grid" } });
    const markup = bar(store);
    expect(markup).toMatch(/aria-pressed="true"[^>]*>Organizzato/);
    expect(markup).toMatch(/aria-pressed="false"[^>]*>Libero/);
  });
});

describe("Svuota: conferma e comando", () => {
  it("la finestra riporta il testo previsto e annullare non cambia nulla", () => {
    const store = storeWith();
    const c = createInteractionController(store);
    const graph = store.getState().graph;
    const logLength = store.getLog().length;
    c.requestClearAll();
    const confirm = c.getUi().confirm;
    expect(confirm?.kind).toBe("clear");
    expect(confirm?.text).toBe(
      "Eliminare tutti i nodi e i collegamenti? Puoi annullare con Cmd/Ctrl+Z.",
    );
    // tutti i nodi sono segnati come destinati a sparire
    expect(confirm?.removed.length).toBe(Object.keys(graph.cards).length);
    c.cancelConfirm();
    expect(c.getUi().confirm).toBeNull();
    expect(store.getState().graph).toBe(graph);
    expect(store.getLog().length).toBe(logLength);
    expect(store.historySize().past).toBe(0);
  });

  it("Esc annulla la conferma", () => {
    const store = storeWith();
    const c = createInteractionController(store);
    c.requestClearAll();
    c.key({ key: "Escape" });
    expect(c.getUi().confirm).toBeNull();
    expect(Object.keys(store.getState().graph.cards).length).toBeGreaterThan(0);
  });

  it("con la conferma aperta i comandi da tastiera del canvas non agiscono", () => {
    const store = storeWith();
    const c = createInteractionController(store);
    store.dispatch({ type: "select", payload: { ids: ["op-join"] } });
    c.requestClearAll();
    expect(c.key({ key: "Delete" })).toBe(false);
    expect(c.key({ key: "z", metaKey: true })).toBe(false);
    expect(store.getState().graph.cards["op-join"]).toBeDefined();
  });

  it("confermato: un solo passo di cronologia, una voce nel registro, libreria intatta; Annulla ripristina tutto", () => {
    const store = storeWith();
    store.dispatch({
      type: "loadDataset",
      payload: {
        name: "a",
        path: "a.csv",
        columns: [{ name: "x", type: "integer", values: [] }] as never,
        rows: 1,
      },
    });
    const c = createInteractionController(store);
    const before = store.getState();
    const past = store.historySize().past;
    const logLength = store.getLog().length;
    c.requestClearAll();
    expect(c.confirmDelete()).toEqual({ ok: true });
    const s = store.getState();
    expect(Object.keys(s.graph.cards)).toEqual([]);
    expect(s.graph.links).toEqual([]);
    expect(s.library).toEqual(before.library);
    expect(store.historySize().past).toBe(past + 1);
    expect(store.getLog().length).toBe(logLength + 1);
    expect(store.getLog().at(-1)?.type).toBe("clearAll");
    store.undo();
    expect(store.getState().graph).toEqual(before.graph);
    expect(store.getState().library).toEqual(before.library);
  });

  it("su un canvas vuoto non si chiede nulla", () => {
    const store = createEtlStore();
    const c = createInteractionController(store);
    c.requestClearAll();
    expect(c.getUi().confirm).toBeNull();
  });
});
```

### `src/etl-canvas/__tests__/drop.test.ts`

84 righe

```ts
import { describe, expect, it } from "vitest";
import { nodeCenter } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
import { handleCanvasDrop, previewCanvasDrop } from "../drop";
import { storeWith } from "./helpers";

const cardCount = (s: EtlStore) => Object.keys(s.getState().graph.cards).length;

describe("handleCanvasDrop (rilascio dalla cassetta, per la Fase 6)", () => {
  it("nel vuoto crea il nodo, un solo passo di cronologia", () => {
    const store = storeWith();
    const n = cardCount(store);
    const r = handleCanvasDrop(store, { component: "filter" }, { x: 1000, y: 700 });
    expect(r.ok).toBe(true);
    expect(cardCount(store)).toBe(n + 1);
    expect(store.historySize().past).toBe(1);
    expect(
      previewCanvasDrop(store, { component: "filter" }, { x: 1500, y: 900 }).outcome,
    ).toBeNull();
  });

  it("una lavorazione su una lavorazione si fonde", () => {
    const store = storeWith();
    const p = nodeCenter(store.getState().graph.cards["op-sort"]!);
    expect(previewCanvasDrop(store, { component: "filter" }, p)).toMatchObject({
      outcome: "merge",
      nodeId: "op-sort",
    });
    const n = cardCount(store);
    expect(handleCanvasDrop(store, { component: "filter" }, p).ok).toBe(true);
    expect(cardCount(store)).toBe(n); // assorbita: non compare da sola
    expect(store.getState().graph.cards["op-sort"]!.components).toHaveLength(2);
    expect(store.historySize().past).toBe(1);
  });

  it("un dataset su una lavorazione si collega; una lavorazione su un dataset si collega al contrario", () => {
    const a = storeWith();
    const onOp = nodeCenter(a.getState().graph.cards["op-join"]!);
    expect(previewCanvasDrop(a, { component: "dataset" }, onOp).outcome).toBe("link");
    handleCanvasDrop(a, { component: "dataset" }, onOp);
    expect(a.getState().graph.links.some((l) => l.to === "op-join")).toBe(true);

    const b = storeWith();
    const onDs = nodeCenter(b.getState().graph.cards["ds1"]!);
    expect(previewCanvasDrop(b, { component: "sort" }, onDs).outcome).toBe("link-reverse");
    handleCanvasDrop(b, { component: "sort" }, onDs);
    expect(b.getState().graph.links.some((l) => l.from === "ds1")).toBe(true);
  });

  it("una lavorazione su un cavo dataset→lavorazione vi si inserisce; un dataset no", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const key = "ds1|op-join";
    const pts = store.getRoutes()[key]!.pts;
    const p = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    expect(previewCanvasDrop(store, { component: "sort" }, p)).toMatchObject({
      outcome: "insert",
      linkKey: key,
    });
    expect(previewCanvasDrop(store, { component: "dataset" }, p).outcome).toBeNull();
    expect(handleCanvasDrop(store, { component: "sort" }, p).ok).toBe(true);
    expect(store.getState().graph.links).not.toContainEqual({ from: "ds1", to: "op-join" });
  });

  it("su un cavo lavorazione→output non si inserisce: cade nel vuoto", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const pts = store.getRoutes()["op-join|out-0"]!.pts;
    const p = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    expect(previewCanvasDrop(store, { component: "sort" }, p).outcome).toBeNull();
  });

  it("è raggiungibile dal controller, con il punto dell'area convertito in mondo", () => {
    const store = storeWith();
    store.dispatch({ type: "setView", payload: { x: 50, y: 20, zoom: 2 } });
    const c = createInteractionController(store);
    expect(c.toWorld(250, 220)).toEqual({ x: 100, y: 100 });
    const n = cardCount(store);
    expect(c.handleCanvasDrop({ component: "limit" }, c.toWorld(1200, 900)).ok).toBe(true);
    expect(cardCount(store)).toBe(n + 1);
  });
});
```

### `src/etl-canvas/__tests__/engine.test.ts`

354 righe

```ts
import { describe, expect, it } from "vitest";
import { ELBOW_R, roundedPath } from "../../etl-layout";
import { createMotionEngine } from "../engine";
import type { AttrEl, GroupLike, LinkInput, PathEl } from "../engine";
import { waitingOpacity } from "../flow";
import { TRANSITION_MS } from "../transitions";
import type { Pt } from "../transitions";
import { fakeEnv } from "./fake-env";

interface FakeEl extends PathEl {
  attrs: Record<string, string>;
}

function el(len = 200): FakeEl {
  const attrs: Record<string, string> = {};
  return {
    attrs,
    style: { opacity: "" },
    setAttribute: (n, v) => void (attrs[n] = v),
    getTotalLength: () => len,
    getPointAtLength: (s) => ({ x: s, y: 0 }),
  };
}

function group() {
  const els = {
    path: el(),
    ghost: el(),
    flow: el(),
    a: el(),
    b: el(),
  };
  const map: Record<string, unknown> = {
    ".ec-link": els.path,
    ".ec-link-ghost": els.ghost,
    ".ec-flow": els.flow,
    '[data-dot="a"]': els.a,
    '[data-dot="b"]': els.b,
  };
  const g: GroupLike = { querySelector: (s) => map[s] ?? null };
  return { g, ...els };
}

const P1: Pt[] = [
  { x: 0, y: 0 },
  { x: 100, y: 0 },
  { x: 100, y: 80 },
];
const P2: Pt[] = [
  { x: 0, y: 20 },
  { x: 140, y: 20 },
  { x: 140, y: 100 },
];
const P3: Pt[] = [
  { x: 0, y: 0 },
  { x: 140, y: 100 },
];

function link(key: string, pts: Pt[], live = true): LinkInput {
  const d = roundedPath(pts, ELBOW_R);
  return { key, live, pts, d, pa: pts[0] as Pt, pb: pts[pts.length - 1] as Pt };
}

function setup() {
  const f = fakeEnv();
  const engine = createMotionEngine();
  const g = group();
  engine.registerLink("a|b", g.g);
  engine.start(f.env);
  return { f, engine, ...g };
}

describe("flusso", () => {
  it("un cavo attivo disegna il tubo a ogni frame; il ciclo gira", () => {
    const { f, engine, flow } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    expect(engine.debug().running).toBe(true);
    f.step(16);
    const d1 = flow.attrs["d"] as string;
    expect(d1.startsWith("M ")).toBe(true);
    f.step(400);
    expect(flow.attrs["d"]).not.toBe(d1);
    expect(engine.debug().running).toBe(true);
  });

  it("un cavo non attivo non ha flusso e non tiene acceso il ciclo", () => {
    const { f, engine, flow } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    f.step(16);
    expect(flow.attrs["d"] ?? "").toBe("");
    expect(engine.debug().running).toBe(false);
    expect(f.pending()).toBe(0);
  });

  it("nessun cavo e nessuna fetta: il ciclo si ferma da solo", () => {
    const { f, engine } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step();
    expect(engine.debug().running).toBe(true);
    engine.update({ links: [], gesturing: false });
    f.step();
    expect(engine.debug().running).toBe(false);
    expect(f.pending()).toBe(0);
  });

  it("durante un gesto di trascinamento il flusso si ferma, e riprende dopo", () => {
    const { f, engine, flow } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step(100);
    expect((flow.attrs["d"] as string).length).toBeGreaterThan(0);
    engine.setGesturing(true);
    expect(flow.attrs["d"]).toBe("");
    f.step(16);
    expect(engine.debug().running).toBe(false);
    engine.setGesturing(false);
    expect(engine.debug().running).toBe(true);
    f.step(16);
    expect((flow.attrs["d"] as string).length).toBeGreaterThan(0);
  });

  it("il flusso parte da 0 quando il cavo compare (t0 del cavo)", () => {
    const { f, engine, flow } = setup();
    f.step(5000);
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step(0);
    const w = flow.attrs["d"] as string;
    expect(w.startsWith("M ")).toBe(true);
    // a 0 ms dal cavo: tratto corto vicino alla porta di uscita (s tra 0 e ~3 px)
    const xs = w
      .slice(2, -2)
      .split(" L ")
      .map((p) => parseFloat(p.split(" ")[0] as string));
    expect(Math.max(...xs)).toBeLessThan(4);
  });
});

describe("attesa delle fette vuote", () => {
  it("la opacità segue la funzione pura a partire dalla registrazione", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    const s = el();
    engine.registerSlice("out-0:1", s);
    f.step(0);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(0), 9);
    f.step(475);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(475), 9);
    f.step(475);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(950), 9);
    expect(engine.debug().running).toBe(true);
  });

  it("la fase si conserva quando React ri-registra lo stesso elemento", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    const s = el();
    engine.registerSlice("x:1", s);
    f.step(300);
    engine.registerSlice("x:1", null);
    engine.registerSlice("x:1", s);
    f.step(0);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(300), 9);
  });

  it("senza fette e senza cavi il ciclo si ferma", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    const s = el();
    engine.registerSlice("x:1", s);
    expect(engine.debug().running).toBe(true);
    engine.registerSlice("x:1", null);
    engine.update({ links: [], gesturing: false });
    f.step();
    expect(engine.debug().running).toBe(false);
    expect(engine.debug().slices).toBe(0);
  });
});

describe("transizione dei percorsi", () => {
  it("stesso numero di punti: si interpola e alla fine si ripristina il percorso calcolato", () => {
    const { f, engine, path, a, b } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    expect(path.attrs["d"]).toBeUndefined(); // primo percorso: nessuna transizione
    const next = link("a|b", P2, false);
    engine.update({ links: [next], gesturing: false });
    expect(path.attrs["d"]).toBe(roundedPath(P1, ELBOW_R)); // parte dal vecchio: niente scatto
    f.step(TRANSITION_MS / 2);
    const mid = P1.map((p, i) => ({
      x: (p.x + (P2[i] as Pt).x) / 2,
      y: (p.y + (P2[i] as Pt).y) / 2,
    }));
    expect(path.attrs["d"]).toBe(roundedPath(mid, ELBOW_R));
    expect(a.attrs["cy"]).toBe(String(mid[0]?.y));
    expect(engine.debug().running).toBe(true);
    f.step(TRANSITION_MS);
    expect(path.attrs["d"]).toBe(next.d);
    expect(a.attrs["cy"]).toBe(String(next.pa.y));
    expect(b.attrs["cx"]).toBe(String(next.pb.x));
    expect(engine.debug().running).toBe(false);
  });

  it("numero di punti diverso: dissolvenza incrociata, senza interpolare la geometria", () => {
    const { f, engine, path, ghost } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    const next = link("a|b", P3, false);
    engine.update({ links: [next], gesturing: false });
    expect(ghost.attrs["d"]).toBe(roundedPath(P1, ELBOW_R));
    f.step(TRANSITION_MS / 2);
    const o = parseFloat(ghost.style.opacity);
    const n = parseFloat(path.style.opacity);
    expect(o).toBeCloseTo(0.5, 9);
    expect(n).toBeCloseTo(0.5, 9);
    expect(o + n).toBeCloseTo(1, 9);
    expect(path.attrs["d"]).toBeUndefined(); // il tracciato nuovo non viene mai deformato
    f.step(TRANSITION_MS);
    expect(ghost.style.opacity).toBe("0");
    expect(ghost.attrs["d"]).toBe("");
    expect(path.style.opacity).toBe("");
    expect(engine.debug().running).toBe(false);
  });

  it("durante un gesto di trascinamento non c'è transizione", () => {
    const { f, engine, path, ghost } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: true });
    engine.update({ links: [link("a|b", P2, false)], gesturing: true });
    engine.update({ links: [link("a|b", P3, false)], gesturing: true });
    f.step(100);
    expect(path.attrs["d"]).toBeUndefined();
    expect(ghost.attrs["d"]).toBeUndefined();
    expect(engine.debug().running).toBe(false);
  });

  it("dopo il gesto, un cambio discreto si anima dall'ultimo percorso mostrato", () => {
    const { f, engine, path } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: true });
    engine.update({ links: [link("a|b", P2, false)], gesturing: false });
    expect(path.attrs["d"]).toBe(roundedPath(P1, ELBOW_R));
    f.step(TRANSITION_MS + 1);
    expect(path.attrs["d"]).toBe(roundedPath(P2, ELBOW_R));
  });

  it("un secondo cambio a metà transizione riparte da ciò che si vede", () => {
    const { f, engine, path } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    engine.update({ links: [link("a|b", P2, false)], gesturing: false });
    f.step(TRANSITION_MS / 2);
    const shown = path.attrs["d"];
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    expect(path.attrs["d"]).toBe(shown);
  });

  it("percorso identico: nessuna transizione", () => {
    const { f, engine, path } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    engine.update({
      links: [
        link(
          "a|b",
          P1.map((p) => ({ ...p })),
          false,
        ),
      ],
      gesturing: false,
    });
    expect(path.attrs["d"]).toBeUndefined();
    f.step(16);
    expect(engine.debug().running).toBe(false);
  });
});

describe("movimento ridotto", () => {
  it("nessun ciclo, transizioni istantanee, flusso fermo a metà cavo, attesa a riposo", () => {
    const f = fakeEnv();
    f.setReduced(true);
    const engine = createMotionEngine();
    const g = group();
    engine.registerLink("a|b", g.g);
    const slice = el();
    engine.registerSlice("x:1", slice);
    engine.start(f.env);
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.update({ links: [link("a|b", P2)], gesturing: false });
    expect(f.rafCalls()).toBe(0);
    expect(engine.debug().running).toBe(false);
    expect(g.path.attrs["d"]).toBeUndefined(); // nessuna interpolazione: resta il percorso calcolato
    expect(g.ghost.attrs["d"]).toBeUndefined();
    const still = g.flow.attrs["d"] as string;
    expect(still.startsWith("M ")).toBe(true); // indicazione statica
    engine.update({ links: [link("a|b", P2)], gesturing: false });
    expect(g.flow.attrs["d"]).toBe(still); // identica a ogni aggiornamento: nessun movimento
    expect(slice.style.opacity).toBe("");
  });

  it("l'attivazione a ciclo acceso ferma tutto e lascia lo stato finale", () => {
    const { f, engine, flow, path } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.update({ links: [link("a|b", P2)], gesturing: false });
    f.step(50);
    f.setReduced(true);
    expect(engine.debug().running).toBe(false);
    expect(path.attrs["d"]).toBe(roundedPath(P2, ELBOW_R));
    expect(flow.attrs["d"]).toBeTruthy();
  });
});

describe("scheda nascosta", () => {
  it("con la scheda nascosta il motore non consuma frame; al ritorno riparte", () => {
    const { f, engine } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step();
    f.setHidden(true);
    const calls = f.rafCalls();
    f.step();
    f.step();
    expect(f.rafCalls()).toBe(calls);
    expect(engine.debug().running).toBe(false);
    f.setHidden(false);
    expect(engine.debug().running).toBe(true);
  });
});

describe("robustezza", () => {
  it("un cavo il cui gruppo non è ancora montato non rompe nulla", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    expect(() => {
      engine.update({ links: [link("x|y", P1)], gesturing: false });
      f.step();
    }).not.toThrow();
  });

  it("gli aggiornamenti prima dell'avvio si applicano all'avvio", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    const g = group();
    engine.registerLink("a|b", g.g);
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.start(f.env);
    f.step(16);
    expect(g.flow.attrs["d"]).toBeTruthy();
  });

  it("stop ferma il ciclo", () => {
    const { f, engine } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.stop();
    expect(f.pending()).toBe(0);
    void ({} as AttrEl);
  });
});
```

### `src/etl-canvas/__tests__/fake-env.ts`

57 righe

```ts
import type { LoopEnv } from "../loop";

/** Ambiente finto: orologio, rAF e i due eventi si controllano a mano. */
export function fakeEnv() {
  let t = 0;
  let hidden = false;
  let reduced = false;
  let nextId = 1;
  const queue = new Map<number, () => void>();
  const vis = new Set<() => void>();
  const mot = new Set<() => void>();
  let rafCalls = 0;
  const env: LoopEnv = {
    raf(cb) {
      rafCalls++;
      const id = nextId++;
      queue.set(id, cb);
      return id;
    },
    caf(id) {
      queue.delete(id);
    },
    now: () => t,
    hidden: () => hidden,
    reducedMotion: () => reduced,
    onVisibilityChange(cb) {
      vis.add(cb);
      return () => vis.delete(cb);
    },
    onReducedMotionChange(cb) {
      mot.add(cb);
      return () => mot.delete(cb);
    },
  };
  return {
    env,
    /** Avanza l'orologio e fa girare i frame in coda. */
    step(dt = 16) {
      t += dt;
      const run = [...queue.values()];
      queue.clear();
      run.forEach((cb) => cb());
    },
    setHidden(h: boolean) {
      hidden = h;
      [...vis].forEach((cb) => cb());
    },
    setReduced(r: boolean) {
      reduced = r;
      [...mot].forEach((cb) => cb());
    },
    pending: () => queue.size,
    rafCalls: () => rafCalls,
    listeners: () => vis.size + mot.size,
  };
}
```

