# 08-scripts-config-b.md

File in questo blocco:

- `scripts/visual-fase4.mjs`
- `scripts/visual-fase4b.mjs`
- `scripts/visual-lib.mjs`
- `tsconfig.json`
- `vite.config.ts`
- `vitest.config.ts`

---

### `scripts/visual-fase4.mjs`

374 righe

```js
#!/usr/bin/env node
/**
 * Verifica visiva della Fase 4a: prototipo e nuovo canvas alla stessa
 * finestra (1440 × 900). Salva in docs/visual/fase4/:
 *   prototipo.png, v2-chiaro.png, v2-scuro.png,
 *   crop-<tipo>-{prototipo,chiaro,scuro}.png  (un nodo per tipo),
 *   misure.json  (posizioni, colori e misure lette dal DOM, per il report).
 *
 * Avvia da solo `vite dev` se non gli si passa BASE_URL.
 * Uso: node scripts/visual-fase4.mjs
 */
import { spawn } from "node:child_process";
import { mkdirSync, readFileSync, writeFileSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath, pathToFileURL } from "node:url";
import { chromium } from "playwright";

const ROOT = resolve(dirname(fileURLToPath(import.meta.url)), "..");
const OUT = resolve(ROOT, "docs/visual/fase4");
const PROTOTYPE = resolve(ROOT, "docs/prototype/isa-fusion-prototype.html");
const VIEWPORT = { width: 1440, height: 900 };
const PORT = Number(process.env.PORT ?? 5199);
const SOLUTION = {
  id: "visual",
  name: "Verifica visiva",
  description: "",
  status: "draft",
  version: "v1",
  chart: "bar",
  series: [],
  updatedAt: "",
  owner: "",
  parameters: [],
  modules: { etl: "draft" },
  shares: [],
};

mkdirSync(OUT, { recursive: true });

// --- server di sviluppo -----------------------------------------------------
async function startServer() {
  if (process.env.BASE_URL) return { base: process.env.BASE_URL, stop() {} };
  const child = spawn(
    "npx",
    ["vite", "dev", "--port", String(PORT), "--strictPort", "--host", "127.0.0.1"],
    {
      cwd: ROOT,
      stdio: ["ignore", "pipe", "pipe"],
    },
  );
  let log = "";
  child.stdout.on("data", (d) => (log += d));
  child.stderr.on("data", (d) => (log += d));
  const base = `http://127.0.0.1:${PORT}`;
  const t0 = Date.now();
  for (;;) {
    if (Date.now() - t0 > 90000) {
      child.kill();
      throw new Error("vite dev non risponde:\n" + log);
    }
    try {
      const r = await fetch(base + "/");
      if (r.status < 500) break;
    } catch {
      /* non ancora pronto */
    }
    await new Promise((r) => setTimeout(r, 500));
  }
  // npx avvia vite come processo figlio: si chiude l'intero gruppo
  return {
    base,
    stop: () => {
      try {
        process.kill(-child.pid);
      } catch {
        /* già terminato */
      }
    },
  };
}

// --- carattere del prototipo (nessuna rete: Manrope locale al posto di Google Fonts) ---
function manropeCss() {
  const dir = resolve(ROOT, "node_modules/@fontsource-variable/manrope/files");
  const face = (name, range) => {
    const b64 = readFileSync(resolve(dir, name)).toString("base64");
    return (
      `@font-face{font-family:'Manrope';font-style:normal;font-weight:200 800;font-display:block;` +
      `src:url(data:font/woff2;base64,${b64}) format('woff2');unicode-range:${range};}`
    );
  };
  return (
    face(
      "manrope-latin-ext-wght-normal.woff2",
      "U+0100-02BA,U+02BD-02C5,U+02C7-02CC,U+02CE-02D7,U+02DD-02FF,U+0304,U+0308,U+0329,U+1D00-1DBF,U+1E00-1E9F,U+1EF2-1EFF,U+2020,U+20A0-20AB,U+20AD-20C0,U+2113,U+2C60-2C7F,U+A720-A7FF",
    ) +
    face(
      "manrope-latin-wght-normal.woff2",
      "U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+0304,U+0308,U+0329,U+2000-206F,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD",
    )
  );
}

