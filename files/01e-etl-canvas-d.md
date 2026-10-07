# 01e-etl-canvas-d.md

File in questo blocco:

- `src/etl-canvas/__tests__/engine.test.ts`
- `src/etl-canvas/__tests__/fake-env.ts`
- `src/etl-canvas/__tests__/flow.test.ts`
- `src/etl-canvas/__tests__/gesture-render.test.ts`
- `src/etl-canvas/__tests__/helpers.ts`
- `src/etl-canvas/__tests__/inspector-logic.test.ts`
- `src/etl-canvas/__tests__/inspector-rules.test.ts`
- `src/etl-canvas/__tests__/inspector.test.tsx`

---

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

199 righe

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
      "ConditionList.tsx",
      "ConnectorSelect.tsx",
      "GroupFrame.tsx",
      "ExpressionPreview.tsx",
      "FilterCondition.tsx",
      "JoinCondition.tsx",
      "JoinSettings.tsx",
      "Segmented.tsx",
      "Columns3.tsx",
      "masterDetail.ts",
      "ListRow.tsx",
      "conditions.ts",
      "joinSides.ts",
      "joinKeys.ts",
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

### `src/etl-canvas/__tests__/inspector.test.tsx`

500 righe

```tsx
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { MULTI_DEFS, createValuesField, defaultParams, outputOf } from "../../etl-core";
import type { Card, ColumnDef, ComponentId, OperationType, Params } from "../../etl-core";
import { createEtlStore, nodeStates } from "../../etl-store";
import type { EtlStore } from "../../etl-store";
import { copy } from "../inspector/copy";
import { Inspector } from "../inspector/Inspector";
import { prototypeScene } from "../seed";

/** Una scena con `op-sort` trasformato in una lavorazione del tipo (o dei tipi) dati. */
function sceneWith(components: ComponentId[], params?: Params[]): EtlStore {
  const scene = prototypeScene();
  const sort = scene.graph.cards["op-sort"] as Card;
  const card: Card = {
    ...sort,
    components,
    params: params ?? components.map((c) => defaultParams(c)),
  };
  return createEtlStore({
    initial: {
      ...scene,
      graph: { ...scene.graph, cards: { ...scene.graph.cards, "op-sort": card } },
    },
  });
}

function connect(store: EtlStore, from = "ds1", to = "op-sort"): void {
  expect(store.dispatch({ type: "connect", payload: { from, to } })).toEqual({ ok: true });
}

function render(store: EtlStore, node: string, step = 0): string {
  store.dispatch({ type: "inspect", payload: { node, step } });
  return renderToStaticMarkup(createElement(Inspector, { store }));
}

const count = (markup: string, needle: string) => markup.split(needle).length - 1;

describe("Inspector: tipi di nodo", () => {
  it("senza nodo: nessun nodo selezionato", () => {
    const store = sceneWith(["sort"]);
    expect(renderToStaticMarkup(createElement(Inspector, { store }))).toContain(
      copy.emptyInspector,
    );
  });

  it("dataset: nome, origine e colonne in sola lettura con il tipo", () => {
    const m = render(sceneWith(["sort"]), "ds1");
    expect(m).toContain('data-kind="dataset"');
    expect(m).toContain(`>${copy.kindSource}<`);
    expect(m).toContain('value="Vendite 2026"');
    expect(m).toContain("Percorso o tabella");
    expect(m).toContain('value="vendite_2026.csv"');
    expect(count(m, 'class="ei-colrow"')).toBe(8);
    expect(m).toContain(">integer<");
    expect(m).toContain(">numerico<");
    expect(m).toContain(">data<");
    // le colonne non si modificano: nessun selettore di colonne
    expect(m).not.toContain('data-picker="columns"');
  });

  it("output: nota del risultato e colonne dello schema del produttore", () => {
    const store = sceneWith(["sort"]);
    connect(store);
    const out = outputOf(store.getState().graph, "op-sort") as string;
    const m = render(store, out);
    expect(m).toContain('data-kind="output"');
    expect(m).toContain(`>${copy.kindResult}<`);
    expect(m).toContain("Risultato generato da");
    expect(count(m, 'class="ei-colrow"')).toBe(8);
  });

  it("lavorazione senza tabella in ingresso: stato bloccato, nessun campo", () => {
    const m = render(sceneWith(["sort"]), "op-sort");
    expect(m).toContain('data-testid="ei-blocked"');
    expect(m).toContain(copy.lockedText);
    expect(m).not.toContain("ei-fieldgroup");
    expect(m).not.toContain("data-picker=");
  });

  it("un join senza ingressi ricorda quante tabelle servono", () => {
    const m = render(sceneWith(["join"]), "op-sort");
    expect(m).toContain(copy.lockedJoin(2));
  });

  it("lavorazione con ingresso: famiglia, nome modificabile, campi del catalogo", () => {
    const store = sceneWith(["sort"]);
    connect(store);
    const m = render(store, "op-sort");
    expect(m).not.toContain("ei-blocked");
    expect(m).toContain("Filtra e ordina"); // la famiglia
    expect(m).toContain('value="Ordina"'); // il nome, in un campo
    expect(m).toContain("Criteri di ordinamento");
    expect(m).toContain('data-picker="columns"');
    expect(m).toContain(copy.inputsCount(1, 1));
  });

  it("l'elenco delle colonne segue il collegamento: cambia con lo schema in ingresso e si blocca se si scollega", () => {
    const store = sceneWith(["sort"]);
    connect(store);
    expect(render(store, "op-sort")).toContain('data-total="8"');
    // un altro dataset, con due colonne: collegato al posto del primo
    const small: ColumnDef[] = [
      { name: "a", type: "stringa", values: ["x"] },
      { name: "b", type: "integer", values: ["1"] },
    ];
    const state = store.getState();
    const ds1 = state.graph.cards["ds1"] as Card;
    const other: Card = {
      ...ds1,
      id: "ds2",
      name: "Piccolo",
      params: [{ ...(ds1.params[0] as Params), columns: small }],
    };
    const store2 = createEtlStore({
      initial: { ...state, graph: { ...state.graph, cards: { ...state.graph.cards, ds2: other } } },
    });
    store2.dispatch({
      type: "deleteLink",
      payload: { link: store2.getState().graph.links[0] as { from: string; to: string } },
    });
    expect(render(store2, "op-sort")).toContain("ei-blocked");
    connect(store2, "ds2", "op-sort");
    expect(render(store2, "op-sort")).toContain('data-total="2"');
  });

  it("filtro e join: una riga comprimibile per condizione, non più la nota «prossima fase»", () => {
    for (const type of ["filter", "join"] as ComponentId[]) {
      const store = sceneWith([type]);
      connect(store);
      const m = render(store, "op-sort");
      expect(m).toContain("ei-row-toggle");
      expect(m).toContain(`${copy.conditionNoun} 1`);
      expect(m).toContain(copy.conditionAdd);
      expect(m).not.toContain('data-picker="columns"');
    }
  });

  it("una selezione multipla senza nodo nell'inspector non mostra nulla da configurare", () => {
    const store = sceneWith(["sort"]);
    store.dispatch({ type: "select", payload: { ids: ["op-sort", "op-join"] } });
    const m = renderToStaticMarkup(createElement(Inspector, { store }));
    expect(m).toContain(copy.emptyInspector);
    expect(m).not.toContain("ei-fieldgroup");
  });
});

describe("Inspector: selettori per tipo di campo, dal catalogo", () => {
  it("ogni campo «columns» di MULTI_DEFS ha il ColumnPicker; ogni campo «column» una scelta singola", () => {
    let checked = 0;
    for (const [type, def] of Object.entries(MULTI_DEFS)) {
      if (!def) continue;
      const store = sceneWith([type as ComponentId]);
      connect(store);
      const m = render(store, "op-sort");
      const columnsFields = def.lists.flatMap((l) => l.fields).filter((f) => f.type === "columns");
      const columnFields = def.lists.flatMap((l) => l.fields).filter((f) => f.type === "column");
      // in ogni lista la prima riga è aperta: un selettore per campo, una volta per lista
      const perList = (t: string) =>
        def.lists.reduce((n, l) => n + l.fields.filter((f) => f.type === t).length, 0);
      expect(count(m, 'data-picker="columns"'), type).toBe(perList("columns"));
      if (columnsFields.length) expect(count(m, 'data-picker="columns"'), type).toBeGreaterThan(0);
      if (columnFields.length) expect(m, type).toContain('data-picker="select"');
      checked++;
    }
    expect(checked).toBe(Object.keys(MULTI_DEFS).length);
    // le operazioni con campi «columns» sono quelle della 6b.0, ricavate dal catalogo
    const multiColumnOps = Object.entries(MULTI_DEFS)
      .filter(([, d]) => d?.lists.some((l) => l.fields.some((f) => f.type === "columns")))
      .map(([t]) => t)
      .sort();
    expect(multiColumnOps).toEqual(
      [
        "aggregate",
        "cast",
        "dedup",
        "fillNa",
        "replaceVal",
        "round",
        "scale",
        "selectCols",
        "sort",
        "textClean",
      ].sort(),
    );
  });

  it("Rinomina e Dividi colonna usano una scelta singola, non il selettore multiplo", () => {
    for (const type of ["rename", "splitCol"] as OperationType[]) {
      const store = sceneWith([type]);
      connect(store);
      const m = render(store, "op-sort");
      expect(m, type).toContain('data-picker="select"');
      expect(m, type).not.toContain('data-picker="columns"');
    }
  });

  it("sostituisci valori: il selettore di valori ha il dominio unito delle colonne scelte", () => {
    const find = { ...createValuesField(), values: ["Nord"] };
    const store = sceneWith(
      ["replaceVal"],
      [{ items: [{ columns: ["regione", "stato"], match: "è uguale a", find, with: "" }] }],
    );
    connect(store);
    const m = render(store, "op-sort");
    // 4 regioni + 4 stati
    expect(m).toContain('data-picker="values"');
    expect(m).toMatch(/data-picker="values"[^>]*data-total="8"/);
  });

  it("cambiare le colonne non azzera i valori: quelli fuori dominio restano in corsivo, con l'avviso", () => {
    const find = { ...createValuesField(), values: ["Nord", "Mare"] };
    const store = sceneWith(
      ["replaceVal"],
      [{ items: [{ columns: ["regione"], match: "è uguale a", find, with: "" }] }],
    );
    connect(store);
    const m = render(store, "op-sort");
    expect(m).toContain(copy.valuesOutside(1));
    expect(m).toContain(copy.valuesOutsideRemove);
    expect(m).toMatch(/class="ei-chip ei-free"[^>]*><span[^>]*>Mare</);
    expect(m).toMatch(/class="ei-chip"[^>]*><span[^>]*>Nord</);
    // i valori scelti sono ancora tutti nel campo
    expect(m).toMatch(/data-picker="values"[^>]*data-selected="2"/);
  });

  it("Riempi vuoti: il valore è un elenco con una colonna, un campo di testo con più colonne", () => {
    const one = sceneWith(["fillNa"], [{ items: [{ columns: ["regione"], value: "" }] }]);
    connect(one);
    const m1 = render(one, "op-sort");
    expect(count(m1, 'data-picker="select"')).toBe(1);
    expect(m1).not.toContain("ei-input");
    const many = sceneWith(["fillNa"], [{ items: [{ columns: ["regione", "stato"], value: "" }] }]);
    connect(many);
    const m2 = render(many, "op-sort");
    expect(m2).not.toContain('data-picker="select"');
    expect(m2).toContain("ei-input");
  });

  it("Raggruppa: con più colonne il nome del risultato è disattivato con il nome automatico", () => {
    const params = (columns: string[]): Params[] => [
      {
        groupBy: [{ columns: ["regione"] }],
        measures: [{ columns, fn: "somma", alias: "tot" }],
      },
    ];
    const many = sceneWith(["aggregate"], params(["importo", "quantita"]));
    connect(many);
    // le misure sono la seconda lista: si apre la sua prima riga
    const m = render(many, "op-sort");
    expect(m).toContain("somma_importo, somma_quantita");
    expect(m).toMatch(/<input[^>]*disabled[^>]*value="somma_importo, somma_quantita"/);
    const one = sceneWith(["aggregate"], params(["importo"]));
    connect(one);
    const m1 = render(one, "op-sort");
    expect(m1).toContain('value="tot"');
    expect(m1).not.toMatch(/<input[^>]*disabled/);
  });
});

describe("Inspector: colonne non presenti nei dati in ingresso (Fase 6b.2, Passo 0)", () => {
  it("avviso con il numero e l'azione «Rimuovi»; la riga resta com'è finché non si preme", () => {
    const store = sceneWith(
      ["cast"],
      [{ items: [{ columns: ["importo", "fantasma", "x"], to: "intero" }] }],
    );
    connect(store);
    const m = render(store, "op-sort");
    expect(m).toContain('data-testid="ei-columns-outside"');
    expect(m).toContain(copy.columnsOutside(2));
    expect(m).toContain(`>${copy.columnsOutsideRemove}<`);
    // nulla si azzera da solo: le tre colonne sono ancora nei parametri
    const par = store.getState().graph.cards["op-sort"]!.params[0] as {
      items: { columns: string[] }[];
    };
    expect(par.items[0]!.columns).toEqual(["importo", "fantasma", "x"]);
    // e il nodo è incompleto (puntino ambra)
    expect(nodeStates(store.getState().graph)["op-sort"]).toBe("Da configurare: Converti tipo");
  });

  it("al singolare e senza avviso quando tutte le colonne ci sono", () => {
    const one = sceneWith(
      ["cast"],
      [{ items: [{ columns: ["importo", "fantasma"], to: "intero" }] }],
    );
    connect(one);
    expect(render(one, "op-sort")).toContain(copy.columnsOutside(1));
    const ok = sceneWith(["cast"], [{ items: [{ columns: ["importo"], to: "intero" }] }]);
    connect(ok);
    expect(render(ok, "op-sort")).not.toContain("ei-columns-outside");
  });

  it("senza schema in ingresso non c'è avviso", () => {
    const store = sceneWith(["cast"], [{ items: [{ columns: ["fantasma"], to: "intero" }] }]);
    // nessun collegamento: la lavorazione è bloccata, nessun campo e nessun avviso
    expect(render(store, "op-sort")).not.toContain("ei-columns-outside");
  });
});

describe("Inspector: tabelle del join assenti nei parametri (Fase 6b.2, Passo 0)", () => {
  /** Due sorgenti, «Vendite» e «Clienti», e il join `op-join`; i collegamenti nell'ordine dato. */
  function joinScene(order: readonly ["ds1" | "ds2", "ds1" | "ds2"]): EtlStore {
    const scene = prototypeScene();
    const ds1 = scene.graph.cards["ds1"] as Card;
    const ds2: Card = { ...ds1, id: "ds2", name: "Clienti", y: ds1.y + 120 };
    const store = createEtlStore({
      initial: {
        ...scene,
        graph: {
          ...scene.graph,
          cards: {
            ...scene.graph.cards,
            ds1: { ...ds1, name: "Vendite" },
            ds2,
          },
        },
      },
    });
    for (const from of order) connect(store, from, "op-join");
    return store;
  }
  /** I valori mostrati dai due campi tabella, nell'ordine in cui compaiono. */
  function shown(markup: string): string[] {
    const after = (label: string) =>
      new RegExp(`>${label}<[\\s\\S]*?ei-select-value[^>]*>([^<]*)<`).exec(markup)?.[1];
    return [after(copy.tableLeft) ?? "", after(copy.tableRight) ?? ""];
  }

  it("la sinistra è la prima tabella in ingresso, la destra la seconda; nulla viene salvato", () => {
    const store = joinScene(["ds1", "ds2"]);
    expect(shown(render(store, "op-join"))).toEqual(["Vendite", "Clienti"]);
    const par = store.getState().graph.cards["op-join"]!.params[0] as Record<string, unknown>;
    expect("leftTable" in par).toBe(false);
    expect("rightTable" in par).toBe(false);
  });

  it("segue l'ordine dei collegamenti", () => {
    expect(shown(render(joinScene(["ds2", "ds1"]), "op-join"))).toEqual(["Clienti", "Vendite"]);
  });
});

describe("Inspector: righe e riassunti", () => {
  it("la prima riga è aperta, il riassunto è quello del catalogo, aggiungi/rimuovi riga", () => {
    const store = sceneWith(
      ["cast"],
      [
        {
          items: [
            { columns: ["importo", "quantita"], to: "intero" },
            { columns: [], to: "testo" },
          ],
        },
      ],
    );
    connect(store);
    const m = render(store, "op-sort");
    expect(count(m, 'class="ei-row ei-open"')).toBe(1);
    expect(count(m, 'class="ei-row"')).toBe(1);
    expect(m).toContain("importo, quantita → intero");
    expect(m).toContain(copy.rowTodo);
    expect(count(m, `aria-label="${copy.rowRemove}"`)).toBe(2); // più di una riga: si può rimuovere
    expect(m).toContain("+ Aggiungi conversione");
  });

  it("i campi globali (tieni/escludi, mantieni) e le note", () => {
    const sel = sceneWith(["selectCols"]);
    connect(sel);
    expect(render(sel, "op-sort")).toContain("Modo");
    const dedup = sceneWith(
      ["dedup"],
      [
        {
          keep: "la prima",
          items: [
            { columns: ["id"], cmp: "esatto" },
            { columns: ["cliente"], cmp: "esatto" },
          ],
        },
      ],
    );
    connect(dedup);
    const m = render(dedup, "op-sort");
    expect(m).toContain("Mantieni");
    expect(m).toContain("Due righe sono duplicate quando coincidono su tutte le chiavi.");
  });
});

describe("Inspector: box combinato", () => {
  it("elenco dei passaggi nell'ordine, quello scelto mostra i parametri", () => {
    const store = sceneWith(["filter", "sort", "cast"]);
    connect(store);
    const m = render(store, "op-sort", 1);
    expect(count(m, '<li class="ei-step')).toBe(3);
    expect(m).toContain(`>${copy.kindBox}<`);
    // il passaggio 2 (Ordina) è quello scelto
    expect(m).toMatch(/class="ei-step ei-on"[^>]*data-step="1"/);
    expect(m).toContain("Criteri di ordinamento");
    expect(m).toContain(copy.stepDetach);
    expect(m).toContain(copy.stepDelete);
  });

  it("prima di un join: tabella di riferimento; dopo: la nota «tabella unica»", () => {
    const store = sceneWith(["filter", "join", "sort"]);
    connect(store);
    const before = render(store, "op-sort", 0);
    expect(before).toContain(copy.tableReference);
    const atJoin = render(store, "op-sort", 1);
    expect(atJoin).toContain(copy.tableLeft);
    expect(atJoin).toContain(copy.tableRight);
    const after = render(store, "op-sort", 2);
    expect(after).toContain(copy.tableSingle(1));
    expect(after).not.toContain(copy.tableReference);
  });
});

describe("pulsanti sul nodo e passaggi", () => {
  it("ogni nodo ha il pulsante × ; l'espansione solo sui box combinati", async () => {
    const { SIZE } = await import("./helpers");
    const { CanvasSurface } = await import("../EtlCanvas");
    const store = sceneWith(["filter", "sort"]);
    const m = renderToStaticMarkup(
      createElement(CanvasSurface, { store, size: SIZE, onExpand: () => {} }),
    );
    expect(count(m, 'class="ec-node-btn ec-del-btn"')).toBe(5); // ds1, filtro, join, box, esporta
    expect(count(m, 'class="ec-node-btn ec-expand-btn"')).toBe(1); // solo il box combinato
    expect(m).toContain(`aria-label="${copy.nodeDelete}"`);
    expect(m).toContain(`aria-label="${copy.nodeExpand}"`);
  });

  it("il pulsante × elimina subito un nodo isolato e chiede conferma per uno collegato", async () => {
    const { createInteractionController } = await import("../interaction");
    const store = sceneWith(["sort"]);
    const c = createInteractionController(store);
    // isolato: via subito, senza domande
    c.requestDeleteNodes(["op-export"]);
    expect(c.getUi().confirm).toBeNull();
    expect(store.getState().graph.cards["op-export"]).toBeUndefined();
    // collegato: conferma con l'anteprima di ciò che sparirebbe (nodesRemovedBy)
    connect(store);
    const out = outputOf(store.getState().graph, "op-sort") as string;
    c.requestDeleteNodes(["op-sort"]);
    const confirm = c.getUi().confirm;
    expect(confirm?.ids).toEqual(["op-sort"]);
    expect(confirm?.removed).toEqual(expect.arrayContaining(["op-sort", out]));
    expect(store.getState().graph.cards["op-sort"]).toBeDefined();
    c.cancelConfirm();
    expect(store.getState().graph.cards["op-sort"]).toBeDefined();
    c.requestDeleteNodes(["op-sort"]);
    expect(c.confirmDelete()).toEqual({ ok: true });
    expect(store.getState().graph.cards["op-sort"]).toBeUndefined();
  });

  it("l'elenco dei passaggi: nell'Inspector sgancio ed eliminazione, nel pannello espanso il menu", async () => {
    const { StepList } = await import("../inspector/StepList");
    const store = sceneWith(["filter", "sort", "cast"]);
    const card = store.getState().graph.cards["op-sort"] as Card;
    const base = { store, card, selectedStep: 1, onSelect: () => {} };
    const insp = renderToStaticMarkup(createElement(StepList, { ...base, variant: "inspector" }));
    expect(count(insp, `aria-label="${copy.stepDetach}"`)).toBe(3);
    expect(count(insp, `aria-label="${copy.stepDelete}"`)).toBe(3);
    expect(insp).not.toContain('aria-haspopup="menu"');
    const exp = renderToStaticMarkup(createElement(StepList, { ...base, variant: "expanded" }));
    expect(count(exp, 'aria-haspopup="menu"')).toBe(3);
    expect(exp).not.toContain(`aria-label="${copy.stepDetach}"`);
    // il riordino da tastiera è annunciato ai lettori di schermo
    expect(exp).toContain('role="status"');
  });

  it("i comandi dei passaggi: riordina, sgancia, elimina (comandi di etl-store)", () => {
    const store = sceneWith(["filter", "sort", "cast"]);
    connect(store);
    store.dispatch({ type: "inspect", payload: { node: "op-sort" } });
    expect(
      store.dispatch({ type: "reorderSteps", payload: { box: "op-sort", from: 0, to: 2 } }),
    ).toEqual({ ok: true });
    expect((store.getState().graph.cards["op-sort"] as Card).components).toEqual([
      "sort",
      "cast",
      "filter",
    ]);
    expect(store.getState().inspector.step).toBe(2);
    expect(store.dispatch({ type: "deleteStep", payload: { box: "op-sort", index: 1 } })).toEqual({
      ok: true,
    });
    expect((store.getState().graph.cards["op-sort"] as Card).components).toEqual([
      "sort",
      "filter",
    ]);
    const before = Object.keys(store.getState().graph.cards).length;
    expect(
      store.dispatch({
        type: "detachStep",
        payload: { box: "op-sort", index: 1, dropPoint: { x: 600, y: 120 } },
      }),
    ).toEqual({ ok: true });
    expect(Object.keys(store.getState().graph.cards).length).toBeGreaterThan(before - 1);
  });
});
```

