# ISA ETL Snapshot

Generated: 2026-09-11T21:45:15Z

## Index
- src/lib/etl-bubble.ts
- src/lib/etl-catalog.ts
- src/lib/etl-display.ts
- src/lib/etl-motion.ts
- src/lib/etl-node-config.ts
- src/lib/etl-schema.ts
- src/lib/etl-workflow.tsx


=== FILE: src/lib/etl-bubble.ts ===
/**
 * Geometria delle "bubble" — gruppi di 2+ card transform combinate
 * insieme (`EtlNode.groupId`). Isolato dal componente canvas perché
 * puramente geometrico (nessun accesso a DOM/React) e perché servirà
 * anche alla fase 3 (animazioni di resize/realign della bubble).
 */

export type BubbleOrientation = "horizontal" | "vertical";

/** Sottoinsieme dei campi di NodeGeometry che servono per la geometria della bubble. */
export type BubbleMember = {
  id: string;
  x: number;
  y: number;
  width: number;
  height: number;
};

export type BubbleRect = {
  x: number;
  y: number;
  width: number;
  height: number;
};

export type BubbleGeometry = {
  groupId: string;
  memberIds: string[];
  /** Bounding box dei membri, SENZA il padding visivo del contenitore disegnato. */
  rect: BubbleRect;
  orientation: BubbleOrientation;
};

/** Bounding box che racchiude tutti i `members`. */
export function computeBubbleRect(
  members: readonly BubbleMember[],
): BubbleRect {
  const first = members[0];

  if (!first) {
    return { x: 0, y: 0, width: 0, height: 0 };
  }

  let minX = first.x;
  let minY = first.y;
  let maxX = first.x + first.width;
  let maxY = first.y + first.height;

  for (const member of members) {
    minX = Math.min(minX, member.x);
    minY = Math.min(minY, member.y);
    maxX = Math.max(maxX, member.x + member.width);
    maxY = Math.max(maxY, member.y + member.height);
  }

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}

/**
 * Orientamento della bubble, derivato dal suo bounding box: più larga
 * che alta -> orizzontale (i membri sono affiancati, tipicamente
 * perché l'ultimo è stato trascinato dentro da sinistra/destra), più
 * alta che larga -> verticale (trascinato da sopra/sotto).
 *
 * Nessuno stato persistito: è una funzione pura del bounding box
 * corrente, ricalcolata a ogni render dalla disposizione reale dei
 * nodi — quindi cambia da sola se l'utente riorganizza le card dentro
 * o dentro/fuori dalla bubble, senza bisogno di tracciare la
 * direzione del gesto di drag.
 *
 * Bounding box (quasi) quadrato: nessuna indicazione nel brief su
 * quale verso preferire in questo caso limite — si sceglie
 * orizzontale come default deterministico, per evitare che
 * l'orientamento "sfarfalli" tra i due valori a parità di dimensioni.
 */
export function resolveBubbleOrientation(
  rect: BubbleRect,
): BubbleOrientation {
  return rect.width >= rect.height ? "horizontal" : "vertical";
}

/** Geometria completa della bubble per un gruppo con 2+ membri; `null` altrimenti (non è una bubble). */
export function computeBubbleGeometry(
  groupId: string,
  members: readonly BubbleMember[],
): BubbleGeometry | null {
  if (members.length < 2) {
    return null;
  }

  const rect = computeBubbleRect(members);

  return {
    groupId,
    memberIds: members.map((member) => member.id),
    rect,
    orientation: resolveBubbleOrientation(rect),
  };
}

/** Tutte le bubble presenti in un insieme di nodi, indicizzate per groupId. */
export function computeBubbles(
  nodes: readonly (BubbleMember & { groupId?: string | undefined })[],
): Map<string, BubbleGeometry> {
  const byGroup = new Map<string, BubbleMember[]>();

  for (const node of nodes) {
    if (!node.groupId) {
      continue;
    }

    const members = byGroup.get(node.groupId) ?? [];
    members.push(node);
    byGroup.set(node.groupId, members);
  }

  const bubbles = new Map<string, BubbleGeometry>();

  for (const [groupId, members] of byGroup) {
    const geometry = computeBubbleGeometry(groupId, members);

    if (geometry) {
      bubbles.set(groupId, geometry);
    }
  }

  return bubbles;
}


=== FILE: src/lib/etl-catalog.ts ===
import {
  Braces,
  Calculator,
  Combine,
  Copy,
  Database,
  Eraser,
  FileSpreadsheet,
  Filter,
  Gauge,
  Globe,
  Grid3x3,
  Layers,
  Pencil,
  Save,
  Search,
  Sigma,
  SortAsc,
  Table2,
} from "lucide-react";

export type ColumnType = "string" | "integer" | "decimal" | "date" | "boolean";

export type ColumnDef = { name: string; type: ColumnType; nullable: boolean };

export type EtlFieldKind = "text" | "select" | "textarea";

export type EtlField = {
  key: string;
  label: string;
  kind: EtlFieldKind;
  placeholder?: string | undefined;
  options?: string[] | undefined;
  defaultValue?: string | undefined;
  required?: boolean | undefined;
};

export type EtlCategory =
  | "sources"
  | "transform"
  | "combine"
  | "aggregate"
  | "output";

export type EtlNodeDef = {
  type: string;
  label: string;
  category: EtlCategory;
  description: string;
  Icon: typeof Database;
  inputs: string[];
  outputs: string[];
  fields: EtlField[];
};

export const ETL_CATEGORIES: { key: EtlCategory; label: string }[] = [
  { key: "sources", label: "Sources" },
  { key: "transform", label: "Transform" },
  { key: "combine", label: "Combine" },
  { key: "aggregate", label: "Aggregate" },
  { key: "output", label: "Output" },
];

/** Dataset di riferimento del workspace (metadati, nessuna esecuzione). */
export const SAMPLE_DATASETS: Record<
  string,
  { rows: number; columns: ColumnDef[] }
