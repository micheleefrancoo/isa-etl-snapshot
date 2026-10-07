# 01b-etl-core-c.md

File in questo blocco:

- `src/etl-core/catalog/params.ts`
- `src/etl-core/data/csv.ts`
- `src/etl-core/index.ts`
- `src/etl-core/logic/expressions.ts`
- `src/etl-core/model/graph.ts`
- `src/etl-core/model/types.ts`

---

### `src/etl-core/catalog/params.ts`

1025 righe

```ts
/**
 * Definizioni dei parametri, valori predefiniti e migrazioni.
 * Porting letterale di PARAM_DEFS, MULTI_DEFS e delle relative costanti
 * (righe 2433-2617 di docs/prototype/isa-fusion-prototype.html), più le
 * migrazioni sparse nelle funzioni `render*` (renderFilter riga 3394,
 * ensureKeys riga 3124).
 */
import type {
  ColumnDef,
  ComponentId,
  FilterCondition,
  FlatRow,
  FilterParams,
  JoinKey,
  JoinOp,
  JoinParams,
  LogicOp,
  MultiFieldDef,
  MultiListDef,
  MultiOperationDef,
  MultiParams,
  MultiRow,
  OperationType,
  Params,
  SimpleFieldDef,
  ValuesField,
} from "../model/types";

// --- Vocabolari (prototipo: righe 2433-2439, 3142, 3158-3159, 3299-3303) ---

export const MULTI_OPS: readonly string[] = ["=", "≠", "è uno di", "non è uno di", "contiene"];
export const NO_VALUE_OPS: readonly string[] = ["è vuoto", "non è vuoto"];
export const FILTER_OPS: readonly string[] = [
  "=",
  "≠",
  "è uno di",
  "non è uno di",
  "contiene",
  ">",
  "<",
  "≥",
  "≤",
  "è vuoto",
  "non è vuoto",
];

export interface Separator {
  readonly label: string;
  readonly ch: string;
}

export const SEPARATORS: readonly Separator[] = [
  { label: "virgola", ch: "," },
  { label: "punto e virgola", ch: ";" },
  { label: "barra verticale", ch: "|" },
  { label: "a capo", ch: "\n" },
];

export const LIST_OPS: readonly string[] = ["è uno di", "non è uno di"];
export const JOIN_OPS: readonly JoinOp[] = ["=", "≠", "<", "≤", ">", "≥"];
export const JOIN_OP_NAME: readonly string[] = [
  "uguale a",
  "diverso da",
  "minore di",
  "minore o uguale a",
  "maggiore di",
  "maggiore o uguale a",
];

export const LOGIC_OPS: readonly LogicOp[] = ["AND", "OR", "XOR", "NAND", "NOR", "XNOR"];
export const LOGIC_HELP: Readonly<Record<LogicOp, string>> = {
  AND: "entrambe vere",
  OR: "almeno una vera",
  XOR: "una sola delle due vera",
  NAND: "non entrambe vere",
  NOR: "nessuna delle due vera",
  XNOR: "entrambe vere o entrambe false",
};

// --- Costruttori di valore predefinito (prototipo: newCondition, VALUES_DEF) ---

/** Prototipo, riga 2440-2442. */
export function newCondition(): FilterCondition {
  return { column: "", op: "=", mode: "list", values: [], text: "", sep: "," };
}

/** Prototipo, riga 2529 (`VALUES_DEF`). */
export function createValuesField(): ValuesField {
  return { mode: "list", values: [], text: "", sep: "," };
}

/** Prototipo, righe 2530-2533. */
export function valuesText(v: string | ValuesField | undefined): string {
  if (v === undefined) return "";
  if (typeof v === "string") return v;
  return v.mode === "list" ? v.values.join(", ") : v.text;
}

/**
 * Correzione intenzionale rispetto al prototipo (Fase 1.1, vedi
 * src/etl-core/NOTE_DIVERGENZE.md — "una sola fonte di verità per i
 * valori"): un campo a più valori conta solo `values`; `text` è solo un
 * formato di transito verso `values` (vedi `normalizeValuesField`), non
 * un secondo modo di essere "compilato". Nel prototipo (righe 2534-2537)
 * `fieldFilled` considerava compilato anche un `text` non vuoto rimasto
 * dalla modalità manuale.
 */
export function fieldFilled(
  f: { readonly type: string },
  v: string | string[] | ValuesField | undefined,
): boolean {
  if (f.type === "columns")
    return Array.isArray(v) && v.some((c) => typeof c === "string" && c !== "");
  if (f.type === "values") {
    const vf = v as ValuesField | undefined;
    return !!(vf && vf.values && vf.values.length > 0);
  }
  return !!(v && String(v).trim().length > 0);
}

/**
 * Correzione intenzionale rispetto al prototipo (Fase 1.1): migrazione
 * unica per ogni campo a più valori, oggi sparsa dentro `pickerHtml`
 * (prototipo, righe 2840-2848). Se `mode` è `'manual'` e `text` non è
 * vuoto, `text` viene diviso SOLO sul separatore registrato `sep` (o `,`
 * se assente), come la migrazione del prototipo: il testo era stato
 * scritto con quel separatore esplicito, quindi con `sep` `;` un valore
 * come "Rossi, Mario" resta intero. I token non vuoti vengono aggiunti a
 * `values` senza duplicati, poi `text` diventa `''` e `mode` diventa
 * `'list'`. Altrimenti il campo torna inalterato (mai mutato: restituisce
 * un nuovo oggetto solo se c'è qualcosa da migrare).
 *
 * La divisione su più separatori insieme riguarda solo l'inserimento dal
 * vivo nel selettore di valori: vedi `splitTokens`.
 */
export function normalizeValuesField(v: ValuesField): ValuesField {
  if (v.mode === "list" || !v.text || !v.text.trim()) return v;
  const tokens = v.text
    .split(v.sep || ",")
    .map((t) => t.trim())
    .filter((t) => t.length > 0);
  const values = v.values.slice();
  for (const t of tokens) {
    if (!values.includes(t)) values.push(t);
  }
  return { mode: "list", values, text: "", sep: v.sep };
}

/**
 * Prototipo, riga 2839 (`splitTokens`): divide un testo incollato o
 * scritto dal vivo nel selettore di valori su `,` `;` `|` e a capo, con
 * trim, senza token vuoti e senza duplicati. Solo per l'interfaccia: la
 * migrazione dei testi salvati usa `normalizeValuesField`, che divide
 * solo sul separatore registrato.
 */
export function splitTokens(text: string | null | undefined): string[] {
  const tokens = String(text ?? "")
    .split(/[,;|\n]/)
    .map((t) => t.trim())
    .filter((t) => t.length > 0);
  return Array.from(new Set(tokens));
}

/** Le colonne di una riga (`columns`), senza elementi non testuali. */
export function columnsOf(row: MultiRow): string[] {
  const v = row["columns"];
  return Array.isArray(v) ? v.filter((c): c is string => typeof c === "string") : [];
}

/**
 * Elenco di colonne per i riassunti: «a, b, c» fino a `max`, altrimenti le
 * prime `max - 1` e il resto contato («a, b +2»). Vuoto senza colonne.
 */
export function columnsText(columns: readonly string[], max = 3): string {
  if (columns.length <= max) return columns.join(", ");
  const shown = Math.max(1, max - 1);
  return `${columns.slice(0, shown).join(", ")} +${columns.length - shown}`;
}

function strField(row: MultiRow, key: string): string {
  const v = row[key];
  return typeof v === "string" ? v : "";
}

// --- PARAM_DEFS (prototipo, righe 2445-2525) --------------------------------

const COLUMN_FIELD = (label = "Colonna"): SimpleFieldDef => ({
  k: "column",
  label,
  type: "column",
  def: "",
});

/**
 * Definizioni a campo semplice, una per tipo di operazione (più `dataset`).
 * `filter` è `'custom'`: i suoi parametri (`FilterParams`) non seguono
 * questo schema generico, esattamente come nel prototipo.
 */
export const PARAM_DEFS: Readonly<Record<ComponentId, readonly SimpleFieldDef[] | "custom">> = {
  dataset: [
    {
      k: "source",
      label: "Origine",
      type: "select",
      opts: ["CSV", "Database", "API", "Foglio di calcolo"],
      def: "CSV",
    },
    { k: "path", label: "Percorso o tabella", type: "text", def: "" },
    {
      k: "header",
      label: "Prima riga di intestazione",
      type: "select",
      opts: ["Sì", "No"],
      def: "Sì",
    },
  ],
  filter: "custom",
  join: [
    {
      k: "type",
      label: "Tipo di join",
      type: "select",
      opts: ["inner", "left", "right", "full"],
      def: "inner",
    },
  ],
  sort: [
    COLUMN_FIELD(),
    {
      k: "dir",
      label: "Direzione",
      type: "select",
      opts: ["crescente", "decrescente"],
      def: "crescente",
    },
  ],
  exportOp: [
    {
      k: "format",
      label: "Formato",
      type: "select",
      opts: ["CSV", "XLSX", "Parquet", "Tabella DB"],
      def: "CSV",
    },
    { k: "dest", label: "Destinazione", type: "text", def: "", req: true },
  ],
  dedup: [
    COLUMN_FIELD("Colonna chiave"),
    {
      k: "keep",
      label: "Mantieni",
      type: "select",
      opts: ["la prima", "l’ultima"],
      def: "la prima",
    },
  ],
  limit: [
    { k: "n", label: "Numero di righe", type: "text", def: "100", req: true },
    { k: "from", label: "Dall’", type: "select", opts: ["inizio", "fine"], def: "inizio" },
  ],
  sample: [
    { k: "pct", label: "Percentuale", type: "text", def: "10", req: true },
    { k: "seed", label: "Seme casuale", type: "text", def: "" },
  ],
  selectCols: [
    COLUMN_FIELD("Colonna da tenere"),
    { k: "mode", label: "Modo", type: "select", opts: ["tieni", "escludi"], def: "tieni" },
  ],
  compute: [
    { k: "name", label: "Nuova colonna", type: "text", def: "", req: true },
    { k: "formula", label: "Formula", type: "text", def: "", req: true },
  ],
  cast: [
    COLUMN_FIELD(),
    {
      k: "to",
      label: "Nuovo tipo",
      type: "select",
      opts: ["intero", "decimale", "testo", "data", "booleano"],
      def: "decimale",
    },
  ],
  round: [COLUMN_FIELD(), { k: "decimals", label: "Decimali", type: "text", def: "2", req: true }],
  scale: [
    COLUMN_FIELD(),
    {
      k: "method",
      label: "Metodo",
      type: "select",
      opts: ["min-max", "z-score", "percentuale"],
      def: "min-max",
    },
  ],
  aggregate: [
    { k: "groupBy", label: "Raggruppa per", type: "column", def: "" },
    { k: "measure", label: "Misura", type: "column", def: "" },
    {
      k: "fn",
      label: "Funzione",
      type: "select",
      opts: ["somma", "media", "conteggio", "minimo", "massimo"],
      def: "somma",
    },
  ],
  textClean: [
    COLUMN_FIELD(),
    {
      k: "action",
      label: "Operazione",
      type: "select",
      opts: ["rimuovi spazi", "maiuscole", "minuscole", "iniziali maiuscole"],
      def: "rimuovi spazi",
    },
  ],
  replaceVal: [
    COLUMN_FIELD(),
    { k: "find", label: "Cerca", type: "text", def: "", req: true },
    { k: "with", label: "Sostituisci con", type: "text", def: "" },
  ],
  splitCol: [
    COLUMN_FIELD(),
    {
      k: "sep",
      label: "Separatore",
      type: "select",
      opts: [",", ";", "|", "spazio", "-"],
      def: ",",
    },
  ],
  rename: [COLUMN_FIELD(), { k: "newName", label: "Nuovo nome", type: "text", def: "", req: true }],
  fillNa: [
    COLUMN_FIELD(),
    { k: "value", label: "Valore di riempimento", type: "text", def: "", req: true },
  ],
  union: [
    { k: "mode", label: "Righe", type: "select", opts: ["tutte", "senza duplicati"], def: "tutte" },
    {
      k: "align",
      label: "Allineamento colonne",
      type: "select",
      opts: ["per nome", "per posizione"],
      def: "per nome",
    },
  ],
};

// --- MULTI_DEFS (prototipo, righe 2538-2582) --------------------------------

const COLF = (label?: string): MultiFieldDef => ({
  k: "column",
  label: label ?? "Colonna",
  type: "column",
  def: "",
});
/** Più colonne in una riga (Fase 6b.0): l'ordine è significativo. */
const COLSF = (label?: string): MultiFieldDef => ({
  k: "columns",
  label: label ?? "Colonne",
  type: "columns",
  def: (): string[] => [],
});
const VALUES_FIELD_DEF = (label: string, req = true): MultiFieldDef => ({
  k: "find",
  label,
  type: "values",
  def: createValuesField,
  req,
});

export const MULTI_DEFS: Readonly<Partial<Record<OperationType, MultiOperationDef>>> = {
  cast: {
    lists: [
      {
        key: "items",
        label: "Colonne da convertire",
        noun: "Conversione",
        add: "Aggiungi conversione",
        fields: [
          COLSF(),
          {
            k: "to",
            label: "Nuovo tipo",
            type: "select",
            opts: ["intero", "decimale", "testo", "data", "booleano"],
            def: "decimale",
          },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          return cols.length ? `${columnsText(cols)} → ${strField(r, "to")}` : null;
        },
      },
    ],
  },
  rename: {
    lists: [
      {
        key: "items",
        label: "Colonne da rinominare",
        noun: "Rinomina",
        add: "Aggiungi colonna",
        fields: [COLF(), { k: "newName", label: "Nuovo nome", type: "text", def: "", req: true }],
        sum: (r) =>
          strField(r, "column")
            ? `${strField(r, "column")} → ${strField(r, "newName") || "…"}`
            : null,
      },
    ],
  },
  fillNa: {
    lists: [
      {
        key: "items",
        label: "Colonne da riempire",
        noun: "Riempimento",
        add: "Aggiungi colonna",
        fields: [
          COLSF(),
          { k: "value", label: "Valore di riempimento", type: "value", def: "", req: true },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          return cols.length ? `${columnsText(cols)} = ${strField(r, "value") || "…"}` : null;
        },
      },
    ],
  },
  replaceVal: {
    lists: [
      {
        key: "items",
        label: "Sostituzioni",
        noun: "Sostituzione",
        add: "Aggiungi sostituzione",
        fields: [
          COLSF(),
          {
            k: "match",
            label: "Quando il valore",
            type: "select",
            opts: [
              "è uguale a",
              "è diverso da",
              "contiene",
              "inizia con",
              "finisce con",
              "corrisponde all’espressione",
            ],
            def: "è uguale a",
          },
          VALUES_FIELD_DEF("Valori da cercare"),
          { k: "with", label: "Sostituisci con", type: "value", def: "" },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          if (!cols.length) return null;
          const match = strField(r, "match") || "è uguale a";
          const find = valuesText(r["find"] as string | ValuesField | undefined) || "…";
          const withVal = strField(r, "with") || "∅";
          return `${columnsText(cols)} ${match} ${find} → ${withVal}`;
        },
      },
    ],
  },
  round: {
    lists: [
      {
        key: "items",
        label: "Colonne da arrotondare",
        noun: "Arrotondamento",
        add: "Aggiungi colonna",
        fields: [COLSF(), { k: "decimals", label: "Decimali", type: "text", def: "2", req: true }],
        sum: (r) => {
          const cols = columnsOf(r);
          return cols.length ? `${columnsText(cols)} · ${strField(r, "decimals")} decimali` : null;
        },
      },
    ],
  },
  scale: {
    lists: [
      {
        key: "items",
        label: "Colonne da normalizzare",
        noun: "Normalizzazione",
        add: "Aggiungi colonna",
        fields: [
          COLSF(),
          {
            k: "method",
            label: "Metodo",
            type: "select",
            opts: ["min-max", "z-score", "percentuale"],
            def: "min-max",
          },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          return cols.length ? `${columnsText(cols)} · ${strField(r, "method")}` : null;
        },
      },
    ],
  },
  textClean: {
    lists: [
      {
        key: "items",
        label: "Colonne da pulire",
        noun: "Pulizia",
        add: "Aggiungi colonna",
        fields: [
          COLSF(),
          {
            k: "action",
            label: "Operazione",
            type: "select",
            opts: ["rimuovi spazi", "maiuscole", "minuscole", "iniziali maiuscole"],
            def: "rimuovi spazi",
          },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          return cols.length ? `${columnsText(cols)} · ${strField(r, "action")}` : null;
        },
      },
    ],
  },
  compute: {
    lists: [
      {
        key: "items",
        label: "Colonne calcolate",
        noun: "Colonna",
        add: "Aggiungi colonna calcolata",
        fields: [
          { k: "name", label: "Nuova colonna", type: "text", def: "", req: true },
          { k: "formula", label: "Formula", type: "text", def: "", req: true },
        ],
        sum: (r) =>
          strField(r, "name") ? `${strField(r, "name")} = ${strField(r, "formula") || "…"}` : null,
      },
    ],
  },
  selectCols: {
    globals: [
      { k: "mode", label: "Modo", type: "select", opts: ["tieni", "escludi"], def: "tieni" },
    ],
    lists: [
      {
        key: "items",
        label: "Colonne",
        noun: "Colonna",
        add: "Aggiungi colonna",
        fields: [COLSF()],
        sum: (r) => {
          const cols = columnsOf(r);
          return cols.length ? columnsText(cols) : null;
        },
      },
    ],
  },
  dedup: {
    globals: [
      {
        k: "keep",
        label: "Mantieni",
        type: "select",
        opts: ["la prima", "l’ultima"],
        def: "la prima",
      },
    ],
    lists: [
      {
        key: "items",
        label: "Colonne chiave",
        noun: "Chiave",
        add: "Aggiungi chiave",
        fields: [
          COLSF(),
          {
            k: "cmp",
            label: "Confronto",
            type: "select",
            opts: ["esatto", "ignora maiuscole", "ignora spazi", "ignora maiuscole e spazi"],
            def: "esatto",
          },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          if (!cols.length) return null;
          const cmp = strField(r, "cmp");
          const text = columnsText(cols);
          return cmp && cmp !== "esatto" ? `${text} · ${cmp}` : text;
        },
        note: "Due righe sono duplicate quando coincidono su tutte le chiavi.",
      },
    ],
  },
  sort: {
    lists: [
      {
        key: "items",
        label: "Criteri di ordinamento",
        noun: "Criterio",
        add: "Aggiungi criterio",
        fields: [
          COLSF(),
          {
            k: "dir",
            label: "Direzione",
            type: "select",
            opts: ["crescente", "decrescente"],
            def: "crescente",
          },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          if (!cols.length) return null;
          return columnsText(cols) + (strField(r, "dir") === "crescente" ? " ↑" : " ↓");
        },
        note: "Il primo criterio è il principale; i successivi decidono a parità del precedente.",
      },
    ],
  },
  aggregate: {
    lists: [
      {
        key: "groupBy",
        label: "Raggruppa per",
        noun: "Chiave",
        add: "Aggiungi chiave",
        fields: [COLSF()],
        sum: (r) => {
          const cols = columnsOf(r);
          return cols.length ? columnsText(cols) : null;
        },
      },
      {
        key: "measures",
        label: "Misure",
        noun: "Misura",
        add: "Aggiungi misura",
        fields: [
          COLSF(),
          {
            k: "fn",
            label: "Funzione",
            type: "select",
            opts: ["somma", "media", "conteggio", "minimo", "massimo"],
            def: "somma",
          },
          { k: "alias", label: "Nome del risultato", type: "text", def: "" },
        ],
        sum: (r) => {
          const cols = columnsOf(r);
          if (!cols.length) return null;
          const fn = strField(r, "fn") || "somma";
          // con più colonne l'alias non si applica
          const alias = cols.length === 1 ? strField(r, "alias") : "";
          return `${fn}(${columnsText(cols)})${alias ? ` → ${alias}` : ""}`;
        },
      },
    ],
  },
};

// --- Valori predefiniti e migrazioni (prototipo, righe 2584-2617, 3124-3140, 3394-3400) ---

function blankRow(list: MultiListDef): MultiRow {
  const row: MultiRow = {};
  for (const f of list.fields) {
    row[f.k] = typeof f.def === "function" ? f.def() : f.def;
  }
  return row;
}

/**
 * Fase 6b.0: una riga con `column: "x"` diventa `columns: ["x"]` (stringa
 * vuota → `[]`); `columns` già presente e non vuoto prevale su `column`.
 * Idempotente: applicata due volte dà lo stesso risultato.
 */
function migrateColumns(row: MultiRow, key: string): string[] {
  const current = row[key];
  const legacy = row["column"];
  const fromLegacy = typeof legacy === "string" && legacy !== "" ? [legacy] : [];
  if (Array.isArray(current)) {
    const list = current.filter((c): c is string => typeof c === "string");
    return list.length > 0 ? list : fromLegacy;
  }
  if (typeof current === "string" && current !== "") return [current];
  return fromLegacy;
}

/** Prototipo, righe 2585-2602: migra il vecchio formato a voce singola in una lista di una riga. */
export function ensureMulti(type: OperationType, par: Params): MultiParams {
  const md = MULTI_DEFS[type];
  if (!md) return par as MultiParams;
  const next: MultiParams = { ...(par as MultiParams) };
  for (const f of md.globals ?? []) {
    if (next[f.k] === undefined) next[f.k] = f.def;
  }
  for (const list of md.lists) {
    const existing = next[list.key];
    if (!Array.isArray(existing)) {
      const row = blankRow(list);
      for (const f of list.fields) {
        if (f.type === "columns") {
          // formato a colonna singola (`column`): diventa una lista di una colonna
          const legacyColumn = next["column"];
          if (typeof legacyColumn === "string" && legacyColumn !== "") row[f.k] = [legacyColumn];
          continue;
        }
        const legacy = next[f.k];
        if (legacy !== undefined && legacy !== "") row[f.k] = legacy as string;
      }
      next[list.key] = [row];
    } else {
      next[list.key] = existing.map((row) => {
        const migrated: MultiRow = { ...row };
        for (const f of list.fields) {
          if (f.type === "columns") {
            migrated[f.k] = migrateColumns(migrated, f.k);
            delete migrated["column"];
          }
          if (f.type === "values") {
            const v = migrated[f.k];
            const asField: ValuesField =
              v === undefined || typeof v !== "object"
                ? v
                  ? { ...createValuesField(), mode: "manual", text: String(v) }
                  : createValuesField()
                : (v as ValuesField);
            migrated[f.k] = normalizeValuesField(asField);
          }
        }
        return migrated;
      });
    }
  }
  return next;
}

/**
 * Prototipo, righe 2604-2612. Correzione intenzionale rispetto al
 * prototipo (Fase 1.1): produce direttamente il formato attuale — il
 * filtro senza il campo `logic` (superato, mai stato lì fin dall'inizio
 * in questo dominio: non ha senso generarlo solo per poi migrarlo), il
 * join con le chiavi già passate da `ensureKeys` (op `'='`, `lmode`/
 * `rmode` `'col'`, `rlist` vuota, `lval`/`rval` vuoti).
 */
export function defaultParams(type: ComponentId): Params {
  if (type === "filter") {
    const params: FilterParams = { conditions: [newCondition()] };
    return params as unknown as Params;
  }
  if (type === "join") {
    const params: JoinParams = {
      type: "inner",
      keys: ensureKeys({ type: "inner", keys: [{ left: "", right: "" }] }),
    };
    return params as unknown as Params;
  }
  if (type !== "dataset" && MULTI_DEFS[type]) return ensureMulti(type, {});
  const defs = PARAM_DEFS[type];
  const out: Record<string, string> = {};
  if (Array.isArray(defs)) {
    for (const f of defs) out[f.k] = f.def;
  }
  return out;
}

/** Prototipo, righe 2613-2617: assicura che `card.params` abbia una voce per componente. */
export function ensureParamsFor(
  components: readonly ComponentId[],
  params: readonly Params[],
): Params[] {
  const next = params.slice();
  while (next.length < components.length) {
    next.push(defaultParams(components[next.length] as ComponentId));
  }
  return next;
}

/**
 * Prototipo, righe 3124-3140. Normalizza anche `rlist` (Fase 1.1: una
 * sola fonte di verità per i valori, vedi `normalizeValuesField`).
 */
export function ensureKeys(par: JoinParams): JoinKey[] {
  let keys = par.keys;
  if (!keys) {
    keys =
      par.leftKey || par.rightKey
        ? [{ left: par.leftKey ?? "", right: par.rightKey ?? "" }]
        : [{ left: "", right: "" }];
  }
  return keys.map((k) => ({
    ...k,
    op: k.op ?? "=",
    lmode: k.lmode ?? "col",
    rmode: k.rmode ?? "col",
    rlist: normalizeValuesField(
      k.rlist && typeof k.rlist === "object" ? k.rlist : createValuesField(),
    ),
    lval: k.lval ?? "",
    rval: k.rval ?? "",
  }));
}

/**
 * Prototipo, righe 3396-3400 (dentro `renderFilter`): il vecchio selettore
 * globale E/O diventa il connettore di ogni condizione dalla seconda in
 * poi. Applica anche (Fase 1.1, "una sola fonte di verità per i valori")
 * la migrazione testo → valori di `normalizeValuesField` a ogni
 * condizione, oggi sparsa dentro `pickerHtml` (prototipo, righe 2840-2848).
 */
export function migrateFilterLogic(par: FilterParams): FilterParams {
  const baseConditions = par.conditions ?? [newCondition()];
  const withLogic = par.logic
    ? baseConditions.map((c, i) =>
        i > 0 && !c.conn
          ? { ...c, conn: par.logic === "O" ? ("OR" as const) : ("AND" as const) }
          : c,
      )
    : baseConditions;
  const conditions = withLogic.map((c) => {
    const normalized = normalizeValuesField(c);
    return normalized === c
      ? c
      : {
          ...c,
          mode: normalized.mode,
          values: normalized.values,
          text: normalized.text,
          sep: normalized.sep,
        };
  });
  const { logic, ...rest } = par;
  void logic;
  return { ...rest, conditions };
}

// --- Riassunti (prototipo, righe 3109-3121, 3143-3194) ----------------------

/** Prototipo, righe 3143-3147: il testo di un lato di una chiave di join. */
export function sideText(
  k: JoinKey,
  side: "l" | "r",
  columnDef: (name: string) => { values: readonly string[] } | null,
): string {
  if (side === "l") return k.lmode === "val" ? (k.lval ? `“${k.lval}”` : "") : k.left;
  if (k.rmode === "val") return k.rval ? `“${k.rval}”` : "";
  if (k.rmode === "list") {
    const t = valuesText(k.rlist);
    return t ? `(${t})` : "";
  }
  return k.right;
  // `columnDef` è accettato per parità di firma con il prototipo (usato dal
  // chiamante per calcolare il dominio proposto), non serve qui.
  void columnDef;
}

/** Prototipo, righe 3149-3153. */
export function keyComplete(k: JoinKey): boolean {
  const l = k.lmode === "val" ? !!(k.lval && String(k.lval).trim()) : !!k.left;
  const r =
    k.rmode === "val"
      ? !!(k.rval && String(k.rval).trim())
      : k.rmode === "list"
        ? fieldFilled({ type: "values" }, k.rlist)
        : !!k.right;
  return l && r;
}

/** Prototipo, righe 3190-3194. */
export function summarizeKey(k: JoinKey): string | null {
  const l = k.lmode === "val" ? (k.lval ? `“${k.lval}”` : "") : k.left;
  const r =
    k.rmode === "val"
      ? k.rval
        ? `“${k.rval}”`
        : ""
      : k.rmode === "list"
        ? valuesText(k.rlist)
          ? `(${valuesText(k.rlist)})`
          : ""
        : k.right;
  if (!l && !r) return null;
  return `${l || "…"} ${k.op ?? "="} ${r || "…"}`;
}

/**
 * Prototipo, righe 3109-3121. Correzione intenzionale rispetto al
 * prototipo (Fase 1.1, vedi src/etl-core/NOTE_DIVERGENZE.md — "una sola
 * fonte di verità per i valori"): per gli operatori in `MULTI_OPS` il
 * riassunto usa sempre `values` (non `text`, e non serve più sapere se
 * la colonna ha un dominio noto: il parametro `columnHasValues` del
 * prototipo è stato rimosso). Per gli altri operatori usa `text`.
 */
export function summarizeCond(c: FilterCondition): string | null {
  if (!c.column) return null;
  if (NO_VALUE_OPS.includes(c.op)) return `${c.column} ${c.op}`;
  const v = MULTI_OPS.includes(c.op) ? c.values.join(", ") : c.text;
  if (!v) return `${c.column} ${c.op} …`;
  return `${c.column} ${c.op} ${v}`;
}

/**
 * Prototipo, righe 3219-3222: nessuna condizione di uguaglianza colonna =
 * colonna significa un confronto incrociato, potenzialmente molto lento.
 */
export function hasEquiJoinCondition(keys: readonly JoinKey[]): boolean {
  return keys.some((k) => k.lmode === "col" && k.rmode === "col" && (k.op ?? "=") === "=");
}

// --- Colonne multiple (Fase 6b.0) -------------------------------------------

/**
 * Nome dei risultati di una misura di «Raggruppa»: con più colonne è sempre
 * `${funzione}_${colonna}` e `alias` non si applica; con una colonna, `alias`
 * oppure `${funzione}_${colonna}`. Una per colonna, nell'ordine elencato.
 */
export function measureNames(row: MultiRow): string[] {
  const cols = columnsOf(row);
  const fn = strField(row, "fn") || "somma";
  const alias = strField(row, "alias");
  if (cols.length === 1) return [alias || `${fn}_${cols[0]}`];
  return cols.map((c) => `${fn}_${c}`);
}

/** Una riga espansa in più del suo indice d'origine non serve: l'ordine è quello delle colonne elencate. */
function expandRow(list: MultiListDef, row: MultiRow): FlatRow[] {
  const cols = columnsOf(row);
  const incomplete = list.fields.some(
    (f) => f.type !== "columns" && f.req && !fieldFilled(f, row[f.k]),
  );
  if (cols.length === 0 || incomplete) return [];
  const { columns, ...rest } = row;
  void columns;
  // con più colonne l'alias non si applica
  return cols.map((column) => ({
    ...rest,
    ...(cols.length > 1 && "alias" in rest ? { alias: "" } : {}),
    column,
  }));
}

/**
 * Righe espanse di un'operazione a voci multiple: una riga con N colonne
 * equivale a N righe identiche negli altri campi, nell'ordine in cui le
 * colonne sono elencate. Le righe incomplete (nessuna colonna o un campo
 * obbligatorio vuoto) si scartano. Restituisce una lista per ciascuna lista
 * dell'operazione (`items`, oppure `groupBy` e `measures`), con la colonna
 * nel campo `column`. Non modifica `params`. Per un tipo senza voci multiple
 * restituisce un oggetto vuoto.
 */
export function flattenRows(type: OperationType, params: Params): Record<string, FlatRow[]> {
  const md = MULTI_DEFS[type];
  if (!md) return {};
  const migrated = ensureMulti(type, params);
  const out: Record<string, FlatRow[]> = {};
  for (const list of md.lists) {
    const rows = migrated[list.key];
    out[list.key] = Array.isArray(rows)
      ? (rows as MultiRow[]).flatMap((row) => expandRow(list, row))
      : [];
  }
  return out;
}

/** Tetto dei valori proposti per un dominio (come `parseCSV`). */
export const DOMAIN_MAX = 500;

/**
 * Valori proposti per più colonne: l'unione, nell'ordine in cui le colonne
 * sono elencate (e, in ciascuna, nell'ordine del suo dominio), senza
 * duplicati, al massimo 500. Le colonne sconosciute si saltano.
 */
export function columnsDomain(
  schema: readonly ColumnDef[],
  columns: readonly string[],
  max = DOMAIN_MAX,
): string[] {
  const seen = new Set<string>();
  const out: string[] = [];
  for (const name of columns) {
    const def = schema.find((c) => c.name === name);
    if (!def) continue;
    for (const v of def.values) {
      if (seen.has(v)) continue;
      seen.add(v);
      out.push(v);
      if (out.length >= max) return out;
    }
  }
  return out;
}

/**
 * I valori scelti che non sono nel dominio: cambiare le colonne di una riga
 * non li azzera (differenza voluta dal prototipo), l'interfaccia li segnala.
 */
export function valuesOutsideDomain(
  values: readonly string[],
  domain: readonly string[],
): string[] {
  const inDomain = new Set(domain);
  return values.filter((v) => !inDomain.has(v));
}

/**
 * Tutte le migrazioni dei parametri di un componente (filtro, join, voci
 * multiple), idempotenti: usata al caricamento di un salvataggio di una
 * versione precedente. Non modifica `par`.
 */
export function ensureParams(type: ComponentId, par: Params): Params {
  if (type === "filter")
    return migrateFilterLogic(par as unknown as FilterParams) as unknown as Params;
  if (type === "join") {
    const jp = par as unknown as JoinParams;
    return { ...jp, keys: ensureKeys(jp) } as unknown as Params;
  }
  if (type !== "dataset" && MULTI_DEFS[type]) return ensureMulti(type, par);
  return par;
}
```

