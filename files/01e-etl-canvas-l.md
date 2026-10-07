# 01e-etl-canvas-l.md

File in questo blocco:

- `src/etl-canvas/panels/EtlWorkspace.tsx`
- `src/etl-canvas/panels/InspectorShell.tsx`
- `src/etl-canvas/panels/Toolbox.tsx`
- `src/etl-canvas/panels/actions.ts`
- `src/etl-canvas/panels/animator.ts`
- `src/etl-canvas/panels/autoFit.ts`
- `src/etl-canvas/panels/csv.ts`
- `src/etl-canvas/panels/dockArea.ts`
- `src/etl-canvas/panels/families.ts`
- `src/etl-canvas/panels/layout.ts`
- `src/etl-canvas/panels/overlayLayout.ts`

---

### `src/etl-canvas/panels/EtlWorkspace.tsx`

187 righe

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
import { browserEnv, createLoop } from "../loop";
import type { Loop } from "../loop";
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
  // un solo ciclo requestAnimationFrame per tutto lo spazio di lavoro (cavi, pannelli, vista): nasce solo nel browser
  const [loop, setLoop] = useState<Loop | null>(null);
  useEffect(() => {
    const l = createLoop(browserEnv());
    setLoop(l);
    return () => l.dispose();
  }, []);
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
      {isClient && loop ? (
        <DockLayout
          store={store}
          actions={actions}
          controller={controller}
          loop={loop}
          canvas={(overlay, notice) => (
            <EtlCanvas
              store={store}
              controller={controller}
              minHeight={0}
              overlay={overlay}
              notice={notice}
              loop={loop}
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
      <Inspector store={store} horizontal={side === "top" || side === "bottom"} />
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
 * DOM. La vista non si compensa qui: dopo ogni cambio la adatta l'animatore
 * (animator.ts, con la regola di autoFit.ts).
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

### `src/etl-canvas/panels/animator.ts`

321 righe

```ts
/**
 * L'animatore dei pannelli e della vista (Fase 6b.1.1): UN solo orologio.
 * A ogni frame del ciclo condiviso (loop.ts) imposta, con lo stesso
 * progresso s(t), sia l'apertura di ciascun pannello (la variabile CSS
 * `--a`, 0 – 1, al posto della transizione CSS di larghezza e altezza) sia
 * la vista (x, y, zoom, interpolati linearmente) nello store. Stessa durata,
 * stessa curva, stesso frame di partenza.
 *
 * Tiene anche lo stato dello zoom automatico: lo zoom di intento (l'ultimo
 * scelto dall'utente: ogni cambio di vista che non viene da qui), il punto di
 * ripristino e l'avviso «nodi fuori dall'area». Nessun accesso diretto al
 * DOM: gli elementi dei pannelli si registrano (`register`) e le misure
 * arrivano da `metrics`.
 */
import type { Card } from "../../etl-core";
import type { EtlStore, PanelKey, Panels, Side, View } from "../../etl-store";
import type { Loop } from "../loop";
import {
  AUTOFIT_MS,
  OUT_OF_VIEW_NOTICE,
  easing,
  interpolateArea,
  interpolateView,
  planChange,
  sameArea,
} from "./autoFit";
import type { Area, Restore } from "./autoFit";
import { restArea } from "./dockArea";
import type { WorkspaceMetrics } from "./dockArea";
import { PANEL_KEYS, panelSize } from "./layout";

/** Un elemento di pannello: basta poter scrivere una variabile CSS. */
export interface PanelEl {
  readonly style: { setProperty(name: string, value: string): void };
}

/** Un pannello spostato su un altro bordo lascia sul vecchio un guscio vuoto che si richiude con lo stesso s. */
export interface GhostView {
  readonly id: string;
  readonly side: Side;
  readonly size: { readonly w: number; readonly h: number };
}

export interface DockSnapshot {
  /** Avviso non bloccante, o `null`. */
  readonly notice: string | null;
  readonly ghosts: readonly GhostView[];
}

export interface AnimatorDeps {
  readonly store: EtlStore;
  /** Il ciclo condiviso del canvas: l'animatore non ne crea un altro. */
  readonly loop: Loop;
  /** Misure dello spazio di lavoro, o `null` se non ancora disponibili. */
  readonly metrics: () => WorkspaceMetrics | null;
}

export interface DockAnimator {
  /** Comincia ad ascoltare lo store e si aggiunge al ciclo; restituisce la funzione per smettere. */
  start(): () => void;
  /** Registra (o, con `null`, toglie) l'elemento di un pannello (`tools`, `insp`) o di un guscio. */
  register(id: string, el: PanelEl | null): void;
  /** Il prossimo cambio dei pannelli è immediato (cambio di scheda: lo spazio non cambia). */
  snapNext(): void;
  /** L'area misurata a riposo (finestra ridimensionata, tetto d'altezza): ricalcola la vista senza animazione. */
  onMeasure(area: Area): void;
  subscribe(cb: () => void): () => void;
  getSnapshot(): DockSnapshot;
  /** Stato interno per le verifiche. */
  debug(): {
    intent: number;
    restore: Restore | null;
    animating: boolean;
    amounts: Record<PanelKey, number>;
  };
}

interface Ghost extends GhostView {
  amount: number;
}

interface Anim {
  t0: number | null;
  readonly from: Record<PanelKey, number>;
  readonly to: Record<PanelKey, number>;
  readonly gFrom: readonly number[];
  readonly v0: View;
  readonly v1: View;
  readonly a0: Area;
  readonly a1: Area;
  viewActive: boolean;
}

const lerp = (a: number, b: number, s: number): number => (s >= 1 ? b : a + (b - a) * s);

export function createDockAnimator(deps: AnimatorDeps): DockAnimator {
  const { store, loop } = deps;
  const els = new Map<string, PanelEl>();
  const listeners = new Set<() => void>();
  const init = store.getState();
  let panels: Panels = init.panels;
  let amounts: Record<PanelKey, number> = {
    tools: panels.tools.open ? 1 : 0,
    insp: panels.insp.open ? 1 : 0,
  };
  let ghosts: Ghost[] = [];
  let ghostSeq = 0;
  let anim: Anim | null = null;
  let curArea: Area | null = null;
  let intent = init.view.zoom;
  let restore: Restore | null = null;
  let lastView = init.view;
  let lastGraph = init.graph;
  let self = false;
  let snap = false;
  let snapshot: DockSnapshot = { notice: null, ghosts: [] };

  const publish = (): void => {
    snapshot = {
      notice: snapshot.notice,
      ghosts: ghosts.map(({ id, side, size }) => ({ id, side, size })),
    };
    listeners.forEach((l) => l());
  };
  const setNotice = (notice: string | null): void => {
    if (snapshot.notice === notice) return;
    snapshot = { ...snapshot, notice };
    listeners.forEach((l) => l());
  };

  const amountOf = (id: string): number =>
    id === "tools" || id === "insp" ? amounts[id] : (ghosts.find((g) => g.id === id)?.amount ?? 0);
  const write = (): void => {
    for (const [id, el] of els) el.style.setProperty("--a", String(amountOf(id)));
  };

  /** Cambia la vista nello store come azione dell'animatore (non è un'azione dell'utente: non cambia l'intento). */
  const setView = (v: View): void => {
    const cur = store.getState().view;
    if (cur.x === v.x && cur.y === v.y && cur.zoom === v.zoom) return;
    self = true;
    try {
      store.dispatch({ type: "setView", payload: { x: v.x, y: v.y, zoom: v.zoom } });
    } finally {
      self = false;
    }
    lastView = store.getState().view;
  };

  const apply = (a: Anim, s: number): void => {
    amounts = {
      tools: lerp(a.from.tools, a.to.tools, s),
      insp: lerp(a.from.insp, a.to.insp, s),
    };
    ghosts.forEach((g, i) => (g.amount = lerp(a.gFrom[i] ?? 0, 0, s)));
    write();
    curArea = interpolateArea(a.a0, a.a1, s);
    if (a.viewActive) setView(interpolateView(a.v0, a.v1, s));
  };

  const finish = (): void => {
    const a = anim;
    if (!a) return;
    apply(a, 1);
    anim = null;
    if (ghosts.length > 0) {
      ghosts = [];
      publish();
    }
  };

  const retarget = (next: Panels): void => {
    const prev = panels;
    panels = next;
    const instant = snap;
    snap = false;
    // un pannello spostato su un altro bordo si apre sul nuovo e lascia un guscio che si richiude sul vecchio
    for (const k of PANEL_KEYS) {
      if (prev[k].side !== next[k].side && amounts[k] > 0) {
        ghosts.push({
          id: `ghost-${k}-${++ghostSeq}`,
          side: prev[k].side,
          size: panelSize(prev, k),
          amount: amounts[k],
        });
        amounts = { ...amounts, [k]: 0 };
      }
    }
    const to: Record<PanelKey, number> = {
      tools: next.tools.open ? 1 : 0,
      insp: next.insp.open ? 1 : 0,
    };
    const ws = deps.metrics();
    if (!ws) {
      amounts = to;
      ghosts = [];
      anim = null;
      write();
      publish();
      return;
    }
    const st = store.getState();
    const a1 = restArea(next, ws);
    const a0 = curArea ?? restArea(prev, ws);
    const plan = planChange({
      cards: Object.values(st.graph.cards) as Card[],
      view: st.view,
      intent,
      prev: a0,
      next: a1,
      restore,
    });
    restore = plan.restore;
    setNotice(plan.fit?.outcome === 3 ? OUT_OF_VIEW_NOTICE : null);
    anim = {
      t0: null,
      from: { ...amounts },
      to,
      gFrom: ghosts.map((g) => g.amount),
      v0: st.view,
      v1: plan.view,
      a0,
      a1,
      viewActive: true,
    };
    publish();
    write();
    if (instant) {
      finish();
      return;
    }
    loop.wake();
  };

  const onStore = (): void => {
    if (self) return;
    const st = store.getState();
    if (st.view !== lastView) {
      // un cambio di vista che non viene da qui è un'azione dell'utente: nuovo intento, niente ripristino, l'animazione si ferma dov'è
      lastView = st.view;
      intent = st.view.zoom;
      restore = null;
      setNotice(null);
      if (anim) anim.viewActive = false;
    }
    if (st.graph !== lastGraph) {
      // nodi spostati, aggiunti o eliminati: la vista di prima non vale più
      lastGraph = st.graph;
      restore = null;
    }
    if (st.panels !== panels) retarget(st.panels);
  };

  const task = {
    frame(now: number): boolean {
      const a = anim;
      if (!a) return false;
      if (a.t0 === null) a.t0 = now;
      const p = Math.min(1, Math.max(0, (now - a.t0) / AUTOFIT_MS));
      if (p >= 1) {
        finish();
        return false;
      }
      apply(a, easing(p));
      return true;
    },
    settle(): void {
      finish();
    },
  };

  return {
    start() {
      const off = loop.add(task);
      const unsub = store.subscribe(onStore);
      return () => {
        unsub();
        off();
      };
    },
    register(id, el) {
      if (!el) {
        els.delete(id);
        return;
      }
      els.set(id, el);
      el.style.setProperty("--a", String(amountOf(id)));
    },
    snapNext() {
      snap = true;
    },
    onMeasure(area) {
      if (anim) return;
      if (!curArea) {
        curArea = area;
        return;
      }
      if (sameArea(curArea, area)) return;
      const st = store.getState();
      const plan = planChange({
        cards: Object.values(st.graph.cards) as Card[],
        view: st.view,
        intent,
        prev: curArea,
        next: area,
        restore,
      });
      restore = plan.restore;
      curArea = area;
      setNotice(plan.fit?.outcome === 3 ? OUT_OF_VIEW_NOTICE : null);
      setView(plan.view);
    },
    subscribe(cb) {
      listeners.add(cb);
      return () => listeners.delete(cb);
    },
    getSnapshot: () => snapshot,
    debug: () => ({ intent, restore, animating: anim !== null, amounts }),
  };
}
```

### `src/etl-canvas/panels/autoFit.ts`

299 righe

```ts
/**
 * Zoom automatico (Fase 6b.1.1): dopo ogni cambio dell'area del canvas
 * nessun nodo che era interamente visibile resta coperto o tagliato. Funzioni
 * pure, niente DOM. Sostituisce `keepVisible` della 6a.2 (mai modificare lo
 * zoom): se lo spazio non basta, scende lo zoom.
 *
 * Le posizioni dei nodi nel mondo non cambiano mai; cambia solo la vista
 * (x, y, zoom). Lo zoom di intento (l'ultimo scelto dall'utente) non lo
 * cambia nessuna modifica automatica: lo tiene chi chiama.
 */
import type { Card } from "../../etl-core";
import { CARD, LABEL_H } from "../../etl-layout";
import type { Size } from "../../etl-layout";
import type { View } from "../../etl-store";
import { MIN_ZOOM, bounds, fitView } from "../view";
import type { Insets } from "../view";

/** Margine dell'area sicura verso i bordi del canvas (token). */
export const SAFE_MARGIN = 24;
/** Durata della transizione di pannello e vista: un solo token per entrambi (era la transizione CSS dei pannelli, 0,3 s). */
export const AUTOFIT_MS = 300;
/** Tolleranza dei confronti geometrici (px) e dello zoom. */
const EPS = 1e-6;
/** Due aree sono «uguali» se differiscono meno di così (px). */
export const AREA_TOLERANCE = 0.5;

/** L'area del canvas e lo spazio occupato dai widget in sovrimpressione (minimappa, zoom). */
export interface Area {
  readonly size: Size;
  readonly insets: Insets;
}

interface Box {
  readonly x1: number;
  readonly y1: number;
  readonly x2: number;
  readonly y2: number;
}

/** Distanza dell'area sicura da ciascun bordo: il margine o l'ingombro del widget, il maggiore. */
export function safeInsets(area: Area): Insets {
  const m = (n: number) => Math.max(SAFE_MARGIN, n);
  return {
    top: m(area.insets.top),
    right: m(area.insets.right),
    bottom: m(area.insets.bottom),
    left: m(area.insets.left),
  };
}

export function safeBox(area: Area): Box {
  const i = safeInsets(area);
  return { x1: i.left, y1: i.top, x2: area.size.w - i.right, y2: area.size.h - i.bottom };
}

/** Rettangolo di un nodo (quadrato più etichetta) sullo schermo, con la vista data. */
function screenBox(c: Pick<Card, "x" | "y">, v: View): Box {
  const x1 = v.x + c.x * v.zoom;
  const y1 = v.y + c.y * v.zoom;
  return { x1, y1, x2: x1 + CARD * v.zoom, y2: y1 + (CARD + LABEL_H) * v.zoom };
}

const within = (b: Box, safe: Box, tol: number): boolean =>
  b.x1 >= safe.x1 - tol && b.y1 >= safe.y1 - tol && b.x2 <= safe.x2 + tol && b.y2 <= safe.y2 + tol;

/** Nodi interamente dentro l'area sicura con la vista data: quelli che il cambio non deve toccare. Un nodo già tagliato non è richiesto. */
export function requiredNodes(cards: readonly Card[], view: View, area: Area): readonly Card[] {
  const safe = safeBox(area);
  return cards.filter((c) => within(screenBox(c, view), safe, EPS));
}

/** `true` se il nodo sta interamente nell'area sicura (con tolleranza `tol`, in px). */
export function nodeInside(c: Pick<Card, "x" | "y">, view: View, area: Area, tol = EPS): boolean {
  return within(screenBox(c, view), safeBox(area), tol);
}

/** Spostamento minimo di un intervallo [lo, hi] perché stia in [a, b]; se non entra, si allinea ad `a`. */
function shiftInto(lo: number, hi: number, a: number, b: number): number {
  if (hi - lo > b - a) return a - lo;
  if (lo < a) return a - lo;
  if (hi > b) return b - hi;
  return 0;
}

/** Esito di `fitTarget`: 1 solo spinta e scorrimento, 2 cambia lo zoom, 3 nemmeno lo zoom minimo basta. */
export type FitOutcome = 1 | 2 | 3;
export type FitReason =
  /** nessun nodo richiesto: la vista non cambia */
  | "nessun-nodo"
  /** i nodi richiesti entrano allo zoom attuale */
  | "entrano"
  /** lo zoom scende perché i nodi richiesti non entrano */
  | "zoom-ridotto"
  /** lo zoom risale verso quello di intento perché ora c'è spazio */
  | "zoom-ripreso"
  /** non entrano nemmeno allo zoom minimo */
  | "zoom-minimo";

export interface FitResult {
  readonly view: View;
  readonly outcome: FitOutcome;
  readonly reason: FitReason;
  /** Quanti nodi erano richiesti. */
  readonly required: number;
}

export interface FitInput {
  readonly cards: readonly Card[];
  /** La vista corrente (in coordinate dell'area corrente). */
  readonly view: View;
  /** Lo zoom scelto dall'utente. */
  readonly intent: number;
  /** L'area prima del cambio: da qui i nodi richiesti. */
  readonly prev: Area;
  /** L'area dopo il cambio. */
  readonly next: Area;
}

/**
 * La vista di arrivo dopo un cambio d'area (vedi la regola in
 * docs/prompt 6b.1.1):
 * 1. i nodi richiesti entrano allo zoom che serve → solo scorrimento minimo
 *    (la spinta del bordo è già nella vista, che è relativa al canvas);
 * 2. altrimenti zoom = min(intento, zoom a cui entrano), riquadro centrato
 *    nell'area sicura (`fitView` sul riquadro dei nodi richiesti);
 * 3. se non entrano nemmeno allo zoom minimo: zoom minimo, riquadro allineato
 *    in alto a sinistra con lo spostamento minimo.
 *
 * «Lo zoom che serve» è min(intento, zoom a cui entrano): se lo zoom attuale
 * è già quello, è il caso 1; se è più basso (una discesa precedente) e ora c'è
 * spazio, risale (caso 2); se è più alto, scende.
 */
export function fitTarget(input: FitInput): FitResult {
  const { cards, view, intent, next } = input;
  const req = requiredNodes(cards, view, input.prev);
  const b = bounds(req);
  if (!b) return { view, outcome: 1, reason: "nessun-nodo", required: 0 };
  const safe = safeBox(next);
  const bw = b.x2 - b.x1;
  const bh = b.y2 - b.y1;
  const zFit = Math.min((safe.x2 - safe.x1) / bw, (safe.y2 - safe.y1) / bh);
  const zt = Math.min(intent, zFit);

  if (zt < MIN_ZOOM - EPS) {
    // caso 3: zoom minimo attorno al centro del riquadro, poi lo spostamento minimo per asse
    const cx = view.x + ((b.x1 + b.x2) / 2) * view.zoom;
    const cy = view.y + ((b.y1 + b.y2) / 2) * view.zoom;
    const z = MIN_ZOOM;
    const x0 = cx - ((b.x1 + b.x2) / 2) * z;
    const y0 = cy - ((b.y1 + b.y2) / 2) * z;
    const dx = shiftInto(x0 + b.x1 * z, x0 + b.x2 * z, safe.x1, safe.x2);
    const dy = shiftInto(y0 + b.y1 * z, y0 + b.y2 * z, safe.y1, safe.y2);
    return {
      view: { x: x0 + dx, y: y0 + dy, zoom: z },
      outcome: 3,
      reason: "zoom-minimo",
      required: req.length,
    };
  }

  if (Math.abs(zt - view.zoom) <= EPS) {
    const lo = { x: view.x + b.x1 * view.zoom, y: view.y + b.y1 * view.zoom };
    const hi = { x: view.x + b.x2 * view.zoom, y: view.y + b.y2 * view.zoom };
    const dx = shiftInto(lo.x, hi.x, safe.x1, safe.x2);
    const dy = shiftInto(lo.y, hi.y, safe.y1, safe.y2);
    return {
      view: dx === 0 && dy === 0 ? view : { ...view, x: view.x + dx, y: view.y + dy },
      outcome: 1,
      reason: "entrano",
      required: req.length,
    };
  }

  const fitted = fitView(req, next.size, safeInsets(next), { pad: 0, maxZoom: zt });
  return {
    view: fitted,
    outcome: 2,
    reason: zt < view.zoom ? "zoom-ridotto" : "zoom-ripreso",
    required: req.length,
  };
}

/** Due aree uguali (dimensioni e ingombri dei widget), entro la tolleranza. */
export function sameArea(a: Area, b: Area, tol = AREA_TOLERANCE): boolean {
  const close = (p: number, q: number) => Math.abs(p - q) <= tol;
  return (
    close(a.size.w, b.size.w) &&
    close(a.size.h, b.size.h) &&
    close(a.insets.top, b.insets.top) &&
    close(a.insets.right, b.insets.right) &&
    close(a.insets.bottom, b.insets.bottom) &&
    close(a.insets.left, b.insets.left)
  );
}

/**
 * Il punto di ripristino è ancora valido: stesse dimensioni e area sicura almeno altrettanto ampia
 * (ingombri uguali o minori su ogni lato). Così ogni nodo che era dentro l'area di partenza lo è
 * ancora nella vista salvata. Con le tacche tra gli ingombri, spostare un pannello su un altro
 * bordo cambia gli ingombri senza stringere l'area: il ripristino vale ancora.
 */
export function canRestore(saved: Area, next: Area, tol = AREA_TOLERANCE): boolean {
  const roomier = (p: number, q: number) => q <= p + tol;
  return (
    Math.abs(saved.size.w - next.size.w) <= tol &&
    Math.abs(saved.size.h - next.size.h) <= tol &&
    roomier(saved.insets.top, next.insets.top) &&
    roomier(saved.insets.right, next.insets.right) &&
    roomier(saved.insets.bottom, next.insets.bottom) &&
    roomier(saved.insets.left, next.insets.left)
  );
}

/** Il punto di ripristino: la vista di prima del primo adattamento automatico e la sua area. */
export interface Restore {
  readonly view: View;
  readonly area: Area;
}

export interface PlanInput extends FitInput {
  readonly restore: Restore | null;
}

export interface Plan {
  readonly view: View;
  /** `null` se la vista ripristinata è quella di prima. */
  readonly fit: FitResult | null;
  /** La vista è tornata esattamente a quella salvata. */
  readonly restored: boolean;
  /** Il punto di ripristino da tenere dopo questo cambio. */
  readonly restore: Restore | null;
}

/**
 * Come `fitTarget`, più il ripristino: se l'area è tornata uguale (o più ampia, vedi
 * `canRestore`) a quella del punto di ripristino (e l'utente non ha toccato nulla, cosa che chi chiama
 * garantisce azzerando `restore`), si torna ESATTAMENTE a quella vista. Il
 * primo adattamento che cambia la vista dopo un'azione dell'utente salva la
 * vista di prima.
 */
export function planChange(input: PlanInput): Plan {
  const { restore, next, view } = input;
  if (restore && canRestore(restore.area, next)) {
    return { view: restore.view, fit: null, restored: true, restore: null };
  }
  const fit = fitTarget(input);
  const changed = fit.view.x !== view.x || fit.view.y !== view.y || fit.view.zoom !== view.zoom;
  return {
    view: fit.view,
    fit,
    restored: false,
    restore: restore ?? (changed ? { view, area: input.prev } : null),
  };
}

/** Interpolazione LINEARE di x, y e zoom (non logaritmica: per convessità, ciò che sta dentro agli estremi sta dentro a ogni s). Estremi esatti. */
export function interpolateView(a: View, b: View, s: number): View {
  if (s <= 0) return a;
  if (s >= 1) return b;
  return {
    x: a.x + (b.x - a.x) * s,
    y: a.y + (b.y - a.y) * s,
    zoom: a.zoom + (b.zoom - a.zoom) * s,
  };
}

/** L'area a metà strada: dimensioni e ingombri interpolati linearmente con lo stesso s. */
export function interpolateArea(a: Area, b: Area, s: number): Area {
  if (s <= 0) return a;
  if (s >= 1) return b;
  const l = (p: number, q: number) => p + (q - p) * s;
  return {
    size: { w: l(a.size.w, b.size.w), h: l(a.size.h, b.size.h) },
    insets: {
      top: l(a.insets.top, b.insets.top),
      right: l(a.insets.right, b.insets.right),
      bottom: l(a.insets.bottom, b.insets.bottom),
      left: l(a.insets.left, b.insets.left),
    },
  };
}

/**
 * Easing puro e monotono, la stessa funzione per pannello e vista: da
 * s ∈ [0, 1] (tempo trascorso / durata) al progresso ∈ [0, 1]. È un
 * seno in entrata e in uscita (pendenza massima π/2 ≈ 1,57): partenza e arrivo
 * dolci e nessun frame che si sposta molto più della media (R4). La curva del
 * prototipo (cubic-bezier 0,32 0,72 0 1) partiva con una pendenza oltre 4 volte
 * la media e non rispettava il limite di 2,5.
 */
export function easing(s: number): number {
  if (s <= 0) return 0;
  if (s >= 1) return 1;
  return (1 - Math.cos(Math.PI * s)) / 2;
}

/** Avviso non bloccante quando il caso 3 lascia nodi fuori dall'area. */
export const OUT_OF_VIEW_NOTICE = "Alcuni nodi sono fuori dall'area visibile: usa Adatta";
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

### `src/etl-canvas/panels/dockArea.ts`

94 righe

```ts
/**
 * L'area del canvas calcolata dalle misure dello spazio di lavoro e dalla
 * misura dei pannelli (anche a metà apertura): funzioni pure, nessun DOM. È
 * la stessa geometria del CSS (griglia di `panels.css`), così l'animatore
 * conosce l'area di ogni frame senza leggere il DOM.
 */
import type { Size } from "../../etl-layout";
import type { PanelKey, Panels, Side } from "../../etl-store";
import type { Area } from "./autoFit";
import {
  EXTENT_PAD,
  MIN_CANVAS_HEIGHT,
  PANEL_KEYS,
  cappedPanelHeight,
  isVertical,
  notchHidden,
  notchOffset,
  panelSize,
} from "./layout";
import { overlayLayout } from "./overlayLayout";
import type { OverlayLayout } from "./overlayLayout";

/** Lo spazio di lavoro: la sua misura e l'altezza della barra dei controlli, che sta sopra a tutto. */
export interface WorkspaceMetrics {
  readonly w: number;
  readonly h: number;
  readonly barH: number;
}

/** Un approdo occupato (anche solo in parte) da un pannello: bordo, misura propria, quanto è aperto (0 – 1). */
export interface Slot {
  readonly side: Side;
  readonly size: { readonly w: number; readonly h: number };
  readonly amount: number;
}

/** Quanto spazio toglie al canvas uno slot, nella misura attuale dello spazio di lavoro. */
export function slotExtent(slot: Slot, ws: WorkspaceMetrics): number {
  const own = isVertical(slot.side)
    ? slot.size.w
    : ws.h > 0
      ? cappedPanelHeight(slot.size.h, ws.h)
      : slot.size.h;
  return slot.amount * (own + EXTENT_PAD);
}

/** Dimensioni del canvas (la riga centrale della griglia) con questi slot. */
export function canvasSize(ws: WorkspaceMetrics, slots: readonly Slot[]): Size {
  const sum = (sides: readonly Side[]) =>
    slots.filter((s) => sides.includes(s.side)).reduce((n, s) => n + slotExtent(s, ws), 0);
  return {
    w: Math.max(0, ws.w - sum(["left", "right"])),
    h: Math.max(MIN_CANVAS_HEIGHT, ws.h - ws.barH - sum(["top", "bottom"])),
  };
}

/** Gli slot dei pannelli nel loro stato, con la parte aperta di ciascuno. */
export function slotsOf(panels: Panels, amounts: Readonly<Record<PanelKey, number>>): Slot[] {
  return PANEL_KEYS.map((k) => ({
    side: panels[k].side,
    size: panelSize(panels, k),
    amount: amounts[k],
  }));
}

/** Gli slot a riposo: aperti del tutto o chiusi. */
export function restSlots(panels: Panels): Slot[] {
  return slotsOf(panels, {
    tools: panels.tools.open ? 1 : 0,
    insp: panels.insp.open ? 1 : 0,
  });
}

/** Disposizione dei widget in sovrimpressione per un'area e uno stato dei pannelli. */
export function overlayFor(panels: Panels, area: Size): OverlayLayout {
  const openSide = PANEL_KEYS.map((k) => (panels[k].open ? panels[k].side : null)).find(Boolean);
  return overlayLayout({
    area,
    openSide: openSide ?? null,
    notches: PANEL_KEYS.map((k) => ({
      key: k,
      side: panels[k].side,
      offset: notchOffset(panels, k),
      visible: !notchHidden(panels, k),
    })),
  });
}

/** L'area a riposo (pannelli aperti o chiusi del tutto) con gli ingombri dei widget. */
export function restArea(panels: Panels, ws: WorkspaceMetrics): Area {
  const size = canvasSize(ws, restSlots(panels));
  return { size, insets: overlayFor(panels, size).insets };
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

131 righe

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
import type { PanelKey, Panels, Side } from "../../etl-store";

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

273 righe

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
 *   il resto; senza posto non si mostra;
 * - le tacche visibili stanno nell'area sicura: ognuna riserva ai nodi, lungo il
 *   proprio bordo, la sua larghezza più `NOTCH_CLEARANCE` (16 px).
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
/** Respiro minimo tra una tacca visibile e i nodi richiesti: le tacche stanno nell'area sicura (Fase 6b.2, Passo 0). */
export const NOTCH_CLEARANCE = 16;
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

/** Fascia da lasciare libera lungo il bordo di ogni tacca visibile: la sua larghezza più il respiro. */
function notchInsets(visible: readonly NotchInput[]): Insets {
  const band = NOTCH_SHORT + NOTCH_CLEARANCE;
  let { top, right, bottom, left } = NO_INSETS;
  for (const n of visible) {
    if (n.side === "top") top = band;
    else if (n.side === "right") right = band;
    else if (n.side === "bottom") bottom = band;
    else left = band;
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

  const widgetInsets = insetsFor(
    [...(minimap.rect ? [minimap.rect] : []), zoom].filter((r) => inside(r, area)),
    area,
  );
  const tabs = notchInsets(input.notches.filter((n) => n.visible));
  const insets: Insets = {
    top: Math.max(widgetInsets.top, tabs.top),
    right: Math.max(widgetInsets.right, tabs.right),
    bottom: Math.max(widgetInsets.bottom, tabs.bottom),
    left: Math.max(widgetInsets.left, tabs.left),
  };
  return { minimap, zoom, hint, notches, insets: area.w > 0 && area.h > 0 ? insets : NO_INSETS };
}
```

