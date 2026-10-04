# 01e-etl-canvas-i.md

File in questo blocco:

- `src/etl-canvas/model.ts`
- `src/etl-canvas/motion.tsx`
- `src/etl-canvas/panels/ControlBar.tsx`
- `src/etl-canvas/panels/Dock.tsx`
- `src/etl-canvas/panels/EtlWorkspace.tsx`
- `src/etl-canvas/panels/InspectorShell.tsx`
- `src/etl-canvas/panels/Toolbox.tsx`
- `src/etl-canvas/panels/actions.ts`
- `src/etl-canvas/panels/csv.ts`
- `src/etl-canvas/panels/families.ts`
- `src/etl-canvas/panels/layout.ts`
- `src/etl-canvas/panels/overlayLayout.ts`

---

### `src/etl-canvas/model.ts`

112 righe

```ts
/**
 * Dal grafo di etl-core a ciò che il canvas disegna: classi e icone di ogni
 * nodo, fette di un output parziale. Funzioni pure.
 */
import { sectionOf } from "../etl-core";
import type { Card, ComponentId } from "../etl-core";

/** Classe di conteggio per la disposizione delle icone (prototipo `countClass`, righe 982-985). */
export function countClass(n: number): string {
  if (n <= 1) return "count-1";
  if (n === 2) return "count-2";
  if (n <= 3) return "count-3";
  if (n <= 6) return "count-6";
  return "count-many";
}

/** Un output è "parziale" se attende altre tabelle (riga 1620). */
export function isPartial(card: Card): boolean {
  return (
    card.kind === "dataset" &&
    card.isOutput === true &&
    card.capacity !== undefined &&
    card.capacity > 1 &&
    (card.filled ?? 0) < card.capacity
  );
}

export interface Slice {
  readonly full: boolean;
}

/** Le fette di un output parziale, riempite da sinistra (righe 1622-1627). */
export function slicesOf(card: Card): Slice[] {
  const n = card.capacity ?? 0;
  const f = card.filled ?? 0;
  return Array.from({ length: n }, (_, i) => ({ full: i < f }));
}

/** Famiglia di operazioni (colore del nodo): una per sezione della cassetta, tranne i dataset. */
export type OpFamily = "filter" | "transform" | "merge" | "output";

const FAMILY_OF_SECTION: Readonly<Record<string, OpFamily>> = {
  rows: "filter",
  xform: "transform",
  merge: "merge",
  out: "output",
};

/** La famiglia di un nodo lavorazione (quella della prima operazione); i dataset non ne hanno. */
export function familyOf(card: Card): OpFamily | undefined {
  if (card.kind !== "op") return undefined;
  const first = card.components[0];
  const section = first === undefined ? null : sectionOf(first);
  return section ? FAMILY_OF_SECTION[section.id] : undefined;
}

export interface NodeView {
  readonly id: string;
  readonly card: Card;
  /** Classi del contenitore del nodo. */
  readonly className: string;
  readonly iconClass: string;
  readonly partial: boolean;
  readonly slices: readonly Slice[];
  readonly family: OpFamily | undefined;
  readonly icons: readonly ComponentId[];
  readonly warn: string | null;
  readonly selected: boolean;
}

/** Classi del nodo (prototipo `createCardEl`, righe 1003-1004, più `partial`, `warn`, `selected`). */
/** Stato dei gesti che cambia l'aspetto di un nodo (classi `ec-dragging`, `ec-drop-*`, `ec-doomed`). */
export interface NodeGestureState {
  readonly dragging?: boolean;
  readonly drop?: "merge" | "link" | "link-reverse" | "displace" | "reject" | null;
  readonly doomed?: boolean;
}

export function nodeView(
  card: Card,
  warn: string | null,
  selected: boolean,
  gesture: NodeGestureState = {},
): NodeView {
  const combined = card.kind === "op" && card.components.length > 1;
  const partial = isPartial(card);
  const classes = ["ec-card"];
  if (card.kind === "dataset") classes.push("ec-dataset");
  if (card.kind === "dataset" && card.isOutput) classes.push("ec-output");
  if (combined) classes.push("ec-combined");
  if (partial) classes.push("ec-partial");
  if (warn) classes.push("ec-warn");
  if (selected) classes.push("ec-selected");
  if (gesture.dragging) classes.push("ec-dragging");
  if (gesture.drop) classes.push(`ec-drop-${gesture.drop}`);
  if (gesture.doomed) classes.push("ec-doomed");
  const icons: ComponentId[] =
    card.kind === "dataset" && card.isOutput ? ["dataset"] : [...card.components];
  return {
    id: card.id,
    card,
    className: classes.join(" "),
    iconClass: partial ? "ec-icon-wrap ec-split" : `ec-icon-wrap ec-${countClass(icons.length)}`,
    partial,
    slices: partial ? slicesOf(card) : [],
    family: familyOf(card),
    icons,
    warn,
    selected,
  };
}
```

