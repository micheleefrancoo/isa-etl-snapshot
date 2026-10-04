# 01e-etl-canvas-c.md

File in questo blocco:

- `src/etl-canvas/__tests__/panels-actions.test.ts`
- `src/etl-canvas/__tests__/panels-layout.test.ts`
- `src/etl-canvas/__tests__/render.test.ts`
- `src/etl-canvas/__tests__/ssr.test.tsx`
- `src/etl-canvas/__tests__/tokens.test.ts`
- `src/etl-canvas/__tests__/toolbox-drop.test.ts`
- `src/etl-canvas/__tests__/toolbox.test.tsx`
- `src/etl-canvas/__tests__/transitions.test.ts`
- `src/etl-canvas/__tests__/view.test.ts`
- `src/etl-canvas/actions.ts`

---

### `src/etl-canvas/__tests__/panels-actions.test.ts`

158 righe

```ts
import { describe, expect, it } from "vitest";
import { createEtlStore, fromSaved, initialState, parseSaved, toSaved } from "../../etl-store";
import type { EtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
import { createPanelActions, followInspector } from "../panels/actions";
import { PANEL_SIZE, EXTENT_PAD } from "../panels/layout";
import { storeWith } from "./helpers";

const TOOLS_EXT = PANEL_SIZE.tools.w + EXTENT_PAD;
const panels = (s: EtlStore) => s.getState().panels;
const view = (s: EtlStore) => s.getState().view;

describe("aprire, chiudere, spostare un pannello", () => {
  it("lo stato iniziale è quello del prototipo: cassetta aperta a sinistra, Inspector chiuso a destra", () => {
    const s = createEtlStore();
    expect(panels(s)).toEqual({
      tools: { side: "left", open: true },
      insp: { side: "right", open: false },
    });
  });

  it("chiudere e riaprire la cassetta a sinistra: la vista compensa, i nodi restano fermi sullo schermo", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    expect(a.close("tools").ok).toBe(true);
    expect(panels(s).tools.open).toBe(false);
    expect(view(s).x).toBe(TOOLS_EXT);
    a.open("tools");
    expect(panels(s).tools.open).toBe(true);
    expect(view(s).x).toBe(0);
  });

  it("l'Inspector a destra si apre e si chiude senza toccare la vista", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    a.open("insp");
    expect(panels(s).insp.open).toBe(true);
    a.close("insp");
    expect(view(s)).toEqual({ x: 0, y: 0, zoom: 1 });
  });

  it("trascinare la tacca su ciascuno dei quattro bordi sposta il pannello e lo riapre", () => {
    for (const side of ["left", "right", "top", "bottom"] as const) {
      const s = createEtlStore();
      const a = createPanelActions(s);
      a.close("tools");
      a.moveTo("tools", side === "left" ? "right" : "left");
      a.moveTo("tools", side);
      expect(panels(s).tools).toEqual({ side, open: true });
    }
  });

  it("dal bordo sinistro a uno non sinistro la vista restituisce lo spazio; sopra e sotto non la toccano", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    a.moveTo("tools", "top");
    expect(panels(s).tools).toEqual({ side: "top", open: true });
    expect(view(s).x).toBe(TOOLS_EXT); // il canvas non cede più larghezza a sinistra
    a.moveTo("tools", "bottom");
    expect(view(s).x).toBe(TOOLS_EXT);
    a.moveTo("tools", "left");
    expect(view(s).x).toBe(0);
  });

  it("due pannelli sullo stesso bordo diventano schede: se ne apre uno alla volta; separati, tornano due pannelli", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    a.moveTo("insp", "left"); // si apre sullo stesso bordo della cassetta
    expect(panels(s).insp).toEqual({ side: "left", open: true });
    expect(panels(s).tools).toEqual({ side: "left", open: false }); // l'altra scheda si chiude
    a.open("tools");
    expect(panels(s).tools.open).toBe(true);
    expect(panels(s).insp.open).toBe(false);
    a.moveTo("insp", "right");
    expect(panels(s).insp.side).toBe("right");
    expect(panels(s).tools.side).toBe("left");
  });

  it("cambiare scheda a sinistra non sposta il canvas", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    a.moveTo("insp", "left");
    const x = view(s).x;
    a.open("tools");
    a.open("insp");
    expect(view(s).x).toBe(x);
  });
});

describe("persistenza", () => {
  it("lato, aperto/chiuso e scheda attiva sopravvivono a salvataggio e caricamento", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    a.moveTo("insp", "top");
    a.moveTo("tools", "top");
    a.open("insp"); // scheda attiva: Inspector, in alto
    const saved = JSON.stringify(toSaved(s.getState()));
    const loaded = parseSaved(saved);
    expect(loaded?.panels).toEqual(panels(s));
    expect(loaded?.panels).toEqual({
      tools: { side: "top", open: false },
      insp: { side: "top", open: true },
    });
    expect(fromSaved(toSaved(s.getState()))?.panels).toEqual(panels(s));
  });

  it("lo stato dei pannelli non è locale ai componenti: un altro store caricato dallo stesso salvataggio ha gli stessi pannelli", () => {
    const s = createEtlStore();
    createPanelActions(s).moveTo("tools", "bottom");
    const copy = createEtlStore({
      initial: parseSaved(JSON.stringify(toSaved(s.getState()))) ?? initialState(),
    });
    expect(copy.getState().panels.tools).toEqual({ side: "bottom", open: true });
  });
});

describe("l'Inspector segue la selezione", () => {
  it("selezionare un nodo lo apre, deselezionare lo chiude", () => {
    const store = storeWith();
    const actions = createPanelActions(store);
    const c = createInteractionController(store);
    const stop = followInspector(store, actions);
    expect(panels(store).insp.open).toBe(false);
    c.down({ kind: "node", id: "op-join" }, { x: 304, y: 226 });
    c.up({ x: 304, y: 226 });
    expect(store.getState().inspector.nodeId).toBe("op-join");
    expect(panels(store).insp.open).toBe(true);
    c.key({ key: "Escape" });
    expect(store.getState().inspector.nodeId).toBeNull();
    expect(panels(store).insp.open).toBe(false);
    stop();
  });

  it("se l'utente lo chiude con un nodo selezionato, resta chiuso finché la selezione non cambia stato", () => {
    const store = storeWith();
    const actions = createPanelActions(store);
    const stop = followInspector(store, actions);
    store.dispatch({ type: "select", payload: { ids: ["op-join"] } });
    store.dispatch({ type: "inspect", payload: { node: "op-join" } });
    expect(panels(store).insp.open).toBe(true);
    actions.close("insp");
    store.dispatch({ type: "inspect", payload: { node: "op-sort" } });
    expect(panels(store).insp.open).toBe(false);
    store.dispatch({ type: "inspect", payload: { node: null } });
    store.dispatch({ type: "inspect", payload: { node: "op-sort" } });
    expect(panels(store).insp.open).toBe(true);
    stop();
  });

  it("smettere di ascoltare lascia i pannelli come sono", () => {
    const store = storeWith();
    const stop = followInspector(store, createPanelActions(store));
    stop();
    store.dispatch({ type: "inspect", payload: { node: "op-join" } });
    expect(panels(store).insp.open).toBe(false);
  });
});
```

