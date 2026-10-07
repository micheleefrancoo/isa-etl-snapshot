# 01b-etl-core-d.md

File in questo blocco:

- `src/etl-core/rules/mutations.ts`
- `src/etl-core/rules/relations.ts`
- `src/etl-core/rules/state.ts`
- `src/etl-core/schema/schema.ts`

---

### `src/etl-core/rules/mutations.ts`

517 righe

```ts
/**
 * Operazioni sul grafo. Porting delle righe 1730-1957, 2137-2248,
 * 4412-4453, 4480-4618 di docs/prototype/isa-fusion-prototype.html —
 * SENZA animazioni, DOM o geometria di posizionamento (Fase 2): ogni
 * funzione qui riceve un `Graph` e restituisce un nuovo `Graph`, mai
 * mutando l'input.
 *
 * Nota sui contatori di denominazione: il prototipo usa contatori globali
 * mutabili (`outCounter`, `comboCounter`) incrementati una volta per
 * sempre. Un dominio a funzioni pure non ha un posto per questo stato: il
 * numero mostrato ("Output N", "Combined Box N") viene invece derivato
 * dal grafo corrente (quanti output/box combinati esistono già). Nell'uso
 * normale (senza eliminare e poi ricreare più volte gli stessi nodi) il
 * risultato è identico al prototipo; è una conseguenza necessaria del
 * vincolo "funzione pura", non un comportamento diverso deliberato.
 */
import { META } from "../catalog/operations";
import { defaultParams } from "../catalog/params";
import { boxCapacity, linkRefusal } from "./relations";
import {
  addLink,
  cardById,
  filterLinks,
  inputsOf,
  outputOf,
  patchCard,
  removeCard,
  removeCards,
  setCard,
} from "../model/graph";
import type {
  Card,
  ComponentId,
  Graph,
  IdGenerator,
  Link,
  OperationResult,
  Params,
  PositionFn,
} from "../model/types";

/** Prototipo, riga 1745 (uso in `spawnOutput`): scostamento semplice, senza evitare sovrapposizioni. */
export const defaultPositionFn: PositionFn = (graph, anchorId) => {
  const anchor = cardById(graph, anchorId);
  if (!anchor) return { x: 0, y: 0 };
  return { x: anchor.x + 200, y: anchor.y };
};

function countOutputs(graph: Graph): number {
  return Object.values(graph.cards).filter((c) => c.isOutput).length;
}

function countCombinedBoxes(graph: Graph): number {
  return Object.values(graph.cards).filter((c) => c.kind === "op" && c.components.length > 1)
    .length;
}

function ensureCardParams(card: Card): Params[] {
  const next = card.params.slice();
  while (next.length < card.components.length) {
    next.push(defaultParams(card.components[next.length] as ComponentId));
  }
  return next;
}

/**
 * Prototipo, righe 1616-1635 (`renderOutputIcon`): qui solo la parte di
 * dominio (capacità/riempimento); il disegno delle "fette" è Fase 2.
 */
export function isPartialOutput(card: Card): boolean {
  return card.capacity !== undefined && card.capacity > 1 && (card.filled ?? 0) < card.capacity;
}

/**
 * Prototipo, righe 1875-1885 (`connect`) UNIFICATA con `linkRefusal`
 * (righe 1901-1915): qui `connect` è l'unico punto d'ingresso e rifiuta
 * cicli, duplicati, capienza superata e output non ancora completo — nel
 * prototipo il controllo dei cicli viveva solo in `linkRefusal`, invocata
 * dall'interazione UI prima di chiamare `connect`; qui le due
 * responsabilità sono unite perché la funzione pura è l'unica autorità
 * sulla validità di un collegamento.
 *
 * Garantisce l'output (Fase 1.1): dopo aver aggiunto il collegamento,
 * chiama `refreshOutput` sul box di destinazione — crea l'output se
 * manca e ne aggiorna `capacity`/`filled`. Invariante risultante: dopo
 * `connect`, ogni lavorazione con almeno un ingresso ha un output.
 */
export function connect(
  graph: Graph,
  sourceId: string,
  targetId: string,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): OperationResult {
  const box = cardById(graph, targetId);
  if (!box) return { ok: false, reason: "Il box di destinazione non esiste" };
  const reason = linkRefusal(graph, sourceId, targetId);
  if (reason) return { ok: false, reason };
  const linked = addLink(graph, { from: sourceId, to: targetId });
  return { ok: true, graph: refreshOutput(linked, targetId, nextId, positionFn) };
}

/**
 * Prototipo, righe 1730-1772 (`spawnOutput`), senza l'animazione di
 * espulsione. Se il box ha già un output lo restituisce senza crearne un
 * altro. Restituisce `null` se il box non esiste.
 */
export function spawnOutput(
  graph: Graph,
  boxId: string,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): { graph: Graph; outputId: string } | null {
  const existing = outputOf(graph, boxId);
  if (existing) return { graph, outputId: existing };
  const box = cardById(graph, boxId);
  if (!box) return null;
  const outputId = nextId();
  const pos = positionFn(graph, boxId);
  const outputCard: Card = {
    id: outputId,
    kind: "dataset",
    isOutput: true,
    components: ["dataset"],
    params: [defaultParams("dataset")],
    name: `Output ${countOutputs(graph) + 1}`,
    x: pos.x,
    y: pos.y,
  };
  const withCard = setCard(graph, outputCard);
  const withLink = addLink(withCard, { from: boxId, to: outputId });
  return { graph: withLink, outputId };
}

/**
 * Prototipo, righe 1659-1672 (`refreshOutput`): crea l'output se manca e
 * aggiorna `capacity`/`filled`. Non fa nulla se il box non ha ingressi
 * (prototipo, riga 1663: `if (n === 0) return;`).
 */
export function refreshOutput(
  graph: Graph,
  boxId: string,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): Graph {
  const box = cardById(graph, boxId);
  if (!box || box.kind !== "op") return graph;
  const n = inputsOf(graph, boxId).length;
  if (n === 0) return graph;
  const spawned = spawnOutput(graph, boxId, nextId, positionFn);
  if (!spawned) return graph;
  const out = cardById(spawned.graph, spawned.outputId);
  if (!out) return spawned.graph;
  const capacity = boxCapacity(box);
  const filled = Math.min(n, capacity);
  return setCard(spawned.graph, patchCard(out, { capacity, filled }));
}

/**
 * Prototipo, righe 4414-4430 (`pruneOutputs`): un output senza produttore,
 * o il cui produttore ha perso tutti i suoi ingressi, non ha ragione di
 * esistere. A cascata.
 */
export function pruneOutputs(graph: Graph): Graph {
  let current = graph;
  let changed = true;
  while (changed) {
    changed = false;
    for (const id of Object.keys(current.cards)) {
      const c = current.cards[id];
      if (!c?.isOutput) continue;
      const prod = current.links.find((l) => l.to === id);
      const alive =
        !!prod && !!current.cards[prod.from] && current.links.some((l) => l.to === prod.from);
      if (!alive) {
        current = removeCard(current, id);
        current = filterLinks(current, (l) => l.from !== id && l.to !== id);
        changed = true;
      }
    }
  }
  return current;
}

/**
 * Prototipo, righe 1638-1648 (`enforceCapacity`): se un box perde
 * capienza (es. ha perso un join), gli ingressi in eccesso vengono
 * rimossi — restano i collegamenti più vecchi.
 */
export function enforceCapacity(graph: Graph, boxId: string): Graph {
  const box = cardById(graph, boxId);
  if (!box || box.kind !== "op") return graph;
  const cap = boxCapacity(box);
  const ins = inputsOf(graph, boxId);
  let next = graph;
  if (ins.length > cap) {
    const surplus = new Set(ins.slice(cap));
    next = filterLinks(next, (l) => !surplus.has(l));
  }
  return pruneOutputs(next);
}

/**
 * Prototipo, righe 4432-4453 (`nodesRemovedBy`): quali nodi sparirebbero
 * davvero eliminando `ids` — il nodo stesso più gli output che restano
 * senza produttore vivo, a cascata.
 */
export function nodesRemovedBy(graph: Graph, uidOrIds: string | readonly string[]): Set<string> {
  const ids = Array.isArray(uidOrIds) ? uidOrIds : [uidOrIds];
  let cards: Record<string, Card> = { ...graph.cards };
  for (const id of ids) delete cards[id];
  let links = graph.links.filter((l) => !ids.includes(l.from) && !ids.includes(l.to));
  let changed = true;
  while (changed) {
    changed = false;
    for (const id of Object.keys(cards)) {
      if (!cards[id]?.isOutput) continue;
      const prod = links.find((l) => l.to === id);
      const alive = !!prod && !!cards[prod.from] && links.some((l) => l.to === prod.from);
      if (!alive) {
        const rest = { ...cards };
        delete rest[id];
        cards = rest;
        links = links.filter((l) => l.from !== id && l.to !== id);
        changed = true;
      }
    }
  }
  const removed = new Set<string>(ids);
  for (const id of Object.keys(graph.cards)) {
    if (!(id in cards)) removed.add(id);
  }
  return removed;
}

/**
 * Prototipo, riga 4407 (dentro `renderAll`, chiamato dopo ogni
 * eliminazione): `refreshOutput` su ogni lavorazione. Ricrea l'output di
 * un box che ha ancora ingressi ma lo ha perso (es. l'output eliminato
 * direttamente, o il collegamento box → output rimosso).
 */
function refreshAllOutputs(graph: Graph, nextId: IdGenerator, positionFn: PositionFn): Graph {
  let next = graph;
  for (const id of Object.keys(graph.cards)) {
    next = refreshOutput(next, id, nextId, positionFn);
  }
  return next;
}

/**
 * Prototipo, righe 4480-4506 (`commitDelete`), solo la parte di dominio
 * (senza l'animazione di rientro degli output): rimuove i nodi
 * effettivamente cancellati da `nodesRemovedBy` e i loro collegamenti.
 *
 * Garantisce l'output (Fase 1.1): come `renderAll` nel prototipo (riga
 * 4407), ogni lavorazione rimasta con almeno un ingresso riottiene un
 * output se lo ha perso.
 */
export function deleteNodes(
  graph: Graph,
  uidOrIds: string | readonly string[],
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): Graph {
  const removed = nodesRemovedBy(graph, uidOrIds);
  const withoutCards = removeCards(graph, removed);
  const next = filterLinks(withoutCards, (l) => !removed.has(l.from) && !removed.has(l.to));
  return refreshAllOutputs(next, nextId, positionFn);
}

/**
 * Prototipo, righe 4565-4570 (`deleteLink`): rimuove un collegamento e
 * poi gli output che ne dipendevano. Garantisce l'output come
 * `deleteNodes` (Fase 1.1).
 */
export function deleteLink(
  graph: Graph,
  link: Link,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): Graph {
  const next = filterLinks(graph, (l) => !(l.from === link.from && l.to === link.to));
  return refreshAllOutputs(pruneOutputs(next), nextId, positionFn);
}

/**
 * Prototipo, righe 1774-1840 (`performMerge`), senza animazioni/DOM.
 * Fonde `draggedId` dentro `targetId` (entrambi lavorazioni): unisce
 * `components`/`params`, rimappa i collegamenti, rimuove auto-anelli e
 * duplicati, e aggiorna l'eventuale output del box risultante.
 */
export function mergeBoxes(
  graph: Graph,
  draggedId: string,
  targetId: string,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): OperationResult {
  const dragged = cardById(graph, draggedId);
  const target = cardById(graph, targetId);
  if (!dragged || !target || dragged.kind !== "op" || target.kind !== "op") {
    return { ok: false, reason: "La fusione richiede due lavorazioni" };
  }
  const draggedParams = ensureCardParams(dragged);
  const targetParams = ensureCardParams(target);
  const merged = [...target.components, ...dragged.components];
  const mergedParams = [...targetParams, ...draggedParams];

  const wasCombined = target.components.length > 1;
  const combinedCount = countCombinedBoxes(graph) + 1;
  const name = wasCombined
    ? target.name
    : combinedCount === 1
      ? "Combined Box"
      : `Combined Box ${combinedCount}`;

  let links: Link[] = graph.links.map((l) => {
    let { from, to } = l;
    if (to === draggedId) to = targetId;
    if (from === draggedId) from = targetId;
    return { from, to };
  });
  links = links.filter((l) => l.from !== l.to);
  const produced = new Set(links.filter((l) => l.from === targetId).map((l) => l.to));
  links = links.filter((l) => !(l.to === targetId && produced.has(l.from)));
  links = links.filter(
    (l, i, arr) => arr.findIndex((o) => o.from === l.from && o.to === l.to) === i,
  );

  const mergedCard: Card = { ...target, components: merged, params: mergedParams, name };
  let next: Graph = { cards: { ...graph.cards }, links };
  next = setCard(next, mergedCard);
  next = removeCard(next, draggedId);

  if (inputsOf(next, targetId).length > 0) {
    next = refreshOutput(next, targetId, nextId, positionFn);
  }
  return { ok: true, graph: next };
}

// --- Inserimento su un collegamento esistente (prototipo, righe 1846-1873) ---

/** Prototipo, righe 1846-1852: solo su un collegamento dataset → lavorazione, con una lavorazione priva di collegamenti. */
export function insertable(graph: Graph, link: Link, nodeId: string): boolean {
  if (link.from === nodeId || link.to === nodeId) return false;
  const a = cardById(graph, link.from);
  const b = cardById(graph, link.to);
  if (!a || !b || a.kind !== "dataset" || b.kind !== "op") return false;
  return !graph.links.some((l) => l.from === nodeId || l.to === nodeId);
}

/**
 * Prototipo, righe 1853-1873 (`insertOnLink`), senza posizionamento
 * geometrico (Fase 2) né lo scioglimento hint testuale (UI). Restituisce
 * `null` se l'inserimento non è consentito (vedi `insertable`).
 */
export function insertOnLink(
  graph: Graph,
  link: Link,
  nodeId: string,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): Graph | null {
  if (!insertable(graph, link, nodeId)) return null;
  let next = filterLinks(graph, (l) => !(l.from === link.from && l.to === link.to));
  next = addLink(next, { from: link.from, to: nodeId });
  const spawned = spawnOutput(next, nodeId, nextId, positionFn);
  if (spawned) {
    next = addLink(spawned.graph, { from: spawned.outputId, to: link.to });
  }
  next = refreshOutput(next, nodeId, nextId, positionFn);
  next = refreshOutput(next, link.to, nextId, positionFn);
  return next;
}

// --- Passaggi di un box combinato (prototipo, righe 2137-2248) --------------

/**
 * Prototipo, righe 2137-2203 (`detachStep`), senza animazioni/DOM/
 * posizionamento geometrico: sgancia il passaggio a `index` dal box
 * `boxId` e lo trasforma in una card autonoma. Restituisce `null` se il
 * box non è combinato (meno di 2 componenti).
 */
export function detachStep(
  graph: Graph,
  boxId: string,
  index: number,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): { graph: Graph; detachedId: string } | null {
  const box = cardById(graph, boxId);
  if (!box || box.kind !== "op" || box.components.length < 2) return null;
  const compId = box.components[index];
  if (compId === undefined) return null;
  const params = ensureCardParams(box);
  const detachedParams = params[index] ?? defaultParams(compId);
  const nextComponents = box.components.filter((_, i) => i !== index);
  const nextParams = params.filter((_, i) => i !== index);
  const stillCombined = nextComponents.length > 1;

  const updatedBox: Card = {
    ...box,
    components: nextComponents,
    params: nextParams,
    name: stillCombined ? box.name : metaLabelOf(nextComponents[0] as ComponentId),
  };

  const detachedId = nextId();
  const pos = positionFn(graph, boxId);
  const detachedCard: Card = {
    id: detachedId,
    kind: "op",
    components: [compId],
    params: [detachedParams],
    name: metaLabelOf(compId),
    x: pos.x,
    y: pos.y,
  };

  let next = setCard(graph, updatedBox);
  next = setCard(next, detachedCard);
  next = enforceCapacity(next, boxId);
  next = refreshOutput(next, boxId, nextId, positionFn);
  return { graph: next, detachedId };
}

function metaLabelOf(component: ComponentId): string {
  return META[component].label;
}

/**
 * Prototipo, righe 2205-2247 (`deleteStep`), senza animazioni/DOM.
 * Restituisce il grafo inalterato se il box non è combinato.
 */
export function deleteStep(
  graph: Graph,
  boxId: string,
  index: number,
  nextId: IdGenerator,
  positionFn: PositionFn = defaultPositionFn,
): Graph {
  const box = cardById(graph, boxId);
  if (!box || box.kind !== "op" || box.components.length < 2) return graph;
  const nextComponents = box.components.filter((_, i) => i !== index);
  const params = ensureCardParams(box);
  const nextParams = params.filter((_, i) => i !== index);
  const stillCombined = nextComponents.length > 1;
  const updatedBox: Card = {
    ...box,
    components: nextComponents,
    params: nextParams,
    name: stillCombined ? box.name : metaLabelOf(nextComponents[0] as ComponentId),
  };
  let next = setCard(graph, updatedBox);
  next = enforceCapacity(next, boxId);
  next = refreshOutput(next, boxId, nextId, positionFn);
  return next;
}

/**
 * Prototipo, righe 2330-2338 (dentro il gestore di riordino): sposta il
 * passaggio a `fromIndex` in `toIndex`, spostando `components` e
 * `params` insieme.
 */
export function reorderSteps(
  graph: Graph,
  boxId: string,
  fromIndex: number,
  toIndex: number,
): Graph {
  const box = cardById(graph, boxId);
  if (!box) return graph;
  const params = ensureCardParams(box);
  const components = box.components.slice();
  const [movedComponent] = components.splice(fromIndex, 1);
  if (movedComponent === undefined) return graph;
  components.splice(toIndex, 0, movedComponent);
  const [movedParam] = params.splice(fromIndex, 1);
  params.splice(toIndex, 0, movedParam ?? defaultParams(movedComponent));
  return setCard(graph, { ...box, components, params });
}

// --- Duplicazione (prototipo, righe 4594-4618) ------------------------------

/**
 * Prototipo, righe 4594-4618 (`duplicateSelection`): duplica i nodi
 * indicati senza collegamenti, escludendo gli output (che non hanno
 * senso senza il box che li produce). Restituisce gli id creati, nello
 * stesso ordine di `ids`.
 */
export function duplicateNodes(
  graph: Graph,
  ids: readonly string[],
  nextId: IdGenerator,
  offset: { x: number; y: number } = { x: 52, y: 52 },
): { graph: Graph; createdIds: string[] } {
  let next = graph;
  const createdIds: string[] = [];
  for (const id of ids) {
    const src = cardById(graph, id);
    if (!src || src.isOutput) continue;
    const newId = nextId();
    const { slot, ...srcWithoutSlot } = src;
    void slot;
    const copy: Card = {
      ...srcWithoutSlot,
      id: newId,
      name: `${src.name} copia`,
      x: src.x + offset.x,
      y: src.y + offset.y,
    };
    next = setCard(next, copy);
    createdIds.push(newId);
  }
  return { graph: next, createdIds };
}
```

