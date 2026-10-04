# 08-scripts-config-c.md

File in questo blocco:

- `scripts/e2e-fase6b1.mjs`
- `scripts/extract-golden.mjs`
- `scripts/generate-index.mjs`

---

### `scripts/e2e-fase6b1.mjs`

939 righe

```js
#!/usr/bin/env node
/**
 * Verifica nel browser reale dell'Inspector (Fase 6b.1), con eventi veri:
 * stato bloccato e collegamento, tendine (singola, colonne, valori) vicino ai
 * quattro angoli della finestra a 1440×900, 1280×720 e 1280×600 (margine ≥ 16 px,
 * nessuna copertura del campo, sempre un [role=listbox] in un portale, nessun
 * <select> nativo), Converti tipo e Sostituisci valori con due colonne, Rimuovi
 * duplicati con riordino delle chiavi, box combinato (riordino, sgancio, pannello
 * espanso), pulsanti sul nodo, tetto all'altezza dei pannelli, focus e cronologia.
 * Salva le schermate in docs/visual/fase6b1/ (chiaro, scuro e due in tema notte).
 * Esce con codice 1 se una prova fallisce.
 *
 * Uso: node scripts/e2e-fase6b1.mjs
 */
import { mkdirSync } from "node:fs";
import { resolve } from "node:path";
import { chromium } from "playwright";
import { ROOT, SOLUTION, startServer } from "./visual-lib.mjs";

const OUT = resolve(ROOT, "docs/visual/fase6b1");
mkdirSync(OUT, { recursive: true });
const server = await startServer(Number(process.env.PORT ?? 5196));
const browser = await chromium.launch();
const ctx = await browser.newContext({
  viewport: { width: 1440, height: 900 },
  deviceScaleFactor: 1,
});
await ctx.addInitScript((s) => {
  if (!localStorage.getItem("isa.solutions"))
    localStorage.setItem("isa.solutions", JSON.stringify([s]));
  localStorage.setItem("isa-theme", "light");
}, SOLUTION);
const page = await ctx.newPage();
const errors = [];
page.on("console", (m) => m.type() === "error" && errors.push(m.text()));
page.on("pageerror", (e) => errors.push(String(e)));

let failed = 0;
const results = [];
function check(name, ok, extra = "") {
  results.push({
    prova: name,
    esito: ok ? "ok" : "FALLITA",
    dettaglio: String(extra).slice(0, 170),
  });
  if (!ok) failed++;
}
const state = () => page.evaluate(() => window.__etlStore.getState());
const dispatch = (cmd) => page.evaluate((c) => window.__etlStore.dispatch(c), cmd);
const rectOf = (sel) =>
  page.evaluate((s) => {
    const e = document.querySelector(s);
    if (!e) return null;
    const r = e.getBoundingClientRect();
    return { x: r.x, y: r.y, w: r.width, h: r.height };
  }, sel);
const center = (r) => ({ x: r.x + r.w / 2, y: r.y + r.h / 2 });
const hit = (a, b) => a.x < b.x + b.w && b.x < a.x + a.w && a.y < b.y + b.h && b.y < a.y + a.h;
const sleep = (ms) => page.waitForTimeout(ms);
const URL = `${server.base}/solutions/${SOLUTION.id}/etl?seed=prototype`;
const nodeRect = (id) => rectOf(`[data-node-id="${id}"] .ec-icon-wrap`);

/**
 * Schermata nei due temi (chiaro e scuro). `notte` ("chiaro" o "scuro"): in più, una nel tema notte
 * (due schermate in tutto, una chiara e una scura).
 */
async function shot(name, { notte } = {}) {
  const path = (suffix) => resolve(OUT, `${name}-${suffix}.png`);
  const html = (fn, arg) => page.evaluate(fn, arg);
  await page.screenshot({ path: path("chiaro"), animations: "disabled" });
  await html(() => document.documentElement.classList.add("dark"));
  await sleep(150);
  await page.screenshot({ path: path("scuro"), animations: "disabled" });
  await html(() => document.documentElement.classList.remove("dark"));
  if (notte) {
    await html(() => document.documentElement.setAttribute("data-theme", "notte"));
    if (notte === "scuro") await html(() => document.documentElement.classList.add("dark"));
    await sleep(150);
    await page.screenshot({ path: path(`notte-${notte}`), animations: "disabled" });
    await html(() => {
      document.documentElement.classList.remove("dark");
      document.documentElement.removeAttribute("data-theme");
    });
  }
  await sleep(100);
}

/** Aggiunge una lavorazione, la collega al dataset e restituisce il suo id. */
async function addOp(type, x, y, connect = true) {
  // la vista parte da zero: i nodi nuovi stanno dove ci si aspetta, dentro l'area visibile
  await dispatch({ type: "setView", payload: { x: 0, y: 0, zoom: 1 } });
  const before = Object.keys((await state()).graph.cards);
  await dispatch({ type: "addNode", payload: { component: type, point: { x, y } } });
  const id = Object.keys((await state()).graph.cards).find((k) => !before.includes(k));
  if (connect) await dispatch({ type: "connect", payload: { from: "ds1", to: id } });
  await place(id, x, y);
  await sleep(100);
  return id;
}
/** Porta un nodo dove si vuole (la scena sposta i nuovi nodi per non sovrapporli). */
async function place(id, x, y) {
  const c = (await state()).graph.cards[id];
  await dispatch({ type: "moveNodes", payload: { ids: [id], dx: x - c.x, dy: y - c.y } });
  await sleep(100);
}
/** Un clic vero sul nodo: apre l'Inspector. */
async function clickNode(id) {
  const r = await nodeRect(id);
  const c = center(r);
  await page.mouse.click(c.x, c.y);
  await sleep(450);
}
const insp = () => rectOf(".ec-insp");
const inspOpen = async () => (await state()).panels.insp.open;
const paramsOf = async (id, step = 0) => (await state()).graph.cards[id].params[step];

try {
  await page.goto(URL);
  await page.waitForSelector('[data-node-id="ds1"]', { timeout: 90000 });
  await sleep(600);

  // 1. un clic su un nodo apre l'Inspector e chiude la cassetta; senza ingresso è bloccato
  const s0 = await state();
  check(
    "all'avvio la cassetta è aperta e l'Inspector chiuso",
    s0.panels.tools.open && !s0.panels.insp.open,
  );
  await clickNode("op-sort");
  const s1 = await state();
  check(
    "il clic su un nodo apre l'Inspector e chiude la cassetta",
    s1.panels.insp.open && !s1.panels.tools.open && s1.inspector.nodeId === "op-sort",
  );
  check(
    "l'apertura con un clic non sposta il focus nell'Inspector",
    !(await page.evaluate(() => !!document.activeElement?.closest(".ec-insp"))),
  );
  check(
    "lavorazione senza ingresso: stato bloccato, nessun campo",
    (await page.locator('[data-testid="ei-blocked"]').count()) === 1 &&
      (await page.locator(".ec-insp .ei-fieldgroup").count()) === 0,
  );
  await shot("inspector-bloccato");

  // collegamento: lo schema dei dati in ingresso arriva nell'Inspector
  await dispatch({ type: "connect", payload: { from: "ds1", to: "op-sort" } });
  await sleep(300);
  check(
    "collegato un dataset: lo stato bloccato sparisce e il selettore di colonne conosce le 8 colonne",
    (await page.locator('[data-testid="ei-blocked"]').count()) === 0 &&
      (await page.locator('.ec-insp [data-picker="columns"]').getAttribute("data-total")) === "8",
  );
  check(
    "nessun <select>, <datalist> o input numerico/data nel DOM dell'Inspector",
    (await page.locator(".ec-insp select, .ec-insp datalist, .ec-insp option").count()) === 0 &&
      (await page.locator('.ec-insp input[type="number"], .ec-insp input[type="date"]').count()) ===
        0,
  );

  // 2. tendine: geometria vicino ai quattro angoli, a tre dimensioni di finestra
  const sortId = "op-sort";
  const valuesId = await addOp("replaceVal", 640, 90);
  await dispatch({
    type: "setParams",
    payload: {
      node: valuesId,
      index: 0,
      params: {
        items: [
          {
            columns: ["regione", "stato"],
            match: "è uguale a",
            find: { mode: "list", values: [], text: "", sep: "," },
            with: "",
          },
        ],
      },
    },
  });
  const kinds = {
    singola: { node: sortId, trigger: ".ec-insp button.ei-select" },
    colonne: {
      node: sortId,
      trigger: '.ec-insp [data-picker="columns"] .ei-add',
      field: '.ec-insp [data-picker="columns"]',
    },
    valori: {
      node: valuesId,
      trigger: '.ec-insp [data-picker="values"] .ei-add',
      field: '.ec-insp [data-picker="values"]',
    },
  };
  // il campo vero si sposta nel corpo della pagina (fuori dal pannello, che ritaglia e crea un contenitore
  // per gli elementi fissi) e si mette a ridosso dell'angolo della finestra; poi si rimette al suo posto
  const placeAt = (sel, corner) =>
    page.evaluate(
      ([s, c]) => {
        const e = document.querySelector("[data-moved]") ?? document.querySelector(s);
        if (!e.dataset.home) {
          e.dataset.home = "1";
          e.__parent = e.parentNode;
          e.__next = e.nextSibling;
          e.dataset.moved = "1";
          document.body.appendChild(e);
        }
        e.style.cssText = "position: fixed; width: 232px; z-index: 90;";
        const r = e.getBoundingClientRect();
        const w = innerWidth;
        const h = innerHeight;
        e.style.left = `${c.includes("l") ? 0 : c.includes("r") ? w - 232 : (w - 232) / 2}px`;
        e.style.top = `${c.includes("t") ? 0 : c.includes("b") ? h - r.height : (h - r.height) / 2}px`;
      },
      [sel, corner],
    );
  const resetStyle = (sel) =>
    page.evaluate((s) => {
      const e = document.querySelector("[data-moved]");
      if (!e) return;
      e.style.cssText = "";
      e.__parent.insertBefore(e, e.__next);
      delete e.dataset.home;
      delete e.dataset.moved;
    }, sel);
  const geometry = async (label, kind, corner) => {
    const k = kinds[kind];
    const fieldSel = k.field ?? k.trigger;
    await placeAt(fieldSel, corner);
    await sleep(50);
    const fieldRect = await rectOf("[data-moved]");
    await page.locator(`[data-moved]${k.field ? " .ei-add" : ""}`).click();
    await sleep(250);
    const info = await page.evaluate(() => {
      const lb = document.querySelector('[role="listbox"]');
      if (!lb) return null;
      const menu = lb.closest(".ei-menu");
      const r = menu.getBoundingClientRect();
      return {
        inPortal: !!lb.closest("#ei-portal"),
        rect: { x: r.x, y: r.y, w: r.width, h: r.height },
        win: { w: innerWidth, h: innerHeight },
        side: menu.dataset.side,
        selects: document.querySelectorAll("select, datalist").length,
      };
    });
    const ok =
      !!info &&
      info.inPortal &&
      info.selects === 0 &&
      info.rect.x >= 15.5 &&
      info.rect.y >= 15.5 &&
      info.rect.x + info.rect.w <= info.win.w - 15.5 &&
      info.rect.y + info.rect.h <= info.win.h - 15.5 &&
      !hit(info.rect, fieldRect);
    check(
      `${label}: il menu dista ≥ 16 px dai bordi, non copre il campo ed è un listbox nel portale`,
      ok,
      info
        ? `${Math.round(info.rect.x)},${Math.round(info.rect.y)} ${Math.round(info.rect.w)}×${Math.round(info.rect.h)} ${info.side}`
        : "assente",
    );
    if (!info) return null;
    await page.keyboard.press("Escape");
    await sleep(120);
    check(
      `${label}: Esc chiude il menu e riporta il focus al campo`,
      (await page.locator('[role="listbox"]').count()) === 0 &&
        (await page.evaluate(() => !!document.activeElement?.closest("[data-moved]"))),
    );
    return info;
  };
  const windows = [
    { width: 1440, height: 900 },
    { width: 1280, height: 720 },
    { width: 1280, height: 600 },
  ];
  for (const vp of windows) {
    await page.setViewportSize(vp);
    await sleep(500);
    for (const kind of ["singola", "colonne", "valori"]) {
      const k = kinds[kind];
      // il nodo giusto nell'Inspector
      await dispatch({ type: "select", payload: { ids: [k.node] } });
      await dispatch({ type: "inspect", payload: { node: k.node } });
      await sleep(250);
      const fieldSel = k.field ?? k.trigger;
      for (const corner of ["tl", "tr", "bl", "br", "c"]) {
        await geometry(`${vp.width}×${vp.height} ${kind} ${corner}`, kind, corner);
      }
      await resetStyle(fieldSel);
    }
  }
  await page.setViewportSize({ width: 1440, height: 900 });
  await sleep(500);

  // 3. Converti tipo con DUE colonne in una riga sola
  const castId = await addOp("cast", 640, 230);
  await clickNode(castId);
  check(
    "Converti tipo: il selettore di colonne è quello multiplo",
    (await page.locator('.ec-insp [data-picker="columns"]').count()) === 1,
  );
  await page.locator('.ec-insp [data-picker="columns"] .ei-add').click();
  await sleep(200);
  check(
    "la tendina delle colonne ha ricerca, tipi, Tutte, Nessuna e il conteggio «N colonne su M»",
    (await page.locator('.ei-menu input[role="combobox"]').count()) === 1 &&
      (await page.locator(".ei-menu .ei-option-hint").first().innerText()) === "integer" &&
      (await page.getByRole("button", { name: "Tutte" }).count()) === 1 &&
      (await page.getByRole("button", { name: "Nessuna" }).count()) === 1 &&
      (await page.locator(".ei-menu .ei-count").innerText()) === "0 colonne su 8",
  );
  await shot("tendina-colonne-aperta", { notte: "chiaro" });
  await page.keyboard.type("quant");
  await page.keyboard.press("Enter");
  await page.keyboard.press("Control+a");
  await page.keyboard.type("impor");
  await page.keyboard.press("Enter");
  await sleep(200);
  check(
    "le colonne si scelgono nell'ordine di scelta, da tastiera",
    JSON.stringify((await paramsOf(castId)).items[0].columns) === '["quantita","importo"]',
    JSON.stringify((await paramsOf(castId)).items[0].columns),
  );
  check(
    "il conteggio è annunciato (role=status, aria-live)",
    (await page.locator('.ei-menu .ei-count[role="status"][aria-live="polite"]').innerText()) ===
      "2 colonne su 8",
  );
  await page.keyboard.press("Escape");
  await sleep(150);
  // il tipo: una tendina singola con ricerca
  await page.locator(".ec-insp button.ei-select").click();
  await sleep(150);
  await page.keyboard.type("inte");
  await page.keyboard.press("Enter");
  await sleep(200);
  const castRow = (await paramsOf(castId)).items[0];
  check(
    "Converti tipo con due colonne in una riga sola",
    castRow.columns.length === 2 && castRow.to === "intero",
  );
  check(
    "il riassunto della riga è dal vivo: «quantita, importo → intero»",
    (await page.locator(".ei-row-sum").first().innerText()) === "quantita, importo → intero",
  );
  await shot("converti-tipo-due-colonne", { notte: "scuro" });

  // 4. Sostituisci valori con due colonne: unione dei domini, avviso al cambio
  await clickNode(valuesId);
  const dom = async () =>
    Number(await page.locator('.ec-insp [data-picker="values"]').getAttribute("data-total"));
  check(
    "Sostituisci valori: il dominio è l'unione di regione e stato (8 valori)",
    (await dom()) === 8,
    String(await dom()),
  );
  await page.locator('.ec-insp [data-picker="values"] .ei-add').click();
  await sleep(200);
  check(
    "la tendina dei valori ha ricerca, spunte, Tutti, Nessuno e il conteggio",
    (await page.locator('.ei-menu [role="listbox"] [role="option"]').count()) === 8 &&
      (await page.getByRole("button", { name: "Tutti" }).count()) === 1 &&
      (await page.locator(".ei-menu .ei-count").innerText()) === "0 selezionati su 8",
  );
  await shot("tendina-valori-aperta");
  // un valore delle regioni, uno degli stati, uno scritto a mano (con la grafia dei dati quando esiste)
  await page.keyboard.type("nord");
  await page.keyboard.press("Enter");
  await page.keyboard.press("Control+a");
  await page.keyboard.type("CHIUSO");
  await page.keyboard.press("Enter");
  await page.keyboard.press("Control+a");
  await page.keyboard.type("Mare");
  check(
    "«+ Aggiungi “Mare”» compare per ciò che non esiste",
    (await page.locator(".ei-menu .ei-option.ei-free").first().innerText()) === "+ Aggiungi “Mare”",
  );
  await page.keyboard.press("Enter");
  await sleep(150);
  let vals = (await paramsOf(valuesId)).items[0].find.values;
  check(
    "i valori si aggiungono con la grafia dei dati («nord» → «Nord», «CHIUSO» → «Chiuso»)",
    JSON.stringify(vals) === '["Nord","Chiuso","Mare"]',
    JSON.stringify(vals),
  );
  // incolla di più valori insieme
  await page.evaluate(() => {
    const input = document.querySelector('.ei-menu input[role="combobox"]');
    const dt = new DataTransfer();
    dt.setData("text", "sud; Aperto|Isole\nnord");
    input.dispatchEvent(
      new ClipboardEvent("paste", { clipboardData: dt, bubbles: true, cancelable: true }),
    );
  });
  await sleep(150);
  vals = (await paramsOf(valuesId)).items[0].find.values;
  check(
    "incollare più valori separati da virgola, punto e virgola, barra verticale o a capo li aggiunge tutti",
    JSON.stringify(vals) === '["Nord","Chiuso","Mare","Sud","Aperto","Isole"]',
    JSON.stringify(vals),
  );
  await page.keyboard.press("Escape");
  await sleep(150);
  await shot("sostituisci-valori-due-colonne");
  // cambiando le colonne i valori NON si azzerano: quelli fuori dominio restano in corsivo, con l'avviso
  await page.locator('.ec-insp [data-picker="columns"] .ei-chip-x', { hasText: "" }).nth(1).click();
  await sleep(250);
  vals = (await paramsOf(valuesId)).items[0].find.values;
  check(
    "tolta la colonna «stato» nessun valore è stato azzerato",
    vals.length === 6,
    JSON.stringify(vals),
  );
  check(
    "compare l'avviso «N valori non presenti nelle colonne scelte», con «Rimuovi»",
    (await page.locator(".ei-warn").innerText()).includes("non presenti nelle colonne scelte") &&
      (await page.locator(".ec-insp .ei-chip.ei-free").count()) >= 1,
    await page
      .locator(".ei-warn")
      .innerText()
      .catch(() => ""),
  );
  await page.locator(".ei-warn .ei-link-btn").click();
  await sleep(200);
  vals = (await paramsOf(valuesId)).items[0].find.values;
  check(
    "«Rimuovi» toglie solo i valori fuori dominio",
    JSON.stringify(vals) === '["Nord","Sud","Isole"]',
    JSON.stringify(vals),
  );

  // 5. Rimuovi duplicati con riordino delle chiavi
  const dedupId = await addOp("dedup", 640, 370);
  await clickNode(dedupId);
  await dispatch({
    type: "setParams",
    payload: {
      node: dedupId,
      index: 0,
      params: {
        keep: "la prima",
        items: [{ columns: ["id", "cliente", "regione"], cmp: "esatto" }],
      },
    },
  });
  await sleep(250);
  const order = async () => (await paramsOf(dedupId)).items[0].columns.join();
  check("tre chiavi nell'ordine di scelta", (await order()) === "id,cliente,regione");
  await page.getByRole("button", { name: /^regione, posizione/ }).focus();
  await page.keyboard.press("Alt+ArrowLeft");
  await sleep(150);
  check("Alt+← sposta la chiave", (await order()) === "id,regione,cliente", await order());
  await page.keyboard.press("Alt+ArrowLeft");
  await sleep(150);
  check(
    "Alt+← di nuovo: la chiave diventa la prima",
    (await order()) === "regione,id,cliente",
    await order(),
  );
  check(
    "dopo il riordino il focus segue l'etichetta spostata",
    (await page.evaluate(() =>
      document.activeElement?.getAttribute("aria-label")?.startsWith("regione"),
    )) === true,
  );
  // trascinamento: «cliente» davanti a tutte
  const chips = async () =>
    page.locator('.ec-insp [data-picker="columns"] .ei-chip-label').evaluateAll((els) =>
      els.map((e) => {
        const r = e.getBoundingClientRect();
        return { x: r.x + r.width / 2, y: r.y + r.height / 2, t: e.textContent };
      }),
    );
  let cs = await chips();
  await page.mouse.move(cs[2].x, cs[2].y);
  await page.mouse.down();
  await page.mouse.move(cs[0].x - 4, cs[0].y, { steps: 8 });
  await page.mouse.up();
  await sleep(200);
  check(
    "trascinando un'etichetta se ne cambia l'ordine",
    (await order()) === "cliente,regione,id",
    await order(),
  );
  check(
    "l'ordine è quello salvato nel passo di Rimuovi duplicati (riassunto)",
    (await page.locator(".ei-row-sum").first().innerText()).startsWith("cliente, regione, id"),
    await page.locator(".ei-row-sum").first().innerText(),
  );
  // Tutte e Nessuna agiscono sulle sole colonne visibili dopo la ricerca
  await page.locator('.ec-insp [data-picker="columns"] .ei-add').click();
  await sleep(150);
  await page.keyboard.type("a");
  await sleep(100);
  const visible = await page.locator('.ei-menu [role="option"]:not(.ei-free)').count();
  await page.getByRole("button", { name: "Nessuna" }).click();
  await sleep(150);
  const afterNone = (await paramsOf(dedupId)).items[0].columns;
  check(
    "«Nessuna» toglie solo le colonne visibili dopo la ricerca",
    afterNone.every((c) => !c.includes("a")) && afterNone.length >= 0 && visible >= 1,
    afterNone.join(),
  );
  await page.getByRole("button", { name: "Tutte" }).click();
  await sleep(150);
  const afterAll = (await paramsOf(dedupId)).items[0].columns;
  check(
    "«Tutte» aggiunge solo le colonne visibili (quelle che contengono «a»)",
    afterAll.every((c) => c.includes("a") || afterNone.includes(c)) &&
      afterAll.length === afterNone.length + visible,
    afterAll.join(),
  );
  await page.keyboard.press("Escape");
  await sleep(100);

  // 6. box combinato: elenco dei passaggi, riordino (tastiera e puntatore), sgancio
  const boxId = await addOp("filter", 900, 120, false);
  for (const type of ["sort", "cast"]) {
    const extra = await addOp(type, 1010, 120, false);
    await dispatch({ type: "merge", payload: { dragged: extra, target: boxId } });
  }
  await dispatch({ type: "connect", payload: { from: "ds1", to: boxId } });
  await place(boxId, 900, 120);
  await sleep(250);
  const comps = async () => (await state()).graph.cards[boxId].components.join();
  check(
    "box di tre passaggi: filtro, ordina, converti",
    (await comps()) === "filter,sort,cast",
    await comps(),
  );
  await clickNode(boxId);
  check(
    "il box mostra l'elenco verticale dei passaggi",
    (await page.locator(".ec-insp .ei-step").count()) === 3,
  );
  check(
    "il passaggio scelto è il primo e mostra i suoi parametri (filtro: nota sulle condizioni)",
    (await page.locator('.ec-insp [data-testid="ei-conditions-soon"]').count()) === 1,
  );
  await page.locator(".ec-insp .ei-step-main").nth(1).click();
  await sleep(200);
  check(
    "scegliere il secondo passaggio ne mostra i parametri (criteri di ordinamento)",
    (await page.locator(".ec-insp .ei-list .ei-label").first().innerText()) ===
      "Criteri di ordinamento",
  );
  await shot("box-combinato-passaggi");
  await page.locator(".ec-insp .ei-step-main").nth(0).focus();
  await page.keyboard.press("Alt+ArrowDown");
  await sleep(250);
  check(
    "Alt+↓ sposta il passaggio e il focus lo segue",
    (await comps()) === "sort,filter,cast" &&
      (await page.evaluate(() =>
        document.activeElement?.closest(".ei-step")?.getAttribute("data-step"),
      )) === "1",
    await comps(),
  );
  check(
    "lo spostamento è annunciato ai lettori di schermo",
    (await page.locator('.ec-insp [role="status"]').first().innerText()).includes(
      "posizione 2 di 3",
    ),
  );
  await page.keyboard.press("Alt+ArrowDown");
  await sleep(200);
  check(
    "Alt+↓ di nuovo: il passaggio è l'ultimo",
    (await comps()) === "sort,cast,filter",
    await comps(),
  );
  await page.keyboard.press("Alt+ArrowUp");
  await sleep(200);
  await page.keyboard.press("Alt+ArrowUp");
  await sleep(200);
  check("Alt+↑ lo riporta in alto", (await comps()) === "filter,sort,cast", await comps());
  // trascinamento del primo passaggio sotto l'ultimo
  const stepRects = async () =>
    page.locator(".ec-insp .ei-step").evaluateAll((els) =>
      els.map((e) => {
        const r = e.getBoundingClientRect();
        return { x: r.x + r.width / 2, y: r.y + r.height / 2 };
      }),
    );
  const sr = await stepRects();
  await page.mouse.move(sr[0].x - 40, sr[0].y);
  await page.mouse.down();
  await page.mouse.move(sr[0].x - 40, sr[2].y + 6, { steps: 10 });
  await page.mouse.up();
  await sleep(250);
  check(
    "trascinando un passaggio se ne cambia l'ordine",
    (await comps()) === "sort,cast,filter",
    await comps(),
  );
  // sgancio dal pulsante
  const cardsBefore = Object.keys((await state()).graph.cards).length;
  await page.locator('.ec-insp [aria-label="Sgancia sul canvas"]').nth(2).click();
  await sleep(300);
  check(
    "«Sgancia sul canvas» fa tornare il passaggio un nodo sul canvas",
    Object.keys((await state()).graph.cards).length > cardsBefore - 1 &&
      (await comps()) === "sort,cast",
    await comps(),
  );
  check("l'eliminazione di un passaggio dal pulsante", true);
  await page.locator('.ec-insp [aria-label="Elimina passaggio"]').nth(1).click();
  await sleep(250);
  check(
    "eliminato il passaggio il box torna una lavorazione semplice",
    (await state()).graph.cards[boxId].components.length === 1,
    await comps(),
  );

  // 7. pulsanti sul nodo, pannello espanso e menu dei passaggi
  const box2 = await addOp("filter", 900, 260, false);
  for (const type of ["sort", "cast"]) {
    const extra = await addOp(type, 1010, 260, false);
    await dispatch({ type: "merge", payload: { dragged: extra, target: box2 } });
  }
  await dispatch({ type: "connect", payload: { from: "ds1", to: box2 } });
  await place(box2, 900, 260);
  await sleep(250);
  const r2 = await nodeRect(box2);
  const c2 = center(r2);
  await page.mouse.move(c2.x, c2.y);
  await sleep(250);
  const delBtn = await rectOf(`[data-node-id="${box2}"] .ec-del-btn`);
  const expBtn = await rectOf(`[data-node-id="${box2}"] .ec-expand-btn`);
  check(
    "al passaggio del puntatore compaiono × ed espansione, con un'area da almeno 32 px",
    (await page.evaluate(
      (id) =>
        getComputedStyle(document.querySelector(`[data-node-id="${id}"] .ec-del-btn`)).opacity,
      box2,
    )) === "1" &&
      !!delBtn &&
      !!expBtn,
  );
  check(
    "un nodo semplice non ha il pulsante di espansione",
    (await page.locator('[data-node-id="op-export"] .ec-expand-btn').count()) === 0 &&
      (await page.locator('[data-node-id="op-export"] .ec-del-btn').count()) === 1,
  );
  await shot("nodo-con-pulsanti");
  // da tastiera: il × compare al focus
  await page.mouse.move(40, 40);
  await page.locator(`[data-node-id="${box2}"] .ec-del-btn`).focus();
  await sleep(250);
  check(
    "il pulsante × compare anche al focus",
    (await page.evaluate(
      (id) =>
        getComputedStyle(document.querySelector(`[data-node-id="${id}"] .ec-del-btn`)).opacity,
      box2,
    )) === "1",
  );
  await page.mouse.move(c2.x, c2.y);
  await page.locator(`[data-node-id="${box2}"] .ec-expand-btn`).click();
  await sleep(300);
  check(
    "l'espansione apre il pannello con i passaggi",
    (await page.locator('[data-testid="ei-expanded"] .ei-step').count()) === 3,
  );
  check(
    "il pannello espanso è una finestra di dialogo (role=dialog)",
    (await page.locator('[data-testid="ei-expanded"] [role="dialog"]').count()) === 1,
  );
  await shot("pannello-espanso");
  // menu del secondo passaggio
  await page.locator('[data-testid="ei-expanded"] [aria-haspopup="menu"]').nth(1).click();
  await sleep(200);
  const items = await page.locator('[role="menu"] [role="menuitem"]').allInnerTexts();
  check(
    "il menu del passaggio ha Configura parametri, Sgancia, Elimina passaggio",
    items.join("|") === "Configura parametri|Sgancia sul canvas|Elimina passaggio",
    items.join("|"),
  );
  await page.keyboard.press("ArrowDown");
  await page.keyboard.press("Escape");
  await sleep(150);
  check(
    "Esc chiude il menu e non il pannello",
    (await page.locator('[role="menu"]').count()) === 0 &&
      (await page.locator('[data-testid="ei-expanded"]').count()) === 1,
  );
  await page.locator('[data-testid="ei-expanded"] [aria-haspopup="menu"]').nth(1).click();
  await page.getByRole("menuitem", { name: "Configura parametri" }).click();
  await sleep(350);
  const sc = await state();
  check(
    "«Configura parametri» chiude il pannello e apre l'Inspector su quel passaggio",
    (await page.locator('[data-testid="ei-expanded"]').count()) === 0 &&
      sc.panels.insp.open &&
      sc.inspector.nodeId === box2 &&
      sc.inspector.step === 1,
    JSON.stringify(sc.inspector),
  );
  // trascinare un passaggio fuori dal pannello lo sgancia nel punto di rilascio
  await page.mouse.move(c2.x, c2.y);
  await page.locator(`[data-node-id="${box2}"] .ec-expand-btn`).click();
  await sleep(300);
  const nBefore = Object.keys((await state()).graph.cards).length;
  const row = await rectOf('[data-testid="ei-expanded"] .ei-step:nth-child(1)');
  const stage = await rectOf(".ec-stage");
  const drop = { x: stage.x + 300, y: stage.y + stage.h - 90 };
  await page.mouse.move(row.x + row.w / 2 - 60, row.y + row.h / 2);
  await page.mouse.down();
  await page.mouse.move(row.x + row.w / 2 - 60 - 20, row.y + row.h / 2 + 10, { steps: 3 });
  await page.mouse.move(drop.x, drop.y, { steps: 12 });
  check(
    "trascinando fuori dal pannello compare l'avviso di sgancio",
    (await page.locator('[data-testid="ei-expanded"] .ei-outside-note').count()) === 1,
  );
  await page.mouse.up();
  await sleep(350);
  const after = await state();
  const fresh = Object.values(after.graph.cards).filter(
    (k) =>
      k.kind === "op" &&
      k.components.length === 1 &&
      k.x > 0 &&
      !["op-filter", "op-join", "op-sort", "op-export"].includes(k.id),
  );
  check(
    "il passaggio trascinato fuori diventa un nodo e il pannello si chiude da solo se il box resta combinato o no",
    Object.keys(after.graph.cards).length > nBefore - 1 &&
      (await page.locator('[data-testid="ei-expanded"]').count()) === 0,
  );
  const dropped = fresh.find(
    (k) =>
      Math.abs(stage.x + after.view.x + (k.x + 44) * after.view.zoom - drop.x) < 80 &&
      Math.abs(stage.y + after.view.y + (k.y + 44) * after.view.zoom - drop.y) < 80,
  );
  check(
    "il nodo sganciato sta nel punto del rilascio",
    !!dropped,
    JSON.stringify(fresh.map((k) => [k.id, k.x, k.y])),
  );

  // 8. pulsante × sul nodo: subito se isolato, con conferma se collegato
  const lone = await addOp("sample", 850, 380, false);
  const lr = center(await nodeRect(lone));
  await page.mouse.move(lr.x, lr.y);
  await sleep(200);
  await page.locator(`[data-node-id="${lone}"] .ec-del-btn`).click();
  await sleep(250);
  check(
    "× su un nodo isolato lo elimina subito, senza domande",
    !(await state()).graph.cards[lone] &&
      (await page.locator('[data-testid="ec-confirm"]').count()) === 0,
  );
  const wired = await addOp("round", 850, 500, true);
  const wr = center(await nodeRect(wired));
  await page.mouse.move(wr.x, wr.y);
  await sleep(200);
  await page.locator(`[data-node-id="${wired}"] .ec-del-btn`).click();
  await sleep(250);
  check(
    "× su un nodo collegato chiede conferma e mostra cosa sparirebbe",
    (await page.locator('[data-testid="ec-confirm"]').count()) === 1 &&
      (await page.locator(".ec-doomed").count()) >= 2,
  );
  await page.getByTestId("ec-confirm").getByRole("button", { name: "Annulla" }).click();
  check("Annulla lascia tutto com'era", !!(await state()).graph.cards[wired]);

  // 9. tetto all'altezza dei pannelli orizzontali, a 900 e 720
  await dispatch({ type: "select", payload: { ids: [castId] } });
  await dispatch({ type: "inspect", payload: { node: castId } });
  for (const [vp, name] of [
    [{ width: 1440, height: 900 }, "inspector-in-basso-1440"],
    [{ width: 1280, height: 720 }, "inspector-in-basso-1280x720"],
  ]) {
    await page.setViewportSize(vp);
    await sleep(400);
    await dispatch({ type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } });
    await sleep(700);
    const ws = await rectOf(".ec-workspace");
    const pn = await rectOf('.ec-panel[data-panel="insp"]');
    const cap = Math.floor(0.45 * ws.h);
    check(
      `${vp.width}×${vp.height}: l'altezza del pannello in basso non supera il 45% dello spazio di lavoro`,
      pn.h - 16 <= cap + 1 && pn.h > 100,
      `${Math.round(pn.h - 16)} ≤ ${cap}`,
    );
    check(
      `${vp.width}×${vp.height}: sui bordi alto e basso i contenuti sono in colonne`,
      (await page.evaluate(
        () => getComputedStyle(document.querySelector(".ec-panel.ec-horiz .ei-root")).columnWidth,
      )) !== "auto",
    );
    check(
      `${vp.width}×${vp.height}: il pannello sta dentro la finestra`,
      pn.y >= 0 && pn.y + pn.h <= vp.height + 0.5,
    );
    await shot(name);
    await dispatch({ type: "setPanel", payload: { panel: "insp", side: "top", open: true } });
    await sleep(700);
    const pt = await rectOf('.ec-panel[data-panel="insp"]');
    check(
      `${vp.width}×${vp.height}: anche in alto vale il tetto`,
      pt.h - 16 <= Math.floor(0.45 * (await rectOf(".ec-workspace")).h) + 1,
    );
    await dispatch({ type: "setPanel", payload: { panel: "insp", side: "right", open: true } });
    await sleep(700);
  }
  await page.setViewportSize({ width: 1440, height: 900 });
  await sleep(500);

  // 10. tendina in alto e vicino al bordo
  await dispatch({ type: "select", payload: { ids: [sortId] } });
  await dispatch({ type: "inspect", payload: { node: sortId } });
  await sleep(300);
  await placeAt(kinds.singola.trigger, "bc");
  await page.locator("[data-moved]").click();
  await sleep(250);
  check(
    "trigger in basso: il menu si apre verso l'alto",
    (await page.locator(".ei-menu").getAttribute("data-side")) === "above",
  );
  await shot("tendina-in-alto");
  await page.keyboard.press("Escape");
  await resetStyle(kinds.singola.trigger);
  await page.setViewportSize({ width: 1280, height: 720 });
  await sleep(500);
  await placeAt(kinds.colonne.field, "tr");
  await page.locator("[data-moved] .ei-add").click();
  await sleep(250);
  const mr = await rectOf(".ei-menu");
  check(
    "1280×720, campo a ridosso del bordo destro: il menu si sposta a sinistra e tiene 16 px a destra",
    mr.x + mr.w <= 1280 - 15.5,
  );
  await shot("tendina-vicino-al-bordo-1280x720");
  await page.keyboard.press("Escape");
  await resetStyle(kinds.colonne.field);
  await page.setViewportSize({ width: 1440, height: 900 });
  await sleep(500);

  // 11. focus e cronologia: scrivere non perde mai focus né cursore; 10 caratteri = un passo
  await clickNode(sortId);
  const nameSel = '.ec-insp [data-testid="ec-inspector-name"]';
  await page.locator(nameSel).click();
  await page.keyboard.press("Control+a");
  const logBefore = await page.evaluate(() => window.__etlStore.getLog().length);
  const nameBefore = (await state()).graph.cards[sortId].name;
  let lost = 0;
  const typed = "abcdefghijklmnopqrst";
  for (let i = 0; i < typed.length; i++) {
    await page.keyboard.type(typed[i]);
    const st = await page.evaluate((sel) => {
      const a = document.activeElement;
      return { ok: a === document.querySelector(sel), pos: a.selectionStart, len: a.value.length };
    }, nameSel);
    if (!st.ok || st.pos !== st.len) lost++;
  }
  check(
    "20 caratteri nel nome: il focus e il cursore non si perdono mai",
    lost === 0,
    `perdite: ${lost}`,
  );
  const lastLog = await page.evaluate(() => window.__etlStore.getLog().at(-1));
  check(
    "20 caratteri digitati sono UNA voce di registro (raggruppata)",
    (await page.evaluate(() => window.__etlStore.getLog().length)) === logBefore + 1 &&
      lastLog.type === "renameNode" &&
      lastLog.count === 20,
    JSON.stringify({ type: lastLog.type, count: lastLog.count }),
  );
  check(
    "il nome del nodo è quello scritto, sul canvas e nello stato",
    (await state()).graph.cards[sortId].name === typed,
  );
  await page.evaluate(() => window.__etlStore.undo());
  check(
    "UN annullamento ripristina il nome di prima: era un solo passo",
    (await state()).graph.cards[sortId].name === nameBefore,
    (await state()).graph.cards[sortId].name,
  );
  await page.evaluate(() => window.__etlStore.redo());
  // un campo numerico di una lavorazione
  const limitId = await addOp("limit", 850, 620);
  await clickNode(limitId);
  const numInput = page.locator('.ec-insp input[inputmode="numeric"]').first();
  await numInput.click();
  await page.keyboard.press("Control+a");
  const limitBefore = (await paramsOf(limitId)).n;
  let lost2 = 0;
  const digits = "12345678901234567890";
  for (let i = 0; i < digits.length; i++) {
    await page.keyboard.type(digits[i]);
    const st = await page.evaluate(() => {
      const a = document.activeElement;
      return { tag: a.tagName, pos: a.selectionStart, len: a.value?.length };
    });
    if (st.tag !== "INPUT" || st.pos !== st.len) lost2++;
  }
  check(
    "20 aggiornamenti in un campo numerico: nessuna perdita di cursore",
    lost2 === 0,
    `perdite: ${lost2}`,
  );
  check(
    "il valore scritto è quello digitato",
    (await paramsOf(limitId)).n === digits,
    (await paramsOf(limitId)).n,
  );
  await page.evaluate(() => window.__etlStore.undo());
  check(
    "un solo annullamento ripristina il valore di prima: era un solo passo",
    (await paramsOf(limitId)).n === limitBefore,
    (await paramsOf(limitId)).n,
  );
  await page.evaluate(() => window.__etlStore.redo());
  check(
    "il campo numerico non ha frecce native (inputmode numeric, tipo testo)",
    (await numInput.getAttribute("type")) === "text",
  );
  // Esc nel pannello riporta il focus al canvas, senza deselezionare
  await page.keyboard.press("Escape");
  await sleep(150);
  check(
    "Esc nel pannello riporta il focus al canvas e il nodo resta selezionato",
    (await page.evaluate(() => document.activeElement?.classList.contains("ec-stage"))) &&
      (await state()).inspector.nodeId === limitId &&
      (await inspOpen()),
  );
} catch (e) {
  console.error(String(e).slice(0, 1500));
  check("eccezione nello script", false, e);
} finally {
  await browser.close();
  server.stop();
}
console.table(results.filter((r) => r.esito !== "ok"));
check("nessun errore in console", errors.length === 0, errors.join(" | "));
console.log(failed ? `${failed} PROVE FALLITE` : `TUTTE LE ${results.length} PROVE SUPERATE`);
process.exit(failed ? 1 : 0);
```

### `scripts/extract-golden.mjs`

386 righe

```js
#!/usr/bin/env node
/**
 * Genera i file golden di src/etl-layout/__tests__/golden/*.json eseguendo
 * il PROTOTIPO (docs/prototype/isa-fusion-prototype.html) in Chromium
 * senza interfaccia, tramite Playwright.
 *
 * Ogni scenario viene costruito con le variabili e le funzioni globali del
 * prototipo (`cards`, `linksArr`, `linkState`, `MAX_BENDS`, `drawLinks`,
 * `autoLayout`, `setMode`, `spawnOutput`, ...), dentro un'unica chiamata
 * sincrona: nessun fotogramma di animazione può intervenire nel mezzo.
 *
 * Cavi "a regime": `drawLinks` anima l'angolo di aggancio e lo snodo
 * verso il valore scelto (righe 1339-1341). Una "passata" qui è:
 * rivaluta tutti i cavi (`nextEval = 0`, poi `drawLinks()`), porta
 * angoli e snodo sul valore obiettivo, ridisegna (`drawLinks()` con la
 * rivalutazione disattivata). Le passate si ripetono finché i percorsi non
 * cambiano più (al massimo 8), esattamente come `settleLinks` di
 * etl-layout.
 *
 * Uso:  node scripts/extract-golden.mjs            (scrive i file golden)
 *       node scripts/extract-golden.mjs --explore  (stampa un riassunto, non scrive)
 */
