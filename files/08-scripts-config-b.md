# 08-scripts-config-b.md

File in questo blocco:

- `scripts/extract-golden.mjs`
- `scripts/generate-index.mjs`
- `scripts/generate-snapshot.mjs`
- `scripts/sync-snapshot.sh`
- `scripts/theme-map.mjs`
- `scripts/token-legacy-files.txt`
- `scripts/visual-compare.mjs`

---

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

### `scripts/generate-snapshot.mjs`

528 righe

```js
#!/usr/bin/env node
// Generates the content of the public isa-etl-snapshot repo (everything
// except INDEX.md, which needs the snapshot repo's commit SHA and is
// generated afterwards by sync-snapshot.sh).
//
// Output directory layout (written under --out <dir>):
//   STATUS.md            (pre-generated by sync-snapshot.sh, copied in)
//   ENV.md
//   TREE.md
//   files/NN-<area>.md
//   reports/*.md
//   manifest.json         (for sync-snapshot.sh + validation: block sizes,
//                          per-block file lists, secrets redacted, files
//                          excluded)
//
// This script never touches git or the network. It only reads the source
// repo's working tree and writes plain files to --out.

import { readFileSync, writeFileSync, mkdirSync, statSync, readdirSync } from "node:fs";
import { join, relative, extname, basename } from "node:path";
import { execFileSync } from "node:child_process";

const REPO_ROOT = process.cwd();
const args = process.argv.slice(2);
const outIdx = args.indexOf("--out");
if (outIdx === -1) {
  console.error("Usage: generate-snapshot.mjs --out <dir>");
  process.exit(1);
}
const OUT_DIR = args[outIdx + 1];

// "60 KB" is treated as 60000 bytes (SI, not KiB) to avoid any ambiguity
// with external tooling that measures size in decimal kilobytes; the
// hard cap is checked against this, while packing targets stay well
// under it for headroom.
const BLOCK_LIMIT_BYTES = 60000;

// ---------------------------------------------------------------------------
// Exclusion rules
// ---------------------------------------------------------------------------

const EXCLUDE_DIR_NAMES = new Set([
  "node_modules",
  ".git",
  "dist",
  "build",
  "coverage",
  ".output",
  ".wrangler",
  ".pw-tmp",
  ".nitro",
  ".vinxi",
]);

const BINARY_EXT = new Set([
  ".ico",
  ".png",
  ".jpg",
  ".jpeg",
  ".gif",
  ".webp",
  ".woff",
  ".woff2",
  ".ttf",
  ".eot",
  ".otf",
]);

const LOCKFILE_NAMES = new Set(["package-lock.json", "bun.lock", "yarn.lock", "pnpm-lock.yaml"]);

function isEnvFile(name) {
  return name === ".env" || name.startsWith(".env.");
}

// ---------------------------------------------------------------------------
// Walk the whole repo (for TREE.md) while respecting excluded dirs.
// ---------------------------------------------------------------------------

function walk(dir, out) {
  for (const entry of readdirSync(dir, { withFileTypes: true })) {
    if (entry.name === basename(OUT_DIR) && dir === REPO_ROOT) continue; // never scan our own scratch out dir if inside repo
    const full = join(dir, entry.name);
    const rel = relative(REPO_ROOT, full);
    if (entry.isDirectory()) {
      if (EXCLUDE_DIR_NAMES.has(entry.name)) continue;
      walk(full, out);
    } else if (entry.isFile()) {
      out.push(rel);
    }
  }
}

const allFiles = [];
walk(REPO_ROOT, allFiles);
allFiles.sort();

function classify(rel) {
  const name = basename(rel);
  const ext = extname(name);
  if (isEnvFile(name)) return "env";
  if (LOCKFILE_NAMES.has(name)) return "lockfile";
  if (BINARY_EXT.has(ext)) return "binary";
  return "text";
}

// ---------------------------------------------------------------------------
// In-scope text files for files/NN-<area>.md content blocks.
// Deliberately explicit allowlist (not "every text file everywhere") --
// see task instructions: src/**, scripts/**, docs/**, test, root configs,
// README, plus a short list of small project-config files that live
// outside those roots.
// ---------------------------------------------------------------------------

const ROOT_ALLOWLIST = new Set([
  "package.json",
  "tsconfig.json",
  "vite.config.ts",
  "vitest.config.ts",
  "eslint.config.js",
  "components.json",
  ".prettierrc",
  ".prettierignore",
  "bunfig.toml",
  ".gitignore",
  "README.md",
  "AGENTS.md",
  "roadmap.md",
]);

const EXTRA_CONFIG_FILES = new Set([
  ".devcontainer/devcontainer.json",
  ".vscode/settings.json",
  ".claude/settings.local.json",
]);

function isInScopeTextFile(rel) {
  if (classify(rel) !== "text") return false;
  if (rel.startsWith("src/") || rel.startsWith("scripts/") || rel.startsWith("docs/")) return true;
  if (!rel.includes("/") && ROOT_ALLOWLIST.has(rel)) return true;
  if (EXTRA_CONFIG_FILES.has(rel)) return true;
  return false;
}

// src/canvas/.reports/*.md are copied to reports/, not files/.
function isReportFile(rel) {
  return rel.startsWith("src/canvas/.reports/") && rel.endsWith(".md");
}

const inScopeFiles = allFiles.filter((f) => isInScopeTextFile(f) && !isReportFile(f));
const reportFiles = allFiles.filter(isReportFile);

// ---------------------------------------------------------------------------
// Secret scanning + redaction
// ---------------------------------------------------------------------------

const SECRET_PATTERNS = [
  { name: "GitHub token", re: /\b(ghp|gho|ghu|ghs|ghr|github_pat)_[A-Za-z0-9_]{20,}\b/g },
  { name: "OpenAI/Anthropic key", re: /\bsk-(ant-)?[A-Za-z0-9_-]{20,}\b/g },
  { name: "AWS access key", re: /\bAKIA[0-9A-Z]{16}\b/g },
  { name: "Slack token", re: /\bxox[baprs]-[A-Za-z0-9-]{10,}\b/g },
  {
    name: "PEM private key",
    re: /-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----/g,
  },
  { name: "credentialed URL", re: /\b[a-z]+:\/\/[^\s\/:@]+:[^\s\/:@]+@[^\s"'>]+/gi },
  {
    name: "secret-like assignment",
    re: /((?:password|secret|api[_-]?key|token|access[_-]?key)\s*[:=]\s*)(["'`]?)([^\s"'`,;]{6,})(\2)/gi,
    replaceGroup: 3,
  },
];