> = {
  "sales_2026.parquet": {
    rows: 125_000,
    columns: [
      { name: "order_id", type: "string", nullable: false },
      { name: "order_date", type: "date", nullable: false },
      { name: "customer_id", type: "string", nullable: false },
      { name: "product", type: "string", nullable: false },
      { name: "status", type: "string", nullable: false },
      { name: "quantity", type: "integer", nullable: false },
      { name: "price", type: "decimal", nullable: false },
      { name: "cost", type: "decimal", nullable: true },
      { name: "revenue", type: "decimal", nullable: false },
      { name: "country", type: "string", nullable: true },
      { name: "channel", type: "string", nullable: true },
      { name: "discount", type: "decimal", nullable: true },
      { name: "is_return", type: "boolean", nullable: false },
      { name: "updated_at", type: "date", nullable: true },
    ],
  },
  "customers.csv": {
    rows: 18_400,
    columns: [
      { name: "customer_id", type: "string", nullable: false },
      { name: "customer_name", type: "string", nullable: false },
      { name: "segment", type: "string", nullable: true },
      { name: "country", type: "string", nullable: true },
      { name: "signup_date", type: "date", nullable: false },
      { name: "active", type: "boolean", nullable: false },
    ],
  },
  "products.csv": {
    rows: 1_260,
    columns: [
      { name: "product", type: "string", nullable: false },
      { name: "category", type: "string", nullable: true },
      { name: "unit_cost", type: "decimal", nullable: true },
      { name: "supplier", type: "string", nullable: true },
    ],
  },
};

export const DATASET_NAMES = Object.keys(SAMPLE_DATASETS);

const COLUMN_TYPES: ColumnType[] = ["string", "integer", "decimal", "date", "boolean"];

export const ETL_NODES: EtlNodeDef[] = [
  {
    type: "source.dataset",
    label: "Dataset",
    category: "sources",
    description: "Dataset registrato nel workspace",
    Icon: Database,
    inputs: [],
    outputs: ["out"],
    fields: [
      {
        key: "dataset",
        label: "Dataset",
        kind: "select",
        options: DATASET_NAMES,
        defaultValue: DATASET_NAMES[0],
        required: true,
      },
    ],
  },
  {
    type: "source.file",
    label: "File",
    category: "sources",
    description: "CSV o Parquet locale",
    Icon: FileSpreadsheet,
    inputs: [],
    outputs: ["out"],
    fields: [
      { key: "path", label: "Percorso file", kind: "text", placeholder: "sales_2026.parquet", required: true },
      { key: "format", label: "Formato", kind: "select", options: ["csv", "parquet"], defaultValue: "parquet" },
      { key: "delimiter", label: "Delimitatore", kind: "select", options: [",", ";", "|", "tab"], defaultValue: "," },
    ],
  },
  {
    type: "source.sql",
    label: "SQL Query",
    category: "sources",
    description: "Query su database relazionale",
    Icon: Braces,
    inputs: [],
    outputs: ["out"],
    fields: [
      { key: "connection", label: "Connessione", kind: "text", placeholder: "warehouse_prod", required: true },
      { key: "query", label: "Query", kind: "textarea", placeholder: "select * from public.orders", required: true },
    ],
  },
  {
    type: "source.api",
    label: "API / External",
    category: "sources",
    description: "Endpoint HTTP JSON",
    Icon: Globe,
    inputs: [],
    outputs: ["out"],
    fields: [
      { key: "url", label: "Endpoint", kind: "text", placeholder: "https://api.esempio.it/v1/dati", required: true },
      { key: "method", label: "Metodo", kind: "select", options: ["GET", "POST"], defaultValue: "GET" },
      { key: "rootPath", label: "Percorso radice", kind: "text", placeholder: "data.items" },
    ],
  },

  {
    type: "transform.filter",
    label: "Filter",
    category: "transform",
    description: "Mantiene le righe che soddisfano la condizione",
    Icon: Filter,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Colonna", kind: "text", placeholder: "status", required: true },
      { key: "operator", label: "Operatore", kind: "select", options: ["=", "!=", ">", ">=", "<", "<=", "contains"], defaultValue: "=" },
      { key: "value", label: "Valore", kind: "text", placeholder: "ACTIVE", required: true },
    ],
  },
  {
    type: "transform.select",
    label: "Select Columns",
    category: "transform",
    description: "Proietta un sottoinsieme di colonne",
    Icon: Table2,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "columns", label: "Colonne", kind: "textarea", placeholder: "order_date, product, revenue", required: true },
    ],
  },
  {
    type: "transform.rename",
    label: "Rename Columns",
    category: "transform",
    description: "Rinomina colonne",
    Icon: Pencil,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "from", label: "Colonna", kind: "text", placeholder: "revenue", required: true },
      { key: "to", label: "Nuovo nome", kind: "text", placeholder: "ricavi", required: true },
    ],
  },
  {
    type: "transform.formula",
    label: "Formula",
    category: "transform",
    description: "Colonna calcolata",
    Icon: Calculator,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Nuova colonna", kind: "text", placeholder: "margine", required: true },
      { key: "expression", label: "Espressione", kind: "textarea", placeholder: "(price - cost) / price", required: true },
      { key: "type", label: "Tipo", kind: "select", options: COLUMN_TYPES, defaultValue: "decimal" },
    ],
  },
  {
    type: "transform.sort",
    label: "Sort",
    category: "transform",
    description: "Ordina il dataset",
    Icon: SortAsc,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Colonna", kind: "text", placeholder: "order_date", required: true },
      { key: "direction", label: "Direzione", kind: "select", options: ["asc", "desc"], defaultValue: "asc" },
    ],
  },
  {
    type: "transform.dedupe",
    label: "Remove Duplicates",
    category: "transform",
    description: "Rimuove righe duplicate",
    Icon: Copy,
    inputs: ["in"],
    outputs: ["out"],
    fields: [{ key: "columns", label: "Chiavi", kind: "text", placeholder: "order_id" }],
  },
  {
    type: "transform.fillna",
    label: "Fill Missing Values",
    category: "transform",
    description: "Sostituisce i valori mancanti",
    Icon: Eraser,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "column", label: "Colonna", kind: "text", placeholder: "cost", required: true },
      { key: "strategy", label: "Strategia", kind: "select", options: ["valore fisso", "media", "mediana", "ffill"], defaultValue: "valore fisso" },
      { key: "value", label: "Valore", kind: "text", placeholder: "0" },
    ],
  },

  {
    type: "combine.join",
    label: "Join",
    category: "combine",
    description: "Unisce due dataset su una chiave",
    Icon: Combine,
    inputs: ["left", "right"],
    outputs: ["out"],
    fields: [
      { key: "key", label: "Chiave", kind: "text", placeholder: "customer_id", required: true },
      { key: "how", label: "Tipo", kind: "select", options: ["inner", "left", "right", "outer"], defaultValue: "inner" },
    ],
  },
  {
    type: "combine.union",
    label: "Union",
    category: "combine",
    description: "Concatena dataset compatibili",
    Icon: Layers,
    inputs: ["a", "b"],
    outputs: ["out"],
    fields: [
      { key: "mode", label: "Modalità", kind: "select", options: ["colonne comuni", "tutte le colonne"], defaultValue: "colonne comuni" },
    ],
  },
  {
    type: "combine.lookup",
    label: "Lookup",
    category: "combine",
    description: "Arricchisce con una colonna di riferimento",
    Icon: Search,
    inputs: ["in", "lookup"],
    outputs: ["out"],
    fields: [
      { key: "key", label: "Chiave", kind: "text", placeholder: "product", required: true },
      { key: "column", label: "Colonna da aggiungere", kind: "text", placeholder: "category", required: true },
    ],
  },

  {
    type: "aggregate.groupBy",
    label: "Group By",
    category: "aggregate",
    description: "Raggruppa e calcola una metrica",
    Icon: Sigma,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "groupBy", label: "Raggruppa per", kind: "text", placeholder: "product", required: true },
      { key: "aggregation", label: "Funzione", kind: "select", options: ["SUM", "AVG", "MIN", "MAX", "COUNT"], defaultValue: "SUM" },
      { key: "metric", label: "Metrica", kind: "text", placeholder: "revenue", required: true },
    ],
  },
  {
    type: "aggregate.aggregate",
    label: "Aggregate",
    category: "aggregate",
    description: "Metrica globale sul dataset",
    Icon: Gauge,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "aggregation", label: "Funzione", kind: "select", options: ["SUM", "AVG", "MIN", "MAX", "COUNT"], defaultValue: "SUM" },
      { key: "metric", label: "Metrica", kind: "text", placeholder: "revenue", required: true },
    ],
  },
  {
    type: "aggregate.pivot",
    label: "Pivot",
    category: "aggregate",
    description: "Ruota i valori in colonne",
    Icon: Grid3x3,
    inputs: ["in"],
    outputs: ["out"],
    fields: [
      { key: "index", label: "Righe", kind: "text", placeholder: "product", required: true },
      { key: "columns", label: "Colonne", kind: "text", placeholder: "country", required: true },
      { key: "metric", label: "Valori", kind: "text", placeholder: "revenue", required: true },
    ],
  },

  {
    type: "output.dataset",
    label: "Output Dataset",
    category: "output",
    description: "Materializza il dataset per Model & Dashboard",
    Icon: Save,
    inputs: ["in"],
    outputs: [],
    fields: [
      { key: "name", label: "Nome dataset", kind: "text", placeholder: "dataset_vendite", required: true },
      { key: "mode", label: "Scrittura", kind: "select", options: ["replace", "append"], defaultValue: "replace" },
    ],
  },
  {
    type: "output.table",
    label: "Table",
    category: "output",
    description: "Tabella nell'interfaccia finale",
    Icon: Table2,
    inputs: ["in"],
    outputs: [],
    fields: [{ key: "title", label: "Titolo", kind: "text", placeholder: "Dettaglio ordini" }],
  },
];

