# Report di validazione — Fase 4b: animazioni del canvas (src/etl-canvas/)

## Esiti

| Controllo | Esito |
| --- | --- |
| `npx tsc --noEmit` | nessun errore |
| `npm run lint` | nessun errore (solo avvisi preesistenti di fast refresh) |
| `npm test` | 400/400 (30 file) |
| `npm run build` | riuscita |
| `node scripts/visual-fase4b.mjs` | riuscito; nessun errore né avviso in console (`docs/visual/fase4b/console.txt`) |

### Test per file

| File | Esito |
| --- | --- |
| `src/canvas/__tests__/panelPositioning.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/canvasBounds.test.ts` | 13/13 |
| `src/canvas/layout/__tests__/dropZones.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/panelRegistry.test.ts` | 10/10 |
| `src/etl-canvas/__tests__/engine.test.ts` | 20/20 |
| `src/etl-canvas/__tests__/flow.test.ts` | 16/16 |
| `src/etl-canvas/__tests__/loop.test.ts` | 10/10 |
| `src/etl-canvas/__tests__/no-reroute.test.ts` | 2/2 |
| `src/etl-canvas/__tests__/render.test.ts` | 14/14 |
| `src/etl-canvas/__tests__/ssr.test.tsx` | 4/4 |
| `src/etl-canvas/__tests__/tokens.test.ts` | 23/23 |
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

Nuovi o ampliati in questa fase (65 test in più rispetto ai 335 della 4a): `flow.test.ts` (16), `transitions.test.ts` (13), `loop.test.ts` (10), `engine.test.ts` (20), `no-reroute.test.ts` (2), più asserzioni in `render.test.ts` e `tokens.test.ts`.

## Cosa è stato fatto

Solo codice nuovo in `src/etl-canvas/`. Nessuna modifica a etl-core, etl-layout, etl-store (il tipo restituito da `getRoutes` ha già `pts` e `d`: non è servita nessuna aggiunta). Nessuna dipendenza nuova.

