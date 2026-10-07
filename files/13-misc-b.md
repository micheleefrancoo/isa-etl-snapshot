# 13-misc-b.md

File in questo blocco:

- `src/theme/__tests__/themes.test.ts`
- `src/theme/boot.ts`
- `src/theme/color.ts`
- `src/theme/derive.ts`
- `src/theme/index.css`
- `src/theme/index.ts`
- `src/theme/layout-tokens.css`
- `src/theme/primitives.css`
- `src/theme/runtime.ts`
- `src/theme/themes/notte.css`
- `src/theme/themes/prototipo.css`

---

### `src/theme/__tests__/themes.test.ts`

147 righe

```ts
import { describe, expect, it } from "vitest";
import { parseColor } from "../color";
import {
  MODES,
  PRIMITIVES_CSS,
  THEMES,
  THEME_FILES,
  parseBlocks,
  rawTokens,
  resolveTokens,
  selectorsOf,
} from "./support";

describe("struttura dei file CSS", () => {
  it("le primitive stanno solo in :root e hanno solo nomi --isa-p-*", () => {
    expect(selectorsOf(PRIMITIVES_CSS)).toEqual([":root"]);
    for (const b of parseBlocks(PRIMITIVES_CSS)) {
      for (const name of Object.keys(b.decls)) expect(name).toMatch(/^--isa-p-[a-z0-9-]+$/);
    }
  });

  it("le primitive sono in OKLCH", () => {
    for (const [name, value] of Object.entries(parseBlocks(PRIMITIVES_CSS)[0]!.decls)) {
      expect(value, name).toMatch(/^oklch\(/);
    }
  });

  it("il tema predefinito usa solo :root e .dark; gli altri solo :root[data-theme] (e .dark)", () => {
    expect(selectorsOf(THEME_FILES.prototipo)).toEqual([":root", ".dark"]);
    expect(selectorsOf(THEME_FILES.notte)).toEqual([
      ':root[data-theme="notte"]',
      ':root.dark[data-theme="notte"]',
    ]);
  });
});

describe("completezza dei temi", () => {
  const protoLight = new Set(Object.keys(parseBlocks(THEME_FILES.prototipo)[0]!.decls));
  const protoDark = new Set(Object.keys(parseBlocks(THEME_FILES.prototipo)[1]!.decls));

  it("«notte» assegna ogni token che assegna il tema predefinito, nei due modi", () => {
    const [light, dark] = parseBlocks(THEME_FILES.notte);
    expect(Object.keys(light!.decls).sort()).toEqual([...protoLight].sort());
    // il blocco scuro ridefinisce tutto ciò che cambia col modo nel predefinito
    for (const name of protoDark) expect(dark!.decls, name).toHaveProperty(name);
  });

  for (const theme of THEMES) {
    for (const mode of MODES) {
      it(`${theme}/${mode}: ogni var() si risolve`, () => {
        expect(() => resolveTokens(theme, mode)).not.toThrow();
      });
    }
  }

  it("il tema predefinito ha i ruoli richiesti dal sistema", () => {
    const t = resolveTokens("prototipo", "light");
    const roles = [
      "surface-base",
      "surface-raised",
      "surface-overlay",
      "text",
      "text-secondary",
      "text-muted",
      "text-on-accent",
      "border",
      "border-strong",
      "accent",
      "accent-active",
      "accent-soft",
      "focus-ring",
      "op-filter",
      "op-filter-soft",
      "op-transform",
      "op-transform-soft",
      "op-merge",
      "op-merge-soft",
      "op-output",
      "op-output-soft",
      "dataset-fill",
      "warning",
      "error",
      "success",
      "radius-control",
      "radius-panel",
      "shadow-glass",
      "blur-glass",
      "duration-fast",
      "duration-base",
      "duration-slow",
    ];
    for (const r of roles) expect(t, r).toHaveProperty(`--isa-${r}`);
  });
});

describe("«notte»: i colori richiesti", () => {
  const near = (value: string, hex: string) => {
    const got = parseColor(value);
    const want = parseColor(hex);
    for (let i = 0; i < 3; i++) expect(Math.abs(got[i]! - want[i]!)).toBeLessThanOrEqual(1);
  };
  const t = resolveTokens("notte", "light");

  it("accento, collegamento, superfici, bordi, testo", () => {
    near(t["--isa-accent"]!, "#1e3a6e");
    near(t["--isa-text-link"]!, "#2563eb");
    near(t["--isa-surface-raised"]!, "#ffffff");
    near(t["--isa-surface-base"]!, "#f8fafc");
    near(t["--isa-border"]!, "#e2e8f0");
    near(t["--isa-text"]!, "#0f172a");
    near(t["--isa-text-secondary"]!, "#64748b");
  });

  it("famiglie di operazioni e avviso", () => {
    near(t["--isa-op-filter"]!, "#2563eb");
    near(t["--isa-op-transform"]!, "#7c3aed");
    near(t["--isa-op-merge"]!, "#ea580c");
    near(t["--isa-op-output"]!, "#16a34a");
    near(t["--isa-warning"]!, "#f59e0b");
  });

  it("ogni famiglia ha la propria tinta tenue, diversa dalle altre", () => {
    const softs = ["filter", "transform", "merge", "output"].map((f) => t[`--isa-op-${f}-soft`]);
    expect(new Set(softs).size).toBe(4);
  });

  it("nel tema predefinito le quattro famiglie coincidono (aspetto identico a prima)", () => {
    const p = resolveTokens("prototipo", "light");
    expect(
      new Set(["filter", "transform", "merge", "output"].map((f) => p[`--isa-op-${f}`])).size,
    ).toBe(1);
    expect(p["--isa-op-filter"]).toBe(p["--isa-tint-ink"]);
    expect(p["--isa-op-filter-soft"]).toBe(p["--isa-tint"]);
  });

  it("il tema predefinito non cambia i valori storici", () => {
    const l = rawTokens("prototipo", "light");
    expect(l["--isa-surface-base"]).toBe("#f5f3ee");
    expect(l["--isa-accent"]).toBe("#6c63ff");
    expect(l["--background"]).toBe("oklch(0.978 0.004 250)");
    expect(l["--primary"]).toBe("oklch(0.52 0.11 272)");
    const d = rawTokens("prototipo", "dark");
    expect(d["--background"]).toBe("oklch(0.19 0.008 260)");
    expect(d["--isa-surface-base"]).toBe("#17181d");
  });
});
```

### `src/theme/boot.ts`

29 righe

```ts
/**
 * Script di avvio, inserito nella testa della pagina (src/routes/__root.tsx):
 * applica tema, modo e tinta dell'accento PRIMA del primo disegno, così non c'è
 * alcun lampo. Non importa nulla: è testo eseguito prima di qualunque modulo.
 * Legge la stessa preferenza di runtime.ts (`isa.theme.v1`, con ripiego sul
 * vecchio `isa-theme` per il modo) e applica i token d'accento già derivati e
 * salvati da `setAccentHue`. Tutto in try/catch: se l'archivio non è
 * disponibile resta il predefinito (tema "prototipo", scuro).
 *
 * Solo in sviluppo, `?theme=<nome>` forza il tema.
 */
import { LEGACY_MODE_KEY, THEME_NAMES, THEME_STORAGE_KEY } from "./runtime";

export function themeBootScript(allowQueryTheme: boolean): string {
  return `(function(){try{
var d=document.documentElement,p=null,m=null,t="prototipo",N=${JSON.stringify(THEME_NAMES)};
try{p=JSON.parse(localStorage.getItem(${JSON.stringify(THEME_STORAGE_KEY)})||"null")}catch(e){}
if(p&&typeof p==="object"){if(p.mode==="light"||p.mode==="dark")m=p.mode;if(N.indexOf(p.theme)>=0)t=p.theme}else p=null;
if(!m){try{m=localStorage.getItem(${JSON.stringify(LEGACY_MODE_KEY)})}catch(e){}if(m!=="light"&&m!=="dark")m="dark"}
${
  allowQueryTheme
    ? `try{var q=new URLSearchParams(location.search).get("theme");if(N.indexOf(q)>=0)t=q}catch(e){}`
    : ""
}
d.classList.toggle("dark",m==="dark");d.setAttribute("data-theme",t);
var a=p&&p.accent&&p.accent[m];if(a&&typeof a==="object")for(var k in a)d.style.setProperty(k,a[k]);
}catch(e){}})();`;
}
```

### `src/theme/color.ts`

115 righe

