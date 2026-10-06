# 08-scripts-config-d.md

File in questo blocco:

- `scripts/e2e-fase6b11.mjs`
- `scripts/extract-golden.mjs`
- `scripts/generate-index.mjs`

---

### `scripts/e2e-fase6b11.mjs`

820 righe

```js
#!/usr/bin/env node
/**
 * Verifica nel browser reale dello zoom automatico (Fase 6b.1.1), con un
 * orologio controllato (come visual-fase4b) che avanza di 16 ms per volta e
 * un campione di OGNI frame: per apri/chiudi della cassetta e dell'Inspector
 * sui quattro bordi, cambio scheda, spostamento su un altro bordo e
 * ridimensionamento della finestra, a 1440×900 e 1280×720 con una scena
 * densa (32 nodi che riempiono tutta l'area sicura a pannelli chiusi).
 *
 * Requisiti verificati:
 *  R1 a riposo ogni nodo richiesto è interamente dentro l'area sicura e non
 *     tocca pannelli né widget;
 *  R2 a ogni frame sta dentro il canvas di quel frame (margine di 24 px, 0,5);
 *  R3 lo zoom è monotono e la transizione dura almeno 10 frame;
 *  R4 nessun salto: lo spostamento di un nodo tra due frame ≤ 2,5 volte la media;
 *  R5 con movimento ridotto lo stato finale è immediato;
 *  R6 apri → chiudi senza altre azioni: la vista finale è quella iniziale.
 * In più: una rotella a metà transizione ferma la vista senza salti; due
 * cambi ravvicinati non producono salti; il rendering lato server non cambia.
 *
 * Modi: (predefinito) tutte le verifiche e le schermate «dopo»;
 *       MODE=prima  solo le schermate «prima» (da un'istanza di sviluppo del
 *       commit precedente: BASE_URL=http://127.0.0.1:PORT).
 * Schermate e misure in docs/visual/fase6b11/. Esce con codice 1 se una prova fallisce.
 *
 * Uso: node scripts/e2e-fase6b11.mjs
 */
import { mkdirSync, writeFileSync } from "node:fs";
import { resolve } from "node:path";
import { chromium } from "playwright";
import { ROOT, SOLUTION, startServer } from "./visual-lib.mjs";

const OUT = resolve(ROOT, "docs/visual/fase6b11");
mkdirSync(OUT, { recursive: true });
const MODE = process.env.MODE ?? "full";
const MARGIN = 24;
const TOL = 0.5;
const STEP_MS = 16;
const DURATION_MS = 300;
const WINDOWS = [
  { w: 1440, h: 900, name: "1440" },
  { w: 1280, h: 720, name: "1280x720" },
];

const server = await startServer(Number(process.env.PORT ?? 5197));
const browser = await chromium.launch();

const CLOCK = `(() => {
  const c = { t: 1000, auto: true };
  window.__clock = c;
  performance.now = () => c.t;
  Date.now = () => 1.7e12 + c.t;
  setInterval(() => { if (c.auto) c.t += 16; }, 16);
  window.__raf = 0;
  const r = window.requestAnimationFrame.bind(window);
  window.requestAnimationFrame = (cb) => { window.__raf++; return r(cb); };
})();`;

/** Il campione di un istante: vista, canvas, rettangoli dei nodi (con l'etichetta: 88 × 110 allo zoom 1), widget, pannelli. */
const SNAP = `window.__snap = (ids) => {
  const rect = (el) => { const r = el.getBoundingClientRect(); return { x: r.left, y: r.top, w: r.width, h: r.height }; };
  const stage = document.querySelector(".ec-stage");
  const c = rect(stage);
  const v = window.__etlStore.getState().view;
  const nodes = {};
  for (const id of ids) {
    const el = document.querySelector('[data-node-id="' + id + '"]');
    if (!el) continue;
    const r = rect(el);
    const k = r.w / 88;
    nodes[id] = { x: r.x, y: r.y, w: 88 * k, h: 110 * k };
  }
  const all = (sel) => [...document.querySelectorAll(sel)].map(rect);
  const a = {};
  for (const k of ["tools", "insp"]) {
    const p = document.querySelector('.ec-panel[data-panel="' + k + '"]');
    a[k] = p ? Number(p.style.getPropertyValue("--a")) : null;
  }
  return {
    t: window.__clock.t,
    view: { x: v.x, y: v.y, zoom: v.zoom },
    canvas: c,
    insets: (document.querySelector(".ec-center")?.dataset.insets ?? "0,0,0,0").split(",").map(Number),
    nodes,
    a,
    widgets: [...all(".ec-minimap"), ...all(".ec-minimap-toggle"), ...all(".ec-zoom"), ...all(".ec-notch:not(.ec-hidden)")],
    panels: all(".ec-panel").filter((r) => r.w > 0 && r.h > 0),
    notice: !!document.querySelector('[data-testid="ec-notice"]'),
    ghosts: document.querySelectorAll(".ec-ghost-panel").length,
    raf: window.__raf,
  };
};`;

const hit = (a, b, tol = TOL) =>
  a.x < b.x + b.w - tol && b.x < a.x + a.w - tol && a.y < b.y + b.h - tol && b.y < a.y + a.h - tol;

async function newPage(vp, { reduced = false, dark = false } = {}) {
  const ctx = await browser.newContext({
    viewport: { width: vp.w, height: vp.h },
    deviceScaleFactor: 1,
    reducedMotion: reduced ? "reduce" : "no-preference",
  });
  await ctx.addInitScript(CLOCK);
  await ctx.addInitScript(
    ([s, d]) => {
      localStorage.setItem("isa.solutions", JSON.stringify([s]));
      localStorage.setItem("isa-theme", d ? "dark" : "light");
    },
    [SOLUTION, dark],
  );
  const page = await ctx.newPage();
  const errors = [];
  page.on("console", (m) => m.type() === "error" && errors.push(m.text()));
  page.on("pageerror", (e) => errors.push(String(e)));
  // l'indirizzo di ogni richiesta fallita (la riga della console dice solo «Failed to load resource»)
  page.on("requestfailed", (r) =>
    errors.push(
      `richiesta fallita: ${r.method()} ${r.url()} (${r.failure()?.errorText}) da ${r.frame()?.url()}`,
    ),
  );
  page.on("console", (m) => {
    if (m.type() === "error" && /Failed to load resource/.test(m.text()))
      errors.push(`posizione: ${m.location().url}`);
  });
  await page.goto(`${server.base}/solutions/${SOLUTION.id}/etl?seed=prototype`);
  await page.waitForSelector('[data-node-id="ds1"]', { timeout: 90000 });
  await page.evaluate(SNAP);
  await page.evaluate(() => document.fonts.ready);
  return { ctx, page, errors };
}

/** Un frame: l'orologio avanza di 16 ms, il ciclo condiviso gira, si campiona. */
const stepSnap = (page, ids) =>
  page.evaluate(
    async ([ids, ms]) => {
      window.__clock.t += ms;
      await new Promise((r) => requestAnimationFrame(() => r()));
      return window.__snap(ids);
    },
    [ids, STEP_MS],
  );
const snap = (page, ids) => page.evaluate((ids) => window.__snap(ids), ids);
const settleFrames = async (page, n = 40) => {
  for (let i = 0; i < n; i++) await stepSnap(page, []);
};
const dispatch = (page, cmd) => page.evaluate((c) => window.__etlStore.dispatch(c), cmd);
const view = (page) => page.evaluate(() => ({ ...window.__etlStore.getState().view }));

/** Chiude i pannelli e porta la scena in uno stato noto: nodi in una griglia che riempie l'area sicura, vista (0, 0, 1). */
async function buildDense(page) {
  await dispatch(page, { type: "setPanel", payload: { panel: "tools", open: false } });
  await dispatch(page, { type: "setPanel", payload: { panel: "insp", open: false } });
  await settleFrames(page, 25);
  await page.evaluate(() => {
    const s = window.__etlStore;
    s.dispatch({ type: "setView", payload: { x: 0, y: 0, zoom: 1 } });
    const st = s.getState();
    const snap = window.__snap([]);
    const [t, r, b, l] = snap.insets;
    const x0 = Math.max(24, l);
    const x1 = snap.canvas.w - Math.max(24, r) - 88;
    const y0 = Math.max(24, t);
    const y1 = snap.canvas.h - Math.max(24, b) - 110;
    const cols = 8;
    const rows = 4;
    const base = st.graph.cards["ds1"];
    const cards = {};
    for (let i = 0; i < cols * rows; i++) {
      const id = "d" + i;
      cards[id] = {
        ...base,
        id,
        name: "N" + (i + 1),
        x: Math.round(x0 + ((i % cols) * (x1 - x0)) / (cols - 1)),
        y: Math.round(y0 + (Math.floor(i / cols) * (y1 - y0)) / (rows - 1)),
      };
    }
    s.replaceState({ ...st, graph: { ...st.graph, cards, links: [] }, selection: [] });
  });
  await settleFrames(page, 3);
}

/** I nodi interamente dentro l'area sicura (margine 24 o ingombro dei widget, il maggiore) a riposo. */
async function requiredIds(page) {
  const s = await page.evaluate(() => {
    const ids = Object.keys(window.__etlStore.getState().graph.cards);
    return window.__snap(ids);
  });
  const [t, r, b, l] = s.insets;
  const safe = {
    x1: s.canvas.x + Math.max(MARGIN, l),
    y1: s.canvas.y + Math.max(MARGIN, t),
    x2: s.canvas.x + s.canvas.w - Math.max(MARGIN, r),
    y2: s.canvas.y + s.canvas.h - Math.max(MARGIN, b),
  };
  return Object.entries(s.nodes)
    .filter(
      ([, n]) =>
        n.x >= safe.x1 - 1e-6 &&
        n.y >= safe.y1 - 1e-6 &&
        n.x + n.w <= safe.x2 + 1e-6 &&
        n.y + n.h <= safe.y2 + 1e-6,
    )
    .map(([id]) => id);
}

let failed = 0;
const results = [];
const measures = [];
function check(name, ok, extra = "") {
  results.push({
    prova: name,
    esito: ok ? "ok" : "FALLITA",
    dettaglio: String(extra).slice(0, 200),
  });
  if (!ok) failed++;
}

/** R1 a riposo. Restituisce l'elenco dei problemi (vuoto se va bene). */
function restProblems(s, ids) {
  const [t, r, b, l] = s.insets;
  const safe = {
    x1: s.canvas.x + Math.max(MARGIN, l),
    y1: s.canvas.y + Math.max(MARGIN, t),
    x2: s.canvas.x + s.canvas.w - Math.max(MARGIN, r),
    y2: s.canvas.y + s.canvas.h - Math.max(MARGIN, b),
  };
  const out = [];
  for (const id of ids) {
    const n = s.nodes[id];
    if (!n) continue;
    if (
      n.x < safe.x1 - TOL ||
      n.y < safe.y1 - TOL ||
      n.x + n.w > safe.x2 + TOL ||
      n.y + n.h > safe.y2 + TOL
    )
      out.push(`${id} fuori dall'area sicura`);
    if (s.widgets.some((w) => hit(n, w))) out.push(`${id} tocca un widget`);
    if (s.panels.some((p) => hit(n, p))) out.push(`${id} tocca un pannello`);
  }
  return out;
}

