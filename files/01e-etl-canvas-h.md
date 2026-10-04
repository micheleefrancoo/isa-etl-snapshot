# 01e-etl-canvas-h.md

File in questo blocco:

- `src/etl-canvas/inspector/inspector.css`
- `src/etl-canvas/inspector/logic.ts`
- `src/etl-canvas/inspector/menu.ts`
- `src/etl-canvas/inspector/params.ts`
- `src/etl-canvas/inspector/useActiveSchema.ts`
- `src/etl-canvas/interaction.ts`
- `src/etl-canvas/loop.ts`

---

### `src/etl-canvas/inspector/inspector.css`

714 righe

```css
/*
 * Inspector (Fase 6b.1). Solo token: [REDATTO], raggi e ombre dai token semantici
 * (--isa-*, Fase T); font-size, margin, padding e gap dalla scala di
 * src/theme/layout-tokens.css (scripts/check-tokens.mjs lo controlla).
 * I menu vivono in un portale sul corpo della pagina (`#ei-portal`), quindi non
 * dipendono dai token del canvas (--ec-*).
 */

/* il pannello dell'Inspector: margine interno e scorrimento a token */
.ec-tb-inner.ec-insp {
  padding: var(--isa-space-5);
  gap: var(--isa-space-4);
  scrollbar-width: thin;
  scrollbar-color: var(--isa-scroll-thumb) transparent;
}
.ec-insp .ec-close-btn {
  width: var(--isa-icon-btn);
  height: var(--isa-icon-btn);
}

.ei-root {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-4);
  min-width: 0;
  color: var(--isa-text);
  font-family: inherit;
  line-height: var(--isa-lh-text);
  font-variant-numeric: tabular-nums;
  overflow-wrap: anywhere;
  hyphens: none;
}
.ei-root *,
.ei-menu *,
.ei-expanded * {
  box-sizing: border-box;
}

/* intestazione */
.ei-header {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-1);
}
.ei-overline {
  font-size: var(--isa-fs-overline);
  font-weight: 800;
  letter-spacing: var(--isa-ls-overline);
  text-transform: uppercase;
  color: var(--isa-text-secondary);
}
.ei-name {
  width: 100%;
  min-height: var(--isa-icon-btn);
  padding: 0;
  border: 0;
  border-bottom: 1px dashed var(--isa-field-border);
  background: transparent;
  color: var(--isa-text);
  font: inherit;
  font-size: var(--isa-fs-title);
  font-weight: 800;
  line-height: var(--isa-lh-title);
  overflow-wrap: anywhere;
}
.ei-name:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 2px;
}

/* campo: etichetta e controllo */
.ei-fieldgroup {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-1);
  min-width: 0;
}
.ei-label {
  font-size: var(--isa-fs-label);
  font-weight: 700;
  color: var(--isa-text-secondary);
}
.ei-help {
  font-size: var(--isa-fs-help);
  font-weight: 500;
  line-height: var(--isa-lh-text);
  color: var(--isa-text-secondary);
  text-wrap: pretty;
  overflow-wrap: anywhere;
}
.ei-empty {
  padding: var(--isa-space-2) 0;
}

.ei-field {
  display: flex;
  align-items: center;
  gap: var(--isa-space-2);
  width: 100%;
  min-height: var(--isa-control-h);
  padding: 0 var(--isa-space-3);
  background: var(--isa-field-bg);
  border: 1px solid var(--isa-field-border);
  border-radius: var(--isa-radius-control);
  color: var(--isa-text);
  font: inherit;
  font-size: var(--isa-fs-value);
  font-weight: 600;
  text-align: left;
}
.ei-field:focus-visible,
.ei-field:focus-within {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 1px;
}
.ei-field:disabled {
  color: var(--isa-text-secondary);
  background: transparent;
  border-style: dashed;
}
.ei-input::placeholder,
.ei-search::placeholder {
  color: var(--isa-field-placeholder);
  font-weight: 500;
}
.ei-select {
  cursor: pointer;
  justify-content: space-between;
}
.ei-select-value {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.ei-select-chevron {
  display: flex;
  flex: none;
  color: var(--isa-text-secondary);
  transition: transform var(--isa-duration-fast);
}
.ei-select[data-open] .ei-select-chevron {
  transform: rotate(180deg);
}
.ei-free {
  font-style: italic;
}
.ei-placeholder {
  color: var(--isa-field-placeholder);
  font-size: var(--isa-fs-value);
  font-weight: 500;
}
.ei-icon {
  width: 14px;
  height: 14px;
  flex: none;
}

/* etichette rimovibili (colonne e valori) */
.ei-chipsfield {
  flex-wrap: wrap;
  align-items: center;
  padding: var(--isa-space-2);
  gap: var(--isa-space-2);
  cursor: default;
}
.ei-chips {
  display: contents;
  list-style: none;
  margin: 0;
  padding: 0;
}
.ei-chip {
  display: inline-flex;
  align-items: center;
  max-width: 100%;
  min-height: var(--isa-icon-btn);
  background: var(--isa-chip-bg);
  color: var(--isa-chip-ink);
  border: 1px solid transparent;
  border-radius: var(--isa-radius-pill);
  font-size: var(--isa-fs-label);
  font-weight: 700;
}
.ei-chip.ei-free {
  background: transparent;
  border: 1px dashed var(--isa-chip-free-border);
}
.ei-chip.ei-dragging {
  position: relative;
  z-index: 5;
  box-shadow: var(--isa-shadow-raised);
  pointer-events: none;
}
.ei-chip-label,
.ei-chip-text {
  all: unset;
  padding: 0 var(--isa-space-1) 0 var(--isa-space-3);
  min-width: 0;
  overflow-wrap: anywhere;
  font-size: var(--isa-fs-label);
  font-weight: 700;
}
.ei-chip-label {
  cursor: grab;
  touch-action: none;
}
.ei-chip-label:focus-visible,
.ei-chip-x:focus-visible,
.ei-add:focus-visible,
.ei-icon-btn:focus-visible,
.ei-link-btn:focus-visible,
.ei-addrow:focus-visible,
.ei-row-toggle:focus-visible,
.ei-step-main:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 1px;
}
.ei-chip-x,
.ei-icon-btn {
  all: unset;
  display: grid;
  place-items: center;
  flex: none;
  width: var(--isa-icon-btn);
  height: var(--isa-icon-btn);
  border-radius: var(--isa-radius-pill);
  color: var(--isa-text-secondary);
  cursor: pointer;
}
.ei-chip-x:hover,
.ei-icon-btn:hover {
  color: var(--isa-text);
  background: var(--isa-option-hover);
}
.ei-add {
  all: unset;
  display: inline-flex;
  align-items: center;
  gap: var(--isa-space-1);
  min-height: var(--isa-icon-btn);
  padding: 0 var(--isa-space-3);
  border: 1px dashed var(--isa-field-border);
  border-radius: var(--isa-radius-pill);
  color: var(--isa-text);
  font-size: var(--isa-fs-label);
  font-weight: 700;
  cursor: pointer;
}
.ei-add:hover {
  background: var(--isa-option-hover);
}
.ei-link-btn {
  all: unset;
  display: inline-flex;
  align-items: center;
  min-height: var(--isa-icon-btn);
  padding: 0 var(--isa-space-2);
  color: var(--isa-text);
  font-size: var(--isa-fs-label);
  font-weight: 700;
  text-decoration: underline;
  cursor: pointer;
}
.ei-warn {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: var(--isa-space-2);
  padding: var(--isa-space-1) var(--isa-space-3);
  border-left: 3px solid var(--isa-warning);
  font-size: var(--isa-fs-help);
  font-weight: 500;
  color: var(--isa-text);
}