```ts
/**
 * Colori e contrasto WCAG: funzioni pure, senza DOM. Servono a derive.ts (che
 * regola la luminosità dell'accento) e ai test dei temi.
 */
export type Rgba = readonly [number, number, number, number];

const clamp01 = (n: number) => Math.min(1, Math.max(0, n));

/** OKLCH (L 0–1, C, H in gradi) → sRGB lineare, senza limitare i canali. */
export function oklchToLinear(L: number, C: number, H: number): [number, number, number] {
  const h = (H * Math.PI) / 180;
  const a = C * Math.cos(h);
  const b = C * Math.sin(h);
  const l = (L + 0.3963377774 * a + 0.2158037573 * b) ** 3;
  const m = (L - 0.1055613458 * a - 0.0638541728 * b) ** 3;
  const s = (L - 0.0894841775 * a - 1.291485548 * b) ** 3;
  return [
    4.0767416621 * l - 3.3077115913 * m + 0.2309699292 * s,
    -1.2684380046 * l + 2.6097574011 * m - 0.3413193965 * s,
    -0.0041960863 * l - 0.7034186147 * m + 1.707614701 * s,
  ];
}

const EPS = 1e-6;

/** Vero se il colore OKLCH sta dentro il gamut sRGB. */
export function inGamut(L: number, C: number, H: number): boolean {
  return oklchToLinear(L, C, H).every((v) => v >= -EPS && v <= 1 + EPS);
}

/** Il colore sRGB (0–255) di un OKLCH; i canali fuori gamut sono limitati. */
export function oklchToRgb(L: number, C: number, H: number): [number, number, number] {
  const enc = (v: number) => {
    const c = clamp01(v);
    return 255 * (c <= 0.0031308 ? 12.92 * c : 1.055 * c ** (1 / 2.4) - 0.055);
  };
  const [r, g, b] = oklchToLinear(L, C, H);
  return [enc(r), enc(g), enc(b)];
}

/** Riduce la croma finché il colore entra nel gamut sRGB (L e H restano). */
export function fitChroma(L: number, C: number, H: number): number {
  if (inGamut(L, C, H)) return C;
  let lo = 0;
  let hi = C;
  for (let i = 0; i < 24; i++) {
    const mid = (lo + hi) / 2;
    if (inGamut(L, mid, H)) lo = mid;
    else hi = mid;
  }
  return lo;
}

/** Valida e interpreta un numero di OKLCH: `0.5`, `50%` (solo L/alpha), `none`. */
function num(token: [REDATTO], percentBase: number): number {
  const t = token.trim();
  if (t === "none") return 0;
  return t.endsWith("%") ? (parseFloat(t) / 100) * percentBase : parseFloat(t);
}

/** `#rgb`, `#rrggbb`, `rgb()`/`rgba()` e `oklch(L C H / alpha)`. */
export function parseColor(value: string): Rgba {
  const v = value.trim().toLowerCase();
  const hex = /^#([0-9a-f]{3}|[0-9a-f]{6})$/.exec(v);
  if (hex) {
    const h = hex[1] as string;
    const full = h.length === 3 ? [...h].map((c) => c + c).join("") : h;
    return [
      parseInt(full.slice(0, 2), 16),
      parseInt(full.slice(2, 4), 16),
      parseInt(full.slice(4, 6), 16),
      1,
    ];
  }
  const rgba = /^rgba?\(([^)]+)\)$/.exec(v);
  if (rgba) {
    const p = (rgba[1] as string).split(",").map((s) => parseFloat(s));
    return [p[0] ?? 0, p[1] ?? 0, p[2] ?? 0, p[3] ?? 1];
  }
  const ok = /^oklch\(\s*([^/)]+?)\s*(?:\/\s*([^)]+?)\s*)?\)$/.exec(v);
  if (ok) {
    const [l = "0", c = "0", h = "0"] = (ok[1] as string).split(/\s+/);
    const [r, g, b] = oklchToRgb(num(l, 1), num(c, 0.4), num(h, 360));
    return [r, g, b, ok[2] === undefined ? 1 : num(ok[2], 1)];
  }
  throw new Error(`colore non riconosciuto: ${value}`);
}

/** Sovrappone `top` (con trasparenza) a `bottom` (opaco). */
export function over(top: Rgba, bottom: Rgba): Rgba {
  const a = top[3];
  return [
    top[0] * a + bottom[0] * (1 - a),
    top[1] * a + bottom[1] * (1 - a),
    top[2] * a + bottom[2] * (1 - a),
    1,
  ];
}

function lin(c: number): number {
  const s = c / 255;
  return s <= 0.03928 ? s / 12.92 : ((s + 0.055) / 1.055) ** 2.4;
}

export function luminance(c: Rgba): number {
  return 0.2126 * lin(c[0]) + 0.7152 * lin(c[1]) + 0.0722 * lin(c[2]);
}

/** Rapporto di contrasto WCAG tra due colori opachi. */
export function contrast(a: Rgba, b: Rgba): number {
  const la = luminance(a);
  const lb = luminance(b);
  return (Math.max(la, lb) + 0.05) / (Math.min(la, lb) + 0.05);
}
```

### `src/theme/derive.ts`

212 righe

```ts
/**
 * Derivazione dell'accento da una sola tinta OKLCH (0–360). Modulo puro: niente DOM.
 *
 * L'utente sceglie la tinta; luminosità e croma le decide il sistema. Ogni
 * colore parte da una luminosità "di gusto" e, se per quella tinta il
 * contrasto richiesto non basta, la luminosità viene spostata a passi di
 * 0,005 (verso lo scuro nel modo chiaro, verso il chiaro nel modo scuro)
 * finché la soglia non è rispettata. La croma, se il colore esce dal gamut
 * sRGB, è ridotta (`fitChroma`), così il colore calcolato è lo stesso su
 * qualunque schermo. Il contrasto è misurato sul valore GIÀ arrotondato che
 * viene emesso.
 *
 * Fondi di riferimento. La funzione non conosce il tema: garantisce i
 * contrasti contro due fondi "peggiori" per modo, più sfavorevoli di qualsiasi
 * superficie reale dei temi (verificato dai test su ogni tema e modo):
 *  - chiaro: grigio OKLCH L 0,90 (il più scuro tra i fondi chiari);
 *  - scuro: grigio OKLCH L 0,32 (il più chiaro tra i fondi scuri).
 *
 * Soglie (margine di 0,05 sopra il minimo, contro gli arrotondamenti):
 *  - testo su accento ≥ 4,5:1 (`onAccent` su `base` e su `active`);
 *  - accento come testo (`text`) ≥ 4,5:1 sui fondi;
 *  - accento (`base`), anello di focus (`focusRing`) ≥ 3:1 sui fondi.
 */
import { contrast, fitChroma, oklchToRgb, parseColor } from "./color";
import type { Rgba } from "./color";

export type Mode = "light" | "dark";

/** Tinta OKLCH → valori dei token d'accento, come stringhe `oklch(...)`. */
export interface AccentTokens {
  /** Riempimenti e bordi d'accento. */
  readonly base: string;
  /** Stato attivo/premuto: più contrasto rispetto a `base`. */
  readonly active: string;
  /** Tinta tenue (sfondi): `base` con trasparenza 16%. */
  readonly soft: string;
  /** Tinta media: `base` con trasparenza 34%. */
  readonly softStrong: string;
  /** Accento usato come colore del testo. */
  readonly text: string;
  /** Testo e icone sopra `base`. */
  readonly onAccent: string;
  /** Anello di focus. */
  readonly focusRing: string;
  /** Fondo pastello (chip delle lavorazioni) e relativo colore di icona. */
  readonly tint: string;
  readonly tintInk: string;
}

const MARGIN = 0.05;
const STEP = 0.005;

const grey = (L: number): Rgba => [...oklchToRgb(L, 0, 0), 1];
/** Fondi di riferimento per modo (vedi l'intestazione). */
export const REFERENCE_SURFACE: Record<Mode, Rgba> = { light: grey(0.9), dark: grey(0.32) };

const WHITE_CSS = "oklch(1 0 0)";
const DARK_INK_CSS = "oklch(0.19 0.008 260)";

interface Lch {
  readonly L: number;
  readonly C: number;
  readonly H: number;
}

const r3 = (n: number) => Math.round(n * 1000) / 1000;
const normHue = (h: number) => ((h % 360) + 360) % 360;

/** Arrotonda come verrà scritto e porta la croma dentro il gamut. */
function make(L: number, C: number, H: number): Lch {
  const l = r3(Math.min(0.99, Math.max(0.01, L)));
  const h = Math.round(normHue(H) * 10) / 10;
  return { L: l, C: r3(Math.floor(fitChroma(l, C, h) * 1000) / 1000), H: h };
}

const rgbOf = (c: Lch): Rgba => [...oklchToRgb(c.L, c.C, c.H), 1];
const css = (c: Lch, alpha?: number) =>
  `oklch(${c.L} ${c.C} ${c.H}${alpha === undefined ? "" : ` / ${alpha}`})`;

/** Sposta la luminosità a passi di STEP nella direzione `dir` finché `ok` è vero. */
function search(H: number, C: number, startL: number, dir: 1 | -1, ok: (c: Lch) => boolean): Lch {
  let L = startL;
  for (;;) {
    const c = make(L, C, H);
    if (ok(c) || L <= 0.02 || L >= 0.98) return c;
    L += dir * STEP;
  }
}

