# 06-styles.md

File in questo blocco:

- `src/styles.css`

---

### `src/styles.css`

264 righe

```css
@import "@fontsource-variable/manrope/wght.css";
@import "tailwindcss" source(none);
@source "../src";
@import "tw-animate-css";
@import "./theme/index.css";

@custom-variant dark (&:is(.dark *));

/*
 * isa design system — glassmorphism.
 * I token (primitive → semantici per tema e modo) sono in src/theme/ e sono
 * esposti a Tailwind tramite @theme inline.
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

