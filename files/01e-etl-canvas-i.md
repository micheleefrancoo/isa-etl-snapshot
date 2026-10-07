# 01e-etl-canvas-i.md

File in questo blocco:

- `src/etl-canvas/inspector/ListRow.tsx`
- `src/etl-canvas/inspector/Menu.tsx`
- `src/etl-canvas/inspector/MultiList.tsx`
- `src/etl-canvas/inspector/NameInput.tsx`
- `src/etl-canvas/inspector/Segmented.tsx`
- `src/etl-canvas/inspector/StepList.tsx`
- `src/etl-canvas/inspector/StyledSelect.tsx`
- `src/etl-canvas/inspector/ValuePicker.tsx`
- `src/etl-canvas/inspector/conditions.ts`

---

### `src/etl-canvas/inspector/ListRow.tsx`

123 righe

```tsx
/**
 * Una riga comprimibile di una lista (una condizione, una chiave di join, una
 * voce di Converti tipo...): intestazione con numero e riassunto dal vivo, il
 * pulsante per toglierla e il corpo con i campi.
 *
 * Nel layout normale (bordi laterali, pannello stretto) la riga si apre e si chiude
 * sul posto, una per volta. Nel layout a tre colonne (`Columns3`) un clic la rende
 * ATTIVA: resta nell'elenco, evidenziata, e il suo titolo e il suo corpo compaiono nel
 * dettaglio (portale); un clic non la comprime.
 */
import { useId } from "react";
import type { CSSProperties, ReactNode } from "react";
import { createPortal } from "react-dom";
import { useMasterDetail } from "./masterDetail";
import { copy } from "./copy";
import { ChevronRight, GripIcon, XIcon } from "./icons";
import type { RowReorder } from "./useReorder";

export interface ListRowProps {
  /** Identificativo della voce (`lista:indice`), per il layout a tre colonne. */
  readonly rowId: string;
  /** «Condizione», «Conversione», ... */
  readonly noun: string;
  /** Posizione, da 0. */
  readonly index: number;
  /** Il riassunto dal vivo, o `null` se la riga è da configurare. */
  readonly summary: string | null;
  /** Aperta sul posto (solo nel layout normale). */
  readonly open: boolean;
  readonly onToggle: () => void;
  /** Senza, il pulsante per togliere la riga non compare (di solito: con una sola riga). */
  readonly onRemove?: (() => void) | undefined;
  readonly removeLabel?: string;
  /** Per riportare il focus su questa riga dopo un'azione che la ridisegna (`data-focus`). */
  readonly focusKey?: string;
  /** Riordino con la maniglia (solo nelle liste in cui l'ordine conta). */
  readonly reorder?: RowReorder | undefined;
  /** Il nome da dire per la maniglia («Criterio 2»). */
  readonly gripLabel?: string;
  /** I campi della riga. */
  readonly children: ReactNode;
}

export function ListRow(props: ListRowProps) {
  const { rowId, noun, index, summary } = props;
  const md = useMasterDetail();
  const bodyId = useId();
  const active = md ? md.active === rowId : props.open;
  const label = `${noun} ${index + 1}`;
  const sum = summary ?? copy.rowTodo;

  return (
    <div
      className={
        "ei-row" + (active ? " ei-open" : "") + (props.reorder?.dragging ? " ei-dragging" : "")
      }
      data-row={index}
      data-active={md && active ? "" : undefined}
      style={props.reorder?.style as CSSProperties | undefined}
    >
      <div className="ei-row-head">
        {props.reorder ? (
          <button
            type="button"
            className="ei-icon-btn ei-row-grip"
            aria-label={copy.rowGrip(props.gripLabel ?? label)}
            aria-keyshortcuts={copy.rowGripKeys}
            title={copy.rowReorderHelp}
            {...props.reorder.gripProps}
          >
            <GripIcon />
          </button>
        ) : null}
        <button
          type="button"
          className="ei-row-toggle"
          data-focus={props.focusKey}
          aria-expanded={md ? undefined : active}
          aria-current={md && active ? "true" : undefined}
          aria-controls={!md && active ? bodyId : undefined}
          onClick={() => (md ? md.setActive(rowId) : props.onToggle())}
        >
          <span className="ei-row-chev">
            <ChevronRight />
          </span>
          <span className="ei-row-n">{label}</span>
          <span className={"ei-row-sum" + (summary ? "" : " ei-todo")} title={sum}>
            {sum}
          </span>
        </button>
        {props.onRemove ? (
          <button
            type="button"
            className="ei-icon-btn"
            aria-label={props.removeLabel ?? copy.rowRemove}
            onClick={props.onRemove}
          >
            <XIcon />
          </button>
        ) : null}
      </div>
      {md ? (
        active && md.head && md.body ? (
          <>
            {createPortal(
              <>
                <div className="ei-md-title">{label}</div>
                <div className={"ei-md-sum" + (summary ? "" : " ei-todo")}>{sum}</div>
              </>,
              md.head,
            )}
            {createPortal(<div className="ei-md-fields">{props.children}</div>, md.body)}
          </>
        ) : null
      ) : active ? (
        <div id={bodyId} className="ei-row-body">
          {props.children}
        </div>
      ) : null}
    </div>
  );
}
```