### `src/etl-core/data/csv.ts`

82 righe

```ts
/**
 * Lettura CSV e deduzione dei tipi. Porting letterale delle righe
 * 4672-4708 di docs/prototype/isa-fusion-prototype.html. Lavora su una
 * stringa già letta, non su `File`/`FileReader` (che non esistono in
 * Node).
 */
import type { ColumnDef, ColumnType } from "../model/types";

export interface ParsedCsv {
  readonly columns: readonly ColumnDef[];
  readonly rows: number;
}

const CANDIDATE_DELIMITERS = [",", ";", "\t", "|"];

function parseLine(line: string, delim: string): string[] {
  const out: string[] = [];
  let cur = "";
  let quoted = false;
  for (let i = 0; i < line.length; i += 1) {
    const ch = line[i];
    if (quoted) {
      if (ch === '"') {
        if (line[i + 1] === '"') {
          cur += '"';
          i += 1;
        } else {
          quoted = false;
        }
      } else {
        cur += ch;
      }
    } else if (ch === '"') {
      quoted = true;
    } else if (ch === delim) {
      out.push(cur);
      cur = "";
    } else {
      cur += ch;
    }
  }
  out.push(cur);
  return out.map((v) => v.trim());
}

/** Prototipo, righe 4672-4708. `null` se il testo non contiene righe non vuote. */
export function parseCSV(text: string): ParsedCsv | null {
  const lines = text
    .replace(/\r/g, "")
    .split("\n")
    .filter((l) => l.trim().length > 0);
  if (lines.length === 0) return null;
  const head = lines[0];
  if (head === undefined) return null;
  const delim =
    CANDIDATE_DELIMITERS.slice().sort((a, b) => head.split(b).length - head.split(a).length)[0] ??
    ",";

  const header = parseLine(head, delim);
  const rows = lines.slice(1, 1001).map((l) => parseLine(l, delim));

  const columns: ColumnDef[] = header.map((name, ci) => {
    const vals = rows.map((r) => r[ci]).filter((v): v is string => v !== undefined && v !== "");
    const isInt = vals.length > 0 && vals.every((v) => /^-?\d+$/.test(v));
    const isNum = vals.length > 0 && vals.every((v) => /^-?\d+([.,]\d+)?$/.test(v));
    const isDate =
      vals.length > 0 &&
      vals.every((v) => /^\d{4}-\d{2}-\d{2}/.test(v) || /^\d{1,2}\/\d{1,2}\/\d{2,4}$/.test(v));
    const type: ColumnType = isInt ? "integer" : isNum ? "numerico" : isDate ? "data" : "stringa";
    const distinct = Array.from(new Set(vals));
    const values =
      type === "integer" || type === "numerico"
        ? distinct
            .slice()
            .sort((a, b) => parseFloat(a.replace(",", ".")) - parseFloat(b.replace(",", ".")))
        : distinct;
    return { name: name || `colonna_${ci + 1}`, type, values: values.slice(0, 500) };
  });

  return { columns, rows: lines.length - 1 };
}
```

