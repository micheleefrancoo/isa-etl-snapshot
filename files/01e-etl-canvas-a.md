# 01e-etl-canvas-a.md

File in questo blocco:

- `src/etl-canvas/EtlCanvas.tsx`
- `src/etl-canvas/Links.tsx`
- `src/etl-canvas/Minimap.tsx`
- `src/etl-canvas/Node.tsx`
- `src/etl-canvas/README.md`
- `src/etl-canvas/__tests__/helpers.ts`
- `src/etl-canvas/__tests__/render.test.ts`
- `src/etl-canvas/__tests__/ssr.test.tsx`
- `src/etl-canvas/__tests__/tokens.test.ts`
- `src/etl-canvas/__tests__/view.test.ts`
- `src/etl-canvas/actions.ts`
- `src/etl-canvas/canvas.css`
- `src/etl-canvas/contrast.ts`
- `src/etl-canvas/icons.tsx`
- `src/etl-canvas/index.ts`
- `src/etl-canvas/model.ts`
- `src/etl-canvas/seed.ts`

---

### `src/etl-canvas/EtlCanvas.tsx`

210 righe

```tsx
import { useEffect, useMemo, useRef, useState, useSyncExternalStore } from "react";
import type { PointerEvent as ReactPointerEvent } from "react";
import { WORLD_H, WORLD_W } from "../etl-layout";
import type { Size } from "../etl-layout";
import { nodeStates } from "../etl-store";
import type { EtlStore } from "../etl-store";
import { useEtlState } from "../etl-store/react";
import { fit, zoomAtPoint, zoomIn, zoomOut, zoomReset } from "./actions";
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
}

/**
 * La resa del canvas per uno store e una dimensione date. Non tocca né
 * `window` né `document` durante il rendering: gli ascoltatori (spazio,
 * rotella) si registrano in un effetto, quindi si può rendere anche in
 * Node (test, rendering lato server).
 */
export function CanvasSurface(props: CanvasSurfaceProps) {
  const { store, size } = props;
  const graph = useEtlState((s) => s.graph, store);
  const view = useEtlState((s) => s.view, store);
  const selection = useEtlState((s) => s.selection, store);
  const maxBends = useEtlState((s) => s.options.maxBends, store);
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

37 righe

```tsx
import { memo } from "react";
import type { Graph } from "../etl-core";
import { linkKey } from "../etl-layout";
import type { LinkRoutes } from "../etl-layout";

