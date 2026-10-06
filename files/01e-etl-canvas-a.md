# 01e-etl-canvas-a.md

File in questo blocco:

- `src/etl-canvas/EtlCanvas.tsx`
- `src/etl-canvas/Links.tsx`
- `src/etl-canvas/Minimap.tsx`
- `src/etl-canvas/NOTE_DIVERGENZE.md`
- `src/etl-canvas/Node.tsx`

---

### `src/etl-canvas/EtlCanvas.tsx`

579 righe

```tsx
import {
  useCallback,
  useEffect,
  useLayoutEffect,
  useMemo,
  useRef,
  useState,
  useSyncExternalStore,
} from "react";
import type { CSSProperties, PointerEvent as ReactPointerEvent } from "react";
import { CARD, WORLD_H, WORLD_W } from "../etl-layout";
import type { Size } from "../etl-layout";
import { linkKey } from "../etl-layout";
import { linkLive, nodeStates } from "../etl-store";
import type { EtlStore } from "../etl-store";
import { useEtlState } from "../etl-store/react";
import { fit, zoomAtPoint, zoomIn, zoomOut, zoomReset } from "./actions";
import { createMotionEngine } from "./engine";
import { createInteractionController } from "./interaction";
import type { DownTarget, InteractionController, PointerInput, PortSide } from "./interaction";
import type { LinkInput } from "./engine";
import { browserEnv } from "./loop";
import type { Loop, LoopEnv } from "./loop";
import { MotionContext } from "./motion";
import "./canvas.css";
import { Links } from "./Links";
import { Minimap } from "./Minimap";
import { nodeView } from "./model";
import { Node } from "./Node";
import { overlayLayout } from "./panels/overlayLayout";
import type { OverlayLayout } from "./panels/overlayLayout";
import { WHEEL_ZOOM_RATE, wheelPan } from "./view";

const PORT_SIDES: readonly string[] = ["r", "b", "l", "t"];

/** Classifica il punto in cui è iniziato il gesto (unico punto in cui si legge il DOM). */
function classify(t: EventTarget | null): DownTarget {
  const el = t as HTMLElement | null;
  if (!el || typeof el.closest !== "function") return { kind: "ignore" };
  if (el.closest(".ec-zoom, .ec-minimap, .ec-minimap-toggle, .ec-confirm, .ec-node-btn")) {
    return { kind: "ignore" };
  }
  const nodeEl = el.closest<HTMLElement>("[data-node-id]");
  const id = nodeEl?.dataset["nodeId"];
  if (id) {
    const port = el.closest<HTMLElement>("[data-port]")?.dataset["port"];
    if (port && PORT_SIDES.includes(port)) return { kind: "port", id, side: port as PortSide };
    return { kind: "node", id };
  }
  return { kind: "background" };
}

function isTyping(t: EventTarget | null): boolean {
  const el = t as HTMLElement | null;
  return !!el && (el.tagName === "INPUT" || el.tagName === "TEXTAREA" || el.isContentEditable);
}

export interface CanvasSurfaceProps {
  readonly store: EtlStore;
  /** Dimensioni dell'area visibile, in pixel. */
  readonly size: Size;
  /** Ambiente delle animazioni (orologio, rAF, visibilità, movimento ridotto); nei test si sostituisce. */
  readonly env?: LoopEnv;
  /** Il ciclo condiviso con lo spazio di lavoro (animatore dei pannelli): se c'è, il motore si aggiunge a quello. */
  readonly loop?: Loop | undefined;
  /** Controller dei gesti condiviso con chi sta fuori dal canvas (la cassetta); se manca, ne nasce uno. */
  readonly controller?: InteractionController | undefined;
  /** Posizione dei widget in sovrimpressione, decisa da `overlayLayout`; se manca, quella senza pannelli. */
  readonly overlay?: OverlayLayout | undefined;
  /** Avviso non bloccante nel suggerimento in sovrimpressione, se nessun suggerimento dei gesti lo occupa. */
  readonly notice?: string | null | undefined;
  /** Espansione di un box combinato (pulsante sul nodo). */
  readonly onExpand?: ((id: string) => void) | undefined;
}

/**
 * La resa del canvas per uno store e una dimensione date. Non tocca né
 * `window` né `document` durante il rendering: gli ascoltatori (spazio,
 * rotella) si registrano in un effetto, quindi si può rendere anche in
 * Node (test, rendering lato server).
 */
export function CanvasSurface(props: CanvasSurfaceProps) {
  const { store, size, env } = props;
  const graph = useEtlState((s) => s.graph, store);
  const view = useEtlState((s) => s.view, store);
  const selection = useEtlState((s) => s.selection, store);
  const maxBends = useEtlState((s) => s.options.maxBends, store);
  const flowOnlyIfValid = useEtlState((s) => s.options.flowOnlyIfValid, store);
  const [engine] = useState(createMotionEngine);
  const [ownController] = useState(() => props.controller ?? createInteractionController(store));
  const controller = props.controller ?? ownController;
  const ui = useSyncExternalStore(controller.subscribe, controller.getUi, controller.getUi);
  const stageRef = useRef<HTMLDivElement>(null);
  const layout = useMemo(
    () => props.overlay ?? overlayLayout({ area: size, openSide: null, notches: [] }),
    [props.overlay, size],
  );
  const [mmOpen, setMmOpen] = useState(false);
  const onDeleteNode = useCallback(
    (id: string) => controller.requestDeleteNodes([id]),
    [controller],
  );
  const cancelRef = useRef<HTMLButtonElement>(null);
  const confirmOpen = ui.confirm !== null;
  // la finestra di conferma si apre con il focus su «Annulla» (Esc annulla)
  useEffect(() => {
    if (confirmOpen) cancelRef.current?.focus();
  }, [confirmOpen]);
  const [spaceDown, setSpaceDown] = useState(false);
  const [panning, setPanning] = useState(false);
  const spaceRef = useRef(false);

  const states = useMemo(() => nodeStates(graph), [graph]);
  // eslint-disable-next-line react-hooks/exhaustive-deps -- i percorsi dipendono da grafo e limite di snodi, che sono nelle dipendenze
  const routes = useMemo(() => store.getRoutes(), [store, graph, maxBends]);
  const cards = useMemo(() => Object.values(graph.cards), [graph]);
  // il riquadro di selezione mostra già i nodi che comprende; il resto della selezione è nello store
  const marqueeIds = ui.marquee?.ids;
  const selected = useMemo(() => new Set(marqueeIds ?? selection), [selection, marqueeIds]);
  const doomed = useMemo(() => new Set(ui.confirm?.removed ?? []), [ui.confirm]);
  const nodes = useMemo(
    () =>
      cards.map((c) =>
        nodeView(c, states[c.id] ?? null, selected.has(c.id), {
          dragging: ui.dragging.includes(c.id),
          drop: ui.drop?.id === c.id ? ui.drop.outcome : null,
          doomed: doomed.has(c.id),
        }),
      ),
    [cards, states, selected, ui.dragging, ui.drop, doomed],
  );

  // animazioni: un solo ciclo condiviso, avviato solo nel browser (gli effetti non girano sul server)
  useLayoutEffect(() => {
    engine.start(env ?? browserEnv(), props.loop);
    return () => engine.stop();
  }, [engine, env, props.loop]);

  // dopo ogni rendering dei cavi: percorsi e cavi attivi per il motore (sola lettura)
  useLayoutEffect(() => {
    const state = store.getState();
    const inputs: LinkInput[] = [];
    for (const l of graph.links) {
      const route = routes[linkKey(l)];
      if (!route) continue;
      inputs.push({
        key: linkKey(l),
        live: linkLive(state, l),
        pts: route.pts,
        d: route.d,
        pa: route.pa,
        pb: route.pb,
      });
    }
    engine.update({ links: inputs, gesturing: store.isGesturing() });
  }, [engine, store, graph, routes, flowOnlyIfValid]);

  // durante un gesto di trascinamento il flusso si ferma (prototipo, riga 1485)
  useEffect(() => {
    // anche il cavo tirato da una porta ferma il flusso (prototipo, riga 3984)
    const sync = () =>
      engine.setGesturing(store.isGesturing() || controller.getUi().tempLink !== null);
    sync();
    const offStore = store.subscribe(sync);
    const offUi = controller.subscribe(sync);
    return () => {
      offStore();
      offUi();
    };
  }, [engine, store, controller]);

  // barra spaziatrice: navigazione temporanea (prototipo, righe 4168-4177)
  useEffect(() => {
    const down = (e: KeyboardEvent) => {
      if (e.code !== "Space" || isTyping(e.target)) return;
      spaceRef.current = true;
      controller.setSpace(true);
      setSpaceDown(true);
      e.preventDefault();
    };
    const up = (e: KeyboardEvent) => {
      if (e.code !== "Space") return;
      spaceRef.current = false;
      controller.setSpace(false);
      setSpaceDown(false);
    };
    document.addEventListener("keydown", down);
    document.addEventListener("keyup", up);
    return () => {
      document.removeEventListener("keydown", down);
      document.removeEventListener("keyup", up);
    };
  }, [controller]);

  // tastiera: Canc, Cmd/Ctrl+D/A/Z/Maiusc+Z/Y, frecce, Esc (prototipo, righe 4629-4650)
  useEffect(() => {
    const key = (e: KeyboardEvent) => {
      if (e.code === "Space") return;
      const handled = controller.key({
        key: e.key,
        metaKey: e.metaKey,
        ctrlKey: e.ctrlKey,
        shiftKey: e.shiftKey,
        typing: isTyping(e.target),
      });
      if (handled) e.preventDefault();
    };
    document.addEventListener("keydown", key);
    return () => document.removeEventListener("keydown", key);
  }, [controller]);

  // un gesto in corso non sopravvive allo smontaggio
  const cleanupRef = useRef<(() => void) | null>(null);
  useEffect(() => () => cleanupRef.current?.(), []);

  // rotella: Cmd/Ctrl = zoom attorno al puntatore; altrimenti sposta la vista (righe 4092-4102)
  useEffect(() => {
    const el = stageRef.current;
    if (!el) return;
    const wheel = (e: WheelEvent) => {
      e.preventDefault();
      const v = store.getState().view;
      if (e.ctrlKey || e.metaKey) {
        const r = el.getBoundingClientRect();
        zoomAtPoint(
          store,
          e.clientX - r.left,
          e.clientY - r.top,
          v.zoom * Math.exp(-e.deltaY * WHEEL_ZOOM_RATE),
        );
      } else {
        const next = wheelPan(v, e);
        store.dispatch({ type: "setView", payload: { x: next.x, y: next.y } });
      }
    };
    el.addEventListener("wheel", wheel, { passive: false });
    return () => el.removeEventListener("wheel", wheel);
  }, [store]);

  // pan (barra spaziatrice + trascinamento, o tasto centrale: righe 4036-4054) e gesti (nodi, porte, sfondo)
  const onPointerDown = (e: ReactPointerEvent<HTMLDivElement>) => {
    if (spaceRef.current || e.button === 1) {
      e.preventDefault();
      const sx = e.clientX;
      const sy = e.clientY;
      const { x: ox, y: oy } = store.getState().view;
      setPanning(true);
      const move = (ev: PointerEvent) =>
        store.dispatch({
          type: "setView",
          payload: { x: ox + (ev.clientX - sx), y: oy + (ev.clientY - sy) },
        });
      const up = () => {
        window.removeEventListener("pointermove", move);
        window.removeEventListener("pointerup", up);
        setPanning(false);
      };
      window.addEventListener("pointermove", move);
      window.addEventListener("pointerup", up);
      return;
    }
    if (e.pointerType === "mouse" && e.button !== 0) return;
    const rect = stageRef.current?.getBoundingClientRect();
    if (!rect) return;
    const input = (ev: { clientX: number; clientY: number; shiftKey: boolean }): PointerInput => ({
      x: ev.clientX - rect.left,
      y: ev.clientY - rect.top,
      shiftKey: ev.shiftKey,
    });
    if (!controller.down(classify(e.target), input(e))) return;
    e.preventDefault();
    const move = (ev: PointerEvent) => controller.move(input(ev));
    const stop = () => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
      window.removeEventListener("pointercancel", cancel);
      cleanupRef.current = null;
    };
    const up = (ev: PointerEvent) => {
      stop();
      controller.up(input(ev));
    };
    const cancel = () => {
      stop();
      controller.cancel();
    };
    cleanupRef.current = cancel;
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
    window.addEventListener("pointercancel", cancel);
  };

  const stageClass = "ec-stage" + (panning ? " ec-panning" : spaceDown ? " ec-pannable" : "");

  return (
    <MotionContext.Provider value={engine}>
      <div className="etl-canvas" data-testid="etl-canvas">
        <div ref={stageRef} className={stageClass} tabIndex={-1} onPointerDown={onPointerDown}>
          <div
            className="ec-world"
            data-testid="ec-world"
            style={{
              width: WORLD_W,
              height: WORLD_H,
              transform: `translate(${view.x}px, ${view.y}px) scale(${view.zoom})`,
            }}
          >
            <Links graph={graph} routes={routes} hot={ui.insertLink} />
            {nodes.map((n) => (
              <Node
                key={n.id}
                node={n}
                onDelete={onDeleteNode}
                {...(props.onExpand ? { onExpand: props.onExpand } : {})}
              />
            ))}
            {ui.tempLink ? (
              <svg
                className={"ec-temp-link" + (ui.tempLink.valid ? " ec-valid" : "")}
                data-testid="ec-temp-link"
                aria-hidden="true"
              >
                <path
                  d={`M ${ui.tempLink.from.x} ${ui.tempLink.from.y} L ${ui.tempLink.to.x} ${ui.tempLink.to.y}`}
                />
                <circle cx={ui.tempLink.to.x} cy={ui.tempLink.to.y} r={4} />
              </svg>
            ) : null}
          </div>
          {ui.marquee ? (
            <div
              className="ec-marquee"
              data-testid="ec-marquee"
              style={{
                left: ui.marquee.rect.x,
                top: ui.marquee.rect.y,
                width: ui.marquee.rect.w,
                height: ui.marquee.rect.h,
              }}
            />
          ) : null}
          {(ui.hint ?? props.notice) && layout.hint ? (
            <div
              className={"ec-hint" + (ui.hint ? "" : " ec-notice")}
              role="status"
              data-testid={ui.hint ? undefined : "ec-notice"}
              style={rectStyle(layout.hint)}
            >
              {ui.hint ?? props.notice}
            </div>
          ) : null}
          {ui.confirm ? (
            <div
              className="ec-confirm"
              role="alertdialog"
              aria-labelledby="ec-confirm-title"
              aria-describedby="ec-confirm-text"
              data-testid="ec-confirm"
              aria-modal={ui.confirm.kind === "clear" || undefined}
              data-kind={ui.confirm.kind ?? "delete"}
              style={
                ui.confirm.kind === "clear"
                  ? confirmCentered(size)
                  : confirmPosition(graph.cards[ui.confirm.ids[0] ?? ""], view, size)
              }
              onPointerDown={(e) => e.stopPropagation()}
              onKeyDown={(e) => {
                // Tab resta tra i due pulsanti; il resto dei tasti non arriva ai comandi del canvas
                if (e.key === "Tab") {
                  const btns = e.currentTarget.querySelectorAll("button");
                  const first = btns[0];
                  const last = btns[btns.length - 1];
                  if (e.shiftKey && document.activeElement === first) {
                    e.preventDefault();
                    last?.focus();
                  } else if (!e.shiftKey && document.activeElement === last) {
                    e.preventDefault();
                    first?.focus();
                  }
                }
              }}
            >
              <div>
                <div className="ec-confirm-title" id="ec-confirm-title">
                  {ui.confirm.title}
                </div>
                <div className="ec-confirm-text" id="ec-confirm-text">
                  {ui.confirm.text}
                </div>
              </div>
              <div className="ec-confirm-actions">
                <button
                  ref={cancelRef}
                  type="button"
                  className="ec-confirm-cancel"
                  onClick={controller.cancelConfirm}
                >
                  Annulla
                </button>
                <button type="button" className="ec-confirm-ok" onClick={controller.confirmDelete}>
                  {ui.confirm.kind === "clear" ? "Svuota" : "Elimina"}
                </button>
              </div>
            </div>
          ) : null}
          {cards.length === 0 ? (
            <div className="ec-empty" data-testid="ec-empty">
              <div className="ec-empty-title">Il canvas è vuoto</div>
              <div className="ec-empty-text">Aggiungi un dataset per iniziare.</div>
            </div>
          ) : null}
          {layout.minimap.rect && (!layout.minimap.compact || mmOpen) ? (
            <Minimap
              cards={cards}
              view={view}
              size={size}
              rect={
                layout.minimap.compact
                  ? (layout.minimap.expanded ?? layout.minimap.rect)
                  : layout.minimap.rect
              }
              onView={(v) => store.dispatch({ type: "setView", payload: v })}
              {...(layout.minimap.compact ? { onClose: () => setMmOpen(false) } : {})}
            />
          ) : null}
          {layout.minimap.rect && layout.minimap.compact && !mmOpen ? (
            <button
              type="button"
              className="ec-minimap-toggle"
              data-testid="minimap-toggle"
              aria-label="Apri la minimappa"
              style={rectStyle(layout.minimap.rect)}
              onPointerDown={(e) => e.stopPropagation()}
              onClick={() => setMmOpen(true)}
            >
              <svg
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                strokeWidth={2.2}
                strokeLinecap="round"
                strokeLinejoin="round"
                aria-hidden="true"
              >
                <rect x="3" y="5" width="18" height="14" rx="2.5" />
                <rect x="7" y="9" width="4" height="4" rx="1" />
                <rect x="13" y="12" width="4" height="3" rx="1" />
              </svg>
            </button>
          ) : null}
          <div
            className="ec-zoom"
            data-testid="ec-zoom"
            style={rectStyle(layout.zoom)}
            onPointerDown={(e) => e.stopPropagation()}
          >
            <button type="button" aria-label="Riduci" onClick={() => zoomOut(store, size)}>
              −
            </button>
            <button type="button" aria-label="Zoom al 100%" onClick={() => zoomReset(store, size)}>
              {Math.round(view.zoom * 100)}%
            </button>
            <button type="button" aria-label="Ingrandisci" onClick={() => zoomIn(store, size)}>
              +
            </button>
            <button
              type="button"
              className="ec-fit"
              onClick={() => fit(store, size, layout.insets)}
            >
              Adatta
            </button>
          </div>
        </div>
      </div>
    </MotionContext.Provider>
  );
}

/** Posizione e misura di un widget, decise da `overlayLayout`. */
function rectStyle(r: { x: number; y: number; w: number; h: number }): CSSProperties {
  return { left: r.x, top: r.y, width: r.w, height: r.h, right: "auto", bottom: "auto" };
}

const CONFIRM_W = 246;
const CONFIRM_H = 168;

/** Svuota: la conferma sta al centro dell'area. */
function confirmCentered(size: Size): { left: number; top: number } {
  return {
    left: Math.max(8, (size.w - CONFIRM_W) / 2),
    top: Math.max(8, (size.h - CONFIRM_H) / 2),
  };
}

/** La conferma sta sopra il nodo (o sotto, se non c'è posto), dentro l'area: prototipo, righe 4515-4522. */
function confirmPosition(
  card: { x: number; y: number } | undefined,
  view: { x: number; y: number; zoom: number },
  size: Size,
): { left: number; top: number } {
  if (!card) return { left: 8, top: 8 };
  const cx = (card.x + CARD / 2) * view.zoom + view.x;
  const top = card.y * view.zoom + view.y - CONFIRM_H - 10;
  const bottom = (card.y + CARD) * view.zoom + view.y + 10;
  return {
    left: Math.max(8, Math.min(size.w - CONFIRM_W - 8, cx - CONFIRM_W / 2)),
    top: top < 8 ? bottom : top,
  };
}

const noopSubscribe = () => () => {};

/**
 * Il canvas. Si monta solo nel browser: sul server e nel primo rendering
 * di idratazione produce sempre lo stesso contenitore vuoto, così non ci
 * sono differenze da riconciliare. Misura l'area con un ResizeObserver.
 */
export function EtlCanvas(props: {
  store: EtlStore;
  controller?: InteractionController;
  /** Altezza minima del contenitore (px). Nello spazio di lavoro è la riga centrale a fissarla (0). */
  minHeight?: number;
  /** Posizione dei widget in sovrimpressione (vedi `overlayLayout`). */
  overlay?: OverlayLayout | undefined;
  /** Avviso non bloccante da mostrare nel suggerimento in sovrimpressione (zoom automatico al minimo). */
  notice?: string | null | undefined;
  /** Ciclo di animazione condiviso con lo spazio di lavoro. */
  loop?: Loop | undefined;
  onExpand?: ((id: string) => void) | undefined;
}) {
  const isClient = useSyncExternalStore(
    noopSubscribe,
    () => true,
    () => false,
  );
  const hostRef = useRef<HTMLDivElement>(null);
  const [size, setSize] = useState<Size | null>(null);

  useEffect(() => {
    const el = hostRef.current;
    if (!el) return;
    const measure = () => {
      const w = el.clientWidth;
      const h = el.clientHeight;
      setSize((prev) => (prev && prev.w === w && prev.h === h ? prev : { w, h }));
    };
    measure();
    const ro = new ResizeObserver(measure);
    ro.observe(el);
    return () => ro.disconnect();
  }, [isClient]);

  return (
    <div
      ref={hostRef}
      className="etl-canvas-host"
      style={{
        position: "relative",
        width: "100%",
        height: "100%",
        minHeight: props.minHeight ?? 520,
      }}
    >
      {isClient && size ? (
        <CanvasSurface
          store={props.store}
          size={size}
          controller={props.controller}
          overlay={props.overlay}
          notice={props.notice}
          loop={props.loop}
          onExpand={props.onExpand}
        />
      ) : null}
    </div>
  );
}
```