function deriveRaw(hue: number, mode: Mode) {
  const H = normHue(hue);
  const ref = REFERENCE_SURFACE[mode];
  const onRgb = parseColor(mode === "light" ? WHITE_CSS : DARK_INK_CSS);
  const on = (c: Lch) => contrast(rgbOf(c), onRgb);
  const onRef = (c: Lch) => contrast(rgbOf(c), ref);
  const dir = mode === "light" ? -1 : 1;

  const base = search(
    H,
    mode === "light" ? 0.19 : 0.17,
    mode === "light" ? 0.6 : 0.68,
    dir,
    (c) => on(c) >= 4.5 + MARGIN && onRef(c) >= 3 + MARGIN,
  );
  const active = search(H, base.C, base.L + dir * 0.06, dir, (c) => on(c) >= 4.5 + MARGIN);
  const text = search(
    H,
    mode === "light" ? 0.17 : 0.12,
    mode === "light" ? 0.55 : 0.75,
    dir,
    (c) => onRef(c) >= 4.5 + MARGIN,
  );
  const focusRing = search(
    H,
    mode === "light" ? 0.19 : 0.15,
    mode === "light" ? 0.62 : 0.7,
    dir,
    (c) => onRef(c) >= 3 + MARGIN,
  );
  const tint = make(mode === "light" ? 0.91 : 0.34, mode === "light" ? 0.04 : 0.07, H);
  const tintInk = mode === "light" ? base : text;
  return {
    H,
    base,
    active,
    text,
    focusRing,
    tint,
    tintInk,
    on: mode === "light" ? WHITE_CSS : DARK_INK_CSS,
  };
}

/**
 * Token d'accento per una tinta (0–360) e un modo. Garantisce i contrasti
 * descritti nell'intestazione per QUALUNQUE tinta.
 */
export function deriveAccent(hue: number, mode: Mode): AccentTokens {
  const d = deriveRaw(hue, mode);
  return {
    base: css(d.base),
    active: css(d.active),
    soft: css(d.base, 0.16),
    softStrong: css(d.base, 0.34),
    text: css(d.text),
    onAccent: d.on,
    focusRing: css(d.focusRing),
    tint: css(d.tint),
    tintInk: css(d.tintInk),
  };
}

/**
 * Le proprietà CSS da scrivere su `<html>` per applicare l'accento: i token
 * d'accento semantici (`--isa-accent*`, `--isa-focus-ring`), quelli del canvas
 * che ne dipendono nel tema predefinito (cavi, minimappa, fette, selezione) e
 * quelli dell'app (`--primary`, `--brand`, `--ring`, …).
 */
export function accentCssVars(hue: number, mode: Mode): Record<string, string> {
  const light = mode === "light";
  const d = deriveRaw(hue, mode);
  const a = deriveAccent(hue, mode);
  const shade = (L: number, C: number, dh = 0) => {
    const c = make(L, C, d.H + dh);
    return css(c);
  };
  const glow = make(d.base.L + 0.07, d.base.C * 0.8, d.H - 14);
  return {
    // semantici
    "--isa-accent": a.base,
    "--isa-accent-active": a.active,
    "--isa-accent-soft": a.soft,
    "--isa-accent-text": a.text,
    "--isa-text-on-accent": a.onAccent,
    "--isa-focus-ring": a.focusRing,
    "--isa-dataset-fill": a.base,
    // canvas
    "--isa-accent-soft-2": a.softStrong,
    "--isa-tint": a.tint,
    "--isa-tint-ink": a.tintInk,
    ...(light ? {} : { "--isa-tint-border": a.base }),
    "--isa-select": a.text,
    "--isa-link": light ? a.softStrong : a.base,
    "--isa-link-dot-tint": light ? a.tint : a.text,
    "--isa-flow": light ? css(d.base, 0.6) : css(d.text, 0.9),
    "--isa-mm-node": light ? shade(0.83, 0.07) : a.base,
    "--isa-mm-node-ds": light ? a.base : a.text,
    "--isa-mm-view-line": light ? a.base : a.text,
    "--isa-mm-view-bg": light ? css(d.base, 0.08) : css(d.text, 0.12),
    "--isa-split-bg": light ? shade(0.955, 0.018) : shade(0.3, 0.045),
    "--isa-split-empty": light ? shade(0.925, 0.03) : shade(0.32, 0.05),
    "--isa-split-empty-ink": a.text,
    "--isa-split-line": css(d.text, 0.55),
    // app
    "--primary": a.base,
    "--primary-foreground": a.onAccent,
    "--ring": a.focusRing,
    "--brand": a.base,
    "--brand-foreground": a.onAccent,
    "--brand-glow": css(glow),
    "--sidebar-primary": a.base,
    "--sidebar-primary-foreground": a.onAccent,
    "--sidebar-ring": a.focusRing,
    "--chart-1": a.base,
  };
}

/** Nomi di tutte le proprietà scritte da `accentCssVars` (per rimuoverle). */
export const ACCENT_VAR_NAMES: readonly string[] = [
  ...new Set([...Object.keys(accentCssVars(0, "light")), ...Object.keys(accentCssVars(0, "dark"))]),
];
```

### `src/theme/index.css`

10 righe

```css
/*
 * Sistema di temi — vedi src/theme/README.md.
 * Ordine: primitive → tema predefinito (`:root`, `.dark`) → altri temi
 * (`:root[data-theme]`, più specifici).
 */
@import "./primitives.css";
@import "./themes/prototipo.css";
@import "./themes/notte.css";
@import "./layout-tokens.css";
```

### `src/theme/index.ts`

13 righe

```ts
export { deriveAccent, accentCssVars, ACCENT_VAR_NAMES } from "./derive";
export type { AccentTokens, Mode } from "./derive";
export {
  THEME_NAMES,
  THEME_STORAGE_KEY,
  DEFAULT_PREFERENCE,
  themeStore,
  setTheme,
  setMode,
  setAccentHue,
} from "./runtime";
export type { Preference, ThemeName } from "./runtime";
```

### `src/theme/layout-tokens.css`

50 righe

```css
/*
 * Token di forma — indipendenti dal tema e dal modo: scala tipografica, scala
 * degli spazi, misure dei controlli e dei menu. Li usa l'Inspector
 * (src/etl-canvas/inspector/) e, in futuro, il resto dell'app.
 *
 * scripts/check-tokens.mjs controlla che in src/etl-canvas/inspector/** i
 * `font-size`, i `margin`, i `padding` e i `gap` usino solo questi token (o
 * 0, auto, percentuali). Nessun testo sotto 11px: --isa-fs-overline è
 * l'unico sotto i 12px (e il suo valore è verificato da un test).
 */
:root {
  /* scala tipografica */
  --isa-fs-title: 17px; /* nome del nodo: peso 800, interlinea 1.3 */
  --isa-fs-overline: 11px; /* famiglia e sezioni: maiuscolo, peso 800, spaziatura .08em */
  --isa-fs-label: 12px; /* etichette dei campi: peso 700, non maiuscolo */
  --isa-fs-value: 14px; /* valori e campi: peso 600 */
  --isa-fs-summary: 13px; /* riassunto di riga: peso 700, una riga con ellissi */
  --isa-fs-help: 12px; /* note e guide: peso 500 */
  --isa-lh-title: 1.3;
  --isa-lh-text: 1.4;
  --isa-ls-overline: 0.08em;

  /* scala degli spazi */
  --isa-space-1: 4px;
  --isa-space-2: 8px;
  --isa-space-3: 12px;
  --isa-space-4: 16px;
  --isa-space-5: 20px;
  --isa-space-6: 24px;
  --isa-space-8: 32px;

  /* controlli: altezza minima, pulsanti con sola icona (area cliccabile ≥ 32px) */
  --isa-control-h: 40px;
  --isa-icon-btn: 32px;

  /* tre colonne dei bordi alto e basso (Impostazioni | Condizioni o Elenco | Dettaglio) */
  --isa-md-min-w: 900px; /* sotto questa larghezza tornano le colonne CSS della 6b.1 */
  --isa-md-general-w: 240px;
  --isa-md-master-w: 310px;

  /* tendine e menu */
  --isa-menu-gap: 8px; /* distanza dal campo */
  --isa-menu-edge: 16px; /* margine minimo dai bordi della finestra */
  --isa-menu-pad: var(--isa-space-2); /* riempimento interno */
  --isa-menu-item-h: 40px;
  --isa-menu-item-px: var(--isa-space-3);
  --isa-menu-max-w: 420px; /* e comunque finestra − 2 × --isa-menu-edge */
  --isa-menu-enter: 120ms;
}
```

### `src/theme/primitives.css`

88 righe

```css
/*
 * Livello 1 — PRIMITIVE (--isa-p-*): tavolozze grezze in OKLCH.
 *
 * Non descrivono un ruolo, solo un colore. NESSUN componente le usa
 * direttamente: le leggono soltanto i temi (src/theme/themes/*.css), che le
 * assegnano ai token semantici (--isa-*). Sono indipendenti dal modo e dal
 * tema; il valore esadecimale sorgente è nel commento a fianco.
 *
 * Il tema "prototipo" non le usa: i suoi valori sono quelli storici dell'app e
 * del canvas, scritti direttamente nell'assegnazione semantica (vedi
 * themes/prototipo.css e README.md).
 */
