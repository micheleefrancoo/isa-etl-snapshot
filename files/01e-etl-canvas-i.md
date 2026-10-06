# 01e-etl-canvas-i.md

File in questo blocco:

- `src/etl-canvas/interaction.ts`
- `src/etl-canvas/loop.ts`
- `src/etl-canvas/model.ts`
- `src/etl-canvas/motion.tsx`
- `src/etl-canvas/panels/ControlBar.tsx`
- `src/etl-canvas/panels/Dock.tsx`
- `src/etl-canvas/panels/EtlWorkspace.tsx`
- `src/etl-canvas/panels/InspectorShell.tsx`

---

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

### `src/etl-canvas/model.ts`

112 righe

```ts
/**
 * Dal grafo di etl-core a ciò che il canvas disegna: classi e icone di ogni
 * nodo, fette di un output parziale. Funzioni pure.
 */
import { sectionOf } from "../etl-core";
import type { Card, ComponentId } from "../etl-core";

/** Classe di conteggio per la disposizione delle icone (prototipo `countClass`, righe 982-985). */
export function countClass(n: number): string {
  if (n <= 1) return "count-1";
  if (n === 2) return "count-2";
  if (n <= 3) return "count-3";
  if (n <= 6) return "count-6";
  return "count-many";
}

/** Un output è "parziale" se attende altre tabelle (riga 1620). */
export function isPartial(card: Card): boolean {
  return (
    card.kind === "dataset" &&
    card.isOutput === true &&
    card.capacity !== undefined &&
    card.capacity > 1 &&
    (card.filled ?? 0) < card.capacity
  );
}

export interface Slice {
  readonly full: boolean;
}

/** Le fette di un output parziale, riempite da sinistra (righe 1622-1627). */
export function slicesOf(card: Card): Slice[] {
  const n = card.capacity ?? 0;
  const f = card.filled ?? 0;
  return Array.from({ length: n }, (_, i) => ({ full: i < f }));
}

/** Famiglia di operazioni (colore del nodo): una per sezione della cassetta, tranne i dataset. */
export type OpFamily = "filter" | "transform" | "merge" | "output";

const FAMILY_OF_SECTION: Readonly<Record<string, OpFamily>> = {
  rows: "filter",
  xform: "transform",
  merge: "merge",
  out: "output",
};

/** La famiglia di un nodo lavorazione (quella della prima operazione); i dataset non ne hanno. */
export function familyOf(card: Card): OpFamily | undefined {
  if (card.kind !== "op") return undefined;
  const first = card.components[0];
  const section = first === undefined ? null : sectionOf(first);
  return section ? FAMILY_OF_SECTION[section.id] : undefined;
}

export interface NodeView {
  readonly id: string;
  readonly card: Card;
  /** Classi del contenitore del nodo. */
  readonly className: string;
  readonly iconClass: string;
  readonly partial: boolean;
  readonly slices: readonly Slice[];
  readonly family: OpFamily | undefined;
  readonly icons: readonly ComponentId[];
  readonly warn: string | null;
  readonly selected: boolean;
}

/** Classi del nodo (prototipo `createCardEl`, righe 1003-1004, più `partial`, `warn`, `selected`). */
/** Stato dei gesti che cambia l'aspetto di un nodo (classi `ec-dragging`, `ec-drop-*`, `ec-doomed`). */
export interface NodeGestureState {
  readonly dragging?: boolean;
  readonly drop?: "merge" | "link" | "link-reverse" | "displace" | "reject" | null;
  readonly doomed?: boolean;
}

export function nodeView(
  card: Card,
  warn: string | null,
  selected: boolean,
  gesture: NodeGestureState = {},
): NodeView {
  const combined = card.kind === "op" && card.components.length > 1;
  const partial = isPartial(card);
  const classes = ["ec-card"];
  if (card.kind === "dataset") classes.push("ec-dataset");
  if (card.kind === "dataset" && card.isOutput) classes.push("ec-output");
  if (combined) classes.push("ec-combined");
  if (partial) classes.push("ec-partial");
  if (warn) classes.push("ec-warn");
  if (selected) classes.push("ec-selected");
  if (gesture.dragging) classes.push("ec-dragging");
  if (gesture.drop) classes.push(`ec-drop-${gesture.drop}`);
  if (gesture.doomed) classes.push("ec-doomed");
  const icons: ComponentId[] =
    card.kind === "dataset" && card.isOutput ? ["dataset"] : [...card.components];
  return {
    id: card.id,
    card,
    className: classes.join(" "),
    iconClass: partial ? "ec-icon-wrap ec-split" : `ec-icon-wrap ec-${countClass(icons.length)}`,
    partial,
    slices: partial ? slicesOf(card) : [],
    family: familyOf(card),
    icons,
    warn,
    selected,
  };
}
```