/**
 * Esegue `act`, poi fa girare i frame fino a transizione finita campionando ciascuno.
 * Restituisce prima/dopo e tutti i frame; verifica R1–R4.
 */
async function transition(
  page,
  label,
  ids,
  act,
  { win, allowNotice = false, still: expectStill = false } = {},
) {
  const before = await snap(page, ids);
  const T0 = before.t;
  await act();
  const frames = [];
  let still = 0;
  let prevKey = null;
  for (let i = 0; i < 80; i++) {
    const f = await stepSnap(page, ids);
    frames.push(f);
    const key = JSON.stringify([f.view, f.canvas, f.a, f.ghosts]);
    still = key === prevKey ? still + 1 : 0;
    prevKey = key;
    if (still >= 3 && i > 3) break;
  }
  const after = frames[frames.length - 1];
  const seq = [before, ...frames];
  // frame in cui qualcosa cambia (vista o canvas)
  const sig = (f) => JSON.stringify([f.view, f.canvas, f.a]);
  const changed = seq.map((f, i) => (i === 0 ? false : sig(f) !== sig(seq[i - 1])));
  const first = changed.indexOf(true);
  const last = changed.lastIndexOf(true);
  const active = first < 0 ? 0 : last - first + 1;

  // R1
  const rest = restProblems(after, ids);
  const noticeOn = after.notice;
  check(
    `${win} ${label}: R1 a riposo i ${ids.length} nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget`,
    rest.length === 0 || (allowNotice && noticeOn),
    rest.slice(0, 3).join("; ") + (noticeOn ? " [avviso attivo]" : ""),
  );
  // R2: a ogni frame dentro il canvas di quel frame, con il margine
  let r2 = 0;
  let r2Worst = 0;
  if (!(allowNotice && noticeOn)) {
    for (const f of frames) {
      for (const id of ids) {
        const n = f.nodes[id];
        if (!n) continue;
        const d = Math.max(
          f.canvas.x + MARGIN - n.x,
          f.canvas.y + MARGIN - n.y,
          n.x + n.w - (f.canvas.x + f.canvas.w - MARGIN),
          n.y + n.h - (f.canvas.y + f.canvas.h - MARGIN),
        );
        r2Worst = Math.max(r2Worst, d);
        if (d > TOL) r2++;
      }
    }
  }
  check(
    `${win} ${label}: R2 a ogni frame (${frames.length}) i nodi richiesti sono dentro il canvas di quel frame`,
    r2 === 0,
    `violazioni ${r2}, scarto massimo ${r2Worst.toFixed(2)} px`,
  );
  // R3: zoom monotono, nessun overshoot
  const zs = seq.map((f) => f.view.zoom);
  const dir = Math.sign(zs[zs.length - 1] - zs[0]);
  let mono = true;
  for (let i = 1; i < zs.length; i++) if ((zs[i] - zs[i - 1]) * (dir || 1) < -1e-9) mono = false;
  const lo = Math.min(zs[0], zs[zs.length - 1]) - 1e-9;
  const hi = Math.max(zs[0], zs[zs.length - 1]) + 1e-9;
  const bounded = zs.every((z) => z >= lo && z <= hi);
  check(
    `${win} ${label}: R3 ${expectStill ? "la vista resta ferma" : "zoom monotono"} (${zs[0].toFixed(3)} → ${zs[zs.length - 1].toFixed(3)}) e transizione di ${active} frame`,
    expectStill
      ? zs.every((z) => z === zs[0]) &&
          seq.every((f) => f.view.x === seq[0].view.x && f.view.y === seq[0].view.y)
      : mono && bounded && active >= 10,
    `monotono=${mono} limitato=${bounded} frame attivi=${active}`,
  );
  // R4: nessun salto
  let worst = 0;
  let meanAt = 0;
  let maxStep = 0;
  if (first >= 0) {
    for (const id of ids) {
      const pts = seq.slice(Math.max(0, first - 1), last + 1).map((f) => f.nodes[id]);
      if (pts.some((p) => !p)) continue;
      const steps = pts.slice(1).map((p, i) => Math.hypot(p.x - pts[i].x, p.y - pts[i].y));
      const total = steps.reduce((a, b) => a + b, 0);
      if (total < 4) continue;
      const mean = total / steps.length;
      const ratio = Math.max(...steps) / mean;
      if (ratio > worst) {
        worst = ratio;
        meanAt = mean;
        maxStep = Math.max(...steps);
      }
    }
  }
  check(
    `${win} ${label}: R4 nessun salto (rapporto massimo ${worst.toFixed(2)} ≤ 2,5)`,
    worst <= 2.5,
    `passo massimo ${maxStep.toFixed(2)} px, media ${meanAt.toFixed(2)} px`,
  );
  measures.push({
    finestra: win,
    prova: label,
    frameCampionati: frames.length,
    frameAttivi: active,
    zoomIniziale: zs[0],
    zoomFinale: zs[zs.length - 1],
    zoomMin: Math.min(...zs),
    zoomMax: Math.max(...zs),
    passoMassimoPx: maxStep,
    rapportoPassoMedia: worst,
    scartoR2Px: r2Worst,
    avviso: noticeOn,
    nodiRichiesti: ids.length,
    durataMs: frames.length ? frames[frames.length - 1].t - T0 : 0,
  });
  return { before, after, frames, seq };
}

