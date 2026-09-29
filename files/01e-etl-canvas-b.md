# 01e-etl-canvas-b.md

File in questo blocco:

- `src/etl-canvas/__tests__/render.test.ts`
- `src/etl-canvas/__tests__/ssr.test.tsx`
- `src/etl-canvas/__tests__/tokens.test.ts`
- `src/etl-canvas/__tests__/transitions.test.ts`
- `src/etl-canvas/__tests__/view.test.ts`
- `src/etl-canvas/actions.ts`
- `src/etl-canvas/canvas.css`
- `src/etl-canvas/contrast.ts`
- `src/etl-canvas/engine.ts`
- `src/etl-canvas/flow.ts`
- `src/etl-canvas/icons.tsx`
- `src/etl-canvas/index.ts`

---

### `src/etl-canvas/__tests__/render.test.ts`

186 righe

```ts
import { describe, expect, it, vi } from "vitest";
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

describe("animazioni e rendering lato server", () => {
  it("il rendering non chiama requestAnimationFrame né matchMedia", () => {
    const g = globalThis as unknown as Record<string, unknown>;
    const raf = vi.fn();
    const mm = vi.fn();
    g["requestAnimationFrame"] = raf;
    g["matchMedia"] = mm;
    try {
      const store = storeWith();
      store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
      html(store);
      expect(raf).not.toHaveBeenCalled();
      expect(mm).not.toHaveBeenCalled();
    } finally {
      delete g["requestAnimationFrame"];
      delete g["matchMedia"];
    }
  });

  it("i cavi rendono anche gli elementi che il motore aggiorna, vuoti e senza movimento", () => {
    const store = storeWith();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    const markup = html(store);
    expect(count(markup, 'class="ec-flow"')).toBe(store.getState().graph.links.length);
    expect(markup).toContain('class="ec-link-ghost"');
    expect(markup).not.toMatch(/<path class="ec-flow" d="M/);
  });
});
```

### `src/etl-canvas/__tests__/ssr.test.tsx`

92 righe

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

  it("la rotta senza parametri rende il canvas nuovo (predefinito), senza errori", async () => {
    const errors = vi.spyOn(console, "error").mockImplementation(() => {});
    const warns = vi.spyOn(console, "warn").mockImplementation(() => {});
    const out = await renderRoute("/solutions/demo/etl");
    expect(relevant(errors.mock.calls)).toEqual([]);
    expect(relevant(warns.mock.calls)).toEqual([]);
    expect(out).toContain("etl-canvas-host");
    expect(out).not.toContain("Saved ·");
    expect(out).not.toContain("ec-stage");
    errors.mockRestore();
    warns.mockRestore();
  });

  it("con ?seed=prototype si rende senza errori", async () => {
    const errors = vi.spyOn(console, "error").mockImplementation(() => {});
    const out = await renderRoute("/solutions/demo/etl?seed=prototype");
    expect(out).toContain("etl-canvas-host");
    expect(relevant(errors.mock.calls)).toEqual([]);
    errors.mockRestore();
  });

  it("con ?canvas=v1 resta raggiungibile il canvas vecchio", async () => {
    const errors = vi.spyOn(console, "error").mockImplementation(() => {});
    const out = await renderRoute("/solutions/demo/etl?canvas=v1");
    expect(out).toContain("Saved ·");
    expect(out).not.toContain("etl-canvas-host");
    expect(relevant(errors.mock.calls)).toEqual([]);
    errors.mockRestore();
  });
});
```

### `src/etl-canvas/__tests__/tokens.test.ts`

90 righe

```ts
import { readFileSync } from "node:fs";
import { describe, expect, it } from "vitest";
import { PAIRS, measure, readTokens } from "../contrast";

const css = readFileSync(new URL("../tokens.css", import.meta.url), "utf8");
const appCss = readFileSync(new URL("../../styles.css", import.meta.url), "utf8");
const prototype = readFileSync(
  new URL("../../../docs/prototype/isa-fusion-prototype.html", import.meta.url),
  "utf8",
);
const light = readTokens(css, "light", appCss);
const dark = readTokens(css, "dark", appCss);

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

