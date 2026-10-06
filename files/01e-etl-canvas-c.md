# 01e-etl-canvas-c.md

File in questo blocco:

- `src/etl-canvas/__tests__/controlbar.test.tsx`
- `src/etl-canvas/__tests__/drop.test.ts`
- `src/etl-canvas/__tests__/engine.test.ts`
- `src/etl-canvas/__tests__/fake-env.ts`
- `src/etl-canvas/__tests__/flow.test.ts`
- `src/etl-canvas/__tests__/gesture-render.test.ts`
- `src/etl-canvas/__tests__/helpers.ts`
- `src/etl-canvas/__tests__/inspector-logic.test.ts`
- `src/etl-canvas/__tests__/inspector-rules.test.ts`

---

### `src/etl-canvas/__tests__/controlbar.test.tsx`

145 righe

```tsx
import { readFileSync, readdirSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { createEtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
import { ControlBar } from "../panels/ControlBar";
import { storeWith } from "./helpers";

const root = resolve(dirname(fileURLToPath(import.meta.url)), "..");

function bar(store: ReturnType<typeof createEtlStore>): string {
  return renderToStaticMarkup(
    createElement(ControlBar, {
      store,
      controller: createInteractionController(store),
      area: { w: 800, h: 500 },
    }),
  );
}

describe("barra dei controlli", () => {
  it("ha Libero/Organizzato, Riordina, Annulla, Ripristina e Svuota", () => {
    const markup = bar(storeWith());
    for (const label of ["Libero", "Organizzato", "Riordina", "Annulla", "Ripristina", "Svuota"]) {
      expect(markup).toContain(label);
    }
    expect(markup).toContain('role="toolbar"');
    // i suggerimenti riportano le scorciatoie
    expect(markup).toContain("Cmd/Ctrl+Z");
    expect(markup).toContain("Cmd/Ctrl+Maiusc+Z");
  });

  it("«Reimposta» e «Funzionalità» non ci sono, né in barra né nei file dei pannelli", () => {
    expect(bar(storeWith())).not.toMatch(/Reimposta|Funzionalit/i);
    const dir = resolve(root, "panels");
    for (const f of readdirSync(dir).filter((n) => /\.(tsx?|css)$/.test(n))) {
      const text = readFileSync(resolve(dir, f), "utf8");
      // i commenti possono citare il pulsante del prototipo che non si porta
      const code = text.replace(/\/\*[\s\S]*?\*\//g, "").replace(/\/\/.*$/gm, "");
      expect(code, f).not.toMatch(/Reimposta|Funzionalit/i);
    }
  });

  it("Annulla e Ripristina sono disabilitati senza cronologia; Svuota e Riordina senza nodi", () => {
    const empty = bar(createEtlStore());
    expect(empty).toMatch(/aria-label="Annulla"[^>]*disabled/);
    expect(empty).toMatch(/aria-label="Ripristina"[^>]*disabled/);
    expect(empty).toMatch(/aria-label="Svuota il canvas"[^>]*disabled/);
    const store = storeWith();
    store.dispatch({ type: "moveNodes", payload: { ids: ["op-join"], dx: 4, dy: 0 } });
    const full = bar(store);
    expect(full).not.toMatch(/aria-label="Annulla"[^>]*disabled/);
    expect(full).toMatch(/aria-label="Ripristina"[^>]*disabled/);
    expect(full).not.toMatch(/aria-label="Svuota il canvas"[^>]*disabled/);
  });

  it("la modalità attiva è indicata con aria-pressed", () => {
    const store = storeWith();
    store.dispatch({ type: "setMode", payload: { mode: "grid" } });
    const markup = bar(store);
    expect(markup).toMatch(/aria-pressed="true"[^>]*>Organizzato/);
    expect(markup).toMatch(/aria-pressed="false"[^>]*>Libero/);
  });
});

describe("Svuota: conferma e comando", () => {
  it("la finestra riporta il testo previsto e annullare non cambia nulla", () => {
    const store = storeWith();
    const c = createInteractionController(store);
    const graph = store.getState().graph;
    const logLength = store.getLog().length;
    c.requestClearAll();
    const confirm = c.getUi().confirm;
    expect(confirm?.kind).toBe("clear");
    expect(confirm?.text).toBe(
      "Eliminare tutti i nodi e i collegamenti? Puoi annullare con Cmd/Ctrl+Z.",
    );
    // tutti i nodi sono segnati come destinati a sparire
    expect(confirm?.removed.length).toBe(Object.keys(graph.cards).length);
    c.cancelConfirm();
    expect(c.getUi().confirm).toBeNull();
    expect(store.getState().graph).toBe(graph);
    expect(store.getLog().length).toBe(logLength);
    expect(store.historySize().past).toBe(0);
  });

  it("Esc annulla la conferma", () => {
    const store = storeWith();
    const c = createInteractionController(store);
    c.requestClearAll();
    c.key({ key: "Escape" });
    expect(c.getUi().confirm).toBeNull();
    expect(Object.keys(store.getState().graph.cards).length).toBeGreaterThan(0);
  });

  it("con la conferma aperta i comandi da tastiera del canvas non agiscono", () => {
    const store = storeWith();
    const c = createInteractionController(store);
    store.dispatch({ type: "select", payload: { ids: ["op-join"] } });
    c.requestClearAll();
    expect(c.key({ key: "Delete" })).toBe(false);
    expect(c.key({ key: "z", metaKey: true })).toBe(false);
    expect(store.getState().graph.cards["op-join"]).toBeDefined();
  });

  it("confermato: un solo passo di cronologia, una voce nel registro, libreria intatta; Annulla ripristina tutto", () => {
    const store = storeWith();
    store.dispatch({
      type: "loadDataset",
      payload: {
        name: "a",
        path: "a.csv",
        columns: [{ name: "x", type: "integer", values: [] }] as never,
        rows: 1,
      },
    });
    const c = createInteractionController(store);
    const before = store.getState();
    const past = store.historySize().past;
    const logLength = store.getLog().length;
    c.requestClearAll();
    expect(c.confirmDelete()).toEqual({ ok: true });
    const s = store.getState();
    expect(Object.keys(s.graph.cards)).toEqual([]);
    expect(s.graph.links).toEqual([]);
    expect(s.library).toEqual(before.library);
    expect(store.historySize().past).toBe(past + 1);
    expect(store.getLog().length).toBe(logLength + 1);
    expect(store.getLog().at(-1)?.type).toBe("clearAll");
    store.undo();
    expect(store.getState().graph).toEqual(before.graph);
    expect(store.getState().library).toEqual(before.library);
  });

  it("su un canvas vuoto non si chiede nulla", () => {
    const store = createEtlStore();
    const c = createInteractionController(store);
    c.requestClearAll();
    expect(c.getUi().confirm).toBeNull();
  });
});
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

### `src/etl-canvas/__tests__/gesture-render.test.ts`

53 righe

```ts
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { Links } from "../Links";
import { nodeView } from "../model";
import { html, nodeHtml, storeWith } from "./helpers";