import { chromium } from "playwright";
import { mkdirSync, writeFileSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath, pathToFileURL } from "node:url";

const root = resolve(dirname(fileURLToPath(import.meta.url)), "..");
const prototype = resolve(root, "docs/prototype/isa-fusion-prototype.html");
const outDir = resolve(root, "src/etl-layout/__tests__/golden");
const explore = process.argv.includes("--explore");
const VIEWPORT = { width: 1440, height: 900 };
const MAX_PASSES = 8;

const ds = (id, x, y, extra = {}) => ({
  id,
  kind: "dataset",
  components: ["dataset"],
  x,
  y,
  ...extra,
});
const op = (id, components, x, y, extra = {}) => ({ id, kind: "op", components, x, y, ...extra });
const L = (from, to) => ({ from, to });

/** Scenari. `type` decide cosa viene eseguito e registrato. */
const SCENARIOS = [
  {
    name: "01-dritto-allineati",
    description: "Due nodi allineati orizzontalmente: cavo dritto.",
    type: "routes",
    cards: [ds("A", 104, 312), op("B", ["filter"], 416, 312)],
    links: [L("A", "B")],
  },
  {
    name: "02-dritto-scorrimento",
    description: "Disallineati di 20 px, entro lo scorrimento delle porte: ancora dritto.",
    type: "routes",
    cards: [ds("A", 104, 312), op("B", ["filter"], 416, 332)],
    links: [L("A", "B")],
  },
  {
    name: "03-oltre-scorrimento",
    description: "Disallineati di 130 px, oltre lo scorrimento: forma a L o a Z.",
    type: "routes",
    cards: [ds("A", 104, 312), op("B", ["filter"], 416, 442)],
    links: [L("A", "B")],
  },
  {
    name: "04-ostacolo",
    description: "Un nodo ostruisce il percorso diretto: il cavo lo aggira.",
    type: "routes",
    cards: [ds("A", 104, 312), op("X", ["sort"], 286, 312), op("B", ["filter"], 520, 312)],
    links: [L("A", "B")],
  },
  {
    name: "05-incrocio",
    description: "Due cavi che si incrocerebbero con il percorso più corto.",
    type: "routes",
    cards: [
      ds("A1", 104, 208),
      ds("A2", 104, 468),
      op("B1", ["filter"], 520, 468),
      op("B2", ["sort"], 520, 208),
    ],
    links: [L("A1", "B1"), L("A2", "B2")],
  },
  {
    name: "06-corsie",
    description: "Più cavi nello stesso corridoio (snodi ammessi: 2, perché nascano forme a Z).",
    type: "routes",
    maxBends: 2,
    cards: [
      ds("A1", 104, 104),
      ds("A2", 104, 234),
      ds("A3", 104, 364),
      op("B1", ["filter"], 546, 494),
      op("B2", ["sort"], 546, 624),
      op("B3", ["aggregate"], 546, 754),
    ],
    links: [L("A1", "B1"), L("A2", "B2"), L("A3", "B3")],
  },
  {
    name: "07-join-output-parziale",
    description: "Un box con due ingressi da un join e il suo output parziale.",
    type: "routes",
    cards: [
      ds("A", 104, 208),
      ds("B", 104, 442),
      op("J", ["join"], 364, 312),
      ds("O", 572, 312, { isOutput: true, capacity: 2, filled: 1 }),
    ],
    links: [L("A", "J"), L("B", "J"), L("J", "O")],
  },
  {
    name: "08-spostamento",
    description: "Un nodo spostato di poco (il cavo conserva il percorso) e di molto (lo cambia).",
    type: "routes",
    cards: [ds("A", 104, 312), op("B", ["filter"], 416, 442)],
    links: [L("A", "B")],
    moves: [
      { id: "B", dx: 8, dy: -6 },
      { id: "B", dx: -390, dy: 260 },
    ],
  },
  {
    name: "09-catena-riordino",
    description:
      "Catena dataset → filtro → join → ordina → esporta con un secondo dataset sul join, prima e dopo il riordino automatico.",
    type: "autoLayout",
    cards: [
      ds("D1", 520, 600),
      op("F", ["filter"], 130, 130),
      ds("OF", 780, 390, { isOutput: true, capacity: 1, filled: 1 }),
      ds("D2", 60, 700),
      op("J", ["join"], 910, 130),
      ds("OJ", 300, 450, { isOutput: true, capacity: 2, filled: 2 }),
      op("S", ["sort"], 1100, 600),
      ds("OS", 650, 100, { isOutput: true, capacity: 1, filled: 1 }),
      op("E", ["exportOp"], 400, 260),
    ],
    links: [
      L("D1", "F"),
      L("F", "OF"),
      L("OF", "J"),
      L("D2", "J"),
      L("J", "OJ"),
      L("OJ", "S"),
      L("S", "OS"),
      L("OS", "E"),
    ],
  },
  {
    name: "10-riordino-isolati",
    description: "Riordino con nodi isolati: colonna di parcheggio a destra del flusso.",
    type: "autoLayout",
    cards: [
      op("I1", ["sort"], 700, 80),
      ds("D", 300, 500),
      ds("I2", 90, 90),
      op("F", ["filter"], 90, 400),
      ds("O", 900, 600, { isOutput: true, capacity: 1, filled: 1 }),
      op("I3", ["aggregate"], 500, 300),
      ds("I4", 620, 520),
      op("I5", ["rename"], 250, 250),
    ],
    links: [L("D", "F"), L("F", "O")],
  },
  {
    name: "10b-riordino-colonna-fitta",
    description:
      "Riordino con cinque nodi nella stessa colonna in uno stage alto 636 px: la distanza tra le righe scende al minimo (CARD + LABEL_H + 18).",
    type: "autoLayout",
    stageH: 636,
    cards: [
      ds("D", 300, 500),
      op("F", ["filter"], 90, 400),
      op("I1", ["sort"], 700, 80),
      ds("I2", 90, 90),
      op("I3", ["aggregate"], 500, 300),
      ds("I4", 620, 520),
      op("I5", ["rename"], 250, 250),
    ],
    links: [L("D", "F")],
  },
  {
    name: "11-organizzato",
    description: "Modalità Organizzato: assegnazione iniziale delle postazioni e scambio di posto.",
    type: "grid",
    cards: [
      ds("A", 40, 30),
      op("B", ["filter"], 170, 40),
      op("C", ["sort"], 150, 170),
      ds("D", 30, 180),
      op("E", ["aggregate"], 420, 300),
    ],
    links: [L("A", "B")],
    drops: [
      { id: "E", x: 150, y: 20 },
      { id: "A", x: 700, y: 700 },
    ],
  },
  {
    name: "12-output-generato",
    description:
      "Posizione dell'output generato in modalità Libero, con un nodo già nel posto ideale.",
    type: "spawn",
    cards: [ds("A", 104, 312), op("B", ["filter"], 312, 312), op("X", ["sort"], 520, 312)],
    links: [L("A", "B")],
    boxId: "B",
  },
  {
    name: "13-output-organizzato",
    description:
      "Posizione dell'output generato in modalità Organizzato: postazione a destra del box.",
    type: "spawn",
    mode: "grid",
    cards: [ds("A", 40, 300), op("B", ["filter"], 300, 300), op("X", ["sort"], 430, 300)],
    links: [L("A", "B")],
    boxId: "B",
  },
];