### `src/etl-canvas/__tests__/transitions.test.ts`

142 righe

```ts
import { describe, expect, it } from "vitest";
import {
  TRANSITION_MS,
  crossfade,
  interpolatePoints,
  planTransition,
  progress,
  sampleTransition,
  samePoints,
} from "../transitions";
import type { Pt } from "../transitions";

const A: Pt[] = [
  { x: 0, y: 0 },
  { x: 100, y: 0 },
  { x: 100, y: 60 },
];
const B: Pt[] = [
  { x: 10, y: 20 },
  { x: 120, y: 40 },
  { x: 100, y: 200 },
];
const STRAIGHT: Pt[] = [
  { x: 0, y: 0 },
  { x: 100, y: 60 },
];

describe("interpolazione dei punti", () => {
  it("a metà transizione ogni punto è la media; all'inizio è il vecchio; alla fine il nuovo", () => {
    const mid = interpolatePoints(A, B, 0.5);
    A.forEach((p, i) => {
      const q = B[i] as Pt;
      expect(mid[i]).toEqual({ x: (p.x + q.x) / 2, y: (p.y + q.y) / 2 });
    });
    expect(interpolatePoints(A, B, 0)).toEqual(A);
    expect(interpolatePoints(A, B, 1)).toEqual(B);
  });

  it("alla fine coincide esattamente con il nuovo percorso, anche con decimali", () => {
    const from = [
      { x: 0.1, y: 0.7 },
      { x: 33.3, y: 9.9 },
    ];
    const to = [
      { x: 0.3, y: 0.2 },
      { x: 33.34, y: 9.91 },
    ];
    expect(interpolatePoints(from, to, 1)).toEqual(to);
  });

  it("un numero diverso di punti non si interpola", () => {
    expect(() => interpolatePoints(A, STRAIGHT, 0.5)).toThrow();
  });

  it("non modifica i percorsi di partenza", () => {
    const a = JSON.stringify(A);
    const b = JSON.stringify(B);
    interpolatePoints(A, B, 0.3);
    expect(JSON.stringify(A)).toBe(a);
    expect(JSON.stringify(B)).toBe(b);
  });
});

describe("dissolvenza incrociata", () => {
  it("le opacità sommano a 1 a ogni istante; a metà valgono 0,5 e 0,5", () => {
    for (let i = 0; i <= 20; i++) {
      const f = crossfade(i / 20);
      expect(f.old + f.next).toBeCloseTo(1, 12);
    }
    expect(crossfade(0.5)).toEqual({ old: 0.5, next: 0.5 });
    expect(crossfade(0)).toEqual({ old: 1, next: 0 });
    expect(crossfade(1)).toEqual({ old: 0, next: 1 });
  });
});

describe("avanzamento", () => {
  it("è lineare, limitato a 0..1, e dura 380 ms", () => {
    expect(TRANSITION_MS).toBe(380);
    expect(progress(0)).toBe(0);
    expect(progress(190)).toBe(0.5);
    expect(progress(380)).toBe(1);
    expect(progress(9999)).toBe(1);
    expect(progress(-5)).toBe(0);
    expect(progress(10, 0)).toBe(1);
  });
});

describe("quando c'è una transizione", () => {
  const base = { gesturing: false, reduced: false };

  it("cavo nuovo o percorso identico: nessuna", () => {
    expect(planTransition({ ...base, prev: null, next: A }).kind).toBe("none");
    expect(planTransition({ ...base, prev: A, next: A.map((p) => ({ ...p })) }).kind).toBe("none");
    expect(samePoints(A, B)).toBe(false);
  });

  it("stesso numero di punti: si interpola; numero diverso: dissolvenza", () => {
    expect(planTransition({ ...base, prev: A, next: B }).kind).toBe("morph");
    expect(planTransition({ ...base, prev: A, next: STRAIGHT }).kind).toBe("fade");
  });

  it("durante un gesto di trascinamento: nessuna, anche se il percorso cambia", () => {
    expect(planTransition({ prev: A, next: B, gesturing: true, reduced: false }).kind).toBe("none");
    expect(planTransition({ prev: A, next: STRAIGHT, gesturing: true, reduced: false }).kind).toBe(
      "none",
    );
  });

  it("con movimento ridotto: istantanea", () => {
    expect(planTransition({ prev: A, next: B, gesturing: false, reduced: true }).kind).toBe("none");
  });
});

describe("stato visivo nel tempo", () => {
  it("interpolazione: a metà tempo la media, a fine tempo il nuovo e `done`", () => {
    const plan = planTransition({ prev: A, next: B, gesturing: false, reduced: false });
    const mid = sampleTransition(plan, TRANSITION_MS / 2)!;
    expect(mid).toMatchObject({ kind: "points", done: false });
    if (mid.kind === "points") expect(mid.pts[0]).toEqual({ x: 5, y: 10 });
    const end = sampleTransition(plan, TRANSITION_MS)!;
    expect(end).toMatchObject({ kind: "points", done: true });
    if (end.kind === "points") expect(end.pts).toEqual(B);
    expect(sampleTransition(plan, TRANSITION_MS * 5)).toMatchObject({ done: true });
  });

  it("dissolvenza: a metà tempo 0,5 + 0,5, a fine tempo solo il nuovo", () => {
    const plan = planTransition({ prev: A, next: STRAIGHT, gesturing: false, reduced: false });
    const mid = sampleTransition(plan, TRANSITION_MS / 2)!;
    expect(mid).toMatchObject({ kind: "fade", old: 0.5, next: 0.5, done: false });
    expect(sampleTransition(plan, TRANSITION_MS)).toMatchObject({
      kind: "fade",
      old: 0,
      next: 1,
      done: true,
    });
  });

  it("nessuna transizione: nessuno stato", () => {
    expect(sampleTransition({ kind: "none" }, 100)).toBeNull();
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

333 righe

```css
/*
 * Aspetto del canvas ETL (Fase 4a): nodi, cavi, controlli, minimappa.
 * Misure e classi del prototipo (docs/prototype/isa-fusion-prototype.html,
 * righe indicate); colori solo dai token di tokens.css. Tutte le regole
 * sono limitate a `.etl-canvas`. Il carattere (Manrope) è quello di tutta l'app,
 * caricato da src/styles.css.
 */
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