describe("a riposo il canvas non mostra nessun gesto", () => {
  const markup = html(storeWith());

  it("nessuna classe di gesto, nessun cavo provvisorio, riquadro, conferma o suggerimento", () => {
    for (const cls of ["ec-dragging", "ec-drop-", "ec-doomed", "ec-link-hot"]) {
      expect(markup).not.toContain(cls);
    }
    for (const id of ["ec-temp-link", "ec-marquee", "ec-confirm"]) {
      expect(markup).not.toContain(id);
    }
    expect(markup).not.toContain("ec-hint");
  });

  it("ogni nodo ha le quattro porte, nascoste a riposo dallo stile", () => {
    const n = nodeHtml(markup, "op-join");
    for (const side of ["t", "b", "l", "r"]) expect(n).toContain(`data-port="${side}"`);
    expect(n.match(/class="ec-port /g)).toHaveLength(4);
    expect(n).toContain('aria-hidden="true"');
  });
});

describe("classi di gesto sul nodo", () => {
  const card = storeWith().getState().graph.cards["op-sort"]!;
  it("trascinamento, esiti, eliminazione", () => {
    expect(nodeView(card, null, false, { dragging: true }).className).toContain("ec-dragging");
    for (const o of ["merge", "link", "link-reverse", "displace", "reject"] as const) {
      expect(nodeView(card, null, false, { drop: o }).className).toContain(`ec-drop-${o}`);
    }
    expect(nodeView(card, null, false, { doomed: true }).className).toContain("ec-doomed");
    expect(nodeView(card, null, false).className).not.toMatch(/ec-(dragging|drop|doomed)/);
  });
});

describe("cavo da inserire", () => {
  it("il cavo indicato prende la classe ec-link-hot, gli altri no", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const graph = store.getState().graph;
    const routes = store.getRoutes();
    const out = renderToStaticMarkup(createElement(Links, { graph, routes, hot: "ds1|op-join" }));
    expect(out.match(/ec-link ec-link-hot/g)).toHaveLength(1);
    const none = renderToStaticMarkup(createElement(Links, { graph, routes }));
    expect(none).not.toContain("ec-link-hot");
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

### `src/etl-canvas/__tests__/inspector-logic.test.ts`

157 righe

```ts
import { describe, expect, it } from "vitest";
import {
  addTokens,
  addVisible,
  addVisibleValues,
  canonicalName,
  canonicalValue,
  clampActive,
  dropIndex,
  filterByQuery,
  fold,
  moveItem,
  moveTarget,
  nextActive,
  pendingTokens,
  removeVisible,
  removeVisibleValues,
  reorderIndex,
  toggleColumn,
  toggleValue,
  withValues,
} from "../inspector/logic";

const L = (...labels: string[]) => labels.map((label) => ({ label }));

describe("ricerca e navigazione", () => {
  it("la ricerca non distingue maiuscole e accenti", () => {
    expect(fold("Città")).toBe("citta");
    expect(filterByQuery(L("Roma", "Città", "Torino"), "CITTA").map((x) => x.label)).toEqual([
      "Città",
    ]);
    expect(filterByQuery(L("a", "b"), "  ").length).toBe(2);
    expect(filterByQuery(L("a"), "zzz")).toEqual([]);
  });

  it("frecce, Home e Fine: si fermano alle estremità, nessun giro", () => {
    expect(nextActive(-1, 4, "ArrowDown")).toBe(0);
    expect(nextActive(0, 4, "ArrowDown")).toBe(1);
    expect(nextActive(3, 4, "ArrowDown")).toBe(3);
    expect(nextActive(-1, 4, "ArrowUp")).toBe(3);
    expect(nextActive(0, 4, "ArrowUp")).toBe(0);
    expect(nextActive(2, 4, "Home")).toBe(0);
    expect(nextActive(1, 4, "End")).toBe(3);
    expect(nextActive(0, 0, "ArrowDown")).toBe(-1);
  });

  it("la voce attiva resta valida quando l'elenco si accorcia", () => {
    expect(clampActive(5, 3)).toBe(2);
    expect(clampActive(-1, 3)).toBe(0);
    expect(clampActive(2, 0)).toBe(-1);
  });
});

describe("colonne scelte: ordine di scelta, Tutte/Nessuna sulle visibili", () => {
  it("una colonna spuntata va in fondo, una tolta lascia l'ordine delle altre", () => {
    let s: string[] = [];
    for (const c of ["c", "a", "b"]) s = toggleColumn(s, c);
    expect(s).toEqual(["c", "a", "b"]);
    expect(toggleColumn(s, "a")).toEqual(["c", "b"]);
  });

  it("«Tutte» aggiunge solo le visibili non scelte; «Nessuna» toglie solo le visibili", () => {
    const selected = ["z", "b"];
    const visible = ["a", "b", "c"]; // dopo un filtro di ricerca
    expect(addVisible(selected, visible)).toEqual(["z", "b", "a", "c"]);
    expect(removeVisible(["z", "b", "a"], visible)).toEqual(["z"]);
    expect(removeVisible(selected, [])).toEqual(selected);
    expect(addVisible(selected, [])).toEqual(selected);
  });

  it("riordino: moveItem e Alt+frecce", () => {
    expect(moveItem(["a", "b", "c"], 0, 2)).toEqual(["b", "c", "a"]);
    expect(moveItem(["a", "b", "c"], 2, 0)).toEqual(["c", "a", "b"]);
    expect(moveItem(["a", "b"], 0, 5)).toEqual(["a", "b"]);
    expect(moveTarget(1, 3, "ArrowLeft")).toBe(0);
    expect(moveTarget(1, 3, "ArrowRight")).toBe(2);
    expect(moveTarget(0, 3, "ArrowLeft")).toBeNull();
    expect(moveTarget(2, 3, "ArrowRight")).toBeNull();
    expect(moveTarget(1, 3, "ArrowUp")).toBe(0);
    expect(moveTarget(1, 3, "x")).toBeNull();
  });

  it("trascinamento: l'indice del segnaposto più vicino", () => {
    expect(dropIndex([10, 60, 130], 70)).toBe(1);
    expect(dropIndex([10, 60, 130], -50)).toBe(0);
    expect(dropIndex([10, 60, 130], 999)).toBe(2);
  });

  it("il nome scritto prende la grafia dei dati", () => {
    expect(canonicalName(["Importo", "Regione"], " importo ")).toBe("Importo");
    expect(canonicalName(["Importo"], "nuova")).toBe("nuova");
  });

  it("le funzioni non modificano l'ingresso", () => {
    const sel = Object.freeze(["a", "b"]);
    expect(() => toggleColumn(sel, "c")).not.toThrow();
    expect(() => addVisible(sel, Object.freeze(["x"]))).not.toThrow();
    expect(() => removeVisible(sel, Object.freeze(["a"]))).not.toThrow();
    expect(() => moveItem(sel, 0, 1)).not.toThrow();
  });
});

describe("valori scelti", () => {
  const domain = ["Nord", "Sud", "Centro"];

  it("la grafia dei dati prevale, anche con maiuscole diverse", () => {
    expect(canonicalValue(domain, "nord")).toBe("Nord");
    expect(canonicalValue(domain, "CENTRO")).toBe("Centro");
    expect(canonicalValue(domain, "Isole")).toBe("Isole");
    expect(addTokens([], domain, "nord, SUD")).toEqual(["Nord", "Sud"]);
    expect(addTokens(["Nord"], domain, "nord")).toEqual(["Nord"]);
  });

  it("incolla più valori separati da virgola, punto e virgola, barra verticale o a capo", () => {
    expect(addTokens([], [], "a, b;c|d\ne")).toEqual(["a", "b", "c", "d", "e"]);
    expect(addTokens(["a"], [], "a,,b , ")).toEqual(["a", "b"]);
  });

  it("«+ Aggiungi» compare per ciò che non esiste; un solo valore già presente nei dati no", () => {
    expect(pendingTokens([], domain, "Isole")).toEqual(["Isole"]);
    expect(pendingTokens([], domain, "Isole, Mare")).toEqual(["Isole", "Mare"]);
    expect(pendingTokens([], domain, "nord")).toEqual([]);
    expect(pendingTokens(["Isole"], domain, "isole")).toEqual([]);
    expect(pendingTokens([], domain, "nord, Mare")).toEqual(["nord", "Mare"]);
    expect(pendingTokens([], domain, "   ")).toEqual([]);
  });

  it("«Tutti» e «Nessuno» sulle sole visibili", () => {
    expect(addVisibleValues(["x"], ["Nord", "Sud"])).toEqual(["x", "Nord", "Sud"]);
    expect(addVisibleValues(["Nord"], ["Nord", "Sud"])).toEqual(["Nord", "Sud"]);
    expect(removeVisibleValues(["x", "Nord", "Sud"], ["Nord", "Sud"])).toEqual(["x"]);
    expect(toggleValue(["a"], "b")).toEqual(["a", "b"]);
    expect(toggleValue(["a", "b"], "a")).toEqual(["b"]);
  });

  it("i valori stanno solo in `values`: il campo si riscrive in modalità elenco, senza testo", () => {
    const old = { mode: "manual" as const, values: ["a"], text: "b, c", sep: ";" };
    expect(withValues(old, ["a", "b"])).toEqual({
      mode: "list",
      values: ["a", "b"],
      text: "",
      sep: ";",
    });
    expect(withValues(undefined, [])).toEqual({ mode: "list", values: [], text: "", sep: "," });
  });
});

describe("riordino dei passaggi", () => {
  it("l'indice di arrivo segue il trascinamento ed è limitato all'elenco", () => {
    expect(reorderIndex(0, 0, 40, 4)).toBe(0);
    expect(reorderIndex(0, 85, 40, 4)).toBe(2);
    expect(reorderIndex(3, -85, 40, 4)).toBe(1);
    expect(reorderIndex(1, 9999, 40, 4)).toBe(3);
    expect(reorderIndex(1, -9999, 40, 4)).toBe(0);
  });
});
```

### `src/etl-canvas/__tests__/inspector-rules.test.ts`

185 righe

```ts
import { spawnSync } from "node:child_process";
import { readFileSync, readdirSync } from "node:fs";
import { dirname, join, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { describe, expect, it } from "vitest";
import { comboAction } from "../inspector/logic";

const here = dirname(fileURLToPath(import.meta.url));
const DIR = resolve(here, "../inspector");
const SCRIPT = resolve(here, "../../../scripts/check-tokens.mjs");
const files = readdirSync(DIR).filter((f) => /\.(tsx?|css)$/.test(f));
const code = (f: string) =>
  readFileSync(join(DIR, f), "utf8")
    .replace(/\/\*[\s\S]*?\*\//g, "")
    .replace(/^\s*\/\/.*$/gm, "")
    .replace(/([^:])\/\/.*$/gm, "$1");
const sources = files.filter((f) => /\.tsx?$/.test(f));
const components = files.filter((f) => f.endsWith(".tsx"));

describe("regole di forma dell'Inspector", () => {
  it("la cartella ha i file previsti", () => {
    for (const f of [
      "Inspector.tsx",
      "Header.tsx",
      "BlockedNotice.tsx",
      "StepList.tsx",
      "Field.tsx",
      "StyledSelect.tsx",
      "ColumnPicker.tsx",
      "ValuePicker.tsx",
      "MultiList.tsx",
      "menu.ts",
      "useActiveSchema.ts",
      "copy.ts",
      "inspector.css",
    ]) {
      expect(files, f).toContain(f);
    }
  });

  it("nessun elemento nativo: <select>, <option>, <datalist> e input number, date, time, color, range", () => {
    for (const f of files) {
      const text = code(f);
      expect(text, f).not.toMatch(/<select\b/i);
      expect(text, f).not.toMatch(/<option\b/i);
      expect(text, f).not.toMatch(/<datalist\b/i);
      expect(text, f).not.toMatch(
        /type\s*=\s*["'{`]*(number|date|datetime-local|time|month|week|color|range)\b/i,
      );
    }
  });

  it("nessun input di tipo checkbox o radio nativo visibile: le spunte sono disegnate", () => {
    for (const f of sources) {
      expect(code(f), f).not.toMatch(/type\s*=\s*["']?(checkbox|radio)\b/i);
    }
  });

  it("ogni menu passa dal componente Menu (portale): nessun role=listbox fuori dai tre selettori", () => {
    const withListbox = sources.filter((f) => /role="listbox"/.test(code(f))).sort();
    expect(withListbox).toEqual(["ColumnPicker.tsx", "StyledSelect.tsx", "ValuePicker.tsx"]);
    for (const f of withListbox) expect(code(f), f).toContain("<Menu");
    expect(code("Menu.tsx")).toContain("createPortal");
  });

  it("font-size, margin, padding e gap vengono tutti dai token (check-tokens esteso)", () => {
    for (const f of files.filter((n) => !n.endsWith(".css") || true)) {
      const r = spawnSync("node", [SCRIPT, "--check-file", join(DIR, f)], { encoding: "utf8" });
      expect(r.status, `${f}: ${r.stderr}`).toBe(0);
    }
  });

  it("nessun testo sotto 11px: i token della scala tipografica", () => {
    const css = readFileSync(resolve(here, "../../theme/layout-tokens.css"), "utf8");
    const sizes = [...css.matchAll(/--isa-fs-([a-z]+):\s*(\d+)px/g)].map(
      (m) => [m[1], Number(m[2])] as const,
    );
    expect(sizes.map(([n]) => n).sort()).toEqual([
      "help",
      "label",
      "overline",
      "summary",
      "title",
      "value",
    ]);
    for (const [name, px] of sizes) expect(px, name).toBeGreaterThanOrEqual(11);
    // il solo testo sotto 12px è la sopralinea
    expect(sizes.filter(([, px]) => px < 12).map(([n]) => n)).toEqual(["overline"]);
    expect(Object.fromEntries(sizes)).toMatchObject({
      title: 17,
      overline: 11,
      label: 12,
      value: 14,
      summary: 13,
      help: 12,
    });
  });

  it("la scala degli spazi è 4, 8, 12, 16, 20, 24, 32 px", () => {
    const css = readFileSync(resolve(here, "../../theme/layout-tokens.css"), "utf8");
    const space = [...css.matchAll(/--isa-space-(\d):\s*(\d+)px/g)].map((m) => [
      Number(m[1]),
      Number(m[2]),
    ]);
    expect(space).toEqual([
      [1, 4],
      [2, 8],
      [3, 12],
      [4, 16],
      [5, 20],
      [6, 24],
      [8, 32],
    ]);
  });

  it("nessuna dimensione del testo in px nei CSS dell'Inspector", () => {
    const css = code("inspector.css");
    for (const m of css.matchAll(/font-size:\s*([^;]+);/g)) {
      expect(m[1], m[0]).toMatch(/^var\(--isa-fs-[a-z]+\)$/);
    }
  });

  it("copy.ts è l'unica fonte dei testi: nessuna stringa italiana nei componenti", () => {
    // due parole di testo di seguito (non pezzi di una classe CSS: `ei-chip`, `data-x`)
    const sentence = /(?<![\w-])[A-Za-zÀ-ÿ]{3,}(?![\w-])[ ]+[A-Za-zÀ-ÿ]{2,}(?![\w-])/;
    for (const f of components) {
      const text = code(f);
      // testo tra i tag di JSX (i segmenti con codice sono generici e funzioni, non testo)
      for (const m of text.matchAll(/>([^<>{}();=]*[A-Za-zÀ-ÿ]{2,}[^<>{}();=]*)</g)) {
        const segment = m[1] ?? "";
        // un segmento con simboli di codice (confronti, operatori) non è testo
        if (!/^[A-Za-zÀ-ÿ0-9\s’'.,:!?…“”-]*$/.test(segment)) continue;
        expect(segment.trim(), `${f}: testo letterale «${segment}»`).toBe("");
      }
      // attributi di testo per le persone
      for (const m of text.matchAll(
        /\b(aria-label|title|placeholder|alt|aria-description)\s*=\s*"([^"]*)"/g,
      )) {
        expect(m[2], `${f}: ${m[1]}="${m[2]}"`).toBe("");
      }
      // stringhe di frase (parole italiane separate da spazi)
      for (const m of text.matchAll(/(["'`])((?:\\.|(?!\1)[^\\\n])*)\1/g)) {
        const lit = m[2] ?? "";
        if (
          /^[\w\s.:#[\]="'()>,*-]*$/.test(lit) &&
          !/[À-ÿ]/.test(lit) &&
          !/^[a-z-]+( [a-z-]+)*$/.test(lit)
        )
          continue;
        expect(sentence.test(lit), `${f}: «${lit}»`).toBe(false);
      }
    }
  });

  it("l'Inspector non importa flattenRows e non legge `row.column`", () => {
    for (const f of sources) {
      const text = code(f);
      expect(text, f).not.toMatch(/\bflattenRows\b/);
      expect(text, f).not.toMatch(/\.column\b/);
      expect(text, f).not.toMatch(/\[\s*["']column["']\s*\]/);
    }
  });

  it("tutti i colori, i raggi e le ombre vengono dai token (check-tokens)", () => {
    for (const f of files) {
      const text = code(f);
      expect(text, f).not.toMatch(/#[0-9a-fA-F]{3,8}\b|\brgba?\(|\bhsla?\(|\boklch\(/);
    }
  });
});

describe("combobox: tasti del campo di ricerca", () => {
  it("frecce, Home e Fine muovono la voce attiva; Invio conferma; Esc chiude; il resto è digitazione", () => {
    expect(comboAction("ArrowDown", 1, 5)).toEqual({ kind: "move", to: 2 });
    expect(comboAction("ArrowUp", 1, 5)).toEqual({ kind: "move", to: 0 });
    expect(comboAction("Home", 3, 5)).toEqual({ kind: "move", to: 0 });
    expect(comboAction("End", 1, 5)).toEqual({ kind: "move", to: 4 });
    expect(comboAction("Enter", 2, 5)).toEqual({ kind: "commit" });
    expect(comboAction("Escape", 2, 5)).toEqual({ kind: "close" });
    expect(comboAction("a", 2, 5)).toEqual({ kind: "none" });
    expect(comboAction("Backspace", 2, 5)).toEqual({ kind: "none" });
    expect(comboAction("ArrowDown", -1, 0)).toEqual({ kind: "move", to: -1 });
  });
});
```

