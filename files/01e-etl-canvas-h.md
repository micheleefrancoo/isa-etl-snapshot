# 01e-etl-canvas-h.md

File in questo blocco:

- `src/etl-canvas/inspector/ColumnPicker.tsx`
- `src/etl-canvas/inspector/Columns3.tsx`
- `src/etl-canvas/inspector/ConditionList.tsx`
- `src/etl-canvas/inspector/ConnectorSelect.tsx`
- `src/etl-canvas/inspector/ExpandedPanel.tsx`
- `src/etl-canvas/inspector/ExpressionPreview.tsx`
- `src/etl-canvas/inspector/Field.tsx`
- `src/etl-canvas/inspector/FilterCondition.tsx`
- `src/etl-canvas/inspector/GroupFrame.tsx`
- `src/etl-canvas/inspector/Header.tsx`
- `src/etl-canvas/inspector/Inspector.tsx`
- `src/etl-canvas/inspector/JoinCondition.tsx`
- `src/etl-canvas/inspector/JoinSettings.tsx`

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

### `src/etl-canvas/inspector/Columns3.tsx`

64 righe

```tsx
/**
 * Il layout a tre colonne dei bordi alto e basso (master-detail, prototipo
 * `arrangeMasterDetail`): Impostazioni | Condizioni e gruppi (o «Elenco») |
 * Dettaglio della voce scelta. Ogni colonna ha un'intestazione fissa e scorre
 * per conto suo. Un clic su una voce la mostra nel dettaglio senza comprimerla;
 * la voce attiva è evidenziata e il titolo del dettaglio ripete numero e riassunto.
 *
 * Le righe (`ListRow`) restano nell'albero React dell'elenco: quando una è attiva
 * il suo corpo e il suo titolo passano nel dettaglio con un portale. Così lo stato,
 * il focus e la posizione di scorrimento sopravvivono ai ridisegni (niente DOM
 * spostato a mano, niente colonne ricostruite).
 *
 * Sotto la larghezza minima (`--isa-md-min-w`) l'Inspector non lo usa e tornano le
 * colonne CSS della 6b.1; sui bordi laterali le stesse parti si impilano.
 */
import { useMemo, useState } from "react";
import type { ReactNode } from "react";
import { copy } from "./copy";
import { MasterDetailContext } from "./masterDetail";
import type { MasterDetail } from "./masterDetail";

export interface Columns3Props {
  /** Impostazioni: nome, tabelle, tipo di join, passaggi del box combinato. */
  readonly general: ReactNode;
  /** Le liste di voci (condizioni e gruppi, oppure righe a voci). */
  readonly list: ReactNode;
  /** «Condizioni e gruppi» per filtro e join, «Elenco» per le operazioni a voci. */
  readonly listTitle: string;
  /** La voce attiva all'inizio. */
  readonly defaultActive: string;
}

export function Columns3(props: Columns3Props) {
  const [active, setActive] = useState(props.defaultActive);
  const [head, setHead] = useState<HTMLElement | null>(null);
  const [body, setBody] = useState<HTMLElement | null>(null);
  const value = useMemo<MasterDetail>(
    () => ({ head, body, active, setActive }),
    [head, body, active],
  );
  return (
    <MasterDetailContext.Provider value={value}>
      <div className="ei-cols3" data-testid="ei-cols3">
        <section className="ei-col ei-col-general" data-col="general">
          <div className="ei-col-head">{copy.colSettings}</div>
          <div className="ei-col-body" data-scroll-col="general">
            {props.general}
          </div>
        </section>
        <section className="ei-col ei-col-master" data-col="master">
          <div className="ei-col-head">{props.listTitle}</div>
          <div className="ei-col-body" data-scroll-col="master">
            {props.list}
          </div>
        </section>
        <section className="ei-col ei-col-detail" data-col="detail" aria-label={copy.colDetail}>
          <div className="ei-col-head ei-md-head" ref={setHead} data-testid="ei-md-head" />
          <div className="ei-col-body" ref={setBody} data-scroll-col="detail" />
        </section>
      </div>
    </MasterDetailContext.Provider>
  );
}
```

### `src/etl-canvas/inspector/ConditionList.tsx`

179 righe

