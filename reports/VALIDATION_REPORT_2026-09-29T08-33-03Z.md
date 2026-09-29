# VALIDATION REPORT — Fase 2: geometria del canvas (`src/etl-layout/`)

Generato: 2026-09-29T08:33:03Z (UTC) — branch `feat/etl-layout`

## 1. Esiti

```
npx tsc --noEmit  → PASS, 0 errori
npm run lint      → 0 errori, 14 warning (tutti pre-esistenti, react-refresh/only-export-components;
                    nessuno in src/etl-layout/ né in scripts/extract-golden.mjs)
npm test          → 203/203 test passati, 16 file
npm run build     → riuscita
```

Verifica React/DOM/localStorage
(`grep -rnE 'from "react"|\bwindow\b|\bdocument\b|localStorage' src/etl-layout --include=*.ts`):
**nessun risultato**. Nessun import da `src/etl-layout/` dentro `src/etl-core/`.

### Test per file

| File | Test |
| --- | ---: |
| src/canvas/\_\_tests\_\_/panelPositioning.test.ts | 9 |
| src/canvas/layout/\_\_tests\_\_/canvasBounds.test.ts | 13 |
| src/canvas/layout/\_\_tests\_\_/dropZones.test.ts | 9 |
| src/canvas/layout/\_\_tests\_\_/panelRegistry.test.ts | 10 |
| src/etl-core/\_\_tests\_\_/csv.test.ts | 9 |
| src/etl-core/\_\_tests\_\_/expressions.test.ts | 7 |
| src/etl-core/\_\_tests\_\_/fase11-requisiti.test.ts | 30 |
| src/etl-core/\_\_tests\_\_/fase11.test.ts | 11 |
| src/etl-core/\_\_tests\_\_/mutations.test.ts | 9 |
| src/etl-core/\_\_tests\_\_/params.test.ts | 11 |
| src/etl-core/\_\_tests\_\_/relations.test.ts | 11 |
| src/etl-core/\_\_tests\_\_/schema.test.ts | 5 |
| src/etl-core/\_\_tests\_\_/state.test.ts | 26 |
| **src/etl-layout/\_\_tests\_\_/golden.test.ts** | **15** |
| **src/etl-layout/\_\_tests\_\_/properties.test.ts** | **8** |
| **src/etl-layout/\_\_tests\_\_/unit.test.ts** | **20** |
| **Totale** | **203** |

Rispetto alla Fase 1.1 (157): +3 in etl-core (Passo 0), +43 in etl-layout.

## 2. Passo 0 — correzione in etl-core

- `normalizeValuesField` divide **solo** sul separatore registrato `sep`
  (o `,`): con `sep` `;` il valore "Rossi, Mario" resta intero.
- `splitTokens` (prototipo, riga 2839: `,` `;` `|` a capo, senza
  duplicati) è esportata da etl-core per l'inserimento dal vivo nel
  selettore; la migrazione non la usa.
- Test in `fase11-requisiti.test.ts`: "con sep ';' un valore 'Rossi,
  Mario' resta intero", "con sep '\n' divide solo sugli a capo",
  "ensureKeys: rlist del join, con sep ';'", e due test di `splitTokens`.
  `NOTE_DIVERGENZE.md` di etl-core aggiornato.

## 3. Funzioni portate (righe del prototipo)

