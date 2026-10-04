# 01e-etl-canvas-f.md

File in questo blocco:

- `src/etl-canvas/panels/EtlWorkspace.tsx`
- `src/etl-canvas/panels/InspectorShell.tsx`
- `src/etl-canvas/panels/Toolbox.tsx`
- `src/etl-canvas/panels/actions.ts`
- `src/etl-canvas/panels/csv.ts`
- `src/etl-canvas/panels/families.ts`
- `src/etl-canvas/panels/layout.ts`
- `src/etl-canvas/panels/overlayLayout.ts`
- `src/etl-canvas/panels/panels.css`
- `src/etl-canvas/panels/ui-icons.tsx`
- `src/etl-canvas/seed.ts`

---

### `src/etl-canvas/panels/EtlWorkspace.tsx`

125 righe

```tsx
/**
 * Lo spazio di lavoro: il canvas al centro, i pannelli (cassetta e Inspector)
 * agganciati ai bordi. È ciò che la rotta ETL monta al posto del solo canvas.
 *
 * Come il canvas, si monta solo nel browser: sul server e nel primo rendering
 * di idratazione produce lo stesso segnaposto (i pannelli dipendono dallo
 * stato salvato nel browser).
 */
import { useEffect, useMemo, useRef, useState, useSyncExternalStore } from "react";
import type { PointerEvent as ReactPointerEvent } from "react";
import type { EtlStore } from "../../etl-store";
import { EtlCanvas } from "../EtlCanvas";
import { Icon } from "../icons";
import type { CanvasDropPayload } from "../drop";
import { createInteractionController } from "../interaction";
import { createPanelActions, followInspector } from "./actions";
import { DockLayout } from "./Dock";
import { familyOfType } from "./families";
import { InspectorShell } from "./InspectorShell";
import { Toolbox } from "./Toolbox";

const noopSubscribe = () => () => {};

interface Ghost {
  readonly x: number;
  readonly y: number;
  readonly payload: CanvasDropPayload;
}

export function EtlWorkspace(props: { store: EtlStore }) {
  const { store } = props;
  const isClient = useSyncExternalStore(
    noopSubscribe,
    () => true,
    () => false,
  );
  const [controller] = useState(() => createInteractionController(store));
  const actions = useMemo(() => createPanelActions(store), [store]);
  const [ghost, setGhost] = useState<Ghost | null>(null);
  const hostRef = useRef<HTMLDivElement>(null);
  const cleanup = useRef<(() => void) | null>(null);

  // l'Inspector si apre con la selezione e si chiude con la deselezione
  useEffect(() => followInspector(store, controller, actions), [store, controller, actions]);
  useEffect(() => () => cleanup.current?.(), []);

  /** Trascinamento di una voce della cassetta (prototipo, righe 4939-5067): un nodo esterno, con la stessa anteprima del trascinamento tra nodi. */
  const onItemPointerDown = (payload: CanvasDropPayload, e: ReactPointerEvent<HTMLElement>) => {
    if (e.button !== 0) return;
    e.preventDefault();
    setGhost({ x: e.clientX, y: e.clientY, payload });
    const stagePoint = (cx: number, cy: number) => {
      const r = hostRef.current?.querySelector(".ec-stage")?.getBoundingClientRect();
      if (!r) return null;
      const inside = cx >= r.left && cx <= r.right && cy >= r.top && cy <= r.bottom;
      return inside ? { x: cx - r.left, y: cy - r.top } : null;
    };
    const move = (ev: PointerEvent) => {
      setGhost({ x: ev.clientX, y: ev.clientY, payload });
      controller.hoverExternal(payload, stagePoint(ev.clientX, ev.clientY));
    };
    const stop = () => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
      window.removeEventListener("pointercancel", abort);
      cleanup.current = null;
      setGhost(null);
    };
    const up = (ev: PointerEvent) => {
      stop();
      controller.dropExternal(payload, stagePoint(ev.clientX, ev.clientY));
    };
    const abort = () => {
      stop();
      controller.dropExternal(payload, null);
    };
    cleanup.current = abort;
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
    window.addEventListener("pointercancel", abort);
  };

  return (
    <div ref={hostRef} className="ec-workspace-host">
      {isClient ? (
        <DockLayout
          store={store}
          actions={actions}
          controller={controller}
          canvas={(overlay) => (
            <EtlCanvas store={store} controller={controller} minHeight={0} overlay={overlay} />
          )}
          content={{
            tools: ({ side }) => (
              <Toolbox
                store={store}
                side={side}
                onClose={() => actions.close("tools")}
                onItemPointerDown={onItemPointerDown}
              />
            ),
            insp: ({ side }) => (
              <InspectorShell store={store} side={side} onClose={() => actions.close("insp")} />
            ),
          }}
          overlay={
            ghost ? (
              <div
                className={"ec-ghost" + (ghost.payload.component === "dataset" ? " ec-source" : "")}
                data-testid="ec-ghost"
                data-family={familyOfType(ghost.payload.component)}
                style={{ left: ghost.x - 44, top: ghost.y - 44 }}
              >
                <Icon id={ghost.payload.component} />
              </div>
            ) : null
          }
        />
      ) : (
        <EtlCanvas store={store} />
      )}
    </div>
  );
}
```

### `src/etl-canvas/panels/InspectorShell.tsx`

40 righe

```tsx
/**
 * Il guscio dell'Inspector (Fase 6a): si apre e si chiude come gli altri
 * pannelli; il suo contenuto è per ora solo il nome del nodo selezionato.
 * Campi, layout a colonne e selettori sono della Fase 6b.
 */
import type { EtlStore, Side } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import { CloseArrow } from "./ui-icons";

export function InspectorShell(props: { store: EtlStore; side: Side; onClose: () => void }) {
  const { store, side } = props;
  const nodeId = useEtlState((s) => s.inspector.nodeId, store);
  const name = useEtlState(
    (s) => (s.inspector.nodeId ? s.graph.cards[s.inspector.nodeId]?.name : undefined),
    store,
  );
  return (
    <div className="ec-tb-inner" data-testid="ec-inspector">
      <div className="ec-tb-head">
        <div className="ec-tb-title">Inspector</div>
        <button
          type="button"
          className="ec-close-btn"
          aria-label="Nascondi l’inspector"
          onClick={props.onClose}
        >
          <CloseArrow side={side} />
        </button>
      </div>
      {nodeId ? (
        <div className="ec-insp-name" data-testid="ec-inspector-name">
          {name ?? nodeId}
        </div>
      ) : (
        <div className="ec-tb-empty">Nessun nodo selezionato</div>
      )}
    </div>
  );
}
```

### `src/etl-canvas/panels/Toolbox.tsx`

166 righe