// --- misure lette dal DOM ------------------------------------------------------
const SELECTORS = {
  prototipo: {
    stage: "#stage",
    node: (id) => `[data-uid="${id}"]`,
    wrap: ".icon-wrap",
    label: ".label",
    dot: ".state-dot",
    zoom: "#zoomCtl",
    minimap: "#minimap",
    fit: "#zoomFit",
    zoomBtn: "#zoomPct",
  },
  v2: {
    stage: ".ec-stage",
    node: (id) => `[data-node-id="${id}"]`,
    wrap: ".ec-icon-wrap",
    label: ".ec-label",
    dot: ".ec-state-dot",
    zoom: ".ec-zoom",
    minimap: ".ec-minimap",
    fit: ".ec-fit",
    zoomBtn: ".ec-zoom button:nth-child(2)",
  },
};
const NODE_IDS = ["ds1", "op-filter", "op-join", "op-sort", "op-export"];

async function measure(page, kind) {
  const sel = { ...SELECTORS[kind], node: undefined };
  const nodeSels = Object.fromEntries(NODE_IDS.map((id) => [id, SELECTORS[kind].node(id)]));
  return page.evaluate(
    ({ sel, nodeSels }) => {
      const stage = document.querySelector(sel.stage);
      const sr = stage.getBoundingClientRect();
      const rel = (el) => {
        if (!el) return null;
        const r = el.getBoundingClientRect();
        return {
          x: +(r.left - sr.left).toFixed(2),
          y: +(r.top - sr.top).toFixed(2),
          w: +r.width.toFixed(2),
          h: +r.height.toFixed(2),
        };
      };
      const cs = (el, props) => {
        if (!el) return null;
        const s = getComputedStyle(el);
        return Object.fromEntries(props.map((p) => [p, s[p]]));
      };
      const out = {
        stage: {
          rect: { w: sr.width, h: sr.height },
          style: cs(stage, ["backgroundColor", "borderRadius"]),
        },
        nodes: {},
      };
      for (const [id, q] of Object.entries(nodeSels)) {
        const n = document.querySelector(q);
        if (!n) continue;
        const wrap = n.querySelector(sel.wrap);
        const label = n.querySelector(sel.label);
        const dot = n.querySelector(sel.dot);
        out.nodes[id] = {
          rect: rel(n),
          wrap: {
            rect: rel(wrap),
            style: cs(wrap, ["backgroundColor", "borderRadius", "color", "opacity", "boxShadow"]),
          },
          label: {
            rect: rel(label),
            style: cs(label, ["fontFamily", "fontSize", "fontWeight", "color", "lineHeight"]),
          },
          icon: cs(wrap.querySelector("svg"), ["width", "height"]),
          dot:
            dot && getComputedStyle(dot).display !== "none"
              ? {
                  rect: rel(dot),
                  style: cs(dot, ["backgroundColor", "borderTopWidth", "borderTopColor"]),
                }
              : null,
        };
      }
      const zoom = document.querySelector(sel.zoom);
      const mm = document.querySelector(sel.minimap);
      const box = [
        "backgroundColor",
        "borderRadius",
        "borderTopWidth",
        "borderTopColor",
        "boxShadow",
        "backdropFilter",
      ];
      out.zoom = {
        rect: rel(zoom),
        fromRight: +(sr.right - zoom.getBoundingClientRect().right).toFixed(2),
        fromBottom: +(sr.bottom - zoom.getBoundingClientRect().bottom).toFixed(2),
        style: cs(zoom, box),
        fit: cs(document.querySelector(sel.fit), [
          "color",
          "fontSize",
          "fontWeight",
          "height",
          "minWidth",
        ]),
        text: document.querySelector(sel.zoomBtn).textContent,
      };
      out.minimap = {
        rect: rel(mm),
        fromLeft: +(mm.getBoundingClientRect().left - sr.left).toFixed(2),
        fromBottom: +(sr.bottom - mm.getBoundingClientRect().bottom).toFixed(2),
        style: cs(mm, box),
      };
      out.body = cs(document.body, ["fontFamily"]);
      return out;
    },
    { sel, nodeSels },
  );
}