### `src/etl-core/rules/relations.ts`

118 righe

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
 *
 * Correzione intenzionale rispetto al prototipo (Fase 1.1, vedi
 * src/etl-core/NOTE_DIVERGENZE.md): i due messaggi di capienza sono
 * generali — "Il box ha già tutte le sue N tabelle" (N = capienza reale,
 * non sempre "due") e "Questo output non è ancora completo: manca ancora
 * una tabella in ingresso" (non specifico al join) — invece dei testi
 * cablati del prototipo.
 */
export function linkRefusal(graph: Graph, dsId: string, boxId: string): string | null {
  const ds = cardById(graph, dsId);
  const box = cardById(graph, boxId);
  if (!ds || !box) return null;
  if (ds.capacity !== undefined && ds.capacity > 1 && (ds.filled ?? 0) < ds.capacity) {
    return "Questo output non è ancora completo: manca ancora una tabella in ingresso";
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
      ? `Il box ha già tutte le sue ${cap} tabelle`
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

121 righe

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
  columnsOf,
  fieldFilled,
  keyComplete,
  MULTI_DEFS,
  MULTI_OPS,
  NO_VALUE_OPS,
} from "../catalog/params";
import { boxCapacity } from "./relations";
import { inputsOf, cardById } from "../model/graph";
import { columnsOutsideSchema, schemaOf } from "../schema/schema";
import type {
  ColumnDef,
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
 *
 * Correzione intenzionale rispetto al prototipo per il filtro (Fase 1.1,
 * vedi src/etl-core/NOTE_DIVERGENZE.md — "una sola fonte di verità per i
 * valori"): un operatore in `MULTI_OPS` è completo solo se `values` non è
 * vuoto (non basta più un `text` residuo); un operatore in `NO_VALUE_OPS`
 * è sempre completo (a colonna impostata); ogni altro operatore richiede
 * `text` non vuoto.
 *
 * Con `schema` (le colonne in ingresso, se note) una riga di un'operazione a voci multiple che
 * elenca una colonna assente dallo schema è incompleta (Fase 6b.2, Passo 0): la colonna non c'è
 * più nei dati. Con lo schema vuoto o sconosciuto non cambia nulla.
 */
export function stepMissing(
  type: ComponentId,
  par: Params | undefined,
  schema?: readonly ColumnDef[] | null,
): boolean {
  if (!par) return true;
  if (type === "filter") {
    const conditions = par["conditions"];
    if (!Array.isArray(conditions) || conditions.length === 0) return true;
    return (conditions as FilterCondition[]).some((c) => {
      if (!c.column) return true;
      if (NO_VALUE_OPS.includes(c.op)) return false;
      if (MULTI_OPS.includes(c.op)) return !(c.values && c.values.length > 0);
      return !(c.text && c.text.trim());
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
      return (rows as MultiRow[]).some(
        (row) =>
          list.fields.some(
            (f) =>
              (f.req || f.type === "column" || f.type === "columns") && !fieldFilled(f, row[f.k]),
          ) ||
          (list.fields.some((f) => f.type === "columns") &&
            columnsOutsideSchema(columnsOf(row), schema).length > 0),
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
  const schema = schemaOf(graph, id);
  const badIndex = d.components.findIndex((c, i) => stepMissing(c, d.params[i], schema));
  if (badIndex < 0) return null;
  const badType = d.components[badIndex] as ComponentId;
  return `Da configurare: ${META[badType].label}`;
}
```

### `src/etl-core/schema/schema.ts`

54 righe

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

/**
 * Le colonne di una riga che non sono nello schema in ingresso (nell'ordine in cui sono elencate,
 * senza ripetizioni). Con lo schema vuoto o sconosciuto non si sa cosa manchi: nessuna.
 */
export function columnsOutsideSchema(
  columns: readonly string[],
  schema: readonly ColumnDef[] | null | undefined,
): string[] {
  if (!schema || schema.length === 0) return [];
  const known = new Set(schema.map((c) => c.name));
  const out: string[] = [];
  for (const name of columns) {
    if (!known.has(name) && !out.includes(name)) out.push(name);
  }
  return out;
}
```