const redactionLog = [];

function scanAndRedact(content, rel) {
  let redacted = content;
  for (const pat of SECRET_PATTERNS) {
    redacted = redacted.replace(pat.re, (match, ...groups) => {
      redactionLog.push({ file: rel, pattern: pat.name });
      if (pat.replaceGroup) {
        // groups: (prefix, quote, value, quote2) via capture groups
        const prefix = groups[0];
        const quote = groups[1];
        return `${prefix}${quote}[REDATTO]${quote}`;
      }
      return "[REDATTO]";
    });
  }
  return redacted;
}

// ---------------------------------------------------------------------------
// Area bucketing
// ---------------------------------------------------------------------------

function areaFor(rel) {
  if (rel.startsWith("src/canvas/")) return "01-canvas";
  if (rel.startsWith("src/etl-core/")) return "01b-etl-core";
  if (rel.startsWith("src/etl-layout/")) return "01c-etl-layout";
  if (rel.startsWith("src/etl-store/")) return "01d-etl-store";
  if (rel.startsWith("src/etl-canvas/")) return "01e-etl-canvas";
  if (rel.startsWith("src/components/isa/etl/")) return "02-isa-etl";
  if (rel.startsWith("src/components/isa/") || rel.startsWith("src/components/ui/"))
    return "03-components";
  if (rel.startsWith("src/lib/") || rel.startsWith("src/hooks/")) return "04-lib-hooks-store";
  if (
    rel.startsWith("src/routes/") ||
    rel === "src/router.tsx" ||
    rel === "src/server.ts" ||
    rel === "src/start.ts" ||
    rel === "src/routeTree.gen.ts"
  )
    return "05-app-pages";
  if (rel === "src/styles.css") return "06-styles";
  if (
    rel.startsWith("scripts/") ||
    (ROOT_ALLOWLIST.has(basename(rel)) &&
      basename(rel) !== "README.md" &&
      basename(rel) !== "AGENTS.md" &&
      basename(rel) !== "roadmap.md") ||
    EXTRA_CONFIG_FILES.has(rel)
  ) {
    if (rel === "README.md" || rel === "AGENTS.md" || rel === "roadmap.md") return "09-docs";
    return "08-scripts-config";
  }
  if (rel === "README.md" || rel === "AGENTS.md" || rel === "roadmap.md") return "09-docs";
  if (rel.startsWith("docs/prototype/")) return "10-prototype";
  if (rel.startsWith("docs/inventory/")) return "11-inventory";
  if (rel.startsWith("docs/")) return "12-docs-other";
  return "13-misc";
}

const langForExt = {
  ".ts": "ts",
  ".tsx": "tsx",
  ".js": "js",
  ".jsx": "jsx",
  ".mjs": "js",
  ".cjs": "js",
  ".css": "css",
  ".json": "json",
  ".sh": "sh",
  ".md": "md",
  ".toml": "toml",
  ".yml": "yaml",
  ".yaml": "yaml",
  ".html": "html",
};
function langFor(rel) {
  return langForExt[extname(rel)] || "";
}

// Group files by area, preserving sorted order within each area.
const byArea = new Map();
for (const f of inScopeFiles) {
  const area = areaFor(f);
  if (!byArea.has(area)) byArea.set(area, []);
  byArea.get(area).push(f);
}

// ---------------------------------------------------------------------------
// Render one file's markdown section; may be split into parts if the file
// alone exceeds BLOCK_LIMIT_BYTES.
// ---------------------------------------------------------------------------

function renderFileSection(rel) {
  const raw = readFileSync(join(REPO_ROOT, rel), "utf8");
  const content = scanAndRedact(raw, rel);
  const lines = content.split("\n").length;
  const lang = langFor(rel);
  const header = `### \`${rel}\`\n\n${lines} righe\n\n`;
  const body = "```" + lang + "\n" + content + (content.endsWith("\n") ? "" : "\n") + "```\n\n";
  return header + body;
}

// Target size for a single file-section (or one part of a split file).
// Kept well under the 60KB hard block limit so that block-level wrapper
// overhead (title, per-block file list) never pushes a written block
// over the limit.
const PART_TARGET_BYTES = 50000;

