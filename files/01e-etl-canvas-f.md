# 01e-etl-canvas-f.md

File in questo blocco:

- `src/etl-canvas/engine.ts`
- `src/etl-canvas/flow.ts`
- `src/etl-canvas/icons.tsx`
- `src/etl-canvas/index.ts`
- `src/etl-canvas/inspector/ActionMenu.tsx`
- `src/etl-canvas/inspector/BlockedNotice.tsx`
- `src/etl-canvas/inspector/ColumnPicker.tsx`
- `src/etl-canvas/inspector/ExpandedPanel.tsx`
- `src/etl-canvas/inspector/Field.tsx`
- `src/etl-canvas/inspector/Header.tsx`
- `src/etl-canvas/inspector/Inspector.tsx`
- `src/etl-canvas/inspector/Menu.tsx`

---

### `src/etl-canvas/engine.ts`

335 righe

```ts
/**
 * Il livello sottile tra le funzioni pure (flow.ts, transitions.ts) e il
 * DOM: ad ogni frame del ciclo condiviso calcola lo stato visivo e ne
 * aggiorna solo gli attributi SVG che cambiano (il tracciato del flusso,
 * l'opacità dell'attesa, `d` durante una transizione). Non ridisegna
 * l'albero React.
 *
 * VINCOLO: qui si legge `route.pts` e `route.d` di `getRoutes()` e non si
 * scrive né si ricalcola mai un percorso. Nessuna chiamata a `settleLinks`
 * o a funzioni di etl-layout che instradano (i test lo verificano). Per
 * disegnare i punti interpolati si usa solo `roundedPath`, che arrotonda
 * punti dati e non sceglie nulla.
 */
import { ELBOW_R, roundedPath } from "../etl-layout";
import { flowWindowFor, tubeOutline, waitingOpacityFor } from "./flow";
import type { Sampler } from "./flow";
import { createLoop } from "./loop";
import type { Loop, LoopEnv } from "./loop";
import { planTransition, sampleTransition } from "./transitions";
import type { Plan, Pt, Visual } from "./transitions";

/** Il minimo di un elemento SVG che serve qui (un vero elemento, o un finto nei test). */
export interface AttrEl {
  setAttribute(name: string, value: string): void;
  style: { opacity: string };
}
export interface PathEl extends AttrEl {
  getTotalLength(): number;
  getPointAtLength(s: number): { x: number; y: number };
}
export interface GroupLike {
  querySelector(selector: string): unknown;
}

export interface LinkInput {
  readonly key: string;
  /** `linkLive` di etl-store: solo i cavi attivi hanno il flusso. */
  readonly live: boolean;
  /** `route.pts` e `route.d` di getRoutes, in sola lettura. */
  readonly pts: readonly Pt[];
  readonly d: string;
  readonly pa: Pt;
  readonly pb: Pt;
}

export interface UpdateInput {
  readonly links: readonly LinkInput[];
  /** Un gesto di trascinamento è in corso (`store.isGesturing()`). */
  readonly gesturing: boolean;
}

interface Handles {
  readonly path: PathEl;
  readonly ghost: AttrEl;
  readonly flow: AttrEl;
  readonly dotA: AttrEl;
  readonly dotB: AttrEl;
}

interface Target {
  readonly pts: readonly Pt[];
  readonly d: string;
  readonly pa: Pt;
  readonly pb: Pt;
}

interface Rec {
  group: GroupLike | null;
  handles: Handles | null;
  live: boolean;
  t0: number | null;
  /** Punti mostrati ora (a metà transizione, quelli interpolati). */
  shown: readonly Pt[] | null;
  target: Target | null;
  tr: { plan: Plan; start: number; oldD: string } | null;
  /** Abbiamo scritto `d`/opacità a mano: a fine transizione si ripristinano. */
  dirty: boolean;
}

export interface MotionEngine {
  registerLink(key: string, group: GroupLike | null): void;
  registerSlice(id: string, el: AttrEl | null): void;
  start(env: LoopEnv): void;
  stop(): void;
  update(input: UpdateInput): void;
  setGesturing(gesturing: boolean): void;
  /** Solo per i test. */
  debug(): { links: number; slices: number; running: boolean };
}

function handlesOf(g: GroupLike): Handles | null {
  const path = g.querySelector(".ec-link") as PathEl | null;
  const ghost = g.querySelector(".ec-link-ghost") as AttrEl | null;
  const flow = g.querySelector(".ec-flow") as AttrEl | null;
  const dotA = g.querySelector('[data-dot="a"]') as AttrEl | null;
  const dotB = g.querySelector('[data-dot="b"]') as AttrEl | null;
  if (!path || !ghost || !flow || !dotA || !dotB) return null;
  return { path, ghost, flow, dotA, dotB };
}

function setDot(el: AttrEl, p: Pt): void {
  el.setAttribute("cx", String(p.x));
  el.setAttribute("cy", String(p.y));
}

export function createMotionEngine(): MotionEngine {
  const links = new Map<string, Rec>();
  const slices = new Map<string, { el: AttrEl | null; t0: number | null }>();
  let env: LoopEnv | null = null;
  let loop: Loop | null = null;
  let off: (() => void) | null = null;
  let gesturing = false;
  let last: UpdateInput | null = null;

  const reduced = (): boolean => (env ? env.reducedMotion() : false);

  function clearFlow(rec: Rec): void {
    rec.handles?.flow.setAttribute("d", "");
  }

  function restore(rec: Rec): void {
    const h = rec.handles;
    const t = rec.target;
    if (!h || !t || !rec.dirty) return;
    h.path.setAttribute("d", t.d);
    h.path.style.opacity = "";
    h.ghost.setAttribute("d", "");
    h.ghost.style.opacity = "0";
    setDot(h.dotA, t.pa);
    setDot(h.dotB, t.pb);
    rec.dirty = false;
  }

  function apply(rec: Rec, v: Visual): void {
    const h = rec.handles;
    const t = rec.target;
    const tr = rec.tr;
    if (!h || !t || !tr) return;
    rec.dirty = true;
    if (v.kind === "points") {
      rec.shown = v.pts;
      h.path.setAttribute("d", roundedPath(v.pts, ELBOW_R));
      const a = v.pts[0];
      const b = v.pts[v.pts.length - 1];
      if (a) setDot(h.dotA, a);
      if (b) setDot(h.dotB, b);
    } else {
      // dissolvenza incrociata: il vecchio tracciato svanisce, il nuovo compare
      h.ghost.setAttribute("d", tr.oldD);
      h.ghost.style.opacity = String(v.old);
      h.path.style.opacity = String(v.next);
    }
  }

  function finish(rec: Rec): void {
    rec.tr = null;
    rec.shown = rec.target ? rec.target.pts : null;
    restore(rec);
  }

  function renderFlow(rec: Rec, now: number): boolean {
    const h = rec.handles;
    if (!h) return false;
    if (!rec.live || gesturing) {
      clearFlow(rec);
      return false;
    }
    let len = 0;
    try {
      len = h.path.getTotalLength();
    } catch {
      len = 0;
    }
    const win = flowWindowFor(len, now - (rec.t0 ?? now), reduced());
    if (!win) {
      clearFlow(rec);
      return true;
    }
    const sample: Sampler = (s) => h.path.getPointAtLength(s);
    h.flow.setAttribute("d", tubeOutline(sample, len, win));
    return true;
  }

  function renderSlices(now: number): void {
    for (const s of slices.values()) {
      s.t0 ??= now;
      if (s.el) s.el.style.opacity = String(waitingOpacityFor(now - s.t0, reduced()));
    }
  }

  function frame(now: number): boolean {
    let busy = false;
    for (const rec of links.values()) {
      if (rec.tr) {
        const v = sampleTransition(rec.tr.plan, now - rec.tr.start);
        if (v) apply(rec, v);
        if (!v || v.done) finish(rec);
        else busy = true;
      }
      if (renderFlow(rec, now)) busy = true;
    }
    if ([...slices.values()].some((s) => s.el)) {
      renderSlices(now);
      busy = true;
    }
    return busy;
  }

  /** Stato finale, senza movimento: transizioni concluse, flusso fermo a metà cavo, attesa a riposo. */
  function settle(): void {
    const now = env ? env.now() : 0;
    for (const rec of links.values()) {
      if (rec.tr) finish(rec);
      renderFlow(rec, now);
    }
    for (const s of slices.values()) if (s.el) s.el.style.opacity = "";
  }

  function applyUpdate(input: UpdateInput): void {
    if (!env) return;
    const now = env.now();
    gesturing = input.gesturing;
    const seen = new Set<string>();
    for (const l of input.links) {
      seen.add(l.key);
      const rec = links.get(l.key) ?? newRec();
      links.set(l.key, rec);
      rec.live = l.live;
      rec.t0 ??= now;
      const prev = rec.shown ?? rec.target?.pts ?? null;
      const plan = planTransition({
        prev,
        next: l.pts,
        gesturing: input.gesturing,
        reduced: env.reducedMotion(),
      });
      const oldD = rec.target?.d ?? l.d;
      rec.target = { pts: l.pts, d: l.d, pa: l.pa, pb: l.pb };
      if (plan.kind === "none") {
        rec.tr = null;
        rec.shown = l.pts;
        restore(rec);
      } else {
        rec.tr = { plan, start: now, oldD };
        const v = sampleTransition(plan, 0);
        if (v) apply(rec, v);
      }
    }
    for (const key of [...links.keys()]) if (!seen.has(key)) links.delete(key);
    // le fette smontate (React azzera il riferimento a ogni rendering): ora si dimenticano davvero
    for (const [id, s] of [...slices]) if (!s.el) slices.delete(id);
    if (env.reducedMotion()) settle();
    loop?.wake();
  }

  function newRec(): Rec {
    return {
      group: null,
      handles: null,
      live: false,
      t0: null,
      shown: null,
      target: null,
      tr: null,
      dirty: false,
    };
  }

  return {
    registerLink(key, group) {
      if (!group) {
        // React smonta il gruppo (o ne cambia il riferimento): si dimentica finché non torna
        const rec = links.get(key);
        if (rec) {
          rec.group = null;
          rec.handles = null;
        }
        return;
      }
      const rec = links.get(key) ?? newRec();
      links.set(key, rec);
      rec.group = group;
      rec.handles = handlesOf(group);
      rec.dirty = false;
    },

    registerSlice(id, el) {
      // React chiama il vecchio riferimento con null e il nuovo con l'elemento a ogni
      // rendering del nodo: la fase dell'attesa si conserva per identificativo
      const known = slices.get(id);
      if (!el) {
        if (known) known.el = null;
        return;
      }
      slices.set(id, { el, t0: known?.t0 ?? (env ? env.now() : null) });
      loop?.wake();
    },

    start(e) {
      this.stop();
      env = e;
      loop = createLoop(e);
      off = loop.add({ frame, settle });
      if (last) applyUpdate(last);
    },

    stop() {
      off?.();
      off = null;
      loop?.dispose();
      loop = null;
      env = null;
    },

    update(input) {
      last = input;
      applyUpdate(input);
    },

    setGesturing(g) {
      if (g === gesturing) return;
      gesturing = g;
      if (last) last = { ...last, gesturing: g };
      for (const rec of links.values()) if (g) clearFlow(rec);
      loop?.wake();
    },

    debug: () => ({
      links: links.size,
      slices: [...slices.values()].filter((s) => s.el).length,
      running: loop?.running() ?? false,
    }),
  };
}
```