### `src/etl-canvas/motion.tsx`

10 righe

```tsx
import { createContext, useContext } from "react";
import type { MotionEngine } from "./engine";

/** Il motore delle animazioni, raggiungibile da cavi e nodi per registrare i propri elementi. */
export const MotionContext = createContext<MotionEngine | null>(null);

export function useMotion(): MotionEngine | null {
  return useContext(MotionContext);
}
```

### `src/etl-canvas/panels/ControlBar.tsx`

97 righe

```tsx
/**
 * La barra dei controlli: una riga fissa sopra l'area del canvas, dentro lo
 * spazio di lavoro (non in sovrimpressione ai nodi). Interruttore
 * Libero/Organizzato (`setMode`), Riordina (`autoLayout`), Annulla e
 * Ripristina (disabilitati senza cronologia), Svuota (`clearAll`, sempre con
 * conferma). Nel prototipo i pulsanti «Funzionalità» e «Reimposta» non sono
 * portati: tutte le funzionalità sono sempre attive.
 */
import type { Size } from "../../etl-layout";
import type { EtlStore } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import type { InteractionController } from "../interaction";
import { RedoIcon, ReorderIcon, TrashIcon, UndoIcon } from "./ui-icons";

export function ControlBar(props: {
  store: EtlStore;
  controller: InteractionController;
  /** Area del canvas: serve a «Riordina»; `null` finché non è misurata. */
  area: Size | null;
}) {
  const { store, controller, area } = props;
  // ogni cambio di stato (anche annulla e ripristina) ridisegna la barra
  const state = useEtlState((s) => s, store);
  const hasNodes = Object.keys(state.graph.cards).length > 0;
  const canUndo = store.canUndo();
  const canRedo = store.canRedo();

  return (
    <div className="ec-bar" role="toolbar" aria-label="Controlli del canvas" data-testid="ec-bar">
      <div className="ec-seg" role="group" aria-label="Disposizione dei nodi">
        <button
          type="button"
          className={"ec-seg-btn" + (state.mode === "free" ? " ec-on" : "")}
          aria-pressed={state.mode === "free"}
          title="Libero: i nodi stanno dove li lasci"
          onClick={() => store.dispatch({ type: "setMode", payload: { mode: "free" } })}
        >
          Libero
        </button>
        <button
          type="button"
          className={"ec-seg-btn" + (state.mode === "grid" ? " ec-on" : "")}
          aria-pressed={state.mode === "grid"}
          title="Organizzato: i nodi si allineano alla griglia"
          onClick={() => store.dispatch({ type: "setMode", payload: { mode: "grid" } })}
        >
          Organizzato
        </button>
      </div>
      <button
        type="button"
        className="ec-bar-btn"
        title="Riordina i nodi"
        disabled={!hasNodes || !area}
        onClick={() => {
          if (area) store.dispatch({ type: "autoLayout", payload: { viewport: area } });
        }}
      >
        <ReorderIcon />
        <span>Riordina</span>
      </button>
      <span className="ec-bar-sep" aria-hidden="true" />
      <button
        type="button"
        className="ec-bar-btn ec-bar-icon"
        aria-label="Annulla"
        title="Annulla (Cmd/Ctrl+Z)"
        disabled={!canUndo}
        onClick={() => store.undo()}
      >
        <UndoIcon />
      </button>
      <button
        type="button"
        className="ec-bar-btn ec-bar-icon"
        aria-label="Ripristina"
        title="Ripristina (Cmd/Ctrl+Maiusc+Z)"
        disabled={!canRedo}
        onClick={() => store.redo()}
      >
        <RedoIcon />
      </button>
      <span className="ec-bar-sep" aria-hidden="true" />
      <button
        type="button"
        className="ec-bar-btn ec-bar-icon ec-bar-danger"
        aria-label="Svuota il canvas"
        title="Svuota il canvas: elimina tutti i nodi e i collegamenti"
        disabled={!hasNodes}
        onClick={() => controller.requestClearAll()}
      >
        <TrashIcon />
      </button>
    </div>
  );
}
```