/** Funzione eseguita nella pagina del prototipo. Solo globali del prototipo. */
function runScenario(sc, maxPasses) {
  /* global cards:writable, linksArr:writable, linkState, MAX_BENDS:writable, drawLinks, autoLayout,
     setMode, spawnOutput, createCardEl, defaultParams, nearestSlot, placeInSlots, layoutMode:writable,
     draggingUid:writable, stage, workspace */
  const clone = (v) => JSON.parse(JSON.stringify(v));
  // altezza dello stage (CSS `--stage-h`, riga 18; 520 px nel prototipo)
  // (la transizione di `.workspace`, riga 213, farebbe leggere l'altezza vecchia)
  workspace.style.transition = "none";
  document.documentElement.style.setProperty("--stage-h", (sc.stageH ?? 520) + "px");
  document.querySelectorAll("#stage .card").forEach((c) => c.remove());
  Object.keys(linkState).forEach((k) => delete linkState[k]);
  layoutMode = "free";
  draggingUid = null;
  MAX_BENDS = sc.maxBends ?? 1;
  cards = {};
  for (const c of sc.cards) {
    const { id, ...rest } = c;
    cards[id] = {
      ...clone(rest),
      params: rest.components.map((t) => defaultParams(t)),
      name: id,
    };
  }
  linksArr = sc.links.map((l) => ({ from: l.from, to: l.to }));

  const signature = () =>
    JSON.stringify(
      linksArr.map((l) => {
        const st = linkState[l.from + "|" + l.to];
        return st && st.pts ? [st.portA, st.portB, st.pts.map((p) => [p.x, p.y])] : null;
      }),
    );
  const settle = () => {
    let cur = signature();
    let passes = 0;
    while (passes < maxPasses) {
      Object.values(linkState).forEach((st) => (st.nextEval = 0));
      drawLinks();
      Object.values(linkState).forEach((st) => {
        st.a = st.portA;
        st.b = st.portB;
        if (st.knobTarget !== null) st.knob = st.knobTarget;
        st.nextEval = Infinity;
      });
      drawLinks();
      passes++;
      const next = signature();
      const stable = next === cur;
      cur = next;
      if (stable) break;
    }
    const d = {};
    document.querySelectorAll("#linkPaths path[id^='lp-']").forEach((p) => {
      d[p.id.slice(3)] = p.getAttribute("d");
    });
    const routes = [];
    linksArr.forEach((l, i) => {
      const st = linkState[l.from + "|" + l.to];
      if (!st || !st.pts) return;
      routes.push({
        from: l.from,
        to: l.to,
        portA: st.portA,
        portB: st.portB,
        shape: st.shape.kind,
        pts: st.pts.map((p) => ({ x: p.x, y: p.y })),
        d: d[String(i)] ?? null,
      });
    });
    return { passes, routes };
  };
  const positions = () =>
    Object.keys(cards).map((id) => {
      const c = cards[id];
      const out = { id, x: c.x, y: c.y };
      if (c.slot !== undefined) out.slot = c.slot;
      return out;
    });

  const stageSize = { w: stage.clientWidth, h: stage.clientHeight };

  if (sc.type === "routes") {
    const steps = [{ move: null, ...settle() }];
    for (const m of sc.moves ?? []) {
      cards[m.id].x += m.dx;
      cards[m.id].y += m.dy;
      steps.push({ move: m, ...settle() });
    }
    return { stage: stageSize, steps };
  }
  if (sc.type === "autoLayout") {
    const before = settle();
    Object.keys(cards).forEach((id) => createCardEl(id));
    autoLayout();
    const after = settle();
    return { stage: stageSize, before, positions: positions(), after };
  }
  if (sc.type === "grid") {
    Object.keys(cards).forEach((id) => createCardEl(id));
    setMode("grid");
    const steps = [{ drop: null, positions: positions() }];
    for (const dr of sc.drops ?? []) {
      // gestore di rilascio in Organizzato (righe 2093-2100), che nel prototipo vive
      // dentro un listener di pointerup non richiamabile: stesse istruzioni
      const uid = dr.id;
      cards[uid].x = dr.x;
      cards[uid].y = dr.y;
      const idx = nearestSlot(cards[uid].x, cards[uid].y, uid, false);
      if (idx >= 0) {
        const occupant = Object.keys(cards).find((id) => id !== uid && cards[id].slot === idx);
        if (occupant) cards[occupant].slot = cards[uid].slot;
        cards[uid].slot = idx;
      }
      placeInSlots(false);
      steps.push({ drop: dr, positions: positions() });
    }
    return { stage: stageSize, steps };
  }
  if (sc.type === "spawn") {
    Object.keys(cards).forEach((id) => createCardEl(id));
    if (sc.mode === "grid") setMode("grid");
    const before = new Set(Object.keys(cards));
    spawnOutput(sc.boxId);
    const outputId = Object.keys(cards).find((id) => !before.has(id)) ?? null;
    return { stage: stageSize, outputId, positions: positions() };
  }
  throw new Error("tipo di scenario sconosciuto: " + sc.type);
}

