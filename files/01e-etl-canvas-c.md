# 01e-etl-canvas-c.md

File in questo blocco:

- `src/etl-canvas/__tests__/autofit.test.ts`
- `src/etl-canvas/__tests__/conditions-ui.test.tsx`
- `src/etl-canvas/__tests__/conditions.test.ts`
- `src/etl-canvas/__tests__/controlbar.test.tsx`
- `src/etl-canvas/__tests__/drop.test.ts`

---

### `src/etl-canvas/__tests__/autofit.test.ts`

471 righe

```ts
import { describe, expect, it } from "vitest";
import type { Card } from "../../etl-core";
import type { Panels, View } from "../../etl-store";
import { ZOOM_MAX } from "../../etl-store";
import { CARD, LABEL_H } from "../../etl-layout";
import {
  AUTOFIT_MS,
  SAFE_MARGIN,
  canRestore,
  easing,
  fitTarget,
  interpolateArea,
  interpolateView,
  nodeInside,
  planChange,
  requiredNodes,
  safeBox,
} from "../panels/autoFit";
import type { Area } from "../panels/autoFit";
import { restArea } from "../panels/dockArea";
import type { WorkspaceMetrics } from "../panels/dockArea";
import { MIN_ZOOM, fitView } from "../view";

const card = (id: string, x: number, y: number): Card => ({ id, x, y }) as Card;
const NOINS = { top: 0, right: 0, bottom: 0, left: 0 };
const area = (w: number, h: number, insets = NOINS): Area => ({ size: { w, h }, insets });
const V1: View = { x: 0, y: 0, zoom: 1 };
const inside = (cards: readonly Card[], v: View, a: Area, tol = 1e-6) =>
  cards.every((c) => nodeInside(c, v, a, tol));

/** Scena rada: sei nodi sparsi. Scena densa: una griglia 7 × 5. */
const sparse: Card[] = [
  card("a", 40, 40),
  card("b", 400, 40),
  card("c", 700, 40),
  card("d", 40, 300),
  card("e", 700, 300),
  card("m", 400, 170),
];
const dense: Card[] = Array.from({ length: 35 }, (_, i) =>
  card(`n${i}`, 40 + (i % 7) * 150, 40 + Math.floor(i / 7) * 130),
);

const WINDOWS = [
  { name: "1440×900", w: 1440, h: 900 },
  { name: "1280×720", w: 1280, h: 720 },
  { name: "1280×600", w: 1280, h: 600 },
] as const;
/** Lo spazio di lavoro: la finestra meno l'intestazione della pagina; la barra dei controlli sta sopra. */
const ws = (w: { w: number; h: number }): WorkspaceMetrics => ({ w: w.w, h: w.h - 60, barH: 44 });
const SIDES = ["left", "right", "top", "bottom"] as const;
const closed: Panels = {
  tools: { side: "left", open: false },
  insp: { side: "right", open: false },
};
const opened = (panel: "tools" | "insp", side: (typeof SIDES)[number]): Panels => ({
  tools: { side: panel === "tools" ? side : "left", open: panel === "tools" },
  insp: { side: panel === "insp" ? side : "right", open: panel === "insp" },
});

describe("requiredNodes", () => {
  it("sono i nodi interamente dentro l'area sicura; un nodo già tagliato non è richiesto", () => {
    const cards = [card("in", 100, 100), card("cut", 950, 100), card("edge", 10, 100)];
    const req = requiredNodes(cards, V1, area(1000, 600));
    expect(req.map((c) => c.id)).toEqual(["in"]); // «cut» sporge a destra, «edge» sta nel margine di 24 px
  });

  it("l'ingombro di un widget restringe l'area sicura", () => {
    const c = [card("n", 100, 460)]; // y2 = 460 + 88 + 26 = 574
    expect(requiredNodes(c, V1, area(1000, 600))).toHaveLength(1);
    expect(requiredNodes(c, V1, area(1000, 600, { ...NOINS, bottom: 100 }))).toHaveLength(0);
    expect(safeBox(area(1000, 600))).toEqual({
      x1: SAFE_MARGIN,
      y1: SAFE_MARGIN,
      x2: 1000 - SAFE_MARGIN,
      y2: 600 - SAFE_MARGIN,
    });
  });
});

describe("fitTarget: i tre esiti", () => {
  it("1: i nodi entrano allo zoom attuale, solo scorrimento minimo, zoom e posizioni invariati", () => {
    const prev = area(1000, 600);
    const next = area(700, 600);
    const few = [card("p", 40, 40), card("q", 440, 40)]; // da 40 a 528
    const r = fitTarget({ cards: few, view: V1, intent: 1, prev, next });
    expect(r.outcome).toBe(1);
    expect(r.reason).toBe("entrano");
    expect(r.view).toEqual({ x: 0, y: 0, zoom: 1 }); // 528 ≤ 700 − 24: non serve scorrere
    const tight = fitTarget({ cards: few, view: V1, intent: 1, prev, next: area(540, 600) });
    expect(tight.outcome).toBe(1);
    expect(tight.view).toEqual({ x: 540 - 24 - 528, y: 0, zoom: 1 });
    expect(inside(few, tight.view, area(540, 600))).toBe(true);
  });

  it("2: non entrano: lo zoom scende e il riquadro dei nodi richiesti sta centrato nell'area", () => {
    const prev = area(1000, 600);
    const next = area(500, 600);
    const r = fitTarget({ cards: sparse, view: V1, intent: 1, prev, next });
    expect(r.outcome).toBe(2);
    expect(r.reason).toBe("zoom-ridotto");
    expect(r.view.zoom).toBeLessThan(1);
    expect(r.view.zoom).toBeGreaterThanOrEqual(MIN_ZOOM);
    expect(inside(sparse, r.view, next)).toBe(true);
    // centrato: avanza quanto resta a sinistra e a destra del riquadro
    const left = r.view.x + 40 * r.view.zoom;
    const right = 500 - (r.view.x + (700 + CARD) * r.view.zoom);
    expect(left).toBeCloseTo(right, 6);
  });

  it("3: nemmeno lo zoom minimo basta: zoom minimo, allineato in alto a sinistra, e il motivo lo dice", () => {
    const wide = [card("a", 40, 40), card("b", 6000, 40), card("c", 40, 3000)];
    const prev = area(8000, 4000);
    const next = area(400, 300);
    const r = fitTarget({ cards: wide, view: V1, intent: 1, prev, next });
    expect(r.outcome).toBe(3);
    expect(r.reason).toBe("zoom-minimo");
    expect(r.view.zoom).toBe(MIN_ZOOM);
    expect(r.view.x + 40 * MIN_ZOOM).toBeCloseTo(SAFE_MARGIN, 6);
    expect(r.view.y + 40 * MIN_ZOOM).toBeCloseTo(SAFE_MARGIN, 6);
  });

  it("nessun nodo richiesto: la vista non cambia (stessa istanza)", () => {
    const r = fitTarget({
      cards: [card("x", 5000, 5000)],
      view: V1,
      intent: 1,
      prev: area(800, 600),
      next: area(300, 300),
    });
    expect(r.outcome).toBe(1);
    expect(r.reason).toBe("nessun-nodo");
    expect(r.view).toBe(V1);
  });

  it("un nodo già tagliato prima non fa scendere lo zoom", () => {
    const cards = [card("p", 40, 40), card("cut", 940, 40)];
    const r = fitTarget({
      cards,
      view: V1,
      intent: 1,
      prev: area(1000, 600),
      next: area(500, 600),
    });
    expect(r.required).toBe(1);
    expect(r.view.zoom).toBe(1);
  });

  it("lo zoom di intento è il tetto: un'area grande non lo supera", () => {
    const small: View = { x: 0, y: 0, zoom: 0.5 };
    const r = fitTarget({
      cards: sparse,
      view: small,
      intent: 0.8,
      prev: area(1000, 600),
      next: area(3000, 2000),
    });
    expect(r.outcome).toBe(2);
    expect(r.reason).toBe("zoom-ripreso");
    expect(r.view.zoom).toBeCloseTo(0.8, 9);
  });

  it("l'intento sale dopo uno zoom manuale: la discesa parte da lì e non supera quello scelto", () => {
    const manual: View = { x: 10, y: 10, zoom: 1.5 };
    const few = [card("p", 40, 40), card("q", 300, 40)];
    const prev = area(1400, 800);
    const next = area(500, 800);
    const r = fitTarget({ cards: few, view: manual, intent: 1.5, prev, next });
    expect(r.view.zoom).toBeLessThanOrEqual(1.5);
    expect(r.view.zoom).toBeLessThan(1.5); // 340 × 1,5 > 452: serve scendere
    expect(inside(few, r.view, next)).toBe(true);
    // con più spazio risale fino all'intento, mai oltre
    const back = fitTarget({
      cards: few,
      view: r.view,
      intent: 1.5,
      prev: next,
      next: area(2000, 900),
    });
    expect(back.view.zoom).toBeCloseTo(1.5, 9);
    expect(ZOOM_MAX).toBeGreaterThanOrEqual(1.5);
  });
});

describe("planChange: punto di ripristino", () => {
  const prev = area(1000, 600);
  const next = area(500, 600);

  it("apri → chiudi senza altre azioni: la vista torna ESATTAMENTE quella di prima", () => {
    const v0: View = { x: 12.345, y: -6.789, zoom: 1 };
    const open = planChange({ cards: sparse, view: v0, intent: 1, prev, next, restore: null });
    expect(open.fit?.outcome).toBe(2);
    expect(open.restore).toEqual({ view: v0, area: prev });
    const close = planChange({
      cards: sparse,
      view: open.view,
      intent: 1,
      prev: next,
      next: area(1000, 600),
      restore: open.restore,
    });
    expect(close.restored).toBe(true);
    expect(close.view).toEqual(v0);
    expect(close.restore).toBeNull();
  });

  it("senza punto di ripristino (azione dell'utente) vale la regola generale con lo zoom di intento", () => {
    const v0: View = { x: 0, y: 0, zoom: 1 };
    const open = planChange({ cards: sparse, view: v0, intent: 1, prev, next, restore: null });
    const close = planChange({
      cards: sparse,
      view: open.view,
      intent: 1,
      prev: next,
      next: prev,
      restore: null,
    });
    expect(close.restored).toBe(false);
    expect(close.fit?.reason).toBe("zoom-ripreso");
    expect(close.view.zoom).toBeCloseTo(1, 9);
  });

  it("il primo adattamento salva la vista di prima; i successivi non la sostituiscono", () => {
    const v0: View = { x: 0, y: 0, zoom: 1 };
    const one = planChange({ cards: sparse, view: v0, intent: 1, prev, next, restore: null });
    const two = planChange({
      cards: sparse,
      view: one.view,
      intent: 1,
      prev: next,
      next: area(400, 600),
      restore: one.restore,
    });
    expect(two.restore).toBe(one.restore);
  });

  it("un cambio che non tocca la vista non crea un punto di ripristino", () => {
    const r = planChange({
      cards: sparse,
      view: V1,
      intent: 1,
      prev,
      next: area(1200, 800),
      restore: null,
    });
    expect(r.restore).toBeNull();
  });
});

describe("canRestore: il ripristino vale se l'area sicura è almeno altrettanto ampia", () => {
  const saved = area(1000, 600, { top: 0, right: 40, bottom: 58, left: 188 });

  it("stessa area: sì", () => {
    expect(canRestore(saved, saved)).toBe(true);
  });

  it("ingombri minori (una tacca si è spostata su un altro bordo): sì", () => {
    expect(canRestore(saved, area(1000, 600, { top: 0, right: 0, bottom: 58, left: 188 }))).toBe(
      true,
    );
  });

  it("un ingombro maggiore su un lato: no", () => {
    expect(canRestore(saved, area(1000, 600, { top: 40, right: 40, bottom: 58, left: 188 }))).toBe(
      false,
    );
  });

  it("dimensioni diverse: no, anche con meno ingombri", () => {
    expect(canRestore(saved, area(900, 600, NOINS))).toBe(false);
    expect(canRestore(saved, area(1000, 500, NOINS))).toBe(false);
  });

  it("planChange ripristina esattamente in un'area più ampia, non in una più stretta", () => {
    const v0: View = { x: 3, y: 4, zoom: 1 };
    const tight = area(1000, 600, { top: 0, right: 40, bottom: 0, left: 0 });
    const open = planChange({
      cards: sparse,
      view: v0,
      intent: 1,
      prev: tight,
      next: area(500, 600),
      restore: null,
    });
    const roomier = planChange({
      cards: sparse,
      view: open.view,
      intent: 1,
      prev: area(500, 600),
      next: area(1000, 600),
      restore: open.restore,
    });
    expect(roomier.restored).toBe(true);
    expect(roomier.view).toEqual(v0);
    const tighter = planChange({
      cards: sparse,
      view: open.view,
      intent: 1,
      prev: area(500, 600),
      next: area(1000, 600, { top: 0, right: 90, bottom: 0, left: 0 }),
      restore: open.restore,
    });
    expect(tighter.restored).toBe(false);
  });
});

describe("tabella pannelli × bordi × finestre × scene", () => {
  let cases = 0;
  const outcomes = { 1: 0, 2: 0, 3: 0 };
  for (const win of WINDOWS) {
    for (const [sceneName, scene] of [
      ["rada", sparse],
      ["densa", dense],
    ] as const) {
      for (const panel of ["tools", "insp"] as const) {
        for (const side of SIDES) {
          it(`${win.name}, scena ${sceneName}, ${panel} su ${side}: R1 e ripristino`, () => {
            const m = ws(win);
            const a0 = restArea(closed, m);
            const a1 = restArea(opened(panel, side), m);
            const v0: View = { x: 0, y: 0, zoom: 1 };
            const open = planChange({
              cards: scene,
              view: v0,
              intent: 1,
              prev: a0,
              next: a1,
              restore: null,
            });
            const fit = open.fit!;
            outcomes[fit.outcome]++;
            cases++;
            const req = requiredNodes(scene, v0, a0);
            if (fit.outcome !== 3) expect(inside(req, open.view, a1)).toBe(true);
            expect(open.view.zoom).toBeLessThanOrEqual(1);
            expect(open.view.zoom).toBeGreaterThanOrEqual(MIN_ZOOM);
            // a ogni istante della transizione i nodi richiesti restano dentro (convessità)
            if (fit.outcome !== 3) {
              for (let s = 0; s <= 1.0001; s += 0.05) {
                expect(
                  inside(req, interpolateView(v0, open.view, s), interpolateArea(a0, a1, s)),
                ).toBe(true);
              }
            }
            const close = planChange({
              cards: scene,
              view: open.view,
              intent: 1,
              prev: a1,
              next: a0,
              restore: open.restore,
            });
            if (open.restore) expect(close.view).toEqual(v0);
            else if (close.fit && close.fit.outcome !== 3)
              expect(inside(requiredNodes(scene, open.view, a1), close.view, a0)).toBe(true);
          });
        }
      }
    }
  }
  it("la tabella tocca tutti e tre gli esiti sul canvas con i pannelli aperti, o almeno 1 e 2", () => {
    expect(cases).toBe(3 * 2 * 2 * 4);
    expect(outcomes[1]).toBeGreaterThan(0);
    expect(outcomes[2]).toBeGreaterThan(0);
  });
});

describe("interpolateView", () => {
  const a: View = { x: 10, y: -20, zoom: 1 };
  const b: View = { x: 110, y: 80, zoom: 0.4 };
  it("estremi esatti e interpolazione lineare in x, y e zoom", () => {
    expect(interpolateView(a, b, 0)).toBe(a);
    expect(interpolateView(a, b, 1)).toBe(b);
    const m = interpolateView(a, b, 0.25);
    expect(m.x).toBeCloseTo(35, 12);
    expect(m.y).toBeCloseTo(5, 12);
    expect(m.zoom).toBeCloseTo(0.85, 12); // lineare, non logaritmica
  });
  it("lo zoom è monotono", () => {
    let prev = a.zoom;
    for (let i = 1; i <= 100; i++) {
      const z = interpolateView(a, b, i / 100).zoom;
      expect(z).toBeLessThanOrEqual(prev);
      prev = z;
    }
  });
});

describe("easing", () => {
  it("valori noti: estremi esatti, simmetrica, pendenza al più π/2", () => {
    expect(easing(0)).toBe(0);
    expect(easing(1)).toBe(1);
    expect(easing(-1)).toBe(0);
    expect(easing(2)).toBe(1);
    expect(easing(0.5)).toBeCloseTo(0.5, 12);
    expect(easing(0.25)).toBeCloseTo((1 - Math.SQRT1_2) / 2, 12);
    expect(easing(0.75)).toBeCloseTo(1 - easing(0.25), 12);
    let maxSlope = 0;
    for (let i = 0; i < 1000; i++)
      maxSlope = Math.max(maxSlope, (easing((i + 1) / 1000) - easing(i / 1000)) * 1000);
    expect(maxSlope).toBeLessThanOrEqual(Math.PI / 2 + 1e-3);
  });
  it("monotona crescente", () => {
    let prev = 0;
    for (let i = 1; i <= 1000; i++) {
      const e = easing(i / 1000);
      expect(e).toBeGreaterThanOrEqual(prev);
      prev = e;
    }
  });
  it("la durata copre almeno 10 frame a 60 fps", () => {
    expect(AUTOFIT_MS / (1000 / 60)).toBeGreaterThanOrEqual(10);
  });
});

describe("convessità: dentro all'inizio e alla fine → dentro a ogni s (200 casi, seme fisso)", () => {
  function rng(seed: number) {
    let t = seed;
    return () => {
      t = (t + 0x6d2b79f5) | 0;
      let r = Math.imul(t ^ (t >>> 15), 1 | t);
      r = (r + Math.imul(r ^ (r >>> 7), 61 | r)) ^ r;
      return ((r ^ (r >>> 14)) >>> 0) / 4294967296;
    };
  }
  it("vale per tutti i casi generati", () => {
    const rand = rng(20261005);
    let checked = 0;
    while (checked < 200) {
      const n = 2 + Math.floor(rand() * 10);
      const cards = Array.from({ length: n }, (_, i) => card(`k${i}`, rand() * 900, rand() * 600));
      const a0 = area(600 + rand() * 900, 300 + rand() * 500, {
        top: rand() * 60,
        right: rand() * 200,
        bottom: rand() * 130,
        left: rand() * 60,
      });
      const a1 = area(300 + rand() * 1200, 200 + rand() * 600, {
        top: rand() * 60,
        right: rand() * 200,
        bottom: rand() * 130,
        left: rand() * 60,
      });
      const v0: View = { x: rand() * 120 - 30, y: rand() * 120 - 30, zoom: 0.5 + rand() * 1.2 };
      const req = requiredNodes(cards, v0, a0);
      if (req.length === 0) continue;
      const fit = fitTarget({ cards, view: v0, intent: v0.zoom, prev: a0, next: a1 });
      if (fit.outcome === 3) continue;
      checked++;
      expect(inside(req, fit.view, a1)).toBe(true);
      for (let i = 0; i <= 40; i++) {
        const s = i / 40;
        expect(inside(req, interpolateView(v0, fit.view, s), interpolateArea(a0, a1, s))).toBe(
          true,
        );
      }
    }
    expect(checked).toBe(200);
  });
});

describe("fitView riusato", () => {
  it("senza opzioni è «Adatta» del prototipo (margine 48, zoom al più 1,25)", () => {
    expect(fitView([{ x: 100, y: 100 }], { w: 2000, h: 2000 }).zoom).toBe(1.25);
  });
  it("con margine 0 e tetto: il riquadro riempie l'area sicura", () => {
    const v = fitView([{ x: 0, y: 0 }], { w: 400, h: 400 }, NOINS, { pad: 0, maxZoom: 5 });
    expect(v.zoom).toBeCloseTo(Math.min(400 / CARD, 400 / (CARD + LABEL_H)), 9);
  });
});
```

