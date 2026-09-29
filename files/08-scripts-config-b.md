# 08-scripts-config-b.md

File in questo blocco:

- `scripts/visual-fase4.mjs`
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
    await page.goto(`${server.base}/solutions/${SOLUTION.id}/etl?canvas=v2&seed=prototype`);
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

