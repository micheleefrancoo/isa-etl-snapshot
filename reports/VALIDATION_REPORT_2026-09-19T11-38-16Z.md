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
