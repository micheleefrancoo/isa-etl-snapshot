# 01e-etl-canvas-d.md

File in questo blocco:

- `src/etl-canvas/__tests__/inspector.test.tsx`
- `src/etl-canvas/__tests__/interaction.test.ts`
- `src/etl-canvas/__tests__/keyboard.test.ts`
- `src/etl-canvas/__tests__/loop.test.ts`
- `src/etl-canvas/__tests__/menu.test.ts`
- `src/etl-canvas/__tests__/no-reroute.test.ts`

---

### `src/etl-canvas/__tests__/inspector.test.tsx`

418 righe

```tsx
import { createElement } from "react";
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";
import { MULTI_DEFS, createValuesField, defaultParams, outputOf } from "../../etl-core";
import type { Card, ColumnDef, ComponentId, OperationType, Params } from "../../etl-core";
import { createEtlStore } from "../../etl-store";
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

  it("filtro e join: solo la nota sulle condizioni", () => {
    for (const type of ["filter", "join"] as ComponentId[]) {
      const store = sceneWith([type]);
      connect(store);
      const m = render(store, "op-sort");
      expect(m).toContain(copy.conditionsSoon);
      expect(m).not.toContain('data-picker="columns"');
      expect(m).not.toContain("ei-row-toggle");
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

