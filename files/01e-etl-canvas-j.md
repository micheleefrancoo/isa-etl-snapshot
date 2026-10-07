# 01e-etl-canvas-j.md

File in questo blocco:

- `src/etl-canvas/inspector/copy.ts`
- `src/etl-canvas/inspector/family.ts`
- `src/etl-canvas/inspector/icons.tsx`
- `src/etl-canvas/inspector/inspector.css`
- `src/etl-canvas/inspector/joinKeys.ts`
- `src/etl-canvas/inspector/joinSides.ts`
- `src/etl-canvas/inspector/logic.ts`
- `src/etl-canvas/inspector/masterDetail.ts`
- `src/etl-canvas/inspector/menu.ts`
- `src/etl-canvas/inspector/params.ts`
- `src/etl-canvas/inspector/useActiveSchema.ts`

---

### `src/etl-canvas/inspector/copy.ts`

168 righe

```ts
/**
 * Tutti i testi dell'Inspector, in italiano e con la sola maiuscola iniziale.
 * Nessuna stringa italiana sparsa nei componenti: un test lo verifica. Le
 * etichette dei campi e delle operazioni vengono dal catalogo di etl-core.
 */
export const copy = {
  // intestazione
  kindSource: "Sorgente",
  kindResult: "Risultato",
  kindBox: "Box combinato",
  kindOperation: "Lavorazione",
  nameLabel: "Nome del nodo",

  // stati
  emptyInspector: "Nessun nodo selezionato",
  lockedText:
    "Collega una tabella a questo nodo per configurarne i parametri: colonne, chiavi e valori dipendono dai dati in ingresso.",
  lockedJoin: (n: number) => `Serve un Join completo: ${n} tabelle in ingresso.`,
  resultNote: (producer: string) =>
    `Risultato generato da ${producer}. I suoi parametri si configurano nei passaggi che lo producono.`,
  producerFallback: "una lavorazione",
  resultIncomplete: "Incompleto: al join manca una tabella.",
  inputsCount: (got: number, cap: number) => `Tabelle in ingresso: ${got} su ${cap}.`,
  columnsTitle: "Colonne",
  columnsNone: "Nessuna colonna nota: carica un dataset con colonne.",
  closeInspector: "Nascondi l’inspector",

  // box combinato
  sequenceTitle: "Sequenza di esecuzione",
  sequenceCount: (n: number) => `${n} passaggi`,
  stepLabel: (n: number, name: string) => `Passaggio ${n}: ${name}`,
  stepReorderHelp: "Trascina il passaggio o usa Alt con le frecce su e giù per cambiarne l’ordine.",
  stepDetach: "Sgancia sul canvas",
  stepDelete: "Elimina passaggio",
  stepConfigure: "Configura parametri",
  stepMenu: "Impostazioni del passaggio",
  stepsMenuLabel: "Azioni del passaggio",
  tableReference: "Tabella di riferimento",
  tableLeft: "Tabella sinistra",
  tableRight: "Tabella destra",
  tableLinkFirst: "Collega le tabelle per scegliere su quale agisce questo passaggio.",
  tableLeftResult: (j: number) =>
    `Tabella sinistra: risultato del join ${j}, già unito nei passaggi precedenti.`,
  tableSingle: (j: number) => `Opera sul risultato del join ${j}: da qui la tabella è una sola.`,
  tableFallback: "tabella",

  // campi e menu
  pickOrType: "Scegli o scrivi",
  typeOr: "Oppure scrivi",
  typePlaceholder: "Scrivi un valore",
  searchPlaceholder: "Cerca",
  noResults: "Nessun risultato",
  useTyped: (text: string) => `Usa “${text}”`,
  menuLabel: "Scelte disponibili",

  // colonne
  columnsPlaceholder: "Nessuna colonna scelta",
  columnsAdd: "Aggiungi colonne",
  columnsSearch: "Cerca una colonna",
  columnsAll: "Tutte",
  columnsNone2: "Nessuna",
  columnsCount: (n: number, m: number) => `${n} ${n === 1 ? "colonna" : "colonne"} su ${m}`,
  columnsRemove: (name: string) => `Rimuovi ${name}`,
  columnsMoveHelp: "Alt con le frecce sinistra e destra cambia l’ordine; si può anche trascinare.",
  columnsAddTyped: (name: string) => `+ Aggiungi “${name}”`,
  columnsFree: "Non presente nei dati",
  columnsMenu: "Colonne",
  columnsOutside: (n: number) =>
    `${n} ${n === 1 ? "colonna non presente" : "colonne non presenti"} nei dati in ingresso`,
  columnsOutsideRemove: "Rimuovi",
  columnsPosition: (name: string, i: number, n: number) => `${name}, posizione ${i} di ${n}`,

  // valori
  valuesPlaceholder: "Nessun valore scelto",
  valuesAdd: "Scegli i valori",
  valuesSearch: "Cerca tra i valori o scrivine di nuovi",
  valuesSearchFree: "Scrivi i valori, anche più insieme",
  valuesAll: "Tutti",
  valuesNone: "Nessuno",
  valuesCount: (n: number, m: number) =>
    m > 0
      ? `${n} ${n === 1 ? "selezionato" : "selezionati"} su ${m}`
      : `${n} ${n === 1 ? "selezionato" : "selezionati"}`,
  valuesAddTyped: (tokens: readonly string[]) =>
    `+ Aggiungi ${tokens.map((t) => `“${t}”`).join(", ")}`,
  valuesRemove: (v: string) => `Rimuovi ${v}`,
  valuesOutside: (n: number) =>
    `${n} ${n === 1 ? "valore non presente" : "valori non presenti"} nelle colonne scelte`,
  valuesOutsideRemove: "Rimuovi",
  valuesMenu: "Valori",
  valuesFree: "Non presente nei dati",

  // voci multiple
  rowToggle: "Mostra o nascondi i campi",
  rowTodo: "Da configurare",
  rowRemove: "Rimuovi voce",
  rowNote: "",
  resultNameAuto: (names: readonly string[]) => `Nome automatico: ${names.join(", ")}`,
  resultNameNone: "Il nome compare quando scegli le colonne",

  // condizioni di filtro
  filterColumn: "Colonna",
  filterColumnType: (type: string) => `tipo: ${type}`,
  filterOperator: "Operatore",
  filterValue: "Valore",
  filterValues: "Valori",
  filterSingleValue: "Valore singolo",

  // join: impostazioni e condizioni
  joinConditions: "Condizioni di unione",
  joinLeftSide: "Lato sinistro",
  joinRightSide: "Lato destro",
  joinCompare: "Confronto",
  joinModeColumn: "Colonna",
  joinModeValue: "Valore",
  joinModeList: "Lista",
  joinValuePlaceholder: "Valore",
  joinPerformance:
    "Nessuna condizione di uguaglianza: il Join confronterà ogni riga con tutte le altre. Su tabelle grandi l’esecuzione può essere molto lenta.",

  // layout a tre colonne
  colSettings: "Impostazioni",
  colConditions: "Condizioni e gruppi",
  colList: "Elenco",
  colDetail: "Dettaglio",

  // riordino delle voci
  rowGripKeys: "Alt+ArrowUp Alt+ArrowDown",
  rowGrip: (name: string) => `Sposta ${name}`,
  rowReorderHelp: "Trascina la maniglia o usa Alt con le frecce su e giù per cambiare l’ordine.",

  // condizioni, connettori e gruppi
  conditionNoun: "Condizione",
  conditionTodo: "da configurare",
  conditionAdd: "Aggiungi condizione",
  conditionRemove: "Rimuovi condizione",
  conditionToggle: "Mostra o nascondi la condizione",
  connectorLabel: (a: number, b: number) => `Connettore tra la condizione ${a} e la ${b}`,
  connectorOption: (op: string, help: string) => `${op} · ${help}`,
  groupTitle: "Gruppo",
  groupDo: "Raggruppa le due condizioni",
  groupSplit: "Dividi il gruppo in questo punto",
  groupUngroup: "Sciogli",
  groupUngroupLabel: "Sciogli il gruppo",
  groupAddIn: "Condizione nel gruppo",
  previewTitle: "Anteprima",
  previewHint:
    "Tra parentesi i gruppi; per il resto le condizioni si combinano nell’ordine in cui compaiono.",
  previewEmpty: "…",
  groupMarkDo: "( )",
  groupMarkSplit: ")(",

  // pulsanti sul nodo
  nodeDelete: "Elimina nodo",
  nodeExpand: "Espandi sequenza",

  // pannello espanso
  expandedSub: (n: number) => `${n} passaggi in sequenza`,
  expandedNote: "Trascina le righe per cambiare l’ordine di esecuzione.",
  expandedNoteOutside: "Rilascia qui fuori per sganciare il passaggio sul canvas",
  expandedClose: "Chiudi",
  expandedTitle: "Sequenza del box",

  // conferme e annunci
  announceMoved: (name: string, pos: number, n: number) =>
    `${name} spostato in posizione ${pos} di ${n}`,
} as const;
```

