# 01e-etl-canvas-g.md

File in questo blocco:

- `src/etl-canvas/inspector/ColumnPicker.tsx`
- `src/etl-canvas/inspector/ExpandedPanel.tsx`
- `src/etl-canvas/inspector/Field.tsx`
- `src/etl-canvas/inspector/Header.tsx`
- `src/etl-canvas/inspector/Inspector.tsx`
- `src/etl-canvas/inspector/Menu.tsx`
- `src/etl-canvas/inspector/MultiList.tsx`
- `src/etl-canvas/inspector/NameInput.tsx`
- `src/etl-canvas/inspector/StepList.tsx`

---

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

### `src/etl-canvas/inspector/MultiList.tsx`

273 righe

```tsx
/**
 * Le operazioni a voci multiple del catalogo (`MULTI_DEFS`): campi globali e
 * liste di righe comprimibili (una aperta per volta), con il riassunto di riga
 * dal vivo (`L.sum`), aggiunta e rimozione di righe e note. Ogni campo di tipo
 * «columns» usa il ColumnPicker; i campi a colonna singola, le scelte e i
 * valori d'una sola colonna usano StyledSelect; i valori il ValuePicker.
 * Il riordino dei criteri di Ordina non è di questa fase.
 */
import { useId, useState } from "react";
import {
  MULTI_DEFS,
  columnsDomain,
  createValuesField,
  fieldFilled,
  measureNames,
} from "../../etl-core";
import type {
  ColumnDef,
  MultiFieldDef,
  MultiListDef,
  MultiRow,
  OperationType,
  Params,
  ValuesField,
} from "../../etl-core";
import { ColumnPicker } from "./ColumnPicker";
import { copy } from "./copy";
import { ChevronRight, XIcon } from "./icons";
import { Field, TextField } from "./Field";
import {
  NUMERIC_KEYS,
  multiOf,
  rowColumns,
  rowsOf,
  withGlobal,
  withRowAdded,
  withRowField,
  withRowRemoved,
} from "./params";
import { StyledSelect } from "./StyledSelect";
import { ValuePicker } from "./ValuePicker";

export interface MultiListProps {
  readonly type: OperationType;
  readonly par: Params;
  readonly schema: readonly ColumnDef[];
  readonly onChange: (params: Params) => void;
}

const asText = (v: unknown): string => (typeof v === "string" ? v : "");

export function MultiList(props: MultiListProps) {
  const { type, par, schema, onChange } = props;
  const def = MULTI_DEFS[type];
  const multi = multiOf(type, par);
  // una riga aperta per volta in ciascuna lista; all'inizio la prima
  const [open, setOpen] = useState<Record<string, number>>({});
  if (!def) return null;
  const names = schema.map((c) => c.name);

  return (
    <>
      {(def.globals ?? []).map((f) => (
        <Field key={f.k} label={f.label}>
          {(labelId) => (
            <StyledSelect
              labelledBy={labelId}
              value={asText(multi[f.k]) || f.def}
              options={(f.opts ?? []).map((o) => ({ value: o, label: o }))}
              onChange={(v) => onChange(withGlobal(type, par, f.k, v))}
            />
          )}
        </Field>
      ))}
      {def.lists.map((list) => {
        const rows = rowsOf(multi, list.key);
        const current = open[list.key] ?? 0;
        return (
          <section key={list.key} className="ei-list" data-list={list.key}>
            <div className="ei-label">{list.label}</div>
            {rows.map((row, i) => (
              <RowView
                key={i}
                type={type}
                list={list}
                row={row}
                index={i}
                count={rows.length}
                isOpen={i === current}
                schema={schema}
                names={names}
                onToggle={() => setOpen({ ...open, [list.key]: i === current ? -1 : i })}
                onField={(k, v) => onChange(withRowField(type, par, list.key, i, k, v))}
                onRemove={() => {
                  onChange(withRowRemoved(type, par, list.key, i));
                  if (current >= rows.length - 1) setOpen({ ...open, [list.key]: rows.length - 2 });
                }}
              />
            ))}
            <button
              type="button"
              className="ei-addrow"
              onClick={() => {
                onChange(withRowAdded(type, par, list));
                setOpen({ ...open, [list.key]: rows.length });
              }}
            >
              + {list.add}
            </button>
            {list.note && rows.length > 1 ? <div className="ei-help">{list.note}</div> : null}
          </section>
        );
      })}
    </>
  );
}

function RowView(props: {
  type: OperationType;
  list: MultiListDef;
  row: MultiRow;
  index: number;
  count: number;
  isOpen: boolean;
  schema: readonly ColumnDef[];
  names: readonly string[];
  onToggle: () => void;
  onField: (key: string, value: MultiRow[string]) => void;
  onRemove: () => void;
}) {
  const { list, row, index, isOpen } = props;
  const bodyId = useId();
  const sum = list.sum(row);
  return (
    <div className={"ei-row" + (isOpen ? " ei-open" : "")} data-row={index}>
      <div className="ei-row-head">
        <button
          type="button"
          className="ei-row-toggle"
          aria-expanded={isOpen}
          aria-controls={isOpen ? bodyId : undefined}
          onClick={props.onToggle}
        >
          <span className="ei-row-chev">
            <ChevronRight />
          </span>
          <span className="ei-row-n">
            {list.noun} {index + 1}
          </span>
          <span className={"ei-row-sum" + (sum ? "" : " ei-todo")} title={sum ?? copy.rowTodo}>
            {sum ?? copy.rowTodo}
          </span>
        </button>
        {props.count > 1 ? (
          <button
            type="button"
            className="ei-icon-btn"
            aria-label={copy.rowRemove}
            onClick={props.onRemove}
          >
            <XIcon />
          </button>
        ) : null}
      </div>
      {isOpen ? (
        <div id={bodyId} className="ei-row-body">
          {list.fields.map((f) => (
            <RowField key={f.k} {...props} f={f} />
          ))}
        </div>
      ) : null}
    </div>
  );
}

function RowField(props: {
  type: OperationType;
  list: MultiListDef;
  row: MultiRow;
  schema: readonly ColumnDef[];
  names: readonly string[];
  onField: (key: string, value: MultiRow[string]) => void;
  f: MultiFieldDef;
}) {
  const { list, row, schema, names, onField, f } = props;
  const columns = rowColumns(row);
  const domain = columnsDomain(schema, columns);
  const inputMode = NUMERIC_KEYS.has(f.k) ? "numeric" : "text";
  return (
    <Field label={f.label}>
      {(labelId) => {
        switch (f.type) {
          case "columns":
            return (
              <ColumnPicker
                labelledBy={labelId}
                value={columns}
                schema={schema}
                onChange={(next) => onField(f.k, next)}
              />
            );
          case "column":
            return (
              <StyledSelect
                labelledBy={labelId}
                allowFree
                value={asText(row[f.k])}
                options={names.map((n) => ({ value: n, label: n }))}
                onChange={(v) => onField(f.k, v)}
              />
            );
          case "select":
            return (
              <StyledSelect
                labelledBy={labelId}
                value={asText(row[f.k]) || (typeof f.def === "string" ? f.def : "")}
                options={(f.opts ?? []).map((o) => ({ value: o, label: o }))}
                onChange={(v) => onField(f.k, v)}
              />
            );
          case "values":
            return (
              <ValuePicker
                labelledBy={labelId}
                value={(row[f.k] as ValuesField | undefined) ?? createValuesField()}
                domain={domain}
                onChange={(next) => onField(f.k, next)}
              />
            );
          case "value":
            // il valore è unico per riga: con una sola colonna si propone l'elenco dei suoi valori, con più colonne si scrive
            return columns.length === 1 && domain.length > 0 ? (
              <StyledSelect
                labelledBy={labelId}
                allowFree
                value={asText(row[f.k])}
                options={domain.map((v) => ({ value: v, label: v }))}
                onChange={(v) => onField(f.k, v)}
              />
            ) : (
              <TextField
                labelledBy={labelId}
                value={asText(row[f.k])}
                onChange={(v) => onField(f.k, v)}
              />
            );
          default:
            if (list.key === "measures" && f.k === "alias" && columns.length > 1) {
              // con più colonne il nome del risultato è automatico: campo disattivato con l'anteprima
              return (
                <TextField
                  labelledBy={labelId}
                  disabled
                  value={measureNames(row).join(", ")}
                  ariaLabel={copy.resultNameAuto(measureNames(row))}
                  onChange={() => {}}
                />
              );
            }
            return (
              <TextField
                labelledBy={labelId}
                inputMode={inputMode}
                value={asText(row[f.k])}
                onChange={(v) => onField(f.k, v)}
              />
            );
        }
      }}
    </Field>
  );
}
```

