# 01e-etl-canvas-b.md

File in questo blocco:

- `src/etl-canvas/__tests__/engine.test.ts`
- `src/etl-canvas/__tests__/fake-env.ts`
- `src/etl-canvas/__tests__/flow.test.ts`
- `src/etl-canvas/__tests__/gesture-render.test.ts`
- `src/etl-canvas/__tests__/helpers.ts`
- `src/etl-canvas/__tests__/interaction.test.ts`
- `src/etl-canvas/__tests__/keyboard.test.ts`
- `src/etl-canvas/__tests__/loop.test.ts`
- `src/etl-canvas/__tests__/no-reroute.test.ts`

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

### `src/etl-canvas/__tests__/interaction.test.ts`

375 righe

```ts
import { describe, expect, it } from "vitest";
import { CARD, nodeCenter } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { createInteractionController, DRAG_THRESHOLD } from "../interaction";
import type { InteractionController } from "../interaction";
import { storeWith } from "./helpers";

/** Il centro di un nodo, in coordinate dell'area (vista identica al mondo: zoom 1, origine 0). */
function center(store: EtlStore, id: string) {
  return nodeCenter(store.getState().graph.cards[id]!);
}

function setup(): { store: EtlStore; c: InteractionController } {
  const store = storeWith();
  return { store, c: createInteractionController(store) };
}

/** Trascina il nodo `id` fino a `to` in 30 aggiornamenti, e rilascia se `release`. */
function drag(
  c: InteractionController,
  store: EtlStore,
  id: string,
  to: { x: number; y: number },
  release = true,
) {
  const from = center(store, id);
  expect(c.down({ kind: "node", id }, { x: from.x, y: from.y })).toBe(true);
  for (let i = 1; i <= 30; i++) {
    c.move({ x: from.x + ((to.x - from.x) * i) / 30, y: from.y + ((to.y - from.y) * i) / 30 });
  }
  if (release) c.up(to);
}

const steps = (store: EtlStore) => store.historySize().past;

describe("trascinamento di un nodo", () => {
  it("sotto la soglia è un click: seleziona, nessun gesto", () => {
    const { store, c } = setup();
    const p = center(store, "op-sort");
    c.down({ kind: "node", id: "op-sort" }, p);
    c.move({ x: p.x + DRAG_THRESHOLD - 1, y: p.y });
    expect(store.isGesturing()).toBe(false);
    c.up({ x: p.x + DRAG_THRESHOLD - 1, y: p.y });
    expect(store.getState().selection).toEqual(["op-sort"]);
    expect(store.getState().inspector.nodeId).toBe("op-sort");
    expect(store.getState().graph.cards["op-sort"]).toMatchObject({ x: 260, y: 338 });
    expect(steps(store)).toBe(0);
  });

  it("30 aggiornamenti e il rilascio fanno UN solo passo di cronologia", () => {
    const { store, c } = setup();
    drag(c, store, "op-sort", { x: 700, y: 600 }, false);
    expect(store.isGesturing()).toBe(true);
    expect(c.getUi().dragging).toEqual(["op-sort"]);
    expect(steps(store)).toBe(0);
    c.up({ x: 700, y: 600 });
    expect(store.isGesturing()).toBe(false);
    expect(steps(store)).toBe(1);
    expect(store.getState().graph.cards["op-sort"]!.x).toBeGreaterThan(500);
    expect(c.getUi().dragging).toEqual([]);
    store.undo();
    expect(store.getState().graph.cards["op-sort"]).toMatchObject({ x: 260, y: 338 });
  });

  it("Esc a metà trascinamento annulla il gesto: nessun passo, nodo al suo posto", () => {
    const { store, c } = setup();
    drag(c, store, "op-sort", { x: 700, y: 600 }, false);
    expect(c.key({ key: "Escape" })).toBe(true);
    expect(store.isGesturing()).toBe(false);
    expect(steps(store)).toBe(0);
    expect(store.getState().graph.cards["op-sort"]).toMatchObject({ x: 260, y: 338 });
  });

  it("con zoom e vista spostata, lo spostamento si divide per lo zoom", () => {
    const { store, c } = setup();
    store.dispatch({ type: "setView", payload: { x: 100, y: 0, zoom: 2 } });
    const start = { x: 100 + (260 + CARD / 2) * 2, y: (338 + CARD / 2) * 2 };
    c.down({ kind: "node", id: "op-sort" }, start);
    c.move({ x: start.x + 80, y: start.y });
    c.up({ x: start.x + 80, y: start.y });
    const x = store.getState().graph.cards["op-sort"]!.x;
    expect(x).toBeGreaterThanOrEqual(260 + 40 - 26);
    expect(x).toBeLessThanOrEqual(260 + 40 + 26);
  });
});

describe("esiti di relation() durante il trascinamento", () => {
  it("fusione: lavorazione su lavorazione → contorno merge, al rilascio una sola fusione", () => {
    const { store, c } = setup();
    const to = center(store, "op-export");
    drag(c, store, "op-sort", to, false);
    expect(c.getUi().drop).toEqual({ id: "op-export", outcome: "merge" });
    expect(c.getUi().hint).toContain("fondere");
    c.up(to);
    const cards = store.getState().graph.cards;
    expect(cards["op-sort"]).toBeUndefined();
    expect(cards["op-export"]!.components).toHaveLength(2);
    expect(steps(store)).toBe(1);
    expect(c.getUi().drop).toBeNull();
  });

  it("collegamento: dataset su lavorazione → link, il dataset torna al suo posto", () => {
    const { store, c } = setup();
    const to = center(store, "op-filter");
    drag(c, store, "ds1", to, false);
    expect(c.getUi().drop).toEqual({ id: "op-filter", outcome: "link" });
    c.up(to);
    expect(store.getState().graph.links).toContainEqual({ from: "ds1", to: "op-filter" });
    expect(store.getState().graph.cards["ds1"]).toMatchObject({ x: 26, y: 182 });
    expect(steps(store)).toBe(1);
  });

  it("collegamento inverso: lavorazione su dataset → link-reverse", () => {
    const { store, c } = setup();
    const to = center(store, "ds1");
    drag(c, store, "op-filter", to, false);
    expect(c.getUi().drop).toEqual({ id: "ds1", outcome: "link-reverse" });
    c.up(to);
    expect(store.getState().graph.links).toContainEqual({ from: "ds1", to: "op-filter" });
    expect(store.getState().graph.cards["op-filter"]).toMatchObject({ x: 260, y: 52 });
  });

  it("spostamento per incompatibilità: due dataset → displace con il motivo di etl-core", () => {
    const { store, c } = setup();
    store.dispatch({
      type: "addNode",
      payload: { component: "dataset", point: { x: 700, y: 500 } },
    });
    const ds2 = Object.values(store.getState().graph.cards).find(
      (k) => k.id !== "ds1" && k.kind === "dataset",
    )!.id;
    const before = Object.keys(store.getState().graph.cards).length;
    const to = center(store, "ds1");
    drag(c, store, ds2, to, false);
    expect(c.getUi().drop).toEqual({ id: "ds1", outcome: "displace" });
    expect(c.getUi().hint).toContain("Due dataset non si fondono");
    c.up(to);
    expect(store.getState().graph.links).toEqual([]);
    expect(Object.keys(store.getState().graph.cards)).toHaveLength(before);
  });

  it("rifiuto: dalla porta di una lavorazione su un'altra lavorazione → reject, nessun collegamento", () => {
    const { store, c } = setup();
    expect(c.down({ kind: "port", id: "op-sort", side: "r" }, center(store, "op-sort"))).toBe(true);
    const to = center(store, "op-export");
    c.move(to);
    expect(c.getUi().drop).toEqual({ id: "op-export", outcome: "reject" });
    expect(c.getUi().tempLink?.valid).toBe(false);
    c.up(to);
    expect(store.getState().graph.links).toEqual([]);
    expect(store.getState().graph.cards["op-export"]!.components).toHaveLength(1);
  });

  it("nel vuoto non c'è nessun contorno", () => {
    const { store, c } = setup();
    drag(c, store, "op-sort", { x: 900, y: 900 }, false);
    expect(c.getUi().drop).toBeNull();
    expect(c.getUi().insertLink).toBeNull();
    c.up({ x: 900, y: 900 });
  });
});

describe("trascinamento su un cavo", () => {
  function withLink() {
    const { store, c } = setup();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const key = Object.keys(store.getRoutes())[0]!;
    const pts = store.getRoutes()[key]!.pts;
    // metà del primo tratto: sul cavo e fuori da qualunque nodo
    const mid = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    return { store, c, key, mid };
  }

  it("lavorazione slegata su un cavo dataset→lavorazione: anteprima e insertOnLink al rilascio", () => {
    const { store, c, key, mid } = withLink();
    const p = center(store, "op-sort");
    c.down({ kind: "node", id: "op-sort" }, p);
    c.move({ x: p.x + 10, y: p.y });
    c.move(mid);
    expect(c.getUi().insertLink).toBe(key);
    expect(c.getUi().hint).toContain("inserire");
    const before = steps(store);
    c.up(mid);
    const links = store.getState().graph.links;
    expect(links).toContainEqual({ from: "ds1", to: "op-sort" });
    // la lavorazione produce un output, che entra nella lavorazione a valle (etl-core insertOnLink)
    const out = links.find((l) => l.from !== "ds1" && l.to === "op-join");
    expect(out).toBeDefined();
    expect(links).toContainEqual({ from: "op-sort", to: out!.from });
    expect(links).not.toContainEqual({ from: "ds1", to: "op-join" });
    expect(steps(store)).toBe(before + 1);
  });

  it("nessuna anteprima se il nodo ha già collegamenti", () => {
    const { store, c, mid } = withLink();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    const p = center(store, "op-filter");
    c.down({ kind: "node", id: "op-filter" }, p);
    c.move({ x: p.x + 10, y: p.y + 10 });
    c.move(mid);
    expect(c.getUi().insertLink).toBeNull();
    c.cancel();
  });

  it("nessuna anteprima su un cavo lavorazione→output, né trascinando un dataset", () => {
    const { store, c } = withLink();
    const outKey = Object.keys(store.getRoutes()).find((k) => k.startsWith("op-join|"));
    expect(outKey).toBeDefined();
    const pts = store.getRoutes()[outKey!]!.pts;
    const mid = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    const p = center(store, "op-sort");
    c.down({ kind: "node", id: "op-sort" }, p);
    c.move({ x: p.x + 10, y: p.y });
    c.move(mid);
    expect(c.getUi().insertLink).toBeNull();
    c.cancel();

    const d = center(store, "ds1");
    c.down({ kind: "node", id: "ds1" }, d);
    c.move({ x: d.x + 10, y: d.y });
    c.move(mid);
    expect(c.getUi().insertLink).toBeNull();
    c.cancel();
  });
});

describe("porte", () => {
  it("si crea solo un collegamento; il nodo di origine non si sposta e non c'è nessun gesto", () => {
    const { store, c } = setup();
    const before = { ...store.getState().graph.cards["ds1"]! };
    expect(c.down({ kind: "port", id: "ds1", side: "r" }, center(store, "ds1"))).toBe(true);
    expect(c.getUi().tempLink?.from).toEqual({ x: before.x + CARD, y: before.y + CARD / 2 });
    const to = center(store, "op-join");
    for (let i = 1; i <= 20; i++) {
      c.move({ x: before.x + CARD / 2 + ((to.x - before.x - CARD / 2) * i) / 20, y: to.y });
      expect(store.isGesturing()).toBe(false);
    }
    expect(c.getUi().drop).toEqual({ id: "op-join", outcome: "link" });
    expect(c.getUi().tempLink?.valid).toBe(true);
    c.up(to);
    expect(store.getState().graph.links).toContainEqual({ from: "ds1", to: "op-join" });
    expect(store.getState().graph.cards["ds1"]).toMatchObject({ x: before.x, y: before.y });
    expect(steps(store)).toBe(1);
    expect(c.getUi().tempLink).toBeNull();
  });

  it("dal lato di una lavorazione verso un dataset si collega in senso inverso", () => {
    const { store, c } = setup();
    c.down({ kind: "port", id: "op-join", side: "l" }, center(store, "op-join"));
    const to = center(store, "ds1");
    c.move(to);
    expect(c.getUi().drop).toEqual({ id: "ds1", outcome: "link-reverse" });
    c.up(to);
    expect(store.getState().graph.links).toContainEqual({ from: "ds1", to: "op-join" });
  });

  it("rilasciato nel vuoto non succede nulla", () => {
    const { store, c } = setup();
    c.down({ kind: "port", id: "ds1", side: "b" }, center(store, "ds1"));
    c.move({ x: 900, y: 700 });
    expect(c.getUi().hint).toContain("Trascina fino al nodo");
    c.up({ x: 900, y: 700 });
    expect(store.getState().graph.links).toEqual([]);
    expect(steps(store)).toBe(0);
  });
});

describe("selezione", () => {
  it("click singolo seleziona e aggiorna l'Inspector nello store", () => {
    const { store, c } = setup();
    const p = center(store, "op-join");
    c.down({ kind: "node", id: "op-join" }, p);
    c.up(p);
    expect(store.getState().selection).toEqual(["op-join"]);
    expect(store.getState().inspector).toEqual({ nodeId: "op-join", step: 0 });
  });

  it("Maiusc+click aggiunge e toglie", () => {
    const { store, c } = setup();
    const click = (id: string, shiftKey: boolean) => {
      const p = center(store, id);
      c.down({ kind: "node", id }, { ...p, shiftKey });
      c.up({ ...p, shiftKey });
    };
    click("op-join", false);
    click("op-sort", true);
    expect(store.getState().selection).toEqual(["op-join", "op-sort"]);
    click("op-join", true);
    expect(store.getState().selection).toEqual(["op-sort"]);
    expect(store.getState().inspector.nodeId).toBe("op-sort");
    click("op-sort", true);
    expect(store.getState().selection).toEqual([]);
    expect(store.getState().inspector.nodeId).toBeNull();
  });

  it("riquadro sul vuoto: seleziona i nodi che tocca", () => {
    const { store, c } = setup();
    expect(c.down({ kind: "background" }, { x: 240, y: 30 })).toBe(true);
    c.move({ x: 250, y: 40 });
    expect(c.getUi().marquee).not.toBeNull();
    c.move({ x: 380, y: 300 });
    expect(c.getUi().marquee!.ids.slice().sort()).toEqual(["op-filter", "op-join"]);
    c.up({ x: 380, y: 300 });
    expect(store.getState().selection.slice().sort()).toEqual(["op-filter", "op-join"]);
    expect(store.getState().inspector.nodeId).not.toBeNull();
    expect(c.getUi().marquee).toBeNull();
    expect(steps(store)).toBe(0);
  });

  it("riquadro con Maiusc si aggiunge alla selezione", () => {
    const { store, c } = setup();
    store.dispatch({ type: "select", payload: { ids: ["ds1"] } });
    c.down({ kind: "background" }, { x: 240, y: 30, shiftKey: true });
    c.move({ x: 380, y: 100 });
    c.up({ x: 380, y: 100 });
    expect(store.getState().selection.slice().sort()).toEqual(["ds1", "op-filter"]);
  });

  it("un click sul vuoto deseleziona", () => {
    const { store, c } = setup();
    store.dispatch({ type: "select", payload: { ids: ["ds1"] } });
    c.down({ kind: "background" }, { x: 900, y: 900 });
    c.up({ x: 900, y: 900 });
    expect(store.getState().selection).toEqual([]);
  });

  it("il gruppo selezionato si trascina insieme, con un solo passo di cronologia", () => {
    const { store, c } = setup();
    store.dispatch({ type: "select", payload: { ids: ["op-join", "op-sort"] } });
    const before = ["op-join", "op-sort"].map((id) => ({ ...store.getState().graph.cards[id]! }));
    const from = center(store, "op-join");
    c.down({ kind: "node", id: "op-join" }, from);
    for (let i = 1; i <= 30; i++) c.move({ x: from.x + 3 * i, y: from.y + 2 * i });
    expect(c.getUi().dragging.slice().sort()).toEqual(["op-join", "op-sort"]);
    c.up({ x: from.x + 90, y: from.y + 60 });
    const after = ["op-join", "op-sort"].map((id) => store.getState().graph.cards[id]!);
    expect(after[0]!.x - before[0]!.x).toBeGreaterThan(60);
    expect(Math.abs(after[0]!.x - before[0]!.x - (after[1]!.x - before[1]!.x))).toBeLessThanOrEqual(
      26,
    );
    expect(steps(store)).toBe(1);
    expect(store.getState().selection.slice().sort()).toEqual(["op-join", "op-sort"]);
  });

  it("un click su un nodo di un gruppo lo seleziona da solo", () => {
    const { store, c } = setup();
    store.dispatch({ type: "select", payload: { ids: ["op-join", "op-sort"] } });
    const p = center(store, "op-join");
    c.down({ kind: "node", id: "op-join" }, p);
    c.up(p);
    expect(store.getState().selection).toEqual(["op-join"]);
  });
});

describe("barra spaziatrice", () => {
  it("con lo spazio premuto nessun gesto di selezione parte", () => {
    const { store, c } = setup();
    c.setSpace(true);
    expect(c.down({ kind: "background" }, { x: 240, y: 30 })).toBe(false);
    expect(c.down({ kind: "node", id: "op-join" }, center(store, "op-join"))).toBe(false);
    c.move({ x: 400, y: 400 });
    expect(c.getUi().marquee).toBeNull();
    expect(c.isActive()).toBe(false);
    c.setSpace(false);
    expect(c.down({ kind: "background" }, { x: 240, y: 30 })).toBe(true);
    c.cancel();
  });

  it("i tasti diversi dal sinistro e i bersagli da ignorare non avviano nulla", () => {
    const { c } = setup();
    expect(c.down({ kind: "background" }, { x: 0, y: 0, button: 1 })).toBe(false);
    expect(c.down({ kind: "ignore" }, { x: 0, y: 0 })).toBe(false);
  });
});
```

