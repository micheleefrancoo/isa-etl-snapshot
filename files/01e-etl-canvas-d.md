# 01e-etl-canvas-d.md

File in questo blocco:

- `src/etl-canvas/__tests__/keyboard.test.ts`
- `src/etl-canvas/__tests__/loop.test.ts`
- `src/etl-canvas/__tests__/menu.test.ts`
- `src/etl-canvas/__tests__/no-reroute.test.ts`
- `src/etl-canvas/__tests__/overlay-layout.test.ts`
- `src/etl-canvas/__tests__/panels-actions.test.ts`
- `src/etl-canvas/__tests__/panels-layout.test.ts`
- `src/etl-canvas/__tests__/render.test.ts`

---

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

### `src/etl-canvas/__tests__/menu.test.ts`

143 righe

```ts
import { readFileSync } from "node:fs";
import { describe, expect, it } from "vitest";
import { MENU_EDGE, MENU_GAP, MENU_MAX_W, menuBox, placeMenu } from "../inspector/menu";
import type { Box } from "../inspector/menu";

const hit = (a: Box, b: Box) =>
  a.x < b.x + b.w && b.x < a.x + a.w && a.y < b.y + b.h && b.y < a.y + a.h;
const WINDOWS = [
  { w: 1440, h: 900 },
  { w: 1280, h: 720 },
  { w: 1280, h: 600 },
  { w: 800, h: 500 },
];
const FIELD_SIZES = [
  { w: 150, h: 40 },
  { w: 232, h: 40 },
  { w: 300, h: 56 },
  { w: 600, h: 40 },
];
const HEIGHTS = [80, 240, 480, 900, 3000];

/** Posizioni del campo: gli angoli, i bordi e il centro della finestra (con un minimo di margine). */
function positions(win: { w: number; h: number }, f: { w: number; h: number }) {
  const xs = [0, 16, (win.w - f.w) / 2, win.w - f.w - 16, win.w - f.w];
  const ys = [0, 16, (win.h - f.h) / 2, win.h - f.h - 16, win.h - f.h];
  return xs.flatMap((x) => ys.map((y) => ({ x, y })));
}

describe("menu.ts: i token e le costanti restano allineati", () => {
  it("MENU_GAP, MENU_EDGE e MENU_MAX_W sono quelli di layout-tokens.css", () => {
    const css = readFileSync(new URL("../../theme/layout-tokens.css", import.meta.url), "utf8");
    const px = (name: string) => Number(new RegExp(`--isa-${name}:\\s*(\\d+)px`).exec(css)?.[1]);
    expect(px("menu-gap")).toBe(MENU_GAP);
    expect(px("menu-edge")).toBe(MENU_EDGE);
    expect(px("menu-max-w")).toBe(MENU_MAX_W);
  });
});

describe("placeMenu: matrice posizione del campo × finestra × altezza del menu", () => {
  it("margine ≥ 16 px da ogni bordo della finestra, nessuna sovrapposizione col campo, distanza 8 px", () => {
    let n = 0;
    for (const win of WINDOWS) {
      for (const fs of FIELD_SIZES) {
        for (const pos of positions(win, fs)) {
          const field: Box = { ...pos, ...fs };
          // il campo deve stare nella finestra
          if (field.x + field.w > win.w || field.y + field.h > win.h) continue;
          for (const naturalHeight of HEIGHTS) {
            const p = placeMenu({ field, win, naturalHeight });
            const box = menuBox(p);
            const label = `${win.w}×${win.h} campo ${JSON.stringify(field)} altezza ${naturalHeight}`;
            if (box.h > 0) {
              expect(box.x, label).toBeGreaterThanOrEqual(MENU_EDGE);
              expect(box.y, label).toBeGreaterThanOrEqual(MENU_EDGE - 0.001);
              expect(box.x + box.w, label).toBeLessThanOrEqual(win.w - MENU_EDGE + 0.001);
              expect(box.y + box.h, label).toBeLessThanOrEqual(win.h - MENU_EDGE + 0.001);
              expect(hit(box, field), label).toBe(false);
              const gap =
                p.side === "below" ? box.y - (field.y + field.h) : field.y - (box.y + box.h);
              expect(gap, label).toBeGreaterThanOrEqual(MENU_GAP - 0.001);
            }
            n++;
          }
        }
      }
    }
    expect(n).toBeGreaterThan(500);
  });

  it("si apre dal lato con più spazio", () => {
    const win = { w: 1440, h: 900 };
    const top = placeMenu({ field: { x: 100, y: 50, w: 232, h: 40 }, win, naturalHeight: 200 });
    expect(top.side).toBe("below");
    const bottom = placeMenu({ field: { x: 100, y: 800, w: 232, h: 40 }, win, naturalHeight: 200 });
    expect(bottom.side).toBe("above");
    // anche se in basso l'altezza naturale entrerebbe: prevale il lato con più spazio
    const mid = placeMenu({ field: { x: 100, y: 500, w: 232, h: 40 }, win, naturalHeight: 150 });
    expect(mid.side).toBe("above");
  });

  it("se l'altezza naturale non entra, l'altezza massima è lo spazio disponibile e il menu scorre", () => {
    const win = { w: 1280, h: 600 };
    const field: Box = { x: 100, y: 200, w: 232, h: 40 };
    const below = win.h - 240 - MENU_GAP - MENU_EDGE;
    const p = placeMenu({ field, win, naturalHeight: 2000 });
    expect(p.side).toBe("below");
    expect(p.maxHeight).toBe(below);
    expect(p.height).toBe(below);
    expect(p.scrolls).toBe(true);
    const small = placeMenu({ field, win, naturalHeight: 100 });
    expect(small.scrolls).toBe(false);
    expect(small.height).toBe(100);
    expect(small.maxHeight).toBe(below);
  });

  it("apertura verso l'alto: il bordo inferiore del menu sta 8 px sopra il campo", () => {
    const field: Box = { x: 100, y: 700, w: 232, h: 40 };
    const p = placeMenu({ field, win: { w: 1440, h: 900 }, naturalHeight: 200 });
    expect(p.side).toBe("above");
    expect(p.top + p.height).toBe(field.y - MENU_GAP);
  });

  it("orizzontalmente si sposta a sinistra quanto basta per tenere 16 px a destra", () => {
    const win = { w: 1440, h: 900 };
    const near = placeMenu({
      field: { x: 1300, y: 100, w: 120, h: 40 },
      win,
      naturalHeight: 200,
      naturalWidth: 300,
    });
    expect(near.width).toBe(300);
    expect(near.left + near.width).toBe(win.w - MENU_EDGE);
    // lontano dal bordo non si sposta
    const far = placeMenu({ field: { x: 100, y: 100, w: 232, h: 40 }, win, naturalHeight: 200 });
    expect(far.left).toBe(100);
    // a sinistra non scende sotto il margine
    const left = placeMenu({ field: { x: 0, y: 100, w: 232, h: 40 }, win, naturalHeight: 200 });
    expect(left.left).toBe(MENU_EDGE);
  });

  it("larghezza: almeno quella del campo, al massimo min(420, finestra − 32)", () => {
    const win = { w: 1440, h: 900 };
    const field: Box = { x: 100, y: 100, w: 232, h: 40 };
    expect(placeMenu({ field, win, naturalHeight: 100 }).width).toBe(232);
    expect(placeMenu({ field, win, naturalHeight: 100, naturalWidth: 100 }).width).toBe(232);
    expect(placeMenu({ field, win, naturalHeight: 100, naturalWidth: 380 }).width).toBe(380);
    expect(placeMenu({ field, win, naturalHeight: 100, naturalWidth: 900 }).width).toBe(MENU_MAX_W);
    const narrow = { w: 360, h: 600 };
    expect(placeMenu({ field, win: narrow, naturalHeight: 100, naturalWidth: 900 }).width).toBe(
      narrow.w - 2 * MENU_EDGE,
    );
  });

  it("è una funzione pura: stesso ingresso, stessa uscita, ingresso intatto", () => {
    const input = Object.freeze({
      field: Object.freeze({ x: 10, y: 20, w: 200, h: 40 }),
      win: Object.freeze({ w: 1000, h: 700 }),
      naturalHeight: 300,
    });
    expect(placeMenu(input)).toEqual(placeMenu(input));
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

### `src/etl-canvas/__tests__/overlay-layout.test.ts`

158 righe

```ts
import { describe, expect, it } from "vitest";
import type { PanelKey, Side } from "../../etl-store";
import {
  HINT_HEIGHT,
  MINIMAP_COMPACT,
  MINIMAP_SIZE,
  ZOOM_SIZE,
  intersects,
  overlayLayout,
} from "../panels/overlayLayout";
import type { OverlayInput, OverlayLayout, Rect } from "../panels/overlayLayout";

