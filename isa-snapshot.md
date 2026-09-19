# ISA ETL Snapshot

Generated: 2026-09-19T11:39:13Z

## Index
- src/canvas/.reports/VALIDATION_REPORT_2026-09-19T10-24-53Z.md
- src/canvas/.reports/VALIDATION_REPORT_2026-09-19T11-38-16Z.md
- src/canvas/FUNCTIONAL_CHECKS.md
- src/canvas/README.md
- src/canvas/__tests__/panelPositioning.test.ts
- src/canvas/components/CanvasContainer.tsx
- src/canvas/hooks/useCanvasBounds.ts
- src/canvas/hooks/usePanelState.ts
- src/canvas/layout/__tests__/canvasBounds.test.ts
- src/canvas/layout/__tests__/dropZones.test.ts
- src/canvas/layout/__tests__/panelRegistry.test.ts
- src/canvas/layout/canvasBounds.ts
- src/canvas/layout/dropZones.ts
- src/canvas/layout/panelRegistry.ts
- src/canvas/layout/surfacePanels.ts
- src/canvas/store/canvasStore.tsx
- src/components/isa/etl/data-preview.tsx
- src/components/isa/etl/inspector.tsx
- src/components/isa/etl/isa-context-menu.tsx
- src/components/isa/etl/settings-panels/aggregate-panel.tsx
- src/components/isa/etl/settings-panels/combine-panel.tsx
- src/components/isa/etl/settings-panels/filter-panel.tsx
- src/components/isa/etl/settings-panels/panel-controls.tsx
- src/components/isa/etl/tool-palette.tsx
- src/components/isa/ui/isa-menu.tsx
- src/lib/etl-bubble.ts
- src/lib/etl-catalog.ts
- src/lib/etl-display.ts
- src/lib/etl-motion.ts
- src/lib/etl-node-config.ts
- src/lib/etl-node-size.ts
- src/lib/etl-schema.ts
- src/lib/etl-workflow.tsx


=== FILE: src/canvas/.reports/VALIDATION_REPORT_2026-09-19T10-24-53Z.md ===
# VALIDATION REPORT — Canvas Edge System (Fase 1)

**Timestamp:** 2026-09-19T10:24:53Z
**Scope:** `src/canvas/layout/`, `src/canvas/store/`, `src/canvas/hooks/`, `src/canvas/components/`

## 1. Linting & type check

```
npx tsc --noEmit                          → PASS, 0 errori
npx eslint src/canvas vitest.config.ts    → PASS, 0 errori
```

5 warning `react-refresh/only-export-components`, tutti su file che
esportano sia un componente React sia hook/tipi dallo stesso modulo
(`canvasStore.tsx`, `CanvasContainer.tsx`). Confermato che è un pattern
già presente e tollerato nella repo: `src/lib/theme.tsx` e
`src/lib/solutions-store.tsx` producono lo stesso identico warning.
Nessun errore, nessun `console.error` introdotto.

## 2. Unit test

```
npm test -- src/canvas/layout/__tests__
```

**32/32 test passati**, 3 file:

| File | Test | Esito |
|---|---|---|
| `canvasBounds.test.ts` | 12 | ✅ tutti passati |
| `dropZones.test.ts` | 10 | ✅ tutti passati |
| `panelRegistry.test.ts` | 10 | ✅ tutti passati |

Coperto: bounds invariati senza pannelli aperti; riduzione bounds per
pannelli "stretch" su ciascun lato; nessuna riduzione per pannelli
"center"; somma corretta di più pannelli aperti; nessuna larghezza/
altezza negativa quando un pannello eccede il contenitore; conversione
bounds→rect; posizionamento pannello stretch/center; clamp e detection
di card fuori bounds; drop-zone con/senza ostacoli; validità di un punto
di drop dentro/fuori dal bounding box della palette e dentro/fuori dai
bounds; registry↔store (stato iniziale, PANEL_OPENED/CLOSED/TOGGLED/
RESIZED, azione su id non registrato ignorata).

_Nota: nel package.json del progetto non esisteva alcun test runner
(`npm test` non era definito). Aggiunto `vitest` come devDependency e
`vitest.config.ts` (con `vite-tsconfig-paths` + `@vitejs/plugin-react`,
già presenti come dipendenze) — nessun altro cambiamento alla toolchain
esistente._

## 3. Consistency check

- Naming: `canvasBounds.ts`, `panelRegistry.ts`, `dropZones.ts`,
  `canvasStore.tsx`, `useCanvasBounds.ts`, `usePanelState.ts`,
  `CanvasContainer.tsx` — coerenti con quanto richiesto.
- Import circolari: **nessuno** (verificato con `npx madge --circular
  --extensions ts,tsx src/canvas` → "No circular dependency found!").
- Ogni funzione esportata ha una singola responsabilità:
  `calculateCanvasBounds` (solo bounds), `computePanelRect` (solo
  geometria di un pannello), `calculateDropZones`/`isValidDropPoint`
  (solo validità drop), `canvasReducer`/`toPanelInstances` (solo stato).
  Nessuna funzione mischia calcolo geometrico e side-effect.
- Separazione netta rispettata: `layout/` è puro TypeScript senza
  React; `store/` è l'unico punto con `useReducer`/Context; `hooks/`
  collega store+DOM (ResizeObserver); `components/` è l'unico punto con
  markup.

## 4. Functional verification (manuale)

Vedi `src/canvas/FUNCTIONAL_CHECKS.md` per il dettaglio. Riassunto:
tutti gli scenari eseguibili in questa fase (senza UI collegata) sono
stati verificati con uno script mirato — bounds che cambiano
correttamente per 1, 2 e 3 pannelli aperti contemporaneamente, drop
respinto dentro il bounding box reale della palette, drop accettato
accanto ad essa (fix del bug 1.2), drop respinto fuori dai bounds
ridotti dall'Inspector. I check che richiedono un componente reale
montato in UI sono documentati come protocollo per la fase 2.

## 5. Cosa è stato fatto

Costruita la struttura richiesta sotto `src/canvas/`:

- **`layout/canvasBounds.ts`** — `calculateCanvasBounds()` (pura,
  ricalcolabile ad ogni cambiamento di container o pannelli),
  `computePanelRect()`, `clampRectToBounds()`, `isRectOutOfBounds()`,
  `boundsToRect()`.
- **`layout/panelRegistry.ts`** — `PANEL_DEFINITIONS`: Tool Palette
  (top, anchor "center"), Inspector (right, anchor "stretch"), Data
  Preview (bottom, anchor "stretch"). Unico punto da toccare per
  registrare un nuovo pannello.
- **`layout/dropZones.ts`** — `calculateDropZones()` (bounds + lista di
  ostacoli = bounding box reali dei pannelli "center" aperti),
  `isValidDropPoint()`.
- **`store/canvasStore.tsx`** — stato open/closed/size per pannello,
  come Context + `useReducer` (stesso pattern già in uso in
  `src/lib/theme.tsx` e `src/lib/solutions-store.tsx`, non Zustand/Redux:
  vedi motivazione nel file). Reducer puro esportato e testato senza
  React.
- **`hooks/useCanvasBounds.ts`** — misura il contenitore con
  `ResizeObserver`, legge lo store, ricalcola bounds/drop-zone con
  `useMemo`, si ricalcola automaticamente ad ogni cambio di container O
  di pannelli.
- **`hooks/usePanelState.ts`** — stato + azioni (`openPanel`,
  `closePanel`, `togglePanel`, `setSize`) per UN pannello, per i
  componenti che possiedono il trigger di apertura.
- **`components/CanvasContainer.tsx`** — il contenitore reattivo:
  misura sé stesso, espone bounds/drop-zone ai figli via context
  (`useCanvasBoundsContext`), e chiama un `onBoundsChange` opzionale
  quando i bounds cambiano davvero (confronto per valore, non per
  riferimento) — il punto di aggancio per la fase 2 (riposizionamento
  card).

Introdotto anche il test runner (`vitest`) mancante dal progetto, con
`vitest.config.ts` minimale.

## 6. Cosa NON è stato fatto (deliberatamente) — feedback architetturale