```tsx
/**
 * La cassetta degli strumenti (prototipo, `buildPalette` righe 4727-4752, e i
 * gestori 4912-4937). Le sezioni e le voci NON sono scritte qui: derivano dal
 * catalogo di etl-core (`SECTIONS`, `META`), così un'operazione aggiunta al
 * dominio compare da sola. La sezione Dataset mostra la libreria di etl-store
 * e il caricamento di un CSV.
 */
import { useRef, useState } from "react";
import type { PointerEvent as ReactPointerEvent } from "react";
import { META, SECTIONS } from "../../etl-core";
import type { ComponentId } from "../../etl-core";
import type { EtlStore, Side } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import { Icon } from "../icons";
import { FAMILY_OF_SECTION } from "./families";
import type { CanvasDropPayload } from "../drop";
import { loadCsvFile } from "./csv";
import { ChevronIcon, CloseArrow, UploadIcon } from "./ui-icons";

export interface ToolboxProps {
  readonly store: EtlStore;
  readonly side: Side;
  readonly onClose: () => void;
  /** Inizio del trascinamento di una voce (il canvas ne mostra l'anteprima): vedi EtlWorkspace. */
  readonly onItemPointerDown: (
    payload: CanvasDropPayload,
    e: ReactPointerEvent<HTMLElement>,
  ) => void;
}

function Item(props: {
  type: ComponentId;
  label: string;
  meta?: string | undefined;
  lib?: string | undefined;
  family?: string | undefined;
  onPointerDown: ToolboxProps["onItemPointerDown"];
}) {
  const { type, lib } = props;
  const payload: CanvasDropPayload = lib
    ? { component: type, libraryId: lib }
    : { component: type };
  return (
    <div
      className={"ec-pal-item" + (type === "dataset" ? " ec-source" : "")}
      data-type={type}
      data-lib={lib}
      data-family={props.family}
      onPointerDown={(e) => props.onPointerDown(payload, e)}
    >
      <div className="ec-pal-chip">
        <Icon id={type} />
      </div>
      <div className="ec-pal-label">{props.label}</div>
      {props.meta ? <div className="ec-lib-meta">{props.meta}</div> : null}
    </div>
  );
}

export function Toolbox(props: ToolboxProps) {
  const { store, side } = props;
  const library = useEtlState((s) => s.library, store);
  const horiz = side === "top" || side === "bottom";
  const [open, setOpen] = useState<Readonly<Record<string, boolean>>>(() =>
    Object.fromEntries(SECTIONS.map((s) => [s.id, true])),
  );
  const [status, setStatus] = useState<string | null>(null);
  const fileRef = useRef<HTMLInputElement>(null);

  const onFile = async (file: File | undefined) => {
    if (!file) return;
    const outcome = await loadCsvFile(store, file);
    setStatus(outcome.message);
    if (outcome.ok) setOpen((o) => ({ ...o, data: true }));
  };

  return (
    <div className="ec-tb-inner" data-testid="ec-toolbox">
      <div className="ec-tb-head">
        <div className="ec-tb-title">Strumenti</div>
        <button
          type="button"
          className="ec-close-btn"
          aria-label="Nascondi la cassetta degli strumenti"
          onClick={props.onClose}
        >
          <CloseArrow side={side} />
        </button>
      </div>
      {SECTIONS.map((sec) => {
        const isOpen = horiz || open[sec.id] !== false;
        return (
          <div key={sec.id} className={"ec-tb-sec" + (isOpen ? " ec-open" : "")} data-sec={sec.id}>
            <button
              type="button"
              className="ec-tb-sec-head"
              aria-expanded={isOpen}
              onClick={() => setOpen((o) => ({ ...o, [sec.id]: !(o[sec.id] !== false) }))}
            >
              <span className="ec-chev">
                <ChevronIcon />
              </span>
              <span className="ec-tb-sec-name">{sec.name}</span>
            </button>
            <div className="ec-tb-sec-body">
              {sec.items ? (
                sec.items.map((t) => (
                  <Item
                    key={t}
                    type={t}
                    label={META[t].label}
                    family={FAMILY_OF_SECTION[sec.id]}
                    onPointerDown={props.onItemPointerDown}
                  />
                ))
              ) : (
                <>
                  <button
                    type="button"
                    className="ec-tb-upload"
                    onClick={() => fileRef.current?.click()}
                  >
                    <UploadIcon />
                    Carica dataset
                  </button>
                  {library.length ? (
                    library.map((lb) => (
                      <Item
                        key={lb.id}
                        type="dataset"
                        lib={lb.id}
                        label={lb.name}
                        meta={`${lb.columns.length} col · ${lb.rows} righe`}
                        onPointerDown={props.onItemPointerDown}
                      />
                    ))
                  ) : (
                    <div className="ec-tb-empty">Nessun dataset caricato</div>
                  )}
                </>
              )}
            </div>
          </div>
        );
      })}
      {status ? (
        <div className="ec-tb-status" role="status">
          {status}
        </div>
      ) : null}
      <input
        ref={fileRef}
        type="file"
        accept=".csv,.tsv,.txt"
        hidden
        data-testid="ec-file-input"
        onChange={(e) => {
          const input = e.currentTarget;
          void onFile(input.files?.[0]);
          input.value = "";
        }}
      />
    </div>
  );
}
```

### `src/etl-canvas/panels/actions.ts`

88 righe

```ts
/**
 * Azioni sui pannelli: comandi di etl-store (`setPanel`) più la regola
 * dell'Inspector che segue la selezione. Nessuna logica di dominio e nessun
 * DOM. La vista non si compensa qui: dopo ogni cambio la tiene visibile
 * `keepVisible` (layout.ts), chiamata da chi misura l'area.
 */
import type { CommandResult, EtlStore, PanelKey, Side } from "../../etl-store";
import type { InteractionController } from "../interaction";

export interface PanelActions {
  open(key: PanelKey): CommandResult;
  close(key: PanelKey): CommandResult;
  /** Sposta il pannello su un altro bordo: si chiude, si sposta e si riapre (prototipo, `setSide`). */
  moveTo(key: PanelKey, side: Side): CommandResult;
  /**
   * Apertura automatica dell'Inspector (clic su un nodo). Se sostituisce la
   * cassetta aperta, lo ricorda: alla chiusura automatica la cassetta si
   * riapre.
   */
  autoOpenInspector(): void;
  /** Chiusura automatica dell'Inspector (deselezione): riapre la cassetta se era stata sostituita. */
  autoCloseInspector(): void;
}

export function createPanelActions(store: EtlStore): PanelActions {
  // la cassetta aperta che l'apertura automatica dell'Inspector ha sostituito
  let replacedTools = false;
  const apply = (payload: { panel: PanelKey; open?: boolean; side?: Side }): CommandResult =>
    store.dispatch({ type: "setPanel", payload });
  // qualunque azione esplicita dell'utente su un pannello azzera la memoria
  const explicit = (payload: { panel: PanelKey; open?: boolean; side?: Side }): CommandResult => {
    replacedTools = false;
    return apply(payload);
  };
  return {
    open: (key) => explicit({ panel: key, open: true }),
    close: (key) => explicit({ panel: key, open: false }),
    moveTo: (key, side) => explicit({ panel: key, side }),
    autoOpenInspector() {
      const { panels } = store.getState();
      if (panels.insp.open) return;
      replacedTools = panels.tools.open;
      apply({ panel: "insp", open: true });
    },
    autoCloseInspector() {
      const { panels } = store.getState();
      const restore = replacedTools;
      replacedTools = false;
      if (!panels.insp.open) return;
      apply({ panel: "insp", open: false });
      if (restore) apply({ panel: "tools", open: true });
    },
  };
}

/**
 * L'Inspector segue la selezione (prototipo: `selectCard` apre, `deselect`
 * chiude — righe 2694-2727), con due differenze volute (Fase 6a.2):
 *
 * - si apre solo al CLIC su un nodo (rilascio senza trascinamento, con un solo
 *   nodo selezionato): mai alla pressione, durante un trascinamento, un
 *   riquadro di selezione, una selezione multipla o dopo un rilascio dalla
 *   cassetta;
 * - si chiude quando non c'è più un nodo nell'inspector (deselezione) e, se
 *   aveva sostituito la cassetta, la riapre.
 *
 * Se l'utente lo chiude con un nodo ancora selezionato, resta chiuso fino al
 * prossimo clic. Restituisce la funzione per smettere di ascoltare.
 */
export function followInspector(
  store: EtlStore,
  controller: Pick<InteractionController, "subscribeClick">,
  actions: PanelActions,
): () => void {
  let had = store.getState().inspector.nodeId !== null;
  const offClick = controller.subscribeClick(() => actions.autoOpenInspector());
  const offStore = store.subscribe(() => {
    const has = store.getState().inspector.nodeId !== null;
    if (has === had) return;
    had = has;
    if (!has) actions.autoCloseInspector();
  });
  return () => {
    offClick();
    offStore();
  };
}
```