### `src/etl-canvas/inspector/family.ts`

13 righe

```ts
/** La famiglia di un nodo, per l'intestazione dell'Inspector. */
import { sectionOf } from "../../etl-core";
import type { Card } from "../../etl-core";
import { copy } from "./copy";

/** La famiglia da mostrare: il tipo di nodo, o la sezione della cassetta per una lavorazione semplice. */
export function familyLabel(card: Card): string {
  if (card.kind === "dataset") return card.isOutput ? copy.kindResult : copy.kindSource;
  if (card.components.length > 1) return copy.kindBox;
  const first = card.components[0];
  return (first ? sectionOf(first)?.name : undefined) ?? copy.kindOperation;
}
```

### `src/etl-canvas/inspector/icons.tsx`

87 righe

```tsx
/** Icone dell'Inspector: tracciati statici, ereditano il colore del testo. */
import type { ReactNode } from "react";

function Svg(props: { children: ReactNode; strokeWidth?: number }) {
  return (
    <svg
      className="ei-icon"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth={props.strokeWidth ?? 2.4}
      strokeLinecap="round"
      strokeLinejoin="round"
      aria-hidden="true"
      focusable="false"
    >
      {props.children}
    </svg>
  );
}

export const ChevronDown = () => (
  <Svg strokeWidth={2.6}>
    <polyline points="6 9 12 15 18 9" />
  </Svg>
);
export const ChevronRight = () => (
  <Svg strokeWidth={2.6}>
    <polyline points="9 18 15 12 9 6" />
  </Svg>
);
export const XIcon = () => (
  <Svg>
    <line x1="17" y1="7" x2="7" y2="17" />
    <line x1="7" y1="7" x2="17" y2="17" />
  </Svg>
);
export const LockIcon = () => (
  <Svg strokeWidth={2}>
    <rect x="4" y="11" width="16" height="9" rx="2" />
    <path d="M8 11V7a4 4 0 0 1 8 0v4" />
  </Svg>
);
export const GripIcon = () => (
  <Svg strokeWidth={2.6}>
    <circle cx="9" cy="6" r="0.6" />
    <circle cx="15" cy="6" r="0.6" />
    <circle cx="9" cy="12" r="0.6" />
    <circle cx="15" cy="12" r="0.6" />
    <circle cx="9" cy="18" r="0.6" />
    <circle cx="15" cy="18" r="0.6" />
  </Svg>
);
export const PlusIcon = () => (
  <Svg>
    <line x1="12" y1="5" x2="12" y2="19" />
    <line x1="5" y1="12" x2="19" y2="12" />
  </Svg>
);
export const CheckIcon = () => (
  <Svg strokeWidth={3}>
    <polyline points="5 12.5 10 17.5 19 7" />
  </Svg>
);
export const DetachIcon = () => (
  <Svg strokeWidth={2.2}>
    <path d="M14 3h7v7" />
    <path d="M21 3l-9 9" />
    <path d="M19 14v5a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V7a2 2 0 0 1 2-2h5" />
  </Svg>
);
export const MoreIcon = () => (
  <Svg strokeWidth={2.6}>
    <circle cx="5" cy="12" r="1" />
    <circle cx="12" cy="12" r="1" />
    <circle cx="19" cy="12" r="1" />
  </Svg>
);
export const ExpandIcon = () => (
  <Svg strokeWidth={2.2}>
    <path d="M8 3H5a2 2 0 0 0-2 2v3" />
    <path d="M21 8V5a2 2 0 0 0-2-2h-3" />
    <path d="M3 16v3a2 2 0 0 0 2 2h3" />
    <path d="M16 21h3a2 2 0 0 0 2-2v-3" />
  </Svg>
);
```