### `src/etl-canvas/flow.ts`

180 righe

```ts
/**
 * Flusso nei cavi ("tubo elastico") e attesa delle fette vuote: calcolo puro.
 * Nessun React, nessun timer, nessun DOM: il tempo trascorso è un argomento.
 * Semantica del prototipo (docs/prototype/isa-fusion-prototype.html):
 * righe 1415-1424 (costanti e profilo), 1478 (smooth01), 1480-1521
 * (`animateBubbles`), 669-670 (attesa).
 */

/** Spessore del cavo, coincide con il tubo a riposo (riga 1415). */
export const BASE_W = 2.1;
/** px al millisecondo: stessa andatura su cavi lunghi e corti (riga 1416). */
export const SPEED = 0.16;
/** Rigonfiamento massimo per lato (riga 1417). */
export const BALL = 4.4;
/** Apertura rapida davanti, richiusura più lenta dietro (riga 1418). */
export const FRONT = 7.5;
export const BACK = 19;
/** Il tubo emerge dalla porta e vi rientra su questo tratto (riga 1510: `/ 22`). */
export const EDGE_FADE = 22;
/** Sotto questa lunghezza il tratto non si disegna (riga 1497: `s1 - s0 < 2`). */
export const MIN_VISIBLE = 2;
/** Campionamento del contorno: un punto ogni 1,6 px, almeno 10 (riga 1500). */
export const SAMPLE_STEP = 1.6;
export const MIN_SAMPLES = 10;

/** Riga 1478. */
export function smooth01(x: number): number {
  const c = Math.max(0, Math.min(1, x));
  return c * c * (3 - 2 * c);
}

/** Profilo gaussiano asimmetrico attorno al punto che avanza (righe 1419-1422). */
export function tubeProfile(u: number): number {
  const sg = u >= 0 ? FRONT : BACK;
  return Math.exp(-(u * u) / (sg * sg));
}

/** Tratto del cavo occupato dal tubo: `sb` è il punto che avanza (riga 1489-1495). */
export interface FlowWindow {
  readonly cycle: number;
  readonly sb: number;
  readonly s0: number;
  readonly s1: number;
}

/**
 * Finestra del tubo a `elapsedMs` dall'inizio del cavo (`now - st.t0`), su
 * un cavo lungo `len`. `null` se non c'è nulla da disegnare (righe 1488,
 * 1497). Il ciclo è `len + BACK * 3.2`: la pallina entra dalla porta di
 * uscita, sparisce oltre quella di ingresso e riparte.
 */
export function flowWindow(len: number, elapsedMs: number): FlowWindow | null {
  if (!(len > 0)) return null;
  const cycle = len + BACK * 3.2;
  const sb = ((elapsedMs * SPEED) % cycle) - BACK * 1.1;
  const s0 = Math.max(0, sb - BACK * 3);
  const s1 = Math.min(len, sb + FRONT * 3.2);
  if (s1 - s0 < MIN_VISIBLE) return null;
  return { cycle, sb, s0, s1 };
}

/**
 * Indicazione statica per chi preferisce meno movimento: lo stesso tubo,
 * fermo a metà cavo. Non esiste nel prototipo (NOTE_DIVERGENZE.md).
 */
export function staticFlowWindow(len: number): FlowWindow | null {
  if (!(len > 0)) return null;
  const sb = len / 2;
  return {
    cycle: len + BACK * 3.2,
    sb,
    s0: Math.max(0, sb - BACK * 3),
    s1: Math.min(len, sb + FRONT * 3.2),
  };
}

/** Finestra da mostrare: quella animata, o quella statica con movimento ridotto. */
export function flowWindowFor(len: number, elapsedMs: number, reduced: boolean): FlowWindow | null {
  return reduced ? staticFlowWindow(len) : flowWindow(len, elapsedMs);
}

export type Sampler = (s: number) => { readonly x: number; readonly y: number };

/**
 * Contorno chiuso del tubo (righe 1500-1516): campiona il percorso `d`
 * già calcolato con `sample` (in un browser `path.getPointAtLength`) e
 * allarga il tratto secondo `tubeProfile`. Non tocca mai il percorso.
 */
export function tubeOutline(sample: Sampler, len: number, win: FlowWindow): string {
  const n = Math.max(MIN_SAMPLES, Math.ceil((win.s1 - win.s0) / SAMPLE_STEP));
  const pts: { x: number; y: number; s: number }[] = [];
  for (let k = 0; k <= n; k++) {
    const s = win.s0 + ((win.s1 - win.s0) * k) / n;
    const q = sample(s);
    pts.push({ x: q.x, y: q.y, s });
  }
  const left: string[] = [];
  const right: string[] = [];
  for (let k = 0; k <= n; k++) {
    const a = pts[Math.max(0, k - 1)] as { x: number; y: number; s: number };
    const b = pts[Math.min(n, k + 1)] as { x: number; y: number; s: number };
    const p = pts[k] as { x: number; y: number; s: number };
    let tx = b.x - a.x;
    let ty = b.y - a.y;
    const tl = Math.hypot(tx, ty) || 1;
    tx /= tl;
    ty /= tl;
    const edge = smooth01(Math.min(p.s, len - p.s) / EDGE_FADE);
    const w = BASE_W / 2 + BALL * tubeProfile(p.s - win.sb) * edge;
    left.push((p.x - ty * w).toFixed(2) + " " + (p.y + tx * w).toFixed(2));
    right.push((p.x + ty * w).toFixed(2) + " " + (p.y - tx * w).toFixed(2));
  }
  return "M " + left.concat(right.reverse()).join(" L ") + " Z";
}

// --- Attesa delle fette vuote --------------------------------------------------

/** `animation: waiting 1.9s ease-in-out infinite` (riga 669). */
export const WAIT_PERIOD = 1900;
/** `@keyframes waiting { 0%,100% {opacity:.45} 50% {opacity:.95} }` (riga 670). */
export const WAIT_LOW = 0.45;
export const WAIT_HIGH = 0.95;
/** Opacità a riposo del simbolo (riga 669: `opacity:.85`), anche con movimento ridotto. */
export const WAIT_REST = 0.85;

/** Funzione di temporizzazione CSS `cubic-bezier(x1, y1, x2, y2)`. */
export function cubicBezier(x1: number, y1: number, x2: number, y2: number): (x: number) => number {
  const cx = 3 * x1;
  const bx = 3 * (x2 - x1) - cx;
  const ax = 1 - cx - bx;
  const cy = 3 * y1;
  const by = 3 * (y2 - y1) - cy;
  const ay = 1 - cy - by;
  const bx_ = (t: number) => ((ax * t + bx) * t + cx) * t;
  const by_ = (t: number) => ((ay * t + by) * t + cy) * t;
  const dx_ = (t: number) => (3 * ax * t + 2 * bx) * t + cx;
  return (x) => {
    if (x <= 0) return 0;
    if (x >= 1) return 1;
    let t = x;
    for (let i = 0; i < 8; i++) {
      const err = bx_(t) - x;
      if (Math.abs(err) < 1e-7) return by_(t);
      const d = dx_(t);
      if (Math.abs(d) < 1e-6) break;
      t -= err / d;
    }
    let lo = 0;
    let hi = 1;
    t = x;
    for (let i = 0; i < 40; i++) {
      const v = bx_(t);
      if (Math.abs(v - x) < 1e-7) break;
      if (v < x) lo = t;
      else hi = t;
      t = (lo + hi) / 2;
    }
    return by_(t);
  };
}

/** `ease-in-out` di CSS. */
export const easeInOut = cubicBezier(0.42, 0, 0.58, 1);

/**
 * Opacità del simbolo `<>` di una fetta vuota a `elapsedMs`: dal basso
 * (0,45) all'alto (0,95) a metà periodo e ritorno, con `ease-in-out` su
 * ciascuna metà, come i `@keyframes` del prototipo.
 */
export function waitingOpacity(elapsedMs: number): number {
  const phase = (((elapsedMs % WAIT_PERIOD) + WAIT_PERIOD) % WAIT_PERIOD) / WAIT_PERIOD;
  const u = phase < 0.5 ? phase * 2 : (1 - phase) * 2;
  return WAIT_LOW + (WAIT_HIGH - WAIT_LOW) * easeInOut(u);
}

/** Opacità da mostrare: animata, o ferma a riposo con movimento ridotto. */
export function waitingOpacityFor(elapsedMs: number, reduced: boolean): number {
  return reduced ? WAIT_REST : waitingOpacity(elapsedMs);
}
```