export const nodeDef = (type: string) => ETL_NODES.find((n) => n.type === type);

export const categoryAccent = (category: EtlCategory) =>
  category === "output"
    ? "badge-type-dashboard"
    : category === "combine" || category === "aggregate"
      ? "badge-type-model"
      : "badge-type-etl";

/** Riepilogo sintetico mostrato nel nodo sul canvas. */
export function nodeSummary(type: string, config: Record<string, string>): string {
  const v = (k: string) => (config[k] ?? "").trim();
  switch (type) {
    case "source.dataset":
      return v("dataset") || "nessun dataset";
    case "source.file":
      return v("path") || "nessun file";
    case "source.sql":
      return v("query") ? v("query").slice(0, 42) : "nessuna query";
    case "source.api":
      return v("url") || "nessun endpoint";
    case "transform.filter":
      return v("column") ? `${v("column")} ${v("operator") || "="} "${v("value")}"` : "condizione mancante";
    case "transform.select":
      return v("columns") || "nessuna colonna";
    case "transform.rename":
      return v("from") ? `${v("from")} → ${v("to")}` : "nessuna colonna";
    case "transform.formula":
      return v("expression") ? `${v("column")} = ${v("expression")}` : "formula mancante";
    case "transform.sort":
      return v("column") ? `${v("column")} ${v("direction") || "asc"}` : "colonna mancante";
    case "transform.dedupe":
      return v("columns") ? `su ${v("columns")}` : "tutte le colonne";
    case "transform.fillna":
      return v("column") ? `${v("column")} · ${v("strategy")}` : "colonna mancante";
    case "combine.join":
      return v("key") ? `${v("how") || "inner"} on ${v("key")}` : "chiave mancante";
    case "combine.union":
      return v("mode") || "colonne comuni";
    case "combine.lookup":
      return v("key") ? `${v("column")} on ${v("key")}` : "chiave mancante";
    case "aggregate.groupBy":
      return v("metric") ? `${v("aggregation")}(${v("metric")}) by ${v("groupBy")}` : "metrica mancante";
    case "aggregate.aggregate":
      return v("metric") ? `${v("aggregation")}(${v("metric")})` : "metrica mancante";
    case "aggregate.pivot":
      return v("index") ? `${v("index")} × ${v("columns")}` : "configurazione mancante";
    case "output.dataset":
      return v("name") || "nome mancante";
    case "output.table":
      return v("title") || "tabella";
    default:
      return "";
  }
}


=== FILE: src/lib/etl-display.ts ===
import type { LucideIcon } from "lucide-react";
import {
  BarChart3,
  Combine,
  Database,
  Save,
  Sigma,
} from "lucide-react";
import type { EtlCategory } from "@/lib/etl-catalog";

/** Impostazioni di visualizzazione delle card nel canvas ETL. */
export type EtlDisplaySettings = {
  metrics: boolean;
  source: boolean;
  formula: boolean;
  status: boolean;
};