// Split a single oversized file-section into consecutive parts, each
// under PART_TARGET_BYTES, cutting on line boundaries.
function renderFileSectionParts(rel) {
  const raw = readFileSync(join(REPO_ROOT, rel), "utf8");
  const content = scanAndRedact(raw, rel);
  const lines = content.split("\n");
  const lang = langFor(rel);
  const totalLines = lines.length;
  const fenceOverhead = ("```" + lang + "\n").length + "```\n\n".length + 250; // header + fences

  const parts = [];
  let cur = [];
  let curBytes = 0;
  for (const line of lines) {
    const lb = Buffer.byteLength(line + "\n", "utf8");
    if (curBytes + lb > PART_TARGET_BYTES - fenceOverhead && cur.length > 0) {
      parts.push(cur);
      cur = [];
      curBytes = 0;
    }
    cur.push(line);
    curBytes += lb;
  }
  if (cur.length > 0) parts.push(cur);

  const total = parts.length;
  return parts.map((partLines, i) => {
    const header = `### \`${rel}\` (parte ${i + 1}/${total})\n\n${totalLines} righe totali\n\n`;
    const body = "```" + lang + "\n" + partLines.join("\n") + "\n```\n\n";
    return header + body;
  });
}

// ---------------------------------------------------------------------------
// Pack an area's files into <=60KB blocks, splitting oversized single
// files into parts, and splitting areas into -a/-b/... suffixed files.
// ---------------------------------------------------------------------------

const manifest = { blocks: [], excluded: [], redactions: redactionLog, areas: {} };

// Exact wrapper bytes for a block containing `files` (block title length
// is fixed-ish regardless of the a/b suffix, so this estimate is exact
// enough -- verified against the real written size below anyway).
function wrapperBytes(area, files) {
  const fileList = files.map((f) => `- \`${f}\``).join("\n");
  const wrapper = `# ${area}-x.md\n\nFile in questo blocco:\n\n${fileList}\n\n---\n\n`;
  return Buffer.byteLength(wrapper, "utf8");
}

function packArea(area, files) {
  const sections = []; // { rel, text, bytes }
  for (const rel of files) {
    const full = join(REPO_ROOT, rel);
    const size = statSync(full).size;
    if (size > PART_TARGET_BYTES) {
      for (const partText of renderFileSectionParts(rel)) {
        sections.push({ rel, text: partText, bytes: Buffer.byteLength(partText, "utf8") });
      }
    } else {
      const text = renderFileSection(rel);
      sections.push({ rel, text, bytes: Buffer.byteLength(text, "utf8") });
    }
  }

  // Greedily pack sections into blocks, keeping order, checking the exact
  // final wrapped size (including the per-block file list) against the
  // hard 60KB limit before committing each addition.
  const blocks = [];
  let curFiles = [];
  let curText = "";
  let curBytes = 0;
  for (const sec of sections) {
    const tentativeFiles = [...new Set([...curFiles, sec.rel])];
    const tentativeBytes = curBytes + sec.bytes + wrapperBytes(area, tentativeFiles);
    if (curFiles.length > 0 && tentativeBytes > BLOCK_LIMIT_BYTES - 500) {
      blocks.push({ files: curFiles, text: curText, bytes: curBytes });
      curFiles = [];
      curText = "";
      curBytes = 0;
    }
    curFiles.push(sec.rel);
    curText += sec.text;
    curBytes += sec.bytes;
  }
  if (curText.length > 0) blocks.push({ files: curFiles, text: curText, bytes: curBytes });

  return blocks;
}

mkdirSync(join(OUT_DIR, "files"), { recursive: true });
mkdirSync(join(OUT_DIR, "reports"), { recursive: true });

const areaKeys = [...byArea.keys()].sort();
for (const area of areaKeys) {
  const files = byArea.get(area);
  const blocks = packArea(area, files);
  const suffixes =
    blocks.length > 1 ? "abcdefghijklmnopqrstuvwxyz".slice(0, blocks.length).split("") : [""];
  const outNames = [];
  blocks.forEach((block, i) => {
    const suffix = blocks.length > 1 ? `-${suffixes[i]}` : "";
    const name = `${area}${suffix}.md`;
    outNames.push(name);
    const uniqueFiles = [...new Set(block.files)];
    const fileList = uniqueFiles.map((f) => `- \`${f}\``).join("\n");
    const content = `# ${name}\n\nFile in questo blocco:\n\n${fileList}\n\n---\n\n${block.text}`;
    const contentBytes = Buffer.byteLength(content, "utf8");
    if (contentBytes > BLOCK_LIMIT_BYTES) {
      throw new Error(
        `Block ${name} is ${contentBytes} bytes, over the ${BLOCK_LIMIT_BYTES}-byte limit`,
      );
    }
    writeFileSync(join(OUT_DIR, "files", name), content, "utf8");
    manifest.blocks.push({
      name: `files/${name}`,
      bytes: contentBytes,
      files: uniqueFiles,
    });
  });
  manifest.areas[area] = { files, outFiles: outNames };
}

// Files that exist on disk but were not in scope for files/ (for the
// dedup/coverage check) -- excludes reports (handled separately) and
// binaries/env/lockfiles which are TREE-only by design.
for (const f of allFiles) {
  if (isReportFile(f)) continue;
  const cls = classify(f);
  if (cls !== "text") {
    manifest.excluded.push({ file: f, reason: cls });
    continue;
  }
  if (!isInScopeTextFile(f)) {
    manifest.excluded.push({ file: f, reason: "out-of-scope-dir" });
  }
}

// ---------------------------------------------------------------------------
// reports/*.md -- copy verbatim (still scan for secrets defensively)
// ---------------------------------------------------------------------------

for (const rel of reportFiles) {
  const raw = readFileSync(join(REPO_ROOT, rel), "utf8");
  const redacted = scanAndRedact(raw, rel);
  writeFileSync(join(OUT_DIR, "reports", basename(rel)), redacted, "utf8");
}
manifest.reportFiles = reportFiles;