### `src/etl-canvas/__tests__/panels-layout.test.ts`

128 righe

```ts
import { describe, expect, it } from "vitest";
import type { Panels } from "../../etl-store";
import {
  EXTENT_PAD,
  PANEL_SIZE,
  activeTab,
  compensate,
  isGrouped,
  nearestSide,
  notchHidden,
  notchOffset,
  openExtent,
  panelExtent,
  panelSize,
  viewCompensation,
  workspaceExtra,
} from "../panels/layout";

const P = (tools: Panels["tools"], insp: Panels["insp"]): Panels => ({ tools, insp });
const closedSplit = P({ side: "left", open: false }, { side: "right", open: false });

describe("misure dei pannelli", () => {
  it("separati hanno la misura propria; in gruppo prendono la maggiore", () => {
    const split = P({ side: "left", open: true }, { side: "right", open: false });
    expect(isGrouped(split)).toBe(false);
    expect(panelSize(split, "tools")).toEqual(PANEL_SIZE.tools);
    expect(panelSize(split, "insp")).toEqual(PANEL_SIZE.insp);
    const grouped = P({ side: "left", open: true }, { side: "left", open: false });
    expect(isGrouped(grouped)).toBe(true);
    const w = Math.max(PANEL_SIZE.tools.w, PANEL_SIZE.insp.w);
    expect(panelSize(grouped, "tools").w).toBe(w);
    expect(panelSize(grouped, "insp").w).toBe(w);
    expect(panelExtent(grouped, "tools")).toBe(w + EXTENT_PAD);
  });

  it("l'ingombro dei bordi verticali è la larghezza, quello degli orizzontali l'altezza", () => {
    const top = P({ side: "top", open: true }, { side: "right", open: false });
    expect(panelExtent(top, "tools")).toBe(PANEL_SIZE.tools.h + EXTENT_PAD);
    expect(workspaceExtra(top)).toBe(PANEL_SIZE.tools.h + EXTENT_PAD);
    expect(workspaceExtra(closedSplit)).toBe(0);
  });

  it("solo i pannelli aperti sottraggono spazio", () => {
    expect(openExtent(closedSplit, "left")).toBe(0);
    expect(
      openExtent(P({ side: "left", open: true }, { side: "right", open: false }), "left"),
    ).toBe(PANEL_SIZE.tools.w + EXTENT_PAD);
  });
});

describe("compensazione della vista", () => {
  const open = (side: "left" | "right" | "top" | "bottom"): Panels =>
    P({ side, open: true }, { side: "right", open: false });

  it("aprendo a sinistra l'origine si sposta indietro, chiudendo avanti: i nodi restano fermi sullo schermo", () => {
    const ext = PANEL_SIZE.tools.w + EXTENT_PAD;
    expect(viewCompensation(closedSplit, open("left"))).toBe(-ext);
    expect(viewCompensation(open("left"), closedSplit)).toBe(ext);
    expect(compensate({ x: 10, y: 5, zoom: 2 }, closedSplit, open("left"))).toEqual({
      x: 10 - ext,
      y: 5,
      zoom: 2,
    });
  });

  it("a destra, sopra e sotto la vista non cambia", () => {
    for (const side of ["right", "top", "bottom"] as const) {
      expect(viewCompensation(closedSplit, open(side))).toBe(0);
      expect(viewCompensation(open(side), closedSplit)).toBe(0);
    }
    const view = { x: 3, y: 4, zoom: 1 };
    expect(compensate(view, closedSplit, open("top"))).toBe(view);
  });

  it("spostare un pannello aperto da sinistra a destra restituisce lo spazio", () => {
    expect(viewCompensation(open("left"), open("right"))).toBe(PANEL_SIZE.tools.w + EXTENT_PAD);
  });

  it("cambiare scheda in un gruppo a sinistra non sposta il canvas (stessa misura)", () => {
    const a = P({ side: "left", open: true }, { side: "left", open: false });
    const b = P({ side: "left", open: false }, { side: "left", open: true });
    expect(viewCompensation(a, b)).toBe(0);
  });

  it("un pannello aperto a sinistra che si unisce all'altro cambia misura e la vista compensa la differenza", () => {
    const alone = P({ side: "left", open: true }, { side: "right", open: false });
    const joined = P({ side: "left", open: true }, { side: "left", open: false });
    const grow = Math.max(PANEL_SIZE.tools.w, PANEL_SIZE.insp.w) - PANEL_SIZE.tools.w;
    expect(viewCompensation(alone, joined)).toBe(-grow);
  });
});

describe("tacche e schede", () => {
  it("il bordo più vicino a un punto", () => {
    const rect = { left: 0, right: 1000, top: 0, bottom: 600 };
    expect(nearestSide({ x: 10, y: 300 }, rect)).toBe("left");
    expect(nearestSide({ x: 990, y: 300 }, rect)).toBe("right");
    expect(nearestSide({ x: 500, y: 12 }, rect)).toBe("top");
    expect(nearestSide({ x: 500, y: 590 }, rect)).toBe("bottom");
  });

  it("due tacche sullo stesso bordo si affiancano", () => {
    expect(notchOffset(closedSplit, "tools")).toBe(0);
    const same = P({ side: "top", open: false }, { side: "top", open: false });
    expect(notchOffset(same, "tools")).toBeLessThan(0);
    expect(notchOffset(same, "insp")).toBeGreaterThan(0);
  });

  it("la tacca si nasconde se il pannello è aperto o se l'altro, sullo stesso bordo, è aperto (si raggiunge dalla scheda)", () => {
    expect(notchHidden(closedSplit, "tools")).toBe(false);
    expect(
      notchHidden(P({ side: "left", open: true }, { side: "right", open: false }), "tools"),
    ).toBe(true);
    const grouped = P({ side: "left", open: true }, { side: "left", open: false });
    expect(notchHidden(grouped, "insp")).toBe(true);
    expect(
      notchHidden(P({ side: "left", open: true }, { side: "right", open: false }), "insp"),
    ).toBe(false);
  });

  it("la scheda attiva è il pannello aperto sul bordo", () => {
    expect(activeTab(closedSplit, "left")).toBeNull();
    expect(activeTab(P({ side: "left", open: false }, { side: "left", open: true }), "left")).toBe(
      "insp",
    );
  });
});
```

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

