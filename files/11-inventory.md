# 11-inventory.md

File in questo blocco:

- `docs/inventory/INVENTARIO_2026-09-28T19-40-33Z.md`
- `docs/inventory/lovable-1e72955.diff`

---

### `docs/inventory/INVENTARIO_2026-09-28T19-40-33Z.md` (parte 1/2)

310 righe totali

```md
# INVENTARIO — stato del codice vs. prototipo `isa-fusion-prototype.html`

Generato: 2026-09-28T19:40:33Z (UTC). Solo lettura: nessun file applicativo è stato modificato per produrre questo rapporto.

**Metodo:** questo rapporto è stato compilato da quattro ricognizioni parallele indipendenti (prototipo/token visivi, architettura del canvas + Fase 2A, architettura schermata ETL/flusso dati, sistema visivo del prodotto/dipendenze/test) più una verifica incrociata finale: ogni fatto riportato da più di una ricognizione è stato confrontato per consistenza, e un campione delle affermazioni più rilevanti (assenza di pan/minimappa, mancato utilizzo di `isValidDropPoint`, disconnessione di `tool-palette.tsx` dal layer `src/canvas/**`, file morti) è stato riverificato direttamente con `grep` prima di essere incluso qui. Nessuna discrepanza è stata trovata tra le ricognizioni; alcuni numeri di riga assenti in una ricognizione sono stati completati con quelli, identici, riportati dalle altre.

**Nota sul nome del file prototipo:** il compito nomina `docs/prototype/isa-canvas-prototype.html`, ma il file effettivamente presente in `docs/prototype/` si chiama `isa-fusion-prototype.html` (259 379 byte, 5097 righe, un solo file con `<style>` e `<script>` inline). Questo rapporto assume che sia quello il prototipo di riferimento — nessun altro file HTML esiste sotto `docs/`. (dedotto: nessuna conferma diretta che sia lo stesso file voluto dal nome originale, ma non esiste alternativa nel repository.)

Questo rapporto **non propone un piano di implementazione**: fotografa lo stato attuale e i divari rispetto al prototipo.

---

## 1. Architettura attuale

### Bootstrap / entry point
- `src/start.ts` (29 righe) — istanza `createStart()` di TanStack Start con due middleware server-side: `errorMiddleware` (cattura eccezioni non gestite in una server function e le trasforma in una pagina d'errore HTML 500 via `src/lib/error-page.ts`) e `createCsrfMiddleware` (protezione CSRF sulle server function, reintrodotta esplicitamente perché definire `start.ts` disattiva quella di default).
- `src/server.ts` (61 righe — non letto integralmente in nessuna delle ricognizioni; dedotto dal nome e dalle convenzioni TanStack Start: entry point SSR lato server. Non risulta specifico al canvas ETL).
- `src/router.tsx` (16 righe) — `getRouter()`: crea il router TanStack con `routeTree` importato da `src/routeTree.gen.ts` (**file generato automaticamente** dal plugin router, 356 righe — non è codice scritto a mano), un `QueryClient` nel context, `scrollRestoration: true`.
- `src/routes/__root.tsx` (144 righe) — route radice: avvolge l'app in `QueryClientProvider`, `ThemeProvider` (`src/lib/theme.tsx`) e `SolutionsProvider` (`src/lib/solutions-store.tsx`); gestisce `HeadContent`/`Scripts` (SSR) e la pagina 404.

### Routing verso la schermata canvas
Routing basato su file (`@tanstack/react-router`), albero rilevante:
```
__root.tsx
└─ routes/solutions.$solutionId.tsx      (111 righe — SolutionWorkspace: chrome del workspace, tabs modulo, <Outlet/>)
   └─ routes/solutions.$solutionId.etl.tsx  (264 righe — EtlWorkspace: la schermata canvas)
```
`solutions.$solutionId.tsx` legge `useSolutions()` per trovare la soluzione dall'URL (`$solutionId`), mostra una pagina "Soluzione non trovata" se assente, e monta le tab dei moduli definite in `src/lib/modules.ts` (`MODULES`: ETL → `/solutions/$solutionId/etl`, Model → `/solutions/$solutionId/model`, Dashboard → `/solutions/$solutionId/dashboard`). `solutions.$solutionId.etl.tsx` è il file di rotta che possiede lo stato locale della schermata ETL e monta `<WorkflowCanvas>`.

### Albero dei componenti (dalla pagina ai singoli nodi)
```
EtlWorkspace (routes/solutions.$solutionId.etl.tsx)
├─ barra superiore (breadcrumb, toggle tema, Undo/Redo, toggle Inspector, toggle Data Preview, Run Workflow)
└─ <WorkflowCanvas> (components/isa/etl/workflow-canvas.tsx, 5037 righe)
   ├─ <CanvasStoreProvider> + <CanvasContainer>  (src/canvas/**, Fase 1/2A: bounds/zoom condivisi)
   ├─ <ToolPalette>              (components/isa/etl/tool-palette.tsx) — cassetta risorse, ancorabile su 4 lati
   ├─ superficie zoomata (surfaceRef, transform: scale(zoom))
   │  ├─ card dei nodi (renderizzate inline dentro WorkflowCanvas, non un componente separato per-nodo)
   │  │  └─ per-nodo: menu impostazioni (IsaMenu con FilterPanel | CombinePanel | AggregatePanel, secondo `getSettingsPanelKind`)
   │  ├─ frecce/cavi SVG (path calcolati da `getBestRoute`, animazione flusso via classe CSS `edge-flow-dot`)
   │  ├─ "bubble" (contorno visivo dei gruppi di card combinate, `groupId`)
   │  ├─ {children} ← qui vengono montati, DENTRO la superficie scalata (Fase 2A):
   │  │  ├─ <Inspector>     (components/isa/etl/inspector.tsx) — pannello destro, "stretch"
   │  │  └─ <DataPreview>   (components/isa/etl/data-preview.tsx) — pannello basso, "stretch"
   │  └─ menu contestuale interno (funzione `CanvasContextMenu`, definita DENTRO workflow-canvas.tsx, righe 1288-1558 — vedi nota sotto sui file inutilizzati)
   └─ context menu / overlay via `createPortal`
```
Non esiste un componente `Node`/`Card` separato: la card di ogni nodo è markup inline dentro il grande JSX di `WorkflowCanvas` (il componente è un unico file di 5037 righe che possiede rendering, geometria, drag, routing e menu). `ToolPalette` **non** fa parte del sistema di bounds condivisi di `src/canvas/**`: gestisce la propria apertura/chiusura/ancoraggio con stato locale (`expanded`, `paletteDock` nel file di rotta) e non chiama mai `usePanelState`/`useCanvasStore` — nonostante `panelRegistry.ts` dichiari una voce `"tool-palette"` per lei (vedi §2 e §8: wiring dichiarato ma inerte, verificato con `grep` diretto — zero import di `@/canvas/**` in `tool-palette.tsx`).

### Gestione dello stato
Quattro sistemi di stato distinti, nessuno condiviso tra loro tranne via props/children:

| Store/hook | File | Pattern | Cosa contiene | Chi lo consuma |
|---|---|---|---|---|
| `useEtlWorkflow(solutionId)` | `src/lib/etl-workflow.tsx` | **Store esterno vanilla** (`Map` a livello di modulo + `useSyncExternalStore`, NON Context/Redux/Zustand) | Il workflow stesso: `nodes[]`, `edges[]`, `layout` ("auto"/"manual"), più `past[]`/`future[]` per undo/redo (cap 40), persistito in `localStorage` sotto `isa.etl.workflow.<solutionId>` | `EtlWorkspace` (unico punto di ingresso); tutte le mutazioni (`addNode`, `moveNode`, `connect`, `groupNodes`, `updateNode`, `removeNode`, `removeEdge`, `setStatuses`, `setLayout`, `undo`, `redo`) passano da qui e ridiscendono come props/callback |
| `CanvasStoreProvider`/`useCanvasStore` | `src/canvas/store/canvasStore.tsx` | Context + `useReducer` | Stato open/closed/size per pannello (`PanelRuntimeState` per id: `inspector`, `data-preview`, `tool-palette` — i 3 id in `PANEL_DEFINITIONS`, ma solo i primi due hanno un consumatore reale — vedi sopra) | `useCanvasBounds`/`CanvasContainer` (per calcolare i bounds), `usePanelState` (chiamato solo da `inspector.tsx` e `data-preview.tsx`, verificato via grep) |
| `SolutionsProvider`/`useSolutions` | `src/lib/solutions-store.tsx` | Context (`createContext`) + stato locale (`useState`/`useCallback`) | Elenco `Solution[]` (nome, status, versione, chart, parametri, shares, moduli), `BatchJob[]`, `ActivityItem[]` — dominio applicativo generale, non specifico ETL | `EtlWorkspace` e `SolutionWorkspace` (solo per trovare la soluzione corrente e mostrare "Saved · versione") |
| `ThemeProvider`/`useTheme` | `src/lib/theme.tsx` | Context + `useState` (34 righe, letto solo di striscio) | tema chiaro/scuro | toggle nella toolbar dell'ETL workspace |

Stato **locale al componente** (non condiviso), tutto dentro `WorkflowCanvas`: `pending` (collegamento/porta in corso), `contextMenu`, `zoom` (nessun `pan` — confermato con grep diretto: zero occorrenze di `panX`/`panY`/`setPan`/`minimap` in tutto `src/`), `hoverEdge`, `grid` (toggle visualizzazione griglia), `display` (`EtlDisplaySettings`: quali metriche mostrare sulle card), `box`/`paletteBox` (dimensioni misurate via `ResizeObserver`), `paletteDock` (lato di ancoraggio della palette: top/bottom/left/right), `freshNodeIds` (nodi appena creati da doppio click, non ancora "raccolti"), `combinePreview` (anteprima della fusione durante il drag). Il drag stesso vive in un `useRef` imperativo (`dragRef`), non in `useState`, per evitare re-render ad ogni frame.

### Flusso dati end-to-end (esempio concreto)
1. Utente clicca una card nel canvas → `WorkflowCanvas` chiama `onSelect(id)` (prop) → `EtlWorkspace` aggiorna `selectedId` (stato locale del file di rotta).
2. `EtlWorkspace` ricalcola `selected = workflow.nodes.find(...)` e lo passa come prop `node` sia a `<Inspector>` sia a `<WorkflowCanvas>` (che lo confronta con `selectedId` per l'evidenziazione `node-selected`).
3. L'utente modifica un campo nell'Inspector → `onChangeConfig(key, value)` (prop, definita in `EtlWorkspace`) → chiama `updateNode(selected.id, { config: { [key]: value } })`, che è `useEtlWorkflow`'s `updateNode` → scrive nello store esterno (`commit()`), che aggiorna `localStorage` e notifica via `emit()`.
4. `useSyncExternalStore` fa ri-renderizzare `EtlWorkspace` con il nuovo `workflow` → il nuovo `workflow.nodes` scende come prop a `WorkflowCanvas`, che richiama `analyzeNode(workflow, node)` per quel nodo (in `inspector.tsx`, `data-preview.tsx` e dentro il rendering di ogni card) e aggiorna schema/righe stimate/errori ovunque servano, nello stesso render.
5. `DataPreview` riceve lo stesso `selected`/`workflow` come prop e ricalcola colonne/righe di anteprima in modo indipendente — è un fratello di `Inspector`, guidato dallo stesso genitore, non legge direttamente da `Inspector`.

Nessuno di questi passaggi usa Context per il workflow: è tutto props-drilling da `EtlWorkspace` in giù, tranne per i tre store elencati sopra (canvas layout, solutions, tema). La sola cosa che Inspector/DataPreview leggono direttamente da una fonte condivisa (non passata da `EtlWorkspace`) è la geometria del layout canvas (`useCanvasBoundsContext`, `usePanelState`), mai i dati del workflow stesso.

---

## 2. Mappa dei file del canvas

Colonne: Percorso | Righe | Ruolo (una riga) | In uso?

### `src/canvas/**` (fondazione geometrica, Fase 1 + Fase 2A)
| Percorso | Righe | Ruolo | In uso? |
|---|---|---|---|
| `src/canvas/README.md` | 142 | Documentazione architetturale del layer canvas | n/a (doc) |
| `src/canvas/FUNCTIONAL_CHECKS.md` | 74 | Checklist manuale Fase 1, **non aggiornata dopo la Fase 2A** (dice ancora "non è ancora collegato a workflow-canvas.tsx" — vedi §3) | n/a (doc, out-of-sync) |
| `src/canvas/layout/canvasBounds.ts` | 218 | Geometria pura: `calculateCanvasBounds`, `computePanelRect`, `clampRectToBounds`, `isRectOutOfBounds` | Sì — usato da `dropZones.ts`, `surfacePanels.ts`, `useCanvasBounds.ts` |
| `src/canvas/layout/panelRegistry.ts` | 60 | `PANEL_DEFINITIONS` (Tool Palette/Inspector/Data Preview: lato, anchor, closedSize) | Sì (per Inspector/Data Preview) — la voce `"tool-palette"` è dichiarata ma **inerte**: `tool-palette.tsx` non la consuma mai (vedi §1, §8) |
| `src/canvas/layout/dropZones.ts` | 58 | `calculateDropZones`, `isValidDropPoint` | **Parziale**: `calculateDropZones` è usato da `useCanvasBounds.ts`; `isValidDropPoint` **non ha alcun chiamante fuori dal proprio file di test** (verificato con grep: unico riferimento in `dropZones.test.ts`) — la logica di drop-validity realmente usata nel drag delle card vive duplicata dentro `workflow-canvas.tsx` (vedi §8) |
| `src/canvas/layout/surfacePanels.ts` | 85 | Fase 2A: `computeSurfacePanelRect`/`surfacePanelStyle` (tecnica del "controscale") | Sì — usato da `inspector.tsx`, `data-preview.tsx` |
| `src/canvas/store/canvasStore.tsx` | 122 | `CanvasStoreProvider`/`useCanvasStore`, stato open/closed/size per pannello | Sì — usato da `workflow-canvas.tsx` (Provider) e da `usePanelState`/`useCanvasBounds` |
| `src/canvas/hooks/useCanvasBounds.ts` | 80 | Hook: misura contenitore + store → bounds/drop-zone reattivi | Sì — usato da `CanvasContainer.tsx` |
| `src/canvas/hooks/usePanelState.ts` | 54 | Hook: stato+azioni di UN pannello (`open`/`close`/`toggle`/`setSize`) | Sì — usato da `inspector.tsx`, `data-preview.tsx` **soli** (grep diretto: nessun'altra chiamata a `usePanelState(` in `src/`) |
| `src/canvas/components/CanvasContainer.tsx` | 107 | Contenitore reattivo + `useCanvasBoundsContext` | Sì — usato da `workflow-canvas.tsx` (wrapper), `inspector.tsx`, `data-preview.tsx` (context) |
| `src/canvas/__tests__/panelPositioning.test.ts` | 178 | 9 test Fase 2A (posizionamento pannelli controscalati) | Sì (test) |
| `src/canvas/layout/__tests__/canvasBounds.test.ts` | 204 | 12 test Fase 1 | Sì (test) |
| `src/canvas/layout/__tests__/dropZones.test.ts` | 149 | 10 test Fase 1 | Sì (test) |
| `src/canvas/layout/__tests__/panelRegistry.test.ts` | 89 | 10 test Fase 1 | Sì (test) |

### `src/components/isa/etl/**`
| Percorso | Righe | Ruolo | In uso? |
|---|---|---|---|
| `workflow-canvas.tsx` | 5037 | Il canvas ETL: rendering nodi/frecce, drag, drop, combine, routing ortogonale, zoom, menu contestuale, palette dock | Sì (componente radice della schermata) |
| `inspector.tsx` | 272 | Pannello destro: dettaglio/config di UN nodo selezionato, colonna singola verticale | Sì |
| `data-preview.tsx` | 165 | Pannello basso: anteprima righe stimate del nodo selezionato | Sì |
| `tool-palette.tsx` | 356 | Cassetta risorse (categorie/nodi del catalogo), drag-to-canvas, doppio click per aggiungere, ancorabile su 4 lati | Sì |
| `isa-context-menu.tsx` | 320 | Componente menu contestuale generico standalone | **NO — nessun importatore nel resto del repo** (verificato con grep). `workflow-canvas.tsx` implementa un proprio `CanvasContextMenu` interno (righe 1288-1558) invece di usare questo file: probabile residuo di una versione precedente, ora codice morto |
| `settings-panels/filter-panel.tsx` | 167 | Editor per nodi Filter (colonna + output mode + value-picker multi-select) | Sì — montato da `workflow-canvas.tsx` |
| `settings-panels/combine-panel.tsx` | 321 | Editor per famiglia Combine (Join/Union/Lookup), form dinamico secondo il tipo | Sì |
| `settings-panels/aggregate-panel.tsx` | 149 | Editor per famiglia Aggregate (Group By/Aggregate/Pivot) | Sì |
| `settings-panels/panel-controls.tsx` | 119 | Primitive UI condivise dai 3 editor sopra (`PanelLabel`, `PanelSelect`, `PanelChipToggleList`, `PanelEmptyState`) | Sì |

### Altri file coinvolti nella schermata canvas
| Percorso | Righe | Ruolo | In uso? |
|---|---|---|---|
| `src/components/isa/ui/isa-menu.tsx` | 684 | Menu/popover custom (non Radix), `createPortal` + posizionamento manuale, usato per il trigger impostazioni per-nodo e per i menu della palette | Sì |
| `src/components/isa/ui/isa-modal.tsx` | 52 | Componente modale generico | **NO — nessun importatore nel resto del repo** (verificato con grep). Codice morto (dedotto: probabile residuo di refactor) |
| `src/lib/etl-workflow.tsx` | 470 | Store esterno del workflow + undo/redo + `autoLayout` + `pipelineOrder` (§1) | Sì |
| `src/lib/etl-catalog.ts` | 427 | Catalogo dei 19 tipi di nodo (5 categorie), campi di configurazione, dataset di esempio | Sì |
| `src/lib/etl-schema.ts` | 254 | Simulazione statica di schema/righe/errori per nodo (`analyzeNode`) — nessuna esecuzione reale | Sì |
| `src/lib/etl-display.ts` | 73 | `EtlDisplaySettings` (quali metriche mostrare su una card) | Sì |
| `src/lib/etl-bubble.ts` | 133 | Geometria pura dei "bubble" (contorno gruppi combinati) | Sì |
| `src/lib/etl-motion.ts` | 80 | Costanti durata/easing per movimenti automatici (non il drag diretto) | Sì |
| `src/lib/etl-node-config.ts` | 223 | Calcolo opzioni per i 3 settings-panels, dato lo schema effettivo | Sì |
| `src/lib/etl-node-size.ts` | 245 | Stima dimensioni card (condivisa da rendering e `autoLayout`) | Sì |
| `src/lib/solutions-store.tsx` | 340 | Store generale "soluzioni" (non specifico ETL), 14 importatori nel repo | Sì, ma solo per breadcrumb/versione |
| `src/lib/modules.ts` | 34 | Registro dei 3 moduli (ETL/Model/Dashboard) e relative route, 6 importatori | Sì |
| `src/lib/theme.tsx` | 34 | Tema chiaro/scuro | Sì |
| `src/routes/solutions.$solutionId.etl.tsx` | 264 | File di rotta: possiede stato locale, monta `WorkflowCanvas`+`Inspector`+`DataPreview` | Sì |
| `src/routes/solutions.$solutionId.tsx` | 111 | File di rotta padre: chrome workspace, tab moduli | Sì |
| `src/routes/__root.tsx` | 144 | Root route: provider globali | Sì |
| `src/router.tsx` | 16 | Crea l'istanza router | Sì |
| `src/routeTree.gen.ts` | 356 | **Generato automaticamente**, non scritto a mano | Sì (build-time) |

**Componenti shadcn/ui (`src/components/ui/**`, 46 file) rilevanti per il canvas ma attualmente NON usati da esso:** `resizable.tsx` (wrapper di `react-resizable-panels`, zero importatori in tutto `src/`) e `tabs.tsx` (wrapper di `@radix-ui/react-tabs`, zero importatori in tutto `src/`). Sono disponibili come dipendenza ma inutilizzati oggi in qualsiasi punto del prodotto.

---

## 3. Stato della Fase 2A

**Risposta diretta:** sì, il lavoro di Fase 2A esiste, è reale (non solo documentato) ed è verificato — ma **vive solo nel working tree non commitato**, non su un branch, non merged, perché non esiste altro branch che `main` e i file coinvolti sono `??`/`M` in `git status` (non committati a nessun commit).

Prove:
- `git branch -a` → solo `main` (nessun altro branch locale o remoto).
- `git log --oneline -20` su `main` → l'ultimo commit è `98f8366 "Add project README"`; nessun commit menziona "Fase 2" o "2A"; `git log --all --oneline -i --grep="fase 2"` e varianti restituiscono zero risultati su qualunque ref.
- `git status --porcelain` → `src/canvas/` intero è `??` (non tracciato), e `src/components/isa/etl/workflow-canvas.tsx`, `inspector.tsx`, `data-preview.tsx`, `src/routes/solutions.$solutionId.etl.tsx` sono `M` (modificati rispetto a `HEAD`, non committati). Questi sono esattamente i file che il report di Fase 2A elenca come "Files Modified".
- `src/canvas/.reports/VALIDATION_REPORT_2026-09-19T11-38-16Z.md` è il report di validazione della Fase 2A stessa (titolo: "Validation Report — Fase 2A (Canvas Integration, Option A)"), datato 2026-09-19T11:38:16Z.

**Cosa fa realmente il codice di Fase 2A** (letto direttamente, non dal report): ha collegato `Inspector` e `DataPreview` al sistema di bounds condivisi di `src/canvas/` usando la tecnica del "pannello controscalato" (`src/canvas/layout/surfacePanels.ts`): i due pannelli vivono ora come `children` di `<WorkflowCanvas>`, montati dentro la stessa superficie scalata (`surfaceRef`) delle card, applicano `transform: scale(1/zoom)` per restare a dimensione fisica costante, e si escludono a vicenda dal calcolo dei propri bounds tramite `canvasStore`. Confermato leggendo `inspector.tsx`/`data-preview.tsx` (import di `useCanvasBoundsContext`, `usePanelState`, `computeSurfacePanelRect`) e `workflow-canvas.tsx` (prop `children`, import di `CanvasContainer`/`CanvasStoreProvider`).

**Cosa la Fase 2A NON ha fatto (dichiarato nel suo stesso report, verificato nel codice):**
- Le card non evitano ancora Inspector/Data Preview durante il drag: `placeNode` in `workflow-canvas.tsx` conosce solo la Tool Palette come ostacolo (confermato: nessun riferimento a Inspector/DataPreview bounds nella logica di drag delle card).
- Nessun auto-layout attorno a questi due pannelli.
- Nessun pan (non esiste come feature nel canvas — solo `zoom`, confermato per assenza totale di `panX`/`panY`/`setPan` in tutto `src/`).
- La Tool Palette non è mai stata portata nel sistema `canvasStore`/`panelRegistry`, nonostante quest'ultimo dichiari una voce per lei (vedi §1, §2, §8) — non è chiaro se questo sia deliberatamente fuori scope o una lacuna non ancora notata (domanda aperta, §9).

**Incongruenza di documentazione trovata:** `src/canvas/FUNCTIONAL_CHECKS.md` (titolo "Fase 1") dice ancora "questo layer non è ancora collegato a `workflow-canvas.tsx`" — frase vera al momento della Fase 1 ma **non aggiornata** dopo la Fase 2A, che invece lo ha collegato (parzialmente, solo per Inspector/DataPreview). `README.md` invece è stato aggiornato con la sezione "Fase 2A" e riflette lo stato corretto.

---

## 4. Sistema visivo

**Tailwind v4, configurazione CSS-first** (nessun `tailwind.config.*`): tutto in `src/styles.css` (403 righe) via `@theme inline`, `@import "tailwindcss"`, `@utility`. `components.json` (shadcn) punta a questo stesso file.

### Token del tema del prodotto (`:root` / `.dark`, spazio colore `oklch`)
| Token | Chiaro | Scuro |
|---|---|---|
| `--background` | `oklch(0.978 0.004 250)` | `oklch(0.19 0.008 260)` |
| `--foreground` | `oklch(0.24 0.015 255)` | `oklch(0.96 0.004 250)` |
| `--primary`/`--brand` | `oklch(0.52 0.11 272)` (viola) | `oklch(0.68 0.11 275)` |
| `--destructive` | `oklch(0.6 0.19 22)` (rosso) | `oklch(0.65 0.18 22)` |
| `--success` | `oklch(0.64 0.11 155)` (verde) | `oklch(0.72 0.11 155)` |
| `--warning` | `oklch(0.76 0.12 75)` (ambra) | `oklch(0.8 0.12 80)` |
| `--glass`/`--glass-strong`/`--glass-border` | bianco a opacità 48%/66%/55% | bianco a opacità 5%/10%/11% |
| `--chart-1..5` | 5 tonalità viola→verde | equivalenti più chiari |
| `--radius` | `1rem` (base), scala `sm..4xl` = `radius ∓ {4,2,0,4,8,12,16}px` | uguale (non ridefinito in dark) |
| `--shadow-glass` | `0 18px 45px -22px oklch(.. / 28%)` | `0 22px 55px -26px oklch(0 0 0 / 55%)` |
| `--type-etl-bg`/`-fg`, `--type-model-*`, `--type-dashboard-*` | badge colorati per tipo modulo | equivalenti invertiti |
| Font | `"Poppins", ui-sans-serif, system-ui, ...` | uguale |

### Token del prototipo, per confronto diretto (letti dal file, non dal prodotto)
Il prototipo usa una palette e una scala completamente diverse, non basate su OKLCH e non condivise col prodotto — nessuna sovrapposizione di valori, solo di *ruolo*:

| Token prototipo | Valore | Ruolo | Equivalente più vicino nel prodotto |
|---|---|---|---|
| `--bg` | `#F5F3EE` | sfondo pagina | `--background` (oklch chiaro) |
| `--ink` | `#262420` | testo primario | `--foreground` |
| `--muted` | `#847E74` | testo secondario | (nessun token dedicato equivalente identificato) |
| `--accent` | `#6C63FF` | brand viola | `--primary`/`--brand` |
| `--surface-strong` | `rgba(255,255,255,0.92)` + `backdrop-filter: blur(24px)` | pannelli "vetro" | `--glass-strong` (stesso ruolo, valori diversi) |
| raggio pannelli | `26px` fisso | contenitori vetro (toolbox/inspector/dialog) | scala `--radius-sm..4xl` (nessun passo esatto a 26px) |
| raggio pillole | `999px` | bottoni/chip/tab | coerente col prodotto (`rounded-full` è già lo standard per badge/chip) |
| ombra pannelli | `0 40px 80px -30px rgba(38,36,32,0.4)` | pannelli elevati | `--shadow-glass` (stesso ruolo, offset/opacità diversi) |
| font | `'Manrope'` (500/700/800) | tutta la UI | `"Poppins"` — famiglia diversa |

Conclusione: **stessa filosofia visiva (glassmorphism, pillole, palette viola-su-neutro), nessun token condiviso** — implementare l'estetica del prototipo richiede una decisione esplicita (riusare i token esistenti del prodotto adattandone i valori, o introdurre i valori del prototipo) piuttosto che un porting 1:1 dei nomi.

### Utility custom del prodotto (`@utility`, riusabili come classi Tailwind)
`glass-panel`/`glass-soft`/`glass-chip` (3 livelli di "vetro smerigliato" con `backdrop-filter: blur()`), `badge-type-{etl,model,dashboard}`, `gradient-brand`/`text-gradient-brand`, `blob` (macchie di sfondo sfocate), `scroll-slim` (scrollbar sottile cross-browser), `edge-flow`/`edge-flow-dot` (animazione del flusso sui cavi — vedi §6), `node-selected`/`node-link-target`/`node-pending` (stati visivi delle card).

### Pattern Tailwind ricorrenti (osservati campionando i componenti isa)
- Pillole/chip: `glass-chip rounded-full px-2.5 py-1 text-[10px] text-muted-foreground` (badge, es. `inspector.tsx:137`).
- Pannelli: `glass-panel absolute z-40 flex flex-col rounded-3xl p-4` (es. `inspector.tsx:90`).
- Bottoni icona circolari: `glass-chip flex size-8 items-center justify-center rounded-full` (barra toolbar, `solutions.$solutionId.etl.tsx:156` ecc.).
- CTA primaria: `gradient-brand ... rounded-full px-4 text-xs font-semibold text-brand-foreground` (bottone Run, `solutions.$solutionId.etl.tsx:196`).

### Componenti UI di base riutilizzabili (`src/components/ui/**`, 46 file shadcn)
Skim degli export, non lettura approfondita di ognuno. Rilevanti per l'implementazione del prototipo:
- **Rilevanti e già usati altrove nel prodotto:** `dropdown-menu.tsx`, `popover.tsx`, `tooltip.tsx`, `dialog.tsx`, `select.tsx` — utilizzabili per pannelli impostazioni/menu.
- **Rilevanti ma NON usati oggi nel canvas (disponibili come dipendenza):** `resizable.tsx` (`react-resizable-panels` — utile per pannelli ridimensionabili/agganciabili), `tabs.tsx` (Radix Tabs — utile per "schede quando due pannelli condividono un bordo", §6 item 16).
- Il resto (`accordion`, `avatar`, `carousel`, `calendar`, `chart` [wrapper Recharts], `sidebar`, `sonner`, ecc.) è generico shadcn, non specifico al canvas ETL.

**Nota:** il canvas ETL non usa quasi nessuno di questi componenti shadcn: Inspector, Data Preview, Tool Palette e i menu impostazioni sono tutti markup/CSS scritti a mano (glass-panel/glass-chip), non composizioni di `src/components/ui/**`. Il menu custom `isa-menu.tsx` (non Radix) è il meccanismo di popover realmente usato nel canvas.

---

## 5. Test

**41 test totali, 4 file, tutti sotto `src/canvas/**`** (nessun test altrove in `src/` — confermato con `find`).

| File | Test | Cosa copre |
|---|---|---|
| `src/canvas/layout/__tests__/canvasBounds.test.ts` | 12 | Bounds invariati senza pannelli; riduzione bounds per pannelli "stretch" su ciascun lato; nessuna riduzione per pannelli "center"; somma di più pannelli aperti; nessuna larghezza/altezza negativa; conversione bounds→rect |
| `src/canvas/layout/__tests__/dropZones.test.ts` | 10 | Drop-zone con/senza ostacoli; validità di un punto dentro/fuori dal bounding box della palette e dentro/fuori dai bounds |
| `src/canvas/layout/__tests__/panelRegistry.test.ts` | 10 | Registry↔store: stato iniziale, azioni `PANEL_OPENED`/`CLOSED`/`TOGGLED`/`RESIZED`, azione su id non registrato ignorata |
| `src/canvas/__tests__/panelPositioning.test.ts` | 9 | Fase 2A: posizionamento pannello stretch da solo; auto-esclusione; conversione physical↔surface a zoom≠1; bottom evita right aperto; clamp su container piccolo; round-trip zoom 0.5/1/1.8 |

**Cosa NON è coperto (copertura zero):**
- `workflow-canvas.tsx` (5037 righe: drag, drop, combine, routing ortogonale, zoom, menu contestuale) — **nessun test**.
- `etl-workflow.tsx` (store, undo/redo, `autoLayout`, `pipelineOrder`, `connect` senza controllo cicli) — **nessun test**.
- `etl-schema.ts` (`analyzeNode`, simulazione schema/errori) — **nessun test**.
- `etl-catalog.ts`, `etl-node-config.ts`, `etl-node-size.ts`, `etl-bubble.ts`, `etl-display.ts` — **nessun test**.
- `inspector.tsx`, `data-preview.tsx`, `tool-palette.tsx`, i 3 settings-panels — **nessun test**.
- `solutions-store.tsx`, `theme.tsx`, routing — **nessun test**.

In sintesi: la sola parte testata è la fondazione geometrica pura (`src/canvas/layout/**` + `store/` + `hooks/` + `components/`, tutta logica senza rendering di card/drag). **Tutta la logica interattiva del canvas ETL (il file più grande e più complesso del repository) non ha copertura automatica.**

---

## 6. Confronto con il prototipo

Legenda: **Sì** = implementato e verificato nel codice prodotto; **Parziale** = esiste un meccanismo correlato ma diverso/incompleto rispetto al prototipo; **No** = nessuna traccia trovata.

| # | Funzionalità | Esiste nel codice? | Dove | Note |
|---|---|---|---|---|
| 1 | Nodi dataset, lavorazione e box combinato | **Sì** | `etl-catalog.ts` (19 tipi, 5 categorie); `EtlNode.groupId` per i box combinati | Il "box combinato" nel prodotto è un gruppo di card con lo stesso `groupId`, renderizzate come card separate + un contorno "bubble" attorno (`etl-bubble.ts`), non una singola card fisica come nel prototipo (`.icon-wrap` con griglia icone interna, prototipo righe 629-653). Concetto equivalente, implementazione visiva diversa |
| 2 | Fusione per trascinamento (drag-to-merge) | **Sì** | `workflow-canvas.tsx`: `pickCombineCandidate` (riga 418), stato `combinePreview`, drop → `onGroupNodes` (riga 2991) → `groupNodes()` in `etl-workflow.tsx` | Il prototipo ha la stessa meccanica con `relation()`/`performMerge()` (righe 1774-1917) e animazioni FLIP+particelle dedicate (`.fusing`, `.burst`); il prodotto non è stato confrontato animazione-per-animazione |
| 3 | Collegamento e generazione dell'output | **Sì** | `workflow-canvas.tsx` righe ~3140-3193 (drag da porta, drop su nodo target) → `onConnect` → `connect()` in `etl-workflow.tsx` | Una porta di input accetta una sola connessione (rimpiazza quella esistente); nessun controllo oltre self-loop (vedi riga 5) |
| 4 | Output parziale del Join | **Parziale** | Meccanismo generico: `etl-schema.ts:49` genera `Input "<port>" non collegato.` per qualunque porta di un nodo non connessa (quindi anche per un Join con solo la tabella sinistra); questo fa scattare lo stato visivo "error" sulla card (`workflow-canvas.tsx:4331-4340`) e appare in Inspector come errore | **Nessuna resa visiva dedicata** come nel prototipo (nessun match per "partial"/"parziale" in tutto `src/`): il prototipo calcola una vera capienza (`boxCapacity()`, riga 1612) e mostra una card con una "fetta" a metà vuota e animata (`.card.partial`, `.icon-wrap.split`, CSS righe 655-671; `renderOutputIcon()` riga 1616); il prodotto mostra solo un errore generico indistinguibile da un campo obbligatorio mancante |
| 5 | Capienza e regole del grafo: cicli, respingimenti | **No** (cicli) / **Parziale** (respingimenti) | `connect()` in `etl-workflow.tsx` (righe 357-375) controlla solo `fromNode !== toNode` e deduplica edge identici — **nessun controllo di ciclo**: A→B→C→A verrebbe accettato senza errori | Il prototipo rifiuta esplicitamente un collegamento che creerebbe un ciclo tramite una DFS dedicata (`reaches()`, riga 1888) centralizzata in `linkRefusal()` (riga 1902) con messaggio in italiano. "Respingimenti" fisici (le card che si scostano per non sovrapporsi) esistono nel prodotto (`pushOutOfOverlap`, `COLLISION_GAP`) ma sono anti-overlap geometrico, non regole di validità del grafo — nel prototipo la stessa idea è `displace()` (riga 1939), usata proprio quando due dataset non fondibili si sovrappongono |
| 6 | Instradamento ortogonale dei cavi | **Sì** | `workflow-canvas.tsx`: `getBestRoute` (riga 1115), con `routeCandidate`, `nearbyObstacles`, `pickBlockingObstacle`, `detourShapesAround`, `segmentHitsNode`, penalità `ROUTE_BLOCKED` per percorsi che attraversano una card | Il prototipo usa un algoritmo dedicato con nomi diversi ma la stessa famiglia di tecnica — brute-force su porta×forma con punteggio (`chooseRoute`/`shapeCandidates`, righe 1214-1277), isteresi anti-jitter, corsie separate per cavi paralleli (`lanes`, riga 1365) — non è la stessa base di codice, ma è concettualmente più elaborato del prodotto su questo punto specifico (isteresi e lane-splitting non risultano presenti nel prodotto — non verificato riga per riga in `getBestRoute`, "(dedotto)" per l'assenza di questi due dettagli specifici) |
| 7 | Animazione del flusso | **Sì** | CSS: utility `edge-flow`/`edge-flow-dot` (`styles.css` righe 288-332, `@keyframes isa-edge-flow-dot`, tecnica `offset-path`/`offset-distance`, nessun `requestAnimationFrame`); applicate in `workflow-canvas.tsx` righe 4090/4109/4125 | Già presente e funzionante nel prodotto, non solo nel prototipo — il prototipo usa invece un loop `requestAnimationFrame` che campiona `getPointAtLength` (`animateBubbles`, riga 1480): stessa resa finale, tecnica diversa (CSS vs rAF) |
| 8 | Pan, zoom e minimappa | **Parziale** (solo zoom) | `zoom` è uno `useState` in `workflow-canvas.tsx` (riga 1668); **nessun pan** (nessun `panX`/`panY`/`setPan` in tutto `src/`, confermato via grep); **nessuna minimappa** (zero match per "minimap" fuori dal prototipo, confermato via grep) | Il prototipo ha un mondo (2600×1600) più grande della viewport con `view.x/y/z`, gestione rotella per pan e Cmd/Ctrl+rotella per zoom attorno al puntatore (`zoomAt`, riga 4104), `fitView()` (riga 4111), più una minimappa funzionante con viewport draggabile e click-to-jump (`renderMinimap`, riga 4134) |
| 9 | Modalità Libero e Organizzato | **Parziale** | Il prodotto ha `LayoutMode = "auto" \| "manual"` (`etl-workflow.tsx:28`): "auto" ricalcola una disposizione a colonne (`autoLayout`) ad ogni commit, "manual" lascia le posizioni utente | Meccanica diversa dal prototipo: lì "Organizzato" (`mode="grid"`) tiene i nodi in **postazioni fisse che si scambiano di posto** (`computeSlots`/`nearestSlot`/`slotTaken`, righe 3910-3920+2092-2100), non un ricalcolo a colonne per profondità di pipeline; "Libero" nel prototipo ha separazione continua da sovrapposizione (`resolveOverlaps`, riga 1551). Concettualmente analoghe (automatico vs manuale) ma **non lo stesso algoritmo** |
| 10 | Riordino automatico | **Sì** (ma diverso) | `autoLayout()` in `etl-workflow.tsx` (righe 94-151): colonne per profondità nella pipeline, larghezza colonna = card più larga, passo adattivo alla dimensione reale delle card | Il prototipo ha un algoritmo di layout a strati fatto in casa, tipo Sugiyama (`autoLayout()`, righe 4220-4317: profondità BFS, pull-back colonne sorgente, ordinamento a baricentro, rilassamento simmetrico) — stesso concetto (layout automatico a livelli), implementazione indipendente e più elaborata nel prototipo |
| 11 | Annulla e ripristina | **Sì** | `etl-workflow.tsx`: `undo`/`redo` (righe 400-426), `past[]`/`future[]` cap 40, bottoni toolbar in `solutions.$solutionId.etl.tsx` (`Undo2`/`Redo2`, righe 151-168) | **Solo via bottone**, nessuna scorciatoia da tastiera (Cmd/Ctrl+Z non gestito — vedi item 12). Il prototipo ha history identica per concetto (`HIST_MAX=50` vs 40) più scorciatoie da tastiera e diffing per animare pop-out/pop-in dei nodi che cambiano (`restoreSnapshot`, riga 4360) |
| 12 | Selezione multipla e tastiera | **No** | `WorkflowCanvas` ha `selectedId: string \| null` — **singolare**, non `selectedIds`/`Set` (confermato: zero match per "selectedIds"/"multiSelect" in `src/`); l'unico `keydown` gestito è `Escape` (riga 1350, per chiudere/annullare un'interazione in corso) | Il prototipo ha selezione multipla completa (rubber-band + Shift-click + Ctrl/Cmd-click, `selectedSet`, righe 1968-4210) e scorciatoie da tastiera ricche: Delete/Backspace elimina, freccie spostano (con step griglia su Shift), Cmd/Ctrl+D duplica, Cmd/Ctrl+A seleziona tutto, Cmd/Ctrl+Z / Shift+Z redo (righe 4629-4650). **Gap netto** |
| 13 | Cassetta degli strumenti con caricamento CSV | **Parziale** | `tool-palette.tsx` (356 righe): elenco risorse per categoria, drag-to-canvas via Pointer Events, doppio click per aggiungere, ancorabile su 4 lati — la "cassetta" esiste. Il nodo `source.file` nel catalogo ha un campo `format` con opzioni `csv`/`parquet`, ma è **un `<select>` + un campo testo per il percorso file**, non un vero upload | **Nessun caricamento file reale**: zero match per "csv"/"upload"/`<input type="file">`/`FileReader` in tutto `src/` (verificato via grep). Il prototipo ha un vero `<input type="file" accept=".csv,.tsv,.txt">` + `FileReader` + parser CSV client-side con sniffing di delimitatore/tipo colonna (righe 810, 4672, 4696-4698, 4921-4926) |
| 14 | Catalogo delle funzioni e dei parametri | **Sì** | `etl-catalog.ts` (427 righe): 19 tipi di nodo in 5 categorie (Sources/Transform/Combine/Aggregate/Output), ciascuno con campi tipizzati (`text`/`select`/`textarea`, `required`, `placeholder`, `defaultValue`) | Il prototipo ha un catalogo analogo ma più granulare: `PARAM_DEFS` (riga 2445) per lo schema base per-tipo-operazione, `MULTI_DEFS` (riga 2538) per operazioni con righe ripetibili (cast, rename, fillNa, ecc.), ciascuna con una funzione `sum()` di riepilogo. Il prodotto non ha un equivalente del riepilogo auto-generato |
| 15 | Inspector verticale e orizzontale a tre colonne | **No** | `inspector.tsx` (272 righe): **una sola colonna verticale** (`<aside>`, `flex flex-col`), fisso sul lato destro (`side: "right"` in `panelRegistry.ts`), nessuna variante orizzontale, nessun layout a tre colonne | Il prototipo ha un vero sistema master-detail: quando ancorato in orizzontale (top/bottom) diventa 3 colonne fianco a fianco — `.insp-general` + `.insp-master` (elenco condizioni) + `.insp-detail` (dettaglio) — via `arrangeMasterDetail()` (righe 3547-3601) che sposta fisicamente nodi DOM, con CSS dedicato (righe 344-373). **Gap netto**: il prodotto non ha alcuna logica di orientamento per l'Inspector |
| 16 | Schede quando due pannelli condividono un bordo | **No** | Nessuna traccia: `panelRegistry.ts` fissa un lato per pannello (Tool Palette: top/center; Inspector: right/stretch; Data Preview: bottom/stretch) — non sono ridisponibili dall'utente, quindi non possono mai "condividere un bordo" | Il prototipo gestisce esplicitamente questo caso con `renderTabs()` (righe 4771-4801): quando due pannelli finiscono sullo stesso lato, iniettano una barra di tab dimensionata al massimo dei due, e `switchTab()` (riga 4803) cambia contenuto con un cross-fade senza ridimensionare il canvas. **Gap netto** |
| 17 | Connettori logici e gruppi di condizioni | **No** | `filter-panel.tsx` (167 righe): **una sola colonna, un solo confronto** (nome colonna + lista di valori da mantenere via chip multi-select) — nessun AND/OR, nessuna condizione multipla, nessun gruppo | Il prototipo ha 6 operatori logici (AND/OR/XOR/NAND/NOR/XNOR, riga 3299 con `LOGIC_HELP` esplicativo), condizioni multiple concatenabili con connettori indipendenti per riga, gruppi annidati/raggruppabili (`groupPair`/`splitAt`, righe 3330-3343), e un'anteprima live dell'espressione booleana risultante (`groupedPreview()`, riga 3371). **Gap netto**, il prodotto è molto più semplice |
| 18 | Selettore di valori | **Sì** | `filter-panel.tsx`: `PanelChipToggleList` (in `panel-controls.tsx:61`) su `getColumnDomainValues()` — multi-select a chip sui valori distinti della colonna scelta | Il prototipo ha un meccanismo più ricco: chip dei valori scelti + ricerca che filtra un dominio noto E permette di aggiungere token liberi non nel dominio (`pickerHtml`/`filterPicker`/`addTokens`, righe 2840-2892), riusato anche per le liste del lato destro di un Join. Il prodotto copre solo il caso base (dominio noto, nessun token libero) |
| 19 | Pannelli agganciabili con tacche | **No** (per Inspector/Data Preview) / **Parziale** (per Tool Palette) | La Tool Palette ha un prop/stato `paletteDock: Dock` (top/bottom/left/right) — può essere spostata su un lato, ma senza un "tacca" visiva di collasso quando chiusa (dedotto: nessun elemento con ruolo di "notch" trovato per la palette) e senza alcun collegamento al sistema `canvasStore`/`panelRegistry` (vedi §1/§2/§8). Inspector e Data Preview sono **fissi**, non spostabili dall'utente (side hardcoded in `panelRegistry.ts`) | Il prototipo ha un sistema completo di tacche (`.notch`, CSS righe 271-295): ogni pannello (toolbox, inspector) può essere trascinato su qualunque dei 4 lati (`nearestSide`, riga 4875 → `setSide()`, riga 4862), si collassa in una piccola tacca quando chiuso (`layoutNotches`, riga 4843), e un edge-highlight anticipa il punto di aggancio durante il drag. **Gap netto** per Inspector/Data Preview |
| 20 | Stati dei nodi | **Sì** (semantica diversa) | `NodeStatus = "ready" \| "running" \| "succeeded" \| "error"` (`etl-workflow.tsx:6`), mappati a colore/etichetta in `STATUS` (`workflow-canvas.tsx:181-200`); lo stato "error" viene forzato quando `analyzeNode(...).errors.length > 0` (riga 4331-4340) | Il prodotto ha stati di **ciclo di vita di esecuzione simulata** (idle→running→succeeded, guidati da `setTimeout` in `runWorkflow()`, nessun motore reale — commento esplicito in `etl-workflow.tsx:50`: "Nessuna esecuzione reale: il motore arriverà nella fase dedicata"), con "error" riusato anche per indicare configurazione incompleta. Il prototipo ha un concetto più mirato e separato di "nodo incompleto/da configurare" (`nodeState()`, riga 1447; `sweepStates()`, riga 1461; classe `.warn` + badge `.state-dot`, righe 183-186) indipendente dall'esecuzione — e un `FEATURES.flowGate` (implementato ma disattivato di default) che fermerebbe l'animazione del flusso sui cavi che toccano un nodo incompleto. Non sono la stessa cosa concettualmente, ma coprono casi simili |

**Nota generale sul prototipo:** è organizzato internamente come un banco di feature flag indipendenti e documentate (`FEATURES` object, righe 913-928: `portDrag`, `cableInsert`, `nodeStates`, `flowGate`, `panZoom`, `minimap`, `multiSelect`, `keyboard`, ciascuna con descrizione in italiano) — il prototipo stesso si descrive come costruito per provare e scartare ogni funzionalità in isolamento. Questo suggerisce un modello utile per come organizzare l'adozione di questi gap (osservazione, non un piano).

---

## 7. Dipendenze

**Presenti e usate per il canvas/prodotto attuale:**
- `lucide-react` — tutte le icone.
- `@radix-ui/react-*` (popover, dropdown-menu, dialog, select, tabs, tooltip, ecc.) — usate da `src/components/ui/**` (shadcn), ma **non** dal canvas ETL stesso (che usa `isa-menu.tsx` custom).
- `recharts` — grafici, usato in `solutions.$solutionId.dashboard.tsx` e nel wrapper `ui/chart.tsx`; **non collegato al canvas ETL** (`mini-chart.tsx`, usato nelle card delle soluzioni nella home, è SVG scritto a mano, non Recharts).
- `react-resizable-panels` — wrapper `ui/resizable.tsx` presente ma **zero importatori** in tutto `src/`: disponibile, non usato.
- `@tanstack/react-router`, `@tanstack/react-start`, `@tanstack/react-query` — routing/SSR/data-fetching applicativo, non canvas-specifico.
- `zod`, `react-hook-form`, `@hookform/resolvers` — validazione form generica (non usata nei settings-panels ETL, che gestiscono la validazione a mano via `errors[]`).

**Assenti (nessuna corrispondenza in `package.json`), verificato per nome libreria:**
- **Drag-and-drop:** nessuna (`dnd-kit`, `react-dnd`, `interact.js` assenti) — il drag di card/porte/palette è tutto Pointer Events scritti a mano (`onPointerDown`/`pointermove`/`pointerup`), esattamente come nel prototipo (vanilla JS).
- **Animazioni:** nessuna libreria (`framer-motion`, `react-spring`, `@react-spring/*` assenti) — le animazioni sono CSS (`transition`, `@keyframes`, `offset-path`) più costanti di durata/easing centralizzate in `etl-motion.ts`.
- **State management dedicato:** nessuno (`zustand`, `redux`, `jotai`, `valtio` assenti) — tutto Context+`useReducer`/`useState` o uno store esterno vanilla con `useSyncExternalStore` (§1).
- **Canvas a nodi / diagrammi:** nessuna libreria dedicata (`react-flow`/`reactflow`, `konva`, `d3` assenti) — routing ortogonale, geometria bubble, drop-zone sono tutti algoritmi scritti a mano in TypeScript puro.

**Conclusione dipendenze:** il prodotto e il prototipo condividono la stessa filosofia — **nessuna libreria per drag, animazioni, canvas-a-nodi o gestione stato del workflow**: tutto hand-rolled in entrambi i casi (Pointer Events + CSS + funzioni pure). Questo significa che implementare i gap del §6 (multi-select, pan/minimappa, notches, tabs-su-bordo-condiviso, condizioni logiche) richiederebbe **scrivere nuovo codice hand-rolled equivalente**, non "collegare una libreria mancante" — a meno che una decisione di prodotto introduca deliberatamente una libreria (es. `react-resizable-panels`, già presente come dipendenza inutilizzata, o `@radix-ui/react-tabs`, idem) per una parte dei gap.

---

## 8. Rischi e vincoli

- **`workflow-canvas.tsx` è un unico file da 5037 righe** senza alcun test automatico sulla sua logica interattiva (§5). Qualunque modifica per chiudere i gap del §6 che tocchi drag/drop/routing dovrà lavorare dentro (o attorno a) questo file senza una rete di sicurezza automatica — solo verifica manuale.
- **Duplicazione di logica di drop-validity:** `src/canvas/layout/dropZones.ts` esporta `isValidDropPoint` (testato, 10 test) ma **inutilizzato** fuori dai test (verificato via grep); `workflow-canvas.tsx` ha una propria logica di validità/collisione (`pushOutOfOverlap`, controlli inline sul bounding box della palette). Estendere le regole di drop-zone (es. per far evitare alle card anche Inspector/Data Preview, dichiarato "non ancora in scope" nel README di Fase 2A) richiede capire quale delle due logiche è quella davvero attiva a runtime, e probabilmente consolidarle.
- **Wiring dichiarato ma inerte tra Tool Palette e `src/canvas/**`:** `panelRegistry.ts` dichiara una voce `"tool-palette"` completa di lato/anchor, coperta da test, ma `tool-palette.tsx` non importa nulla da `@/canvas/**` e non chiama mai `usePanelState`/`useCanvasStore` (verificato via grep: zero import). La palette gestisce apertura/ancoraggio con stato locale proprio. Chi lavora sul layer canvas potrebbe assumere, guardando solo `panelRegistry.ts`, che la palette partecipi al sistema di bounds condivisi — non è così.
- **`connect()` senza controllo di ciclo** (§6 item 5): questo è debito di correttezza pre-esistente, non introdotto da questo rapporto — un utente può oggi costruire un grafo con un ciclo tramite l'UI. `analyzeNode` ha un guard `depth > 24` che limita l'esplosione nella simulazione di schema, ma `pipelineOrder` (usato per l'ordine di esecuzione simulata in `runWorkflow`) non ha un guard equivalente esplicito (dedotto dalla lettura del codice, non testato con un caso reale — vedi §9).
- **File di documentazione fuori sincrono:** `FUNCTIONAL_CHECKS.md` descrive uno stato (Fase 1, non collegato) superato dalla Fase 2A — rischio che un futuro contributor si fidi di questo file e faccia asserzioni sbagliate sullo stato di integrazione.
- **Codice morto non rimosso:** `isa-context-menu.tsx` (320 righe) e `isa-modal.tsx` (52 righe) sono completamente inutilizzati (§2, verificato via grep) — non sono un rischio funzionale, ma aggiungono superficie da leggere/mantenere per errore.
- **Stato del workflow in `localStorage` senza namespacing per utente/ambiente oltre `solutionId`:** `isa.etl.workflow.<solutionId>` — non un problema per il gap col prototipo, ma un vincolo da conoscere se si cambia lo schema di `EtlWorkflow` (retrocompatibilità del parsing in `load()`, che già tollera solo `layout` mancante/malformato, non altri campi).
- **Nessuna esecuzione reale del workflow** (`runWorkflow` è un timer simulato) — qualunque feature del prototipo che assuma dati "veri" (es. l'anteprima parziale del Join con conteggio righe reale) nel prodotto lavora già su stime statiche (`analyzeNode`), non su esecuzione: è un vincolo architetturale pre-esistente, non specifico a questo confronto.
- **Accoppiamento forte fra `etl-node-size.ts` e `etl-workflow.tsx`:** `autoLayout()` importa `estimateNodeSize` per calcolare il passo di griglia adattivo — un cambiamento alla resa visiva delle card (dimensioni) si propaga silenziosamente al layout automatico; documentato nei commenti del codice stesso, non è un difetto ma un vincolo da rispettare.
- **Nessun token visivo condiviso tra prodotto e prototipo** (§4): la palette OKLCH del prodotto e la palette hex/Manrope del prototipo non hanno valori in comune, solo ruoli simili — riprodurre l'estetica del prototipo richiede decidere esplicitamente se e come riconciliare le due scale di token, non è un porting automatico.

---

## 9. Domande aperte

- Il file prototipo si chiama `isa-fusion-prototype.html`, non `isa-canvas-prototype.html` come nominato nell'incarico — è lo stesso prototipo di riferimento voluto, o ne esiste un altro non ancora caricato in `docs/prototype/`?
- `src/server.ts` (61 righe) non è stato letto integralmente in nessuna delle ricognizioni (solo dedotto dal nome/convenzioni TanStack Start) — non risulta specifico al canvas ETL, quindi non approfondito, ma resta da verificare se contenga logica rilevante.
- Non è stato verificato a runtime (solo da lettura statica del codice) cosa succeda realmente creando un ciclo A→B→C→A tramite l'UI — se il prodotto va in loop infinito, si blocca silenziosamente, o degrada in modo sicuro. `analyzeNode` ha un guard `depth > 24` ma non è chiaro se copra anche il percorso di `pipelineOrder`.
- Non è chiaro se la mancanza di controllo-cicli in `connect()` sia una scelta deliberata (dedotto: nessun commento nel codice ne parla, a differenza di altre scelte documentate esplicitamente nei commenti di `etl-workflow.tsx`) o una lacuna non ancora notata.
- Non è chiaro se la mancata integrazione della Tool Palette nel sistema `canvasStore`/`panelRegistry` (§1, §2, §8) sia deliberatamente fuori scope della Fase 2A (che riguardava solo Inspector/Data Preview, per come documentata) o una lacuna da chiudere in una fase successiva.
```

### `docs/inventory/INVENTARIO_2026-09-28T19-40-33Z.md` (parte 2/2)

310 righe totali

```md
- Il dettaglio esatto di tutte le opzioni di Join/Union/Lookup in `combine-panel.tsx` (321 righe) e la lista completa dei 19 nodi con tutti i loro campi in `etl-catalog.ts` (427 righe) sono stati letti solo in parte in alcune delle ricognizioni — la struttura generale è verificata, non ogni singolo campo.
- Il rapporto non ha eseguito l'app nel browser (vincolo "solo lettura, nessuna modifica" interpretato anche come "nessuna esecuzione", per non rischiare side-effect su `localStorage`/stato) — tutte le osservazioni sul comportamento a runtime (es. animazione del flusso, drag) sono dedotte dal codice sorgente, non osservate visivamente in questa sessione.

```

### `docs/inventory/lovable-1e72955.diff`

43 righe

```
diff --git a/src/components/isa/etl/tool-palette.tsx b/src/components/isa/etl/tool-palette.tsx
index 509cc05..873fb9b 100644
--- a/src/components/isa/etl/tool-palette.tsx
+++ b/src/components/isa/etl/tool-palette.tsx
@@ -23,7 +23,12 @@ function NodeChip({ node, onAdd }: { node: EtlNodeDef; onAdd: (type: string) =>
     <button
       type="button"
       draggable
-      onDragStart={(event) => event.dataTransfer.setData("application/isa-node", type)}
+      onDragStart={(event) => {
+        // Safari richiede un tipo standard oltre a quello custom.
+        event.dataTransfer.effectAllowed = "copy";
+        event.dataTransfer.setData("application/isa-node", type);
+        event.dataTransfer.setData("text/plain", `isa-node:${type}`);
+      }}
       onClick={() => onAdd(type)}
       title={description}
       aria-label={`Aggiungi ${label}`}
diff --git a/src/components/isa/etl/workflow-canvas.tsx b/src/components/isa/etl/workflow-canvas.tsx
index 6b587c9..07be47e 100644
--- a/src/components/isa/etl/workflow-canvas.tsx
+++ b/src/components/isa/etl/workflow-canvas.tsx
@@ -216,10 +216,17 @@ export function WorkflowCanvas({
       ref={boxRef}
       data-palette-workspace
       className="glass-soft relative min-h-0 min-w-0 flex-1 overflow-hidden rounded-3xl"
-      onDragOver={(e) => e.preventDefault()}
+      onDragEnter={(e) => e.preventDefault()}
+      onDragOver={(e) => {
+        e.preventDefault();
+        e.dataTransfer.dropEffect = "copy";
+      }}
       onDrop={(e) => {
         e.preventDefault();
-        const type = e.dataTransfer.getData("application/isa-node");
+        const plain = e.dataTransfer.getData("text/plain");
+        const type =
+          e.dataTransfer.getData("application/isa-node") ||
+          (plain.startsWith("isa-node:") ? plain.slice("isa-node:".length) : "");
         if (!type) return;
         const p = toLocal(e.clientX, e.clientY);
         onAddAt(type, clampX(p.x - NODE_W / 2), clampY(p.y - NODE_H / 2));
```