export const DEFAULT_DISPLAY: EtlDisplaySettings = {
  metrics: false,
  source: false,
  formula: false,
  status: true,
};

export const DISPLAY_OPTIONS: { key: keyof EtlDisplaySettings; label: string }[] = [
  { key: "metrics", label: "Mostra metriche righe/colonne" },
  { key: "source", label: "Mostra nome file sorgente" },
  { key: "formula", label: "Mostra formule" },
  { key: "status", label: "Mostra stato esecuzione" },
];

export const CATEGORY_ICONS: Record<EtlCategory, LucideIcon> = {
  sources: Database,
  transform: Sigma,
  combine: Combine,
  aggregate: BarChart3,
  output: Save,
};

/** Layout interno delle card "transform" (tutto ciò che non è "sources"). */
export type TransformCardLayout = {
  /** Lato del riquadro icona, in % del lato quadrato della card. */
  iconBoxSize: string;
  /** Lato del glifo Lucide, in % del riquadro icona. */
  iconGlyphSize: string;
};

/**
 * Calcolo puro del layout icona-centrale delle card "transform" (fase 1
 * del redesign, isolato dal JSX come richiesto per la futura estensione
 * con agenti/tool AI sulla codebase).
 *
 * Assunzione non specificata nel brief: 40% è stato scelto come valore
 * centrale del range indicato (35-45%) per il riquadro icona, con il
 * glifo Lucide al 50% del riquadro (indicazione "fulcro visivo
 * dominante", nessuna proporzione glifo/riquadro data). Libreria icone:
 * lucide-react, già usata in tutto il progetto.
 */
export function getTransformCardLayout(): TransformCardLayout {
  return {
    iconBoxSize: "40%",
    iconGlyphSize: "50%",
  };
}


=== FILE: src/lib/etl-motion.ts ===
/**
 * Costanti di durata/easing per i movimenti AUTOMATICI (indotti) di
 * card, bubble e frecce sul canvas ETL — centralizzate qui invece che
 * ripetute nei vari punti di workflow-canvas.tsx che le usano.
 *
 * Non riguardano mai il drag diretto dell'utente: quel movimento resta
 * sempre 1:1 col puntatore, senza transizione (vedi il commento su
 * `isUserDriven` in workflow-canvas.tsx per come viene distinto un
 * movimento "automatico" da un drag attivo).
 */

/** Durata di un movimento automatico, in ms. Range indicato: 150-250ms. */
export const AUTO_MOVE_DURATION_MS = 200;

/** Easing "morbido in uscita" per i movimenti automatici. */
export const AUTO_MOVE_EASING = "cubic-bezier(0.22, 1, 0.36, 1)";

const timing = `${AUTO_MOVE_DURATION_MS}ms ${AUTO_MOVE_EASING}`;

const transitionOf = (properties: readonly string[]): string =>
  properties.map((property) => `${property} ${timing}`).join(", ");

/** Transizione CSS applicata alla card quando si muove per un motivo diverso dal drag dell'utente. */
export const CARD_AUTO_MOVE_TRANSITION = transitionOf([
  "left",
  "top",
  "width",
  "height",
]);

/** Transizione CSS applicata al contenitore di una bubble (resize/orientamento/membri che entrano o escono). */
export const BUBBLE_AUTO_MOVE_TRANSITION = transitionOf([
  "left",
  "top",
  "width",
  "height",
]);

/**
 * Transizione CSS applicata al path di una freccia quando segue un
 * movimento automatico della card/bubble a cui è agganciata. Stessa
 * durata/easing della card, così i due non si muovono mai a scatti
 * indipendenti l'uno dall'altro.
 */
export const EDGE_AUTO_MOVE_TRANSITION = transitionOf([
  "d",
  "stroke-width",
  "stroke-opacity",
]);


=== FILE: src/lib/etl-node-config.ts ===
/**
 * Fase 4 — calcolo delle opzioni disponibili per i pannelli
 * impostazioni, dato lo schema EFFETTIVO del nodo (via `analyzeNode`),
 * separato dal rendering (i componenti React stanno sotto
 * src/components/isa/etl/settings-panels/).
 *
 * Nessuna esecuzione reale: come `etl-schema.ts`, sono stime
 * authoring-time basate sui `SAMPLE_DATASETS` del catalogo.
 */
import type { ColumnDef } from "./etl-catalog";
import { nodeDef } from "./etl-catalog";
import { analyzeNode, previewRows } from "./etl-schema";
import type { EtlNode, EtlWorkflow } from "./etl-workflow";

export type SettingsPanelKind =
  | "filter"
  | "combine"
  | "aggregate";

/**
 * Quale pannello impostazioni dedicato mostrare per un dato tipo di
 * nodo (fase 4) — `null` per i tipi non ancora coperti da un pannello
 * dedicato, che restano sul menu azioni generico esistente.
 */
export function getSettingsPanelKind(
  nodeType: string,
): SettingsPanelKind | null {
  if (nodeType === "transform.filter") {
    return "filter";
  }

  if (nodeType.startsWith("combine.")) {
    return "combine";
  }

  if (nodeType.startsWith("aggregate.")) {
    return "aggregate";
  }

  return null;
}

/* -------------------------------------------------------------------------- */
/*                                FILTER                                      */
/* -------------------------------------------------------------------------- */

/** Colonne disponibili per il filtro: lo schema in INGRESSO al nodo Filter. */
export function getFilterColumnOptions(
  workflow: EtlWorkflow,
  node: EtlNode,
): ColumnDef[] {
  return analyzeNode(workflow, node).inputColumns;
}

/**
 * Valori di dominio di una colonna, per il multi-select dei valori da
 * filtrare. Non esiste un dataset reale dietro `SAMPLE_DATASETS` (solo
 * nome/tipo/nullable per colonna): come placeholder strutturalmente
 * corretto riusiamo il generatore deterministico già presente in
 * `previewRows` (stesso seed => stessi valori a ogni render) invece di
 * inventare una seconda fonte di dati finti.
 */
export function getColumnDomainValues(
  column: ColumnDef,
  seed: string,
  sampleSize = 10,
): string[] {
  const rows = previewRows([column], seed, sampleSize);
  const values = rows.map((row) => row[0]).filter((v): v is string => v !== undefined);
  return Array.from(new Set(values));
}