### `src/etl-core/index.ts`

109 righe

```ts
/**
 * Esportazioni pubbliche del dominio ETL (Fase 1). Vedi README.md per la
 * tabella di corrispondenza con le funzioni del prototipo
 * docs/prototype/isa-fusion-prototype.html.
 */

// --- Modello ------------------------------------------------------------
export * from "./model/types";
export {
  createGraph,
  cardById,
  inputsOf,
  outputOf,
  setCard,
  removeCard,
  removeCards,
  addLink,
  filterLinks,
  withLinks,
  patchCard,
  withParamAt,
  createSequentialIdGenerator,
} from "./model/graph";

// --- Catalogo -------------------------------------------------------------
export { ICONS, EMPTY_SLOT_ICON } from "./catalog/icons";
export { META, SECTIONS, MERGE_OPS, sectionOf } from "./catalog/operations";
export type { OperationMeta, SectionDef } from "./catalog/operations";
export {
  PARAM_DEFS,
  MULTI_DEFS,
  MULTI_OPS,
  NO_VALUE_OPS,
  FILTER_OPS,
  SEPARATORS,
  LIST_OPS,
  JOIN_OPS,
  JOIN_OP_NAME,
  LOGIC_OPS,
  LOGIC_HELP,
  newCondition,
  createValuesField,
  normalizeValuesField,
  splitTokens,
  valuesText,
  fieldFilled,
  ensureMulti,
  ensureParams,
  columnsOf,
  columnsText,
  measureNames,
  flattenRows,
  columnsDomain,
  valuesOutsideDomain,
  DOMAIN_MAX,
  defaultParams,
  ensureParamsFor,
  ensureKeys,
  migrateFilterLogic,
  sideText,
  keyComplete,
  summarizeKey,
  summarizeCond,
  hasEquiJoinCondition,
} from "./catalog/params";
export type { Separator } from "./catalog/params";

// --- Regole ---------------------------------------------------------------
export { boxCapacity, reaches, linkRefusal, relation, compatiblePair } from "./rules/relations";
export type { Relation, RelationResult } from "./rules/relations";
export {
  connect,
  spawnOutput,
  refreshOutput,
  pruneOutputs,
  enforceCapacity,
  nodesRemovedBy,
  deleteNodes,
  deleteLink,
  mergeBoxes,
  insertable,
  insertOnLink,
  detachStep,
  deleteStep,
  reorderSteps,
  duplicateNodes,
  isPartialOutput,
  defaultPositionFn,
} from "./rules/mutations";
export { stepMissing, nodeState } from "./rules/state";

// --- Logica -----------------------------------------------------------
export {
  groupRuns,
  normalizeGroups,
  groupPair,
  splitAt,
  ungroup,
  addToGroup,
  leftAssoc,
  groupedPreview,
} from "./logic/expressions";
export type { Groupable, GroupRun } from "./logic/expressions";

// --- Schema e CSV -----------------------------------------------------
export { schemaOf, columnsOutsideSchema } from "./schema/schema";
export { parseCSV } from "./data/csv";
export type { ParsedCsv } from "./data/csv";
```

