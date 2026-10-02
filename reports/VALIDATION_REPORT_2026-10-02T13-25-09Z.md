# Report di validazione — Fase 5: interazioni

**Stato: COMPLETO** sul branch `feat/interactions`. Nessun comportamento ha richiesto di fermarsi: ogni gesto chiama una funzione già presente in etl-core, etl-layout o etl-store (una sola modifica fuori dal canvas, vedi «Punti di attenzione»).

## Esiti

| Controllo | Esito |
| --- | --- |
| `npx tsc --noEmit` | nessun errore |
| `npm run lint` | 0 errori (14 avvisi preesistenti) |
| `npm test` | 764/764 (41 file; prima della fase: 590 in 37 file) |
| `npm run build` | riuscita |
| `node scripts/check-tokens.mjs` | 0 violazioni nei file controllati (nessun colore, raggio o ombra letterale nei file toccati); debito preesistente invariato: 26 |
| `node scripts/e2e-fase5.mjs` | **32/32 prove** nel browser reale (Chromium, Pointer Events veri, tastiera, rotella), nessun errore in console |
| `visual-fase4.mjs`, `visual-fase4b.mjs` | riusciti, «nessun errore né avviso in console» |

### Test per file

| File | Superati |
| --- | --- |
| `src/canvas/__tests__/panelPositioning.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/canvasBounds.test.ts` | 13/13 |
| `src/canvas/layout/__tests__/dropZones.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/panelRegistry.test.ts` | 10/10 |
| `src/etl-canvas/__tests__/drop.test.ts` | 6/6 |
| `src/etl-canvas/__tests__/engine.test.ts` | 20/20 |
| `src/etl-canvas/__tests__/flow.test.ts` | 16/16 |
| `src/etl-canvas/__tests__/gesture-render.test.ts` | 4/4 |
| `src/etl-canvas/__tests__/interaction.test.ts` | 25/25 |
| `src/etl-canvas/__tests__/keyboard.test.ts` | 15/15 |
| `src/etl-canvas/__tests__/loop.test.ts` | 10/10 |
| `src/etl-canvas/__tests__/no-reroute.test.ts` | 2/2 |
| `src/etl-canvas/__tests__/render.test.ts` | 14/14 |
| `src/etl-canvas/__tests__/ssr.test.tsx` | 4/4 |
| `src/etl-canvas/__tests__/tokens.test.ts` | 60/60 |
| `src/etl-canvas/__tests__/transitions.test.ts` | 13/13 |
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
| `src/lib/theme.test.tsx` | 1/1 |
| `src/theme/__tests__/check-tokens.test.ts` | 4/4 |
| `src/theme/__tests__/contrast.test.ts` | 227/227 |
| `src/theme/__tests__/derive.test.ts` | 10/10 |
| `src/theme/__tests__/readme.test.ts` | 1/1 |
| `src/theme/__tests__/runtime.test.ts` | 20/20 |
| `src/theme/__tests__/themes.test.ts` | 14/14 |

I nuovi: `interaction.test.ts` (25), `keyboard.test.ts` (15), `drop.test.ts` (6), `gesture-render.test.ts` (4), più i contrasti dei nuovi token (`src/theme/__tests__/contrast.test.ts`).

## Gesto → funzione di dominio chiamata

Con le righe del prototipo, nel README di `src/etl-canvas/` (sezione «Fase 5 — Gesti»).

| Gesto | Funzione chiamata |
| --- | --- |
| Trascinare un nodo | `store.beginGesture` / `updateGesture` / `commitGesture` / `cancelGesture` (un solo passo di cronologia) |
| Esito sopra un altro nodo | etl-core `relation`; al rilascio `dropAt` → `mergeBoxes` / `connect`; lo spostamento è `displace` dentro `updateGesture` |
| Trascinare su un cavo | etl-layout `linkAt`, etl-core `insertable`; al rilascio `dropAt` → `insertOnLink` |
| Porte | etl-layout `nodePorts` (punto di partenza), comando `connect`; nessun gesto di spostamento |
| Rilascio dalla cassetta | comando `addNode` (+ `paletteRelation`): `handleCanvasDrop(store, payload, point)` e `previewCanvasDrop` in `drop.ts`, anche come metodi del controller |
| Click, Maiusc+click, riquadro, click sul vuoto | comandi `select` + `inspect`; etl-layout `nodeRect`, `nodeAt` |
| Trascinare il gruppo | gesto di etl-store con più `ids` |
| Canc / Backspace | etl-core `nodesRemovedBy` (anteprima) + comando `deleteNodes` |
| Frecce (2 px; Maiusc = una cella) | comando `moveNodes` |
| Cmd/Ctrl+D, +A, Esc | comandi `duplicate`, `select`/`inspect` |
| Cmd/Ctrl+Z, +Maiusc+Z, Ctrl+Y | `store.undo()` / `store.redo()` |
| Pan (spazio, tasto centrale), zoom (Cmd/Ctrl+rotella) | già in 4a (`setView`) |

