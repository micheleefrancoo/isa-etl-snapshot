# 01e-etl-canvas-g.md

File in questo blocco:

- `src/etl-canvas/inspector/MultiList.tsx`
- `src/etl-canvas/inspector/NameInput.tsx`
- `src/etl-canvas/inspector/StepList.tsx`
- `src/etl-canvas/inspector/StyledSelect.tsx`
- `src/etl-canvas/inspector/ValuePicker.tsx`
- `src/etl-canvas/inspector/copy.ts`
- `src/etl-canvas/inspector/family.ts`
- `src/etl-canvas/inspector/icons.tsx`

---

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

### `src/etl-canvas/inspector/StyledSelect.tsx`

212 righe

```tsx
/**
 * Scelta singola con ricerca (sostituisce `<select>` e `<datalist>`): un campo
 * che apre il nostro menu in un portale. Il menu ha un campo di ricerca che è
 * il combobox ARIA (frecce, Home, Fine, Invio, Esc, digitazione) e un elenco
 * listbox; con `allowFree` si può anche scrivere un valore non presente
 * («oppure scrivi», in corsivo). Il focus torna al campo alla chiusura.
 */
import { useCallback, useEffect, useId, useMemo, useRef, useState } from "react";
import type { KeyboardEvent } from "react";
import { copy } from "./copy";
import { ChevronDown } from "./icons";
import { clampActive, comboAction, filterByQuery, fold } from "./logic";
import { Menu } from "./Menu";

export interface SelectOption {
  readonly value: string;
  readonly label: string;
  /** Testo discreto a destra (per esempio il tipo di una colonna). */
  readonly hint?: string;
}

export interface StyledSelectProps {
  readonly value: string;
  readonly options: readonly SelectOption[];
  readonly onChange: (value: string) => void;
  readonly labelledBy?: string | undefined;
  readonly ariaLabel?: string | undefined;
  /** Si può scrivere un valore non presente nell'elenco. */
  readonly allowFree?: boolean;
  readonly placeholder?: string;
  readonly disabled?: boolean;
}

interface Row {
  readonly value: string;
  readonly label: string;
  readonly hint?: string | undefined;
  readonly free?: boolean;
}

export function StyledSelect(props: StyledSelectProps) {
  const { value, options, onChange, allowFree, disabled } = props;
  const id = useId();
  const listId = `${id}-list`;
  const triggerRef = useRef<HTMLButtonElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);
  const listRef = useRef<HTMLDivElement>(null);
  const [open, setOpen] = useState(false);
  const [query, setQuery] = useState("");
  const [active, setActive] = useState(-1);

  const rows = useMemo<Row[]>(() => {
    const found: Row[] = filterByQuery(options, query).map((o) => ({
      value: o.value,
      label: o.label,
      hint: o.hint,
    }));
    const typed = query.trim();
    const exact = options.some((o) => fold(o.label) === fold(typed) || o.value === typed);
    if (allowFree && typed && !exact) {
      found.push({ value: typed, label: copy.useTyped(typed), free: true });
    }
    return found;
  }, [options, query, allowFree]);
  const act = clampActive(active, rows.length);

  const close = useCallback((returnFocus: boolean) => {
    setOpen(false);
    setQuery("");
    if (returnFocus) triggerRef.current?.focus();
  }, []);
  const openWith = (seed: string) => {
    if (disabled) return;
    setQuery(seed);
    const i = options.findIndex((o) => o.value === value);
    setActive(seed ? 0 : i);
    setOpen(true);
  };
  const choose = (v: string) => {
    onChange(v);
    close(true);
  };

  // il focus passa al campo di ricerca; la voce attiva resta in vista
  useEffect(() => {
    if (open) inputRef.current?.focus();
  }, [open]);
  useEffect(() => {
    if (!open || act < 0) return;
    listRef.current?.querySelector(`[data-index="${act}"]`)?.scrollIntoView({ block: "nearest" });
  }, [open, act, rows.length]);

  const onTriggerKey = (e: KeyboardEvent<HTMLButtonElement>) => {
    if (e.key === "ArrowDown" || e.key === "ArrowUp") {
      e.preventDefault();
      openWith("");
    } else if (e.key.length === 1 && !e.ctrlKey && !e.metaKey && !e.altKey && e.key !== " ") {
      e.preventDefault();
      openWith(e.key);
    }
  };
  const onInputKey = (e: KeyboardEvent<HTMLInputElement>) => {
    const a = comboAction(e.key, act, rows.length);
    if (a.kind === "move") {
      e.preventDefault();
      setActive(a.to);
    } else if (a.kind === "commit") {
      e.preventDefault();
      const row = rows[act];
      if (row) choose(row.value);
      else if (allowFree && query.trim()) choose(query.trim());
    } else if (a.kind === "close") {
      e.preventDefault();
      e.stopPropagation();
      close(true);
    } else if (e.key === "Tab") {
      close(false);
    }
  };

  const current = options.find((o) => o.value === value);
  const shown = current?.label ?? value;
  const isFree = value !== "" && !current;

  return (
    <>
      <button
        ref={triggerRef}
        type="button"
        className="ei-field ei-select"
        data-picker="select"
        aria-haspopup="listbox"
        aria-expanded={open}
        aria-controls={open ? listId : undefined}
        aria-labelledby={props.labelledBy}
        aria-label={props.labelledBy ? undefined : props.ariaLabel}
        disabled={disabled}
        data-open={open || undefined}
        onClick={() => (open ? close(false) : openWith(""))}
        onKeyDown={onTriggerKey}
      >
        <span className={"ei-select-value" + (isFree ? " ei-free" : "")}>
          {shown || <span className="ei-placeholder">{props.placeholder ?? copy.pickOrType}</span>}
        </span>
        <span className="ei-select-chevron">
          <ChevronDown />
        </span>
      </button>
      {open ? (
        <Menu anchor={triggerRef} onClose={() => close(false)} onEscape={() => close(true)}>
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
              aria-label={copy.searchPlaceholder}
              placeholder={copy.searchPlaceholder}
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
            aria-label={copy.menuLabel}
            data-scroll=""
          >
            {rows.length === 0 ? <div className="ei-menu-empty">{copy.noResults}</div> : null}
            {rows.map((r, i) => (
              <div
                key={`${r.free ? "free:" : ""}${r.value}`}
                id={`${id}-opt-${i}`}
                data-index={i}
                role="option"
                aria-selected={!r.free && r.value === value}
                className={
                  "ei-option" +
                  (i === act ? " ei-active" : "") +
                  (!r.free && r.value === value ? " ei-selected" : "") +
                  (r.free ? " ei-free" : "")
                }
                onPointerDown={(e) => e.preventDefault()}
                onPointerMove={() => setActive(i)}
                onClick={() => choose(r.value)}
              >
                <span className="ei-option-label">{r.label}</span>
                {r.hint ? <span className="ei-option-hint">{r.hint}</span> : null}
              </div>
            ))}
          </div>
          {allowFree && !query.trim() ? (
            <div className="ei-menu-foot ei-help">{copy.typeOr}</div>
          ) : null}
        </Menu>
      ) : null}
    </>
  );
}
```