/* stato bloccato */
.ei-lock {
  display: flex;
  align-items: flex-start;
  gap: var(--isa-space-3);
  padding: var(--isa-space-4);
  border: 1px dashed var(--isa-field-border);
  border-radius: var(--isa-radius-control);
  color: var(--isa-lock-ink);
  font-size: var(--isa-fs-help);
  font-weight: 500;
  line-height: var(--isa-lh-text);
  text-wrap: pretty;
}
.ei-lock .ei-icon {
  width: 16px;
  height: 16px;
  align-self: flex-start;
}

/* liste a voci multiple */
.ei-list {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-2);
  min-width: 0;
}
.ei-row {
  border: 1px solid var(--isa-field-border);
  border-radius: var(--isa-radius-control);
  background: var(--isa-field-bg);
  min-width: 0;
}
.ei-row-head {
  display: flex;
  align-items: center;
  gap: var(--isa-space-1);
  padding-right: var(--isa-space-1);
}
.ei-row-toggle {
  all: unset;
  display: flex;
  flex: 1;
  align-items: center;
  gap: var(--isa-space-2);
  min-width: 0;
  min-height: var(--isa-control-h);
  padding: 0 var(--isa-space-3);
  cursor: pointer;
}
.ei-row-chev {
  display: flex;
  flex: none;
  color: var(--isa-text-secondary);
  transition: transform var(--isa-duration-fast);
}
.ei-open > .ei-row-head .ei-row-chev {
  transform: rotate(90deg);
}
.ei-row-n {
  flex: none;
  font-size: var(--isa-fs-label);
  font-weight: 700;
  color: var(--isa-text-secondary);
}
.ei-row-sum {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: var(--isa-fs-summary);
  font-weight: 700;
}
.ei-row-sum.ei-todo {
  font-weight: 500;
  color: var(--isa-text-secondary);
}
.ei-row-body {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-4);
  padding: var(--isa-space-3) var(--isa-space-3) var(--isa-space-4);
  border-top: 1px solid var(--isa-border-strong);
}
.ei-addrow {
  all: unset;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: var(--isa-control-h);
  border: 1px dashed var(--isa-field-border);
  border-radius: var(--isa-radius-control);
  color: var(--isa-text);
  font-size: var(--isa-fs-label);
  font-weight: 700;
  cursor: pointer;
}
.ei-addrow:hover {
  background: var(--isa-option-hover);
}