// ---------------------------------------------------------------------------
// TREE.md -- full tree (excluding heavy/junk dirs), line counts for text
// files, size-only for binaries/lockfiles.
// ---------------------------------------------------------------------------

function countLines(full) {
  try {
    const content = readFileSync(full, "utf8");
    return content.split("\n").length;
  } catch {
    return null;
  }
}

let treeLines = [
  "# TREE.md",
  "",
  "Albero completo (esclusi node_modules, dist, build, coverage, .git, cache di build).",
  "",
];
for (const f of allFiles) {
  const full = join(REPO_ROOT, f);
  const cls = classify(f);
  const size = statSync(full).size;
  if (cls === "text") {
    const lines = countLines(full);
    treeLines.push(`- \`${f}\` — ${lines} righe (${size} B)`);
  } else {
    treeLines.push(`- \`${f}\` — ${cls}, ${size} B`);
  }
}
writeFileSync(join(OUT_DIR, "TREE.md"), treeLines.join("\n") + "\n", "utf8");

// ---------------------------------------------------------------------------
// ENV.md
// ---------------------------------------------------------------------------

function tryRead(rel) {
  try {
    return readFileSync(join(REPO_ROOT, rel), "utf8");
  } catch {
    return null;
  }
}

const envParts = ["# ENV.md", ""];
const envFiles = [
  "package.json",
  "tsconfig.json",
  "vite.config.ts",
  "vitest.config.ts",
  "eslint.config.js",
  "components.json",
  ".prettierrc",
  ".prettierignore",
  "src/styles.css",
];
for (const rel of envFiles) {
  const content = tryRead(rel);
  if (content == null) {
    envParts.push(`## \`${rel}\`\n\n_Assente nel repository._\n`);
    continue;
  }
  const lang = langFor(rel) || "text";
  envParts.push(`## \`${rel}\`\n\n\`\`\`${lang}\n${scanAndRedact(content, rel)}\n\`\`\`\n`);
}
let nodeVersion = process.version;
let npmVersion = "?";
try {
  npmVersion = execFileSync("npm", ["--version"], { encoding: "utf8" }).trim();
} catch {
  npmVersion = "(comando npm non disponibile)";
}
envParts.push(`## Versioni runtime\n\n- node: ${nodeVersion}\n- npm: ${npmVersion}\n`);

writeFileSync(join(OUT_DIR, "ENV.md"), envParts.join("\n") + "\n", "utf8");

// ---------------------------------------------------------------------------
// manifest.json
// ---------------------------------------------------------------------------

manifest.inScopeFileCount = inScopeFiles.length;
manifest.reportFileCount = reportFiles.length;
writeFileSync(join(OUT_DIR, "manifest.json"), JSON.stringify(manifest, null, 2), "utf8");

console.log(
  JSON.stringify(
    {
      blocks: manifest.blocks.length,
      inScopeFiles: inScopeFiles.length,
      reportFiles: reportFiles.length,
      redactions: redactionLog.length,
      excluded: manifest.excluded.length,
    },
    null,
    2,
  ),
);
```

### `scripts/sync-snapshot.sh`

238 righe

```sh
#!/usr/bin/env bash
# Publish a full, verifiable text snapshot of this repository to a
# dedicated PUBLIC GitHub repo (micheleefrancoo/isa-etl-snapshot), so an
# external assistant can fetch every relevant file anonymously via
# raw.githubusercontent.com URLs written into INDEX.md (no pagination, no
# auth, no robots.txt block -- unlike gists).
#
# isa-glass-platform itself is PRIVATE, so its own raw URLs require an
# auth header a plain "paste this URL" workflow can't provide. That's why
# this pushes to a separate, dedicated public repo containing nothing but
# generated snapshot content, instead of a branch of this repo.
#
# This script never touches this repo's git state (no checkout/branch/
# stash/commit here) -- it only reads the working tree, runs read-only
# status checks, and pushes generated files to the snapshot repo.
#
# Usage: ./scripts/sync-snapshot.sh   (no arguments)
# Use the authenticated user's OAuth token, not the Codespaces
# GITHUB_TOKEN (which lacks access to the isa-etl-snapshot repo).
unset GITHUB_TOKEN
set -euo pipefail

SNAPSHOT_REPO_NAME="isa-etl-snapshot"
SNAPSHOT_BRANCH="main"

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$REPO_ROOT"

WORK_DIR="$(mktemp -d)"
trap 'rm -rf "$WORK_DIR"' EXIT

TIMESTAMP="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"

ORIGIN_URL="$(git remote get-url origin)"
SNAPSHOT_OWNER="$(printf '%s' "$ORIGIN_URL" | sed -E 's#.*[:/]([^/]+)/[^/]+(\.git)?$#\1#')"
SNAPSHOT_REPO="$SNAPSHOT_OWNER/$SNAPSHOT_REPO_NAME"

SNAP_DIR="$WORK_DIR/snapshot"
mkdir -p "$SNAP_DIR"

echo "== Building STATUS.md (running checks; this photographs current state, fixes nothing) =="

STATUS_MD="$SNAP_DIR/STATUS.md"
{
  echo "# STATUS.md"
  echo
  echo "Generato: $TIMESTAMP (UTC)"
  echo
} > "$STATUS_MD"