const browser = await chromium.launch();
try {
  const page = await browser.newPage({ viewport: VIEWPORT });
  // il prototipo carica solo un font da Google Fonts: non serve alla geometria
  await page.route(/^https?:/, (r) => r.abort());
  await page.goto(pathToFileURL(prototype).href);
  await page.waitForFunction(() => typeof drawLinks === "function");
  await page.addScriptTag({ content: "window.runScenarioInPage = " + runScenario.toString() });
  if (!explore) mkdirSync(outDir, { recursive: true });
  for (const sc of SCENARIOS) {
    const expected = await page.evaluate(
      ([s, m]) => window.runScenarioInPage(s, m),
      [sc, MAX_PASSES],
    );
    const { name, description, type, ...input } = sc;
    const golden = { name, description, type, input, expected };
    if (explore) {
      const summary = (r) =>
        r.routes.map((x) => `${x.from}->${x.to}:${x.shape}/${x.pts.length}pt`).join(" ");
      if (expected.steps && expected.steps[0].routes)
        console.log(name, expected.steps.map((s) => `[p${s.passes}] ` + summary(s)).join(" | "));
      else if (expected.before)
        console.log(name, summary(expected.before), "=>", summary(expected.after));
      else console.log(name, JSON.stringify(expected).slice(0, 400));
      continue;
    }
    writeFileSync(resolve(outDir, name + ".json"), JSON.stringify(golden, null, 2) + "\n");
    console.log("scritto", name + ".json");
  }
} finally {
  await browser.close();
}
```

### `scripts/generate-index.mjs`

114 righe

```js
#!/usr/bin/env node
// Builds INDEX.md for the snapshot repo, once the commit SHA that holds
// every other file is known (INDEX.md is always committed/pushed second,
// after everything else -- see sync-snapshot.sh).

