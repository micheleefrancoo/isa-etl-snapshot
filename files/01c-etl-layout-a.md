# 01c-etl-layout-a.md

File in questo blocco:

- `src/etl-layout/NOTE_DIVERGENZE.md`
- `src/etl-layout/README.md`
- `src/etl-layout/__tests__/golden.test.ts`
- `src/etl-layout/__tests__/golden/01-dritto-allineati.json`
- `src/etl-layout/__tests__/golden/02-dritto-scorrimento.json`
- `src/etl-layout/__tests__/golden/03-oltre-scorrimento.json`
- `src/etl-layout/__tests__/golden/04-ostacolo.json`
- `src/etl-layout/__tests__/golden/05-incrocio.json`
- `src/etl-layout/__tests__/golden/06-corsie.json`
- `src/etl-layout/__tests__/golden/07-join-output-parziale.json`

---

### `src/etl-layout/NOTE_DIVERGENZE.md`

150 righe

```md
# Note di divergenza — etl-layout

Differenze rispetto al prototipo (`docs/prototype/isa-fusion-prototype.html`).
Tre gruppi: le **correzioni intenzionali** della Fase 2.1, i
**comportamenti replicati** (accettati come parte della resa approvata del
prototipo) e le **scelte necessarie** per rendere pura una geometria che
nel prototipo vive nel DOM e nel tempo.

## Correzioni intenzionali (Fase 2.1)

### C1. Convergenza dei cavi: aggiornamento sequenziale

Nel prototipo (`drawLinks`, righe 1321-1337) ogni cavo conta gli incroci
con i percorsi del fotogramma precedente di tutti gli altri
(aggiornamento simultaneo): due cavi possono inseguirsi senza fermarsi
(lo scenario golden `06-corsie` arrivava al limite di 8 passate, come il
ramo "prima del riordino" di `09-catena-riordino`). Nel prodotto si
vedrebbero cavi che cambiano percorso da soli a ogni ricalcolo.

Ora (`links.ts`, `layoutLinks`): in ogni passata i cavi si valutano
nell'ordine dei collegamenti; ciascuno conta gli incroci con i percorsi
già aggiornati in questa passata per i cavi che lo precedono e con quelli
della passata precedente per i successivi. Gli incroci si contano sui
percorsi di base (`basePts`, prima degli scostamenti sulla stessa porta e
delle corsie), che dipendono solo dalla scelta del singolo cavo. La
stabilità resta: la scelta segue le regole del prototipo (`chooseRoute`),
ma un cavo cambia percorso solo se l'alternativa costa **strettamente**
meno del percorso attuale. Poiché un incrocio costa uguale ai due cavi
coinvolti, ogni cambio fa scendere il costo totale del disegno: le passate
terminano sempre, e il risultato è un punto fisso.

I golden il cui risultato cambia (`06-corsie`, `09-catena-riordino`) non
sono rigenerati dal prototipo: le nuove attese sono in
`__tests__/golden/corretti/`, con il confronto prima/dopo (passate,
incroci, snodi, lunghezza).

### C2. Un dritto quasi allineato è perfettamente dritto

Prima era il comportamento replicato n. 1. `shapeCandidates` (righe 1218,
1223): sotto 1,5 px di differenza la forma era `straight` senza
scorrimento, e `orthogonalize` lasciava uno scalino (3 punti). Ora gli
agganci scorrono di metà ciascuno anche sotto la soglia: un cavo di forma
dritta ha sempre esattamente 2 punti.

### C3. `displace`: centri coincidenti spinti verso il basso

Prima era il n. 4. Righe 1942-1945: `len = Math.hypot(vx, vy) || 1`
rendeva irraggiungibile il ramo di ripiego `if (len < 1) { vx = 0; vy = 1; }`,
e la spinta era nulla. Ora (`displaceTarget`) con centri coincidenti, o
più vicini di 1 px, la spinta va verso il basso.

### C4. `displace` usa il limite inferiore di `clampCard`

Prima era il n. 5. Riga 1950: `worldH() - CARD - 26`, 2 px oltre
`clampCard` (riga 1539, `worldH() - CARD - LABEL_H - 6`). Ora `displace`
usa `clampPoint`. Il punto di rilascio di un passaggio sganciato (riga 2176) conserva il limite del prototipo (`DROP_BOTTOM`).

### C5. Passaggio sganciato: sotto il box, senza toccarlo

Prima era il n. 6. Riga 2178: `freeSpot(box.x, box.y + CARD + 34)`: a 122
px il punto toccava sempre il box (soglia di `overlapsAny`: 128 px) e
`freeSpot` lo spostava di lato. Ora lo scostamento è
`CARD + LABEL_H + 18` = 128 px, la distanza minima che evita la
sovrapposizione (`DETACH_OFFSET_Y`): il passaggio resta sotto il box.

### C6. Scambio di posto con un nodo senza postazione

Prima era il n. 7. Righe 2096-2098: se il nodo rilasciato non aveva una
postazione, chi occupava quella di arrivo restava senza. Ora
(`dropInSlot`) va nella postazione libera più vicina; più in generale,
dopo `dropInSlot` ogni nodo ha una postazione e nessuna postazione ha due
nodi (test di proprietà).

## Comportamenti replicati

### R1. Le priorità di `chooseRoute` sono pesi sommati, non una gerarchia stretta

Righe 1267-1268: `cost·1e6 + overBends·6000 + cr·3500 + back·3000 +
sameSide·1800 + overshoot·14 + lunghezza + changePen·0,8`. L'ordine dei
pesi segue le priorità (nodi attraversati, limite di snodi, incroci,
ripiegamenti, forma a U, lunghezza), ma i termini si sommano e si
compensano: due incroci (7000) contano più di uno snodo oltre il limite
(6000), due ripiegamenti (6000) quanto uno snodo, una fuoriuscita di oltre
~430 px (×14) più di uno snodo. È voluto: il compromesso tra i criteri è
parte della resa del prototipo. Il test di proprietà sul limite di snodi
verifica la regola solo quando esiste un'alternativa che non attraversa
nodi e non ripiega.

**Comportamento accettato: fa parte della resa approvata del prototipo.**

### R2. La stabilità guarda solo le porte

Riga 1271: il candidato "da conservare" è il migliore **con le stesse
porte** del percorso precedente, non con la stessa forma. Dalla Fase 2.1
questa regola decide ancora l'alternativa; il cambio avviene solo se
l'alternativa costa strettamente meno del percorso attuale (C1).

**Comportamento accettato: fa parte della resa approvata del prototipo.**

### R3. `autoLayout`: la distanza minima tra le righe è quasi sempre invisibile

Righe 4285-4287: la distanza minima `CARD + LABEL_H + 18` si applica solo
a colonne che quasi non entrano nell'area visibile; in quel caso il limite
superiore del mondo, o `resolveOverlaps` (che separa sotto
`CARD + LABEL_H + 28`), di solito la cancella. Si vede solo in una finestra
stretta di altezze (scenario golden `10b`, area alta 636 px).

**Comportamento accettato: fa parte della resa approvata del prototipo.**

## Scelte necessarie (non correzioni)

### Cavi a regime

Il prototipo anima porte e snodi (righe 1339-1341) e rivaluta un cavo al
più ogni 110 ms (riga 1322). Qui si calcola lo stato finale di quelle
animazioni: angolo = porta scelta, snodo = valore obiettivo
(`layoutLinks`). `settleLinks` ripete la valutazione finché i percorsi non
cambiano (massimo 8 passate; con la correzione C1 il limite non viene
raggiunto). Lo script dei golden applica al prototipo lo stesso protocollo
a passate, chiamando il suo `drawLinks` (vedi
`scripts/extract-golden.mjs`).

### Dimensioni esplicite

- Mondo: il prototipo usa `WORLD_W × WORLD_H` con pan/zoom attivo
  (`FEATURES.panZoom = true`, riga 919) e l'area visibile altrimenti; qui
  il mondo è un parametro (`world`, predefinito 2600 × 1600).
- Area visibile: `autoLayout` centra colonne e righe su
  `stage.clientWidth/clientHeight`; qui è il parametro `viewport`.
- Etichetta: il prototipo non ha un rettangolo dell'etichetta, è il DOM
  (`.label`, righe 626, 681-682). Qui `labelRect` usa la larghezza massima
  (96 px), centrata, alta `LABEL_H - 8`: l'area reale del testo può essere
  più stretta.

### Individuazione

Nel prototipo è `document.elementFromPoint` (righe 2013, 3992, 4960), con
i cavi resi sensibili da tracciati trasparenti larghi 16 px (riga 1402).
Qui: `nodeAt` sul quadrato o sull'etichetta, vince il nodo creato dopo
(come l'ordine nel DOM); `linkAt` misura la distanza dal percorso
arrotondato (curve di raccordo campionate) con tolleranza 8 px, vince il
cavo che viene dopo.

### Rilascio in modalità Organizzato

Lo scambio di posto vive dentro il gestore `pointerup` del prototipo
(righe 2093-2100), non richiamabile dall'esterno: `dropInSlot` ne porta le
istruzioni (con la correzione C6), e lo script dei golden le esegue sulle
strutture del prototipo.
```