### `src/etl-core/logic/expressions.ts`

172 righe

```ts
/**
 * Connettori, gruppi e anteprima delle espressioni logiche (condizioni di
 * filtro, chiavi di join). Porting letterale delle righe 3298-3392 di
 * docs/prototype/isa-fusion-prototype.html — MENO `logicPreview`
 * (superseduta da `groupedPreview`, non portata) e MENO l'HTML: qui
 * l'anteprima è una stringa pura.
 */
import type { LogicOp } from "../model/types";

/** Una voce che partecipa a un'espressione logica: una condizione o una chiave di join. */
export interface Groupable {
  readonly conn?: LogicOp;
  readonly g?: string;
}

export interface GroupRun {
  readonly s: number;
  readonly e: number;
  readonly g: string | null;
}

/** Prototipo, righe 3312-3323. */
export function groupRuns<T extends Groupable>(list: readonly T[]): GroupRun[] {
  const runs: GroupRun[] = [];
  let i = 0;
  while (i < list.length) {
    const g = list[i]?.g;
    let j = i;
    if (g) {
      while (j + 1 < list.length && list[j + 1]?.g === g) j += 1;
    }
    runs.push({ s: i, e: j, g: g ?? null });
    i = j + 1;
  }
  return runs;
}

/**
 * Prototipo, righe 3325-3328: un gruppo di una sola condizione non ha
 * senso e si scioglie. Restituisce una nuova lista (non muta l'input).
 */
export function normalizeGroups<T extends Groupable>(list: readonly T[]): T[] {
  const counts = new Map<string, number>();
  for (const item of list) {
    if (item.g) counts.set(item.g, (counts.get(item.g) ?? 0) + 1);
  }
  return list.map((item) => {
    if (item.g && (counts.get(item.g) ?? 0) < 2) {
      const { g, ...rest } = item;
      void g;
      return rest as T;
    }
    return item;
  });
}

/**
 * Prototipo, righe 3330-3336: raggruppa la condizione a `index-1` con quella
 * a `index`. Se appartenevano a due gruppi diversi, i due gruppi si
 * fondono. Restituisce una nuova lista.
 */
export function groupPair<T extends Groupable>(
  list: readonly T[],
  index: number,
  newGroupId: () => string,
): T[] {
  const a = list[index - 1];
  const b = list[index];
  if (!a || !b) return list.slice();
  const g = a.g ?? b.g ?? newGroupId();
  const otherGroup = a.g && b.g && a.g !== b.g ? b.g : null;
  return list.map((item) => {
    if (item === a || item === b) return { ...item, g } as T;
    if (otherGroup && item.g === otherGroup) return { ...item, g } as T;
    return item;
  });
}

/**
 * Prototipo, righe 3338-3343: divide il gruppo a partire da `index` in un
 * nuovo gruppo (tutto ciò che segue, appartenente allo stesso gruppo
 * originale, viene rinumerato). Restituisce una nuova lista.
 */
export function splitAt<T extends Groupable>(
  list: readonly T[],
  index: number,
  newGroupId: () => string,
): T[] {
  const g = list[index]?.g;
  if (!g) return list.slice();
  const ng = newGroupId();
  const next = list.slice();
  for (let k = index; k < next.length && next[k]?.g === g; k += 1) {
    next[k] = { ...next[k], g: ng } as T;
  }
  return next;
}

/** Prototipo, riga 3614: sciogliere un gruppo dissocia tutte le sue condizioni. */
export function ungroup<T extends Groupable>(list: readonly T[], groupId: string): T[] {
  return list.map((item) => {
    if (item.g !== groupId) return item;
    const { g, ...rest } = item;
    void g;
    return rest as T;
  });
}

/**
 * Prototipo, righe 3615-3621: aggiunge una nuova voce subito dopo l'ultima
 * del gruppo indicato. `makeItem` costruisce la voce di base (una
 * `FilterCondition` o una `JoinKey`), a cui viene assegnato `conn:'AND'` e
 * il gruppo. Restituisce la nuova lista e l'indice della voce inserita.
 */
export function addToGroup<T extends Groupable>(
  list: readonly T[],
  groupId: string,
  makeItem: () => Omit<T, "conn" | "g">,
): { list: T[]; index: number } {
  let last = -1;
  list.forEach((item, i) => {
    if (item.g === groupId) last = i;
  });
  const insertAt = last + 1;
  const item = { ...makeItem(), conn: "AND" as const, g: groupId } as T;
  const next = list.slice();
  next.splice(insertAt, 0, item);
  return { list: next, index: insertAt };
}

/**
 * Prototipo, righe 3366-3369 (`leftAssoc`): valutazione da sinistra a
 * destra, con parentesi a partire dal terzo elemento. `conns[i]` è il
 * connettore tra `parts[i-1]` e `parts[i]` (ignorato per `i === 0`).
 */
export function leftAssoc(
  parts: readonly string[],
  conns: readonly (LogicOp | undefined)[],
): string {
  let expr = parts[0] ?? "";
  for (let i = 1; i < parts.length; i += 1) {
    const left = i > 1 ? `(${expr})` : expr;
    expr = `${left} ${conns[i] ?? "AND"} ${parts[i] ?? ""}`;
  }
  return expr;
}

/**
 * Prototipo, righe 3371-3383 (`groupedPreview`), SENZA involucro HTML: solo
 * la stringa dell'espressione. Restituisce `''` se la lista ha meno di 2
 * voci (non c'è nulla da combinare).
 */
export function groupedPreview<T extends Groupable>(
  list: readonly T[],
  summarize: (item: T) => string,
): string {
  if (list.length < 2) return "";
  const runs = groupRuns(list);
  const parts = runs.map((r) => {
    if (!r.g) return summarize(list[r.s] as T);
    const inner: string[] = [];
    const innerConns: (LogicOp | undefined)[] = [];
    for (let k = r.s; k <= r.e; k += 1) {
      inner.push(summarize(list[k] as T));
      innerConns.push(k === r.s ? undefined : list[k]?.conn);
    }
    return `(${leftAssoc(inner, innerConns)})`;
  });
  const topConns = runs.map((r) => list[r.s]?.conn);
  return leftAssoc(parts, topConns);
}
```

