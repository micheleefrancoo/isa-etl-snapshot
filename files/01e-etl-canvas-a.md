# 01e-etl-canvas-a.md

File in questo blocco:

- `src/etl-canvas/EtlCanvas.tsx`
- `src/etl-canvas/Links.tsx`
- `src/etl-canvas/Minimap.tsx`
- `src/etl-canvas/NOTE_DIVERGENZE.md`
- `src/etl-canvas/Node.tsx`
- `src/etl-canvas/README.md`
- `src/etl-canvas/__tests__/drop.test.ts`

---

### `src/etl-canvas/EtlCanvas.tsx`

431 righe

```tsx
import { useEffect, useLayoutEffect, useMemo, useRef, useState, useSyncExternalStore } from "react";
import type { PointerEvent as ReactPointerEvent } from "react";
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
import type { LoopEnv } from "./loop";
import { MotionContext } from "./motion";
import "./canvas.css";
import { Links } from "./Links";
import { Minimap } from "./Minimap";
import { nodeView } from "./model";
import { Node } from "./Node";
import { WHEEL_ZOOM_RATE } from "./view";

const PORT_SIDES: readonly string[] = ["r", "b", "l", "t"];

/** Classifica il punto in cui è iniziato il gesto (unico punto in cui si legge il DOM). */
function classify(t: EventTarget | null): DownTarget {
  const el = t as HTMLElement | null;
  if (!el || typeof el.closest !== "function") return { kind: "ignore" };
  if (el.closest(".ec-zoom, .ec-minimap, .ec-confirm")) return { kind: "ignore" };
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
  /** Controller dei gesti condiviso con chi sta fuori dal canvas (la cassetta); se manca, ne nasce uno. */
  readonly controller?: InteractionController | undefined;
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
    engine.start(env ?? browserEnv());
    return () => engine.stop();
  }, [engine, env]);

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
        store.dispatch({ type: "setView", payload: { x: v.x - e.deltaX, y: v.y - e.deltaY } });
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
        <div ref={stageRef} className={stageClass} onPointerDown={onPointerDown}>
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
              <Node key={n.id} node={n} />
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
          {ui.hint ? (
            <div className="ec-hint" role="status">
              {ui.hint}
            </div>
          ) : null}
          {ui.confirm ? (
            <div
              className="ec-confirm"
              role="alertdialog"
              aria-labelledby="ec-confirm-title"
              aria-describedby="ec-confirm-text"
              data-testid="ec-confirm"
              style={confirmPosition(graph.cards[ui.confirm.ids[0] ?? ""], view, size)}
              onPointerDown={(e) => e.stopPropagation()}
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
                  type="button"
                  className="ec-confirm-cancel"
                  onClick={controller.cancelConfirm}
                >
                  Annulla
                </button>
                <button type="button" className="ec-confirm-ok" onClick={controller.confirmDelete}>
                  Elimina
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
          <Minimap
            cards={cards}
            view={view}
            size={size}
            onView={(v) => store.dispatch({ type: "setView", payload: v })}
          />
          <div className="ec-zoom" data-testid="ec-zoom" onPointerDown={(e) => e.stopPropagation()}>
            <button type="button" aria-label="Riduci" onClick={() => zoomOut(store, size)}>
              −
            </button>
            <button type="button" aria-label="Zoom al 100%" onClick={() => zoomReset(store, size)}>
              {Math.round(view.zoom * 100)}%
            </button>
            <button type="button" aria-label="Ingrandisci" onClick={() => zoomIn(store, size)}>
              +
            </button>
            <button type="button" className="ec-fit" onClick={() => fit(store, size)}>
              Adatta
            </button>
          </div>
        </div>
      </div>
    </MotionContext.Provider>
  );
}

const CONFIRM_W = 246;
const CONFIRM_H = 168;

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
export function EtlCanvas(props: { store: EtlStore; controller?: InteractionController }) {
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
      style={{ position: "relative", width: "100%", height: "100%", minHeight: 520 }}
    >
      {isClient && size ? (
        <CanvasSurface store={props.store} size={size} controller={props.controller} />
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

72 righe

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
  onView: (view: View) => void;
}) {
  const { cards, view, size, onView } = props;
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
      aria-hidden="true"
      onPointerDown={onPointerDown}
    >
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

70 righe

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
```