Come richiesto dalle note del contesto strategico ("se scopri che devi
riscrivere parti importanti del canvas oggi, fermati e comunicalo"),
segnalo questo prima di procedere oltre:

**Il nuovo layer non è ancora collegato a `workflow-canvas.tsx`.**
Motivo: quel componente (5000+ righe) gestisce oggi la geometria delle
card in un sistema di coordinate "superficie" scalato da `zoom`
(`surfaceW`/`surfaceH`), mentre Inspector e Data Preview — renderizzati
un livello sopra, in `solutions.$solutionId.etl.tsx` — sono posizionati
in coordinate schermo assolute, indipendenti dallo zoom. `placeNode()` e
il calcolo di `paletteRect` dentro `workflow-canvas.tsx` già fanno
correttamente ciò che PARTE 2.3 chiede per la sola Tool Palette (bounding
box reale, non fascia intera) — ma solo per lei, perché è l'unico
pannello che vive nello stesso sistema di coordinate delle card.

Estendere `calculateCanvasBounds` al clamp reale delle card per Inspector
e Data Preview richiede prima una decisione: portare quei due pannelli
nel sistema di coordinate "superficie" (cambiando come sono renderizzati
— fuori scope dichiarato per questa fase, "non toccare il rendering
visivo") oppure convertire i loro bounds in coordinate superficie al
volo (dividendo per `zoom` — fattibile, ma è logica che oggi vive solo
dentro `workflow-canvas.tsx`, quindi richiede comunque di toccare quel
file). Ho scelto di non prendere questa decisione da solo in una fase
esplicitamente delimitata a "coordinate e geometria, non UI, non drag
interattivo": la fondazione qui sopra è già corretta e generale per
qualunque sistema di coordinate la si alimenti, il collegamento è
lavoro — e una decisione — di fase 2.

## 7. Checklist finale

- [x] Tutti i test passano (32/32)
- [x] Nessun errore TypeScript
- [x] Nessun errore linting
- [x] La palette (nel nuovo layer) blocca solo il proprio bounding box
      reale, non l'intera fascia — verificato via `dropZones.test.ts` e
      script manuale
- [x] `calculateCanvasBounds()` cambia quando i pannelli si aprono/chiudono
      — verificato via `canvasBounds.test.ts`
- [ ] Le card si riadattano quando i bounds cambiano — **non verificabile
      in questa fase**: richiede l'integrazione con `workflow-canvas.tsx`
      discussa al punto 6. `clampRectToBounds`/`isRectOutOfBounds` sono
      pronte e testate per quando quel collegamento verrà fatto.
- [x] Nessun `console.error` o warning non commentato (i 5 warning
      residui sono lint style, pattern preesistente nella repo)
- [x] README aggiornato (`src/canvas/README.md`) con come usare
      `useCanvasBounds`/`usePanelState`/`CanvasContainer`

## 8. Sync snapshot

`scripts/sync-snapshot.sh` aggiornato per includere `src/canvas/**/*.ts(x)`
e `src/canvas/**/*.md` (README, FUNCTIONAL_CHECKS, questo report) nel
bucket "main" esistente — nessun nuovo file fisso creato, coerente con la
policy dello script ("redistribute across these SAME files").

Eseguito con successo alle **2026-09-19T10:29:01Z**. Nota tecnica: il
token `GITHUB_TOKEN` di Codespaces (attivo di default) non ha accesso al
repo `isa-etl-snapshot` (403) — usato invece l'account OAuth con scope
`repo` già presente in `gh auth status` (`env -u GITHUB_TOKEN
./scripts/sync-snapshot.sh`). Nessuna modifica permanente alla
configurazione gh: solo un override di environment per l'invocazione.

Raw URL aggiornati:
- https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/main/isa-snapshot-workflow-canvas.md
- https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/main/isa-snapshot.md


=== FILE: src/canvas/.reports/VALIDATION_REPORT_2026-09-19T11-38-16Z.md ===
# Validation Report — Fase 2A (Canvas Integration, Option A)

**Date:** 2026-09-19T11:38:16Z
**Approach:** Inspector/Data Preview nel sistema Superficie ("pannello controscalato")

## Deviazioni dal prompt di partenza (dichiarate subito, come richiesto)

Il prompt di Fase 2A conteneva pseudocodice che assumeva una struttura
diversa da quella reale del repo. Ho seguito l'INTENTO (Option A) ma
adattato l'implementazione ai file reali, per rispettare la nota
esplicita "Non toccare Fase 1: canvasBounds.ts e dropZones.ts rimangono
come sono":

1. **`CanvasBoundsContext` non vive in `canvasStore.tsx`.** Nel repo
   reale vive in `src/canvas/components/CanvasContainer.tsx` (Fase 1).
   `canvasStore.tsx` gestisce SOLO lo stato open/closed/size dei pannelli
   — `zoom` non c'entra con quello stato, quindi non l'ho toccato. Ho
   aggiunto `zoom` al context di `CanvasContainer.tsx`.
2. **`computePanelRect` non è stato modificato.** Il prompt proponeva una
   firma diversa (`computePanelRect(bounds, side, width, mode)`), incompatibile
   con quella esistente e testata (32 test Fase 1). L'ho riusata AS-IS
   componendola in un nuovo file, `src/canvas/layout/surfacePanels.ts`
   (`computeSurfacePanelRect`), che non tocca `canvasBounds.ts`/`dropZones.ts`.
3. **`clampRectToBounds` ritorna un `Point` (x/y), non un `Rect` completo**
   — il prompt assumeva `clamped.width`/`clamped.height`. Gestito
   correttamente in `computeSurfacePanelRect` (width/height vengono dal
   rect calcolato, non dal clamp).
4. **Nessun `ref` esterno su `CanvasContainer`.** Il prompt passava
   `ref={canvasRef}` al componente. Nella realtà, `workflow-canvas.tsx`
   possiede già `surfaceRef`/`boxRef` propri; ho esteso `CanvasContainer`
   con un prop `containerSize` che, se fornito, salta la misura interna
   (nessun wrapper DOM aggiunto) — così il chiamante passa le dimensioni
   che già calcola (`surfaceW`/`surfaceH`), senza duplicare ResizeObserver
   né introdurre un nodo DOM superfluo nell'albero esistente.
5. **Nessun `pan`.** Il canvas oggi non ha stato di pan (solo `zoom`,
   origine fissa in alto a sinistra) — non citato nel prompt come
   assunzione, verificato leggendo il codice.
6. **Auto-esclusione dei pannelli.** Il prompt non affrontava un problema
   reale: un pannello che legge `bounds` calcolati includendo SE STESSO
   si "farebbe spazio" due volte. Risolto in `computeSurfacePanelRect`
   escludendo il pannello dalla lista prima di calcolare i suoi stessi
   bounds — coperto da test dedicati (`panelPositioning.test.ts`).
7. **`inset` prop di `DataPreview` rimosso**, non solo esteso: era
   l'esatto hack ad-hoc (larghezza magica `21.5rem`) che questo sistema
   sostituisce con bounds reali condivisi via canvasStore. Tenerlo
   avrebbe significato duplicare la stessa informazione in due posti.

## Test Results

```
npm test -- src/canvas          → 41/41 passed (32 Fase 1 + 9 nuovi)
npx tsc --noEmit                → PASS, 0 errori
npx madge --circular ...        → nessun import circolare
```

Nuovo file: `src/canvas/__tests__/panelPositioning.test.ts` (9 test):
posizionamento di un pannello stretch da solo; auto-esclusione (un
pannello non "fa spazio" a se stesso, anche con uno stato stantio nello
store); conversione physical↔surface units con zoom ≠ 1; un pannello
bottom-stretch evita un pannello right-stretch aperto (Inspector + Data
Preview); l'auto-esclusione funziona anche per il pannello bottom;
clamp quando la dimensione fisica eccede il container; `surfacePanelStyle`
lascia `left`/`top` invariati e scala `width`/`height` per `zoom`;
round-trip a zoom 0.5/1/1.8 → la dimensione fisica torna sempre la stessa.

## Linting

```
npx eslint src/canvas src/components/isa/etl/inspector.tsx \
  src/components/isa/etl/data-preview.tsx \
  src/routes/solutions.\$solutionId.etl.tsx
```

**0 errori nei file di mia proprietà** (`src/canvas/**`, `inspector.tsx`,
`data-preview.tsx`), solo i 5 warning `react-refresh/only-export-components`
già noti e tollerati da Fase 1 (stesso pattern di `theme.tsx`/
`solutions-store.tsx`).

`workflow-canvas.tsx` e `solutions.$solutionId.etl.tsx` hanno errori
prettier PREESISTENTI (non introdotti ora): verificato confrontando con
`git show HEAD:...` (342 errori già a HEAD, prima ancora delle modifiche
non commesse di questa sessione) — il file ha uno stile "un token per
riga" che diverge sistematicamente dalla config Prettier del progetto,
tollerato dal repo da prima di questo lavoro.

**Delta onestamente introdotto da me:** ho scelto di NON reindentare
l'intero `<section>` (1400+ righe) per assorbire i due nuovi livelli di
nesting (`CanvasStoreProvider` → `CanvasContainer`) — avrebbe prodotto un
diff enorme e illeggibile su codice non mio. Ho invece annidato i nuovi
wrapper "a piatto" (stesso livello di indentazione del `<section>`
originale). Risultato: ~20 righe con un mismatch di indentazione
puramente cosmetico rispetto a quanto Prettier vorrebbe, isolate ai 3
punti che ho toccato (apertura, punto di montaggio `{children}`,
chiusura). Nessun impatto funzionale — confermato da tsc pulito e dalla
verifica visiva nel browser.

## Functional Verification (reale, nel browser — non solo documentata)

A differenza di Fase 1 (dove l'integrazione non esisteva ancora e i
check erano solo protocollo), qui il sistema è collegato: ho avviato
`npm run dev`, creato una soluzione di test via UI, e guidato Chromium
headless (Playwright, browser già in cache in questo devcontainer) su
`/solutions/:id/etl`. Screenshot allegati in
`/tmp/.../scratchpad/0{2..7}-*.png` (sessione locale, non nel repo).

| Check | Esito |
|---|---|
| Canvas si carica, empty state corretto | ✅ |
| Apertura Inspector (right, stretch) | ✅ posizionato correttamente, nessun overlap con le card |
| Apertura anche Data Preview (bottom, stretch) mentre Inspector è aperto | ✅ **Data Preview si ferma esattamente dove inizia l'Inspector — zero overlap, senza alcun prop `inset` cablato a mano** |
| Zoom in (100% → 140%) con entrambi aperti | ✅ la card "Dataset" scala visibilmente; Inspector e Data Preview restano IDENTICI in dimensione fisica e posizione (controscale confermato visivamente) |
| Zoom out (→ 60%) | ✅ stesso comportamento, card rimpicciolita, pannelli invariati |
| Chiusura di entrambi i pannelli | ✅ nessun artefatto residuo, canvas torna allo stato pulito |
| Errori console durante l'intera sessione | **0** (`page.on("console")`/`page.on("pageerror")` non hanno registrato nulla) |

**Non verificato in questa fase (dichiarato, non taciuto):**
- Drag di un nodo con Inspector/Data Preview aperti — le card non evitano
  ancora questi pannelli (vedi "Non ancora in scope" nel README).
- Auto-layout attorno ai pannelli — stesso motivo.
- Pan del canvas — non esiste ancora come feature (solo zoom).

Questi tre punti erano nella checklist del prompt originale ma
richiedono di toccare `placeNode`/l'auto-layout in `workflow-canvas.tsx`
e `etl-workflow.tsx` — esplicitamente fuori scope per Fase 2A (limitata a
Inspector/Data Preview) tanto quanto lo era per Fase 1.

## Files Modified

- `src/canvas/layout/surfacePanels.ts` (nuovo) — `computeSurfacePanelRect`, `surfacePanelStyle`
- `src/canvas/hooks/useCanvasBounds.ts` — supporto `externalSize`, espone `panels`
- `src/canvas/components/CanvasContainer.tsx` — prop `zoom`/`containerSize`, context esteso
- `src/canvas/__tests__/panelPositioning.test.ts` (nuovo, 9 test)
- `src/components/isa/etl/workflow-canvas.tsx` — prop `children`, wrap `CanvasStoreProvider`/`CanvasContainer`, monta `{children}` dentro `surfaceRef`
- `src/components/isa/etl/inspector.tsx` — posizionamento via bounds condivisi, rimosso `top-14/right-3/bottom-3` hardcoded
- `src/components/isa/etl/data-preview.tsx` — posizionamento via bounds condivisi, rimosso prop `inset`
- `src/routes/solutions.$solutionId.etl.tsx` — Inspector/DataPreview spostati da sibling a children di `<WorkflowCanvas>`
- `src/canvas/README.md` — sezione "Fase 2A" aggiunta

**Non toccati (Fase 1, come richiesto):** `src/canvas/layout/canvasBounds.ts`, `src/canvas/layout/dropZones.ts`, `src/canvas/layout/panelRegistry.ts`, `src/canvas/store/canvasStore.tsx`, `src/canvas/hooks/usePanelState.ts`.

## Notes & Trade-offs

- Inspector/Data Preview "appartengono" ora logicamente al canvas
  (coordinate superficie), coerente con l'intento del prompt.
- `transform: scale(1/zoom)` mantiene le dimensioni fisiche costanti —
  confermato visivamente a 60/100/140%.
- Rendering resta DOM, non SVG.
- L'assenza di overlap Inspector↔Data Preview non è un CSS hardcoded ma
  una conseguenza diretta dei bounds condivisi in canvasStore — se in
  futuro l'Inspector cambia larghezza, Data Preview si adatta da sola.
- Le insets estetiche (margini interni: 56px sopra l'Inspector per i
  controlli zoom, 12px sugli altri lati) sono costanti locali nei due
  componenti, non parte del sistema di bounds condiviso — sono pura resa
  visiva, non geometria che altri pannelli devono conoscere.

## Snapshot

`./scripts/sync-snapshot.sh` (con `unset GITHUB_TOKEN` già presente
nello script, aggiunto dall'utente) eseguito con successo dopo questo
report — vedi conferma nel messaggio di chat.


=== FILE: src/canvas/FUNCTIONAL_CHECKS.md ===
# Functional Checks — Canvas Edge System (Fase 1)

Stato attuale: questo layer (`src/canvas/`) è verificato con 32 unit test
automatici (geometria pura) più uno script manuale mirato (sotto). Non è
ancora collegato a `workflow-canvas.tsx` — vedi "Stato dell'integrazione"
in `src/canvas/README.md` per il perché. Di conseguenza non esiste oggi
un percorso in UI (click su un bottone reale) per riprodurre questi
check: quelli elencati in "Dopo l'integrazione" sono pronti da eseguire
non appena un componente monta `CanvasStoreProvider` + `CanvasContainer`.

## Eseguiti ora (senza UI, sulla logica pura)

Script eseguito con `npx tsx`, valori realistici presi dalle dimensioni
di palette/inspector/preview osservate in `workflow-canvas.tsx` /
`inspector.tsx` / `data-preview.tsx`:

| Scenario | Input | Risultato | Atteso |
|---|---|---|---|
| Solo Tool Palette aperta (top, center) | container 1440×820, palette 480×52 | bounds invariati (1440×820) | ✅ la palette non deve bloccare la fascia |
| + Inspector aperto (right, stretch, 320px) | come sopra | `right` passa a 1120, width 1120 | ✅ |
| + Data Preview aperta (bottom, stretch, 260px) | come sopra | `bottom` passa a 560, height 560 | ✅ tre pannelli si sommano correttamente |
| Drop nel bounding box reale della palette | punto (720, 20) | rifiutato | ✅ 3.1: non si crea una card sotto la palette |
| Drop appena a sinistra della palette, stessa fascia orizzontale | punto (400, 20) | accettato | ✅ 2.3: lo spazio accanto alla palette resta usabile |
| Drop nella striscia riservata all'Inspector (fuori bounds) | punto (1200, 300) | rifiutato | ✅ |
| Drop in area libera del canvas | punto (200, 300) | accettato | ✅ |

Ripetibile con:
```bash
npx tsx -e "$(cat <<'EOF'
import { calculateCanvasBounds } from './src/canvas/layout/canvasBounds.ts';
import { calculateDropZones, isValidDropPoint } from './src/canvas/layout/dropZones.ts';
// ... vedi src/canvas/.reports per lo script completo usato
EOF
)"
```
(o più semplicemente: `npm test -- src/canvas/layout/__tests__`, che copre
gli stessi scenari come asserzioni automatiche.)

## Dopo l'integrazione (da eseguire quando un componente reale monta CanvasContainer)

1. **`calculateCanvasBounds()` cambia con l'apertura/chiusura di un pannello.**
   - Avvia l'app (`npm run dev`), apri la pagina che monta `CanvasContainer`.
   - Apri l'Inspector (o il pannello "stretch" collegato): verifica via
     `useCanvasBoundsContext()` (es. loggato temporaneamente, o con
     React DevTools sul context) che `bounds.right`/`bounds.width`
     diminuiscano esattamente della larghezza misurata del pannello.
   - Chiudilo: i bounds devono tornare esattamente ai valori precedenti
     (nessuna deriva cumulativa).

2. **Drop zones per drag-and-drop dalla palette.**
   - Trascina una risorsa dalla Tool Palette verso un punto coperto dal
     suo stesso bounding box: il drop deve essere respinto
     (`isValidDropPoint` → false).
   - Trascina verso un punto libero accanto alla palette (stesso lato,
     ma fuori dal suo rettangolo): il drop deve essere accettato anche
     se è "nella stessa fascia" — questo è il fix del bug 1.2 (prima
     l'intera fascia era bloccata).

3. **Anti-sovrapposizione quando i bounds cambiano.**
   - Posiziona una card vicino al bordo destro del canvas.
   - Apri l'Inspector (che si aggancia a destra): la card ora cade sotto
     `bounds.right` → `isRectOutOfBounds` deve restituire `true`.
   - Il componente che possiede i nodi (fase 2, non ancora collegato)
     dovrebbe reagire all'`onBoundsChange` di `CanvasContainer` chiamando
     `clampRectToBounds` sulle card fuori bordo e animando la transizione
     via CSS, non con uno scatto istantaneo.

## Esito

Tutti i check "Eseguiti ora" sono passati (vedi tabella). I check "Dopo
l'integrazione" sono documentati come protocollo ma non eseguibili in
questa fase perché — per scelta architetturale motivata in
`src/canvas/README.md` — l'integrazione con `workflow-canvas.tsx` è
rimandata alla fase successiva.


=== FILE: src/canvas/README.md ===
# Canvas Edge System

Fondamenta geometriche del Canvas: calcolo di quanto spazio ha davvero a
disposizione una volta sottratti i pannelli ausiliari (Tool Palette,
Inspector, Data Preview) che possono aprirsi/chiudersi ai suoi bordi.

Fase 2A ha collegato Inspector e Data Preview a questo layer: vivono ora
DENTRO la superficie zoomata di `workflow-canvas.tsx`, non toccano il drag
delle card. Vedi "Fase 2A — Inspector/Data Preview nella superficie" più
sotto.

## Struttura

```
src/canvas/
├── layout/
│   ├── canvasBounds.ts     # calculateCanvasBounds(), computePanelRect(), clampRectToBounds()
│   ├── panelRegistry.ts    # PANEL_DEFINITIONS: l'unico posto dove annunciare un nuovo pannello
│   ├── dropZones.ts        # calculateDropZones(), isValidDropPoint()
│   ├── surfacePanels.ts    # Fase 2A: geometria dei pannelli "controscalati" dentro la superficie
│   └── __tests__/
├── store/
│   └── canvasStore.tsx     # CanvasStoreProvider + useCanvasStore(): stato open/closed/size dei pannelli
├── hooks/
│   ├── useCanvasBounds.ts  # ascolta resize del contenitore (o dimensioni esterne) + store
│   └── usePanelState.ts    # stato + azioni (open/close/toggle/resize) di UN pannello
├── components/
│   └── CanvasContainer.tsx # il contenitore reattivo: bounds + zoom via context
└── __tests__/
    └── panelPositioning.test.ts  # Fase 2A: geometria dei pannelli controscalati
```

## Concetti chiave

- **Pannello "stretch"** (Inspector, Data Preview): occupa l'intera
  striscia lungo il proprio lato. Da aperto, riduce lo spazio disponibile
  del canvas su quel lato — `calculateCanvasBounds` lo sottrae dal
  rettangolo esterno.
- **Pannello "center"** (Tool Palette): galleggia centrato sul proprio
  lato. Da aperto NON riduce lo spazio disponibile (le card possono
  stare sopra/sotto/accanto ad esso) — ma il suo bounding box reale va
  escluso dalle drop zone, non trattato come una fascia a tutta
  larghezza/altezza (bug 1.2 del contesto strategico). Questo è il
  compito di `calculateDropZones`, non di `calculateCanvasBounds`.

## Come usarlo

```tsx
import { CanvasStoreProvider } from "@/canvas/store/canvasStore";
import { CanvasContainer, useCanvasBoundsContext } from "@/canvas/components/CanvasContainer";
import { usePanelState } from "@/canvas/hooks/usePanelState";

function Workspace() {
  return (
    <CanvasStoreProvider>
      <CanvasContainer className="relative flex-1" onBoundsChange={(bounds) => {
        // punto di aggancio per riposizionare le card che sconfinano (PARTE 2.2)
      }}>
        <CanvasSurface />
      </CanvasContainer>
    </CanvasStoreProvider>
  );
}

function CanvasSurface() {
  const { bounds, obstacles } = useCanvasBoundsContext();
  // bounds cambia automaticamente quando un pannello si apre/chiude o
  // cambia dimensione — nessun ricalcolo manuale necessario qui.
}

function InspectorToggleButton() {
  const inspector = usePanelState("inspector");
  return <button onClick={inspector.togglePanel}>{inspector.open ? "Chiudi" : "Apri"} Inspector</button>;
}
```

`CanvasContainer` accetta anche `containerSize` (Fase 2A: dimensioni già
note al chiamante, in unità superficie — salta il ResizeObserver interno)
e `zoom` (esposto ai figli via context, default 1). Quando `containerSize`
è passato non renderizza un proprio elemento DOM: è un puro Context
provider, per non introdurre un wrapper superfluo dentro un albero DOM
già esistente — vedi come lo usa `workflow-canvas.tsx`.

Un pannello che misura sé stesso (es. la Tool Palette, la cui larghezza
dipende da quante icone mostra) chiama `usePanelState(id).setSize(...)`
dentro un `ResizeObserver`, esattamente come fa oggi `workflow-canvas.tsx`
con `paletteBox` — quella logica di misura non cambia, cambia solo dove
il risultato viene scritto (il canvasStore condiviso invece di uno state
locale).

## Aggiungere un nuovo pannello

1. Aggiungi una riga a `PANEL_DEFINITIONS` in `panelRegistry.ts` (id,
   lato, anchor, closedSize).
2. Nel componente del pannello, usa `usePanelState(id)` per leggere/settare
   `open` e `size`.
3. Non serve toccare `canvasBounds.ts` o `dropZones.ts`: leggono il
   registry a runtime tramite `toPanelInstances`.

## Fase 2A — Inspector/Data Preview nella superficie

Prima di questa fase, Inspector e Data Preview vivevano fuori dalla
superficie zoomata di `workflow-canvas.tsx` (coordinate schermo assolute,
un sistema diverso da quello di card e frecce). Ora sono `children` di
`<WorkflowCanvas>` (passati dal file di rotta), montati DENTRO
`surfaceRef` — lo stesso div con `transform: scale(zoom)` che contiene
le card.

**La tecnica — pannello "controscalato":** l'elemento vive dentro la
superficie scalata ma applica il proprio `transform: scale(1/zoom)`,
annullando lo zoom del genitore per la propria dimensione (resta a
grandezza fisica costante sullo schermo), mentre la sua POSIZIONE
(`left`/`top`) resta in unità superficie e quindi segue naturalmente lo
zoom, restando ancorato al bordo giusto. `src/canvas/layout/surfacePanels.ts`
incapsula le due conversioni di unità (`computeSurfacePanelRect`,
`surfacePanelStyle`), componendo `canvasBounds.ts` senza modificarlo.

**Perché niente overlap tra Inspector e Data Preview:** ogni pannello si
posiziona escludendo SE STESSO dalla lista prima di calcolare i bounds
(altrimenti "farebbe spazio a se stesso" due volte), ma include gli
ALTRI pannelli aperti — così Data Preview (bottom, stretch) vede
automaticamente i bounds già ridotti dall'Inspector (right, stretch) e
non ci si estende sotto.

**Sincronizzazione stato:** il prop `open` di ciascun componente resta la
fonte di verità (posseduta dal file di rotta, invariata); un `useEffect`
lo specchia in canvasStore (`usePanelState(id).openPanel()/closePanel()`)
così gli ALTRI pannelli lo vedono per farsi spazio. Non serve più il
vecchio prop `inset` di `DataPreview` (larghezza magica `21.5rem`
cablata a mano) — rimosso.

**Verificato manualmente nel browser** (screenshot + nessun errore
console) a zoom 60%/100%/140%, con Inspector e Data Preview aperti
insieme: nessuna sovrapposizione, dimensione fisica dei pannelli
invariata al variare dello zoom. Dettagli nel VALIDATION_REPORT di Fase
2A in `.reports/`.

**Non ancora in scope** (Fase 2A si è limitata a Inspector/Data Preview):
le card e l'auto-layout non "evitano" ancora Inspector/Data Preview — il
loro clamp (`placeNode` in `workflow-canvas.tsx`) resta quello di prima,
consapevole solo della Tool Palette. Estendere l'anti-sovrapposizione
delle card a QUESTI pannelli è lavoro di una fase successiva.


=== FILE: src/canvas/__tests__/panelPositioning.test.ts ===
import { describe, expect, it } from "vitest";

import type { PanelInstance } from "../layout/canvasBounds";
import { computeSurfacePanelRect, surfacePanelStyle } from "../layout/surfacePanels";

const CONTAINER = { width: 1000, height: 600 };

function panel(
  overrides: Partial<PanelInstance> & Pick<PanelInstance, "id" | "side">,
): PanelInstance {
  return {
    anchor: "stretch",
    open: false,
    size: { width: 0, height: 0 },
    ...overrides,
  };
}

describe("computeSurfacePanelRect", () => {
  it("positions a lone stretch panel flush to its side, at zoom 1", () => {
    const inspector = panel({
      id: "inspector",
      side: "right",
      open: true,
      size: { width: 0, height: 0 },
    });

    const rect = computeSurfacePanelRect(
      CONTAINER,
      [inspector],
      "inspector",
      { side: "right", anchor: "stretch", physicalSize: { width: 320, height: 0 } },
      1,
    );

    expect(rect.x).toBe(1000 - 320);
    expect(rect.width).toBe(320);
    expect(rect.height).toBe(600);
  });

  it("does not make a panel avoid itself (self-exclusion)", () => {
    // Anche se il panel "inspector" è presente nella lista con una size
    // diversa da quella richiesta ora, il suo bounding box non deve
    // ridurre lo spazio calcolato per se stesso.
    const staleInspector = panel({
      id: "inspector",
      side: "right",
      open: true,
      size: { width: 999, height: 0 },
    });

    const rect = computeSurfacePanelRect(
      CONTAINER,
      [staleInspector],
      "inspector",
      { side: "right", anchor: "stretch", physicalSize: { width: 320, height: 0 } },
      1,
    );

    expect(rect.x).toBe(1000 - 320);
    expect(rect.width).toBe(320);
  });

  it("converts physical size to surface units when zoom != 1", () => {
    const zoom = 2;
    const rect = computeSurfacePanelRect(
      CONTAINER,
      [],
      "inspector",
      { side: "right", anchor: "stretch", physicalSize: { width: 320, height: 0 } },
      zoom,
    );

    // In unità superficie, 320px fisici occupano 320/zoom.
    expect(rect.width).toBeCloseTo(320 / zoom);
    expect(rect.x).toBeCloseTo(1000 - 320 / zoom);
  });

  it("makes a bottom stretch panel avoid an open right stretch panel (Inspector + Data Preview)", () => {
    const inspector = panel({
      id: "inspector",
      side: "right",
      open: true,
      size: { width: 320, height: 0 },
    });

    const dataPreview = panel({
      id: "data-preview",
      side: "bottom",
      open: true,
      size: { width: 0, height: 250 },
    });

    const rect = computeSurfacePanelRect(
      CONTAINER,
      [inspector, dataPreview],
      "data-preview",
      { side: "bottom", anchor: "stretch", physicalSize: { width: 0, height: 250 } },
      1,
    );

    // Non deve estendersi sotto l'Inspector: la sua larghezza si ferma dove inizia l'Inspector.
    expect(rect.x + rect.width).toBeLessThanOrEqual(1000 - 320);
  });

  it("does not let Data Preview's own (stale) entry shrink its own bounds", () => {
    const inspector = panel({
      id: "inspector",
      side: "right",
      open: true,
      size: { width: 320, height: 0 },
    });

    const staleDataPreview = panel({
      id: "data-preview",
      side: "bottom",
      open: true,
      size: { width: 0, height: 9999 },
    });

    const rect = computeSurfacePanelRect(
      CONTAINER,
      [inspector, staleDataPreview],
      "data-preview",
      { side: "bottom", anchor: "stretch", physicalSize: { width: 0, height: 250 } },
      1,
    );

    expect(rect.height).toBe(250);
  });

  it("clamps position to bounds when the physical size would overflow", () => {
    const rect = computeSurfacePanelRect(
      CONTAINER,
      [],
      "inspector",
      { side: "right", anchor: "stretch", physicalSize: { width: 5000, height: 0 } },
      1,
    );

    expect(rect.x).toBeGreaterThanOrEqual(0);
  });
});

describe("surfacePanelStyle", () => {
  it("leaves left/top untouched (already in surface units) and scales width/height by zoom", () => {
    const rect = { x: 100, y: 50, width: 160, height: 300 };
    const style = surfacePanelStyle(rect, 2);

    expect(style.left).toBe(100);
    expect(style.top).toBe(50);
    expect(style.width).toBe(320);
    expect(style.height).toBe(600);
  });

  it("at zoom 1, width/height pass through unchanged", () => {
    const rect = { x: 10, y: 10, width: 320, height: 600 };
    const style = surfacePanelStyle(rect, 1);

    expect(style.width).toBe(320);
    expect(style.height).toBe(600);
  });

  it("round-trips computeSurfacePanelRect's physicalSize back to the same physical pixels regardless of zoom", () => {
    for (const zoom of [0.5, 1, 1.8]) {
      const rect = computeSurfacePanelRect(
        CONTAINER,
        [],
        "inspector",
        { side: "right", anchor: "stretch", physicalSize: { width: 320, height: 0 } },
        zoom,
      );
      const style = surfacePanelStyle(rect, zoom);

      expect(style.width).toBeCloseTo(320);
    }
  });
});


=== FILE: src/canvas/components/CanvasContainer.tsx ===
import { createContext, useContext, useEffect, useRef } from "react";
import type { ReactNode } from "react";

import { useCanvasBounds } from "../hooks/useCanvasBounds";
import type { CanvasBounds, CanvasContainerSize } from "../layout/canvasBounds";
import type { DropZoneMap } from "../layout/dropZones";

type CanvasBoundsContextValue = ReturnType<typeof useCanvasBounds> & {
  /**
   * Fase 2A: livello di zoom della superficie che ospita questo canvas
   * (1 = nessuno zoom). I pannelli che vivono DENTRO la superficie
   * zoomata (vedi src/canvas/layout/surfacePanels.ts) lo usano per
   * restare a dimensione fisica costante sullo schermo via un
   * controscale CSS. Un canvas standalone (non dentro una superficie
   * scalata) può ignorarlo: resta 1 di default.
   */
  zoom: number;
};

const CanvasBoundsContext = createContext<CanvasBoundsContextValue | null>(null);

/** Legge i bounds correnti senza rimisurare: usarlo nei componenti figli di CanvasContainer. */
export function useCanvasBoundsContext(): CanvasBoundsContextValue {
  const ctx = useContext(CanvasBoundsContext);

  if (!ctx) {
    throw new Error("useCanvasBoundsContext must be used inside CanvasContainer");
  }

  return ctx;
}

/**
 * Il contenitore "che respira" (PARTE 1 del contesto strategico): misura
 * se stesso con un ResizeObserver, ascolta canvasStore per i pannelli
 * aperti/chiusi, e ricalcola bounds + drop zone ogni volta che uno dei
 * due cambia — poi li espone ai figli via context, così nessun
 * componente sotto deve rimisurare o richiamare calculateCanvasBounds
 * per conto proprio (PARTE 1.3: "una sorgente di verità per i bordi").
 *
 * `onBoundsChange` è il punto di aggancio per PARTE 2.2 (riposizionare
 * le card che sconfinano quando i bounds cambiano): questo componente
 * NON possiede né tocca lo stato dei nodi — si limita a notificare un
 * cambiamento dei bounds. La logica di riposizionamento vive nel
 * componente che possiede i nodi (workflow-canvas.tsx), da collegare in
 * una fase successiva.
 */
export function CanvasContainer({
  children,
  className,
  zoom = 1,
  containerSize,
  onBoundsChange,
}: {
  children: ReactNode;
  className?: string;
  /** Fase 2A: vedi CanvasBoundsContextValue.zoom sopra. */
  zoom?: number;
  /**
   * Fase 2A: quando fornito, salta la misura via ResizeObserver e usa
   * direttamente queste dimensioni — già note al chiamante in unità
   * superficie (es. workflow-canvas.tsx passa `{ width: surfaceW,
   * height: surfaceH }`). In questo caso il componente non renderizza
   * un proprio elemento DOM: è un puro Context provider, per non
   * inserire un wrapper superfluo dentro un albero DOM già esistente
   * (la superficie scalata di workflow-canvas.tsx, dove `ref` andrebbe
   * comunque sprecato perché le dimensioni arrivano già calcolate).
   */
  containerSize?: CanvasContainerSize;
  onBoundsChange?: (bounds: CanvasBounds, dropZones: DropZoneMap) => void;
}) {
  const containerRef = useRef<HTMLDivElement>(null);
  const boundsState = useCanvasBounds(containerRef, containerSize);
  const previousBoundsRef = useRef<CanvasBounds | null>(null);

  useEffect(() => {
    const previous = previousBoundsRef.current;
    const current = boundsState.bounds;

    const changed =
      !previous ||
      previous.top !== current.top ||
      previous.left !== current.left ||
      previous.right !== current.right ||
      previous.bottom !== current.bottom;

    if (changed) {
      previousBoundsRef.current = current;
      onBoundsChange?.(current, boundsState.dropZones);
    }
  }, [boundsState, onBoundsChange]);

  const value: CanvasBoundsContextValue = { ...boundsState, zoom };
  const content = (
    <CanvasBoundsContext.Provider value={value}>{children}</CanvasBoundsContext.Provider>
  );

  if (containerSize) {
    return content;
  }

  return (
    <div ref={containerRef} className={className} data-canvas-container="">
      {content}
    </div>
  );
}


=== FILE: src/canvas/hooks/useCanvasBounds.ts ===
import { useEffect, useMemo, useState } from "react";
import type { RefObject } from "react";

import type { CanvasContainerSize } from "../layout/canvasBounds";
import { calculateDropZones } from "../layout/dropZones";
import { toPanelInstances, useCanvasStore } from "../store/canvasStore";

/**
 * Ascolta i cambiamenti che possono alterare lo spazio disponibile del
 * canvas — resize del contenitore E apertura/chiusura/resize di un
 * pannello — e ricalcola bounds + drop zone di conseguenza (PARTE 1.3
 * del contesto strategico: "una sorgente di verità per i bordi").
 *
 * `containerRef` è l'elemento la cui area disponibile stiamo misurando
 * (tipicamente il wrapper che contiene canvas + pannelli ausiliari).
 *
 * `externalSize` (Fase 2A): quando il chiamante conosce già le proprie
 * dimensioni in unità superficie (es. `surfaceW`/`surfaceH` di
 * workflow-canvas.tsx, che dipendono da `zoom` e non dal semplice
 * `clientWidth`/`clientHeight` dell'elemento), passarlo qui salta la
 * misura via ResizeObserver — misurare il DOM darebbe le dimensioni
 * SCHERMO sbagliate per un container che deve ragionare in unità
 * superficie.
 */
export function useCanvasBounds(
  containerRef: RefObject<HTMLElement | null>,
  externalSize?: CanvasContainerSize,
) {
  const { state } = useCanvasStore();

  const [measuredContainer, setMeasuredContainer] = useState<CanvasContainerSize>({
    width: 0,
    height: 0,
  });

  const externalWidth = externalSize?.width;
  const externalHeight = externalSize?.height;

  useEffect(() => {
    if (externalWidth !== undefined) {
      return;
    }

    const el = containerRef.current;

    if (!el) {
      return;
    }

    const measure = () => setMeasuredContainer({ width: el.clientWidth, height: el.clientHeight });

    measure();

    const observer = new ResizeObserver(measure);
    observer.observe(el);

    return () => observer.disconnect();
  }, [containerRef, externalWidth]);

  const container: CanvasContainerSize =
    externalWidth !== undefined && externalHeight !== undefined
      ? { width: externalWidth, height: externalHeight }
      : measuredContainer;

  const panels = useMemo(() => toPanelInstances(state), [state]);

  const dropZones = useMemo(
    () => calculateDropZones(container, panels),
    // eslint-disable-next-line react-hooks/exhaustive-deps -- `container` è ricreato ad ogni render con gli stessi valori quando invariato: i suoi due campi primitivi bastano a decidere se ricalcolare.
    [container.width, container.height, panels],
  );

  return {
    container,
    panels,
    bounds: dropZones.bounds,
    obstacles: dropZones.obstacles,
    dropZones,
  };
}


=== FILE: src/canvas/hooks/usePanelState.ts ===
import { useCallback, useMemo } from "react";

import type { PanelSize } from "../layout/canvasBounds";
import { getPanelDefinition } from "../layout/panelRegistry";
import { useCanvasStore } from "../store/canvasStore";

/* Riferimento stabile: evita di ricreare un oggetto ad ogni render quando il pannello non ha ancora uno stato runtime (rompe altrimenti la memoizzazione di useMemo più sotto). */
const ZERO_SIZE: PanelSize = { width: 0, height: 0 };

/**
 * Stato + azioni di un singolo pannello registrato in panelRegistry.ts,
 * per i componenti che possiedono il trigger di apertura/chiusura
 * (es. il bottone "Data preview" nella toolbar) e non hanno bisogno di
 * conoscere l'intero canvasStore.
 */
export function usePanelState(panelId: string) {
  if (!getPanelDefinition(panelId) && process.env["NODE_ENV"] !== "production") {
    console.warn(
      `usePanelState("${panelId}"): nessun pannello con questo id in panelRegistry.ts — le azioni non avranno effetto.`,
    );
  }

  const { state, dispatch } = useCanvasStore();

  const runtime = state.panels[panelId];

  const open = runtime?.open ?? false;
  const size = runtime?.size ?? ZERO_SIZE;

  const openPanel = useCallback(
    () => dispatch({ type: "PANEL_OPENED", id: panelId }),
    [dispatch, panelId],
  );

  const closePanel = useCallback(
    () => dispatch({ type: "PANEL_CLOSED", id: panelId }),
    [dispatch, panelId],
  );

  const togglePanel = useCallback(
    () => dispatch({ type: "PANEL_TOGGLED", id: panelId }),
    [dispatch, panelId],
  );

  const setSize = useCallback(
    (nextSize: PanelSize) => dispatch({ type: "PANEL_RESIZED", id: panelId, size: nextSize }),
    [dispatch, panelId],
  );

  return useMemo(
    () => ({ open, size, openPanel, closePanel, togglePanel, setSize }),
    [open, size, openPanel, closePanel, togglePanel, setSize],
  );
}


=== FILE: src/canvas/layout/__tests__/canvasBounds.test.ts ===
import { describe, expect, it } from "vitest";

import {
  boundsToRect,
  calculateCanvasBounds,
  clampRectToBounds,
  computePanelRect,
  isRectOutOfBounds,
  type PanelInstance,
} from "../canvasBounds";

const CONTAINER = { width: 1200, height: 800 };

function panel(
  overrides: Partial<PanelInstance> & Pick<PanelInstance, "id" | "side">,
): PanelInstance {
  return {
    anchor: "stretch",
    open: false,
    size: { width: 0, height: 0 },
    ...overrides,
  };
}

describe("calculateCanvasBounds", () => {
  it("returns the full container when no panels are open", () => {
    const bounds = calculateCanvasBounds(CONTAINER, []);

    expect(bounds).toEqual({
      top: 0,
      left: 0,
      right: 1200,
      bottom: 800,
      width: 1200,
      height: 800,
    });
  });

  it("shrinks the right edge for an open stretch panel docked right", () => {
    const inspector = panel({
      id: "inspector",
      side: "right",
      open: true,
      size: { width: 320, height: 0 },
    });

    const bounds = calculateCanvasBounds(CONTAINER, [inspector]);

    expect(bounds.right).toBe(1200 - 320);
    expect(bounds.width).toBe(1200 - 320);
    expect(bounds.height).toBe(800);
  });

  it("shrinks the bottom edge for an open stretch panel docked bottom", () => {
    const preview = panel({
      id: "data-preview",
      side: "bottom",
      open: true,
      size: { width: 0, height: 240 },
    });

    const bounds = calculateCanvasBounds(CONTAINER, [preview]);

    expect(bounds.bottom).toBe(800 - 240);
    expect(bounds.height).toBe(800 - 240);
  });

  it("changes when a panel's open state flips — the whole point of the reactive bounds", () => {
    const inspectorClosed = panel({
      id: "inspector",
      side: "right",
      open: false,
      size: { width: 320, height: 0 },
    });

    const inspectorOpen = { ...inspectorClosed, open: true };

    const closedBounds = calculateCanvasBounds(CONTAINER, [inspectorClosed]);
    const openBounds = calculateCanvasBounds(CONTAINER, [inspectorOpen]);

    expect(closedBounds).not.toEqual(openBounds);
    expect(closedBounds.width).toBe(1200);
    expect(openBounds.width).toBe(1200 - 320);
  });

  it("ignores 'center' anchored panels — they never shrink the outer bounds", () => {
    const palette = panel({
      id: "tool-palette",
      side: "top",
      anchor: "center",
      open: true,
      size: { width: 400, height: 56 },
    });

    const bounds = calculateCanvasBounds(CONTAINER, [palette]);

    expect(bounds).toEqual({
      top: 0,
      left: 0,
      right: 1200,
      bottom: 800,
      width: 1200,
      height: 800,
    });
  });

  it("stacks multiple open stretch panels on different sides", () => {
    const inspector = panel({
      id: "inspector",
      side: "right",
      open: true,
      size: { width: 320, height: 0 },
    });

    const preview = panel({
      id: "data-preview",
      side: "bottom",
      open: true,
      size: { width: 0, height: 240 },
    });

    const bounds = calculateCanvasBounds(CONTAINER, [inspector, preview]);

    expect(bounds).toEqual({
      top: 0,
      left: 0,
      right: 880,
      bottom: 560,
      width: 880,
      height: 560,
    });
  });

  it("never produces negative width/height when a panel is larger than the container", () => {
    const oversized = panel({
      id: "inspector",
      side: "right",
      open: true,
      size: { width: 5000, height: 0 },
    });

    const bounds = calculateCanvasBounds(CONTAINER, [oversized]);

    expect(bounds.width).toBe(0);
    expect(bounds.right).toBe(bounds.left);
  });
});

describe("boundsToRect", () => {
  it("maps top/left/right/bottom into an x/y/width/height rect", () => {
    const bounds = calculateCanvasBounds(CONTAINER, [
      panel({ id: "inspector", side: "right", open: true, size: { width: 300, height: 0 } }),
    ]);

    expect(boundsToRect(bounds)).toEqual({ x: 0, y: 0, width: 900, height: 800 });
  });
});

describe("computePanelRect", () => {
  it("spans the full width for a stretch panel docked bottom", () => {
    const rect = computePanelRect(CONTAINER, {
      side: "bottom",
      anchor: "stretch",
      size: { width: 0, height: 240 },
    });

    expect(rect).toEqual({ x: 0, y: 560, width: 1200, height: 240 });
  });

  it("centers a 'center' anchored panel along its docked side", () => {
    const rect = computePanelRect(CONTAINER, {
      side: "top",
      anchor: "center",
      size: { width: 400, height: 56 },
    });

    expect(rect).toEqual({ x: 400, y: 0, width: 400, height: 56 });
  });
});

describe("clampRectToBounds / isRectOutOfBounds", () => {
  const bounds = calculateCanvasBounds(CONTAINER, [
    panel({ id: "inspector", side: "right", open: true, size: { width: 300, height: 0 } }),
  ]);

  it("flags a card that now sits under the newly-opened panel as out of bounds", () => {
    const card = { x: 950, y: 100, width: 128, height: 128 };
    expect(isRectOutOfBounds(card, bounds)).toBe(true);
  });

  it("pulls an out-of-bounds card back inside without resizing it", () => {
    const card = { x: 950, y: 100, width: 128, height: 128 };
    const clamped = clampRectToBounds(card, bounds);

    expect(clamped.x + card.width).toBeLessThanOrEqual(bounds.right);
    expect(clamped.x).toBeGreaterThanOrEqual(bounds.left);
  });

  it("leaves an in-bounds card untouched", () => {
    const card = { x: 40, y: 40, width: 128, height: 128 };
    expect(isRectOutOfBounds(card, bounds)).toBe(false);
    expect(clampRectToBounds(card, bounds)).toEqual({ x: 40, y: 40 });
  });
});


=== FILE: src/canvas/layout/__tests__/dropZones.test.ts ===
import { describe, expect, it } from "vitest";

import type { PanelInstance } from "../canvasBounds";
import { calculateDropZones, isPointInRect, isValidDropPoint } from "../dropZones";

const CONTAINER = { width: 1200, height: 800 };

function panel(
  overrides: Partial<PanelInstance> & Pick<PanelInstance, "id" | "side">,
): PanelInstance {
  return {
    anchor: "stretch",
    open: false,
    size: { width: 0, height: 0 },
    ...overrides,
  };
}

describe("calculateDropZones", () => {
  it("has no obstacles and full bounds when every panel is closed", () => {
    const palette = panel({
      id: "tool-palette",
      side: "top",
      anchor: "center",
      open: false,
      size: { width: 400, height: 56 },
    });

    const zones = calculateDropZones(CONTAINER, [palette]);

    expect(zones.obstacles).toHaveLength(0);
    expect(zones.bounds.width).toBe(1200);
    expect(zones.bounds.height).toBe(800);
  });

  it("adds the palette's real bounding box as an obstacle when open, without shrinking bounds", () => {
    const palette = panel({
      id: "tool-palette",
      side: "top",
      anchor: "center",
      open: true,
      size: { width: 400, height: 56 },
    });

    const zones = calculateDropZones(CONTAINER, [palette]);

    // Bug 1.2: la palette non deve più bloccare l'intera fascia superiore.
    expect(zones.bounds).toEqual({
      top: 0,
      left: 0,
      right: 1200,
      bottom: 800,
      width: 1200,
      height: 800,
    });
    expect(zones.obstacles).toEqual([{ x: 400, y: 0, width: 400, height: 56 }]);
  });

  it("carves out an obstacle per open 'center' panel, ignoring closed ones", () => {
    const open = panel({
      id: "tool-palette",
      side: "left",
      anchor: "center",
      open: true,
      size: { width: 60, height: 300 },
    });
    const closed = panel({
      id: "other",
      side: "right",
      anchor: "center",
      open: false,
      size: { width: 60, height: 300 },
    });

    const zones = calculateDropZones(CONTAINER, [open, closed]);

    expect(zones.obstacles).toHaveLength(1);
  });

  it("recomputes both bounds and obstacles when the panel configuration changes", () => {
    const inspector = panel({
      id: "inspector",
      side: "right",
      anchor: "stretch",
      open: true,
      size: { width: 320, height: 0 },
    });
    const palette = panel({
      id: "tool-palette",
      side: "top",
      anchor: "center",
      open: true,
      size: { width: 400, height: 56 },
    });

    const before = calculateDropZones(CONTAINER, [palette]);
    const after = calculateDropZones(CONTAINER, [palette, inspector]);

    expect(before.bounds.width).toBe(1200);
    expect(after.bounds.width).toBe(1200 - 320);
    // La palette resta un ostacolo identico: l'inspector non la sposta.
    expect(after.obstacles).toEqual(before.obstacles);
  });
});

describe("isPointInRect", () => {
  it("treats the rect edges as inclusive", () => {
    const rect = { x: 10, y: 10, width: 100, height: 50 };
    expect(isPointInRect({ x: 10, y: 10 }, rect)).toBe(true);
    expect(isPointInRect({ x: 110, y: 60 }, rect)).toBe(true);
    expect(isPointInRect({ x: 111, y: 60 }, rect)).toBe(false);
  });
});

describe("isValidDropPoint", () => {
  const palette = panel({
    id: "tool-palette",
    side: "top",
    anchor: "center",
    open: true,
    size: { width: 400, height: 56 },
  });

  const inspector = panel({
    id: "inspector",
    side: "right",
    anchor: "stretch",
    open: true,
    size: { width: 320, height: 0 },
  });

  const zones = calculateDropZones(CONTAINER, [palette, inspector]);

  it("rejects a drop inside the open palette's bounding box", () => {
    expect(isValidDropPoint({ x: 600, y: 20 }, zones)).toBe(false);
  });

  it("accepts a drop next to the palette, within canvas bounds", () => {
    expect(isValidDropPoint({ x: 100, y: 20 }, zones)).toBe(true);
  });

  it("rejects a drop under the reserved inspector strip (outside bounds)", () => {
    expect(isValidDropPoint({ x: 950, y: 400 }, zones)).toBe(false);
  });

  it("accepts a drop in open canvas space away from every panel", () => {
    expect(isValidDropPoint({ x: 100, y: 400 }, zones)).toBe(true);
  });
});


=== FILE: src/canvas/layout/__tests__/panelRegistry.test.ts ===
import { describe, expect, it } from "vitest";

import { getPanelDefinition, isRegisteredPanel, PANEL_DEFINITIONS } from "../panelRegistry";
import { canvasReducer, createInitialCanvasState, toPanelInstances } from "../../store/canvasStore";

describe("PANEL_DEFINITIONS", () => {
  it("has a unique id for every panel", () => {
    const ids = PANEL_DEFINITIONS.map((panel) => panel.id);
    expect(new Set(ids).size).toBe(ids.length);
  });

  it("declares the three panels known to the ETL workspace today", () => {
    expect(PANEL_DEFINITIONS.map((p) => p.id).sort()).toEqual([
      "data-preview",
      "inspector",
      "tool-palette",
    ]);
  });

  it("marks the tool palette as 'center' anchored and the others as 'stretch'", () => {
    expect(getPanelDefinition("tool-palette")?.anchor).toBe("center");
    expect(getPanelDefinition("inspector")?.anchor).toBe("stretch");
    expect(getPanelDefinition("data-preview")?.anchor).toBe("stretch");
  });
});

describe("getPanelDefinition / isRegisteredPanel", () => {
  it("finds a registered panel by id", () => {
    expect(getPanelDefinition("inspector")).toBeDefined();
    expect(isRegisteredPanel("inspector")).toBe(true);
  });

  it("returns undefined/false for an unknown id", () => {
    expect(getPanelDefinition("not-a-panel")).toBeUndefined();
    expect(isRegisteredPanel("not-a-panel")).toBe(false);
  });
});

describe("registry ↔ store: toPanelInstances reflects real state", () => {
  it("starts every registered panel closed", () => {
    const initial = createInitialCanvasState();
    const instances = toPanelInstances(initial);

    expect(instances).toHaveLength(PANEL_DEFINITIONS.length);
    expect(instances.every((instance) => instance.open === false)).toBe(true);
  });

  it("reflects a PANEL_OPENED action for exactly the targeted panel", () => {
    const initial = createInitialCanvasState();
    const next = canvasReducer(initial, { type: "PANEL_OPENED", id: "inspector" });

    const instances = toPanelInstances(next);
    const inspector = instances.find((instance) => instance.id === "inspector");
    const preview = instances.find((instance) => instance.id === "data-preview");

    expect(inspector?.open).toBe(true);
    expect(preview?.open).toBe(false);
  });

  it("reflects a PANEL_RESIZED action's measured size", () => {
    const initial = createInitialCanvasState();
    const next = canvasReducer(initial, {
      type: "PANEL_RESIZED",
      id: "tool-palette",
      size: { width: 420, height: 60 },
    });

    const instances = toPanelInstances(next);
    const palette = instances.find((instance) => instance.id === "tool-palette");

    expect(palette?.size).toEqual({ width: 420, height: 60 });
  });

  it("ignores an action targeting an id absent from the registry", () => {
    const initial = createInitialCanvasState();
    const next = canvasReducer(initial, { type: "PANEL_OPENED", id: "not-a-panel" });

    expect(next).toEqual(initial);
  });

  it("PANEL_TOGGLED flips only the current open state", () => {
    const initial = createInitialCanvasState();
    const opened = canvasReducer(initial, { type: "PANEL_TOGGLED", id: "data-preview" });
    const closedAgain = canvasReducer(opened, { type: "PANEL_TOGGLED", id: "data-preview" });

    expect(toPanelInstances(opened).find((p) => p.id === "data-preview")?.open).toBe(true);
    expect(toPanelInstances(closedAgain).find((p) => p.id === "data-preview")?.open).toBe(false);
  });
});


=== FILE: src/canvas/layout/canvasBounds.ts ===
/**
 * Canvas Edge System — geometria pura, nessuna dipendenza da React o dal
 * resto della repo: calcola quanto spazio il canvas ha davvero a
 * disposizione una volta sottratti i pannelli ausiliari agganciati ai
 * suoi lati. Nessuno stato qui dentro: le funzioni sono pure, quindi
 * facilmente testabili e riutilizzabili sia lato hook (useCanvasBounds)
 * sia lato test.
 */

export type Point = {
  x: number;
  y: number;
};

export type Rect = Point & {
  width: number;
  height: number;
};

export type PanelSide = "top" | "right" | "bottom" | "left";

/**
 * "stretch": il pannello occupa l'intera striscia lungo il proprio lato
 * (es. Inspector, Data Preview) — quando è aperto, riduce lo spazio
 * disponibile del canvas su quel lato.
 *
 * "center": il pannello galleggia centrato sul proprio lato (es. Tool
 * Palette) — NON riduce lo spazio disponibile del canvas (le card
 * possono stare sopra/sotto/accanto ad esso), ma il suo bounding box
 * reale va comunque escluso dalle drop zone (vedi dropZones.ts).
 */
export type PanelAnchor = "stretch" | "center";

export type PanelSize = {
  width: number;
  height: number;
};

/** Stato "live" di un pannello: definizione + stato runtime, così come lo consuma calculateCanvasBounds. */
export type PanelInstance = {
  id: string;
  side: PanelSide;
  anchor: PanelAnchor;
  open: boolean;
  size: PanelSize;
};

export type CanvasContainerSize = {
  width: number;
  height: number;
};

export type CanvasBounds = {
  top: number;
  left: number;
  right: number;
  bottom: number;
  width: number;
  height: number;
};

/**
 * Il rettangolo effettivamente disponibile per il canvas, una volta
 * sottratti tutti i pannelli "stretch" attualmente aperti. I pannelli
 * "center" (palette) non alterano questo rettangolo: la loro presenza è
 * gestita come "buco" dalle drop zone (dropZones.ts), non come riduzione
 * del perimetro — perché lo spazio accanto a un pannello centrato resta
 * usabile.
 *
 * Pura funzione di (container, panels): nessun accesso al DOM. Va
 * richiamata ogni volta che container o panels cambiano — vedi
 * useCanvasBounds per il lato reattivo.
 */
export function calculateCanvasBounds(
  container: CanvasContainerSize,
  panels: readonly PanelInstance[],
): CanvasBounds {
  let top = 0;
  let left = 0;
  let right = Math.max(0, container.width);
  let bottom = Math.max(0, container.height);

  for (const panel of panels) {
    if (!panel.open || panel.anchor !== "stretch") {
      continue;
    }

    switch (panel.side) {
      case "top":
        top = Math.min(bottom, top + panel.size.height);
        break;

      case "bottom":
        bottom = Math.max(top, bottom - panel.size.height);
        break;

      case "left":
        left = Math.min(right, left + panel.size.width);
        break;

      case "right":
        right = Math.max(left, right - panel.size.width);
        break;
    }
  }

  return {
    top,
    left,
    right,
    bottom,
    width: Math.max(0, right - left),
    height: Math.max(0, bottom - top),
  };
}

export function boundsToRect(bounds: CanvasBounds): Rect {
  return {
    x: bounds.left,
    y: bounds.top,
    width: bounds.width,
    height: bounds.height,
  };
}

/**
 * Posizione/dimensione reale di un pannello dentro `container`, in base
 * al proprio lato e ancoraggio. Un pannello "stretch" copre l'intera
 * striscia del lato; un pannello "center" è centrato sull'asse
 * trasversale del lato (stessa regola già usata per la Tool Palette in
 * workflow-canvas.tsx, qui generalizzata a qualunque pannello del
 * registry).
 */
export function computePanelRect(
  container: CanvasContainerSize,
  panel: Pick<PanelInstance, "side" | "anchor" | "size">,
): Rect {
  const { side, anchor, size } = panel;

  switch (side) {
    case "top":
      return anchor === "stretch"
        ? { x: 0, y: 0, width: container.width, height: size.height }
        : {
            x: (container.width - size.width) / 2,
            y: 0,
            width: size.width,
            height: size.height,
          };

    case "bottom":
      return anchor === "stretch"
        ? {
            x: 0,
            y: Math.max(0, container.height - size.height),
            width: container.width,
            height: size.height,
          }
        : {
            x: (container.width - size.width) / 2,
            y: Math.max(0, container.height - size.height),
            width: size.width,
            height: size.height,
          };

    case "left":
      return anchor === "stretch"
        ? { x: 0, y: 0, width: size.width, height: container.height }
        : {
            x: 0,
            y: (container.height - size.height) / 2,
            width: size.width,
            height: size.height,
          };

    case "right":
      return anchor === "stretch"
        ? {
            x: Math.max(0, container.width - size.width),
            y: 0,
            width: size.width,
            height: container.height,
          }
        : {
            x: Math.max(0, container.width - size.width),
            y: (container.height - size.height) / 2,
            width: size.width,
            height: size.height,
          };
  }
}

/**
 * Riporta `rect` dentro `bounds`, senza alterarne le dimensioni.
 * Usata dall'handler di "canvas bounds changed" (PARTE 2.2) per
 * ricollocare card che sconfinano quando i pannelli cambiano stato —
 * la transizione fluida è responsabilità del chiamante (CSS transition
 * sulla posizione), qui c'è solo il calcolo del punto sicuro.
 */
export function clampRectToBounds(rect: Rect, bounds: CanvasBounds): Point {
  const maxX = Math.max(bounds.left, bounds.right - rect.width);
  const maxY = Math.max(bounds.top, bounds.bottom - rect.height);

  return {
    x: Math.min(Math.max(rect.x, bounds.left), maxX),
    y: Math.min(Math.max(rect.y, bounds.top), maxY),
  };
}

/** `rect` sconfina rispetto a `bounds`? Usata per decidere se serve riposizionare. */
export function isRectOutOfBounds(rect: Rect, bounds: CanvasBounds): boolean {
  return (
    rect.x < bounds.left ||
    rect.y < bounds.top ||
    rect.x + rect.width > bounds.right ||
    rect.y + rect.height > bounds.bottom
  );
}


=== FILE: src/canvas/layout/dropZones.ts ===
import type { CanvasBounds, CanvasContainerSize, PanelInstance, Point, Rect } from "./canvasBounds";
import { calculateCanvasBounds, computePanelRect } from "./canvasBounds";

/**
 * Zona di drop valida per il canvas: il rettangolo `bounds` (già al netto
 * dei pannelli "stretch", vedi canvasBounds.ts) MENO i rettangoli
 * `obstacles` — il bounding box reale dei pannelli "center" (Tool
 * Palette) attualmente aperti. Un punto è un drop valido se cade dentro
 * `bounds` e fuori da ogni obstacle.
 */
export type DropZoneMap = {
  bounds: CanvasBounds;
  obstacles: Rect[];
};

export function calculateDropZones(
  container: CanvasContainerSize,
  panels: readonly PanelInstance[],
): DropZoneMap {
  const bounds = calculateCanvasBounds(container, panels);

  const obstacles = panels
    .filter((panel) => panel.open && panel.anchor === "center")
    .map((panel) => computePanelRect(container, panel));

  return { bounds, obstacles };
}

export function isPointInRect(point: Point, rect: Rect): boolean {
  return (
    point.x >= rect.x &&
    point.x <= rect.x + rect.width &&
    point.y >= rect.y &&
    point.y <= rect.y + rect.height
  );
}

function isPointInBounds(point: Point, bounds: CanvasBounds): boolean {
  return (
    point.x >= bounds.left &&
    point.x <= bounds.right &&
    point.y >= bounds.top &&
    point.y <= bounds.bottom
  );
}

/**
 * Il punto di drop (3.1 nel contesto strategico) è valido se cade dentro
 * i bounds reali del canvas e non dentro il bounding box di un pannello
 * "center" aperto (es. la Tool Palette).
 */
export function isValidDropPoint(point: Point, dropZones: DropZoneMap): boolean {
  if (!isPointInBounds(point, dropZones.bounds)) {
    return false;
  }

  return !dropZones.obstacles.some((obstacle) => isPointInRect(point, obstacle));
}


=== FILE: src/canvas/layout/panelRegistry.ts ===
import type { PanelAnchor, PanelSide, PanelSize } from "./canvasBounds";

/**
 * Registro dichiarativo dei pannelli ausiliari che possono ridurre lo
 * spazio del canvas. È l'unico posto in cui un nuovo pannello va
 * annunciato: aggiungere un pannello significa aggiungere una riga qui,
 * non toccare la logica di calcolo bounds/drop-zone.
 *
 * `closedSize` è quasi sempre zero (il pannello scompare del tutto da
 * chiuso), ma è esplicito nel tipo per supportare — senza cambiare la
 * shape — un futuro pannello che resta parzialmente visibile da chiuso
 * (es. una linguetta).
 */
export type PanelDefinition = {
  id: string;
  label: string;
  side: PanelSide;
  anchor: PanelAnchor;
  closedSize: PanelSize;
};

const ZERO_SIZE: PanelSize = { width: 0, height: 0 };

/**
 * I tre pannelli ausiliari oggi presenti nel workspace ETL (bug 1.1 nel
 * contesto strategico): Tool Palette, Inspector, Data Preview. Vedi
 * README.md di questa cartella per la mappatura side/anchor di ciascuno
 * e per come collegarne uno nuovo.
 */
export const PANEL_DEFINITIONS: readonly PanelDefinition[] = [
  {
    id: "tool-palette",
    label: "Tool Palette",
    side: "top",
    anchor: "center",
    closedSize: ZERO_SIZE,
  },
  {
    id: "inspector",
    label: "Inspector",
    side: "right",
    anchor: "stretch",
    closedSize: ZERO_SIZE,
  },
  {
    id: "data-preview",
    label: "Data Preview",
    side: "bottom",
    anchor: "stretch",
    closedSize: ZERO_SIZE,
  },
];

export function getPanelDefinition(id: string): PanelDefinition | undefined {
  return PANEL_DEFINITIONS.find((panel) => panel.id === id);
}

export function isRegisteredPanel(id: string): boolean {
  return getPanelDefinition(id) !== undefined;
}


=== FILE: src/canvas/layout/surfacePanels.ts ===
import type {
  CanvasContainerSize,
  PanelAnchor,
  PanelInstance,
  PanelSide,
  PanelSize,
  Rect,
} from "./canvasBounds";
import { calculateCanvasBounds, clampRectToBounds, computePanelRect } from "./canvasBounds";

/**
 * Fase 2A — geometria di un pannello "controscalato": vive DENTRO la
 * superficie zoomata di workflow-canvas.tsx (`transform: scale(zoom)`)
 * ma applica il proprio `scale(1/zoom)` per restare a dimensione fisica
 * costante sullo schermo, indipendente dallo zoom — la stessa tecnica
 * già usata dalla Tool Palette (che però vive FUORI dalla superficie).
 *
 * Compone solo le funzioni pure di canvasBounds.ts, senza modificarle
 * (Fase 1 resta intoccata, come richiesto).
 *
 * Due conversioni di unità da non confondere, fonte facile di bug:
 * - `physicalSize`: pixel fisici costanti — quello che l'utente vede a
 *   schermo, invariante rispetto a `zoom` grazie al controscale.
 * - unità superficie: `physicalSize / zoom` — il sistema di coordinate
 *   in cui vivono le card e (già oggi) l'ingombro della Tool Palette.
 *   calculateCanvasBounds/computePanelRect lavorano SEMPRE in unità
 *   superficie: bisogna convertire physicalSize prima di passarglielo.
 *
 * Un pannello non deve "fare spazio" a se stesso: i bounds usati per
 * posizionarlo escludono il pannello stesso dalla lista — altrimenti la
 * sua stessa presenza in canvasStore ridurrebbe lo spazio a disposizione
 * due volte (una nel bounds generale, una qui).
 */
export function computeSurfacePanelRect(
  container: CanvasContainerSize,
  panels: readonly PanelInstance[],
  panelId: string,
  geometry: { side: PanelSide; anchor: PanelAnchor; physicalSize: PanelSize },
  zoom: number,
): Rect {
  const bounds = calculateCanvasBounds(
    container,
    panels.filter((panel) => panel.id !== panelId),
  );

  const surfaceSize: PanelSize = {
    width: geometry.physicalSize.width / zoom,
    height: geometry.physicalSize.height / zoom,
  };

  const local = computePanelRect(
    { width: bounds.width, height: bounds.height },
    { side: geometry.side, anchor: geometry.anchor, size: surfaceSize },
  );

  const translated: Rect = {
    x: local.x + bounds.left,
    y: local.y + bounds.top,
    width: local.width,
    height: local.height,
  };

  const clampedPosition = clampRectToBounds(translated, bounds);

  return { ...translated, ...clampedPosition };
}

/**
 * `rect` (in unità superficie, da computeSurfacePanelRect) → stile CSS
 * per l'elemento controscalato: `left`/`top` restano in unità
 * superficie (il genitore scalato li reinterpreta correttamente),
 * `width`/`height` vanno moltiplicati per `zoom` per tornare a pixel
 * fisici costanti — l'inverso della conversione fatta sopra.
 */
export function surfacePanelStyle(
  rect: Rect,
  zoom: number,
): { left: number; top: number; width: number; height: number } {
  return {
    left: rect.x,
    top: rect.y,
    width: rect.width * zoom,
    height: rect.height * zoom,
  };
}


=== FILE: src/canvas/store/canvasStore.tsx ===
import { createContext, useContext, useMemo, useReducer } from "react";

import type { PanelInstance, PanelSize } from "../layout/canvasBounds";
import { PANEL_DEFINITIONS } from "../layout/panelRegistry";

/**
 * Stato dei pannelli aperti/chiusi: la sorgente di verità che, insieme
 * alla dimensione del contenitore canvas, alimenta
 * calculateCanvasBounds/calculateDropZones (src/canvas/layout).
 *
 * Context + useReducer invece di Zustand/Redux: il resto della repo
 * gestisce già stato condiviso così (src/lib/theme.tsx,
 * src/lib/solutions-store.tsx) e il caso d'uso — tre pannelli, poche
 * azioni — non giustifica una libreria di state management in più.
 */
export type PanelRuntimeState = {
  open: boolean;
  size: PanelSize;
};

export type CanvasStoreState = {
  panels: Record<string, PanelRuntimeState>;
};

export type CanvasAction =
  | { type: "PANEL_OPENED"; id: string }
  | { type: "PANEL_CLOSED"; id: string }
  | { type: "PANEL_TOGGLED"; id: string }
  | { type: "PANEL_RESIZED"; id: string; size: PanelSize };

export function createInitialCanvasState(): CanvasStoreState {
  const panels: Record<string, PanelRuntimeState> = {};

  for (const def of PANEL_DEFINITIONS) {
    panels[def.id] = { open: false, size: { ...def.closedSize } };
  }

  return { panels };
}

function patchPanel(
  state: CanvasStoreState,
  id: string,
  patch: Partial<PanelRuntimeState>,
): CanvasStoreState {
  const current = state.panels[id];

  if (!current) {
    // Pannello non registrato in panelRegistry.ts: azione ignorata invece
    // di introdurre silenziosamente una entry "orfana" nello stato.
    return state;
  }

  return {
    panels: {
      ...state.panels,
      [id]: { ...current, ...patch },
    },
  };
}

/** Pura, senza React: testata direttamente in canvasBounds.test.ts / panelRegistry.test.ts. */
export function canvasReducer(state: CanvasStoreState, action: CanvasAction): CanvasStoreState {
  switch (action.type) {
    case "PANEL_OPENED":
      return patchPanel(state, action.id, { open: true });

    case "PANEL_CLOSED":
      return patchPanel(state, action.id, { open: false });

    case "PANEL_TOGGLED": {
      const current = state.panels[action.id];
      return current ? patchPanel(state, action.id, { open: !current.open }) : state;
    }

    case "PANEL_RESIZED":
      return patchPanel(state, action.id, { size: action.size });

    default:
      return state;
  }
}

/** Converte lo stato runtime + il registry statico in ciò che consuma calculateCanvasBounds/calculateDropZones. */
export function toPanelInstances(state: CanvasStoreState): PanelInstance[] {
  return PANEL_DEFINITIONS.map((def) => {
    const runtime = state.panels[def.id];

    return {
      id: def.id,
      side: def.side,
      anchor: def.anchor,
      open: runtime?.open ?? false,
      size: runtime?.size ?? def.closedSize,
    };
  });
}

type CanvasStoreContextValue = {
  state: CanvasStoreState;
  dispatch: React.Dispatch<CanvasAction>;
};

const CanvasStoreContext = createContext<CanvasStoreContextValue | null>(null);

export function CanvasStoreProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(canvasReducer, undefined, createInitialCanvasState);

  const value = useMemo(() => ({ state, dispatch }), [state]);

  return <CanvasStoreContext.Provider value={value}>{children}</CanvasStoreContext.Provider>;
}

export function useCanvasStore(): CanvasStoreContextValue {
  const ctx = useContext(CanvasStoreContext);

  if (!ctx) {
    throw new Error("useCanvasStore must be used inside CanvasStoreProvider");
  }

  return ctx;
}


=== FILE: src/components/isa/etl/data-preview.tsx ===
import { useEffect } from "react";
import { Table2, X } from "lucide-react";
import { useCanvasBoundsContext } from "@/canvas/components/CanvasContainer";
import { usePanelState } from "@/canvas/hooks/usePanelState";
import { computeSurfacePanelRect, surfacePanelStyle } from "@/canvas/layout/surfacePanels";
import { analyzeNode, formatRows, previewRows } from "@/lib/etl-schema";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";

/**
 * Fase 2A: Data Preview vive dentro la superficie zoomata (vedi
 * workflow-canvas.tsx / src/canvas/layout/surfacePanels.ts). Evita
 * l'Inspector automaticamente — tramite i bounds condivisi in
 * canvasStore, non più tramite il vecchio prop `inset` cablato a mano
 * con una larghezza magica (21.5rem).
 */

/** Altezza fisica: stessa proporzione (42% dell'altezza del canvas) della vecchia classe Tailwind max-h-[42%], ma calcolata (non un CSS max-height) perché ora è il canvas a doverne conoscere l'ingombro per i bounds. */
const PREVIEW_HEIGHT_RATIO = 0.42;
const PREVIEW_HEIGHT_MIN = 160;

/* Margini puramente estetici (stessi bottom-3/left-3/right-3 di prima) — non entrano nel calcolo dei bounds condivisi. */
const PREVIEW_INSET_LEFT = 12;
const PREVIEW_INSET_RIGHT = 12;
const PREVIEW_INSET_BOTTOM = 12;

function previewPhysicalHeight(containerHeight: number, zoom: number): number {
  return Math.max(PREVIEW_HEIGHT_MIN, Math.round(containerHeight * zoom * PREVIEW_HEIGHT_RATIO));
}

/** Cassetto Data preview: si apre solo su comando e si chiude con la X. */
export function DataPreview({
  workflow,
  node,
  open,
  onClose,
}: {
  workflow: EtlWorkflow;
  node: EtlNode | null;
  open: boolean;
  onClose: () => void;
}) {
  const { openPanel, closePanel, setSize } = usePanelState("data-preview");
  const { container, panels, zoom } = useCanvasBoundsContext();

  useEffect(() => {
    if (open) {
      openPanel();
    } else {
      closePanel();
    }
  }, [open, openPanel, closePanel]);

  useEffect(() => {
    const physicalHeight = previewPhysicalHeight(container.height, zoom);
    setSize({ width: 0, height: physicalHeight / zoom });
  }, [container.height, zoom, setSize]);

  if (!open) return null;

  const analysis = node ? analyzeNode(workflow, node) : null;
  const columns = analysis?.columns ?? [];
  const rows = node ? previewRows(columns, node.id) : [];

  const rect = computeSurfacePanelRect(
    container,
    panels,
    "data-preview",
    {
      side: "bottom",
      anchor: "stretch",
      physicalSize: { width: 0, height: previewPhysicalHeight(container.height, zoom) },
    },
    zoom,
  );

  const style = surfacePanelStyle(rect, zoom);

  return (
    <section
      className="glass-panel absolute z-40 flex flex-col rounded-3xl"
      style={{
        left: style.left + PREVIEW_INSET_LEFT / zoom,
        top: style.top,
        width: Math.max(0, style.width - PREVIEW_INSET_LEFT - PREVIEW_INSET_RIGHT),
        height: Math.max(0, style.height - PREVIEW_INSET_BOTTOM),
        transform: `scale(${1 / zoom})`,
        transformOrigin: "top left",
        transition: "left 0.2s ease, top 0.2s ease, width 0.2s ease",
        background: "color-mix(in srgb, var(--background) 92%, transparent)",
      }}
    >
      <div className="flex items-center gap-2 px-4 py-2.5">
        <Table2 className="size-4 text-muted-foreground" />
        <span className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
          Data preview
        </span>
        <span className="glass-chip truncate rounded-full px-2.5 py-1 text-[10px] text-muted-foreground">
          {node ? node.title : "nessun nodo selezionato"}
        </span>
        {analysis && (
          <span className="glass-chip hidden rounded-full px-2.5 py-1 text-[10px] text-muted-foreground sm:inline">
            {formatRows(analysis.rows)} rows · {columns.length} cols
          </span>
        )}
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi data preview"
          className="glass-chip ml-auto flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>
      </div>

      <div className="scroll-slim min-h-0 flex-1 overflow-auto border-t border-border/60 px-4 py-3">
        {!node ? (
          <p className="text-xs text-muted-foreground">
            Seleziona un nodo nel canvas per ispezionarne i dati.
          </p>
        ) : columns.length === 0 ? (
          <p className="text-xs text-muted-foreground">
            Completa la configurazione del nodo nell'Inspector per definire lo schema in uscita.
          </p>
        ) : (
          <>
            <div className="overflow-x-auto rounded-2xl border border-border/60">
              <table className="w-full text-left text-xs">
                <thead>
                  <tr>
                    {columns.map((c) => (
                      <th key={c.name} className="whitespace-nowrap px-3 py-2 font-semibold">
                        {c.name}
                        <span className="ml-1.5 text-[9px] font-normal text-muted-foreground">
                          {c.type}
                        </span>
                      </th>
                    ))}
                  </tr>
                </thead>
                <tbody>
                  {rows.map((row, i) => (
                    <tr key={i} className="border-t border-border/60">
                      {row.map((cell, j) => (
                        <td
                          key={j}
                          className="whitespace-nowrap px-3 py-1.5 font-mono text-[11px] text-muted-foreground"
                        >
                          {cell}
                        </td>
                      ))}
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
            <p className="mt-2 text-[10px] text-muted-foreground">
              Anteprima di authoring: i dati reali compariranno quando il motore di esecuzione sarà
              collegato.
            </p>
          </>
        )}
      </div>
    </section>
  );
}


=== FILE: src/components/isa/etl/inspector.tsx ===
import { useEffect } from "react";
import { AlertTriangle, Table2, Trash2, X } from "lucide-react";
import { useCanvasBoundsContext } from "@/canvas/components/CanvasContainer";
import { usePanelState } from "@/canvas/hooks/usePanelState";
import { computeSurfacePanelRect, surfacePanelStyle } from "@/canvas/layout/surfacePanels";
import type { ColumnDef } from "@/lib/etl-catalog";
import { categoryAccent, nodeDef } from "@/lib/etl-catalog";
import { analyzeNode, formatRows } from "@/lib/etl-schema";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";

/**
 * Fase 2A: l'Inspector vive dentro la superficie zoomata del canvas
 * (vedi workflow-canvas.tsx) ma applica un controscale per restare a
 * dimensione fisica costante sullo schermo — vedi
 * src/canvas/layout/surfacePanels.ts per la tecnica e il perché.
 */

/** Larghezza fisica costante (px schermo, indipendente da zoom) — prima classe Tailwind w-[min(20rem,...)]. */
const INSPECTOR_WIDTH = 320;

/* Margini puramente estetici (spazio per i controlli zoom/griglia in alto, un po' d'aria in basso) — non entrano nel calcolo dei bounds condivisi. */
const INSPECTOR_INSET_TOP = 56;
const INSPECTOR_INSET_BOTTOM = 12;

/** Cassetto Inspector: si apre solo su comando e si chiude con la X. */
export function Inspector({
  workflow,
  node,
  open,
  onClose,
  onChangeTitle,
  onChangeConfig,
  onRemove,
  onPreview,
}: {
  workflow: EtlWorkflow;
  node: EtlNode | null;
  open: boolean;
  onClose: () => void;
  onChangeTitle: (value: string) => void;
  onChangeConfig: (key: string, value: string) => void;
  onRemove: () => void;
  onPreview: () => void;
}) {
  const { openPanel, closePanel, setSize } = usePanelState("inspector");
  const { container, panels, zoom } = useCanvasBoundsContext();

  /*
   * `open` (prop) resta la fonte di verità per "l'Inspector è aperto":
   * qui la specchiamo in canvasStore, così Data Preview (o un futuro
   * pannello) può sapere che deve farle spazio senza che nessuno gliela
   * passi esplicitamente (bug del vecchio `inset` prop, rimosso).
   */
  useEffect(() => {
    if (open) {
      openPanel();
    } else {
      closePanel();
    }
  }, [open, openPanel, closePanel]);

  /*
   * La larghezza è fisica-costante (non dipende dal contenuto), quindi
   * non serve un ResizeObserver: solo ricalcolare la sua controparte in
   * unità superficie (÷ zoom) quando zoom o lo spazio disponibile
   * cambiano.
   */
  useEffect(() => {
    const physicalWidth = Math.max(0, Math.min(INSPECTOR_WIDTH, container.width * zoom - 24));
    setSize({ width: physicalWidth / zoom, height: 0 });
  }, [container.width, zoom, setSize]);

  if (!open) return null;

  const def = node ? nodeDef(node.type) : undefined;
  const analysis = node ? analyzeNode(workflow, node) : null;

  const rect = computeSurfacePanelRect(
    container,
    panels,
    "inspector",
    { side: "right", anchor: "stretch", physicalSize: { width: INSPECTOR_WIDTH, height: 0 } },
    zoom,
  );

  const style = surfacePanelStyle(rect, zoom);

  return (
    <aside
      className="glass-panel absolute z-40 flex flex-col rounded-3xl p-4"
      style={{
        left: style.left,
        top: style.top + INSPECTOR_INSET_TOP / zoom,
        width: style.width,
        height: Math.max(0, style.height - INSPECTOR_INSET_TOP - INSPECTOR_INSET_BOTTOM),
        transform: `scale(${1 / zoom})`,
        transformOrigin: "top left",
        transition: "left 0.2s ease, top 0.2s ease, height 0.2s ease",
        background: "color-mix(in srgb, var(--background) 92%, transparent)",
      }}
    >
      <div className="flex items-center gap-2">
        <span className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
          Inspector
        </span>
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi inspector"
          className="glass-chip ml-auto flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>
      </div>

      {!node || !def || !analysis ? (
        <p className="mt-3 text-xs text-muted-foreground">
          Seleziona un nodo nel canvas per configurarne i parametri.
        </p>
      ) : (
        <div className="scroll-slim mt-3 min-h-0 flex-1 space-y-4 overflow-y-auto pr-1">
          <div className="flex items-center gap-2">
            <span
              className={`${categoryAccent(def.category)} flex size-8 shrink-0 items-center justify-center rounded-xl`}
            >
              <def.Icon className="size-4" />
            </span>
            <span className="min-w-0">
              <span className="block truncate text-sm font-semibold">{def.label}</span>
              <span className="block truncate text-[11px] text-muted-foreground">
                {def.description}
              </span>
            </span>
          </div>

          <div className="flex flex-wrap gap-1.5">
            <span className="glass-chip rounded-full px-2.5 py-1 text-[10px] text-muted-foreground">
              {formatRows(analysis.rows)} rows
            </span>
            <span className="glass-chip rounded-full px-2.5 py-1 text-[10px] text-muted-foreground">
              {analysis.columns.length} cols
            </span>
            <span
              className="glass-chip rounded-full px-2.5 py-1 text-[10px] font-semibold"
              style={{ color: analysis.errors.length ? "var(--destructive)" : "var(--success)" }}
            >
              {analysis.errors.length ? "Non valido" : "Valido"}
            </span>
          </div>

          {analysis.errors.length > 0 && (
            <ul className="glass-chip space-y-1.5 rounded-2xl p-3">
              {analysis.errors.map((err) => (
                <li key={err} className="flex gap-2 text-[11px] text-destructive">
                  <AlertTriangle className="mt-0.5 size-3.5 shrink-0" />
                  {err}
                </li>
              ))}
            </ul>
          )}

          <div>
            <label htmlFor="isa-node-title" className="text-xs font-medium">
              Nome nodo
            </label>
            <input
              id="isa-node-title"
              value={node.title}
              onChange={(e) => onChangeTitle(e.target.value)}
              className="glass-chip mt-1.5 h-10 w-full rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
            />
          </div>

          {def.fields.map((field) => {
            const id = `isa-field-${field.key}`;
            const value = node.config[field.key] ?? "";
            return (
              <div key={field.key}>
                <label htmlFor={id} className="text-xs font-medium">
                  {field.label}
                  {field.required && <span className="text-destructive"> *</span>}
                </label>
                {field.kind === "textarea" ? (
                  <textarea
                    id={id}
                    value={value}
                    rows={3}
                    placeholder={field.placeholder}
                    onChange={(e) => onChangeConfig(field.key, e.target.value)}
                    className="glass-chip mt-1.5 w-full resize-none rounded-2xl px-3 py-2 font-mono text-xs outline-none focus:ring-2 focus:ring-ring"
                  />
                ) : field.kind === "select" ? (
                  <select
                    id={id}
                    value={value}
                    onChange={(e) => onChangeConfig(field.key, e.target.value)}
                    className="glass-chip mt-1.5 h-10 w-full rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
                  >
                    {field.options?.map((opt) => (
                      <option key={opt} value={opt}>
                        {opt}
                      </option>
                    ))}
                  </select>
                ) : (
                  <input
                    id={id}
                    value={value}
                    placeholder={field.placeholder}
                    onChange={(e) => onChangeConfig(field.key, e.target.value)}
                    className="glass-chip mt-1.5 h-10 w-full rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
                  />
                )}
              </div>
            );
          })}

          <div>
            <span className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
              Schema evolution
            </span>
            <div className="mt-2 grid gap-2 sm:grid-cols-2">
              <SchemaList title="Input" columns={analysis.inputColumns} />
              <SchemaList title="Output" columns={analysis.columns} />
            </div>
          </div>

          <button
            type="button"
            onClick={onPreview}
            className="glass-chip flex h-10 w-full items-center justify-center gap-2 rounded-full text-sm font-medium transition hover:brightness-105"
          >
            <Table2 className="size-4" />
            Preview data
          </button>

          <button
            type="button"
            onClick={onRemove}
            className="glass-chip flex h-10 w-full items-center justify-center gap-2 rounded-full text-sm font-medium text-destructive transition hover:brightness-105"
          >
            <Trash2 className="size-4" />
            Elimina nodo
          </button>
        </div>
      )}
    </aside>
  );
}

function SchemaList({ title, columns }: { title: string; columns: ColumnDef[] }) {
  return (
    <div className="glass-chip rounded-2xl p-2.5">
      <span className="text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
        {title}
      </span>
      {columns.length === 0 ? (
        <p className="mt-1.5 text-[11px] text-muted-foreground">—</p>
      ) : (
        <ul className="scroll-slim mt-1.5 max-h-40 space-y-1 overflow-y-auto pr-1">
          {columns.map((c) => (
            <li key={c.name} className="flex items-baseline gap-1.5">
              <span className="min-w-0 flex-1 truncate text-[11px] font-medium">{c.name}</span>
              <span className="text-[9px] text-muted-foreground">{c.type}</span>
              {c.nullable && <span className="text-[9px] text-muted-foreground/70">null</span>}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}


=== FILE: src/components/isa/etl/isa-context-menu.tsx ===
import {
  Copy,
  Grid3x3,
  LayoutGrid,
  Maximize2,
  Plus,
  Trash2,
  Unlink,
} from "lucide-react";
import {
  useEffect,
  useLayoutEffect,
  useRef,
  useState,
} from "react";
import { createPortal } from "react-dom";

type ContextTarget = {
  nodeId: string | null;
};

type IsaContextMenuProps = {
  open: boolean;
  x: number;
  y: number;
  boundaryRef: React.RefObject<HTMLElement | null>;
  target: ContextTarget;
  grid: boolean;
  onAddDataset: () => void;
  onFitView: () => void;
  onToggleGrid: () => void;
  onAutoLayout: () => void;
  onDuplicateNode?: (id: string) => void;
  onRemoveNode?: (id: string) => void;
  onUnlinkNode?: (id: string) => void;
  onClose: () => void;
};

const MENU_WIDTH = 224;
const MENU_MARGIN = 8;

export function IsaContextMenu({
  open,
  x,
  y,
  boundaryRef,
  target,
  grid,
  onAddDataset,
  onFitView,
  onToggleGrid,
  onAutoLayout,
  onDuplicateNode,
  onRemoveNode,
  onUnlinkNode,
  onClose,
}: IsaContextMenuProps) {
  const menuRef =
    useRef<HTMLDivElement>(null);

  const [position, setPosition] = useState({
    left: x,
    top: y,
  });

  useLayoutEffect(() => {
    if (!open) return;

    const boundary =
      boundaryRef.current;

    const menu =
      menuRef.current;

    if (!boundary || !menu) return;

    const boundaryRect =
      boundary.getBoundingClientRect();

    const menuRect =
      menu.getBoundingClientRect();

    const maxLeft =
      Math.max(
        boundaryRect.left + MENU_MARGIN,
        boundaryRect.right -
          menuRect.width -
          MENU_MARGIN,
      );

    const maxTop =
      Math.max(
        boundaryRect.top + MENU_MARGIN,
        boundaryRect.bottom -
          menuRect.height -
          MENU_MARGIN,
      );

    setPosition({
      left: Math.min(
        Math.max(
          x,
          boundaryRect.left +
            MENU_MARGIN,
        ),
        maxLeft,
      ),
      top: Math.min(
        Math.max(
          y,
          boundaryRect.top +
            MENU_MARGIN,
        ),
        maxTop,
      ),
    });
  }, [
    open,
    x,
    y,
    boundaryRef,
    target.nodeId,
  ]);

  useEffect(() => {
    if (!open) return;

    const handlePointerDown = (
      event: PointerEvent,
    ) => {
      if (
        !menuRef.current?.contains(
          event.target as Node,
        )
      ) {
        onClose();
      }
    };

    const handleKeyDown = (
      event: KeyboardEvent,
    ) => {
      if (event.key === "Escape") {
        onClose();
      }
    };

    document.addEventListener(
      "pointerdown",
      handlePointerDown,
    );

    document.addEventListener(
      "keydown",
      handleKeyDown,
    );

    return () => {
      document.removeEventListener(
        "pointerdown",
        handlePointerDown,
      );

      document.removeEventListener(
        "keydown",
        handleKeyDown,
      );
    };
  }, [open, onClose]);

  if (
    !open ||
    typeof document ===
      "undefined"
  ) {
    return null;
  }

  const hasNode =
    Boolean(target.nodeId);

  return createPortal(
    <div
      ref={menuRef}
      role="menu"
      aria-label="Menu contestuale IsA"
      className="fixed z-100 w-56 overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl"
      style={{
        left: position.left,
        top: position.top,
      }}
      onPointerDown={(event) =>
        event.stopPropagation()
      }
      onContextMenu={(event) =>
        event.preventDefault()
      }
    >
      {hasNode ? (
        <>
          <button
            type="button"
            role="menuitem"
            className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
            onClick={() => {
              if (
                target.nodeId &&
                onDuplicateNode
              ) {
                onDuplicateNode(
                  target.nodeId,
                );
              }
              onClose();
            }}
          >
            <Copy className="size-4" />
            Duplica nodo
          </button>

          <button
            type="button"
            role="menuitem"
            className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
            onClick={() => {
              if (
                target.nodeId &&
                onUnlinkNode
              ) {
                onUnlinkNode(
                  target.nodeId,
                );
              }
              onClose();
            }}
          >
            <Unlink className="size-4" />
            Scollega input
          </button>

          <button
            type="button"
            role="menuitem"
            className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left text-destructive transition hover:bg-muted"
            onClick={() => {
              if (
                target.nodeId &&
                onRemoveNode
              ) {
                onRemoveNode(
                  target.nodeId,
                );
              }
              onClose();
            }}
          >
            <Trash2 className="size-4" />
            Elimina nodo
          </button>

          <span className="my-1 block h-px bg-border" />
        </>
      ) : null}

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onAddDataset();
          onClose();
        }}
      >
        <Plus className="size-4" />
        Aggiungi dataset
      </button>

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onAutoLayout();
          onClose();
        }}
      >
        <LayoutGrid className="size-4" />
        Disponi automaticamente
      </button>

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onFitView();
          onClose();
        }}
      >
        <Maximize2 className="size-4" />
        Adatta vista
      </button>

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onToggleGrid();
          onClose();
        }}
      >
        <Grid3x3 className="size-4" />
        {grid
          ? "Nascondi griglia"
          : "Mostra griglia"}
      </button>
    </div>,
    document.body,
  );
}

=== FILE: src/components/isa/etl/settings-panels/aggregate-panel.tsx ===
import { nodeDef } from "@/lib/etl-catalog";
import { getAggregateFieldSpecs } from "@/lib/etl-node-config";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";
import {
  PanelChipToggleList,
  PanelLabel,
  PanelSelect,
} from "@/components/isa/etl/settings-panels/panel-controls";

/**
 * Pannello impostazioni per la famiglia "aggregate" (fase 4, punto 3):
 * Group By / Aggregate / Pivot condividono lo stesso principio —
 * selettori (singoli o multipli) dallo schema in ingresso reale,
 * invece dei campi di testo libero del catalogo generico.
 */
export function AggregatePanel({
  workflow,
  node,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const fieldSpecs = getAggregateFieldSpecs(
    workflow,
    node,
  );

  const aggregationOptions =
    nodeDef(node.type)?.fields.find(
      (field) =>
        field.key === "aggregation",
    )?.options ?? [];

  return (
    <div className="space-y-3">
      {fieldSpecs.map((spec) => {
        if (spec.kind === "multi") {
          const selected = (
            node.config[spec.key] ?? ""
          )
            .split(",")
            .map((v) => v.trim())
            .filter(Boolean);

          const values =
            spec.options.map(
              (c) => c.name,
            );

          return (
            <div key={spec.key}>
              <PanelLabel>
                {spec.label}
              </PanelLabel>

              <PanelChipToggleList
                values={values}
                selected={selected}
                onToggle={(value) => {
                  const next =
                    selected.includes(
                      value,
                    )
                      ? selected.filter(
                          (v) =>
                            v !==
                            value,
                        )
                      : [
                          ...selected,
                          value,
                        ];

                  onConfigChange({
                    [spec.key]:
                      next.join(
                        ", ",
                      ),
                  });
                }}
              />
            </div>
          );
        }

        return (
          <div key={spec.key}>
            <PanelLabel>
              {spec.label}
            </PanelLabel>

            <PanelSelect
              value={
                node.config[
                  spec.key
                ] ?? ""
              }
              onChange={(value) =>
                onConfigChange({
                  [spec.key]: value,
                })
              }
              options={spec.options}
            />
          </div>
        );
      })}

      {aggregationOptions.length > 0 && (
        <div>
          <PanelLabel>
            Funzione
          </PanelLabel>

          <select
            value={
              node.config[
                "aggregation"
              ] ?? ""
            }
            onChange={(event) =>
              onConfigChange({
                aggregation:
                  event.target
                    .value,
              })
            }
            className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
          >
            {aggregationOptions.map(
              (option) => (
                <option
                  key={option}
                  value={option}
                >
                  {option}
                </option>
              ),
            )}
          </select>
        </div>
      )}
    </div>
  );
}


=== FILE: src/components/isa/etl/settings-panels/combine-panel.tsx ===
import { nodeDef } from "@/lib/etl-catalog";
import {
  getCombineInputs,
  getFilterColumnOptions,
  getJoinTypeOptions,
  joinPairConfigKeys,
} from "@/lib/etl-node-config";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";
import {
  PanelEmptyState,
  PanelLabel,
  PanelSelect,
} from "@/components/isa/etl/settings-panels/panel-controls";

/**
 * Pannello impostazioni per la famiglia "combine" (fase 4, punto 2).
 * Il contenuto dipende dal tipo di nodo combine (join/union/lookup —
 * modalità GIA' esistenti come EtlNodeDef distinti nel catalogo,
 * nessun nuovo tipo introdotto qui): la modalità "join" ottiene il
 * form dinamico completo richiesto dal brief; union/lookup restano
 * con i loro campi esistenti, resi dinamici dove lo schema lo
 * permette, così tutte le modalità della famiglia condividono lo
 * stesso pannello invece di UI separate e duplicate.
 */
export function CombinePanel({
  workflow,
  node,
  memberIds,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  /** Id dei membri della bubble del nodo (fase 2), incluso `node.id`; assente/singolo = nodo non raggruppato. */
  memberIds?: readonly string[] | undefined;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  if (node.type === "combine.join") {
    return (
      <JoinFields
        workflow={workflow}
        node={node}
        memberIds={memberIds}
        onConfigChange={onConfigChange}
      />
    );
  }

  if (node.type === "combine.lookup") {
    return (
      <LookupFields
        workflow={workflow}
        node={node}
        onConfigChange={onConfigChange}
      />
    );
  }

  return (
    <UnionFields
      node={node}
      onConfigChange={onConfigChange}
    />
  );
}

function JoinFields({
  workflow,
  node,
  memberIds,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  memberIds?: readonly string[] | undefined;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const inputs = getCombineInputs(
    workflow,
    node,
    memberIds,
  );

  const joinTypeOptions =
    getJoinTypeOptions();

  const joinType =
    node.config["how"] ||
    joinTypeOptions[0] ||
    "inner";

  if (inputs.length < 2) {
    return (
      <PanelEmptyState>
        Collega almeno 2 dataset per
        configurare la join (attualmente{" "}
        {inputs.length}
        ).
      </PanelEmptyState>
    );
  }

  const pairs = inputs
    .slice(0, -1)
    .map((left, index) => ({
      index,
      left,
      right: inputs[index + 1]!,
    }));

  return (
    <div className="space-y-3">
      <div>
        <PanelLabel>
          Tipo di join
        </PanelLabel>

        <select
          value={joinType}
          onChange={(event) =>
            onConfigChange({
              how: event.target
                .value,
            })
          }
          className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
        >
          {joinTypeOptions.map(
            (option) => (
              <option
                key={option}
                value={option}
              >
                {option}
              </option>
            ),
          )}
        </select>
      </div>

      {pairs.map(
        ({ index, left, right }) => {
          const keys =
            joinPairConfigKeys(index);

          return (
            <div
              key={`${left.nodeId}-${right.nodeId}`}
              className="glass-chip rounded-xl p-2.5"
            >
              <span className="block truncate text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                {left.title} ⋈{" "}
                {right.title}
              </span>

              <div className="mt-1.5 grid grid-cols-2 gap-2">
                <div>
                  <PanelLabel>
                    {left.title}
                  </PanelLabel>

                  <PanelSelect
                    value={
                      node.config[
                        keys.left
                      ] ?? ""
                    }
                    onChange={(
                      value,
                    ) =>
                      onConfigChange({
                        [keys.left]:
                          value,
                      })
                    }
                    options={
                      left.columns
                    }
                  />
                </div>

                <div>
                  <PanelLabel>
                    {right.title}
                  </PanelLabel>

                  <PanelSelect
                    value={
                      node.config[
                        keys.right
                      ] ?? ""
                    }
                    onChange={(
                      value,
                    ) =>
                      onConfigChange({
                        [keys.right]:
                          value,
                      })
                    }
                    options={
                      right.columns
                    }
                  />
                </div>
              </div>
            </div>
          );
        },
      )}
    </div>
  );
}

function LookupFields({
  workflow,
  node,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const inputColumns =
    getFilterColumnOptions(
      workflow,
      node,
    );

  return (
    <div className="space-y-3">
      <div>
        <PanelLabel>
          Chiave di lookup
        </PanelLabel>

        <PanelSelect
          value={
            node.config["key"] ?? ""
          }
          onChange={(value) =>
            onConfigChange({
              key: value,
            })
          }
          options={inputColumns}
        />
      </div>

      <div>
        <PanelLabel>
          Nuova colonna da aggiungere
        </PanelLabel>

        <input
          value={
            node.config["column"] ??
            ""
          }
          placeholder="category"
          onChange={(event) =>
            onConfigChange({
              column:
                event.target.value,
            })
          }
          className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
        />
      </div>
    </div>
  );
}

function UnionFields({
  node,
  onConfigChange,
}: {
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const modeOptions =
    nodeDef("combine.union")?.fields.find(
      (field) => field.key === "mode",
    )?.options ?? [];

  return (
    <div>
      <PanelLabel>
        Modalità
      </PanelLabel>

      <select
        value={
          node.config["mode"] ?? ""
        }
        onChange={(event) =>
          onConfigChange({
            mode: event.target.value,
          })
        }
        className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
      >
        {modeOptions.map((option) => (
          <option
            key={option}
            value={option}
          >
            {option}
          </option>
        ))}
      </select>
    </div>
  );
}


=== FILE: src/components/isa/etl/settings-panels/filter-panel.tsx ===
import {
  getColumnDomainValues,
  getFilterColumnOptions,
} from "@/lib/etl-node-config";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";
import {
  PanelChipToggleList,
  PanelLabel,
  PanelSelect,
} from "@/components/isa/etl/settings-panels/panel-controls";

/**
 * Pannello impostazioni per i nodi "Filter" (fase 4, punto 1):
 * colonna dallo schema in ingresso reale, output "solo colonna" vs
 * "intero dataset", valori da multi-select sul dominio della colonna
 * — mai testo libero.
 *
 * Config scritta: `column` (esistente, riusata), `value` (nuovo
 * significato: elenco dei valori selezionati, comma-joined — resta
 * compatibile con `nodeSummary`/Inspector, che lo trattano già come
 * stringa opaca), `outputMode` ("column" | "dataset", nuova chiave).
 */
export function FilterPanel({
  workflow,
  node,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const columns = getFilterColumnOptions(
    workflow,
    node,
  );

  const selectedColumnName =
    node.config["column"] ?? "";

  const selectedColumn = columns.find(
    (c) => c.name === selectedColumnName,
  );

  const outputMode =
    node.config["outputMode"] === "column"
      ? "column"
      : "dataset";

  const selectedValues = (
    node.config["value"] ?? ""
  )
    .split(",")
    .map((v) => v.trim())
    .filter(Boolean);

  const domainValues = selectedColumn
    ? getColumnDomainValues(
        selectedColumn,
        `${node.id}:${selectedColumn.name}`,
      )
    : [];

  const toggleValue = (value: string) => {
    const next = selectedValues.includes(
      value,
    )
      ? selectedValues.filter(
          (v) => v !== value,
        )
      : [...selectedValues, value];

    onConfigChange({
      value: next.join(", "),
    });
  };

  return (
    <div className="space-y-3">
      <div>
        <PanelLabel>
          Colonna da filtrare
        </PanelLabel>

        <PanelSelect
          value={selectedColumnName}
          onChange={(value) =>
            onConfigChange({
              column: value,
              /* Cambiare colonna invalida i valori scelti sulla precedente. */
              value: "",
            })
          }
          options={columns}
          placeholder={
            columns.length
              ? "Seleziona colonna…"
              : "Nessuno schema in ingresso"
          }
        />
      </div>

      <div>
        <PanelLabel>
          Output
        </PanelLabel>

        <div className="mt-1.5 flex gap-1.5">
          {(
            [
              {
                key: "dataset",
                label: "Intero dataset filtrato",
              },
              {
                key: "column",
                label: "Solo quella colonna",
              },
            ] as const
          ).map((option) => (
            <button
              key={option.key}
              type="button"
              aria-pressed={
                outputMode ===
                option.key
              }
              onClick={() =>
                onConfigChange({
                  outputMode:
                    option.key,
                })
              }
              className={`flex-1 rounded-xl border px-2.5 py-1.5 text-[11px] font-medium transition ${
                outputMode ===
                option.key
                  ? "border-transparent gradient-brand text-brand-foreground"
                  : "glass-chip border-transparent text-muted-foreground hover:text-foreground"
              }`}
            >
              {option.label}
            </button>
          ))}
        </div>
      </div>

      <div>
        <PanelLabel>
          Valori da mantenere
        </PanelLabel>

        {selectedColumn ? (
          <PanelChipToggleList
            values={domainValues}
            selected={selectedValues}
            onToggle={toggleValue}
          />
        ) : (
          <p className="mt-1.5 text-[11px] text-muted-foreground">
            Scegli prima una colonna.
          </p>
        )}
      </div>
    </div>
  );
}


=== FILE: src/components/isa/etl/settings-panels/panel-controls.tsx ===
import { Check } from "lucide-react";
import type { ColumnDef } from "@/lib/etl-catalog";

/**
 * Controlli condivisi dai pannelli impostazioni della fase 4
 * (FilterPanel, CombineJoinPanel, AggregatePanel): stile coerente con
 * il resto del popover (`glass-chip`), niente testo libero dove il
 * brief chiede selettori guidati dallo schema.
 */

export function PanelLabel({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <span className="block text-[11px] font-medium text-muted-foreground">
      {children}
    </span>
  );
}

export function PanelSelect({
  value,
  onChange,
  options,
  placeholder = "Seleziona colonna…",
  disabled,
}: {
  value: string;
  onChange: (value: string) => void;
  options: ColumnDef[];
  placeholder?: string;
  disabled?: boolean;
}) {
  return (
    <select
      value={value}
      disabled={disabled}
      onChange={(event) =>
        onChange(event.target.value)
      }
      className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring disabled:opacity-50"
    >
      <option value="">
        {placeholder}
      </option>
      {options.map((column) => (
        <option
          key={column.name}
          value={column.name}
        >
          {column.name} · {column.type}
        </option>
      ))}
    </select>
  );
}

/** Lista di chip selezionabili (multi-select) invece di testo libero. */
export function PanelChipToggleList({
  values,
  selected,
  onToggle,
}: {
  values: string[];
  selected: string[];
  onToggle: (value: string) => void;
}) {
  if (values.length === 0) {
    return (
      <p className="mt-1.5 text-[11px] text-muted-foreground">
        Nessun valore disponibile.
      </p>
    );
  }

  return (
    <div className="mt-1.5 flex flex-wrap gap-1.5">
      {values.map((value) => {
        const isSelected =
          selected.includes(value);

        return (
          <button
            key={value}
            type="button"
            onClick={() =>
              onToggle(value)
            }
            aria-pressed={isSelected}
            className={`flex items-center gap-1 rounded-full border px-2.5 py-1 text-[11px] transition ${
              isSelected
                ? "border-transparent gradient-brand text-brand-foreground"
                : "glass-chip border-transparent text-muted-foreground hover:text-foreground"
            }`}
          >
            {isSelected && (
              <Check className="size-3" />
            )}
            {value}
          </button>
        );
      })}
    </div>
  );
}

export function PanelEmptyState({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <p className="px-1 py-2 text-[11px] leading-relaxed text-muted-foreground">
      {children}
    </p>
  );
}


=== FILE: src/components/isa/etl/tool-palette.tsx ===
import {
  AlignEndHorizontal,
  AlignEndVertical,
  AlignStartHorizontal,
  AlignStartVertical,
  Boxes,
  GripVertical,
  List,
  Shrink,
} from "lucide-react";
import { useCallback, useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";
import { IsaMenu, IsaMenuItem } from "@/components/isa/ui/isa-menu";
import type { EtlCategory, EtlNodeDef } from "@/lib/etl-catalog";
import { ETL_CATEGORIES, ETL_NODES, categoryAccent } from "@/lib/etl-catalog";
import { CATEGORY_ICONS } from "@/lib/etl-display";

export type Dock = "top" | "bottom" | "left" | "right";

/*
 * Bug 1.1: sotto una soglia minima di spostamento un pointerdown+up
 * sul chip è un click/doppio click, non un drag — evita di far
 * scattare un fantasma di drag per una semplice pressione.
 */
const RESOURCE_DRAG_THRESHOLD = 4;

type ResourceDrag = {
  node: EtlNodeDef;
  x: number;
  y: number;
};

function NodeChip({
  node,
  onAdd,
  onDragStart,
}: {
  node: EtlNodeDef;
  /** Doppio click (PARTE B): niente più singolo click per aggiungere. */
  onAdd: (type: string, clientX: number, clientY: number) => void;
  /**
   * Bug 1.1: avvio del drag-to-canvas via Pointer Events, non più HTML5
   * Drag & Drop (in conflitto con i Pointer Events usati ovunque altro
   * sul canvas — card, palette, porte). Lo stato del drag vive nel
   * genitore ToolPalette (un solo ghost alla volta, coerente con la
   * palette stessa).
   */
  onDragStart: (
    node: EtlNodeDef,
    event: React.PointerEvent<HTMLButtonElement>,
  ) => void;
}) {
  const { label, description, Icon, category } = node;

  const handleDoubleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
    event.stopPropagation();
    onAdd(node.type, event.clientX, event.clientY);
  };

  return (
    <button
      type="button"
      onPointerDown={(event) => onDragStart(node, event)}
      onDoubleClick={handleDoubleClick}
      title={description}
      aria-label={`Aggiungi ${label}`}
      className="flex shrink-0 select-none items-center gap-1.5 rounded-lg px-2.5 py-1.5 text-left text-[12px] font-medium text-muted-foreground transition hover:bg-muted hover:text-foreground"
      style={{
        userSelect: "none",
        touchAction: "none",
      }}
    >
      <span
        className={`${categoryAccent(category)} flex size-5 shrink-0 items-center justify-center rounded-md`}
      >
        <Icon className="pointer-events-none size-3" />
      </span>

      <span className="pointer-events-none truncate">{label}</span>
    </button>
  );
}

export function ToolPalette({
  dock,
  onDockChange,
  onAdd,
  onDragAdd,
}: {
  dock: Dock;
  onDockChange: (dock: Dock) => void;
  /** Doppio click su un NodeChip (PARTE B): coordinate schermo del click, la conversione in coordinate canvas la fa WorkflowCanvas. */
  onAdd: (type: string, clientX: number, clientY: number) => void;
  /**
   * Bug 1.1: drag-and-drop di un NodeChip sul canvas, coordinate
   * schermo del punto di rilascio (stessa conversione di `onAdd`, fatta
   * da WorkflowCanvas). Chiamato solo se il rilascio avviene
   * effettivamente sopra il canvas.
   */
  onDragAdd: (type: string, clientX: number, clientY: number) => void;
}) {
  const [open, setOpen] = useState<EtlCategory | null>(null);
  const [expanded, setExpanded] = useState(false);
  const [ghost, setGhost] = useState<{ x: number; y: number } | null>(null);
  const [resourceDrag, setResourceDrag] = useState<ResourceDrag | null>(null);
  const ref = useRef<HTMLDivElement>(null);
  const workspaceRef = useRef<HTMLElement | null>(null);
  const vertical = dock === "left" || dock === "right";
  const secondLevel = expanded || open !== null;
  const dragging = ghost !== null;

  useEffect(() => {
    workspaceRef.current =
      ref.current?.closest<HTMLElement>("[data-palette-workspace]") ?? null;
  }, []);

  const startResourceDrag = useCallback(
    (node: EtlNodeDef, event: React.PointerEvent<HTMLButtonElement>) => {
      if (event.button !== 0) return;

      const startX = event.clientX;
      const startY = event.clientY;
      let dragStarted = false;

      const move = (pointer: PointerEvent) => {
        if (!dragStarted) {
          const moved = Math.hypot(pointer.clientX - startX, pointer.clientY - startY);
          if (moved < RESOURCE_DRAG_THRESHOLD) return;
          dragStarted = true;
        }
        setResourceDrag({ node, x: pointer.clientX, y: pointer.clientY });
      };

      const cleanup = () => {
        window.removeEventListener("pointermove", move);
        window.removeEventListener("pointerup", finish);
        window.removeEventListener("pointercancel", cleanup);
      };

      const finish = (pointer: PointerEvent) => {
        cleanup();
        setResourceDrag(null);

        if (!dragStarted) return;

        // Il rilascio conta solo se avviene sopra il canvas: fuori da
        // lì (es. sulla stessa palette) il drag viene semplicemente
        // annullato, coerente col comportamento di un vero drag & drop.
        const dropTarget = document.elementFromPoint(pointer.clientX, pointer.clientY);
        if (dropTarget?.closest("[data-palette-workspace]")) {
          onDragAdd(node.type, pointer.clientX, pointer.clientY);
        }
      };

      window.addEventListener("pointermove", move);
      window.addEventListener("pointerup", finish);
      window.addEventListener("pointercancel", cleanup);
    },
    [onDragAdd],
  );

  const startDrag = (event: React.PointerEvent) => {
    event.preventDefault();
    event.stopPropagation();
    const workspace = ref.current?.closest<HTMLElement>("[data-palette-workspace]");
    if (!workspace) return;

    setGhost({ x: event.clientX, y: event.clientY });

    const move = (pointer: PointerEvent) => {
      setGhost({ x: pointer.clientX, y: pointer.clientY });
    };
    const finish = (pointer: PointerEvent) => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", finish);
      setGhost(null);
      const rect = workspace.getBoundingClientRect();
      const distances: Array<[Dock, number]> = [
        ["left", Math.abs(pointer.clientX - rect.left)],
        ["right", Math.abs(rect.right - pointer.clientX)],
        ["top", Math.abs(pointer.clientY - rect.top)],
        ["bottom", Math.abs(rect.bottom - pointer.clientY)],
      ];
      distances.sort((a, b) => a[1] - b[1]);
      const nearest = distances[0]?.[0];
      if (nearest && nearest !== dock) onDockChange(nearest);
    };
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", finish);
  };

  const categories = expanded
    ? ETL_CATEGORIES
    : ETL_CATEGORIES.filter((category) => category.key === open);

  const mainBar = (
    <div
      className={`glass-panel flex shrink-0 items-center gap-1 rounded-full p-1.5 shadow-lg transition-all duration-200 ${
        vertical ? "flex-col" : "flex-row"
      } ${dragging ? "scale-75 opacity-30" : "animate-scale-in"}`}
    >
      <button
        type="button"
        onPointerDown={startDrag}
        title="Trascina verso un bordo"
        aria-label="Sposta la barra delle risorse"
        className="flex size-8 cursor-grab items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground active:cursor-grabbing"
        style={{ touchAction: "none" }}
      >
        <GripVertical className={`size-4 ${vertical ? "" : "rotate-90"}`} />
      </button>

      {ETL_CATEGORIES.map((category) => {
        const Icon = CATEGORY_ICONS[category.key];
        const active = !expanded && open === category.key;
        return (
          <button
            key={category.key}
            type="button"
            onClick={() => {
              setExpanded(false);
              setOpen((current) => (current === category.key ? null : category.key));
            }}
            title={category.label}
            aria-label={category.label}
            aria-pressed={active}
            className={`flex size-8 items-center justify-center rounded-full transition ${
              active ? `${categoryAccent(category.key)} scale-105` : "text-muted-foreground hover:text-foreground"
            }`}
          >
            <Icon className="size-4" />
          </button>
        );
      })}

      <button
        type="button"
        onClick={() => {
          setExpanded((value) => !value);
          setOpen(null);
        }}
        aria-pressed={expanded}
        title={expanded ? "Riduci la barra" : "Espandi tutte le risorse"}
        aria-label={expanded ? "Riduci la barra" : "Espandi tutte le risorse"}
        className={`flex size-8 items-center justify-center rounded-full transition ${
          expanded ? "text-foreground" : "text-muted-foreground hover:text-foreground"
        }`}
      >
        {expanded ? <Shrink className="size-4" /> : <List className="size-4" />}
      </button>

      <span className={`bg-border ${vertical ? "my-0.5 h-px w-4" : "mx-0.5 h-4 w-px"}`} />

      <IsaMenu
        label="Posizione barra"
        Icon={vertical ? AlignStartVertical : AlignStartHorizontal}
        boundaryRef={workspaceRef}
        placement="auto"
      >
        {(close) => (
          <>
            <IsaMenuItem Icon={AlignStartHorizontal} label="Aggancia in alto" onClick={() => { onDockChange("top"); close(); }} />
            <IsaMenuItem Icon={AlignEndHorizontal} label="Aggancia in basso" onClick={() => { onDockChange("bottom"); close(); }} />
            <IsaMenuItem Icon={AlignStartVertical} label="Aggancia a sinistra" onClick={() => { onDockChange("left"); close(); }} />
            <IsaMenuItem Icon={AlignEndVertical} label="Aggancia a destra" onClick={() => { onDockChange("right"); close(); }} />
          </>
        )}
      </IsaMenu>
    </div>
  );

  const resources = secondLevel && !dragging ? (
    <div
      className={`glass-panel scroll-slim min-h-0 min-w-0 rounded-2xl p-1.5 shadow-lg ${
        vertical ? "max-h-[26rem] overflow-y-auto" : "max-w-full overflow-x-auto"
      }`}
    >
      <div className={vertical ? "flex flex-col gap-2" : "flex min-w-max items-start gap-3"}>
        {categories.map((category) => {
          const CategoryIcon = CATEGORY_ICONS[category.key];
          const nodes = ETL_NODES.filter((node) => node.category === category.key);
          return (
            <div key={category.key} className="min-w-0">
              {expanded && (
                <span className="flex items-center gap-1.5 px-2.5 py-1 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                  <CategoryIcon className="size-3" />
                  {category.label}
                </span>
              )}
              <div className={vertical ? "flex flex-col gap-0.5" : "flex items-center gap-0.5"}>
                {nodes.map((node) => (
                  <NodeChip
                    key={node.type}
                    node={node}
                    onAdd={onAdd}
                    onDragStart={startResourceDrag}
                  />
                ))}
              </div>
            </div>
          );
        })}
      </div>
    </div>
  ) : null;

  const reverse = dock === "bottom" || dock === "right";

  return (
    <>
      <aside
        key={`${dock}-${String(secondLevel)}`}
        ref={ref}
        aria-label="Risorse ETL"
        onPointerDown={(event) => event.stopPropagation()}
        className={`relative z-40 flex shrink-0 items-center justify-center gap-1.5 ${
          vertical
            ? `${reverse ? "flex-row-reverse" : "flex-row"} max-w-[19rem]`
            : `${reverse ? "flex-col-reverse" : "flex-col"} max-w-full`
        } p-1.5`}
      >
        {mainBar}
        {resources}
      </aside>

      {ghost &&
        createPortal(
          <div
            aria-hidden
            className="glass-panel animate-scale-in pointer-events-none fixed z-50 flex size-11 items-center justify-center rounded-full shadow-xl"
            style={{ left: ghost.x - 22, top: ghost.y - 22 }}
          >
            <Boxes className="size-4 text-muted-foreground" />
          </div>,
          document.body,
        )}

      {resourceDrag &&
        createPortal(
          <div
            aria-hidden
            className="glass-panel animate-scale-in pointer-events-none fixed z-50 flex items-center gap-1.5 rounded-lg px-2.5 py-1.5 text-[12px] font-medium text-foreground shadow-xl"
            style={{ left: resourceDrag.x + 14, top: resourceDrag.y + 14 }}
          >
            <span
              className={`${categoryAccent(resourceDrag.node.category)} flex size-5 shrink-0 items-center justify-center rounded-md`}
            >
              <resourceDrag.node.Icon className="size-3" />
            </span>
            <span className="truncate">{resourceDrag.node.label}</span>
          </div>,
          document.body,
        )}
    </>
  );
}


=== FILE: src/components/isa/ui/isa-menu.tsx ===
import type { Pencil } from "lucide-react";
import {
  Check,
  MoreHorizontal,
} from "lucide-react";
import { createPortal } from "react-dom";
import {
  useCallback,
  useEffect,
  useLayoutEffect,
  useRef,
  useState,
} from "react";
import {
  PRESS_SCALE,
  PRESS_TRANSITION,
  SPRING_CLOSE_DURATION_MS,
  SPRING_CLOSE_TRANSITION,
  SPRING_ORIGIN_SCALE,
  SPRING_OPEN_TRANSITION,
} from "@/lib/etl-motion";

type MenuSide =
  | "top"
  | "bottom"
  | "left"
  | "right";

type MenuPlacement =
  | "auto"
  | MenuSide;

type MenuPosition = {
  top: number;
  left: number;
};

const MENU_WIDTH = 208;
const MENU_GAP = 8;
const MENU_MARGIN = 8;

export function IsaMenu({
  label,
  children,
  align = "right",
  Icon = MoreHorizontal,
  triggerClassName = "",
  boundaryRef,
  placement = "bottom",
  variant = "chip",
  width = MENU_WIDTH,
  onOpenChange,
}: {
  label: string;
  children: (
    close: () => void,
  ) => React.ReactNode;
  align?: "left" | "right";
  Icon?: typeof Pencil;
  triggerClassName?: string;
  boundaryRef?: React.RefObject<
    HTMLElement | null
  >;
  placement?: MenuPlacement;
  variant?: "chip" | "bare";
  /** Larghezza del popover in px (default MENU_WIDTH) — es. per i pannelli impostazioni della fase 4, più larghi di un menu azioni. */
  width?: number | undefined;
  /**
   * Notifica il chiamante quando il menu si apre/chiude — usato da chi
   * ospita il trigger (es. una card nodo) per applicare il proprio
   * feedback "pressed" all'intero contenitore, non solo al bottone
   * icona (punto 2.1 del redesign iOS).
   */
  onOpenChange?: (
    open: boolean,
  ) => void;
}) {
  const [open, setOpen] =
    useState(false);

  /*
   * `rendered` tiene il popover nel DOM anche durante la chiusura, per
   * poter animare il rientro verso il trigger invece di sparire di
   * scatto; `visible` guida la transizione scale/opacity vera e propria
   * e viene alzata un frame dopo il mount (serve un primo commit con lo
   * stato "piccolo/trasparente" prima di animare verso quello finale).
   */
  const [rendered, setRendered] =
    useState(false);

  const [visible, setVisible] =
    useState(false);

  const [side, setSide] =
    useState<MenuSide>("bottom");

  const [position, setPosition] =
    useState<MenuPosition | null>(
      null,
    );

  const rootRef =
    useRef<HTMLDivElement>(null);

  const triggerRef =
    useRef<HTMLButtonElement>(null);

  const menuRef =
    useRef<HTMLDivElement>(null);

  const closeTimeoutRef =
    useRef<number | null>(null);

  const constrained =
    Boolean(
      boundaryRef &&
        placement === "auto",
    );

  const clearCloseTimeout =
    useCallback(() => {
      if (
        closeTimeoutRef.current !==
        null
      ) {
        window.clearTimeout(
          closeTimeoutRef.current,
        );
        closeTimeoutRef.current =
          null;
      }
    }, []);

  const close = useCallback(
    () => {
      setOpen(false);
      onOpenChange?.(false);
    },
    [onOpenChange],
  );

  /*
   * Sequenza di chiusura: appena `open` torna false il popover resta
   * montato (`rendered`) ma `visible` si abbassa, innescando la
   * transizione di rientro verso il trigger; solo al termine di quella
   * transizione lo smontiamo davvero e liberiamo `position`. Se il
   * trigger viene ripremuto durante il rientro, `clearCloseTimeout` (nel
   * toggle) annulla lo smontaggio pendente.
   */
  useEffect(() => {
    if (open || !rendered) {
      return;
    }

    setVisible(false);
    clearCloseTimeout();

    closeTimeoutRef.current =
      window.setTimeout(() => {
        setRendered(false);
        setPosition(null);
        closeTimeoutRef.current =
          null;
      }, SPRING_CLOSE_DURATION_MS);

    return clearCloseTimeout;
  }, [
    open,
    rendered,
    clearCloseTimeout,
  ]);

  const calculatePosition =
    useCallback(() => {
      if (!constrained) {
        return;
      }

      const trigger =
        triggerRef.current;

      const menu =
        menuRef.current;

      const boundary =
        boundaryRef?.current;

      if (
        !trigger ||
        !menu ||
        !boundary
      ) {
        return;
      }

      const triggerRect =
        trigger.getBoundingClientRect();

      const menuRect =
        menu.getBoundingClientRect();

      const boundaryRect =
        boundary.getBoundingClientRect();

      const menuWidth =
        menuRect.width ||
        width;

      const menuHeight =
        menuRect.height;

      const available: Record<
        MenuSide,
        number
      > = {
        top:
          triggerRect.top -
          boundaryRect.top -
          MENU_GAP -
          MENU_MARGIN,

        bottom:
          boundaryRect.bottom -
          triggerRect.bottom -
          MENU_GAP -
          MENU_MARGIN,

        left:
          triggerRect.left -
          boundaryRect.left -
          MENU_GAP -
          MENU_MARGIN,

        right:
          boundaryRect.right -
          triggerRect.right -
          MENU_GAP -
          MENU_MARGIN,
      };

      const fits: Record<
        MenuSide,
        boolean
      > = {
        top:
          available.top >=
          menuHeight,

        bottom:
          available.bottom >=
          menuHeight,

        left:
          available.left >=
          menuWidth,

        right:
          available.right >=
          menuWidth,
      };

      /*
       * Ordine preferenziale:
       * prima sotto, poi sopra, poi lato destro/sinistro.
       *
       * Se un lato non ha spazio sufficiente,
       * viene provato il successivo.
       */
      const order: MenuSide[] = [
        "bottom",
        "top",
        "right",
        "left",
      ];

      let side:
        | MenuSide
        | null = null;

      for (
        const candidate of order
      ) {
        if (fits[candidate]) {
          side = candidate;
          break;
        }
      }

      /*
       * Se il Canvas è troppo piccolo per contenere
       * il menu interamente su un lato, scegliamo il lato
       * con più spazio e facciamo un clamp finale.
       */
      if (!side) {
        const fallback =
          (
            Object.keys(
              available,
            ) as MenuSide[]
          ).sort(
            (a, b) =>
              available[b] -
              available[a],
          );

        side =
          fallback[0] ??
          "bottom";
      }

      let left =
        triggerRect.right -
        menuWidth;

      let top =
        triggerRect.bottom +
        MENU_GAP;

      if (
        side === "bottom"
      ) {
        top =
          triggerRect.bottom +
          MENU_GAP;

        left =
          align === "left"
            ? triggerRect.left
            : triggerRect.right -
              menuWidth;
      }

      if (
        side === "top"
      ) {
        top =
          triggerRect.top -
          menuHeight -
          MENU_GAP;

        left =
          align === "left"
            ? triggerRect.left
            : triggerRect.right -
              menuWidth;
      }

      if (
        side === "right"
      ) {
        left =
          triggerRect.right +
          MENU_GAP;

        top =
          triggerRect.top;
      }

      if (
        side === "left"
      ) {
        left =
          triggerRect.left -
          menuWidth -
          MENU_GAP;

        top =
          triggerRect.top;
      }

      const minLeft =
        boundaryRect.left +
        MENU_MARGIN;

      const maxLeft =
        Math.max(
          minLeft,
          boundaryRect.right -
            menuWidth -
            MENU_MARGIN,
        );

      const minTop =
        boundaryRect.top +
        MENU_MARGIN;

      const maxTop =
        Math.max(
          minTop,
          boundaryRect.bottom -
            menuHeight -
            MENU_MARGIN,
        );

      left = Math.min(
        Math.max(left, minLeft),
        maxLeft,
      );

      top = Math.min(
        Math.max(top, minTop),
        maxTop,
      );

      setPosition({
        left,
        top,
      });
    }, [
      align,
      boundaryRef,
      constrained,
      width,
    ]);

  useLayoutEffect(() => {
    if (
      !open ||
      !constrained
    ) {
      return;
    }

    calculatePosition();

    const frame =
      requestAnimationFrame(
        calculatePosition,
      );

    return () =>
      cancelAnimationFrame(
        frame,
      );
  }, [
    open,
    constrained,
    calculatePosition,
  ]);

  useEffect(() => {
    if (!open) {
      return;
    }

    const handlePointerDown =
      (event: PointerEvent) => {
        const target =
          event.target as Node;

        const insideTrigger =
          rootRef.current?.contains(
            target,
          );

        const insideMenu =
          menuRef.current?.contains(
            target,
          );

        if (
          !insideTrigger &&
          !insideMenu
        ) {
          close();
        }
      };

    const handleKeyDown =
      (event: KeyboardEvent) => {
        if (
          event.key === "Escape"
        ) {
          close();
        }
      };

    document.addEventListener(
      "pointerdown",
      handlePointerDown,
    );

    document.addEventListener(
      "keydown",
      handleKeyDown,
    );

    return () => {
      document.removeEventListener(
        "pointerdown",
        handlePointerDown,
      );

      document.removeEventListener(
        "keydown",
        handleKeyDown,
      );
    };
  }, [
    open,
    close,
  ]);

  useEffect(() => {
    if (
      !open ||
      !constrained
    ) {
      return;
    }

    const reposition =
      () => {
        calculatePosition();
      };

    window.addEventListener(
      "resize",
      reposition,
    );

    window.addEventListener(
      "scroll",
      reposition,
      true,
    );

    return () => {
      window.removeEventListener(
        "resize",
        reposition,
      );

      window.removeEventListener(
        "scroll",
        reposition,
        true,
      );
    };
  }, [
    open,
    constrained,
    calculatePosition,
  ]);

  const menu = open ? (
    <div
      ref={menuRef}
      role="menu"
      onPointerDown={(event) =>
        event.stopPropagation()
      }
      className={
        constrained
          ? "fixed z-[80] overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl"
          : `absolute ${
              align === "right"
                ? "right-0"
                : "left-0"
            } top-10 z-30 overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl`
      }
      style={
        constrained
          ? {
              width,
              left:
                position?.left ??
                -10000,

              top:
                position?.top ??
                -10000,

              visibility:
                position
                  ? "visible"
                  : "hidden",
            }
          : { width }
      }
    >
      {children(close)}
    </div>
  ) : null;

  return (
    <div
      ref={rootRef}
      className="relative"
    >
      <button
        ref={triggerRef}
        type="button"
        onClick={() => {
          if (open) {
            close();
          } else {
            setOpen(true);
          }
        }}
        aria-label={label}
        aria-expanded={open}
        className={`flex size-8 items-center justify-center transition hover:text-foreground ${
          variant === "chip"
            ? "glass-chip rounded-full text-muted-foreground"
            : "text-muted-foreground"
        } ${triggerClassName}`}
      >
        <Icon className="size-4" />
      </button>

      {constrained &&
      typeof document !==
        "undefined"
        ? createPortal(
            menu,
            document.body,
          )
        : menu}
    </div>
  );
}

export function IsaMenuItem({
  Icon,
  label,
  onClick,
  danger,
}: {
  Icon: typeof Pencil;
  label: string;
  onClick: () => void;
  danger?: boolean;
}) {
  return (
    <button
      type="button"
      onClick={onClick}
      className={`flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted ${
        danger
          ? "text-destructive"
          : "text-foreground"
      }`}
    >
      <Icon className="size-4" />
      {label}
    </button>
  );
}

export function IsaMenuCheckItem({
  label,
  checked,
  onToggle,
}: {
  label: string;
  checked: boolean;
  onToggle: () => void;
}) {
  return (
    <button
      type="button"
      role="menuitemcheckbox"
      aria-checked={checked}
      onClick={onToggle}
      className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left text-foreground transition hover:bg-muted"
    >
      <span
        className={`flex size-4 shrink-0 items-center justify-center rounded-[5px] border ${
          checked
            ? "gradient-brand border-transparent text-brand-foreground"
            : "border-border"
        }`}
      >
        {checked && (
          <Check className="size-3" />
        )}
      </span>

      <span className="min-w-0 flex-1 truncate text-[13px]">
        {label}
      </span>
    </button>
  );
}

=== FILE: src/lib/etl-bubble.ts ===
/**
 * Geometria delle "bubble" — gruppi di 2+ card transform combinate
 * insieme (`EtlNode.groupId`). Isolato dal componente canvas perché
 * puramente geometrico (nessun accesso a DOM/React) e perché servirà
 * anche alla fase 3 (animazioni di resize/realign della bubble).
 */

export type BubbleOrientation = "horizontal" | "vertical";

/** Sottoinsieme dei campi di NodeGeometry che servono per la geometria della bubble. */
export type BubbleMember = {
  id: string;
  x: number;
  y: number;
  width: number;
  height: number;
};

export type BubbleRect = {
  x: number;
  y: number;
  width: number;
  height: number;
};

export type BubbleGeometry = {
  groupId: string;
  memberIds: string[];
  /** Bounding box dei membri, SENZA il padding visivo del contenitore disegnato. */
  rect: BubbleRect;
  orientation: BubbleOrientation;
};

/** Bounding box che racchiude tutti i `members`. */
export function computeBubbleRect(
  members: readonly BubbleMember[],
): BubbleRect {
  const first = members[0];

  if (!first) {
    return { x: 0, y: 0, width: 0, height: 0 };
  }

  let minX = first.x;
  let minY = first.y;
  let maxX = first.x + first.width;
  let maxY = first.y + first.height;

  for (const member of members) {
    minX = Math.min(minX, member.x);
    minY = Math.min(minY, member.y);
    maxX = Math.max(maxX, member.x + member.width);
    maxY = Math.max(maxY, member.y + member.height);
  }

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}

/**
 * Orientamento della bubble, derivato dal suo bounding box: più larga
 * che alta -> orizzontale (i membri sono affiancati, tipicamente
 * perché l'ultimo è stato trascinato dentro da sinistra/destra), più
 * alta che larga -> verticale (trascinato da sopra/sotto).
 *
 * Nessuno stato persistito: è una funzione pura del bounding box
 * corrente, ricalcolata a ogni render dalla disposizione reale dei
 * nodi — quindi cambia da sola se l'utente riorganizza le card dentro
 * o dentro/fuori dalla bubble, senza bisogno di tracciare la
 * direzione del gesto di drag.
 *
 * Bounding box (quasi) quadrato: nessuna indicazione nel brief su
 * quale verso preferire in questo caso limite — si sceglie
 * orizzontale come default deterministico, per evitare che
 * l'orientamento "sfarfalli" tra i due valori a parità di dimensioni.
 */
export function resolveBubbleOrientation(
  rect: BubbleRect,
): BubbleOrientation {
  return rect.width >= rect.height ? "horizontal" : "vertical";
}

/** Geometria completa della bubble per un gruppo con 2+ membri; `null` altrimenti (non è una bubble). */
export function computeBubbleGeometry(
  groupId: string,
  members: readonly BubbleMember[],
): BubbleGeometry | null {
  if (members.length < 2) {
    return null;
  }

  const rect = computeBubbleRect(members);

  return {
    groupId,
    memberIds: members.map((member) => member.id),
    rect,
    orientation: resolveBubbleOrientation(rect),
  };
}

/** Tutte le bubble presenti in un insieme di nodi, indicizzate per groupId. */
export function computeBubbles(
  nodes: readonly (BubbleMember & { groupId?: string | undefined })[],
): Map<string, BubbleGeometry> {
  const byGroup = new Map<string, BubbleMember[]>();

  for (const node of nodes) {
    if (!node.groupId) {
      continue;
    }

    const members = byGroup.get(node.groupId) ?? [];
    members.push(node);
    byGroup.set(node.groupId, members);
  }

  const bubbles = new Map<string, BubbleGeometry>();

  for (const [groupId, members] of byGroup) {
    const geometry = computeBubbleGeometry(groupId, members);

    if (geometry) {
      bubbles.set(groupId, geometry);
    }
  }

  return bubbles;
}


=== FILE: src/lib/etl-catalog.ts ===
import {
  Braces,
  Calculator,
  Combine,
  Copy,
  Database,
  Eraser,
  FileSpreadsheet,
  Filter,
  Gauge,
  Globe,
  Grid3x3,
  Layers,
  Pencil,
  Save,
  Search,
  Sigma,
  SortAsc,
  Table2,
} from "lucide-react";

export type ColumnType = "string" | "integer" | "decimal" | "date" | "boolean";

export type ColumnDef = { name: string; type: ColumnType; nullable: boolean };

export type EtlFieldKind = "text" | "select" | "textarea";

export type EtlField = {
  key: string;
  label: string;
  kind: EtlFieldKind;
  placeholder?: string | undefined;
  options?: string[] | undefined;
  defaultValue?: string | undefined;
  required?: boolean | undefined;
};

export type EtlCategory =
  | "sources"
  | "transform"
  | "combine"
  | "aggregate"
  | "output";

export type EtlNodeDef = {
  type: string;
  label: string;
  category: EtlCategory;
  description: string;
  Icon: typeof Database;
  inputs: string[];
  outputs: string[];
  fields: EtlField[];
};

export const ETL_CATEGORIES: { key: EtlCategory; label: string }[] = [
  { key: "sources", label: "Sources" },
  { key: "transform", label: "Transform" },
  { key: "combine", label: "Combine" },
  { key: "aggregate", label: "Aggregate" },
  { key: "output", label: "Output" },
];

/** Dataset di riferimento del workspace (metadati, nessuna esecuzione). */
export const SAMPLE_DATASETS: Record<
  string,
  { rows: number; columns: ColumnDef[] }
> = {
  "sales_2026.parquet": {
    rows: 125_000,
    columns: [
      { name: "order_id", type: "string", nullable: false },
      { name: "order_date", type: "date", nullable: false },
      { name: "customer_id", type: "string", nullable: false },
      { name: "product", type: "string", nullable: false },
      { name: "status", type: "string", nullable: false },
      { name: "quantity", type: "integer", nullable: false },
      { name: "price", type: "decimal", nullable: false },
      { name: "cost", type: "decimal", nullable: true },
      { name: "revenue", type: "decimal", nullable: false },
      { name: "country", type: "string", nullable: true },
      { name: "channel", type: "string", nullable: true },
      { name: "discount", type: "decimal", nullable: true },
      { name: "is_return", type: "boolean", nullable: false },
      { name: "updated_at", type: "date", nullable: true },
    ],
  },
  "customers.csv": {
    rows: 18_400,
    columns: [
      { name: "customer_id", type: "string", nullable: false },
      { name: "customer_name", type: "string", nullable: false },
      { name: "segment", type: "string", nullable: true },
      { name: "country", type: "string", nullable: true },
      { name: "signup_date", type: "date", nullable: false },
      { name: "active", type: "boolean", nullable: false },
    ],
  },
  "products.csv": {
    rows: 1_260,
    columns: [
      { name: "product", type: "string", nullable: false },
      { name: "category", type: "string", nullable: true },
      { name: "unit_cost", type: "decimal", nullable: true },
      { name: "supplier", type: "string", nullable: true },
    ],
  },
};

export const DATASET_NAMES = Object.keys(SAMPLE_DATASETS);

const COLUMN_TYPES: ColumnType[] = ["string", "integer", "decimal", "date", "boolean"];

export const ETL_NODES: EtlNodeDef[] = [
  {
    type: "source.dataset",
    label: "Dataset",
    category: "sources",
    description: "Dataset registrato nel workspace",
    Icon: Database,
    inputs: [],
    outputs: ["out"],
    fields: [
      {
        key: "dataset",
        label: "Dataset",
        kind: "select",
        options: DATASET_NAMES,
        defaultValue: DATASET_NAMES[0],
        required: true,
      },
    ],
  },
  {
    type: "source.file",
    label: "File",
    category: "sources",
    description: "CSV o Parquet locale",
    Icon: FileSpreadsheet,
    inputs: [],
    outputs: ["out"],
    fields: [
      { key: "path", label: "Percorso file", kind: "text", placeholder: "sales_2026.parquet", required: true },
      { key: "format", label: "Formato", kind: "select", options: ["csv", "parquet"], defaultValue: "parquet" },
      { key: "delimiter", label: "Delimitatore", kind: "select", options: [",", ";", "|", "tab"], defaultValue: "," },
    ],
  },
  {
    type: "source.sql",
    label: "SQL Query",
    category: "sources",
    description: "Query su database relazionale",
    Icon: Braces,
    inputs: [],
    outputs: ["out"],
    fields: [
      { key: "connection", label: "Connessione", kind: "text", placeholder: "warehouse_prod", required: true },
      { key: "query", label: "Query", kind: "textarea", placeholder: "select * from public.orders", required: true },
    ],
  },
  {
    type: "source.api",
    label: "API / External",
    category: "sources",
    description: "Endpoint HTTP JSON",
    Icon: Globe,
    inputs: [],
    outputs: ["out"],
    fields: [
      { key: "url", label: "Endpoint", kind: "text", placeholder: "https://api.esempio.it/v1/dati", required: true },
      { key: "method", label: "Metodo", kind: "select", options: ["GET", "POST"], defaultValue: "GET" },
      { key: "rootPath", label: "Percorso radice", kind: "text", placeholder: "data.items" },
    ],
  },

  {
    type: "transform.filter",
    label: "Filter",
    category: "transform",
    description: "Mantiene le righe che soddisfano la condizione",
    Icon: Filter,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Colonna", kind: "text", placeholder: "status", required: true },
      { key: "operator", label: "Operatore", kind: "select", options: ["=", "!=", ">", ">=", "<", "<=", "contains"], defaultValue: "=" },
      { key: "value", label: "Valore", kind: "text", placeholder: "ACTIVE", required: true },
    ],
  },
  {
    type: "transform.select",
    label: "Select Columns",
    category: "transform",
    description: "Proietta un sottoinsieme di colonne",
    Icon: Table2,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "columns", label: "Colonne", kind: "textarea", placeholder: "order_date, product, revenue", required: true },
    ],
  },
  {
    type: "transform.rename",
    label: "Rename Columns",
    category: "transform",
    description: "Rinomina colonne",
    Icon: Pencil,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "from", label: "Colonna", kind: "text", placeholder: "revenue", required: true },
      { key: "to", label: "Nuovo nome", kind: "text", placeholder: "ricavi", required: true },
    ],
  },
  {
    type: "transform.formula",
    label: "Formula",
    category: "transform",
    description: "Colonna calcolata",
    Icon: Calculator,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Nuova colonna", kind: "text", placeholder: "margine", required: true },
      { key: "expression", label: "Espressione", kind: "textarea", placeholder: "(price - cost) / price", required: true },
      { key: "type", label: "Tipo", kind: "select", options: COLUMN_TYPES, defaultValue: "decimal" },
    ],
  },
  {
    type: "transform.sort",
    label: "Sort",
    category: "transform",
    description: "Ordina il dataset",
    Icon: SortAsc,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Colonna", kind: "text", placeholder: "order_date", required: true },
      { key: "direction", label: "Direzione", kind: "select", options: ["asc", "desc"], defaultValue: "asc" },
    ],
  },
  {
    type: "transform.dedupe",
    label: "Remove Duplicates",
    category: "transform",
    description: "Rimuove righe duplicate",
    Icon: Copy,
    inputs: ["in"],
    outputs: ["out"],
    fields: [{ key: "columns", label: "Chiavi", kind: "text", placeholder: "order_id" }],
  },
  {
    type: "transform.fillna",
    label: "Fill Missing Values",
    category: "transform",
    description: "Sostituisce i valori mancanti",
    Icon: Eraser,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Colonna", kind: "text", placeholder: "cost", required: true },
      { key: "strategy", label: "Strategia", kind: "select", options: ["valore fisso", "media", "mediana", "ffill"], defaultValue: "valore fisso" },
      { key: "value", label: "Valore", kind: "text", placeholder: "0" },
    ],
  },

  {
    type: "combine.join",
    label: "Join",
    category: "combine",
    description: "Unisce due dataset su una chiave",
    Icon: Combine,
    inputs: ["left", "right"],
    outputs: ["out"],
    fields: [
      { key: "key", label: "Chiave", kind: "text", placeholder: "customer_id", required: true },
      { key: "how", label: "Tipo", kind: "select", options: ["inner", "left", "right", "outer"], defaultValue: "inner" },
    ],
  },
  {
    type: "combine.union",
    label: "Union",
    category: "combine",
    description: "Concatena dataset compatibili",
    Icon: Layers,
    inputs: ["a", "b"],
    outputs: ["out"],
    fields: [
      { key: "mode", label: "Modalità", kind: "select", options: ["colonne comuni", "tutte le colonne"], defaultValue: "colonne comuni" },
    ],
  },
  {
    type: "combine.lookup",
    label: "Lookup",
    category: "combine",
    description: "Arricchisce con una colonna di riferimento",
    Icon: Search,
    inputs: ["in", "lookup"],
    outputs: ["out"],
    fields: [
      { key: "key", label: "Chiave", kind: "text", placeholder: "product", required: true },
      { key: "column", label: "Colonna da aggiungere", kind: "text", placeholder: "category", required: true },
    ],
  },

  {
    type: "aggregate.groupBy",
    label: "Group By",
    category: "aggregate",
    description: "Raggruppa e calcola una metrica",
    Icon: Sigma,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "groupBy", label: "Raggruppa per", kind: "text", placeholder: "product", required: true },
      { key: "aggregation", label: "Funzione", kind: "select", options: ["SUM", "AVG", "MIN", "MAX", "COUNT"], defaultValue: "SUM" },
      { key: "metric", label: "Metrica", kind: "text", placeholder: "revenue", required: true },
    ],
  },
  {
    type: "aggregate.aggregate",
    label: "Aggregate",
    category: "aggregate",
    description: "Metrica globale sul dataset",
    Icon: Gauge,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "aggregation", label: "Funzione", kind: "select", options: ["SUM", "AVG", "MIN", "MAX", "COUNT"], defaultValue: "SUM" },
      { key: "metric", label: "Metrica", kind: "text", placeholder: "revenue", required: true },
    ],
  },
  {
    type: "aggregate.pivot",
    label: "Pivot",
    category: "aggregate",
    description: "Ruota i valori in colonne",
    Icon: Grid3x3,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "index", label: "Righe", kind: "text", placeholder: "product", required: true },
      { key: "columns", label: "Colonne", kind: "text", placeholder: "country", required: true },
      { key: "metric", label: "Valori", kind: "text", placeholder: "revenue", required: true },
    ],
  },

  {
    type: "output.dataset",
    label: "Output Dataset",
    category: "output",
    description: "Materializza il dataset per Model & Dashboard",
    Icon: Save,
    inputs: ["in"],
    outputs: [],
    fields: [
      { key: "name", label: "Nome dataset", kind: "text", placeholder: "dataset_vendite", required: true },
      { key: "mode", label: "Scrittura", kind: "select", options: ["replace", "append"], defaultValue: "replace" },
    ],
  },
  {
    type: "output.table",
    label: "Table",
    category: "output",
    description: "Tabella nell'interfaccia finale",
    Icon: Table2,
    inputs: ["in"],
    outputs: [],
    fields: [{ key: "title", label: "Titolo", kind: "text", placeholder: "Dettaglio ordini" }],
  },
];