### `src/etl-layout/README.md`

193 righe

```md
# etl-layout — Fase 2: geometria del canvas in TypeScript puro

Porta dal prototipo (`docs/prototype/isa-fusion-prototype.html`) tutta la
geometria del canvas: dimensioni dei nodi, porte, instradamento dei cavi,
percorsi SVG, corsie, griglia, postazioni della modalità Organizzato,
separazione dei nodi, riordino automatico, posizionamento dei nodi
generati, individuazione di nodi e cavi sotto un punto.

- Funzioni pure: ricevono un `Graph` (di etl-core) e restituiscono un
  nuovo `Graph` o un valore, mai modificando l'input.
- Nessun React, `window`, `document`, `localStorage`: gira identico in Node.
- Può importare da `src/etl-core/`, mai viceversa.
- Costanti numeriche identiche al prototipo, in `constants.ts`, ciascuna
  con la riga di provenienza.
- Nessuna animazione (Fase 5): i cavi sono calcolati **a regime**, vedi
  sotto.

## Moduli

```
constants.ts    costanti numeriche (con riga del prototipo)
types.ts        Point, Rect, Anchor, Shape, RouteChoice, LinkRoute, ...
nodes.ts        rettangoli di nodo ed etichetta, ostacoli, porte
routing.ts      forme candidate, costruzione dei punti, costo, chooseRoute
path.ts         percorso SVG con raccordi (stringa) e i suoi tratti
links.ts        layoutLinks / settleLinks: tutti i cavi, scostamenti, corsie
free.ts         modalità Libero: limiti, griglia, separazione, freeSpot, displace
slots.ts        modalità Organizzato: postazioni, occupazione, scambio
autoLayout.ts   riordino automatico
placement.ts    PositionFn per i nodi generati, relocateAfter
hitTest.ts      nodo e cavo sotto un punto
__tests__/      golden (parità con il prototipo), proprietà, unità
```

## Cavi a regime

Nel prototipo `drawLinks` (righe 1300-1411) rivaluta ogni cavo al più ogni
110 ms e anima l'angolo di aggancio (`easeAngle`, 0,065 per fotogramma) e
lo snodo (0,075 per fotogramma) verso il valore scelto. Qui:

- `layoutLinks(graph, prev?, opts?)` è una rivalutazione completa di tutti
  i cavi con angoli e snodi già sul valore obiettivo. `prev` (facoltativo)
  è il risultato precedente: da lì viene il percorso attuale di ogni cavo
  (stabilità) e i percorsi degli altri con cui contare gli incroci.
  Dalla Fase 2.1 l'aggiornamento è **sequenziale**: ogni cavo vede i
  percorsi già aggiornati dei cavi che lo precedono e quelli della
  passata precedente per i successivi, e cambia percorso solo se
  l'alternativa costa strettamente meno (NOTE_DIVERGENZE.md, C1). Così le
  passate terminano sempre in un punto fisso.
- `settleLinks(graph, prev?, opts?)` ripete `layoutLinks` finché i
  percorsi non cambiano più (al massimo 8 passate, limite che con
  l'aggiornamento sequenziale non viene raggiunto).
- `opts.maxBends` è il limite di snodi (predefinito 1, come `MAX_BENDS`
  del prototipo); `opts.draggingId` è il nodo in mano, che non fa da
  ostacolo.

## Posizionamento dei nodi generati

Una `PositionFn` di etl-core sceglie solo il punto del nuovo nodo. Nel
prototipo subito dopo gli altri nodi si scansano; qui quel secondo passo
è `settleNewNode`. Sequenze equivalenti al prototipo:

| Evento                   | Prototipo                  | Qui                                                                                                                                            |
| ------------------------ | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Output generato          | `spawnOutput` (1730-1772)  | `spawnOutput(g, box, nextId, outputPositionFn({mode}))`, poi `settleNewNode(g, outId, {mode})`                                                 |
| Passaggio sganciato      | `detachStep` (2137-2203)   | `detachStep(g, box, i, nextId, detachPositionFn({mode, dropPoint}))`, poi `settleNewNode(g, id, {mode})`                                       |
| Nodo inserito su un cavo | `insertOnLink` (1853-1873) | `moveNode(g, x, insertPosition(g, link))`, poi `insertOnLink(g, link, x, nextId, outputPositionFn({mode}))`, poi `settleNewNode(g, x, {mode})` |
| Dopo una fusione         | `performMerge` (1826-1834) | `relocateAfterMerge(g, target, {mode})`                                                                                                        |

## Inventario delle funzioni geometriche del prototipo

Tutte le funzioni geometriche del prototipo, con righe e responsabilità.
**Portata** indica dove si trova qui; le funzioni non portate sono
elencate con il motivo.

### Nodi e porte

| Funzione                                         | Righe              | Responsabilità                                | Portata                                                                       |
| ------------------------------------------------ | ------------------ | --------------------------------------------- | ----------------------------------------------------------------------------- |
| costanti `CARD`, `LABEL_H`, CSS `.card`/`.label` | 909, 1524, 626-682 | dimensioni di nodo ed etichetta               | `constants.ts`, `nodes.ts` (`nodeRect`, `labelRect`, `footprint`)             |
| `borderPoint`                                    | 1031-1035          | punto sul bordo del quadrato in una direzione | `nodes.ts`                                                                    |
| `axisOf`                                         | 1050-1052          | asse di uscita di una porta                   | `nodes.ts`                                                                    |
| (porte `PORTS`)                                  | 1018               | le quattro porte                              | `nodes.ts` (`nodePorts`)                                                      |
| `nearestPort`                                    | 1023-1030          | porta più vicina a un angolo                  | **non portata**: usata solo da `stickyPort`                                   |
| `stickyPort`                                     | 1037-1042          | isteresi sul cambio di porta                  | **non portata**: mai chiamata nel prototipo (la stabilità è in `chooseRoute`) |
| `easeAngle`                                      | 1043-1046          | animazione dell'angolo di aggancio            | **non portata**: animazione (Fase 5); qui a regime                            |
| `dotFill`                                        | 1047               | colore del pallino di aggancio                | **non portata**: stile, non geometria                                         |

### Instradamento dei cavi

| Funzione                                                | Righe     | Responsabilità                                                        | Portata                                                                      |
| ------------------------------------------------------- | --------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `segHitsRect`                                           | 1056-1066 | un tratto attraversa un rettangolo                                    | `routing.ts`                                                                 |
| `routeCost`                                             | 1067-1073 | attraversamenti di ostacoli                                           | `routing.ts`                                                                 |
| `buildRoute`                                            | 1075-1083 | percorso a Z con snodo `knob`                                         | `routing.ts`                                                                 |
| `defaultKnob`                                           | 1084-1088 | snodo centrale                                                        | **non portata**: mai chiamata (lo stesso calcolo è dentro `shapeCandidates`) |
| `countBends`                                            | 1089-1098 | numero di snodi                                                       | `routing.ts`                                                                 |
| `segCross`                                              | 1100-1120 | incrocio/sovrapposizione tra due tratti                               | `routing.ts`                                                                 |
| `countCrossings`                                        | 1121-1128 | incroci con gli altri cavi                                            | `routing.ts`                                                                 |
| `overshoot`                                             | 1131-1141 | uscita dal rettangolo degli agganci                                   | `routing.ts`                                                                 |
| `routeLength`                                           | 1142-1146 | lunghezza                                                             | `routing.ts`                                                                 |
| `slide`                                                 | 1148-1152 | scorrimento dell'aggancio lungo il lato                               | `routing.ts`                                                                 |
| `shapeAnchors`                                          | 1153-1155 | agganci scorsi di una forma                                           | `routing.ts`, dentro `buildFromShape`                                        |
| `orthogonalize`                                         | 1157-1171 | nessun segmento obliquo                                               | `routing.ts`                                                                 |
| `removeReversals`                                       | 1173-1189 | nessuna inversione né punto superfluo                                 | `routing.ts`                                                                 |
| `buildFromShape`                                        | 1190-1200 | punti di una forma                                                    | `routing.ts`                                                                 |
| `dirSign`                                               | 1201      | direzione di un tratto                                                | `routing.ts` (interna)                                                       |
| `backtrack`                                             | 1203-1212 | ripiegamenti rispetto alle porte                                      | `routing.ts`                                                                 |
| `shapeCandidates`                                       | 1214-1240 | forme dritta, a L, a Z                                                | `routing.ts`                                                                 |
| `chooseRoute`                                           | 1243-1279 | scelta con costo e stabilità                                          | `routing.ts` (`chooseRoute`, `routeCandidates`)                              |
| `roundedPath`                                           | 1281-1298 | percorso SVG con raccordi                                             | `path.ts` (`roundedPath`, `roundedPieces`)                                   |
| `drawLinks`                                             | 1300-1411 | tutti i cavi: porte, scostamenti sulla stessa porta, corsie, percorsi | `links.ts` (`layoutLinks`, `settleLinks`), senza animazione né DOM           |
| `shift`                                                 | 1359-1362 | scostamento lungo il bordo                                            | `links.ts` (interna)                                                         |
| `tubeProfile`, `animateBubbles`, `linkLive`, `smooth01` | 1419-1521 | animazione del flusso nei cavi                                        | **non portate**: animazione (Fase 5)                                         |

### Modalità Libero

| Funzione                 | Righe           | Responsabilità                                  | Portata                                                                                  |
| ------------------------ | --------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `applyPositions`         | 1525-1536       | scrive le posizioni nel DOM                     | **non portata**: DOM                                                                     |
| `clampCard`              | 1537-1540       | limiti del mondo                                | `free.ts` (`clampPoint`)                                                                 |
| (`GRID`, aggancio)       | 1541, 1582-1583 | griglia                                         | `free.ts` (`snapToGrid`)                                                                 |
| `compatiblePair`         | 1543-1550       | coppie che non si respingono                    | già in etl-core (`rules/relations.ts`), usata da `free.ts`                               |
| `resolveOverlaps`        | 1551-1589       | separazione dei nodi                            | `free.ts` (`resolveOverlaps`, `separateWhileDragging`, `dropFree`)                       |
| `overlapsAny`            | 1591-1594       | un punto tocca un nodo                          | `free.ts`                                                                                |
| `freeSpot`               | 1595-1608       | primo punto libero a spirale                    | `free.ts`                                                                                |
| `displace`               | 1939-1957       | spinta di un nodo incompatibile                 | `free.ts`                                                                                |
| `anyOverlap`             | 2630-2638       | c'è almeno una sovrapposizione                  | `free.ts`                                                                                |
| `handleStageWidthChange` | 2639-2675       | posizioni "home" al restringersi della finestra | **non portata**: con pan/zoom attivo (come nel prototipo) si limita a ridisegnare i cavi |

### Modalità Organizzato

| Funzione                                 | Righe                | Responsabilità                    | Portata                                   |
| ---------------------------------------- | -------------------- | --------------------------------- | ----------------------------------------- |
| `computeSlots`                           | 3910-3917            | griglia delle postazioni          | `slots.ts`                                |
| `slotTaken`                              | 3918-3920            | postazione occupata               | `slots.ts`                                |
| `nearestSlot`                            | 3921-3929            | postazione più vicina             | `slots.ts`                                |
| `firstFreeSlot`                          | 3930-3933            | libera più vicina                 | `slots.ts`                                |
| `placeInSlots`                           | 3934-3942            | ogni nodo nella sua postazione    | `slots.ts`                                |
| `assignSlots`, `assignSlotsKeepingOrder` | 3943-3951, 2679-2684 | assegnazione in ordine di lettura | `slots.ts` (`assignSlots`)                |
| `setMode`                                | 3952-3966            | cambio di modalità                | `slots.ts` (`assignSlots` / `clearSlots`) |
| (rilascio in Organizzato)                | 2093-2100            | scambio di posto                  | `slots.ts` (`dropInSlot`)                 |
| `outputSlotFor`                          | 1709-1727            | postazione per un output          | `slots.ts`                                |

### Riordino automatico e nodi generati

| Funzione                         | Righe     | Responsabilità                             | Portata                                                         |
| -------------------------------- | --------- | ------------------------------------------ | --------------------------------------------------------------- |
| `autoLayout`                     | 4220-4329 | colonne, baricentro, simmetria, parcheggio | `autoLayout.ts`                                                 |
| `relocateAfter`                  | 1675-1706 | sposta un nodo a valle di un altro         | `placement.ts`                                                  |
| `spawnOutput` (posizione)        | 1738-1746 | dove nasce un output                       | `placement.ts` (`outputPositionFn`)                             |
| `insertOnLink` (posizione)       | 1859-1860 | nodo a metà del cavo                       | `placement.ts` (`insertPosition`)                               |
| `detachStep` (posizione)         | 2175-2181 | dove finisce un passaggio sganciato        | `placement.ts` (`detachPositionFn`)                             |
| `performMerge` (posizione)       | 1826-1834 | riposizionamento dopo la fusione           | `placement.ts` (`relocateAfterMerge`)                           |
| `duplicateSelection` (posizione) | 4605      | copia spostata di `GRID * 2`               | già in etl-core (`duplicateNodes`, scostamento 52 = `GRID * 2`) |
| `renderOutputIcon`               | 1616-1635 | disegno delle "fette" dell'output          | **non portata**: disegno (Fase 5)                               |

### Individuazione e vista

| Funzione                                                     | Righe              | Responsabilità          | Portata                                                             |
| ------------------------------------------------------------ | ------------------ | ----------------------- | ------------------------------------------------------------------- |
| `document.elementFromPoint` sul nodo                         | 2013, 3992, 4960   | nodo sotto il puntatore | `hitTest.ts` (`nodeAt`)                                             |
| tracciati `.hit` larghi 16 px                                | 1402               | cavo sotto il puntatore | `hitTest.ts` (`linkAt`, tolleranza 8 px)                            |
| `worldW`, `worldH`                                           | 938-939            | dimensioni del mondo    | `constants.ts` (`WORLD_W`, `WORLD_H`), parametro `world`            |
| `applyView`, `toWorld`, `zoomAt`, `fitView`, `renderMinimap` | 940-950, 4104-4151 | pan, zoom, minimappa    | **non portate**: vista e interfaccia (fuori dall'ambito della fase) |

## Test di parità

`scripts/extract-golden.mjs` apre il prototipo in Chromium (Playwright),
costruisce gli scenari con le variabili e le funzioni globali del
prototipo e scrive `__tests__/golden/*.json`. `__tests__/golden.test.ts`
confronta l'implementazione con i file golden (tolleranza 0,5 px). I file
golden sono committati; per rigenerarli:

```
node scripts/extract-golden.mjs            # scrive i file
node scripts/extract-golden.mjs --explore  # stampa un riassunto
```

Golden corretti (Fase 2.1): gli scenari il cui risultato cambia per le
correzioni intenzionali (`06-corsie`, `09-catena-riordino`) hanno le nuove
attese in `__tests__/golden/corretti/`, con il confronto prima/dopo. Non
si rigenerano dal prototipo ma dall'implementazione:

```
UPDATE_CORRETTI=1 npx vitest run src/etl-layout/__tests__/golden.test.ts
```

Tutti gli altri golden restano identici al prototipo.

Vedi `NOTE_DIVERGENZE.md` per le correzioni intenzionali, i comportamenti
del prototipo replicati e le scelte necessarie.
```

### `src/etl-layout/__tests__/golden.test.ts`

442 righe

```ts
import { existsSync, mkdirSync, readdirSync, readFileSync, writeFileSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { describe, expect, it } from "vitest";
import { defaultParams, spawnOutput } from "../../etl-core";
import type { Card, ComponentId, Graph } from "../../etl-core";
import {
  assignSlots,
  autoLayout,
  countBends,
  countCrossings,
  routeLength,
  dropInSlot,
  outputPositionFn,
  settleLinks,
  settleNewNode,
  withPositions,
} from "..";
import type { LinkRoute, LinkRoutes, Point } from "..";

/**
 * Parità con il prototipo: i file golden sono generati eseguendo
 * docs/prototype/isa-fusion-prototype.html in Chromium
 * (scripts/extract-golden.mjs). Tolleranza di 0,5 px sulle coordinate.
 *
 * Golden corretti (Fase 2.1): i pochi scenari il cui risultato cambia per
 * effetto delle correzioni intenzionali (NOTE_DIVERGENZE.md) hanno le
 * nuove attese in golden/corretti/<nome>.json, con il confronto prima/dopo.
 * NON si rigenerano dal prototipo: si rigenerano dall'implementazione con
 *   UPDATE_CORRETTI=1 npx vitest run src/etl-layout/__tests__/golden.test.ts
 * che scrive un file corretto solo per gli scenari che differiscono dal
 * prototipo. Tutti gli altri devono restare identici al prototipo.
 */