### `src/etl-canvas/inspector/NameInput.tsx`

43 righe

```tsx
/**
 * Il nome di un nodo, modificabile in linea. Ogni carattere scritto è un comando
 * `renameNode` (un solo passo di annullamento: chiave di raggruppamento esistente);
 * il campo non perde mai focus né cursore e un nome vuoto non si applica.
 */
import { useState } from "react";
import type { Card } from "../../etl-core";
import type { EtlStore } from "../../etl-store";
import { copy } from "./copy";

export function NameInput(props: {
  readonly store: EtlStore;
  readonly card: Card;
  readonly className: string;
  readonly testId?: string;
  readonly id?: string;
}) {
  const { store, card } = props;
  const [draft, setDraft] = useState<string | null>(null);
  return (
    <input
      type="text"
      id={props.id}
      className={props.className}
      data-testid={props.testId}
      aria-label={copy.nameLabel}
      autoComplete="off"
      spellCheck={false}
      value={draft ?? card.name}
      onChange={(e) => {
        setDraft(e.target.value);
        if (e.target.value.trim()) {
          store.dispatch({ type: "renameNode", payload: { node: card.id, name: e.target.value } });
        }
      }}
      onBlur={() => setDraft(null)}
      onKeyDown={(e) => {
        if (e.key === "Enter") e.currentTarget.blur();
      }}
    />
  );
}
```