### `src/etl-canvas/panels/Dock.tsx`

381 righe

```tsx
/**
 * Il guscio comune dei pannelli: quattro approdi attorno al canvas (sopra,
 * sinistra, destra, sotto), tacca quando un pannello è chiuso, trascinamento
 * della tacca su un altro bordo, schede condivise quando due pannelli stanno
 * sullo stesso bordo (prototipo, righe 4756-4905, CSS 211-346).
 *
 * Lo stato (lato, aperto/chiuso, scheda attiva = il pannello aperto sul
 * bordo) è in etl-store; qui si legge e si cambia solo con i comandi
 * `setPanel`/`setView` (vedi actions.ts). Niente accesso a window/document
 * durante il rendering: gli ascoltatori nascono nei gestori degli eventi.
 */
import { useEffect, useMemo, useRef, useState, useSyncExternalStore } from "react";
import type { CSSProperties, PointerEvent as ReactPointerEvent, ReactNode } from "react";
import type { Size } from "../../etl-layout";
import type { EtlStore, PanelKey, Panels, Side } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import type { InteractionController } from "../interaction";
import type { Loop } from "../loop";
import type { PanelActions } from "./actions";
import { createDockAnimator } from "./animator";
import type { DockAnimator, GhostView } from "./animator";
import { overlayFor } from "./dockArea";
import { ControlBar } from "./ControlBar";
import {
  MIN_CANVAS_HEIGHT,
  cappedPanelHeight,
  PANEL_KEYS,
  PANEL_LABEL,
  PANEL_NAME,
  SIDES,
  SIDE_NAME,
  isGrouped,
  isVertical,
  nearestSide,
  notchHidden,
  notchOffset,
  panelSize,
} from "./layout";
import type { OverlayLayout } from "./overlayLayout";
import "./panels.css";
import { InspectorIcon, ToolsIcon } from "./ui-icons";

/** Soglia prima che la tacca si stacchi dal bordo (prototipo, riga 4884). */
const NOTCH_DRAG_THRESHOLD = 5;
/** Durata della dissolvenza del contenuto al cambio di scheda (riga 4809). */
const TAB_IN_MS = 280;

const TAB_ICON: Record<PanelKey, () => ReactNode> = {
  tools: () => <ToolsIcon />,
  insp: () => <InspectorIcon />,
};

export interface DockLayoutProps {
  readonly store: EtlStore;
  readonly actions: PanelActions;
  /** Controller dei gesti: la barra dei controlli chiede la conferma di «Svuota» al canvas. */
  readonly controller: InteractionController;
  /** Il canvas, al centro; riceve la disposizione dei widget in sovrimpressione, quando l'area è misurata. */
  readonly canvas: (overlay: OverlayLayout | undefined, notice: string | null) => ReactNode;
  /** Il ciclo di animazione condiviso con il canvas: l'animatore dei pannelli e della vista ci si aggiunge. */
  readonly loop: Loop;
  /** Contenuto di ciascun pannello; `side` è il bordo corrente, `horiz` l'orientamento. */
  readonly content: Readonly<Record<PanelKey, (ctx: { side: Side; horiz: boolean }) => ReactNode>>;
  /** Altro da disegnare sopra lo spazio di lavoro (per esempio l'anteprima del trascinamento). */
  readonly overlay?: ReactNode;
}

interface NotchDrag {
  readonly key: PanelKey;
  readonly x: number;
  readonly y: number;
  readonly side: Side;
}

export function DockLayout(props: DockLayoutProps) {
  const { store, actions, controller, canvas, content, overlay, loop } = props;
  const panels = useEtlState((s) => s.panels, store);
  const centerRef = useRef<HTMLDivElement>(null);
  const workspaceRef = useRef<HTMLDivElement>(null);
  // altezza dello spazio di lavoro: da qui il tetto all'altezza dei pannelli orizzontali
  const [workspaceH, setWorkspaceH] = useState(0);
  useEffect(() => {
    const el = workspaceRef.current;
    if (!el) return;
    const measure = () => setWorkspaceH(el.clientHeight);
    measure();
    const ro = new ResizeObserver(measure);
    ro.observe(el);
    return () => ro.disconnect();
  }, []);
  // misura dell'area del canvas (la riga centrale): da qui la disposizione dei widget e la visibilità dei nodi
  const [area, setArea] = useState<Size | null>(null);
  useEffect(() => {
    const el = centerRef.current;
    if (!el) return;
    const measure = () => {
      const w = el.clientWidth;
      const h = el.clientHeight;
      setArea((prev) => (prev && prev.w === w && prev.h === h ? prev : { w, h }));
    };
    measure();
    const ro = new ResizeObserver(measure);
    ro.observe(el);
    return () => ro.disconnect();
  }, []);
  const layout = useMemo(() => (area ? overlayFor(panels, area) : undefined), [area, panels]);
  // un solo orologio: l'animatore imposta a ogni frame l'apertura dei pannelli e la vista
  const [animator] = useState<DockAnimator>(() =>
    createDockAnimator({
      store,
      loop,
      metrics: () => {
        const ws = workspaceRef.current;
        if (!ws) return null;
        const bar = ws.querySelector(".ec-bar-row");
        return {
          w: ws.clientWidth,
          h: ws.clientHeight,
          barH: bar?.getBoundingClientRect().height ?? 0,
        };
      },
    }),
  );
  useEffect(() => animator.start(), [animator]);
  const dock = useSyncExternalStore(animator.subscribe, animator.getSnapshot, animator.getSnapshot);
  // l'area misurata a riposo cambia (finestra, tetto d'altezza): la vista segue senza animazione
  useEffect(() => {
    if (!area || !layout) return;
    // `area` è intera (clientWidth) e può essere un valore di un frame fa: l'animatore riceve la misura di adesso, frazionaria come quella calcolata
    const r = centerRef.current?.getBoundingClientRect();
    animator.onMeasure({ size: r ? { w: r.width, h: r.height } : area, insets: layout.insets });
  }, [area, layout, animator]);
  const [drag, setDrag] = useState<NotchDrag | null>(null);
  // cambiare scheda sostituisce il contenuto sul posto: niente animazione di larghezza, solo una dissolvenza
  const [tabIn, setTabIn] = useState<PanelKey | null>(null);
  const timers = useRef<ReturnType<typeof setTimeout>[]>([]);
  const cleanup = useRef<(() => void) | null>(null);

  useEffect(
    () => () => {
      timers.current.forEach(clearTimeout);
      cleanup.current?.();
    },
    [],
  );

  const switchTab = (key: PanelKey) => {
    animator.snapNext();
    actions.open(key);
    setTabIn(key);
    timers.current.push(setTimeout(() => setTabIn(null), TAB_IN_MS));
  };

  const onNotchPointerDown = (key: PanelKey, e: ReactPointerEvent<HTMLButtonElement>) => {
    if (e.button !== 0) return;
    e.preventDefault();
    e.stopPropagation();
    const sx = e.clientX;
    const sy = e.clientY;
    let moved = false;
    const sideAt = (cx: number, cy: number): Side => {
      const r = centerRef.current?.getBoundingClientRect();
      return r ? nearestSide({ x: cx, y: cy }, r) : panels[key].side;
    };
    const move = (ev: PointerEvent) => {
      if (!moved && Math.hypot(ev.clientX - sx, ev.clientY - sy) < NOTCH_DRAG_THRESHOLD) return;
      moved = true;
      setDrag({ key, x: ev.clientX, y: ev.clientY, side: sideAt(ev.clientX, ev.clientY) });
    };
    const stop = () => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
      window.removeEventListener("pointercancel", abort);
      cleanup.current = null;
      setDrag(null);
    };
    const up = (ev: PointerEvent) => {
      stop();
      // un click apre soltanto
      if (!moved) {
        actions.open(key);
        return;
      }
      const side = sideAt(ev.clientX, ev.clientY);
      if (side === store.getState().panels[key].side) actions.open(key);
      else actions.moveTo(key, side);
    };
    const abort = () => stop();
    cleanup.current = abort;
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
    window.addEventListener("pointercancel", abort);
  };

  return (
    <div
      ref={workspaceRef}
      className="ec-workspace"
      data-testid="ec-workspace"
      // l'area di lavoro ha l'altezza del contenitore: i pannelli in alto e in basso la sottraggono al canvas
      style={{ "--ec-canvas-min-h": `${MIN_CANVAS_HEIGHT}px` } as CSSProperties}
    >
      {SIDES.map((side) => (
        <div key={side} className={`ec-dock ec-dock-${side}`} data-dock={side}>
          {PANEL_KEYS.filter((k) => panels[k].side === side).map((k) => (
            <PanelShell
              key={k}
              pkey={k}
              panels={panels}
              workspaceH={workspaceH}
              animator={animator}
              tabIn={tabIn === k}
              onSwitch={switchTab}
            >
              {content[k]({ side, horiz: !isVertical(side) })}
            </PanelShell>
          ))}
          {dock.ghosts
            .filter((g) => g.side === side)
            .map((g) => (
              <GhostShell key={g.id} ghost={g} workspaceH={workspaceH} animator={animator} />
            ))}
        </div>
      ))}
      <div className="ec-bar-row">
        <ControlBar store={store} controller={controller} area={area} />
      </div>
      <div
        className="ec-center"
        ref={centerRef}
        // ingombri dei widget (alto, destra, basso, sinistra): le verifiche nel browser leggono l'area sicura da qui
        data-insets={
          layout
            ? [
                layout.insets.top,
                layout.insets.right,
                layout.insets.bottom,
                layout.insets.left,
              ].join(",")
            : undefined
        }
      >
        {canvas(layout, dock.notice)}
        <div
          className={"ec-edge-hint" + (drag ? ` ec-on ec-e-${drag.side}` : "")}
          data-testid="ec-edge-hint"
        />
        {PANEL_KEYS.map((k) => {
          const dragging = drag?.key === k;
          const hidden = !dragging && notchHidden(panels, k);
          const side = panels[k].side;
          const rect = layout?.notches[k];
          const pos: CSSProperties = dragging
            ? { left: (drag?.x ?? 0) - 17, top: (drag?.y ?? 0) - 17 }
            : rect
              ? { left: rect.x, top: rect.y }
              : {};
          return (
            <button
              key={k}
              type="button"
              className={
                `ec-notch ec-notch-${side}` +
                (hidden ? " ec-hidden" : "") +
                (dragging ? " ec-dragging" : "")
              }
              style={pos}
              data-notch={k}
              tabIndex={hidden ? -1 : 0}
              aria-hidden={hidden || undefined}
              aria-label={`${PANEL_LABEL[k]}: clicca per aprire, trascina per spostarlo`}
              onPointerDown={(e) => onNotchPointerDown(k, e)}
              onClick={(e) => {
                e.stopPropagation();
                // da tastiera (detail 0) il clic apre; col puntatore ha già aperto il rilascio
                if (e.detail === 0) actions.open(k);
              }}
            >
              {TAB_ICON[k]()}
            </button>
          );
        })}
        {drag && layout?.hint ? (
          <div
            className="ec-hint"
            role="status"
            style={{
              left: layout.hint.x,
              top: layout.hint.y,
              width: layout.hint.w,
              height: layout.hint.h,
            }}
          >
            Rilascia per agganciare {PANEL_NAME[drag.key]} al bordo {SIDE_NAME[drag.side]}
          </div>
        ) : null}
      </div>
      {overlay}
    </div>
  );
}

/** Un pannello nel suo approdo: si apre e si chiude cedendo spazio al canvas; in gruppo mostra le schede. */
function PanelShell(props: {
  pkey: PanelKey;
  panels: Panels;
  workspaceH: number;
  animator: DockAnimator;
  tabIn: boolean;
  onSwitch: (key: PanelKey) => void;
  children: ReactNode;
}) {
  const { pkey, panels, animator, tabIn, onSwitch } = props;
  const { side, open } = panels[pkey];
  const grouped = isGrouped(panels);
  const size = panelSize(panels, pkey);
  // sui bordi orizzontali l'altezza ha un tetto (45% dello spazio di lavoro); il contenuto scorre dentro
  const ph =
    isVertical(side) || props.workspaceH <= 0
      ? size.h
      : cappedPanelHeight(size.h, props.workspaceH);
  const cls = [
    "ec-panel",
    `ec-side-${side}`,
    open ? "ec-open" : "",
    grouped ? "ec-grouped" : "",
    isVertical(side) ? "" : "ec-horiz",
    tabIn ? "ec-tab-in" : "",
  ]
    .filter(Boolean)
    .join(" ");
  return (
    <aside
      className={cls}
      data-panel={pkey}
      data-side={side}
      aria-label={PANEL_LABEL[pkey]}
      inert={!open}
      ref={(el) => animator.register(pkey, el)}
      style={{ "--pw": `${size.w}px`, "--ph": `${ph}px` } as CSSProperties}
    >
      {grouped ? (
        <nav className="ec-dock-tabs" aria-label="Pannelli">
          {PANEL_KEYS.map((t) => (
            <button
              key={t}
              type="button"
              className={"ec-dock-tab" + (t === pkey ? " ec-on" : "")}
              aria-pressed={t === pkey}
              title={PANEL_LABEL[t]}
              onClick={() => t !== pkey && onSwitch(t)}
            >
              {TAB_ICON[t]()}
              <span>{PANEL_LABEL[t]}</span>
            </button>
          ))}
        </nav>
      ) : null}
      <div className="ec-panel-body">{props.children}</div>
    </aside>
  );
}

/** Il guscio vuoto che un pannello spostato lascia sul vecchio bordo: si richiude con lo stesso orologio. */
function GhostShell(props: { ghost: GhostView; workspaceH: number; animator: DockAnimator }) {
  const { ghost, animator } = props;
  const ph =
    isVertical(ghost.side) || props.workspaceH <= 0
      ? ghost.size.h
      : cappedPanelHeight(ghost.size.h, props.workspaceH);
  return (
    <aside
      className={`ec-panel ec-ghost-panel ec-side-${ghost.side}${isVertical(ghost.side) ? "" : " ec-horiz"}`}
      aria-hidden="true"
      inert
      ref={(el) => animator.register(ghost.id, el)}
      style={{ "--pw": `${ghost.size.w}px`, "--ph": `${ph}px` } as CSSProperties}
    />
  );
}
```