### `src/etl-canvas/inspector/inspector.css`

984 righe

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
  flex: 1 1 0;
  min-width: 0;
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

/* scelta tra poche voci (Colonna | Valore | Lista) */
.ei-seg {
  display: inline-flex;
  align-self: flex-start;
  max-width: 100%;
  padding: var(--isa-space-1);
  border-radius: var(--isa-radius-pill);
  background: var(--isa-option-hover);
}
.ei-seg-btn {
  all: unset;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: var(--isa-icon-btn);
  padding: 0 var(--isa-space-3);
  border-radius: var(--isa-radius-pill);
  color: var(--isa-text-secondary);
  font-size: var(--isa-fs-label);
  font-weight: 700;
  cursor: pointer;
  white-space: nowrap;
}
.ei-seg-btn.ei-on {
  background: var(--isa-surface-raised);
  color: var(--isa-accent);
  box-shadow: var(--isa-shadow-raised);
}
.ei-seg-btn:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 1px;
}

/* pastiglia del connettore (variante di StyledSelect) */
.ei-pill {
  all: unset;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--isa-space-1);
  min-height: var(--isa-icon-btn);
  padding: 0 var(--isa-space-3);
  border-radius: var(--isa-radius-pill);
  background: var(--isa-accent-soft);
  color: var(--isa-accent);
  font-size: var(--isa-fs-label);
  font-weight: 800;
  letter-spacing: 0.04em;
  cursor: pointer;
  white-space: nowrap;
}
.ei-pill:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 1px;
}
.ei-pill .ei-select-value {
  flex: none;
  overflow: visible;
}