### `src/etl-canvas/inspector/Menu.tsx`

178 righe

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

/** Il campo è ancora visibile: dentro la finestra e dentro ogni contenitore che lo ritaglia o lo fa scorrere. */
function isInView(el: HTMLElement): boolean {
  const r = el.getBoundingClientRect();
  if (r.bottom <= 0 || r.top >= window.innerHeight || r.right <= 0 || r.left >= window.innerWidth) return false;
  for (let p = el.parentElement; p; p = p.parentElement) {
    const o = getComputedStyle(p);
    if (!/(auto|scroll|hidden|clip)/.test(o.overflowY + o.overflowX)) continue;
    const b = p.getBoundingClientRect();
    if (r.bottom <= b.top || r.top >= b.bottom || r.right <= b.left || r.left >= b.right) return false;
  }
  return true;
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
      // il pannello può scorrere da sé (un valore aggiunto, il focus che porta in vista un campo):
      // il menu segue il campo e si chiude solo quando il campo esce dalla vista
      const a = anchor.current;
      if (a && isInView(a)) place();
      else onClose();
    };
    document.addEventListener("pointerdown", away, true);
    window.addEventListener("scroll", scrolled, true);
    window.addEventListener("resize", onClose);
    return () => {
      document.removeEventListener("pointerdown", away, true);
      window.removeEventListener("scroll", scrolled, true);
      window.removeEventListener("resize", onClose);
    };
  }, [anchor, inside, onClose, place]);

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

335 righe