/** Zoom a cui i nodi richiesti entrerebbero nell'area finale (senza il limite dello zoom minimo). */
function neededZoom(before, after, ids) {
  const xs = ids.map((id) => before.nodes[id]);
  const bw = Math.max(...xs.map((n) => n.x + n.w)) - Math.min(...xs.map((n) => n.x));
  const bh = Math.max(...xs.map((n) => n.y + n.h)) - Math.min(...xs.map((n) => n.y));
  const [t, r, b, l] = after.insets;
  const sw = after.canvas.w - Math.max(MARGIN, l) - Math.max(MARGIN, r);
  const sh = after.canvas.h - Math.max(MARGIN, t) - Math.max(MARGIN, b);
  return Math.min((sw / bw) * before.view.zoom, (sh / bh) * before.view.zoom);
}

/** Schermata nei due temi. */
async function shot(page, name) {
  const path = (s) => resolve(OUT, `${name}-${s}.png`);
  await page.screenshot({ path: path("chiaro"), animations: "disabled" });
  await page.evaluate(() => document.documentElement.classList.add("dark"));
  await page.waitForTimeout(150);
  await page.screenshot({ path: path("scuro"), animations: "disabled" });
  await page.evaluate(() => document.documentElement.classList.remove("dark"));
  await page.waitForTimeout(100);
}