// --- scena per i ritagli: output parziale, output pieno, box combinato -----------------
const CROPS = {
  prototipo: {
    dataset: '[data-uid="ds1"]',
    lavorazione: '[data-uid="op-filter"]',
    combinato: ".card.combined",
    "output-parziale": ".card.output.partial",
    "output-pieno": ".card.output:not(.partial)",
  },
  v2: {
    dataset: '[data-node-id="ds1"]',
    lavorazione: '[data-node-id="op-filter"]',
    combinato: ".ec-combined",
    "output-parziale": ".ec-output.ec-partial",
    "output-pieno": ".ec-output:not(.ec-partial)",
  },
};

async function cropScene(page, kind, theme) {
  if (kind === "prototipo") {
    await page.evaluate(() => {
      connect("ds1", "op-join");
      connect("ds1", "op-filter");
      performMerge("op-sort", "op-export");
    });
  } else {
    await page.evaluate(() => {
      const s = window.__etlStore;
      s.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
      s.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
      s.dispatch({ type: "merge", payload: { dragged: "op-sort", target: "op-export" } });
    });
  }
  await page.waitForTimeout(1200);
  const suffix = kind === "prototipo" ? "prototipo" : theme;
  // la scena con i cavi, intera, e le misure dei cavi
  await page.evaluate((k) => {
    const st = document.querySelector(k === "prototipo" ? "#stage" : ".ec-stage");
    st.scrollIntoView({ block: "center" });
  }, kind);
  await page.screenshot({ path: resolve(OUT, `cavi-${suffix}.png`) });
  const cables = await page.evaluate((k) => {
    const paths = [
      ...document.querySelectorAll(k === "prototipo" ? "#linkPaths path" : ".ec-link"),
    ];
    const dots = [
      ...document.querySelectorAll(k === "prototipo" ? "#linkPaths circle" : ".ec-links circle"),
    ];
    const cs = (el, props) => Object.fromEntries(props.map((q) => [q, getComputedStyle(el)[q]]));
    return {
      count: paths.length,
      path: paths[0]
        ? cs(paths[0], ["stroke", "strokeWidth", "strokeLinecap", "strokeLinejoin", "fill"])
        : null,
      dots: dots.length,
      dot: dots[0] ? { ...cs(dots[0], ["fill"]), r: dots[0].getAttribute("r") } : null,
      d: paths.map((el) => el.getAttribute("d")),
    };
  }, kind);
  measures[kind === "prototipo" ? "prototipo" : `v2-${theme}`].cavi = cables;
  for (const [name, q] of Object.entries(CROPS[kind])) {
    const el = page.locator(q).first();
    await el.scrollIntoViewIfNeeded();
    // il ritaglio include lo spazio attorno al nodo, alla scala 3 per vedere i dettagli
    const b = await el.boundingBox();
    if (!b) throw new Error(`ritaglio ${name} (${kind}): nodo non trovato`);
    await page.screenshot({
      path: resolve(OUT, `crop-${name}-${suffix}.png`),
      clip: {
        x: Math.max(0, b.x - 14),
        y: Math.max(0, b.y - 14),
        width: b.width + 28,
        height: b.height + 28,
      },
    });
  }
}