const TOL = 0.5;
const dir = resolve(dirname(fileURLToPath(import.meta.url)), "golden");
const correctedDir = resolve(dir, "corretti");
const UPDATE = process.env["UPDATE_CORRETTI"] === "1";

interface GoldenCard {
  id: string;
  kind: "dataset" | "op";
  components: ComponentId[];
  x: number;
  y: number;
  isOutput?: boolean;
  capacity?: number;
  filled?: number;
}
interface GoldenRoute {
  from: string;
  to: string;
  portA: number;
  portB: number;
  shape: string;
  pts: Point[];
  d: string | null;
}
interface GoldenSettle {
  passes: number;
  routes: GoldenRoute[];
}
interface GoldenPos {
  id: string;
  x: number;
  y: number;
  slot?: number;
}
interface Golden {
  name: string;
  description: string;
  type: "routes" | "autoLayout" | "grid" | "spawn";
  input: {
    cards: GoldenCard[];
    links: { from: string; to: string }[];
    maxBends?: number;
    moves?: { id: string; dx: number; dy: number }[];
    drops?: { id: string; x: number; y: number }[];
    boxId?: string;
    mode?: "free" | "grid";
  };
  expected: {
    stage: { w: number; h: number };
    steps?: (GoldenSettle & { move: unknown })[] & { drop: unknown; positions: GoldenPos[] }[];
    before?: GoldenSettle;
    after?: GoldenSettle;
    positions?: GoldenPos[];
    outputId?: string;
  };
}