### `src/etl-canvas/Links.tsx`

79 righe

```tsx
import { memo, useCallback } from "react";
import type { Graph, Link } from "../etl-core";
import { linkKey } from "../etl-layout";
import type { LinkRoute, LinkRoutes } from "../etl-layout";
import { useMotion } from "./motion";

/**
 * Un cavo: il percorso calcolato da etl-store (invariato), i suoi capi, e
 * due elementi che il motore delle animazioni aggiorna a mano (il
 * tracciato precedente durante una dissolvenza, il tubo del flusso).
 * Senza animazioni (rendering lato server, test) restano vuoti.
 */
const LinkView = memo(function LinkView(props: {
  k: string;
  route: LinkRoute;
  fromDataset: boolean;
  toDataset: boolean;
  hot: boolean;
}) {
  const { k, route } = props;
  const motion = useMotion();
  const ref = useCallback(
    (g: SVGGElement | null) => {
      motion?.registerLink(k, g);
    },
    [motion, k],
  );
  return (
    <g ref={ref} data-link={k}>
      <path className="ec-link-ghost" d="" opacity={0} />
      <path className={props.hot ? "ec-link ec-link-hot" : "ec-link"} d={route.d} />
      <circle
        data-dot="a"
        className={props.fromDataset ? "ec-link-dot-ds" : "ec-link-dot-op"}
        cx={route.pa.x}
        cy={route.pa.y}
        r={2.6}
      />
      <circle
        data-dot="b"
        className={props.toDataset ? "ec-link-dot-ds" : "ec-link-dot-op"}
        cx={route.pb.x}
        cy={route.pb.y}
        r={2.6}
      />
      <path className="ec-flow" d="" />
    </g>
  );
});

/** I cavi (prototipo `drawLinks`, righe 1399-1409). */
export const Links = memo(function Links(props: {
  graph: Graph;
  routes: LinkRoutes;
  /** Chiave del cavo in cui si inserirebbe la lavorazione trascinata. */
  hot?: string | null;
}) {
  const { graph, routes, hot = null } = props;
  return (
    <svg className="ec-links" width="100%" height="100%" aria-hidden="true">
      {graph.links.map((l: Link) => {
        const k = linkKey(l);
        const route = routes[k];
        if (!route) return null;
        return (
          <LinkView
            key={k}
            k={k}
            route={route}
            fromDataset={graph.cards[l.from]?.kind === "dataset"}
            toDataset={graph.cards[l.to]?.kind === "dataset"}
            hot={k === hot}
          />
        );
      })}
    </svg>
  );
});
```