:root {
  /* Neutri (slate) */
  --isa-p-white: oklch(1 0 89.88); /* #ffffff */
  --isa-p-slate-50: oklch(0.9842 0.0034 247.86); /* #f8fafc */
  --isa-p-slate-100: oklch(0.9683 0.0069 247.9); /* #f1f5f9 */
  --isa-p-slate-200: oklch(0.9288 0.0126 255.51); /* #e2e8f0 */
  --isa-p-slate-300: oklch(0.869 0.0198 252.89); /* #cbd5e1 */
  --isa-p-slate-400: oklch(0.7107 0.0351 256.79); /* #94a3b8 */
  --isa-p-slate-500: oklch(0.5544 0.0407 257.42); /* #64748b */
  --isa-p-slate-600: oklch(0.4455 0.0374 257.28); /* #475569 */
  --isa-p-slate-900: oklch(0.2077 0.0398 265.75); /* #0f172a */

  /* Notte (superfici scure) */
  --isa-p-night-950: oklch(0.1831 0.0309 263.38); /* #0b1220 */
  --isa-p-night-900: oklch(0.2204 0.0416 264.83); /* #111a2e */
  --isa-p-night-800: oklch(0.2468 0.0441 266.33); /* #172036 */
  --isa-p-night-700: oklch(0.3091 0.0491 261.52); /* #223049 */

  /* Navy (accento del tema notte) */
  --isa-p-navy-700: oklch(0.3566 0.0964 261.55); /* #1e3a6e */
  --isa-p-navy-800: oklch(0.3066 0.0795 261.01); /* #172e57 */
  --isa-p-accent-300: oklch(0.7166 0.1089 262.81); /* #7fa3e8 */
  --isa-p-accent-400: oklch(0.6693 0.1223 262.02); /* #6b94e0 */

  /* Azzurro */
  --isa-p-blue-100: oklch(0.9319 0.0316 255.59); /* #dbeafe */
  --isa-p-blue-400: oklch(0.7137 0.1434 254.62); /* #60a5fa */
  --isa-p-blue-600: oklch(0.5461 0.2152 262.88); /* #2563eb */
  --isa-p-blue-800: oklch(0.4244 0.1809 265.64); /* #1e40af */

  /* Viola */
  --isa-p-violet-100: oklch(0.9433 0.0284 294.59); /* #ede9fe */
  --isa-p-violet-400: oklch(0.709 0.1592 293.54); /* #a78bfa */
  --isa-p-violet-600: oklch(0.5413 0.2466 293.01); /* #7c3aed */
  --isa-p-violet-800: oklch(0.432 0.2106 292.76); /* #5b21b6 */

  /* Arancio */
  --isa-p-orange-100: oklch(0.9542 0.0372 75.16); /* #ffedd5 */
  --isa-p-orange-400: oklch(0.7576 0.159 55.93); /* #fb923c */
  --isa-p-orange-600: oklch(0.6461 0.1943 41.12); /* #ea580c */

  /* Verde */
  --isa-p-green-100: oklch(0.9624 0.0434 156.74); /* #dcfce7 */
  --isa-p-green-400: oklch(0.8003 0.1821 151.71); /* #4ade80 */
  --isa-p-green-600: oklch(0.6271 0.1699 149.21); /* #16a34a */
  --isa-p-green-800: oklch(0.4479 0.1083 151.33); /* #166534 */

  /* Ambra e rosso */
  --isa-p-amber-400: oklch(0.8369 0.1644 84.43); /* #fbbf24 */
  --isa-p-amber-500: oklch(0.7686 0.1647 70.08); /* #f59e0b */
  --isa-p-red-400: oklch(0.7106 0.1661 22.22); /* #f87171 */
  --isa-p-red-600: oklch(0.5771 0.2152 27.33); /* #dc2626 */

  /* Varianti con trasparenza (il numero è la percentuale di opacità) */
  --isa-p-slate-900-a8: oklch(0.2077 0.0398 265.75 / 8%);
  --isa-p-slate-900-a14: oklch(0.2077 0.0398 265.75 / 14%);
  --isa-p-slate-400-a18: oklch(0.7107 0.0351 256.79 / 18%);
  --isa-p-slate-400-a32: oklch(0.7107 0.0351 256.79 / 32%);
  --isa-p-navy-700-a10: oklch(0.3566 0.0964 261.55 / 10%);
  --isa-p-navy-700-a16: oklch(0.3566 0.0964 261.55 / 16%);
  --isa-p-accent-300-a16: oklch(0.7166 0.1089 262.81 / 16%);
  --isa-p-accent-300-a28: oklch(0.7166 0.1089 262.81 / 28%);
  --isa-p-blue-400-a16: oklch(0.7137 0.1434 254.62 / 16%);
  --isa-p-blue-600-a12: oklch(0.5461 0.2152 262.88 / 12%);
  --isa-p-violet-400-a16: oklch(0.709 0.1592 293.54 / 16%);
  --isa-p-violet-600-a12: oklch(0.5413 0.2466 293.01 / 12%);
  --isa-p-orange-400-a16: oklch(0.7576 0.159 55.93 / 16%);
  --isa-p-orange-600-a12: oklch(0.6461 0.1943 41.12 / 12%);
  --isa-p-green-400-a16: oklch(0.8003 0.1821 151.71 / 16%);
  --isa-p-green-600-a12: oklch(0.6271 0.1699 149.21 / 12%);
  --isa-p-white-a8: oklch(1 0 89.88 / 8%);
  --isa-p-white-a12: oklch(1 0 89.88 / 12%);
  --isa-p-white-a16: oklch(1 0 89.88 / 16%);
  --isa-p-black-a40: oklch(0 0 0 / 40%);
}
```

### `src/theme/runtime.ts`

212 righe

```ts
/**
 * Preferenza di tema (nome, modo, tinta dell'accento): lettura, applicazione a
 * `<html>` e salvataggio in localStorage (`isa.theme.v1`).
 *
 * Regola fondamentale: LEGGERE NON SCRIVE MAI. Si scrive solo in risposta a
 * `setTheme` / `setMode` / `setAccentHue`. (Il vecchio ThemeProvider scriveva
 * «dark» prima di leggere il valore salvato, e il chiaro non sopravviveva a un
 * ricaricamento.)
 *
 * Nessuna interfaccia utente: l'API è `setTheme`, `setMode`, `setAccentHue`.
 */
import { ACCENT_VAR_NAMES, accentCssVars } from "./derive";
import type { Mode } from "./derive";

export type { Mode };

export const THEME_STORAGE_KEY = "isa.theme.v1";
/** Chiave del vecchio ThemeProvider: si legge (mai si scrive) per non perdere il modo già scelto. */
export const LEGACY_MODE_KEY = "isa-theme";

export const THEME_NAMES = ["prototipo", "notte"] as const;
export type ThemeName = (typeof THEME_NAMES)[number];

export interface Preference {
  readonly theme: ThemeName;
  readonly mode: Mode;
  /** Tinta OKLCH 0–360 dell'accento, o null per quella del tema. */
  readonly accentHue: number | null;
}

/** Come l'app si è sempre aperta: tema predefinito, modo scuro. */
export const DEFAULT_PREFERENCE: Preference = { theme: "prototipo", mode: "dark", accentHue: null };

export const isThemeName = (v: unknown): v is ThemeName =>
  typeof v === "string" && (THEME_NAMES as readonly string[]).includes(v);
const isMode = (v: unknown): v is Mode => v === "light" || v === "dark";

/** Porta una tinta in [0, 360); null se non è un numero finito. */
export function normalizeHue(hue: unknown): number | null {
  if (typeof hue !== "number" || !Number.isFinite(hue)) return null;
  return Math.round((((hue % 360) + 360) % 360) * 10) / 10;
}

/** Interpreta ciò che c'è in localStorage; qualunque dato non valido torna al predefinito. */
export function parsePreference(raw: string | null, legacyMode: string | null = null): Preference {
  let data: Record<string, unknown> = {};
  if (raw) {
    try {
      const parsed: unknown = JSON.parse(raw);
      if (parsed && typeof parsed === "object") data = parsed as Record<string, unknown>;
    } catch {
      /* dato rovinato: si usa il predefinito */
    }
  }
  const mode = isMode(data["mode"])
    ? data["mode"]
    : isMode(legacyMode)
      ? legacyMode
      : DEFAULT_PREFERENCE.mode;
  return {
    theme: isThemeName(data["theme"]) ? data["theme"] : DEFAULT_PREFERENCE.theme,
    mode,
    accentHue: normalizeHue(data["accentHue"]),
  };
}

/**
 * Il testo da salvare. Insieme alla tinta si salvano i token già derivati per i
 * due modi, così lo script di avvio (che non può importare nulla) li applica
 * prima del primo disegno senza rifare il calcolo.
 */
export function serializePreference(pref: Preference): string {
  return JSON.stringify({
    v: 1,
    theme: pref.theme,
    mode: pref.mode,
    accentHue: pref.accentHue,
    accent:
      pref.accentHue === null
        ? null
        : {
            light: accentCssVars(pref.accentHue, "light"),
            dark: accentCssVars(pref.accentHue, "dark"),
          },
  });
}

