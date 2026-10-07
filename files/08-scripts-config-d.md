# 08-scripts-config-d.md

File in questo blocco:

- `scripts/e2e-fase6b2.mjs`
- `scripts/extract-golden.mjs`
- `scripts/generate-index.mjs`

---

### `scripts/e2e-fase6b2.mjs`

827 righe

```js
#!/usr/bin/env node
/**
 * Verifica nel browser reale delle condizioni di filtro e join (Fase 6b.2), con eventi veri
 * (clic, tastiera, puntatore), a 1440×900 e 1280×720, con l'Inspector a destra, in basso e in alto:
 *  - filtro: tre condizioni, raggruppa le ultime due, cambia il connettore interno, legge l'anteprima;
 *    scioglie e riprova; un annullamento per un gesto di digitazione;
 *  - join: colonna = colonna, solo disuguaglianze (avviso di prestazioni), valore e lista; scelta
 *    Colonna | Valore | Lista da tastiera; colonne per tabella;
 *  - layout a tre colonne a 1440 e a 1280: larghezze, nessun overflow orizzontale, scorrimento
 *    indipendente, intestazioni fisse, voce attiva che sopravvive ai ridisegni, ritorno alle colonne
 *    CSS sotto la larghezza minima; sui bordi laterali le righe si impilano;
 *  - operazioni a voci («Elenco») e riordino dei criteri di Ordina (maniglia con il puntatore e Alt+↑/↓,
 *    un annullamento per spostamento);
 *  - ogni tendina a ≥ 16 px dai bordi della finestra e senza coprire il campo;
 *  - il canvas resta con tutti i nodi richiesti dentro l'area dopo ogni apertura.
 * Salva le schermate in docs/visual/fase6b2/ (chiaro, scuro e due in tema notte) e le misure in
 * misure.json. Esce con codice 1 se una prova fallisce.
 *
 * Uso: node scripts/e2e-fase6b2.mjs
 */
import { mkdirSync, writeFileSync } from "node:fs";
import { resolve } from "node:path";
import { chromium } from "playwright";
import { ROOT, SOLUTION, startServer } from "./visual-lib.mjs";

const OUT = resolve(ROOT, "docs/visual/fase6b2");
mkdirSync(OUT, { recursive: true });
const server = await startServer(Number(process.env.PORT ?? 5198));
const browser = await chromium.launch();

const WINDOWS = [
  { width: 1440, height: 900, name: "1440" },
  { width: 1280, height: 720, name: "1280x720" },
];
const SIDES = ["right", "bottom", "top"];
const MARGIN = 16;
const sleep = (page, ms) => page.waitForTimeout(ms);

let failed = 0;
const results = [];
const measures = [];
function check(name, ok, extra = "") {
  results.push({ prova: name, esito: ok ? "ok" : "FALLITA", dettaglio: String(extra).slice(0, 220) });
  if (!ok) failed++;
}
const errors = [];
let retries = 0;

const blankCond = { column: "", op: "=", mode: "list", values: [], text: "", sep: "," };
const blankKey = {
  left: "",
  right: "",
  op: "=",
  lmode: "col",
  rmode: "col",
  lval: "",
  rval: "",
  rlist: { mode: "list", values: [], text: "", sep: "," },
};

async function newPage(vp) {
  const ctx = await browser.newContext({ viewport: { width: vp.width, height: vp.height }, deviceScaleFactor: 1 });
  await ctx.addInitScript((s) => {
    localStorage.setItem("isa.solutions", JSON.stringify([s]));
    localStorage.setItem("isa-theme", "light");
  }, SOLUTION);
  const page = await ctx.newPage();
  page.on("console", (m) => m.type() === "error" && errors.push(m.text()));
  page.on("pageerror", (e) => errors.push(String(e)));
  page.on("requestfailed", (r) =>
    errors.push(`richiesta fallita: ${r.method()} ${r.url()} (${r.failure()?.errorText})`),
  );
  await page.goto(`${server.base}/solutions/${SOLUTION.id}/etl?seed=prototype`);
  await page.waitForSelector('[data-node-id="ds1"]', { timeout: 90000 });
  await sleep(page, 600);
  return { ctx, page };
}

/** Gli strumenti di un test, legati a una pagina. */
function tools(page) {
  const dispatch = (c) => page.evaluate((cmd) => window.__etlStore.dispatch(cmd), c);
  const state = () => page.evaluate(() => window.__etlStore.getState());
  const paramsOf = async (node, step = 0) => (await state()).graph.cards[node].params[step];
  const rectOf = (sel) =>
    page.evaluate((s) => {
      const e = document.querySelector(s);
      if (!e) return null;
      const r = e.getBoundingClientRect();
      return { x: r.x, y: r.y, w: r.width, h: r.height };
    }, sel);
  const hit = (a, b) => a.x < b.x + b.w && b.x < a.x + a.w && a.y < b.y + b.h && b.y < a.y + a.h;
  /** Dove stanno i campi della voce aperta (sul posto) o attiva (nel dettaglio). */
  const body = () => page.locator(".ec-insp .ei-row-body, .ec-insp .ei-md-fields");
  const sel = (n) => body().locator("button.ei-select").nth(n);
  const preview = async () =>
    (await page.locator('[data-testid="ei-preview-text"]').innerText().catch(() => "")).trim();

  const openInspector = async (node, side) => {
    // riparte da zero: l'Inspector chiuso dimentica la voce attiva dello scenario precedente
    await dispatch({ type: "setPanel", payload: { panel: "insp", side, open: false } });
    await sleep(page, 300);
    await dispatch({ type: "setPanel", payload: { panel: "insp", side, open: true } });
    await dispatch({ type: "inspect", payload: { node, step: 0 } });
    await sleep(page, 800);
  };

  /** Apre una tendina, misura il menu e la sua distanza dai bordi e dal campo; chiude con Esc. */
  async function checkMenu(label, trigger, fieldSelector) {
    // il campo si misura DOPO il clic: Playwright porta l'elemento in vista e il pannello scorre
    const measureField = () =>
      fieldSelector
        ? rectOf(fieldSelector)
        : trigger.evaluate((e) => {
            const r = e.getBoundingClientRect();
            return { x: r.x, y: r.y, w: r.width, h: r.height };
          });
    await trigger.click();
    if (!(await page.waitForSelector('[role="listbox"]', { timeout: 1000 }).then(() => true, () => false))) {
      retries++;
      await trigger.click();
      await page.waitForSelector('[role="listbox"]', { timeout: 1500 }).catch(() => {});
    }
    await sleep(page, 250);
    const field = await measureField();
    const info = await page.evaluate(() => {
      const lb = document.querySelector('[role="listbox"]');
      if (!lb) return null;
      const menu = lb.closest(".ei-menu");
      const r = menu.getBoundingClientRect();
      return {
        inPortal: !!lb.closest("#ei-portal"),
        rect: { x: r.x, y: r.y, w: r.width, h: r.height },
        win: { w: innerWidth, h: innerHeight },
        selects: document.querySelectorAll("select, datalist").length,
      };
    });
    const ok =
      !!info &&
      info.inPortal &&
      info.selects === 0 &&
      info.rect.x >= MARGIN - 0.5 &&
      info.rect.y >= MARGIN - 0.5 &&
      info.rect.x + info.rect.w <= info.win.w - MARGIN + 0.5 &&
      info.rect.y + info.rect.h <= info.win.h - MARGIN + 0.5 &&
      !hit(info.rect, field);
    check(
      `${label}: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale`,
      ok,
      info ? `${Math.round(info.rect.x)},${Math.round(info.rect.y)} ${Math.round(info.rect.w)}×${Math.round(info.rect.h)}` : "assente",
    );
    if (info) {
      measures.push({ prova: label, menu: info.rect, finestra: info.win });
      await page.keyboard.press("Escape");
      await sleep(page, 150);
    }
    return info;
  }

  /** Apre il campo dei valori («+»); se il primo clic si perde in un ridisegno, ne dà un secondo. */
  async function openValues() {
    const add = body().locator('[data-picker="values"] .ei-add');
    for (let i = 0; i < 2; i++) {
      await add.click();
      await sleep(page, 200);
      if (await page.evaluate(() => document.activeElement?.tagName === "INPUT" && !!document.activeElement.closest("#ei-portal, .ec-insp"))) return;
      retries++;
    }
  }

  /** Sceglie una voce di una tendina scrivendo nella ricerca e premendo Invio. */
  async function pick(trigger, text) {
    await sleep(page, 150);
    let opened = false;
    for (let i = 0; i < 3 && !opened; i++) {
      if (i > 0) retries++;
      if ((await page.locator('[role="listbox"]').count()) === 0) await trigger.click();
      opened = await page.waitForSelector('[role="listbox"]', { timeout: 1200 }).then(() => true, () => false);
    }
    if (!opened) {
      retries++;
      await trigger.click();
      opened = await page.waitForSelector('[role="listbox"]', { timeout: 1500 }).then(() => true, () => false);
    }
    if (!opened) {
      check(`pick «${text}»: la tendina si apre`, false, `${page.viewportSize().width}px, campi: ${await page.locator(".ec-insp button.ei-select").count()}`);
      return;
    }
    await sleep(page, 200);
    await page.keyboard.type(text);
    await sleep(page, 100);
    await page.keyboard.press("Enter");
    await page.waitForSelector('[role="listbox"]', { state: "detached", timeout: 2000 }).catch(() => {});
    await sleep(page, 250);
  }

  /** I nodi richiesti (interamente dentro il canvas con 24 px di margine) prima di un cambio. */
  const visibleNodes = () =>
    page.evaluate(() => {
      const st = document.querySelector(".ec-stage").getBoundingClientRect();
      return [...document.querySelectorAll("[data-node-id]")]
        .map((el) => {
          const r = el.getBoundingClientRect();
          return { id: el.dataset.nodeId, x: r.left, y: r.top, w: r.width, h: r.height };
        })
        .map((n) => ({ ...n, stage: { x: st.left, y: st.top, w: st.width, h: st.height } }));
    });
  const insideStage = (n, margin = 24) =>
    n.x >= n.stage.x + margin - 1 &&
    n.y >= n.stage.y + margin - 1 &&
    n.x + n.w <= n.stage.x + n.stage.w - margin + 1 &&
    n.y + n.h <= n.stage.y + n.stage.h - margin + 1;

  return { openValues, dispatch, state, paramsOf, rectOf, hit, body, sel, preview, openInspector, checkMenu, pick, visibleNodes, insideStage };
}

/** Schermata nei due temi (chiaro e scuro), e a richiesta nel tema notte. */
async function shot(page, name, { notte } = {}) {
  const path = (s) => resolve(OUT, `${name}-${s}.png`);
  await page.screenshot({ path: path("chiaro"), animations: "disabled" });
  await page.evaluate(() => document.documentElement.classList.add("dark"));
  await sleep(page, 150);
  await page.screenshot({ path: path("scuro"), animations: "disabled" });
  await page.evaluate(() => document.documentElement.classList.remove("dark"));
  if (notte) {
    await page.evaluate(() => document.documentElement.setAttribute("data-theme", "notte"));
    if (notte === "scuro") await page.evaluate(() => document.documentElement.classList.add("dark"));
    await sleep(page, 150);
    await page.screenshot({ path: path(`notte-${notte}`), animations: "disabled" });
    await page.evaluate(() => {
      document.documentElement.classList.remove("dark");
      document.documentElement.removeAttribute("data-theme");
    });
  }
  await sleep(page, 100);
}

// ---------------------------------------------------------------------------------------------
// scene di prova

/** Una seconda sorgente («Clienti») per il join: colonne diverse dalla prima. */
async function addClienti(page) {
  await page.evaluate(() => {
    const st = window.__etlStore.getState();
    const ds1 = st.graph.cards["ds1"];
    const cols = [
      { name: "cliente", type: "stringa", values: ["Acme", "Borealis", "Cedro"] },
      { name: "agente", type: "stringa", values: ["Rossi", "Bianchi"] },
      { name: "sconto", type: "numerico", values: ["5", "10"] },
    ];
    const ds2 = {
      ...ds1,
      id: "ds2",
      name: "Clienti",
      y: ds1.y + 140,
      params: [{ ...ds1.params[0], path: "clienti.csv", columns: cols }],
    };
    window.__etlStore.replaceState({ ...st, graph: { ...st.graph, cards: { ...st.graph.cards, ds2 } } });
  });
}

// ---------------------------------------------------------------------------------------------
// 1. filtro: tre condizioni, gruppo, connettore, anteprima, scioglimento, annullamento

async function filterScenario(page, T, W, side, withShots) {
  const tag = `${W} ${side}`;
  await T.dispatch({
    type: "setParams",
    payload: { node: "op-filter", index: 0, params: { conditions: [{ ...blankCond }] } },
  });
  await T.openInspector("op-filter", side);
  const nodes0 = (await T.visibleNodes()).filter((n) => T.insideStage(n));

  // condizione 1: regione = Nord
  await T.pick(T.sel(0), "regione");
  await T.openValues();
  await page.keyboard.type("Nord");
  await page.keyboard.press("Enter");
  await sleep(page, 150);
  // condizione 2: importo > 100 (valore scritto)
  await page.locator('.ec-insp [data-focus="add"]').click();
  await sleep(page, 200);
  await T.pick(T.sel(0), "importo");
  await T.pick(T.sel(1), ">");
  await T.body().locator("input.ei-input").click();
  await page.keyboard.type("100");
  await sleep(page, 100);
  // condizione 3: stato = Chiuso
  await page.locator('.ec-insp [data-focus="add"]').click();
  await sleep(page, 200);
  await T.pick(T.sel(0), "stato");
  await T.openValues();
  await page.keyboard.type("Chiuso");
  await page.keyboard.press("Enter");
  await sleep(page, 200);

  const conds = (await T.paramsOf("op-filter")).conditions;
  check(
    `${tag} filtro: tre condizioni create con eventi veri (regione = Nord, importo > 100, stato = Chiuso)`,
    conds.length === 3 &&
      conds[0].column === "regione" && conds[0].op === "=" && conds[0].values.join() === "Nord" &&
      conds[1].column === "importo" && conds[1].op === ">" && conds[1].text === "100" &&
      conds[2].column === "stato" && conds[2].op === "=" && conds[2].values.join() === "Chiuso",
    JSON.stringify(conds.map((c) => [c.column, c.op, c.values, c.text])),
  );
  check(
    `${tag} filtro: ogni condizione ha un connettore tra sé e la precedente, nessun selettore globale E/O`,
    (await page.locator(".ec-insp button.ei-pill").count()) === 2 &&
      (await page.getByText("Tutte le condizioni").count()) === 0,
  );
  check(
    `${tag} filtro: anteprima senza gruppi, da sinistra a destra`,
    (await T.preview()) === "(regione = Nord AND importo > 100) AND stato = Chiuso",
    await T.preview(),
  );

  // raggruppa le ultime due; il focus resta su un pulsante equivalente
  await page.locator('.ec-insp [data-focus="conn-2"]').click();
  await sleep(page, 250);
  check(
    `${tag} filtro: «Raggruppa» sulle ultime due crea un gruppo (riquadro «Gruppo» con «Sciogli»)`,
    (await page.locator('.ec-insp [role="group"][aria-label="Gruppo"]').count()) === 1 &&
      (await page.getByRole("button", { name: "Sciogli il gruppo" }).count()) === 1,
  );
  check(
    `${tag} filtro: dopo aver raggruppato il focus non si perde (resta sul pulsante del connettore)`,
    await page.evaluate(() => document.activeElement?.getAttribute("data-focus") === "conn-2"),
  );
  // cambia il connettore interno
  const innerPill = page.locator('.ec-insp button[aria-label="Connettore tra la condizione 2 e la 3"]');
  await T.checkMenu(`${tag} connettore`, innerPill);
  if (withShots) {
    await innerPill.click();
    await sleep(page, 250);
    await shot(page, "connettore-aperto");
    await page.keyboard.press("Escape");
    await sleep(page, 150);
  }
  await T.pick(innerPill, "OR");
  check(
    `${tag} filtro: il connettore interno diventa OR e l'anteprima è «regione = Nord AND (importo > 100 OR stato = Chiuso)»`,
    (await T.preview()) === "regione = Nord AND (importo > 100 OR stato = Chiuso)",
    await T.preview(),
  );
  const grouped = (await T.paramsOf("op-filter")).conditions;
  check(
    `${tag} filtro: le ultime due condizioni stanno nello stesso gruppo, la prima no`,
    !grouped[0].g && grouped[1].g && grouped[1].g === grouped[2].g && grouped[2].conn === "OR",
  );
  if (withShots) {
    await shot(page, "condizioni-filtro-gruppi", { notte: "chiaro" });
    await page.locator('[data-testid="ei-preview"]').scrollIntoViewIfNeeded();
    await sleep(page, 150);
    await shot(page, "anteprima");
  }

  // un annullamento per un gesto di digitazione (più di un secondo dopo l'ultima modifica)
  await page.locator('.ec-insp [data-row="1"] .ei-row-toggle').click();
  await sleep(page, 1300);
  await T.body().locator("input.ei-input").click();
  await page.keyboard.press("Control+a");
  await page.keyboard.type("250", { delay: 40 });
  await sleep(page, 150);
  check(
    `${tag} filtro: scrivere nel campo non fa perdere il focus`,
    await page.evaluate(() => document.activeElement?.classList.contains("ei-input") === true),
  );
  const typed = (await T.paramsOf("op-filter")).conditions[1].text;
  await page.getByRole("button", { name: "Annulla", exact: true }).click();
  await sleep(page, 200);
  const undone = (await T.paramsOf("op-filter")).conditions[1].text;
  check(
    `${tag} filtro: un solo annullamento riporta il campo a prima della digitazione`,
    typed === "250" && undone === "100",
    `${typed} → ${undone}`,
  );

  // scioglie e riprova
  await page.getByRole("button", { name: "Sciogli il gruppo" }).click();
  await sleep(page, 250);
  check(
    `${tag} filtro: «Sciogli» toglie il gruppo, le condizioni restano e l'anteprima segue`,
    (await page.locator('.ec-insp [role="group"][aria-label="Gruppo"]').count()) === 0 &&
      (await T.paramsOf("op-filter")).conditions.length === 3 &&
      (await T.preview()) === "(regione = Nord AND importo > 100) OR stato = Chiuso",
    await T.preview(),
  );
  await page.locator('.ec-insp [data-focus="conn-1"]').click();
  await sleep(page, 250);
  check(
    `${tag} filtro: raggruppa di nuovo (prime due) e l'anteprima porta le parentesi sul gruppo`,
    (await page.locator('.ec-insp [role="group"][aria-label="Gruppo"]').count()) === 1 &&
      (await T.preview()) === "(regione = Nord AND importo > 100) OR stato = Chiuso",
    await T.preview(),
  );

  // tendine: colonna, operatore, valori, connettore esterno
  await page.locator('.ec-insp [data-row="2"] .ei-row-toggle').click();
  await sleep(page, 200);
  await T.checkMenu(`${tag} colonna`, T.sel(0));
  await T.checkMenu(`${tag} operatore`, T.sel(1));
  await T.checkMenu(`${tag} valori`, T.body().locator('[data-picker="values"] .ei-add'), ".ec-insp [data-picker=\"values\"]");
  await T.checkMenu(
    `${tag} connettore esterno`,
    page.locator('.ec-insp button[aria-label="Connettore tra la condizione 2 e la 3"]'),
  );

  // aggiungi nel gruppo e rimuovi: un gruppo con una sola condizione si scioglie da sé
  await page.getByRole("button", { name: /Condizione nel gruppo/ }).click();
  await sleep(page, 250);
  check(
    `${tag} filtro: «+ Condizione nel gruppo» aggiunge una voce dentro il gruppo`,
    (await T.paramsOf("op-filter")).conditions.filter((c) => c.g).length === 3,
  );
  await page.locator('.ec-insp [data-row="2"]').getByRole("button", { name: "Rimuovi condizione" }).click();
  await sleep(page, 250);
  await page.locator('.ec-insp [data-row="1"]').getByRole("button", { name: "Rimuovi condizione" }).click();
  await sleep(page, 250);
  const left = (await T.paramsOf("op-filter")).conditions;
  check(
    `${tag} filtro: un gruppo che resta con una sola condizione si scioglie da sé`,
    left.length === 2 && left.every((c) => !c.g),
    JSON.stringify(left.map((c) => c.g ?? null)),
  );

  // il canvas: i nodi richiesti restano dentro l'area
  const after = await T.visibleNodes();
  const bad = nodes0.filter((n0) => {
    const n = after.find((a) => a.id === n0.id);
    return !n || !T.insideStage(n);
  });
  check(`${tag} canvas: i nodi richiesti restano dentro l'area con l'Inspector aperto`, bad.length === 0, bad.map((b) => b.id).join());
}