### `src/etl-canvas/motion.tsx`

10 righe

```tsx
import { createContext, useContext } from "react";
import type { MotionEngine } from "./engine";

/** Il motore delle animazioni, raggiungibile da cavi e nodi per registrare i propri elementi. */
export const MotionContext = createContext<MotionEngine | null>(null);

export function useMotion(): MotionEngine | null {
  return useContext(MotionContext);
}
```

### `src/etl-canvas/panels/ControlBar.tsx`

97 righe

```tsx
/**
 * La barra dei controlli: una riga fissa sopra l'area del canvas, dentro lo
 * spazio di lavoro (non in sovrimpressione ai nodi). Interruttore
 * Libero/Organizzato (`setMode`), Riordina (`autoLayout`), Annulla e
 * Ripristina (disabilitati senza cronologia), Svuota (`clearAll`, sempre con
 * conferma). Nel prototipo i pulsanti «Funzionalità» e «Reimposta» non sono
 * portati: tutte le funzionalità sono sempre attive.
 */
import type { Size } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import type { InteractionController } from "../interaction";
import { RedoIcon, ReorderIcon, TrashIcon, UndoIcon } from "./ui-icons";

export function ControlBar(props: {
  store: EtlStore;
  controller: InteractionController;
  /** Area del canvas: serve a «Riordina»; `null` finché non è misurata. */
  area: Size | null;
}) {
  const { store, controller, area } = props;
  // ogni cambio di stato (anche annulla e ripristina) ridisegna la barra
  const state = useEtlState((s) => s, store);
  const hasNodes = Object.keys(state.graph.cards).length > 0;
  const canUndo = store.canUndo();
  const canRedo = store.canRedo();

  return (
    <div className="ec-bar" role="toolbar" aria-label="Controlli del canvas" data-testid="ec-bar">
      <div className="ec-seg" role="group" aria-label="Disposizione dei nodi">
        <button
          type="button"
          className={"ec-seg-btn" + (state.mode === "free" ? " ec-on" : "")}
          aria-pressed={state.mode === "free"}
          title="Libero: i nodi stanno dove li lasci"
          onClick={() => store.dispatch({ type: "setMode", payload: { mode: "free" } })}
        >
          Libero
        </button>
        <button
          type="button"
          className={"ec-seg-btn" + (state.mode === "grid" ? " ec-on" : "")}
          aria-pressed={state.mode === "grid"}
          title="Organizzato: i nodi si allineano alla griglia"
          onClick={() => store.dispatch({ type: "setMode", payload: { mode: "grid" } })}
        >
          Organizzato
        </button>
      </div>
      <button
        type="button"
        className="ec-bar-btn"
        title="Riordina i nodi"
        disabled={!hasNodes || !area}
        onClick={() => {
          if (area) store.dispatch({ type: "autoLayout", payload: { viewport: area } });
        }}
      >
        <ReorderIcon />
        <span>Riordina</span>
      </button>
      <span className="ec-bar-sep" aria-hidden="true" />
      <button
        type="button"
        className="ec-bar-btn ec-bar-icon"
        aria-label="Annulla"
        title="Annulla (Cmd/Ctrl+Z)"
        disabled={!canUndo}
        onClick={() => store.undo()}
      >
        <UndoIcon />
      </button>
      <button
        type="button"
        className="ec-bar-btn ec-bar-icon"
        aria-label="Ripristina"
        title="Ripristina (Cmd/Ctrl+Maiusc+Z)"
        disabled={!canRedo}
        onClick={() => store.redo()}
      >
        <RedoIcon />
      </button>
      <span className="ec-bar-sep" aria-hidden="true" />
      <button
        type="button"
        className="ec-bar-btn ec-bar-icon ec-bar-danger"
        aria-label="Svuota il canvas"
        title="Svuota il canvas: elimina tutti i nodi e i collegamenti"
        disabled={!hasNodes}
        onClick={() => controller.requestClearAll()}
      >
        <TrashIcon />
      </button>
    </div>
  );
}
```