/** Il sottoinsieme di <html> che serve ad applicare la preferenza (e che un finto elemento può imitare nei test). */
export interface RootLike {
  classList: { toggle(name: string, force?: boolean): boolean };
  setAttribute(name: string, value: string): void;
  style: { setProperty(name: string, value: string): void; removeProperty(name: string): string };
}

export interface StorageLike {
  getItem(key: string): string | null;
  setItem(key: string, value: string): void;
}

/** Scrive su <html> classe del modo, `data-theme` e, se c'è, i token dell'accento. */
export function applyPreference(root: RootLike, pref: Preference): void {
  root.classList.toggle("dark", pref.mode === "dark");
  root.setAttribute("data-theme", pref.theme);
  for (const name of ACCENT_VAR_NAMES) root.style.removeProperty(name);
  if (pref.accentHue !== null) {
    for (const [name, value] of Object.entries(accentCssVars(pref.accentHue, pref.mode))) {
      root.style.setProperty(name, value);
    }
  }
}

/** Solo in sviluppo: `?theme=notte` forza il tema, per le verifiche. */
export function themeFromSearch(search: string): ThemeName | null {
  const value = new URLSearchParams(search).get("theme");
  return isThemeName(value) ? value : null;
}

export interface ThemeStoreOptions {
  readonly storage?: StorageLike | null;
  readonly root?: RootLike | null;
  /** Query string della pagina; il parametro `theme` conta solo con `allowQueryTheme`. */
  readonly search?: string;
  readonly allowQueryTheme?: boolean;
}

export interface ThemeStore {
  get(): Preference;
  subscribe(listener: () => void): () => void;
  /** Applica la preferenza corrente a <html> senza salvare nulla. */
  sync(): void;
  setTheme(name: ThemeName): void;
  setMode(mode: Mode): void;
  toggleMode(): void;
  setAccentHue(hue: number | null): void;
}

export function createThemeStore(options: ThemeStoreOptions = {}): ThemeStore {
  const { storage = null, root = null, search = "", allowQueryTheme = false } = options;
  const read = (key: string): string | null => {
    try {
      return storage ? storage.getItem(key) : null;
    } catch {
      return null;
    }
  };
  let saved = parsePreference(read(THEME_STORAGE_KEY), read(LEGACY_MODE_KEY));
  let forced: ThemeName | null = allowQueryTheme ? themeFromSearch(search) : null;
  let current: Preference = forced ? { ...saved, theme: forced } : saved;
  const listeners = new Set<() => void>();

  const commit = (next: Preference, persist: boolean) => {
    saved = persist ? next : saved;
    current = next;
    if (persist) {
      try {
        storage?.setItem(THEME_STORAGE_KEY, serializePreference(next));
      } catch {
        /* archivio non disponibile o pieno: la preferenza vale solo per questa sessione */
      }
    }
    if (root) applyPreference(root, current);
    for (const l of listeners) l();
  };

  return {
    get: () => current,
    subscribe(listener) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
    sync() {
      if (root) applyPreference(root, current);
    },
    setTheme(name) {
      forced = null;
      commit({ ...current, theme: name }, true);
    },
    setMode(mode) {
      commit({ ...current, mode }, true);
    },
    toggleMode() {
      commit({ ...current, mode: current.mode === "dark" ? "light" : "dark" }, true);
    },
    setAccentHue(hue) {
      commit({ ...current, accentHue: normalizeHue(hue) }, true);
    },
  };
}

function browserStorage(): StorageLike | null {
  try {
    return typeof window === "undefined" ? null : window.localStorage;
  } catch {
    return null;
  }
}

/** L'archivio del browser (inerte sul server e nei test in Node). */
export const themeStore: ThemeStore =
  typeof window === "undefined"
    ? createThemeStore()
    : createThemeStore({
        storage: browserStorage(),
        root: document.documentElement,
        search: window.location.search,
        allowQueryTheme: import.meta.env.DEV,
      });

export const setTheme = (name: ThemeName) => themeStore.setTheme(name);
export const setMode = (mode: Mode) => themeStore.setMode(mode);
export const setAccentHue = (hue: number | null) => themeStore.setAccentHue(hue);
```

### `src/theme/themes/notte.css`

290 righe

```css
/*
 * Tema "notte" — tema di prova ispirato a RiskHub. Livello 2: assegna i token
 * semantici (stessi nomi del tema "prototipo") a primitive (--isa-p-*).
 *
 * Chiaro: accento #1E3A6E, collegamento #2563EB, superfici #FFFFFF e #F8FAFC,
 * bordi #E2E8F0, testo #0F172A / #64748B; famiglie di operazioni azzurro
 * #2563EB, viola #7C3AED, arancio #EA580C, verde #16A34A (ciascuna con la sua
 * tinta tenue), avviso #F59E0B.
 * Scuro: stessi ruoli e stesse tinte, schiarite dove serve per i contrasti
 * (test: src/theme/__tests__/contrast.test.ts).
 *
 * Selettori: `:root[data-theme="notte"]` batte `:root` e `.dark` del tema
 * predefinito; `:root.dark[data-theme="notte"]` batte il precedente. Il tema
 * definisce TUTTI i token (nessuno ricade sul predefinito): lo verifica un test.
 */
