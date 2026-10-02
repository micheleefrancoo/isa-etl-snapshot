# 08-scripts-config-a.md

File in questo blocco:

- `.claude/settings.local.json`
- `.devcontainer/devcontainer.json`
- `.gitignore`
- `.prettierignore`
- `.prettierrc`
- `.vscode/settings.json`
- `components.json`
- `eslint.config.js`
- `package.json`
- `scripts/check-tokens.mjs`
- `scripts/e2e-fase5.mjs`
- `scripts/extract-golden.mjs`
- `scripts/generate-index.mjs`

---

### `.claude/settings.local.json`

23 righe

```json
{
  "permissions": {
    "allow": [
      "Bash",
      "Read",
      "Write",
      "Edit"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(rm -rf /:*)",
      "Bash(git push --force:*)",
      "Bash(git push -f:*)",
      "Bash(git reset --hard:*)",
      "Bash(git checkout -- .:*)",
      "Bash(git clean -fd:*)",
      "Bash(> :*)",
      "Bash(chmod -R 777:*)",
      "Bash(curl * | sh)",
      "Bash(curl * | bash)"
    ]
  }
}
```

### `.devcontainer/devcontainer.json`

16 righe

```json
{
  "name": "isa-glass-platform",
  "customizations": {
    "vscode": {
      "extensions": [
        "bradlc.vscode-tailwindcss"
      ],
      "settings": {
        "css.validate": false,
        "less.validate": false,
        "scss.validate": false
      }
    }
  }
}
```

### `.gitignore`

34 righe

```
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

node_modules
dist
dist-ssr
.output
.vinxi
.tanstack/**
.nitro
*.local

# Wrangler / Cloudflare
.wrangler/
.dev.vars

# Editor directories and files
.vscode/*
!.vscode/extensions.json
.idea
.DS_Store
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?
.pw-tmp/
```

### `.prettierignore`

9 righe

```
node_modules
dist
.output
.vinxi
pnpm-lock.yaml
package-lock.json
bun.lock
routeTree.gen.ts
```

### `.prettierrc`

7 righe

```
{
  "printWidth": 100,
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all"
}
```

### `.vscode/settings.json`

6 righe

```json
{
  "css.validate": false,
  "less.validate": false,
  "scss.validate": false
}
```

### `components.json`

23 righe

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "css": "src/styles.css",
    "baseColor": "slate",
    "cssVariables": true,
    "prefix": ""
  },
  "iconLibrary": "lucide",
  "rtl": false,
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "registries": {}
}
```

### `eslint.config.js`

41 righe

```js
import js from "@eslint/js";
import eslintPluginPrettier from "eslint-plugin-prettier/recommended";
import globals from "globals";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";
import tseslint from "typescript-eslint";