114 righe

```ts
import { readFileSync } from "node:fs";
import { describe, expect, it } from "vitest";
import { resolveTokens } from "../../theme/__tests__/support";
import { PAIRS, measure, readTokens } from "../contrast";

const css = readFileSync(new URL("../tokens.css", import.meta.url), "utf8");
const prototype = readFileSync(
  new URL("../../../docs/prototype/isa-fusion-prototype.html", import.meta.url),
  "utf8",
);
const light = readTokens(css, resolveTokens("prototipo", "light"));
const dark = readTokens(css, resolveTokens("prototipo", "dark"));

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

/**
 * Deroghe note del tema «notte». Stessa disciplina bidirezionale di
 * KNOWN_EXCEPTIONS (src/theme/__tests__/checks.ts): `floor` è il rapporto
 * misurato; il test fallisce se peggiora e fallisce anche se la coppia torna
 * a rispettare la soglia senza che la deroga sia stata tolta a mano.
 * Avviso #F59E0B (colore richiesto dalla specifica) sul fondo chiaro: 2,1476:1.
 * NOTA: rivedere nella revisione di stile dopo la Fase 6.
 */
const NOTTE_EXCEPTIONS = new Map([["light: Indicatore ambra", { floor: 2.1476 }]]);

describe("tema notte: contrasto del canvas", () => {
  for (const mode of ["light", "dark"] as const) {
    const tokens = readTokens(css, resolveTokens("notte", mode));
    for (const pair of PAIRS) {
      const exception = NOTTE_EXCEPTIONS.get(`${mode}: ${pair.role}`);
      if (exception) {
        it(`${mode}: ${pair.role}: deroga nota (rivedere dopo la Fase 6), non deve peggiorare`, () => {
          const ratio = measure(tokens, pair);
          expect(ratio, "la coppia ora rispetta la soglia: togliere la deroga").toBeLessThan(
            pair.min,
          );
          expect(ratio).toBeGreaterThanOrEqual(exception.floor);
        });
        continue;
      }
      it(`${mode}: ${pair.role}: almeno ${pair.min}:1`, () => {
        expect(measure(tokens, pair)).toBeGreaterThanOrEqual(pair.min);
      });
    }
  }
});
```