:root[data-theme="notte"] {
  --radius: 0.75rem;

  /* ── Token dell'app ── */
  --background: var(--isa-p-slate-50);
  --foreground: var(--isa-p-slate-900);
  --card: var(--isa-p-white);
  --card-foreground: var(--isa-p-slate-900);
  --popover: var(--isa-p-white);
  --popover-foreground: var(--isa-p-slate-900);
  --primary: var(--isa-p-navy-700);
  --primary-foreground: var(--isa-p-white);
  --secondary: var(--isa-p-slate-100);
  --secondary-foreground: var(--isa-p-slate-900);
  --muted: var(--isa-p-slate-100);
  --muted-foreground: var(--isa-p-slate-500);
  --accent: var(--isa-p-navy-700-a10);
  --accent-foreground: var(--isa-p-navy-700);
  --destructive: var(--isa-p-red-600);
  --destructive-foreground: var(--isa-p-white);
  --border: var(--isa-p-slate-200);
  --input: var(--isa-p-slate-200);
  --ring: var(--isa-p-blue-600);
  --brand: var(--isa-p-navy-700);
  --brand-foreground: var(--isa-p-white);
  --brand-glow: var(--isa-p-blue-600);
  --success: var(--isa-p-green-600);
  --warning: var(--isa-p-amber-500);
  --glass: var(--isa-p-white);
  --glass-strong: var(--isa-p-white);
  --glass-border: var(--isa-p-slate-200);
  --chart-1: var(--isa-p-blue-600);
  --chart-2: var(--isa-p-violet-600);
  --chart-3: var(--isa-p-orange-600);
  --chart-4: var(--isa-p-green-600);
  --chart-5: var(--isa-p-amber-500);
  --blob-1: var(--isa-p-slate-200);
  --blob-2: var(--isa-p-blue-100);
  --blob-3: var(--isa-p-slate-100);
  --type-etl-bg: var(--isa-p-blue-100);
  --type-etl-fg: var(--isa-p-blue-800);
  --type-model-bg: var(--isa-p-violet-100);
  --type-model-fg: var(--isa-p-violet-800);
  --type-dashboard-bg: var(--isa-p-green-100);
  --type-dashboard-fg: var(--isa-p-green-800);
  --shadow-glass:
    0 1px 3px 0 var(--isa-p-slate-900-a8), 0 8px 20px -12px var(--isa-p-slate-900-a14);
  --sidebar: var(--isa-p-white);
  --sidebar-foreground: var(--isa-p-slate-900);
  --sidebar-primary: var(--isa-p-navy-700);
  --sidebar-primary-foreground: var(--isa-p-white);
  --sidebar-accent: var(--isa-p-slate-100);
  --sidebar-accent-foreground: var(--isa-p-slate-900);
  --sidebar-border: var(--isa-p-slate-200);
  --sidebar-ring: var(--isa-p-blue-600);

  /* ── Semantici condivisi (--isa-*) ── */
  --isa-surface-base: var(--isa-p-slate-50);
  --isa-surface-raised: var(--isa-p-white);
  --isa-surface-overlay: var(--isa-p-white);
  --isa-text: var(--isa-p-slate-900);
  --isa-text-secondary: var(--isa-p-slate-500);
  --isa-text-muted: var(--isa-p-slate-500);
  --isa-text-on-accent: var(--isa-p-white);
  --isa-text-link: var(--isa-p-blue-600);
  --isa-border: var(--isa-p-slate-200);
  --isa-border-strong: var(--isa-p-slate-300);
  --isa-accent: var(--isa-p-navy-700);
  --isa-accent-active: var(--isa-p-navy-800);
  --isa-accent-soft: var(--isa-p-navy-700-a10);
  --isa-accent-text: var(--isa-p-navy-700);
  --isa-focus-ring: var(--isa-p-blue-600);
  --isa-op-filter: var(--isa-p-blue-600);
  --isa-op-filter-soft: var(--isa-p-blue-100);
  --isa-op-transform: var(--isa-p-violet-600);
  --isa-op-transform-soft: var(--isa-p-violet-100);
  --isa-op-merge: var(--isa-p-orange-600);
  --isa-op-merge-soft: var(--isa-p-orange-100);
  --isa-op-output: var(--isa-p-green-600);
  --isa-op-output-soft: var(--isa-p-green-100);
  --isa-dataset-fill: var(--isa-p-navy-700);
  --isa-warning: var(--isa-p-amber-500);
  --isa-warning-ring: var(--isa-p-slate-50);
  --isa-error: var(--isa-p-red-600);
  --isa-success: var(--isa-p-green-600);
  --isa-radius-control: calc(var(--radius) - 2px);
  --isa-radius-panel: calc(var(--radius) + 4px);
  --isa-radius-node-op: 18px;
  --isa-radius-node-fill: 22px;
  --isa-radius-pill: 999px;
  --isa-radius-xs: 2px;
  --isa-radius-sm: 4px;
  --isa-shadow-glass:
    0 1px 2px 0 var(--isa-p-slate-900-a8), 0 6px 16px -10px var(--isa-p-slate-900-a14);
  --isa-shadow-raised:
    0 1px 3px 0 var(--isa-p-slate-900-a8), 0 8px 20px -12px var(--isa-p-slate-900-a14);
  --isa-blur-glass: 6px;
  --isa-duration-fast: 100ms;
  --isa-duration-base: 160ms;
  --isa-duration-slow: 260ms;

  /* ── Semantici del canvas ETL ── */
  /* ── Gesti (Fase 5) ── */
  --isa-drop-merge: var(--isa-select);
  --isa-drop-link: var(--isa-p-green-600);
  --isa-drop-link-reverse: var(--isa-drop-link);
  --isa-drop-displace: var(--isa-p-orange-600);
  --isa-drop-reject: var(--isa-p-red-600);
  --isa-drop-insert: var(--isa-select);
  --isa-doomed: var(--isa-drop-reject);
  --isa-marquee-line: var(--isa-select);
  --isa-marquee-fill: var(--isa-mm-view-bg);
  --isa-temp-link: var(--isa-accent);
  --isa-temp-link-muted: var(--isa-accent-soft-2);
  --isa-port-fill: var(--isa-p-white);
  --isa-port-line: var(--isa-accent);
  --isa-danger: var(--isa-p-red-600);
  --isa-text-on-danger: var(--isa-p-white);
  --isa-shadow-drag: 0 12px 24px -12px var(--isa-p-slate-900-a14);
  --isa-shadow-overlay: 0 24px 48px -24px var(--isa-p-slate-900-a14);
  --isa-ring-select: 0 0 0 3px var(--isa-select);
  --isa-ring-drop-merge: 0 0 0 3px var(--isa-drop-merge);
  --isa-ring-drop-link: 0 0 0 3px var(--isa-drop-link);
  --isa-outline-node-op: inset 0 0 0 1.5px var(--isa-tint-border);
  --isa-stage: var(--isa-p-white);
  --isa-accent-soft-2: var(--isa-p-navy-700-a16);
  --isa-tint: var(--isa-p-slate-200);
  --isa-tint-ink: var(--isa-p-navy-700);
  --isa-tint-border: var(--isa-p-slate-500);
  --isa-split-bg: var(--isa-p-slate-100);
  --isa-split-empty: var(--isa-p-slate-200);
  --isa-split-empty-ink: var(--isa-p-slate-600);
  --isa-split-line: var(--isa-p-slate-400);
  --isa-select: var(--isa-p-blue-600);
  --isa-link: var(--isa-p-slate-500);
  --isa-link-dot-tint: var(--isa-p-slate-200);
  --isa-flow: var(--isa-p-blue-600);
  --isa-mm-node: var(--isa-p-slate-500);
  --isa-mm-node-ds: var(--isa-p-navy-700);
  --isa-mm-view-line: var(--isa-p-blue-600);
  --isa-mm-view-bg: var(--isa-p-blue-600-a12);

  /* ── Inspector (Fase 6b.1): campi, voci di menu, etichette, caselle, scorrimento ── */
  --isa-field-bg: var(--isa-surface-overlay);
  --isa-field-border: var(--isa-text-muted);
  --isa-field-placeholder: var(--isa-text-secondary);
  --isa-option-hover: var(--isa-accent-soft);
  --isa-option-selected: var(--isa-accent-soft);
  --isa-chip-bg: var(--isa-accent-soft);
  --isa-chip-ink: var(--isa-text);
  --isa-chip-free-border: var(--isa-text-secondary);
  --isa-check-border: var(--isa-text-secondary);
  --isa-check-fill: var(--isa-accent);
  --isa-check-mark: var(--isa-text-on-accent);
  --isa-scroll-thumb: var(--isa-text-muted);
  --isa-lock-ink: var(--isa-text-secondary);
  --isa-scrim: var(--isa-p-black-a40);
}

:root.dark[data-theme="notte"] {
  /* ── Token dell'app ── */
  --background: var(--isa-p-night-950);
  --foreground: var(--isa-p-slate-100);
  --card: var(--isa-p-night-900);
  --card-foreground: var(--isa-p-slate-100);
  --popover: var(--isa-p-night-800);
  --popover-foreground: var(--isa-p-slate-100);
  --primary: var(--isa-p-accent-300);
  --primary-foreground: var(--isa-p-night-950);
  --secondary: var(--isa-p-night-700);
  --secondary-foreground: var(--isa-p-slate-100);
  --muted: var(--isa-p-night-700);
  --muted-foreground: var(--isa-p-slate-400);
  --accent: var(--isa-p-accent-300-a16);
  --accent-foreground: var(--isa-p-slate-100);
  --destructive: var(--isa-p-red-400);
  --destructive-foreground: var(--isa-p-night-950);
  --border: var(--isa-p-slate-400-a18);
  --input: var(--isa-p-slate-400-a32);
  --ring: var(--isa-p-blue-400);
  --brand: var(--isa-p-accent-300);
  --brand-foreground: var(--isa-p-night-950);
  --brand-glow: var(--isa-p-blue-400);
  --success: var(--isa-p-green-400);
  --warning: var(--isa-p-amber-400);
  --glass: var(--isa-p-night-900);
  --glass-strong: var(--isa-p-night-800);
  --glass-border: var(--isa-p-slate-400-a18);
  --chart-1: var(--isa-p-blue-400);
  --chart-2: var(--isa-p-violet-400);
  --chart-3: var(--isa-p-orange-400);
  --chart-4: var(--isa-p-green-400);
  --chart-5: var(--isa-p-amber-400);
  --blob-1: var(--isa-p-night-700);
  --blob-2: var(--isa-p-night-800);
  --blob-3: var(--isa-p-night-900);
  --type-etl-bg: var(--isa-p-blue-400-a16);
  --type-etl-fg: var(--isa-p-blue-400);
  --type-model-bg: var(--isa-p-violet-400-a16);
  --type-model-fg: var(--isa-p-violet-400);
  --type-dashboard-bg: var(--isa-p-green-400-a16);
  --type-dashboard-fg: var(--isa-p-green-400);
  --shadow-glass: 0 12px 32px -16px var(--isa-p-black-a40);
  --sidebar: var(--isa-p-night-900);
  --sidebar-foreground: var(--isa-p-slate-100);
  --sidebar-primary: var(--isa-p-accent-300);
  --sidebar-primary-foreground: var(--isa-p-night-950);
  --sidebar-accent: var(--isa-p-white-a8);
  --sidebar-accent-foreground: var(--isa-p-slate-100);
  --sidebar-border: var(--isa-p-slate-400-a18);
  --sidebar-ring: var(--isa-p-blue-400);

  /* ── Semantici condivisi (--isa-*), modo scuro ── */
  --isa-surface-base: var(--isa-p-night-950);
  --isa-surface-raised: var(--isa-p-night-900);
  --isa-surface-overlay: var(--isa-p-night-800);
  --isa-text: var(--isa-p-slate-100);
  --isa-text-secondary: var(--isa-p-slate-400);
  --isa-text-muted: var(--isa-p-slate-400);
  --isa-text-on-accent: var(--isa-p-night-950);
  --isa-text-link: var(--isa-p-blue-400);
  --isa-border: var(--isa-p-slate-400-a18);
  --isa-border-strong: var(--isa-p-slate-400-a32);
  --isa-accent: var(--isa-p-accent-300);
  --isa-accent-active: var(--isa-p-accent-400);
  --isa-accent-soft: var(--isa-p-accent-300-a16);
  --isa-accent-text: var(--isa-p-accent-300);
  --isa-focus-ring: var(--isa-p-blue-400);
  --isa-op-filter: var(--isa-p-blue-400);
  --isa-op-filter-soft: var(--isa-p-blue-400-a16);
  --isa-op-transform: var(--isa-p-violet-400);
  --isa-op-transform-soft: var(--isa-p-violet-400-a16);
  --isa-op-merge: var(--isa-p-orange-400);
  --isa-op-merge-soft: var(--isa-p-orange-400-a16);
  --isa-op-output: var(--isa-p-green-400);
  --isa-op-output-soft: var(--isa-p-green-400-a16);
  --isa-dataset-fill: var(--isa-p-accent-300);
  --isa-warning: var(--isa-p-amber-400);
  --isa-warning-ring: var(--isa-p-night-950);
  --isa-error: var(--isa-p-red-400);
  --isa-success: var(--isa-p-green-400);
  --isa-shadow-glass: 0 10px 24px -14px var(--isa-p-black-a40);
  --isa-shadow-raised: 0 12px 32px -16px var(--isa-p-black-a40);

  /* ── Semantici del canvas ETL, modo scuro ── */
  --isa-stage: var(--isa-p-white-a8);
  --isa-accent-soft-2: var(--isa-p-accent-300-a28);
  --isa-tint: var(--isa-p-night-700);
  --isa-tint-ink: var(--isa-p-accent-300);
  --isa-tint-border: var(--isa-p-slate-500);
  --isa-split-bg: var(--isa-p-night-800);
  --isa-split-empty: var(--isa-p-night-700);
  --isa-split-empty-ink: var(--isa-p-slate-400);
  --isa-split-line: var(--isa-p-slate-400);
  --isa-select: var(--isa-p-blue-400);
  --isa-link: var(--isa-p-slate-400);
  --isa-link-dot-tint: var(--isa-p-night-700);
  --isa-flow: var(--isa-p-blue-400);
  --isa-mm-node: var(--isa-p-slate-500);
  --isa-mm-node-ds: var(--isa-p-accent-300);
  --isa-mm-view-line: var(--isa-p-blue-400);
  --isa-mm-view-bg: var(--isa-p-blue-400-a16);

  /* ── Gesti (Fase 5), modo scuro ── */
  --isa-drop-link: var(--isa-p-green-400);
  --isa-drop-displace: var(--isa-p-orange-400);
  --isa-drop-reject: var(--isa-p-red-400);
  --isa-port-fill: var(--isa-p-night-800);
  --isa-danger: var(--isa-p-red-600);
  --isa-text-on-danger: var(--isa-p-white);
  --isa-shadow-drag: 0 12px 24px -12px var(--isa-p-black-a40);
  --isa-shadow-overlay: 0 24px 48px -24px var(--isa-p-black-a40);
  --isa-scrim: var(--isa-p-black-a40);
}
```

### `src/theme/themes/prototipo.css`

284 righe

```css
/*
 * Tema "prototipo" (predefinito) — livello 2: assegnazione dei token semantici.
 *
 * I valori sono quelli storici e NON cambiano: l'app (token OKLCH di
 * shadcn: --background, --primary, …) e il canvas ETL (--isa-*, dal
 * prototipo docs/prototype/isa-fusion-prototype.html) restano due tavolozze
 * distinte, messe soltanto sotto la stessa architettura. Perciò qui i valori
 * sono scritti direttamente (non passano da --isa-p-*).
 *
 * `:root` = modo chiaro, `.dark` = modo scuro. Un altro tema si seleziona con
 * `data-theme` su <html> e li sostituisce per intero (themes/notte.css).
 * Il blocco `.dark` ridefinisce solo ciò che cambia col modo; ciò che non
 * cambia (raggi, sfocature, durate) resta quello di `:root`.
 */