export default tseslint.config(
  { ignores: ["dist", ".output", ".vinxi"] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ["**/*.{ts,tsx}"],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },
    plugins: {
      "react-hooks": reactHooks,
      "react-refresh": reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      "no-restricted-imports": [
        "error",
        {
          paths: [
            {
              name: "server-only",
              message:
                "TanStack Start does not use the Next.js `server-only` package. Rename the module to `*.server.ts` or mark it with `@tanstack/react-start/server-only`.",
            },
          ],
        },
      ],
      "react-refresh/only-export-components": ["warn", { allowConstantExport: true }],
      "@typescript-eslint/no-unused-vars": "off",
    },
  },
  eslintPluginPrettier,
);
```

### `package.json`

97 righe

```json
{
  "name": "tanstack_start_ts",
  "private": true,
  "sideEffects": false,
  "type": "module",
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "build:dev": "vite build --mode development",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier --write .",
    "test": "vitest run",
    "check:tokens": "node scripts/check-tokens.mjs"
  },
  "overrides": {
    "rolldown": "1.2.1"
  },
  "dependencies": {
    "@fontsource-variable/manrope": "^5.3.0",
    "@hookform/resolvers": "^5.2.2",
    "@radix-ui/react-accordion": "^1.2.12",
    "@radix-ui/react-alert-dialog": "^1.1.15",
    "@radix-ui/react-aspect-ratio": "^1.1.8",
    "@radix-ui/react-avatar": "^1.1.11",
    "@radix-ui/react-checkbox": "^1.3.3",
    "@radix-ui/react-collapsible": "^1.1.12",
    "@radix-ui/react-context-menu": "^2.2.16",
    "@radix-ui/react-dialog": "^1.1.15",
    "@radix-ui/react-dropdown-menu": "^2.1.16",
    "@radix-ui/react-hover-card": "^1.1.15",
    "@radix-ui/react-label": "^2.1.8",
    "@radix-ui/react-menubar": "^1.1.16",
    "@radix-ui/react-navigation-menu": "^1.2.14",
    "@radix-ui/react-popover": "^1.1.15",
    "@radix-ui/react-progress": "^1.1.8",
    "@radix-ui/react-radio-group": "^1.3.8",
    "@radix-ui/react-scroll-area": "^1.2.10",
    "@radix-ui/react-select": "^2.2.6",
    "@radix-ui/react-separator": "^1.1.8",
    "@radix-ui/react-slider": "^1.3.6",
    "@radix-ui/react-slot": "^1.2.4",
    "@radix-ui/react-switch": "^1.2.6",
    "@radix-ui/react-tabs": "^1.1.13",
    "@radix-ui/react-toggle": "^1.1.10",
    "@radix-ui/react-toggle-group": "^1.1.11",
    "@radix-ui/react-tooltip": "^1.2.8",
    "@tailwindcss/vite": "^4.2.1",
    "@tanstack/react-query": "^5.101.1",
    "@tanstack/react-router": "1.170.18",
    "@tanstack/react-start": "1.168.32",
    "@tanstack/router-plugin": "1.168.23",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "cmdk": "^1.1.1",
    "date-fns": "^4.1.0",
    "embla-carousel-react": "^8.6.0",
    "input-otp": "^1.4.2",
    "lucide-react": "^0.575.0",
    "react": "^19.2.0",
    "react-day-picker": "^9.14.0",
    "react-dom": "^19.2.0",
    "react-hook-form": "^7.71.2",
    "react-resizable-panels": "^4.6.5",
    "recharts": "^2.15.4",
    "sonner": "^2.0.7",
    "tailwind-merge": "^3.5.0",
    "tailwindcss": "^4.2.1",
    "tw-animate-css": "^1.3.4",
    "vaul": "^1.1.2",
    "vite-tsconfig-paths": "^6.0.2",
    "zod": "^3.25.76"
  },
  "devDependencies": {
    "@eslint/js": "^9.32.0",
    "@tanstack/devtools-vite": "^0.8.5",
    "@types/node": "^22.16.5",
    "@types/react": "^19.2.0",
    "@types/react-dom": "^19.2.0",
    "@vitejs/plugin-react": "^5.2.0",
    "eslint": "^9.32.0",
    "eslint-config-prettier": "^10.1.1",
    "eslint-plugin-prettier": "^5.2.6",
    "eslint-plugin-react-hooks": "^5.2.0",
    "eslint-plugin-react-refresh": "^0.4.20",
    "globals": "^15.15.0",
    "lightningcss": "^1.33.0",
    "nitro": "3.0.260603-beta",
    "playwright": "^1.63.0",
    "prettier": "^3.7.3",
    "typescript": "^5.8.3",
    "typescript-eslint": "^8.56.1",
    "vite": "8.1.5",
    "vitest": "^5.0.1"
  }
}
```

### `scripts/check-tokens.mjs`

207 righe

```js
#!/usr/bin/env node
/**
 * Disciplina dei token (Fase T). Fallisce (exit 1) se trova valori scritti a
 * mano fuori dai file dei token:
 *   - colori letterali: #hex, rgb()/rgba(), hsl()/hsla(), oklch(), oklab(), lab(), lch(), hwb();
 *   - raggi letterali (border-radius, borderRadius, rounded-[..]);
 *   - ombre letterali (box-shadow, text-shadow, drop-shadow, boxShadow, shadow-[..]).
 * Un raggio o un'ombra sono a norma solo se il valore è `0`/`none`/`inherit`
 * oppure è composto soltanto da `var(--token)`.
 *
 * Dove si applica (CONTROLLATI, l'esito conta):
 *   - tutto src/etl-canvas/;
 *   - ogni file css/ts/tsx sotto src/ che NON è nell'elenco dei file preesistenti
 *     (scripts/token-legacy-files.txt, congelato a Fase T).
 * Esenti: i file dei token (src/theme/), i test (__tests__, *.test.*), i file
 * generati (*.gen.ts) e i report.
 *
 * Il resto dell'app esistente NON viene corretto: i suoi valori scritti a mano
 * sono il DEBITO da saldare nel restyling. Si elenca (file e riga) con
 *   node scripts/check-tokens.mjs --debt            stampa l'elenco
 *   node scripts/check-tokens.mjs --write-debt F.md salva l'elenco in Markdown
 *   node scripts/check-tokens.mjs --check-file F    controlla solo F come file nuovo
 * ma non fa fallire il controllo.
 */