/* -------------------------------------------------------------------------- */
/*                                COMBINE / JOIN                              */
/* -------------------------------------------------------------------------- */

export type CombineInputSchema = {
  nodeId: string;
  title: string;
  columns: ColumnDef[];
};

/**
 * Dataset esterni collegati a un nodo "combine" (o alla sua bubble, se
 * fa parte di un gruppo — fase 2): un ingresso per ogni edge che entra
 * nell'insieme di nodi da FUORI l'insieme stesso, deduplicato per nodo
 * sorgente. `memberIds` è l'insieme di id della bubble (inclusi
 * `node.id`); se assente o con un solo elemento si usa solo `node.id`
 * — così la funzione non deve conoscere la geometria della bubble
 * (nessuna dipendenza da lib/etl-bubble.ts), solo l'insieme di id.
 */
export function getCombineInputs(
  workflow: EtlWorkflow,
  node: EtlNode,
  memberIds?: readonly string[],
): CombineInputSchema[] {
  const members = new Set(
    memberIds && memberIds.length > 1 ? memberIds : [node.id],
  );

  const seen = new Set<string>();
  const inputs: CombineInputSchema[] = [];

  for (const edge of workflow.edges) {
    if (!members.has(edge.toNode) || members.has(edge.fromNode)) {
      continue;
    }

    if (seen.has(edge.fromNode)) {
      continue;
    }

    seen.add(edge.fromNode);

    const fromNode = workflow.nodes.find(
      (n) => n.id === edge.fromNode,
    );

    if (!fromNode) {
      continue;
    }

    inputs.push({
      nodeId: fromNode.id,
      title: fromNode.title,
      columns: analyzeNode(workflow, fromNode).columns,
    });
  }

  return inputs;
}

/** Opzioni del tipo di join, riusando quelle già dichiarate sul campo "how" del catalogo (nessuna lista duplicata). */
export function getJoinTypeOptions(): string[] {
  return (
    nodeDef("combine.join")?.fields.find(
      (field) => field.key === "how",
    )?.options ?? ["inner", "left", "right", "outer"]
  );
}

/** Chiavi di config per la colonna di join lato sinistro/destro della coppia `pairIndex` (0-based, consecutiva: input[i] ⋈ input[i+1]). */
export function joinPairConfigKeys(pairIndex: number): {
  left: string;
  right: string;
} {
  return {
    left: `join_pair_${pairIndex}_left`,
    right: `join_pair_${pairIndex}_right`,
  };
}

/* -------------------------------------------------------------------------- */
/*                                AGGREGATE                                   */
/* -------------------------------------------------------------------------- */

export type AggregateFieldSpec = {
  key: string;
  label: string;
  kind: "single" | "multi";
  options: ColumnDef[];
};

/** Selettori dinamici (colonna singola o multipla) per groupBy/aggregate/pivot, dallo schema effettivo in ingresso. */
export function getAggregateFieldSpecs(
  workflow: EtlWorkflow,
  node: EtlNode,
): AggregateFieldSpec[] {
  const inputColumns = analyzeNode(workflow, node).inputColumns;

  switch (node.type) {
    case "aggregate.groupBy":
      return [
        {
          key: "groupBy",
          label: "Raggruppa per",
          kind: "multi",
          options: inputColumns,
        },
        {
          key: "metric",
          label: "Metrica",
          kind: "single",
          options: inputColumns,
        },
      ];

    case "aggregate.aggregate":
      return [
        {
          key: "metric",
          label: "Metrica",
          kind: "single",
          options: inputColumns,
        },
      ];

    case "aggregate.pivot":
      return [
        {
          key: "index",
          label: "Righe",
          kind: "single",
          options: inputColumns,
        },
        {
          key: "columns",
          label: "Colonne",
          kind: "single",
          options: inputColumns,
        },
        {
          key: "metric",
          label: "Valori",
          kind: "single",
          options: inputColumns,
        },
      ];

    default:
      return [];
  }
}


=== FILE: src/lib/etl-schema.ts ===
import type { ColumnDef, ColumnType } from "./etl-catalog";
import { SAMPLE_DATASETS, nodeDef } from "./etl-catalog";
import type { EtlNode, EtlWorkflow } from "./etl-workflow";

export type NodeAnalysis = {
  inputColumns: ColumnDef[];
  columns: ColumnDef[];
  inRows: number | null;
  rows: number | null;
  errors: string[];
};

const splitList = (value: string) =>
  value
    .split(/[,\n]/)
    .map((s) => s.trim())
    .filter(Boolean);

const upstream = (workflow: EtlWorkflow, nodeId: string, port: string) => {
  const edge = workflow.edges.find((e) => e.toNode === nodeId && e.toPort === port);
  if (!edge) return undefined;
  return workflow.nodes.find((n) => n.id === edge.fromNode);
};

const datasetOf = (node: EtlNode) => {
  const name = node.config["dataset"] ?? "";
  return SAMPLE_DATASETS[name];
};