```tsx
/**
 * La lista di condizioni condivisa da filtro e join: righe comprimibili con il
 * riassunto dal vivo (una aperta per volta), aggiungi e rimuovi, e tra una
 * condizione e la successiva il connettore (ConnectorSelect) con il pulsante
 * «Raggruppa» ( ) per le due condizioni che separa. Un gruppo (un solo livello)
 * è un riquadro con «Sciogli», «Dividi qui» )( sui connettori interni e «+ Condizione
 * nel gruppo». Sotto: l'anteprima dell'espressione. Tutto passa da `onChange`, che
 * i chiamanti collegano a `setParams`; le regole dei gruppi sono quelle del dominio.
 *
 * Dopo un'azione che ridisegna un pulsante (raggruppa, dividi, sciogli) il focus
 * torna su un elemento equivalente: non si perde mai.
 */
import { Fragment, useLayoutEffect, useRef, useState } from "react";
import type { ReactNode } from "react";
import type { Groupable } from "../../etl-core";
import { rowIdOf, useActiveGuard, useMasterDetail } from "./masterDetail";
import type { MakeItem } from "./conditions";
import {
  addInGroup,
  addItem,
  groupAt,
  openAfterRemove,
  removeItem,
  runsOf,
  setConnector,
  splitGroupAt,
  ungroupById,
} from "./conditions";
import { ConnectorSelect } from "./ConnectorSelect";
import { copy } from "./copy";
import { ExpressionPreview } from "./ExpressionPreview";
import { GroupFrame } from "./GroupFrame";
import { ListRow } from "./ListRow";

export interface ConditionListProps<T extends Groupable> {
  /** Identificativo della lista nel layout a tre colonne (`conditions`, `keys`). */
  readonly listId: string;
  readonly items: readonly T[];
  readonly summarize: (item: T) => string | null;
  /** Una voce vuota; la lista le dà il connettore e, nel gruppo, il gruppo. */
  readonly makeItem: MakeItem<T>;
  readonly onChange: (items: T[]) => void;
  /** I campi di una voce; `change` la sostituisce. */
  readonly renderBody: (item: T, index: number, change: (next: T) => void) => ReactNode;
  /** Titolo della lista nel layout normale (per esempio «Condizioni di unione»). */
  readonly label?: string;
  /** Dopo l'anteprima (per esempio l'avviso di prestazioni del join). */
  readonly footer?: ReactNode;
}

export function ConditionList<T extends Groupable>(props: ConditionListProps<T>) {
  const { listId, items, onChange } = props;
  const md = useMasterDetail();
  const [open, setOpen] = useState(0);
  const rootRef = useRef<HTMLDivElement>(null);
  const pendingFocus = useRef<string | null>(null);
  useActiveGuard(listId, items.length);

  // dopo un ridisegno che sostituisce un pulsante, il focus va sul suo equivalente
  useLayoutEffect(() => {
    const key = pendingFocus.current;
    if (!key) return;
    pendingFocus.current = null;
    rootRef.current?.querySelector<HTMLElement>(`[data-focus="${key}"]`)?.focus();
  });

  const current = Math.max(-1, Math.min(open, items.length - 1));
  const prefix = `${listId}:`;
  const activeIndex =
    md && md.active.startsWith(prefix) ? Number(md.active.slice(prefix.length)) : current;
  const activate = (index: number) => {
    setOpen(index);
    md?.setActive(rowIdOf(listId, index));
  };
  const apply = (next: T[], focus?: string) => {
    if (focus) pendingFocus.current = focus;
    onChange(next);
  };

  const connector = (k: number, inside: boolean) => (
    <div key={`conn-${k}`} className="ei-conn">
      <span className="ei-conn-line" />
      <ConnectorSelect
        value={items[k]?.conn}
        after={k + 1}
        onChange={(op) => onChange(setConnector(items, k, op))}
      />
      <button
        type="button"
        className="ei-conn-btn"
        data-focus={`conn-${k}`}
        aria-label={inside ? copy.groupSplit : copy.groupDo}
        title={inside ? copy.groupSplit : copy.groupDo}
        onClick={() => apply(inside ? splitGroupAt(items, k) : groupAt(items, k), `conn-${k}`)}
      >
        {inside ? copy.groupMarkSplit : copy.groupMarkDo}
      </button>
      <span className="ei-conn-line" />
    </div>
  );

  const row = (k: number) => {
    const item = items[k] as T;
    return (
      <ListRow
        key={`row-${k}`}
        rowId={rowIdOf(listId, k)}
        noun={copy.conditionNoun}
        index={k}
        summary={props.summarize(item)}
        open={k === current}
        focusKey={`row-${k}`}
        onToggle={() => setOpen(k === current ? -1 : k)}
        removeLabel={copy.conditionRemove}
        onRemove={
          items.length > 1
            ? () => {
                const next = removeItem(items, k);
                const index = openAfterRemove(md ? activeIndex : current, k, next.length);
                onChange(next);
                setOpen(index);
                if (md && index >= 0) md.setActive(rowIdOf(listId, index));
              }
            : undefined
        }
      >
        {props.renderBody(item, k, (next) => onChange(items.map((x, i) => (i === k ? next : x))))}
      </ListRow>
    );
  };

  return (
    <div ref={rootRef} className="ei-conditions" data-list={listId}>
      {props.label && !md ? <div className="ei-label">{props.label}</div> : null}
      {runsOf(items).map((run) => {
        const rows: ReactNode[] = [];
        for (let k = run.start; k <= run.end; k += 1) {
          if (k > run.start) rows.push(connector(k, true));
          rows.push(row(k));
        }
        return (
          <Fragment key={`run-${run.start}`}>
            {run.start > 0 ? connector(run.start, false) : null}
            {run.group ? (
              <GroupFrame
                groupId={run.group}
                onUngroup={() => apply(ungroupById(items, run.group as string), "add")}
                onAddIn={() => {
                  const r = addInGroup(items, run.group as string, props.makeItem);
                  apply(r.list, `row-${r.index}`);
                  activate(r.index);
                }}
              >
                {rows}
              </GroupFrame>
            ) : (
              rows
            )}
          </Fragment>
        );
      })}
      <button
        type="button"
        className="ei-addrow"
        data-focus="add"
        onClick={() => {
          const r = addItem(items, props.makeItem);
          onChange(r.list);
          activate(r.index);
        }}
      >
        + {copy.conditionAdd}
      </button>
      <ExpressionPreview items={items} summarize={props.summarize} />
      {props.footer}
    </div>
  );
}
```