### `src/etl-canvas/__tests__/conditions-ui.test.tsx`

465 righe

```tsx
import { readFileSync, readdirSync } from "node:fs";
import { dirname, join, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { LOGIC_HELP, LOGIC_OPS, createValuesField, newCondition } from "../../etl-core";
import type { Card, FilterCondition, JoinKey, Params } from "../../etl-core";
import { createEtlStore } from "../../etl-store";
import type { EtlStore } from "../../etl-store";
import { CONNECTOR_OPTIONS, ConnectorSelect } from "../inspector/ConnectorSelect";
import { copy } from "../inspector/copy";
import { Inspector } from "../inspector/Inspector";
import { prototypeScene } from "../seed";

const here = dirname(fileURLToPath(import.meta.url));
const INSPECTOR_DIR = resolve(here, "../inspector");

/** La scena con il nodo `node` collegato al dataset e con i parametri dati. */
function scene(node: "op-filter" | "op-join" | "op-sort", params: Params): EtlStore {
  const s = prototypeScene();
  const card = s.graph.cards[node] as Card;
  const store = createEtlStore({
    initial: {
      ...s,
      graph: { ...s.graph, cards: { ...s.graph.cards, [node]: { ...card, params: [params] } } },
    },
  });
  expect(store.dispatch({ type: "connect", payload: { from: "ds1", to: node } })).toEqual({
    ok: true,
  });
  return store;
}

function render(store: EtlStore, node: string, horizontal = false): string {
  store.dispatch({ type: "inspect", payload: { node, step: 0 } });
  return renderToStaticMarkup(createElement(Inspector, { store, horizontal }));
}

const count = (m: string, needle: string) => m.split(needle).length - 1;
const cond = (n: Partial<FilterCondition>): FilterCondition => ({ ...newCondition(), ...n });
const key = (n: Partial<JoinKey>): JoinKey => ({
  left: "",
  right: "",
  op: "=",
  lmode: "col",
  rmode: "col",
  lval: "",
  rval: "",
  rlist: createValuesField(),
  ...n,
});
const previewOf = (m: string) =>
  /data-testid="ei-preview-text">([^<]*)</.exec(m)?.[1]?.replace(/&gt;/g, ">");

describe("ConnectorSelect", () => {
  it("ha tutti e sei gli operatori, ciascuno con il suo testo di aiuto", () => {
    expect(CONNECTOR_OPTIONS.map((o) => o.value)).toEqual([
      "AND",
      "OR",
      "XOR",
      "NAND",
      "NOR",
      "XNOR",
    ]);
    expect(CONNECTOR_OPTIONS.map((o) => o.value)).toEqual([...LOGIC_OPS]);
    for (const op of LOGIC_OPS) {
      const o = CONNECTOR_OPTIONS.find((x) => x.value === op);
      expect(o?.label).toBe(`${op} · ${LOGIC_HELP[op]}`);
      expect(o?.short).toBe(op);
    }
    expect(CONNECTOR_OPTIONS[0]?.label).toBe("AND · entrambe vere");
    expect(CONNECTOR_OPTIONS[2]?.label).toBe("XOR · una sola delle due vera");
  });

  it("è una pastiglia con un nome accessibile che dice quali condizioni collega", () => {
    const m = renderToStaticMarkup(
      createElement(ConnectorSelect, { value: "XOR", onChange: () => {}, after: 3 }),
    );
    expect(m).toContain('aria-label="Connettore tra la condizione 2 e la 3"');
    expect(m).toContain('aria-haspopup="listbox"');
    expect(m).toContain("ei-pill");
    expect(m).toContain(">XOR<"); // nel campo chiuso solo il nome, non tutta l'etichetta
    expect(m).not.toContain("una sola delle due vera");
  });

  it("senza valore mostra AND", () => {
    const m = renderToStaticMarkup(
      createElement(ConnectorSelect, { value: undefined, onChange: () => {}, after: 2 }),
    );
    expect(m).toContain(">AND<");
  });

  it("nessun selettore globale E/O nel codice dell'Inspector", () => {
    for (const f of readdirSync(INSPECTOR_DIR).filter((n) => /\.tsx?$/.test(n))) {
      const text = readFileSync(join(INSPECTOR_DIR, f), "utf8");
      expect(text, f).not.toMatch(/\.logic\b|\blogic\s*[:=]\s*["']/);
      expect(text, f).not.toMatch(/["']E["']\s*\|\s*["']O["']/);
    }
    const m = render(
      scene("op-filter", { conditions: [cond({ column: "regione" }), cond({ column: "stato" })] }),
      "op-filter",
    );
    expect(m).not.toMatch(/aria-label="[^"]*(globale|Tutte le condizioni|Qualsiasi)/);
    expect(count(m, 'class="ei-pill ei-select"')).toBe(1); // un connettore, tra due condizioni
  });

  it("un filtro con il vecchio selettore E/O si migra nei connettori", () => {
    const store = scene("op-filter", {
      logic: "O",
      conditions: [
        cond({ column: "regione", values: ["Nord"] }),
        cond({ column: "stato", values: ["Chiuso"] }),
      ],
    });
    expect(previewOf(render(store, "op-filter"))).toBe("regione = Nord OR stato = Chiuso");
  });
});

describe("anteprima e gruppi nell'Inspector", () => {
  it("«regione = Nord AND (importo > 100 OR stato = Chiuso)»", () => {
    const store = scene("op-filter", {
      conditions: [
        cond({ column: "regione", op: "=", values: ["Nord"] }),
        cond({ column: "importo", op: ">", text: "100", conn: "AND", g: "g1" }),
        cond({ column: "stato", op: "=", values: ["Chiuso"], conn: "OR", g: "g1" }),
      ],
    });
    const m = render(store, "op-filter");
    expect(previewOf(m)).toBe("regione = Nord AND (importo > 100 OR stato = Chiuso)");
    // un gruppo con «Sciogli» e «+ Condizione nel gruppo»; un connettore esterno e uno interno
    expect(count(m, 'role="group" aria-label="Gruppo"')).toBe(1);
    expect(m).toContain(`>${copy.groupUngroup}<`);
    expect(m).toContain(copy.groupAddIn);
    expect(count(m, `aria-label="${copy.groupDo}"`)).toBe(1); // ( ) sul connettore esterno
    expect(count(m, `aria-label="${copy.groupSplit}"`)).toBe(1); // )( sul connettore interno
    expect(count(m, 'class="ei-pill ei-select"')).toBe(2);
  });

  it("«(regione è uno di Nord, Centro OR importo > 100) XOR stato = Chiuso»", () => {
    const store = scene("op-filter", {
      conditions: [
        cond({ column: "regione", op: "è uno di", values: ["Nord", "Centro"], g: "g1" }),
        cond({ column: "importo", op: ">", text: "100", conn: "OR", g: "g1" }),
        cond({ column: "stato", op: "=", values: ["Chiuso"], conn: "XOR" }),
      ],
    });
    expect(previewOf(render(store, "op-filter"))).toBe(
      "(regione è uno di Nord, Centro OR importo > 100) XOR stato = Chiuso",
    );
  });

  it("l'anteprima è un riquadro con etichetta e non è una regione annunciata", () => {
    const store = scene("op-filter", {
      conditions: [cond({ column: "regione", values: ["Nord"] }), cond({ column: "stato" })],
    });
    const m = render(store, "op-filter");
    expect(m).toContain('data-testid="ei-preview"');
    expect(m).toContain(`>${copy.previewTitle}<`);
    const box = /<div class="ei-preview"[\s\S]*?<\/div><\/div>/.exec(m)?.[0] ?? "";
    expect(box).toContain('role="group"');
    expect(box).toContain("aria-labelledby");
    expect(box).not.toMatch(/aria-live|role="status"|role="alert"/);
  });

  it("con una sola condizione non c'è anteprima né connettore", () => {
    const m = render(
      scene("op-filter", { conditions: [cond({ column: "regione" })] }),
      "op-filter",
    );
    expect(m).not.toContain("ei-preview");
    expect(m).not.toContain("ei-pill");
  });
});

describe("FilterCondition", () => {
  const one = (c: Partial<FilterCondition>) =>
    render(scene("op-filter", { conditions: [cond({ column: "regione", ...c })] }), "op-filter");

  it("colonna con il tipo sotto, operatore, valori (selettore) per gli operatori a più valori", () => {
    const m = one({ op: "è uno di", values: ["Nord"] });
    expect(m).toContain(`>${copy.filterColumn}<`);
    expect(m).toContain(copy.filterColumnType("stringa"));
    expect(m).toContain(`>${copy.filterOperator}<`);
    expect(m).toContain(`>${copy.filterValues}<`);
    expect(m).toContain('data-picker="values"');
  });

  it("nessun valore per gli operatori senza valore", () => {
    for (const op of ["è vuoto", "non è vuoto"] as const) {
      const m = one({ op });
      expect(m, op).not.toContain('data-picker="values"');
      expect(m, op).not.toContain(`>${copy.filterValue}<`);
      expect(m, op).not.toContain(`>${copy.filterValues}<`);
    }
  });

  it("campo di testo (mai input number) per gli altri operatori, numerico se la colonna è numerica", () => {
    const numeric = one({ column: "importo", op: ">", text: "5" });
    expect(numeric).toContain(`>${copy.filterValue}<`);
    expect(numeric).toContain('inputMode="numeric"');
    expect(numeric).not.toContain('type="number"');
    const text = one({ column: "regione", op: ">", text: "N" });
    expect(text).toContain('inputMode="text"');
    expect(text).not.toContain('type="number"');
    const date = one({ column: "data", op: "<", text: "2026" });
    expect(date).toContain('inputMode="text"'); // le date non sono numeri
  });

  it("i valori fuori dominio restano (non si azzerano cambiando colonna), in corsivo con l'avviso e «Rimuovi»", () => {
    const store = scene("op-filter", {
      conditions: [cond({ column: "stato", op: "=", values: ["Nord", "Chiuso"] })],
    });
    const m = render(store, "op-filter");
    expect(m).toContain(copy.valuesOutside(1)); // «Nord» non è tra i valori di «stato»
    expect(m).toContain(`>${copy.valuesOutsideRemove}<`);
    expect(m).toContain("ei-free");
    const par = store.getState().graph.cards["op-filter"]!.params[0] as {
      conditions: FilterCondition[];
    };
    expect(par.conditions[0]!.values).toEqual(["Nord", "Chiuso"]);
  });

  it("il valore proposto dipende dalla colonna scelta (dominio della colonna)", () => {
    // con «stato» il selettore propone i valori di «stato»: l'avviso non compare se sono in dominio
    const m = one({ column: "stato", op: "=", values: ["Chiuso"] });
    expect(m).not.toContain(copy.valuesOutside(1));
  });
});

describe("JoinCondition", () => {
  const joinWith = (keys: JoinKey[]) => scene("op-join", { type: "inner", keys });
  const render1 = (keys: JoinKey[]) => render(joinWith(keys), "op-join");

  it("lato sinistro: Colonna | Valore; lato destro: Colonna | Valore | Lista (radiogroup)", () => {
    const m = render1([key({ left: "regione", right: "regione" })]);
    expect(count(m, 'role="radiogroup"')).toBe(2);
    expect(m).toContain(`>${copy.joinLeftSide}<`);
    expect(m).toContain(`>${copy.joinCompare}<`);
    expect(m).toContain(`>${copy.joinRightSide}<`);
    const radios = [...m.matchAll(/role="radio"[^>]*>([^<]*)</g)].map((r) => r[1]);
    expect(radios).toEqual(["Colonna", "Valore", "Colonna", "Valore", "Lista"]);
    expect(m).toContain("= uguale a");
  });

  it("modalità valore: tendina con i valori della colonna dell'ALTRO lato, o campo di testo", () => {
    // sinistra = valore, destra = colonna «regione» → il valore propone i valori di «regione»
    const withDomain = render1([key({ lmode: "val", lval: "Nord", right: "regione" })]);
    expect(withDomain).toContain('data-picker="select"');
    expect(withDomain).toContain("Nord");
    // senza colonna dall'altro lato non c'è un dominio: si scrive
    const without = render1([key({ lmode: "val", lval: "Nord", right: "" })]);
    expect(without).toContain(`placeholder="${copy.joinValuePlaceholder}"`);
  });

  it("modalità lista a destra: selettore di valori con il dominio della colonna a sinistra, confronto «è uno di»", () => {
    const m = render1([
      key({
        left: "regione",
        op: "è uno di",
        rmode: "list",
        rlist: { ...createValuesField(), values: ["Nord", "Sud"] },
      }),
    ]);
    expect(m).toContain('data-picker="values"');
    expect(m).toContain("è uno di");
    expect(m).not.toContain("= uguale a");
    expect(m).toContain("regione è uno di (Nord, Sud)"); // riassunto dal vivo
  });

  it("il riassunto di riga è quello del dominio", () => {
    const m = render1([key({ left: "regione", right: "stato", op: "≠" })]);
    expect(m).toContain("regione ≠ stato");
  });
});

describe("avviso di prestazioni del join", () => {
  const warn = (keys: JoinKey[]) =>
    render(scene("op-join", { type: "inner", keys }), "op-join").includes("ei-join-perf");

  it("presente con sole disuguaglianze", () => {
    expect(warn([key({ left: "importo", right: "importo", op: "<" })])).toBe(true);
    expect(
      warn([
        key({ left: "importo", right: "importo", op: "<" }),
        key({ left: "id", right: "id", op: "≥", conn: "AND" }),
      ]),
    ).toBe(true);
  });

  it("presente con uguaglianze colonna = valore", () => {
    expect(warn([key({ left: "regione", lmode: "col", rmode: "val", rval: "Nord" })])).toBe(true);
    expect(warn([key({ lmode: "val", lval: "Nord", right: "regione" })])).toBe(true);
  });

  it("assente con almeno una colonna = colonna", () => {
    expect(warn([key({ left: "id", right: "id" })])).toBe(false);
    expect(
      warn([
        key({ left: "importo", right: "importo", op: "<" }),
        key({ left: "id", right: "id", conn: "OR" }),
      ]),
    ).toBe(false);
  });

  it("assente senza nessuna condizione completa", () => {
    expect(warn([key({})])).toBe(false);
  });

  it("dice che il join confronterà ogni riga con tutte le altre", () => {
    const m = render(
      scene("op-join", {
        type: "inner",
        keys: [key({ left: "importo", right: "importo", op: "<" })],
      }),
      "op-join",
    );
    expect(m).toContain("Nessuna condizione di uguaglianza");
    expect(m).toContain("ogni riga con tutte le altre");
  });
});

describe("layout a tre colonne", () => {
  const filter = () =>
    scene("op-filter", {
      conditions: [cond({ column: "regione", values: ["Nord"] }), cond({ column: "stato" })],
    });

  it("sui bordi alto e basso: Impostazioni | Condizioni e gruppi | Dettaglio", () => {
    const m = render(filter(), "op-filter", true);
    expect(m).toContain('data-testid="ei-cols3"');
    expect([...m.matchAll(/data-col="([a-z]+)"/g)].map((c) => c[1])).toEqual([
      "general",
      "master",
      "detail",
    ]);
    const heads = [...m.matchAll(/class="ei-col-head[^"]*"[^>]*>([^<]*)</g)].map((h) => h[1]);
    expect(heads).toEqual([copy.colSettings, copy.colConditions, ""]);
  });

  it("nome del nodo, tabelle e ingressi stanno nelle Impostazioni; le condizioni nella colonna centrale", () => {
    const m = render(filter(), "op-filter", true);
    const general = /data-col="general"[\s\S]*?data-col="master"/.exec(m)?.[0] ?? "";
    const master = /data-col="master"[\s\S]*?data-col="detail"/.exec(m)?.[0] ?? "";
    expect(general).toContain("ei-header");
    expect(general).toContain(copy.inputsCount(1, 1));
    expect(general).not.toContain("ei-row");
    expect(master).toContain(`${copy.conditionNoun} 1`);
    expect(master).toContain(`${copy.conditionNoun} 2`);
    expect(master).toContain("ei-preview");
  });

  it("il join mette tabelle e tipo di join nelle Impostazioni", () => {
    const m = render(
      scene("op-join", { type: "left", keys: [key({ left: "id", right: "id" })] }),
      "op-join",
      true,
    );
    const general = /data-col="general"[\s\S]*?data-col="master"/.exec(m)?.[0] ?? "";
    expect(general).toContain(`>${copy.tableLeft}<`);
    expect(general).toContain(`>${copy.tableRight}<`);
    expect(general).toContain("Tipo di join");
  });

  it("le operazioni a voci hanno «Elenco» al posto di «Condizioni e gruppi»", () => {
    const m = render(
      scene("op-sort", { items: [{ columns: ["regione"], dir: "crescente" }] }),
      "op-sort",
      true,
    );
    expect(m).toContain(`>${copy.colList}<`);
    expect(m).not.toContain(copy.colConditions);
  });

  it("le operazioni senza voci (Limita righe) restano nelle colonne CSS", () => {
    const store = scene("op-sort", { n: "10" });
    // op-sort come «Limita righe»: tipo senza righe
    const s = store.getState();
    const card = s.graph.cards["op-sort"] as Card;
    store.replaceState({
      ...s,
      graph: {
        ...s.graph,
        cards: {
          ...s.graph.cards,
          "op-sort": { ...card, components: ["limit"], params: [{ n: "10" }] },
        },
      },
    });
    expect(render(store, "op-sort", true)).not.toContain("ei-cols3");
  });

  it("sui bordi laterali le stesse parti si impilano con le righe comprimibili", () => {
    const m = render(filter(), "op-filter", false);
    expect(m).not.toContain("ei-cols3");
    expect(m).toContain('class="ei-row ei-open"'); // la prima riga è aperta sul posto
    expect(m).toContain("ei-row-body");
  });

  it("il CSS: colonne che scorrono per conto loro, intestazioni fisse, token per le larghezze", () => {
    const css = readFileSync(join(INSPECTOR_DIR, "inspector.css"), "utf8");
    expect(css).toMatch(/\.ei-col-body\s*\{[^}]*overflow-y:\s*auto/);
    expect(css).toMatch(/\.ei-col-head\s*\{[^}]*flex:\s*none/);
    expect(css).toMatch(/\.ei-col\s*\{[^}]*min-height:\s*0/);
    expect(css).toContain("var(--isa-md-general-w)");
    expect(css).toContain("var(--isa-md-master-w)");
    expect(css).not.toMatch(/\.ei-cols3[^{]*\{[^}]*overflow-x:\s*(auto|scroll)/);
    const tokens = readFileSync(resolve(here, "../../theme/layout-tokens.css"), "utf8");
    expect(tokens).toMatch(/--isa-md-min-w:\s*900px/);
    expect(tokens).toMatch(/--isa-md-general-w:\s*240px/);
    expect(tokens).toMatch(/--isa-md-master-w:\s*310px/);
  });
});

describe("Ordina: maniglie di riordino", () => {
  const criteria = [
    { columns: ["regione"], dir: "crescente" },
    { columns: ["importo"], dir: "decrescente" },
  ];

  it("una maniglia per criterio, con nome accessibile, scorciatoia e aiuto", () => {
    const m = render(scene("op-sort", { items: criteria }), "op-sort");
    expect(count(m, 'class="ei-icon-btn ei-row-grip"')).toBe(2);
    expect(m).toContain(`aria-label="${copy.rowGrip("Criterio 1")}"`);
    expect(m).toContain('aria-keyshortcuts="Alt+ArrowUp Alt+ArrowDown"');
    expect(m).toContain(copy.rowReorderHelp);
    expect(m).toContain("Il primo criterio è il principale");
  });

  it("con un solo criterio non c'è maniglia", () => {
    const m = render(scene("op-sort", { items: criteria.slice(0, 1) }), "op-sort");
    expect(m).not.toContain("ei-row-grip");
  });

  it("le altre liste non si riordinano (Raggruppa per, Rimuovi duplicati)", () => {
    const s = prototypeScene();
    const card = s.graph.cards["op-sort"] as Card;
    const store = createEtlStore({
      initial: {
        ...s,
        graph: {
          ...s.graph,
          cards: {
            ...s.graph.cards,
            "op-sort": {
              ...card,
              components: ["dedup"],
              params: [
                {
                  items: [
                    { columns: ["id"], cmp: "esatto" },
                    { columns: ["cliente"], cmp: "esatto" },
                  ],
                },
              ],
            },
          },
        },
      },
    });
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-sort" } });
    expect(render(store, "op-sort")).not.toContain("ei-row-grip");
  });
});
```