/** Stima statica di schema e volumi: authoring-time, nessuna esecuzione reale. */
export function analyzeNode(
  workflow: EtlWorkflow,
  node: EtlNode,
  depth = 0,
): NodeAnalysis {
  const def = nodeDef(node.type);
  const errors: string[] = [];
  if (!def || depth > 24) return { inputColumns: [], columns: [], inRows: null, rows: null, errors };

  for (const field of def.fields) {
    if (field.required && !(node.config[field.key] ?? "").trim()) {
      errors.push(`Campo obbligatorio mancante: ${field.label}.`);
    }
  }

  const parents = def.inputs.map((port) => {
    const parent = upstream(workflow, node.id, port);
    if (!parent) {
      errors.push(`Input "${port}" non collegato.`);
      return null;
    }
    return { port, analysis: analyzeNode(workflow, parent, depth + 1) };
  });

  const primary = parents.find((p) => p !== null)?.analysis;
  const inputColumns = primary?.columns ?? [];
  const inRows = primary?.rows ?? null;
  const cfg = node.config;
  const has = (name: string) => inputColumns.some((c) => c.name === name);
  const requireColumn = (key: string) => {
    const name = (cfg[key] ?? "").trim();
    if (name && inputColumns.length > 0 && !has(name)) {
      errors.push(`Colonna "${name}" non presente nello schema in ingresso.`);
    }
  };

  let columns = inputColumns;
  let rows = inRows;

  switch (node.type) {
    case "source.dataset": {
      const ds = datasetOf(node);
      if (!ds) {
        columns = [];
        rows = null;
      } else {
        columns = ds.columns;
        rows = ds.rows;
      }
      break;
    }
    case "source.file": {
      const guess = SAMPLE_DATASETS[(cfg["path"] ?? "").trim()];
      columns = guess?.columns ?? [];
      rows = guess?.rows ?? null;
      break;
    }
    case "source.sql":
    case "source.api": {
      columns = [];
      rows = null;
      break;
    }
    case "transform.filter": {
      requireColumn("column");
      rows = inRows === null ? null : Math.round(inRows * 0.66);
      break;
    }
    case "transform.select": {
      const wanted = splitList(cfg["columns"] ?? "");
      for (const name of wanted) {
        if (inputColumns.length > 0 && !has(name)) {
          errors.push(`Colonna "${name}" non presente nello schema in ingresso.`);
        }
      }
      columns = wanted.length
        ? wanted.map(
            (name) =>
              inputColumns.find((c) => c.name === name) ?? {
                name,
                type: "string" as ColumnType,
                nullable: true,
              },
          )
        : inputColumns;
      break;
    }
    case "transform.rename": {
      requireColumn("from");
      const from = (cfg["from"] ?? "").trim();
      const to = (cfg["to"] ?? "").trim();
      columns = inputColumns.map((c) => (c.name === from && to ? { ...c, name: to } : c));
      break;
    }
    case "transform.formula": {
      const name = (cfg["column"] ?? "").trim();
      columns = name
        ? [...inputColumns, { name, type: (cfg["type"] as ColumnType) || "decimal", nullable: true }]
        : inputColumns;
      break;
    }
    case "transform.sort": {
      requireColumn("column");
      break;
    }
    case "transform.dedupe": {
      rows = inRows === null ? null : Math.round(inRows * 0.92);
      break;
    }
    case "transform.fillna": {
      requireColumn("column");
      const name = (cfg["column"] ?? "").trim();
      columns = inputColumns.map((c) => (c.name === name ? { ...c, nullable: false } : c));
      break;
    }
    case "combine.join": {
      const right = parents[1]?.analysis;
      const rightCols = (right?.columns ?? []).filter(
        (c) => !inputColumns.some((l) => l.name === c.name),
      );
      columns = [...inputColumns, ...rightCols];
      rows =
        inRows === null
          ? null
          : (cfg["how"] ?? "inner") === "inner"
            ? Math.round(inRows * 0.88)
            : inRows;
      break;
    }
    case "combine.union": {
      const b = parents[1]?.analysis;
      rows = inRows === null || b?.rows == null ? inRows : inRows + b.rows;
      break;
    }
    case "combine.lookup": {
      const name = (cfg["column"] ?? "").trim();
      columns = name ? [...inputColumns, { name, type: "string", nullable: true }] : inputColumns;
      break;
    }
    case "aggregate.groupBy": {
      const groups = splitList(cfg["groupBy"] ?? "");
      for (const g of groups) {
        if (inputColumns.length > 0 && !has(g)) {
          errors.push(`Colonna "${g}" non presente nello schema in ingresso.`);
        }
      }
      requireColumn("metric");
      const metric = (cfg["metric"] ?? "metrica").trim();
      const agg = cfg["aggregation"] ?? "SUM";
      columns = [
        ...groups.map(
          (g) => inputColumns.find((c) => c.name === g) ?? { name: g, type: "string" as ColumnType, nullable: false },
        ),
        {
          name: `${agg.toLowerCase()}_${metric}`,
          type: agg === "COUNT" ? "integer" : "decimal",
          nullable: false,
        },
      ];
      rows = inRows === null ? null : Math.max(1, Math.round(inRows / 1500));
      break;
    }
    case "aggregate.aggregate": {
      requireColumn("metric");
      const metric = (cfg["metric"] ?? "metrica").trim();
      const agg = cfg["aggregation"] ?? "SUM";
      columns = [
        { name: `${agg.toLowerCase()}_${metric}`, type: agg === "COUNT" ? "integer" : "decimal", nullable: false },
      ];
      rows = 1;
      break;
    }
    case "aggregate.pivot": {
      const index = (cfg["index"] ?? "").trim();
      columns = [
        { name: index || "index", type: "string", nullable: false },
        { name: `${(cfg["columns"] ?? "colonna").trim()}_a`, type: "decimal", nullable: true },
        { name: `${(cfg["columns"] ?? "colonna").trim()}_b`, type: "decimal", nullable: true },
      ];
      rows = inRows === null ? null : Math.max(1, Math.round(inRows / 2000));
      break;
    }
    default:
      break;
  }

  return { inputColumns, columns, inRows, rows, errors };
}

export const formatRows = (rows: number | null) =>
  rows === null
    ? "—"
    : rows >= 1_000_000
      ? `${(rows / 1_000_000).toFixed(1)}M`
      : rows >= 1_000
        ? `${Math.round(rows / 1_000)}k`
        : String(rows);

/** Righe di anteprima deterministiche, coerenti col tipo di colonna. */
export function previewRows(columns: ColumnDef[], seed: string, count = 8) {
  let h = 0;
  for (let i = 0; i < seed.length; i += 1) h = (h * 31 + seed.charCodeAt(i)) % 100_000;
  const rnd = (n: number) => {
    h = (h * 1103515245 + 12345) % 2147483648;
    return h % n;
  };
  const words = ["Alpha", "Beta", "Gamma", "Delta", "Omega", "Nord", "Sud", "Retail", "Online"];
  return Array.from({ length: count }, (_, r) =>
    columns.map((col) => {
      switch (col.type) {
        case "integer":
          return String(rnd(400) + 1);
        case "decimal":
          return (rnd(900_00) / 100).toFixed(2);
        case "date":
          return `2026-0${(rnd(9) + 1).toString()}-${String(rnd(27) + 1).padStart(2, "0")}`;
        case "boolean":
          return rnd(2) === 0 ? "true" : "false";
        default:
          return `${words[rnd(words.length)]}-${r + 1}${rnd(90) + 10}`;
      }
    }),
  );
}