### `src/etl-canvas/panels/csv.ts`

37 righe

```ts
/**
 * Caricamento di un dataset dalla cassetta (prototipo, righe 4921-4937): la
 * lettura e la deduzione dei tipi sono `parseCSV` di etl-core, la libreria è
 * quella di etl-store (`loadCsv` → comando `loadDataset`, che conserva solo i
 * metadati: nome, percorso, colonne, righe — mai il contenuto del file).
 */
import type { EtlStore } from "../../etl-store";

export interface CsvLoadOutcome {
  readonly ok: boolean;
  /** Messaggio per l'utente (prototipo, riga 4934 e 4927). */
  readonly message: string;
}

/** Carica il testo di un CSV già letto. */
export function loadCsvText(store: EtlStore, fileName: string, text: string): CsvLoadOutcome {
  const result = store.loadCsv(text, fileName);
  if (!result.ok) return { ok: false, message: result.reason };
  const item = store.getState().library.at(-1);
  return {
    ok: true,
    message: item
      ? `${fileName} caricato: ${item.columns.length} colonne, ${item.rows} righe. Trascinalo sul canvas.`
      : `${fileName} caricato.`,
  };
}

/** Legge un file scelto dall'utente (solo nel browser: `FileReader`) e lo carica. */
export function loadCsvFile(store: EtlStore, file: File): Promise<CsvLoadOutcome> {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = () => resolve(loadCsvText(store, file.name, String(reader.result ?? "")));
    reader.onerror = () => resolve({ ok: false, message: "Il file non si può leggere" });
    reader.readAsText(file);
  });
}
```

### `src/etl-canvas/panels/families.ts`

17 righe

```ts
import { sectionOf } from "../../etl-core";
import type { ComponentId } from "../../etl-core";

/** Famiglia di colore di una sezione della cassetta (la stessa dei nodi sul canvas: `data-family`). */
export const FAMILY_OF_SECTION: Readonly<Record<string, string>> = {
  rows: "filter",
  xform: "transform",
  merge: "merge",
  out: "output",
};

/** Famiglia di un componente, o `undefined` per il dataset. */
export function familyOfType(type: ComponentId): string | undefined {
  const section = sectionOf(type);
  return section ? FAMILY_OF_SECTION[section.id] : undefined;
}
```

### `src/etl-canvas/panels/layout.ts`

216 righe

```ts
/**
 * Geometria dei pannelli agganciabili: funzioni pure sullo stato dei pannelli
 * di etl-store (`Panels`: lato e aperto/chiuso di ciascuno). Misure del
 * prototipo (docs/prototype/isa-fusion-prototype.html, righe 4756-4900), salvo
 * la regola della vista (vedi `keepVisible`).
 *
 * Nessuna logica di dominio: misure e posizione della vista sono
 * geometria dell'interfaccia.
 */
import type { Card } from "../../etl-core";
import { CARD, LABEL_H } from "../../etl-layout";
import type { Size } from "../../etl-layout";
import type { PanelKey, Panels, Side, View } from "../../etl-store";
import type { Insets } from "../view";

/** I due pannelli, nell'ordine delle schede (prototipo, riga 4797). */
export const PANEL_KEYS: readonly PanelKey[] = ["tools", "insp"];

export const PANEL_LABEL: Readonly<Record<PanelKey, string>> = {
  tools: "Strumenti",
  insp: "Inspector",
};

/** Nome del pannello nei suggerimenti (riga 4759-4760). */
export const PANEL_NAME: Readonly<Record<PanelKey, string>> = {
  tools: "gli strumenti",
  insp: "l’inspector",
};

export const SIDE_NAME: Readonly<Record<Side, string>> = {
  left: "sinistro",
  right: "destro",
  top: "superiore",
  bottom: "inferiore",
};

export const SIDES: readonly Side[] = ["left", "right", "top", "bottom"];

/** Misure proprie di ciascun pannello: larghezza (bordi verticali) e altezza (orizzontali). Righe 4759-4760. */
export const PANEL_SIZE: Readonly<Record<PanelKey, { readonly w: number; readonly h: number }>> = {
  tools: { w: 264, h: 206 },
  insp: { w: 308, h: 300 },
};

/** Margine verso il canvas, parte della misura del pannello (CSS `.panel`, riga 227). */
export const EXTENT_PAD = 16;

/** Distanza tra due tacche sullo stesso bordo (riga 4850). */
export const NOTCH_SPREAD = 40;

export const isVertical = (side: Side): boolean => side === "left" || side === "right";

/** I due pannelli stanno sullo stesso bordo: diventano schede di un unico pannello. */
export function isGrouped(panels: Panels): boolean {
  return panels.tools.side === panels.insp.side;
}

/** Misura effettiva: in gruppo entrambi prendono la maggiore, così cambiare scheda non sposta il canvas (riga 4772). */
export function panelSize(panels: Panels, key: PanelKey): { w: number; h: number } {
  if (!isGrouped(panels)) return { ...PANEL_SIZE[key] };
  return {
    w: Math.max(PANEL_SIZE.tools.w, PANEL_SIZE.insp.w),
    h: Math.max(PANEL_SIZE.tools.h, PANEL_SIZE.insp.h),
  };
}

/** Quanto spazio sottrae al canvas un pannello aperto (riga 4763). */
export function panelExtent(panels: Panels, key: PanelKey): number {
  const s = panelSize(panels, key);
  return (isVertical(panels[key].side) ? s.w : s.h) + EXTENT_PAD;
}

/** Spazio sottratto al canvas dai pannelli aperti sul bordo `side`. */
export function openExtent(panels: Panels, side: Side): number {
  return PANEL_KEYS.filter((k) => panels[k].open && panels[k].side === side).reduce(
    (sum, k) => sum + panelExtent(panels, k),
    0,
  );
}

/** Area visibile con lo spazio dei widget in sovrimpressione già escluso (margine di sicurezza). */
export interface Visibility {
  readonly size: Size;
  readonly insets: Insets;
}

interface Box {
  readonly x1: number;
  readonly y1: number;
  readonly x2: number;
  readonly y2: number;
}

/** Rettangolo di un nodo (quadrato più etichetta) sullo schermo, con la vista data. */
function screenBox(c: Pick<Card, "x" | "y">, v: View): Box {
  const x1 = v.x + c.x * v.zoom;
  const y1 = v.y + c.y * v.zoom;
  return { x1, y1, x2: x1 + CARD * v.zoom, y2: y1 + (CARD + LABEL_H) * v.zoom };
}

function safeBox(a: Visibility): Box {
  return {
    x1: a.insets.left,
    y1: a.insets.top,
    x2: a.size.w - a.insets.right,
    y2: a.size.h - a.insets.bottom,
  };
}

/** Identificativi dei nodi interamente dentro l'area visibile sicura. */
export function visibleIds(
  cards: readonly Card[],
  view: View,
  area: Visibility,
): readonly string[] {
  const safe = safeBox(area);
  return cards
    .filter((c) => {
      const b = screenBox(c, view);
      return b.x1 >= safe.x1 && b.y1 >= safe.y1 && b.x2 <= safe.x2 && b.y2 <= safe.y2;
    })
    .map((c) => c.id);
}

/** Spostamento minimo di un intervallo [lo, hi] perché stia in [a, b]; se non entra, si allinea ad `a`. */
function shiftInto(lo: number, hi: number, a: number, b: number): number {
  if (hi - lo > b - a) return a - lo;
  if (lo < a) return a - lo;
  if (hi > b) return b - hi;
  return 0;
}

/**
 * Regola dei pannelli (Fase 6a.2, sostituisce quella del prototipo «nodi
 * fermi sullo schermo», righe 4780-4784 e 4818-4821): un pannello aperto
 * riduce l'area del canvas e non copre mai un nodo. Le posizioni nel mondo
 * non cambiano e lo zoom nemmeno; cambia solo la vista. Il canvas si
 * sposta con i suoi bordi (a sinistra e in alto il bordo avanza e i nodi
 * vanno con lui; a destra e in basso restano dove sono rispetto
 * all'origine) e, se l'area si restringe, i nodi che prima erano interamente
 * visibili e ora sarebbero fuori si riportano dentro con lo scorrimento
 * minimo. Se l'insieme non entra nell'area, si allinea al bordo di partenza
 * (sinistra, alto) e il resto resta raggiungibile con scorrimento e
 * minimappa.
 *
 * `prev` è l'area prima del cambio (apertura, chiusura, cambio di scheda,
 * spostamento della tacca, ridimensionamento), `next` quella dopo. Restituisce
 * la stessa vista se non serve scorrere.
 */
export function keepVisible(
  cards: readonly Card[],
  view: View,
  prev: Visibility,
  next: Visibility,
): View {
  const was = new Set(visibleIds(cards, view, prev));
  if (was.size === 0) return view;
  const boxes = cards.filter((c) => was.has(c.id)).map((c) => screenBox(c, view));
  const lo = {
    x: Math.min(...boxes.map((b) => b.x1)),
    y: Math.min(...boxes.map((b) => b.y1)),
  };
  const hi = {
    x: Math.max(...boxes.map((b) => b.x2)),
    y: Math.max(...boxes.map((b) => b.y2)),
  };
  const safe = safeBox(next);
  const dx = shiftInto(lo.x, hi.x, safe.x1, safe.x2);
  const dy = shiftInto(lo.y, hi.y, safe.y1, safe.y2);
  return dx === 0 && dy === 0 ? view : { ...view, x: view.x + dx, y: view.y + dy };
}

/**
 * Altezza minima del canvas, la riga centrale dello spazio di lavoro. Se i
 * pannelli in alto e in basso lasciano meno spazio, scorre il contenitore
 * dello spazio di lavoro, non la pagina.
 */
export const MIN_CANVAS_HEIGHT = 160;

/** Il bordo del canvas più vicino a un punto (riga 4851-4856). */
export function nearestSide(
  point: { readonly x: number; readonly y: number },
  rect: {
    readonly left: number;
    readonly right: number;
    readonly top: number;
    readonly bottom: number;
  },
): Side {
  const d: Record<Side, number> = {
    left: Math.abs(point.x - rect.left),
    right: Math.abs(rect.right - point.x),
    top: Math.abs(point.y - rect.top),
    bottom: Math.abs(rect.bottom - point.y),
  };
  return SIDES.slice().sort((a, b) => d[a] - d[b])[0] as Side;
}

/** Scostamento della tacca lungo il bordo: due tacche sullo stesso bordo si affiancano (riga 4850). */
export function notchOffset(panels: Panels, key: PanelKey): number {
  const mates = PANEL_KEYS.filter((k) => panels[k].side === panels[key].side);
  if (mates.length < 2) return 0;
  return mates.indexOf(key) === 0 ? -NOTCH_SPREAD : NOTCH_SPREAD;
}

/** La tacca si nasconde se il pannello è aperto o se si raggiunge dalla scheda dell'altro, aperto sullo stesso bordo (righe 4856-4858). */
export function notchHidden(panels: Panels, key: PanelKey): boolean {
  const other = PANEL_KEYS.find((k) => k !== key) as PanelKey;
  return panels[key].open || (panels[other].side === panels[key].side && panels[other].open);
}

/** Il pannello aperto sul bordo `side` (la scheda attiva), se c'è. */
export function activeTab(panels: Panels, side: Side): PanelKey | null {
  return PANEL_KEYS.find((k) => panels[k].side === side && panels[k].open) ?? null;
}
```