export const nodeDef = (type: string) => ETL_NODES.find((n) => n.type === type);

export const categoryAccent = (category: EtlCategory) =>
  category === "output"
    ? "badge-type-dashboard"
    : category === "combine" || category === "aggregate"
      ? "badge-type-model"
      : "badge-type-etl";

/** Riepilogo sintetico mostrato nel nodo sul canvas. */
export function nodeSummary(type: string, config: Record<string, string>): string {
  const v = (k: string) => (config[k] ?? "").trim();
  switch (type) {
    case "source.dataset":
      return v("dataset") || "nessun dataset";
    case "source.file":
      return v("path") || "nessun file";
    case "source.sql":
      return v("query") ? v("query").slice(0, 42) : "nessuna query";
    case "source.api":
      return v("url") || "nessun endpoint";
    case "transform.filter":
      return v("column") ? `${v("column")} ${v("operator") || "="} "${v("value")}"` : "condizione mancante";
    case "transform.select":
      return v("columns") || "nessuna colonna";
    case "transform.rename":
      return v("from") ? `${v("from")} → ${v("to")}` : "nessuna colonna";
    case "transform.formula":
      return v("expression") ? `${v("column")} = ${v("expression")}` : "formula mancante";
    case "transform.sort":
      return v("column") ? `${v("column")} ${v("direction") || "asc"}` : "colonna mancante";
    case "transform.dedupe":
      return v("columns") ? `su ${v("columns")}` : "tutte le colonne";
    case "transform.fillna":
      return v("column") ? `${v("column")} · ${v("strategy")}` : "colonna mancante";
    case "combine.join":
      return v("key") ? `${v("how") || "inner"} on ${v("key")}` : "chiave mancante";
    case "combine.union":
      return v("mode") || "colonne comuni";
    case "combine.lookup":
      return v("key") ? `${v("column")} on ${v("key")}` : "chiave mancante";
    case "aggregate.groupBy":
      return v("metric") ? `${v("aggregation")}(${v("metric")}) by ${v("groupBy")}` : "metrica mancante";
    case "aggregate.aggregate":
      return v("metric") ? `${v("aggregation")}(${v("metric")})` : "metrica mancante";
    case "aggregate.pivot":
      return v("index") ? `${v("index")} × ${v("columns")}` : "configurazione mancante";
    case "output.dataset":
      return v("name") || "nome mancante";
    case "output.table":
      return v("title") || "tabella";
    default:
      return "";
  }
}