import { execFileSync } from "node:child_process";
import { readFileSync, writeFileSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

const ROOT = resolve(dirname(fileURLToPath(import.meta.url)), "..");
const LEGACY = new Set(
  readFileSync(resolve(ROOT, "scripts/token-legacy-files.txt"), "utf8").split("\n").filter(Boolean),
);

const COLOR =
  /#[0-9a-fA-F]{8}\b|#[0-9a-fA-F]{6}\b|#[0-9a-fA-F]{3,4}\b|\b(?:rgba?|hsla?|oklch|oklab|lab|lch|hwb)\(/g;
const SHADOW_PROPS = /(?:^|-)(?:box-shadow|text-shadow|boxShadow|textShadow)$|shadow$/i;
const RADIUS_PROPS = /radius$/i;

/** Il valore è fatto solo di `var(...)` (con eventuale fallback assente) o di parole neutre. */
function isTokenOnly(value) {
  const v = value.trim().replace(/\s*!important$/, "");
  if (/^(?:0|none|inherit|initial|unset|revert|currentcolor|transparent)$/i.test(v)) return true;
  const rest = v.replace(/var\(--[a-z0-9-]+\)/gi, "").trim();
  return rest === "";
}

/** Toglie i commenti lasciando intatti i ritorni a capo (le righe restano quelle). */
function stripCss(src) {
  return src.replace(/\/\*[\s\S]*?\*\//g, (m) => m.replace(/[^\n]/g, " "));
}

function stripTs(src) {
  let out = "";
  let i = 0;
  while (i < src.length) {
    const c = src[i];
    const n = src[i + 1];
    if (c === "/" && n === "/") {
      while (i < src.length && src[i] !== "\n") ((out += " "), i++);
    } else if (c === "/" && n === "*") {
      while (i < src.length && !(src[i] === "*" && src[i + 1] === "/"))
        ((out += src[i] === "\n" ? "\n" : " "), i++);
      out += "  ";
      i += 2;
    } else if (c === '"' || c === "'" || c === "`") {
      const q = c;
      out += c;
      i++;
      while (i < src.length && src[i] !== q) {
        if (src[i] === "\\") ((out += src[i]), i++);
        out += src[i] ?? "";
        i++;
      }
      out += q;
      i++;
    } else {
      out += c;
      i++;
    }
  }
  return out;
}

const lineOf = (text, index) => text.slice(0, index).split("\n").length;

function scanCss(text) {
  const found = [];
  for (const m of text.matchAll(/([a-zA-Z-]+)\s*:\s*([^;{}]+)/g)) {
    const [, prop, value] = m;
    const line = lineOf(text, m.index);
    for (const c of value.matchAll(COLOR)) found.push({ line, kind: "colore", text: c[0] });
    if (RADIUS_PROPS.test(prop) && !isTokenOnly(value))
      found.push({ line, kind: "raggio", text: `${prop}: ${value.trim()}` });
    if ((SHADOW_PROPS.test(prop) || /drop-shadow\(/.test(value)) && !isTokenOnly(value))
      found.push({ line, kind: "ombra", text: `${prop}: ${value.trim().replace(/\s+/g, " ")}` });
  }
  // utility Tailwind in @apply
  return found;
}

function scanTs(text) {
  const found = [];
  const lines = text.split("\n");
  lines.forEach((ln, idx) => {
    const line = idx + 1;
    for (const c of ln.matchAll(COLOR)) {
      // un `#` seguito da cifre/lettere esadecimali dentro una stringa; esclude i frammenti di URL non esadecimali
      found.push({ line, kind: "colore", text: c[0] });
    }
    for (const m of ln.matchAll(/\b(borderRadius|boxShadow|textShadow)\s*:\s*([^,}]+)/g)) {
      const value = m[2].trim().replace(/^["'`]|["'`]$/g, "");
      if (!isTokenOnly(value))
        found.push({
          line,
          kind: m[1] === "borderRadius" ? "raggio" : "ombra",
          text: `${m[1]}: ${m[2].trim()}`,
        });
    }
    for (const m of ln.matchAll(
      /\b(rounded(?:-[a-z]{1,2})?|shadow(?:-[a-z]{1,2})?)-\[([^\]]+)\]/g,
    )) {
      if (!/^(?:var\(--[a-z0-9-]+\)|inherit|0)$/.test(m[2]))
        found.push({ line, kind: m[1].startsWith("rounded") ? "raggio" : "ombra", text: m[0] });
    }
  });
  return found;
}

function listFiles() {
  const out = execFileSync("git", ["ls-files", "-co", "--exclude-standard", "--", "src"], {
    cwd: ROOT,
    encoding: "utf8",
  });
  return out.split("\n").filter((f) => /\.(css|ts|tsx)$/.test(f));
}

const isExempt = (f) =>
  f.startsWith("src/theme/") ||
  /(^|\/)__tests__\//.test(f) ||
  /\.test\.[tj]sx?$/.test(f) ||
  /\.gen\.ts$/.test(f) ||
  f.includes("/.reports/");

function scan(file) {
  const raw = readFileSync(resolve(ROOT, file), "utf8");
  return file.endsWith(".css") ? scanCss(stripCss(raw)) : scanTs(stripTs(raw));
}

const args = process.argv.slice(2);
const strict = [];
const debt = [];

// `--check-file F`: controlla solo F, come file nuovo (usato dai test).
const cf = args.indexOf("--check-file");
if (cf >= 0) {
  const target = resolve(args[cf + 1]);
  const raw = readFileSync(target, "utf8");
  const hits = target.endsWith(".css") ? scanCss(stripCss(raw)) : scanTs(stripTs(raw));
  for (const h of hits) console.error(`${args[cf + 1]}:${h.line}  [${h.kind}] ${h.text}`);
  process.exit(hits.length ? 1 : 0);
}

for (const file of listFiles()) {
  if (isExempt(file)) continue;
  const isStrict = file.startsWith("src/etl-canvas/") || !LEGACY.has(file);
  const hits = scan(file).map((h) => ({ file, ...h }));
  (isStrict ? strict : debt).push(...hits);
}

const fmt = (h) => `${h.file}:${h.line}  [${h.kind}] ${h.text}`;
const count = (hits) => {
  const by = {};
  for (const h of hits) by[h.kind] = (by[h.kind] ?? 0) + 1;
  return (
    Object.entries(by)
      .map(([k, n]) => `${n} ${k}`)
      .join(", ") || "nessuno"
  );
};

if (args.includes("--debt")) console.log(debt.map(fmt).join("\n"));

const wi = args.indexOf("--write-debt");
if (wi >= 0) {
  const byFile = new Map();
  for (const h of debt) byFile.set(h.file, [...(byFile.get(h.file) ?? []), h]);
  let md = `# Debito dei token: [REDATTO] scritti a mano\n\n`;
  md += `Generato da \`node scripts/check-tokens.mjs --write-debt\`. Elenca i colori, i raggi e le ombre letterali dei file di \`src/\` preesistenti alla Fase T (\`scripts/token-legacy-files.txt\`), da portare sui token semantici nel restyling. Il controllo non li fa fallire.\n\n`;
  md += `**Totale: ${debt.length}** (${count(debt)}) in ${byFile.size} file.\n\n`;
  for (const [file, hits] of [...byFile].sort()) {
    md += `## \`${file}\` (${hits.length})\n\n`;
    for (const h of hits) md += `- riga ${h.line} — ${h.kind}: \`${h.text.replace(/`/g, "'")}\`\n`;
    md += "\n";
  }
  writeFileSync(resolve(ROOT, args[wi + 1]), md);
}

console.log(
  `check-tokens: ${strict.length} violazioni nei file controllati; debito preesistente: ${debt.length} (${count(debt)}).`,
);
if (strict.length) {
  console.error("\nValori scritti a mano nei file controllati (usare i token semantici --isa-*):");
  console.error(strict.map(fmt).join("\n"));
  process.exit(1);
}
```

### `scripts/e2e-fase5.mjs`

294 righe

```js
#!/usr/bin/env node
/**
 * Verifica nel browser reale dei gesti della Fase 5 (Pointer Events veri,
 * tastiera, rotella) sul canvas con la scena del prototipo. Controlla lo stato
 * di etl-store (`window.__etlStore`, solo in sviluppo) e il DOM, e salva
 * alcune schermate in docs/visual/fase5/. Esce con codice 1 al primo errore.
 *
 * Uso: node scripts/e2e-fase5.mjs
 */
import { mkdirSync } from "node:fs";
import { resolve } from "node:path";
import { chromium } from "playwright";
import { ROOT, SOLUTION, startServer } from "./visual-lib.mjs";

const OUT = resolve(ROOT, "docs/visual/fase5");
mkdirSync(OUT, { recursive: true });
const server = await startServer(Number(process.env.PORT ?? 5196));
const browser = await chromium.launch();
const ctx = await browser.newContext({
  viewport: { width: 1440, height: 900 },
  deviceScaleFactor: 1,
});
await ctx.addInitScript((s) => {
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
  results.push({ prova: name, esito: ok ? "ok" : "FALLITA", dettaglio: extra });
  if (!ok) failed++;
}
const state = () => page.evaluate(() => window.__etlStore.getState());
const centerOf = async (id) => {
  const b = await page.locator(`[data-node-id="${id}"] .ec-icon-wrap`).boundingBox();
  return { x: b.x + b.width / 2, y: b.y + b.height / 2 };
};
async function dragTo(from, to, { steps = 20, hold = false } = {}) {
  await page.mouse.move(from.x, from.y);
  await page.mouse.down();
  for (let i = 1; i <= steps; i++)
    await page.mouse.move(
      from.x + ((to.x - from.x) * i) / steps,
      from.y + ((to.y - from.y) * i) / steps,
    );
  if (!hold) await page.mouse.up();
}
const shot = (name) => page.screenshot({ path: resolve(OUT, name), animations: "disabled" });

try {
  await page.goto(`${server.base}/solutions/${SOLUTION.id}/etl?seed=prototype`);
  await page.waitForSelector('[data-node-id="ds1"]', { timeout: 90000 });
  await page.evaluate(() => document.fonts.ready);
  await page.waitForTimeout(500);

  // 1. click: seleziona
  let c = await centerOf("op-sort");
  await page.mouse.click(c.x, c.y);
  let s = await state();
  check(
    "click seleziona il nodo e punta l'Inspector",
    s.selection.join() === "op-sort" && s.inspector.nodeId === "op-sort",
  );
  check(
    "il nodo selezionato ha la classe ec-selected",
    (await page.locator('[data-node-id="op-sort"].ec-selected').count()) === 1,
  );

  // 2. trascinamento su un altro nodo: dataset su lavorazione → collegamento, il dataset torna al suo posto
  const ds0 = (await state()).graph.cards.ds1;
  await dragTo(await centerOf("ds1"), await centerOf("op-filter"), { hold: true });
  check(
    "durante il trascinamento: contorno link sul bersaglio",
    (await page.locator('[data-node-id="op-filter"].ec-drop-link').count()) === 1,
  );
  check(
    "durante il trascinamento: il nodo ha ec-dragging",
    (await page.locator('[data-node-id="ds1"].ec-dragging').count()) === 1,
  );
  await shot("trascinamento-collegamento.png");
  await page.mouse.up();
  s = await state();
  check(
    "rilascio: collegamento creato",
    s.graph.links.some((l) => l.from === "ds1" && l.to === "op-filter"),
  );
  check(
    "rilascio: il dataset torna al suo posto",
    s.graph.cards.ds1.x === ds0.x && s.graph.cards.ds1.y === ds0.y,
  );
  check(
    "nessuna classe di gesto residua",
    (await page.locator(".ec-dragging, [class*=ec-drop-]").count()) === 0,
  );

  // 3. fusione
  await dragTo(await centerOf("op-sort"), await centerOf("op-export"), { hold: true });
  check(
    "contorno merge sul bersaglio",
    (await page.locator('[data-node-id="op-export"].ec-drop-merge').count()) === 1,
  );
  await shot("trascinamento-fusione.png");
  await page.mouse.up();
  s = await state();
  check(
    "fusione eseguita",
    !s.graph.cards["op-sort"] && s.graph.cards["op-export"].components.length === 2,
  );

  // 4. annulla con Ctrl+Z (un solo passo per il gesto)
  await page.keyboard.press("Control+z");
  s = await state();
  check(
    "Ctrl+Z annulla la fusione in un passo",
    !!s.graph.cards["op-sort"] && s.graph.cards["op-export"].components.length === 1,
  );
  await page.keyboard.press("Control+Shift+z");
  check("Ctrl+Maiusc+Z la ripristina", !(await state()).graph.cards["op-sort"]);
  await page.keyboard.press("Control+z");

  // 5. porta: si crea solo un collegamento, il nodo non si sposta
  c = await centerOf("op-join");
  await page.mouse.move(c.x, c.y);
  await page.waitForTimeout(250);
  const portBox = await page.locator('[data-node-id="op-join"] .ec-port-l').boundingBox();
  const joinBefore = { ...(await state()).graph.cards["op-join"] };
  const dsCenter = await centerOf("ds1");
  await dragTo({ x: portBox.x + portBox.width / 2, y: portBox.y + portBox.height / 2 }, dsCenter, {
    hold: true,
  });
  check(
    "cavo provvisorio visibile durante il trascinamento da una porta",
    (await page.locator('[data-testid="ec-temp-link"].ec-valid').count()) === 1,
  );
  await shot("trascinamento-porta.png");
  await page.mouse.up();
  s = await state();
  check(
    "porta: collegamento dataset → lavorazione",
    s.graph.links.some((l) => l.from === "ds1" && l.to === "op-join"),
  );
  check(
    "porta: la lavorazione non si è spostata",
    s.graph.cards["op-join"].x === joinBefore.x && s.graph.cards["op-join"].y === joinBefore.y,
  );

  // 5b. clic su un cavo: lo elimina; Ctrl+Z lo ripristina
  const hit = await page.evaluate(() => {
    const st = window.__etlStore;
    const pts = st.getRoutes()["ds1|op-join"].pts;
    const v = st.getState().view;
    const r = document.querySelector(".ec-stage").getBoundingClientRect();
    return {
      x: r.left + v.x + ((pts[0].x + pts[1].x) / 2) * v.zoom,
      y: r.top + v.y + ((pts[0].y + pts[1].y) / 2) * v.zoom,
    };
  });
  await page.mouse.click(hit.x, hit.y);
  check(
    "clic sul cavo: collegamento eliminato",
    !(await state()).graph.links.some((l) => l.from === "ds1" && l.to === "op-join"),
  );
  await page.keyboard.press("Control+z");
  check(
    "Ctrl+Z ripristina il cavo",
    (await state()).graph.links.some((l) => l.from === "ds1" && l.to === "op-join"),
  );

  // 6. riquadro di selezione sul vuoto
  const stage = await page.locator(".ec-stage").boundingBox();
  await page.mouse.move(stage.x + 700, stage.y + 60);
  await page.mouse.down();
  await page.mouse.move(stage.x + 500, stage.y + 200, { steps: 5 });
  check(
    "il riquadro è visibile durante il gesto",
    (await page.locator('[data-testid="ec-marquee"]').count()) === 1,
  );
  await page.mouse.move(stage.x + 200, stage.y + 420, { steps: 10 });
  await shot("riquadro-selezione.png");
  await page.mouse.up();
  s = await state();
  check("riquadro: seleziona più nodi", s.selection.length >= 3, s.selection.join());
  check(
    "riquadro: sparisce al rilascio",
    (await page.locator('[data-testid="ec-marquee"]').count()) === 0,
  );

  // 7. gruppo trascinato insieme
  const before = Object.fromEntries(
    s.selection.map((id) => [id, { x: s.graph.cards[id].x, y: s.graph.cards[id].y }]),
  );
  const first = s.selection[0];
  const fc = await centerOf(first);
  await dragTo(fc, { x: fc.x + 60, y: fc.y + 40 });
  s = await state();
  const moved = s.selection.filter(
    (id) => s.graph.cards[id].x !== before[id].x || s.graph.cards[id].y !== before[id].y,
  );
  check(
    "il gruppo selezionato si sposta insieme",
    moved.length === s.selection.length && s.selection.length >= 3,
    `${moved.length}/${s.selection.length}`,
  );

  // 8. Esc deseleziona
  await page.keyboard.press("Escape");
  check("Esc deseleziona", (await state()).selection.length === 0);

  // 9. barra spaziatrice: la vista si sposta, nessun riquadro
  const v0 = (await state()).view;
  await page.keyboard.down("Space");
  await page.mouse.move(stage.x + 900, stage.y + 600);
  await page.mouse.down();
  await page.mouse.move(stage.x + 800, stage.y + 560, { steps: 5 });
  check(
    "con lo spazio non compare il riquadro",
    (await page.locator('[data-testid="ec-marquee"]').count()) === 0,
  );
  await page.mouse.up();
  await page.keyboard.up("Space");
  const v1 = (await state()).view;
  check(
    "con lo spazio la vista si sposta",
    v1.x === v0.x - 100 && v1.y === v0.y - 40,
    JSON.stringify(v1),
  );
  check("con lo spazio la selezione non cambia", (await state()).selection.length === 0);

  // 10. rotella: Ctrl = zoom, senza = pan
  await page.mouse.move(stage.x + 600, stage.y + 400);
  await page.keyboard.down("Control");
  await page.mouse.wheel(0, -200);
  await page.keyboard.up("Control");
  await page.waitForTimeout(100);
  const v2 = (await state()).view;
  check("Ctrl+rotella ingrandisce", v2.zoom > v1.zoom, `${v1.zoom} → ${v2.zoom}`);
  await page.mouse.wheel(0, 120);
  await page.waitForTimeout(100);
  check("rotella semplice sposta la vista", (await state()).view.y === v2.y - 120);

  await page.getByRole("button", { name: "Adatta" }).click();
  await page.waitForTimeout(200);

  // 11. Canc su nodo collegato: conferma con anteprima, poi eliminazione
  c = await centerOf("op-filter");
  await page.mouse.click(c.x, c.y);
  await page.keyboard.press("Delete");
  check(
    "Canc su nodo collegato apre la conferma",
    (await page.locator('[data-testid="ec-confirm"]').count()) === 1,
  );
  check(
    "i nodi che sparirebbero sono evidenziati",
    (await page.locator(".ec-doomed").count()) >= 1,
  );
  check("nessuna eliminazione prima di confermare", !!(await state()).graph.cards["op-filter"]);
  await shot("conferma-eliminazione.png");
  await page.getByRole("button", { name: "Annulla" }).click();
  check(
    "Annulla chiude la conferma",
    (await page.locator('[data-testid="ec-confirm"]').count()) === 0 &&
      !!(await state()).graph.cards["op-filter"],
  );
  await page.keyboard.press("Delete");
  await page.getByRole("button", { name: "Elimina" }).click();
  check("Elimina rimuove il nodo", !(await state()).graph.cards["op-filter"]);

  // 12. duplica e seleziona tutto
  c = await centerOf("ds1");
  await page.mouse.click(c.x, c.y);
  const n0 = Object.keys((await state()).graph.cards).length;
  await page.keyboard.press("Control+d");
  check("Ctrl+D duplica", Object.keys((await state()).graph.cards).length === n0 + 1);
  await page.keyboard.press("Control+a");
  s = await state();
  check("Ctrl+A seleziona tutto", s.selection.length === Object.keys(s.graph.cards).length);

  // 13. nessun errore in console
  check("nessun errore in console", errors.length === 0, errors.join(" | ").slice(0, 300));
} catch (e) {
  check("eccezione nello script", false, String(e).slice(0, 200));
} finally {
  await browser.close();
  server.stop();
}
console.table(results);
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