### `src/etl-canvas/Minimap.tsx`

88 righe

```tsx
import { useRef } from "react";
import type { PointerEvent as ReactPointerEvent } from "react";
import type { Card } from "../etl-core";
import { CARD } from "../etl-layout";
import type { Size } from "../etl-layout";
import type { View } from "../etl-store";
import { MM_NODE_MIN, minimapFrame, viewFromMinimap, visibleWorld } from "./view";

/** Minimappa: il flusso in miniatura e la porzione visibile (prototipo `renderMinimap`, righe 4133-4165). */
export function Minimap(props: {
  cards: readonly Card[];
  view: View;
  size: Size;
  /** Posizione e misura decise da `overlayLayout`. */
  rect: { x: number; y: number; w: number; h: number };
  onView: (view: View) => void;
  /** Presente quando la minimappa è stata espansa da un pulsante compatto. */
  onClose?: () => void;
}) {
  const { cards, view, size, rect, onView, onClose } = props;
  const ref = useRef<HTMLDivElement>(null);
  const frame = minimapFrame(cards, view, size);
  const v = visibleWorld(view, size);

  const go = (e: { clientX: number; clientY: number }) => {
    const el = ref.current;
    if (!el) return;
    const r = el.getBoundingClientRect();
    onView(viewFromMinimap(frame, view, size, e.clientX - r.left, e.clientY - r.top));
  };

  const onPointerDown = (e: ReactPointerEvent<HTMLDivElement>) => {
    e.stopPropagation();
    go(e);
    const move = (ev: PointerEvent) => go(ev);
    const up = () => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
    };
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
  };

  return (
    <div
      ref={ref}
      className="ec-minimap"
      data-testid="minimap"
      aria-hidden={onClose ? undefined : true}
      onPointerDown={onPointerDown}
      style={{ left: rect.x, top: rect.y, width: rect.w, height: rect.h, bottom: "auto" }}
    >
      {onClose ? (
        <button
          type="button"
          className="ec-mm-close"
          aria-label="Chiudi la minimappa"
          onPointerDown={(e) => e.stopPropagation()}
          onClick={onClose}
        >
          ×
        </button>
      ) : null}
      {cards.map((c) => (
        <div
          key={c.id}
          className={c.kind === "dataset" ? "ec-mm-node ec-ds" : "ec-mm-node"}
          style={{
            left: frame.ox + (c.x - frame.x1) * frame.k,
            top: frame.oy + (c.y - frame.y1) * frame.k,
            width: Math.max(MM_NODE_MIN, CARD * frame.k),
            height: Math.max(MM_NODE_MIN, CARD * frame.k),
          }}
        />
      ))}
      <div
        className="ec-mm-view"
        style={{
          left: frame.ox + (v.x1 - frame.x1) * frame.k,
          top: frame.oy + (v.y1 - frame.y1) * frame.k,
          width: (v.x2 - v.x1) * frame.k,
          height: (v.y2 - v.y1) * frame.k,
        }}
      />
    </div>
  );
}
```

