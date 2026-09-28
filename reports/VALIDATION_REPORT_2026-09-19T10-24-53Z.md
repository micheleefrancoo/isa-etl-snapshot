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