### `src/etl-canvas/panels/overlayLayout.ts`

249 righe

```ts
/**
 * Disposizione dei widget in sovrimpressione al canvas (minimappa, controlli
 * di zoom, suggerimento di rilascio, tacche dei pannelli chiusi): funzione
 * pura, nessun DOM. Decide le posizioni a partire dalla misura dell'area e dal
 * bordo del pannello aperto, così nessun componente scrive una posizione a
 * mano e due widget non si sovrappongono mai.
 *
 * Regole:
 * - minimappa in basso a sinistra; con il pannello in basso va in alto a
 *   sinistra (lontano dal pannello); con il pannello in alto resta in basso a
 *   sinistra;
 * - controlli di zoom in basso a destra;
 * - se un widget ne tocca un altro (area piccola) la minimappa, nell'ordine,
 *   passa all'angolo opposto, poi si riduce a un pulsante compatto (che si
 *   espande al clic), poi prova gli altri angoli; se nemmeno così c'è posto
 *   non si mostra (`rect: null`);
 * - il suggerimento prova in basso e in alto, al centro, e ovunque si evita
 *   il resto; senza posto non si mostra.
 *
 * Le misure dei widget sono quelle del CSS del canvas (`canvas.css`,
 * `panels.css`), che le prende da qui.
 */
import type { Size } from "../../etl-layout";
import type { PanelKey, Side } from "../../etl-store";
import type { Insets } from "../view";

export interface Rect {
  readonly x: number;
  readonly y: number;
  readonly w: number;
  readonly h: number;
}

export type Corner = "bl" | "br" | "tl" | "tr";

/** Distanza dei widget dal bordo dell'area (prototipo: 12 px, riga 152). */
export const OVERLAY_MARGIN = 12;
/** Spazio minimo tra due widget. */
export const OVERLAY_GAP = 4;
/** Respiro tra un widget e i nodi (margine di sicurezza dell'area visibile). */
export const SAFE_GAP = 8;
export const MINIMAP_SIZE = { w: 168, h: 104 } as const;
export const MINIMAP_COMPACT = 40;
export const ZOOM_SIZE = { w: 176, h: 38 } as const;
/** Tacca: lato lungo e lato corto (prototipo, CSS `.notch`, righe 290-293). */
export const NOTCH_LONG = 66;
export const NOTCH_SHORT = 24;
export const HINT_HEIGHT = 30;
export const HINT_WIDTHS = [360, 240] as const;

export interface NotchInput {
  readonly key: PanelKey;
  readonly side: Side;
  /** Scostamento lungo il bordo (due tacche sullo stesso bordo si affiancano). */
  readonly offset: number;
  /** Visibile (pannello chiuso) o nascosta: una tacca nascosta non occupa posto. */
  readonly visible: boolean;
}

export interface OverlayInput {
  /** Area del canvas (senza i pannelli). */
  readonly area: Size;
  /** Bordo del pannello aperto, se ce n'è uno. */
  readonly openSide: Side | null;
  readonly notches: readonly NotchInput[];
}

export interface OverlayLayout {
  readonly minimap: {
    /** Posizione occupata (piena o compatta); `null` se non c'è posto. */
    readonly rect: Rect | null;
    readonly corner: Corner | null;
    readonly compact: boolean;
    /** Posizione da piena, ancorata allo stesso angolo: la usa il pulsante compatto quando si espande. */
    readonly expanded: Rect | null;
  };
  readonly zoom: Rect;
  readonly hint: Rect | null;
  readonly notches: Readonly<Record<PanelKey, Rect>>;
  /** Spazio che i nodi devono evitare per restare visibili (anche per «Adatta»). */
  readonly insets: Insets;
}

const NO_INSETS: Insets = { top: 0, right: 0, bottom: 0, left: 0 };

export function intersects(a: Rect, b: Rect): boolean {
  return a.x < b.x + b.w && b.x < a.x + a.w && a.y < b.y + b.h && b.y < a.y + a.h;
}

const inflate = (r: Rect, by: number): Rect => ({
  x: r.x - by,
  y: r.y - by,
  w: r.w + by * 2,
  h: r.h + by * 2,
});

function inside(r: Rect, area: Size): boolean {
  return r.x >= 0 && r.y >= 0 && r.x + r.w <= area.w && r.y + r.h <= area.h;
}

/** Angolo dell'area; `inward` allontana il widget dal bordo orizzontale (per scavalcare una tacca). */
function cornerRect(corner: Corner, size: { w: number; h: number }, area: Size, inward = 0): Rect {
  const m = OVERLAY_MARGIN;
  const top = corner === "tl" || corner === "tr";
  return {
    x: corner === "bl" || corner === "tl" ? m : area.w - m - size.w,
    y: top ? m + inward : area.h - m - size.h - inward,
    w: size.w,
    h: size.h,
  };
}

/** Quanto spostare un widget verso l'interno per scavalcare una tacca (24 px) più il respiro. */
const CLEAR_NOTCH = NOTCH_SHORT + OVERLAY_GAP * 2;

const OPPOSITE: Record<Corner, Corner> = { bl: "tr", br: "tl", tl: "br", tr: "bl" };
const ALL_CORNERS: readonly Corner[] = ["bl", "tl", "br", "tr"];

/** Rettangolo di una tacca sul suo bordo, al centro più lo scostamento. */
export function notchRect(side: Side, offset: number, area: Size): Rect {
  switch (side) {
    case "left":
      return { x: 0, y: area.h / 2 - NOTCH_LONG / 2 + offset, w: NOTCH_SHORT, h: NOTCH_LONG };
    case "right":
      return {
        x: area.w - NOTCH_SHORT,
        y: area.h / 2 - NOTCH_LONG / 2 + offset,
        w: NOTCH_SHORT,
        h: NOTCH_LONG,
      };
    case "top":
      return { x: area.w / 2 - NOTCH_LONG / 2 + offset, y: 0, w: NOTCH_LONG, h: NOTCH_SHORT };
    case "bottom":
      return {
        x: area.w / 2 - NOTCH_LONG / 2 + offset,
        y: area.h - NOTCH_SHORT,
        w: NOTCH_LONG,
        h: NOTCH_SHORT,
      };
  }
}

/** Spazio da riservare ai nodi per un widget: la fascia meno costosa tra quella verticale e quella orizzontale. */
function insetsFor(rects: readonly Rect[], area: Size): Insets {
  let top = 0;
  let right = 0;
  let bottom = 0;
  let left = 0;
  for (const r of rects) {
    const upper = r.y + r.h / 2 < area.h / 2;
    const leftSide = r.x + r.w / 2 < area.w / 2;
    const vertical = (upper ? r.y + r.h : area.h - r.y) + SAFE_GAP;
    const horizontal = (leftSide ? r.x + r.w : area.w - r.x) + SAFE_GAP;
    if (vertical / Math.max(1, area.h) <= horizontal / Math.max(1, area.w)) {
      if (upper) top = Math.max(top, vertical);
      else bottom = Math.max(bottom, vertical);
    } else if (leftSide) left = Math.max(left, horizontal);
    else right = Math.max(right, horizontal);
  }
  return { top, right, bottom, left };
}

export function overlayLayout(input: OverlayInput): OverlayLayout {
  const { area, openSide } = input;

  const notches = {} as Record<PanelKey, Rect>;
  const obstacles: Rect[] = [];
  for (const n of input.notches) {
    const r = notchRect(n.side, n.offset, area);
    notches[n.key] = r;
    if (n.visible) obstacles.push(r);
  }
  const free = (r: Rect, others: readonly Rect[]): boolean =>
    inside(r, area) &&
    others.every((o) => !intersects(inflate(r, OVERLAY_GAP / 2), inflate(o, OVERLAY_GAP / 2)));

  // controlli di zoom: in basso a destra; solo in aree minuscole provano gli altri angoli
  let zoom = cornerRect("br", ZOOM_SIZE, area);
  zoomSearch: for (const inward of [0, CLEAR_NOTCH]) {
    for (const c of ["br", "bl", "tr", "tl"] as const) {
      const r = cornerRect(c, ZOOM_SIZE, area, inward);
      if (free(r, obstacles)) {
        zoom = r;
        break zoomSearch;
      }
    }
  }
  const placed: Rect[] = [...obstacles, zoom];

  // minimappa
  const preferred: Corner = openSide === "bottom" ? "tl" : "bl";
  const full = (c: Corner, inward = 0): Rect => cornerRect(c, MINIMAP_SIZE, area, inward);
  const compact = (c: Corner, inward = 0): Rect =>
    cornerRect(c, { w: MINIMAP_COMPACT, h: MINIMAP_COMPACT }, area, inward);
  const others = ALL_CORNERS.filter((c) => c !== preferred && c !== OPPOSITE[preferred]);
  const sequence: { corner: Corner; compact: boolean }[] = [
    { corner: preferred, compact: false },
    { corner: OPPOSITE[preferred], compact: false },
    { corner: preferred, compact: true },
    { corner: OPPOSITE[preferred], compact: true },
    ...others.map((corner) => ({ corner, compact: true })),
  ];
  // se nemmeno così c'è posto, si riprova scavalcando le tacche
  const candidates = [0, CLEAR_NOTCH].flatMap((inward) => sequence.map((c) => ({ ...c, inward })));
  let minimap: OverlayLayout["minimap"] = {
    rect: null,
    corner: null,
    compact: false,
    expanded: null,
  };
  for (const cand of candidates) {
    const rect = cand.compact ? compact(cand.corner, cand.inward) : full(cand.corner, cand.inward);
    if (free(rect, placed)) {
      minimap = {
        rect,
        corner: cand.corner,
        compact: cand.compact,
        expanded: cand.compact ? full(cand.corner, cand.inward) : rect,
      };
      placed.push(rect);
      break;
    }
  }

  // suggerimento di rilascio: al centro, in basso o in alto
  let hint: Rect | null = null;
  search: for (const width of HINT_WIDTHS) {
    const w = Math.min(width, Math.floor(area.w * 0.7));
    for (const y of [
      area.h - OVERLAY_MARGIN - HINT_HEIGHT,
      OVERLAY_MARGIN,
      area.h - OVERLAY_MARGIN - HINT_HEIGHT - CLEAR_NOTCH,
      OVERLAY_MARGIN + CLEAR_NOTCH,
    ]) {
      const r = { x: Math.round((area.w - w) / 2), y, w, h: HINT_HEIGHT };
      if (free(r, placed)) {
        hint = r;
        break search;
      }
    }
  }

  const insets = insetsFor(
    [...(minimap.rect ? [minimap.rect] : []), zoom].filter((r) => inside(r, area)),
    area,
  );
  return { minimap, zoom, hint, notches, insets: area.w > 0 && area.h > 0 ? insets : NO_INSETS };
}
```