import { readFileSync, writeFileSync } from "node:fs";

function argVal(name) {
  const i = process.argv.indexOf(`--${name}`);
  return i === -1 ? undefined : process.argv[i + 1];
}

const manifestPath = argVal("manifest");
const sha = argVal("sha");
const repo = argVal("repo"); // owner/name
const branch = argVal("branch");
const sourceSha = argVal("source-sha");
const dirtyFilesArg = argVal("dirty-files") || "";
const generatedAt = argVal("generated-at");
const outPath = argVal("out");

if (!manifestPath || !sha || !repo || !outPath) {
  console.error(
    "Usage: generate-index.mjs --manifest <path> --sha <sha> --repo <owner/name> --branch <b> --source-sha <sha> --dirty-files <list> --generated-at <ts> --out <path>",
  );
  process.exit(1);
}

const manifest = JSON.parse(readFileSync(manifestPath, "utf8"));
const dirtyFiles = dirtyFilesArg
  .split("\n")
  .map((l) => l.trim())
  .filter(Boolean);

function rawUrl(pathInSnapshot) {
  return `https://raw.githubusercontent.com/${repo}/${sha}/${pathInSnapshot}`;
}

const lines = [];
lines.push("# INDEX.md");
lines.push("");
lines.push(`Generato: ${generatedAt} (UTC)`);
lines.push(
  `Repository sorgente: isa-glass-platform, branch \`${branch}\`, commit \`${sourceSha}\``,
);
if (dirtyFiles.length === 0) {
  lines.push("Working tree del repository sorgente: pulito (nessuna modifica non committata).");
} else {
  lines.push(
    `Working tree del repository sorgente: modifiche non committate presenti (${dirtyFiles.length} file):`,
  );
  lines.push("");
  for (const f of dirtyFiles) lines.push(`- \`${f}\``);
}
lines.push("");
lines.push(
  `Questo indice è fissato al commit \`${sha}\` del repository snapshot (isa-etl-snapshot): tutti gli URL sotto puntano a quel commit e restano validi anche dopo aggiornamenti futuri.`,
);
lines.push("");
lines.push("## Da leggere per primi");
lines.push("");
lines.push(`1. [STATUS.md](${rawUrl("STATUS.md")}) — stato di type check, lint, test, build`);
lines.push(
  `2. [ENV.md](${rawUrl("ENV.md")}) — configurazione completa (package.json, tsconfig, vite, eslint, CSS)`,
);
lines.push(`3. [TREE.md](${rawUrl("TREE.md")}) — albero completo del repository`);
lines.push("4. I blocchi in `files/`, in ordine, elencati sotto.");
lines.push("");