/* condizioni, connettori e gruppi (Fase 6b.2) */
.ei-conditions {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-3);
  min-width: 0;
}
.ei-conn {
  display: flex;
  align-items: center;
  gap: var(--isa-space-2);
}
.ei-conn-line {
  flex: 1;
  height: 1px;
  background: var(--isa-border-strong);
}
.ei-conn-btn {
  all: unset;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex: none;
  min-width: var(--isa-icon-btn);
  min-height: var(--isa-icon-btn);
  padding: 0 var(--isa-space-2);
  border-radius: var(--isa-radius-pill);
  background: var(--isa-option-hover);
  color: var(--isa-text-secondary);
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: var(--isa-fs-label);
  font-weight: 800;
  cursor: pointer;
}
.ei-conn-btn:hover {
  background: var(--isa-accent-soft);
  color: var(--isa-accent);
}
.ei-conn-btn:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 1px;
}
.ei-group {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-3);
  padding: var(--isa-space-3);
  border: 1px dashed var(--isa-accent);
  border-radius: var(--isa-radius-panel);
  background: var(--isa-accent-soft);
}
.ei-group-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--isa-space-2);
}
.ei-group-title {
  font-size: var(--isa-fs-overline);
  font-weight: 800;
  letter-spacing: var(--isa-ls-overline);
  text-transform: uppercase;
  color: var(--isa-accent);
}
.ei-addin {
  background: var(--isa-field-bg);
}
.ei-preview {
  display: flex;
  flex-direction: column;
  gap: var(--isa-space-1);
  padding: var(--isa-space-3);
  border-radius: var(--isa-radius-control);
  background: var(--isa-option-hover);
}
.ei-preview-text {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
  font-size: var(--isa-fs-help);
  line-height: 1.55;
  color: var(--isa-text);
  overflow-wrap: anywhere;
}