/* flusso nei cavi (riga 1517: fill rgba(108,99,255,0.6)) e tracciato uscente di una dissolvenza */
.etl-canvas .ec-flow {
  fill: var(--ec-flow);
  pointer-events: none;
}
.etl-canvas .ec-link-ghost {
  fill: none;
  stroke: var(--ec-link);
  stroke-width: 2.1;
  stroke-linecap: round;
  stroke-linejoin: round;
  pointer-events: none;
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

137 righe

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

function readBlock(css: string, selector: string, prefix: string): Record<string, string> {
  const start = css.indexOf(`\n${selector} {`);
  if (start < 0) throw new Error(`blocco non trovato: ${selector}`);
  const end = css.indexOf("\n}", start);
  const out: Record<string, string> = {};
  for (const m of css
    .slice(start, end)
    .matchAll(new RegExp(`(${prefix}[a-z0-9-]+):\\s*([^;]+);`, "g"))) {
    out[m[1] as string] = (m[2] as string).trim();
  }
  return out;
}

/**
 * Legge i token di un tema da tokens.css: `dark` = blocco `.dark .etl-canvas`.
 * I `var(--isa-*)` sono risolti con le primitive di `styles.css` (`:root`,
 * `.dark`; nel tema scuro, per le primitive non ridefinite, vale il chiaro).
 */
export function readTokens(
  css: string,
  theme: "light" | "dark",
  primitivesCss: string,
): Record<string, string> {
  const tokens = readBlock(css, theme === "dark" ? ".dark .etl-canvas" : ".etl-canvas", "--ec-");
  const primitives = {
    ...readBlock(primitivesCss, ":root", "--isa-"),
    ...(theme === "dark" ? readBlock(primitivesCss, ".dark", "--isa-") : {}),
  };
  const out: Record<string, string> = {};
  for (const [name, value] of Object.entries(tokens)) {
    out[name] = value.replace(/var\((--isa-[a-z0-9-]+)\)/g, (_, ref: string) => {
      const resolved = primitives[ref];
      if (resolved === undefined) throw new Error(`primitiva non trovata: ${ref}`);
      return resolved;
    });
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
  { role: "Flusso nei cavi", fg: "--ec-flow", bg: "canvas", min: 3 },
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

### `src/etl-canvas/engine.ts`

335 righe

```ts
/**
 * Il livello sottile tra le funzioni pure (flow.ts, transitions.ts) e il
 * DOM: ad ogni frame del ciclo condiviso calcola lo stato visivo e ne
 * aggiorna solo gli attributi SVG che cambiano (il tracciato del flusso,
 * l'opacità dell'attesa, `d` durante una transizione). Non ridisegna
 * l'albero React.
 *
 * VINCOLO: qui si legge `route.pts` e `route.d` di `getRoutes()` e non si
 * scrive né si ricalcola mai un percorso. Nessuna chiamata a `settleLinks`
 * o a funzioni di etl-layout che instradano (i test lo verificano). Per
 * disegnare i punti interpolati si usa solo `roundedPath`, che arrotonda
 * punti dati e non sceglie nulla.
 */
import { ELBOW_R, roundedPath } from "../etl-layout";
import { flowWindowFor, tubeOutline, waitingOpacityFor } from "./flow";
import type { Sampler } from "./flow";
import { createLoop } from "./loop";
import type { Loop, LoopEnv } from "./loop";
import { planTransition, sampleTransition } from "./transitions";
import type { Plan, Pt, Visual } from "./transitions";

/** Il minimo di un elemento SVG che serve qui (un vero elemento, o un finto nei test). */
export interface AttrEl {
  setAttribute(name: string, value: string): void;
  style: { opacity: string };
}
export interface PathEl extends AttrEl {
  getTotalLength(): number;
  getPointAtLength(s: number): { x: number; y: number };
}
export interface GroupLike {
  querySelector(selector: string): unknown;
}

export interface LinkInput {
  readonly key: string;
  /** `linkLive` di etl-store: solo i cavi attivi hanno il flusso. */
  readonly live: boolean;
  /** `route.pts` e `route.d` di getRoutes, in sola lettura. */
  readonly pts: readonly Pt[];
  readonly d: string;
  readonly pa: Pt;
  readonly pb: Pt;
}

export interface UpdateInput {
  readonly links: readonly LinkInput[];
  /** Un gesto di trascinamento è in corso (`store.isGesturing()`). */
  readonly gesturing: boolean;
}

interface Handles {
  readonly path: PathEl;
  readonly ghost: AttrEl;
  readonly flow: AttrEl;
  readonly dotA: AttrEl;
  readonly dotB: AttrEl;
}

interface Target {
  readonly pts: readonly Pt[];
  readonly d: string;
  readonly pa: Pt;
  readonly pb: Pt;
}

interface Rec {
  group: GroupLike | null;
  handles: Handles | null;
  live: boolean;
  t0: number | null;
  /** Punti mostrati ora (a metà transizione, quelli interpolati). */
  shown: readonly Pt[] | null;
  target: Target | null;
  tr: { plan: Plan; start: number; oldD: string } | null;
  /** Abbiamo scritto `d`/opacità a mano: a fine transizione si ripristinano. */
  dirty: boolean;
}

export interface MotionEngine {
  registerLink(key: string, group: GroupLike | null): void;
  registerSlice(id: string, el: AttrEl | null): void;
  start(env: LoopEnv): void;
  stop(): void;
  update(input: UpdateInput): void;
  setGesturing(gesturing: boolean): void;
  /** Solo per i test. */
  debug(): { links: number; slices: number; running: boolean };
}

function handlesOf(g: GroupLike): Handles | null {
  const path = g.querySelector(".ec-link") as PathEl | null;
  const ghost = g.querySelector(".ec-link-ghost") as AttrEl | null;
  const flow = g.querySelector(".ec-flow") as AttrEl | null;
  const dotA = g.querySelector('[data-dot="a"]') as AttrEl | null;
  const dotB = g.querySelector('[data-dot="b"]') as AttrEl | null;
  if (!path || !ghost || !flow || !dotA || !dotB) return null;
  return { path, ghost, flow, dotA, dotB };
}

function setDot(el: AttrEl, p: Pt): void {
  el.setAttribute("cx", String(p.x));
  el.setAttribute("cy", String(p.y));
}

export function createMotionEngine(): MotionEngine {
  const links = new Map<string, Rec>();
  const slices = new Map<string, { el: AttrEl | null; t0: number | null }>();
  let env: LoopEnv | null = null;
  let loop: Loop | null = null;
  let off: (() => void) | null = null;
  let gesturing = false;
  let last: UpdateInput | null = null;

  const reduced = (): boolean => (env ? env.reducedMotion() : false);

  function clearFlow(rec: Rec): void {
    rec.handles?.flow.setAttribute("d", "");
  }

  function restore(rec: Rec): void {
    const h = rec.handles;
    const t = rec.target;
    if (!h || !t || !rec.dirty) return;
    h.path.setAttribute("d", t.d);
    h.path.style.opacity = "";
    h.ghost.setAttribute("d", "");
    h.ghost.style.opacity = "0";
    setDot(h.dotA, t.pa);
    setDot(h.dotB, t.pb);
    rec.dirty = false;
  }

  function apply(rec: Rec, v: Visual): void {
    const h = rec.handles;
    const t = rec.target;
    const tr = rec.tr;
    if (!h || !t || !tr) return;
    rec.dirty = true;
    if (v.kind === "points") {
      rec.shown = v.pts;
      h.path.setAttribute("d", roundedPath(v.pts, ELBOW_R));
      const a = v.pts[0];
      const b = v.pts[v.pts.length - 1];
      if (a) setDot(h.dotA, a);
      if (b) setDot(h.dotB, b);
    } else {
      // dissolvenza incrociata: il vecchio tracciato svanisce, il nuovo compare
      h.ghost.setAttribute("d", tr.oldD);
      h.ghost.style.opacity = String(v.old);
      h.path.style.opacity = String(v.next);
    }
  }

  function finish(rec: Rec): void {
    rec.tr = null;
    rec.shown = rec.target ? rec.target.pts : null;
    restore(rec);
  }

  function renderFlow(rec: Rec, now: number): boolean {
    const h = rec.handles;
    if (!h) return false;
    if (!rec.live || gesturing) {
      clearFlow(rec);
      return false;
    }
    let len = 0;
    try {
      len = h.path.getTotalLength();
    } catch {
      len = 0;
    }
    const win = flowWindowFor(len, now - (rec.t0 ?? now), reduced());
    if (!win) {
      clearFlow(rec);
      return true;
    }
    const sample: Sampler = (s) => h.path.getPointAtLength(s);
    h.flow.setAttribute("d", tubeOutline(sample, len, win));
    return true;
  }

  function renderSlices(now: number): void {
    for (const s of slices.values()) {
      s.t0 ??= now;
      if (s.el) s.el.style.opacity = String(waitingOpacityFor(now - s.t0, reduced()));
    }
  }

  function frame(now: number): boolean {
    let busy = false;
    for (const rec of links.values()) {
      if (rec.tr) {
        const v = sampleTransition(rec.tr.plan, now - rec.tr.start);
        if (v) apply(rec, v);
        if (!v || v.done) finish(rec);
        else busy = true;
      }
      if (renderFlow(rec, now)) busy = true;
    }
    if ([...slices.values()].some((s) => s.el)) {
      renderSlices(now);
      busy = true;
    }
    return busy;
  }

  /** Stato finale, senza movimento: transizioni concluse, flusso fermo a metà cavo, attesa a riposo. */
  function settle(): void {
    const now = env ? env.now() : 0;
    for (const rec of links.values()) {
      if (rec.tr) finish(rec);
      renderFlow(rec, now);
    }
    for (const s of slices.values()) if (s.el) s.el.style.opacity = "";
  }

  function applyUpdate(input: UpdateInput): void {
    if (!env) return;
    const now = env.now();
    gesturing = input.gesturing;
    const seen = new Set<string>();
    for (const l of input.links) {
      seen.add(l.key);
      const rec = links.get(l.key) ?? newRec();
      links.set(l.key, rec);
      rec.live = l.live;
      rec.t0 ??= now;
      const prev = rec.shown ?? rec.target?.pts ?? null;
      const plan = planTransition({
        prev,
        next: l.pts,
        gesturing: input.gesturing,
        reduced: env.reducedMotion(),
      });
      const oldD = rec.target?.d ?? l.d;
      rec.target = { pts: l.pts, d: l.d, pa: l.pa, pb: l.pb };
      if (plan.kind === "none") {
        rec.tr = null;
        rec.shown = l.pts;
        restore(rec);
      } else {
        rec.tr = { plan, start: now, oldD };
        const v = sampleTransition(plan, 0);
        if (v) apply(rec, v);
      }
    }
    for (const key of [...links.keys()]) if (!seen.has(key)) links.delete(key);
    // le fette smontate (React azzera il riferimento a ogni rendering): ora si dimenticano davvero
    for (const [id, s] of [...slices]) if (!s.el) slices.delete(id);
    if (env.reducedMotion()) settle();
    loop?.wake();
  }

  function newRec(): Rec {
    return {
      group: null,
      handles: null,
      live: false,
      t0: null,
      shown: null,
      target: null,
      tr: null,
      dirty: false,
    };
  }

  return {
    registerLink(key, group) {
      if (!group) {
        // React smonta il gruppo (o ne cambia il riferimento): si dimentica finché non torna
        const rec = links.get(key);
        if (rec) {
          rec.group = null;
          rec.handles = null;
        }
        return;
      }
      const rec = links.get(key) ?? newRec();
      links.set(key, rec);
      rec.group = group;
      rec.handles = handlesOf(group);
      rec.dirty = false;
    },

    registerSlice(id, el) {
      // React chiama il vecchio riferimento con null e il nuovo con l'elemento a ogni
      // rendering del nodo: la fase dell'attesa si conserva per identificativo
      const known = slices.get(id);
      if (!el) {
        if (known) known.el = null;
        return;
      }
      slices.set(id, { el, t0: known?.t0 ?? (env ? env.now() : null) });
      loop?.wake();
    },

    start(e) {
      this.stop();
      env = e;
      loop = createLoop(e);
      off = loop.add({ frame, settle });
      if (last) applyUpdate(last);
    },

    stop() {
      off?.();
      off = null;
      loop?.dispose();
      loop = null;
      env = null;
    },

    update(input) {
      last = input;
      applyUpdate(input);
    },

    setGesturing(g) {
      if (g === gesturing) return;
      gesturing = g;
      if (last) last = { ...last, gesturing: g };
      for (const rec of links.values()) if (g) clearFlow(rec);
      loop?.wake();
    },

    debug: () => ({
      links: links.size,
      slices: [...slices.values()].filter((s) => s.el).length,
      running: loop?.running() ?? false,
    }),
  };
}
```

### `src/etl-canvas/flow.ts`

180 righe

```ts
/**
 * Flusso nei cavi ("tubo elastico") e attesa delle fette vuote: calcolo puro.
 * Nessun React, nessun timer, nessun DOM: il tempo trascorso è un argomento.
 * Semantica del prototipo (docs/prototype/isa-fusion-prototype.html):
 * righe 1415-1424 (costanti e profilo), 1478 (smooth01), 1480-1521
 * (`animateBubbles`), 669-670 (attesa).
 */

/** Spessore del cavo, coincide con il tubo a riposo (riga 1415). */
export const BASE_W = 2.1;
/** px al millisecondo: stessa andatura su cavi lunghi e corti (riga 1416). */
export const SPEED = 0.16;
/** Rigonfiamento massimo per lato (riga 1417). */
export const BALL = 4.4;
/** Apertura rapida davanti, richiusura più lenta dietro (riga 1418). */
export const FRONT = 7.5;
export const BACK = 19;
/** Il tubo emerge dalla porta e vi rientra su questo tratto (riga 1510: `/ 22`). */
export const EDGE_FADE = 22;
/** Sotto questa lunghezza il tratto non si disegna (riga 1497: `s1 - s0 < 2`). */
export const MIN_VISIBLE = 2;
/** Campionamento del contorno: un punto ogni 1,6 px, almeno 10 (riga 1500). */
export const SAMPLE_STEP = 1.6;
export const MIN_SAMPLES = 10;

/** Riga 1478. */
export function smooth01(x: number): number {
  const c = Math.max(0, Math.min(1, x));
  return c * c * (3 - 2 * c);
}

/** Profilo gaussiano asimmetrico attorno al punto che avanza (righe 1419-1422). */
export function tubeProfile(u: number): number {
  const sg = u >= 0 ? FRONT : BACK;
  return Math.exp(-(u * u) / (sg * sg));
}

/** Tratto del cavo occupato dal tubo: `sb` è il punto che avanza (riga 1489-1495). */
export interface FlowWindow {
  readonly cycle: number;
  readonly sb: number;
  readonly s0: number;
  readonly s1: number;
}

/**
 * Finestra del tubo a `elapsedMs` dall'inizio del cavo (`now - st.t0`), su
 * un cavo lungo `len`. `null` se non c'è nulla da disegnare (righe 1488,
 * 1497). Il ciclo è `len + BACK * 3.2`: la pallina entra dalla porta di
 * uscita, sparisce oltre quella di ingresso e riparte.
 */
export function flowWindow(len: number, elapsedMs: number): FlowWindow | null {
  if (!(len > 0)) return null;
  const cycle = len + BACK * 3.2;
  const sb = ((elapsedMs * SPEED) % cycle) - BACK * 1.1;
  const s0 = Math.max(0, sb - BACK * 3);
  const s1 = Math.min(len, sb + FRONT * 3.2);
  if (s1 - s0 < MIN_VISIBLE) return null;
  return { cycle, sb, s0, s1 };
}

/**
 * Indicazione statica per chi preferisce meno movimento: lo stesso tubo,
 * fermo a metà cavo. Non esiste nel prototipo (NOTE_DIVERGENZE.md).
 */
export function staticFlowWindow(len: number): FlowWindow | null {
  if (!(len > 0)) return null;
  const sb = len / 2;
  return {
    cycle: len + BACK * 3.2,
    sb,
    s0: Math.max(0, sb - BACK * 3),
    s1: Math.min(len, sb + FRONT * 3.2),
  };
}

/** Finestra da mostrare: quella animata, o quella statica con movimento ridotto. */
export function flowWindowFor(len: number, elapsedMs: number, reduced: boolean): FlowWindow | null {
  return reduced ? staticFlowWindow(len) : flowWindow(len, elapsedMs);
}

export type Sampler = (s: number) => { readonly x: number; readonly y: number };

/**
 * Contorno chiuso del tubo (righe 1500-1516): campiona il percorso `d`
 * già calcolato con `sample` (in un browser `path.getPointAtLength`) e
 * allarga il tratto secondo `tubeProfile`. Non tocca mai il percorso.
 */
export function tubeOutline(sample: Sampler, len: number, win: FlowWindow): string {
  const n = Math.max(MIN_SAMPLES, Math.ceil((win.s1 - win.s0) / SAMPLE_STEP));
  const pts: { x: number; y: number; s: number }[] = [];
  for (let k = 0; k <= n; k++) {
    const s = win.s0 + ((win.s1 - win.s0) * k) / n;
    const q = sample(s);
    pts.push({ x: q.x, y: q.y, s });
  }
  const left: string[] = [];
  const right: string[] = [];
  for (let k = 0; k <= n; k++) {
    const a = pts[Math.max(0, k - 1)] as { x: number; y: number; s: number };
    const b = pts[Math.min(n, k + 1)] as { x: number; y: number; s: number };
    const p = pts[k] as { x: number; y: number; s: number };
    let tx = b.x - a.x;
    let ty = b.y - a.y;
    const tl = Math.hypot(tx, ty) || 1;
    tx /= tl;
    ty /= tl;
    const edge = smooth01(Math.min(p.s, len - p.s) / EDGE_FADE);
    const w = BASE_W / 2 + BALL * tubeProfile(p.s - win.sb) * edge;
    left.push((p.x - ty * w).toFixed(2) + " " + (p.y + tx * w).toFixed(2));
    right.push((p.x + ty * w).toFixed(2) + " " + (p.y - tx * w).toFixed(2));
  }
  return "M " + left.concat(right.reverse()).join(" L ") + " Z";
}

// --- Attesa delle fette vuote --------------------------------------------------

/** `animation: waiting 1.9s ease-in-out infinite` (riga 669). */
export const WAIT_PERIOD = 1900;
/** `@keyframes waiting { 0%,100% {opacity:.45} 50% {opacity:.95} }` (riga 670). */
export const WAIT_LOW = 0.45;
export const WAIT_HIGH = 0.95;
/** Opacità a riposo del simbolo (riga 669: `opacity:.85`), anche con movimento ridotto. */
export const WAIT_REST = 0.85;

/** Funzione di temporizzazione CSS `cubic-bezier(x1, y1, x2, y2)`. */
export function cubicBezier(x1: number, y1: number, x2: number, y2: number): (x: number) => number {
  const cx = 3 * x1;
  const bx = 3 * (x2 - x1) - cx;
  const ax = 1 - cx - bx;
  const cy = 3 * y1;
  const by = 3 * (y2 - y1) - cy;
  const ay = 1 - cy - by;
  const bx_ = (t: number) => ((ax * t + bx) * t + cx) * t;
  const by_ = (t: number) => ((ay * t + by) * t + cy) * t;
  const dx_ = (t: number) => (3 * ax * t + 2 * bx) * t + cx;
  return (x) => {
    if (x <= 0) return 0;
    if (x >= 1) return 1;
    let t = x;
    for (let i = 0; i < 8; i++) {
      const err = bx_(t) - x;
      if (Math.abs(err) < 1e-7) return by_(t);
      const d = dx_(t);
      if (Math.abs(d) < 1e-6) break;
      t -= err / d;
    }
    let lo = 0;
    let hi = 1;
    t = x;
    for (let i = 0; i < 40; i++) {
      const v = bx_(t);
      if (Math.abs(v - x) < 1e-7) break;
      if (v < x) lo = t;
      else hi = t;
      t = (lo + hi) / 2;
    }
    return by_(t);
  };
}

/** `ease-in-out` di CSS. */
export const easeInOut = cubicBezier(0.42, 0, 0.58, 1);

/**
 * Opacità del simbolo `<>` di una fetta vuota a `elapsedMs`: dal basso
 * (0,45) all'alto (0,95) a metà periodo e ritorno, con `ease-in-out` su
 * ciascuna metà, come i `@keyframes` del prototipo.
 */
export function waitingOpacity(elapsedMs: number): number {
  const phase = (((elapsedMs % WAIT_PERIOD) + WAIT_PERIOD) % WAIT_PERIOD) / WAIT_PERIOD;
  const u = phase < 0.5 ? phase * 2 : (1 - phase) * 2;
  return WAIT_LOW + (WAIT_HIGH - WAIT_LOW) * easeInOut(u);
}

/** Opacità da mostrare: animata, o ferma a riposo con movimento ridotto. */
export function waitingOpacityFor(elapsedMs: number, reduced: boolean): number {
  return reduced ? WAIT_REST : waitingOpacity(elapsedMs);
}
```

### `src/etl-canvas/icons.tsx`

24 righe

```tsx
import { EMPTY_SLOT_ICON, ICONS } from "../etl-core";
import type { ComponentId } from "../etl-core";

/** Icona del catalogo di etl-core (prototipo `svgTag`, righe 979-981). I tracciati sono costanti del catalogo, mai dati dell'utente. */
export function Icon(props: {
  id: ComponentId | "empty";
  svgRef?: (el: SVGSVGElement | null) => void;
}) {
  const inner = props.id === "empty" ? EMPTY_SLOT_ICON : ICONS[props.id];
  return (
    <svg
      ref={props.svgRef}
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