```tsx
/**
 * Le operazioni a voci multiple del catalogo (`MULTI_DEFS`): campi globali e
 * liste di righe comprimibili (una aperta per volta), con il riassunto di riga
 * dal vivo (`L.sum`), aggiunta e rimozione di righe e note. Ogni campo di tipo
 * «columns» usa il ColumnPicker; i campi a colonna singola, le scelte e i
 * valori d'una sola colonna usano StyledSelect; i valori il ValuePicker.
 * Il riordino dei criteri di Ordina non è di questa fase.
 */
import { useRef, useState } from "react";
import {
  MULTI_DEFS,
  columnsDomain,
  columnsOutsideSchema,
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
import { rowIdOf, useActiveGuard, useMasterDetail } from "./masterDetail";
import { Field, TextField } from "./Field";
import { ListRow } from "./ListRow";
import { indexAfterMove } from "./logic";
import { useReorder } from "./useReorder";
import type { RowReorder } from "./useReorder";
import {
  NUMERIC_KEYS,
  REORDERABLE_LISTS,
  multiOf,
  rowColumns,
  rowsOf,
  withGlobal,
  withRowAdded,
  withRowField,
  withRowMoved,
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

/** I campi globali di un'operazione a voci (per esempio «Modo» di Seleziona colonne): stanno nelle Impostazioni. */
export function MultiGlobals(props: MultiListProps) {
  const { type, par, onChange } = props;
  const def = MULTI_DEFS[type];
  if (!def) return null;
  const multi = multiOf(type, par);
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
    </>
  );
}

/** Le liste di righe (una aperta per volta; nel layout a tre colonne, una attiva nel dettaglio). */
export function MultiRows(props: MultiListProps) {
  const def = MULTI_DEFS[props.type];
  if (!def) return null;
  return (
    <>
      {def.lists.map((list) => (
        <ListSection key={list.key} {...props} list={list} />
      ))}
    </>
  );
}

/** Una lista di righe: aggiungi, rimuovi e, dove l'ordine conta, riordina con la maniglia. */
function ListSection(props: MultiListProps & { readonly list: MultiListDef }) {
  const { type, par, schema, list, onChange } = props;
  const multi = multiOf(type, par);
  const md = useMasterDetail();
  const rows = rowsOf(multi, list.key);
  const [current, setCurrent] = useState(0);
  const sectionRef = useRef<HTMLElement>(null);
  const names = schema.map((c) => c.name);
  const reorderable = REORDERABLE_LISTS[type] === list.key && rows.length > 1;
  const prefix = `${list.key}:`;
  const activeIndex =
    md && md.active.startsWith(prefix) ? Number(md.active.slice(prefix.length)) : current;
  useActiveGuard(list.key, rows.length);

  const { row: reorderOf, announcement } = useReorder({
    containerRef: sectionRef,
    count: rows.length,
    onMove: (from, to) => {
      onChange(withRowMoved(type, par, list.key, from, to));
      // la voce aperta o attiva segue la riga spostata
      setCurrent(indexAfterMove(current, from, to));
      if (md && md.active.startsWith(prefix)) {
        md.setActive(rowIdOf(list.key, indexAfterMove(activeIndex, from, to)));
      }
    },
    describe: (i) => list.sum(rows[i] ?? {}) ?? `${list.noun} ${i + 1}`,
    announce: copy.announceMoved,
  });

  return (
    <section ref={sectionRef} className="ei-list" data-list={list.key}>
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
          reorder={reorderable ? reorderOf(i) : undefined}
          onToggle={() => setCurrent(i === current ? -1 : i)}
          onField={(k, v) => onChange(withRowField(type, par, list.key, i, k, v))}
          onRemove={() => {
            onChange(withRowRemoved(type, par, list.key, i));
            if (current >= rows.length - 1) setCurrent(rows.length - 2);
            if (md && md.active === rowIdOf(list.key, i)) {
              md.setActive(rowIdOf(list.key, Math.max(0, Math.min(i, rows.length - 2))));
            }
          }}
        />
      ))}
      <button
        type="button"
        className="ei-addrow"
        onClick={() => {
          onChange(withRowAdded(type, par, list));
          setCurrent(rows.length);
          md?.setActive(rowIdOf(list.key, rows.length));
        }}
      >
        + {list.add}
      </button>
      {list.note && rows.length > 1 ? <div className="ei-help">{list.note}</div> : null}
      {reorderable ? (
        <div className="ei-visually-hidden" role="status" aria-live="polite">
          {announcement}
        </div>
      ) : null}
    </section>
  );
}

/** Globali e righe insieme: il layout normale (bordi laterali). */
export function MultiList(props: MultiListProps) {
  return (
    <>
      <MultiGlobals {...props} />
      <MultiRows {...props} />
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
  reorder: RowReorder | undefined;
  onField: (key: string, value: MultiRow[string]) => void;
  onRemove: () => void;
}) {
  const { list, row, index } = props;
  return (
    <ListRow
      reorder={props.reorder}
      rowId={rowIdOf(list.key, index)}
      noun={list.noun}
      index={index}
      summary={list.sum(row)}
      open={props.isOpen}
      onToggle={props.onToggle}
      onRemove={props.count > 1 ? props.onRemove : undefined}
    >
      {list.fields.map((f) => (
        <RowField key={f.k} {...props} f={f} />
      ))}
    </ListRow>
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
          case "columns": {
            // colonne che non sono più nei dati in ingresso: si segnalano, non si tolgono da sole
            const outside = columnsOutsideSchema(columns, schema);
            return (
              <>
                <ColumnPicker
                  labelledBy={labelId}
                  value={columns}
                  schema={schema}
                  onChange={(next) => onField(f.k, next)}
                />
                {outside.length > 0 ? (
                  <div className="ei-warn" role="status" data-testid="ei-columns-outside">
                    <span>{copy.columnsOutside(outside.length)}</span>
                    <button
                      type="button"
                      className="ei-link-btn"
                      onClick={() =>
                        onField(
                          f.k,
                          columns.filter((c) => !outside.includes(c)),
                        )
                      }
                    >
                      {copy.columnsOutsideRemove}
                    </button>
                  </div>
                ) : null}
              </>
            );
          }
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

### `src/etl-canvas/inspector/Segmented.tsx`

72 righe

```tsx
/**
 * Scelta tra poche voci affiancate («Colonna | Valore | Lista»): un radiogroup
 * ARIA, non radio nativi. Una sola voce ha il tabindex 0 (la scelta); le frecce,
 * Home e Fine spostano la scelta e il focus insieme, con giro; Spazio e Invio
 * scelgono la voce a fuoco.
 */