### `src/etl-canvas/panels/panels.css`

724 righe

```css
/*
 * Pannelli agganciabili e cassetta degli strumenti (Fase 6a). Misure e
 * transizioni del prototipo (docs/prototype/isa-fusion-prototype.html, righe
 * indicate); colori, raggi e ombre solo dai token di ../tokens.css
 * (`--ec-*`, che leggono i semantici `--isa-*` dei temi). L'orientamento
 * (colonna sui bordi verticali, fascia sugli orizzontali) è la classe
 * `ec-horiz`, scelta dal bordo.
 */

/* spazio di lavoro: il canvas al centro, quattro approdi attorno — righe 211-220 */
.ec-workspace-host {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 0;
  /* se i pannelli in alto e in basso lasciano al canvas meno del minimo, scorre questo contenitore, non la pagina */
  overflow-x: hidden;
  overflow-y: auto;
}
.ec-workspace {
  display: grid;
  width: 100%;
  /* l'altezza è quella del contenitore: sopra e sotto i pannelli tolgono altezza al canvas (riga centrale) */
  height: 100%;
  grid-template-columns: auto minmax(0, 1fr) auto;
  grid-template-rows: auto auto minmax(var(--ec-canvas-min-h), 1fr) auto;
  color: var(--ec-ink);
  font-family: var(--ec-font);
}
.ec-workspace * {
  box-sizing: border-box;
}
/* barra dei controlli: una riga fissa sopra tutto, dentro lo spazio di lavoro (Fase 6a.2) */
.ec-bar-row {
  grid-column: 1 / 4;
  grid-row: 1;
  min-width: 0;
}
.ec-dock-top {
  grid-column: 1 / 4;
  grid-row: 2;
  display: flex;
  flex-direction: column;
}
.ec-dock-bottom {
  grid-column: 1 / 4;
  grid-row: 4;
  display: flex;
  flex-direction: column;
}
.ec-dock-left {
  grid-column: 1;
  grid-row: 3;
  display: flex;
}
.ec-dock-right {
  grid-column: 3;
  grid-row: 3;
  display: flex;
}
.ec-center {
  grid-column: 2;
  grid-row: 3;
  position: relative;
  min-width: 0;
  min-height: 0;
}
/* dentro lo spazio di lavoro l'altezza minima del canvas è quella della riga centrale (MIN_CANVAS_HEIGHT), non i 520px del canvas isolato */
.ec-workspace .ec-center .etl-canvas {
  min-height: 0;
}

/* un pannello cede spazio aprendosi; il margine verso il canvas fa parte della sua misura — righe 224-240 */
.ec-panel {
  overflow: hidden;
  position: relative;
  flex: 0 0 auto;
  transition:
    width 0.3s cubic-bezier(0.32, 0.72, 0, 1),
    height 0.3s cubic-bezier(0.32, 0.72, 0, 1);
}
.ec-panel.ec-side-left,
.ec-panel.ec-side-right {
  width: 0;
  height: 100%;
}
.ec-panel.ec-side-left.ec-open,
.ec-panel.ec-side-right.ec-open {
  width: calc(var(--pw) + 16px);
}
.ec-panel.ec-side-top,
.ec-panel.ec-side-bottom {
  height: 0;
  width: 100%;
}
.ec-panel.ec-side-top.ec-open,
.ec-panel.ec-side-bottom.ec-open {
  height: calc(var(--ph) + 16px);
}
.ec-panel-body {
  position: absolute;
}
.ec-panel.ec-side-left > .ec-panel-body {
  left: 0;
  top: 0;
  width: var(--pw);
  height: 100%;
}
.ec-panel.ec-side-right > .ec-panel-body {
  right: 0;
  top: 0;
  width: var(--pw);
  height: 100%;
}
.ec-panel.ec-side-top > .ec-panel-body {
  left: 0;
  top: 0;
  height: var(--ph);
  width: 100%;
}
.ec-panel.ec-side-bottom > .ec-panel-body {
  left: 0;
  bottom: 0;
  height: var(--ph);
  width: 100%;
}
.ec-panel.ec-instant {
  transition: none !important;
}

/* due pannelli sullo stesso bordo diventano schede di un unico pannello — righe 305-328 */
.ec-dock-tabs {
  position: absolute;
  z-index: 3;
  display: flex;
  gap: 4px;
  padding: 4px;
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  border-radius: var(--ec-r-minimap);
}
.ec-dock-tab {
  all: unset;
  cursor: pointer;
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  padding: 7px 10px;
  border-radius: var(--ec-r-sm);
  font-family: inherit;
  font-size: 12px;
  font-weight: 700;
  color: var(--ec-empty-ink);
  transition:
    background 0.18s,
    color 0.18s;
}
.ec-dock-tab svg {
  width: 14px;
  height: 14px;
  flex-shrink: 0;
}
.ec-dock-tab:hover {
  color: var(--ec-ink);
}
.ec-dock-tab:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-dock-tab.ec-on {
  background: var(--ec-accent);
  color: var(--ec-node-fill-ink);
}
.ec-panel.ec-grouped.ec-side-left > .ec-dock-tabs {
  left: 0;
  top: 0;
  width: var(--pw);
  height: 42px;
}
.ec-panel.ec-grouped.ec-side-right > .ec-dock-tabs {
  right: 0;
  top: 0;
  width: var(--pw);
  height: 42px;
}
.ec-panel.ec-grouped.ec-side-left > .ec-panel-body,
.ec-panel.ec-grouped.ec-side-right > .ec-panel-body {
  top: 50px;
  height: calc(100% - 50px);
}
.ec-panel.ec-grouped.ec-horiz > .ec-dock-tabs {
  left: 0;
  width: 48px;
  height: var(--ph);
  flex-direction: column;
}
.ec-panel.ec-grouped.ec-side-top > .ec-dock-tabs {
  top: 0;
}
.ec-panel.ec-grouped.ec-side-bottom > .ec-dock-tabs {
  bottom: 0;
}
.ec-panel.ec-grouped.ec-horiz > .ec-panel-body {
  left: 56px;
  width: calc(100% - 56px);
}
.ec-panel.ec-grouped.ec-horiz .ec-dock-tab span {
  display: none;
}
@keyframes ec-tab-in {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}
.ec-panel.ec-tab-in > .ec-panel-body {
  animation: ec-tab-in 0.26s cubic-bezier(0.32, 0.72, 0, 1);
}

/* tacca: il pannello chiuso resta raggiungibile, e si trascina su un altro bordo — righe 270-296 */
.ec-notch {
  all: unset;
  cursor: grab;
  position: absolute;
  z-index: 16;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--ec-surface-strong);
  color: var(--ec-accent-text);
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  transition:
    opacity 0.2s,
    transform 0.3s cubic-bezier(0.32, 0.72, 0, 1);
  touch-action: none;
}
.ec-notch:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 2px;
}
.ec-notch svg {
  width: 14px;
  height: 14px;
}
.ec-notch-left {
  left: 0;
  width: 24px;
  height: 66px;
  border-left: 0;
  border-top-right-radius: var(--ec-r-minimap);
  border-bottom-right-radius: var(--ec-r-minimap);
}
.ec-notch-right {
  right: 0;
  width: 24px;
  height: 66px;
  border-right: 0;
  border-top-left-radius: var(--ec-r-minimap);
  border-bottom-left-radius: var(--ec-r-minimap);
}
.ec-notch-top {
  top: 0;
  height: 24px;
  width: 66px;
  border-top: 0;
  border-bottom-left-radius: var(--ec-r-minimap);
  border-bottom-right-radius: var(--ec-r-minimap);
}
.ec-notch-bottom {
  bottom: 0;
  height: 24px;
  width: 66px;
  border-bottom: 0;
  border-top-left-radius: var(--ec-r-minimap);
  border-top-right-radius: var(--ec-r-minimap);
}
.ec-notch.ec-hidden {
  opacity: 0;
  pointer-events: none;
}
.ec-notch.ec-dragging {
  position: fixed;
  right: auto;
  bottom: auto;
  transition: none;
  cursor: grabbing;
  border-radius: var(--ec-r-minimap);
  border: 1px solid var(--ec-accent);
  width: 34px;
  height: 34px;
  box-shadow: var(--ec-drag-shadow);
}
.ec-edge-hint {
  position: absolute;
  z-index: 15;
  background: var(--ec-accent);
  opacity: 0;
  pointer-events: none;
  border-radius: var(--ec-r-pill);
  transition: opacity 0.15s;
}
.ec-edge-hint.ec-on {
  opacity: 0.55;
}
.ec-edge-hint.ec-e-left {
  left: 4px;
  top: 12%;
  width: 4px;
  height: 76%;
}
.ec-edge-hint.ec-e-right {
  right: 4px;
  top: 12%;
  width: 4px;
  height: 76%;
}
.ec-edge-hint.ec-e-top {
  top: 4px;
  left: 12%;
  height: 4px;
  width: 76%;
}
.ec-edge-hint.ec-e-bottom {
  bottom: 4px;
  left: 12%;
  height: 4px;
  width: 76%;
}
.ec-workspace .ec-center .ec-hint {
  position: absolute;
  /* posizione e misura: inline, da overlayLayout.ts */
  box-sizing: border-box;
  z-index: 16;
  padding: 0 14px;
  line-height: 28px;
  text-align: center;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  border-radius: var(--ec-r-pill);
  background: var(--ec-surface-strong);
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  color: var(--ec-ink);
  font-size: 12px;
  font-weight: 600;
  pointer-events: none;
}

/* contenuto del pannello (cassetta e Inspector) — righe 235-268 */
.ec-tb-inner {
  box-sizing: border-box;
  width: 100%;
  height: 100%;
  overflow-y: auto;
  overscroll-behavior: contain;
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  border-radius: var(--ec-r-fill);
  padding: 18px 16px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.ec-tb-inner > * {
  flex: 0 0 auto;
}
.ec-tb-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;
}
.ec-tb-title {
  font-weight: 800;
  font-size: 15px;
}
.ec-close-btn {
  all: unset;
  cursor: pointer;
  display: grid;
  place-items: center;
  width: 26px;
  height: 26px;
  border-radius: var(--ec-r-pill);
  color: var(--ec-empty-ink);
}
.ec-close-btn:hover {
  color: var(--ec-ink);
  background: var(--ec-accent-soft);
}
.ec-close-btn:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-close-arrow {
  display: inline-flex;
  transition: transform 0.2s;
}
.ec-tb-sec {
  border-top: 1px solid var(--ec-panel-border);
  padding-top: 8px;
}
.ec-tb-sec:first-of-type {
  border-top: 0;
}
.ec-tb-sec-head {
  all: unset;
  cursor: pointer;
  width: 100%;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 4px 2px 8px;
}
.ec-tb-sec-head:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-chev {
  display: inline-flex;
  transition: transform 0.2s;
}
.ec-chev svg {
  width: 11px;
  height: 11px;
}
.ec-tb-sec.ec-open .ec-chev {
  transform: rotate(90deg);
  color: var(--ec-accent-text);
}
.ec-tb-sec-name {
  font-size: 10px;
  font-weight: 800;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--ec-empty-ink);
}
.ec-tb-sec-body {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px 4px;
  max-height: 0;
  overflow: hidden;
  opacity: 0;
  transition:
    max-height 0.3s cubic-bezier(0.32, 0.72, 0, 1),
    opacity 0.2s,
    padding 0.3s;
}
.ec-tb-sec.ec-open .ec-tb-sec-body {
  max-height: 640px;
  opacity: 1;
  padding-bottom: 10px;
}

/* voci — righe 40-53 */
.ec-pal-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  width: auto;
  cursor: grab;
  touch-action: none;
}
.ec-pal-item:active {
  cursor: grabbing;
}
.ec-pal-chip {
  width: 44px;
  height: 44px;
  border-radius: var(--ec-r-sm);
  background: var(--ec-node-op);
  color: var(--ec-node-op-ink);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.18s cubic-bezier(0.34, 1.56, 0.64, 1);
}
.ec-pal-item:hover .ec-pal-chip {
  transform: translateY(-2px) scale(1.05);
}
.ec-pal-item.ec-source .ec-pal-chip {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
}
.ec-pal-chip svg {
  width: 20px;
  height: 20px;
}
.ec-pal-label {
  font-size: 9.5px;
  font-weight: 700;
  color: var(--ec-empty-ink);
  text-align: center;
  line-height: 1.2;
}
.ec-lib-meta {
  font-size: 8.5px;
  color: var(--ec-empty-ink);
  font-weight: 600;
  margin-top: -3px;
  text-align: center;
}
.ec-tb-upload {
  all: unset;
  cursor: pointer;
  grid-column: 1 / -1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  padding: 9px 12px;
  border-radius: var(--ec-r-sm);
  border: 1.5px dashed var(--ec-accent-soft-2);
  font-size: 12px;
  font-weight: 700;
  color: var(--ec-accent-text);
  background: var(--ec-mm-view-bg);
}
.ec-tb-upload:hover {
  background: var(--ec-accent-soft);
}
.ec-tb-upload:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 1px;
}
.ec-tb-upload svg {
  width: 14px;
  height: 14px;
}
.ec-tb-empty {
  grid-column: 1 / -1;
  font-size: 11px;
  color: var(--ec-empty-ink);
  font-weight: 600;
  text-align: center;
  padding: 2px 0 4px;
}
.ec-tb-status {
  font-size: 11px;
  font-weight: 600;
  color: var(--ec-empty-ink);
  line-height: 1.4;
}
.ec-insp-name {
  font-size: 13px;
  font-weight: 700;
  color: var(--ec-ink);
  overflow-wrap: anywhere;
}

/* anteprima del trascinamento dalla cassetta — righe 55-63 */
.ec-ghost {
  position: fixed;
  z-index: 60;
  pointer-events: none;
  width: 88px;
  height: 88px;
  border-radius: var(--ec-r-op);
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--ec-node-op);
  color: var(--ec-node-op-ink);
  opacity: 0.9;
  box-shadow: var(--ec-drag-shadow);
}
.ec-ghost.ec-source {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
  border-radius: var(--ec-r-fill);
}
.ec-ghost svg {
  width: 26px;
  height: 26px;
}

/* orientamento orizzontale (bordi alto e basso): le sezioni scorrono in fila — righe 329-338 */
.ec-panel.ec-horiz .ec-tb-inner {
  flex-direction: row;
  overflow-x: auto;
  overflow-y: hidden;
  gap: 14px;
  align-items: stretch;
}
.ec-panel.ec-horiz .ec-tb-head {
  flex-direction: column;
  align-items: flex-start;
  justify-content: flex-start;
  gap: 8px;
  margin: 0;
}
.ec-panel.ec-horiz .ec-tb-sec {
  border-top: 0;
  border-left: 1px solid var(--ec-panel-border);
  padding: 0 0 0 12px;
}
.ec-panel.ec-horiz .ec-tb-sec-head {
  pointer-events: none;
}
.ec-panel.ec-horiz .ec-chev {
  display: none;
}
.ec-panel.ec-horiz .ec-tb-sec-body {
  max-height: none;
  opacity: 1;
  padding: 0;
  grid-template-columns: none;
  grid-template-rows: repeat(2, auto);
  grid-auto-flow: column;
  grid-auto-columns: 66px;
}
.ec-panel.ec-horiz .ec-tb-upload {
  grid-column: auto;
  grid-row: span 2;
  flex-direction: column;
  width: 66px;
  padding: 8px 4px;
  text-align: center;
}

@media (prefers-reduced-motion: reduce) {
  .ec-workspace,
  .ec-panel,
  .ec-notch,
  .ec-tb-sec-body,
  .ec-pal-chip {
    transition: none;
  }
  .ec-panel.ec-tab-in > .ec-panel-body {
    animation: none;
  }
}

/* barra dei controlli — solo token */
.ec-bar {
  display: flex;
  align-items: center;
  flex-wrap: nowrap;
  gap: 6px;
  min-width: 0;
  overflow-x: auto;
  padding: 6px 4px 8px;
}
.ec-seg {
  display: flex;
  padding: 3px;
  gap: 2px;
  border-radius: var(--ec-r-pill);
  background: var(--ec-surface-strong);
  border: 1px solid var(--ec-panel-border);
}
.ec-seg-btn,
.ec-bar-btn {
  all: unset;
  cursor: pointer;
  box-sizing: border-box;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  flex: none;
  height: 28px;
  padding: 0 12px;
  border-radius: var(--ec-r-pill);
  font-family: var(--ec-font);
  font-size: 12px;
  font-weight: 700;
  color: var(--ec-ink);
  white-space: nowrap;
}
.ec-bar-btn {
  background: var(--ec-surface-strong);
  border: 1px solid var(--ec-panel-border);
}
.ec-bar-icon {
  width: 32px;
  padding: 0;
  justify-content: center;
}
.ec-seg-btn.ec-on {
  background: var(--ec-accent-soft);
  color: var(--ec-accent-text);
}
.ec-bar svg {
  width: 15px;
  height: 15px;
  flex: none;
}
.ec-seg-btn:hover,
.ec-bar-btn:hover:not(:disabled) {
  color: var(--ec-accent-text);
}
.ec-bar-danger:hover:not(:disabled) {
  color: var(--ec-danger);
}
.ec-seg-btn:focus-visible,
.ec-bar-btn:focus-visible {
  outline: 2px solid var(--ec-select);
  outline-offset: 2px;
}
.ec-bar-btn:disabled {
  cursor: default;
  color: var(--ec-muted);
}
.ec-bar-sep {
  flex: none;
  align-self: stretch;
  margin: 4px 2px;
  border-left: 1px solid var(--ec-panel-border);
}
```