### `src/etl-canvas/icons.tsx`

24 righe

```tsx
import { EMPTY_SLOT_ICON, ICONS } from "../etl-core";
import type { ComponentId } from "../etl-core";

/** Icona del catalogo di etl-core (prototipo `svgTag`, righe 979-981). I tracciati sono costanti del catalogo, mai dati dell'utente. */
export function Icon(props: {
  id: ComponentId | "empty";
  svgRef?: (el: SVGSVGElement | null) => void;
}) {
  const inner = props.id === "empty" ? EMPTY_SLOT_ICON : ICONS[props.id];
  return (
    <svg
      ref={props.svgRef}
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth={2}
      strokeLinecap="round"
      strokeLinejoin="round"
      aria-hidden="true"
      dangerouslySetInnerHTML={{ __html: inner }}
    />
  );
}
```

### `src/etl-canvas/index.ts`

21 righe

```ts
/**
 * etl-canvas — Fase 4a: resa visiva del canvas ETL. Importa da etl-core,
 * etl-layout ed etl-store; nessuno di questi importa da qui.
 */
export { EtlCanvas, CanvasSurface } from "./EtlCanvas";
export type { CanvasSurfaceProps } from "./EtlCanvas";
export { prototypeScene } from "./seed";
export { fit, zoomIn, zoomOut, zoomReset, zoomAtPoint } from "./actions";
export { fitView, zoomAt, minimapFrame, bounds } from "./view";
export { nodeView, countClass, isPartial, slicesOf } from "./model";
export { createInteractionController } from "./interaction";
export type {
  InteractionController,
  InteractionUi,
  DownTarget,
  PointerInput,
  KeyInput,
} from "./interaction";
export { handleCanvasDrop, previewCanvasDrop } from "./drop";
export type { CanvasDropPayload, DropPreview } from "./drop";
```

### `src/etl-canvas/inspector/ActionMenu.tsx`

72 righe