// --- esecuzione ---------------------------------------------------------------------------
const server = await startServer();
const browser = await chromium.launch();
const measures = {};
const issues = [];
try {
  // prototipo
  {
    const ctx = await browser.newContext({ viewport: VIEWPORT, deviceScaleFactor: 1 });
    const page = await ctx.newPage();
    await page.route("https://fonts.googleapis.com/**", (r) =>
      r.fulfill({ contentType: "text/css", body: manropeCss() }),
    );
    await page.goto(pathToFileURL(PROTOTYPE).href);
    await page.evaluate(() => document.fonts.ready);
    // il canvas del prototipo sta sotto le istruzioni: si porta al centro della finestra
    await page.evaluate(() => document.getElementById("stage").scrollIntoView({ block: "center" }));
    await page.waitForTimeout(800);
    await page.screenshot({ path: resolve(OUT, "prototipo.png") });
    measures.prototipo = await measure(page, "prototipo");
    await cropScene(page, "prototipo", "chiaro");
    await ctx.close();
  }
  // nuovo canvas, chiaro e scuro
  for (const theme of ["chiaro", "scuro"]) {
    const ctx = await browser.newContext({ viewport: VIEWPORT, deviceScaleFactor: 1 });
    await ctx.addInitScript(
      ([solution, dark]) => {
        localStorage.setItem("isa.solutions", JSON.stringify([solution]));
        localStorage.setItem("isa-theme", dark ? "dark" : "light");
      },
      [SOLUTION, theme === "scuro"],
    );
    const page = await ctx.newPage();
    page.on("console", (m) => {
      if (m.type() === "error" || m.type() === "warning")
        issues.push(`[${theme}] ${m.type()}: ${m.text()}`);
    });
    page.on("pageerror", (e) => issues.push(`[${theme}] pageerror: ${e.message}`));
    await page.goto(`${server.base}/solutions/${SOLUTION.id}/etl?seed=prototype`);
    await page.waitForSelector('[data-node-id="ds1"]', { timeout: 60000 });
    await page.evaluate(() => document.fonts.ready);
    // In sviluppo (StrictMode) ThemeProvider sovrascrive il tema salvato prima di leggerlo
    // (src/lib/theme.tsx, difetto preesistente): il tema si fissa con la stessa classe `.dark`.
    await page.evaluate(
      (dark) => document.documentElement.classList.toggle("dark", dark),
      theme === "scuro",
    );
    await page.waitForTimeout(800);
    await page.screenshot({ path: resolve(OUT, `v2-${theme}.png`) });
    measures[`v2-${theme}`] = await measure(page, "v2");
    measures[`v2-${theme}`].font = await page.evaluate(() => ({
      loaded: [...document.fonts].filter((f) => f.status === "loaded").map((f) => f.family),
    }));
    await cropScene(page, "v2", theme);
    await ctx.close();
  }
  writeFileSync(resolve(OUT, "misure.json"), JSON.stringify(measures, null, 2) + "\n");
  writeFileSync(
    resolve(OUT, "console.txt"),
    issues.length ? issues.join("\n") + "\n" : "nessun errore né avviso in console\n",
  );
  console.log("schermate in", OUT);
  console.log(
    issues.length
      ? `console: ${issues.length} messaggi (vedi console.txt)`
      : "console: nessun errore né avviso",
  );
} finally {
  await browser.close();
  server.stop();
}
process.exit(0);
```

### `scripts/visual-fase4b.mjs`

318 righe

```js
#!/usr/bin/env node
/**
 * Verifica visiva della Fase 4b (animazioni). Prototipo e nuovo canvas alla
 * stessa finestra (1440 × 900), con un orologio controllato, a tre istanti
 * dall'avvio del flusso: 0, 250 e 500 ms.
 *
 * Orologio: `performance.now` e `Date.now` sono sostituiti prima del
 * caricamento. Durante la preparazione della scena l'orologio avanza da
 * solo (così transizioni e assestamenti si concludono); poi si ferma, i cavi
 * vengono ricreati (nel prototipo `t0` dei cavi, nel nuovo canvas
 * annulla/ripristina) e si portano le lancette a 0, 250 e 500 ms.
 * L'attesa del prototipo è un'animazione CSS: si mette in pausa e si
 * imposta `currentTime` sullo stesso istante.
 *
 * Salva in docs/visual/fase4b/: {prototipo,v2-chiaro,v2-scuro}-t{0,250,500}.png,
 * v2-chiaro-movimento-ridotto.png, prototipo.webm, v2-chiaro.webm,
 * misure.json (flusso, risparmio energetico) e console.txt.
 *
 * Uso: node scripts/visual-fase4b.mjs
 */
import { mkdirSync, readdirSync, renameSync, rmSync, writeFileSync } from "node:fs";
import { resolve } from "node:path";
import { pathToFileURL } from "node:url";
import { chromium } from "playwright";
import { ROOT, SOLUTION, manropeCss, startServer } from "./visual-lib.mjs";