## Cosa è coperto dai test

- **Un test per ogni esito di `relation()`** (con eventi puntatore simulati): fusione, collegamento, collegamento inverso, spostamento per incompatibilità (due dataset, con il motivo di etl-core), rifiuto (dalla porta verso una lavorazione).
- 30 aggiornamenti = **un solo passo di cronologia**; annullare riporta il nodo al suo posto.
- Inserimento su cavo: accettato per una lavorazione slegata su un cavo dataset→lavorazione; **nessuna anteprima** se il nodo ha già collegamenti, se il cavo è lavorazione→output, se si trascina un dataset.
- Porte: nessun gesto di store, il nodo di origine non si sposta; punto di partenza = centro del lato.
- Selezione: click, Maiusc+click (aggiunge e toglie), riquadro (anche con Maiusc), click sul vuoto, gruppo trascinato insieme (un passo).
- Tastiera: Canc/Backspace (immediato se isolato; conferma se collegato, con anteprima di ciò che sparisce e testi singolare/plurale), Annulla/Esc, Cmd/Ctrl+D, Cmd/Ctrl+A, Esc, frecce (singolo e Maiusc; tenere premuto = un passo), annulla/ripristina, e nessuna scorciatoia in un campo di testo.
- Barra spaziatrice: non avvia selezione né trascinamento (controller + browser reale); **rotella invariata** (Ctrl+rotella zoom, rotella semplice sposta la vista, provato nel browser).
- A riposo il markup non contiene nessuna classe di gesto; i nodi hanno le quattro porte (nascoste a riposo dallo stile).

## Identità a riposo

- Schermate del tema predefinito (`visual-temi.mjs prototipo` contro i riferimenti committati): **0 pixel diversi** su 4 immagini, 1.296.000 pixel ciascuna.
- Fase 4 e 4b, regione del canvas (1.039.104 pixel): `v2-chiaro`, `v2-scuro`, movimento ridotto, t250, t500 (chiaro e scuro): **0 pixel diversi**. t0: 12 (chiaro, scarto 26) e 3 (scuro, scarto 4) pixel, il rumore di antialiasing del tubo del flusso già misurato nelle fasi precedenti. Le immagini rigenerate non sono state ricommitate.
- Le porte hanno opacità 0 a riposo; compaiono solo al passaggio del puntatore.

## Punti di attenzione (nessuno ha richiesto di fermarsi)

1. **Una modifica fuori dal canvas, senza logica nuova**: `paletteRelation` (etl-store/reduce.ts) era privata; ora è esportata (una parola) per l'anteprima del rilascio. Il suo codice non è cambiato.
2. **Selezione e Inspector**: `select` non tocca l'Inspector, quindi il livello dei gesti invia anche `inspect` (nodo = primo selezionato o quello già aperto, passaggio 0). I pannelli (`setPanel`) non si toccano: l'apertura dell'Inspector è della Fase 6.
3. **Soglia**: `DRAG_THRESHOLD` = 4 px. Il prototipo usa 5, ma non è una costante di etl-layout: si usa la soglia già in uso nell'app (cassetta attuale, 4 px) e uguale a quella del riquadro del prototipo.
4. **Inserimento su cavo**: il nodo in mano è un ostacolo e fa scansare i cavi, quindi il puntatore sta spesso lontano dal cavo disegnato (anche nel prototipo). Il test si fa sui percorsi attuali e su quelli di prima del gesto.
5. **Riquadro di selezione**: durante il trascinamento non scrive nello store (ogni `select` entra nel registro delle attività): lo mostra dallo stato dei gesti e seleziona al rilascio.
6. **Differenze dal prototipo**: Esc annulla un trascinamento in corso; annulla/ripristina non agiscono con il fuoco in un campo di testo; Cmd/Ctrl+D con la sola selezione di output non fa nulla (come etl-core).
7. **Non collegato (fuori dall'elenco della fase)**: il clic su un cavo che lo elimina (prototipo 4577) e i pulsanti di eliminazione/espansione sul nodo. `deleteLink` esiste; **decidi tu** se collegarlo ora o nella Fase 6.
8. **Nuovi token** (`--isa-drop-*`, `--isa-doomed`, `--isa-marquee-*`, `--isa-temp-link*`, `--isa-port-*`, `--isa-danger`, `--isa-shadow-drag/overlay`) in entrambi i temi, con i contrasti ≥ 3:1 (≥ 4,5:1 per il testo sul pulsante «Elimina») verificati per ogni tema e modo. Mappa aggiornata in `src/theme/README.md`.

## Schermate (`docs/visual/fase5/`)

`trascinamento-collegamento.png`, `trascinamento-fusione.png`, `trascinamento-porta.png`, `riquadro-selezione.png`, `conferma-eliminazione.png` (tema predefinito, chiaro).