=== FILE: src/lib/etl-display.ts ===
import type { LucideIcon } from "lucide-react";
import {
  BarChart3,
  Combine,
  Database,
  Save,
  Sigma,
} from "lucide-react";
import type { EtlCategory } from "@/lib/etl-catalog";

/** Impostazioni di visualizzazione delle card nel canvas ETL. */
export type EtlDisplaySettings = {
  metrics: boolean;
  source: boolean;
  formula: boolean;
  status: boolean;
};

export const DEFAULT_DISPLAY: EtlDisplaySettings = {
  metrics: false,
  source: false,
  formula: false,
  status: true,
};

export const DISPLAY_OPTIONS: { key: keyof EtlDisplaySettings; label: string }[] = [
  { key: "metrics", label: "Mostra metriche righe/colonne" },
  { key: "source", label: "Mostra nome file sorgente" },
  { key: "formula", label: "Mostra formule" },
  { key: "status", label: "Mostra stato esecuzione" },
];

export const CATEGORY_ICONS: Record<EtlCategory, LucideIcon> = {
  sources: Database,
  transform: Sigma,
  combine: Combine,
  aggregate: BarChart3,
  output: Save,
};

/** Layout interno icona-centrale di una card, in pixel. */
export type CardIconLayout = {
  /** Lato del riquadro icona, in px. */
  iconBoxSize: number;
  /** Lato del glifo Lucide, in px. */
  iconGlyphSize: number;
};