/** Una striscia di sei fotogrammi (0, 20, 40, 60, 80, 100 % della transizione), con il riquadro del canvas evidenziato. */
async function strip(page, ids, act, file) {
  const T0 = (await snap(page, [])).t;
  await act();
  const shots = [];
  let elapsed = 0;
  for (const p of [0, 0.2, 0.4, 0.6, 0.8, 1]) {
    const target = Math.round(p * DURATION_MS);
    while (elapsed < target || (p === 1 && elapsed < DURATION_MS + STEP_MS)) {
      await stepSnap(page, []);
      elapsed += STEP_MS;
    }
    await page.evaluate(() => {
      const c = document.querySelector(".ec-center");
      c.style.outline = "4px solid #ff2d95";
      c.style.outlineOffset = "-4px";
    });
    shots.push((await page.screenshot({ animations: "disabled" })).toString("base64"));
    await page.evaluate(() => (document.querySelector(".ec-center").style.outline = ""));
  }
  void T0;
  const sheet = await page.context().newPage();
  const labels = ["0 %", "20 %", "40 %", "60 %", "80 %", "100 %"];
  await sheet.setViewportSize({ width: 3180, height: 440 });
  await sheet.setContent(
    `<body style="margin:0;background:#222;display:flex;gap:6px;padding:6px;font:600 20px sans-serif;color:#fff">` +
      shots
        .map(
          (b, i) =>
            `<div><div style="padding:2px 0">${labels[i]}</div><img style="width:520px;display:block" src="data:image/png;base64,${b}"></div>`,
        )
        .join("") +
      `</body>`,
  );
  await sheet.screenshot({ path: resolve(OUT, file), fullPage: true });
  await sheet.close();
}