const OUT = resolve(ROOT, "docs/visual/fase4b");
const PROTOTYPE = resolve(ROOT, "docs/prototype/isa-fusion-prototype.html");
const VIEWPORT = { width: 1440, height: 900 };
const INSTANTS = [0, 250, 500];
/** Istante (ms) in cui si ferma l'orologio e "parte" il flusso. */
const FREEZE = 100000;
const PORT = Number(process.env.PORT ?? 5199);

mkdirSync(OUT, { recursive: true });

const CLOCK = `(() => {
  const c = { t: 1000, auto: true };
  window.__clock = c;
  performance.now = () => c.t;
  Date.now = () => 1.7e12 + c.t;
  setInterval(() => { if (c.auto) c.t += 16; }, 16);
})();`;

const RAF_COUNTER = `(() => {
  window.__raf = 0;
  const r = window.requestAnimationFrame.bind(window);
  window.requestAnimationFrame = (cb) => { window.__raf++; return r(cb); };
})();`;

async function newContext(browser, extra = {}) {
  return browser.newContext({ viewport: VIEWPORT, deviceScaleFactor: 1, ...extra });
}

async function v2Page(ctx, base, theme, seed = true) {
  await ctx.addInitScript(
    ([solution, dark]) => {
      localStorage.setItem("isa.solutions", JSON.stringify([solution]));
      localStorage.setItem("isa-theme", dark ? "dark" : "light");
    },
    [SOLUTION, theme === "scuro"],
  );
  const page = await ctx.newPage();
  await page.goto(`${base}/solutions/${SOLUTION.id}/etl${seed ? "?seed=prototype" : ""}`);
  if (seed) await page.waitForSelector('[data-node-id="ds1"]', { timeout: 60000 });
  else await page.waitForSelector(".ec-stage", { timeout: 60000 });
  await page.evaluate(() => document.fonts.ready);
  // in sviluppo ThemeProvider sovrascrive il tema salvato: si fissa con la classe `.dark`
  await page.evaluate(
    (dark) => document.documentElement.classList.toggle("dark", dark),
    theme === "scuro",
  );
  return page;
}

async function protoPage(ctx) {
  const page = await ctx.newPage();
  await page.route("https://fonts.googleapis.com/**", (r) =>
    r.fulfill({ contentType: "text/css", body: manropeCss() }),
  );
  await page.goto(pathToFileURL(PROTOTYPE).href);
  await page.evaluate(() => document.fonts.ready);
  await page.evaluate(() => document.getElementById("stage").scrollIntoView({ block: "center" }));
  return page;
}

/** Stessa scena in entrambi: un join con una sola tabella (output parziale), un filtro, un box combinato. */
async function scene(page, kind) {
  if (kind === "prototipo") {
    await page.evaluate(() => {
      connect("ds1", "op-join");
      connect("ds1", "op-filter");
      performMerge("op-sort", "op-export");
    });
  } else {
    await page.evaluate(() => {
      const s = window.__etlStore;
      s.dispatch({ type: "connect", payload: { from: "ds1", to: "op-join" } });
      s.dispatch({ type: "connect", payload: { from: "ds1", to: "op-filter" } });
      s.dispatch({ type: "merge", payload: { dragged: "op-sort", target: "op-export" } });
    });
  }
  await page.waitForTimeout(2500); // l'orologio finto avanza: transizioni e assestamenti si concludono
}

/** Ferma l'orologio a FREEZE e fa ripartire i cavi (e le fette) da zero. */
async function restart(page, kind) {
  if (kind === "prototipo") {
    await page.evaluate((F) => {
      __clock.auto = false;
      __clock.t = F;
      Object.values(linkState).forEach((st) => (st.t0 = F));
      document.getAnimations().forEach((a) => a.pause());
    }, FREEZE);
  } else {
    await page.evaluate((F) => {
      __clock.auto = false;
      __clock.t = F;
      const s = window.__etlStore;
      while (s.canUndo()) s.undo();
    }, FREEZE);
    await page.waitForTimeout(400);
    await page.evaluate(() => {
      const s = window.__etlStore;
      while (s.canRedo()) s.redo();
    });
  }
  await page.waitForTimeout(600);
}

