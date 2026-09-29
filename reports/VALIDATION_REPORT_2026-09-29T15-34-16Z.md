# Report di validazione — Fase 4a: il canvas visibile (src/etl-canvas/)

## Esiti

| Controllo | Esito |
| --- | --- |
| `npx tsc --noEmit` | nessun errore |
| `npm run lint` | nessun errore |
| `npm test` | 335/335 (25 file) |
| `npm run build` | riuscita |
| `node scripts/visual-fase4.mjs` | riuscito; nessun errore né avviso in console (`docs/visual/fase4/console.txt`) |

### Test per file

| File | Esito |
| --- | --- |
| `src/canvas/__tests__/panelPositioning.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/canvasBounds.test.ts` | 13/13 |
| `src/canvas/layout/__tests__/dropZones.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/panelRegistry.test.ts` | 10/10 |
| `src/etl-canvas/__tests__/render.test.ts` | 12/12 |
| `src/etl-canvas/__tests__/ssr.test.tsx` | 3/3 |
| `src/etl-canvas/__tests__/tokens.test.ts` | 22/22 |
| `src/etl-canvas/__tests__/view.test.ts` | 12/12 |
| `src/etl-core/__tests__/csv.test.ts` | 9/9 |
| `src/etl-core/__tests__/expressions.test.ts` | 7/7 |
| `src/etl-core/__tests__/fase11-requisiti.test.ts` | 30/30 |
| `src/etl-core/__tests__/fase11.test.ts` | 11/11 |
| `src/etl-core/__tests__/mutations.test.ts` | 9/9 |
| `src/etl-core/__tests__/params.test.ts` | 11/11 |
| `src/etl-core/__tests__/relations.test.ts` | 11/11 |
| `src/etl-core/__tests__/schema.test.ts` | 5/5 |
| `src/etl-core/__tests__/state.test.ts` | 26/26 |
| `src/etl-layout/__tests__/golden.test.ts` | 17/17 |
| `src/etl-layout/__tests__/properties.test.ts` | 11/11 |
| `src/etl-layout/__tests__/unit.test.ts` | 22/22 |
| `src/etl-store/__tests__/grouping.test.ts` | 11/11 |
| `src/etl-store/__tests__/persistence.test.ts` | 9/9 |
| `src/etl-store/__tests__/react.test.ts` | 3/3 |
| `src/etl-store/__tests__/reduce.test.ts` | 40/40 |
| `src/etl-store/__tests__/store.test.ts` | 13/13 |

Nuovi (49): `render.test.ts` (scena del prototipo: 5 nodi e posizioni; classi per tipo; indicatore ambra; cavi; output pieno; output parziale con due fette di cui una vuota; box combinato; selezione; vista; stato vuoto), `view.test.ts` (Adatta con tutti i nodi visibili, limiti di zoom, zoom sul puntatore, minimappa), `tokens.test.ts` (colori chiari presenti nel prototipo; contrasti del tema scuro), `ssr.test.tsx` (rendering lato server di `EtlCanvas` e della rotta con e senza `?canvas=v2`).

## Cosa è stato fatto

- `src/etl-canvas/`: token (`tokens.css`), nodi, cavi statici da `getRoutes`, pan (spazio + trascinamento, tasto centrale, rotella), zoom con Cmd/Ctrl + rotella attorno al puntatore, controlli (−, %, +, Adatta), minimappa, stato vuoto. La vista passa da `setView` di etl-store (fuori da registro e cronologia).
- Rotta `solutions.$solutionId.etl.tsx`: nuovo canvas solo con `?canvas=v2`; senza parametro resta il vecchio (non modificato). Store per soluzione con salvataggio `isa.etl.v2.<solutionId>`. Solo in sviluppo: `seed=prototype` carica la scena del prototipo se il canvas è vuoto, e `window.__etlStore` espone lo store (usato dallo script delle schermate).
- Montaggio solo nel browser: sul server e nel primo rendering di idratazione esce sempre lo stesso contenitore vuoto; nessun avviso in console.
- Unica dipendenza nuova: `@fontsource-variable/manrope`, importata solo da `canvas.css`.
- Area dello snapshot `01e-etl-canvas`.

### Difetto trovato e corretto durante la verifica visiva