### `src/etl-canvas/inspector/ConnectorSelect.tsx`

43 righe

```tsx
/**
 * Il connettore logico tra due condizioni: una pastiglia con una tendina
 * (AND, OR, XOR, NAND, NOR, XNOR), ciascuno con il proprio testo di aiuto da
 * `LOGIC_HELP` («AND · entrambe vere»). Nessun selettore globale E/O: ogni
 * condizione dalla seconda in poi dice come si combina con ciò che la precede.
 * È uno StyledSelect (portale, margine dai bordi, tastiera) nella variante pastiglia.
 */
import { LOGIC_HELP, LOGIC_OPS } from "../../etl-core";
import type { LogicOp } from "../../etl-core";
import { copy } from "./copy";
import { StyledSelect } from "./StyledSelect";
import type { SelectOption } from "./StyledSelect";

/** Larghezza del menu: la voce più lunga («XNOR · entrambe vere o entrambe false») entra su una riga. */
const MENU_WIDTH = 300;

/** Le sei voci, nell'ordine del dominio: valore «AND», etichetta «AND · entrambe vere». */
export const CONNECTOR_OPTIONS: readonly SelectOption[] = LOGIC_OPS.map((op) => ({
  value: op,
  label: copy.connectorOption(op, LOGIC_HELP[op]),
  short: op,
}));

export interface ConnectorSelectProps {
  readonly value: LogicOp | undefined;
  readonly onChange: (op: LogicOp) => void;
  /** Posizione della condizione che segue (da 2): serve al nome accessibile. */
  readonly after: number;
}

export function ConnectorSelect(props: ConnectorSelectProps) {
  return (
    <StyledSelect
      variant="pill"
      menuWidth={MENU_WIDTH}
      ariaLabel={copy.connectorLabel(props.after - 1, props.after)}
      value={props.value ?? "AND"}
      options={CONNECTOR_OPTIONS}
      onChange={(v) => props.onChange(v as LogicOp)}
    />
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

### `src/etl-canvas/inspector/ExpressionPreview.tsx`

31 righe

```tsx
/**
 * L'anteprima dell'espressione logica, come testo, dalla funzione del dominio
 * (`groupedPreview`): tra parentesi i gruppi, il resto da sinistra a destra. Un
 * riquadro con un'etichetta; NON è una regione «live»: non si annuncia a ogni
 * digitazione, si legge quando ci si arriva.
 */
import { useId } from "react";
import { groupedPreview } from "../../etl-core";
import type { Groupable } from "../../etl-core";
import { copy } from "./copy";

