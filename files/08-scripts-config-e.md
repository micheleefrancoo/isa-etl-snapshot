# 08-scripts-config-e.md

File in questo blocco:

- `scripts/visual-fase4b.mjs`
- `scripts/visual-lib.mjs`
- `scripts/visual-temi.mjs`
- `tsconfig.json`
- `vite.config.ts`
- `vitest.config.ts`

---

### `scripts/visual-fase4b.mjs`

323 righe

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
  // il canvas nudo ha i pannelli chiusi (Fase 6a: la cassetta si apre da sola): si chiudono nello store, senza compensare la vista e senza transizione
  await page.addStyleTag({ content: ".ec-workspace, .ec-panel { transition: none !important; }" });
  await page.evaluate(() =>
    window.__etlStore.dispatch({ type: "setPanel", payload: { panel: "tools", open: false } }),
  );
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

### `scripts/visual-temi.mjs`

107 righe

```js
#!/usr/bin/env node
/**
 * Verifica visiva del sistema di temi (Fase T). Schermate di due pagine
 * (elenco soluzioni e canvas ETL, 1440 × 900) in chiaro e in scuro.
 *
 * Uso: node scripts/visual-temi.mjs <gruppo>
 *   prototipo  tema predefinito → docs/visual/temi/prototipo-{chiaro,scuro}-{soluzioni,canvas}.png
 *              (da confrontare al pixel con scripts/visual-compare.mjs)
 *   notte      tema "notte"     → docs/visual/temi/notte-{chiaro,scuro}-{soluzioni,canvas}.png
 *   tinte      deriveAccent     → docs/visual/temi/tinta-{0,140,280}-{chiaro,scuro}-canvas.png
 *
 * Il modo (chiaro/scuro) arriva da localStorage, come per un utente reale (nel
 * canvas del tema predefinito, dalla classe `.dark`: vedi `shot`); il tema da
 * `?theme=` e la tinta da `setAccentHue` (solo sviluppo).
 */
import { mkdirSync } from "node:fs";
import { resolve } from "node:path";
import { chromium } from "playwright";
import { ROOT, SOLUTION, startServer } from "./visual-lib.mjs";

const OUT = resolve(ROOT, "docs/visual/temi");
const VIEWPORT = { width: 1440, height: 900 };
const PORT = Number(process.env.PORT ?? 5198);
const group = process.argv[2];
if (!["prototipo", "notte", "tinte"].includes(group)) {
  console.error("Uso: node scripts/visual-temi.mjs prototipo|notte|tinte");
  process.exit(2);
}
mkdirSync(OUT, { recursive: true });

const server = await startServer(PORT);
const browser = await chromium.launch();

/**
 * `via`: come si imposta il modo. "archivio" = localStorage, letto dallo script
 * di avvio (il meccanismo reale); "classe" = classe `.dark` dopo il caricamento,
 * come nelle fasi precedenti (lo stato React non cambia: l'etichetta del
 * pulsante resta quella di prima).
 */
async function shot(mode, path, file, { theme, hue, via = "archivio" } = {}) {
  const ctx = await browser.newContext({
    viewport: VIEWPORT,
    deviceScaleFactor: 1,
    reducedMotion: "reduce",
  });
  await ctx.addInitScript(
    ([solution, m]) => {
      localStorage.setItem("isa.solutions", JSON.stringify([solution]));
      if (m) localStorage.setItem("isa-theme", m);
    },
    [SOLUTION, via === "archivio" ? (mode === "scuro" ? "dark" : "light") : null],
  );
  const page = await ctx.newPage();
  const sep = path.includes("?") ? "&" : "?";
  await page.goto(`${server.base}${path}${theme ? `${sep}theme=${theme}` : ""}`);
  if (path.includes("/etl")) {
    await page.waitForSelector('[data-node-id="ds1"]', { timeout: 60000 });
    // il canvas nudo ha i pannelli chiusi (Fase 6a: la cassetta si apre da sola): si chiudono nello store, senza compensare la vista e senza transizione
    await page.addStyleTag({
      content: ".ec-workspace, .ec-panel { transition: none !important; }",
    });
    await page.evaluate(() =>
      window.__etlStore.dispatch({ type: "setPanel", payload: { panel: "tools", open: false } }),
    );
  } else {
    await page.waitForSelector("main, [data-slot], h1", { timeout: 60000 });
  }
  await page.evaluate(() => document.fonts.ready);
  if (via === "classe")
    await page.evaluate(
      (dark) => document.documentElement.classList.toggle("dark", dark),
      mode === "scuro",
    );
  if (hue !== undefined) {
    await page.evaluate(async (h) => {
      const m = await import("/src/theme/runtime.ts");
      m.setAccentHue(h);
    }, hue);
  }
  await page.waitForTimeout(1500);
  await page.screenshot({ path: resolve(OUT, file), animations: "disabled" });
  await ctx.close();
  console.log("salvato", file);
}

const CANVAS = `/solutions/${SOLUTION.id}/etl?seed=prototype`;
try {
  if (group === "tinte") {
    for (const hue of [0, 140, 280])
      for (const mode of ["chiaro", "scuro"])
        await shot(mode, CANVAS, `tinta-${hue}-${mode}-canvas.png`, { hue });
  } else {
    const theme = group === "notte" ? "notte" : undefined;
    for (const mode of ["chiaro", "scuro"]) {
      await shot(mode, "/", `${group}-${mode}-soluzioni.png`, { theme });
      // per il tema predefinito il canvas segue il metodo delle fasi precedenti, per il confronto con i riferimenti
      await shot(mode, CANVAS, `${group}-${mode}-canvas.png`, {
        theme,
        via: group === "prototipo" ? "classe" : "archivio",
      });
    }
  }
} finally {
  await browser.close();
  server.stop();
}
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

93 righe

```ts
import { defineConfig, loadEnv } from "vite";
import { devtools } from "@tanstack/devtools-vite";
import { tanstackStart } from "@tanstack/react-start/plugin/vite";
import tailwindcss from "@tailwindcss/vite";
import viteReact from "@vitejs/plugin-react";
import { nitro } from "nitro/vite";
import tsConfigPaths from "vite-tsconfig-paths";