run_check() {
  local title="$1"
  local note="$2"
  shift 2
  local out_file="$WORK_DIR/check_out.txt"
  local start end duration status exit_code

  echo "## $title" >> "$STATUS_MD"
  echo >> "$STATUS_MD"
  if [ -n "$note" ]; then
    echo "$note" >> "$STATUS_MD"
    echo >> "$STATUS_MD"
  fi
  echo "Comando: \`$*\`" >> "$STATUS_MD"
  echo >> "$STATUS_MD"

  start=$(date +%s)
  set +e
  "$@" > "$out_file" 2>&1
  exit_code=$?
  set -e
  end=$(date +%s)
  duration=$((end - start))

  if [ "$exit_code" -eq 0 ]; then
    status="OK (exit 0)"
  else
    status="FALLITO (exit $exit_code)"
  fi

  echo "Esito: $status" >> "$STATUS_MD"
  echo "Durata: ${duration}s" >> "$STATUS_MD"
  echo >> "$STATUS_MD"
  echo "Ultime 60 righe di output:" >> "$STATUS_MD"
  echo '```' >> "$STATUS_MD"
  tail -n 60 "$out_file" >> "$STATUS_MD"
  echo '```' >> "$STATUS_MD"
  echo >> "$STATUS_MD"
}

run_check "Type check" "Nessuno script \"typecheck\" in package.json: eseguito il comando diretto." npx tsc --noEmit
run_check "Lint (npm run lint)" "" npm run lint
run_check "Test (npm test / vitest run)" "" npm test
run_check "Build (npm run build)" "" npm run build
run_check "Disciplina dei token" "Colori, raggi e ombre letterali nei file controllati (scripts/check-tokens.mjs)." node scripts/check-tokens.mjs

{
  echo "## Git log (ultimi 20 commit)"
  echo
  echo '```'
  git log --oneline -20
  echo '```'
  echo
  echo "## Branch"
  echo
  echo '```'
  git branch -a
  echo '```'
  echo
} >> "$STATUS_MD"

OTHER_BRANCHES="$(git for-each-ref --format='%(refname:short)' refs/heads/ | grep -v '^main$' || true)"
if [ -z "$OTHER_BRANCHES" ]; then
  {
    echo "## Branch diversi da main"
    echo
    echo "Nessun branch locale diverso da \`main\`."
    echo
  } >> "$STATUS_MD"
else
  {
    echo "## Branch diversi da main"
    echo
  } >> "$STATUS_MD"
  while IFS= read -r b; do
    [ -z "$b" ] && continue
    {
      echo "### \`$b\`"
      echo
      echo "Ultimo commit:"
      echo '```'
      git log -1 --oneline "$b"
      echo '```'
      echo
      echo "Diff stat rispetto a main:"
      echo '```'
      git diff --stat "main...$b"
      echo '```'
      echo
    } >> "$STATUS_MD"
  done <<< "$OTHER_BRANCHES"
fi

echo "== Generating ENV.md, TREE.md, files/, reports/ =="
node scripts/generate-snapshot.mjs --out "$SNAP_DIR" | tee "$WORK_DIR/generate-summary.json"

echo "== Writing legacy-link stubs =="
STUB_URL="https://raw.githubusercontent.com/$SNAPSHOT_REPO/$SNAPSHOT_BRANCH/INDEX.md"
for legacy in isa-snapshot.md isa-snapshot-workflow-canvas.md; do
  cat > "$SNAP_DIR/$legacy" <<EOF
# Spostato

Questo file è stato sostituito da un indice unico e più completo.

Vai a **[INDEX.md]($STUB_URL)** per l'elenco di tutti i file e i relativi URL raw fissati al commit.
EOF
done

echo "== Cloning $SNAPSHOT_REPO =="
CLONE_DIR="$WORK_DIR/repo"
if ! gh repo clone "$SNAPSHOT_REPO" "$CLONE_DIR" -- --depth 1 --quiet 2>"$WORK_DIR/clone_err.txt"; then
  cat "$WORK_DIR/clone_err.txt" >&2
  exit 1
fi

echo "== Syncing generated content into clone (everything except INDEX.md) =="
# Preserve the previous run's INDEX.md (if any) untouched across the
# wipe-and-repopulate below, so the first commit's diff never includes it
# -- it is only ever touched by the second commit, after the new SHA is
# known.
PREV_INDEX="$WORK_DIR/prev_INDEX.md"
if [ -f "$CLONE_DIR/INDEX.md" ]; then
  cp "$CLONE_DIR/INDEX.md" "$PREV_INDEX"
fi

# Remove everything in the clone except .git, then repopulate from the
# freshly generated snapshot, so stale files from a previous run's layout
# never linger.
find "$CLONE_DIR" -mindepth 1 -maxdepth 1 ! -name ".git" -exec rm -rf {} +
cp -R "$SNAP_DIR"/. "$CLONE_DIR"/
# manifest.json travels with the repo too -- useful for anyone re-running
# validation later, and costs nothing (small, no secrets: scanned above).

if [ -f "$PREV_INDEX" ]; then
  cp "$PREV_INDEX" "$CLONE_DIR/INDEX.md"
fi

cd "$CLONE_DIR"
git add -A
git status --porcelain > "$WORK_DIR/first_commit_status.txt"

if [ -s "$WORK_DIR/first_commit_status.txt" ]; then
  git commit -q -m "chore: update snapshot $TIMESTAMP"
  git push -q origin "$SNAPSHOT_BRANCH"
  echo "Pushed content commit."
else
  echo "No content changes since last run."
fi

SHA="$(git rev-parse HEAD)"
echo "Snapshot content commit SHA: $SHA"

cd "$REPO_ROOT"