### `src/etl-canvas/__tests__/toolbox-drop.test.ts`

115 righe

```ts
import { describe, expect, it } from "vitest";
import { nodeCenter } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
import type { InteractionController } from "../interaction";
import { loadCsvText } from "../panels/csv";
import { storeWith } from "./helpers";

function setup(): { store: EtlStore; c: InteractionController } {
  const store = storeWith();
  return { store, c: createInteractionController(store) };
}
const count = (s: EtlStore) => Object.keys(s.getState().graph.cards).length;
const centerOf = (s: EtlStore, id: string) => nodeCenter(s.getState().graph.cards[id]!);

describe("trascinamento dalla cassetta al canvas (un nodo esterno)", () => {
  it("sul vuoto crea un nodo isolato, in un passo di cronologia", () => {
    const { store, c } = setup();
    const n = count(store);
    c.hoverExternal({ component: "limit" }, { x: 900, y: 700 });
    expect(c.getUi().drop).toBeNull();
    expect(c.getUi().insertLink).toBeNull();
    const r = c.dropExternal({ component: "limit" }, { x: 900, y: 700 });
    expect(r?.ok).toBe(true);
    expect(count(store)).toBe(n + 1);
    expect(store.getState().graph.links).toEqual([]);
    expect(store.historySize().past).toBe(1);
    expect(c.getUi().drop).toBeNull();
  });

  it("sopra un box compatibile mostra il contorno di fusione e al rilascio si fonde", () => {
    const { store, c } = setup();
    const p = centerOf(store, "op-sort");
    c.hoverExternal({ component: "filter" }, p);
    expect(c.getUi().drop).toEqual({ id: "op-sort", outcome: "merge" });
    expect(c.getUi().hint).toBe("Rilascia per fondere direttamente nel box");
    const n = count(store);
    c.dropExternal({ component: "filter" }, p);
    expect(count(store)).toBe(n);
    expect(store.getState().graph.cards["op-sort"]!.components).toEqual(["sort", "filter"]);
    expect(c.getUi().drop).toBeNull();
    expect(c.getUi().hint).toBeNull();
  });

  it("un dataset della libreria sopra una lavorazione mostra il collegamento e si collega", () => {
    const { store, c } = setup();
    loadCsvText(store, "clienti.csv", "id,nome\n1,Acme\n2,Delta");
    const payload = { component: "dataset", libraryId: "lib-1" } as const;
    const p = centerOf(store, "op-join");
    c.hoverExternal(payload, p);
    expect(c.getUi().drop).toEqual({ id: "op-join", outcome: "link" });
    c.dropExternal(payload, p);
    const g = store.getState().graph;
    const created = Object.values(g.cards).find((k) => k.name === "clienti")!;
    expect(created).toBeDefined();
    expect(g.links).toContainEqual({ from: created.id, to: "op-join" });
  });

  it("una lavorazione sopra un dataset mostra il collegamento inverso", () => {
    const { store, c } = setup();
    c.hoverExternal({ component: "sort" }, centerOf(store, "ds1"));
    expect(c.getUi().drop).toEqual({ id: "ds1", outcome: "link-reverse" });
    c.dropExternal({ component: "sort" }, centerOf(store, "ds1"));
    expect(store.getState().graph.links.some((l) => l.from === "ds1")).toBe(true);
  });

  it("su un cavo dataset→lavorazione evidenzia il cavo e vi si inserisce", () => {
    const { store, c } = setup();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const pts = store.getRoutes()["ds1|op-join"]!.pts;
    const mid = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    c.hoverExternal({ component: "sort" }, mid);
    expect(c.getUi().insertLink).toBe("ds1|op-join");
    expect(c.getUi().hint).toContain("inserire");
    c.dropExternal({ component: "sort" }, mid);
    expect(store.getState().graph.links).not.toContainEqual({ from: "ds1", to: "op-join" });
    expect(store.getState().graph.links.some((l) => l.from === "ds1" && l.to !== "op-join")).toBe(
      true,
    );
  });

  it("un dataset non si inserisce in un cavo: nessuna anteprima", () => {
    const { store, c } = setup();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const pts = store.getRoutes()["ds1|op-join"]!.pts;
    c.hoverExternal({ component: "dataset" }, { x: (pts[0]!.x + pts[1]!.x) / 2, y: pts[0]!.y });
    expect(c.getUi().insertLink).toBeNull();
  });

  it("fuori dall'area non succede nulla e le anteprime si cancellano", () => {
    const { store, c } = setup();
    c.hoverExternal({ component: "filter" }, centerOf(store, "op-sort"));
    c.hoverExternal({ component: "filter" }, null);
    expect(c.getUi().drop).toBeNull();
    const n = count(store);
    expect(c.dropExternal({ component: "filter" }, null)).toBeNull();
    expect(count(store)).toBe(n);
    expect(store.historySize().past).toBe(0);
  });

  it("l'anteprima è la stessa del trascinamento tra nodi (stessa forma dello stato)", () => {
    const { store, c } = setup();
    c.hoverExternal({ component: "filter" }, centerOf(store, "op-sort"));
    const external = c.getUi().drop;
    c.hoverExternal({ component: "filter" }, null);
    const from = centerOf(store, "op-filter");
    c.down({ kind: "node", id: "op-filter" }, from);
    c.move({ x: from.x + 10, y: from.y });
    const to = centerOf(store, "op-sort");
    c.move(to);
    expect(c.getUi().drop).toEqual(external);
    c.cancel();
  });
});
```