const SIDES: readonly Side[] = ["left", "right", "top", "bottom"];
/** Aree del canvas di finestre 1440×900, 1280×720 e 1280×600 con un pannello o la barra, più aree strette. */
const AREAS = [
  { w: 1384, h: 690 },
  { w: 1384, h: 516 },
  { w: 1384, h: 246 },
  { w: 1232, h: 510 },
  { w: 1232, h: 288 },
  { w: 1232, h: 168 },
  { w: 720, h: 516 },
  { w: 560, h: 300 },
  { w: 420, h: 260 },
  { w: 300, h: 200 },
];

function scenario(
  area: { w: number; h: number },
  open: PanelKey | null,
  tools: Side,
  insp: Side,
): OverlayInput {
  const sideOf = (k: PanelKey) => (k === "tools" ? tools : insp);
  return {
    area,
    openSide: open ? sideOf(open) : null,
    notches: (["tools", "insp"] as const).map((key) => ({
      key,
      side: sideOf(key),
      offset: tools === insp ? (key === "tools" ? -40 : 40) : 0,
      visible: key !== open && !(open && tools === insp),
    })),
  };
}

/** Tutti gli ingombri non nulli: minimappa, zoom, suggerimento e tacche visibili. */
function rectsOf(input: OverlayInput, l: OverlayLayout): { name: string; r: Rect }[] {
  const out: { name: string; r: Rect }[] = [{ name: "zoom", r: l.zoom }];
  if (l.minimap.rect) out.push({ name: "minimappa", r: l.minimap.rect });
  if (l.hint) out.push({ name: "suggerimento", r: l.hint });
  for (const n of input.notches)
    if (n.visible) out.push({ name: `tacca ${n.key}`, r: l.notches[n.key] });
  return out;
}