async function at(page, kind, ms) {
  await page.evaluate(
    ([F, t, k]) => {
      __clock.t = F + t;
      if (k === "prototipo") document.getAnimations().forEach((a) => (a.currentTime = t));
    },
    [FREEZE, ms, kind],
  );
  await page.waitForTimeout(350);
}

/** Tubi del flusso visibili: rettangolo di ingombro di ciascuno (in coordinate dello stage). */
async function flowBoxes(page, kind) {
  return page.evaluate((k) => {
    const stage = document
      .querySelector(k === "prototipo" ? "#stage" : ".ec-stage")
      .getBoundingClientRect();
    const els = [
      ...document.querySelectorAll(k === "prototipo" ? "#bubbles path" : ".ec-flow"),
    ].filter((e) => (e.getAttribute("d") || "").length > 0);
    return els
      .map((e) => {
        const r = e.getBoundingClientRect();
        return {
          x: +(r.left - stage.left).toFixed(1),
          y: +(r.top - stage.top).toFixed(1),
          w: +r.width.toFixed(1),
          h: +r.height.toFixed(1),
          fill: getComputedStyle(e).fill,
        };
      })
      .sort((a, b) => a.y - b.y || a.x - b.x);
  }, kind);
}

async function waitingOpacities(page, kind) {
  return page.evaluate((k) => {
    const sel = k === "prototipo" ? ".icon-wrap.split .half.empty svg" : ".ec-slice-empty svg";
    return [...document.querySelectorAll(sel)].map(
      (e) => +parseFloat(getComputedStyle(e).opacity).toFixed(3),
    );
  }, kind);
}