### `src/etl-canvas/__tests__/toolbox.test.tsx`

187 righe

```tsx
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { META, SECTIONS } from "../../etl-core";
import { createEtlStore } from "../../etl-store";
import { createPanelActions } from "../panels/actions";
import { DockLayout } from "../panels/Dock";
import { InspectorShell } from "../panels/InspectorShell";
import { Toolbox } from "../panels/Toolbox";
import { loadCsvText } from "../panels/csv";
import { storeWith } from "./helpers";

const noop = () => {};
const render = (store = createEtlStore(), side: "left" | "top" = "left") =>
  renderToStaticMarkup(
    createElement(Toolbox, { store, side, onClose: noop, onItemPointerDown: noop }),
  );

/** Le sezioni nel markup, nell'ordine: id e voci (tipo). */
function parse(markup: string): { id: string; types: string[]; labels: string[] }[] {
  return markup
    .split(/<div class="ec-tb-sec(?: ec-open)?" data-sec/)
    .slice(1)
    .map((chunk) => ({
      id: /^="([^"]+)"/.exec(chunk)![1]!,
      types: [...chunk.matchAll(/data-type="([^"]+)"/g)].map((m) => m[1]!),
      labels: [...chunk.matchAll(/class="ec-pal-label">([^<]*)</g)].map((m) => m[1]!),
    }));
}

describe("la cassetta deriva dal catalogo di etl-core", () => {
  it("le sezioni e le voci corrispondono uno a uno a SECTIONS e META, nello stesso ordine", () => {
    const rendered = parse(render());
    // il catalogo, non una lista scritta nel test
    const expected = SECTIONS.map((s) => ({
      id: s.id,
      types: s.items ? [...s.items] : [],
      labels: s.items ? s.items.map((t) => META[t].label) : [],
    }));
    expect(rendered).toEqual(expected);
  });

  it("ogni operazione del catalogo compare in una sezione (e nessuna è inventata)", () => {
    const inToolbox = parse(render()).flatMap((s) => s.types);
    const catalog = (Object.keys(META) as (keyof typeof META)[]).filter((k) => k !== "dataset");
    expect(inToolbox.slice().sort()).toEqual(catalog.slice().sort());
  });

  it("ogni sezione ha il suo nome del catalogo ed è comprimibile", () => {
    const markup = render();
    for (const s of SECTIONS) expect(markup).toContain(`>${s.name}<`);
    expect(markup.match(/aria-expanded="true"/g)).toHaveLength(SECTIONS.length);
    expect(markup.match(/class="ec-tb-sec-head"/g)).toHaveLength(SECTIONS.length);
  });

  it("le voci delle operazioni hanno la famiglia di colore della loro sezione", () => {
    const markup = render();
    expect(markup).toMatch(/data-type="filter"[^>]*data-family="filter"/);
    expect(markup).toMatch(/data-type="join"[^>]*data-family="merge"/);
    expect(markup).toMatch(/data-type="exportOp"[^>]*data-family="output"/);
  });
});

describe("sezione Dataset e libreria", () => {
  it("senza dataset caricati: il pulsante di caricamento e il messaggio", () => {
    const markup = render();
    expect(markup).toContain("Carica dataset");
    expect(markup).toContain("Nessun dataset caricato");
    expect(markup).toContain('accept=".csv,.tsv,.txt"');
  });

  it("un CSV di prova produce le colonne e i tipi attesi e compare nella libreria e nella cassetta", () => {
    const store = createEtlStore();
    const csv = [
      "id;cliente;importo;data;peso",
      "1;Acme;10,5;2026-01-03;1.5",
      "2;Borealis;20;2026-01-04;2",
      "3;Acme;30,25;2026-02-01;3",
    ].join("\n");
    const out = loadCsvText(store, "vendite.csv", csv);
    expect(out.ok).toBe(true);
    expect(out.message).toBe("vendite.csv caricato: 5 colonne, 3 righe. Trascinalo sul canvas.");
    const lib = store.getState().library;
    expect(lib).toHaveLength(1);
    expect(lib[0]).toMatchObject({ id: "lib-1", name: "vendite", path: "vendite.csv", rows: 3 });
    expect(lib[0]!.columns.map((c) => [c.name, c.type])).toEqual([
      ["id", "integer"],
      ["cliente", "stringa"],
      ["importo", "numerico"],
      ["data", "data"],
      ["peso", "numerico"],
    ]);
    expect(lib[0]!.columns[1]!.values).toEqual(["Acme", "Borealis"]);
    // compare come voce trascinabile nella sezione Dataset
    const markup = render(store);
    expect(markup).toContain('data-lib="lib-1"');
    expect(markup).toContain(">vendite<");
    expect(markup).toContain("5 col · 3 righe");
    expect(markup).not.toContain("Nessun dataset caricato");
  });

  it("un file senza colonne leggibili non entra nella libreria e dà il messaggio del prototipo", () => {
    const store = createEtlStore();
    const out = loadCsvText(store, "vuoto.csv", "\n\n");
    expect(out).toEqual({ ok: false, message: "Il file non contiene colonne leggibili" });
    expect(store.getState().library).toEqual([]);
  });

  it("la libreria non conserva il contenuto del file, solo metadati", () => {
    const store = createEtlStore();
    loadCsvText(store, "a.csv", "x,y\n1,2\n3,4");
    const item = store.getState().library[0]!;
    expect(Object.keys(item).sort()).toEqual(["columns", "id", "name", "path", "rows"]);
  });
});

describe("orientamento e schede", () => {
  const content = {
    tools: () => createElement("div", { "data-x": "tools" }),
    insp: () => createElement("div", { "data-x": "insp" }),
  };
  const layout = (store: ReturnType<typeof createEtlStore>) =>
    renderToStaticMarkup(
      createElement(DockLayout, {
        store,
        actions: createPanelActions(store),
        canvas: createElement("div", { "data-x": "canvas" }),
        content,
      }),
    );

  it("a sinistra la cassetta è una colonna aperta; l'Inspector, chiuso a destra, ha la tacca visibile", () => {
    const markup = layout(createEtlStore());
    expect(markup).toMatch(/class="ec-panel ec-side-left ec-open"/);
    expect(markup).toMatch(/class="ec-panel ec-side-right"/);
    expect(markup).not.toContain("ec-horiz");
    expect(markup).not.toContain("ec-dock-tabs");
    // tacca della cassetta nascosta (aperta), quella dell'Inspector no
    expect(markup).toMatch(/ec-notch ec-notch-left ec-hidden/);
    expect(markup).toMatch(/ec-notch ec-notch-right"/);
  });

  it("sui bordi orizzontali il pannello è una fascia (ec-horiz) e l'area di lavoro cresce", () => {
    const store = createEtlStore();
    createPanelActions(store).moveTo("tools", "bottom");
    const markup = layout(store);
    expect(markup).toMatch(/class="ec-panel ec-side-bottom ec-open ec-horiz"/);
    expect(markup).toContain("height:calc(100% + 222px)");
  });

  it("due pannelli sullo stesso bordo mostrano le schede, con quella attiva evidenziata", () => {
    const store = createEtlStore();
    createPanelActions(store).moveTo("insp", "left");
    const markup = layout(store);
    expect(markup).toContain("ec-dock-tabs");
    expect(markup.match(/class="ec-dock-tab( ec-on)?"/g)).toHaveLength(4); // due schede in ciascuno dei due pannelli
    expect(markup).toMatch(/class="ec-dock-tab ec-on"[^>]*aria-pressed="true"/);
    expect(markup).toContain("ec-grouped");
  });

  it("un pannello chiuso non è raggiungibile da tastiera (inert)", () => {
    const markup = layout(createEtlStore());
    expect(markup).toMatch(/<aside[^>]*data-panel="insp"[^>]*inert/);
    expect(markup).not.toMatch(/<aside[^>]*data-panel="tools"[^>]*inert/);
  });
});

describe("guscio dell'Inspector", () => {
  it("mostra il nome del nodo selezionato, nient'altro", () => {
    const store = storeWith();
    store.dispatch({ type: "inspect", payload: { node: "op-join" } });
    const markup = renderToStaticMarkup(
      createElement(InspectorShell, { store, side: "right", onClose: noop }),
    );
    expect(markup).toContain(">Unisci (Join)<");
    expect(markup).not.toContain("<input");
    expect(markup).not.toContain("<select");
  });

  it("senza selezione dice che non c'è nessun nodo", () => {
    const markup = renderToStaticMarkup(
      createElement(InspectorShell, { store: storeWith(), side: "right", onClose: noop }),
    );
    expect(markup).toContain("Nessun nodo selezionato");
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