export function ExpressionPreview<T extends Groupable>(props: {
  readonly items: readonly T[];
  readonly summarize: (item: T) => string | null;
}) {
  const labelId = useId();
  const text = groupedPreview(props.items, (item) => props.summarize(item) ?? copy.previewEmpty);
  if (!text) return null;
  return (
    <div className="ei-preview" role="group" aria-labelledby={labelId} data-testid="ei-preview">
      <div id={labelId} className="ei-label">
        {copy.previewTitle}
      </div>
      <code className="ei-preview-text" data-testid="ei-preview-text">
        {text}
      </code>
      <div className="ei-help">{copy.previewHint}</div>
    </div>
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

### `src/etl-canvas/inspector/FilterCondition.tsx`

141 righe

```tsx
/**
 * I campi di una condizione di filtro: colonna (scelta singola, con il tipo sotto),
 * operatore, valore. Il valore manca per gli operatori senza valore (`NO_VALUE_OPS`);
 * per quelli a più valori (`MULTI_OPS`) è il ValuePicker con il dominio della colonna; per
 * gli altri è un campo di testo, con tastiera numerica se la colonna è numerica (mai
 * `input type=number`). Cambiando colonna i valori NON si azzerano: quelli fuori dal
 * dominio restano in corsivo con l'avviso e «Rimuovi» (stessa regola delle liste).
 *
 * Gli operatori sono tutti quelli del dominio (`FILTER_OPS`): nel dominio non c'è una
 * funzione che li adatti al tipo della colonna (vedi il report della Fase 6b.2).
 */
import {
  FILTER_OPS,
  MULTI_OPS,
  NO_VALUE_OPS,
  columnsDomain,
  createValuesField,
  migrateFilterLogic,
  newCondition,
  summarizeCond,
} from "../../etl-core";
import type {
  ColumnDef,
  FilterCondition as FilterConditionData,
  FilterParams,
  Params,
  ValuesField,
} from "../../etl-core";
import { ConditionList } from "./ConditionList";
import { copy } from "./copy";
import { Field, TextField } from "./Field";
import { isNumericType, typeOfColumn } from "./joinSides";
import { StyledSelect } from "./StyledSelect";
import { ValuePicker } from "./ValuePicker";

export interface FilterConditionProps {
  readonly cond: FilterConditionData;
  readonly schema: readonly ColumnDef[];
  readonly onChange: (next: FilterConditionData) => void;
}

/** I valori di una condizione come campo dei valori (la condizione ha già `mode`, `values`, `text`, `sep`). */
const valuesOf = (c: FilterConditionData): ValuesField => ({
  ...createValuesField(),
  mode: c.mode,
  values: c.values,
  text: c.text,
  sep: c.sep,
});

export function FilterCondition(props: FilterConditionProps) {
  const { cond, schema, onChange } = props;
  // la colonna si legge per destrutturazione: nell'Inspector `.column` è dei campi a voci multiple
  const { column, op } = cond;
  const type = typeOfColumn(schema, column);
  const multi = MULTI_OPS.includes(op);
  const domain = columnsDomain(schema, column ? [column] : []);

  return (
    <>
      <Field label={copy.filterColumn} help={type ? copy.filterColumnType(type) : undefined}>
        {(labelId) => (
          <StyledSelect
            labelledBy={labelId}
            allowFree
            value={column}
            options={schema.map((c) => ({ value: c.name, label: c.name, hint: c.type }))}
            // i valori scelti restano: se non sono più nel dominio si vedono in corsivo con «Rimuovi»
            onChange={(v) => onChange({ ...cond, column: v })}
          />
        )}
      </Field>
      <Field label={copy.filterOperator}>
        {(labelId) => (
          <StyledSelect
            labelledBy={labelId}
            value={op}
            options={FILTER_OPS.map((o) => ({ value: o, label: o }))}
            onChange={(v) => onChange({ ...cond, op: v as FilterConditionData["op"] })}
          />
        )}
      </Field>
      {NO_VALUE_OPS.includes(op) ? null : multi ? (
        <Field label={copy.filterValues}>
          {(labelId) => (
            <ValuePicker
              labelledBy={labelId}
              value={valuesOf(cond)}
              domain={domain}
              onChange={(next) =>
                onChange({
                  ...cond,
                  mode: next.mode,
                  values: next.values,
                  text: next.text,
                  sep: next.sep,
                })
              }
            />
          )}
        </Field>
      ) : (
        <Field label={copy.filterValue}>
          {(labelId) => (
            <TextField
              labelledBy={labelId}
              placeholder={copy.filterSingleValue}
              inputMode={isNumericType(type) ? "numeric" : "text"}
              value={cond.text}
              onChange={(v) => onChange({ ...cond, text: v })}
            />
          )}
        </Field>
      )}
    </>
  );
}

/** La lista delle condizioni di un filtro: righe, connettori, gruppi e anteprima. */
export function FilterConditions(props: {
  readonly par: Params;
  readonly schema: readonly ColumnDef[];
  readonly onChange: (par: Params) => void;
}) {
  const { par, schema, onChange } = props;
  // il vecchio selettore globale E/O si migra qui (e sparisce dai parametri alla prima modifica)
  const migrated = migrateFilterLogic(par as unknown as FilterParams);
  return (
    <ConditionList<FilterConditionData>
      listId="conditions"
      items={migrated.conditions}
      summarize={summarizeCond}
      makeItem={newCondition}
      onChange={(next) => onChange({ ...migrated, conditions: next } as unknown as Params)}
      renderBody={(cond, _i, change) => (
        <FilterCondition cond={cond} schema={schema} onChange={change} />
      )}
    />
  );
}
```

### `src/etl-canvas/inspector/GroupFrame.tsx`

35 righe

```tsx
/**
 * Il riquadro tratteggiato di un gruppo di condizioni (un solo livello, come il
 * dominio): intestazione «Gruppo» con «Sciogli» e, in fondo, «+ Condizione nel
 * gruppo». Dentro stanno le righe e i connettori interni (con «Dividi qui»).
 */
import type { ReactNode } from "react";
import { copy } from "./copy";

export function GroupFrame(props: {
  readonly groupId: string;
  readonly onUngroup: () => void;
  readonly onAddIn: () => void;
  readonly children: ReactNode;
}) {
  return (
    <div className="ei-group" role="group" aria-label={copy.groupTitle} data-group={props.groupId}>
      <div className="ei-group-head">
        <span className="ei-group-title">{copy.groupTitle}</span>
        <button
          type="button"
          className="ei-link-btn"
          aria-label={copy.groupUngroupLabel}
          onClick={props.onUngroup}
        >
          {copy.groupUngroup}
        </button>
      </div>
      {props.children}
      <button type="button" className="ei-addrow ei-addin" onClick={props.onAddIn}>
        + {copy.groupAddIn}
      </button>
    </div>
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

333 righe

```tsx
/**
 * Il contenuto dell'Inspector (Fase 6b.1), montato da `panels/InspectorShell`.
 * Legge il nodo di `etl-store` (`inspector`) e mostra, per tipo di nodo:
 * stato bloccato (lavorazione senza ingresso), dataset e output (nome, origine,
 * colonne in sola lettura), lavorazioni (campi e liste del catalogo, con colonne
 * e valori dallo schema in ingresso) e box combinati (elenco dei passaggi).
 * Filtro e join hanno condizioni con connettori, gruppi e anteprima (Fase 6b.2); sui bordi
 * alto e basso, se il pannello è abbastanza largo, il contenuto sta in tre colonne
 * (Impostazioni | Condizioni o Elenco | Dettaglio, `Columns3`).
 *
 * Scrive solo con i comandi `setParams`, `renameNode`, `inspect` e quelli dei
 * passaggi; legge e scrive SOLO `columns` (mai `flattenRows`). Il focus non si
 * perde mai scrivendo: i campi hanno chiavi stabili e nulla viene ricreato.
 */
import { useLayoutEffect, useRef, useState } from "react";
import type { KeyboardEvent, ReactNode, RefObject } from "react";
import { MULTI_DEFS, PARAM_DEFS, boxCapacity, inputsOf } from "../../etl-core";
import type { Card, Graph, Params, SimpleFieldDef } from "../../etl-core";
import type { EtlStore } from "../../etl-store";
import { useEtlState } from "../../etl-store/react";
import { BlockedNotice } from "./BlockedNotice";
import { Columns3 } from "./Columns3";
import { copy } from "./copy";
import { Field, TextField } from "./Field";
import { FilterConditions } from "./FilterCondition";
import { Header } from "./Header";
import { joinSideSchemas } from "./joinSides";
import { JoinConditions } from "./JoinCondition";
import { JoinSettings } from "./JoinSettings";
import { rowIdOf } from "./masterDetail";
import { MultiGlobals, MultiList, MultiRows } from "./MultiList";
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

/** Larghezza minima del pannello per le tre colonne: il token `--isa-md-min-w` (circa 900 px). */
function useWideEnough(ref: RefObject<HTMLElement | null>, enabled: boolean): boolean {
  // prima della misura si presume largo: il layout effect misura prima che il browser disegni
  const [wide, setWide] = useState(true);
  useLayoutEffect(() => {
    const el = ref.current;
    if (!el || !enabled) return;
    const measure = () => {
      const min = parseFloat(getComputedStyle(el).getPropertyValue("--isa-md-min-w"));
      setWide(!Number.isFinite(min) || el.clientWidth >= min);
    };
    measure();
    const ro = new ResizeObserver(measure);
    ro.observe(el);
    return () => ro.disconnect();
  }, [ref, enabled]);
  return wide;
}

export function Inspector(props: {
  readonly store: EtlStore;
  /** Il pannello sta sul bordo alto o basso: tre colonne, se c'è abbastanza larghezza. */
  readonly horizontal?: boolean;
}) {
  const { store } = props;
  const horizontal = props.horizontal ?? false;
  const nodeId = useEtlState((s) => s.inspector.nodeId, store);
  const step = useEtlState((s) => s.inspector.step, store);
  const card = useEtlState(
    (s) => (s.inspector.nodeId ? s.graph.cards[s.inspector.nodeId] : undefined),
    store,
  );
  const graph = useEtlState((s) => s.graph, store);
  const schema = useActiveSchema(store, nodeId);
  const rootRef = useRef<HTMLDivElement>(null);
  const wide = useWideEnough(rootRef, horizontal);

  if (!card) {
    return (
      <div className="ei-root" data-testid="ei-root" onKeyDown={returnToCanvas}>
        <div className="ei-help ei-empty">{copy.emptyInspector}</div>
      </div>
    );
  }
  return (
    <div
      ref={rootRef}
      className="ei-root"
      data-testid="ei-root"
      data-kind={kindOf(card)}
      onKeyDown={returnToCanvas}
    >
      <Body
        store={store}
        card={card}
        graph={graph}
        step={step}
        schema={schema}
        header={<Header store={store} card={card} />}
        threeColumns={horizontal && wide}
      />
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
  header: ReactNode;
  threeColumns: boolean;
}) {
  const { store, card, graph, schema, header } = props;

  if (card.kind === "dataset" && card.isOutput) {
    const producer = graph.links.find((l) => l.to === card.id);
    const producerName = (producer && graph.cards[producer.from]?.name) || copy.producerFallback;
    const incomplete =
      card.capacity !== undefined && card.capacity > 1 && (card.filled ?? 0) < card.capacity;
    return (
      <>
        {header}
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
        {header}
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
  if (inputs.length === 0)
    return (
      <>
        {header}
        <BlockedNotice capacity={boxCapacity(card)} />
      </>
    );

  const step = Math.max(0, Math.min(props.step, card.components.length - 1));
  const type = card.components[step] ?? card.components[0];
  const par: Params = card.params[step] ?? {};
  const setParams = (p: Params) =>
    store.dispatch({ type: "setParams", payload: { node: card.id, index: step, params: p } });
  const names = schema.map((c) => c.name);

  // le parti: impostazioni (passaggi, tabelle, campi globali, ingressi) e voci (condizioni o righe)
  const steps =
    card.components.length > 1 ? (
      <StepList
        store={store}
        card={card}
        selectedStep={step}
        variant="inspector"
        onSelect={(i) => store.dispatch({ type: "inspect", payload: { node: card.id, step: i } })}
      />
    ) : null;
  const tables = (
    <JoinSettings card={card} graph={graph} step={step} par={par} onChange={setParams} />
  );
  const inputsNote = (
    <div className="ei-help">{copy.inputsCount(inputs.length, boxCapacity(card))}</div>
  );

  let rows: ReactNode = null;
  let rowsKind: "conditions" | "list" | null = null;
  let firstRow = "";
  let globals: ReactNode = null;
  if (type === "filter") {
    rowsKind = "conditions";
    firstRow = rowIdOf("conditions", 0);
    rows = <FilterConditions par={par} schema={schema} onChange={setParams} />;
  } else if (type === "join") {
    rowsKind = "conditions";
    firstRow = rowIdOf("keys", 0);
    const sides = joinSideSchemas(graph, card, step, par, schema, copy.tableFallback);
    rows = <JoinConditions par={par} left={sides.left} right={sides.right} onChange={setParams} />;
  } else if (type && isMulti(type)) {
    rowsKind = "list";
    const first = MULTI_DEFS[type]?.lists[0]?.key ?? "items";
    firstRow = rowIdOf(first, 0);
    const multi = { type, par, schema, onChange: setParams };
    globals = <MultiGlobals {...multi} />;
    rows = <MultiRows {...multi} />;
  }

  // bordi alto e basso, pannello largo: tre colonne
  if (props.threeColumns && rowsKind) {
    return (
      <Columns3
        key={`${card.id}:${step}`}
        listTitle={rowsKind === "conditions" ? copy.colConditions : copy.colList}
        defaultActive={firstRow}
        general={
          <>
            {header}
            {steps}
            {tables}
            {globals}
            {inputsNote}
          </>
        }
        list={rows}
      />
    );
  }

  return (
    <>
      {header}
      {steps}
      {tables}
      {rowsKind ? (
        <>
          {globals}
          {rows}
        </>
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
      {inputsNote}
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
```

### `src/etl-canvas/inspector/JoinCondition.tsx`

206 righe

```tsx
/**
 * I campi di una condizione di join (prototipo `joinSideHtml` e `joinOpHtml`): lato
 * sinistro (Colonna | Valore), confronto, lato destro (Colonna | Valore | Lista).
 * Le colonne di ciascun lato vengono dalla propria tabella (o dall'unione, vedi
 * `joinSides.ts`). Un valore è una tendina con i valori della colonna dell'ALTRO lato e
 * «oppure scrivi» (in corsivo); la lista è il ValuePicker con il dominio della colonna
 * dell'altro lato. Con una lista a destra il confronto è «è uno di» / «non è uno di».
 */
import {
  JOIN_OPS,
  JOIN_OP_NAME,
  LIST_OPS,
  createValuesField,
  ensureKeys,
  hasEquiJoinCondition,
  keyComplete,
  summarizeKey,
} from "../../etl-core";
import type {
  ColumnDef,
  JoinKey,
  JoinOp,
  JoinParams,
  JoinRightMode,
  JoinSideMode,
  Params,
} from "../../etl-core";
import { ConditionList } from "./ConditionList";
import { copy } from "./copy";
import { Field, TextField } from "./Field";
import { domainOfColumn } from "./joinSides";
import { blankJoinKey, withLeftMode, withRightMode } from "./joinKeys";
import { Segmented } from "./Segmented";
import { StyledSelect } from "./StyledSelect";
import { ValuePicker } from "./ValuePicker";

export interface JoinConditionProps {
  readonly item: JoinKey;
  /** Colonne del lato sinistro e del lato destro. */
  readonly left: readonly ColumnDef[];
  readonly right: readonly ColumnDef[];
  readonly onChange: (next: JoinKey) => void;
}

const LEFT_MODES: readonly { value: JoinSideMode; label: string }[] = [
  { value: "col", label: copy.joinModeColumn },
  { value: "val", label: copy.joinModeValue },
];
const RIGHT_MODES: readonly { value: JoinRightMode; label: string }[] = [
  { value: "col", label: copy.joinModeColumn },
  { value: "val", label: copy.joinModeValue },
  { value: "list", label: copy.joinModeList },
];

export function JoinCondition(props: JoinConditionProps) {
  const { item, left, right, onChange } = props;
  const lmode = item.lmode ?? "col";
  const rmode = item.rmode ?? "col";
  const op = item.op ?? "=";
  // il dominio da proporre è quello della colonna sull'altro lato del confronto
  const leftDomain = domainOfColumn(right, rmode === "col" ? item.right : "");
  const rightDomain = domainOfColumn(left, lmode === "col" ? item.left : "");
  const ops: readonly string[] = rmode === "list" ? LIST_OPS : JOIN_OPS;
  const opLabel = (o: string): string => {
    const i = JOIN_OPS.indexOf(o as JoinOp);
    return i >= 0 ? `${o} ${JOIN_OP_NAME[i] ?? ""}`.trim() : o;
  };

  const columnSelect = (
    labelId: string,
    schema: readonly ColumnDef[],
    value: string,
    set: (v: string) => void,
  ) => (
    <StyledSelect
      labelledBy={labelId}
      allowFree
      value={value}
      options={schema.map((c) => ({ value: c.name, label: c.name, hint: c.type }))}
      onChange={set}
    />
  );
  const valueControl = (
    labelId: string,
    domain: readonly string[],
    value: string,
    set: (v: string) => void,
  ) =>
    domain.length > 0 ? (
      <StyledSelect
        labelledBy={labelId}
        allowFree
        value={value}
        placeholder={copy.pickOrType}
        options={domain.map((v) => ({ value: v, label: v }))}
        onChange={set}
      />
    ) : (
      <TextField
        labelledBy={labelId}
        placeholder={copy.joinValuePlaceholder}
        value={value}
        onChange={set}
      />
    );

  return (
    <>
      <Field label={copy.joinLeftSide}>
        {(labelId) => (
          <>
            <Segmented
              labelledBy={labelId}
              value={lmode}
              options={LEFT_MODES}
              onChange={(m) => onChange(withLeftMode(item, m))}
            />
            {lmode === "col"
              ? columnSelect(labelId, left, item.left, (v) => onChange({ ...item, left: v }))
              : valueControl(labelId, leftDomain, item.lval ?? "", (v) =>
                  onChange({ ...item, lval: v }),
                )}
          </>
        )}
      </Field>
      <Field label={copy.joinCompare}>
        {(labelId) => (
          <StyledSelect
            labelledBy={labelId}
            value={op}
            options={ops.map((o) => ({ value: o, label: opLabel(o) }))}
            onChange={(v) => onChange({ ...item, op: v as JoinOp })}
          />
        )}
      </Field>
      <Field label={copy.joinRightSide}>
        {(labelId) => (
          <>
            <Segmented
              labelledBy={labelId}
              value={rmode}
              options={RIGHT_MODES}
              onChange={(m) => onChange(withRightMode(item, m))}
            />
            {rmode === "col" ? (
              columnSelect(labelId, right, item.right, (v) => onChange({ ...item, right: v }))
            ) : rmode === "val" ? (
              valueControl(labelId, rightDomain, item.rval ?? "", (v) =>
                onChange({ ...item, rval: v }),
              )
            ) : (
              <ValuePicker
                labelledBy={labelId}
                value={item.rlist ?? createValuesField()}
                domain={rightDomain}
                onChange={(next) => onChange({ ...item, rlist: next })}
              />
            )}
          </>
        )}
      </Field>
    </>
  );
}

/** La lista delle condizioni di unione, con l'avviso di prestazioni se nessuna è colonna = colonna. */
export function JoinConditions(props: {
  readonly par: Params;
  readonly left: readonly ColumnDef[];
  readonly right: readonly ColumnDef[];
  readonly onChange: (par: Params) => void;
}) {
  const { par, left, right, onChange } = props;
  const jp = par as unknown as JoinParams;
  const keys = ensureKeys(jp);
  // senza nessuna uguaglianza ogni riga va confrontata con ogni altra: su tabelle grandi è molto lento
  const configured = keys.filter(keyComplete);
  const slow = configured.length > 0 && !hasEquiJoinCondition(configured);
  return (
    <ConditionList<JoinKey>
      listId="keys"
      label={copy.joinConditions}
      items={keys}
      summarize={summarizeKey}
      makeItem={blankJoinKey}
      onChange={(next) => {
        // il vecchio formato a chiave singola sparisce alla prima modifica
        const { leftKey, rightKey, ...rest } = jp;
        void leftKey;
        void rightKey;
        onChange({ ...rest, keys: next } as unknown as Params);
      }}
      renderBody={(item, _i, change) => (
        <JoinCondition item={item} left={left} right={right} onChange={change} />
      )}
      footer={
        slow ? (
          <div className="ei-warn" role="status" data-testid="ei-join-perf">
            {copy.joinPerformance}
          </div>
        ) : null
      }
    />
  );
}
```

### `src/etl-canvas/inspector/JoinSettings.tsx`

80 righe

```tsx
/**
 * Le impostazioni di un passaggio che dipende dalle tabelle in ingresso (prototipo, righe
 * 3747-3779): i join si applicano nell'ordine in cui compaiono e ognuno consuma una tabella in
 * più. Un passaggio di join sceglie la tabella sinistra (solo per il primo: dal secondo la
 * sinistra è il risultato del join precedente), la destra e il tipo di join; prima di un join un
 * passaggio ha la tabella di riferimento; dopo, la tabella è una sola (nota).
 *
 * Il valore proposto (la sinistra è la prima tabella in ingresso, la destra la seconda) si mostra
 * senza scriverlo: si salva solo quando l'utente sceglie. I valori del tipo di join vengono dal catalogo.
 */
import { MERGE_OPS, PARAM_DEFS } from "../../etl-core";
import type { Card, Graph, Params, SimpleFieldDef } from "../../etl-core";
import { copy } from "./copy";
import { Field } from "./Field";
import { inputNames, joinIndex, resolveTable } from "./joinSides";
import type { TableKey } from "./joinSides";
import { textParam, withParam } from "./params";
import { StyledSelect } from "./StyledSelect";

const isMerge = (type: string | undefined): boolean =>
  !!type && (MERGE_OPS as readonly string[]).includes(type);

export function JoinSettings(props: {
  readonly card: Card;
  readonly graph: Graph;
  readonly step: number;
  readonly par: Params;
  readonly onChange: (par: Params) => void;
}) {
  const { card, graph, step, par, onChange } = props;
  const hasJoin = card.components.some((c) => isMerge(c));
  if (!hasJoin) return null;
  const names = inputNames(graph, card, copy.tableFallback);
  if (names.length === 0) return <div className="ei-help">{copy.tableLinkFirst}</div>;

  const table = (key: TableKey, label: string) => (
    <Field key={key} label={label}>
      {(labelId) => (
        <StyledSelect
          labelledBy={labelId}
          value={resolveTable(textParam(par, key), key, names)}
          options={names.map((n) => ({ value: n, label: n }))}
          onChange={(v) => onChange(withParam(par, key, v))}
        />
      )}
    </Field>
  );

  const type = card.components[step];
  if (isMerge(type)) {
    const j = joinIndex(card, step);
    const typeDef = (PARAM_DEFS.join as readonly SimpleFieldDef[])[0];
    return (
      <>
        {j === 0 ? (
          table("leftTable", copy.tableLeft)
        ) : (
          <div className="ei-help">{copy.tableLeftResult(j)}</div>
        )}
        {table("rightTable", copy.tableRight)}
        {typeDef ? (
          <Field label={typeDef.label}>
            {(labelId) => (
              <StyledSelect
                labelledBy={labelId}
                value={textParam(par, typeDef.k) || typeDef.def}
                options={(typeDef.opts ?? []).map((o) => ({ value: o, label: o }))}
                onChange={(v) => onChange(withParam(par, typeDef.k, v))}
              />
            )}
          </Field>
        ) : null}
      </>
    );
  }
  const joinsBefore = card.components.filter((c, i) => i < step && isMerge(c)).length;
  if (joinsBefore === 0) return table("table", copy.tableReference);
  return <div className="ei-help">{copy.tableSingle(joinsBefore)}</div>;
}
```