### `src/etl-canvas/panels/Dock.tsx`

346 righe

```tsx
/**
 * Il guscio comune dei pannelli: quattro approdi attorno al canvas (sopra,
 * sinistra, destra, sotto), tacca quando un pannello è chiuso, trascinamento
 * della tacca su un altro bordo, schede condivise quando due pannelli stanno
 * sullo stesso bordo (prototipo, righe 4756-4905, CSS 211-346).
 *
 * Lo stato (lato, aperto/chiuso, scheda attiva = il pannello aperto sul
 * bordo) è in etl-store; qui si legge e si cambia solo con i comandi
 * `setPanel`/`setView` (vedi actions.ts). Niente accesso a window/document
 * durante il rendering: gli ascoltatori nascono nei gestori degli eventi.
 */
import { useEffect, useMemo, useRef, useState } from "react";
import type { CSSProperties, PointerEvent as ReactPointerEvent, ReactNode } from "react";
import type { Size } from "../../etl-layout";
import type { EtlStore, PanelKey, Panels, Side } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import type { InteractionController } from "../interaction";
import type { PanelActions } from "./actions";
import { ControlBar } from "./ControlBar";
import {
  MIN_CANVAS_HEIGHT,
  cappedPanelHeight,
  PANEL_KEYS,
  PANEL_LABEL,
  PANEL_NAME,
  SIDES,
  SIDE_NAME,
  isGrouped,
  isVertical,
  keepVisible,
  nearestSide,
  notchHidden,
  notchOffset,
  panelSize,
} from "./layout";
import { overlayLayout } from "./overlayLayout";
import type { OverlayLayout } from "./overlayLayout";
import "./panels.css";
import { InspectorIcon, ToolsIcon } from "./ui-icons";

/** Soglia prima che la tacca si stacchi dal bordo (prototipo, riga 4884). */
const NOTCH_DRAG_THRESHOLD = 5;
/** Durata della dissolvenza del contenuto al cambio di scheda (riga 4809). */
const TAB_IN_MS = 280;

const TAB_ICON: Record<PanelKey, () => ReactNode> = {
  tools: () => <ToolsIcon />,
  insp: () => <InspectorIcon />,
};

export interface DockLayoutProps {
  readonly store: EtlStore;
  readonly actions: PanelActions;
  /** Controller dei gesti: la barra dei controlli chiede la conferma di «Svuota» al canvas. */
  readonly controller: InteractionController;
  /** Il canvas, al centro; riceve la disposizione dei widget in sovrimpressione, quando l'area è misurata. */
  readonly canvas: (overlay: OverlayLayout | undefined) => ReactNode;
  /** Contenuto di ciascun pannello; `side` è il bordo corrente, `horiz` l'orientamento. */
  readonly content: Readonly<Record<PanelKey, (ctx: { side: Side; horiz: boolean }) => ReactNode>>;
  /** Altro da disegnare sopra lo spazio di lavoro (per esempio l'anteprima del trascinamento). */
  readonly overlay?: ReactNode;
}

interface NotchDrag {
  readonly key: PanelKey;
  readonly x: number;
  readonly y: number;
  readonly side: Side;
}

export function DockLayout(props: DockLayoutProps) {
  const { store, actions, controller, canvas, content, overlay } = props;
  const panels = useEtlState((s) => s.panels, store);
  const centerRef = useRef<HTMLDivElement>(null);
  const workspaceRef = useRef<HTMLDivElement>(null);
  // altezza dello spazio di lavoro: da qui il tetto all'altezza dei pannelli orizzontali
  const [workspaceH, setWorkspaceH] = useState(0);
  useEffect(() => {
    const el = workspaceRef.current;
    if (!el) return;
    const measure = () => setWorkspaceH(el.clientHeight);
    measure();
    const ro = new ResizeObserver(measure);
    ro.observe(el);
    return () => ro.disconnect();
  }, []);
  // misura dell'area del canvas (la riga centrale): da qui la disposizione dei widget e la visibilità dei nodi
  const [area, setArea] = useState<Size | null>(null);
  useEffect(() => {
    const el = centerRef.current;
    if (!el) return;
    const measure = () => {
      const w = el.clientWidth;
      const h = el.clientHeight;
      setArea((prev) => (prev && prev.w === w && prev.h === h ? prev : { w, h }));
    };
    measure();
    const ro = new ResizeObserver(measure);
    ro.observe(el);
    return () => ro.disconnect();
  }, []);
  const openSide =
    PANEL_KEYS.map((k) => (panels[k].open ? panels[k].side : null)).find(Boolean) ?? null;
  const layout = useMemo(
    () =>
      area
        ? overlayLayout({
            area,
            openSide,
            notches: PANEL_KEYS.map((k) => ({
              key: k,
              side: panels[k].side,
              offset: notchOffset(panels, k),
              visible: !notchHidden(panels, k),
            })),
          })
        : undefined,
    [area, openSide, panels],
  );
  // dopo ogni cambio (apertura, chiusura, scheda, bordo, finestra) i nodi che erano interamente visibili lo restano
  const lastVisibility = useRef<{ size: Size; insets: OverlayLayout["insets"] } | null>(null);
  useEffect(() => {
    if (!area || !layout) return;
    const next = { size: area, insets: layout.insets };
    const prev = lastVisibility.current;
    lastVisibility.current = next;
    if (!prev) return;
    const st = store.getState();
    const view = keepVisible(Object.values(st.graph.cards), st.view, prev, next);
    if (view !== st.view) store.dispatch({ type: "setView", payload: { x: view.x, y: view.y } });
  }, [area, layout, store]);
  const [drag, setDrag] = useState<NotchDrag | null>(null);
  // cambiare scheda sostituisce il contenuto sul posto: niente animazione di larghezza, solo una dissolvenza
  const [instant, setInstant] = useState(false);
  const [tabIn, setTabIn] = useState<PanelKey | null>(null);
  const timers = useRef<ReturnType<typeof setTimeout>[]>([]);
  const cleanup = useRef<(() => void) | null>(null);

  useEffect(
    () => () => {
      timers.current.forEach(clearTimeout);
      cleanup.current?.();
    },
    [],
  );

  const switchTab = (key: PanelKey) => {
    setInstant(true);
    actions.open(key);
    setTabIn(key);
    timers.current.push(setTimeout(() => setTabIn(null), TAB_IN_MS));
    // il ripristino delle transizioni avviene dopo due disegni (riga 4811)
    timers.current.push(setTimeout(() => setInstant(false), 60));
  };

  const onNotchPointerDown = (key: PanelKey, e: ReactPointerEvent<HTMLButtonElement>) => {
    if (e.button !== 0) return;
    e.preventDefault();
    e.stopPropagation();
    const sx = e.clientX;
    const sy = e.clientY;
    let moved = false;
    const sideAt = (cx: number, cy: number): Side => {
      const r = centerRef.current?.getBoundingClientRect();
      return r ? nearestSide({ x: cx, y: cy }, r) : panels[key].side;
    };
    const move = (ev: PointerEvent) => {
      if (!moved && Math.hypot(ev.clientX - sx, ev.clientY - sy) < NOTCH_DRAG_THRESHOLD) return;
      moved = true;
      setDrag({ key, x: ev.clientX, y: ev.clientY, side: sideAt(ev.clientX, ev.clientY) });
    };
    const stop = () => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
      window.removeEventListener("pointercancel", abort);
      cleanup.current = null;
      setDrag(null);
    };
    const up = (ev: PointerEvent) => {
      stop();
      // un click apre soltanto
      if (!moved) {
        actions.open(key);
        return;
      }
      const side = sideAt(ev.clientX, ev.clientY);
      if (side === store.getState().panels[key].side) actions.open(key);
      else actions.moveTo(key, side);
    };
    const abort = () => stop();
    cleanup.current = abort;
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
    window.addEventListener("pointercancel", abort);
  };

  return (
    <div
      ref={workspaceRef}
      className="ec-workspace"
      data-testid="ec-workspace"
      // l'area di lavoro ha l'altezza del contenitore: i pannelli in alto e in basso la sottraggono al canvas
      style={{ "--ec-canvas-min-h": `${MIN_CANVAS_HEIGHT}px` } as CSSProperties}
    >
      {SIDES.map((side) => (
        <div key={side} className={`ec-dock ec-dock-${side}`} data-dock={side}>
          {PANEL_KEYS.filter((k) => panels[k].side === side).map((k) => (
            <PanelShell
              key={k}
              pkey={k}
              panels={panels}
              workspaceH={workspaceH}
              instant={instant}
              tabIn={tabIn === k}
              onSwitch={switchTab}
            >
              {content[k]({ side, horiz: !isVertical(side) })}
            </PanelShell>
          ))}
        </div>
      ))}
      <div className="ec-bar-row">
        <ControlBar store={store} controller={controller} area={area} />
      </div>
      <div className="ec-center" ref={centerRef}>
        {canvas(layout)}
        <div
          className={"ec-edge-hint" + (drag ? ` ec-on ec-e-${drag.side}` : "")}
          data-testid="ec-edge-hint"
        />
        {PANEL_KEYS.map((k) => {
          const dragging = drag?.key === k;
          const hidden = !dragging && notchHidden(panels, k);
          const side = panels[k].side;
          const rect = layout?.notches[k];
          const pos: CSSProperties = dragging
            ? { left: (drag?.x ?? 0) - 17, top: (drag?.y ?? 0) - 17 }
            : rect
              ? { left: rect.x, top: rect.y }
              : {};
          return (
            <button
              key={k}
              type="button"
              className={
                `ec-notch ec-notch-${side}` +
                (hidden ? " ec-hidden" : "") +
                (dragging ? " ec-dragging" : "")
              }
              style={pos}
              data-notch={k}
              tabIndex={hidden ? -1 : 0}
              aria-hidden={hidden || undefined}
              aria-label={`${PANEL_LABEL[k]}: clicca per aprire, trascina per spostarlo`}
              onPointerDown={(e) => onNotchPointerDown(k, e)}
              onClick={(e) => {
                e.stopPropagation();
                // da tastiera (detail 0) il clic apre; col puntatore ha già aperto il rilascio
                if (e.detail === 0) actions.open(k);
              }}
            >
              {TAB_ICON[k]()}
            </button>
          );
        })}
        {drag && layout?.hint ? (
          <div
            className="ec-hint"
            role="status"
            style={{
              left: layout.hint.x,
              top: layout.hint.y,
              width: layout.hint.w,
              height: layout.hint.h,
            }}
          >
            Rilascia per agganciare {PANEL_NAME[drag.key]} al bordo {SIDE_NAME[drag.side]}
          </div>
        ) : null}
      </div>
      {overlay}
    </div>
  );
}

/** Un pannello nel suo approdo: si apre e si chiude cedendo spazio al canvas; in gruppo mostra le schede. */
function PanelShell(props: {
  pkey: PanelKey;
  panels: Panels;
  workspaceH: number;
  instant: boolean;
  tabIn: boolean;
  onSwitch: (key: PanelKey) => void;
  children: ReactNode;
}) {
  const { pkey, panels, instant, tabIn, onSwitch } = props;
  const { side, open } = panels[pkey];
  const grouped = isGrouped(panels);
  const size = panelSize(panels, pkey);
  // sui bordi orizzontali l'altezza ha un tetto (45% dello spazio di lavoro); il contenuto scorre dentro
  const ph =
    isVertical(side) || props.workspaceH <= 0
      ? size.h
      : cappedPanelHeight(size.h, props.workspaceH);
  const cls = [
    "ec-panel",
    `ec-side-${side}`,
    open ? "ec-open" : "",
    grouped ? "ec-grouped" : "",
    isVertical(side) ? "" : "ec-horiz",
    instant ? "ec-instant" : "",
    tabIn ? "ec-tab-in" : "",
  ]
    .filter(Boolean)
    .join(" ");
  return (
    <aside
      className={cls}
      data-panel={pkey}
      data-side={side}
      aria-label={PANEL_LABEL[pkey]}
      inert={!open}
      style={{ "--pw": `${size.w}px`, "--ph": `${ph}px` } as CSSProperties}
    >
      {grouped ? (
        <nav className="ec-dock-tabs" aria-label="Pannelli">
          {PANEL_KEYS.map((t) => (
            <button
              key={t}
              type="button"
              className={"ec-dock-tab" + (t === pkey ? " ec-on" : "")}
              aria-pressed={t === pkey}
              title={PANEL_LABEL[t]}
              onClick={() => t !== pkey && onSwitch(t)}
            >
              {TAB_ICON[t]()}
              <span>{PANEL_LABEL[t]}</span>
            </button>
          ))}
        </nav>
      ) : null}
      <div className="ec-panel-body">{props.children}</div>
    </aside>
  );
}
```