```tsx
/**
 * Menu di azioni (ruolo menu) in un portale: usato dal pannello espanso per i
 * passaggi. Frecce, Home, Fine, Invio, Esc; il focus parte dalla prima voce e,
 * alla chiusura, torna al pulsante che l'ha aperto.
 */
import { useEffect, useRef, useState } from "react";
import type { KeyboardEvent, RefObject } from "react";
import { copy } from "./copy";
import { Menu } from "./Menu";

export interface ActionItem {
  readonly id: string;
  readonly label: string;
  readonly danger?: boolean;
  readonly onSelect: () => void;
}

export function ActionMenu(props: {
  readonly anchor: RefObject<HTMLElement | null>;
  readonly items: readonly ActionItem[];
  readonly onClose: (returnFocus: boolean) => void;
  readonly ariaLabel?: string;
}) {
  const { items } = props;
  const [active, setActive] = useState(0);
  const refs = useRef<(HTMLButtonElement | null)[]>([]);
  useEffect(() => refs.current[active]?.focus(), [active]);
  const onKey = (e: KeyboardEvent) => {
    const n = items.length;
    if (e.key === "ArrowDown") setActive((a) => (a + 1) % n);
    else if (e.key === "ArrowUp") setActive((a) => (a - 1 + n) % n);
    else if (e.key === "Home") setActive(0);
    else if (e.key === "End") setActive(n - 1);
    else if (e.key === "Tab") {
      props.onClose(false);
      return;
    } else return;
    e.preventDefault();
  };
  return (
    <Menu
      anchor={props.anchor}
      role="menu"
      ariaLabel={props.ariaLabel ?? copy.stepsMenuLabel}
      onClose={() => props.onClose(false)}
      onEscape={() => props.onClose(true)}
    >
      <div className="ei-menu-scroll" data-scroll="" onKeyDown={onKey}>
        {items.map((it, i) => (
          <button
            key={it.id}
            ref={(el) => {
              refs.current[i] = el;
            }}
            type="button"
            role="menuitem"
            tabIndex={i === active ? 0 : -1}
            className={"ei-option ei-menuitem" + (it.danger ? " ei-danger" : "")}
            onPointerMove={() => setActive(i)}
            onClick={() => {
              it.onSelect();
              props.onClose(false);
            }}
          >
            <span className="ei-option-label">{it.label}</span>
          </button>
        ))}
      </div>
    </Menu>
  );
}
```

### `src/etl-canvas/inspector/BlockedNotice.tsx`

16 righe

```tsx
/** Stato bloccato (prototipo, righe 3708-3719): una lavorazione senza tabella in ingresso non ha nulla da configurare. */
import { copy } from "./copy";
import { LockIcon } from "./icons";

export function BlockedNotice(props: { readonly capacity: number }) {
  return (
    <>
      <div className="ei-lock" role="status" data-testid="ei-blocked">
        <LockIcon />
        <div>{copy.lockedText}</div>
      </div>
      {props.capacity > 1 ? <div className="ei-help">{copy.lockedJoin(props.capacity)}</div> : null}
    </>
  );
}
```

### `src/etl-canvas/inspector/ColumnPicker.tsx`

342 righe