### `src/etl-canvas/NOTE_DIVERGENZE.md`

173 righe

```md
# Note di divergenza — etl-canvas (Fase 4b)

Scelte in cui le animazioni del canvas differiscono dal prototipo
`docs/prototype/isa-fusion-prototype.html`, o che il prototipo non specifica.

## 1. Transizione dei percorsi: interpolazione dei punti, non easing degli angoli

**Prototipo.** Non ha una transizione tra due percorsi. `drawLinks` (righe
1300-1409) ricalcola la scelta della porta ogni 110 ms e avvicina, a ogni
frame, l'angolo di ciascuna porta (`easeAngle`, righe 1043-1046, coefficiente
0,065 a frame, righe 1339-1340) e la posizione dello snodo (`knob`, 0,075 a
frame, riga 1341). L'animazione dipende quindi dal numero di frame, non dal
tempo, ed è mescolata con il calcolo del percorso.

**Qui.** I percorsi arrivano già calcolati da etl-layout/etl-store (Fase 2.1) e
non si ricalcolano. Quando `route.pts` cambia tra due render per un motivo
diverso da un trascinamento:

- stesso numero di punti → ogni punto si interpola linearmente nel tempo;
- numero di punti diverso → dissolvenza incrociata tra il vecchio e il nuovo
  tracciato, senza deformare la geometria.

Durata 380 ms (la stessa di `.world.easing`, riga 136), andamento lineare nel
tempo: rende le interpolazioni esatte e verificabili (a metà, la media; alla
fine, il percorso nuovo). Nessuna transizione per un cavo nuovo, per un
percorso identico (tolleranza 0,01 px), durante un gesto di trascinamento (già
continuo) né con `prefers-reduced-motion`.

## 2. Dissolvenza incrociata: opacità lineari che sommano a 1

Il prototipo non ha dissolvenze tra tracciati. Scelta: opacità del vecchio
`1 − t` e del nuovo `t`, quindi la somma è sempre 1 (a metà: 0,5 + 0,5).
Il vecchio tracciato è un elemento separato (`.ec-link-ghost`) che scompare a
fine transizione; il nuovo è quello di sempre.

## 3. Retarget a metà transizione

Se il percorso cambia di nuovo mentre una transizione è in corso, la nuova
parte da ciò che si vede in quel momento (i punti interpolati), non dal
vecchio percorso. Se stava dissolvendo, il vecchio tracciato è quello
precedente. Il prototipo non ha il caso.

## 4. Flusso con movimento ridotto

Il prototipo ignora `prefers-reduced-motion`. Qui, con la preferenza attiva,
il flusso è lo stesso tubo, fermo a metà cavo (`sb = len / 2`), uguale a ogni
aggiornamento; le fette vuote restano all'opacità di riposo 0,85; nessun ciclo
di animazione parte.

## 5. I capi del cavo durante una dissolvenza

I due punti d'estremità (r = 2,6) seguono l'interpolazione nel caso «stesso
numero di punti»; in una dissolvenza compaiono subito nella posizione nuova
(con due tracciati diversi non c'è una corrispondenza tra i capi).

## 6. Attesa delle fette vuote pilotata dal ciclo condiviso

Nel prototipo è un'animazione CSS (`@keyframes waiting`). Qui l'opacità è la
stessa funzione, calcolata da `waitingOpacity(t)` e applicata dal ciclo
condiviso, per avere un solo ciclo e poterla verificare (e fermare con la
scheda nascosta). La fase parte dall'istante in cui la fetta compare, come
l'animazione CSS del prototipo parte dalla creazione dell'elemento; è
conservata quando React ri-renderizza il nodo.

## 7. Colore del flusso nel tema scuro

Il prototipo ha solo il tema chiaro (`rgba(108,99,255,0.6)`, riga 1517). Nel
tema scuro il flusso è `rgba(168,163,255,0.9)` (token `--ec-flow`), con
contrasto ≥ 3:1 sul canvas, verificato da `tokens.test.ts`.

## 8. Pannelli in alto e in basso: tolgono altezza al canvas (Fase 6a.1)

**Prototipo.** Aprendo un pannello su un bordo orizzontale il canvas conserva
la propria altezza e a crescere è l'area di lavoro (`fitWorkspace`, righe
4819-4824: altezza del canvas più quella dei pannelli sopra e sotto).
Funziona perché la pagina del prototipo scorre. `compensate` (righe
4815-4818) agisce solo sul bordo sinistro: con un pannello in alto il canvas
scende insieme al pannello e i nodi con lui.

**Qui.** Il contenitore dello spazio di lavoro ha l'altezza della finestra e
non scorre con la pagina: con la regola del prototipo l'area di lavoro supera il
contenitore e il pannello in basso resta tagliato (misurato a 1440 × 900: area
di lavoro 960 px in un contenitore di 738, pannello da y = 884 a 1106 in una
finestra di 900), mentre con il pannello in alto il canvas scende e minimappa e
controlli di zoom escono dall'area visibile. Ora i pannelli orizzontali
sottraggono altezza al canvas, come i laterali sottraggono larghezza:

- l'area di lavoro mantiene l'altezza del suo contenitore e il canvas (riga
  centrale della griglia) si restringe;
- la vista segue la regola del § 9 (spinta senza sovrapposizioni), non una
  compensazione;
- il canvas ha un'altezza minima (`MIN_CANVAS_HEIGHT` in `panels/layout.ts`):
  sotto quella soglia scorre il contenitore dello spazio di lavoro;
- due pannelli come schede in alto o in basso usano l'altezza maggiore dei due.

## 9. Pannelli: spinta senza sovrapposizioni, un pannello alla volta (Fase 6a.2)

**Prototipo.** Aprendo o chiudendo un pannello a sinistra la vista si
compensa (`compensate`, righe 4815-4818: `view.x` ± l'ingombro del pannello)
così i nodi restano fermi sullo schermo; sugli altri bordi non si fa nulla,
perché la pagina scorre. Cassetta e Inspector possono essere aperti insieme
(uno per bordo); solo sullo stesso bordo si escludono (righe 4826-4833).

**Qui.** Il prodotto ha un canvas a tutta altezza e un pannello non deve mai
coprire un nodo. La regola è un'altra:

- un pannello aperto occupa spazio e riduce l'area del canvas, senza mai
  sovrapporsi;
- le posizioni dei nodi nel mondo non cambiano mai (Libero e Organizzato) e lo
  zoom nemmeno: cambia solo la vista;
- nessuna compensazione: i nodi si spostano con il bordo del canvas (a sinistra
  e in alto con il bordo che avanza; a destra e in basso restano dove sono
  rispetto all'origine);
- dopo ogni cambio (apertura, chiusura, scheda, tacca su un altro bordo,
  ridimensionamento) `keepVisible` (`panels/layout.ts`) riporta dentro i nodi
  che erano interamente visibili, con lo scorrimento minimo; se l'insieme non
  entra si allinea al bordo di partenza (sinistra, alto) e il resto resta
  raggiungibile con scorrimento e minimappa;
- il margine di sicurezza dell'area visibile esclude lo spazio dei widget in
  sovrimpressione (`overlayLayout`), anche per «Adatta»;
- un solo pannello è aperto alla volta, su qualunque bordo (il comando
  `setPanel` chiude l'altro nello stesso aggiornamento); un caricamento con
  entrambi aperti lascia aperta la cassetta;
- l'Inspector si apre solo al clic su un nodo (non alla pressione, né durante
  un trascinamento, un riquadro, una selezione multipla o dopo un rilascio
  dalla cassetta); se ha sostituito la cassetta, la deselezione la riapre, e
  qualunque azione esplicita sui pannelli azzera questa memoria.

## 10. Minimappa e widget: posizione decisa da `overlayLayout`

Il prototipo ha la minimappa in basso a sinistra e i controlli di zoom in
basso a destra (righe 152-161, 138-150) con posizioni scritte nel CSS. Qui le
posizioni le decide `panels/overlayLayout.ts` a partire dal bordo del pannello
aperto e dalla misura dell'area: con il pannello in basso la minimappa va in
alto a sinistra; se l'area è troppo piccola passa all'angolo opposto e poi
diventa un pulsante compatto che si espande al clic. Due widget non si
sovrappongono mai.

## 11. Barra dei controlli

Il prototipo ha una barra con il pulsante «Funzionalità» e «Reimposta». Qui la
barra (`panels/ControlBar.tsx`) ha Libero/Organizzato, Riordina, Annulla,
Ripristina e Svuota; «Funzionalità» e «Reimposta» non ci sono. Svuota è il
comando `clearAll` (un passo di cronologia, nel registro; la libreria non si
tocca) e chiede sempre conferma.

## 12. Inspector: forma e comportamenti diversi dal prototipo (Fase 6b.1)

- **Nessun elemento nativo.** Il prototipo usa `<datalist>` per le colonne e un menu
  disegnato a mano dentro il pannello (poi staccato in `body`, righe 2793-2810). Qui
  ogni scelta è un nostro componente: un campo che apre un menu in un portale
  (`#ei-portal`), con ricerca (combobox ARIA), posizionato da una funzione pura
  (`placeMenu`): 8 px dal campo, 16 px dai bordi della finestra, dal lato con più spazio
  (il prototipo preferiva sotto se ci stava), con scorrimento interno.