### `src/etl-canvas/panels/ui-icons.tsx`

113 righe

```tsx
/** Icone dell'interfaccia dei pannelli (prototipo, righe 4790-4793, 813-818, 893): tracciati statici, nessun dato dell'utente. */
import type { Side } from "../../etl-store";

function Svg(props: { children: React.ReactNode; size?: number }) {
  return (
    <svg
      viewBox="0 0 24 24"
      width={props.size}
      height={props.size}
      fill="none"
      stroke="currentColor"
      strokeWidth={2.2}
      strokeLinecap="round"
      strokeLinejoin="round"
      aria-hidden="true"
    >
      {props.children}
    </svg>
  );
}

export function ToolsIcon() {
  return (
    <Svg>
      <rect x="4" y="4" width="6.5" height="6.5" rx="1.5" />
      <rect x="13.5" y="4" width="6.5" height="6.5" rx="1.5" />
      <rect x="4" y="13.5" width="6.5" height="6.5" rx="1.5" />
      <rect x="13.5" y="13.5" width="6.5" height="6.5" rx="1.5" />
    </Svg>
  );
}

export function InspectorIcon() {
  return (
    <Svg>
      <line x1="4" y1="7" x2="20" y2="7" />
      <line x1="4" y1="17" x2="20" y2="17" />
      <circle cx="9" cy="7" r="2.4" />
      <circle cx="15" cy="17" r="2.4" />
    </Svg>
  );
}

export function UploadIcon() {
  return (
    <Svg>
      <path d="M12 16V4" />
      <path d="M7 9l5-5 5 5" />
      <path d="M5 20h14" />
    </Svg>
  );
}

export function ChevronIcon() {
  return (
    <Svg>
      <polyline points="9 18 15 12 9 6" />
    </Svg>
  );
}

/** Freccia di chiusura: indica sempre il bordo verso cui il pannello rientra (riga 285-289). */
export function CloseArrow(props: { side: Side }) {
  const turn = { left: 0, right: 180, top: 90, bottom: -90 }[props.side];
  return (
    <span className="ec-close-arrow" style={{ transform: `rotate(${turn}deg)` }}>
      <Svg size={13}>
        <polyline points="15 6 9 12 15 18" />
      </Svg>
    </span>
  );
}

export function UndoIcon() {
  return (
    <Svg>
      <path d="M9 14 4 9l5-5" />
      <path d="M4 9h10.5a5.5 5.5 0 0 1 0 11H11" />
    </Svg>
  );
}

export function RedoIcon() {
  return (
    <Svg>
      <path d="m15 14 5-5-5-5" />
      <path d="M20 9H9.5a5.5 5.5 0 0 0 0 11H13" />
    </Svg>
  );
}

export function ReorderIcon() {
  return (
    <Svg>
      <rect x="4" y="4" width="6" height="6" rx="1.5" />
      <rect x="14" y="4" width="6" height="6" rx="1.5" />
      <rect x="4" y="14" width="6" height="6" rx="1.5" />
      <rect x="14" y="14" width="6" height="6" rx="1.5" />
    </Svg>
  );
}

export function TrashIcon() {
  return (
    <Svg>
      <path d="M4 7h16" />
      <path d="M9 7V4.5h6V7" />
      <path d="M6.5 7l1 12.5h9l1-12.5" />
      <path d="M10 11v5M14 11v5" />
    </Svg>
  );
}
```