describe("overlayLayout: matrice bordi × misure", () => {
  it("nessun widget interseca un altro widget o una tacca, e stanno tutti dentro l'area", () => {
    let cases = 0;
    for (const area of AREAS) {
      for (const open of [null, "tools", "insp"] as const) {
        for (const tools of SIDES) {
          for (const insp of SIDES) {
            const input = scenario(area, open, tools, insp);
            const l = overlayLayout(input);
            const rects = rectsOf(input, l);
            const label = `${area.w}×${area.h} aperto=${open} cassetta=${tools} inspector=${insp}`;
            for (const { name, r } of rects) {
              if (name.startsWith("tacca")) continue;
              expect(
                r.x >= 0 && r.y >= 0 && r.x + r.w <= area.w && r.y + r.h <= area.h,
                `${label}: ${name} fuori dall'area`,
              ).toBe(true);
            }
            for (let i = 0; i < rects.length; i++) {
              for (let j = i + 1; j < rects.length; j++) {
                expect(
                  intersects(rects[i]!.r, rects[j]!.r),
                  `${label}: ${rects[i]!.name} tocca ${rects[j]!.name}`,
                ).toBe(false);
              }
            }
            cases++;
          }
        }
      }
    }
    expect(cases).toBe(AREAS.length * 3 * 16);
  });
});