```tsx
/**
 * Selettore multiplo di colonne: le scelte sono etichette rimovibili, NELL'ORDINE
 * DI SCELTA (conta per Ordina, Rimuovi duplicati, Raggruppa), riordinabili con
 * il trascinamento e con Alt+←/→. Il menu (in un portale) ha ricerca, ogni
 * colonna col suo tipo, «Tutte» e «Nessuna» (sulle sole colonne visibili dopo
 * la ricerca) e il conteggio «N colonne su M», annunciato ai lettori di schermo.
 */
import { useEffect, useId, useMemo, useRef, useState } from "react";
import type { KeyboardEvent, PointerEvent } from "react";
import type { ColumnDef } from "../../etl-core";
import { copy } from "./copy";
import { CheckIcon, PlusIcon, XIcon } from "./icons";
import {
  addVisible,
  canonicalName,
  clampActive,
  comboAction,
  filterByQuery,
  fold,
  moveItem,
  moveTarget,
  nearestIndex,
  removeVisible,
  toggleColumn,
} from "./logic";
import { Menu } from "./Menu";

export interface ColumnPickerProps {
  readonly value: readonly string[];
  readonly schema: readonly ColumnDef[];
  readonly onChange: (columns: string[]) => void;
  readonly labelledBy?: string | undefined;
  readonly ariaLabel?: string | undefined;
}

interface Row {
  readonly name: string;
  readonly label: string;
  readonly hint?: string;
  readonly free?: boolean;
}

/** Spostamento minimo prima che un clic su un'etichetta diventi un trascinamento. */
const DRAG_START_PX = 4;

export function ColumnPicker(props: ColumnPickerProps) {
  const { value, schema, onChange } = props;
  const id = useId();
  const listId = `${id}-list`;
  const fieldRef = useRef<HTMLDivElement>(null);
  const addRef = useRef<HTMLButtonElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);
  const listRef = useRef<HTMLDivElement>(null);
  const chipRefs = useRef<(HTMLButtonElement | null)[]>([]);
  const focusIndex = useRef<number | null>(null);
  const [open, setOpen] = useState(false);
  const [query, setQuery] = useState("");
  const [active, setActive] = useState(0);
  const [drag, setDrag] = useState<{ index: number; dx: number; dy: number } | null>(null);
  const names = useMemo(() => schema.map((c) => c.name), [schema]);

  const rows = useMemo<Row[]>(() => {
    const found: Row[] = filterByQuery(
      schema.map((c) => ({ name: c.name, label: c.name, hint: c.type })),
      query,
    );
    const typed = query.trim();
    const exists =
      names.some((n) => fold(n) === fold(typed)) || value.some((v) => fold(v) === fold(typed));
    if (typed && !exists)
      found.push({ name: typed, label: copy.columnsAddTyped(typed), free: true });
    return found;
  }, [schema, names, query, value]);
  const act = clampActive(active, rows.length);
  const visibleNames = rows.filter((r) => !r.free).map((r) => r.name);

  const setColumns = (next: string[]) => onChange(next);
  const close = (returnFocus: boolean) => {
    setOpen(false);
    setQuery("");
    if (returnFocus) addRef.current?.focus();
  };

  useEffect(() => {
    if (open) inputRef.current?.focus();
  }, [open]);
  useEffect(() => {
    if (!open || act < 0) return;
    listRef.current?.querySelector(`[data-index="${act}"]`)?.scrollIntoView({ block: "nearest" });
  }, [open, act, rows.length]);
  // dopo un riordino da tastiera il focus segue l'etichetta spostata
  useEffect(() => {
    if (focusIndex.current !== null) {
      chipRefs.current[focusIndex.current]?.focus();
      focusIndex.current = null;
    }
  });

  const toggleRow = (r: Row) => {
    if (r.free) {
      setColumns([...value, canonicalName(names, r.name)]);
      setQuery("");
    } else setColumns(toggleColumn(value, r.name));
  };

  // il Tab resta nel menu (Tutte, Nessuna); alla fine il menu si chiude da solo
  const onInputKey = (e: KeyboardEvent<HTMLInputElement>) => {
    const a = comboAction(e.key, act, rows.length);
    if (a.kind === "move") {
      e.preventDefault();
      setActive(a.to);
    } else if (a.kind === "commit") {
      e.preventDefault();
      const r = rows[act];
      if (r) toggleRow(r);
    } else if (a.kind === "close") {
      e.preventDefault();
      e.stopPropagation();
      close(true);
    }
  };

  // --- etichette: tastiera e trascinamento -----------------------------------
  const onChipKey = (e: KeyboardEvent<HTMLButtonElement>, i: number) => {
    if (e.altKey && (e.key === "ArrowLeft" || e.key === "ArrowRight")) {
      e.preventDefault();
      const to = moveTarget(i, value.length, e.key);
      if (to !== null) {
        focusIndex.current = to;
        setColumns(moveItem(value, i, to));
      }
    } else if (e.key === "Backspace" || e.key === "Delete") {
      e.preventDefault();
      setColumns(value.filter((_, k) => k !== i));
      focusIndex.current = Math.max(0, Math.min(i, value.length - 2));
      if (value.length <= 1) addRef.current?.focus();
    }
  };
  const onChipPointerDown = (e: PointerEvent<HTMLButtonElement>, i: number) => {
    if (e.button !== 0) return;
    const sx = e.clientX;
    const sy = e.clientY;
    let moved = false;
    const el = e.currentTarget;
    // i centri delle etichette prima che quella trascinata si muova: il suo centro seguirebbe il puntatore
    const centers = chipRefs.current.slice(0, value.length).map((c) => {
      const r = c?.getBoundingClientRect();
      return r ? { x: r.left + r.width / 2, y: r.top + r.height / 2 } : { x: 0, y: 0 };
    });
    el.setPointerCapture(e.pointerId);
    const move = (ev: globalThis.PointerEvent) => {
      const dx = ev.clientX - sx;
      const dy = ev.clientY - sy;
      if (!moved && Math.hypot(dx, dy) < DRAG_START_PX) return;
      moved = true;
      setDrag({ index: i, dx, dy });
    };
    const up = (ev: globalThis.PointerEvent) => {
      el.removeEventListener("pointermove", move);
      el.removeEventListener("pointerup", up);
      el.removeEventListener("pointercancel", up);
      setDrag(null);
      if (!moved) return;
      const to = nearestIndex(centers, { x: ev.clientX, y: ev.clientY });
      // il clic che segue un trascinamento non deve fare altro
      el.addEventListener("click", (c) => c.stopPropagation(), { once: true, capture: true });
      if (to !== i) {
        focusIndex.current = to;
        setColumns(moveItem(value, i, to));
      }
    };
    el.addEventListener("pointermove", move);
    el.addEventListener("pointerup", up);
    el.addEventListener("pointercancel", up);
  };

  const countText = copy.columnsCount(value.length, schema.length);

  return (
    <>
      <div
        ref={fieldRef}
        className="ei-field ei-chipsfield"
        data-picker="columns"
        data-selected={value.length}
        data-total={schema.length}
        role="group"
        aria-labelledby={props.labelledBy}
        aria-label={props.labelledBy ? undefined : props.ariaLabel}
        data-open={open || undefined}
      >
        {value.length === 0 ? (
          <span className="ei-placeholder">{copy.columnsPlaceholder}</span>
        ) : null}
        <ul className="ei-chips">
          {value.map((name, i) => {
            const known = names.includes(name);
            const dragging = drag?.index === i;
            return (
              <li
                key={name}
                className={"ei-chip" + (known ? "" : " ei-free") + (dragging ? " ei-dragging" : "")}
                style={
                  dragging && drag
                    ? { transform: `translate(${drag.dx}px, ${drag.dy}px)` }
                    : undefined
                }
              >
                <button
                  ref={(el) => {
                    chipRefs.current[i] = el;
                  }}
                  type="button"
                  className="ei-chip-label"
                  title={known ? copy.columnsMoveHelp : copy.columnsFree}
                  aria-label={copy.columnsPosition(name, i + 1, value.length)}
                  onKeyDown={(e) => onChipKey(e, i)}
                  onPointerDown={(e) => onChipPointerDown(e, i)}
                >
                  {name}
                </button>
                <button
                  type="button"
                  className="ei-chip-x"
                  aria-label={copy.columnsRemove(name)}
                  onClick={() => setColumns(value.filter((_, k) => k !== i))}
                >
                  <XIcon />
                </button>
              </li>
            );
          })}
        </ul>
        <button
          ref={addRef}
          type="button"
          className="ei-add"
          aria-haspopup="listbox"
          aria-expanded={open}
          aria-controls={open ? listId : undefined}
          onClick={() => (open ? close(false) : setOpen(true))}
        >
          <PlusIcon />
          <span>{copy.columnsAdd}</span>
        </button>
      </div>
      {open ? (
        <Menu
          anchor={fieldRef}
          onClose={() => close(false)}
          onEscape={() => close(true)}
          ariaLabel={copy.columnsMenu}
        >
          <div className="ei-menu-head">
            <input
              ref={inputRef}
              type="text"
              className="ei-search"
              role="combobox"
              aria-expanded="true"
              aria-controls={listId}
              aria-autocomplete="list"
              aria-activedescendant={act >= 0 ? `${id}-opt-${act}` : undefined}
              aria-label={copy.columnsSearch}
              placeholder={copy.columnsSearch}
              autoComplete="off"
              spellCheck={false}
              value={query}
              onChange={(e) => {
                setQuery(e.target.value);
                setActive(0);
              }}
              onKeyDown={onInputKey}
            />
          </div>
          <div
            ref={listRef}
            id={listId}
            className="ei-menu-scroll"
            role="listbox"
            aria-multiselectable="true"
            aria-label={copy.columnsMenu}
            data-scroll=""
          >
            {rows.length === 0 ? <div className="ei-menu-empty">{copy.noResults}</div> : null}
            {rows.map((r, i) => {
              const on = !r.free && value.includes(r.name);
              return (
                <div
                  key={`${r.free ? "free:" : ""}${r.name}`}
                  id={`${id}-opt-${i}`}
                  data-index={i}
                  role="option"
                  aria-selected={on}
                  className={
                    "ei-option" +
                    (i === act ? " ei-active" : "") +
                    (on ? " ei-selected" : "") +
                    (r.free ? " ei-free" : "")
                  }
                  onPointerDown={(e) => e.preventDefault()}
                  onPointerMove={() => setActive(i)}
                  onClick={() => toggleRow(r)}
                >
                  {r.free ? null : (
                    <span className="ei-checkbox" data-on={on || undefined} aria-hidden="true">
                      {on ? <CheckIcon /> : null}
                    </span>
                  )}
                  <span className="ei-option-label">{r.label}</span>
                  {r.hint ? <span className="ei-option-hint">{r.hint}</span> : null}
                </div>
              );
            })}
          </div>
          <div className="ei-menu-foot">
            <span className="ei-count" role="status" aria-live="polite">
              {countText}
            </span>
            <span className="ei-menu-actions">
              <button
                type="button"
                className="ei-link-btn"
                onClick={() => setColumns(addVisible(value, visibleNames))}
              >
                {copy.columnsAll}
              </button>
              <button
                type="button"
                className="ei-link-btn"
                onClick={() => setColumns(removeVisible(value, visibleNames))}
              >
                {copy.columnsNone2}
              </button>
            </span>
          </div>
        </Menu>
      ) : null}
    </>
  );
}
```

### `src/etl-canvas/inspector/ExpandedPanel.tsx`

115 righe