### `src/etl-canvas/Node.tsx`

57 righe

```tsx
import { memo } from "react";
import { Icon } from "./icons";
import { useMotion } from "./motion";
import type { NodeView } from "./model";

/** Le quattro porte da cui si tira un collegamento (prototipo, righe 168-181). */
const PORT_SIDES = ["t", "b", "l", "r"] as const;

/** Un nodo: quadrato con icone, etichetta, indicatore ambra (prototipo `createCardEl`, righe 1000-1015). */
export const Node = memo(function Node(props: { node: NodeView }) {
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

### `src/etl-canvas/README.md`

226 righe

```md
# etl-canvas — Fasi 4a, 4b, 5 e 6a: il canvas, le sue animazioni, i gesti e i pannelli

Resa visiva del canvas ETL in React, fedele al prototipo
`docs/prototype/isa-fusion-prototype.html`. Solo **vista**: token, nodi,
cavi, pan, zoom, controlli di zoom, minimappa (4a) e animazioni: flusso nei
cavi, attesa delle fette vuote, transizione dei percorsi (4b), e i gesti (Fase 5): trascinamento, fusione,
collegamento, porte, selezione, tastiera, e i pannelli (Fase 6a): cassetta degli
strumenti e guscio dell'Inspector, agganciabili ai quattro bordi. Il contenuto
dell'Inspector (Fase 6b) non c'è ancora.

Importa da `etl-core`, `etl-layout` ed `etl-store`; nessuno di questi importa
da qui. Non usa il vecchio stato (`src/lib/etl-workflow.tsx`): legge e
scrive solo attraverso `etl-store`.

## Moduli

```
tokens.css       token --ec-* (livello 3) con ambito .etl-canvas: leggono solo i token semantici --isa-* dei temi (src/theme/); tema predefinito chiaro = prototipo, scuro progettato
canvas.css       aspetto di nodi, cavi, controlli, minimappa; importa tokens.css
EtlCanvas.tsx    EtlCanvas (solo browser, misura l'area) e CanvasSurface (la resa, rendibile anche in Node)
Node.tsx         un nodo: chip, icone, fette, etichetta, indicatore ambra
Links.tsx        i cavi da store.getRoutes()
Minimap.tsx      minimappa e clic/trascinamento per spostare la vista
icons.tsx        icone del catalogo di etl-core
model.ts         grafo → classi e fette di ogni nodo (puro)
view.ts          zoom, Adatta, minimappa (puro, numeri del prototipo)
actions.ts       fit/zoomIn/zoomOut/zoomReset/zoomAtPoint: applicano la vista con setView
seed.ts          scena iniziale del prototipo (solo sviluppo)
contrast.ts      contrasto WCAG tra i token (test e report)
flow.ts          4b, puro: finestra e contorno del tubo del flusso, opacità dell'attesa
transitions.ts   4b, puro: interpolazione dei punti, dissolvenza incrociata, piano della transizione
loop.ts          4b: UN ciclo requestAnimationFrame condiviso (+ ambiente del browser)
engine.ts        4b: livello sottile che applica lo stato visivo agli attributi SVG
motion.tsx       4b: contesto con cui cavi e nodi registrano i propri elementi
interaction.ts   5: controller dei gesti (puntatore, tastiera) → comandi di etl-store. Puro, senza DOM
drop.ts          5: handleCanvasDrop / previewCanvasDrop, il rilascio di un nuovo elemento (per la Fase 6)
```

## Uso

```tsx
const store = usePersistentEtlStore(solutionId); // etl-store/react
<EtlCanvas store={store} />;
```

Il contenitore deve avere un'altezza (minimo 520 px). Nella rotta
`solutions.$solutionId.etl.tsx` il nuovo canvas è quello **predefinito**; il
vecchio (codice invariato) si raggiunge solo con `?canvas=v1`. Solo in
sviluppo, `?seed=prototype` carica la scena del prototipo se il canvas è
vuoto (in produzione `seed` è ignorato) e `window.__etlStore` espone lo
store alla console e allo script delle schermate.