### `src/etl-canvas/__tests__/keyboard.test.ts`

171 righe

```ts
import { describe, expect, it } from "vitest";
import { GRID } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { createInteractionController, NUDGE_STEP } from "../interaction";
import type { InteractionController } from "../interaction";
import { storeWith } from "./helpers";

function setup(): { store: EtlStore; c: InteractionController } {
  const store = storeWith();
  return { store, c: createInteractionController(store) };
}
const select = (store: EtlStore, ...ids: string[]) =>
  store.dispatch({ type: "select", payload: { ids } });
const cards = (store: EtlStore) => store.getState().graph.cards;

describe("Canc / Backspace", () => {
  it("nodo isolato: eliminazione immediata, senza conferma", () => {
    const { store, c } = setup();
    select(store, "op-sort");
    expect(c.key({ key: "Delete" })).toBe(true);
    expect(c.getUi().confirm).toBeNull();
    expect(cards(store)["op-sort"]).toBeUndefined();
  });

  it("Backspace fa lo stesso", () => {
    const { store, c } = setup();
    select(store, "op-sort");
    c.key({ key: "Backspace" });
    expect(cards(store)["op-sort"]).toBeUndefined();
  });

  it("nodi collegati: chiede conferma e mostra in anteprima ciò che sparirà (nodesRemovedBy)", () => {
    const { store, c } = setup();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    select(store, "op-filter");
    c.key({ key: "Delete" });
    const confirm = c.getUi().confirm;
    expect(confirm).not.toBeNull();
    expect(confirm!.title).toBe("Eliminare il nodo?");
    // il box e il suo output a valle
    expect(confirm!.removed).toContain("op-filter");
    expect(confirm!.removed.length).toBe(2);
    expect(confirm!.text).toContain("1 risultato a valle");
    // niente è stato eliminato finché non si conferma
    expect(cards(store)["op-filter"]).toBeDefined();
    const r = c.confirmDelete();
    expect(r?.ok).toBe(true);
    expect(cards(store)["op-filter"]).toBeUndefined();
    expect(c.getUi().confirm).toBeNull();
  });

  it("più nodi: titolo e testo al plurale", () => {
    const { store, c } = setup();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    select(store, "op-filter", "op-sort");
    c.key({ key: "Delete" });
    expect(c.getUi().confirm!.title).toBe("Eliminare 2 nodi?");
    expect(c.getUi().confirm!.text).toContain("1 risultato a valle");
  });

  it("Annulla e Esc chiudono la conferma senza eliminare", () => {
    const { store, c } = setup();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    select(store, "op-filter");
    c.key({ key: "Delete" });
    c.cancelConfirm();
    expect(c.getUi().confirm).toBeNull();
    c.key({ key: "Delete" });
    expect(c.key({ key: "Escape" })).toBe(true);
    expect(c.getUi().confirm).toBeNull();
    expect(cards(store)["op-filter"]).toBeDefined();
    // il primo Esc ha chiuso la conferma, la selezione è rimasta
    expect(store.getState().selection).toEqual(["op-filter"]);
  });

  it("senza selezione non fa nulla; dentro un campo di testo nemmeno", () => {
    const { store, c } = setup();
    expect(c.key({ key: "Delete" })).toBe(false);
    select(store, "op-sort");
    expect(c.key({ key: "Delete", typing: true })).toBe(false);
    expect(cards(store)["op-sort"]).toBeDefined();
  });
});

describe("duplica, seleziona tutto, deseleziona", () => {
  for (const mod of [{ metaKey: true }, { ctrlKey: true }]) {
    it(`Cmd/Ctrl+D duplica la selezione e seleziona le copie (${Object.keys(mod)[0]})`, () => {
      const { store, c } = setup();
      select(store, "op-sort");
      const before = Object.keys(cards(store));
      expect(c.key({ key: "d", ...mod })).toBe(true);
      const created = Object.keys(cards(store)).filter((id) => !before.includes(id));
      expect(created).toHaveLength(1);
      expect(store.getState().selection).toEqual(created);
      expect(store.getState().inspector.nodeId).toBe(created[0]);
      expect(cards(store)[created[0]!]!.name).toBe(cards(store)["op-sort"]!.name + " copia");
    });
  }

  it("Cmd/Ctrl+A seleziona tutto", () => {
    const { store, c } = setup();
    expect(c.key({ key: "a", metaKey: true })).toBe(true);
    expect(store.getState().selection.slice().sort()).toEqual(Object.keys(cards(store)).sort());
    expect(store.getState().inspector.nodeId).not.toBeNull();
  });

  it("Esc deseleziona", () => {
    const { store, c } = setup();
    select(store, "ds1", "op-join");
    expect(c.key({ key: "Escape" })).toBe(true);
    expect(store.getState().selection).toEqual([]);
    expect(store.getState().inspector.nodeId).toBeNull();
  });
});

describe("frecce", () => {
  it("passo singolo e, con Maiusc, passo ampio (come nel prototipo: 2 px e una cella)", () => {
    expect(NUDGE_STEP).toBe(2);
    const { store, c } = setup();
    select(store, "op-sort");
    const x0 = cards(store)["op-sort"]!.x;
    const y0 = cards(store)["op-sort"]!.y;
    c.key({ key: "ArrowRight" });
    expect(cards(store)["op-sort"]!.x).toBe(x0 + 2);
    c.key({ key: "ArrowDown", shiftKey: true });
    expect(cards(store)["op-sort"]!.y).toBe(y0 + GRID);
    c.key({ key: "ArrowLeft" });
    c.key({ key: "ArrowUp" });
    expect(cards(store)["op-sort"]).toMatchObject({ x: x0, y: y0 + GRID - 2 });
  });

  it("sposta l'intera selezione; tenere premuto un tasto è un solo passo di cronologia", () => {
    const { store, c } = setup();
    select(store, "op-sort", "op-export");
    for (let i = 0; i < 10; i++) c.key({ key: "ArrowRight" });
    expect(cards(store)["op-sort"]!.x).toBe(280);
    expect(cards(store)["op-export"]!.x).toBe(462);
    expect(store.historySize().past).toBe(1);
  });

  it("senza selezione non fa nulla (e non intercetta il tasto)", () => {
    const { c } = setup();
    expect(c.key({ key: "ArrowRight" })).toBe(false);
  });
});

describe("annulla e ripristina", () => {
  it("Cmd+Z annulla; Cmd+Maiusc+Z e Ctrl+Y ripristinano", () => {
    const { store, c } = setup();
    select(store, "op-sort");
    c.key({ key: "ArrowRight", shiftKey: true });
    const moved = cards(store)["op-sort"]!.x;
    expect(c.key({ key: "z", metaKey: true })).toBe(true);
    expect(cards(store)["op-sort"]!.x).toBe(260);
    expect(c.key({ key: "z", metaKey: true, shiftKey: true })).toBe(true);
    expect(cards(store)["op-sort"]!.x).toBe(moved);
    c.key({ key: "z", ctrlKey: true });
    expect(cards(store)["op-sort"]!.x).toBe(260);
    expect(c.key({ key: "y", ctrlKey: true })).toBe(true);
    expect(cards(store)["op-sort"]!.x).toBe(moved);
  });

  it("Cmd+Z con il fuoco in un campo di testo lascia l'annulla del campo", () => {
    const { store, c } = setup();
    select(store, "op-sort");
    c.key({ key: "ArrowRight" });
    expect(c.key({ key: "z", metaKey: true, typing: true })).toBe(false);
    expect(cards(store)["op-sort"]!.x).toBe(262);
  });
});
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