/**
 * Calcolo puro del layout icona-centrale di una card ETL (fase 2 del
 * redesign, "la card è l'icona": esteso a TUTTE le categorie, non solo
 * "transform" — isolato dal JSX come richiesto per la futura estensione
 * con agenti/tool AI sulla codebase).
 *
 * Percentuali invariate dalla fase 1 (40% del lato per il riquadro
 * icona, 50% del riquadro per il glifo — assunzione originale: valore
 * centrale del range 35-45% indicato nel brief, con "fulcro visivo
 * dominante" per il glifo). Calcolate qui in PIXEL sul lato MINORE tra
 * width/height (non una % CSS diretta) perché le card "sources" non
 * sono forzate quadrate come le "transform": una % CSS su width e
 * height separatamente distorcerebbe l'icona su una card rettangolare.
 */
export function getCardIconLayout(
  width: number,
  height: number,
): CardIconLayout {
  const side = Math.min(width, height);
  const iconBoxSize = side * 0.4;
  return {
    iconBoxSize,
    iconGlyphSize: iconBoxSize * 0.5,
  };
}


=== FILE: src/lib/etl-motion.ts ===
/**
 * Costanti di durata/easing per i movimenti AUTOMATICI (indotti) di
 * card, bubble e frecce sul canvas ETL — centralizzate qui invece che
 * ripetute nei vari punti di workflow-canvas.tsx che le usano.
 *
 * Non riguardano mai il drag diretto dell'utente: quel movimento resta
 * sempre 1:1 col puntatore, senza transizione (vedi il commento su
 * `isUserDriven` in workflow-canvas.tsx per come viene distinto un
 * movimento "automatico" da un drag attivo).
 */

