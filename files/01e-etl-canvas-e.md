# 01e-etl-canvas-e.md

File in questo blocco:

- `src/etl-canvas/__tests__/interaction.test.ts`
- `src/etl-canvas/__tests__/keyboard.test.ts`
- `src/etl-canvas/__tests__/loop.test.ts`
- `src/etl-canvas/__tests__/menu.test.ts`
- `src/etl-canvas/__tests__/no-reroute.test.ts`
- `src/etl-canvas/__tests__/overlay-layout.test.ts`

---

### `src/etl-canvas/__tests__/interaction.test.ts`

428 righe

```ts
import { describe, expect, it } from "vitest";
import { CARD, DRAG_THRESHOLD_PX, nodeCenter } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { createInteractionController } from "../interaction";
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
    c.move({ x: p.x + DRAG_THRESHOLD_PX - 1, y: p.y });
    expect(store.isGesturing()).toBe(false);
    c.up({ x: p.x + DRAG_THRESHOLD_PX - 1, y: p.y });
    expect(store.getState().selection).toEqual(["op-sort"]);
    expect(store.getState().inspector.nodeId).toBe("op-sort");
    expect(store.getState().graph.cards["op-sort"]).toMatchObject({ x: 260, y: 338 });
    expect(steps(store)).toBe(0);
  });

  it("esattamente alla soglia (5 px) il gesto parte", () => {
    const { store, c } = setup();
    const p = center(store, "op-sort");
    c.down({ kind: "node", id: "op-sort" }, p);
    c.move({ x: p.x + DRAG_THRESHOLD_PX, y: p.y });
    expect(store.isGesturing()).toBe(true);
    c.cancel();
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

describe("clic su un cavo", () => {
  function withLink() {
    const { store, c } = setup();
    store.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    const pts = store.getRoutes()["ds1|op-join"]!.pts;
    const mid = { x: (pts[0]!.x + pts[1]!.x) / 2, y: (pts[0]!.y + pts[1]!.y) / 2 };
    return { store, c, mid };
  }

  it("un click elimina il collegamento (deleteLink) in un solo passo; l'output a valle sparisce con lui", () => {
    const { store, c, mid } = withLink();
    const before = steps(store);
    expect(c.down({ kind: "background" }, mid)).toBe(true);
    expect(c.getUi().marquee).toBeNull();
    c.up(mid);
    const links = store.getState().graph.links;
    expect(links).not.toContainEqual({ from: "ds1", to: "op-join" });
    expect(links.some((l) => l.from === "op-join")).toBe(false);
    expect(steps(store)).toBe(before + 1);
    store.undo();
    expect(store.getState().graph.links).toContainEqual({ from: "ds1", to: "op-join" });
  });

  it("un trascinamento che parte dal cavo non lo elimina", () => {
    const { store, c, mid } = withLink();
    c.down({ kind: "background" }, mid);
    c.move({ x: mid.x + 20, y: mid.y + 20 });
    c.up({ x: mid.x + 20, y: mid.y + 20 });
    expect(store.getState().graph.links).toContainEqual({ from: "ds1", to: "op-join" });
  });

  it("con lo spazio premuto non elimina; lontano dal cavo parte il riquadro", () => {
    const { store, c, mid } = withLink();
    c.setSpace(true);
    expect(c.down({ kind: "background" }, mid)).toBe(false);
    c.setSpace(false);
    expect(c.down({ kind: "background" }, { x: mid.x, y: mid.y + 200 })).toBe(true);
    c.move({ x: mid.x + 30, y: mid.y + 240 });
    expect(c.getUi().marquee).not.toBeNull();
    c.cancel();
    expect(store.getState().graph.links).toContainEqual({ from: "ds1", to: "op-join" });
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

221 righe

```ts
import { describe, expect, it } from "vitest";
import type { PanelKey, Side } from "../../etl-store";
import {
  HINT_HEIGHT,
  MINIMAP_COMPACT,
  MINIMAP_SIZE,
  NOTCH_CLEARANCE,
  NOTCH_SHORT,
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

describe("le tacche stanno nell'area sicura (Fase 6b.2, Passo 0)", () => {
  const area = { w: 1384, h: 690 };
  const base = (
    visible: { tools: boolean; insp: boolean },
    tools: Side,
    insp: Side,
  ): OverlayInput => ({
    area,
    openSide: null,
    notches: [
      { key: "tools", side: tools, offset: 0, visible: visible.tools },
      { key: "insp", side: insp, offset: 0, visible: visible.insp },
    ],
  });

  it("il respiro è 16 px e ogni tacca visibile riserva la sua larghezza più il respiro sul suo bordo", () => {
    expect(NOTCH_CLEARANCE).toBe(16);
    const l = overlayLayout(base({ tools: true, insp: true }, "left", "right"));
    expect(l.insets.left).toBeGreaterThanOrEqual(NOTCH_SHORT + NOTCH_CLEARANCE);
    expect(l.insets.right).toBeGreaterThanOrEqual(NOTCH_SHORT + NOTCH_CLEARANCE);
  });

  it("per ciascun bordo", () => {
    for (const side of SIDES) {
      const l = overlayLayout(
        base({ tools: true, insp: false }, side, side === "left" ? "right" : "left"),
      );
      expect(l.insets[side], side).toBeGreaterThanOrEqual(NOTCH_SHORT + NOTCH_CLEARANCE);
    }
  });

  it("una tacca nascosta (pannello aperto) non riserva nulla: stessi ingombri che senza tacche", () => {
    const hidden = overlayLayout(base({ tools: false, insp: false }, "left", "right"));
    const none = overlayLayout({ area, openSide: null, notches: [] });
    expect(hidden.insets).toEqual(none.insets);
  });

  it("un nodo al limite dell'area sicura dista dalla tacca almeno 16 px", () => {
    for (const [tools, insp] of [
      ["left", "right"],
      ["top", "bottom"],
      ["right", "left"],
    ] as const) {
      const input = base({ tools: true, insp: true }, tools, insp);
      const l = overlayLayout(input);
      for (const n of Object.values(l.notches)) {
        // la fascia dei nodi (area meno gli ingombri) non tocca la tacca più vicino di 16 px
        const safe = {
          x1: l.insets.left,
          y1: l.insets.top,
          x2: area.w - l.insets.right,
          y2: area.h - l.insets.bottom,
        };
        const gapX = Math.max(0, n.x - safe.x2, safe.x1 - (n.x + n.w));
        const gapY = Math.max(0, n.y - safe.y2, safe.y1 - (n.y + n.h));
        expect(Math.hypot(gapX, gapY)).toBeGreaterThanOrEqual(NOTCH_CLEARANCE - 1e-9);
      }
    }
  });
});
```