### `src/etl-canvas/inspector/ValuePicker.tsx`

293 righe

```tsx
/**
 * Selettore di valori (versione finale del prototipo, `pickerHtml`): i valori
 * scelti sono etichette rimovibili (in corsivo quelli assenti dai dati); il
 * menu (in un portale) ha ricerca che filtra, «+ Aggiungi “x”» per ciò che non
 * esiste (Invio), incolla di più valori (virgola, punto e virgola, barra
 * verticale, a capo), elenco a spunta con scorrimento (max 170 px), «Tutti» e
 * «Nessuno» sui soli valori visibili e il conteggio annunciato.
 *
 * L'elenco proposto è `columnsDomain` delle colonne della riga (lo calcola chi
 * lo usa). Cambiando le colonne i valori NON si azzerano: quelli fuori dominio
 * restano in corsivo, con l'avviso e l'azione «Rimuovi». I valori stanno solo
 * in `values`.
 */
import { useEffect, useId, useMemo, useRef, useState } from "react";
import type { ClipboardEvent, KeyboardEvent } from "react";
import { valuesOutsideDomain } from "../../etl-core";
import type { ValuesField } from "../../etl-core";
import { copy } from "./copy";
import { CheckIcon, PlusIcon, XIcon } from "./icons";
import {
  addTokens,
  addVisibleValues,
  clampActive,
  comboAction,
  filterByQuery,
  pendingTokens,
  removeVisibleValues,
  toggleValue,
  withValues,
} from "./logic";
import { Menu } from "./Menu";

export interface ValuePickerProps {
  readonly value: ValuesField | undefined;
  readonly domain: readonly string[];
  readonly onChange: (field: ValuesField) => void;
  readonly labelledBy?: string | undefined;
  readonly ariaLabel?: string | undefined;
}

type Row =
  | { readonly kind: "add"; readonly tokens: readonly string[]; readonly label: string }
  | {
      readonly kind: "value";
      readonly value: string;
      readonly label: string;
      readonly free: boolean;
    };

const NO_VALUES: readonly string[] = [];
const SEPARATORS = /[,;|\n]/;

export function ValuePicker(props: ValuePickerProps) {
  const { value: field, domain, onChange } = props;
  const values = useMemo(() => field?.values ?? NO_VALUES, [field]);
  const id = useId();
  const listId = `${id}-list`;
  const fieldRef = useRef<HTMLDivElement>(null);
  const addRef = useRef<HTMLButtonElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);
  const listRef = useRef<HTMLDivElement>(null);
  const [open, setOpen] = useState(false);
  const [query, setQuery] = useState("");
  const [active, setActive] = useState(0);

  const outside = useMemo(
    () => (domain.length > 0 ? valuesOutsideDomain(values, domain) : []),
    [values, domain],
  );
  const set = (next: string[]) => onChange(withValues(field, next));

  // elenco: i valori dei dati, poi quelli scelti ma assenti dai dati (si possono togliere)
  const rows = useMemo<Row[]>(() => {
    const extra = values.filter((v) => !domain.includes(v));
    const all = [...domain, ...extra].map((v) => ({ label: v, free: !domain.includes(v) }));
    const list: Row[] = filterByQuery(all, query).map((r) => ({
      kind: "value",
      value: r.label,
      label: r.label,
      free: r.free,
    }));
    const tokens = pendingTokens(values, domain, query);
    if (tokens.length) list.unshift({ kind: "add", tokens, label: copy.valuesAddTyped(tokens) });
    return list;
  }, [domain, values, query]);
  const act = clampActive(active, rows.length);
  const visible = rows.flatMap((r) => (r.kind === "value" ? [r.value] : []));

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

  const activate = (r: Row) => {
    if (r.kind === "add") {
      set(addTokens(values, domain, query));
      setQuery("");
      setActive(0);
    } else set(toggleValue(values, r.value));
  };
  const onKey = (e: KeyboardEvent<HTMLInputElement>) => {
    const a = comboAction(e.key, act, rows.length);
    if (a.kind === "move") {
      e.preventDefault();
      setActive(a.to);
    } else if (a.kind === "commit") {
      e.preventDefault();
      const r = rows[act];
      if (r) activate(r);
    } else if (a.kind === "close") {
      e.preventDefault();
      e.stopPropagation();
      close(true);
    }
  };
  // incollando più valori insieme si aggiungono subito, con la grafia dei dati
  const onPaste = (e: ClipboardEvent<HTMLInputElement>) => {
    const text = e.clipboardData.getData("text");
    if (!SEPARATORS.test(text)) return;
    e.preventDefault();
    set(addTokens(values, domain, text));
    setQuery("");
  };

  return (
    <>
      <div
        ref={fieldRef}
        className="ei-field ei-chipsfield"
        data-picker="values"
        data-selected={values.length}
        data-total={domain.length}
        role="group"
        aria-labelledby={props.labelledBy}
        aria-label={props.labelledBy ? undefined : props.ariaLabel}
        data-open={open || undefined}
      >
        {values.length === 0 ? (
          <span className="ei-placeholder">{copy.valuesPlaceholder}</span>
        ) : null}
        <ul className="ei-chips">
          {values.map((v) => {
            const free = domain.length > 0 ? !domain.includes(v) : false;
            return (
              <li
                key={v}
                className={"ei-chip" + (free ? " ei-free" : "")}
                title={free ? copy.valuesFree : undefined}
              >
                <span className="ei-chip-text">{v}</span>
                <button
                  type="button"
                  className="ei-chip-x"
                  aria-label={copy.valuesRemove(v)}
                  onClick={() => set(values.filter((x) => x !== v))}
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
          <span>{copy.valuesAdd}</span>
        </button>
      </div>
      {outside.length > 0 ? (
        <div className="ei-warn" role="status">
          <span>{copy.valuesOutside(outside.length)}</span>
          <button
            type="button"
            className="ei-link-btn"
            onClick={() => set(values.filter((v) => !outside.includes(v)))}
          >
            {copy.valuesOutsideRemove}
          </button>
        </div>
      ) : null}
      {open ? (
        <Menu
          anchor={fieldRef}
          onClose={() => close(false)}
          onEscape={() => close(true)}
          ariaLabel={copy.valuesMenu}
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
              aria-label={domain.length ? copy.valuesSearch : copy.valuesSearchFree}
              placeholder={domain.length ? copy.valuesSearch : copy.valuesSearchFree}
              autoComplete="off"
              spellCheck={false}
              value={query}
              onChange={(e) => {
                setQuery(e.target.value);
                setActive(0);
              }}
              onKeyDown={onKey}
              onPaste={onPaste}
            />
          </div>
          <div
            ref={listRef}
            id={listId}
            className="ei-menu-scroll ei-values-list"
            role="listbox"
            aria-multiselectable="true"
            aria-label={copy.valuesMenu}
            data-scroll=""
          >
            {rows.length === 0 ? <div className="ei-menu-empty">{copy.noResults}</div> : null}
            {rows.map((r, i) => {
              const on = r.kind === "value" && values.includes(r.value);
              return (
                <div
                  key={r.kind === "add" ? "add" : r.value}
                  id={`${id}-opt-${i}`}
                  data-index={i}
                  role="option"
                  aria-selected={on}
                  className={
                    "ei-option" +
                    (i === act ? " ei-active" : "") +
                    (on ? " ei-selected" : "") +
                    (r.kind === "add" || r.free ? " ei-free" : "")
                  }
                  onPointerDown={(e) => e.preventDefault()}
                  onPointerMove={() => setActive(i)}
                  onClick={() => activate(r)}
                >
                  {r.kind === "add" ? null : (
                    <span className="ei-checkbox" data-on={on || undefined} aria-hidden="true">
                      {on ? <CheckIcon /> : null}
                    </span>
                  )}
                  <span className="ei-option-label">{r.label}</span>
                </div>
              );
            })}
          </div>
          <div className="ei-menu-foot">
            <span className="ei-count" role="status" aria-live="polite">
              {copy.valuesCount(values.length, domain.length)}
            </span>
            {domain.length > 0 ? (
              <span className="ei-menu-actions">
                <button
                  type="button"
                  className="ei-link-btn"
                  onClick={() => set(addVisibleValues(values, visible))}
                >
                  {copy.valuesAll}
                </button>
                <button
                  type="button"
                  className="ei-link-btn"
                  onClick={() => set(removeVisibleValues(values, visible))}
                >
                  {copy.valuesNone}
                </button>
              </span>
            ) : null}
          </div>
        </Menu>
      ) : null}
    </>
  );
}
```

### `src/etl-canvas/inspector/copy.ts`

114 righe

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
  conditionsSoon: "Le condizioni arrivano nella prossima fase",
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