function load(): Golden[] {
  return readdirSync(dir)
    .filter((f) => f.endsWith(".json"))
    .sort()
    .map((f) => JSON.parse(readFileSync(resolve(dir, f), "utf8")) as Golden);
}

function buildGraph(g: Golden): Graph {
  const cards: Record<string, Card> = {};
  for (const c of g.input.cards) {
    cards[c.id] = {
      id: c.id,
      kind: c.kind,
      components: c.components,
      params: c.components.map((t) => defaultParams(t)),
      name: c.id,
      x: c.x,
      y: c.y,
      ...(c.isOutput ? { isOutput: true } : {}),
      ...(c.capacity !== undefined ? { capacity: c.capacity } : {}),
      ...(c.filled !== undefined ? { filled: c.filled } : {}),
    };
  }
  return { cards, links: g.input.links.map((l) => ({ ...l })) };
}

function expectPts(actual: readonly Point[], expected: readonly Point[], label: string): void {
  expect(actual.length, label + " (numero di punti)").toBe(expected.length);
  actual.forEach((p, i) => {
    const q = expected[i] as Point;
    expect(Math.abs(p.x - q.x), `${label} punto ${i} x`).toBeLessThanOrEqual(TOL);
    expect(Math.abs(p.y - q.y), `${label} punto ${i} y`).toBeLessThanOrEqual(TOL);
  });
}