/* tre colonne dei bordi alto e basso: Impostazioni | Condizioni o Elenco | Dettaglio */
.ec-panel.ec-horiz .ei-root:has(> .ei-cols3) {
  display: flex;
  column-width: auto;
  overflow: hidden;
}
.ec-panel.ec-horiz .ec-tb-inner.ec-insp:has(.ei-cols3) {
  overflow: hidden;
}
.ei-cols3 {
  display: flex;
  flex: 1 1 auto;
  min-width: 0;
  height: 100%;
}
.ei-col {
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
}
.ei-col-general {
  flex: 0 0 var(--isa-md-general-w);
  padding-right: var(--isa-space-4);
}
.ei-col-master {
  flex: 0 0 var(--isa-md-master-w);
  padding: 0 var(--isa-space-4);
  border-left: 1px solid var(--isa-border-strong);
}
.ei-col-detail {
  flex: 1 1 0;
  padding-left: var(--isa-space-4);
  border-left: 1px solid var(--isa-border-strong);
}
.ei-col-head {
  flex: none;
  min-height: var(--isa-icon-btn);
  padding-bottom: var(--isa-space-2);
  border-bottom: 1px solid var(--isa-border-strong);
  font-size: var(--isa-fs-overline);
  font-weight: 800;
  letter-spacing: var(--isa-ls-overline);
  text-transform: uppercase;
  color: var(--isa-text-secondary);
}
.ei-col-body {
  display: flex;
  flex: 1 1 auto;
  flex-direction: column;
  gap: var(--isa-space-3);
  min-height: 0;
  padding-top: var(--isa-space-3);
  overflow-y: auto;
  overscroll-behavior: contain;
  scrollbar-width: thin;
  scrollbar-color: var(--isa-scroll-thumb) transparent;
}
.ei-col-body > * {
  flex: none;
}
.ei-md-head {
  text-transform: none;
  letter-spacing: normal;
}
.ei-md-title {
  font-size: var(--isa-fs-overline);
  font-weight: 800;
  letter-spacing: var(--isa-ls-overline);
  text-transform: uppercase;
  color: var(--isa-accent);
}
.ei-md-sum {
  font-size: var(--isa-fs-value);
  font-weight: 800;
  color: var(--isa-text);
  overflow-wrap: anywhere;
}
.ei-md-sum.ei-todo {
  font-style: italic;
  font-weight: 700;
  color: var(--isa-text-secondary);
}
.ei-md-fields {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(210px, 1fr));
  gap: var(--isa-space-4) var(--isa-space-5);
  align-items: start;
}
.ei-row[data-active] {
  border-color: var(--isa-accent);
  background: var(--isa-accent-soft);
}
.ei-row[data-active] > .ei-row-head .ei-row-chev {
  transform: none;
  color: var(--isa-accent);
}

/* maniglia di riordino delle righe (criteri di Ordina) */
.ei-row-grip {
  cursor: grab;
  touch-action: none;
}
.ei-row-grip:focus-visible {
  outline: 2px solid var(--isa-focus-ring);
  outline-offset: 1px;
}
.ei-row.ei-dragging {
  position: relative;
  z-index: 5;
  background: var(--isa-option-selected);
  box-shadow: var(--isa-shadow-raised);
}
.ei-list > .ei-row {
  transition: transform var(--isa-duration-base);
}
.ei-list > .ei-row.ei-dragging {
  transition: none;
}
@media (prefers-reduced-motion: reduce) {
  .ei-list > .ei-row {
    transition: none;
  }
}
```

### `src/etl-canvas/inspector/joinKeys.ts`

26 righe

```ts
/**
 * Cambi di modalità di una condizione di join, a funzioni pure (prototipo, `bindJoin`, righe
 * 3243-3253): il confronto segue il tipo del lato destro. Con una lista a destra il confronto
 * diventa di appartenenza («è uno di»); uscendo dalla lista torna «=». Il lato sinistro cambia
 * modalità senza toccare il confronto.
 */
import { LIST_OPS } from "../../etl-core";
import type { JoinKey, JoinOp, JoinRightMode, JoinSideMode } from "../../etl-core";

export function withLeftMode(k: JoinKey, mode: JoinSideMode): JoinKey {
  return { ...k, lmode: mode };
}

export function withRightMode(k: JoinKey, mode: JoinRightMode): JoinKey {
  const wasList = k.rmode === "list";
  let op: JoinOp = k.op ?? "=";
  if (mode === "list" && !LIST_OPS.includes(op)) op = LIST_OPS[0] as JoinOp;
  if (mode !== "list" && wasList) op = "=";
  return { ...k, rmode: mode, op };
}

/** Una condizione di join vuota (colonna = colonna): la lista le dà il connettore. */
export function blankJoinKey(): Omit<JoinKey, "conn" | "g"> {
  return { left: "", right: "", op: "=", lmode: "col", rmode: "col", lval: "", rval: "" };
}
```

### `src/etl-canvas/inspector/joinSides.ts`

87 righe

```ts
/**
 * Le tabelle di un passaggio di join e le colonne di ciascun lato. Funzioni pure.
 *
 * Tabelle assenti nei parametri (regola di etl-core, README «Tabelle di riferimento
 * del join»): la sinistra è la prima tabella in ingresso, la destra la seconda (se ce
 * n'è una sola, la prima); un valore salvato che non è più tra le tabelle si tratta come
 * assente. Nulla si scrive finché l'utente non sceglie.
 *
 * Colonne dei lati: il lato sinistro vede le colonne della tabella sinistra e il destro
 * quelle della tabella destra (`schemaOf` sul singolo ingresso). Dal secondo join in poi la
 * sinistra è il risultato del join precedente: tutte le colonne in ingresso. Se lo schema di
 * una tabella non è noto si ripiega sull'unione, come il prototipo.
 */