// ---------------------------------------------------------------------------------------------

if (MODE === "prima") {
  // le schermate di prima: il commit precedente (nodi tagliati dal bordo del canvas, un widget ne copre uno)
  for (const win of WINDOWS) {
    const { ctx, page } = await newPage(win);
    await buildDense(page);
    await dispatch(page, {
      type: "setPanel",
      payload: { panel: "insp", side: "bottom", open: true },
    });
    await page.waitForTimeout(900);
    await shot(page, `inspector-in-basso-${win.name}-prima`);
    await ctx.close();
  }
  await browser.close();
  server.stop();
  console.log("schermate «prima» salvate in", OUT);
  process.exit(0);
}

const allErrors = [];
for (const win of WINDOWS) {
  const W = win.name;
  const { ctx, page, errors } = await newPage(win);
  await page.evaluate(() => (window.__clock.auto = false));
  await buildDense(page);
  const ids = await requiredIds(page);
  check(
    `${W}: la scena densa ha 32 nodi e tutti sono richiesti a pannelli chiusi`,
    ids.length === 32,
    `${ids.length} richiesti`,
  );
  const v0 = await view(page);
  const reset = async () => {
    await dispatch(page, { type: "setPanel", payload: { panel: "tools", open: false } });
    await dispatch(page, { type: "setPanel", payload: { panel: "insp", open: false } });
    await settleFrames(page, 25);
    await dispatch(page, {
      type: "setPanel",
      payload: { panel: "tools", side: "left", open: false },
    });
    await dispatch(page, {
      type: "setPanel",
      payload: { panel: "insp", side: "right", open: false },
    });
    await dispatch(page, { type: "setView", payload: v0 });
    await settleFrames(page, 3);
  };
  check(
    `${W}: nessuna transizione CSS di larghezza o altezza sui pannelli (un solo orologio)`,
    await page.evaluate(() =>
      [...document.querySelectorAll(".ec-panel")].every((p) => {
        const props = getComputedStyle(p).transitionProperty;
        return !/width|height|all/.test(props) || getComputedStyle(p).transitionDuration === "0s";
      }),
    ),
  );
  const raf0 = (await snap(page, [])).raf;

  // 1. apri e chiudi cassetta e Inspector sui quattro bordi
  for (const panel of ["tools", "insp"]) {
    for (const side of ["left", "right", "top", "bottom"]) {
      const name = `${panel === "tools" ? "cassetta" : "Inspector"} ${side}`;
      const open = await transition(
        page,
        `apri ${name}`,
        ids,
        () => dispatch(page, { type: "setPanel", payload: { panel, side, open: true } }),
        { win: W, allowNotice: true },
      );
      if (panel === "insp" && side === "bottom") {
        const z = neededZoom(open.before, open.after, ids);
        measures.push({
          finestra: W,
          prova: "zoom necessario per i nodi richiesti, Inspector in basso",
          zoomNecessario: z,
          zoomFinale: open.after.view.zoom,
          avviso: open.after.notice,
        });
      }
      if (panel === "tools" && side === "left" && !MODE.startsWith("noshots")) {
        await shot(page, `cassetta-a-sinistra-scena-densa-${W}`);
      }
      if (panel === "insp" && side === "bottom") await shot(page, `inspector-in-basso-${W}-dopo`);
      const close = await transition(
        page,
        `chiudi ${name}`,
        ids,
        () => dispatch(page, { type: "setPanel", payload: { panel, open: false } }),
        { win: W, allowNotice: true },
      );
      const vf = close.after.view;
      check(
        `${W} ${name}: R6 apri → chiudi, la vista finale è quella iniziale`,
        Math.abs(vf.x - v0.x) <= TOL &&
          Math.abs(vf.y - v0.y) <= TOL &&
          Math.abs(vf.zoom - v0.zoom) <= 1e-3,
        `iniziale ${JSON.stringify(v0)} finale ${JSON.stringify({ x: +vf.x.toFixed(3), y: +vf.y.toFixed(3), zoom: +vf.zoom.toFixed(4) })}`,
      );
      await reset();
    }
  }

  // 2. cambio scheda (due pannelli sullo stesso bordo) e passaggio da un pannello all'altro su bordi diversi
  await dispatch(page, {
    type: "setPanel",
    payload: { panel: "tools", side: "bottom", open: true },
  });
  await dispatch(page, {
    type: "setPanel",
    payload: { panel: "insp", side: "bottom", open: false },
  });
  await settleFrames(page, 25);
  const tabs = await transition(
    page,
    "cambio scheda (cassetta → Inspector, stesso bordo)",
    ids,
    () => page.locator('.ec-dock-tab[title="Inspector"][aria-pressed="false"]').click(),
    { win: W, allowNotice: true, still: true },
  );
  check(
    `${W}: il cambio scheda non sposta il canvas (stessa area, vista invariata)`,
    Math.abs(tabs.after.canvas.h - tabs.before.canvas.h) <= TOL &&
      tabs.after.view.zoom === tabs.before.view.zoom,
  );
  await transition(
    page,
    "cambio scheda (Inspector → cassetta)",
    ids,
    () => page.locator('.ec-dock-tab[title="Strumenti"][aria-pressed="false"]').click(),
    { win: W, allowNotice: true, still: true },
  );
  await reset();
  await dispatch(page, { type: "setPanel", payload: { panel: "tools", side: "left", open: true } });
  await settleFrames(page, 25);
  await transition(
    page,
    "dalla cassetta a sinistra all'Inspector a destra",
    ids,
    () =>
      dispatch(page, { type: "setPanel", payload: { panel: "insp", side: "right", open: true } }),
    { win: W, allowNotice: true },
  );
  await reset();

  // 3. spostamento su un altro bordo
  await dispatch(page, { type: "setPanel", payload: { panel: "insp", side: "right", open: true } });
  await settleFrames(page, 25);
  for (const side of ["bottom", "top", "left"]) {
    const m = await transition(
      page,
      `sposta l'Inspector su ${side}`,
      ids,
      () => dispatch(page, { type: "setPanel", payload: { panel: "insp", side } }),
      { win: W, allowNotice: true },
    );
    check(`${W}: dopo lo spostamento su ${side} non resta nessun guscio`, m.after.ghosts === 0);
  }
  await reset();

  // 4. ridimensionamento della finestra con l'Inspector in basso
  await dispatch(page, {
    type: "setPanel",
    payload: { panel: "insp", side: "bottom", open: true },
  });
  await settleFrames(page, 25);
  const atOpen = await snap(page, ids);
  const vOpen = atOpen.view;
  let problems = 0;
  let worstResize = "";
  const sizes = [];
  for (let i = 1; i <= 12; i++) sizes.push({ width: win.w - i * 20, height: win.h - i * 6 });
  for (const s of sizes) {
    await page.setViewportSize(s);
    await page.evaluate(
      () => new Promise((r) => requestAnimationFrame(() => requestAnimationFrame(() => r()))),
    );
    const sn = await snap(page, ids);
    const p = restProblems(sn, ids);
    if (p.length && !sn.notice) {
      problems++;
      worstResize = `${s.width}×${s.height}: ${p[0]}`;
    }
  }
  check(
    `${W} ridimensionamento: a ogni passo (${sizes.length}) i nodi richiesti restano dentro l'area sicura`,
    problems === 0,
    worstResize,
  );
  await page.setViewportSize({ width: win.w, height: win.h });
  await page.evaluate(
    () => new Promise((r) => requestAnimationFrame(() => requestAnimationFrame(() => r()))),
  );
  const back = await snap(page, ids);
  check(
    `${W} ridimensionamento: tornando alla dimensione di prima la vista è quella di prima`,
    Math.abs(back.view.x - vOpen.x) <= TOL &&
      Math.abs(back.view.y - vOpen.y) <= TOL &&
      Math.abs(back.view.zoom - vOpen.zoom) <= 1e-3,
    `${JSON.stringify(vOpen)} → ${JSON.stringify(back.view)}`,
  );
  await reset();

  // 5. una rotella a metà transizione annulla l'animazione e la vista resta continua
  {
    await dispatch(page, {
      type: "setPanel",
      payload: { panel: "insp", side: "bottom", open: true },
    });
    for (let i = 0; i < 7; i++) await stepSnap(page, ids);
    const mid = await snap(page, ids);
    const c = mid.canvas;
    await page.mouse.move(c.x + c.w / 2, c.y + c.h / 2);
    await page.mouse.wheel(0, 40);
    await page.waitForTimeout(50);
    const afterWheel = await snap(page, ids);
    check(
      `${W} rotella a metà transizione: la vista è continua (solo lo scorrimento della rotella, 40 px)`,
      Math.abs(afterWheel.view.x - mid.view.x) <= TOL &&
        Math.abs(afterWheel.view.y - (mid.view.y - 40)) <= TOL &&
        afterWheel.view.zoom === mid.view.zoom,
      `${JSON.stringify(mid.view)} → ${JSON.stringify(afterWheel.view)}`,
    );
    const post = [];
    for (let i = 0; i < 30; i++) post.push(await stepSnap(page, ids));
    check(
      `${W} rotella a metà transizione: l'animazione della vista è annullata (resta dov'è) e il pannello finisce`,
      post.every(
        (f) =>
          f.view.zoom === afterWheel.view.zoom &&
          f.view.x === afterWheel.view.x &&
          f.view.y === afterWheel.view.y,
      ) && post[post.length - 1].a.insp === 1,
    );
    await reset();
  }

  // 6. due cambi ravvicinati: nessun salto
  {
    await dispatch(page, {
      type: "setPanel",
      payload: { panel: "insp", side: "bottom", open: true },
    });
    const seq = [await snap(page, ids)];
    for (let i = 0; i < 6; i++) seq.push(await stepSnap(page, ids));
    await dispatch(page, { type: "setPanel", payload: { panel: "insp", open: false } });
    seq.push(await snap(page, ids));
    for (let i = 0; i < 60; i++) seq.push(await stepSnap(page, ids));
    let worst = 0;
    let jump = 0;
    // l'inversione è un nuovo cambio: il rapporto passo massimo / media si calcola su ciascuna delle due animazioni (la seconda è più corta della prima)
    const INVERT = 7; // indice del campione preso subito dopo la chiusura
    for (const id of ids) {
      const pts = seq.map((f) => f.nodes[id]);
      const steps = pts.slice(1).map((p, i) => Math.hypot(p.x - pts[i].x, p.y - pts[i].y));
      jump = Math.max(jump, ...steps);
      for (const seg of [steps.slice(0, INVERT - 1), steps.slice(INVERT - 1)]) {
        const active = seg.filter((s) => s > 1e-6);
        const total = active.reduce((a, b) => a + b, 0);
        if (total < 4) continue;
        worst = Math.max(worst, Math.max(...seg) / (total / active.length));
      }
    }
    check(
      `${W} due cambi ravvicinati (apri e subito chiudi): nessun salto (rapporto ${worst.toFixed(2)} ≤ 2,5)`,
      worst <= 2.5,
      `passo massimo ${jump.toFixed(2)} px`,
    );
    const last = seq[seq.length - 1];
    check(
      `${W} due cambi ravvicinati: alla fine la vista è tornata a quella iniziale`,
      Math.abs(last.view.x - v0.x) <= TOL &&
        Math.abs(last.view.y - v0.y) <= TOL &&
        Math.abs(last.view.zoom - v0.zoom) <= 1e-3,
      JSON.stringify(last.view),
    );
    await reset();
  }

  // 7. un solo requestAnimationFrame per frame: i frame di animazione non ne creano altri oltre al ciclo condiviso
  {
    await dispatch(page, {
      type: "setPanel",
      payload: { panel: "insp", side: "bottom", open: true },
    });
    const r0 = (await snap(page, [])).raf;
    let steps = 0;
    for (let i = 0; i < 30; i++) {
      await stepSnap(page, []);
      steps++;
    }
    const r1 = (await snap(page, [])).raf;
    // ogni passo chiede un rAF suo (lo scrive questo script) e il ciclo ne chiede al più uno: al più 2 per passo, e nessuno a ciclo fermo
    check(
      `${W}: i frame dell'animazione usano solo il ciclo condiviso (rAF per passo ≤ 2: ${((r1 - r0) / steps).toFixed(2)})`,
      (r1 - r0) / steps <= 2.01,
    );
    await reset();
    void raf0;
  }
  allErrors.push(...errors);
  await ctx.close();
}

