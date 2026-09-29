# 01e-etl-canvas-a.md

File in questo blocco:

- `src/etl-canvas/EtlCanvas.tsx`
- `src/etl-canvas/Links.tsx`
- `src/etl-canvas/Minimap.tsx`
- `src/etl-canvas/NOTE_DIVERGENZE.md`
- `src/etl-canvas/Node.tsx`
- `src/etl-canvas/README.md`
- `src/etl-canvas/__tests__/engine.test.ts`
- `src/etl-canvas/__tests__/fake-env.ts`
- `src/etl-canvas/__tests__/flow.test.ts`
- `src/etl-canvas/__tests__/helpers.ts`
- `src/etl-canvas/__tests__/loop.test.ts`
- `src/etl-canvas/__tests__/no-reroute.test.ts`

---

### `src/etl-canvas/EtlCanvas.tsx`

253 righe

```tsx
import { useEffect, useLayoutEffect, useMemo, useRef, useState, useSyncExternalStore } from "react";
import type { PointerEvent as ReactPointerEvent } from "react";
import { WORLD_H, WORLD_W } from "../etl-layout";
import type { Size } from "../etl-layout";
import { linkKey } from "../etl-layout";
import { linkLive, nodeStates } from "../etl-store";
import type { EtlStore } from "../etl-store";
import { useEtlState } from "../etl-store/react";
import { fit, zoomAtPoint, zoomIn, zoomOut, zoomReset } from "./actions";
import { createMotionEngine } from "./engine";
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
  const stageRef = useRef<HTMLDivElement>(null);
  const [spaceDown, setSpaceDown] = useState(false);
  const [panning, setPanning] = useState(false);
  const spaceRef = useRef(false);

  const states = useMemo(() => nodeStates(graph), [graph]);
  // eslint-disable-next-line react-hooks/exhaustive-deps -- i percorsi dipendono da grafo e limite di snodi, che sono nelle dipendenze
  const routes = useMemo(() => store.getRoutes(), [store, graph, maxBends]);
  const cards = useMemo(() => Object.values(graph.cards), [graph]);
  const selected = useMemo(() => new Set(selection), [selection]);
  const nodes = useMemo(
    () => cards.map((c) => nodeView(c, states[c.id] ?? null, selected.has(c.id))),
    [cards, states, selected],
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
    engine.setGesturing(store.isGesturing());
    return store.subscribe(() => engine.setGesturing(store.isGesturing()));
  }, [engine, store]);

  // barra spaziatrice: navigazione temporanea (prototipo, righe 4168-4177)
  useEffect(() => {
    const down = (e: KeyboardEvent) => {
      if (e.code !== "Space" || isTyping(e.target)) return;
      spaceRef.current = true;
      setSpaceDown(true);
      e.preventDefault();
    };
    const up = (e: KeyboardEvent) => {
      if (e.code !== "Space") return;
      spaceRef.current = false;
      setSpaceDown(false);
    };
    document.addEventListener("keydown", down);
    document.addEventListener("keyup", up);
    return () => {
      document.removeEventListener("keydown", down);
      document.removeEventListener("keyup", up);
    };
  }, []);

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

  // pan: barra spaziatrice + trascinamento, o tasto centrale (righe 4036-4054)
  const onPointerDown = (e: ReactPointerEvent<HTMLDivElement>) => {
    if (!(spaceRef.current || e.button === 1)) return;
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
            <Links graph={graph} routes={routes} />
            {nodes.map((n) => (
              <Node key={n.id} node={n} />
            ))}
          </div>
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

const noopSubscribe = () => () => {};

/**
 * Il canvas. Si monta solo nel browser: sul server e nel primo rendering
 * di idratazione produce sempre lo stesso contenitore vuoto, così non ci
 * sono differenze da riconciliare. Misura l'area con un ResizeObserver.
 */
export function EtlCanvas(props: { store: EtlStore }) {
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
      {isClient && size ? <CanvasSurface store={props.store} size={size} /> : null}
    </div>
  );
}
```

### `src/etl-canvas/Links.tsx`