function expectRoutes(routes: LinkRoutes, expected: GoldenSettle, label: string): void {
  const keys = Object.keys(routes);
  expect(keys, label + " (cavi)").toEqual(expected.routes.map((r) => r.from + "|" + r.to));
  for (const r of expected.routes) {
    const got = routes[r.from + "|" + r.to] as LinkRoute;
    const l = `${label} ${r.from}->${r.to}`;
    expect(got.portA, l + " portA").toBeCloseTo(r.portA, 9);
    expect(got.portB, l + " portB").toBeCloseTo(r.portB, 9);
    expect(got.shape.kind, l + " forma").toBe(r.shape);
    expectPts(got.pts, r.pts, l);
    expect(got.d, l + " percorso SVG").toBe(r.d);
  }
}

function expectPositions(
  graph: Graph,
  expected: GoldenPos[],
  label: string,
  rename = new Map<string, string>(),
): void {
  const ids = Object.keys(graph.cards).map((id) => rename.get(id) ?? id);
  expect(ids, label + " (nodi)").toEqual(expected.map((p) => p.id));
  for (const p of expected) {
    const id = [...rename.entries()].find(([, v]) => v === p.id)?.[0] ?? p.id;
    const c = graph.cards[id] as Card;
    expect(Math.abs(c.x - p.x), `${label} ${p.id}.x`).toBeLessThanOrEqual(TOL);
    expect(Math.abs(c.y - p.y), `${label} ${p.id}.y`).toBeLessThanOrEqual(TOL);
    expect(c.slot, `${label} ${p.id}.slot`).toBe(p.slot);
  }
}

type Expected = Golden["expected"];

function toGoldenSettle(routes: LinkRoutes, passes: number): GoldenSettle {
  return {
    passes,
    routes: Object.values(routes).map((r) => ({
      from: r.from,
      to: r.to,
      portA: r.portA,
      portB: r.portB,
      shape: r.shape.kind,
      pts: r.pts.map((p) => ({ x: p.x, y: p.y })),
      d: r.d,
    })),
  };
}

function toPositions(graph: Graph, rename = new Map<string, string>()): GoldenPos[] {
  return Object.values(graph.cards).map((c) => ({
    id: rename.get(c.id) ?? c.id,
    x: c.x,
    y: c.y,
    ...(c.slot !== undefined ? { slot: c.slot } : {}),
  }));
}

