# 01-canvas.md

File in questo blocco:

- `src/canvas/FUNCTIONAL_CHECKS.md`
- `src/canvas/README.md`
- `src/canvas/__tests__/panelPositioning.test.ts`
- `src/canvas/components/CanvasContainer.tsx`
- `src/canvas/hooks/useCanvasBounds.ts`
- `src/canvas/hooks/usePanelState.ts`
- `src/canvas/layout/__tests__/canvasBounds.test.ts`
- `src/canvas/layout/__tests__/dropZones.test.ts`
- `src/canvas/layout/__tests__/panelRegistry.test.ts`
- `src/canvas/layout/canvasBounds.ts`
- `src/canvas/layout/dropZones.ts`
- `src/canvas/layout/panelRegistry.ts`
- `src/canvas/layout/surfacePanels.ts`
- `src/canvas/store/canvasStore.tsx`

---

### `src/canvas/FUNCTIONAL_CHECKS.md`

75 righe

```md
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
```

### `src/canvas/README.md`

143 righe

```md
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
```

### `src/canvas/__tests__/panelPositioning.test.ts`

179 righe

```ts
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
```

### `src/canvas/components/CanvasContainer.tsx`

108 righe

```tsx
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
```

### `src/canvas/hooks/useCanvasBounds.ts`

81 righe

```ts
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
```

### `src/canvas/hooks/usePanelState.ts`

55 righe

```ts
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
```

### `src/canvas/layout/__tests__/canvasBounds.test.ts`

205 righe

```ts
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
```

### `src/canvas/layout/__tests__/dropZones.test.ts`

150 righe

```ts
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
```

### `src/canvas/layout/__tests__/panelRegistry.test.ts`

90 righe

```ts
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
```

### `src/canvas/layout/canvasBounds.ts`

219 righe

```ts
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
```

### `src/canvas/layout/dropZones.ts`

59 righe

```ts
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
```

### `src/canvas/layout/panelRegistry.ts`

61 righe

```ts
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
```

### `src/canvas/layout/surfacePanels.ts`

86 righe

```ts
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
```

### `src/canvas/store/canvasStore.tsx`

123 righe

```tsx
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
```