// 8. movimento ridotto: stato finale immediato (R5)
for (const win of WINDOWS) {
  const W = win.name;
  const { ctx, page, errors } = await newPage(win, { reduced: true });
  await page.evaluate(() => (window.__clock.auto = false));
  await buildDense(page);
  const ids = await requiredIds(page);
  await dispatch(page, {
    type: "setPanel",
    payload: { panel: "insp", side: "bottom", open: true },
  });
  await page.evaluate(() => new Promise((r) => setTimeout(r, 30)));
  const s = await snap(page, ids);
  check(`${W} R5 movimento ridotto: subito lo stato finale (Inspector aperto)`, s.a.insp === 1);
  const prob = restProblems(s, ids);
  check(
    `${W} R5 movimento ridotto: R1 vale subito`,
    prob.length === 0 || s.notice,
    prob.slice(0, 2).join("; "),
  );
  const frames = [];
  for (let i = 0; i < 5; i++) frames.push(await stepSnap(page, ids));
  check(
    `${W} R5 movimento ridotto: nessun frame di animazione (vista e canvas fermi)`,
    frames.every((f) => JSON.stringify([f.view, f.canvas]) === JSON.stringify([s.view, s.canvas])),
  );
  await dispatch(page, { type: "setPanel", payload: { panel: "insp", open: false } });
  await page.evaluate(() => new Promise((r) => setTimeout(r, 30)));
  const c = await snap(page, ids);
  const v0 = { x: 0, y: 0, zoom: 1 };
  check(
    `${W} R5 movimento ridotto: apri → chiudi torna alla vista iniziale (R6)`,
    Math.abs(c.view.x - v0.x) <= TOL &&
      Math.abs(c.view.y - v0.y) <= TOL &&
      Math.abs(c.view.zoom - v0.zoom) <= 1e-3,
    JSON.stringify(c.view),
  );
  allErrors.push(...errors);
  await ctx.close();
}