/** Esegue lo scenario con l'implementazione e restituisce il risultato nel formato dei golden. */
function compute(g: Golden): Expected {
  let graph = buildGraph(g);
  const maxBends = g.input.maxBends ?? 1;
  const stage = g.expected.stage;
  if (g.type === "routes") {
    let prev: LinkRoutes = {};
    const steps: (GoldenSettle & { move: unknown })[] = [];
    const moves = [null, ...(g.input.moves ?? [])];
    for (const move of moves) {
      if (move) {
        const c = graph.cards[move.id] as Card;
        graph = withPositions(graph, new Map([[move.id, { x: c.x + move.dx, y: c.y + move.dy }]]));
      }
      const { routes, passes } = settleLinks(graph, prev, { maxBends });
      steps.push({ move, ...toGoldenSettle(routes, passes) });
      prev = routes;
    }
    return { stage, steps } as Expected;
  }
  if (g.type === "autoLayout") {
    const before = settleLinks(graph, {}, { maxBends });
    graph = autoLayout(graph, { viewport: stage });
    const positions = toPositions(graph);
    const after = settleLinks(graph, before.routes, { maxBends });
    return {
      stage,
      before: toGoldenSettle(before.routes, before.passes),
      positions,
      after: toGoldenSettle(after.routes, after.passes),
    };
  }
  if (g.type === "grid") {
    graph = assignSlots(graph);
    const steps: { drop: unknown; positions: GoldenPos[] }[] = [
      { drop: null, positions: toPositions(graph) },
    ];
    for (const drop of g.input.drops ?? []) {
      graph = dropInSlot(graph, drop.id, drop.x, drop.y);
      steps.push({ drop, positions: toPositions(graph) });
    }
    return { stage, steps } as Expected;
  }
  const boxId = g.input.boxId as string;
  const mode = g.input.mode ?? "free";
  if (mode === "grid") graph = assignSlots(graph);
  const spawned = spawnOutput(graph, boxId, () => "OUT", outputPositionFn({ mode }));
  expect(spawned).not.toBeNull();
  graph = settleNewNode(spawned?.graph as Graph, "OUT", { mode });
  const outputId = g.expected.outputId as string;
  return { stage, outputId, positions: toPositions(graph, new Map([["OUT", outputId]])) };
}

function sameSettle(a: GoldenSettle, b: GoldenSettle): boolean {
  if (a.routes.length !== b.routes.length) return false;
  return a.routes.every((r, i) => {
    const q = b.routes[i] as GoldenRoute;
    return (
      r.from === q.from &&
      r.to === q.to &&
      Math.abs(r.portA - q.portA) < 1e-9 &&
      Math.abs(r.portB - q.portB) < 1e-9 &&
      r.shape === q.shape &&
      r.d === q.d &&
      r.pts.length === q.pts.length &&
      r.pts.every((p, j) => {
        const o = q.pts[j] as Point;
        return Math.abs(p.x - o.x) <= TOL && Math.abs(p.y - o.y) <= TOL;
      })
    );
  });
}

/** Tutti i "risultati di cavi" di uno scenario, con un'etichetta. */
function settlesOf(e: Expected): [string, GoldenSettle][] {
  if (e.before && e.after)
    return [
      ["prima del riordino", e.before],
      ["dopo il riordino", e.after],
    ];
  return (e.steps ?? [])
    .filter((s) => Array.isArray((s as GoldenSettle).routes))
    .map((s, i) => [i === 0 ? "iniziale" : `dopo lo spostamento ${i}`, s as GoldenSettle]);
}

/** Incroci (coppie di tratti fra cavi diversi), snodi e lunghezza totale. */
function metrics(s: GoldenSettle): {
  passate: number;
  incroci: number;
  snodi: number;
  lunghezza: number;
} {
  let incroci = 0;
  s.routes.forEach((r, i) => {
    for (let j = i + 1; j < s.routes.length; j++)
      incroci += countCrossings(r.pts, [(s.routes[j] as GoldenRoute).pts]);
  });
  return {
    passate: s.passes,
    incroci,
    snodi: s.routes.reduce((n, r) => n + countBends(r.pts), 0),
    lunghezza: Math.round(s.routes.reduce((n, r) => n + routeLength(r.pts), 0) * 100) / 100,
  };
}

function resultChanged(prototype: Expected, actual: Expected): boolean {
  const a = settlesOf(prototype);
  const b = settlesOf(actual);
  if (a.length !== b.length) return true;
  if (a.some(([, s], i) => !sameSettle(s, (b[i] as [string, GoldenSettle])[1]))) return true;
  return JSON.stringify(prototype.positions ?? null) !== JSON.stringify(actual.positions ?? null)
    ? !(prototype.positions ?? []).every((p, i) => {
        const q = (actual.positions ?? [])[i];
        return !!q && q.id === p.id && Math.abs(q.x - p.x) <= TOL && Math.abs(q.y - p.y) <= TOL;
      })
    : false;
}

interface Corrected {
  name: string;
  description: string;
  motivo: string;
  confronto: {
    risultato: string;
    prototipo: ReturnType<typeof metrics>;
    corretto: ReturnType<typeof metrics>;
  }[];
  expected: Expected;
}

const goldens = load();

if (UPDATE) {
  mkdirSync(correctedDir, { recursive: true });
  for (const g of goldens) {
    if (g.type !== "routes" && g.type !== "autoLayout") continue;
    const actual = compute(g);
    if (!resultChanged(g.expected, actual)) continue;
    const proto = settlesOf(g.expected);
    const mine = settlesOf(actual);
    const corrected: Corrected = {
      name: g.name,
      description: g.description,
      motivo:
        "Convergenza dei cavi (Fase 2.1): aggiornamento sequenziale con miglioramento stretto. " +
        "Il prototipo aggiorna i cavi in simultanea e qui non converge o converge altrove.",
      confronto: proto.map(([label, s], i) => ({
        risultato: label,
        prototipo: metrics(s),
        corretto: metrics((mine[i] as [string, GoldenSettle])[1]),
      })),
      expected: actual,
    };
    writeFileSync(
      resolve(correctedDir, g.name + ".json"),
      JSON.stringify(corrected, null, 2) + "\n",
    );
  }
}