// ---------------------------------------------------------------------------------------------
// 2. join

async function joinScenario(page, T, W, side, withShots) {
  const tag = `${W} ${side}`;
  await T.dispatch({
    type: "setParams",
    payload: { node: "op-join", index: 0, params: { type: "inner", keys: [{ ...blankKey }] } },
  });
  await T.openInspector("op-join", side);
  const radios = (group) => page.locator(".ec-insp [role=radiogroup]").nth(group).getByRole("radio");

  // colonna = colonna, ciascun lato dalla propria tabella
  await T.pick(T.sel(0), "cliente");
  await T.pick(T.sel(2), "cliente");
  let keys = (await T.paramsOf("op-join")).keys;
  check(
    `${tag} join: colonna = colonna (cliente = cliente) con eventi veri`,
    keys[0].left === "cliente" && keys[0].right === "cliente" && keys[0].op === "=",
    JSON.stringify(keys[0]),
  );
  check(
    `${tag} join: con una colonna = colonna non c'è l'avviso di prestazioni`,
    (await page.locator('[data-testid="ei-join-perf"]').count()) === 0,
  );
  // colonne per lato: la sinistra dalla tabella sinistra (Vendite), la destra dalla destra (Clienti)
  const optionsOf = async (trigger) => {
    await trigger.click();
    await sleep(page, 200);
    const t = await page.locator(".ei-menu .ei-option-label").allInnerTexts();
    await page.keyboard.press("Escape");
    await sleep(page, 150);
    return t;
  };
  const leftCols = await optionsOf(T.sel(0));
  const rightCols = await optionsOf(T.sel(2));
  check(
    `${tag} join: le colonne del lato sinistro vengono dalla tabella sinistra, quelle del destro dalla destra`,
    leftCols.includes("regione") && !leftCols.includes("agente") && rightCols.includes("agente") && !rightCols.includes("regione"),
    `sinistra: ${leftCols.join("|")} — destra: ${rightCols.join("|")}`,
  );

  // solo disuguaglianze: avviso
  await T.pick(T.sel(1), "minore di");
  check(
    `${tag} join: solo disuguaglianze → avviso di prestazioni`,
    (await page.locator('[data-testid="ei-join-perf"]').count()) === 1 &&
      (await page.locator('[data-testid="ei-join-perf"]').innerText()).includes("Nessuna condizione di uguaglianza"),
  );
  if (withShots) await shot(page, "join-avviso-prestazioni");
  await T.pick(T.sel(1), "uguale a");
  check(
    `${tag} join: tornando a «=» l'avviso sparisce`,
    (await page.locator('[data-testid="ei-join-perf"]').count()) === 0,
  );

  // valore a destra (dominio della colonna dell'altro lato) → colonna = valore: avviso
  await radios(1).filter({ hasText: "Valore" }).click();
  await sleep(page, 250);
  await T.pick(T.sel(2), "Acme");
  keys = (await T.paramsOf("op-join")).keys;
  check(
    `${tag} join: lato destro «Valore» con i valori della colonna dell'altro lato (cliente = “Acme”)`,
    keys[0].rmode === "val" && keys[0].rval === "Acme",
    JSON.stringify(keys[0]),
  );
  check(
    `${tag} join: colonna = valore non è un'uguaglianza tra colonne → avviso`,
    (await page.locator('[data-testid="ei-join-perf"]').count()) === 1,
  );

  // lista a destra: il confronto diventa «è uno di»
  await radios(1).filter({ hasText: "Lista" }).click();
  await sleep(page, 250);
  keys = (await T.paramsOf("op-join")).keys;
  check(
    `${tag} join: con una lista a destra il confronto passa a «è uno di»`,
    keys[0].rmode === "list" && keys[0].op === "è uno di",
    JSON.stringify([keys[0].rmode, keys[0].op]),
  );
  await T.openValues();
  await page.keyboard.type("Acme");
  await page.keyboard.press("Enter");
  await sleep(page, 250);
  await page.keyboard.press("Control+a");
  await page.keyboard.type("Borealis");
  await page.keyboard.press("Enter");
  await sleep(page, 200);
  keys = (await T.paramsOf("op-join")).keys;
  check(
    `${tag} join: lista di valori dal dominio della colonna dell'altro lato (Acme, Borealis)`,
    keys[0].rlist.values.join() === "Acme,Borealis",
    JSON.stringify(keys[0].rlist.values),
  );
  await T.checkMenu(`${tag} join confronto`, T.sel(1));
  await T.checkMenu(`${tag} join valori della lista`, T.body().locator('[data-picker="values"] .ei-add'), '.ec-insp [data-picker="values"]');
  // uscendo dalla lista il confronto torna «=»
  await radios(1).filter({ hasText: "Colonna" }).click();
  await sleep(page, 250);
  keys = (await T.paramsOf("op-join")).keys;
  check(`${tag} join: uscendo dalla lista il confronto torna «=»`, keys[0].rmode === "col" && keys[0].op === "=");

  // scelta Colonna | Valore | Lista da tastiera (radiogroup)
  await radios(1).filter({ hasText: "Colonna" }).focus();
  await page.keyboard.press("ArrowRight");
  await sleep(page, 150);
  let r1 = await page.evaluate(() => {
    const g = document.querySelectorAll(".ec-insp [role=radiogroup]")[1];
    const on = g.querySelector('[aria-checked="true"]');
    return { on: on?.textContent, focused: document.activeElement === on, tab: on?.getAttribute("tabindex") };
  });
  check(
    `${tag} join: freccia destra sul radiogroup sceglie «Valore» e sposta il focus (tabindex 0 solo sulla scelta)`,
    r1.on === "Valore" && r1.focused && r1.tab === "0",
    JSON.stringify(r1),
  );
  await page.keyboard.press("End");
  await sleep(page, 150);
  r1 = await page.evaluate(() => document.querySelectorAll(".ec-insp [role=radiogroup]")[1].querySelector('[aria-checked="true"]')?.textContent);
  check(`${tag} join: Fine sceglie «Lista»`, r1 === "Lista", String(r1));
  await page.keyboard.press("Home");
  await sleep(page, 150);
  r1 = await page.evaluate(() => document.querySelectorAll(".ec-insp [role=radiogroup]")[1].querySelector('[aria-checked="true"]')?.textContent);
  check(`${tag} join: Home sceglie «Colonna»`, r1 === "Colonna", String(r1));

  // una seconda condizione colonna = colonna: nessun avviso; il tipo di join dal catalogo
  await T.pick(T.sel(2), "cliente");
  if (withShots) {
    // tre condizioni: colonna, valore e lista (per la schermata)
    await T.dispatch({
      type: "setParams",
      payload: {
        node: "op-join",
        index: 0,
        params: {
          type: "left",
          keys: [
            { ...blankKey, left: "cliente", right: "cliente" },
            { ...blankKey, left: "importo", rmode: "val", rval: "100", op: "≥", conn: "AND" },
            { ...blankKey, left: "regione", rmode: "list", op: "è uno di", rlist: { mode: "list", values: ["Nord", "Sud"], text: "", sep: "," }, conn: "AND" },
          ],
        },
      },
    });
    await sleep(page, 400);
    await shot(page, "join-condizioni-colonna-valore-lista");
  }
  await page.locator('.ec-insp [data-focus="add"]').click();
  await sleep(page, 250);
  await T.pick(T.sel(0), "id");
  await T.pick(T.sel(2), "cliente");
  const typeSel = page.locator(".ec-insp .ei-fieldgroup", { has: page.locator(".ei-label", { hasText: "Tipo di join" }) }).locator("button.ei-select");
  await T.pick(typeSel, "left");
  const par = await T.paramsOf("op-join");
  check(`${tag} join: il tipo di join si sceglie dalle impostazioni (valori del catalogo)`, par.type === "left", par.type);
}