### `src/etl-canvas/panels/EtlWorkspace.tsx`

175 righe

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
import { ExpandedPanel } from "../inspector/ExpandedPanel";
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
  // il box combinato aperto nel pannello espanso
  const [expanded, setExpanded] = useState<string | null>(null);
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
      {isClient && expanded ? (
        <ExpandedPanel
          store={store}
          nodeId={expanded}
          onClose={() => {
            const id = expanded;
            setExpanded(null);
            // il focus torna al pulsante di espansione del nodo
            setTimeout(
              () =>
                hostRef.current
                  ?.querySelector<HTMLElement>(`[data-node-id="${id}"] .ec-expand-btn`)
                  ?.focus(),
              0,
            );
          }}
          onConfigure={(index) => {
            setExpanded(null);
            store.dispatch({ type: "select", payload: { ids: [expanded] } });
            store.dispatch({ type: "inspect", payload: { node: expanded, step: index } });
            actions.open("insp");
          }}
          onDetachOutside={(index, x, y) => {
            // il punto di rilascio vive nel mondo, se cade dentro il canvas
            const stage = hostRef.current?.querySelector(".ec-stage")?.getBoundingClientRect();
            const inside =
              !!stage && x >= stage.left && x <= stage.right && y >= stage.top && y <= stage.bottom;
            setExpanded(null);
            store.dispatch({
              type: "detachStep",
              payload: {
                box: expanded,
                index,
                ...(inside && stage
                  ? { dropPoint: controller.toWorld(x - stage.left, y - stage.top) }
                  : {}),
              },
            });
          }}
        />
      ) : null}
      {isClient ? (
        <DockLayout
          store={store}
          actions={actions}
          controller={controller}
          canvas={(overlay) => (
            <EtlCanvas
              store={store}
              controller={controller}
              minHeight={0}
              overlay={overlay}
              onExpand={setExpanded}
            />
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

29 righe

```tsx
/**
 * Il guscio dell'Inspector: si apre e si chiude come gli altri pannelli e ospita
 * il contenuto di `inspector/` (Fase 6b.1).
 */
import type { EtlStore, Side } from "../../etl-store";
import { copy } from "../inspector/copy";
import { Inspector } from "../inspector/Inspector";
import { CloseArrow } from "./ui-icons";

export function InspectorShell(props: { store: EtlStore; side: Side; onClose: () => void }) {
  const { store, side } = props;
  return (
    <div className="ec-tb-inner ec-insp" data-testid="ec-inspector">
      <div className="ec-tb-head">
        <div className="ec-tb-title">Inspector</div>
        <button
          type="button"
          className="ec-close-btn"
          aria-label={copy.closeInspector}
          onClick={props.onClose}
        >
          <CloseArrow side={side} />
        </button>
      </div>
      <Inspector store={store} />
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

227 righe

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

/**
 * Tetto all'altezza dei pannelli sui bordi alto e basso (Fase 6b.1): la loro
 * altezza propria, ma non oltre il 45% dell'altezza disponibile (lo spazio di
 * lavoro); il contenuto scorre dentro il pannello.
 */
export const PANEL_HEIGHT_CAP = 0.45;

export function cappedPanelHeight(own: number, available: number): number {
  return Math.max(0, Math.min(own, Math.floor(available * PANEL_HEIGHT_CAP)));
}

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