function correctedFor(name: string): Corrected | null {
  const f = resolve(correctedDir, name + ".json");
  return existsSync(f) ? (JSON.parse(readFileSync(f, "utf8")) as Corrected) : null;
}

function expectSame(actual: Expected, expected: Expected, g: Golden): void {
  const a = settlesOf(actual);
  const e = settlesOf(expected);
  expect(a.length).toBe(e.length);
  e.forEach(([label, s], i) => {
    const got = (a[i] as [string, GoldenSettle])[1];
    expect(got.passes, `${label}: passate`).toBe(s.passes);
    const routes: Record<string, LinkRoute> = {};
    for (const r of got.routes)
      routes[r.from + "|" + r.to] = {
        ...r,
        shape: { kind: r.shape } as LinkRoute["shape"],
        basePts: r.pts,
        horiz: false,
        pa: r.pts[0] as Point,
        pb: r.pts[r.pts.length - 1] as Point,
        d: r.d ?? "",
      };
    expectRoutes(routes, s, label);
  });
  if (g.type === "autoLayout") {
    expect((actual.positions ?? []).map((p) => p.id)).toEqual(
      (expected.positions ?? []).map((p) => p.id),
    );
    (expected.positions ?? []).forEach((p, i) => {
      const q = (actual.positions ?? [])[i] as GoldenPos;
      expect(Math.abs(q.x - p.x), `${p.id}.x`).toBeLessThanOrEqual(TOL);
      expect(Math.abs(q.y - p.y), `${p.id}.y`).toBeLessThanOrEqual(TOL);
    });
  }
  if (g.type === "grid" || g.type === "spawn") {
    const eSteps =
      g.type === "grid" ? (expected.steps ?? []) : [{ positions: expected.positions ?? [] }];
    const aSteps =
      g.type === "grid" ? (actual.steps ?? []) : [{ positions: actual.positions ?? [] }];
    expect(aSteps.length).toBe(eSteps.length);
    eSteps.forEach((st, i) => {
      const got = (aSteps[i] as { positions: GoldenPos[] }).positions;
      expect(
        got.map((p) => p.id),
        `passo ${i} (nodi)`,
      ).toEqual(st.positions.map((p) => p.id));
      st.positions.forEach((p, j) => {
        const q = got[j] as GoldenPos;
        expect(Math.abs(q.x - p.x), `passo ${i} ${p.id}.x`).toBeLessThanOrEqual(TOL);
        expect(Math.abs(q.y - p.y), `passo ${i} ${p.id}.y`).toBeLessThanOrEqual(TOL);
        expect(q.slot, `passo ${i} ${p.id}.slot`).toBe(p.slot);
      });
    });
  }
}

describe("parità con il prototipo (file golden)", () => {
  it("i file golden ci sono", () => {
    expect(goldens.length).toBeGreaterThanOrEqual(14);
  });

  for (const g of goldens) {
    const corrected = correctedFor(g.name);
    const title = corrected
      ? `${g.name} [corretto, Fase 2.1]: ${g.description}`
      : `${g.name}: ${g.description}`;
    it(title, () => {
      expectSame(compute(g), corrected ? corrected.expected : g.expected, g);
    });
  }
});