/* elenco dei passaggi di un box combinato */
.ei-steps {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-2);
}
.ei-steplist {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-1);
  list-style: none;
  margin: 0;
  padding: 0;
}
.ei-step {
  display: flex;
  align-items: center;
  gap: var(--isa-space-1);
  border-radius: var(--isa-radius-control);
  touch-action: none;
  transition: transform var(--isa-duration-base);
}
.ei-step.ei-on {
  background: var(--isa-option-selected);
}
.ei-step.ei-dragging {
  position: relative;
  z-index: 5;
  transition: none;
  background: var(--isa-option-selected);
  box-shadow: var(--isa-shadow-raised);
}
.ei-step.ei-outside {
  opacity: 0.6;
}
.ei-step-main {
  all: unset;
  display: flex;
  flex: 1;
  align-items: center;
  gap: var(--isa-space-2);
  min-width: 0;
  min-height: var(--isa-control-h);
  padding: 0 var(--isa-space-2);
  font-size: var(--isa-fs-summary);
  font-weight: 700;
  cursor: grab;
}
.ei-grip {
  display: flex;
  color: var(--isa-text-secondary);
}
.ei-step-n {
  font-size: var(--isa-fs-label);
  font-weight: 700;
  color: var(--isa-text-secondary);
}
.ei-step-icon {
  display: flex;
  color: var(--isa-text);
}
.ei-step-icon svg {
  width: 16px;
  height: 16px;
}
.ei-step-name {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.ei-outside-note {
  text-align: center;
}

/* colonne in sola lettura (dataset e output) */
.ei-colist {
  display: flex;
  flex-direction: column;
  list-style: none;
  margin: 0;
  padding: 0;
}
.ei-colrow {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: var(--isa-space-3);
  padding: var(--isa-space-1) 0;
  border-bottom: 1px solid var(--isa-border-strong);
  font-size: var(--isa-fs-value);
  font-weight: 600;
}
.ei-coltype {
  flex: none;
  font-size: var(--isa-fs-help);
  font-weight: 500;
  color: var(--isa-text-secondary);
}

/* menu e tendine (in un portale sul corpo della pagina) */
.ei-portal {
  position: fixed;
  inset: 0;
  z-index: 100;
  pointer-events: none;
}
.ei-menu {
  position: fixed;
  z-index: 120;
  display: flex;
  flex-direction: column;
  padding: var(--isa-menu-pad);
  background: var(--isa-surface-overlay);
  color: var(--isa-text);
  border: 1px solid var(--isa-border-strong);
  border-radius: var(--isa-radius-panel);
  box-shadow: var(--isa-shadow-raised);
  overflow: hidden;
  font-family: inherit;
  font-variant-numeric: tabular-nums;
  pointer-events: auto;
  animation: ei-enter var(--isa-menu-enter) ease-out;
}
@keyframes ei-enter {
  from {
    opacity: 0;
    transform: translateY(-4px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}
.ei-menu[data-side="above"] {
  animation-name: ei-enter-above;
}
@keyframes ei-enter-above {
  from {
    opacity: 0;
    transform: translateY(4px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}
@media (prefers-reduced-motion: reduce) {
  .ei-menu {
    animation: none;
  }
  .ei-select-chevron,
  .ei-row-chev,
  .ei-step {
    transition: none;
  }
}
.ei-menu-head {
  flex: none;
  padding-bottom: var(--isa-space-2);
}
.ei-search {
  width: 100%;
  min-height: var(--isa-control-h);
  padding: 0 var(--isa-space-3);
  background: var(--isa-field-bg);
  border: 1px solid var(--isa-field-border);
  border-radius: var(--isa-radius-control);
  color: var(--isa-text);
  font: inherit;
  font-size: var(--isa-fs-value);
  font-weight: 600;
}
.ei-search:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 1px;
}
.ei-menu-scroll {
  display: flex;
  flex: 1 1 auto;
  flex-direction: column;
  min-height: 0;
  overflow-y: auto;
  overscroll-behavior: contain;
  scrollbar-width: thin;
  scrollbar-color: var(--isa-scroll-thumb) transparent;
}
.ei-values-list {
  max-height: 170px;
}
.ei-option {
  all: unset;
  display: flex;
  flex: none;
  align-items: center;
  gap: var(--isa-space-3);
  min-height: var(--isa-menu-item-h);
  padding: 0 var(--isa-menu-item-px);
  border-radius: var(--isa-radius-control);
  font-size: var(--isa-fs-value);
  font-weight: 600;
  cursor: pointer;
  box-sizing: border-box;
}
.ei-option.ei-active {
  background: var(--isa-option-hover);
}
.ei-option.ei-selected {
  background: var(--isa-option-selected);
  font-weight: 800;
}
.ei-option.ei-danger {
  color: var(--isa-error);
}
.ei-option:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: -2px;
}
.ei-option-label {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.ei-option-hint {
  flex: none;
  font-size: var(--isa-fs-help);
  font-weight: 500;
  color: var(--isa-text-secondary);
}
.ei-checkbox {
  display: grid;
  place-items: center;
  flex: none;
  width: 20px;
  height: 20px;
  border: 1px solid var(--isa-check-border);
  border-radius: var(--isa-radius-sm);
  color: var(--isa-check-mark);
}
.ei-checkbox[data-on] {
  background: var(--isa-check-fill);
  border-color: var(--isa-check-fill);
}
.ei-checkbox .ei-icon {
  width: 14px;
  height: 14px;
}
.ei-menu-foot {
  display: flex;
  flex: none;
  align-items: center;
  justify-content: space-between;
  gap: var(--isa-space-2);
  padding: var(--isa-space-2) var(--isa-space-3) 0;
}
.ei-menu-actions {
  display: flex;
  flex: none;
  gap: var(--isa-space-1);
}
.ei-count {
  white-space: nowrap;
  font-size: var(--isa-fs-help);
  font-weight: 500;
  color: var(--isa-text-secondary);
}
.ei-menu-empty {
  padding: var(--isa-space-3);
  font-size: var(--isa-fs-help);
  color: var(--isa-text-secondary);
}
.ei-visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
}

/* bordi alto e basso: gli stessi contenuti in colonne (prima del master-detail della 6b.2) */
.ec-panel.ec-horiz .ec-tb-inner.ec-insp {
  overflow-y: hidden;
  overflow-x: auto;
}
.ec-panel.ec-horiz .ei-root {
  display: block;
  height: 100%;
  column-width: 280px;
  column-gap: var(--isa-space-6);
  column-fill: auto;
}
.ec-panel.ec-horiz .ei-root > * {
  break-inside: avoid;
  margin-bottom: var(--isa-space-4);
}

/* pannello espanso di un box combinato */
.ei-expanded-backdrop {
  position: fixed;
  inset: 0;
  z-index: 110;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--isa-space-6);
  background: var(--isa-scrim);
  pointer-events: auto;
}
.ei-expanded {
  width: 100%;
  max-width: 420px;
  max-height: 100%;
  overflow: auto;
  padding: var(--isa-space-6);
  background: var(--isa-surface-overlay);
  border: 1px solid var(--isa-border-strong);
  border-radius: var(--isa-radius-panel);
  box-shadow: var(--isa-shadow-overlay);
  scrollbar-width: thin;
  scrollbar-color: var(--isa-scroll-thumb) transparent;
}
.ei-expanded-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: var(--isa-space-3);
}
.ei-expanded-head > div:first-child {
  flex: 1;
  min-width: 0;
}
```

### `src/etl-canvas/inspector/logic.ts`

218 righe

```ts
/**
 * Logica pura dei selettori dell'Inspector (nessun DOM, nessun React): ricerca,
 * navigazione da tastiera, colonne scelte e loro ordine, valori scelti,
 * riordino dei passaggi. I componenti la usano e i test la provano da soli.
 */
import { splitTokens } from "../../etl-core";
import type { ValuesField } from "../../etl-core";

// --- ricerca e navigazione ----------------------------------------------------

/** Minuscole e senza accenti: la ricerca non distingue. */
export function fold(text: string): string {
  return text.normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase();
}

/** Le voci la cui etichetta contiene la ricerca (vuota = tutte), nell'ordine dato. */
export function filterByQuery<T extends { readonly label: string }>(
  items: readonly T[],
  query: string,
): T[] {
  const q = fold(query.trim());
  return q ? items.filter((i) => fold(i.label).includes(q)) : items.slice();
}

export type NavKey = "ArrowDown" | "ArrowUp" | "Home" | "End";

/** Voce attiva dopo un tasto di navigazione (nessun giro: si ferma alle estremità). `-1` = nessuna. */
export function nextActive(current: number, count: number, key: NavKey): number {
  if (count <= 0) return -1;
  switch (key) {
    case "Home":
      return 0;
    case "End":
      return count - 1;
    case "ArrowDown":
      return current < 0 ? 0 : Math.min(count - 1, current + 1);
    case "ArrowUp":
      return current < 0 ? count - 1 : Math.max(0, current - 1);
  }
}

export type ComboAction =
  | { readonly kind: "move"; readonly to: number }
  | { readonly kind: "commit" }
  | { readonly kind: "close" }
  | { readonly kind: "none" };

/**
 * Cosa fa un tasto nel campo di ricerca di una tendina (combobox ARIA): frecce, Home e
 * Fine spostano la voce attiva, Invio conferma, Esc chiude; il resto è digitazione.
 */
export function comboAction(key: string, active: number, count: number): ComboAction {
  if (key === "ArrowDown" || key === "ArrowUp" || key === "Home" || key === "End") {
    return { kind: "move", to: nextActive(active, count, key) };
  }
  if (key === "Enter") return { kind: "commit" };
  if (key === "Escape") return { kind: "close" };
  return { kind: "none" };
}

/** Mantiene valida la voce attiva quando l'elenco cambia (ricerca). */
export function clampActive(current: number, count: number): number {
  if (count <= 0) return -1;
  return current < 0 ? 0 : Math.min(current, count - 1);
}

// --- colonne scelte -----------------------------------------------------------

/** Spunta o toglie una colonna: la nuova va in fondo, l'ordine delle altre resta. */
export function toggleColumn(selected: readonly string[], name: string): string[] {
  return selected.includes(name) ? selected.filter((c) => c !== name) : [...selected, name];
}

/** «Tutte»: aggiunge le colonne visibili non ancora scelte, nell'ordine in cui sono mostrate. */
export function addVisible(selected: readonly string[], visible: readonly string[]): string[] {
  return [...selected, ...visible.filter((c) => !selected.includes(c))];
}

/** «Nessuna»: toglie le colonne visibili, le altre restano nel loro ordine. */
export function removeVisible(selected: readonly string[], visible: readonly string[]): string[] {
  return selected.filter((c) => !visible.includes(c));
}

/** Sposta l'elemento `from` nella posizione `to` (indici già validi); lista nuova. */
export function moveItem<T>(list: readonly T[], from: number, to: number): T[] {
  if (from === to || from < 0 || to < 0 || from >= list.length || to >= list.length) {
    return list.slice();
  }
  const next = list.slice();
  const [item] = next.splice(from, 1);
  next.splice(to, 0, item as T);
  return next;
}

/** Nuova posizione per Alt+freccia (sinistra/su = −1, destra/giù = +1); `null` se non si muove. */
export function moveTarget(index: number, count: number, key: string): number | null {
  const delta =
    key === "ArrowLeft" || key === "ArrowUp"
      ? -1
      : key === "ArrowRight" || key === "ArrowDown"
        ? 1
        : 0;
  const to = index + delta;
  return delta === 0 || to < 0 || to >= count ? null : to;
}

/** Posizione di rilascio di un trascinamento orizzontale o verticale: indice del segnaposto più vicino. */
export function dropIndex(positions: readonly number[], pointer: number): number {
  let best = 0;
  let bestDistance = Infinity;
  positions.forEach((p, i) => {
    const d = Math.abs(p - pointer);
    if (d < bestDistance) {
      bestDistance = d;
      best = i;
    }
  });
  return best;
}

/** Il nome scritto, con la grafia dei dati se esiste (senza distinguere le maiuscole). */
export function canonicalName(known: readonly string[], typed: string): string {
  const t = typed.trim();
  return known.find((k) => k.toLowerCase() === t.toLowerCase()) ?? t;
}

// --- valori scelti ------------------------------------------------------------

/** Un valore scritto prende la grafia dei dati se esiste, senza distinguere le maiuscole. */
export function canonicalValue(domain: readonly string[], token: [REDATTO] string {
  return domain.find((d) => d.toLowerCase() === token.toLowerCase()) ?? token;
}

/**
 * I valori che «+ Aggiungi» aggiungerebbe: i pezzi del testo (virgola, punto e
 * virgola, barra verticale, a capo) non già scelti. Se resta un solo pezzo e
 * coincide con un valore dei dati, non serve aggiungere: lo mostra la ricerca.
 */
export function pendingTokens(
  values: readonly string[],
  domain: readonly string[],
  text: string,
): string[] {
  const tokens = splitTokens(text).filter(
    (t) => !values.some((v) => v.toLowerCase() === t.toLowerCase()),
  );
  const onlyExisting =
    tokens.length === 1 &&
    domain.some((d) => d.toLowerCase() === (tokens[0] as string).toLowerCase());
  return tokens.length && !onlyExisting ? tokens : [];
}

/** Aggiunge i pezzi del testo ai valori (con la grafia dei dati, senza duplicati). */
export function addTokens(
  values: readonly string[],
  domain: readonly string[],
  text: string,
): string[] {
  const next = values.slice();
  for (const token of splitTokens(text)) {
    const value = canonicalValue(domain, token);
    if (!next.includes(value)) next.push(value);
  }
  return next;
}

export function toggleValue(values: readonly string[], value: string): string[] {
  return values.includes(value) ? values.filter((v) => v !== value) : [...values, value];
}

/** «Tutti»: i valori visibili non ancora scelti si aggiungono in fondo. */
export function addVisibleValues(values: readonly string[], visible: readonly string[]): string[] {
  return [...values, ...visible.filter((v) => !values.includes(v))];
}

/** «Nessuno»: toglie i valori visibili. */
export function removeVisibleValues(
  values: readonly string[],
  visible: readonly string[],
): string[] {
  return values.filter((v) => !visible.includes(v));
}

/** I valori stanno solo in `values`: il campo si riscrive in modalità elenco, senza testo. */
export function withValues(field: ValuesField | undefined, values: readonly string[]): ValuesField {
  return { mode: "list", values: values.slice(), text: "", sep: field?.sep ?? "," };
}

// --- riordino dei passaggi (puntatore) ----------------------------------------

/** Indice di arrivo di una riga trascinata di `dy` pixel (righe alte `rowHeight`). */
export function reorderIndex(
  startIndex: number,
  dy: number,
  rowHeight: number,
  count: number,
): number {
  const idx = Math.round(startIndex + dy / rowHeight);
  return Math.max(0, Math.min(count - 1, idx));
}

/** Indice del centro più vicino al punto (per riordinare etichette che vanno a capo). */
export function nearestIndex(
  centers: readonly { readonly x: number; readonly y: number }[],
  point: { readonly x: number; readonly y: number },
): number {
  let best = 0;
  let bestDistance = Infinity;
  centers.forEach((c, i) => {
    const d = Math.hypot(c.x - point.x, c.y - point.y);
    if (d < bestDistance) {
      bestDistance = d;
      best = i;
    }
  });
  return best;
}
```

### `src/etl-canvas/inspector/menu.ts`

82 righe

```ts
/**
 * Posizionamento delle tendine e dei menu dell'Inspector: funzione pura, senza
 * DOM. Le misure sono quelle dei token di forma (src/theme/layout-tokens.css:
 * --isa-menu-gap, --isa-menu-edge, --isa-menu-max-w); un test le tiene allineate.
 *
 * Regole:
 * - distanza dal campo `MENU_GAP`; margine minimo dai bordi della FINESTRA
 *   `MENU_EDGE` in ogni direzione;
 * - larghezza minima = quella del campo, massima min(MENU_MAX_W, finestra − 2 × MENU_EDGE);
 * - si apre dal lato con più spazio; se lì non entra l'altezza naturale, l'altezza
 *   massima è lo spazio disponibile (già senza il margine) e il menu scorre dentro;
 * - orizzontalmente parte dal bordo sinistro del campo e si sposta a sinistra quanto
 *   basta per tenere il margine a destra (e mai oltre il margine a sinistra).
 */

export const MENU_GAP = 8;
export const MENU_EDGE = 16;
export const MENU_MAX_W = 420;

export interface Box {
  readonly x: number;
  readonly y: number;
  readonly w: number;
  readonly h: number;
}

export interface MenuInput {
  /** Rettangolo del campo, in coordinate della finestra. */
  readonly field: Box;
  /** Dimensioni della finestra. */
  readonly win: { readonly w: number; readonly h: number };
  /** Altezza naturale del menu (tutte le voci, padding compreso). */
  readonly naturalHeight: number;
  /** Larghezza naturale del contenuto, se maggiore di quella del campo. */
  readonly naturalWidth?: number;
}

export interface MenuPlacement {
  readonly side: "below" | "above";
  readonly left: number;
  readonly top: number;
  readonly width: number;
  /** Altezza effettiva: quella naturale, o lo spazio disponibile se non entra. */
  readonly height: number;
  /** Altezza massima da applicare al menu (scorre dentro oltre questa). */
  readonly maxHeight: number;
  /** Il contenuto non entra: serve lo scorrimento interno. */
  readonly scrolls: boolean;
}

/** Il rettangolo che occupa il menu. */
export function menuBox(p: MenuPlacement): Box {
  return { x: p.left, y: p.top, w: p.width, h: p.height };
}

export function placeMenu(input: MenuInput): MenuPlacement {
  const { field, win } = input;
  const maxWidth = Math.max(0, Math.min(MENU_MAX_W, win.w - 2 * MENU_EDGE));
  const wanted = Math.max(field.w, input.naturalWidth ?? 0);
  const width = Math.min(wanted, maxWidth);

  // spazio per il menu: dal campo (più la distanza) al margine della finestra
  const below = win.h - (field.y + field.h) - MENU_GAP - MENU_EDGE;
  const above = field.y - MENU_GAP - MENU_EDGE;
  const side: MenuPlacement["side"] = below >= above ? "below" : "above";
  const space = Math.max(0, side === "below" ? below : above);
  const height = Math.min(input.naturalHeight, space);

  const top = side === "below" ? field.y + field.h + MENU_GAP : field.y - MENU_GAP - height;
  const left = Math.max(MENU_EDGE, Math.min(field.x, win.w - MENU_EDGE - width));

  return {
    side,
    left,
    top,
    width,
    height,
    maxHeight: space,
    scrolls: input.naturalHeight > space,
  };
}
```

### `src/etl-canvas/inspector/params.ts`

85 righe

```ts
/**
 * Lettura e scrittura dei parametri di un passaggio, a funzioni pure: restituiscono
 * parametri NUOVI, mai modificati sul posto. L'Inspector legge e scrive SOLO
 * `columns` (mai le righe espanse di `flattenRows`); le righe a colonna singola
 * del vecchio formato si migrano con `ensureMulti`.
 */
import { MULTI_DEFS, columnsOf, defaultParams, ensureMulti } from "../../etl-core";
import type { MultiListDef, MultiParams, MultiRow, OperationType, Params } from "../../etl-core";

/** Il campo `key` (testo) di parametri semplici. */
export function textParam(par: Params, key: string): string {
  const v = par[key];
  return typeof v === "string" ? v : "";
}

export function withParam(par: Params, key: string, value: string): Params {
  return { ...par, [key]: value };
}

/** I parametri di un'operazione a voci multiple, già nel formato attuale (colonne multiple). */
export function multiOf(type: OperationType, par: Params): MultiParams {
  return ensureMulti(type, par);
}

export function rowsOf(multi: MultiParams, key: string): MultiRow[] {
  const rows = multi[key];
  return Array.isArray(rows) ? rows : [];
}

/** Una riga nuova di una lista, con i valori predefiniti del catalogo. */
export function blankRowOf(type: OperationType, list: MultiListDef): MultiRow {
  const base = defaultParams(type) as MultiParams;
  const row = rowsOf(base, list.key)[0];
  return row ? { ...row } : {};
}

/** Cambia un campo di una riga; gli altri campi (e i valori già scelti) restano. */
export function withRowField(
  type: OperationType,
  par: Params,
  listKey: string,
  index: number,
  fieldKey: string,
  value: MultiRow[string],
): Params {
  const multi = multiOf(type, par);
  const rows = rowsOf(multi, listKey).map((r, i) =>
    i === index ? { ...r, [fieldKey]: value } : r,
  );
  return { ...multi, [listKey]: rows };
}

export function withRowAdded(type: OperationType, par: Params, list: MultiListDef): Params {
  const multi = multiOf(type, par);
  return { ...multi, [list.key]: [...rowsOf(multi, list.key), blankRowOf(type, list)] };
}

export function withRowRemoved(
  type: OperationType,
  par: Params,
  listKey: string,
  index: number,
): Params {
  const multi = multiOf(type, par);
  return { ...multi, [listKey]: rowsOf(multi, listKey).filter((_, i) => i !== index) };
}

/** Campo globale di un'operazione a voci multiple (per esempio «tieni/escludi»). */
export function withGlobal(type: OperationType, par: Params, key: string, value: string): Params {
  return { ...multiOf(type, par), [key]: value };
}

/** Le colonne di una riga. */
export function rowColumns(row: MultiRow): string[] {
  return columnsOf(row);
}

/** Le operazioni a voci multiple: dal catalogo, non da un elenco scritto qui. */
export function isMulti(type: string): type is OperationType {
  return Object.prototype.hasOwnProperty.call(MULTI_DEFS, type);
}

/** Campi di testo che contengono numeri: tastiera numerica, senza frecce native. */
export const NUMERIC_KEYS: ReadonlySet<string> = new Set(["n", "pct", "decimals", "seed"]);
```

### `src/etl-canvas/inspector/useActiveSchema.ts`

14 righe

```ts
/** Lo schema dei dati in ingresso a un nodo (`schemaOf`): segue i collegamenti del grafo. */
import { useMemo } from "react";
import { schemaOf } from "../../etl-core";
import type { ColumnDef } from "../../etl-core";
import type { EtlStore } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";

const NONE: readonly ColumnDef[] = [];

export function useActiveSchema(store: EtlStore, nodeId: string | null): readonly ColumnDef[] {
  const graph = useEtlState((s) => s.graph, store);
  return useMemo(() => (nodeId ? (schemaOf(graph, nodeId) ?? NONE) : NONE), [graph, nodeId]);
}
```

### `src/etl-canvas/interaction.ts`

671 righe

```ts
/**
 * Il livello dei gesti del canvas: dal puntatore e dalla tastiera ai comandi
 * di etl-store. TypeScript puro, senza DOM né React: gli eventi arrivano già
 * tradotti (coordinate relative all'area, bersaglio classificato), così si
 * può provare con eventi simulati.
 *
 * Questo livello NON contiene logica di dominio. Ogni gesto chiama una
 * funzione che esiste già:
 *   trascinamento     → store.beginGesture / updateGesture / commitGesture / cancelGesture
 *   esito del rilascio → etl-core `relation`, `insertable`; etl-layout `nodeAt`, `linkAt`
 *   porte             → comando `connect`; geometria `nodePorts`
 *   selezione         → comandi `select` e `inspect`; geometria `nodeRect`
 *   tastiera          → comandi `deleteNodes`, `duplicate`, `moveNodes`, `select`; `undo`/`redo`
 *   anteprima eliminazione → etl-core `nodesRemovedBy`
 * Vedi la tabella nel README.
 */
import { boxCapacity, inputsOf, insertable, nodesRemovedBy, relation } from "../etl-core";
import type { Graph, Link } from "../etl-core";
import {
  DRAG_THRESHOLD_PX,
  GRID,
  linkAt,
  linkKey,
  nodeAt,
  nodePorts,
  nodeRect,
} from "../etl-layout";
import type { LinkRoutes, Point, Rect } from "../etl-layout";
import type { CommandResult, EtlStore } from "../etl-store";
import { handleCanvasDrop, previewCanvasDrop } from "./drop";
import type { CanvasDropPayload, DropPreview } from "./drop";
import { toWorld } from "./view";

/** Soglia del riquadro di selezione (prototipo, riga 4064). */
export const MARQUEE_THRESHOLD = 4;
/** Passo singolo delle frecce (prototipo, riga 4638: `GRID` con Maiusc, altrimenti 2). */
export const NUDGE_STEP = 2;

/** Esito mostrato su un nodo durante un trascinamento. */
export type DragOutcome = "merge" | "link" | "link-reverse" | "displace" | "reject";

/** Dove è iniziato il gesto, classificato dal livello che legge il DOM. */
export type DownTarget =
  | { readonly kind: "node"; readonly id: string }
  | { readonly kind: "port"; readonly id: string; readonly side: PortSide }
  | { readonly kind: "background" }
  | { readonly kind: "ignore" };

export type PortSide = "r" | "b" | "l" | "t";
/** Ordine di `PORTS` (etl-layout): destra, sotto, sinistra, sopra. */
const PORT_INDEX: Readonly<Record<PortSide, number>> = { r: 0, b: 1, l: 2, t: 3 };

/** Puntatore in coordinate dell'area (pixel dello schermo relativi in alto a sinistra dell'area). */
export interface PointerInput {
  readonly x: number;
  readonly y: number;
  readonly shiftKey?: boolean;
  readonly button?: number;
}

export interface KeyInput {
  readonly key: string;
  readonly metaKey?: boolean;
  readonly ctrlKey?: boolean;
  readonly shiftKey?: boolean;
  /** Il fuoco è in un campo di testo: le scorciatoie non si applicano. */
  readonly typing?: boolean;
}

export interface ConfirmState {
  /** «delete» (Canc, Fase 5) o «clear» (Svuota, Fase 6a.2: `ids` è vuoto, tutto il canvas sparisce). */
  readonly kind?: "delete" | "clear";
  readonly ids: readonly string[];
  /** Tutto ciò che sparirebbe: i nodi scelti e gli output a valle (etl-core `nodesRemovedBy`). */
  readonly removed: readonly string[];
  readonly title: string;
  readonly text: string;
}

export interface InteractionUi {
  /** Nodi in movimento. */
  readonly dragging: readonly string[];
  /** Nodo sotto il puntatore e cosa succederebbe al rilascio. */
  readonly drop: { readonly id: string; readonly outcome: DragOutcome } | null;
  /** Chiave (`da|a`) del cavo in cui si inserirebbe la lavorazione. */
  readonly insertLink: string | null;
  /** Cavo provvisorio tirato da una porta (coordinate del mondo). */
  readonly tempLink: { readonly from: Point; readonly to: Point; readonly valid: boolean } | null;
  /** Riquadro di selezione (coordinate dell'area) e nodi che comprende. */
  readonly marquee: { readonly rect: Rect; readonly ids: readonly string[] } | null;
  readonly confirm: ConfirmState | null;
  readonly hint: string | null;
}

export const IDLE_UI: InteractionUi = {
  dragging: [],
  drop: null,
  insertLink: null,
  tempLink: null,
  marquee: null,
  confirm: null,
  hint: null,
};

type Active =
  | {
      readonly kind: "node";
      readonly id: string;
      readonly start: Point;
      readonly shift: boolean;
      readonly group: readonly string[] | null;
      readonly canInsert: boolean;
      moved: boolean;
      /** Esito mostrato nell'ultimo aggiornamento: è quello applicato al rilascio. */
      drop: { id: string; outcome: DragOutcome } | null;
      insertLink: Link | null;
      /** Percorsi dei cavi prima del gesto (vedi `moveNode`). */
      startRoutes: LinkRoutes | null;
    }
  | {
      readonly kind: "port";
      readonly id: string;
      readonly from: Point;
      rel: "link" | "link-reverse" | null;
      target: string | null;
    }
  | {
      /** Pressione su un cavo: un click lo elimina (prototipo, righe 4575-4580). */
      readonly kind: "link";
      readonly link: Link;
      readonly start: Point;
      moved: boolean;
    }
  | {
      readonly kind: "marquee";
      readonly start: Point;
      readonly base: readonly string[];
      moved: boolean;
      ids: readonly string[];
    };

export interface InteractionController {
  getUi(): InteractionUi;
  subscribe(listener: () => void): () => void;
  /** Un gesto è in corso (il livello DOM ascolta i movimenti finché è vero). */
  isActive(): boolean;
  /** Barra spaziatrice premuta: navigazione, nessun gesto di selezione. */
  setSpace(down: boolean): void;
  down(target: DownTarget, input: PointerInput): boolean;
  move(input: PointerInput): void;
  up(input: PointerInput): void;
  /** Interrompe il gesto (pointercancel, Esc). */
  cancel(): void;
  key(input: KeyInput): boolean;
  confirmDelete(): CommandResult | null;
  cancelConfirm(): void;
  /** Elimina i nodi indicati (pulsante × sul nodo): subito se isolati, altrimenti con la conferma di `nodesRemovedBy`. */
  requestDeleteNodes(ids: readonly string[]): void;
  /** Chiede conferma per svuotare il canvas (comando `clearAll`); non fa nulla se è già vuoto. */
  requestClearAll(): void;
  /**
   * Un clic su un nodo (rilascio senza trascinamento) che lascia selezionato
   * solo quel nodo: è l'unico evento che apre l'Inspector. `id` è il nodo.
   */
  subscribeClick(listener: (id: string) => void): () => void;
  /** Punto dell'area → coordinate del mondo (per chi rilascia dalla cassetta). */
  toWorld(x: number, y: number): Point;
  /** Rilascio di un nuovo elemento: vedi drop.ts. */
  handleCanvasDrop(payload: CanvasDropPayload, point: Point): CommandResult;
  previewCanvasDrop(payload: CanvasDropPayload, point: Point): DropPreview;
  /**
   * Un elemento trascinato da fuori (la cassetta) passa sopra il canvas:
   * mostra gli stessi contorni del trascinamento tra nodi. `point` è in
   * coordinate dell'area, `null` se il puntatore è fuori dall'area.
   */
  hoverExternal(payload: CanvasDropPayload, point: Point | null): void;
  /** Rilascio dell'elemento trascinato da fuori; `null` se il puntatore è fuori dall'area. */
  dropExternal(payload: CanvasDropPayload, point: Point | null): CommandResult | null;
}

function sameIds(a: readonly string[], b: readonly string[]): boolean {
  return a.length === b.length && a.every((id, i) => id === b[i]);
}

/** Testi della conferma (prototipo, righe 4509-4552). */
function confirmCopy(
  ids: readonly string[],
  removedCount: number,
): { title: string; text: string } {
  const downstream = removedCount - ids.length;
  if (ids.length === 1) {
    const n = downstream;
    return {
      title: "Eliminare il nodo?",
      text:
        n > 0
          ? `Fa parte del flusso. Verranno rimossi i suoi collegamenti e ${n}${n > 1 ? " risultati a valle." : " risultato a valle."}`
          : "Fa parte del flusso: i suoi collegamenti verranno rimossi.",
    };
  }
  return {
    title: `Eliminare ${ids.length} nodi?`,
    text:
      "Alcuni fanno parte del flusso: verranno rimossi i loro collegamenti" +
      (downstream > 0
        ? ` e ${downstream} ${downstream > 1 ? "risultati" : "risultato"} a valle.`
        : "."),
  };
}

export function createInteractionController(store: EtlStore): InteractionController {
  let ui: InteractionUi = IDLE_UI;
  let active: Active | null = null;
  let space = false;
  const listeners = new Set<() => void>();
  const clickListeners = new Set<(id: string) => void>();

  const setUi = (patch: Partial<InteractionUi>): void => {
    const next = { ...ui, ...patch };
    const keys = Object.keys(next) as (keyof InteractionUi)[];
    if (keys.every((k) => next[k] === ui[k])) return;
    ui = next;
    for (const l of [...listeners]) l();
  };
  const graph = (): Graph => store.getState().graph;
  const worldOf = (p: Point): Point => toWorld(store.getState().view, p.x, p.y);
  const linkOfKey = (key: string): Link | null =>
    graph().links.find((l) => linkKey(l) === key) ?? null;

  /** `select` + `inspect`: l'Inspector segue la selezione (nessun pannello, solo stato). */
  const applySelection = (ids: readonly string[]): void => {
    const s = store.getState();
    const unique = Array.from(new Set(ids));
    if (!sameIds(s.selection, unique)) store.dispatch({ type: "select", payload: { ids: unique } });
    const keep = s.inspector.nodeId && unique.includes(s.inspector.nodeId);
    const node = unique.length ? (keep ? s.inspector.nodeId : (unique[0] as string)) : null;
    if (node !== s.inspector.nodeId || (node !== null && s.inspector.step !== 0 && !keep)) {
      store.dispatch({ type: "inspect", payload: { node, step: 0 } });
    }
  };

  const hintForLink = (g: Graph, dragged: string, over: string): string => {
    const boxId = relation(g, dragged, over).relation === "link" ? over : dragged;
    const box = g.cards[boxId];
    const need = box ? boxCapacity(box) - inputsOf(g, boxId).length : 1;
    return need > 1
      ? "Rilascia: sarà la tabella di sinistra, poi servirà la seconda"
      : "Rilascia per collegare e generare l’output";
  };

  // --- trascinamento di un nodo ---------------------------------------------------

  const moveNode = (a: Extract<Active, { kind: "node" }>, input: PointerInput): void => {
    const view = store.getState().view;
    const sx = input.x - a.start.x;
    const sy = input.y - a.start.y;
    if (!a.moved) {
      if (Math.hypot(sx, sy) < DRAG_THRESHOLD_PX) return;
      const ids = a.group ?? [a.id];
      if (a.canInsert) a.startRoutes = store.getRoutes();
      const r = store.beginGesture({ ids });
      if (!r.ok) {
        active = null;
        return;
      }
      a.moved = true;
      setUi({ dragging: ids });
    }
    const dx = sx / view.zoom;
    const dy = sy / view.zoom;
    if (a.group) {
      store.updateGesture({ dx, dy });
      return;
    }
    const p = worldOf(input);
    const over = nodeAt(graph(), p, a.id);
    // l'esito si calcola sul grafo PRIMA dell'aggiornamento: l'aggiornamento può spingere via l'altro nodo
    const rel = over ? relation(graph(), a.id, over) : null;
    store.updateGesture({ dx, dy, over });

    if (over && rel) {
      const outcome: DragOutcome = rel.relation ?? "reject";
      a.drop = { id: over, outcome };
      a.insertLink = null;
      setUi({
        drop: a.drop,
        insertLink: null,
        hint:
          outcome === "merge"
            ? "Rilascia per fondere le lavorazioni"
            : outcome === "link" || outcome === "link-reverse"
              ? hintForLink(graph(), a.id, over)
              : (rel.displaceReason ?? null),
      });
      return;
    }
    a.drop = null;
    // sopra un cavo, senza un nodo sotto: una lavorazione slegata si inserisce nel collegamento
    let insert: Link | null = null;
    if (a.canInsert) {
      // il nodo in mano è un ostacolo e fa scansare i cavi: si prova anche sui percorsi di prima del gesto
      const key = linkAt(store.getRoutes(), p) ?? (a.startRoutes ? linkAt(a.startRoutes, p) : null);
      const link = key ? linkOfKey(key) : null;
      if (link && insertable(graph(), link, a.id)) insert = link;
    }
    a.insertLink = insert;
    setUi({
      drop: null,
      insertLink: insert ? linkKey(insert) : null,
      hint: insert ? "Rilascia per inserire la lavorazione nel collegamento" : null,
    });
  };

  const releaseNode = (a: Extract<Active, { kind: "node" }>, input: PointerInput): void => {
    if (!a.moved) {
      // un click: Maiusc aggiunge o toglie; altrimenti seleziona solo questo (riga 2069)
      if (a.shift || input.shiftKey) {
        const sel = store.getState().selection;
        applySelection(sel.includes(a.id) ? sel.filter((x) => x !== a.id) : [...sel, a.id]);
      } else applySelection([a.id]);
      const sel = store.getState().selection;
      if (!a.shift && !input.shiftKey && sel.length === 1 && sel[0] === a.id)
        for (const l of [...clickListeners]) l(a.id);
      return;
    }
    const outcome = a.drop?.outcome;
    const target =
      a.drop && (outcome === "merge" || outcome === "link" || outcome === "link-reverse")
        ? { node: a.drop.id }
        : a.insertLink
          ? { link: a.insertLink }
          : undefined;
    store.commitGesture(target ? { target } : {});
  };

  // --- porte ----------------------------------------------------------------------

  const movePort = (a: Extract<Active, { kind: "port" }>, input: PointerInput): void => {
    const p = worldOf(input);
    const g = graph();
    const over = nodeAt(g, p, a.id);
    const r = over ? relation(g, a.id, over) : null;
    // dalle porte si collega soltanto: fusione e spostamento diventano rifiuto (riga 3990)
    const rel = r && (r.relation === "link" || r.relation === "link-reverse") ? r.relation : null;
    a.rel = rel;
    a.target = rel ? over : null;
    setUi({
      tempLink: { from: a.from, to: p, valid: !!rel },
      drop: over ? { id: over, outcome: rel ?? "reject" } : null,
      hint: rel
        ? "Rilascia per collegare"
        : over
          ? (r?.displaceReason ?? "Questi due nodi non si possono collegare")
          : "Trascina fino al nodo da collegare",
    });
  };

  const releasePort = (a: Extract<Active, { kind: "port" }>): void => {
    if (a.rel && a.target) {
      if (a.rel === "link")
        store.dispatch({ type: "connect", payload: { from: a.id, to: a.target } });
      else store.dispatch({ type: "connect", payload: { from: a.target, to: a.id } });
    }
  };

  // --- riquadro di selezione ------------------------------------------------------

  const marqueeRect = (a: Extract<Active, { kind: "marquee" }>, input: PointerInput): Rect => ({
    x: Math.min(a.start.x, input.x),
    y: Math.min(a.start.y, input.y),
    w: Math.abs(input.x - a.start.x),
    h: Math.abs(input.y - a.start.y),
  });

  /** Nodi il cui quadrato tocca il riquadro (prototipo, righe 4071-4076: il riquadro è in coordinate dello schermo). */
  const idsInRect = (rect: Rect): string[] => {
    const a = worldOf({ x: rect.x, y: rect.y });
    const b = worldOf({ x: rect.x + rect.w, y: rect.y + rect.h });
    const out: string[] = [];
    for (const c of Object.values(graph().cards)) {
      const r = nodeRect(c);
      if (r.x + r.w > a.x && r.x < b.x && r.y + r.h > a.y && r.y < b.y) out.push(c.id);
    }
    return out;
  };

  // --- eliminazione ---------------------------------------------------------------

  /** `only`: elimina solo questi nodi (il pulsante × di un nodo); altrimenti la selezione. */
  const requestDelete = (only?: readonly string[]): void => {
    const ids = (only ?? store.getState().selection).filter((id) => !!graph().cards[id]);
    if (!ids.length) return;
    const attached = graph().links.some((l) => ids.includes(l.from) || ids.includes(l.to));
    if (!attached) {
      store.dispatch({ type: "deleteNodes", payload: { ids } });
      return;
    }
    const removed = [...nodesRemovedBy(graph(), ids)];
    setUi({ confirm: { ids, removed, ...confirmCopy(ids, removed.length) } });
  };

  const requestClearAll = (): void => {
    const removed = Object.keys(graph().cards);
    if (!removed.length) return;
    setUi({
      confirm: {
        kind: "clear",
        ids: [],
        removed,
        title: "Svuotare il canvas?",
        text: "Eliminare tutti i nodi e i collegamenti? Puoi annullare con Cmd/Ctrl+Z.",
      },
    });
  };

  const nudge = (dx: number, dy: number): void => {
    const ids = store.getState().selection;
    if (ids.length) store.dispatch({ type: "moveNodes", payload: { ids, dx, dy } });
  };

  const finish = (): void => {
    active = null;
    setUi({
      dragging: [],
      drop: null,
      insertLink: null,
      tempLink: null,
      marquee: null,
      hint: null,
    });
  };

  const controller: InteractionController = {
    getUi: () => ui,
    subscribe(l) {
      listeners.add(l);
      return () => listeners.delete(l);
    },
    isActive: () => active !== null,
    setSpace(down) {
      space = down;
    },

    down(target, input) {
      if (active) return false;
      if (input.button !== undefined && input.button !== 0) return false;
      if (target.kind === "ignore") return false;
      if (ui.confirm) setUi({ confirm: null });
      if (space) return false;
      const start = { x: input.x, y: input.y };

      if (target.kind === "port") {
        const card = graph().cards[target.id];
        if (!card) return false;
        const port = nodePorts(card)[PORT_INDEX[target.side]];
        if (!port) return false;
        const from = { x: port.anchor.x, y: port.anchor.y };
        active = { kind: "port", id: target.id, from, rel: null, target: null };
        setUi({
          dragging: [target.id],
          tempLink: { from, to: worldOf(start), valid: false },
          hint: "Trascina fino al nodo da collegare",
        });
        return true;
      }

      if (target.kind === "node") {
        const card = graph().cards[target.id];
        if (!card) return false;
        const sel = store.getState().selection;
        const group =
          sel.includes(target.id) && sel.length > 1
            ? sel.filter((id) => !!graph().cards[id])
            : null;
        const canInsert =
          !group &&
          card.kind === "op" &&
          !graph().links.some((l) => l.from === target.id || l.to === target.id);
        active = {
          kind: "node",
          id: target.id,
          start,
          shift: !!input.shiftKey,
          group,
          canInsert,
          moved: false,
          drop: null,
          insertLink: null,
          startRoutes: null,
        };
        return true;
      }

      // su un cavo: niente riquadro, il click lo elimina (nel prototipo il cavo non è "sfondo", riga 4033)
      const hit = linkAt(store.getRoutes(), worldOf(start));
      const link = hit ? linkOfKey(hit) : null;
      if (link) {
        active = { kind: "link", link, start, moved: false };
        return true;
      }

      // sfondo: riquadro di selezione (Maiusc = si aggiunge alla selezione)
      const base = input.shiftKey ? [...store.getState().selection] : [];
      active = { kind: "marquee", start, base, moved: false, ids: base };
      return true;
    },

    move(input) {
      const a = active;
      if (!a) return;
      if (a.kind === "node") moveNode(a, input);
      else if (a.kind === "link") {
        if (Math.hypot(input.x - a.start.x, input.y - a.start.y) >= DRAG_THRESHOLD_PX)
          a.moved = true;
      } else if (a.kind === "port") movePort(a, input);
      else {
        if (!a.moved && Math.hypot(input.x - a.start.x, input.y - a.start.y) < MARQUEE_THRESHOLD)
          return;
        a.moved = true;
        const rect = marqueeRect(a, input);
        const ids = Array.from(new Set([...a.base, ...idsInRect(rect)]));
        a.ids = ids;
        setUi({ marquee: { rect, ids } });
      }
    },

    up(input) {
      const a = active;
      if (!a) return;
      if (a.kind === "node") {
        releaseNode(a, input);
      } else if (a.kind === "port") {
        releasePort(a);
      } else if (a.kind === "link") {
        if (!a.moved) store.dispatch({ type: "deleteLink", payload: { link: a.link } });
      } else if (a.moved) {
        applySelection(a.ids);
      } else if (store.getState().selection.length || store.getState().inspector.nodeId) {
        // un click sul vuoto deseleziona (riga 4584)
        applySelection([]);
      }
      finish();
    },

    cancel() {
      const a = active;
      if (!a) return;
      if (a.kind === "node" && a.moved) store.cancelGesture();
      finish();
    },

    key(e) {
      if (e.typing) return false;
      const mod = !!(e.metaKey || e.ctrlKey);
      const k = e.key.length === 1 ? e.key.toLowerCase() : e.key;
      // con una conferma aperta restano attivi solo Esc (annulla) e il normale uso da tastiera della finestra
      if (ui.confirm && k !== "Escape") return false;
      if (!mod) {
        if (k === "Delete" || k === "Backspace") {
          if (!store.getState().selection.length) return false;
          requestDelete();
          return true;
        }
        if (k === "Escape") {
          if (active) {
            controller.cancel();
            return true;
          }
          if (ui.confirm) {
            setUi({ confirm: null });
            return true;
          }
          applySelection([]);
          return true;
        }
        const step = e.shiftKey ? GRID : NUDGE_STEP;
        const arrows: Record<string, [number, number]> = {
          ArrowLeft: [-step, 0],
          ArrowRight: [step, 0],
          ArrowUp: [0, -step],
          ArrowDown: [0, step],
        };
        const d = arrows[k];
        if (d && store.getState().selection.length) {
          nudge(d[0], d[1]);
          return true;
        }
        return false;
      }
      if (k === "d") {
        const r = store.dispatch({
          type: "duplicate",
          payload: { ids: store.getState().selection },
        });
        if (r.ok) applySelection(store.getState().selection);
        return true;
      }
      if (k === "a") {
        applySelection(Object.keys(graph().cards));
        return true;
      }
      if (k === "z" && e.shiftKey) {
        store.redo();
        return true;
      }
      if (k === "z") {
        store.undo();
        return true;
      }
      if (k === "y") {
        store.redo();
        return true;
      }
      return false;
    },

    confirmDelete() {
      const c = ui.confirm;
      if (!c) return null;
      setUi({ confirm: null });
      if (c.kind === "clear") return store.dispatch({ type: "clearAll", payload: {} });
      return store.dispatch({ type: "deleteNodes", payload: { ids: c.ids } });
    },
    cancelConfirm() {
      setUi({ confirm: null });
    },
    requestClearAll,
    requestDeleteNodes: (ids) => requestDelete(ids),

    hoverExternal(payload, point) {
      if (active) return;
      if (!point) {
        setUi({ drop: null, insertLink: null, hint: null });
        return;
      }
      const preview = previewCanvasDrop(store, payload, worldOf(point));
      const outcome = preview.outcome;
      setUi({
        drop:
          preview.nodeId &&
          (outcome === "merge" || outcome === "link" || outcome === "link-reverse")
            ? { id: preview.nodeId, outcome }
            : null,
        insertLink: preview.linkKey ?? null,
        hint:
          outcome === "merge"
            ? "Rilascia per fondere direttamente nel box"
            : outcome === "link" || outcome === "link-reverse"
              ? "Rilascia per collegare"
              : outcome === "insert"
                ? "Rilascia per inserire la lavorazione nel collegamento"
                : null,
      });
    },
    dropExternal(payload, point) {
      setUi({ drop: null, insertLink: null, hint: null });
      return point ? handleCanvasDrop(store, payload, worldOf(point)) : null;
    },

    subscribeClick(l) {
      clickListeners.add(l);
      return () => clickListeners.delete(l);
    },

    toWorld: (x, y) => toWorld(store.getState().view, x, y),
    handleCanvasDrop: (payload, point) => handleCanvasDrop(store, payload, point),
    previewCanvasDrop: (payload, point) => previewCanvasDrop(store, payload, point),
  };
  return controller;
}
```

### `src/etl-canvas/loop.ts`

118 righe

```ts
/**
 * Un solo ciclo requestAnimationFrame per tutto il canvas. Si ferma
 * quando la scheda è nascosta e quando nessun compito ha nulla da
 * animare; con `prefers-reduced-motion` non parte mai (i compiti
 * mostrano lo stato finale con `settle`). Tutto ciò che tocca il browser
 * passa da `LoopEnv`, quindi nei test si sostituisce.
 */

export interface LoopEnv {
  raf(cb: () => void): number;
  caf(id: number): void;
  /** Orologio in millisecondi (nel browser `performance.now`). */
  now(): number;
  hidden(): boolean;
  reducedMotion(): boolean;
  onVisibilityChange(cb: () => void): () => void;
  onReducedMotionChange(cb: () => void): () => void;
}

export interface Task {
  /** Disegna il frame a `now`; `true` se ha ancora qualcosa da animare. */
  frame(now: number): boolean;
  /** Mostra lo stato finale, senza movimento (movimento ridotto). */
  settle(): void;
}

export interface Loop {
  add(task: Task): () => void;
  /** C'è (forse) lavoro nuovo: se serve, il ciclo riparte. */
  wake(): void;
  running(): boolean;
  dispose(): void;
}

export function createLoop(env: LoopEnv): Loop {
  const tasks = new Set<Task>();
  let id: number | null = null;

  const cancel = (): void => {
    if (id !== null) {
      env.caf(id);
      id = null;
    }
  };

  const settleAll = (): void => {
    for (const t of [...tasks]) t.settle();
  };

  const tick = (): void => {
    id = null;
    const now = env.now();
    let busy = false;
    for (const t of [...tasks]) if (t.frame(now)) busy = true;
    if (busy && !env.hidden() && !env.reducedMotion()) id = env.raf(tick);
  };

  const schedule = (): void => {
    if (id !== null || tasks.size === 0) return;
    if (env.hidden()) return;
    if (env.reducedMotion()) {
      settleAll();
      return;
    }
    id = env.raf(tick);
  };

  const offVisibility = env.onVisibilityChange(() => {
    if (env.hidden()) cancel();
    else schedule();
  });
  const offMotion = env.onReducedMotionChange(() => {
    if (env.reducedMotion()) {
      cancel();
      settleAll();
    } else schedule();
  });

  return {
    add(task) {
      tasks.add(task);
      schedule();
      return () => {
        tasks.delete(task);
        if (tasks.size === 0) cancel();
      };
    },
    wake: schedule,
    running: () => id !== null,
    dispose() {
      cancel();
      tasks.clear();
      offVisibility();
      offMotion();
    },
  };
}

/** Ambiente reale. Va creato solo nel browser (in un effetto), mai durante il rendering. */
export function browserEnv(): LoopEnv {
  const mq = window.matchMedia("(prefers-reduced-motion: reduce)");
  return {
    raf: (cb) => requestAnimationFrame(() => cb()),
    caf: (id) => cancelAnimationFrame(id),
    now: () => performance.now(),
    hidden: () => document.hidden,
    reducedMotion: () => mq.matches,
    onVisibilityChange(cb) {
      document.addEventListener("visibilitychange", cb);
      return () => document.removeEventListener("visibilitychange", cb);
    },
    onReducedMotionChange(cb) {
      mq.addEventListener("change", cb);
      return () => mq.removeEventListener("change", cb);
    },
  };
}
```