Le fette di un output parziale usavano la classe `ec-empty`, la stessa dello stato vuoto del canvas (`position:absolute; inset:0`): la fetta vuota copriva tutto il nodo. I test sul markup non lo vedevano; lo ha mostrato il confronto con il prototipo. Classi ora `ec-slice`, `ec-slice-full`, `ec-slice-empty`, con un'asserzione di regressione.

## Differenze rispetto al prototipo (tema chiaro)

Misurate dal DOM (`docs/visual/fase4/misure.json`, stessa finestra 1440 × 900), scena iniziale del prototipo. **Identici**: posizione e dimensione di ogni nodo (26,182 · 260,52 · 260,182 · 260,338 · 442,338; quadrato 88 × 88; nodo 88 × 109,13), colore, raggio (22 / 26 px), colore e dimensione delle icone (26 px), etichetta (rettangolo, corpo 10,5 px, peso 700, colore), indicatore ambra (13 px, posizione, colore #E0A23B, bordo), controlli di zoom (176 × 38, a 12 px da destra e dal basso, vetro, ombra, sfocatura, «Adatta»), minimappa (168 × 104, a 12 px da sinistra e dal basso, vetro, raggio 14), e dei cavi: tratto `rgba(108,99,255,0.34)`, spessore 2,1, estremità e raccordi arrotondati, capi r = 2,6 con lo stesso riempimento.

Differenze, anche minime:

1. **Nome del carattere**: `"Manrope Variable"` (file variabile di Fontsource) invece di `Manrope` statico da Google Fonts (pesi 500/700/800). Stessa famiglia; il variabile copre tutti i pesi.
2. **Nodi lavorazione**: `box-shadow: inset 0 0 0 1.5px transparent`, cioè un bordo che nel tema chiaro è trasparente (esiste per il tema scuro). Nessun effetto visibile.
3. **Dimensione dell'area**: lo stage del prototipo è 712 × 520 (la cassetta a sinistra ne occupa una parte, altezza fissa); quello nuovo riempie il contenitore (1408 × 738). Le posizioni relative allo stage sono identiche.
4. **Carattere del resto della pagina**: Poppins (l'app) invece di Manrope: voluto.
5. **Stato vuoto**: elemento nuovo, assente nel prototipo. Testo con `--ec-empty-ink` (#6A645A chiaro, contrasto 5,46:1) invece di `--ec-muted` (#847E74, 3,75:1).
6. **Scena iniziale**: la colonna `categoria` ha tipo `object` nel prototipo, `stringa` in etl-core.
7. **Non ancora presenti** (fasi successive): cassetta e tacca dell'Inspector, pulsante ×, porte, pulsante di espansione, effetti al passaggio del mouse e di trascinamento, animazioni (flusso sui cavi, «in attesa» sulle fette vuote), sfondo di pagina e intestazione del prototipo. La griglia: il prototipo non disegna nessuna griglia (solo lo sfondo di `.stage`), e nemmeno il canvas nuovo.
8. Con la rotella senza modificatore la vista si sposta (come nel prototipo, righe 4092-4102); non richiesto ma coerente.

Osservazione sui valori del prototipo (non sono differenze): in tema chiaro alcune coppie non raggiungono i minimi richiesti al tema scuro: etichetta accento su fondo 4,02:1, etichetta dell'output parziale 3,75:1, «Adatta» 4,29:1, cavo 1,53:1, indicatore ambra 2,08:1. Sono i valori del prototipo, mantenuti identici come richiesto.

Schermate: `docs/visual/fase4/` — `prototipo.png`, `v2-chiaro.png`, `v2-scuro.png`, `cavi-{prototipo,chiaro,scuro}.png` (scena con cavi, output parziale e box combinato) e i ritagli `crop-{dataset,lavorazione,combinato,output-parziale,output-pieno}-{prototipo,chiaro,scuro}.png`.

## Tema scuro: tabella dei token

Ruoli derivati dal tema scuro dell'app (`.dark` in `src/styles.css`); l'accento resta #6C63FF sui riempimenti. Contrasti calcolati da `contrast.ts` sui valori di `tokens.css` (sfondo del canvas = `--ec-bg` con sopra `--ec-stage`; vetro = `--ec-surface-strong` sul canvas); verificati anche dai test.

| Ruolo | Token | Chiaro | Scuro | Contrasto scuro | Minimo |
|---|---|---|---|---|---|
| Etichetta dataset/output | `--ec-accent-text` su canvas | #6c63ff | #a8a3ff | 7.13:1 | 4.5:1 |
| Etichetta lavorazione | `--ec-ink` su canvas | #262420 | #f1f2f5 | 14.31:1 | 4.5:1 |
| Etichetta output parziale | `--ec-muted` su canvas | #847e74 | #a9abb3 | 6.99:1 | 4.5:1 |
| Icona su nodo pieno | `--ec-node-fill-ink` su `--ec-node-fill` | #ffffff | #ffffff | 4.32:1 | 3:1 |
| Icona su nodo lavorazione | `--ec-node-op-ink` su `--ec-node-op` | #6c63ff | #d0ccff | 7.09:1 | 3:1 |
| Nodo pieno su canvas | `--ec-node-fill` su canvas | #6c63ff | #6c63ff | 3.71:1 | 3:1 |
| Bordo del nodo lavorazione | `--ec-node-op-border` su canvas | rgba(0, 0, 0, 0) | #7f78e6 | 4.37:1 | 3:1 |
| Icona fetta vuota | `--ec-split-empty-ink` su `--ec-split-empty` | #8f88c7 | #a8a3e6 | 5.71:1 | 3:1 |
| Cavo | `--ec-link` su canvas | rgba(108, 99, 255, 0.34) | #7f78e6 | 4.37:1 | 3:1 |
| Indicatore ambra | `--ec-warn` su canvas | #e0a23b | #e8b34f | 8.39:1 | 3:1 |
| Contorno di selezione | `--ec-select` su canvas | #6c63ff | #a8a3ff | 7.13:1 | 3:1 |
| Testo dei controlli di zoom | `--ec-ink` su vetro | #262420 | #f1f2f5 | 13.67:1 | 4.5:1 |
| «Adatta» | `--ec-accent-text` su vetro | #6c63ff | #a8a3ff | 6.81:1 | 4.5:1 |
| Nodo nella minimappa | `--ec-mm-node` su vetro | #cfc9ef | #7b74d9 | 3.89:1 | 3:1 |
| Nodo dataset nella minimappa | `--ec-mm-node-ds` su vetro | #6c63ff | #a8a3ff | 6.81:1 | 3:1 |
| Riquadro visibile (minimappa) | `--ec-mm-view-line` su vetro | #6c63ff | #a8a3ff | 6.81:1 | 3:1 |
| Testo dello stato vuoto | `--ec-empty-ink` su canvas | #6a645a | #a9abb3 | 6.99:1 | 4.5:1 |
| Titolo dello stato vuoto | `--ec-ink` su canvas | #262420 | #f1f2f5 | 14.31:1 | 4.5:1 |


Altri token scuri: sfondo `#17181d`, stage `rgba(255,255,255,0.04)`, vetro `rgba(36,37,45,0.92)`, bordo del vetro `rgba(255,255,255,0.11)`, nodo lavorazione `#3a3670`, fondo fetta `#2a2843`, separatore fette `rgba(168,163,255,0.55)`, anello dell'indicatore `#17181d`. Il nodo lavorazione (#3A3670) ha solo 1,48:1 sul canvas: la sua evidenza è affidata al bordo (4,37:1) e all'icona (7,09:1), come consente il criterio per gli elementi non testuali.

## Note

- **Difetto preesistente (non toccato)**: in sviluppo, con StrictMode, `ThemeProvider` (`src/lib/theme.tsx`) scrive «dark» in localStorage prima di leggere il tema salvato, quindi il tema chiaro salvato non si ripristina al ricaricamento. Lo script delle schermate fissa il tema con la classe `.dark`.
- Il rendering lato server non contiene mai il canvas: le soluzioni si leggono da localStorage solo nel browser, quindi sul server la rotta mostra già il caso «soluzione non trovata». Il test della rotta simula una soluzione per rendere davvero la pagina.
- Adatta ha zoom minimo 0,35 come nel prototipo: con una scena più grande di ciò che quello zoom contiene (es. 2600 × 1600 in una finestra 320 × 240) si ferma al limite (coperto da un test).