import { MERGE_OPS, inputsOf, schemaOf } from "../../etl-core";
import type { Card, ColumnDef, Graph, Params } from "../../etl-core";
import { textParam } from "./params";

export type TableKey = "leftTable" | "rightTable" | "table";

/** I nomi delle tabelle in ingresso, nell'ordine dei collegamenti. */
export function inputNames(graph: Graph, card: Card, fallback: string): string[] {
  return inputsOf(graph, card.id).map((l) => graph.cards[l.from]?.name ?? fallback);
}

/** La tabella mostrata per un campo: quella salvata se c'è ancora, altrimenti la predefinita. */
export function resolveTable(stored: string, key: TableKey, names: readonly string[]): string {
  if (stored && names.includes(stored)) return stored;
  return (key === "rightTable" && names[1] ? names[1] : names[0]) ?? "";
}

/** Posizione del passaggio tra i join del box (0 = il primo join), o -1 se non è un join. */
export function joinIndex(card: Card, step: number): number {
  const positions: number[] = [];
  card.components.forEach((c, i) => {
    if ((MERGE_OPS as readonly string[]).includes(c)) positions.push(i);
  });
  return positions.indexOf(step);
}

export interface JoinSideSchemas {
  readonly left: readonly ColumnDef[];
  readonly right: readonly ColumnDef[];
  /** `true` se entrambi i lati vengono dalla propria tabella; `false` se si è ripiegato sull'unione. */
  readonly perTable: boolean;
}

export function joinSideSchemas(
  graph: Graph,
  card: Card,
  step: number,
  par: Params,
  union: readonly ColumnDef[],
  fallbackName: string,
): JoinSideSchemas {
  const links = inputsOf(graph, card.id);
  const names = inputNames(graph, card, fallbackName);
  const of = (name: string): readonly ColumnDef[] | null => {
    const i = names.indexOf(name);
    const link = i >= 0 ? links[i] : undefined;
    return link ? schemaOf(graph, link.from) : null;
  };
  const first = joinIndex(card, step) <= 0;
  const left = first ? of(resolveTable(textParam(par, "leftTable"), "leftTable", names)) : null;
  const right = of(resolveTable(textParam(par, "rightTable"), "rightTable", names));
  return {
    left: left ?? union,
    right: right ?? union,
    perTable: left !== null && right !== null,
  };
}

/** Il dominio (valori distinti) della colonna `name` in uno schema, se si conosce. */
export function domainOfColumn(schema: readonly ColumnDef[], name: string): string[] {
  const def = name ? schema.find((c) => c.name === name) : undefined;
  return def ? [...def.values] : [];
}

/** Il tipo di una colonna, se la si conosce. */
export function typeOfColumn(schema: readonly ColumnDef[], name: string): ColumnDef["type"] | null {
  return schema.find((c) => c.name === name)?.type ?? null;
}