## Fase 5 — Gesti

Questo livello **non contiene logica di dominio**: `interaction.ts` traduce
eventi già classificati (coordinate dell'area, bersaglio del gesto) in
chiamate a funzioni che esistevano. `EtlCanvas.tsx` è l'unico punto che
legge il DOM (`classify`) e registra gli ascoltatori (Pointer Events su
`window` durante il gesto, `keydown` sul documento). Il controller si prova
con eventi simulati (`__tests__/interaction.test.ts`, `keyboard.test.ts`,
`drop.test.ts`); `scripts/e2e-fase5.mjs` prova gli stessi gesti con
Pointer Events veri in Chromium.

| Gesto                                                                   | Prototipo (righe)                                                                                 | Funzione chiamata oggi                                                                                                                                       |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Trascinare un nodo                                                      | `stage` `pointerdown` 1959-2107 (`onMove` 1978-2056, `onUp` 2058-2105)                            | `store.beginGesture` / `updateGesture` / `commitGesture` / `cancelGesture` (Fase 3); il rilascio è `dropAt` di etl-store (un passo di cronologia)            |
| Esito sopra un altro nodo (fusione, collegamento, inverso, spostamento) | 2034-2056                                                                                         | etl-core `relation` (1917-1936); `performMerge`/`connect` al rilascio: `dropAt` → `mergeBoxes`/`connect`; lo spostamento è `displace` dentro `updateGesture` |
| Trascinare su un cavo (inserimento)                                     | 2015-2030, 2092                                                                                   | etl-layout `linkAt`; etl-core `insertable`; al rilascio `dropAt` → `insertOnLink`                                                                            |
| Tirare un cavo da una porta                                             | 3974-4024                                                                                         | etl-layout `nodePorts` (punto di partenza); comando `connect`                                                                                                |
| Rilasciare dalla cassetta o dalla libreria                              | `paletteEl` `pointerdown` 4939-5067                                                               | comando `addNode` (con `paletteRelation` per l'anteprima): `handleCanvasDrop` / `previewCanvasDrop` in `drop.ts`                                             |
| Click, Maiusc+click                                                     | 2063-2071, `selectCard` 2713, `toggleInSelection` 2693                                            | comandi `select` + `inspect`                                                                                                                                 |
| Riquadro di selezione                                                   | 4058-4089                                                                                         | etl-layout `nodeRect`; comandi `select` + `inspect`                                                                                                          |
| Trascinare il gruppo                                                    | 1969-2001, 2076-2081                                                                              | gesto di etl-store con più `ids` (`dropAt` ricompone i sovrapposti)                                                                                          |
| Click sul vuoto                                                         | 4584                                                                                              | comandi `select` + `inspect` (vuoti)                                                                                                                         |
| Click su un cavo                                                        | `linkHits` `click` 4575-4580, `deleteLink` 4565                                                   | comando `deleteLink` (senza conferma, come nel prototipo; un trascinamento che parte dal cavo non elimina)                                                   |
| Canc / Backspace                                                        | `deleteMany` 4538, `deleteCard` 4558, `commitDelete` 4480, `nodesRemovedBy` 4432; tasto 4630-4634 | `nodesRemovedBy` (anteprima) e comando `deleteNodes`                                                                                                         |
| Frecce (2 px, Maiusc = `GRID`)                                          | 4619-4627; tasto 4642-4644                                                                        | comando `moveNodes` (tenere premuto = un solo passo, Fase 3.1)                                                                                               |
| Cmd/Ctrl+D                                                              | 4594-4618, 4645                                                                                   | comando `duplicate`                                                                                                                                          |
| Cmd/Ctrl+A, Esc                                                         | 4646; Esc 4636                                                                                    | comandi `select` + `inspect`                                                                                                                                 |
| Cmd/Ctrl+Z, +Maiusc+Z, Ctrl+Y                                           | `undo` 4389, `redo` 4395; tasti 4650-4653                                                         | `store.undo()` / `store.redo()`                                                                                                                              |
| Pan: spazio o tasto centrale; zoom Cmd/Ctrl+rotella                     | 4036-4054, 4166-4180, 4092-4102                                                                   | già collegati nella 4a (`setView`); la 5 verifica che convivano con la selezione                                                                             |

**Esiti mostrati durante il trascinamento** (`InteractionUi.drop`), ciascuno
con colore e stile di contorno propri, tutti da token semantici
(`--isa-drop-*`): `merge` (anello pieno), `link` (anello pieno), `link-reverse`
(tratteggio), `displace` (punteggiato, col motivo di etl-core nel
suggerimento), `reject` (continuo). Il cavo in cui si inserirebbe la
lavorazione prende `ec-link-hot`; i nodi che l'eliminazione porterebbe via
(`nodesRemovedBy`: scelti + output a valle) prendono `ec-doomed` mentre la
conferma è aperta.

**Scelte e differenze dal prototipo**

- _Soglia di avvio_: `DRAG_THRESHOLD_PX` = 5 px, in `etl-layout/constants.ts`
  (prototipo, riga 1982). Sotto la soglia è un click. Il riquadro di
  selezione usa 4 px (prototipo, riga 4064).
- _Inserimento su cavo_: il nodo in mano è un ostacolo e fa scansare i cavi;
  il puntatore resta quindi spesso lontano dal cavo disegnato. Il test sul
  cavo si fa sui percorsi attuali **e** su quelli di prima del gesto
  (`startRoutes`).
- _Selezione_: `select` e `inspect` insieme; i pannelli (`setPanel`) non si
  toccano (l'apertura dell'Inspector è della Fase 6). Il riquadro non
  scrive nello store mentre si trascina (il registro delle attività si
  riempirebbe): lo mostra con `InteractionUi.marquee` e seleziona al rilascio.
- _Esc_ durante un trascinamento lo annulla (`cancelGesture`); poi chiude la
  conferma; poi deseleziona.
- _Scorciatoie di annulla/ripristina_ non agiscono con il fuoco in un campo di
  testo (il prototipo le applicava sempre): lì vale l'annulla del campo.
- _Non collegati_ (fuori dall'elenco della fase): i pulsanti di eliminazione ed
  espansione sul nodo (prototipo 4584, Fase 6).

## Fase 6a — Pannelli e cassetta

Codice in `panels/` (`Dock.tsx`, `Toolbox.tsx`, `InspectorShell.tsx`,
`EtlWorkspace.tsx`; logica pura in `layout.ts`, `actions.ts`, `csv.ts`,
`families.ts`). Lo stato (lato, aperto/chiuso, scheda attiva) è in `etl-store`
(`panels`) e si salva con il resto. Prototipo:
`docs/prototype/isa-fusion-prototype.html`.

| Elemento                                                                            | Prototipo (righe)                                                                                  | Qui                                                                                                                        |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Struttura, griglia dei quattro bordi, colonna o fascia                              | CSS 211-346 (`#dock-*` 216-219), HTML 808-838                                                      | `Dock.tsx`, `panels.css`, `layout.ts`                                                                                      |
| Apertura e chiusura, tacca visibile solo da chiuso                                  | `setPanelOpen` 4835-4860, `layoutNotches` 4847-4858, CSS `.notch` 272-295                          | comando `setPanel`; `layout.ts` (la tacca si nasconde se il pannello è aperto o raggiungibile da una scheda)               |
| Trascinare la tacca su un altro bordo (soglia, un clic apre)                        | 4881-4905 (soglia 4884, «un click apre soltanto» 4899)                                             | `Dock.tsx` (gesto della tacca), `actions.ts` (`setSide`: chiude, sposta, riapre)                                           |
| Due pannelli sullo stesso bordo: schede, contenuto sul posto                        | `dock-tabs` CSS 306-327, `switchTab` 4787-4806; nessuna animazione di apertura                     | `Dock.tsx`, `layout.ts` (`grouped`, scheda attiva); la larghezza non cambia al cambio di scheda                            |
| Compensazione della vista sui bordi verticali                                       | 4780-4784, 4818-4821                                                                               | `layout.ts` (spostamento della vista) → comando di vista di etl-store; su bordo orizzontale cresce l'area di lavoro        |
| Sezioni della cassetta (Dataset, Filtra e ordina, Trasforma, Merge e union, Output) | `buildPalette` 4727-4752, `SECTIONS`, `palItem` 4727-4733                                          | `Toolbox.tsx`, `families.ts`: sezioni e voci derivano da `etl-core/catalog/operations.ts`, non da una lista a mano         |
| Sezione comprimibile                                                                | `.tb-sec-head` 246-250, clic 4914-4919 (senza ridisegno)                                           | stato locale di `Toolbox.tsx` per sezione                                                                                  |
| Caricamento CSV e deduzione dei tipi                                                | `parseCSV` 4672-4725 (tipo: integer, numerico, data, stringa: righe 4696-4699), `change` 4921-4937 | `panels/csv.ts` → `store.loadCsv` (`parseCSV` di etl-core, comando `loadDataset`); voce trascinabile nella sezione Dataset |
| Trascinare una voce dalla cassetta al canvas                                        | `paletteEl` `pointerdown` 4939-5067, `paletteRelation` 4711                                        | `EtlWorkspace.tsx` → `handleCanvasDrop` / `previewCanvasDrop` (`drop.ts`)                                                  |
| Inspector (guscio): si apre con la selezione, si chiude senza                       | `selectCard` / `deselect` 2692-2727, `openInspector` 2692                                          | `InspectorShell.tsx` (solo il nome del nodo), `actions.ts` (segue la selezione)                                            |

Non portati: il pulsante «Funzionalità» (tutte le funzionalità sono sempre
attive, Fase T). Prova nel browser: `scripts/e2e-fase6a.mjs` (36 prove,
schermate in `docs/visual/fase6a/`).

## Rendering lato server

`EtlCanvas` produce, sul server e nel primo rendering di idratazione, lo
stesso contenitore vuoto (`useSyncExternalStore` con snapshot server
`false`); dopo l'idratazione misura l'area con un `ResizeObserver` e monta
`CanvasSurface`. Nessun accesso a `window`/`document` durante il rendering.

## Carattere e temi

- Manrope (`@fontsource-variable/manrope`) è il carattere di tutta l'app,
  caricato da `src/styles.css`; `--ec-font` deriva da `--font-sans`. Nessuna
  richiesta a server esterni: i file sono serviti dall'app, e il browser
  scarica un sottoinsieme solo se il testo lo usa.
- Raggi dello stage e della minimappa derivati da `--radius` del tema (nel
  tema predefinito, stessi 20 e 14 px del prototipo). Gli altri token restano del canvas: vedi il
  report della fase di fondazione per i ruoli che l'app definisce con valori
  diversi.
- Modo e tema arrivano da `<html>` (classe `.dark` e `data-theme`, vedi
  `src/theme/README.md`); `tokens.css` non ha blocchi per modo: i token
  semantici variano da soli. Le famiglie di operazioni (`data-family` sul
  nodo) leggono `--isa-op-*`; nel tema predefinito coincidono tutte con la
  tinta unica del prototipo.
- Tema chiaro = valori del prototipo, con la riga di provenienza accanto a
  ogni token (un test verifica che i colori chiari compaiano nel prototipo).
  Tema scuro = derivato dal tema scuro dell'app, con la stessa tinta
  d'accento; i contrasti sono verificati da `__tests__/tokens.test.ts`
  (testo ≥ 4,5:1, elementi non testuali ≥ 3:1).

## Vista

`store.getState().view` (`x`, `y`, `zoom`) è l'unica fonte: pan, zoom e
minimappa la modificano con `setView`, che non entra né nel registro né
nella cronologia. Pan: barra spaziatrice + trascinamento, tasto centrale,
rotella (come nel prototipo, senza modificatore sposta la vista);
Cmd/Ctrl + rotella = zoom attorno al puntatore. Zoom tra 0,35 e 2; Adatta
usa margine 48 e zoom al più 1,25 (come il prototipo), quindi con una scena
più grande di ciò che lo zoom minimo può contenere si ferma a 0,35.

## Verifica visiva

`node scripts/visual-fase4.mjs` avvia `vite dev`, apre prototipo e nuovo
canvas a 1440 × 900 e salva in `docs/visual/fase4/`: `prototipo.png`,
`v2-chiaro.png`, `v2-scuro.png`, un ritaglio per tipo di nodo e tema, e
`misure.json` (posizioni, colori, misure lette dal DOM).

## Animazioni (Fase 4b)

Le animazioni sono un effetto visivo sopra percorsi **già calcolati**: il
motore legge `route.pts` e `route.d` di `getRoutes()` e non li scrive né li
ricalcola. Non chiama mai `settleLinks` né altre funzioni di etl-layout che
instradino (`__tests__/no-reroute.test.ts` lo verifica con uno spy); per
disegnare i punti interpolati usa solo `roundedPath`, che arrotonda punti
dati. Nessuna modifica a etl-core, etl-layout, etl-store.

### Routine del prototipo portate

| Cosa                                                                                                                                                        | Prototipo (righe)      | Qui                              |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | -------------------------------- |
| Costanti del tubo: `BASE_W` 2,1 · `SPEED` 0,16 px/ms · `BALL` 4,4 · `FRONT` 7,5 · `BACK` 19                                                                 | 1415-1418              | `flow.ts`                        |
| Profilo del tubo, gaussiana asimmetrica: `sg = u ≥ 0 ? FRONT : BACK`, `exp(-u²/sg²)`                                                                        | 1419-1422              | `tubeProfile`                    |
| `smooth01(x) = x²(3-2x)` sul tratto `min(s, len-s)/22` (il tubo emerge dalla porta e vi rientra)                                                            | 1478, 1510             | `smooth01`, `EDGE_FADE`          |
| Ciclo `len + BACK·3,2`; punto che avanza `sb = ((now-t0)·SPEED) % ciclo − BACK·1,1`; tratto `[max(0, sb−BACK·3), min(len, sb+FRONT·3,2)]`; niente se < 2 px | 1488-1497              | `flowWindow`                     |
| Contorno: un campione ogni 1,6 px (min. 10), normale alla tangente, mezzo spessore `BASE_W/2 + BALL·profilo·bordo`, riempimento `rgba(108,99,255,0.6)`      | 1499-1517              | `tubeOutline`, token `--ec-flow` |
| Flusso solo sui collegamenti attivi (`linkLive`)                                                                                                            | 1474-1477, 1489        | `linkLive` di etl-store          |
| Flusso fermo durante lo spostamento di un nodo (`flowPaused`)                                                                                               | 1423, 1485, 1985, 2072 | `engine.setGesturing`            |
| `t0` del cavo = istante del primo disegno                                                                                                                   | 1313                   | `t0` di ogni cavo nel motore     |
| Attesa: `animation: waiting 1.9s ease-in-out infinite`; `0%,100% {opacity:.45}`, `50% {opacity:.95}`; a riposo `.85`                                        | 669-670                | `waitingOpacity`                 |
| Ciclo di disegno: un solo `requestAnimationFrame` per tutto il canvas                                                                                       | 1480-1521              | `loop.ts`                        |
| Durata `.38s` della transizione della vista (`.world.easing`)                                                                                               | 136                    | `TRANSITION_MS`                  |

Valori derivati da un calcolo, non copiati a occhio: il ciclo (`len +
BACK·3,2`), la posizione (`sb`), gli estremi del tratto e il numero di
campioni si calcolano con le formule sopra; l'opacità dell'attesa è la
funzione `cubic-bezier(.42,0,.58,1)` di CSS applicata a ciascuna metà del
periodo. Le differenze e le scelte nuove sono in `NOTE_DIVERGENZE.md`.

### Risparmio energetico

- Un solo ciclo condiviso (`loop.ts`), non un timer per cavo.
- Si ferma da solo quando nessun compito ha nulla da animare (nessun cavo
  attivo, nessuna fetta vuota, nessuna transizione in corso) e riparte
  quando arriva lavoro nuovo (`wake`).
- Si ferma con `document.hidden` e riparte con `visibilitychange`.
- Con `prefers-reduced-motion: reduce` il ciclo non parte mai: il flusso è un
  tubo fermo a metà cavo, le transizioni sono istantanee, l'attesa resta a
  riposo (0,85). Se la preferenza cambia a canvas aperto, il ciclo si ferma o
  riparte.
- Nel rendering non si toccano `requestAnimationFrame`, `matchMedia`,
  `document`: il motore si avvia in un effetto, con l'ambiente del browser.
```

### `src/etl-canvas/__tests__/drop.test.ts`

84 righe

```ts
import { describe, expect, it } from "vitest";
import { nodeCenter } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
import { handleCanvasDrop, previewCanvasDrop } from "../drop";
import { storeWith } from "./helpers";

const cardCount = (s: EtlStore) => Object.keys(s.getState().graph.cards).length;

describe("handleCanvasDrop (rilascio dalla cassetta, per la Fase 6)", () => {
  it("nel vuoto crea il nodo, un solo passo di cronologia", () => {
    const store = storeWith();
    const n = cardCount(store);
    const r = handleCanvasDrop(store, { component: "filter" }, { x: 1000, y: 700 });
    expect(r.ok).toBe(true);
    expect(cardCount(store)).toBe(n + 1);
    expect(store.historySize().past).toBe(1);
    expect(
      previewCanvasDrop(store, { component: "filter" }, { x: 1500, y: 900 }).outcome,
    ).toBeNull();
  });

  it("una lavorazione su una lavorazione si fonde", () => {
    const store = storeWith();
    const p = nodeCenter(store.getState().graph.cards["op-sort"]!);
    expect(previewCanvasDrop(store, { component: "filter" }, p)).toMatchObject({
      outcome: "merge",
      nodeId: "op-sort",
    });
    const n = cardCount(store);
    expect(handleCanvasDrop(store, { component: "filter" }, p).ok).toBe(true);
    expect(cardCount(store)).toBe(n); // assorbita: non compare da sola
    expect(store.getState().graph.cards["op-sort"]!.components).toHaveLength(2);
    expect(store.historySize().past).toBe(1);
  });

  it("un dataset su una lavorazione si collega; una lavorazione su un dataset si collega al contrario", () => {
    const a = storeWith();
    const onOp = nodeCenter(a.getState().graph.cards["op-join"]!);
    expect(previewCanvasDrop(a, { component: "dataset" }, onOp).outcome).toBe("link");
    handleCanvasDrop(a, { component: "dataset" }, onOp);
    expect(a.getState().graph.links.some((l) => l.to === "op-join")).toBe(true);

    const b = storeWith();
    const onDs = nodeCenter(b.getState().graph.cards["ds1"]!);
    expect(previewCanvasDrop(b, { component: "sort" }, onDs).outcome).toBe("link-reverse");
    handleCanvasDrop(b, { component: "sort" }, onDs);
    expect(b.getState().graph.links.some((l) => l.from === "ds1")).toBe(true);
  });

  it("una lavorazione su un cavo dataset→lavorazione vi si inserisce; un dataset no", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const key = "ds1|op-join";
    const pts = store.getRoutes()[key]!.pts;
    const p = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    expect(previewCanvasDrop(store, { component: "sort" }, p)).toMatchObject({
      outcome: "insert",
      linkKey: key,
    });
    expect(previewCanvasDrop(store, { component: "dataset" }, p).outcome).toBeNull();
    expect(handleCanvasDrop(store, { component: "sort" }, p).ok).toBe(true);
    expect(store.getState().graph.links).not.toContainEqual({ from: "ds1", to: "op-join" });
  });

  it("su un cavo lavorazione→output non si inserisce: cade nel vuoto", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const pts = store.getRoutes()["op-join|out-0"]!.pts;
    const p = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    expect(previewCanvasDrop(store, { component: "sort" }, p).outcome).toBeNull();
  });

  it("è raggiungibile dal controller, con il punto dell'area convertito in mondo", () => {
    const store = storeWith();
    store.dispatch({ type: "setView", payload: { x: 50, y: 20, zoom: 2 } });
    const c = createInteractionController(store);
    expect(c.toWorld(250, 220)).toEqual({ x: 100, y: 100 });
    const n = cardCount(store);
    expect(c.handleCanvasDrop({ component: "limit" }, c.toWorld(1200, 900)).ok).toBe(true);
    expect(cardCount(store)).toBe(n + 1);
  });
});
```