- **Colonne multiple e valori.** Cambiare le colonne di una riga non azzera i valori
  scelti (nel prototipo sì, righe 3014-3021): vedi `etl-core/NOTE_DIVERGENZE.md` § 5.
- **Tabelle di un join.** Il prototipo scrive subito nei parametri la tabella scelta di
  default (righe 3753-3756); qui il valore proposto si mostra senza scriverlo, e si
  salva solo quando l'utente sceglie.
- **Join: tipo.** Il campo «Tipo di join» non compare finché non arrivano le
  condizioni (Fase 6b.2): in questa fase filtro e join mostrano solo la nota.
- **Nome in linea.** Il prototipo usa un elemento `contenteditable`; qui è un campo di
  testo. Un nome vuoto non si applica e al ritorno del focus torna quello di prima.
- **Passaggi.** Lo sgancio è anche un pulsante nell'elenco dell'Inspector (nel
  prototipo solo dal pannello espanso); il riordino è anche da tastiera (Alt+↑/↓).
- **Pannello espanso.** Resta una finestra di dialogo con sfondo, ma si chiude con Esc,
  ha il focus dentro e lo restituisce al pulsante di espansione del nodo.
- **Pulsante ×.** Elimina solo quel nodo (come nel prototipo, righe 4585-4590), con la
  stessa anteprima e conferma di `nodesRemovedBy` della Fase 5; compare anche al focus,
  e l'area che si preme è di almeno 32 px.