echo "== Generating INDEX.md pinned to $SHA =="
SOURCE_SHA="$(git rev-parse HEAD)"
SOURCE_BRANCH="$(git rev-parse --abbrev-ref HEAD)"
DIRTY_FILES="$(git status --porcelain | awk '{ $1=""; print substr($0,2) }')"

node scripts/generate-index.mjs \
  --manifest "$SNAP_DIR/manifest.json" \
  --sha "$SHA" \
  --repo "$SNAPSHOT_REPO" \
  --branch "$SOURCE_BRANCH" \
  --source-sha "$SOURCE_SHA" \
  --dirty-files "$DIRTY_FILES" \
  --generated-at "$TIMESTAMP" \
  --out "$CLONE_DIR/INDEX.md"

cd "$CLONE_DIR"
git add INDEX.md
if ! git diff --cached --quiet; then
  git commit -q -m "chore: update INDEX.md $TIMESTAMP"
  git push -q origin "$SNAPSHOT_BRANCH"
  echo "Pushed INDEX.md commit."
else
  echo "INDEX.md unchanged."
fi

FINAL_SHA="$(git rev-parse HEAD)"
cd "$REPO_ROOT"

echo
echo "== Done =="
echo "INDEX.md (branch $SNAPSHOT_BRANCH, moving target): https://raw.githubusercontent.com/$SNAPSHOT_REPO/$SNAPSHOT_BRANCH/INDEX.md"
echo "INDEX.md (fissato al commit $FINAL_SHA di questo run) -- ultima riga, sempre stampata:"
echo "https://raw.githubusercontent.com/$SNAPSHOT_REPO/$FINAL_SHA/INDEX.md"
```

### `scripts/theme-map.mjs`

96 righe

```js
#!/usr/bin/env node
/**
 * Mappa dei token dei temi: token semantico → valore, per ogni tema e modo.
 * Legge src/theme/primitives.css e src/theme/themes/*.css e riscrive la
 * sezione compresa tra i marcatori di src/theme/README.md.
 *
 *   node scripts/theme-map.mjs --write   aggiorna il README
 *   node scripts/theme-map.mjs           stampa la mappa
 * Il test src/theme/__tests__/readme.test.ts verifica che il README sia aggiornato.
 */
import { readFileSync, writeFileSync } from "node:fs";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

const ROOT = resolve(dirname(fileURLToPath(import.meta.url)), "..");
export const README = resolve(ROOT, "src/theme/README.md");
export const BEGIN =
  "<!-- BEGIN token-map (generata da scripts/theme-map.mjs: non modificare a mano) -->";
export const END = "<!-- END token-map -->";

const read = (p) => readFileSync(resolve(ROOT, p), "utf8");