- `flow.ts` (puro): finestra e contorno del «tubo elastico», opacità dell'attesa, `cubic-bezier`.
- `transitions.ts` (puro): interpolazione dei punti, dissolvenza incrociata, piano della transizione.
- `loop.ts`: **un solo** ciclo `requestAnimationFrame` per tutto il canvas, con ambiente iniettabile (`browserEnv` solo nel browser).
- `engine.ts`: livello sottile che, a ogni frame, applica lo stato visivo ai soli attributi SVG che cambiano (`d` del tubo, opacità dell'attesa, `d` durante una transizione); nessun ridisegno dell'albero React.
- Integrazione: cavi e fette si registrano presso il motore (`motion.tsx`); il motore si avvia in un effetto di layout, mai durante il rendering.

## Funzioni animate portate dal prototipo (righe)

| Cosa | Prototipo | Qui |
| --- | --- | --- |
| Costanti del tubo `BASE_W` 2,1 · `SPEED` 0,16 px/ms · `BALL` 4,4 · `FRONT` 7,5 · `BACK` 19 | 1415-1418 | `flow.ts` |
| Profilo del tubo `exp(-u²/sg²)`, `sg = u ≥ 0 ? FRONT : BACK` | 1419-1422 | `tubeProfile` |
| `smooth01` sul tratto `min(s, len-s)/22` | 1478, 1510 | `smooth01`, `EDGE_FADE` |
| Ciclo `len + BACK·3,2`, punto `sb`, tratto `[sb−BACK·3, sb+FRONT·3,2]`, minimo 2 px | 1488-1497 | `flowWindow` |
| Contorno: campione ogni 1,6 px (min. 10), riempimento `rgba(108,99,255,0.6)` | 1499-1517 | `tubeOutline`, `--ec-flow` |
| Flusso solo sui cavi attivi (`linkLive`) | 1474-1477 | `linkLive` di etl-store |
| Flusso fermo durante lo spostamento di un nodo (`flowPaused`) | 1423, 1485, 1985, 2072 | `engine.setGesturing` |
| `t0` del cavo | 1313 | `t0` per cavo nel motore |
| Attesa: `waiting 1.9s ease-in-out infinite`, `.45 ↔ .95`, riposo `.85` | 669-670 | `waitingOpacity` |
| Durata della transizione della vista `.38s` | 136 | `TRANSITION_MS` |

Le formule (ciclo, posizione, estremi del tratto, numero di campioni, curva di temporizzazione) sono riportate come formule, non come numeri. Le scelte che il prototipo non specifica o in cui si discosta sono in `src/etl-canvas/NOTE_DIVERGENZE.md`: in particolare il prototipo non ha transizioni tra percorsi (fa easing per frame degli angoli delle porte e dello snodo, mescolato al ricalcolo, righe 1043-1046 e 1339-1341); qui i percorsi arrivano già calcolati e si interpolano (stesso numero di punti) o si dissolvono (numero diverso), 380 ms lineari, dissolvenza con opacità `1−t` / `t` (somma sempre 1).

## Vincolo principale: nessun ricalcolo dei percorsi

`__tests__/no-reroute.test.ts` sostituisce con spy `settleLinks`, `layoutLinks`, `chooseRoute`, `buildRoute`, `routeCandidates`, `shapeCandidates` e `autoLayout`; poi fa girare flusso, attesa, transizioni, un gesto e il ritorno, con 160 frame. Risultato: **nessuna delle sette funzioni viene chiamata** e `JSON.stringify(getRoutes())` è identico prima e dopo. Uno spy di controllo verifica che `settleLinks` scatti invece quando lo store ricalcola dopo una modifica del grafo. Per disegnare i punti interpolati il motore usa solo `roundedPath` (arrotonda punti dati).

## Risparmio energetico (test e misura nel browser reale)

| Condizione | Test | Browser reale (Chromium, 1 s di `requestAnimationFrame`) |
| --- | --- | --- |
| Movimento normale, 4 cavi attivi + 1 fetta in attesa | `loop.test.ts`, `engine.test.ts` | 60 frame/s (un solo ciclo condiviso) |
| **Scheda nascosta** (`document.hidden` + `visibilitychange`) | il ciclo si ferma subito, non riprogramma; riparte al ritorno | **0** frame/s; al ritorno visibile 61 |
| **Nulla da animare** (nessun cavo, nessuna fetta, nessuna transizione) | il ciclo si ferma da solo dopo il primo frame senza lavoro, e riparte con `wake` | **0** frame/s |
| **`prefers-reduced-motion: reduce`** | il ciclo non parte (0 chiamate a `raf`); le transizioni sono istantanee; le funzioni restituiscono lo stato finale (flusso fermo a metà cavo, attesa a 0,85); se la preferenza cambia a canvas aperto, si ferma o riparte | **0** frame/s; 4 tubi statici identici a ogni istante; attesa 0,85 |
| Rendering lato server | `render.test.ts`: `requestAnimationFrame` e `matchMedia` non vengono mai chiamati | — |

## Verifica visiva

Con orologio controllato (`performance.now`/`Date.now` sostituiti prima del caricamento; scena con un join a una tabella, un filtro e un box combinato), a 0, 250 e 500 ms dall'avvio del flusso, 1440 × 900. File in `docs/visual/fase4b/`: `prototipo-t{0,250,500}.png`, `v2-chiaro-t*.png`, `v2-scuro-t*.png`, `v2-chiaro-movimento-ridotto.png`, `prototipo.webm`, `v2-chiaro.webm`, `misure.json`.

Confronto numerico (`misure.json`), ingombro di ciascuno dei 4 tubi:

| Istante | Prototipo (x, y, largh., alt.) del primo tubo | Nuovo canvas | Scarto massimo |
| --- | --- | --- | --- |
| 0 ms | 348, 95,0, 3,1 × 2,1 | 348, 94,9, 3,1 × 2,1 | 0,1 px |
| 250 ms | 348, 90,8, 43,1 × 10,5 | 348, 90,8, 43,1 × 10,5 | 0 px |
| 500 ms | 350,1, 90,5, 81 × 10,9 | 350,1, 90,6, 81 × 10,9 | 0,1 px |

Opacità del simbolo `<>` della fetta vuota: prototipo e nuovo canvas 0,45 · 0,522 · 0,723 ai tre istanti (identiche).

Differenze visibili:

1. **Colore del flusso nel tema scuro**: `rgba(168,163,255,0.9)` (il prototipo ha solo il chiaro, `rgba(108,99,255,0.6)`); contrasto 6,05:1 sul canvas.
2. **Movimento ridotto**: non esiste nel prototipo; qui il tubo è fermo a metà cavo.
3. **Transizioni dei percorsi**: nel prototipo l'assestamento è un easing continuo degli angoli e dello snodo per frame; qui è un'interpolazione o dissolvenza di 380 ms (vedi NOTE_DIVERGENZE.md). Non si vede alle tre istantanee (i percorsi sono già assestati).
4. **Fase dell'attesa**: l'animazione CSS del prototipo parte alla creazione dell'elemento; qui parte alla registrazione della fetta, con la stessa funzione: coincide ai tre istanti perché entrambe partono insieme.
5. **Video**: due file separati (`prototipo.webm`, `v2-chiaro.webm`, 3 s ciascuno, orologio reale), non affiancati: l'ffmpeg incluso in Playwright non ha i filtri per comporli e non si può aggiungere una dipendenza.
6. Le forme dei cavi coincidono nella scena provata; nelle scene in cui l'instradamento del prototipo (continuo) e quello di etl-layout (Fase 2.1) scelgono percorsi diversi, i tubi seguono il percorso di ciascuno.
7. Sfondo di pagina, intestazione, cassetta e tacca dell'Inspector del prototipo non ci sono nel nuovo canvas (fasi 5 e 6).

## Note

- Nel tema chiaro il flusso (valore del prototipo) ha 2,21:1 sul canvas: identico al prototipo, mantenuto.
- Lo script usa `?seed=prototype` (dalla 4a il canvas nuovo è predefinito; `?canvas=v2` non serve più).
- Per fissare il tempo, lo script ricrea i cavi con annulla/ripristina (nuovo canvas) e azzera `t0` (prototipo): non è codice dell'applicazione.