describe("golden corretti (Fase 2.1)", () => {
  const files = existsSync(correctedDir)
    ? readdirSync(correctedDir).filter((f) => f.endsWith(".json"))
    : [];

  it("ogni golden corretto corrisponde a uno scenario del prototipo e differisce davvero", () => {
    for (const f of files) {
      const c = JSON.parse(readFileSync(resolve(correctedDir, f), "utf8")) as Corrected;
      const g = goldens.find((x) => x.name === c.name);
      expect(g, f).toBeDefined();
      expect(resultChanged((g as Golden).expected, c.expected), f).toBe(true);
    }
  });

  it("i golden corretti non peggiorano incroci e snodi e convergono prima del limite", () => {
    for (const f of files) {
      const c = JSON.parse(readFileSync(resolve(correctedDir, f), "utf8")) as Corrected;
      for (const row of c.confronto) {
        expect(row.corretto.incroci, `${f} ${row.risultato}`).toBeLessThanOrEqual(
          row.prototipo.incroci,
        );
        expect(row.corretto.passate, `${f} ${row.risultato}`).toBeLessThan(8);
      }
    }
  });
});
```

### `src/etl-layout/__tests__/golden/01-dritto-allineati.json`

62 righe

```json
{
  "name": "01-dritto-allineati",
  "description": "Due nodi allineati orizzontalmente: cavo dritto.",
  "type": "routes",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 312
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 416,
        "y": 312
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "B",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "straight",
            "pts": [
              {
                "x": 192,
                "y": 356
              },
              {
                "x": 416,
                "y": 356
              }
            ],
            "d": "M 192.00 356.00 L 416.00 356.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/02-dritto-scorrimento.json`

62 righe

```json
{
  "name": "02-dritto-scorrimento",
  "description": "Disallineati di 20 px, entro lo scorrimento delle porte: ancora dritto.",
  "type": "routes",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 312
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 416,
        "y": 332
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "B",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "straight",
            "pts": [
              {
                "x": 192,
                "y": 366
              },
              {
                "x": 416,
                "y": 366
              }
            ],
            "d": "M 192.00 366.00 L 416.00 366.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/03-oltre-scorrimento.json`

66 righe

```json
{
  "name": "03-oltre-scorrimento",
  "description": "Disallineati di 130 px, oltre lo scorrimento: forma a L o a Z.",
  "type": "routes",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 312
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 416,
        "y": 442
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "B",
            "portA": 0,
            "portB": -1.5707963267948966,
            "shape": "L",
            "pts": [
              {
                "x": 192,
                "y": 356
              },
              {
                "x": 460,
                "y": 356
              },
              {
                "x": 460,
                "y": 442
              }
            ],
            "d": "M 192.00 356.00 L 449.00 356.00 Q 460.00 356.00 460.00 367.00 L 460.00 442.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/04-ostacolo.json`

77 righe

```json
{
  "name": "04-ostacolo",
  "description": "Un nodo ostruisce il percorso diretto: il cavo lo aggira.",
  "type": "routes",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 312
      },
      {
        "id": "X",
        "kind": "op",
        "components": ["sort"],
        "x": 286,
        "y": 312
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 520,
        "y": 312
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "B",
            "portA": -1.5707963267948966,
            "portB": -1.5707963267948966,
            "shape": "Z",
            "pts": [
              {
                "x": 148,
                "y": 312
              },
              {
                "x": 148,
                "y": 289
              },
              {
                "x": 564,
                "y": 289
              },
              {
                "x": 564,
                "y": 312
              }
            ],
            "d": "M 148.00 312.00 L 148.00 300.00 Q 148.00 289.00 159.00 289.00 L 553.00 289.00 Q 564.00 289.00 564.00 300.00 L 564.00 312.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/05-incrocio.json`

114 righe

```json
{
  "name": "05-incrocio",
  "description": "Due cavi che si incrocerebbero con il percorso più corto.",
  "type": "routes",
  "input": {
    "cards": [
      {
        "id": "A1",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 208
      },
      {
        "id": "A2",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 468
      },
      {
        "id": "B1",
        "kind": "op",
        "components": ["filter"],
        "x": 520,
        "y": 468
      },
      {
        "id": "B2",
        "kind": "op",
        "components": ["sort"],
        "x": 520,
        "y": 208
      }
    ],
    "links": [
      {
        "from": "A1",
        "to": "B1"
      },
      {
        "from": "A2",
        "to": "B2"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A1",
            "to": "B1",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 252
              },
              {
                "x": 356,
                "y": 252
              },
              {
                "x": 356,
                "y": 512
              },
              {
                "x": 520,
                "y": 512
              }
            ],
            "d": "M 192.00 252.00 L 345.00 252.00 Q 356.00 252.00 356.00 263.00 L 356.00 501.00 Q 356.00 512.00 367.00 512.00 L 520.00 512.00"
          },
          {
            "from": "A2",
            "to": "B2",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 512
              },
              {
                "x": 370,
                "y": 512
              },
              {
                "x": 370,
                "y": 252
              },
              {
                "x": 520,
                "y": 252
              }
            ],
            "d": "M 192.00 512.00 L 359.00 512.00 Q 370.00 512.00 370.00 501.00 L 370.00 263.00 Q 370.00 252.00 381.00 252.00 L 520.00 252.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/06-corsie.json`

159 righe

```json
{
  "name": "06-corsie",
  "description": "Più cavi nello stesso corridoio (snodi ammessi: 2, perché nascano forme a Z).",
  "type": "routes",
  "input": {
    "maxBends": 2,
    "cards": [
      {
        "id": "A1",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 104
      },
      {
        "id": "A2",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 234
      },
      {
        "id": "A3",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 364
      },
      {
        "id": "B1",
        "kind": "op",
        "components": ["filter"],
        "x": 546,
        "y": 494
      },
      {
        "id": "B2",
        "kind": "op",
        "components": ["sort"],
        "x": 546,
        "y": 624
      },
      {
        "id": "B3",
        "kind": "op",
        "components": ["aggregate"],
        "x": 546,
        "y": 754
      }
    ],
    "links": [
      {
        "from": "A1",
        "to": "B1"
      },
      {
        "from": "A2",
        "to": "B2"
      },
      {
        "from": "A3",
        "to": "B3"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 8,
        "routes": [
          {
            "from": "A1",
            "to": "B1",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 148
              },
              {
                "x": 523,
                "y": 148
              },
              {
                "x": 523,
                "y": 538
              },
              {
                "x": 546,
                "y": 538
              }
            ],
            "d": "M 192.00 148.00 L 512.00 148.00 Q 523.00 148.00 523.00 159.00 L 523.00 527.00 Q 523.00 538.00 534.00 538.00 L 546.00 538.00"
          },
          {
            "from": "A2",
            "to": "B2",
            "portA": 0,
            "portB": 0,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 278
              },
              {
                "x": 657,
                "y": 278
              },
              {
                "x": 657,
                "y": 668
              },
              {
                "x": 634,
                "y": 668
              }
            ],
            "d": "M 192.00 278.00 L 646.00 278.00 Q 657.00 278.00 657.00 289.00 L 657.00 657.00 Q 657.00 668.00 646.00 668.00 L 634.00 668.00"
          },
          {
            "from": "A3",
            "to": "B3",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 408
              },
              {
                "x": 215,
                "y": 408
              },
              {
                "x": 215,
                "y": 798
              },
              {
                "x": 546,
                "y": 798
              }
            ],
            "d": "M 192.00 408.00 L 204.00 408.00 Q 215.00 408.00 215.00 419.00 L 215.00 787.00 Q 215.00 798.00 226.00 798.00 L 546.00 798.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/07-join-output-parziale.json`

131 righe

```json
{
  "name": "07-join-output-parziale",
  "description": "Un box con due ingressi da un join e il suo output parziale.",
  "type": "routes",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 208
      },
      {
        "id": "B",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 442
      },
      {
        "id": "J",
        "kind": "op",
        "components": ["join"],
        "x": 364,
        "y": 312
      },
      {
        "id": "O",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 572,
        "y": 312,
        "isOutput": true,
        "capacity": 2,
        "filled": 1
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "J"
      },
      {
        "from": "B",
        "to": "J"
      },
      {
        "from": "J",
        "to": "O"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "J",
            "portA": 0,
            "portB": -1.5707963267948966,
            "shape": "L",
            "pts": [
              {
                "x": 192,
                "y": 252
              },
              {
                "x": 408,
                "y": 252
              },
              {
                "x": 408,
                "y": 312
              }
            ],
            "d": "M 192.00 252.00 L 397.00 252.00 Q 408.00 252.00 408.00 263.00 L 408.00 312.00"
          },
          {
            "from": "B",
            "to": "J",
            "portA": 0,
            "portB": 1.5707963267948966,
            "shape": "L",
            "pts": [
              {
                "x": 192,
                "y": 486
              },
              {
                "x": 408,
                "y": 486
              },
              {
                "x": 408,
                "y": 400
              }
            ],
            "d": "M 192.00 486.00 L 397.00 486.00 Q 408.00 486.00 408.00 475.00 L 408.00 400.00"
          },
          {
            "from": "J",
            "to": "O",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "straight",
            "pts": [
              {
                "x": 452,
                "y": 356
              },
              {
                "x": 572,
                "y": 356
              }
            ],
            "d": "M 452.00 356.00 L 572.00 356.00"
          }
        ]
      }
    ]
  }
}
```