```tsx
/**
 * Il pannello espanso di un box combinato (prototipo, righe 855-866, 2117-2135 e
 * 2250-2400): i passaggi in sequenza, riordinabili; ognuno ha un menu
 * («Configura parametri», «Sgancia», «Elimina passaggio») e trascinarlo fuori dal
 * pannello lo sgancia sul canvas, nel punto di rilascio. Si chiude con Esc, con
 * il pulsante o con un clic sullo sfondo, e da solo se il box non è più combinato.
 */
import { useEffect, useRef } from "react";
import type { KeyboardEvent } from "react";
import { createPortal } from "react-dom";
import type { EtlStore } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import { copy } from "./copy";
import { XIcon } from "./icons";
import { NameInput } from "./NameInput";
import { StepList } from "./StepList";

export function ExpandedPanel(props: {
  readonly store: EtlStore;
  readonly nodeId: string;
  readonly onClose: () => void;
  /** «Configura parametri»: apre l'Inspector su quel passaggio. */
  readonly onConfigure: (index: number) => void;
  /** Rilascio fuori dal pannello: sgancia il passaggio nel punto indicato (coordinate della finestra). */
  readonly onDetachOutside: (index: number, clientX: number, clientY: number) => void;
}) {
  const { store, nodeId } = props;
  const card = useEtlState((s) => s.graph.cards[nodeId], store);
  const step = useEtlState((s) => s.inspector.step, store);
  const panelRef = useRef<HTMLDivElement>(null);
  const closeRef = useRef<HTMLButtonElement>(null);
  const combined = !!card && card.kind === "op" && card.components.length > 1;

  // non più combinato (sgancio dell'ultimo passaggio, eliminazione): si chiude da solo
  useEffect(() => {
    if (!combined) props.onClose();
  }, [combined, props]);
  useEffect(() => {
    closeRef.current?.focus();
  }, []);

  if (!card || !combined || typeof document === "undefined") return null;

  const onKey = (e: KeyboardEvent<HTMLDivElement>) => {
    if (e.key === "Escape") {
      e.preventDefault();
      e.stopPropagation();
      props.onClose();
    } else if (e.key === "Tab") {
      // il focus resta dentro il pannello
      const items = Array.from(
        panelRef.current?.querySelectorAll<HTMLElement>(
          "button:not([disabled]), input:not([disabled])",
        ) ?? [],
      );
      const first = items[0];
      const last = items[items.length - 1];
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last?.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first?.focus();
      }
    }
  };

  return createPortal(
    <div
      className="ei-expanded-backdrop"
      data-testid="ei-expanded"
      onKeyDown={onKey}
      onPointerDown={(e) => {
        if (e.target === e.currentTarget) props.onClose();
      }}
    >
      <div
        ref={panelRef}
        className="ei-expanded ei-root"
        role="dialog"
        aria-modal="true"
        aria-label={copy.expandedTitle}
      >
        <div className="ei-expanded-head">
          <div>
            <NameInput store={store} card={card} className="ei-name" testId="ei-expanded-name" />
            <div className="ei-help">{copy.expandedSub(card.components.length)}</div>
          </div>
          <button
            ref={closeRef}
            type="button"
            className="ei-icon-btn"
            aria-label={copy.expandedClose}
            onClick={props.onClose}
          >
            <XIcon />
          </button>
        </div>
        <div className="ei-help">{copy.expandedNote}</div>
        <StepList
          store={store}
          card={card}
          selectedStep={step}
          variant="expanded"
          containerRef={panelRef}
          onSelect={() => {}}
          onConfigure={props.onConfigure}
          onDetachOutside={props.onDetachOutside}
        />
      </div>
    </div>,
    document.getElementById("ei-portal") ?? document.body,
  );
}
```

### `src/etl-canvas/inspector/Field.tsx`

47 righe

```tsx
/** Un campo dell'Inspector: etichetta, controllo, nota; più il campo di testo (senza frecce native). */
import { useId } from "react";
import type { ReactNode } from "react";

export function Field(props: {
  readonly label: string;
  readonly children: (labelId: string) => ReactNode;
  readonly help?: ReactNode;
}) {
  const labelId = useId();
  return (
    <div className="ei-fieldgroup">
      <div id={labelId} className="ei-label">
        {props.label}
      </div>
      {props.children(labelId)}
      {props.help ? <div className="ei-help">{props.help}</div> : null}
    </div>
  );
}

export function TextField(props: {
  readonly value: string;
  readonly onChange: (value: string) => void;
  readonly labelledBy?: string | undefined;
  readonly ariaLabel?: string | undefined;
  readonly inputMode?: "text" | "numeric" | "decimal";
  readonly disabled?: boolean;
  readonly placeholder?: string;
}) {
  return (
    <input
      type="text"
      className="ei-field ei-input"
      inputMode={props.inputMode ?? "text"}
      aria-labelledby={props.labelledBy}
      aria-label={props.labelledBy ? undefined : props.ariaLabel}
      autoComplete="off"
      spellCheck={false}
      disabled={props.disabled}
      placeholder={props.placeholder}
      value={props.value}
      onChange={(e) => props.onChange(e.target.value)}
    />
  );
}
```

### `src/etl-canvas/inspector/Header.tsx`

22 righe

```tsx
/** Intestazione dell'Inspector: la famiglia del nodo (testo piccolo in maiuscolo) e il nome, modificabile in linea. */
import type { Card } from "../../etl-core";
import type { EtlStore } from "../../etl-store";
import { familyLabel } from "./family";
import { NameInput } from "./NameInput";

export function Header(props: { readonly store: EtlStore; readonly card: Card }) {
  return (
    <div className="ei-header">
      <div className="ei-overline" data-testid="ei-family">
        {familyLabel(props.card)}
      </div>
      <NameInput
        store={props.store}
        card={props.card}
        className="ei-name"
        testId="ec-inspector-name"
      />
    </div>
  );
}
```

### `src/etl-canvas/inspector/Inspector.tsx`

287 righe