/** I cavi: percorsi calcolati da etl-store, statici (prototipo `drawLinks`, righe 1399-1409). */
export const Links = memo(function Links(props: { graph: Graph; routes: LinkRoutes }) {
  const { graph, routes } = props;
  return (
    <svg className="ec-links" width="100%" height="100%" aria-hidden="true">
      {graph.links.map((l) => {
        const route = routes[linkKey(l)];
        if (!route) return null;
        const a = graph.cards[l.from];
        const b = graph.cards[l.to];
        return (
          <g key={linkKey(l)} data-link={linkKey(l)}>
            <path className="ec-link" d={route.d} />
            <circle
              className={a?.kind === "dataset" ? "ec-link-dot-ds" : "ec-link-dot-op"}
              cx={route.pa.x}
              cy={route.pa.y}
              r={2.6}
            />
            <circle
              className={b?.kind === "dataset" ? "ec-link-dot-ds" : "ec-link-dot-op"}
              cx={route.pb.x}
              cy={route.pb.y}
              r={2.6}
            />
          </g>
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

### `src/etl-canvas/Node.tsx`

35 righe

```tsx
import { memo } from "react";
import { Icon } from "./icons";
import type { NodeView } from "./model";

/** Un nodo: quadrato con icone, etichetta, indicatore ambra (prototipo `createCardEl`, righe 1000-1015). */
export const Node = memo(function Node(props: { node: NodeView }) {
  const { node } = props;
  const { card } = node;
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
                <Icon id={s.full ? "dataset" : "empty"} />
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

81 righe

```md
# etl-canvas — Fase 4a: il canvas visibile

Resa visiva del canvas ETL in React, fedele al prototipo
`docs/prototype/isa-fusion-prototype.html`. Solo **vista**: token, nodi,
cavi (statici), pan, zoom, controlli di zoom, minimappa. Trascinamento,
fusione, collegamento, selezione, tastiera (Fase 5), cassetta e Inspector
(Fase 6) e animazioni dei cavi (Fase 4b) non ci sono ancora.

Importa da `etl-core`, `etl-layout` ed `etl-store`; nessuno di questi importa
da qui. Non usa il vecchio stato (`src/lib/etl-workflow.tsx`): legge e
scrive solo attraverso `etl-store`.

## Moduli

```
tokens.css       token, con ambito .etl-canvas; tema chiaro = prototipo (con la riga), scuro progettato
canvas.css       aspetto di nodi, cavi, controlli, minimappa; carica Manrope e tokens.css
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
```

## Uso

```tsx
const store = usePersistentEtlStore(solutionId); // etl-store/react
<EtlCanvas store={store} />;
```

Il contenitore deve avere un'altezza (minimo 520 px). Nella rotta
`solutions.$solutionId.etl.tsx` il nuovo canvas compare **solo** con
`?canvas=v2`; senza parametro resta il canvas vecchio. Solo in sviluppo,
`?canvas=v2&seed=prototype` carica la scena del prototipo se il canvas è
vuoto (in produzione `seed` è ignorato) e `window.__etlStore` espone lo
store alla console e allo script delle schermate.

## Rendering lato server

`EtlCanvas` produce, sul server e nel primo rendering di idratazione, lo
stesso contenitore vuoto (`useSyncExternalStore` con snapshot server
`false`); dopo l'idratazione misura l'area con un `ResizeObserver` e monta
`CanvasSurface`. Nessun accesso a `window`/`document` durante il rendering.

## Carattere e temi

- Manrope (`@fontsource-variable/manrope`, unica dipendenza nuova) solo
  dentro `.etl-canvas`; il resto dell'app resta in Poppins. Nessuna
  richiesta a server esterni: i file sono serviti dall'app, e il browser
  scarica un sottoinsieme solo se il testo lo usa.
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

### `src/etl-canvas/__tests__/render.test.ts`

157 righe

```ts
import { describe, expect, it } from "vitest";
import { EMPTY_SLOT_ICON } from "../../etl-core";
import { createEtlStore, initialState } from "../../etl-store";
import type { EtlStore } from "../../etl-store";
import { prototypeScene } from "../seed";
import { html, nodeHtml, storeWith } from "./helpers";

function count(markup: string, needle: string): number {
  return markup.split(needle).length - 1;
}

function ids(store: EtlStore): string[] {
  return Object.keys(store.getState().graph.cards);
}

describe("scena del prototipo", () => {
  it("rende 5 nodi con le posizioni del prototipo e nessun cavo", () => {
    const store = storeWith();
    const markup = html(store);
    expect(count(markup, "data-node-id=")).toBe(5);
    expect(count(markup, 'class="ec-link"')).toBe(0);
    expect(nodeHtml(markup, "ds1")).toContain("left:26px;top:182px");
    expect(nodeHtml(markup, "op-filter")).toContain("left:260px;top:52px");
    expect(nodeHtml(markup, "op-join")).toContain("left:260px;top:182px");
    expect(nodeHtml(markup, "op-sort")).toContain("left:260px;top:338px");
    expect(nodeHtml(markup, "op-export")).toContain("left:442px;top:338px");
    expect(markup).toContain("Vendite 2026");
    expect(markup).toContain("Filtra Righe");
    expect(markup).toContain("Unisci (Join)");
  });

  it("classi per tipo di nodo", () => {
    const markup = html(storeWith());
    expect(nodeHtml(markup, "ds1")).toMatch(/class="ec-card ec-dataset"/);
    for (const id of ["op-filter", "op-join", "op-sort", "op-export"]) {
      expect(nodeHtml(markup, id)).toMatch(/class="ec-card( ec-warn)?"/);
      expect(nodeHtml(markup, id)).not.toContain("ec-dataset");
    }
  });

  it("indicatore ambra sui nodi incompleti, non sul dataset completo", () => {
    const markup = html(storeWith());
    expect(nodeHtml(markup, "ds1")).not.toContain("ec-state-dot");
    for (const id of ["op-filter", "op-join", "op-sort", "op-export"]) {
      expect(nodeHtml(markup, id)).toContain("ec-state-dot");
      expect(nodeHtml(markup, id)).toContain("ec-warn");
    }
    // il motivo di etl-core è nell'attributo title
    expect(nodeHtml(markup, "op-join")).toContain("Mancano tabelle in ingresso");
  });

  it("icona a una colonna per i nodi semplici", () => {
    expect(nodeHtml(html(storeWith()), "op-filter")).toContain("ec-icon-wrap ec-count-1");
  });
});

describe("cavi, output e box combinati", () => {
  function connected(): { store: EtlStore; ds: string } {
    const store = storeWith();
    const r = store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    expect(r).toEqual({ ok: true });
    return { store, ds: "ds1" };
  }

  it("un cavo per collegamento, con i due capi", () => {
    const { store } = connected();
    const links = store.getState().graph.links;
    expect(links.length).toBeGreaterThanOrEqual(2); // ds → filtro e filtro → output generato
    const markup = html(store);
    expect(count(markup, 'class="ec-link"')).toBe(links.length);
    expect(count(markup, 'r="2.6"')).toBe(links.length * 2);
    expect(markup).toMatch(/<path class="ec-link" d="M /);
  });

  it("l'output generato ha le classi dataset e output", () => {
    const { store } = connected();
    const out = ids(store).find((id) => store.getState().graph.cards[id]?.isOutput);
    expect(out).toBeDefined();
    expect(nodeHtml(html(store), out as string)).toMatch(/class="ec-card ec-dataset ec-output"/);
  });

  it("output parziale di un join con una sola tabella: due fette, una vuota", () => {
    const store = storeWith();
    expect(store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } })).toEqual({
      ok: true,
    });
    const out = ids(store).find((id) => store.getState().graph.cards[id]?.isOutput) as string;
    const card = store.getState().graph.cards[out];
    expect(card?.capacity).toBe(2);
    expect(card?.filled).toBe(1);
    const node = nodeHtml(html(store), out);
    expect(node).toContain("ec-split");
    // le fette non devono usare la classe dello stato vuoto del canvas (position:absolute; inset:0)
    expect(node).not.toMatch(/class="[^"]*\bec-empty\b/);
    expect(node).toContain("ec-partial");
    expect(count(node, "ec-slice ")).toBe(2);
    expect(count(node, "ec-slice ec-slice-full")).toBe(1);
    expect(count(node, "ec-slice ec-slice-empty")).toBe(1);
    // la fetta vuota mostra il simbolo <>
    const empty = node.slice(node.indexOf("ec-slice ec-slice-empty"));
    expect(empty).toContain(EMPTY_SLOT_ICON.slice(0, 27));
    // la fetta piena è la prima (riempita da sinistra)
    expect(node.indexOf("ec-slice ec-slice-full")).toBeLessThan(
      node.indexOf("ec-slice ec-slice-empty"),
    );
  });

  it("box combinato: classe combined e icone in file da 3", () => {
    const store = storeWith();
    expect(
      store.dispatch({ type: "merge", payload: { dragged: "op-sort", target: "op-filter" } }),
    ).toEqual({ ok: true });
    const box = ids(store).find(
      (id) => (store.getState().graph.cards[id]?.components.length ?? 0) > 1,
    );
    expect(box).toBeDefined();
    const node = nodeHtml(html(store), box as string);
    expect(node).toContain("ec-combined");
    expect(node).toContain("ec-icon-wrap ec-count-2");
  });
});

describe("selezione, vista, stato vuoto", () => {
  it("contorno di selezione per i nodi in selection", () => {
    const store = storeWith();
    store.dispatch({ type: "select", payload: { ids: ["op-sort"] } });
    const markup = html(store);
    expect(nodeHtml(markup, "op-sort")).toContain("ec-selected");
    expect(nodeHtml(markup, "op-filter")).not.toContain("ec-selected");
  });

  it("la vista dello store diventa la trasformazione del mondo e la percentuale", () => {
    const store = storeWith();
    store.dispatch({ type: "setView", payload: { x: 40, y: -12, zoom: 1.5 } });
    const markup = html(store);
    expect(markup).toContain("translate(40px, -12px) scale(1.5)");
    expect(markup).toContain(">150%<");
  });

  it("canvas vuoto: stato vuoto centrato, senza minimappa di nodi", () => {
    const markup = html(createEtlStore({ initial: initialState() }));
    expect(markup).toContain("ec-empty");
    expect(markup).toContain("Aggiungi un dataset");
    expect(count(markup, "ec-mm-node")).toBe(0);
  });

  it("con nodi lo stato vuoto non c'è; controlli e minimappa ci sono", () => {
    const markup = html(storeWith());
    expect(markup).not.toContain("ec-empty");
    expect(markup).toContain('aria-label="Riduci"');
    expect(markup).toContain('aria-label="Ingrandisci"');
    expect(markup).toContain(">Adatta<");
    expect(count(markup, "ec-mm-node")).toBeGreaterThanOrEqual(5);
    expect(prototypeScene().graph.links).toHaveLength(0);
  });
});
```

### `src/etl-canvas/__tests__/ssr.test.tsx`

84 righe

```tsx
import { createMemoryHistory, RouterProvider } from "@tanstack/react-router";
import { QueryClient } from "@tanstack/react-query";
import { createRouter } from "@tanstack/react-router";
import { createElement } from "react";
import { renderToString } from "react-dom/server";
import { describe, expect, it, vi } from "vitest";
import { createEtlStore } from "../../etl-store";
import { EtlCanvas } from "../EtlCanvas";
import { routeTree } from "../../routeTree.gen";

// Le soluzioni vivono nel browser (localStorage): sul server non ce n'è nessuna e la
// rotta mostrerebbe "Soluzione non trovata". Se ne simula una per rendere davvero la pagina.
vi.mock("@/lib/solutions-store", async (importOriginal) => {
  const actual = await importOriginal<typeof import("@/lib/solutions-store")>();
  return {
    ...actual,
    useSolutions: () => ({
      ...actual.useSolutions(),
      solutions: [
        {
          id: "demo",
          name: "Demo",
          version: "v1",
          modules: { etl: "draft" },
          shares: [],
          parameters: [],
          series: [],
        },
      ],
    }),
  };
});

/**
 * In vitest `styles.css?url` vale "": React se ne lamenta per il link della
 * radice, per qualunque rotta. Non dipende dal canvas, quindi si ignora.
 */
function relevant(calls: unknown[][]): unknown[][] {
  return calls.filter((c) => !/precedence|empty string/.test(String(c[0])));
}

async function renderRoute(url: string): Promise<string> {
  const router = createRouter({
    routeTree,
    context: { queryClient: new QueryClient() },
    history: createMemoryHistory({ initialEntries: [url] }),
  });
  await router.load();
  return renderToString(createElement(RouterProvider, { router }));
}

describe("rendering lato server", () => {
  it("EtlCanvas sul server produce solo il contenitore, senza canvas né errori", () => {
    const errors = vi.spyOn(console, "error").mockImplementation(() => {});
    const out = renderToString(createElement(EtlCanvas, { store: createEtlStore() }));
    expect(out).toContain("etl-canvas-host");
    expect(out).not.toContain("ec-stage");
    expect(errors).not.toHaveBeenCalled();
    errors.mockRestore();
  });

  it("la rotta con ?canvas=v2 si rende senza errori", async () => {
    const errors = vi.spyOn(console, "error").mockImplementation(() => {});
    const warns = vi.spyOn(console, "warn").mockImplementation(() => {});
    const out = await renderRoute("/solutions/demo/etl?canvas=v2&seed=prototype");
    expect(out.length).toBeGreaterThan(0);
    expect(relevant(errors.mock.calls)).toEqual([]);
    expect(relevant(warns.mock.calls)).toEqual([]);
    expect(out).toContain("canvas v2");
    expect(out).not.toContain("ec-stage");
    errors.mockRestore();
    warns.mockRestore();
  });

  it("la rotta senza parametro si rende ancora (canvas vecchio)", async () => {
    const errors = vi.spyOn(console, "error").mockImplementation(() => {});
    const out = await renderRoute("/solutions/demo/etl");
    expect(out.length).toBeGreaterThan(0);
    expect(out).not.toContain("canvas v2");
    expect(relevant(errors.mock.calls)).toEqual([]);
    errors.mockRestore();
  });
});
```

### `src/etl-canvas/__tests__/tokens.test.ts`

89 righe

```ts
import { readFileSync } from "node:fs";
import { describe, expect, it } from "vitest";
import { PAIRS, measure, readTokens } from "../contrast";

const css = readFileSync(new URL("../tokens.css", import.meta.url), "utf8");
const prototype = readFileSync(
  new URL("../../../docs/prototype/isa-fusion-prototype.html", import.meta.url),
  "utf8",
);
const light = readTokens(css, "light");
const dark = readTokens(css, "dark");

/** Il prototipo scrive i colori in forme diverse (`#E1DCF0`, `rgba(108,99,255,0.34)`): si confrontano normalizzati. */
function norm(s: string): string {
  return s
    .toLowerCase()
    .replace(/\s+/g, "")
    .replace(/0\.(\d)0+\b/g, "0.$1");
}
const proto = norm(prototype);

/** Token del tema chiaro che non vengono dal prototipo (nuovi o di misura). */
const NEW_TOKENS = new Set([
  "--ec-node-op-border",
  "--ec-empty-ink",
  "--ec-accent-text",
  "--ec-font",
  "--ec-glass-shadow",
  "--ec-glass-blur",
  "--ec-r-op",
  "--ec-r-fill",
  "--ec-r-stage",
  "--ec-r-minimap",
  "--ec-r-pill",
  "--ec-node-fill-ink",
  "--ec-output-opacity",
  "--ec-link-dot-fill",
  "--ec-link-dot-op",
  "--ec-select",
  "--ec-mm-node-ds",
  "--ec-mm-view-line",
  "--ec-node-op-ink",
  "--ec-ink",
  "--ec-accent",
]);

describe("tema chiaro: valori del prototipo", () => {
  it("ogni colore chiaro è scritto nel prototipo", () => {
    for (const [name, value] of Object.entries(light)) {
      if (NEW_TOKENS.has(name) || !/^(#|rgba?\()/.test(value)) continue;
      expect(proto, `${name}: ${value}`).toContain(norm(value));
    }
  });

  it("i valori chiave coincidono con le variabili CSS del prototipo", () => {
    expect(light["--ec-bg"]).toBe("#f5f3ee");
    expect(light["--ec-ink"]).toBe("#262420");
    expect(light["--ec-muted"]).toBe("#847e74");
    expect(light["--ec-accent"]).toBe("#6c63ff");
    expect(light["--ec-accent-soft"]).toBe("rgba(108, 99, 255, 0.16)");
    expect(light["--ec-accent-soft-2"]).toBe("rgba(108, 99, 255, 0.34)");
    expect(light["--ec-surface-strong"]).toBe("rgba(255, 255, 255, 0.92)");
    expect(light["--ec-panel-border"]).toBe("rgba(38, 36, 32, 0.06)");
    expect(light["--ec-r-op"]).toBe("22px");
    expect(light["--ec-r-fill"]).toBe("26px");
  });
});

describe("tema scuro: contrasto", () => {
  it("definisce gli stessi token del tema chiaro", () => {
    for (const name of Object.keys(light)) {
      if (name === "--ec-font" || name.startsWith("--ec-r-") || name === "--ec-glass-blur")
        continue;
      expect(dark, name).toHaveProperty(name);
    }
  });

  for (const pair of PAIRS) {
    it(`${pair.role}: almeno ${pair.min}:1`, () => {
      expect(measure(dark, pair)).toBeGreaterThanOrEqual(pair.min);
    });
  }

  it("l'accento resta #6C63FF sui riempimenti", () => {
    expect(dark["--ec-accent"]).toBe("#6c63ff");
    expect(dark["--ec-node-fill"]).toBe("#6c63ff");
  });
});
```

### `src/etl-canvas/__tests__/view.test.ts`

148 righe

```ts
import { describe, expect, it } from "vitest";
import { CARD, LABEL_H } from "../../etl-layout";
import { ZOOM_MAX, ZOOM_MIN } from "../../etl-store";
import { fit, zoomAtPoint, zoomIn, zoomOut, zoomReset } from "../actions";
import { prototypeScene } from "../seed";
import { bounds, fitView, minimapFrame, toWorld, viewFromMinimap, zoomAt } from "../view";
import { SIZE, storeWith } from "./helpers";

function allInside(store: ReturnType<typeof storeWith>, size = SIZE): void {
  const { view, graph } = store.getState();
  for (const c of Object.values(graph.cards)) {
    const x1 = c.x * view.zoom + view.x;
    const y1 = c.y * view.zoom + view.y;
    const x2 = (c.x + CARD) * view.zoom + view.x;
    const y2 = (c.y + CARD + LABEL_H) * view.zoom + view.y;
    expect(x1, c.id).toBeGreaterThanOrEqual(0);
    expect(y1, c.id).toBeGreaterThanOrEqual(0);
    expect(x2, c.id).toBeLessThanOrEqual(size.w);
    expect(y2, c.id).toBeLessThanOrEqual(size.h);
  }
}

describe("Adatta", () => {
  it("dopo la chiamata tutti i nodi rientrano nell'area visibile", () => {
    const store = storeWith();
    store.dispatch({ type: "setView", payload: { x: -900, y: 400, zoom: 2 } });
    fit(store, SIZE);
    allInside(store);
  });

  it("vale per finestre piccole e grandi", () => {
    for (const size of [
      { w: 320, h: 240 },
      { w: 1440, h: 900 },
      { w: 600, h: 1200 },
    ]) {
      const store = storeWith();
      fit(store, size);
      allInside(store, size);
    }
  });

  it("vale anche per una scena sparsa su tutto il mondo", () => {
    const size = { w: 1200, h: 800 };
    const store = storeWith();
    const s = prototypeScene();
    const far = { ...s.graph.cards["op-export"]!, x: 2400, y: 1400 };
    store.replaceState({
      ...s,
      graph: { ...s.graph, cards: { ...s.graph.cards, "op-export": far } },
    });
    fit(store, size);
    allInside(store, size);
  });

  it("oltre il limite di zoom (0,35) Adatta si ferma al limite, come nel prototipo", () => {
    const store = storeWith();
    const s = prototypeScene();
    const far = { ...s.graph.cards["op-export"]!, x: 2400, y: 1400 };
    store.replaceState({
      ...s,
      graph: { ...s.graph, cards: { ...s.graph.cards, "op-export": far } },
    });
    fit(store, { w: 320, h: 240 });
    expect(store.getState().view.zoom).toBe(ZOOM_MIN);
  });

  it("con il margine del prototipo (48) e zoom al più 1,25", () => {
    const one = [{ x: 100, y: 100 }];
    const v = fitView(one, { w: 2000, h: 2000 });
    expect(v.zoom).toBe(1.25);
    const b = bounds(one)!;
    // centrato
    expect(v.x + b.x1 * v.zoom).toBeCloseTo((2000 - (b.x2 - b.x1) * v.zoom) / 2, 6);
  });

  it("canvas vuoto: vista di partenza", () => {
    expect(fitView([], SIZE)).toEqual({ x: 0, y: 0, zoom: 1 });
  });
});

describe("zoom", () => {
  it("limiti del prototipo (0,35 – 2)", () => {
    const store = storeWith();
    for (let i = 0; i < 40; i++) zoomIn(store, SIZE);
    expect(store.getState().view.zoom).toBe(ZOOM_MAX);
    for (let i = 0; i < 80; i++) zoomOut(store, SIZE);
    expect(store.getState().view.zoom).toBe(ZOOM_MIN);
    zoomReset(store, SIZE);
    expect(store.getState().view.zoom).toBe(1);
  });

  it("attorno al puntatore il punto del mondo sotto il puntatore non si muove", () => {
    const store = storeWith();
    store.dispatch({ type: "setView", payload: { x: 30, y: 50, zoom: 0.8 } });
    const before = toWorld(store.getState().view, 400, 300);
    zoomAtPoint(store, 400, 300, 1.7);
    const after = toWorld(store.getState().view, 400, 300);
    expect(after.x).toBeCloseTo(before.x, 6);
    expect(after.y).toBeCloseTo(before.y, 6);
    expect(store.getState().view.zoom).toBeCloseTo(1.7, 6);
  });

  it("zoomAt è puro e rispetta i limiti", () => {
    const v = { x: 0, y: 0, zoom: 1 };
    expect(zoomAt(v, 10, 10, 99).zoom).toBe(ZOOM_MAX);
    expect(v).toEqual({ x: 0, y: 0, zoom: 1 });
  });

  it("la vista passa da etl-store: la modifica notifica gli ascoltatori", () => {
    const store = storeWith();
    let n = 0;
    store.subscribe(() => n++);
    zoomIn(store, SIZE);
    expect(n).toBe(1);
    // e non entra nel registro né nella cronologia
    expect(store.getLog().some((e) => e.type === "setView")).toBe(false);
    expect(store.canUndo()).toBe(false);
  });
});

describe("minimappa", () => {
  it("contiene tutti i nodi e la porzione visibile nel riquadro 168 × 104", () => {
    const store = storeWith();
    const cards = Object.values(store.getState().graph.cards);
    const frame = minimapFrame(cards, store.getState().view, SIZE);
    for (const c of cards) {
      const l = frame.ox + (c.x - frame.x1) * frame.k;
      const t = frame.oy + (c.y - frame.y1) * frame.k;
      expect(l).toBeGreaterThanOrEqual(0);
      expect(t).toBeGreaterThanOrEqual(0);
      expect(l + CARD * frame.k).toBeLessThanOrEqual(168 + 1e-9);
      expect(t + CARD * frame.k).toBeLessThanOrEqual(104 + 1e-9);
    }
  });

  it("un clic sulla minimappa porta quel punto al centro dell'area", () => {
    const store = storeWith();
    const cards = Object.values(store.getState().graph.cards);
    const view = store.getState().view;
    const frame = minimapFrame(cards, view, SIZE);
    const next = viewFromMinimap(frame, view, SIZE, 84, 52);
    const center = toWorld(next, SIZE.w / 2, SIZE.h / 2);
    expect(center.x).toBeCloseTo(frame.x1 + (84 - frame.ox) / frame.k, 6);
    expect(center.y).toBeCloseTo(frame.y1 + (52 - frame.oy) / frame.k, 6);
  });
});
```

### `src/etl-canvas/actions.ts`

41 righe

```ts
/** Azioni sulla vista, applicate attraverso etl-store (`setView`). */
import type { Size } from "../etl-layout";
import type { EtlStore } from "../etl-store";
import { fitView, zoomAt, zoomCentered, ZOOM_STEP } from "./view";

/** "Adatta": inquadra tutti i nodi. */
export function fit(store: EtlStore, size: Size): void {
  store.dispatch({
    type: "setView",
    payload: fitView(Object.values(store.getState().graph.cards), size),
  });
}

export function zoomBy(store: EtlStore, size: Size, factor: number): void {
  const view = store.getState().view;
  store.dispatch({ type: "setView", payload: zoomCentered(view, size, view.zoom * factor) });
}

export function zoomIn(store: EtlStore, size: Size): void {
  zoomBy(store, size, ZOOM_STEP);
}

export function zoomOut(store: EtlStore, size: Size): void {
  zoomBy(store, size, 1 / ZOOM_STEP);
}

export function zoomReset(store: EtlStore, size: Size): void {
  store.dispatch({
    type: "setView",
    payload: zoomCentered(store.getState().view, size, 1),
  });
}

/** Zoom attorno al puntatore (`px`, `py` relativi all'area). */
export function zoomAtPoint(store: EtlStore, px: number, py: number, zoom: number): void {
  store.dispatch({
    type: "setView",
    payload: zoomAt(store.getState().view, px, py, zoom),
  });
}
```

### `src/etl-canvas/canvas.css`

319 righe

```css
/*
 * Aspetto del canvas ETL (Fase 4a): nodi, cavi, controlli, minimappa.
 * Misure e classi del prototipo (docs/prototype/isa-fusion-prototype.html,
 * righe indicate); colori solo dai token di tokens.css. Tutte le regole
 * sono limitate a `.etl-canvas`.
 */
@import "@fontsource-variable/manrope/wght.css";
@import "./tokens.css";

.etl-canvas {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 520px;
  box-sizing: border-box;
  padding: 0;
  background: var(--ec-bg);
  color: var(--ec-ink);
  font-family: var(--ec-font);
  border-radius: var(--ec-r-stage);
}
.etl-canvas *,
.etl-canvas *::before,
.etl-canvas *::after {
  box-sizing: border-box;
  font-family: inherit;
}

/* riga 621-624 */
.etl-canvas .ec-stage {
  position: absolute;
  inset: 0;
  user-select: none;
  touch-action: none;
  border-radius: var(--ec-r-stage);
  background: var(--ec-stage);
  overflow: hidden;
}
.etl-canvas .ec-stage.ec-pannable {
  cursor: grab;
}
.etl-canvas .ec-stage.ec-panning {
  cursor: grabbing;
}

/* righe 3-4 di .world e .links (righe 205-206) */
.etl-canvas .ec-world {
  position: absolute;
  left: 0;
  top: 0;
  transform-origin: 0 0;
}
.etl-canvas .ec-links {
  position: absolute;
  inset: 0;
  overflow: visible;
  pointer-events: none;
}

/* nodo — riga 626 */
.etl-canvas .ec-card {
  position: absolute;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  width: 88px;
  z-index: 2;
}
.etl-canvas .ec-icon-wrap {
  position: relative;
  width: 88px;
  height: 88px;
  border-radius: var(--ec-r-op);
  background: var(--ec-node-op);
  color: var(--ec-node-op-ink);
  box-shadow: inset 0 0 0 1.5px var(--ec-node-op-border);
  display: grid;
  place-content: center;
  justify-items: center;
  align-items: center;
  gap: 6px;
  padding: 7px;
}
.etl-canvas .ec-dataset .ec-icon-wrap {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
  border-radius: var(--ec-r-fill);
  box-shadow: none;
}
.etl-canvas .ec-output .ec-icon-wrap {
  opacity: var(--ec-output-opacity);
}
.etl-canvas .ec-selected .ec-icon-wrap {
  box-shadow: 0 0 0 3px var(--ec-select);
}
.etl-canvas .ec-icon-wrap svg {
  display: block;
  flex-shrink: 0;
}
.etl-canvas .ec-count-1 {
  grid-template-columns: repeat(1, auto);
}
.etl-canvas .ec-count-2 {
  grid-template-columns: repeat(2, auto);
}
.etl-canvas .ec-count-3,
.etl-canvas .ec-count-6,
.etl-canvas .ec-count-many {
  grid-template-columns: repeat(3, auto);
}
.etl-canvas .ec-count-1 svg {
  width: 26px;
  height: 26px;
}
.etl-canvas .ec-count-2 svg {
  width: 20px;
  height: 20px;
}
.etl-canvas .ec-count-3 svg {
  width: 17px;
  height: 17px;
}
.etl-canvas .ec-count-6 svg {
  width: 15px;
  height: 15px;
}
.etl-canvas .ec-count-many svg {
  width: 12px;
  height: 12px;
}

/* output parziale: una fetta per tabella attesa — righe 656-669 */
.etl-canvas .ec-dataset .ec-icon-wrap.ec-split {
  padding: 0;
  display: flex;
  overflow: hidden;
  background: var(--ec-split-bg);
  gap: 0;
}
.etl-canvas .ec-split .ec-slice {
  flex: 1 1 0;
  min-width: 0;
  height: 100%;
  padding: 2px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.etl-canvas .ec-split .ec-slice.ec-slice-full {
  background: var(--ec-node-fill);
  color: var(--ec-node-fill-ink);
}
.etl-canvas .ec-split .ec-slice.ec-slice-empty {
  background: var(--ec-split-empty);
  color: var(--ec-split-empty-ink);
}
.etl-canvas .ec-split .ec-slice + .ec-slice {
  border-left: 1.5px dashed var(--ec-split-line);
}
.etl-canvas .ec-split .ec-slice svg {
  width: 100%;
  height: auto;
  max-width: 22px;
  max-height: 100%;
}
.etl-canvas .ec-split .ec-slice.ec-slice-empty svg {
  opacity: 0.85;
}

/* etichetta — righe 680-683 */
.etl-canvas .ec-label {
  font-size: 10.5px;
  font-weight: 700;
  text-align: center;
  line-height: 1.25;
  color: var(--ec-ink);
  max-width: 96px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  padding: 0 2px;
}
.etl-canvas .ec-dataset .ec-label {
  color: var(--ec-accent-text);
}
.etl-canvas .ec-partial .ec-label {
  color: var(--ec-muted);
  font-style: italic;
}

/* indicatore ambra — righe 183-187 */
.etl-canvas .ec-state-dot {
  position: absolute;
  right: -4px;
  bottom: -4px;
  width: 13px;
  height: 13px;
  border-radius: 999px;
  background: var(--ec-warn);
  border: 2.5px solid var(--ec-warn-ring);
  z-index: 4;
}

/* cavi — righe 1404-1408 */
.etl-canvas .ec-link {
  fill: none;
  stroke: var(--ec-link);
  stroke-width: 2.1;
  stroke-linecap: round;
  stroke-linejoin: round;
}
.etl-canvas .ec-link-dot-ds {
  fill: var(--ec-link-dot-fill);
}
.etl-canvas .ec-link-dot-op {
  fill: var(--ec-link-dot-op);
}

/* controlli di zoom — righe 138-150 */
.etl-canvas .ec-zoom {
  position: absolute;
  right: 12px;
  bottom: 12px;
  z-index: 15;
  display: flex;
  align-items: center;
  gap: 2px;
  padding: 4px;
  border-radius: var(--ec-r-pill);
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
}
.etl-canvas .ec-zoom button {
  all: unset;
  cursor: pointer;
  min-width: 28px;
  height: 28px;
  padding: 0 8px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--ec-r-pill);
  font-family: var(--ec-font);
  font-size: 12.5px;
  font-weight: 700;
  color: var(--ec-ink);
}
.etl-canvas .ec-zoom button:hover {
  background: var(--ec-accent-soft);
  color: var(--ec-accent-text);
}
.etl-canvas .ec-zoom button:focus-visible {
  outline: 2px solid var(--ec-accent-text);
  outline-offset: 1px;
}
.etl-canvas .ec-zoom .ec-fit {
  color: var(--ec-accent-text);
}

/* minimappa — righe 152-161 */
.etl-canvas .ec-minimap {
  position: absolute;
  left: 12px;
  bottom: 12px;
  width: 168px;
  height: 104px;
  z-index: 15;
  border-radius: var(--ec-r-minimap);
  background: var(--ec-surface-strong);
  backdrop-filter: blur(var(--ec-glass-blur));
  border: 1px solid var(--ec-panel-border);
  box-shadow: var(--ec-glass-shadow);
  overflow: hidden;
  cursor: pointer;
}
.etl-canvas .ec-mm-node {
  position: absolute;
  border-radius: 2px;
  background: var(--ec-mm-node);
}
.etl-canvas .ec-mm-node.ec-ds {
  background: var(--ec-mm-node-ds);
}
.etl-canvas .ec-mm-view {
  position: absolute;
  border: 1.5px solid var(--ec-mm-view-line);
  border-radius: 4px;
  background: var(--ec-mm-view-bg);
  pointer-events: none;
}

/* stato vuoto */
.etl-canvas .ec-empty {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  text-align: center;
  pointer-events: none;
  z-index: 1;
}
.etl-canvas .ec-empty-title {
  font-size: 15px;
  font-weight: 800;
  color: var(--ec-ink);
}
.etl-canvas .ec-empty-text {
  font-size: 12.5px;
  font-weight: 600;
  color: var(--ec-empty-ink);
}
```

### `src/etl-canvas/contrast.ts`

111 righe

```ts
/**
 * Contrasto WCAG tra i token di `tokens.css`. Usato dai test e da
 * `scripts/contrast-fase4.mjs` per la tabella del report.
 */
export type Rgba = readonly [number, number, number, number];

export function parseColor(value: string): Rgba {
  const v = value.trim().toLowerCase();
  const hex = /^#([0-9a-f]{3}|[0-9a-f]{6})$/.exec(v);
  if (hex) {
    const h = hex[1] as string;
    const full = h.length === 3 ? [...h].map((c) => c + c).join("") : h;
    return [
      parseInt(full.slice(0, 2), 16),
      parseInt(full.slice(2, 4), 16),
      parseInt(full.slice(4, 6), 16),
      1,
    ];
  }
  const rgba = /^rgba?\(([^)]+)\)$/.exec(v);
  if (rgba) {
    const p = (rgba[1] as string).split(",").map((s) => parseFloat(s));
    return [p[0] ?? 0, p[1] ?? 0, p[2] ?? 0, p[3] ?? 1];
  }
  throw new Error(`colore non riconosciuto: ${value}`);
}

/** Sovrappone `top` (con trasparenza) a `bottom` (opaco). */
export function over(top: Rgba, bottom: Rgba): Rgba {
  const a = top[3];
  return [
    top[0] * a + bottom[0] * (1 - a),
    top[1] * a + bottom[1] * (1 - a),
    top[2] * a + bottom[2] * (1 - a),
    1,
  ];
}

function lin(c: number): number {
  const s = c / 255;
  return s <= 0.03928 ? s / 12.92 : ((s + 0.055) / 1.055) ** 2.4;
}

export function luminance(c: Rgba): number {
  return 0.2126 * lin(c[0]) + 0.7152 * lin(c[1]) + 0.0722 * lin(c[2]);
}

export function contrast(a: Rgba, b: Rgba): number {
  const la = luminance(a);
  const lb = luminance(b);
  return (Math.max(la, lb) + 0.05) / (Math.min(la, lb) + 0.05);
}

/** Legge i token di un tema da tokens.css: `dark` = blocco `.dark .etl-canvas`. */
export function readTokens(css: string, theme: "light" | "dark"): Record<string, string> {
  const selector = theme === "dark" ? ".dark .etl-canvas" : ".etl-canvas";
  const start = css.indexOf(`\n${selector} {`);
  if (start < 0) throw new Error(`blocco non trovato: ${selector}`);
  const end = css.indexOf("}", start);
  const body = css.slice(start, end);
  const out: Record<string, string> = {};
  for (const m of body.matchAll(/(--ec-[a-z0-9-]+):\s*([^;]+);/g)) {
    out[m[1] as string] = (m[2] as string).trim();
  }
  return out;
}

export interface ContrastPair {
  readonly role: string;
  /** Token in primo piano e token di sfondo. */
  readonly fg: string;
  readonly bg: string;
  /** 4.5 per il testo, 3 per gli elementi non testuali. */
  readonly min: number;
}

/**
 * Coppie da verificare. Il fondo del canvas è `--ec-bg` con sopra
 * `--ec-stage`; il vetro è `--ec-surface-strong` sopra il canvas.
 */
export const PAIRS: readonly ContrastPair[] = [
  { role: "Etichetta dataset/output", fg: "--ec-accent-text", bg: "canvas", min: 4.5 },
  { role: "Etichetta lavorazione", fg: "--ec-ink", bg: "canvas", min: 4.5 },
  { role: "Etichetta output parziale", fg: "--ec-muted", bg: "canvas", min: 4.5 },
  { role: "Icona su nodo pieno", fg: "--ec-node-fill-ink", bg: "--ec-node-fill", min: 3 },
  { role: "Icona su nodo lavorazione", fg: "--ec-node-op-ink", bg: "--ec-node-op", min: 3 },
  { role: "Nodo pieno su canvas", fg: "--ec-node-fill", bg: "canvas", min: 3 },
  { role: "Bordo del nodo lavorazione", fg: "--ec-node-op-border", bg: "canvas", min: 3 },
  { role: "Icona fetta vuota", fg: "--ec-split-empty-ink", bg: "--ec-split-empty", min: 3 },
  { role: "Cavo", fg: "--ec-link", bg: "canvas", min: 3 },
  { role: "Indicatore ambra", fg: "--ec-warn", bg: "canvas", min: 3 },
  { role: "Contorno di selezione", fg: "--ec-select", bg: "canvas", min: 3 },
  { role: "Testo dei controlli di zoom", fg: "--ec-ink", bg: "glass", min: 4.5 },
  { role: "«Adatta»", fg: "--ec-accent-text", bg: "glass", min: 4.5 },
  { role: "Nodo nella minimappa", fg: "--ec-mm-node", bg: "glass", min: 3 },
  { role: "Nodo dataset nella minimappa", fg: "--ec-mm-node-ds", bg: "glass", min: 3 },
  { role: "Riquadro visibile (minimappa)", fg: "--ec-mm-view-line", bg: "glass", min: 3 },
  { role: "Testo dello stato vuoto", fg: "--ec-empty-ink", bg: "canvas", min: 4.5 },
  { role: "Titolo dello stato vuoto", fg: "--ec-ink", bg: "canvas", min: 4.5 },
];

export function measure(tokens: Record<string, string>, pair: ContrastPair): number {
  const get = (name: string): Rgba => parseColor(tokens[name] ?? "#000");
  const canvas = over(get("--ec-stage"), over(get("--ec-bg"), [255, 255, 255, 1]));
  const glass = over(get("--ec-surface-strong"), canvas);
  const bg =
    pair.bg === "canvas" ? canvas : pair.bg === "glass" ? glass : over(get(pair.bg), canvas);
  const fg = over(get(pair.fg), bg);
  return contrast(fg, bg);
}
```

### `src/etl-canvas/icons.tsx`

20 righe

```tsx
import { EMPTY_SLOT_ICON, ICONS } from "../etl-core";
import type { ComponentId } from "../etl-core";

/** Icona del catalogo di etl-core (prototipo `svgTag`, righe 979-981). I tracciati sono costanti del catalogo, mai dati dell'utente. */
export function Icon(props: { id: ComponentId | "empty" }) {
  const inner = props.id === "empty" ? EMPTY_SLOT_ICON : ICONS[props.id];
  return (
    <svg
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth={2}
      strokeLinecap="round"
      strokeLinejoin="round"
      aria-hidden="true"
      dangerouslySetInnerHTML={{ __html: inner }}
    />
  );
}
```

### `src/etl-canvas/index.ts`

11 righe

```ts
/**
 * etl-canvas — Fase 4a: resa visiva del canvas ETL. Importa da etl-core,
 * etl-layout ed etl-store; nessuno di questi importa da qui.
 */
export { EtlCanvas, CanvasSurface } from "./EtlCanvas";
export type { CanvasSurfaceProps } from "./EtlCanvas";
export { prototypeScene } from "./seed";
export { fit, zoomIn, zoomOut, zoomReset, zoomAtPoint } from "./actions";
export { fitView, zoomAt, minimapFrame, bounds } from "./view";
export { nodeView, countClass, isPartial, slicesOf } from "./model";
```

### `src/etl-canvas/model.ts`

76 righe

```ts
/**
 * Dal grafo di etl-core a ciò che il canvas disegna: classi e icone di ogni
 * nodo, fette di un output parziale. Funzioni pure.
 */
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

export interface NodeView {
  readonly id: string;
  readonly card: Card;
  /** Classi del contenitore del nodo. */
  readonly className: string;
  readonly iconClass: string;
  readonly partial: boolean;
  readonly slices: readonly Slice[];
  readonly icons: readonly ComponentId[];
  readonly warn: string | null;
  readonly selected: boolean;
}

/** Classi del nodo (prototipo `createCardEl`, righe 1003-1004, più `partial`, `warn`, `selected`). */
export function nodeView(card: Card, warn: string | null, selected: boolean): NodeView {
  const combined = card.kind === "op" && card.components.length > 1;
  const partial = isPartial(card);
  const classes = ["ec-card"];
  if (card.kind === "dataset") classes.push("ec-dataset");
  if (card.kind === "dataset" && card.isOutput) classes.push("ec-output");
  if (combined) classes.push("ec-combined");
  if (partial) classes.push("ec-partial");
  if (warn) classes.push("ec-warn");
  if (selected) classes.push("ec-selected");
  const icons: ComponentId[] =
    card.kind === "dataset" && card.isOutput ? ["dataset"] : [...card.components];
  return {
    id: card.id,
    card,
    className: classes.join(" "),
    iconClass: partial ? "ec-icon-wrap ec-split" : `ec-icon-wrap ec-${countClass(icons.length)}`,
    partial,
    slices: partial ? slicesOf(card) : [],
    icons,
    warn,
    selected,
  };
}
```

### `src/etl-canvas/seed.ts`

68 righe

```ts
/**
 * Scena iniziale del prototipo (`init`, righe 5070-5083 di
 * docs/prototype/isa-fusion-prototype.html): Vendite 2026, Filtra Righe,
 * Unisci, Ordina, Esporta, con posizioni identiche e nessun cavo. Serve
 * solo in sviluppo (`?canvas=v2&seed=prototype`).
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