### `src/etl-canvas/__tests__/conditions.test.ts`

185 righe

```ts
import { describe, expect, it } from "vitest";
import { groupedPreview, newCondition, summarizeCond } from "../../etl-core";
import type { FilterCondition } from "../../etl-core";
import {
  addInGroup,
  addItem,
  groupAt,
  nextGroupId,
  openAfterRemove,
  removeItem,
  runsOf,
  setConnector,
  splitGroupAt,
  ungroupById,
} from "../inspector/conditions";

/** Una condizione con colonna, operatore e valori. */
function cond(column: string, op: FilterCondition["op"], v: string | string[]): FilterCondition {
  const base = newCondition();
  return Array.isArray(v) ? { ...base, column, op, values: v } : { ...base, column, op, text: v };
}

const preview = (list: readonly FilterCondition[]) =>
  groupedPreview(list, (c) => summarizeCond(c) ?? "…");

const gs = (list: readonly FilterCondition[]) => list.map((c) => c.g ?? null);

/** Tre condizioni: regione = Nord, importo > 100, stato = Chiuso. */
const three = (): FilterCondition[] => [
  cond("regione", "=", ["Nord"]),
  cond("importo", ">", "100"),
  cond("stato", "=", ["Chiuso"]),
];

describe("anteprima dell'espressione (stringa del dominio)", () => {
  it("«regione = Nord AND (importo > 100 OR stato = Chiuso)»", () => {
    let list = three();
    list = setConnector(list, 1, "AND");
    list = setConnector(list, 2, "OR");
    list = groupAt(list, 2); // raggruppa le ultime due
    expect(preview(list)).toBe("regione = Nord AND (importo > 100 OR stato = Chiuso)");
  });

  it("«(regione è uno di Nord, Centro OR importo > 100) XOR stato = Chiuso»", () => {
    let list: FilterCondition[] = [
      cond("regione", "è uno di", ["Nord", "Centro"]),
      cond("importo", ">", "100"),
      cond("stato", "=", ["Chiuso"]),
    ];
    list = setConnector(list, 1, "OR");
    list = setConnector(list, 2, "XOR");
    list = groupAt(list, 1); // il gruppo parte dalla prima condizione
    expect(preview(list)).toBe(
      "(regione è uno di Nord, Centro OR importo > 100) XOR stato = Chiuso",
    );
  });

  it("senza gruppi si valuta da sinistra a destra con le parentesi dal terzo elemento", () => {
    let list = three();
    list = setConnector(list, 1, "OR");
    list = setConnector(list, 2, "NAND");
    expect(preview(list)).toBe("(regione = Nord OR importo > 100) NAND stato = Chiuso");
  });

  it("con una sola condizione non c'è nulla da combinare", () => {
    expect(preview(three().slice(0, 1))).toBe("");
  });

  it("tutti e sei i connettori compaiono nell'anteprima", () => {
    for (const op of ["AND", "OR", "XOR", "NAND", "NOR", "XNOR"] as const) {
      const list = setConnector(three().slice(0, 2), 1, op);
      expect(preview(list)).toBe(`regione = Nord ${op} importo > 100`);
    }
  });
});

describe("gruppi", () => {
  it("raggruppa dalla prima condizione: le prime due formano un gruppo", () => {
    const list = groupAt(three(), 1);
    expect(gs(list)).toEqual(["g1", "g1", null]);
    expect(runsOf(list)).toEqual([
      { start: 0, end: 1, group: "g1" },
      { start: 2, end: 2, group: null },
    ]);
  });

  it("raggruppa le ultime due", () => {
    expect(gs(groupAt(three(), 2))).toEqual([null, "g1", "g1"]);
  });

  it("una terza condizione accanto a un gruppo vi entra", () => {
    const list = groupAt(groupAt(three(), 1), 2);
    expect(gs(list)).toEqual(["g1", "g1", "g1"]);
  });

  it("scioglie: le condizioni restano, senza gruppo", () => {
    const grouped = groupAt(three(), 2);
    const list = ungroupById(grouped, "g1");
    expect(gs(list)).toEqual([null, null, null]);
    expect(list.map((c) => c.column)).toEqual(["regione", "importo", "stato"]);
  });

  it("fonde due gruppi adiacenti", () => {
    let list = [...three(), cond("data", "<", "2026")];
    list = groupAt(list, 1); // (0,1)
    list = groupAt(list, 3); // (2,3): un secondo gruppo
    expect(new Set(list.map((c) => c.g)).size).toBe(2);
    list = groupAt(list, 2); // tra la 1 (gruppo 1) e la 2 (gruppo 2): si fondono
    expect(gs(list)).toEqual(["g1", "g1", "g1", "g1"]);
  });

  it("un gruppo che resta con una sola condizione si scioglie da sé", () => {
    const grouped = groupAt(three(), 2); // (1,2)
    const list = removeItem(grouped, 2);
    expect(list.map((c) => c.column)).toEqual(["regione", "importo"]);
    expect(gs(list)).toEqual([null, null]);
  });

  it("divide un gruppo in un punto: il pezzo di una sola condizione si scioglie", () => {
    const four = [...three(), cond("data", "<", "2026")];
    const grouped = groupAt(groupAt(groupAt(four, 1), 2), 3);
    expect(gs(grouped)).toEqual(["g1", "g1", "g1", "g1"]);
    const split = splitGroupAt(grouped, 2); // (0,1) e (2,3)
    expect(gs(split)).toEqual(["g1", "g1", "g2", "g2"]);
    const lone = splitGroupAt(grouped, 1); // (0) e (1,2,3): la prima resta libera
    expect(gs(lone)).toEqual([null, "g2", "g2", "g2"]);
  });

  it("aggiunge una condizione in fondo al gruppo, con connettore AND", () => {
    const grouped = groupAt(three(), 1); // (0,1) e 2 libera
    const { list, index } = addInGroup(grouped, "g1", () => ({ ...newCondition() }));
    expect(index).toBe(2);
    expect(gs(list)).toEqual(["g1", "g1", "g1", null]);
    expect(list[2]?.conn).toBe("AND");
  });

  it("un identificativo nuovo non coincide mai con uno in uso", () => {
    expect(nextGroupId([])).toBe("g1");
    expect(nextGroupId([{ g: "g3" }, { g: "g1" }, {}])).toBe("g4");
    const four = [...three(), cond("data", "<", "2026")];
    const list = groupAt(groupAt(four, 1), 3);
    expect(new Set(list.filter((c) => c.g).map((c) => c.g)).size).toBe(2);
  });
});

describe("aggiungi, rimuovi, connettore", () => {
  it("aggiungere mette la condizione in fondo con il connettore AND", () => {
    const { list, index } = addItem(three(), () => ({ ...newCondition() }));
    expect(list).toHaveLength(4);
    expect(index).toBe(3);
    expect(list[3]?.conn).toBe("AND");
  });

  it("rimuovere toglie la voce e lascia le altre nell'ordine", () => {
    expect(removeItem(three(), 1).map((c) => c.column)).toEqual(["regione", "stato"]);
  });

  it("cambiare il connettore tocca solo quella voce", () => {
    const list = setConnector(three(), 2, "XNOR");
    expect(list.map((c) => c.conn)).toEqual([undefined, undefined, "XNOR"]);
  });

  it("l'indice aperto segue le rimozioni", () => {
    expect(openAfterRemove(2, 0, 2)).toBe(1); // era dopo la rimossa: scala di uno
    expect(openAfterRemove(1, 1, 2)).toBe(1); // era la rimossa: resta la successiva
    expect(openAfterRemove(2, 2, 2)).toBe(1); // era l'ultima: va alla nuova ultima
    expect(openAfterRemove(0, 0, 0)).toBe(-1);
  });
});

describe("le funzioni non modificano la lista di partenza", () => {
  it("nessuna azione tocca l'input", () => {
    const list = three();
    const snapshot = JSON.stringify(list);
    groupAt(list, 1);
    splitGroupAt(groupAt(list, 1), 1);
    ungroupById(groupAt(list, 1), "g1");
    removeItem(list, 0);
    setConnector(list, 1, "OR");
    addItem(list, () => ({ ...newCondition() }));
    addInGroup(groupAt(list, 1), "g1", () => ({ ...newCondition() }));
    expect(JSON.stringify(list)).toBe(snapshot);
  });
});
```

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