=== FILE: src/lib/etl-workflow.tsx ===
import { useCallback, useSyncExternalStore } from "react";
import { nodeDef } from "./etl-catalog";

export type NodeStatus = "ready" | "running" | "succeeded" | "error";

export type EtlNode = {
  id: string;
  type: string;
  title: string;
  x: number;
  y: number;
  config: Record<string, string>;
  status: NodeStatus;
  /** Nodi "transform" combinati insieme condividono lo stesso groupId. Opzionale e retrocompatibile. */
  groupId?: string | undefined;
};

export type EtlEdge = {
  id: string;
  fromNode: string;
  fromPort: string;
  toNode: string;
  toPort: string;
};

export type LayoutMode = "auto" | "manual";

export type EtlWorkflow = {
  nodes: EtlNode[];
  edges: EtlEdge[];
  layout: LayoutMode;
};

type Entry = {
  present: EtlWorkflow;
  past: EtlWorkflow[];
  future: EtlWorkflow[];
  savedAt: number;
};

const EMPTY: EtlWorkflow = { nodes: [], edges: [], layout: "auto" };
const EMPTY_ENTRY: Entry = { present: EMPTY, past: [], future: [], savedAt: 0 };

/** Passo della griglia di auto-layout (deve restare coerente con il canvas). */
export const LAYOUT_STEP_X = 260;
export const LAYOUT_STEP_Y = 120;
export const LAYOUT_ORIGIN = 32;

// Stato client-side dell'authoring del workflow, per soluzione.
// Nessuna esecuzione reale: il motore arriverà nella fase dedicata.
const store = new Map<string, Entry>();
const listeners = new Set<() => void>();
const emit = () => listeners.forEach((l) => l());

const uid = (p: string) => `${p}-${Math.random().toString(36).slice(2, 8)}`;

const node = (
  type: string,
  title: string,
  x: number,
  y: number,
  config: Record<string, string>,
): EtlNode => ({ id: uid("node"), type, title, x, y, config, status: "ready" });

/** Un gruppo con un solo membro non è più un gruppo: gli toglie il groupId. */
function dissolveSingletonGroups(nodes: EtlNode[]): EtlNode[] {
  const counts = new Map<string, number>();
  for (const n of nodes) {
    if (n.groupId) counts.set(n.groupId, (counts.get(n.groupId) ?? 0) + 1);
  }
  return nodes.map((n) => (n.groupId && (counts.get(n.groupId) ?? 0) < 2 ? { ...n, groupId: undefined } : n));
}

/** Disposizione automatica a colonne, seguendo la direzione del flusso dati. */
export function autoLayout(workflow: EtlWorkflow): EtlNode[] {
  const depth = new Map<string, number>();
  const order = pipelineOrder(workflow);
  for (const n of order) {
    const parents = workflow.edges.filter((e) => e.toNode === n.id);
    const d = parents.length
      ? Math.max(...parents.map((e) => (depth.get(e.fromNode) ?? 0) + 1))
      : 0;
    depth.set(n.id, d);
  }
  const perColumn = new Map<number, number>();
  const positions = new Map<string, { x: number; y: number }>();
  for (const n of order) {
    const d = depth.get(n.id) ?? 0;
    const row = perColumn.get(d) ?? 0;
    perColumn.set(d, row + 1);
    positions.set(n.id, {
      x: LAYOUT_ORIGIN + d * LAYOUT_STEP_X,
      y: LAYOUT_ORIGIN + row * LAYOUT_STEP_Y,
    });
  }
  return workflow.nodes.map((n) => ({ ...n, ...(positions.get(n.id) ?? {}) }));
}

const arranged = (w: EtlWorkflow): EtlWorkflow =>
  w.layout === "auto" ? { ...w, nodes: autoLayout(w) } : w;

const storageKey = (solutionId: string) => `isa.etl.workflow.${solutionId}`;

function load(solutionId: string): EtlWorkflow | null {
  if (typeof window === "undefined") return null;
  try {
    const raw = window.localStorage.getItem(storageKey(solutionId));
    if (!raw) return null;
    const parsed = JSON.parse(raw) as Partial<EtlWorkflow>;
    if (!Array.isArray(parsed.nodes) || !Array.isArray(parsed.edges)) return null;
    return {
      nodes: parsed.nodes,
      edges: parsed.edges,
      layout: parsed.layout === "manual" ? "manual" : "auto",
    };
  } catch {
    return null;
  }
}

function persist(solutionId: string, workflow: EtlWorkflow) {
  if (typeof window === "undefined") return;
  try {
    window.localStorage.setItem(storageKey(solutionId), JSON.stringify(workflow));
  } catch {
    /* storage non disponibile: lo stato resta in memoria */
  }
}

function entryOf(solutionId: string): Entry {
  const existing = store.get(solutionId);
  if (existing) return existing;
  const restored = load(solutionId);
  const created: Entry = {
    present: restored ? arranged(restored) : EMPTY,
    past: [],
    future: [],
    savedAt: Date.now(),
  };
  store.set(solutionId, created);
  return created;
}

const commit = (solutionId: string, next: EtlWorkflow, history = true) => {
  const entry = entryOf(solutionId);
  const present = arranged(next);
  store.set(solutionId, {
    present,
    past: history ? [...entry.past.slice(-40), entry.present] : entry.past,
    future: history ? [] : entry.future,
    savedAt: Date.now(),
  });
  persist(solutionId, present);
  emit();
};