import { useRef } from "react";
import type { KeyboardEvent } from "react";
import { segmentedKey } from "./logic";

export interface SegmentedOption<V extends string> {
  readonly value: V;
  readonly label: string;
}

export interface SegmentedProps<V extends string> {
  readonly value: V;
  readonly options: readonly SegmentedOption<V>[];
  readonly onChange: (value: V) => void;
  readonly labelledBy?: string | undefined;
  readonly ariaLabel?: string | undefined;
}

export function Segmented<V extends string>(props: SegmentedProps<V>) {
  const { value, options, onChange } = props;
  const refs = useRef<(HTMLButtonElement | null)[]>([]);
  const current = Math.max(
    0,
    options.findIndex((o) => o.value === value),
  );

  const onKey = (e: KeyboardEvent<HTMLButtonElement>, index: number) => {
    if (e.altKey || e.ctrlKey || e.metaKey) return;
    const action = segmentedKey(index, current, options.length, e.key);
    if (!action) return;
    e.preventDefault();
    refs.current[action.focus]?.focus();
    const chosen = action.choose === null ? undefined : options[action.choose];
    if (chosen) onChange(chosen.value);
  };

  return (
    <div
      className="ei-seg"
      role="radiogroup"
      aria-labelledby={props.labelledBy}
      aria-label={props.labelledBy ? undefined : props.ariaLabel}
    >
      {options.map((o, i) => (
        <button
          key={o.value}
          ref={(el) => {
            refs.current[i] = el;
          }}
          type="button"
          role="radio"
          aria-checked={i === current}
          tabIndex={i === current ? 0 : -1}
          className={"ei-seg-btn" + (i === current ? " ei-on" : "")}
          data-value={o.value}
          onClick={() => {
            if (i !== current) onChange(o.value);
          }}
          onKeyDown={(e) => onKey(e, i)}
        >
          {o.label}
        </button>
      ))}
    </div>
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

224 righe

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
  /** Testo breve nel campo chiuso, al posto dell'etichetta (per esempio «AND» per «AND · entrambe vere»). */
  readonly short?: string;
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
  /** `pill`: pastiglia compatta (i connettori); `field`: campo a tutta larghezza. */
  readonly variant?: "field" | "pill";
  /** Larghezza del menu, se maggiore di quella del campo (px). */
  readonly menuWidth?: number;
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
  const shown = current?.short ?? current?.label ?? value;
  const pill = props.variant === "pill";
  const isFree = value !== "" && !current;

  return (
    <>
      <button
        ref={triggerRef}
        type="button"
        className={pill ? "ei-pill ei-select" : "ei-field ei-select"}
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
        <Menu
          anchor={triggerRef}
          onClose={() => close(false)}
          onEscape={() => close(true)}
          {...(props.menuWidth !== undefined ? { naturalWidth: props.menuWidth } : {})}
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

### `src/etl-canvas/inspector/conditions.ts`

103 righe

```ts
/**
 * Transizioni di stato delle liste di condizioni (filtro e join): funzioni pure
 * che restituiscono una lista NUOVA, mai modificata sul posto. Non c'è logica di
 * dominio qui: raggruppare, dividere, sciogliere e aggiungere nel gruppo sono le
 * funzioni di `etl-core` (`groupPair`, `splitAt`, `ungroup`, `addToGroup`,
 * `normalizeGroups`); qui si aggiungono solo l'identificativo di un gruppo nuovo
 * e l'ordine delle operazioni (dopo ogni azione un gruppo di una sola voce si
 * scioglie). Tutto passa poi da `setParams`.
 */
import {
  addToGroup,
  groupPair,
  groupRuns,
  normalizeGroups,
  splitAt,
  ungroup,
} from "../../etl-core";
import type { Groupable, LogicOp } from "../../etl-core";

/** Una voce senza connettore né gruppo: la lista li assegna. */
export type Blank<T> = Omit<T, "conn" | "g">;
/** Costruisce una voce vuota. */
export type MakeItem<T> = () => Blank<T>;

/**
 * Identificativo per un gruppo nuovo: «g» e il numero successivo al più alto già
 * in uso nella lista. Deterministico (niente orologio né casualità), unico nella lista.
 */
export function nextGroupId(items: readonly Groupable[]): string {
  let max = 0;
  for (const item of items) {
    const m = item.g ? /^g(\d+)/.exec(item.g) : null;
    if (m) max = Math.max(max, Number(m[1]));
  }
  return `g${max + 1}`;
}

/** Aggiunge una voce in fondo, con il connettore «AND» (la prima non ne ha bisogno ma lo ignora). */
export function addItem<T extends Groupable>(
  items: readonly T[],
  make: MakeItem<T>,
): { list: T[]; index: number } {
  const item = { ...make(), conn: "AND" as const } as T;
  return { list: [...items, item], index: items.length };
}

/** Toglie la voce `index`; un gruppo che resta con una sola voce si scioglie. */
export function removeItem<T extends Groupable>(items: readonly T[], index: number): T[] {
  return normalizeGroups(items.filter((_, i) => i !== index));
}

/** Cambia il connettore della voce `index` (quello con la voce che la precede). */
export function setConnector<T extends Groupable>(
  items: readonly T[],
  index: number,
  conn: LogicOp,
): T[] {
  return items.map((item, i) => (i === index ? { ...item, conn } : item));
}

/** Raggruppa la voce `index - 1` con la `index`; due gruppi adiacenti si fondono. */
export function groupAt<T extends Groupable>(items: readonly T[], index: number): T[] {
  return normalizeGroups(groupPair(items, index, () => nextGroupId(items)));
}

/** Divide il gruppo della voce `index` in quel punto; i gruppi di una sola voce si sciolgono. */
export function splitGroupAt<T extends Groupable>(items: readonly T[], index: number): T[] {
  return normalizeGroups(splitAt(items, index, () => nextGroupId(items)));
}

/** Scioglie il gruppo: le sue voci restano, senza gruppo. */
export function ungroupById<T extends Groupable>(items: readonly T[], groupId: string): T[] {
  return normalizeGroups(ungroup(items, groupId));
}

/** Aggiunge una voce nuova in fondo al gruppo (con il connettore «AND»). */
export function addInGroup<T extends Groupable>(
  items: readonly T[],
  groupId: string,
  make: MakeItem<T>,
): { list: T[]; index: number } {
  const r = addToGroup(items, groupId, make);
  return { list: normalizeGroups(r.list), index: r.index };
}

/** L'indice aperto dopo aver tolto la voce `removed` da una lista che ora ha `count` voci. */
export function openAfterRemove(open: number, removed: number, count: number): number {
  if (count <= 0) return -1;
  if (open > removed) return open - 1;
  return Math.min(open, count - 1);
}

/** Struttura per il disegno: ogni tratto della lista è una voce libera o un gruppo (indici di inizio e fine). */
export interface Run {
  readonly start: number;
  readonly end: number;
  readonly group: string | null;
}

export function runsOf(items: readonly Groupable[]): Run[] {
  return groupRuns(items).map((r) => ({ start: r.s, end: r.e, group: r.g }));
}
```