// ---------------------------------------------------------------------------------------------
// 3. layout a tre colonne

async function columnsScenario(page, T, W, side, withShots) {
  const tag = `${W} ${side}`;
  const many = Array.from({ length: 7 }, (_, i) => ({
    ...blankCond,
    column: ["regione", "stato", "cliente", "categoria", "id", "importo", "data"][i],
    op: i % 2 ? "≠" : "=",
    values: [["Nord"], ["Chiuso"], ["Acme"], ["Hardware"], ["1"], ["300"], ["2026-01-03"]][i],
    ...(i > 0 ? { conn: "AND" } : {}),
  }));
  await T.dispatch({ type: "setParams", payload: { node: "op-filter", index: 0, params: { conditions: many } } });
  await T.openInspector("op-filter", side);
  const three = (await page.locator('[data-testid="ei-cols3"]').count()) === 1;
  if (side === "right") {
    check(`${tag} layout: sul bordo laterale le parti si impilano con le righe comprimibili (nessuna tre colonne)`, !three && (await page.locator(".ec-insp .ei-row-body").count()) === 1);
    return;
  }
  check(`${tag} layout: a 1280 e 1440, sui bordi alto e basso, tre colonne`, three);
  const cols = await page.evaluate(() => {
    const w = (s) => document.querySelector(s)?.getBoundingClientRect().width ?? 0;
    const root = document.querySelector('[data-testid="ei-cols3"]').getBoundingClientRect();
    return { general: w(".ei-col-general"), master: w(".ei-col-master"), detail: w(".ei-col-detail"), root: root.width };
  });
  check(
    `${tag} layout: larghezze da token (Impostazioni 240, centrale 310) e il dettaglio prende il resto`,
    Math.abs(cols.general - 240) <= 1 && Math.abs(cols.master - 310) <= 1 && Math.abs(cols.general + cols.master + cols.detail - cols.root) <= 1.5,
    JSON.stringify(cols),
  );
  const heads = await page.locator(".ec-insp .ei-col-head").first().evaluate(() => [...document.querySelectorAll(".ec-insp .ei-col-head")].slice(0, 2).map((e) => e.textContent));
  check(`${tag} layout: intestazioni «Impostazioni» e «Condizioni e gruppi»`, heads.join("|") === "Impostazioni|Condizioni e gruppi", heads.join("|"));
  const overflow = await page.evaluate(() => {
    const insp = document.querySelector(".ec-insp");
    const c3 = document.querySelector(".ei-cols3");
    return { insp: insp.scrollWidth - insp.clientWidth, c3: c3.scrollWidth - c3.clientWidth, page: document.documentElement.scrollWidth - innerWidth };
  });
  check(`${tag} layout: nessun overflow orizzontale`, overflow.insp <= 1 && overflow.c3 <= 1 && overflow.page <= 0, JSON.stringify(overflow));

  // scorrimento indipendente e intestazioni fisse
  const master = page.locator('[data-scroll-col="master"]');
  const headY0 = await page.locator(".ei-col-master .ei-col-head").evaluate((e) => e.getBoundingClientRect().y);
  const scrollable = await master.evaluate((e) => e.scrollHeight > e.clientHeight + 4);
  await master.evaluate((e) => (e.scrollTop = 80));
  await sleep(page, 100);
  const sc = await page.evaluate(() => ({
    master: document.querySelector('[data-scroll-col="master"]').scrollTop,
    detail: document.querySelector('[data-scroll-col="detail"]').scrollTop,
    general: document.querySelector('[data-scroll-col="general"]').scrollTop,
    headY: document.querySelector(".ei-col-master .ei-col-head").getBoundingClientRect().y,
  }));
  check(`${tag} layout: la colonna centrale scorre per conto suo, le altre restano ferme`, scrollable && sc.master > 0 && sc.detail === 0 && sc.general === 0, JSON.stringify(sc));
  check(`${tag} layout: l'intestazione della colonna resta ferma mentre il contenuto scorre`, Math.abs(sc.headY - headY0) < 0.5, `${headY0} → ${sc.headY}`);

  // un clic su una voce la mostra nel dettaglio senza comprimerla; la voce attiva è evidenziata
  await page.locator('.ec-insp [data-row="3"] .ei-row-toggle').click();
  await sleep(page, 200);
  const t3 = await page.evaluate(() => ({
    title: document.querySelector(".ei-md-title")?.textContent,
    sum: document.querySelector(".ei-md-sum")?.textContent,
    rowSum: document.querySelector('.ei-row[data-row="3"] .ei-row-sum')?.textContent,
    active: [...document.querySelectorAll(".ei-row[data-active]")].map((r) => r.dataset.row).join(),
    bodiesInMaster: document.querySelectorAll(".ei-col-master .ei-row-body").length,
  }));
  check(
    `${tag} layout: il clic su «Condizione 4» la mostra nel dettaglio, evidenziata e senza comprimerla; il titolo ripete numero e riassunto`,
    t3.title === "Condizione 4" && t3.sum === t3.rowSum && t3.active === "3" && t3.bodiesInMaster === 0,
    JSON.stringify(t3),
  );
  // la voce attiva e lo scorrimento sopravvivono ai ridisegni (si scrive, il pannello si ridisegna)
  const masterBefore = await master.evaluate((e) => e.scrollTop);
  await page.locator(".ei-col-detail button.ei-select").nth(1).click();
  await sleep(page, 150);
  await page.keyboard.type("contiene");
  await page.keyboard.press("Enter");
  await sleep(page, 250);
  const after = await page.evaluate(() => ({
    title: document.querySelector(".ei-md-title")?.textContent,
    active: [...document.querySelectorAll(".ei-row[data-active]")].map((r) => r.dataset.row).join(),
    master: document.querySelector('[data-scroll-col="master"]').scrollTop,
    sum: document.querySelector('.ei-row[data-row="3"] .ei-row-sum')?.textContent,
  }));
  check(
    `${tag} layout: dopo una modifica la voce attiva e la posizione di scorrimento sono quelle di prima, e il riassunto è aggiornato`,
    after.title === "Condizione 4" && after.active === "3" && Math.abs(after.master - masterBefore) <= 1 && /contiene/.test(after.sum),
    JSON.stringify(after),
  );
  if (withShots) {
    await master.evaluate((e) => (e.scrollTop = 0));
    await page.locator('.ec-insp [data-row="1"] .ei-row-toggle').click();
    await sleep(page, 300);
    if (W === "1440" && side === "bottom") await shot(page, "inspector-tre-colonne-1440", { notte: "scuro" });
    if (W === "1280x720" && side === "bottom") await shot(page, "inspector-tre-colonne-1280x720");
    if (side === "top" && W === "1440") await shot(page, "inspector-tre-colonne-alto");
  }
}