// 9. le strisce di sei fotogrammi: apertura e chiusura dell'Inspector in basso
if (!process.env.NO_STRIPS) {
  const win = WINDOWS[0];
  const { ctx, page } = await newPage(win);
  await page.evaluate(() => (window.__clock.auto = false));
  await buildDense(page);
  const ids = await requiredIds(page);
  await strip(
    page,
    ids,
    () =>
      dispatch(page, { type: "setPanel", payload: { panel: "insp", side: "bottom", open: true } }),
    "striscia-apertura-inspector-in-basso.png",
  );
  await settleFrames(page, 30);
  await strip(
    page,
    ids,
    () => dispatch(page, { type: "setPanel", payload: { panel: "insp", open: false } }),
    "striscia-chiusura-inspector-in-basso.png",
  );
  await ctx.close();
}

check("nessun errore in console", allErrors.length === 0, allErrors.slice(0, 2).join(" | "));

writeFileSync(
  resolve(OUT, "misure.json"),
  JSON.stringify({ risultati: results, misure: measures }, null, 2),
);
console.table(results.map((r) => ({ prova: r.prova.slice(0, 120), esito: r.esito })));
const frames = measures.filter((m) => m.frameCampionati).reduce((n, m) => n + m.frameCampionati, 0);
const z = measures.filter((m) => m.zoomMin !== undefined);
console.log(
  `\nprove: ${results.length}, fallite: ${failed}; frame campionati: ${frames}; zoom min ${Math.min(...z.map((m) => m.zoomMin)).toFixed(3)}, max ${Math.max(...z.map((m) => m.zoomMax)).toFixed(3)}; passo massimo ${Math.max(...z.map((m) => m.passoMassimoPx ?? 0)).toFixed(2)} px`,
);
for (const m of measures.filter((m) => m.zoomNecessario !== undefined))
  console.log(
    `zoom necessario ${m.finestra}, Inspector in basso: ${m.zoomNecessario.toFixed(3)} (finale ${m.zoomFinale.toFixed(3)}, avviso ${m.avviso})`,
  );
await browser.close();
server.stop();
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