/** Durata di un movimento automatico, in ms. Range indicato: 150-250ms. */
export const AUTO_MOVE_DURATION_MS = 200;

/** Easing "morbido in uscita" per i movimenti automatici. */
export const AUTO_MOVE_EASING = "cubic-bezier(0.22, 1, 0.36, 1)";

const timing = `${AUTO_MOVE_DURATION_MS}ms ${AUTO_MOVE_EASING}`;

const transitionOf = (properties: readonly string[]): string =>
  properties.map((property) => `${property} ${timing}`).join(", ");

/** Transizione CSS applicata alla card quando si muove per un motivo diverso dal drag dell'utente. */
export const CARD_AUTO_MOVE_TRANSITION = transitionOf([
  "left",
  "top",
  "width",
  "height",
]);

/** Transizione CSS applicata al contenitore di una bubble (resize/orientamento/membri che entrano o escono). */
export const BUBBLE_AUTO_MOVE_TRANSITION = transitionOf([
  "left",
  "top",
  "width",
  "height",
]);

/**
 * Transizione CSS applicata al path di una freccia quando segue un
 * movimento automatico della card/bubble a cui è agganciata. Stessa
 * durata/easing della card, così i due non si muovono mai a scatti
 * indipendenti l'uno dall'altro.
 */