:root {
  --radius: 1rem;
  --background: oklch(0.978 0.004 250);
  --foreground: oklch(0.24 0.015 255);
  --card: oklch(1 0 0 / 70%);
  --card-foreground: oklch(0.24 0.015 255);
  --popover: oklch(1 0 0 / 99%);
  --popover-foreground: oklch(0.24 0.015 255);
  --primary: oklch(0.52 0.11 272);
  --primary-foreground: oklch(0.99 0.002 250);
  --secondary: oklch(0.95 0.006 250 / 72%);
  --secondary-foreground: oklch(0.32 0.015 255);
  --muted: oklch(0.955 0.005 250 / 72%);
  --muted-foreground: oklch(0.53 0.012 255);
  --accent: oklch(0.94 0.02 270 / 75%);
  --accent-foreground: oklch(0.32 0.03 270);
  --destructive: oklch(0.6 0.19 22);
  --destructive-foreground: oklch(0.99 0.002 250);
  --border: oklch(0.55 0.012 255 / 14%);
  --input: oklch(0.55 0.012 255 / 18%);
  --ring: oklch(0.58 0.1 272);
  --brand: oklch(0.52 0.11 272);
  --brand-foreground: oklch(0.99 0.002 250);
  --brand-glow: oklch(0.6 0.09 258);
  --success: oklch(0.64 0.11 155);
  --warning: oklch(0.76 0.12 75);
  --glass: oklch(1 0 0 / 48%);
  --glass-strong: oklch(1 0 0 / 66%);
  --glass-border: oklch(1 0 0 / 55%);
  --chart-1: oklch(0.56 0.11 272);
  --chart-2: oklch(0.66 0.09 225);
  --chart-3: oklch(0.62 0.06 255);
  --chart-4: oklch(0.7 0.1 165);
  --chart-5: oklch(0.76 0.11 85);
  --blob-1: oklch(0.86 0.035 265);
  --blob-2: oklch(0.88 0.035 225);
  --blob-3: oklch(0.9 0.02 250);
  --type-etl-bg: oklch(0.93 0.035 215 / 62%);
  --type-etl-fg: oklch(0.35 0.09 215);
  --type-model-bg: oklch(0.93 0.035 272 / 62%);
  --type-model-fg: oklch(0.35 0.09 272);
  --type-dashboard-bg: oklch(0.93 0.035 155 / 62%);
  --type-dashboard-fg: oklch(0.32 0.09 155);
  --shadow-glass: 0 18px 45px -22px oklch(0.35 0.01 255 / 28%);
  --sidebar: oklch(1 0 0 / 55%);
  --sidebar-foreground: oklch(0.3 0.015 255);
  --sidebar-primary: oklch(0.52 0.11 272);
  --sidebar-primary-foreground: oklch(0.99 0.002 250);
  --sidebar-accent: oklch(1 0 0 / 72%);
  --sidebar-accent-foreground: oklch(0.3 0.03 270);
  --sidebar-border: oklch(1 0 0 / 60%);
  --sidebar-ring: oklch(0.58 0.1 272);

  /* ── Semantici condivisi (--isa-*): usati da canvas e, in futuro, da tutta l'app ── */
  /* superfici: base (sfondo pagina), rialzata (vetro dei controlli), sovrapposta (menu, popover) */
  --isa-surface-base: #f5f3ee;
  --isa-surface-raised: rgba(255, 255, 255, 0.92);
  --isa-surface-overlay: #ffffff;
  /* testo: principale, secondario (≥ 4,5:1), tenue (decorativo), su accento, collegamento */
  --isa-text: #262420;
  --isa-text-secondary: #6a645a;
  --isa-text-muted: #847e74;
  --isa-text-on-accent: #ffffff;
  --isa-text-link: #6c63ff;
  /* bordi */
  --isa-border: rgba(38, 36, 32, 0.06);
  --isa-border-strong: rgba(38, 36, 32, 0.16);
  /* accento: base, attivo, tenue; come testo; anello di focus */
  --isa-accent: #6c63ff;
  --isa-accent-active: #5a51e6;
  --isa-accent-soft: rgba(108, 99, 255, 0.16);
  --isa-accent-text: #6c63ff;
  --isa-focus-ring: #6c63ff;
  /* famiglie di operazioni (filtra-ordina, trasforma, merge-union, output) e relativa tinta tenue */
  --isa-op-filter: var(--isa-tint-ink);
  --isa-op-filter-soft: var(--isa-tint);
  --isa-op-transform: var(--isa-tint-ink);
  --isa-op-transform-soft: var(--isa-tint);
  --isa-op-merge: var(--isa-tint-ink);
  --isa-op-merge-soft: var(--isa-tint);
  --isa-op-output: var(--isa-tint-ink);
  --isa-op-output-soft: var(--isa-tint);
  /* riempimento del dataset */
  --isa-dataset-fill: #6c63ff;
  /* stati */
  --isa-warning: #e0a23b;
  --isa-warning-ring: #f7f5f1;
  --isa-error: oklch(0.6 0.19 22);
  --isa-success: oklch(0.64 0.11 155);
  /* raggi (derivati da --radius: stessi valori di prima, 14 e 20 px) */
  --isa-radius-control: calc(var(--radius) - 2px);
  --isa-radius-panel: calc(var(--radius) + 4px);
  --isa-radius-node-op: 22px;
  --isa-radius-node-fill: 26px;
  --isa-radius-pill: 999px;
  --isa-radius-xs: 2px;
  --isa-radius-sm: 4px;
  /* ombre, sfocature, durate */
  --isa-shadow-glass: 0 10px 24px -14px rgba(38, 36, 32, 0.4);
  --isa-shadow-raised: 0 18px 45px -22px oklch(0.35 0.01 255 / 28%);
  --isa-blur-glass: 16px;
  --isa-duration-fast: 120ms;
  --isa-duration-base: 200ms;
  --isa-duration-slow: 320ms;

  /* ── Semantici del canvas ETL ── */
  /* ── Gesti (Fase 5): esiti del trascinamento, riquadro, cavo provvisorio, porte, conferma ── */
  --isa-drop-merge: var(--isa-select);
  --isa-drop-link: oklch(0.5 0.12 155);
  --isa-drop-link-reverse: var(--isa-drop-link);
  --isa-drop-displace: oklch(0.52 0.12 65);
  --isa-drop-reject: oklch(0.52 0.19 25);
  --isa-drop-insert: var(--isa-select);
  --isa-doomed: var(--isa-drop-reject);
  --isa-marquee-line: var(--isa-select);
  --isa-marquee-fill: var(--isa-mm-view-bg);
  --isa-temp-link: var(--isa-accent);
  --isa-temp-link-muted: var(--isa-accent-soft-2);
  --isa-port-fill: #ffffff;
  --isa-port-line: var(--isa-accent);
  --isa-danger: #b23a3a;
  --isa-text-on-danger: #ffffff;
  --isa-shadow-drag: 0 18px 30px -14px rgba(38, 36, 32, 0.4);
  --isa-shadow-overlay: 0 40px 80px -30px rgba(38, 36, 32, 0.4);
  --isa-ring-select: 0 0 0 3px var(--isa-select);
  --isa-ring-drop-merge: 0 0 0 3px var(--isa-drop-merge);
  --isa-ring-drop-link: 0 0 0 3px var(--isa-drop-link);
  --isa-outline-node-op: inset 0 0 0 1.5px var(--isa-tint-border);
  --isa-stage: rgba(255, 255, 255, 0.32);
  --isa-accent-soft-2: rgba(108, 99, 255, 0.34);
  --isa-tint: #e1dcf0;
  --isa-tint-ink: #6c63ff;
  --isa-tint-border: rgba(0, 0, 0, 0);
  --isa-split-bg: #efedf7;
  --isa-split-empty: #e6e3f5;
  --isa-split-empty-ink: #8f88c7;
  --isa-split-line: rgba(108, 99, 255, 0.45);
  --isa-select: #6c63ff;
  --isa-link: rgba(108, 99, 255, 0.34);
  --isa-link-dot-tint: #e1dcf0;
  --isa-flow: rgba(108, 99, 255, 0.6);
  --isa-mm-node: #cfc9ef;
  --isa-mm-node-ds: #6c63ff;
  --isa-mm-view-line: #6c63ff;
  --isa-mm-view-bg: rgba(108, 99, 255, 0.08);

  /* ── Inspector (Fase 6b.1): campi, voci di menu, etichette, caselle, scorrimento ── */
  --isa-field-bg: var(--isa-surface-overlay);
  --isa-field-border: var(--isa-text-muted);
  --isa-field-placeholder: var(--isa-text-secondary);
  --isa-option-hover: var(--isa-accent-soft);
  --isa-option-selected: var(--isa-accent-soft);
  --isa-chip-bg: var(--isa-accent-soft);
  --isa-chip-ink: var(--isa-text);
  --isa-chip-free-border: var(--isa-text-secondary);
  --isa-check-border: var(--isa-text-secondary);
  --isa-check-fill: var(--isa-accent);
  --isa-check-mark: var(--isa-text-on-accent);
  --isa-scroll-thumb: var(--isa-text-muted);
  --isa-lock-ink: var(--isa-text-secondary);
  --isa-scrim: rgba(38, 36, 32, 0.32);
}