```tsx
/**
 * Il contenuto dell'Inspector (Fase 6b.1), montato da `panels/InspectorShell`.
 * Legge il nodo di `etl-store` (`inspector`) e mostra, per tipo di nodo:
 * stato bloccato (lavorazione senza ingresso), dataset e output (nome, origine,
 * colonne in sola lettura), lavorazioni (campi e liste del catalogo, con colonne
 * e valori dallo schema in ingresso) e box combinati (elenco dei passaggi).
 * Le condizioni di filtro e join sono della Fase 6b.2.
 *
 * Scrive solo con i comandi `setParams`, `renameNode`, `inspect` e quelli dei
 * passaggi; legge e scrive SOLO `columns` (mai `flattenRows`). Il focus non si
 * perde mai scrivendo: i campi hanno chiavi stabili e nulla viene ricreato.
 */
import type { KeyboardEvent } from "react";
import { MERGE_OPS, PARAM_DEFS, boxCapacity, inputsOf } from "../../etl-core";
import type { Card, Graph, Params, SimpleFieldDef } from "../../etl-core";
import type { EtlStore } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import { BlockedNotice } from "./BlockedNotice";
import { copy } from "./copy";
import { Field, TextField } from "./Field";
import { Header } from "./Header";
import { MultiList } from "./MultiList";
import { NUMERIC_KEYS, isMulti, textParam, withParam } from "./params";
import { StepList } from "./StepList";
import { StyledSelect } from "./StyledSelect";
import { useActiveSchema } from "./useActiveSchema";
import "./inspector.css";

/** Esc nel pannello riporta il focus al canvas, senza deselezionare. */
function returnToCanvas(e: KeyboardEvent<HTMLElement>): void {
  if (e.key !== "Escape" || e.defaultPrevented) return;
  e.stopPropagation();
  document.querySelector<HTMLElement>(".ec-stage")?.focus();
}

export function Inspector(props: { readonly store: EtlStore }) {
  const { store } = props;
  const nodeId = useEtlState((s) => s.inspector.nodeId, store);
  const step = useEtlState((s) => s.inspector.step, store);
  const card = useEtlState(
    (s) => (s.inspector.nodeId ? s.graph.cards[s.inspector.nodeId] : undefined),
    store,
  );
  const graph = useEtlState((s) => s.graph, store);
  const schema = useActiveSchema(store, nodeId);

  if (!card) {
    return (
      <div className="ei-root" data-testid="ei-root" onKeyDown={returnToCanvas}>
        <div className="ei-help ei-empty">{copy.emptyInspector}</div>
      </div>
    );
  }
  return (
    <div
      className="ei-root"
      data-testid="ei-root"
      data-kind={kindOf(card)}
      onKeyDown={returnToCanvas}
    >
      <Header store={store} card={card} />
      <Body store={store} card={card} graph={graph} step={step} schema={schema} />
    </div>
  );
}

function kindOf(card: Card): string {
  if (card.kind === "dataset") return card.isOutput ? "output" : "dataset";
  return card.components.length > 1 ? "box" : "op";
}

function Body(props: {
  store: EtlStore;
  card: Card;
  graph: Graph;
  step: number;
  schema: ReturnType<typeof useActiveSchema>;
}) {
  const { store, card, graph, schema } = props;

  if (card.kind === "dataset" && card.isOutput) {
    const producer = graph.links.find((l) => l.to === card.id);
    const producerName = (producer && graph.cards[producer.from]?.name) || copy.producerFallback;
    const incomplete =
      card.capacity !== undefined && card.capacity > 1 && (card.filled ?? 0) < card.capacity;
    return (
      <>
        <div className="ei-help">{copy.resultNote(producerName)}</div>
        {incomplete ? <div className="ei-help">{copy.resultIncomplete}</div> : null}
        <ColumnsReadOnly schema={schema} />
      </>
    );
  }

  if (card.kind === "dataset") {
    const par = card.params[0] ?? {};
    return (
      <>
        <SimpleFields
          defs={PARAM_DEFS.dataset as readonly SimpleFieldDef[]}
          par={par}
          names={[]}
          onChange={(p) =>
            store.dispatch({ type: "setParams", payload: { node: card.id, index: 0, params: p } })
          }
        />
        <ColumnsReadOnly schema={schema} />
      </>
    );
  }

  // lavorazione senza tabella in ingresso: stato bloccato, nessun campo
  const inputs = inputsOf(graph, card.id);
  if (inputs.length === 0) return <BlockedNotice capacity={boxCapacity(card)} />;

  const step = Math.max(0, Math.min(props.step, card.components.length - 1));
  const type = card.components[step] ?? card.components[0];
  const par: Params = card.params[step] ?? {};
  const setParams = (p: Params) =>
    store.dispatch({ type: "setParams", payload: { node: card.id, index: step, params: p } });
  const names = schema.map((c) => c.name);

  return (
    <>
      {card.components.length > 1 ? (
        <StepList
          store={store}
          card={card}
          selectedStep={step}
          variant="inspector"
          onSelect={(i) => store.dispatch({ type: "inspect", payload: { node: card.id, step: i } })}
        />
      ) : null}
      <JoinTables card={card} graph={graph} step={step} par={par} onChange={setParams} />
      {type === "filter" || type === "join" ? (
        <div className="ei-help" data-testid="ei-conditions-soon">
          {copy.conditionsSoon}
        </div>
      ) : type && isMulti(type) ? (
        <MultiList type={type} par={par} schema={schema} onChange={setParams} />
      ) : (
        <SimpleFields
          defs={
            (Array.isArray(PARAM_DEFS[type as keyof typeof PARAM_DEFS])
              ? PARAM_DEFS[type as keyof typeof PARAM_DEFS]
              : []) as readonly SimpleFieldDef[]
          }
          par={par}
          names={names}
          onChange={setParams}
        />
      )}
      <div className="ei-help">{copy.inputsCount(inputs.length, boxCapacity(card))}</div>
    </>
  );
}

/** Campi semplici del catalogo (`PARAM_DEFS`): testo, scelta, colonna. */
function SimpleFields(props: {
  defs: readonly SimpleFieldDef[];
  par: Params;
  names: readonly string[];
  onChange: (par: Params) => void;
}) {
  const { defs, par, names, onChange } = props;
  return (
    <>
      {defs.map((f) => (
        <Field key={f.k} label={f.label}>
          {(labelId) => {
            const value = textParam(par, f.k);
            if (f.type === "select")
              return (
                <StyledSelect
                  labelledBy={labelId}
                  value={value || f.def}
                  options={(f.opts ?? []).map((o) => ({ value: o, label: o }))}
                  onChange={(v) => onChange(withParam(par, f.k, v))}
                />
              );
            if (f.type === "column")
              return (
                <StyledSelect
                  labelledBy={labelId}
                  allowFree
                  value={value}
                  options={names.map((n) => ({ value: n, label: n }))}
                  onChange={(v) => onChange(withParam(par, f.k, v))}
                />
              );
            return (
              <TextField
                labelledBy={labelId}
                inputMode={NUMERIC_KEYS.has(f.k) ? "numeric" : "text"}
                value={value}
                onChange={(v) => onChange(withParam(par, f.k, v))}
              />
            );
          }}
        </Field>
      ))}
    </>
  );
}

/** Dataset e output: le colonne, in sola lettura, col tipo. */
function ColumnsReadOnly(props: { schema: ReturnType<typeof useActiveSchema> }) {
  const { schema } = props;
  return (
    <section className="ei-list" aria-label={copy.columnsTitle}>
      <div className="ei-label">{copy.columnsTitle}</div>
      {schema.length === 0 ? (
        <div className="ei-help">{copy.columnsNone}</div>
      ) : (
        <ul className="ei-colist" data-testid="ei-columns">
          {schema.map((c) => (
            <li key={c.name} className="ei-colrow">
              <span className="ei-colname">{c.name}</span>
              <span className="ei-coltype">{c.type}</span>
            </li>
          ))}
        </ul>
      )}
    </section>
  );
}

/**
 * Le tabelle su cui agisce un passaggio (prototipo, righe 3747-3774): i join si
 * applicano nell'ordine in cui compaiono e ognuno consuma una tabella in più.
 * Prima di un join: tabella di riferimento; dopo: la nota «tabella unica».
 */
function JoinTables(props: {
  card: Card;
  graph: Graph;
  step: number;
  par: Params;
  onChange: (par: Params) => void;
}) {
  const { card, graph, step, par, onChange } = props;
  const joinPos: number[] = [];
  card.components.forEach((c, i) => {
    if ((MERGE_OPS as readonly string[]).includes(c)) joinPos.push(i);
  });
  if (joinPos.length === 0) return null;
  const names = inputsOf(graph, card.id).map(
    (l) => graph.cards[l.from]?.name ?? copy.tableFallback,
  );
  if (names.length === 0) return <div className="ei-help">{copy.tableLinkFirst}</div>;

  const field = (key: string, label: string) => {
    const stored = textParam(par, key);
    const fallback = key === "rightTable" && names[1] ? names[1] : names[0];
    const value = stored && names.includes(stored) ? stored : (fallback ?? "");
    return (
      <Field key={key} label={label}>
        {(labelId) => (
          <StyledSelect
            labelledBy={labelId}
            value={value}
            options={names.map((n) => ({ value: n, label: n }))}
            onChange={(v) => onChange(withParam(par, key, v))}
          />
        )}
      </Field>
    );
  };

  const type = card.components[step];
  const joinsBefore = joinPos.filter((p) => p < step).length;
  if (type && (MERGE_OPS as readonly string[]).includes(type)) {
    const j = joinPos.indexOf(step);
    return (
      <>
        {j === 0 ? (
          field("leftTable", copy.tableLeft)
        ) : (
          <div className="ei-help">{copy.tableLeftResult(j)}</div>
        )}
        {field("rightTable", copy.tableRight)}
      </>
    );
  }
  if (joinsBefore === 0) return field("table", copy.tableReference);
  return <div className="ei-help">{copy.tableSingle(joinsBefore)}</div>;
}
```