### `src/etl-core/model/graph.ts`

98 righe

```ts
/**
 * Creazione e lettura del grafo. Porting di frammenti sparsi nel
 * prototipo (l'oggetto globale `cards`/`linksArr` diventa un valore
 * immutabile `Graph`).
 */
import type { Card, Graph, IdGenerator, Link, Params } from "./types";

export function createGraph(): Graph {
  return { cards: {}, links: [] };
}

/** Prototipo: `cards[uid]`. */
export function cardById(graph: Graph, id: string): Card | undefined {
  return graph.cards[id];
}

/** Prototipo, riga 1613: `linksArr.filter(l => l.to === boxUid)`. */
export function inputsOf(graph: Graph, boxId: string): Link[] {
  return graph.links.filter((l) => l.to === boxId);
}

/** Prototipo, riga 1614: `linksArr.find(l => l.from === boxUid)`. */
export function outputOf(graph: Graph, boxId: string): string | null {
  const l = graph.links.find((link) => link.from === boxId);
  return l ? l.to : null;
}

// --- Helper immutabili per rules/mutations.ts --------------------------

/** Nuovo grafo con una card creata o sostituita. */
export function setCard(graph: Graph, card: Card): Graph {
  return { cards: { ...graph.cards, [card.id]: card }, links: graph.links };
}

/** Nuovo grafo senza la card indicata. */
export function removeCard(graph: Graph, id: string): Graph {
  if (!(id in graph.cards)) return graph;
  const cards = { ...graph.cards };
  delete cards[id];
  return { cards, links: graph.links };
}

/** Nuovo grafo senza le card indicate. */
export function removeCards(graph: Graph, ids: ReadonlySet<string> | readonly string[]): Graph {
  const idSet = ids instanceof Set ? ids : new Set(ids);
  if (idSet.size === 0) return graph;
  const cards = { ...graph.cards };
  let changed = false;
  for (const id of idSet) {
    if (id in cards) {
      delete cards[id];
      changed = true;
    }
  }
  return changed ? { cards, links: graph.links } : graph;
}

/** Nuovo grafo con un collegamento in più. */
export function addLink(graph: Graph, link: Link): Graph {
  return { cards: graph.cards, links: [...graph.links, link] };
}

/** Nuovo grafo con i soli collegamenti che soddisfano il predicato. */
export function filterLinks(graph: Graph, predicate: (link: Link) => boolean): Graph {
  const links = graph.links.filter(predicate);
  return links.length === graph.links.length ? graph : { cards: graph.cards, links };
}

/** Nuovo grafo con i collegamenti sostituiti interamente. */
export function withLinks(graph: Graph, links: readonly Link[]): Graph {
  return { cards: graph.cards, links };
}

/** Nuova card con alcuni campi sostituiti (equivalente a `Object.assign({}, card, patch)`). */
export function patchCard(card: Card, patch: Partial<Card>): Card {
  return { ...card, ...patch };
}

/** Nuova card con un parametro di un componente sostituito. */
export function withParamAt(card: Card, index: number, params: Params): Card {
  const next = card.params.slice();
  next[index] = params;
  return { ...card, params: next };
}

/**
 * Generatore di id deterministico e iniettabile (prototipo: `uidCounter`,
 * `outCounter`, ... contatori globali con prefisso). Non richiesto in
 * produzione con questa firma esatta: qualunque `IdGenerator` va bene.
 */
export function createSequentialIdGenerator(prefix = ""): IdGenerator {
  let counter = 0;
  return () => {
    counter += 1;
    return `${prefix}${counter}`;
  };
}
```