const issues = [];
const measures = {
  instants: INSTANTS,
  freezeMs: FREEZE,
  prototipo: {},
  "v2-chiaro": {},
  "v2-scuro": {},
};
const server = await startServer(PORT);
const browser = await chromium.launch();
try {
  // --- prototipo ---------------------------------------------------------------------------------
  {
    const ctx = await newContext(browser);
    await ctx.addInitScript(CLOCK);
    const page = await protoPage(ctx);
    await scene(page, "prototipo");
    await restart(page, "prototipo");
    for (const ms of INSTANTS) {
      await at(page, "prototipo", ms);
      await page.screenshot({ path: resolve(OUT, `prototipo-t${ms}.png`) });
      measures.prototipo[`t${ms}`] = {
        flusso: await flowBoxes(page, "prototipo"),
        attesa: await waitingOpacities(page, "prototipo"),
      };
    }
    await ctx.close();
  }
  // --- nuovo canvas, chiaro e scuro -----------------------------------------------------------------
  for (const theme of ["chiaro", "scuro"]) {
    const ctx = await newContext(browser);
    await ctx.addInitScript(CLOCK);
    const page = await v2Page(ctx, server.base, theme);
    page.on("console", (m) => {
      if (m.type() === "error" || m.type() === "warning")
        issues.push(`[${theme}] ${m.type()}: ${m.text()}`);
    });
    page.on("pageerror", (e) => issues.push(`[${theme}] pageerror: ${e.message}`));
    await scene(page, "v2");
    await restart(page, "v2");
    for (const ms of INSTANTS) {
      await at(page, "v2", ms);
      await page.screenshot({ path: resolve(OUT, `v2-${theme}-t${ms}.png`) });
      measures[`v2-${theme}`][`t${ms}`] = {
        flusso: await flowBoxes(page, "v2"),
        attesa: await waitingOpacities(page, "v2"),
      };
    }
    await ctx.close();
  }

  // --- risparmio energetico, nel browser reale (orologio vero) -------------------------------------------
  const energy = {};
  const rafDelta = async (page, ms) => {
    const a = await page.evaluate(() => window.__raf);
    await page.waitForTimeout(ms);
    return (await page.evaluate(() => window.__raf)) - a;
  };
  {
    // movimento normale
    const ctx = await newContext(browser);
    await ctx.addInitScript(RAF_COUNTER);
    const page = await v2Page(ctx, server.base, "chiaro");
    await scene(page, "v2");
    energy.movimentoNormale = { rafInUnSecondo: await rafDelta(page, 1000) };
    // scheda nascosta
    await page.evaluate(() => {
      Object.defineProperty(document, "hidden", { configurable: true, get: () => true });
      document.dispatchEvent(new Event("visibilitychange"));
    });
    await page.waitForTimeout(200);
    energy.schedaNascosta = { rafInUnSecondo: await rafDelta(page, 1000) };
    await page.evaluate(() => {
      Object.defineProperty(document, "hidden", { configurable: true, get: () => false });
      document.dispatchEvent(new Event("visibilitychange"));
    });
    await page.waitForTimeout(200);
    energy.schedaTornataVisibile = { rafInUnSecondo: await rafDelta(page, 1000) };
    // nulla da animare: annulla tutto (nessun cavo, nessuna fetta)
    await page.evaluate(() => {
      const s = window.__etlStore;
      while (s.canUndo()) s.undo();
    });
    await page.waitForTimeout(600);
    energy.nullaDaAnimare = {
      rafInUnSecondo: await rafDelta(page, 1000),
      cavi: await page.locator(".ec-link").count(),
    };
    await ctx.close();
  }
  {
    // movimento ridotto
    const ctx = await newContext(browser, { reducedMotion: "reduce" });
    await ctx.addInitScript(RAF_COUNTER);
    const page = await v2Page(ctx, server.base, "chiaro");
    await scene(page, "v2");
    const boxes1 = await flowBoxes(page, "v2");
    energy.movimentoRidotto = { rafInUnSecondo: await rafDelta(page, 1000) };
    const boxes2 = await flowBoxes(page, "v2");
    energy.movimentoRidotto.flussoStatico =
      JSON.stringify(boxes1) === JSON.stringify(boxes2) && boxes1.length > 0;
    energy.movimentoRidotto.tubi = boxes1.length;
    energy.movimentoRidotto.attesa = await waitingOpacities(page, "v2");
    await page.screenshot({ path: resolve(OUT, "v2-chiaro-movimento-ridotto.png") });
    await ctx.close();
  }
  measures.risparmioEnergetico = energy;

  // --- video, con orologio vero ---------------------------------------------------------------------------
  const tmp = resolve(OUT, ".video");
  rmSync(tmp, { recursive: true, force: true });
  for (const kind of ["prototipo", "v2-chiaro"]) {
    const dir = resolve(tmp, kind);
    const ctx = await newContext(browser, {
      recordVideo: { dir, size: { width: 960, height: 600 } },
    });
    const page =
      kind === "prototipo" ? await protoPage(ctx) : await v2Page(ctx, server.base, "chiaro");
    await scene(page, kind === "prototipo" ? "prototipo" : "v2");
    await page.waitForTimeout(3000);
    await ctx.close();
    const file = readdirSync(dir).find((f) => f.endsWith(".webm"));
    renameSync(resolve(dir, file), resolve(OUT, `${kind}.webm`));
  }
  rmSync(tmp, { recursive: true, force: true });

  writeFileSync(resolve(OUT, "misure.json"), JSON.stringify(measures, null, 2) + "\n");
  writeFileSync(
    resolve(OUT, "console.txt"),
    issues.length ? issues.join("\n") + "\n" : "nessun errore né avviso in console\n",
  );
  console.log("schermate in", OUT);
  console.log(JSON.stringify(energy, null, 2));
  console.log(
    issues.length
      ? `console: ${issues.length} messaggi (vedi console.txt)`
      : "console: nessun errore né avviso",
  );
} finally {
  await browser.close();
  server.stop();
}
process.exit(0);
```

### `scripts/visual-lib.mjs`

87 righe

```js
/** Utilità condivise dagli script di verifica visiva: server di sviluppo e carattere del prototipo. */
import { spawn } from "node:child_process";
import { readFileSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

export const ROOT = resolve(dirname(fileURLToPath(import.meta.url)), "..");

/** Avvia `vite dev` (o usa BASE_URL) e attende che risponda. `stop()` chiude l'intero gruppo di processi. */
export async function startServer(port) {
  if (process.env.BASE_URL) return { base: process.env.BASE_URL, stop() {} };
  const child = spawn(
    "npx",
    ["vite", "dev", "--port", String(port), "--strictPort", "--host", "127.0.0.1"],
    {
      cwd: ROOT,
      detached: true,
      stdio: ["ignore", "pipe", "pipe"],
    },
  );
  let log = "";
  child.stdout.on("data", (d) => (log += d));
  child.stderr.on("data", (d) => (log += d));
  const base = `http://127.0.0.1:${port}`;
  const t0 = Date.now();
  for (;;) {
    if (Date.now() - t0 > 90000) {
      child.kill();
      throw new Error("vite dev non risponde:\n" + log);
    }
    try {
      const r = await fetch(base + "/");
      if (r.status < 500) break;
    } catch {
      /* non ancora pronto */
    }
    await new Promise((r) => setTimeout(r, 500));
  }
  return {
    base,
    stop: () => {
      try {
        process.kill(-child.pid);
      } catch {
        /* già terminato */
      }
    },
  };
}

/** Manrope locale al posto di Google Fonts (nessuna rete), per il prototipo. */
export function manropeCss() {
  const dir = resolve(ROOT, "node_modules/@fontsource-variable/manrope/files");
  const face = (name, range) => {
    const b64 = readFileSync(resolve(dir, name)).toString("base64");
    return (
      `@font-face{font-family:'Manrope';font-style:normal;font-weight:200 800;font-display:block;` +
      `src:url(data:font/woff2;base64,${b64}) format('woff2');unicode-range:${range};}`
    );
  };
  return (
    face(
      "manrope-latin-ext-wght-normal.woff2",
      "U+0100-02BA,U+02BD-02C5,U+02C7-02CC,U+02CE-02D7,U+02DD-02FF,U+0304,U+0308,U+0329,U+1D00-1DBF,U+1E00-1E9F,U+1EF2-1EFF,U+2020,U+20A0-20AB,U+20AD-20C0,U+2113,U+2C60-2C7F,U+A720-A7FF",
    ) +
    face(
      "manrope-latin-wght-normal.woff2",
      "U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+0304,U+0308,U+0329,U+2000-206F,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD",
    )
  );
}

export const SOLUTION = {
  id: "visual",
  name: "Verifica visiva",
  description: "",
  status: "draft",
  version: "v1",
  chart: "bar",
  series: [],
  updatedAt: "",
  owner: "",
  parameters: [],
  modules: { etl: "draft" },
  shares: [],
};
```

### `tsconfig.json`

31 righe

```json
{
  "include": ["src/**/*.ts", "src/**/*.tsx", "vite.config.ts", "eslint.config.js"],
  "compilerOptions": {
    "target": "ES2022",
    "jsx": "react-jsx",
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["vite/client"],

    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": false,
    "noEmit": true,

    "skipLibCheck": true,
    "strict": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noFallthroughCasesInSwitch": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noPropertyAccessFromIndexSignature": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedSideEffectImports": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### `vite.config.ts`

16 righe

```ts
// @lovable.dev/vite-tanstack-config already includes the following — do NOT add them manually
// or the app will break with duplicate plugins:
//   - TanStack devtools (dev-only, first), tanstackStart, viteReact, tailwindcss, tsConfigPaths,
//     nitro (build-only using cloudflare as a default target), VITE_* env injection, @ path alias,
//     React/TanStack dedupe, error logger plugins, and sandbox detection (port/host/strictPort).
// You can pass additional config via defineConfig({ vite: { ... }, etc... }) if needed.
import { defineConfig } from "@lovable.dev/vite-tanstack-config";

export default defineConfig({
  tanstackStart: {
    // Redirect TanStack Start's bundled server entry to src/server.ts (our SSR error wrapper).
    // nitro/vite builds from this
    server: { entry: "server" },
  },
});
```

### `vitest.config.ts`

11 righe

```ts
import react from "@vitejs/plugin-react";
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: "node",
  },
});
```