export function useEtlWorkflow(solutionId: string) {
  const entry = useSyncExternalStore(
    (cb) => {
      listeners.add(cb);
      return () => listeners.delete(cb);
    },
    () => entryOf(solutionId),
    () => EMPTY_ENTRY,
  );
  const workflow = entry.present;

  const addNode = useCallback(
    (type: string, x: number, y: number, config?: Record<string, string>, title?: string) => {
      const def = nodeDef(type);
      if (!def) return undefined;
      const base: Record<string, string> = {};
      for (const f of def.fields) base[f.key] = f.defaultValue ?? "";
      const created = node(type, title ?? def.label, x, y, { ...base, ...config });
      const current = entryOf(solutionId).present;
      commit(solutionId, { ...current, nodes: [...current.nodes, created] });
      return created.id;
    },
    [solutionId],
  );

  const moveNode = useCallback(
    (id: string, x: number, y: number, history = false) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          // trascinare un nodo passa automaticamente al posizionamento manuale
          layout: "manual",
          nodes: current.nodes.map((n) => (n.id === id ? { ...n, x, y } : n)),
        },
        history,
      );
    },
    [solutionId],
  );

  const setLayout = useCallback(
    (layout: LayoutMode) => {
      const current = entryOf(solutionId).present;
      commit(solutionId, { ...current, layout });
    },
    [solutionId],
  );


  const updateNode = useCallback(
    (
      id: string,
      patch: { title?: string; config?: Record<string, string>; status?: NodeStatus },
      history = true,
    ) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          nodes: current.nodes.map((n) =>
            n.id === id
              ? {
                  ...n,
                  ...(patch.title !== undefined ? { title: patch.title } : {}),
                  ...(patch.status !== undefined ? { status: patch.status } : {}),
                  config: { ...n.config, ...patch.config },
                }
              : n,
          ),
        },
        history,
      );
    },
    [solutionId],
  );

  const removeNode = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      const nodes = dissolveSingletonGroups(current.nodes.filter((n) => n.id !== id));
      commit(solutionId, {
        ...current,
        nodes,
        edges: current.edges.filter((e) => e.fromNode !== id && e.toNode !== id),
      });
    },
    [solutionId],
  );

  const groupNodes = useCallback(
    (ids: string[]) => {
      if (ids.length < 2) return;
      const current = entryOf(solutionId).present;
      // Riunisce anche i membri di eventuali gruppi già esistenti a cui
      // appartengono gli id passati, così due gruppi che vengono uniti
      // finiscono davvero sotto un unico groupId, senza lasciare membri
      // orfani nel gruppo vecchio.
      const involvedGroupIds = new Set(
        current.nodes.filter((n) => ids.includes(n.id) && n.groupId).map((n) => n.groupId!),
      );
      const fullIds = new Set(ids);
      for (const n of current.nodes) {
        if (n.groupId && involvedGroupIds.has(n.groupId)) fullIds.add(n.id);
      }
      const groupId = involvedGroupIds.values().next().value ?? uid("group");
      commit(solutionId, {
        ...current,
        nodes: current.nodes.map((n) => (fullIds.has(n.id) ? { ...n, groupId } : n)),
      });
    },
    [solutionId],
  );

  const ungroupNode = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      const node = current.nodes.find((n) => n.id === id);
      if (!node?.groupId) return;
      const cleared = current.nodes.map((n) => (n.id === id ? { ...n, groupId: undefined } : n));
      commit(solutionId, { ...current, nodes: dissolveSingletonGroups(cleared) });
    },
    [solutionId],
  );

  const connect = useCallback(
    (fromNode: string, fromPort: string, toNode: string, toPort: string) => {
      if (fromNode === toNode) return;
      const current = entryOf(solutionId).present;
      if (
        current.edges.some(
          (e) => e.fromNode === fromNode && e.fromPort === fromPort && e.toNode === toNode && e.toPort === toPort,
        )
      )
        return;
      // una porta di input accetta una sola connessione
      const edges = current.edges.filter((e) => !(e.toNode === toNode && e.toPort === toPort));
      commit(solutionId, {
        ...current,
        edges: [...edges, { id: uid("edge"), fromNode, fromPort, toNode, toPort }],
      });
    },
    [solutionId],
  );

  const removeEdge = useCallback(
    (id: string) => {
      const current = entryOf(solutionId).present;
      commit(solutionId, { ...current, edges: current.edges.filter((e) => e.id !== id) });
    },
    [solutionId],
  );

  const setStatuses = useCallback(
    (updates: Record<string, NodeStatus>) => {
      const current = entryOf(solutionId).present;
      commit(
        solutionId,
        {
          ...current,
          nodes: current.nodes.map((n) => (updates[n.id] ? { ...n, status: updates[n.id]! } : n)),
        },
        false,
      );
    },
    [solutionId],
  );

  const undo = useCallback(() => {
    const e = entryOf(solutionId);
    const previous = e.past[e.past.length - 1];
    if (!previous) return;
    store.set(solutionId, {
      present: previous,
      past: e.past.slice(0, -1),
      future: [e.present, ...e.future].slice(0, 40),
      savedAt: Date.now(),
    });
    persist(solutionId, previous);
    emit();
  }, [solutionId]);

  const redo = useCallback(() => {
    const e = entryOf(solutionId);
    const next = e.future[0];
    if (!next) return;
    store.set(solutionId, {
      present: next,
      past: [...e.past, e.present],
      future: e.future.slice(1),
      savedAt: Date.now(),
    });
    persist(solutionId, next);
    emit();
  }, [solutionId]);

  return {
    workflow,
    canUndo: entry.past.length > 0,
    canRedo: entry.future.length > 0,
    addNode,
    moveNode,
    setLayout,
    updateNode,
    removeNode,
    groupNodes,
    ungroupNode,
    connect,
    removeEdge,
    setStatuses,
    undo,
    redo,
  };
}

/** Ordine topologico di esecuzione. */
export function pipelineOrder(workflow: EtlWorkflow): EtlNode[] {
  const indeg = new Map(workflow.nodes.map((n) => [n.id, 0]));
  for (const e of workflow.edges) indeg.set(e.toNode, (indeg.get(e.toNode) ?? 0) + 1);
  const queue = workflow.nodes.filter((n) => (indeg.get(n.id) ?? 0) === 0);
  const out: EtlNode[] = [];
  const seen = new Set<string>();
  while (queue.length) {
    const n = queue.shift()!;
    if (seen.has(n.id)) continue;
    seen.add(n.id);
    out.push(n);
    for (const e of workflow.edges.filter((x) => x.fromNode === n.id)) {
      const left = (indeg.get(e.toNode) ?? 0) - 1;
      indeg.set(e.toNode, left);
      if (left <= 0) {
        const target = workflow.nodes.find((x) => x.id === e.toNode);
        if (target) queue.push(target);
      }
    }
  }
  for (const n of workflow.nodes) if (!seen.has(n.id)) out.push(n);
  return out;
}