describe("overlayLayout: regole di posizione", () => {
  const roomy = { w: 1384, h: 690 };

  it("senza pannello aperto: minimappa in basso a sinistra, zoom in basso a destra", () => {
    const l = overlayLayout(scenario(roomy, null, "left", "right"));
    expect(l.minimap.corner).toBe("bl");
    expect(l.minimap.compact).toBe(false);
    expect(l.minimap.rect).toMatchObject({ w: MINIMAP_SIZE.w, h: MINIMAP_SIZE.h });
    expect(l.minimap.rect!.y + l.minimap.rect!.h).toBeLessThan(roomy.h);
    expect(l.zoom).toMatchObject({ w: ZOOM_SIZE.w, h: ZOOM_SIZE.h });
    expect(l.zoom.x + l.zoom.w).toBeLessThan(roomy.w);
    expect(l.zoom.y).toBeGreaterThan(roomy.h / 2);
  });

  it("con il pannello in basso la minimappa va in alto a sinistra; con il pannello in alto resta in basso a sinistra", () => {
    const bottom = overlayLayout(scenario(roomy, "tools", "bottom", "right"));
    expect(bottom.minimap.corner).toBe("tl");
    expect(bottom.minimap.rect!.y).toBeLessThan(roomy.h / 2);
    const top = overlayLayout(scenario(roomy, "tools", "top", "right"));
    expect(top.minimap.corner).toBe("bl");
    for (const side of ["left", "right"] as const) {
      expect(overlayLayout(scenario(roomy, "tools", side, "right")).minimap.corner).toBe("bl");
    }
  });

  it("i controlli di zoom restano in basso a destra con ogni pannello", () => {
    for (const side of SIDES) {
      const l = overlayLayout(scenario(roomy, "tools", side, "right"));
      expect(l.zoom.x + l.zoom.w).toBeGreaterThan(roomy.w - 40);
      expect(l.zoom.y + l.zoom.h).toBeGreaterThan(roomy.h - 40);
    }
  });

  it("area troppo piccola: la minimappa passa all'angolo opposto, poi diventa un pulsante compatto", () => {
    // con le due tacche sul bordo inferiore e lo zoom, la minimappa in basso a sinistra non ci sta
    const tight = { w: 330, h: 150 };
    const l = overlayLayout(scenario(tight, null, "bottom", "bottom"));
    expect(l.minimap.compact || l.minimap.corner !== "bl").toBe(true);
    // area minuscola: compatta
    const tiny = overlayLayout(scenario({ w: 260, h: 200 }, null, "left", "right"));
    expect(tiny.minimap.compact || tiny.minimap.rect === null).toBe(true);
    if (tiny.minimap.rect) {
      expect(tiny.minimap.rect.w).toBe(MINIMAP_COMPACT);
      expect(tiny.minimap.expanded).toMatchObject({ w: MINIMAP_SIZE.w, h: MINIMAP_SIZE.h });
    }
  });

  it("il suggerimento sta al centro, in basso o in alto, e non tocca nulla", () => {
    for (const area of AREAS) {
      const input = scenario(area, null, "bottom", "top");
      const l = overlayLayout(input);
      if (!l.hint) continue;
      expect(l.hint.h).toBe(HINT_HEIGHT);
      expect(Math.abs(l.hint.x + l.hint.w / 2 - area.w / 2)).toBeLessThanOrEqual(1);
    }
  });

  it("gli ingombri dei widget diventano margini di sicurezza dell'area visibile", () => {
    const l = overlayLayout(scenario(roomy, null, "left", "right"));
    const total = l.insets.top + l.insets.right + l.insets.bottom + l.insets.left;
    expect(total).toBeGreaterThan(0);
    // i margini non portano via più di metà dell'area in nessuna direzione
    expect(l.insets.left + l.insets.right).toBeLessThan(roomy.w / 2);
    expect(l.insets.top + l.insets.bottom).toBeLessThan(roomy.h / 2);
  });
});
```

### `src/etl-canvas/__tests__/panels-actions.test.ts`

242 righe

```ts
import { describe, expect, it } from "vitest";
import { createEtlStore, fromSaved, initialState, parseSaved, toSaved } from "../../etl-store";
import type { EtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
import type { InteractionController } from "../interaction";
import { createPanelActions, followInspector } from "../panels/actions";
import { storeWith } from "./helpers";

const panels = (s: EtlStore) => s.getState().panels;
const view = (s: EtlStore) => s.getState().view;
const open = (s: EtlStore) => [panels(s).tools.open, panels(s).insp.open];

describe("aprire, chiudere, spostare un pannello", () => {
  it("lo stato iniziale è quello del prototipo: cassetta aperta a sinistra, Inspector chiuso a destra", () => {
    const s = createEtlStore();
    expect(panels(s)).toEqual({
      tools: { side: "left", open: true },
      insp: { side: "right", open: false },
    });
  });

  it("le azioni sui pannelli non toccano mai la vista (la tiene visibile keepVisible, a parte)", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    a.close("tools");
    a.open("tools");
    a.moveTo("tools", "top");
    a.moveTo("tools", "bottom");
    a.moveTo("insp", "left");
    a.open("insp");
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

  it("due pannelli sullo stesso bordo diventano schede: se ne apre uno alla volta; separati, tornano due pannelli", () => {
    const s = createEtlStore();
    const a = createPanelActions(s);
    a.moveTo("insp", "left"); // si apre sullo stesso bordo della cassetta
    expect(panels(s).insp).toEqual({ side: "left", open: true });
    expect(panels(s).tools).toEqual({ side: "left", open: false }); // l'altra scheda si chiude
    a.open("tools");
    expect(open(s)).toEqual([true, false]);
    a.moveTo("insp", "right");
    expect(panels(s).insp.side).toBe("right");
    expect(panels(s).tools.side).toBe("left");
  });

  it("un solo pannello aperto alla volta anche su bordi diversi, per ogni combinazione di bordi", () => {
    const sides = ["left", "right", "top", "bottom"] as const;
    for (const a of sides) {
      for (const b of sides) {
        const s = createEtlStore();
        const act = createPanelActions(s);
        act.moveTo("tools", a);
        act.moveTo("insp", b);
        act.open("insp"); // sullo stesso bordo di prima il solo spostamento non apre: come il clic sulla tacca
        expect(open(s)).toEqual([false, true]);
        act.open("tools");
        expect(open(s)).toEqual([true, false]);
        act.open("insp");
        expect(open(s)).toEqual([false, true]);
      }
    }
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

/** Un clic: pressione e rilascio nello stesso punto. */
function click(c: InteractionController, id: string, shiftKey = false): void {
  c.down({ kind: "node", id }, { x: 304, y: 226, shiftKey });
  c.up({ x: 304, y: 226, shiftKey });
}

function setup() {
  const store = storeWith();
  const actions = createPanelActions(store);
  const controller = createInteractionController(store);
  const stop = followInspector(store, controller, actions);
  return { store, actions, controller, stop };
}

describe("l'Inspector segue la selezione: apertura solo al clic", () => {
  it("un clic su un nodo apre l'Inspector con il suo nome e chiude la cassetta; deselezionare chiude l'Inspector e riapre la cassetta", () => {
    const { store, controller, stop } = setup();
    expect(open(store)).toEqual([true, false]);
    click(controller, "op-join");
    expect(store.getState().inspector.nodeId).toBe("op-join");
    expect(open(store)).toEqual([false, true]);
    controller.key({ key: "Escape" });
    expect(store.getState().inspector.nodeId).toBeNull();
    expect(open(store)).toEqual([true, false]); // memoria di sostituzione
    stop();
  });

  it("non apre alla sola pressione", () => {
    const { store, controller, stop } = setup();
    controller.down({ kind: "node", id: "op-join" }, { x: 304, y: 226 });
    expect(open(store)).toEqual([true, false]);
    controller.up({ x: 304, y: 226 });
    expect(open(store)).toEqual([false, true]);
    stop();
  });

  it("non apre durante né dopo un trascinamento", () => {
    const { store, controller, stop } = setup();
    controller.down({ kind: "node", id: "op-join" }, { x: 304, y: 226 });
    controller.move({ x: 340, y: 260 });
    expect(open(store)).toEqual([true, false]);
    controller.up({ x: 340, y: 260 });
    expect(open(store)).toEqual([true, false]);
    stop();
  });

  it("non apre con un riquadro di selezione", () => {
    const { store, controller, stop } = setup();
    controller.down({ kind: "background" }, { x: 2, y: 2 });
    controller.move({ x: 2000, y: 2000 });
    controller.up({ x: 2000, y: 2000 });
    expect(store.getState().selection.length).toBeGreaterThan(1);
    expect(open(store)).toEqual([true, false]);
    stop();
  });

  it("non apre con una selezione multipla (Maiusc+clic)", () => {
    const { store, controller, stop } = setup();
    click(controller, "op-join", true);
    expect(open(store)).toEqual([true, false]);
    click(controller, "op-sort", true);
    expect(store.getState().selection).toHaveLength(2);
    expect(open(store)).toEqual([true, false]);
    stop();
  });

  it("non apre dopo un rilascio dalla cassetta: creare nodi in serie non fa sparire la cassetta", () => {
    const { store, controller, stop } = setup();
    for (let i = 0; i < 3; i++) {
      controller.hoverExternal({ component: "filter" }, { x: 900 + i * 30, y: 500 });
      controller.dropExternal({ component: "filter" }, { x: 900 + i * 30, y: 500 });
      expect(open(store)).toEqual([true, false]);
    }
    stop();
  });

  it("se l'utente lo chiude con un nodo selezionato, resta chiuso finché non si clicca di nuovo", () => {
    const { store, actions, controller, stop } = setup();
    click(controller, "op-join");
    expect(open(store)).toEqual([false, true]);
    actions.close("insp");
    expect(open(store)).toEqual([false, false]);
    click(controller, "op-sort");
    expect(open(store)).toEqual([false, true]);
    stop();
  });

  it("smettere di ascoltare lascia i pannelli come sono", () => {
    const { store, controller, stop } = setup();
    stop();
    click(controller, "op-join");
    expect(open(store)).toEqual([true, false]);
  });
});

describe("memoria di sostituzione", () => {
  it("se la cassetta era chiusa, la chiusura automatica dell'Inspector non la apre", () => {
    const { store, actions, controller, stop } = setup();
    actions.close("tools");
    click(controller, "op-join");
    expect(open(store)).toEqual([false, true]);
    controller.key({ key: "Escape" });
    expect(open(store)).toEqual([false, false]);
    stop();
  });

  it("qualunque azione esplicita sui pannelli azzera la memoria", () => {
    const explicit: [string, (a: ReturnType<typeof createPanelActions>) => void][] = [
      ["chiusura dell'Inspector", (a) => a.close("insp")],
      ["tacca o scheda: apertura dell'Inspector", (a) => a.open("insp")],
      ["tacca o scheda: apertura della cassetta", (a) => a.open("tools")],
      ["trascinamento della tacca", (a) => a.moveTo("insp", "top")],
    ];
    for (const [name, act] of explicit) {
      const { store, actions, controller, stop } = setup();
      click(controller, "op-join"); // la cassetta è stata sostituita: memoria attiva
      act(actions);
      const before = open(store);
      // un Inspector chiuso da deselezione non deve riaprire la cassetta se la memoria è azzerata
      actions.close("tools");
      controller.key({ key: "Escape" });
      expect(open(store), name).toEqual([false, false]);
      void before;
      stop();
    }
  });

  it("senza azioni esplicite la memoria resta: cassetta → Inspector → cassetta più volte", () => {
    const { store, controller, stop } = setup();
    for (let i = 0; i < 3; i++) {
      click(controller, "op-join");
      expect(open(store)).toEqual([false, true]);
      controller.key({ key: "Escape" });
      expect(open(store)).toEqual([true, false]);
    }
    stop();
  });
});
```

### `src/etl-canvas/__tests__/panels-layout.test.ts`

259 righe

```ts
import { describe, expect, it } from "vitest";
import type { Card } from "../../etl-core";
import type { Panels, View } from "../../etl-store";
import {
  EXTENT_PAD,
  MIN_CANVAS_HEIGHT,
  PANEL_HEIGHT_CAP,
  cappedPanelHeight,
  PANEL_SIZE,
  activeTab,
  isGrouped,
  nearestSide,
  notchHidden,
  notchOffset,
  openExtent,
  panelExtent,
  panelSize,
  keepVisible,
  visibleIds,
} from "../panels/layout";
import { createPanelActions } from "../panels/actions";
import { storeWith } from "./helpers";

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
    expect(openExtent(top, "top")).toBe(PANEL_SIZE.tools.h + EXTENT_PAD);
    expect(openExtent(closedSplit, "top")).toBe(0);
  });

  it("due pannelli come schede in alto o in basso usano l'altezza maggiore dei due", () => {
    for (const side of ["top", "bottom"] as const) {
      const grouped = P({ side, open: true }, { side, open: false });
      const h = Math.max(PANEL_SIZE.tools.h, PANEL_SIZE.insp.h);
      expect(panelSize(grouped, "tools").h).toBe(h);
      expect(panelSize(grouped, "insp").h).toBe(h);
      expect(openExtent(grouped, side)).toBe(h + EXTENT_PAD);
    }
  });

  it("l'altezza minima del canvas è una misura positiva, definita in layout.ts", () => {
    expect(MIN_CANVAS_HEIGHT).toBeGreaterThan(0);
  });

  it("solo i pannelli aperti sottraggono spazio", () => {
    expect(openExtent(closedSplit, "left")).toBe(0);
    expect(
      openExtent(P({ side: "left", open: true }, { side: "right", open: false }), "left"),
    ).toBe(PANEL_SIZE.tools.w + EXTENT_PAD);
  });
});

describe("spinta e visibilità dei nodi (keepVisible)", () => {
  const card = (id: string, x: number, y: number): Card => ({ id, x, y }) as Card;
  const view0 = { x: 0, y: 0, zoom: 1 };
  const NOINS = { top: 0, right: 0, bottom: 0, left: 0 };
  const vis = (w: number, h: number, insets = NOINS) => ({ size: { w, h }, insets });
  /** Nodi sparsi su tutta un'area di 1000 × 600. */
  const cards = [
    card("a", 20, 20),
    card("b", 400, 20),
    card("c", 880, 20),
    card("d", 20, 480),
    card("e", 880, 480),
    card("m", 450, 250),
  ];
  const fullyIn = (ids: readonly string[], v: View, area: ReturnType<typeof vis>) =>
    ids.every((id) => visibleIds(cards, v, area).includes(id));

  it("se tutto resta visibile la vista non cambia (stessa istanza)", () => {
    const v = keepVisible(cards, view0, vis(1000, 600), vis(1000, 600));
    expect(v).toBe(view0);
    expect(keepVisible(cards, view0, vis(1000, 600), vis(1200, 800))).toBe(view0);
  });

  it("per ogni bordo i nodi prima interamente visibili restano interamente visibili: nessuno è coperto dal pannello", () => {
    const prev = vis(1000, 600);
    // a ciascun bordo il pannello toglie spazio da un lato dell'area
    const shrink: Record<string, { w: number; h: number }> = {
      left: { w: 1000 - (PANEL_SIZE.tools.w + EXTENT_PAD), h: 600 },
      right: { w: 1000 - (PANEL_SIZE.tools.w + EXTENT_PAD), h: 600 },
      top: { w: 1000, h: 600 - (PANEL_SIZE.tools.h + EXTENT_PAD) },
      bottom: { w: 1000, h: 600 - (PANEL_SIZE.tools.h + EXTENT_PAD) },
    };
    const before = visibleIds(cards, view0, prev);
    expect(before).toHaveLength(cards.length);
    for (const side of ["left", "right", "top", "bottom"]) {
      const next = vis(shrink[side]!.w, shrink[side]!.h);
      const v = keepVisible(cards, view0, prev, next);
      expect(v.zoom, side).toBe(1);
      // l'insieme entra: tutti dentro la nuova area
      const bbox = { w: 880 + 88 - 20, h: 480 + 88 + 26 - 20 };
      if (bbox.w <= next.size.w && bbox.h <= next.size.h)
        expect(fullyIn(before, v, next), side).toBe(true);
    }
  });

  it("lo scorrimento è il minimo necessario, solo sull'asse che serve", () => {
    // il bordo destro avanza di 300: il nodo più a destra (x2 = 968) deve rientrare in 700
    const v = keepVisible(cards, view0, vis(1000, 600), vis(700, 600));
    expect(v.y).toBe(0);
    // l'insieme dei nodi è largo 948 (20→968): non entra in 700, si allinea a sinistra (bordo di partenza)
    expect(v.x).toBe(-20 + 0); // x1 minimo = 20 → 0
  });

  it("insieme che entra: scorre del minimo", () => {
    const few = [card("p", 20, 20), card("q", 440, 20)]; // da x = 20 a x = 528
    const next = vis(520, 600);
    const v = keepVisible(few, view0, vis(1000, 600), next);
    expect(v.x).toBe(-8); // 528 - 520: solo quanto serve
    expect(v.y).toBe(0);
    expect(visibleIds(few as Card[], v, next)).toEqual(["p", "q"]);
  });

  it("insieme troppo largo: si allinea al bordo di partenza (sinistra e alto) e il resto resta raggiungibile", () => {
    const next = vis(300, 200, { top: 10, right: 0, bottom: 0, left: 30 });
    const v = keepVisible(cards, view0, vis(1000, 600), next);
    const boxes = cards.map((c) => ({ x: c.x + v.x, y: c.y + v.y }));
    expect(Math.min(...boxes.map((b) => b.x))).toBe(30); // inizia dal margine sinistro sicuro
    expect(Math.min(...boxes.map((b) => b.y))).toBe(10); // e da quello superiore
    expect(v.zoom).toBe(1);
  });

  it("il margine di sicurezza tiene fuori i nodi dagli ingombri dei widget", () => {
    const insets = { top: 0, right: 0, bottom: 100, left: 0 };
    const near = [card("n", 100, 460)]; // y1 = 460, y2 = 460 + 88 + 26 = 574: dentro 600, fuori da 500
    expect(visibleIds(near, view0, vis(1000, 600))).toEqual(["n"]);
    expect(visibleIds(near, view0, vis(1000, 600, insets))).toEqual([]);
  });

  it("i nodi non interamente visibili prima non fanno scorrere la vista", () => {
    const off = [card("x", 1500, 100), card("y", 100, 100)];
    const v = keepVisible(off, view0, vis(1000, 600), vis(800, 600));
    expect(v).toBe(view0); // y resta dentro; x non contava
  });

  it("lo zoom non cambia mai", () => {
    const z = { x: 10, y: 20, zoom: 1.7 };
    for (const [w, h] of [
      [300, 300],
      [700, 400],
      [1200, 900],
    ] as const) {
      expect(keepVisible(cards, z, vis(1000, 600), vis(w, h)).zoom).toBe(1.7);
    }
  });
});

describe("le posizioni nel mondo non cambiano mai (Libero e Organizzato)", () => {
  for (const mode of ["free", "grid"] as const) {
    it(`modalità ${mode}: aprire, chiudere e spostare i pannelli e tenere visibili i nodi non tocca il grafo`, () => {
      const store = storeWith();
      store.dispatch({ type: "setMode", payload: { mode } });
      const graph = store.getState().graph;
      const actions = createPanelActions(store);
      const prev = { size: { w: 1000, h: 600 }, insets: { top: 0, right: 0, bottom: 0, left: 0 } };
      for (const side of ["right", "top", "bottom", "left"] as const) {
        actions.moveTo("tools", side);
        const st = store.getState();
        const v = keepVisible(Object.values(st.graph.cards), st.view, prev, {
          size: { w: 700, h: 300 },
          insets: prev.insets,
        });
        store.dispatch({ type: "setView", payload: { x: v.x, y: v.y } });
        actions.close("tools");
      }
      expect(store.getState().graph).toBe(graph); // stessa istanza: nessun nodo si è mosso
      expect(store.getState().view.zoom).toBe(1);
    });
  }
});

describe("schede in alto e in basso", () => {
  it("due pannelli come schede usano l'altezza maggiore dei due, anche per la spinta", () => {
    for (const side of ["top", "bottom"] as const) {
      const grouped = P({ side, open: false }, { side, open: true });
      const h = Math.max(PANEL_SIZE.tools.h, PANEL_SIZE.insp.h);
      expect(panelSize(grouped, "tools").h).toBe(h);
      expect(openExtent(grouped, side)).toBe(h + EXTENT_PAD);
    }
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

describe("tetto all'altezza dei pannelli orizzontali (Fase 6b.1)", () => {
  it("min(altezza propria, 45% dell'altezza disponibile)", () => {
    expect(PANEL_HEIGHT_CAP).toBe(0.45);
    expect(cappedPanelHeight(300, 1000)).toBe(300); // 450 > 300: resta la propria
    expect(cappedPanelHeight(300, 600)).toBe(270);
    expect(cappedPanelHeight(206, 438)).toBe(197); // 1280 × 600: 45% di 438, arrotondato per difetto
    expect(cappedPanelHeight(300, 738)).toBe(300); // 900: il tetto è 332
    expect(cappedPanelHeight(300, 558)).toBe(251); // 720: il tetto è 251
    expect(cappedPanelHeight(300, 0)).toBe(0);
  });

  it("non supera mai il 45% e non è mai negativo", () => {
    for (const own of [100, 206, 300, 900]) {
      for (const avail of [0, 100, 438, 558, 738, 2000]) {
        const h = cappedPanelHeight(own, avail);
        expect(h).toBeLessThanOrEqual(own);
        expect(h).toBeLessThanOrEqual(avail * 0.45 + 0.0001);
        expect(h).toBeGreaterThanOrEqual(0);
      }
    }
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