function blocks(css) {
  const clean = css.replace(/\/\*[\s\S]*?\*\//g, "");
  const out = [];
  for (const m of clean.matchAll(/([^{}]+)\{([^{}]*)\}/g)) {
    const decls = {};
    for (const d of m[2].matchAll(/(--[a-z0-9-]+)\s*:\s*([^;]+);/g))
      decls[d[1]] = d[2].trim().replace(/\s+/g, " ");
    out.push({ selector: m[1].trim().replace(/\s+/g, " "), decls });
  }
  return out;
}

const decls = (css, sel) =>
  Object.assign(
    {},
    ...blocks(css)
      .filter((b) => b.selector === sel)
      .map((b) => b.decls),
  );

export function buildTokenMap() {
  const primCss = read("src/theme/primitives.css");
  const hexOf = {};
  for (const m of primCss.matchAll(/(--isa-p-[a-z0-9-]+):[^;]+;\s*\/\*\s*(#[0-9a-f]{6})/g))
    hexOf[m[1]] = m[2];
  const proto = read("src/theme/themes/prototipo.css");
  const notte = read("src/theme/themes/notte.css");
  const cols = {
    "prototipo chiaro": decls(proto, ":root"),
    "prototipo scuro": { ...decls(proto, ":root"), ...decls(proto, ".dark") },
    "notte chiaro": decls(notte, ':root[data-theme="notte"]'),
    "notte scuro": {
      ...decls(notte, ':root[data-theme="notte"]'),
      ...decls(notte, ':root.dark[data-theme="notte"]'),
    },
  };
  const names = Object.keys(decls(proto, ":root"));
  const show = (v) => {
    if (v === undefined) return "—";
    const m = /^var\((--isa-p-[a-z0-9-]+)\)$/.exec(v);
    const text = m && hexOf[m[1]] ? `${v} (${hexOf[m[1]]})` : v;
    return "`" + text.replace(/\|/g, "\\|") + "`";
  };
  const groups = [
    ["Token dell'app (livello semantico, famiglia shadcn)", (n) => !n.startsWith("--isa-")],
    ["Token semantici condivisi (`--isa-*`)", (n) => n.startsWith("--isa-")],
  ];
  const head = Object.keys(cols);
  let md = "";
  for (const [title, pick] of groups) {
    md += `#### ${title}\n\n| Token | ${head.join(" | ")} |\n|---|${head.map(() => "---").join("|")}|\n`;
    for (const n of names.filter(pick))
      md += `| \`${n}\` | ${head.map((h) => show(cols[h][n])).join(" | ")} |\n`;
    md += "\n";
  }
  return { markdown: md.trimEnd() + "\n", count: names.length, columns: head };
}

export function renderReadmeSection() {
  return `${BEGIN}\n\n${buildTokenMap().markdown}\n${END}`;
}

if (process.argv[1] === fileURLToPath(import.meta.url)) {
  if (process.argv.includes("--write")) {
    const text = readFileSync(README, "utf8");
    const a = text.indexOf(BEGIN);
    const b = text.indexOf(END);
    if (a < 0 || b < 0) throw new Error("marcatori non trovati in src/theme/README.md");
    writeFileSync(README, text.slice(0, a) + renderReadmeSection() + text.slice(b + END.length));
  } else {
    console.log(buildTokenMap().markdown);
  }
}
```

### `scripts/token-legacy-files.txt`

169 righe

```
src/canvas/__tests__/panelPositioning.test.ts
src/canvas/components/CanvasContainer.tsx
src/canvas/hooks/useCanvasBounds.ts
src/canvas/hooks/usePanelState.ts
src/canvas/layout/__tests__/canvasBounds.test.ts
src/canvas/layout/__tests__/dropZones.test.ts
src/canvas/layout/__tests__/panelRegistry.test.ts
src/canvas/layout/canvasBounds.ts
src/canvas/layout/dropZones.ts
src/canvas/layout/panelRegistry.ts
src/canvas/layout/surfacePanels.ts
src/canvas/store/canvasStore.tsx
src/components/isa/app-shell.tsx
src/components/isa/back-button.tsx
src/components/isa/etl/data-preview.tsx
src/components/isa/etl/inspector.tsx
src/components/isa/etl/isa-context-menu.tsx
src/components/isa/etl/settings-panels/aggregate-panel.tsx
src/components/isa/etl/settings-panels/combine-panel.tsx
src/components/isa/etl/settings-panels/filter-panel.tsx
src/components/isa/etl/settings-panels/panel-controls.tsx
src/components/isa/etl/tool-palette.tsx
src/components/isa/etl/workflow-canvas.tsx
src/components/isa/header.tsx
src/components/isa/logo.tsx
src/components/isa/mini-chart.tsx
src/components/isa/module-picker-modal.tsx
src/components/isa/new-solution-modal.tsx
src/components/isa/share-modal.tsx
src/components/isa/sidebar.tsx
src/components/isa/solution-card.tsx
src/components/isa/solution-row.tsx
src/components/isa/ui/isa-menu.tsx
src/components/isa/ui/isa-modal.tsx
src/components/isa/widget-panel.tsx
src/components/ui/accordion.tsx
src/components/ui/alert-dialog.tsx
src/components/ui/alert.tsx
src/components/ui/aspect-ratio.tsx
src/components/ui/avatar.tsx
src/components/ui/badge.tsx
src/components/ui/breadcrumb.tsx
src/components/ui/button.tsx
src/components/ui/calendar.tsx
src/components/ui/card.tsx
src/components/ui/carousel.tsx
src/components/ui/chart.tsx
src/components/ui/checkbox.tsx
src/components/ui/collapsible.tsx
src/components/ui/command.tsx
src/components/ui/context-menu.tsx
src/components/ui/dialog.tsx
src/components/ui/drawer.tsx
src/components/ui/dropdown-menu.tsx
src/components/ui/form.tsx
src/components/ui/hover-card.tsx
src/components/ui/input-otp.tsx
src/components/ui/input.tsx
src/components/ui/label.tsx
src/components/ui/menubar.tsx
src/components/ui/navigation-menu.tsx
src/components/ui/pagination.tsx
src/components/ui/popover.tsx
src/components/ui/progress.tsx
src/components/ui/radio-group.tsx
src/components/ui/resizable.tsx
src/components/ui/scroll-area.tsx
src/components/ui/select.tsx
src/components/ui/separator.tsx
src/components/ui/sheet.tsx
src/components/ui/sidebar.tsx
src/components/ui/skeleton.tsx
src/components/ui/slider.tsx
src/components/ui/sonner.tsx
src/components/ui/switch.tsx
src/components/ui/table.tsx
src/components/ui/tabs.tsx
src/components/ui/textarea.tsx
src/components/ui/toggle-group.tsx
src/components/ui/toggle.tsx
src/components/ui/tooltip.tsx
src/etl-core/__tests__/csv.test.ts
src/etl-core/__tests__/expressions.test.ts
src/etl-core/__tests__/fase11-requisiti.test.ts
src/etl-core/__tests__/fase11.test.ts
src/etl-core/__tests__/helpers.ts
src/etl-core/__tests__/mutations.test.ts
src/etl-core/__tests__/params.test.ts
src/etl-core/__tests__/relations.test.ts
src/etl-core/__tests__/schema.test.ts
src/etl-core/__tests__/state.test.ts
src/etl-core/catalog/icons.ts
src/etl-core/catalog/operations.ts
src/etl-core/catalog/params.ts
src/etl-core/data/csv.ts
src/etl-core/index.ts
src/etl-core/logic/expressions.ts
src/etl-core/model/graph.ts
src/etl-core/model/types.ts
src/etl-core/rules/mutations.ts
src/etl-core/rules/relations.ts
src/etl-core/rules/state.ts
src/etl-core/schema/schema.ts
src/etl-layout/__tests__/golden.test.ts
src/etl-layout/__tests__/properties.test.ts
src/etl-layout/__tests__/unit.test.ts
src/etl-layout/autoLayout.ts
src/etl-layout/constants.ts
src/etl-layout/free.ts
src/etl-layout/hitTest.ts
src/etl-layout/index.ts
src/etl-layout/links.ts
src/etl-layout/nodes.ts
src/etl-layout/path.ts
src/etl-layout/placement.ts
src/etl-layout/routing.ts
src/etl-layout/slots.ts
src/etl-layout/types.ts
src/etl-store/__tests__/grouping.test.ts
src/etl-store/__tests__/helpers.ts
src/etl-store/__tests__/persistence.test.ts
src/etl-store/__tests__/react.test.ts
src/etl-store/__tests__/reduce.test.ts
src/etl-store/__tests__/store.test.ts
src/etl-store/derived.ts
src/etl-store/index.ts
src/etl-store/persistence.ts
src/etl-store/react.ts
src/etl-store/reduce.ts
src/etl-store/serialize.ts
src/etl-store/state.ts
src/etl-store/store.ts
src/etl-store/types.ts
src/hooks/use-mobile.tsx
src/lib/error-capture.ts
src/lib/error-page.ts
src/lib/etl-bubble.ts
src/lib/etl-catalog.ts
src/lib/etl-display.ts
src/lib/etl-motion.ts
src/lib/etl-node-config.ts
src/lib/etl-node-size.ts
src/lib/etl-schema.ts
src/lib/etl-workflow.tsx
src/lib/modules.ts
src/lib/solutions-store.tsx
src/lib/theme.tsx
src/lib/utils.ts
src/routeTree.gen.ts
src/router.tsx
src/routes/__root.tsx
src/routes/activity.tsx
src/routes/favorites.tsx
src/routes/index.tsx
src/routes/settings.tsx
src/routes/shared.tsx
src/routes/solutions.$solutionId.dashboard.tsx
src/routes/solutions.$solutionId.etl.tsx
src/routes/solutions.$solutionId.index.tsx
src/routes/solutions.$solutionId.model.tsx
src/routes/solutions.$solutionId.tsx
src/routes/teams.tsx
src/routes/templates.tsx
src/routes/trash.tsx
src/routes/users.tsx
src/server.ts
src/start.ts
src/styles.css
```

### `scripts/visual-compare.mjs`

99 righe

```js
#!/usr/bin/env node
/**
 * Confronto pixel per pixel tra le schermate nel working tree e quelle
 * committate in HEAD (o in un altro riferimento git). Nessuna dipendenza
 * nuova: le immagini si decodificano in Chromium (Playwright) con un canvas.
 *
 * Uso: node scripts/visual-compare.mjs [--ref HEAD] <file.png>[@x,y,w,h] ...
 *   `@x,y,w,h` limita il confronto a un rettangolo (per escludere il resto
 *   della pagina, per esempio l'intestazione dell'app).
 * Esce con codice 1 se una qualunque coppia differisce.
 */
import { execFileSync } from "node:child_process";
import { readFileSync } from "node:fs";
import { resolve } from "node:path";
import { chromium } from "playwright";
import { ROOT } from "./visual-lib.mjs";

const args = process.argv.slice(2);
let ref = "HEAD";
const items = [];
for (let i = 0; i < args.length; i++) {
  if (args[i] === "--ref") ref = args[++i];
  else items.push(args[i]);
}

const dataUrl = (buf) => "data:image/png;base64," + buf.toString("base64");
const browser = await chromium.launch();
const page = await browser.newPage();
let failed = 0;
const rows = [];
for (const item of items) {
  const [file, clip] = item.split("@");
  const now = readFileSync(resolve(ROOT, file));
  let old;
  try {
    old = execFileSync("git", ["show", `${ref}:${file}`], { cwd: ROOT, maxBuffer: 1 << 28 });
  } catch {
    rows.push({ file, esito: "assente in " + ref });
    failed++;
    continue;
  }
  const r = await page.evaluate(
    async ([a, b, clip]) => {
      const load = (src) =>
        new Promise((res, rej) => {
          const img = new Image();
          img.onload = () => res(img);
          img.onerror = rej;
          img.src = src;
        });
      const [ia, ib] = await Promise.all([load(a), load(b)]);
      if (ia.width !== ib.width || ia.height !== ib.height)
        return { size: [ia.width, ia.height, ib.width, ib.height] };
      const [x, y, w, h] = clip ? clip.split(",").map(Number) : [0, 0, ia.width, ia.height];
      const px = (img) => {
        const c = document.createElement("canvas");
        c.width = w;
        c.height = h;
        const ctx = c.getContext("2d");
        ctx.drawImage(img, x, y, w, h, 0, 0, w, h);
        return ctx.getImageData(0, 0, w, h).data;
      };
      const da = px(ia);
      const db = px(ib);
      let diff = 0;
      let max = 0;
      for (let i = 0; i < da.length; i += 4) {
        const d = Math.max(
          Math.abs(da[i] - db[i]),
          Math.abs(da[i + 1] - db[i + 1]),
          Math.abs(da[i + 2] - db[i + 2]),
          Math.abs(da[i + 3] - db[i + 3]),
        );
        if (d > 0) diff++;
        if (d > max) max = d;
      }
      return { pixels: w * h, diff, max };
    },
    [dataUrl(old), dataUrl(now), clip ?? null],
  );
  if (r.size) {
    rows.push({ file, esito: `dimensioni diverse ${r.size.slice(0, 2)} → ${r.size.slice(2)}` });
    failed++;
  } else {
    rows.push({
      file,
      regione: clip ?? "intera",
      pixel: r.pixels,
      differenti: r.diff,
      scartoMax: r.max,
    });
    if (r.diff > 0) failed++;
  }
}
await browser.close();
console.table(rows);
console.log(failed ? `DIFFERENZE in ${failed} file` : "TUTTE LE IMMAGINI COINCIDONO al pixel");
process.exit(failed ? 1 : 0);
```