.dark {
  --background: oklch(0.19 0.008 260);
  --foreground: oklch(0.96 0.004 250);
  --card: oklch(0.3 0.01 260 / 45%);
  --card-foreground: oklch(0.96 0.004 250);
  --popover: oklch(0.23 0.009 260 / 99%);
  --popover-foreground: oklch(0.96 0.004 250);
  --primary: oklch(0.68 0.11 275);
  --primary-foreground: oklch(0.19 0.008 260);
  --secondary: oklch(0.32 0.01 260 / 60%);
  --secondary-foreground: oklch(0.94 0.004 250);
  --muted: oklch(0.32 0.008 260 / 55%);
  --muted-foreground: oklch(0.75 0.008 260);
  --accent: oklch(0.4 0.035 275 / 55%);
  --accent-foreground: oklch(0.95 0.01 270);
  --destructive: oklch(0.65 0.18 22);
  --destructive-foreground: oklch(0.98 0.002 250);
  --border: oklch(1 0 0 / 11%);
  --input: oklch(1 0 0 / 15%);
  --ring: oklch(0.68 0.11 275);
  --brand: oklch(0.66 0.11 275);
  --brand-foreground: oklch(0.98 0.004 250);
  --brand-glow: oklch(0.68 0.08 250);
  --success: oklch(0.72 0.11 155);
  --warning: oklch(0.8 0.12 80);
  --glass: oklch(1 0 0 / 5%);
  --glass-strong: oklch(1 0 0 / 10%);
  --glass-border: oklch(1 0 0 / 11%);
  --chart-1: oklch(0.68 0.11 275);
  --chart-2: oklch(0.72 0.09 225);
  --chart-3: oklch(0.7 0.05 255);
  --chart-4: oklch(0.76 0.1 165);
  --chart-5: oklch(0.82 0.11 85);
  --blob-1: oklch(0.42 0.04 265);
  --blob-2: oklch(0.4 0.04 230);
  --blob-3: oklch(0.38 0.02 255);
  --type-etl-bg: oklch(0.4 0.035 215 / 55%);
  --type-etl-fg: oklch(0.86 0.06 215);
  --type-model-bg: oklch(0.4 0.035 272 / 55%);
  --type-model-fg: oklch(0.88 0.06 272);
  --type-dashboard-bg: oklch(0.4 0.035 155 / 55%);
  --type-dashboard-fg: oklch(0.86 0.06 155);
  --shadow-glass: 0 22px 55px -26px oklch(0 0 0 / 55%);
  --sidebar: oklch(1 0 0 / 6%);
  --sidebar-foreground: oklch(0.94 0.004 250);
  --sidebar-primary: oklch(0.68 0.11 275);
  --sidebar-primary-foreground: oklch(0.19 0.008 260);
  --sidebar-accent: oklch(1 0 0 / 11%);
  --sidebar-accent-foreground: oklch(0.96 0.004 250);
  --sidebar-border: oklch(1 0 0 / 13%);
  --sidebar-ring: oklch(0.68 0.11 275);

  /* ── Semantici condivisi (--isa-*), modo scuro ── */
  --isa-surface-base: #17181d;
  --isa-surface-raised: rgba(36, 37, 45, 0.92);
  --isa-surface-overlay: #24252d;
  --isa-text: #f1f2f5;
  --isa-text-secondary: #a9abb3;
  --isa-text-muted: #a9abb3;
  --isa-text-on-accent: #ffffff;
  --isa-text-link: #a8a3ff;
  --isa-border: rgba(255, 255, 255, 0.11);
  --isa-border-strong: rgba(255, 255, 255, 0.22);
  --isa-accent: #6c63ff;
  --isa-accent-active: #8a83ff;
  --isa-accent-soft: rgba(108, 99, 255, 0.28);
  --isa-accent-text: #a8a3ff;
  --isa-focus-ring: #a8a3ff;
  --isa-dataset-fill: #6c63ff;
  --isa-warning: #e8b34f;
  --isa-warning-ring: #17181d;
  --isa-error: oklch(0.65 0.18 22);
  --isa-success: oklch(0.72 0.11 155);
  --isa-shadow-glass: 0 10px 24px -14px rgba(0, 0, 0, 0.6);
  --isa-shadow-raised: 0 22px 55px -26px oklch(0 0 0 / 55%);

  /* ── Semantici del canvas ETL, modo scuro ── */
  --isa-stage: rgba(255, 255, 255, 0.04);
  --isa-accent-soft-2: rgba(108, 99, 255, 0.5);
  --isa-tint: #3a3670;
  --isa-tint-ink: #d0ccff;
  --isa-tint-border: #7f78e6;
  --isa-split-bg: #2a2843;
  --isa-split-empty: #2e2c4d;
  --isa-split-empty-ink: #a8a3e6;
  --isa-split-line: rgba(168, 163, 255, 0.55);
  --isa-select: #a8a3ff;
  --isa-link: #7f78e6;
  --isa-link-dot-tint: #a8a3ff;
  --isa-flow: rgba(168, 163, 255, 0.9);
  --isa-mm-node: #7b74d9;
  --isa-mm-node-ds: #a8a3ff;
  --isa-mm-view-line: #a8a3ff;
  --isa-mm-view-bg: rgba(168, 163, 255, 0.12);

  /* ── Gesti (Fase 5), modo scuro ── */
  --isa-drop-link: oklch(0.74 0.12 155);
  --isa-drop-displace: var(--isa-warning);
  --isa-drop-reject: oklch(0.72 0.16 22);
  --isa-port-fill: var(--isa-surface-overlay);
  --isa-danger: #b23a3a;
  --isa-text-on-danger: #ffffff;
  --isa-shadow-drag: 0 18px 30px -14px rgba(0, 0, 0, 0.6);
  --isa-shadow-overlay: 0 40px 80px -30px rgba(0, 0, 0, 0.6);
  --isa-scrim: rgba(0, 0, 0, 0.5);
}
```