### `src/etl-canvas/inspector/Menu.tsx`

161 righe

```tsx
/**
 * Il menu dell'Inspector: sempre un nostro componente (mai un menu del
 * sistema), in un portale sul corpo della pagina, posizionato da `placeMenu`
 * (menu.ts). Si chiude con Esc (a cura di chi lo usa), clic fuori, scorrimento
 * del pannello e ridimensionamento della finestra.
 */
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import type { CSSProperties, ReactNode, RefObject } from "react";
import { createPortal } from "react-dom";
import { placeMenu } from "./menu";
import type { MenuPlacement } from "./menu";

const PORTAL_ID = "ei-portal";

/** Il contenitore dei menu, creato alla prima richiesta (solo nel browser). */
function portalRoot(): HTMLElement {
  let el = document.getElementById(PORTAL_ID);
  if (!el) {
    el = document.createElement("div");
    el.id = PORTAL_ID;
    el.className = "ei-portal";
    document.body.appendChild(el);
  }
  return el;
}

/** Altezza naturale del menu: il suo riempimento più i figli (l'elenco scorrevole conta per intero, fino al suo tetto). */
function naturalHeight(menu: HTMLElement): number {
  const cs = getComputedStyle(menu);
  let h =
    parseFloat(cs.paddingTop) +
    parseFloat(cs.paddingBottom) +
    parseFloat(cs.borderTopWidth) +
    parseFloat(cs.borderBottomWidth);
  for (const kid of Array.from(menu.children) as HTMLElement[]) {
    if (kid.dataset["scroll"] !== undefined) {
      const cap = parseFloat(getComputedStyle(kid).maxHeight);
      h += Number.isFinite(cap) ? Math.min(kid.scrollHeight, cap) : kid.scrollHeight;
    } else h += kid.offsetHeight;
  }
  return Math.ceil(h);
}

export interface MenuProps {
  /** L'elemento a cui si ancora (il campo). */
  readonly anchor: RefObject<HTMLElement | null>;
  readonly onClose: () => void;
  readonly children: ReactNode;
  readonly className?: string;
  /** Altri elementi che contano come «dentro» per il clic fuori (di solito il campo). */
  readonly inside?: readonly RefObject<HTMLElement | null>[];
  readonly id?: string;
  readonly ariaLabel?: string;
  readonly role?: "menu" | "dialog" | "presentation";
  /** Esc con il focus dentro il menu (fuori dal campo di ricerca): chiude e riporta il focus al campo. */
  readonly onEscape?: () => void;
  /** Larghezza naturale del contenuto, se maggiore di quella del campo. */
  readonly naturalWidth?: number;
}

export function Menu(props: MenuProps) {
  const { anchor, onClose, children, inside } = props;
  const menuRef = useRef<HTMLDivElement>(null);
  const [placement, setPlacement] = useState<MenuPlacement | null>(null);
  // il portale esiste subito (il menu si monta solo nel browser, dopo un'azione dell'utente):
  // così chi lo usa può dare il focus al campo di ricerca già al primo effetto
  const [root] = useState<HTMLElement | null>(() =>
    typeof document === "undefined" ? null : portalRoot(),
  );

  const place = useCallback(() => {
    const a = anchor.current;
    const m = menuRef.current;
    if (!a || !m) return;
    const r = a.getBoundingClientRect();
    setPlacement(
      placeMenu({
        field: { x: r.left, y: r.top, w: r.width, h: r.height },
        win: { w: window.innerWidth, h: window.innerHeight },
        naturalHeight: naturalHeight(m),
        ...(props.naturalWidth !== undefined ? { naturalWidth: props.naturalWidth } : {}),
      }),
    );
  }, [anchor, props.naturalWidth]);

  // prima misura, e nuova misura quando il contenuto cambia (ricerca, voci aggiunte)
  useLayoutEffect(() => {
    if (!root) return;
    place();
    const m = menuRef.current;
    if (!m) return;
    const ro = new ResizeObserver(() => place());
    for (const kid of Array.from(m.children)) ro.observe(kid);
    return () => ro.disconnect();
  }, [root, place, children]);

  // chiusura: clic fuori, scorrimento del pannello, ridimensionamento
  useEffect(() => {
    const away = (e: Event) => {
      const t = e.target as Node | null;
      if (!t) return;
      if (menuRef.current?.contains(t)) return;
      if (anchor.current?.contains(t)) return;
      if (inside?.some((r) => r.current?.contains(t))) return;
      onClose();
    };
    const scrolled = (e: Event) => {
      const t = e.target as Node | null;
      if (t && menuRef.current?.contains(t)) return;
      onClose();
    };
    document.addEventListener("pointerdown", away, true);
    window.addEventListener("scroll", scrolled, true);
    window.addEventListener("resize", onClose);
    return () => {
      document.removeEventListener("pointerdown", away, true);
      window.removeEventListener("scroll", scrolled, true);
      window.removeEventListener("resize", onClose);
    };
  }, [anchor, inside, onClose]);

  if (!root) return null;
  const style: CSSProperties = placement
    ? {
        left: placement.left,
        top: placement.top,
        width: placement.width,
        maxHeight: placement.maxHeight,
      }
    : { left: 0, top: 0, opacity: 0, pointerEvents: "none", width: anchor.current?.offsetWidth };
  return createPortal(
    <div
      ref={menuRef}
      id={props.id}
      className={"ei-menu" + (props.className ? ` ${props.className}` : "")}
      data-side={placement?.side}
      role={props.role ?? "presentation"}
      aria-label={props.ariaLabel}
      style={style}
      onKeyDown={(e) => {
        if (e.key === "Escape" && props.onEscape) {
          e.preventDefault();
          e.stopPropagation();
          props.onEscape();
        }
      }}
      onBlur={(e) => {
        // il focus esce dal menu verso altro (non dal campo): il menu si chiude
        const next = e.relatedTarget as Node | null;
        if (!next) return;
        if (menuRef.current?.contains(next) || anchor.current?.contains(next)) return;
        if (inside?.some((r) => r.current?.contains(next))) return;
        onClose();
      }}
    >
      {children}
    </div>,
    root,
  );
}
```