| Area | Funzioni | Righe | Modulo |
| --- | --- | --- | --- |
| Nodi e porte | `borderPoint`, `axisOf`, porte `PORTS`, rettangoli di nodo/etichetta/ostacolo | 1018, 1031-1035, 1050-1052, 1318-1319, CSS 626-682 | `nodes.ts` |
| Instradamento | `segHitsRect`, `routeCost`, `buildRoute`, `countBends`, `segCross`, `countCrossings`, `overshoot`, `routeLength`, `slide`, `shapeAnchors`, `orthogonalize`, `removeReversals`, `buildFromShape`, `dirSign`, `backtrack`, `shapeCandidates`, `chooseRoute` | 1056-1279 | `routing.ts` |
| Percorso SVG | `roundedPath` | 1281-1298 | `path.ts` |
| Tutti i cavi | `drawLinks` (porte, scostamenti sulla stessa porta, corsie, percorsi), `shift` | 1300-1411 | `links.ts` |
| Libero | `clampCard`, griglia, `resolveOverlaps`, `overlapsAny`, `freeSpot`, `displace`, `anyOverlap` (`compatiblePair` è già in etl-core) | 1537-1608, 1939-1957, 2630-2638 | `free.ts` |
| Organizzato | `computeSlots`, `slotTaken`, `nearestSlot`, `firstFreeSlot`, `placeInSlots`, `assignSlots`/`assignSlotsKeepingOrder`, scambio al rilascio, `outputSlotFor` | 3910-3951, 2679-2684, 2093-2100, 1709-1727 | `slots.ts` |
| Riordino | `autoLayout` | 4220-4329 | `autoLayout.ts` |
| Nodi generati | `relocateAfter`, posizione in `spawnOutput`, `insertOnLink`, `detachStep`, `performMerge` | 1675-1706, 1738-1746, 1859-1860, 2175-2181, 1826-1834 | `placement.ts` |
| Individuazione | `elementFromPoint` sui nodi, tracciati `.hit` da 16 px | 2013, 3992, 4960, 1402 | `hitTest.ts` |

Non portate (elenco completo con il motivo in `src/etl-layout/README.md`):
`nearestPort` e `stickyPort` (mai usate), `defaultKnob` (mai chiamata),
`easeAngle` e l'animazione dei cavi (Fase 5), `dotFill`, `applyPositions`,
`renderOutputIcon` (DOM/disegno), `handleStageWidthChange` (con pan/zoom
attivo si limita a ridisegnare i cavi), vista e minimappa (`applyView`,
`toWorld`, `zoomAt`, `fitView`, `renderMinimap`).

Costanti: tutte in `src/etl-layout/constants.ts`, identiche al prototipo,
ciascuna con la riga.

## 4. Scenari golden

Generati da `scripts/extract-golden.mjs`, che esegue il prototipo in
Chromium (Playwright 1.63, nuova dipendenza di sviluppo) usando le sue
variabili e funzioni globali. Confronto in `golden.test.ts`, tolleranza
0,5 px; la stringa SVG viene confrontata identica, le porte a 1e-9.
"pN" = passate fino a regime.

| Scenario | Tipo | Esito del prototipo | Parità |
| --- | --- | --- | --- |
| 01-dritto-allineati | cavi | p2, dritto | ✅ |
| 02-dritto-scorrimento (20 px) | cavi | p2, dritto con scorrimento delle porte | ✅ |
| 03-oltre-scorrimento (130 px) | cavi | p2, a L | ✅ |
| 04-ostacolo | cavi | p2, a Z che aggira il nodo | ✅ |
| 05-incrocio | cavi | p2, due Z su corsie distinte (snodi a 14 px) | ✅ |
| 06-corsie (snodi ammessi 2) | cavi | p8 (limite, non converge), tre Z | ✅ |
| 07-join-output-parziale | cavi | p2, L/L/dritto | ✅ |
| 08-spostamento | cavi | piccolo spostamento: stesse porte e forma; grande: porte cambiate | ✅ |
| 09-catena-riordino | riordino | prima p8 L/L/L/Z/Z/Z/Z/L → posizioni → dopo p2 quasi tutti dritti | ✅ |
| 10-riordino-isolati | riordino | colonna di parcheggio a destra, con separazione | ✅ |
| 10b-riordino-colonna-fitta (stage 636 px) | riordino | righe alla distanza minima (128 px) | ✅ |
| 11-organizzato | postazioni | assegnazione A:0 B:1 C:21 D:20 E:43; scambio E↔B; A in postazione libera 105 | ✅ |
| 12-output-generato (Libero) | nodo generato | output spostato da `freeSpot` a (634, 312) | ✅ |
| 13-output-organizzato | nodo generato | output nella postazione 44 | ✅ |

## 5. Test di proprietà (40 grafi casuali, generatore deterministico)

- ogni segmento orizzontale o verticale (limite di snodi 0, 1 e 2);
- nessuna inversione di direzione né punto superfluo;
- un percorso dritto resta un solo segmento (o uno scalino sotto 1,5 px,
  vedi divergenza 1);