72 righe

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
      <path className="ec-link" d={route.d} />
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
export const Links = memo(function Links(props: { graph: Graph; routes: LinkRoutes }) {
  const { graph, routes } = props;
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

45 righe

```tsx
import { memo } from "react";
import { Icon } from "./icons";
import { useMotion } from "./motion";
import type { NodeView } from "./model";

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
    </div>
  );
});
```

### `src/etl-canvas/README.md`

136 righe

```md
# etl-canvas — Fasi 4a e 4b: il canvas visibile e le sue animazioni

Resa visiva del canvas ETL in React, fedele al prototipo
`docs/prototype/isa-fusion-prototype.html`. Solo **vista**: token, nodi,
cavi, pan, zoom, controlli di zoom, minimappa (4a) e animazioni: flusso nei
cavi, attesa delle fette vuote, transizione dei percorsi (4b).
Trascinamento, fusione, collegamento, selezione, tastiera (Fase 5), cassetta
e Inspector (Fase 6) non ci sono ancora.

Importa da `etl-core`, `etl-layout` ed `etl-store`; nessuno di questi importa
da qui. Non usa il vecchio stato (`src/lib/etl-workflow.tsx`): legge e
scrive solo attraverso `etl-store`.

## Moduli

```
tokens.css       token --ec-* con ambito .etl-canvas, alias delle primitive --isa-* di src/styles.css (valori: tema chiaro = prototipo, scuro progettato)
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
- Raggi dello stage e della minimappa derivati da `--radius` dell'app (stessi
  20 e 14 px del prototipo). Gli altri token restano del canvas: vedi il
  report della fase di fondazione per i ruoli che l'app definisce con valori
  diversi.
- Il tema segue la classe `.dark` sull'elemento radice, quella già impostata
  da `src/lib/theme.tsx`. Nessun meccanismo nuovo.
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

### `src/etl-canvas/__tests__/engine.test.ts`

354 righe

```ts
import { describe, expect, it } from "vitest";
import { ELBOW_R, roundedPath } from "../../etl-layout";
import { createMotionEngine } from "../engine";
import type { AttrEl, GroupLike, LinkInput, PathEl } from "../engine";
import { waitingOpacity } from "../flow";
import { TRANSITION_MS } from "../transitions";
import type { Pt } from "../transitions";
import { fakeEnv } from "./fake-env";

interface FakeEl extends PathEl {
  attrs: Record<string, string>;
}

function el(len = 200): FakeEl {
  const attrs: Record<string, string> = {};
  return {
    attrs,
    style: { opacity: "" },
    setAttribute: (n, v) => void (attrs[n] = v),
    getTotalLength: () => len,
    getPointAtLength: (s) => ({ x: s, y: 0 }),
  };
}

function group() {
  const els = {
    path: el(),
    ghost: el(),
    flow: el(),
    a: el(),
    b: el(),
  };
  const map: Record<string, unknown> = {
    ".ec-link": els.path,
    ".ec-link-ghost": els.ghost,
    ".ec-flow": els.flow,
    '[data-dot="a"]': els.a,
    '[data-dot="b"]': els.b,
  };
  const g: GroupLike = { querySelector: (s) => map[s] ?? null };
  return { g, ...els };
}

const P1: Pt[] = [
  { x: 0, y: 0 },
  { x: 100, y: 0 },
  { x: 100, y: 80 },
];
const P2: Pt[] = [
  { x: 0, y: 20 },
  { x: 140, y: 20 },
  { x: 140, y: 100 },
];
const P3: Pt[] = [
  { x: 0, y: 0 },
  { x: 140, y: 100 },
];

function link(key: string, pts: Pt[], live = true): LinkInput {
  const d = roundedPath(pts, ELBOW_R);
  return { key, live, pts, d, pa: pts[0] as Pt, pb: pts[pts.length - 1] as Pt };
}

function setup() {
  const f = fakeEnv();
  const engine = createMotionEngine();
  const g = group();
  engine.registerLink("a|b", g.g);
  engine.start(f.env);
  return { f, engine, ...g };
}

describe("flusso", () => {
  it("un cavo attivo disegna il tubo a ogni frame; il ciclo gira", () => {
    const { f, engine, flow } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    expect(engine.debug().running).toBe(true);
    f.step(16);
    const d1 = flow.attrs["d"] as string;
    expect(d1.startsWith("M ")).toBe(true);
    f.step(400);
    expect(flow.attrs["d"]).not.toBe(d1);
    expect(engine.debug().running).toBe(true);
  });

  it("un cavo non attivo non ha flusso e non tiene acceso il ciclo", () => {
    const { f, engine, flow } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    f.step(16);
    expect(flow.attrs["d"] ?? "").toBe("");
    expect(engine.debug().running).toBe(false);
    expect(f.pending()).toBe(0);
  });

  it("nessun cavo e nessuna fetta: il ciclo si ferma da solo", () => {
    const { f, engine } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step();
    expect(engine.debug().running).toBe(true);
    engine.update({ links: [], gesturing: false });
    f.step();
    expect(engine.debug().running).toBe(false);
    expect(f.pending()).toBe(0);
  });

  it("durante un gesto di trascinamento il flusso si ferma, e riprende dopo", () => {
    const { f, engine, flow } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step(100);
    expect((flow.attrs["d"] as string).length).toBeGreaterThan(0);
    engine.setGesturing(true);
    expect(flow.attrs["d"]).toBe("");
    f.step(16);
    expect(engine.debug().running).toBe(false);
    engine.setGesturing(false);
    expect(engine.debug().running).toBe(true);
    f.step(16);
    expect((flow.attrs["d"] as string).length).toBeGreaterThan(0);
  });

  it("il flusso parte da 0 quando il cavo compare (t0 del cavo)", () => {
    const { f, engine, flow } = setup();
    f.step(5000);
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step(0);
    const w = flow.attrs["d"] as string;
    expect(w.startsWith("M ")).toBe(true);
    // a 0 ms dal cavo: tratto corto vicino alla porta di uscita (s tra 0 e ~3 px)
    const xs = w
      .slice(2, -2)
      .split(" L ")
      .map((p) => parseFloat(p.split(" ")[0] as string));
    expect(Math.max(...xs)).toBeLessThan(4);
  });
});

describe("attesa delle fette vuote", () => {
  it("la opacità segue la funzione pura a partire dalla registrazione", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    const s = el();
    engine.registerSlice("out-0:1", s);
    f.step(0);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(0), 9);
    f.step(475);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(475), 9);
    f.step(475);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(950), 9);
    expect(engine.debug().running).toBe(true);
  });

  it("la fase si conserva quando React ri-registra lo stesso elemento", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    const s = el();
    engine.registerSlice("x:1", s);
    f.step(300);
    engine.registerSlice("x:1", null);
    engine.registerSlice("x:1", s);
    f.step(0);
    expect(parseFloat(s.style.opacity)).toBeCloseTo(waitingOpacity(300), 9);
  });

  it("senza fette e senza cavi il ciclo si ferma", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    const s = el();
    engine.registerSlice("x:1", s);
    expect(engine.debug().running).toBe(true);
    engine.registerSlice("x:1", null);
    engine.update({ links: [], gesturing: false });
    f.step();
    expect(engine.debug().running).toBe(false);
    expect(engine.debug().slices).toBe(0);
  });
});

describe("transizione dei percorsi", () => {
  it("stesso numero di punti: si interpola e alla fine si ripristina il percorso calcolato", () => {
    const { f, engine, path, a, b } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    expect(path.attrs["d"]).toBeUndefined(); // primo percorso: nessuna transizione
    const next = link("a|b", P2, false);
    engine.update({ links: [next], gesturing: false });
    expect(path.attrs["d"]).toBe(roundedPath(P1, ELBOW_R)); // parte dal vecchio: niente scatto
    f.step(TRANSITION_MS / 2);
    const mid = P1.map((p, i) => ({
      x: (p.x + (P2[i] as Pt).x) / 2,
      y: (p.y + (P2[i] as Pt).y) / 2,
    }));
    expect(path.attrs["d"]).toBe(roundedPath(mid, ELBOW_R));
    expect(a.attrs["cy"]).toBe(String(mid[0]?.y));
    expect(engine.debug().running).toBe(true);
    f.step(TRANSITION_MS);
    expect(path.attrs["d"]).toBe(next.d);
    expect(a.attrs["cy"]).toBe(String(next.pa.y));
    expect(b.attrs["cx"]).toBe(String(next.pb.x));
    expect(engine.debug().running).toBe(false);
  });

  it("numero di punti diverso: dissolvenza incrociata, senza interpolare la geometria", () => {
    const { f, engine, path, ghost } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    const next = link("a|b", P3, false);
    engine.update({ links: [next], gesturing: false });
    expect(ghost.attrs["d"]).toBe(roundedPath(P1, ELBOW_R));
    f.step(TRANSITION_MS / 2);
    const o = parseFloat(ghost.style.opacity);
    const n = parseFloat(path.style.opacity);
    expect(o).toBeCloseTo(0.5, 9);
    expect(n).toBeCloseTo(0.5, 9);
    expect(o + n).toBeCloseTo(1, 9);
    expect(path.attrs["d"]).toBeUndefined(); // il tracciato nuovo non viene mai deformato
    f.step(TRANSITION_MS);
    expect(ghost.style.opacity).toBe("0");
    expect(ghost.attrs["d"]).toBe("");
    expect(path.style.opacity).toBe("");
    expect(engine.debug().running).toBe(false);
  });

  it("durante un gesto di trascinamento non c'è transizione", () => {
    const { f, engine, path, ghost } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: true });
    engine.update({ links: [link("a|b", P2, false)], gesturing: true });
    engine.update({ links: [link("a|b", P3, false)], gesturing: true });
    f.step(100);
    expect(path.attrs["d"]).toBeUndefined();
    expect(ghost.attrs["d"]).toBeUndefined();
    expect(engine.debug().running).toBe(false);
  });

  it("dopo il gesto, un cambio discreto si anima dall'ultimo percorso mostrato", () => {
    const { f, engine, path } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: true });
    engine.update({ links: [link("a|b", P2, false)], gesturing: false });
    expect(path.attrs["d"]).toBe(roundedPath(P1, ELBOW_R));
    f.step(TRANSITION_MS + 1);
    expect(path.attrs["d"]).toBe(roundedPath(P2, ELBOW_R));
  });

  it("un secondo cambio a metà transizione riparte da ciò che si vede", () => {
    const { f, engine, path } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    engine.update({ links: [link("a|b", P2, false)], gesturing: false });
    f.step(TRANSITION_MS / 2);
    const shown = path.attrs["d"];
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    expect(path.attrs["d"]).toBe(shown);
  });

  it("percorso identico: nessuna transizione", () => {
    const { f, engine, path } = setup();
    engine.update({ links: [link("a|b", P1, false)], gesturing: false });
    engine.update({
      links: [
        link(
          "a|b",
          P1.map((p) => ({ ...p })),
          false,
        ),
      ],
      gesturing: false,
    });
    expect(path.attrs["d"]).toBeUndefined();
    f.step(16);
    expect(engine.debug().running).toBe(false);
  });
});

describe("movimento ridotto", () => {
  it("nessun ciclo, transizioni istantanee, flusso fermo a metà cavo, attesa a riposo", () => {
    const f = fakeEnv();
    f.setReduced(true);
    const engine = createMotionEngine();
    const g = group();
    engine.registerLink("a|b", g.g);
    const slice = el();
    engine.registerSlice("x:1", slice);
    engine.start(f.env);
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.update({ links: [link("a|b", P2)], gesturing: false });
    expect(f.rafCalls()).toBe(0);
    expect(engine.debug().running).toBe(false);
    expect(g.path.attrs["d"]).toBeUndefined(); // nessuna interpolazione: resta il percorso calcolato
    expect(g.ghost.attrs["d"]).toBeUndefined();
    const still = g.flow.attrs["d"] as string;
    expect(still.startsWith("M ")).toBe(true); // indicazione statica
    engine.update({ links: [link("a|b", P2)], gesturing: false });
    expect(g.flow.attrs["d"]).toBe(still); // identica a ogni aggiornamento: nessun movimento
    expect(slice.style.opacity).toBe("");
  });

  it("l'attivazione a ciclo acceso ferma tutto e lascia lo stato finale", () => {
    const { f, engine, flow, path } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.update({ links: [link("a|b", P2)], gesturing: false });
    f.step(50);
    f.setReduced(true);
    expect(engine.debug().running).toBe(false);
    expect(path.attrs["d"]).toBe(roundedPath(P2, ELBOW_R));
    expect(flow.attrs["d"]).toBeTruthy();
  });
});

describe("scheda nascosta", () => {
  it("con la scheda nascosta il motore non consuma frame; al ritorno riparte", () => {
    const { f, engine } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    f.step();
    f.setHidden(true);
    const calls = f.rafCalls();
    f.step();
    f.step();
    expect(f.rafCalls()).toBe(calls);
    expect(engine.debug().running).toBe(false);
    f.setHidden(false);
    expect(engine.debug().running).toBe(true);
  });
});

describe("robustezza", () => {
  it("un cavo il cui gruppo non è ancora montato non rompe nulla", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    expect(() => {
      engine.update({ links: [link("x|y", P1)], gesturing: false });
      f.step();
    }).not.toThrow();
  });

  it("gli aggiornamenti prima dell'avvio si applicano all'avvio", () => {
    const f = fakeEnv();
    const engine = createMotionEngine();
    const g = group();
    engine.registerLink("a|b", g.g);
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.start(f.env);
    f.step(16);
    expect(g.flow.attrs["d"]).toBeTruthy();
  });

  it("stop ferma il ciclo", () => {
    const { f, engine } = setup();
    engine.update({ links: [link("a|b", P1)], gesturing: false });
    engine.stop();
    expect(f.pending()).toBe(0);
    void ({} as AttrEl);
  });
});
```

### `src/etl-canvas/__tests__/fake-env.ts`

57 righe

```ts
import type { LoopEnv } from "../loop";

/** Ambiente finto: orologio, rAF e i due eventi si controllano a mano. */
export function fakeEnv() {
  let t = 0;
  let hidden = false;
  let reduced = false;
  let nextId = 1;
  const queue = new Map<number, () => void>();
  const vis = new Set<() => void>();
  const mot = new Set<() => void>();
  let rafCalls = 0;
  const env: LoopEnv = {
    raf(cb) {
      rafCalls++;
      const id = nextId++;
      queue.set(id, cb);
      return id;
    },
    caf(id) {
      queue.delete(id);
    },
    now: () => t,
    hidden: () => hidden,
    reducedMotion: () => reduced,
    onVisibilityChange(cb) {
      vis.add(cb);
      return () => vis.delete(cb);
    },
    onReducedMotionChange(cb) {
      mot.add(cb);
      return () => mot.delete(cb);
    },
  };
  return {
    env,
    /** Avanza l'orologio e fa girare i frame in coda. */
    step(dt = 16) {
      t += dt;
      const run = [...queue.values()];
      queue.clear();
      run.forEach((cb) => cb());
    },
    setHidden(h: boolean) {
      hidden = h;
      [...vis].forEach((cb) => cb());
    },
    setReduced(r: boolean) {
      reduced = r;
      [...mot].forEach((cb) => cb());
    },
    pending: () => queue.size,
    rafCalls: () => rafCalls,
    listeners: () => vis.size + mot.size,
  };
}
```

### `src/etl-canvas/__tests__/flow.test.ts`

163 righe

```ts
import { describe, expect, it } from "vitest";
import {
  BACK,
  BALL,
  BASE_W,
  FRONT,
  SPEED,
  WAIT_HIGH,
  WAIT_LOW,
  WAIT_PERIOD,
  WAIT_REST,
  cubicBezier,
  easeInOut,
  flowWindow,
  flowWindowFor,
  smooth01,
  staticFlowWindow,
  tubeOutline,
  tubeProfile,
  waitingOpacity,
  waitingOpacityFor,
} from "../flow";

const LEN = 300;
/** Il ciclo del prototipo (riga 1489): lunghezza del cavo + BACK * 3.2. */
const CYCLE = LEN + BACK * 3.2;

describe("costanti del prototipo (righe 1415-1418)", () => {
  it("valgono quelle del prototipo", () => {
    expect([BASE_W, SPEED, BALL, FRONT, BACK]).toEqual([2.1, 0.16, 4.4, 7.5, 19]);
  });
});

describe("profilo del tubo", () => {
  it("è 1 sul punto che avanza; davanti si chiude più in fretta che dietro", () => {
    expect(tubeProfile(0)).toBe(1);
    expect(tubeProfile(FRONT)).toBeCloseTo(Math.exp(-1), 12);
    expect(tubeProfile(-BACK)).toBeCloseTo(Math.exp(-1), 12);
    expect(tubeProfile(FRONT)).toBeCloseTo(tubeProfile(-BACK), 12);
    expect(tubeProfile(10)).toBeLessThan(tubeProfile(-10));
  });

  it("smooth01 come nel prototipo (riga 1478)", () => {
    expect(smooth01(-1)).toBe(0);
    expect(smooth01(0.5)).toBe(0.5);
    expect(smooth01(2)).toBe(1);
    expect(smooth01(0.25)).toBeCloseTo(0.15625, 12);
  });
});

describe("finestra del flusso (righe 1488-1497)", () => {
  it("a 0 ms: la pallina entra dalla porta di uscita", () => {
    const w = flowWindow(LEN, 0)!;
    expect(w.cycle).toBeCloseTo(CYCLE, 9);
    expect(w.sb).toBeCloseTo(-BACK * 1.1, 9);
    expect(w.s0).toBe(0);
    expect(w.s1).toBeCloseTo(-BACK * 1.1 + FRONT * 3.2, 9);
  });

  it("a metà ciclo: nel mezzo del cavo, con coda BACK*3 e testa FRONT*3.2", () => {
    const elapsed = CYCLE / 2 / SPEED;
    const w = flowWindow(LEN, elapsed)!;
    const sb = CYCLE / 2 - BACK * 1.1;
    expect(w.sb).toBeCloseTo(sb, 9);
    expect(w.s0).toBeCloseTo(sb - BACK * 3, 9);
    expect(w.s1).toBeCloseTo(sb + FRONT * 3.2, 9);
  });

  it("a fine ciclo: la testa esce dal cavo e la coda è ancora dentro; poi riparte", () => {
    const before = flowWindow(LEN, (CYCLE - 1) / SPEED)!;
    expect(before.s1).toBe(LEN);
    expect(before.s0).toBeGreaterThan(LEN - 2 * BACK * 3);
    const after = flowWindow(LEN, (CYCLE + 1) / SPEED)!;
    expect(after.s0).toBe(0);
    expect(after.s1).toBeCloseTo(1 - BACK * 1.1 + FRONT * 3.2, 9);
  });

  it("è deterministico e periodico", () => {
    expect(flowWindow(LEN, 700)).toEqual(flowWindow(LEN, 700));
    const a = flowWindow(LEN, 500)!;
    const b = flowWindow(LEN, 500 + CYCLE / SPEED)!;
    expect(b.sb).toBeCloseTo(a.sb, 6);
  });

  it("niente da disegnare su un cavo di lunghezza nulla o troppo corto", () => {
    expect(flowWindow(0, 100)).toBeNull();
    expect(flowWindow(1, 0)).toBeNull();
  });

  it("con movimento ridotto è fermo a metà cavo, uguale a ogni istante", () => {
    const s = staticFlowWindow(LEN)!;
    expect(s.sb).toBe(LEN / 2);
    expect(flowWindowFor(LEN, 0, true)).toEqual(s);
    expect(flowWindowFor(LEN, 12345, true)).toEqual(s);
    expect(flowWindowFor(LEN, 12345, false)).toEqual(flowWindow(LEN, 12345));
  });
});

describe("contorno del tubo (righe 1500-1516)", () => {
  const sample = (s: number) => ({ x: s, y: 0 });
  const win = flowWindow(LEN, CYCLE / 2 / SPEED)!;
  const d = tubeOutline(sample, LEN, win);
  const parts = d.slice(2, -2).split(" L ");

  it("è un contorno chiuso con due lati campionati ogni ~1,6 px", () => {
    expect(d.startsWith("M ")).toBe(true);
    expect(d.endsWith(" Z")).toBe(true);
    const n = Math.max(10, Math.ceil((win.s1 - win.s0) / 1.6));
    expect(parts).toHaveLength(2 * (n + 1));
  });

  it("al punto che avanza lo spessore è BASE_W/2 + BALL per lato; ai capi è BASE_W/2", () => {
    const ys = parts.map((p) => Math.abs(parseFloat(p.split(" ")[1] as string)));
    expect(Math.max(...ys)).toBeGreaterThan(BASE_W / 2 + BALL - 0.15);
    expect(Math.max(...ys)).toBeLessThanOrEqual(BASE_W / 2 + BALL + 0.01);
    const tail = flowWindow(LEN, 0)!; // testa vicino alla porta: il tubo emerge dal bordo
    const y0 = tubeOutline(sample, LEN, tail).slice(2, -2).split(" L ");
    expect(Math.abs(parseFloat((y0[0] as string).split(" ")[1] as string))).toBeCloseTo(
      BASE_W / 2,
      6,
    );
  });

  it("segue il percorso dato: non lo modifica né lo ricalcola", () => {
    const calls: number[] = [];
    tubeOutline((s) => (calls.push(s), { x: s, y: 0 }), LEN, win);
    expect(calls.every((s) => s >= win.s0 - 1e-9 && s <= win.s1 + 1e-9)).toBe(true);
  });
});

describe("attesa delle fette vuote (righe 669-670)", () => {
  it("cubic-bezier: estremi, simmetria e valore noto di ease-in-out", () => {
    expect(easeInOut(0)).toBe(0);
    expect(easeInOut(1)).toBe(1);
    expect(easeInOut(0.5)).toBeCloseTo(0.5, 6);
    expect(easeInOut(0.25)).toBeCloseTo(0.1291, 3);
    expect(cubicBezier(0, 0, 1, 1)(0.3)).toBeCloseTo(0.3, 6);
  });

  it("0 ms → 0,45; un quarto → a metà; metà periodo → 0,95; fine periodo → 0,45", () => {
    expect(waitingOpacity(0)).toBeCloseTo(WAIT_LOW, 9);
    expect(waitingOpacity(WAIT_PERIOD / 4)).toBeCloseTo((WAIT_LOW + WAIT_HIGH) / 2, 5);
    expect(waitingOpacity(WAIT_PERIOD / 2)).toBeCloseTo(WAIT_HIGH, 9);
    expect(waitingOpacity((WAIT_PERIOD * 3) / 4)).toBeCloseTo((WAIT_LOW + WAIT_HIGH) / 2, 5);
    expect(waitingOpacity(WAIT_PERIOD)).toBeCloseTo(WAIT_LOW, 9);
    expect(waitingOpacity(WAIT_PERIOD * 7 + 100)).toBeCloseTo(waitingOpacity(100), 9);
  });

  it("resta sempre tra 0,45 e 0,95", () => {
    for (let t = 0; t < 4000; t += 37) {
      const o = waitingOpacity(t);
      expect(o).toBeGreaterThanOrEqual(WAIT_LOW - 1e-9);
      expect(o).toBeLessThanOrEqual(WAIT_HIGH + 1e-9);
    }
  });

  it("con movimento ridotto è ferma a riposo (0,85), a ogni istante", () => {
    expect(waitingOpacityFor(0, true)).toBe(WAIT_REST);
    expect(waitingOpacityFor(777, true)).toBe(WAIT_REST);
    expect(waitingOpacityFor(777, false)).toBe(waitingOpacity(777));
  });
});
```

### `src/etl-canvas/__tests__/helpers.ts`

26 righe

```ts
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { createEtlStore } from "../../etl-store";
import type { EtlStore } from "../../etl-store";
import { CanvasSurface } from "../EtlCanvas";
import { prototypeScene } from "../seed";

export const SIZE = { w: 1000, h: 640 };

export function storeWith(state = prototypeScene()): EtlStore {
  return createEtlStore({ initial: state });
}

export function html(store: EtlStore, size = SIZE): string {
  return renderToStaticMarkup(createElement(CanvasSurface, { store, size }));
}

/** Il frammento di HTML di un nodo, dal suo `<div class="ec-card ...">` al successivo. */
export function nodeHtml(markup: string, id: string): string {
  const at = markup.indexOf(`data-node-id="${id}"`);
  if (at < 0) throw new Error(`nodo ${id} non reso`);
  const start = markup.lastIndexOf("<div", at);
  const next = markup.indexOf('<div class="ec-card', at);
  return markup.slice(start, next < 0 ? undefined : next);
}
```

### `src/etl-canvas/__tests__/loop.test.ts`

165 righe

```ts
import { describe, expect, it, vi } from "vitest";
import { createLoop } from "../loop";
import type { Task } from "../loop";
import { fakeEnv } from "./fake-env";

function task(busy: () => boolean): Task & { frames: number[]; settled: number } {
  const t = {
    frames: [] as number[],
    settled: 0,
    frame(now: number) {
      t.frames.push(now);
      return busy();
    },
    settle() {
      t.settled++;
    },
  };
  return t;
}

describe("ciclo condiviso", () => {
  it("un solo rAF alla volta, con qualunque numero di compiti", () => {
    const f = fakeEnv();
    const loop = createLoop(f.env);
    const a = task(() => true);
    const b = task(() => true);
    loop.add(a);
    loop.add(b);
    expect(f.pending()).toBe(1);
    f.step();
    expect(f.pending()).toBe(1);
    expect(a.frames).toHaveLength(1);
    expect(b.frames).toHaveLength(1);
    loop.dispose();
  });

  it("si ferma da solo quando nessun compito ha nulla da animare, e riparte con wake", () => {
    const f = fakeEnv();
    let busy = true;
    const loop = createLoop(f.env);
    loop.add(task(() => busy));
    f.step();
    f.step();
    expect(loop.running()).toBe(true);
    busy = false;
    f.step();
    expect(loop.running()).toBe(false);
    expect(f.pending()).toBe(0);
    const calls = f.rafCalls();
    f.step();
    expect(f.rafCalls()).toBe(calls);
    busy = true;
    loop.wake();
    expect(loop.running()).toBe(true);
    loop.dispose();
  });

  it("senza compiti non parte", () => {
    const f = fakeEnv();
    const loop = createLoop(f.env);
    loop.wake();
    expect(f.rafCalls()).toBe(0);
    const off = loop.add(task(() => true));
    off();
    expect(f.pending()).toBe(0);
    loop.dispose();
  });

  it("si ferma quando la scheda è nascosta e riparte quando torna visibile", () => {
    const f = fakeEnv();
    const t = task(() => true);
    const loop = createLoop(f.env);
    loop.add(t);
    f.step();
    expect(loop.running()).toBe(true);
    f.setHidden(true);
    expect(loop.running()).toBe(false);
    expect(f.pending()).toBe(0);
    const frames = t.frames.length;
    f.step();
    f.step();
    expect(t.frames).toHaveLength(frames);
    f.setHidden(false);
    expect(loop.running()).toBe(true);
    f.step();
    expect(t.frames.length).toBe(frames + 1);
    loop.dispose();
  });

  it("con la scheda già nascosta non parte affatto", () => {
    const f = fakeEnv();
    f.setHidden(true);
    const loop = createLoop(f.env);
    loop.add(task(() => true));
    expect(f.rafCalls()).toBe(0);
    loop.dispose();
  });

  it("non riprogramma un frame se la scheda si nasconde durante il frame", () => {
    const f = fakeEnv();
    const loop = createLoop(f.env);
    loop.add({
      frame: () => {
        f.setHidden(true);
        return true;
      },
      settle: () => {},
    });
    f.step();
    expect(loop.running()).toBe(false);
    loop.dispose();
  });

  it("movimento ridotto: il ciclo non parte mai, i compiti mostrano lo stato finale", () => {
    const f = fakeEnv();
    f.setReduced(true);
    const t = task(() => true);
    const loop = createLoop(f.env);
    loop.add(t);
    loop.wake();
    expect(f.rafCalls()).toBe(0);
    expect(loop.running()).toBe(false);
    expect(t.frames).toHaveLength(0);
    expect(t.settled).toBeGreaterThanOrEqual(1);
    loop.dispose();
  });

  it("se il movimento ridotto si attiva mentre gira, si ferma e mostra lo stato finale; se si disattiva, riparte", () => {
    const f = fakeEnv();
    const t = task(() => true);
    const loop = createLoop(f.env);
    loop.add(t);
    f.step();
    expect(loop.running()).toBe(true);
    f.setReduced(true);
    expect(loop.running()).toBe(false);
    expect(t.settled).toBe(1);
    f.setReduced(false);
    expect(loop.running()).toBe(true);
    loop.dispose();
  });

  it("dispose ferma il ciclo e toglie gli ascoltatori", () => {
    const f = fakeEnv();
    const loop = createLoop(f.env);
    loop.add(task(() => true));
    expect(f.listeners()).toBe(2);
    loop.dispose();
    expect(f.pending()).toBe(0);
    expect(f.listeners()).toBe(0);
  });

  it("i frame ricevono l'orologio dell'ambiente", () => {
    const f = fakeEnv();
    const t = task(() => true);
    const loop = createLoop(f.env);
    loop.add(t);
    f.step(100);
    f.step(50);
    expect(t.frames).toEqual([100, 150]);
    loop.dispose();
    void vi;
  });
});
```

### `src/etl-canvas/__tests__/no-reroute.test.ts`

129 righe

```ts
/**
 * Vincolo della Fase 4b: le animazioni sono un effetto visivo sopra
 * percorsi già calcolati. Nessuna animazione richiama settleLinks (né alcuna
 * funzione di etl-layout che instradi) e nessuna altera i percorsi di
 * getRoutes.
 */
import { describe, expect, it, vi } from "vitest";

vi.mock("../../etl-layout", async (importOriginal) => {
  const actual = await importOriginal<typeof import("../../etl-layout")>();
  return {
    ...actual,
    settleLinks: vi.fn(actual.settleLinks),
    layoutLinks: vi.fn(actual.layoutLinks),
    chooseRoute: vi.fn(actual.chooseRoute),
    buildRoute: vi.fn(actual.buildRoute),
    routeCandidates: vi.fn(actual.routeCandidates),
    shapeCandidates: vi.fn(actual.shapeCandidates),
    autoLayout: vi.fn(actual.autoLayout),
  };
});

import * as layout from "../../etl-layout";
import { linkKey } from "../../etl-layout";
import { createMotionEngine } from "../engine";
import type { LinkInput } from "../engine";
import { fakeEnv } from "./fake-env";
import { storeWith } from "./helpers";

const ROUTING = [
  "settleLinks",
  "layoutLinks",
  "chooseRoute",
  "buildRoute",
  "routeCandidates",
  "shapeCandidates",
  "autoLayout",
] as const;

function el() {
  const attrs: Record<string, string> = {};
  return {
    attrs,
    style: { opacity: "" },
    setAttribute: (n: string, v: string) => void (attrs[n] = v),
    getTotalLength: () => 300,
    getPointAtLength: (s: number) => ({ x: s, y: 0 }),
  };
}

function inputs(store: ReturnType<typeof storeWith>): LinkInput[] {
  const routes = store.getRoutes();
  return store.getState().graph.links.flatMap((l) => {
    const r = routes[linkKey(l)];
    return r ? [{ key: linkKey(l), live: true, pts: r.pts, d: r.d, pa: r.pa, pb: r.pb }] : [];
  });
}

describe("le animazioni non ricalcolano né alterano i percorsi", () => {
  it("nessuna funzione di instradamento viene chiamata mentre girano flusso, attesa e transizioni", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    const before = inputs(store);
    expect(before.length).toBeGreaterThan(1);
    const routesBefore = JSON.stringify(store.getRoutes());

    // da qui in poi il calcolo dei percorsi è già avvenuto: si azzerano i contatori
    for (const name of ROUTING) (layout[name] as unknown as ReturnType<typeof vi.fn>).mockClear();

    const f = fakeEnv();
    const engine = createMotionEngine();
    const groups = new Map<string, ReturnType<typeof el>[]>();
    for (const l of before) {
      const els = [el(), el(), el(), el(), el()];
      groups.set(l.key, els);
      const map: Record<string, unknown> = {
        ".ec-link": els[0],
        ".ec-link-ghost": els[1],
        ".ec-flow": els[2],
        '[data-dot="a"]': els[3],
        '[data-dot="b"]': els[4],
      };
      engine.registerLink(l.key, { querySelector: (s) => map[s] ?? null });
    }
    engine.registerSlice("out-0:1", el());
    engine.start(f.env);
    engine.update({ links: before, gesturing: false });
    for (let i = 0; i < 40; i++) f.step(16);

    // un cambio discreto dei percorsi (spostato a mano per simulare autoLayout): si anima
    const moved = before.map((l) => {
      const pts = l.pts.map((p) => ({ x: p.x + 30, y: p.y + 10 }));
      return { ...l, pts, pa: pts[0]!, pb: pts[pts.length - 1]! };
    });
    engine.update({ links: moved, gesturing: false });
    for (let i = 0; i < 40; i++) f.step(16);
    engine.update({ links: before, gesturing: true });
    for (let i = 0; i < 10; i++) f.step(16);
    engine.update({ links: before, gesturing: false });
    for (let i = 0; i < 40; i++) f.step(16);

    for (const name of ROUTING) {
      expect(layout[name], name).not.toHaveBeenCalled();
    }
    // e i percorsi restituiti da getRoutes sono rimasti identici
    expect(JSON.stringify(store.getRoutes())).toBe(routesBefore);
    expect(JSON.stringify(inputs(store))).toBe(JSON.stringify(before));
  });

  it("con lo store reale: una modifica del grafo ricalcola i percorsi solo tramite lo store, non le animazioni", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    store.getRoutes();
    (layout.settleLinks as unknown as ReturnType<typeof vi.fn>).mockClear();
    const f = fakeEnv();
    const engine = createMotionEngine();
    engine.start(f.env);
    engine.update({ links: inputs(store), gesturing: false });
    for (let i = 0; i < 30; i++) f.step(16);
    // le animazioni hanno girato e settleLinks non è stato invocato da loro
    expect(layout.settleLinks).not.toHaveBeenCalled();
    // lo store invece ricalcola quando cambia il grafo (controllo di sanità dello spy)
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-sort" } });
    store.getRoutes();
    expect(layout.settleLinks).toHaveBeenCalled();
  });
});
```