/** Colonna numerica: tastiera numerica nel campo di testo (mai input number). */
export function isNumericType(type: ColumnDef["type"] | null): boolean {
  return type === "integer" || type === "numerico";
}
```

### `src/etl-canvas/inspector/logic.ts`

286 righe

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

/**
 * Segmento di arrivo per un tasto di un radiogroup (frecce, Home, Fine); oltre l'ultimo si torna
 * al primo e viceversa. `null` se il tasto non sposta la scelta.
 */
export function segmentedTarget(index: number, count: number, key: string): number | null {
  if (count <= 0) return null;
  switch (key) {
    case "ArrowRight":
    case "ArrowDown":
      return (index + 1) % count;
    case "ArrowLeft":
    case "ArrowUp":
      return (index - 1 + count) % count;
    case "Home":
      return 0;
    case "End":
      return count - 1;
    default:
      return null;
  }
}

/** Cosa fa un tasto su una voce di un radiogroup: dove va il focus e quale voce si sceglie (`null` = nessuna nuova). */
export function segmentedKey(
  index: number,
  current: number,
  count: number,
  key: string,
): { focus: number; choose: number | null } | null {
  if (key === " " || key === "Enter")
    return { focus: index, choose: index === current ? null : index };
  const to = segmentedTarget(index, count, key);
  return to === null ? null : { focus: to, choose: to === current ? null : to };
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

/**
 * Dove finisce una riga trascinata di `dy` pixel: l'indice della riga il cui centro (a riposo)
 * è più vicino al centro della riga trascinata. Le righe possono avere altezze diverse
 * (una aperta è più alta delle altre).
 */
export function dropTarget(
  rects: readonly { readonly top: number; readonly height: number }[],
  from: number,
  dy: number,
): number {
  const start = rects[from];
  if (!start) return from;
  const center = start.top + start.height / 2 + dy;
  let best = from;
  let bestDistance = Infinity;
  rects.forEach((r, i) => {
    const d = Math.abs(r.top + r.height / 2 - center);
    if (d < bestDistance) {
      bestDistance = d;
      best = i;
    }
  });
  return best;
}

/** Dove sta ora la voce che stava in `index` dopo aver spostato `from` in `to`. */
export function indexAfterMove(index: number, from: number, to: number): number {
  if (index === from) return to;
  if (from < index && index <= to) return index - 1;
  if (to <= index && index < from) return index + 1;
  return index;
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

### `src/etl-canvas/inspector/masterDetail.ts`

43 righe

```ts
/**
 * Lo stato condiviso del layout a tre colonne (`Columns3`): quale voce è attiva e dove
 * vanno il titolo e il corpo del dettaglio. Le righe (`ListRow`) e le liste lo leggono; senza
 * `Columns3` il contesto è `null` e le righe si aprono sul posto.
 */
import { createContext, useContext, useEffect } from "react";

export interface MasterDetail {
  /** Dove va il titolo del dettaglio, e dove il suo corpo (`null` finché non sono montati). */
  readonly head: HTMLElement | null;
  readonly body: HTMLElement | null;
  /** La voce attiva: `lista:indice`. */
  readonly active: string;
  readonly setActive: (rowId: string) => void;
}

/** `null` = layout normale: le righe si comprimono e si aprono sul posto. */
export const MasterDetailContext = createContext<MasterDetail | null>(null);

export function useMasterDetail(): MasterDetail | null {
  return useContext(MasterDetailContext);
}

/** Identificativo di una voce: la lista e la posizione. */
export const rowIdOf = (listId: string, index: number): string => `${listId}:${index}`;

/**
 * Tiene valida la voce attiva quando una lista si accorcia (rimozione, annullamento):
 * se punta oltre l'ultima voce, passa all'ultima. Senza il layout a tre colonne non fa nulla.
 */
export function useActiveGuard(listId: string, count: number): void {
  const md = useMasterDetail();
  const active = md?.active;
  const setActive = md?.setActive;
  useEffect(() => {
    if (!active || !setActive || count <= 0) return;
    const prefix = `${listId}:`;
    if (!active.startsWith(prefix)) return;
    const index = Number(active.slice(prefix.length));
    if (Number.isInteger(index) && index >= count) setActive(rowIdOf(listId, count - 1));
  }, [active, setActive, listId, count]);
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

104 righe

```ts
/**
 * Lettura e scrittura dei parametri di un passaggio, a funzioni pure: restituiscono
 * parametri NUOVI, mai modificati sul posto. L'Inspector legge e scrive SOLO
 * `columns` (mai le righe espanse di `flattenRows`); le righe a colonna singola
 * del vecchio formato si migrano con `ensureMulti`.
 */
import { MULTI_DEFS, columnsOf, defaultParams, ensureMulti } from "../../etl-core";
import { moveItem } from "./logic";
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

/**
 * Le liste in cui l'ordine delle voci conta e si può cambiare (tipo di operazione → chiave della
 * lista): i criteri di Ordina, dove il primo è il principale (la nota è già in `MULTI_DEFS`).
 */
export const REORDERABLE_LISTS: Readonly<Record<string, string>> = { sort: "items" };

/** Sposta una voce di una lista da `from` a `to` (con `moveItem`); gli altri campi restano. */
export function withRowMoved(
  type: OperationType,
  par: Params,
  listKey: string,
  from: number,
  to: number,
): Params {
  const multi = multiOf(type, par);
  return { ...multi, [listKey]: moveItem(rowsOf(multi, listKey), from, to) };
}
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