export const EDGE_AUTO_MOVE_TRANSITION = transitionOf([
  "d",
  "stroke-width",
  "stroke-opacity",
]);

/*
 * Registro "iOS" (Parte 2 del redesign): apertura di un elemento che si
 * dispiega da un punto d'origine — menu contestuali, box di gruppo — con
 * una molla che supera leggermente il valore finale prima di assestarsi.
 * Distinto da AUTO_MOVE_EASING (lineare, per i riposizionamenti indotti
 * delle card): quello resta invariato, questo si usa solo per le nuove
 * animazioni "a comparsa" introdotte qui.
 */
export const SPRING_EASE = "cubic-bezier(0.34, 1.56, 0.64, 1)";

/** Durata di apertura in stile iOS, in ms. Range indicato: 300-350ms. */
export const SPRING_OPEN_DURATION_MS = 320;

/** Scala di partenza di un elemento che si dispiega (menu, box di gruppo). */
export const SPRING_ORIGIN_SCALE = 0.15;

/** Transizione di apertura (scale + opacity) per il registro "iOS". */
export const SPRING_OPEN_TRANSITION = `transform ${SPRING_OPEN_DURATION_MS}ms ${SPRING_EASE}, opacity ${SPRING_OPEN_DURATION_MS}ms ${SPRING_EASE}`;

/*
 * Chiusura: nessun overshoot (una molla che "rimbalza" mentre un elemento
 * si ritira dà una sensazione innaturale) — solo un rientro rapido verso
 * il punto di origine, più breve dell'apertura.
 */
export const SPRING_CLOSE_DURATION_MS = 160;
export const SPRING_CLOSE_TRANSITION = `transform ${SPRING_CLOSE_DURATION_MS}ms ease-in, opacity ${SPRING_CLOSE_DURATION_MS}ms ease-in`;

/** Scala "pressed" applicata all'elemento che ancora un menu/box mentre è aperto. */
export const PRESS_SCALE = 0.94;
export const PRESS_TRANSITION = "transform 150ms ease-out";


=== FILE: src/lib/etl-node-config.ts ===
/**
 * Fase 4 — calcolo delle opzioni disponibili per i pannelli
 * impostazioni, dato lo schema EFFETTIVO del nodo (via `analyzeNode`),
 * separato dal rendering (i componenti React stanno sotto
 * src/components/isa/etl/settings-panels/).
 *
 * Nessuna esecuzione reale: come `etl-schema.ts`, sono stime
 * authoring-time basate sui `SAMPLE_DATASETS` del catalogo.
 */
import type { ColumnDef } from "./etl-catalog";
import { nodeDef } from "./etl-catalog";
import { analyzeNode, previewRows } from "./etl-schema";
import type { EtlNode, EtlWorkflow } from "./etl-workflow";

export type SettingsPanelKind =
  | "filter"
  | "combine"
  | "aggregate";

/**
 * Quale pannello impostazioni dedicato mostrare per un dato tipo di
 * nodo (fase 4) — `null` per i tipi non ancora coperti da un pannello
 * dedicato, che restano sul menu azioni generico esistente.
 */
export function getSettingsPanelKind(
  nodeType: string,
): SettingsPanelKind | null {
  if (nodeType === "transform.filter") {
    return "filter";
  }

  if (nodeType.startsWith("combine.")) {
    return "combine";
  }

  if (nodeType.startsWith("aggregate.")) {
    return "aggregate";
  }

  return null;
}

/* -------------------------------------------------------------------------- */
/*                                FILTER                                      */
/* -------------------------------------------------------------------------- */

/** Colonne disponibili per il filtro: lo schema in INGRESSO al nodo Filter. */
export function getFilterColumnOptions(
  workflow: EtlWorkflow,
  node: EtlNode,
): ColumnDef[] {
  return analyzeNode(workflow, node).inputColumns;
}

/**
 * Valori di dominio di una colonna, per il multi-select dei valori da
 * filtrare. Non esiste un dataset reale dietro `SAMPLE_DATASETS` (solo
 * nome/tipo/nullable per colonna): come placeholder strutturalmente
 * corretto riusiamo il generatore deterministico già presente in
 * `previewRows` (stesso seed => stessi valori a ogni render) invece di
 * inventare una seconda fonte di dati finti.
 */
export function getColumnDomainValues(
  column: ColumnDef,
  seed: string,
  sampleSize = 10,
): string[] {
  const rows = previewRows([column], seed, sampleSize);
  const values = rows.map((row) => row[0]).filter((v): v is string => v !== undefined);
  return Array.from(new Set(values));
}

/* -------------------------------------------------------------------------- */
/*                                COMBINE / JOIN                              */
/* -------------------------------------------------------------------------- */

export type CombineInputSchema = {
  nodeId: string;
  title: string;
  columns: ColumnDef[];
};

/**
 * Dataset esterni collegati a un nodo "combine" (o alla sua bubble, se
 * fa parte di un gruppo — fase 2): un ingresso per ogni edge che entra
 * nell'insieme di nodi da FUORI l'insieme stesso, deduplicato per nodo
 * sorgente. `memberIds` è l'insieme di id della bubble (inclusi
 * `node.id`); se assente o con un solo elemento si usa solo `node.id`
 * — così la funzione non deve conoscere la geometria della bubble
 * (nessuna dipendenza da lib/etl-bubble.ts), solo l'insieme di id.
 */
export function getCombineInputs(
  workflow: EtlWorkflow,
  node: EtlNode,
  memberIds?: readonly string[],
): CombineInputSchema[] {
  const members = new Set(
    memberIds && memberIds.length > 1 ? memberIds : [node.id],
  );

  const seen = new Set<string>();
  const inputs: CombineInputSchema[] = [];

  for (const edge of workflow.edges) {
    if (!members.has(edge.toNode) || members.has(edge.fromNode)) {
      continue;
    }

    if (seen.has(edge.fromNode)) {
      continue;
    }

    seen.add(edge.fromNode);

    const fromNode = workflow.nodes.find(
      (n) => n.id === edge.fromNode,
    );

    if (!fromNode) {
      continue;
    }

    inputs.push({
      nodeId: fromNode.id,
      title: fromNode.title,
      columns: analyzeNode(workflow, fromNode).columns,
    });
  }

  return inputs;
}

/** Opzioni del tipo di join, riusando quelle già dichiarate sul campo "how" del catalogo (nessuna lista duplicata). */
export function getJoinTypeOptions(): string[] {
  return (
    nodeDef("combine.join")?.fields.find(
      (field) => field.key === "how",
    )?.options ?? ["inner", "left", "right", "outer"]
  );
}

/** Chiavi di config per la colonna di join lato sinistro/destro della coppia `pairIndex` (0-based, consecutiva: input[i] ⋈ input[i+1]). */
export function joinPairConfigKeys(pairIndex: number): {
  left: string;
  right: string;
} {
  return {
    left: `join_pair_${pairIndex}_left`,
    right: `join_pair_${pairIndex}_right`,
  };
}

/* -------------------------------------------------------------------------- */
/*                                AGGREGATE                                   */
/* -------------------------------------------------------------------------- */

export type AggregateFieldSpec = {
  key: string;
  label: string;
  kind: "single" | "multi";
  options: ColumnDef[];
};

/** Selettori dinamici (colonna singola o multipla) per groupBy/aggregate/pivot, dallo schema effettivo in ingresso. */
export function getAggregateFieldSpecs(
  workflow: EtlWorkflow,
  node: EtlNode,
): AggregateFieldSpec[] {
  const inputColumns = analyzeNode(workflow, node).inputColumns;

  switch (node.type) {
    case "aggregate.groupBy":
      return [
        {
          key: "groupBy",
          label: "Raggruppa per",
          kind: "multi",
          options: inputColumns,
        },
        {
          key: "metric",
          label: "Metrica",
          kind: "single",
          options: inputColumns,
        },
      ];

    case "aggregate.aggregate":
      return [
        {
          key: "metric",
          label: "Metrica",
          kind: "single",
          options: inputColumns,
        },
      ];

    case "aggregate.pivot":
      return [
        {
          key: "index",
          label: "Righe",
          kind: "single",
          options: inputColumns,
        },
        {
          key: "columns",
          label: "Colonne",
          kind: "single",
          options: inputColumns,
        },
        {
          key: "metric",
          label: "Valori",
          kind: "single",
          options: inputColumns,
        },
      ];

    default:
      return [];
  }
}


=== FILE: src/lib/etl-node-size.ts ===
import { nodeDef, nodeSummary } from "./etl-catalog";
import type { EtlDisplaySettings } from "./etl-display";
import { DEFAULT_DISPLAY } from "./etl-display";
import { analyzeNode, formatRows } from "./etl-schema";
import type { EtlNode, EtlWorkflow } from "./etl-workflow";

/**
 * Stima delle dimensioni di una card e costanti di spaziatura correlate
 * — condivise da workflow-canvas.tsx (rendering) e etl-workflow.tsx
 * (autoLayout, che deve conoscere la dimensione REALE delle card per
 * calcolare un passo di griglia adattivo, PARTE C del redesign) così la
 * logica di calcolo dimensioni vive in UN solo posto e i due non
 * possono disallinearsi.
 */

export type NodeSize = {
  width: number;
  height: number;
};

export const NODE_W = 128;
export const NODE_H = 128;

const MIN_NODE_SIZE = 112;
export const MIN_NODE_HEIGHT = 88;
const MAX_NODE_SIZE = 248;

/* Margine/gap standard tra elementi del canvas (card, colonne, righe). */
export const ROUTE_GAP = 24;

/**
 * Altezza della card per un dato `width` e un dato `showDetail` (mostrare
 * o no la riga di dettaglio/formula). Isolata dal resto di
 * `estimateNodeSize` perché serve calcolarla due volte: una con la
 * regola di visibilità "reale" del nodo, una con quella dei nodi
 * `sources` (per il lato del quadrato dei nodi transform, vedi sotto).
 */
function estimateCardHeight(
  node: EtlNode,
  display: EtlDisplaySettings,
  width: number,
  detail: string,
  showDetail: boolean,
): number {
  const CARD_PADDING = 12;
  const BODY_GAP = 12;

  /*
   * Header: icona (36px) affiancata a titolo + label.
   * Il titolo può andare a capo: stimiamo le righe in base
   * allo spazio orizzontale realmente disponibile.
   */
  const headerTextWidth = Math.max(
    40,
    width - 36 - 8 - 28 - CARD_PADDING * 2,
  );

  const titleLines = Math.min(
    3,
    Math.max(
      1,
      Math.ceil(
        (node.title.length * 6.4) /
          headerTextWidth,
      ),
    ),
  );

  const HEADER_HEIGHT = Math.max(
    36,
    titleLines * 15 + 12,
  );

  let bodyHeight = 0;

  if (showDetail && detail) {
    const charsPerLine = Math.max(
      1,
      Math.floor(
        (width -
          CARD_PADDING * 2 -
          20) /
          5.2,
      ),
    );

    const lines = Math.min(
      4,
      Math.max(
        1,
        Math.ceil(
          detail.length / charsPerLine,
        ),
      ),
    );

    bodyHeight += 16 + lines * 14;
  }

  if (display.metrics) {
    bodyHeight +=
      (bodyHeight > 0 ? 8 : 0) + 24;
  }

  if (display.status) {
    bodyHeight +=
      (bodyHeight > 0 ? 6 : 0) + 20;
  }

  if (bodyHeight === 0) {
    /* Solo la label di fallback. */
    bodyHeight = 16;
  }

  return Math.min(
    MAX_NODE_SIZE,
    Math.max(
      MIN_NODE_HEIGHT,
      Math.ceil(
        (CARD_PADDING * 2 +
          HEADER_HEIGHT +
          BODY_GAP +
          bodyHeight) /
          8,
      ) * 8,
    ),
  );
}

/**
 * `display` di default a `DEFAULT_DISPLAY`: autoLayout() (etl-workflow.ts)
 * non ha accesso alle preferenze di visualizzazione dell'utente (stato
 * locale del componente canvas, non del workflow persistito), quindi usa
 * questa baseline coerente per stimare le dimensioni ai fini della
 * spaziatura a griglia. Il rendering reale (workflow-canvas.tsx) passa
 * sempre il `display` live dell'utente.
 */
