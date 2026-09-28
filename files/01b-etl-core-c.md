# 01b-etl-core-c.md

File in questo blocco:

- `src/etl-core/rules/relations.ts`
- `src/etl-core/rules/state.ts`
- `src/etl-core/schema/schema.ts`

---

### `src/etl-core/rules/relations.ts`

116 righe

```ts
/**
 * Capienza, cicli, compatibilità e motivi di rifiuto. Porting letterale
 * delle righe 1543-1550 e 1610-1936 di
 * docs/prototype/isa-fusion-prototype.html.
 */
import { MERGE_OPS } from "../catalog/operations";
import { cardById, inputsOf } from "../model/graph";
import type { Card, Graph } from "../model/types";

/** Prototipo, riga 1612: 1 più il numero di componenti in MERGE_OPS. */
export function boxCapacity(card: Card): number {
  return 1 + card.components.filter((c) => (MERGE_OPS as readonly string[]).includes(c)).length;
}

/**
 * Prototipo, righe 1887-1899: `fromUid` raggiunge `toUid` seguendo i
 * collegamenti in avanti, a qualunque distanza (DFS iterativa).
 */
export function reaches(graph: Graph, fromId: string, toId: string): boolean {
  const seen = new Set<string>();
  const stack = [fromId];
  while (stack.length > 0) {
    const cur = stack.pop();
    if (cur === undefined) break;
    if (cur === toId) return true;
    if (seen.has(cur)) continue;
    seen.add(cur);
    for (const l of graph.links) {
      if (l.from === cur) stack.push(l.to);
    }
  }
  return false;
}

/**
 * Prototipo, righe 1901-1915: perché collegare `dsId` (un dataset) a
 * `boxId` (una lavorazione) sarebbe rifiutato, oppure `null` se è valido.
 * I testi sono quelli del prototipo (con gli accenti ripristinati).
 *
 * Nota (vedi src/etl-core/NOTE_DIVERGENZE.md): il messaggio di capienza
 * superata dice sempre "due tabelle", anche quando `cap` è maggiore di 2
 * (un box con più di un'operazione di MERGE_OPS) — replicato fedelmente.
 */
export function linkRefusal(graph: Graph, dsId: string, boxId: string): string | null {
  const ds = cardById(graph, dsId);
  const box = cardById(graph, boxId);
  if (!ds || !box) return null;
  if (ds.capacity !== undefined && ds.capacity > 1 && (ds.filled ?? 0) < ds.capacity) {
    return "Questo output non è ancora completo: al join manca una tabella";
  }
  if (reaches(graph, boxId, dsId)) {
    return "Un box non può agganciarsi a ciò che produce: sarebbe un ciclo infinito";
  }
  const cur = inputsOf(graph, boxId);
  if (cur.some((l) => l.from === dsId)) {
    return "Questa tabella è già collegata al box";
  }
  const cap = boxCapacity(box);
  if (cur.length >= cap) {
    return cap > 1
      ? "Il join ha già le sue due tabelle"
      : "Il box accetta una sola tabella in ingresso";
  }
  return null;
}

export type Relation = "merge" | "link" | "link-reverse" | "displace" | null;

export interface RelationResult {
  readonly relation: Relation;
  /** Motivo dello spostamento, presente solo quando `relation === 'displace'`. */
  readonly displaceReason?: string;
}

/**
 * Prototipo, righe 1917-1936: cosa succede accostando `aId` a `bId`
 * (l'ordine conta: chi viene trascinato su chi).
 */
export function relation(graph: Graph, aId: string, bId: string): RelationResult {
  const a = cardById(graph, aId);
  const b = cardById(graph, bId);
  if (!a || !b) return { relation: null };
  if (a.kind === "op" && b.kind === "op") return { relation: "merge" };
  if (a.kind === "dataset" && b.kind === "op") {
    const why = linkRefusal(graph, aId, bId);
    if (why) return { relation: "displace", displaceReason: why };
    return { relation: "link" };
  }
  if (a.kind === "op" && b.kind === "dataset") {
    const why = linkRefusal(graph, bId, aId);
    if (why) return { relation: "displace", displaceReason: why };
    return { relation: "link-reverse" };
  }
  // a.kind === 'dataset' && b.kind === 'dataset'
  return {
    relation: "displace",
    displaceReason: "Due dataset non si fondono: serve una lavorazione, ad esempio un Join",
  };
}

/**
 * Prototipo, righe 1543-1550: due nodi "compatibili" (che si fonderebbero
 * o collegherebbero) non si respingono geometricamente — quell'aspetto è
 * fuori dall'ambito di questa fase, ma la regola di compatibilità è
 * domain-pure e viene portata qui.
 */
export function compatiblePair(graph: Graph, aId: string, bId: string): boolean {
  const a = cardById(graph, aId);
  const b = cardById(graph, bId);
  if (!a || !b) return false;
  if (a.kind === "op" && b.kind === "op") return true;
  if (a.kind === "dataset" && b.kind === "op") return linkRefusal(graph, aId, bId) === null;
  if (a.kind === "op" && b.kind === "dataset") return linkRefusal(graph, bId, aId) === null;
  return false;
}
```

### `src/etl-core/rules/state.ts`

96 righe