### `src/etl-core/model/types.ts`

237 righe

```ts
/**
 * Modello dati del dominio ETL, porting del prototipo
 * docs/prototype/isa-fusion-prototype.html (oggetto `cards` + array
 * `linksArr`). Nessuna dipendenza da React/DOM: deve funzionare identico
 * in Node (rendering lato server di TanStack Start).
 */

// --- Colonne e schema -------------------------------------------------------

/** Tipo dedotto da parseCSV (data/csv.ts). Il prototipo usa questi 4 valori. */
export type ColumnType = "integer" | "numerico" | "data" | "stringa";

export interface ColumnDef {
  readonly name: string;
  readonly type: ColumnType;
  /** Valori distinti noti (fino a 500), usati dai selettori di valore. */
  readonly values: readonly string[];
}

// --- Operazioni ---------------------------------------------------------

/** I 19 tipi di operazione del prototipo (catalog/operations.ts). */
export type OperationType =
  | "filter"
  | "join"
  | "sort"
  | "exportOp"
  | "dedup"
  | "limit"
  | "sample"
  | "selectCols"
  | "compute"
  | "cast"
  | "round"
  | "scale"
  | "aggregate"
  | "textClean"
  | "replaceVal"
  | "splitCol"
  | "rename"
  | "fillNa"
  | "union";

/** Uno dei componenti di una card: un'operazione, oppure lo pseudo-tipo 'dataset'. */
export type ComponentId = OperationType | "dataset";

// --- Parametri per voci a campo semplice (PARAM_DEFS) -----------------------

export type SimpleFieldType = "text" | "select" | "column";

export interface SimpleFieldDef {
  readonly k: string;
  readonly label: string;
  readonly type: SimpleFieldType;
  readonly opts?: readonly string[];
  readonly def: string;
  readonly req?: boolean;
}

// --- Parametri a voci multiple (MULTI_DEFS) ---------------------------------

/** Valori scelti dal dominio di una colonna: elenco a spunta oppure testo con separatore. */
export interface ValuesField {
  mode: "list" | "manual";
  values: string[];
  text: string;
  sep: string;
}

export type MultiFieldType = "text" | "select" | "column" | "columns" | "value" | "values";

export interface MultiFieldDef {
  readonly k: string;
  readonly label: string;
  readonly type: MultiFieldType;
  readonly opts?: readonly string[];
  /** Valore predefinito, oppure funzione che lo produce (per i campi `values` e `columns`). */
  readonly def: string | (() => ValuesField) | (() => string[]);
  readonly req?: boolean;
}

/** Una riga di una lista MULTI_DEFS (es. una conversione, una chiave di raggruppamento). */
export interface MultiRow {
  [fieldKey: string]: string | string[] | ValuesField | undefined;
}

/**
 * Riga espansa (`flattenRows`): una colonna sola, `column`, e gli altri campi
 * della riga d'origine. Una riga con N colonne diventa N righe espanse.
 */
export interface FlatRow {
  [fieldKey: string]: string | ValuesField | undefined;
  column: string;
}

export interface MultiListDef {
  readonly key: string;
  readonly label: string;
  readonly noun: string;
  readonly add: string;
  readonly fields: readonly MultiFieldDef[];
  readonly sum: (row: MultiRow) => string | null;
  readonly note?: string;
}

export interface MultiOperationDef {
  readonly globals?: readonly SimpleFieldDef[];
  readonly lists: readonly MultiListDef[];
}

/** Parametri di un'operazione a voci multiple: globali + una o più liste di righe. */
export interface MultiParams {
  [key: string]: string | MultiRow[] | undefined;
}

// --- Filtro -----------------------------------------------------------------

export type LogicOp = "AND" | "OR" | "XOR" | "NAND" | "NOR" | "XNOR";

export type FilterOp =
  | "="
  | "≠" // ≠
  | "è uno di" // è uno di
  | "non è uno di" // non è uno di
  | "contiene"
  | ">"
  | "<"
  | "≥" // ≥
  | "≤" // ≤
  | "è vuoto" // è vuoto
  | "non è vuoto"; // non è vuoto

/** Una condizione del filtro. Dalla seconda in poi porta il proprio connettore. */
export interface FilterCondition {
  column: string;
  op: FilterOp;
  mode: "list" | "manual";
  values: string[];
  text: string;
  sep: string;
  /** Connettore con la condizione precedente (assente sulla prima). */
  conn?: LogicOp;
  /** Identificativo di gruppo: condizioni contigue con lo stesso id si valutano insieme. */
  g?: string;
}

export interface FilterParams {
  /** Vecchio formato (selettore globale E/O): migrato da migrateFilterLogic. */
  logic?: "E" | "O";
  conditions: FilterCondition[];
}

// --- Join ---------------------------------------------------------------

export type JoinType = "inner" | "left" | "right" | "full";
export type JoinSideMode = "col" | "val";
export type JoinRightMode = "col" | "val" | "list";
export type JoinOp = "=" | "≠" | "<" | "≤" | ">" | "≥" | "è uno di" | "non è uno di";

/** Una condizione di unione: lato sinistro e destro, ciascuno colonna, valore o (a destra) lista. */
export interface JoinKey {
  left: string;
  right: string;
  op?: JoinOp;
  lmode?: JoinSideMode;
  rmode?: JoinRightMode;
  lval?: string;
  rval?: string;
  rlist?: ValuesField;
  conn?: LogicOp;
  g?: string;
}

export interface JoinParams {
  type: JoinType;
  keys: JoinKey[];
  /** Vecchio formato a chiave singola: migrato da ensureKeys. */
  leftKey?: string;
  rightKey?: string;
}

// --- Dataset ------------------------------------------------------------

export interface DatasetParams {
  source?: string;
  path?: string;
  header?: string;
  /** Presenti solo dopo il caricamento (parseCSV): rendono il dataset una sorgente di schema. */
  columns?: ColumnDef[];
}

/** Parametri generici: ogni funzione che dispatcha su `type` restringe questo tipo. */
export type Params = Record<string, unknown>;

// --- Grafo ------------------------------------------------------------------

export interface Card {
  readonly id: string;
  readonly kind: "dataset" | "op";
  /**
   * Tipi di operazione in ordine di esecuzione; per un dataset è sempre
   * `['dataset']`. Un box combinato ha più di un componente.
   */
  readonly components: readonly ComponentId[];
  /** Un oggetto parametri per componente, nello stesso ordine. */
  readonly params: readonly Params[];
  readonly name: string;
  readonly x: number;
  readonly y: number;
  /** Postazione nella modalità Organizzato (fuori dall'ambito di questa fase). */
  readonly slot?: number;
  /** Solo per i dataset generati come output di un box. */
  readonly isOutput?: boolean;
  readonly capacity?: number;
  readonly filled?: number;
}

export interface Link {
  readonly from: string;
  readonly to: string;
}

export interface Graph {
  readonly cards: Readonly<Record<string, Card>>;
  readonly links: readonly Link[];
}

/** Generatore di identificativi iniettabile, per test deterministici. */
export type IdGenerator = () => string;

/** Funzione di posizionamento per i nodi generati dal dominio (output, nodi sganciati, ...). */
export type PositionFn = (graph: Graph, anchorId: string) => { x: number; y: number };

/** Esito di un'operazione che può essere rifiutata con un motivo testuale. */
export type OperationResult =
  { readonly ok: true; readonly graph: Graph } | { readonly ok: false; readonly reason: string };
```