async function narrowScenario(page, T, W) {
  // sotto la larghezza minima tornano le colonne CSS della 6b.1
  await T.openInspector("op-filter", "bottom");
  const vp = page.viewportSize();
  await page.setViewportSize({ width: 860, height: vp.height });
  await sleep(page, 700);
  const narrow = await page.evaluate(() => ({
    cols3: document.querySelectorAll('[data-testid="ei-cols3"]').length,
    columnWidth: getComputedStyle(document.querySelector(".ei-root")).columnWidth,
    rootWidth: document.querySelector(".ei-root").clientWidth,
    rows: document.querySelectorAll(".ec-insp .ei-row").length,
  }));
  check(
    `${W}: sotto la larghezza minima (token, 900 px) tornano le colonne CSS della 6b.1`,
    narrow.cols3 === 0 && narrow.columnWidth === "280px" && narrow.rows > 0 && narrow.rootWidth < 900,
    JSON.stringify(narrow),
  );
  await page.setViewportSize(vp);
  await sleep(page, 700);
  check(`${W}: ripristinata la larghezza tornano le tre colonne`, (await page.locator('[data-testid="ei-cols3"]').count()) === 1);
}

// ---------------------------------------------------------------------------------------------
// 4. operazioni a voci e riordino dei criteri

async function listScenario(page, T, W, side, withShots) {
  const tag = `${W} ${side}`;
  // operazione a voci: Converti tipo con due righe → «Elenco»
  await T.dispatch({
    type: "setParams",
    payload: { node: "op-sort", index: 0, params: { items: [
      { columns: ["regione"], dir: "crescente" },
      { columns: ["importo"], dir: "decrescente" },
      { columns: ["stato"], dir: "crescente" },
    ] } },
  });
  await T.openInspector("op-sort", side);
  const three = (await page.locator('[data-testid="ei-cols3"]').count()) === 1;
  if (side !== "right") {
    check(`${tag} elenco: l'operazione a voci ha «Elenco» come colonna centrale`, three && (await page.locator(".ei-col-master .ei-col-head").textContent()) === "Elenco");
    if (withShots) await shot(page, "operazione-a-voci-tre-colonne");
  }
  const order = async () => (await T.paramsOf("op-sort")).items.map((r) => r.columns[0]).join();
  const grip = (i) => page.locator(`.ec-insp .ei-list > [data-row="${i}"] .ei-row-grip`);

  check(`${tag} ordina: una maniglia per criterio, con nome accessibile`, (await page.locator(".ec-insp .ei-row-grip").count()) === 3 && (await grip(0).getAttribute("aria-label")) === "Sposta Criterio 1");

  // tastiera: Alt+↓ sulla maniglia
  await grip(0).focus();
  await page.keyboard.press("Alt+ArrowDown");
  await sleep(page, 250);
  check(`${tag} ordina: Alt+↓ sposta il primo criterio in seconda posizione`, (await order()) === "importo,regione,stato", await order());
  check(
    `${tag} ordina: il focus segue la riga spostata e lo spostamento è annunciato`,
    (await page.evaluate(() => document.activeElement?.closest("[data-row]")?.dataset.row)) === "1" &&
      /posizione 2 di 3/.test(await page.locator('.ec-insp .ei-list [role="status"]').innerText()),
  );
  await sleep(page, 1200);
  await page.keyboard.press("Alt+ArrowUp");
  await sleep(page, 250);
  check(`${tag} ordina: Alt+↑ lo riporta in prima posizione`, (await order()) === "regione,importo,stato", await order());
  // un annullamento per spostamento
  await sleep(page, 1200);
  await page.keyboard.press("Alt+ArrowDown");
  await sleep(page, 250);
  await page.getByRole("button", { name: "Annulla", exact: true }).click();
  await sleep(page, 250);
  check(`${tag} ordina: annullare uno spostamento da tastiera lo riporta com'era, in un passo`, (await order()) === "regione,importo,stato", await order());

  // puntatore: la maniglia del primo criterio fino all'ultimo
  await sleep(page, 1200);
  const g0 = await grip(0).boundingBox();
  const last = await page.locator(".ec-insp .ei-list > [data-row=\"2\"]").boundingBox();
  await page.mouse.move(g0.x + g0.width / 2, g0.y + g0.height / 2);
  await page.mouse.down();
  const targetY = last.y + last.height / 2;
  const steps = 12;
  for (let i = 1; i <= steps; i++) {
    await page.mouse.move(g0.x + g0.width / 2, g0.y + g0.height / 2 + ((targetY - (g0.y + g0.height / 2)) * i) / steps);
    await sleep(page, 15);
  }
  if (withShots) await shot(page, "ordina-riordino");
  await page.mouse.up();
  await sleep(page, 300);
  check(`${tag} ordina: trascinare la maniglia del primo criterio in fondo lo porta per ultimo`, (await order()) === "importo,stato,regione", await order());
  await page.getByRole("button", { name: "Annulla", exact: true }).click();
  await sleep(page, 250);
  check(`${tag} ordina: un solo annullamento riporta il trascinamento com'era`, (await order()) === "regione,importo,stato", await order());
  // Esc durante il trascinamento annulla
  await sleep(page, 1200);
  const g1 = await grip(0).boundingBox();
  await page.mouse.move(g1.x + g1.width / 2, g1.y + g1.height / 2);
  await page.mouse.down();
  await page.mouse.move(g1.x + g1.width / 2, g1.y + g1.height / 2 + 120, { steps: 6 });
  await page.keyboard.press("Escape");
  await page.mouse.up();
  await sleep(page, 250);
  check(`${tag} ordina: Esc durante il trascinamento lo annulla`, (await order()) === "regione,importo,stato", await order());
}