- `chooseRoute`, e un cavo isolato a regime, non attraversano nodi quando
  esiste un'alternativa;
- il limite di snodi è rispettato quando esiste un'alternativa che non
  attraversa nodi e non ripiega (vedi divergenza 2);
- il riordino è deterministico, non modifica l'input e mette i nodi
  collegati in colonne crescenti.

## 6. Verifica di copertura delle costanti

Ogni costante di `constants.ts` è stata alterata di +1 (i pesi di `SCORE`
di ×1,5 + 1) e i test di etl-layout rieseguiti. Fanno fallire almeno un
test: `CARD`, `LABEL_H`, `WORLD_MARGIN`, `STUB`, `ELBOW_R`, `MAX_BENDS`,
`OBST_PAD`, `KNOB_OBSTACLE_MARGIN`, `KNOB_STEP`, `KNOB_STEPS`,
`SCORE.overBends`, `PORT_SPREAD`, `LANE_GAP`, `GRID`, `OVERLAP_PAD`,
`OVERLAP_MARGIN`, `FREE_SPOT_STEP_X`, `SLOT_W`, `SLOT_H`, `SLOT_M`,
`AUTO_COL`, `AUTO_ROW_MIN`, `AUTO_ROW_MAX`, `AUTO_X0_MIN`,
`DETACH_OFFSET_Y`, `RELOCATE_STEP_X`. Dopo la verifica sono stati aggiunti
test con valori letterali per `LABEL_GAP`, `LABEL_MAX_W`, `WORLD_W`,
`WORLD_H`, `LINK_HIT_WIDTH`, `DISPLACE_BOTTOM`.

Non rilevate da un +1: le soglie e i margini che gli scenari non
raggiungono (`SELF_OBSTACLE_INSET`, `PORT_SLACK_INSET`, `STRAIGHT_EPS`,
`EPS`, `CROSS_*`, `LANE_NEAR`, `OVERLAP_ITERATIONS`, `FREE_SPOT_RADIUS`,
`FREE_SPOT_STEP_Y`, `AUTO_AVAIL_MARGIN`, `AUTO_Y0_MIN`, i numeri di
passate del riordino, `RELOCATE_STEP_Y`, `RELOCATE_TRIES`,
`OUTPUT_SLOT_DY_WEIGHT`); quelle mascherate dall'aggancio alla griglia
(`OUTPUT_OFFSET_X`, `OUTPUT_EDGE_MARGIN`, `DISPLACE_PUSH`); i pesi di
`SCORE` alterati in proporzione, che lasciano invariate le scelte
(alterazioni forti, come `crossing: 1` o `portChange: 0`, fanno fallire i
golden). I valori restano quelli del prototipo, verificati riga per riga.

## 7. Divergenze

Nessuna correzione al prototipo. In `src/etl-layout/NOTE_DIVERGENZE.md`:

1. un dritto con agganci disallineati di meno di 1,5 px lascia uno scalino;
2. i pesi di `chooseRoute` sono sommati, non una gerarchia stretta;
3. la stabilità guarda solo le porte, non la forma;
4. `displace` non spinge due nodi con centri coincidenti;
5. `displace` usa un limite inferiore di 2 px diverso da `clampCard`;
6. il punto predefinito di un passaggio sganciato tocca sempre il box;
7. lo scambio con un nodo senza postazione lascia l'altro senza postazione;
8. la distanza minima tra le righe del riordino è quasi sempre invisibile.

Scelte necessarie: cavi calcolati a regime (senza animazioni), mondo e area
visibile come parametri, rettangolo dell'etichetta dalla larghezza massima
CSS, individuazione geometrica al posto del DOM.

## 8. Altri file toccati

- `package.json`, `package-lock.json`, `bun.lock`: Playwright come
  dipendenza di sviluppo (autorizzata). `bun.lock` era già indietro
  rispetto a `package.json` (mancava vitest): la sincronizzazione lo
  riallinea.
- `scripts/extract-golden.mjs`: nuovo, genera i golden.
- `scripts/generate-snapshot.mjs`: nuova area `01c-etl-layout`, perché
  la cartella non finisca in "13-misc".
