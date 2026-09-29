# 06-styles.md

File in questo blocco:

- `src/styles.css`

---

### `src/styles.css`

441 righe

```css
@import "@fontsource-variable/manrope/wght.css";
@import "tailwindcss" source(none);
@source "../src";
@import "tw-animate-css";

@custom-variant dark (&:is(.dark *));

/*
 * isa design system — glassmorphism.
 * Tokens live in :root / .dark and are exposed to Tailwind via @theme inline.
 */

@theme inline {
  --font-sans: "Manrope Variable", "Manrope", system-ui, sans-serif;
  --radius-sm: calc(var(--radius) - 4px);
  --radius-md: calc(var(--radius) - 2px);
  --radius-lg: var(--radius);
  --radius-xl: calc(var(--radius) + 4px);
  --radius-2xl: calc(var(--radius) + 8px);
  --radius-3xl: calc(var(--radius) + 12px);
  --radius-4xl: calc(var(--radius) + 16px);
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-popover: var(--popover);
  --color-popover-foreground: var(--popover-foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);
  --color-destructive: var(--destructive);
  --color-destructive-foreground: var(--destructive-foreground);
  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);
  --color-ring-offset-background: var(--background);
  --color-chart-1: var(--chart-1);
  --color-chart-2: var(--chart-2);
  --color-chart-3: var(--chart-3);
  --color-chart-4: var(--chart-4);
  --color-chart-5: var(--chart-5);
  --color-glass: var(--glass);
  --color-glass-strong: var(--glass-strong);
  --color-glass-border: var(--glass-border);
  --color-brand: var(--brand);
  --color-brand-foreground: var(--brand-foreground);
  --color-brand-glow: var(--brand-glow);
  --color-success: var(--success);
  --color-warning: var(--warning);
  --color-sidebar: var(--sidebar);
  --color-sidebar-foreground: var(--sidebar-foreground);
  --color-sidebar-primary: var(--sidebar-primary);
  --color-sidebar-primary-foreground: var(--sidebar-primary-foreground);
  --color-sidebar-accent: var(--sidebar-accent);
  --color-sidebar-accent-foreground: var(--sidebar-accent-foreground);
  --color-sidebar-border: var(--sidebar-border);
  --color-sidebar-ring: var(--sidebar-ring);
}

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

  /*
   * Primitive condivise (--isa-*). Valori del canvas ETL promossi a primitive:
   * il canvas le legge tramite i propri token (--ec-*, etl-canvas/tokens.css).
   * I token dell'app sopra NON derivano ancora da qui: verranno riportati
   * sulle primitive nel restyling della palette.
   */
  --isa-bg: #f5f3ee;
  --isa-stage: rgba(255, 255, 255, 0.32);
  --isa-surface-strong: rgba(255, 255, 255, 0.92);
  --isa-panel-border: rgba(38, 36, 32, 0.06);
  --isa-ink: #262420;
  --isa-muted: #847e74;
  --isa-empty-ink: #6a645a;
  --isa-accent: #6c63ff;
  --isa-accent-text: #6c63ff;
  --isa-accent-soft: rgba(108, 99, 255, 0.16);
  --isa-accent-soft-2: rgba(108, 99, 255, 0.34);
  --isa-tint: #e1dcf0;
  --isa-tint-ink: #6c63ff;
  --isa-tint-border: rgba(0, 0, 0, 0);
  --isa-on-accent: #ffffff;
  --isa-split-bg: #efedf7;
  --isa-split-empty: #e6e3f5;
  --isa-split-empty-ink: #8f88c7;
  --isa-split-line: rgba(108, 99, 255, 0.45);
  --isa-amber: #e0a23b;
  --isa-amber-ring: #f7f5f1;
  --isa-select: #6c63ff;
  --isa-link: rgba(108, 99, 255, 0.34);
  --isa-link-dot-tint: #e1dcf0;
  --isa-flow: rgba(108, 99, 255, 0.6);
  --isa-mm-node: #cfc9ef;
  --isa-mm-node-ds: #6c63ff;
  --isa-mm-view-line: #6c63ff;
  --isa-mm-view-bg: rgba(108, 99, 255, 0.08);
  --isa-glass-shadow: 0 10px 24px -14px rgba(38, 36, 32, 0.4);
  --isa-r-node-op: 22px;
  --isa-r-node-fill: 26px;
  --isa-glass-blur: 16px;
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

  /* Primitive condivise (--isa-*), tema scuro. */
  --isa-bg: #17181d;
  --isa-stage: rgba(255, 255, 255, 0.04);
  --isa-surface-strong: rgba(36, 37, 45, 0.92);
  --isa-panel-border: rgba(255, 255, 255, 0.11);
  --isa-ink: #f1f2f5;
  --isa-muted: #a9abb3;
  --isa-empty-ink: #a9abb3;
  --isa-accent: #6c63ff;
  --isa-accent-text: #a8a3ff;
  --isa-accent-soft: rgba(108, 99, 255, 0.28);
  --isa-accent-soft-2: rgba(108, 99, 255, 0.5);
  --isa-tint: #3a3670;
  --isa-tint-ink: #d0ccff;
  --isa-tint-border: #7f78e6;
  --isa-on-accent: #ffffff;
  --isa-split-bg: #2a2843;
  --isa-split-empty: #2e2c4d;
  --isa-split-empty-ink: #a8a3e6;
  --isa-split-line: rgba(168, 163, 255, 0.55);
  --isa-amber: #e8b34f;
  --isa-amber-ring: #17181d;
  --isa-select: #a8a3ff;
  --isa-link: #7f78e6;
  --isa-link-dot-tint: #a8a3ff;
  --isa-flow: rgba(168, 163, 255, 0.9);
  --isa-mm-node: #7b74d9;
  --isa-mm-node-ds: #a8a3ff;
  --isa-mm-view-line: #a8a3ff;
  --isa-mm-view-bg: rgba(168, 163, 255, 0.12);
  --isa-glass-shadow: 0 10px 24px -14px rgba(0, 0, 0, 0.6);
}

@layer base {
  * {
    box-sizing: border-box;
    border-color: var(--border);
    user-select: none;
    -webkit-user-select: none;
  }

  body {
    margin: 0;
    background-color: var(--background);
    color: var(--foreground);
    font-family: var(--font-sans);
    -webkit-font-smoothing: antialiased;
    user-select: none;
    -webkit-user-select: none;
    -webkit-user-drag: none;
  }

  button,
  a,
  input,
  textarea,
  select,
  option,
  [draggable="true"] {
    user-select: none;
    -webkit-user-select: none;
    -webkit-user-drag: none;
  }
}

/* Frosted surface */
@utility glass-panel {
  background: var(--glass);
  border: 1px solid var(--glass-border);
  backdrop-filter: blur(40px) saturate(125%);
  -webkit-backdrop-filter: blur(40px) saturate(125%);
  box-shadow: var(--shadow-glass);
}

@utility glass-soft {
  background: var(--glass);
  border: 1px solid var(--glass-border);
  backdrop-filter: blur(28px) saturate(120%);
  -webkit-backdrop-filter: blur(28px) saturate(120%);
}

@utility glass-chip {
  background: var(--glass-strong);
  border: 1px solid var(--glass-border);
  backdrop-filter: blur(22px) saturate(120%);
  -webkit-backdrop-filter: blur(22px) saturate(120%);
}

@utility badge-type-etl {
  background: var(--type-etl-bg);
  color: var(--type-etl-fg);
}

@utility badge-type-model {
  background: var(--type-model-bg);
  color: var(--type-model-fg);
}

@utility badge-type-dashboard {
  background: var(--type-dashboard-bg);
  color: var(--type-dashboard-fg);
}

@utility gradient-brand {
  background-image: linear-gradient(135deg, var(--brand) 0%, var(--brand-glow) 100%);
}

@utility text-gradient-brand {
  background-image: linear-gradient(120deg, var(--brand) 0%, var(--brand-glow) 100%);
  background-clip: text;
  color: transparent;
}

@utility blob {
  position: absolute;
  border-radius: 9999px;
  filter: blur(130px);
  opacity: 0.35;
  pointer-events: none;
}

@utility scroll-slim {
  scrollbar-width: thin;

  &::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }

  &::-webkit-scrollbar-thumb {
    background: var(--glass-border);
    border-radius: 9999px;
  }
}

@keyframes isa-dash {
  to {
    stroke-dashoffset: -24px;
  }
}

@utility edge-flow {
  stroke-dasharray: 6 7;
  animation: isa-dash 0.9s linear infinite;
}

/*
 * La transizione del path delle frecce (fase 3) è applicata inline in
 * workflow-canvas.tsx con le costanti di lib/etl-motion.ts — condivise
 * con card e bubble, invece di un valore fisso qui — così può essere
 * disattivata per gli edge agganciati a un nodo sotto drag diretto
 * dell'utente (che deve restare 1:1 col puntatore, senza transizione).
 * `d` non è interpolabile in tutti i browser (specialmente se il
 * numero di comandi del path cambia tra un render e l'altro): dove non
 * supportato, degrada senza errori a un aggiornamento istantaneo.
 */

@keyframes isa-edge-flow-dot {
  0% {
    offset-distance: 0%;
    opacity: 0;
  }
  10% {
    opacity: var(--edge-flow-peak, 0.8);
  }
  88% {
    opacity: var(--edge-flow-peak, 0.8);
  }
  100% {
    offset-distance: 100%;
    opacity: 0;
  }
}

/*
 * Piccola "particella luminosa" che percorre il path della freccia
 * nella direzione del flusso dati (da from a to — vedi il commento
 * nel componente). offset-path segue nativamente le curve del path,
 * quindi l'animazione è gestita interamente dal browser: nessun
 * requestAnimationFrame, nessun ricalcolo del path per frame.
 */
@utility edge-flow-dot {
  offset-rotate: 0deg;
  filter: drop-shadow(0 0 3px var(--brand-glow));
  animation: isa-edge-flow-dot 2.6s linear infinite;
}

@utility node-selected {
  border-color: color-mix(in oklab, var(--brand) 55%, var(--glass-border));

  box-shadow:
    0 0 0 1px color-mix(in oklab, var(--brand) 45%, transparent),
    0 0 24px color-mix(in oklab, var(--brand) 28%, transparent);
}

/* Card evidenziata come destinazione di un collegamento in corso */
@utility node-link-target {
  border-color: color-mix(in oklab, var(--brand) 70%, var(--glass-border));

  box-shadow:
    0 0 0 2px var(--brand),
    0 0 26px color-mix(in oklab, var(--brand) 35%, transparent);
}

/*
 * Card creata con doppio click dalla palette (PARTE B) e non ancora
 * "raccolta" con il primo pointerdown — indicatore statico, nessuna
 * animazione di pulse (verrà aggiunta in un passaggio dedicato).
 * `outline` invece di `border-color` così non confligge con
 * node-selected/node-link-target, che possono comparire insieme.
 */
@utility node-pending {
  outline: 2px dashed color-mix(in oklab, var(--brand) 55%, transparent);
  outline-offset: 2px;
}

/* Canvas interaction */
.isa-canvas {
  user-select: none;
  -webkit-user-select: none;
  -webkit-user-drag: none;
}

.isa-canvas *,
.isa-canvas *::before,
.isa-canvas *::after {
  user-select: none;
  -webkit-user-select: none;
  -webkit-user-drag: none;
}
```