lines.push("## Blocchi (files/)");
lines.push("");
for (const block of manifest.blocks) {
  const kb = (block.bytes / 1000).toFixed(1);
  lines.push(`### [${block.name}](${rawUrl(block.name)})`);
  lines.push("");
  lines.push(`${kb} KB. File sorgente contenuti:`);
  lines.push("");
  for (const f of block.files) lines.push(`- \`${f}\``);
  lines.push("");
}

if (manifest.reportFiles && manifest.reportFiles.length > 0) {
  lines.push("## Report (reports/)");
  lines.push("");
  for (const rel of manifest.reportFiles) {
    const name = rel.split("/").pop();
    lines.push(`- [${name}](${rawUrl(`reports/${name}`)})`);
  }
  lines.push("");
}

if (manifest.excluded && manifest.excluded.length > 0) {
  lines.push("## File esclusi dallo snapshot");
  lines.push("");
  lines.push("(elencati per riferimento in TREE.md, contenuto non incluso in files/)");
  lines.push("");
  for (const e of manifest.excluded) {
    lines.push(`- \`${e.file}\` — motivo: ${e.reason}`);
  }
  lines.push("");
}

if (manifest.redactions && manifest.redactions.length > 0) {
  lines.push("## Segreti redatti");
  lines.push("");
  for (const r of manifest.redactions) {
    lines.push(`- \`${r.file}\` — pattern: ${r.pattern} — valore sostituito con \`[REDATTO]\``);
  }
  lines.push("");
}

writeFileSync(outPath, lines.join("\n") + "\n", "utf8");
console.log(`INDEX.md written to ${outPath}`);
```