// ---------------------------------------------------------------------------------------------

try {
  for (const win of WINDOWS) {
    const { ctx, page } = await newPage(win);
    const T = tools(page);
    // il nodo di prova: filtro, join (con due sorgenti) e Ordina collegati
    await addClienti(page);
    await T.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
    await T.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
    await T.dispatch({ type: "connect", payload: { from: "ds2", to: "op-join" } });
    await T.dispatch({ type: "connect", payload: { from: "ds1", to: "op-sort" } });
    await sleep(page, 400);
    const first = win.name === "1440";
    for (const side of SIDES) {
      await filterScenario(page, T, win.name, side, first && side === "right");
      await joinScenario(page, T, win.name, side, first && side === "right");
      await columnsScenario(page, T, win.name, side, first || win.name === "1280x720");
      await listScenario(page, T, win.name, side, first && side === "bottom");
    }
    await narrowScenario(page, T, win.name);
    await ctx.close();
  }
} finally {
  await browser.close();
  server.stop();
}

check("nessun errore in console", errors.length === 0, errors.slice(0, 2).join(" | "));
writeFileSync(resolve(OUT, "misure.json"), JSON.stringify({ risultati: results, misure: measures }, null, 2));
console.table(results.map((r, i) => ({ "#": i, prova: r.prova.slice(0, 118), esito: r.esito })));
for (const r of results) if (r.esito !== "ok") console.log("FAIL", r.prova, "=>", r.dettaglio);
console.log(`\nprove: ${results.length}, fallite: ${failed}, secondi clic sulle tendine: ${retries}`);
const menus = measures.filter((m) => m.menu);
if (menus.length) {
  const gap = (m) => Math.min(m.menu.x, m.menu.y, m.finestra.w - (m.menu.x + m.menu.w), m.finestra.h - (m.menu.y + m.menu.h));
  console.log(`tendine misurate: ${menus.length}, distanza minima dai bordi: ${Math.min(...menus.map(gap)).toFixed(1)} px`);
}
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