- **Tetto d'altezza.** I pannelli sui bordi alto e basso non superano il 45%
  dell'altezza dello spazio di lavoro; il contenuto scorre dentro.
```

### `src/etl-canvas/Node.tsx`

93 righe

```tsx
import { memo } from "react";
import { Icon } from "./icons";
import { useMotion } from "./motion";
import type { NodeView } from "./model";
import { copy } from "./inspector/copy";
import { ExpandIcon, XIcon } from "./inspector/icons";

/** Le quattro porte da cui si tira un collegamento (prototipo, righe 168-181). */
const PORT_SIDES = ["t", "b", "l", "r"] as const;

/** Un nodo: quadrato con icone, etichetta, indicatore ambra (prototipo `createCardEl`, righe 1000-1015). */
export const Node = memo(function Node(props: {
  node: NodeView;
  /** Pulsante × (visibile al passaggio e al focus). */
  onDelete?: (id: string) => void;
  /** Pulsante di espansione dei box combinati. */
  onExpand?: (id: string) => void;
}) {
  const { node } = props;
  const { card } = node;
  const motion = useMotion();
  return (
    <div
      className={node.className}
      data-node-id={node.id}
      data-kind={card.kind}
      data-family={node.family}
      style={{ left: card.x, top: card.y }}
    >
      <div
        className={node.iconClass}
        data-n={node.partial ? String(node.slices.length) : undefined}
      >
        {node.partial
          ? node.slices.map((s, i) => (
              <div key={i} className={`ec-slice ${s.full ? "ec-slice-full" : "ec-slice-empty"}`}>
                <Icon
                  id={s.full ? "dataset" : "empty"}
                  {...(s.full
                    ? {}
                    : {
                        svgRef: (el: SVGSVGElement | null) =>
                          motion?.registerSlice(`${node.id}:${i}`, el),
                      })}
                />
              </div>
            ))
          : node.icons.map((id, i) => <Icon key={i} id={id} />)}
        {node.warn ? (
          <span className="ec-state-dot" role="img" aria-label={node.warn} title={node.warn} />
        ) : null}
      </div>
      <div className="ec-label">{card.name}</div>
      {props.onDelete ? (
        <button
          type="button"
          className="ec-node-btn ec-del-btn"
          aria-label={copy.nodeDelete}
          title={copy.nodeDelete}
          onClick={(e) => {
            e.stopPropagation();
            props.onDelete?.(node.id);
          }}
        >
          <XIcon />
        </button>
      ) : null}
      {props.onExpand && card.kind === "op" && card.components.length > 1 ? (
        <button
          type="button"
          className="ec-node-btn ec-expand-btn"
          aria-label={copy.nodeExpand}
          title={copy.nodeExpand}
          onClick={(e) => {
            e.stopPropagation();
            props.onExpand?.(node.id);
          }}
        >
          <ExpandIcon />
        </button>
      ) : null}
      {PORT_SIDES.map((side) => (
        <span
          key={side}
          className={`ec-port ec-port-${side}`}
          data-port={side}
          aria-hidden="true"
        />
      ))}
    </div>
  );
});
```