### `src/etl-canvas/panels/EtlWorkspace.tsx`

187 righe

```tsx
/**
 * Lo spazio di lavoro: il canvas al centro, i pannelli (cassetta e Inspector)
 * agganciati ai bordi. È ciò che la rotta ETL monta al posto del solo canvas.
 *
 * Come il canvas, si monta solo nel browser: sul server e nel primo rendering
 * di idratazione produce lo stesso segnaposto (i pannelli dipendono dallo
 * stato salvato nel browser).
 */
import { useEffect, useMemo, useRef, useState, useSyncExternalStore } from "react";
import type { PointerEvent as ReactPointerEvent } from "react";
import type { EtlStore } from "../../etl-store";
import { EtlCanvas } from "../EtlCanvas";
import { Icon } from "../icons";
import type { CanvasDropPayload } from "../drop";
import { createInteractionController } from "../interaction";
import { browserEnv, createLoop } from "../loop";
import type { Loop } from "../loop";
import { createPanelActions, followInspector } from "./actions";
import { DockLayout } from "./Dock";
import { familyOfType } from "./families";
import { ExpandedPanel } from "../inspector/ExpandedPanel";
import { InspectorShell } from "./InspectorShell";
import { Toolbox } from "./Toolbox";

const noopSubscribe = () => () => {};

interface Ghost {
  readonly x: number;
  readonly y: number;
  readonly payload: CanvasDropPayload;
}

export function EtlWorkspace(props: { store: EtlStore }) {
  const { store } = props;
  const isClient = useSyncExternalStore(
    noopSubscribe,
    () => true,
    () => false,
  );
  const [controller] = useState(() => createInteractionController(store));
  const actions = useMemo(() => createPanelActions(store), [store]);
  const [ghost, setGhost] = useState<Ghost | null>(null);
  // il box combinato aperto nel pannello espanso
  const [expanded, setExpanded] = useState<string | null>(null);
  // un solo ciclo requestAnimationFrame per tutto lo spazio di lavoro (cavi, pannelli, vista): nasce solo nel browser
  const [loop, setLoop] = useState<Loop | null>(null);
  useEffect(() => {
    const l = createLoop(browserEnv());
    setLoop(l);
    return () => l.dispose();
  }, []);
  const hostRef = useRef<HTMLDivElement>(null);
  const cleanup = useRef<(() => void) | null>(null);

  // l'Inspector si apre con la selezione e si chiude con la deselezione
  useEffect(() => followInspector(store, controller, actions), [store, controller, actions]);
  useEffect(() => () => cleanup.current?.(), []);

  /** Trascinamento di una voce della cassetta (prototipo, righe 4939-5067): un nodo esterno, con la stessa anteprima del trascinamento tra nodi. */
  const onItemPointerDown = (payload: CanvasDropPayload, e: ReactPointerEvent<HTMLElement>) => {
    if (e.button !== 0) return;
    e.preventDefault();
    setGhost({ x: e.clientX, y: e.clientY, payload });
    const stagePoint = (cx: number, cy: number) => {
      const r = hostRef.current?.querySelector(".ec-stage")?.getBoundingClientRect();
      if (!r) return null;
      const inside = cx >= r.left && cx <= r.right && cy >= r.top && cy <= r.bottom;
      return inside ? { x: cx - r.left, y: cy - r.top } : null;
    };
    const move = (ev: PointerEvent) => {
      setGhost({ x: ev.clientX, y: ev.clientY, payload });
      controller.hoverExternal(payload, stagePoint(ev.clientX, ev.clientY));
    };
    const stop = () => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
      window.removeEventListener("pointercancel", abort);
      cleanup.current = null;
      setGhost(null);
    };
    const up = (ev: PointerEvent) => {
      stop();
      controller.dropExternal(payload, stagePoint(ev.clientX, ev.clientY));
    };
    const abort = () => {
      stop();
      controller.dropExternal(payload, null);
    };
    cleanup.current = abort;
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
    window.addEventListener("pointercancel", abort);
  };

  return (
    <div ref={hostRef} className="ec-workspace-host">
      {isClient && expanded ? (
        <ExpandedPanel
          store={store}
          nodeId={expanded}
          onClose={() => {
            const id = expanded;
            setExpanded(null);
            // il focus torna al pulsante di espansione del nodo
            setTimeout(
              () =>
                hostRef.current
                  ?.querySelector<HTMLElement>(`[data-node-id="${id}"] .ec-expand-btn`)
                  ?.focus(),
              0,
            );
          }}
          onConfigure={(index) => {
            setExpanded(null);
            store.dispatch({ type: "select", payload: { ids: [expanded] } });
            store.dispatch({ type: "inspect", payload: { node: expanded, step: index } });
            actions.open("insp");
          }}
          onDetachOutside={(index, x, y) => {
            // il punto di rilascio vive nel mondo, se cade dentro il canvas
            const stage = hostRef.current?.querySelector(".ec-stage")?.getBoundingClientRect();
            const inside =
              !!stage && x >= stage.left && x <= stage.right && y >= stage.top && y <= stage.bottom;
            setExpanded(null);
            store.dispatch({
              type: "detachStep",
              payload: {
                box: expanded,
                index,
                ...(inside && stage
                  ? { dropPoint: controller.toWorld(x - stage.left, y - stage.top) }
                  : {}),
              },
            });
          }}
        />
      ) : null}
      {isClient && loop ? (
        <DockLayout
          store={store}
          actions={actions}
          controller={controller}
          loop={loop}
          canvas={(overlay, notice) => (
            <EtlCanvas
              store={store}
              controller={controller}
              minHeight={0}
              overlay={overlay}
              notice={notice}
              loop={loop}
              onExpand={setExpanded}
            />
          )}
          content={{
            tools: ({ side }) => (
              <Toolbox
                store={store}
                side={side}
                onClose={() => actions.close("tools")}
                onItemPointerDown={onItemPointerDown}
              />
            ),
            insp: ({ side }) => (
              <InspectorShell store={store} side={side} onClose={() => actions.close("insp")} />
            ),
          }}
          overlay={
            ghost ? (
              <div
                className={"ec-ghost" + (ghost.payload.component === "dataset" ? " ec-source" : "")}
                data-testid="ec-ghost"
                data-family={familyOfType(ghost.payload.component)}
                style={{ left: ghost.x - 44, top: ghost.y - 44 }}
              >
                <Icon id={ghost.payload.component} />
              </div>
            ) : null
          }
        />
      ) : (
        <EtlCanvas store={store} />
      )}
    </div>
  );
}
```

### `src/etl-canvas/panels/InspectorShell.tsx`

29 righe

```tsx
/**
 * Il guscio dell'Inspector: si apre e si chiude come gli altri pannelli e ospita
 * il contenuto di `inspector/` (Fase 6b.1).
 */
import type { EtlStore, Side } from "../../etl-store";
import { copy } from "../inspector/copy";
import { Inspector } from "../inspector/Inspector";
import { CloseArrow } from "./ui-icons";

export function InspectorShell(props: { store: EtlStore; side: Side; onClose: () => void }) {
  const { store, side } = props;
  return (
    <div className="ec-tb-inner ec-insp" data-testid="ec-inspector">
      <div className="ec-tb-head">
        <div className="ec-tb-title">Inspector</div>
        <button
          type="button"
          className="ec-close-btn"
          aria-label={copy.closeInspector}
          onClick={props.onClose}
        >
          <CloseArrow side={side} />
        </button>
      </div>
      <Inspector store={store} />
    </div>
  );
}
```