export function estimateNodeSize(
  node: EtlNode,
  workflow: EtlWorkflow,
  display: EtlDisplaySettings = DEFAULT_DISPLAY,
): NodeSize {
  const def = nodeDef(node.type);

  if (!def) {
    return {
      width: NODE_W,
      height: MIN_NODE_HEIGHT,
    };
  }

  const analysis = analyzeNode(
    workflow,
    node,
  );

  const detail = nodeSummary(
    node.type,
    node.config,
  );

  const isSource =
    def.category === "sources";

  const showDetail =
    (display.source && isSource) ||
    (display.formula && !isSource);

  const metricsText = display.metrics
    ? `${formatRows(analysis.rows)} rows · ${analysis.columns.length} cols`
    : "";

  /* ------------------------------------------------------------------ */
  /*  LARGHEZZA — guidata dal testo più lungo (header / metriche).      */
  /* ------------------------------------------------------------------ */

  const longestText = Math.max(
    node.title.length,
    def.label.length,
    metricsText.length,
    8,
  );

  const textWidth = Math.min(
    210,
    Math.max(
      90,
      longestText * 6.2,
    ),
  );

  /*
   * Header: icona (size-9) + gap + testo + menu (size-7) + padding card.
   */
  const headerWidth =
    textWidth + 36 + 8 + 28 + 24;

  const width = Math.min(
    MAX_NODE_SIZE,
    Math.max(
      MIN_NODE_SIZE,
      Math.ceil(headerWidth / 8) * 8,
    ),
  );

  /* ------------------------------------------------------------------ */
  /*  ALTEZZA — si adatta al contenuto reale, senza spazio in eccesso.  */
  /* ------------------------------------------------------------------ */

  if (!isSource) {
    /*
     * Card "transform" (ogni categoria diversa da sources): sempre
     * quadrata, con lato pari all'altezza che avrebbe una card Dataset
     * con lo stesso livello di dettaglio visualizzato — non un valore
     * fisso, ma la stessa `estimateCardHeight` usata per i nodi
     * sources, così i due tipi restano visivamente allineati anche se
     * cambiano i display settings.
     */
    const side = estimateCardHeight(
      node,
      display,
      width,
      detail,
      display.source,
    );

    return {
      width: side,
      height: side,
    };
  }

  const height = estimateCardHeight(
    node,
    display,
    width,
    detail,
    showDetail,
  );

  return {
    width,
    height,
  };
}


=== FILE: src/lib/etl-schema.ts ===
import type { ColumnDef, ColumnType } from "./etl-catalog";
import { SAMPLE_DATASETS, nodeDef } from "./etl-catalog";
import type { EtlNode, EtlWorkflow } from "./etl-workflow";

export type NodeAnalysis = {
  inputColumns: ColumnDef[];
  columns: ColumnDef[];
  inRows: number | null;
  rows: number | null;
  errors: string[];
};

const splitList = (value: string) =>
  value
    .split(/[,\n]/)
    .map((s) => s.trim())
    .filter(Boolean);

const upstream = (workflow: EtlWorkflow, nodeId: string, port: string) => {
  const edge = workflow.edges.find((e) => e.toNode === nodeId && e.toPort === port);
  if (!edge) return undefined;
  return workflow.nodes.find((n) => n.id === edge.fromNode);
};

const datasetOf = (node: EtlNode) => {
  const name = node.config["dataset"] ?? "";
  return SAMPLE_DATASETS[name];
};

/** Stima statica di schema e volumi: authoring-time, nessuna esecuzione reale. */
export function analyzeNode(
  workflow: EtlWorkflow,
  node: EtlNode,
  depth = 0,
): NodeAnalysis {
  const def = nodeDef(node.type);
  const errors: string[] = [];
  if (!def || depth > 24) return { inputColumns: [], columns: [], inRows: null, rows: null, errors };

  for (const field of def.fields) {
    if (field.required && !(node.config[field.key] ?? "").trim()) {
      errors.push(`Campo obbligatorio mancante: ${field.label}.`);
    }
  }

  const parents = def.inputs.map((port) => {
    const parent = upstream(workflow, node.id, port);
    if (!parent) {
      errors.push(`Input "${port}" non collegato.`);
      return null;
    }
    return { port, analysis: analyzeNode(workflow, parent, depth + 1) };
  });

  const primary = parents.find((p) => p !== null)?.analysis;
  const inputColumns = primary?.columns ?? [];
  const inRows = primary?.rows ?? null;
  const cfg = node.config;
  const has = (name: string) => inputColumns.some((c) => c.name === name);
  const requireColumn = (key: string) => {
    const name = (cfg[key] ?? "").trim();
    if (name && inputColumns.length > 0 && !has(name)) {
      errors.push(`Colonna "${name}" non presente nello schema in ingresso.`);
    }
  };

  let columns = inputColumns;
  let rows = inRows;

  switch (node.type) {
    case "source.dataset": {
      const ds = datasetOf(node);
      if (!ds) {
        columns = [];
        rows = null;
      } else {
        columns = ds.columns;
        rows = ds.rows;
      }
      break;
    }
    case "source.file": {
      const guess = SAMPLE_DATASETS[(cfg["path"] ?? "").trim()];
      columns = guess?.columns ?? [];
      rows = guess?.rows ?? null;
      break;
    }
    case "source.sql":
    case "source.api": {
      columns = [];
      rows = null;
      break;
    }
    case "transform.filter": {
      requireColumn("column");
      rows = inRows === null ? null : Math.round(inRows * 0.66);
      break;
    }
    case "transform.select": {
      const wanted = splitList(cfg["columns"] ?? "");
      for (const name of wanted) {
        if (inputColumns.length > 0 && !has(name)) {
          errors.push(`Colonna "${name}" non presente nello schema in ingresso.`);
        }
      }
      columns = wanted.length
        ? wanted.map(
            (name) =>
              inputColumns.find((c) => c.name === name) ?? {
                name,
                type: "string" as ColumnType,
                nullable: true,
              },
          )
        : inputColumns;
      break;
    }
    case "transform.rename": {
      requireColumn("from");
      const from = (cfg["from"] ?? "").trim();
      const to = (cfg["to"] ?? "").trim();
      columns = inputColumns.map((c) => (c.name === from && to ? { ...c, name: to } : c));
      break;
    }
    case "transform.formula": {
      const name = (cfg["column"] ?? "").trim();
      columns = name
        ? [...inputColumns, { name, type: (cfg["type"] as ColumnType) || "decimal", nullable: true }]
        : inputColumns;
      break;
    }
    case "transform.sort": {
      requireColumn("column");
      break;
    }
    case "transform.dedupe": {
      rows = inRows === null ? null : Math.round(inRows * 0.92);
      break;
    }
    case "transform.fillna": {
      requireColumn("column");
      const name = (cfg["column"] ?? "").trim();
      columns = inputColumns.map((c) => (c.name === name ? { ...c, nullable: false } : c));
      break;
    }
    case "combine.join": {
      const right = parents[1]?.analysis;
      const rightCols = (right?.columns ?? []).filter(
        (c) => !inputColumns.some((l) => l.name === c.name),
      );
      columns = [...inputColumns, ...rightCols];
      rows =
        inRows === null
          ? null
          : (cfg["how"] ?? "inner") === "inner"
            ? Math.round(inRows * 0.88)
            : inRows;
      break;
    }
    case "combine.union": {
      const b = parents[1]?.analysis;
      rows = inRows === null || b?.rows == null ? inRows : inRows + b.rows;
      break;
    }
    case "combine.lookup": {
      const name = (cfg["column"] ?? "").trim();
      columns = name ? [...inputColumns, { name, type: "string", nullable: true }] : inputColumns;
      break;
    }
    case "aggregate.groupBy": {
      const groups = splitList(cfg["groupBy"] ?? "");
      for (const g of groups) {
        if (inputColumns.length > 0 && !has(g)) {
          errors.push(`Colonna "${g}" non presente nello schema in ingresso.`);
        }
      }
      requireColumn("metric");
      const metric = (cfg["metric"] ?? "metrica").trim();
      const agg = cfg["aggregation"] ?? "SUM";
      columns = [
        ...groups.map(
          (g) => inputColumns.find((c) => c.name === g) ?? { name: g, type: "string" as ColumnType, nullable: false },
        ),
        {
          name: `${agg.toLowerCase()}_${metric}`,
          type: agg === "COUNT" ? "integer" : "decimal",
          nullable: false,
        },
      ];
      rows = inRows === null ? null : Math.max(1, Math.round(inRows / 1500));
      break;
    }
    case "aggregate.aggregate": {
      requireColumn("metric");
      const metric = (cfg["metric"] ?? "metrica").trim();
      const agg = cfg["aggregation"] ?? "SUM";
      columns = [
        { name: `${agg.toLowerCase()}_${metric}`, type: agg === "COUNT" ? "integer" : "decimal", nullable: false },
      ];
      rows = 1;
      break;
    }
    case "aggregate.pivot": {
      const index = (cfg["index"] ?? "").trim();
      columns = [
        { name: index || "index", type: "string", nullable: false },
        { name: `${(cfg["columns"] ?? "colonna").trim()}_a`, type: "decimal", nullable: true },
        { name: `${(cfg["columns"] ?? "colonna").trim()}_b`, type: "decimal", nullable: true },
      ];
      rows = inRows === null ? null : Math.max(1, Math.round(inRows / 2000));
      break;
    }
    default:
      break;
  }

  return { inputColumns, columns, inRows, rows, errors };
}

export const formatRows = (rows: number | null) =>
  rows === null
    ? "—"
    : rows >= 1_000_000
      ? `${(rows / 1_000_000).toFixed(1)}M`
      : rows >= 1_000
        ? `${Math.round(rows / 1_000)}k`
        : String(rows);

/** Righe di anteprima deterministiche, coerenti col tipo di colonna. */
export function previewRows(columns: ColumnDef[], seed: string, count = 8) {
  let h = 0;
  for (let i = 0; i < seed.length; i += 1) h = (h * 31 + seed.charCodeAt(i)) % 100_000;
  const rnd = (n: number) => {
    h = (h * 1103515245 + 12345) % 2147483648;
    return h % n;
  };
  const words = ["Alpha", "Beta", "Gamma", "Delta", "Omega", "Nord", "Sud", "Retail", "Online"];
  return Array.from({ length: count }, (_, r) =>
    columns.map((col) => {
      switch (col.type) {
        case "integer":
          return String(rnd(400) + 1);
        case "decimal":
          return (rnd(900_00) / 100).toFixed(2);
        case "date":
          return `2026-0${(rnd(9) + 1).toString()}-${String(rnd(27) + 1).padStart(2, "0")}`;
        case "boolean":
          return rnd(2) === 0 ? "true" : "false";
        default:
          return `${words[rnd(words.length)]}-${r + 1}${rnd(90) + 10}`;
      }
    }),
  );
}


=== FILE: src/lib/etl-workflow.tsx ===
import { useCallback, useSyncExternalStore } from "react";
import { nodeDef } from "./etl-catalog";
import type { NodeSize } from "./etl-node-size";
import { NODE_H, NODE_W, ROUTE_GAP, estimateNodeSize } from "./etl-node-size";

export type NodeStatus = "ready" | "running" | "succeeded" | "error";

export type EtlNode = {
  id: string;
  type: string;
  title: string;
  x: number;
  y: number;
  config: Record<string, string>;
  status: NodeStatus;
  /** Nodi "transform" combinati insieme condividono lo stesso groupId. Opzionale e retrocompatibile. */
  groupId?: string | undefined;
};

export type EtlEdge = {
  id: string;
  fromNode: string;
  fromPort: string;
  toNode: string;
  toPort: string;
};

export type LayoutMode = "auto" | "manual";

export type EtlWorkflow = {
  nodes: EtlNode[];
  edges: EtlEdge[];
  layout: LayoutMode;
};

type Entry = {
  present: EtlWorkflow;
  past: EtlWorkflow[];
  future: EtlWorkflow[];
  savedAt: number;
};

const EMPTY: EtlWorkflow = { nodes: [], edges: [], layout: "auto" };
const EMPTY_ENTRY: Entry = { present: EMPTY, past: [], future: [], savedAt: 0 };

/** Origine della griglia di auto-layout. */
export const LAYOUT_ORIGIN = 32;

// Stato client-side dell'authoring del workflow, per soluzione.
// Nessuna esecuzione reale: il motore arriverà nella fase dedicata.
const store = new Map<string, Entry>();
const listeners = new Set<() => void>();
const emit = () => listeners.forEach((l) => l());

const uid = (p: string) => `${p}-${Math.random().toString(36).slice(2, 8)}`;

const node = (
  type: string,
  title: string,
  x: number,
  y: number,
  config: Record<string, string>,
): EtlNode => ({ id: uid("node"), type, title, x, y, config, status: "ready" });

/** Un gruppo con un solo membro non è più un gruppo: gli toglie il groupId. */
function dissolveSingletonGroups(nodes: EtlNode[]): EtlNode[] {
  const counts = new Map<string, number>();
  for (const n of nodes) {
    if (n.groupId) counts.set(n.groupId, (counts.get(n.groupId) ?? 0) + 1);
  }
  return nodes.map((n) => (n.groupId && (counts.get(n.groupId) ?? 0) < 2 ? { ...n, groupId: undefined } : n));
}

/**
 * Disposizione automatica a colonne, seguendo la direzione del flusso
 * dati. Passo di griglia ADATTIVO (PARTE C del redesign): usa la
 * dimensione REALE di ogni card (stessa `estimateNodeSize` del
 * rendering, condivisa via lib/etl-node-size.ts per non disallineare le
 * due logiche) invece di un passo fisso — che con card di dimensioni
 * molto diverse (una card sources può essere più larga o più stretta di
 * una transform quadrata) causava sovrapposizioni o spazi vuoti
 * eccessivi.
 *
 * `estimateNodeSize` qui non riceve i `display` settings dell'utente
 * (stato locale del componente canvas, non del workflow persistito):
 * usa la sua baseline di default, coerente in ogni run di autoLayout
 * indipendentemente da chi/quando lo invoca.
 *
 * Un nodo appena aggiunto e ancora privo di collegamenti ha profondità
 * 0: finisce quindi nella prima colonna, all'ultima riga (comportamento
 * voluto, PARTE B — niente eccezioni per rispettare la posizione di
 * drop/doppio click quando il layout è "automatico").
 */
export function autoLayout(workflow: EtlWorkflow): EtlNode[] {
  const depth = new Map<string, number>();
  const order = pipelineOrder(workflow);
  for (const n of order) {
    const parents = workflow.edges.filter((e) => e.toNode === n.id);
    const d = parents.length
      ? Math.max(...parents.map((e) => (depth.get(e.fromNode) ?? 0) + 1))
      : 0;
    depth.set(n.id, d);
  }

  const sizes = new Map<string, NodeSize>();
  for (const n of workflow.nodes) {
    sizes.set(n.id, estimateNodeSize(n, workflow));
  }

  const columns = new Map<number, EtlNode[]>();
  for (const n of order) {
    const d = depth.get(n.id) ?? 0;
    const list = columns.get(d);
    if (list) {
      list.push(n);
    } else {
      columns.set(d, [n]);
    }
  }

  const maxDepth = Math.max(0, ...Array.from(columns.keys()));

  /*
   * Larghezza di ogni colonna = card più larga che contiene, così le
   * card della colonna successiva non si sovrappongono mai a quelle
   * larghe di questa. Righe allineate in ALTO nella colonna (non
   * centrate): impilate una sotto l'altra con l'altezza reale di
   * ciascuna, non un passo fisso.
   */
  const columnX: number[] = [];
  let x = LAYOUT_ORIGIN;
  for (let d = 0; d <= maxDepth; d++) {
    columnX.push(x);
    const widest = Math.max(
      NODE_W,
      ...(columns.get(d) ?? []).map((n) => sizes.get(n.id)?.width ?? NODE_W),
    );
    x += widest + ROUTE_GAP;
  }

  const positions = new Map<string, { x: number; y: number }>();
  for (let d = 0; d <= maxDepth; d++) {
    let y = LAYOUT_ORIGIN;
    for (const n of columns.get(d) ?? []) {
      positions.set(n.id, { x: columnX[d]!, y });
      y += (sizes.get(n.id)?.height ?? NODE_H) + ROUTE_GAP;
    }
  }

  return workflow.nodes.map((n) => ({ ...n, ...(positions.get(n.id) ?? {}) }));
}

const arranged = (w: EtlWorkflow): EtlWorkflow =>
  w.layout === "auto" ? { ...w, nodes: autoLayout(w) } : w;

const storageKey = (solutionId: string) => `isa.etl.workflow.${solutionId}`;

function load(solutionId: string): EtlWorkflow | null {
  if (typeof window === "undefined") return null;
  try {
    const raw = window.localStorage.getItem(storageKey(solutionId));
    if (!raw) return null;
    const parsed = JSON.parse(raw) as Partial<EtlWorkflow>;
    if (!Array.isArray(parsed.nodes) || !Array.isArray(parsed.edges)) return null;
    return {
      nodes: parsed.nodes,
      edges: parsed.edges,
      layout: parsed.layout === "manual" ? "manual" : "auto",
    };
  } catch {
    return null;
  }
}

function persist(solutionId: string, workflow: EtlWorkflow) {
  if (typeof window === "undefined") return;
  try {
    window.localStorage.setItem(storageKey(solutionId), JSON.stringify(workflow));
  } catch {
    /* storage non disponibile: lo stato resta in memoria */
  }
}

function entryOf(solutionId: string): Entry {
  const existing = store.get(solutionId);
  if (existing) return existing;
  const restored = load(solutionId);
  const created: Entry = {
    present: restored ? arranged(restored) : EMPTY,
    past: [],
    future: [],
    savedAt: Date.now(),
  };
  store.set(solutionId, created);
  return created;
}

const commit = (solutionId: string, next: EtlWorkflow, history = true) => {
  const entry = entryOf(solutionId);
  const present = arranged(next);
  store.set(solutionId, {
    present,
    past: history ? [...entry.past.slice(-40), entry.present] : entry.past,
    future: history ? [] : entry.future,
    savedAt: Date.now(),
  });
  persist(solutionId, present);
  emit();
};

export function useEtlWorkflow(solutionId: string) {
  const entry = useSyncExternalStore(
    (cb) => {
      listeners.add(cb);
      return () => listeners.delete(cb);
    },
    () => entryOf(solutionId),
    () => EMPTY_ENTRY,
  );
  const workflow = entry.present;

  const addNode = useCallback(
    (
      type: string,
      x: number,
      y: number,
      config?: Record<string, string>,
      title?: string,
      // Bug 1.3: un nodo aggiunto via drag-and-drop esplicito è
      // un'intenzione di posizionamento manuale, esattamente come
      // moveNode — se il workflow resta in layout "auto", `commit()`
      // (via `arranged()`) ricalcolerebbe subito tutte le posizioni con
      // autoLayout(), facendo "sparire" la card dal punto di rilascio.
      // Il doppio click dalla palette NON passa questo flag: per quel
      // percorso lo schema a colonne dell'auto-layout vince sempre,
      // comportamento invariato.
      manual = false,
    ) => {
      const def = nodeDef(type);
      if (!def) return undefined;
      const base: Record<string, string> = {};
      for (const f of def.fields) base[f.key] = f.defaultValue ?? "";
      const created = node(type, title ?? def.label, x, y, { ...base, ...config });
      const current = entryOf(solutionId).present;
      commit(solutionId, {
        ...current,
        ...(manual ? { layout: "manual" as const } : {}),
        nodes: [...current.nodes, created],
      });
      return created.id;
    },
    [solutionId],
  );

  const moveNode = useCallback(
    (id: string, x: number, y: number, history = false) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          // trascinare un nodo passa automaticamente al posizionamento manuale
          layout: "manual",
          nodes: current.nodes.map((n) => (n.id === id ? { ...n, x, y } : n)),
        },
        history,
      );
    },
    [solutionId],
  );

  const setLayout = useCallback(
    (layout: LayoutMode) => {
      const current = entryOf(solutionId).present;
      commit(solutionId, { ...current, layout });
    },
    [solutionId],
  );


  const updateNode = useCallback(
    (
      id: string,
      patch: { title?: string; config?: Record<string, string>; status?: NodeStatus },
      history = true,
    ) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          nodes: current.nodes.map((n) =>
            n.id === id
              ? {
                  ...n,
                  ...(patch.title !== undefined ? { title: patch.title } : {}),
                  ...(patch.status !== undefined ? { status: patch.status } : {}),
                  config: { ...n.config, ...patch.config },
                }
              : n,
          ),
        },
        history,
      );
    },
    [solutionId],
  );

  const removeNode = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      const nodes = dissolveSingletonGroups(current.nodes.filter((n) => n.id !== id));
      commit(solutionId, {
        ...current,
        nodes,
        edges: current.edges.filter((e) => e.fromNode !== id && e.toNode !== id),
      });
    },
    [solutionId],
  );

  const groupNodes = useCallback(
    (ids: string[]) => {
      if (ids.length < 2) return;
      const current = entryOf(solutionId).present;
      // Riunisce anche i membri di eventuali gruppi già esistenti a cui
      // appartengono gli id passati, così due gruppi che vengono uniti
      // finiscono davvero sotto un unico groupId, senza lasciare membri
      // orfani nel gruppo vecchio.
      const involvedGroupIds = new Set(
        current.nodes.filter((n) => ids.includes(n.id) && n.groupId).map((n) => n.groupId!),
      );
      const fullIds = new Set(ids);
      for (const n of current.nodes) {
        if (n.groupId && involvedGroupIds.has(n.groupId)) fullIds.add(n.id);
      }
      const groupId = involvedGroupIds.values().next().value ?? uid("group");
      commit(solutionId, {
        ...current,
        nodes: current.nodes.map((n) => (fullIds.has(n.id) ? { ...n, groupId } : n)),
      });
    },
    [solutionId],
  );

  const ungroupNode = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      const node = current.nodes.find((n) => n.id === id);
      if (!node?.groupId) return;
      const cleared = current.nodes.map((n) => (n.id === id ? { ...n, groupId: undefined } : n));
      commit(solutionId, { ...current, nodes: dissolveSingletonGroups(cleared) });
    },
    [solutionId],
  );

  const connect = useCallback(
    (fromNode: string, fromPort: string, toNode: string, toPort: string) => {
      if (fromNode === toNode) return;
      const current = entryOf(solutionId).present;
      if (
        current.edges.some(
          (e) => e.fromNode === fromNode && e.fromPort === fromPort && e.toNode === toNode && e.toPort === toPort,
        )
      )
        return;
      // una porta di input accetta una sola connessione
      const edges = current.edges.filter((e) => !(e.toNode === toNode && e.toPort === toPort));
      commit(solutionId, {
        ...current,
        edges: [...edges, { id: uid("edge"), fromNode, fromPort, toNode, toPort }],
      });
    },
    [solutionId],
  );

  const removeEdge = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      commit(solutionId, { ...current, edges: current.edges.filter((e) => e.id !== id) });
    },
    [solutionId],
  );

  const setStatuses = useCallback(
    (updates: Record<string, NodeStatus>) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          nodes: current.nodes.map((n) => (updates[n.id] ? { ...n, status: updates[n.id]! } : n)),
        },
        false,
      );
    },
    [solutionId],
  );

  const undo = useCallback(() => {
    const e = entryOf(solutionId);
    const previous = e.past[e.past.length - 1];
    if (!previous) return;
    store.set(solutionId, {
      present: previous,
      past: e.past.slice(0, -1),
      future: [e.present, ...e.future].slice(0, 40),
      savedAt: Date.now(),
    });
    persist(solutionId, previous);
    emit();
  }, [solutionId]);

  const redo = useCallback(() => {
    const e = entryOf(solutionId);
    const next = e.future[0];
    if (!next) return;
    store.set(solutionId, {
      present: next,
      past: [...e.past, e.present],
      future: e.future.slice(1),
      savedAt: Date.now(),
    });
    persist(solutionId, next);
    emit();
  }, [solutionId]);

  return {
    workflow,
    canUndo: entry.past.length > 0,
    canRedo: entry.future.length > 0,
    addNode,
    moveNode,
    setLayout,
    updateNode,
    removeNode,
    groupNodes,
    ungroupNode,
    connect,
    removeEdge,
    setStatuses,
    undo,
    redo,
  };
}

/** Ordine topologico di esecuzione. */
export function pipelineOrder(workflow: EtlWorkflow): EtlNode[] {
  const indeg = new Map(workflow.nodes.map((n) => [n.id, 0]));
  for (const e of workflow.edges) indeg.set(e.toNode, (indeg.get(e.toNode) ?? 0) + 1);
  const queue = workflow.nodes.filter((n) => (indeg.get(n.id) ?? 0) === 0);
  const out: EtlNode[] = [];
  const seen = new Set<string>();
  while (queue.length) {
    const n = queue.shift()!;
    if (seen.has(n.id)) continue;
    seen.add(n.id);
    out.push(n);
    for (const e of workflow.edges.filter((x) => x.fromNode === n.id)) {
      const left = (indeg.get(e.toNode) ?? 0) - 1;
      indeg.set(e.toNode, left);
      if (left <= 0) {
        const target = workflow.nodes.find((x) => x.id === e.toNode);
        if (target) queue.push(target);
      }
    }
  }
  for (const n of workflow.nodes) if (!seen.has(n.id)) out.push(n);
  return out;
}

