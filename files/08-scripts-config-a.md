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
- `scripts/e2e-fase6a.mjs`

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

299 righe

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
  // il canvas nudo ha i pannelli chiusi (Fase 6a: la cassetta si apre da sola): si chiudono nello store, senza compensare la vista e senza transizione
  await page.addStyleTag({ content: ".ec-workspace, .ec-panel { transition: none !important; }" });
  await page.evaluate(() =>
    window.__etlStore.dispatch({ type: "setPanel", payload: { panel: "tools", open: false } }),
  );
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
  await page.getByTestId("ec-confirm").getByRole("button", { name: "Annulla" }).click();
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
      (await page.locator('[data-testid="ec-inspector-name"]').innerText()) === "Unisci (Join)",
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