### `src/etl-canvas/seed.ts`

68 righe

```ts
/**
 * Scena iniziale del prototipo (`init`, righe 5070-5083 di
 * docs/prototype/isa-fusion-prototype.html): Vendite 2026, Filtra Righe,
 * Unisci, Ordina, Esporta, con posizioni identiche e nessun cavo. Serve
 * solo in sviluppo (`?seed=prototype`).
 */
import { META, defaultParams } from "../etl-core";
import type { Card, ColumnDef, ComponentId } from "../etl-core";
import { initialState } from "../etl-store";
import type { EtlState } from "../etl-store";

/** `SCHEMA` del prototipo (righe 2405-2414). Il tipo "object" della colonna `categoria` è "stringa" in etl-core. */
const SCHEMA: ColumnDef[] = [
  { name: "id", type: "integer", values: Array.from({ length: 30 }, (_, i) => String(i + 1)) },
  { name: "cliente", type: "stringa", values: ["Acme", "Borealis", "Cedro", "Delta", "Eureka"] },
  { name: "regione", type: "stringa", values: ["Nord", "Centro", "Sud", "Isole"] },
  { name: "categoria", type: "stringa", values: ["Hardware", "Software", "Servizi", "Consulenza"] },
  { name: "stato", type: "stringa", values: ["Aperto", "In corso", "Chiuso", "Annullato"] },
  {
    name: "quantita",
    type: "integer",
    values: ["1", "2", "3", "5", "8", "10", "12", "20", "25", "50"],
  },
  { name: "importo", type: "numerico", values: ["45.2", "80", "120.5", "300", "512.9", "1049"] },
  {
    name: "data",
    type: "data",
    values: ["2026-01-03", "2026-01-04", "2026-01-05", "2026-01-06", "2026-02-01"],
  },
];

function op(id: string, type: ComponentId, x: number, y: number): Card {
  return {
    id,
    kind: "op",
    components: [type],
    params: [defaultParams(type)],
    name: META[type].label,
    x,
    y,
  };
}

export function prototypeScene(): EtlState {
  const ds: Card = {
    id: "ds1",
    kind: "dataset",
    components: ["dataset"],
    params: [{ ...defaultParams("dataset"), path: "vendite_2026.csv", columns: SCHEMA }],
    name: META.dataset.label,
    x: 26,
    y: 182,
  };
  const cards = [
    ds,
    op("op-filter", "filter", 260, 52),
    op("op-join", "join", 260, 182),
    op("op-sort", "sort", 260, 338),
    op("op-export", "exportOp", 442, 338),
  ];
  const base = initialState();
  return {
    ...base,
    graph: { cards: Object.fromEntries(cards.map((c) => [c.id, c])), links: [] },
    counters: { ...base.counters, ds: 1 },
  };
}
```