```ts
/**
 * Completezza dei parametri e stato dei nodi. Porting letterale delle
 * righe 1426-1459 di docs/prototype/isa-fusion-prototype.html.
 */
import { META } from "../catalog/operations";
import {
  PARAM_DEFS,
  ensureKeys,
  ensureMulti,
  fieldFilled,
  keyComplete,
  MULTI_DEFS,
  NO_VALUE_OPS,
} from "../catalog/params";
import { boxCapacity } from "./relations";
import { inputsOf, cardById } from "../model/graph";
import type {
  ComponentId,
  FilterCondition,
  Graph,
  JoinParams,
  MultiRow,
  Params,
} from "../model/types";

/**
 * Prototipo, righe 1426-1446: cosa manca perché il componente `type`, con
 * i parametri `par`, sia considerato completo. `true` = incompleto.
 */
export function stepMissing(type: ComponentId, par: Params | undefined): boolean {
  if (!par) return true;
  if (type === "filter") {
    const conditions = par["conditions"];
    if (!Array.isArray(conditions) || conditions.length === 0) return true;
    return (conditions as FilterCondition[]).some((c) => {
      if (!c.column) return true;
      if (NO_VALUE_OPS.includes(c.op)) return false;
      const hasText = !!(c.text && c.text.trim());
      const hasValues = !!(c.values && c.values.length > 0);
      return !hasText && !hasValues;
    });
  }
  if (type === "join") {
    const keys = ensureKeys(par as unknown as JoinParams);
    if (keys.length === 0) return true;
    return keys.some((k) => !keyComplete(k));
  }
  if (type !== "dataset" && MULTI_DEFS[type]) {
    const md = MULTI_DEFS[type];
    if (!md) return false;
    const migrated = ensureMulti(type, par);
    return md.lists.some((list) => {
      const rows = migrated[list.key];
      if (!Array.isArray(rows) || rows.length === 0) return true;
      return (rows as MultiRow[]).some((row) =>
        list.fields.some((f) => (f.req || f.type === "column") && !fieldFilled(f, row[f.k])),
      );
    });
  }
  if (type === "exportOp") {
    const dest = par["dest"];
    return !(typeof dest === "string" && dest.trim().length > 0);
  }
  const defs = PARAM_DEFS[type];
  if (!Array.isArray(defs)) return false;
  return defs.some((f) => {
    if (!f.req && f.type !== "column") return false;
    const v = par[f.k];
    return !(typeof v === "string" && v.trim().length > 0);
  });
}

/**
 * Prototipo, righe 1447-1459: messaggio che descrive perché un nodo non è
 * pronto, oppure `null` se è completo.
 */
export function nodeState(graph: Graph, id: string): string | null {
  const d = cardById(graph, id);
  if (!d) return null;
  if (d.kind === "dataset") {
    if (d.isOutput) {
      return d.capacity !== undefined && d.capacity > 1 && (d.filled ?? 0) < d.capacity
        ? "In attesa delle tabelle mancanti"
        : null;
    }
    const p0 = d.params[0];
    const path = p0 ? p0["path"] : undefined;
    return typeof path === "string" && path.trim().length > 0 ? null : "Origine da configurare";
  }
  if (inputsOf(graph, id).length < boxCapacity(d)) return "Mancano tabelle in ingresso";
  const badIndex = d.components.findIndex((c, i) => stepMissing(c, d.params[i]));
  if (badIndex < 0) return null;
  const badType = d.components[badIndex] as ComponentId;
  return `Da configurare: ${META[badType].label}`;
}
```

### `src/etl-core/schema/schema.ts`

37 righe

```ts
/**
 * Propagazione delle colonne. Porting letterale delle righe 2415-2430 di
 * docs/prototype/isa-fusion-prototype.html.
 */
import { cardById, inputsOf } from "../model/graph";
import type { ColumnDef, DatasetParams, Graph } from "../model/types";

/**
 * Le colonne di una sorgente vengono dai suoi parametri (`params[0].columns`,
 * popolate da `parseCSV` al caricamento); quelle di un output, attraverso il
 * box che lo produce; quelle di una lavorazione sono l'unione delle colonne
 * dei suoi ingressi (nell'ordine in cui compaiono, senza duplicati per nome).
 * `null` se non determinabile. Guardia di profondità come nel prototipo
 * (righe 2415-2418), a protezione da un ciclo non ancora rilevato altrove.
 */
export function schemaOf(graph: Graph, id: string, depth = 0): ColumnDef[] | null {
  if (depth > 24) return null;
  const d = cardById(graph, id);
  if (!d) return null;
  if (d.kind === "dataset") {
    if (!d.isOutput) {
      const p0 = d.params[0] as DatasetParams | undefined;
      return p0?.columns ?? null;
    }
    const prod = graph.links.find((l) => l.to === id);
    return prod ? schemaOf(graph, prod.from, depth + 1) : null;
  }
  const out: ColumnDef[] = [];
  for (const l of inputsOf(graph, id)) {
    const cols = schemaOf(graph, l.from, depth + 1) ?? [];
    for (const c of cols) {
      if (!out.some((o) => o.name === c.name)) out.push(c);
    }
  }
  return out.length ? out : null;
}
```

