# 08-scripts-config-b.md

File in questo blocco:

- `scripts/e2e-fase6a.mjs`

---

### `scripts/e2e-fase6a.mjs`

770 righe

```js
#!/usr/bin/env node
/**
 * Verifica nel browser reale dei pannelli e della cassetta (Fase 6a): apertura,
 * chiusura, tacche, trascinamento tra i quattro bordi, schede condivise,
 * compensazione della vista, Inspector che segue la selezione, caricamento di
 * un CSV, trascinamento dalla cassetta, persistenza. Salva le schermate in
 * docs/visual/fase6a/. Esce con codice 1 se una prova fallisce.
 *
 * Uso: node scripts/e2e-fase6a.mjs
 */
import { mkdirSync, writeFileSync } from "node:fs";
import { resolve } from "node:path";
import { chromium } from "playwright";
import { ROOT, SOLUTION, startServer } from "./visual-lib.mjs";

const OUT = resolve(ROOT, "docs/visual/fase6a");
mkdirSync(OUT, { recursive: true });
const server = await startServer(Number(process.env.PORT ?? 5194));
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
const moves = [];
const results = [];
function check(name, ok, extra = "") {
  results.push({
    prova: name,
    esito: ok ? "ok" : "FALLITA",
    dettaglio: String(extra).slice(0, 160),
  });
  if (!ok) failed++;
}
const state = () => page.evaluate(() => window.__etlStore.getState());
const box = async (sel) => await page.locator(sel).first().boundingBox();
const center = (b) => ({ x: b.x + b.width / 2, y: b.y + b.height / 2 });
const shot = (name) => page.screenshot({ path: resolve(OUT, name), animations: "disabled" });
const nodeBox = (id) => box(`[data-node-id="${id}"] .ec-icon-wrap`);
const URL = `${server.base}/solutions/${SOLUTION.id}/etl?seed=prototype`;

async function dragNotch(key, side) {
  const n = await box(`[data-notch="${key}"]`);
  const s = await box(".ec-stage");
  const from = center(n);
  const to = {
    left: { x: s.x + 6, y: s.y + s.height / 2 },
    right: { x: s.x + s.width - 6, y: s.y + s.height / 2 },
    top: { x: s.x + s.width / 2, y: s.y + 6 },
    bottom: { x: s.x + s.width / 2, y: s.y + s.height - 6 },
  }[side];
  await page.mouse.move(from.x, from.y);
  await page.mouse.down();
  await page.mouse.move((from.x + to.x) / 2, (from.y + to.y) / 2, { steps: 6 });
  await page.mouse.move(to.x, to.y, { steps: 6 });
  if (process.env.HOLD) await page.waitForTimeout(200);
  await page.mouse.up();
  await page.waitForTimeout(500);
}
const panelSide = (key) => page.locator(`[data-panel="${key}"]`).getAttribute("data-side");
const isOpen = async (key) => (await state()).panels[key].open;

try {
  await page.goto(URL);
  await page.waitForSelector('[data-node-id="ds1"]', { timeout: 90000 });
  await page.waitForTimeout(600);

  // 1. la cassetta è aperta a sinistra e deriva dal catalogo
  check(
    "la cassetta è aperta a sinistra all'avvio",
    (await panelSide("tools")) === "left" && (await isOpen("tools")),
  );
  const secs = await page.locator("[data-sec]").evaluateAll((els) => els.map((e) => e.dataset.sec));
  check(
    "cinque sezioni nell'ordine del catalogo",
    secs.join() === "data,rows,xform,merge,out",
    secs.join(),
  );
  const items = await page.locator(".ec-pal-item").count();
  check("tutte le operazioni del catalogo sono presenti (19)", items === 19, String(items));
  await shot("cassetta-sinistra.png");

  // 2. sezione comprimibile
  await page.locator('[data-sec="rows"] .ec-tb-sec-head').click();
  check("una sezione si comprime", (await page.locator('[data-sec="rows"].ec-open').count()) === 0);
  await page.locator('[data-sec="rows"] .ec-tb-sec-head').click();

  // helper di geometria (Fase 6a.2): nodi, area del canvas, pannello aperto, widget
  const rectOf = (sel) =>
    page.evaluate((s) => {
      const e = document.querySelector(s);
      if (!e) return null;
      const r = e.getBoundingClientRect();
      return { x: r.x, y: r.y, w: r.width, h: r.height };
    }, sel);
  const hit = (a, b) => a.x < b.x + b.w && b.x < a.x + a.w && a.y < b.y + b.h && b.y < a.y + a.h;
  const nodeRects = () =>
    page.evaluate(() =>
      Object.fromEntries(
        [...document.querySelectorAll("[data-node-id]")].map((e) => {
          const r = e.getBoundingClientRect();
          return [e.dataset.nodeId, { x: r.x, y: r.y, w: r.width, h: r.height }];
        }),
      ),
    );
  const insideRect = (r, o) =>
    r.x >= o.x - 0.5 &&
    r.y >= o.y - 0.5 &&
    r.x + r.w <= o.x + o.w + 0.5 &&
    r.y + r.h <= o.y + o.h + 0.5;
  const visibleNodeIds = async () => {
    const st = await rectOf(".ec-stage");
    const nodes = await nodeRects();
    return Object.keys(nodes).filter((id) => insideRect(nodes[id], st));
  };
  const openPanelRect = () => rectOf(".ec-panel.ec-open");
  const closeBtn = (name) => page.getByRole("button", { name });
  const TOOLS_CLOSE = "Nascondi la cassetta degli strumenti";
  const INSP_CLOSE = "Nascondi l’inspector";

  // 3. chiusura e riapertura a sinistra: i nodi si spostano con il bordo del canvas (spinta), nel mondo non si muovono
  const world0 = JSON.stringify((await state()).graph.cards);
  const before = await nodeBox("op-join");
  const stage0 = await box(".ec-stage");
  await closeBtn(TOOLS_CLOSE).click();
  await page.waitForTimeout(600);
  const after = await nodeBox("op-join");
  const stage1 = await box(".ec-stage");
  check(
    "chiudendo la cassetta a sinistra il bordo del canvas avanza e i nodi vanno con lui",
    Math.abs(after.x - before.x - (stage1.x - stage0.x)) < 1.5 && stage1.x < stage0.x,
    `${before.x}→${after.x}`,
  );
  check(
    "la posizione dei nodi rispetto al canvas non cambia",
    Math.abs(after.x - stage1.x - (before.x - stage0.x)) < 1.5 &&
      Math.abs(after.y - before.y) < 1.5,
  );
  check(
    "la cassetta è chiusa e la sua tacca è visibile",
    !(await isOpen("tools")) &&
      (await page.locator('[data-notch="tools"]:not(.ec-hidden)').count()) === 1,
  );
  await page.locator('[data-notch="tools"]').click();
  await page.waitForTimeout(600);
  const reopened = await nodeBox("op-join");
  check(
    "un clic sulla tacca riapre; i nodi tornano dove erano",
    (await isOpen("tools")) && Math.abs(reopened.x - before.x) < 1.5,
    `${before.x}→${reopened.x}`,
  );
  check(
    "le posizioni nel mondo non sono cambiate",
    JSON.stringify((await state()).graph.cards) === world0,
  );
  await shot("cassetta-sinistra.png");

  // 4. trascinamento tra i quattro bordi: nessun nodo prima visibile finisce coperto dal pannello
  await closeBtn(TOOLS_CLOSE).click();
  await page.waitForTimeout(500);
  const edgeShot = {
    right: "cassetta-destra.png",
    top: "cassetta-alto.png",
    bottom: "cassetta-basso.png",
  };
  for (const side of ["right", "top", "bottom", "left"]) {
    const visBefore = await visibleNodeIds();
    const nb = await nodeRects();
    const w0 = JSON.stringify((await state()).graph.cards);
    await dragNotch("tools", side);
    check(
      `la tacca trascinata sul bordo ${side} sposta la cassetta`,
      (await panelSide("tools")) === side && (await isOpen("tools")),
      await panelSide("tools"),
    );
    const panel = await openPanelRect();
    const st = await rectOf(".ec-stage");
    const na = await nodeRects();
    const covered = visBefore.filter((id) => hit(na[id], panel) || !insideRect(na[id], st));
    check(
      `bordo ${side}: i nodi prima visibili sono ancora interamente visibili e non coperti dal pannello`,
      covered.length === 0,
      covered.join(),
    );
    check(
      `bordo ${side}: i nodi nel mondo non si sono mossi`,
      JSON.stringify((await state()).graph.cards) === w0,
    );
    moves.push({
      bordo: side,
      zoom: (await state()).view.zoom,
      nodi: Object.fromEntries(
        Object.keys(nb).map((id) => [
          id,
          {
            prima: `${Math.round(nb[id].x)},${Math.round(nb[id].y)}`,
            dopo: `${Math.round(na[id].x)},${Math.round(na[id].y)}`,
          },
        ]),
      ),
    });
    if (side === "top" || side === "bottom") {
      check(
        `bordo ${side}: la cassetta è una fascia orizzontale`,
        (await page.locator(`[data-panel="tools"].ec-horiz`).count()) === 1,
      );
    }
    if (side !== "left") {
      await shot(edgeShot[side]);
      if (side === "top") await shot("nodi-spinti-pannello-in-alto.png");
      if (side === "bottom") await shot("minimappa-con-pannello-in-basso.png");
      await closeBtn(TOOLS_CLOSE).click();
      await page.waitForTimeout(500);
    }
  }

  // 4b. geometria dei widget e dei pannelli orizzontali per ogni bordo e finestra
  const widgetCheck = async (label) => {
    const st = await rectOf(".ec-stage");
    const bar = await rectOf('[data-testid="ec-bar"]');
    const items = [];
    const mm =
      (await rectOf('[data-testid="minimap"]')) ?? (await rectOf('[data-testid="minimap-toggle"]'));
    if (mm) items.push(["minimappa", mm]);
    items.push(["zoom", await rectOf('[data-testid="ec-zoom"]')]);
    for (const k of ["tools", "insp"]) {
      const n = await page.locator(`[data-notch="${k}"]:not(.ec-hidden)`).count();
      if (n) items.push([`tacca ${k}`, await rectOf(`[data-notch="${k}"]`)]);
    }
    const problems = [];
    for (const [name, r] of items) {
      if (name !== "minimappa" || r) {
        if (!insideRect(r, st)) problems.push(`${name} fuori dall'area`);
        if (hit(r, bar)) problems.push(`${name} tocca la barra`);
      }
    }
    for (let i = 0; i < items.length; i++)
      for (let j = i + 1; j < items.length; j++)
        if (hit(items[i][1], items[j][1])) problems.push(`${items[i][0]} tocca ${items[j][0]}`);
    const vp = page.viewportSize();
    const panel = await openPanelRect();
    if (panel && !insideRect(panel, { x: 0, y: 0, w: vp.width, h: vp.height }))
      problems.push("il pannello esce dalla finestra");
    const doc = await page.evaluate(() => document.scrollingElement.scrollHeight <= innerHeight);
    if (!doc) problems.push("la pagina scorre");
    check(
      `${label}: nessun widget si tocca e tutto sta nell'area`,
      problems.length === 0,
      problems.join("; "),
    );
    return { mm, st };
  };
  for (const vp of [
    { width: 1440, height: 900 },
    { width: 1280, height: 720 },
    { width: 1280, height: 600 },
  ]) {
    await page.setViewportSize(vp);
    await page.waitForTimeout(700);
    for (const side of ["left", "right", "top", "bottom"]) {
      if (await isOpen("tools")) {
        await closeBtn(TOOLS_CLOSE).click();
        await page.waitForTimeout(450);
      }
      await dragNotch("tools", side);
      const label = `${vp.width}×${vp.height}, cassetta a ${side}`;
      const { mm, st } = await widgetCheck(label);
      if (side === "bottom" && vp.width === 1440 && mm)
        check(
          "pannello in basso: la minimappa è in alto a sinistra",
          mm.y < st.y + st.h / 2 && mm.x < st.x + 40,
          `${Math.round(mm.x)},${Math.round(mm.y)}`,
        );
      if (side !== "bottom" && vp.width === 1440 && mm)
        check(
          `pannello a ${side}: la minimappa è in basso a sinistra`,
          mm.y > st.y + st.h / 2 && mm.x < st.x + 40,
          `${Math.round(mm.x)},${Math.round(mm.y)}`,
        );
      if (vp.height === 600 && side === "bottom")
        await page.screenshot({
          path: resolve(OUT, "cassetta-basso-finestra-bassa.png"),
          animations: "disabled",
        });
    }
    // chiuso: tacche visibili, nessuna sovrapposizione
    await closeBtn(TOOLS_CLOSE).click();
    await page.waitForTimeout(450);
    await widgetCheck(`${vp.width}×${vp.height}, pannelli chiusi`);
    await page.locator('[data-notch="tools"]').click();
    await page.waitForTimeout(450);
  }
  await page.setViewportSize({ width: 1440, height: 900 });
  await page.waitForTimeout(700);
  await shot("cassetta-basso-orizzontale.png").catch(() => {});

  // sotto l'altezza minima scorre il contenitore dello spazio di lavoro, non la pagina
  await page.setViewportSize({ width: 1280, height: 520 });
  await page.waitForTimeout(700);
  const low = await page.evaluate(() => {
    const host = document.querySelector(".ec-workspace-host");
    return {
      scrolls: host.scrollHeight > host.clientHeight + 1,
      page: document.scrollingElement.scrollHeight <= innerHeight,
      stage: document.querySelector(".ec-stage").getBoundingClientRect().height,
    };
  });
  check(
    "finestra molto bassa: il canvas non scende sotto il minimo e scorre il contenitore, non la pagina",
    low.page && low.stage >= 160,
    JSON.stringify(low),
  );
  await page.setViewportSize({ width: 1440, height: 900 });
  await page.waitForTimeout(700);

  // ripristino della disposizione iniziale: cassetta a sinistra, Inspector chiuso a destra
  if (await isOpen("tools")) {
    await closeBtn(TOOLS_CLOSE).click();
    await page.waitForTimeout(450);
  }
  await dragNotch("tools", "left");
  check(
    "disposizione iniziale ripristinata",
    (await panelSide("tools")) === "left" && (await isOpen("tools")) && !(await isOpen("insp")),
  );

  // 5. schede: un solo pannello aperto alla volta, anche condividendo il bordo
  await closeBtn(TOOLS_CLOSE).click();
  await page.waitForTimeout(400);
  await page.locator('[data-notch="tools"]').click();
  await page.waitForTimeout(500);
  await dragNotch("insp", "left");
  check(
    "l'Inspector trascinato sul bordo della cassetta: schede sullo stesso bordo",
    (await panelSide("insp")) === "left" &&
      (await page.locator(".ec-panel.ec-grouped").count()) === 2,
  );
  check("è aperta una sola scheda", (await isOpen("insp")) && !(await isOpen("tools")));
  await shot("schede-stesso-bordo.png");
  const w1 = (await box('[data-panel="insp"]')).width;
  await page.locator('[data-panel="insp"] .ec-dock-tab', { hasText: "Strumenti" }).click();
  await page.waitForTimeout(100);
  check(
    "cambiare scheda sostituisce il contenuto sul posto",
    (await isOpen("tools")) && !(await isOpen("insp")),
  );
  await page.waitForTimeout(400);
  const w2 = (await box('[data-panel="tools"]')).width;
  check("la larghezza non cambia al cambio di scheda", Math.abs(w1 - w2) < 1.5, `${w1} vs ${w2}`);
  const nTab0 = await nodeBox("op-join");
  await page.locator('[data-panel="tools"] .ec-dock-tab', { hasText: "Inspector" }).click();
  await page.waitForTimeout(500);
  const nTab1 = await nodeBox("op-join");
  check(
    "il cambio di scheda non sposta i nodi",
    Math.abs(nTab1.x - nTab0.x) < 1.5 && Math.abs(nTab1.y - nTab0.y) < 1.5,
  );
  // separati di nuovo
  await closeBtn(INSP_CLOSE).click();
  await page.waitForTimeout(400);
  await dragNotch("insp", "right");
  check(
    "spostato su un altro bordo torna un pannello separato",
    (await panelSide("insp")) === "right" &&
      (await page.locator(".ec-panel.ec-grouped").count()) === 0,
  );
  check(
    "spostare l'Inspector su un bordo diverso lo apre e chiude la cassetta",
    (await isOpen("insp")) && !(await isOpen("tools")),
  );
  await page.locator('[data-notch="tools"]').click();
  await page.waitForTimeout(500);
  check(
    "aprire la cassetta chiude l'Inspector anche su un altro bordo",
    (await isOpen("tools")) && !(await isOpen("insp")),
  );

  // 6. l'Inspector si apre solo al clic su un nodo
  await page.waitForTimeout(400);
  const j = center(await nodeBox("op-join"));
  await page.mouse.move(j.x, j.y);
  await page.mouse.down();
  await page.waitForTimeout(150);
  check(
    "alla pressione l'Inspector non si apre",
    !(await isOpen("insp")) && (await isOpen("tools")),
  );
  await page.mouse.up();
  await page.waitForTimeout(500);
  check(
    "al clic si apre l'Inspector con il nome del nodo e la cassetta si chiude",
    (await isOpen("insp")) &&
      !(await isOpen("tools")) &&
      (await page.locator(`[data-testid="ec-inspector-name"]`).inputValue()) === "Unisci (Join)",
  );
  await page.keyboard.press("Escape");
  await page.waitForTimeout(500);
  check(
    "deselezionare chiude l'Inspector e riapre la cassetta (memoria di sostituzione)",
    !(await isOpen("insp")) && (await isOpen("tools")),
  );
  // un'azione esplicita azzera la memoria
  await page.mouse.click(j.x, j.y);
  await page.waitForTimeout(500);
  await closeBtn(INSP_CLOSE).click();
  await page.waitForTimeout(450);
  await page.keyboard.press("Escape");
  await page.waitForTimeout(450);
  check(
    "chiudere l'Inspector a mano azzera la memoria: la cassetta non si riapre",
    !(await isOpen("insp")) && !(await isOpen("tools")),
  );
  await page.locator('[data-notch="tools"]').click();
  await page.waitForTimeout(500);
  // trascinare un nodo non apre
  const sortB = center(await nodeBox("op-sort"));
  await page.mouse.move(sortB.x, sortB.y);
  await page.mouse.down();
  await page.mouse.move(sortB.x + 40, sortB.y + 30, { steps: 6 });
  await page.waitForTimeout(100);
  check("durante un trascinamento l'Inspector non si apre", !(await isOpen("insp")));
  await page.mouse.up();
  await page.waitForTimeout(300);
  check(
    "dopo un trascinamento l'Inspector non si apre",
    !(await isOpen("insp")) && (await isOpen("tools")),
  );
  await page.keyboard.press("Control+z");
  await page.waitForTimeout(300);
  // riquadro di selezione
  const stNow = await box(".ec-stage");
  await page.mouse.move(stNow.x + stNow.width - 120, stNow.y + 8);
  await page.mouse.down();
  await page.mouse.move(stNow.x + 4, stNow.y + stNow.height - 140, { steps: 8 });
  await page.mouse.up();
  await page.waitForTimeout(300);
  check(
    "un riquadro di selezione non apre l'Inspector",
    (await state()).selection.length > 1 && !(await isOpen("insp")) && (await isOpen("tools")),
    String((await state()).selection.length),
  );
  await page.keyboard.press("Escape");
  await page.waitForTimeout(300);
  // selezione multipla con Maiusc
  await page.keyboard.down("Shift");
  await page.mouse.click(j.x, j.y);
  const d1 = center(await nodeBox("ds1"));
  await page.mouse.click(d1.x, d1.y);
  await page.keyboard.up("Shift");
  await page.waitForTimeout(300);
  check(
    "una selezione multipla non apre l'Inspector",
    (await state()).selection.length === 2 && !(await isOpen("insp")) && (await isOpen("tools")),
  );
  await page.keyboard.press("Escape");
  await page.waitForTimeout(300);

  // 7. caricamento CSV e trascinamento dalla cassetta
  await page.locator('[data-testid="ec-file-input"]').setInputFiles({
    name: "clienti.csv",
    mimeType: "text/csv",
    buffer: Buffer.from("id,nome,fatturato\n1,Acme,10.5\n2,Delta,20\n3,Eureka,31.25\n"),
  });
  await page.waitForSelector('.ec-pal-item[data-lib="lib-1"]', { timeout: 5000 });
  check(
    "il CSV compare nella libreria come voce trascinabile",
    (await page.locator('.ec-pal-item[data-lib="lib-1"] .ec-lib-meta').innerText()) ===
      "3 col · 3 righe",
  );
  check(
    "il messaggio di caricamento è visibile",
    (await page.locator(".ec-tb-status").innerText()).includes(
      "clienti.csv caricato: 3 colonne, 3 righe",
    ),
  );
  const lib = (await state()).library[0];
  check(
    "colonne e tipi dedotti",
    lib.columns.map((c) => c.type).join() === "integer,stringa,numerico",
    lib.columns.map((c) => c.type).join(),
  );

  const stage = await box(".ec-stage");
  const n0 = Object.keys((await state()).graph.cards).length;
  // nel vuoto: nodo isolato
  let src = center(await box('.ec-pal-item[data-type="limit"]'));
  await page.mouse.move(src.x, src.y);
  await page.mouse.down();
  await page.mouse.move(stage.x + 700, stage.y + 300, { steps: 12 });
  check(
    "durante il trascinamento compare l'anteprima (ghost)",
    (await page.locator('[data-testid="ec-ghost"]').count()) === 1,
  );
  await page.mouse.up();
  check(
    "rilasciato sul vuoto crea un nodo isolato",
    Object.keys((await state()).graph.cards).length === n0 + 1 &&
      (await state()).graph.links.length === 0,
  );

  // su un box compatibile: contorno di fusione e fusione
  src = center(await box('.ec-pal-item[data-type="filter"]'));
  const sortC = center(await nodeBox("op-sort"));
  await page.mouse.move(src.x, src.y);
  await page.mouse.down();
  await page.mouse.move(sortC.x, sortC.y, { steps: 14 });
  check(
    "sopra un box compatibile compare il contorno di fusione",
    (await page.locator('[data-node-id="op-sort"].ec-drop-merge').count()) === 1,
  );
  await shot("trascinamento-dalla-cassetta.png");
  await page.mouse.up();
  check(
    "al rilascio la voce si fonde nel box",
    (await state()).graph.cards["op-sort"].components.length === 2,
  );

  check(
    "dopo i rilasci dalla cassetta, la cassetta resta aperta e l'Inspector chiuso",
    (await isOpen("tools")) && !(await isOpen("insp")),
  );

  // dataset della libreria su una lavorazione: collegamento
  src = center(await box('.ec-pal-item[data-lib="lib-1"]'));
  const joinC = center(await nodeBox("op-join"));
  await page.mouse.move(src.x, src.y);
  await page.mouse.down();
  await page.mouse.move(joinC.x, joinC.y, { steps: 14 });
  check(
    "un dataset sopra una lavorazione mostra il collegamento",
    (await page.locator('[data-node-id="op-join"].ec-drop-link').count()) === 1,
  );
  await page.mouse.up();
  const g = (await state()).graph;
  const created = Object.values(g.cards).find((k) => k.name === "clienti");
  check(
    "il dataset caricato è collegato",
    !!created && g.links.some((l) => l.from === created.id && l.to === "op-join"),
  );

  // su un cavo valido: inserimento
  const cable = await page.evaluate(() => {
    const st = window.__etlStore;
    const k = Object.keys(st.getRoutes()).find((x) =>
      x.startsWith(
        st
          .getState()
          .graph.links.find(
            (l) =>
              st.getState().graph.cards[l.from].kind === "dataset" &&
              st.getState().graph.cards[l.to].kind === "op",
          ).from + "|",
      ),
    );
    const pts = st.getRoutes()[k].pts;
    const v = st.getState().view;
    const r = document.querySelector(".ec-stage").getBoundingClientRect();
    return {
      key: k,
      x: r.left + v.x + ((pts[0].x + pts[1].x) / 2) * v.zoom,
      y: r.top + v.y + ((pts[0].y + pts[1].y) / 2) * v.zoom,
    };
  });
  src = center(await box('.ec-pal-item[data-type="sort"]'));
  await page.mouse.move(src.x, src.y);
  await page.mouse.down();
  await page.mouse.move(cable.x, cable.y, { steps: 14 });
  check(
    "su un cavo valido il cavo si evidenzia",
    (await page.locator(".ec-link.ec-link-hot").count()) === 1,
  );
  await shot("trascinamento-dalla-cassetta-su-cavo.png");
  await page.mouse.up();
  check(
    "al rilascio la lavorazione si inserisce nel cavo",
    !(await state()).graph.links.some((l) => `${l.from}|${l.to}` === cable.key),
  );

  // 9. barra dei controlli: una riga sopra il canvas, dentro lo spazio di lavoro
  const barR = await rectOf('[data-testid="ec-bar"]');
  const stR = await rectOf(".ec-stage");
  const wsR = await rectOf(".ec-workspace");
  check(
    "la barra è una riga fissa sopra l'area del canvas, dentro lo spazio di lavoro",
    barR.y + barR.h <= stR.y + 0.5 && barR.y >= wsR.y - 0.5 && !hit(barR, stR),
    `${Math.round(barR.y + barR.h)} ≤ ${Math.round(stR.y)}`,
  );
  const names = await page
    .locator('[data-testid="ec-bar"] button')
    .evaluateAll((els) => els.map((e) => e.getAttribute("aria-label") ?? e.textContent.trim()));
  check(
    "la barra ha Libero, Organizzato, Riordina, Annulla, Ripristina, Svuota (e non Funzionalità né Reimposta)",
    ["Libero", "Organizzato", "Riordina", "Annulla", "Ripristina", "Svuota il canvas"].every((n) =>
      names.includes(n),
    ) && !names.some((n) => /Reimposta|Funzionalit/i.test(n)),
    names.join(" | "),
  );
  await shot("barra-controlli.png");
  await page.getByRole("button", { name: "Organizzato" }).click();
  check("«Organizzato» imposta la modalità a griglia", (await state()).mode === "grid");
  await page.getByRole("button", { name: "Libero" }).click();
  check("«Libero» la ripristina", (await state()).mode === "free");
  const logLen = await page.evaluate(() => window.__etlStore.getLog().length);
  await page.getByRole("button", { name: "Riordina" }).click();
  check(
    "«Riordina» esegue autoLayout",
    (await page.evaluate(() => window.__etlStore.getLog().at(-1).type)) === "autoLayout" &&
      (await page.evaluate(() => window.__etlStore.getLog().length)) === logLen + 1,
  );
  check(
    "Annulla è abilitato dopo un'azione",
    !(await page.getByRole("button", { name: "Annulla", exact: true }).first().isDisabled()),
  );
  await page.getByRole("button", { name: "Annulla", exact: true }).first().click();
  check(
    "Annulla e Ripristina si usano dalla barra",
    !(await page.getByRole("button", { name: "Ripristina" }).isDisabled()),
  );

  // Svuota: sempre con conferma
  const cardsBefore = Object.keys((await state()).graph.cards).length;
  const libBefore = (await state()).library.length;
  const pastBefore = await page.evaluate(() => window.__etlStore.historySize().past);
  await page.getByRole("button", { name: "Svuota il canvas" }).click();
  await page.waitForSelector('[data-testid="ec-confirm"][data-kind="clear"]');
  check(
    "Svuota chiede conferma, con il testo previsto",
    (await page.locator('[data-testid="ec-confirm"]').innerText()).includes(
      "Eliminare tutti i nodi e i collegamenti? Puoi annullare con Cmd/Ctrl+Z.",
    ) && Object.keys((await state()).graph.cards).length === cardsBefore,
  );
  check(
    "il focus è sul pulsante Annulla",
    (await page.evaluate(() => document.activeElement?.textContent?.trim())) === "Annulla",
  );
  await shot("svuota-conferma.png");
  await page.keyboard.press("Escape");
  check(
    "Esc annulla: nessun nodo eliminato",
    (await page.locator('[data-testid="ec-confirm"]').count()) === 0 &&
      Object.keys((await state()).graph.cards).length === cardsBefore,
  );
  await page.getByRole("button", { name: "Svuota il canvas" }).click();
  await page.getByTestId("ec-confirm").getByRole("button", { name: "Annulla" }).click();
  check(
    "il pulsante Annulla della finestra non cambia nulla",
    Object.keys((await state()).graph.cards).length === cardsBefore,
  );
  await page.getByRole("button", { name: "Svuota il canvas" }).click();
  await page.getByTestId("ec-confirm").getByRole("button", { name: "Svuota" }).click();
  await page.waitForTimeout(300);
  const cleared = await state();
  check(
    "confermato: nessun nodo né collegamento, libreria intatta, un solo passo di cronologia",
    Object.keys(cleared.graph.cards).length === 0 &&
      cleared.graph.links.length === 0 &&
      cleared.library.length === libBefore &&
      (await page.evaluate(() => window.__etlStore.historySize().past)) === pastBefore + 1 &&
      (await page.evaluate(() => window.__etlStore.getLog().at(-1).type)) === "clearAll",
  );
  await page.getByRole("button", { name: "Annulla", exact: true }).first().click();
  await page.waitForTimeout(300);
  check(
    "Annulla ripristina tutti i nodi",
    Object.keys((await state()).graph.cards).length === cardsBefore &&
      (await state()).library.length === libBefore,
  );

  // 10. rotella: pan, Maiusc orizzontale, Cmd/Ctrl zoom; spazio e tasto centrale restano
  const sc = center(await box(".ec-stage"));
  await page.mouse.move(sc.x, sc.y);
  let v0 = (await state()).view;
  await page.mouse.wheel(0, 120);
  await page.waitForTimeout(150);
  let v1 = (await state()).view;
  check(
    "rotella semplice: scorre in verticale",
    v1.y < v0.y - 100 && Math.abs(v1.x - v0.x) < 1 && v1.zoom === v0.zoom,
    `${v0.y}→${v1.y}`,
  );
  await page.keyboard.down("Shift");
  await page.mouse.wheel(0, 120);
  await page.keyboard.up("Shift");
  await page.waitForTimeout(150);
  let v2 = (await state()).view;
  check(
    "Maiusc+rotella: scorre in orizzontale",
    v2.x < v1.x - 100 && Math.abs(v2.y - v1.y) < 1,
    `${v1.x}→${v2.x}`,
  );
  await page.keyboard.down("Control");
  await page.mouse.wheel(0, -200);
  await page.keyboard.up("Control");
  await page.waitForTimeout(150);
  let v3 = (await state()).view;
  check("Ctrl+rotella: zoom", v3.zoom > v2.zoom, `${v2.zoom}→${v3.zoom}`);
  await page.keyboard.down("Space");
  await page.mouse.move(sc.x, sc.y);
  await page.mouse.down();
  await page.mouse.move(sc.x + 50, sc.y + 30, { steps: 4 });
  await page.mouse.up();
  await page.keyboard.up("Space");
  let v4 = (await state()).view;
  check(
    "la barra spaziatrice con il trascinamento scorre ancora",
    Math.abs(v4.x - v3.x - 50) < 2 && Math.abs(v4.y - v3.y - 30) < 2,
  );
  await page.mouse.move(sc.x, sc.y);
  await page.mouse.down({ button: "middle" });
  await page.mouse.move(sc.x - 40, sc.y - 20, { steps: 4 });
  await page.mouse.up({ button: "middle" });
  let v5 = (await state()).view;
  check(
    "il tasto centrale scorre ancora",
    Math.abs(v5.x - v4.x + 40) < 2 && Math.abs(v5.y - v4.y + 20) < 2,
  );

  // 8. persistenza dei pannelli dopo il ricaricamento
  await dragNotch("tools", "right").catch(() => {});
  if (!(await isOpen("tools")) || (await panelSide("tools")) !== "right") {
    await page
      .locator('[data-notch="tools"]')
      .click()
      .catch(() => {});
  }
  await page
    .getByRole("button", { name: "Nascondi la cassetta degli strumenti" })
    .click()
    .catch(() => {});
  await page.waitForTimeout(300);
  await dragNotch("tools", "right");
  const savedSide = await panelSide("tools");
  await page.waitForTimeout(1500);
  await page.reload();
  await page.waitForSelector("[data-panel]", { timeout: 90000 });
  await page.waitForTimeout(600);
  check(
    "lato e stato dei pannelli sopravvivono al ricaricamento",
    (await panelSide("tools")) === savedSide && savedSide === "right" && (await isOpen("tools")),
    `${savedSide} → ${await panelSide("tools")}`,
  );

  check("nessun errore in console", errors.length === 0, errors.join(" | "));
} catch (e) {
  check("eccezione nello script", false, e);
} finally {
  await browser.close();
  server.stop();
}
writeFileSync(resolve(OUT, "posizioni-nodi.json"), JSON.stringify(moves, null, 2) + "\n");
console.table(
  moves.map((m) => ({
    bordo: m.bordo,
    ...Object.fromEntries(Object.entries(m.nodi).map(([id, p]) => [id, `${p.prima} → ${p.dopo}`])),
  })),
);
console.table(results);
console.log(failed ? `${failed} PROVE FALLITE` : `TUTTE LE ${results.length} PROVE SUPERATE`);
process.exit(failed ? 1 : 0);
```