// Configurazione esplicita (prima delegata a un pacchetto esterno).
// Ordine dei plugin: devtools (solo dev), tailwind, percorsi di tsconfig,
// TanStack Start, nitro (solo build), React.
export default defineConfig(({ command, mode }) => {
  const isDev = mode === "development";
  const viteEnv = loadEnv(mode, process.cwd(), "VITE_");

  return {
    define: Object.fromEntries(
      Object.entries(viteEnv).map(([key, value]) => [
        `import.meta.env.${key}`,
        JSON.stringify(value),
      ]),
    ),
    ...(command === "build" && isDev
      ? {
          environments: {
            client: { define: { "process.env.NODE_ENV": JSON.stringify("development") } },
          },
        }
      : {}),
    css: { transformer: "lightningcss" },
    resolve: {
      alias: { "@": `${process.cwd()}/src` },
      dedupe: [
        "react",
        "react-dom",
        "react/jsx-runtime",
        "react/jsx-dev-runtime",
        "@tanstack/react-query",
        "@tanstack/query-core",
      ],
    },
    optimizeDeps: {
      include: [
        "react",
        "react-dom",
        "react-dom/client",
        "react/jsx-runtime",
        "react/jsx-dev-runtime",
      ],
      ignoreOutdatedRequests: true,
    },
    server: {
      host: "::",
      port: 8080,
      watch: { awaitWriteFinish: { stabilityThreshold: 1000, pollInterval: 100 } },
    },
    plugins: [
      ...(isDev
        ? [
            devtools({
              logging: false,
              eventBusConfig: { enabled: false },
              enhancedLogs: { enabled: false },
              consolePiping: { enabled: false },
              removeDevtoolsOnBuild: false,
              injectSource: { enabled: true },
            }),
          ]
        : []),
      tailwindcss(),
      tsConfigPaths({ projects: ["./tsconfig.json"] }),
      tanstackStart({
        // Redirect TanStack Start's bundled server entry to src/server.ts (our SSR error wrapper).
        server: { entry: "server" },
        importProtection: {
          behavior: "error",
          client: { files: ["**/server/**"], specifiers: ["server-only"] },
        },
      }),
      // Deploy: Cloudflare (non ancora in produzione), solo in build.
      ...(command === "build"
        ? [
            nitro({
              preset: "cloudflare-module",
              cloudflare: { nodeCompat: true, deployConfig: true },
            }),
          ]
        : []),
      viteReact(),
    ],
  };
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