### `src/etl-canvas/inspector/StepList.tsx`

267 righe

```tsx
/**
 * L'elenco verticale dei passaggi di un box combinato, riordinabile con il
 * puntatore e da tastiera (Alt+↑/↓). Nell'Inspector ogni passaggio ha lo sgancio
 * e l'eliminazione; nel pannello espanso ha un menu («Configura parametri»,
 * «Sgancia», «Elimina passaggio») e trascinarlo fuori dal pannello lo sgancia.
 * Comandi di etl-store: `reorderSteps`, `deleteStep`, `detachStep`.
 */
import { useEffect, useRef, useState } from "react";
import type { KeyboardEvent, PointerEvent, RefObject } from "react";
import { META } from "../../etl-core";
import type { Card } from "../../etl-core";
import type { EtlStore } from "../../etl-store";
import { Icon } from "../icons";
import { ActionMenu } from "./ActionMenu";
import { copy } from "./copy";
import { DetachIcon, GripIcon, MoreIcon, XIcon } from "./icons";
import { moveTarget, reorderIndex } from "./logic";

export interface StepListProps {
  readonly store: EtlStore;
  readonly card: Card;
  readonly selectedStep: number;
  readonly variant: "inspector" | "expanded";
  /** Un passaggio scelto (clic o Invio). */
  readonly onSelect: (index: number) => void;
  /** Solo nel pannello espanso: «Configura parametri». */
  readonly onConfigure?: (index: number) => void;
  /** Solo nel pannello espanso: il riquadro del pannello, per sapere se si è usciti. */
  readonly containerRef?: RefObject<HTMLElement | null>;
  /** Solo nel pannello espanso: rilasciato fuori dal pannello (coordinate della finestra). */
  readonly onDetachOutside?: (index: number, clientX: number, clientY: number) => void;
}

interface Drag {
  readonly index: number;
  readonly to: number;
  readonly dy: number;
  readonly dx: number;
  readonly rowHeight: number;
  readonly outside: boolean;
}

/** Soglia prima che un clic su una riga diventi un trascinamento (prototipo: 4 px). */
const DRAG_START_PX = 4;
/** Quanto fuori dal pannello conta come «fuori» (prototipo, riga 2297). */
const OUTSIDE_MARGIN_PX = 12;

export function StepList(props: StepListProps) {
  const { store, card, selectedStep, variant } = props;
  const listRef = useRef<HTMLUListElement>(null);
  const mainRefs = useRef<(HTMLButtonElement | null)[]>([]);
  const focusIndex = useRef<number | null>(null);
  const justDragged = useRef(false);
  const [drag, setDrag] = useState<Drag | null>(null);
  const [announce, setAnnounce] = useState("");
  const [menuFor, setMenuFor] = useState<number | null>(null);
  const gearRefs = useRef<(HTMLButtonElement | null)[]>([]);
  const n = card.components.length;

  // dopo un riordino da tastiera il focus segue il passaggio spostato
  useEffect(() => {
    if (focusIndex.current !== null) {
      mainRefs.current[focusIndex.current]?.focus();
      focusIndex.current = null;
    }
  });

  const reorder = (from: number, to: number) =>
    store.dispatch({ type: "reorderSteps", payload: { box: card.id, from, to } });
  const remove = (index: number) =>
    store.dispatch({ type: "deleteStep", payload: { box: card.id, index } });
  const detach = (index: number) =>
    store.dispatch({ type: "detachStep", payload: { box: card.id, index } });

  const onKey = (e: KeyboardEvent<HTMLButtonElement>, i: number) => {
    if (e.altKey && (e.key === "ArrowUp" || e.key === "ArrowDown")) {
      e.preventDefault();
      const to = moveTarget(i, n, e.key);
      if (to === null) return;
      focusIndex.current = to;
      reorder(i, to);
      setAnnounce(
        copy.announceMoved(META[card.components[i] as keyof typeof META].label, to + 1, n),
      );
    }
  };

  const onPointerDown = (e: PointerEvent<HTMLLIElement>, i: number) => {
    if (e.button !== 0) return;
    if ((e.target as HTMLElement).closest(".ei-step-btn")) return; // i pulsanti non avviano il trascinamento
    const rows = Array.from(listRef.current?.children ?? []) as HTMLElement[];
    const rects = rows.map((r) => r.getBoundingClientRect());
    const rowHeight =
      rects.length > 1
        ? (rects[1] as DOMRect).top - (rects[0] as DOMRect).top
        : (rects[0]?.height ?? 40);
    const panel = props.containerRef?.current?.getBoundingClientRect();
    const sx = e.clientX;
    const sy = e.clientY;
    let moved = false;
    let last: Drag | null = null;
    const move = (ev: globalThis.PointerEvent) => {
      const dx = ev.clientX - sx;
      const dy = ev.clientY - sy;
      if (!moved && Math.hypot(dx, dy) < DRAG_START_PX) return;
      moved = true;
      const outside =
        variant === "expanded" &&
        !!panel &&
        (ev.clientX < panel.left - OUTSIDE_MARGIN_PX ||
          ev.clientX > panel.right + OUTSIDE_MARGIN_PX ||
          ev.clientY < panel.top - OUTSIDE_MARGIN_PX ||
          ev.clientY > panel.bottom + OUTSIDE_MARGIN_PX);
      last = { index: i, to: reorderIndex(i, dy, rowHeight, n), dy, dx, rowHeight, outside };
      setDrag(last);
    };
    const up = (ev: globalThis.PointerEvent) => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", up);
      window.removeEventListener("pointercancel", up);
      setDrag(null);
      if (!moved || !last) return;
      justDragged.current = true;
      setTimeout(() => (justDragged.current = false), 0);
      if (last.outside) props.onDetachOutside?.(i, ev.clientX, ev.clientY);
      else if (last.to !== i) {
        focusIndex.current = last.to;
        reorder(i, last.to);
      }
    };
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", up);
    window.addEventListener("pointercancel", up);
  };

  /** Spostamento verticale delle altre righe mentre una viene trascinata (come nel prototipo). */
  const shiftOf = (i: number): number => {
    if (!drag || drag.outside || i === drag.index) return 0;
    if (drag.index < drag.to && i > drag.index && i <= drag.to) return -drag.rowHeight;
    if (drag.index > drag.to && i >= drag.to && i < drag.index) return drag.rowHeight;
    return 0;
  };

  return (
    <div className="ei-steps" data-variant={variant}>
      <ul ref={listRef} className="ei-steplist" aria-label={copy.sequenceTitle}>
        {card.components.map((type, i) => {
          const dragging = drag?.index === i;
          const label = META[type].label;
          const style = dragging
            ? {
                transform: drag?.outside
                  ? `translate(${drag.dx}px, ${drag.dy}px) scale(0.9)`
                  : `translateY(${drag?.dy ?? 0}px)`,
              }
            : { transform: `translateY(${shiftOf(i)}px)` };
          return (
            <li
              key={`${i}:${type}`}
              className={
                "ei-step" +
                (i === selectedStep ? " ei-on" : "") +
                (dragging ? " ei-dragging" : "") +
                (dragging && drag?.outside ? " ei-outside" : "")
              }
              style={style}
              data-step={i}
              onPointerDown={(e) => onPointerDown(e, i)}
            >
              <button
                ref={(el) => {
                  mainRefs.current[i] = el;
                }}
                type="button"
                className="ei-step-main"
                aria-current={i === selectedStep ? "step" : undefined}
                aria-label={copy.stepLabel(i + 1, label)}
                title={copy.stepReorderHelp}
                onKeyDown={(e) => onKey(e, i)}
                onClick={() => {
                  if (!justDragged.current) props.onSelect(i);
                }}
              >
                <span className="ei-grip">
                  <GripIcon />
                </span>
                <span className="ei-step-n">{i + 1}</span>
                <span className="ei-step-icon">
                  <Icon id={type} />
                </span>
                <span className="ei-step-name">{label}</span>
              </button>
              {variant === "inspector" ? (
                <>
                  <button
                    type="button"
                    className="ei-icon-btn ei-step-btn"
                    aria-label={copy.stepDetach}
                    title={copy.stepDetach}
                    onClick={() => detach(i)}
                  >
                    <DetachIcon />
                  </button>
                  <button
                    type="button"
                    className="ei-icon-btn ei-step-btn"
                    aria-label={copy.stepDelete}
                    title={copy.stepDelete}
                    onClick={() => remove(i)}
                  >
                    <XIcon />
                  </button>
                </>
              ) : (
                <>
                  <button
                    ref={(el) => {
                      gearRefs.current[i] = el;
                    }}
                    type="button"
                    className="ei-icon-btn ei-step-btn"
                    aria-label={copy.stepMenu}
                    aria-haspopup="menu"
                    aria-expanded={menuFor === i}
                    onClick={() => setMenuFor(menuFor === i ? null : i)}
                  >
                    <MoreIcon />
                  </button>
                  {menuFor === i ? (
                    <ActionMenu
                      anchor={{ current: gearRefs.current[i] ?? null }}
                      onClose={(back) => {
                        setMenuFor(null);
                        if (back) gearRefs.current[i]?.focus();
                      }}
                      items={[
                        {
                          id: "configure",
                          label: copy.stepConfigure,
                          onSelect: () => props.onConfigure?.(i),
                        },
                        { id: "detach", label: copy.stepDetach, onSelect: () => detach(i) },
                        {
                          id: "delete",
                          label: copy.stepDelete,
                          danger: true,
                          onSelect: () => remove(i),
                        },
                      ]}
                    />
                  ) : null}
                </>
              )}
            </li>
          );
        })}
      </ul>
      {drag?.outside ? (
        <div className="ei-help ei-outside-note">{copy.expandedNoteOutside}</div>
      ) : null}
      <div className="ei-visually-hidden" role="status" aria-live="polite">
        {announce}
      </div>
    </div>
  );
}
```

