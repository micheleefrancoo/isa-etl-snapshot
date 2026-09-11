# ISA ETL Snapshot

Generated: 2026-09-11T21:45:15Z

## Index
- src/components/isa/etl/data-preview.tsx
- src/components/isa/etl/inspector.tsx
- src/components/isa/etl/isa-context-menu.tsx
- src/components/isa/etl/settings-panels/aggregate-panel.tsx
- src/components/isa/etl/settings-panels/combine-panel.tsx
- src/components/isa/etl/settings-panels/filter-panel.tsx
- src/components/isa/etl/settings-panels/panel-controls.tsx
- src/components/isa/etl/tool-palette.tsx
- src/components/isa/etl/workflow-canvas.tsx


=== FILE: src/components/isa/etl/data-preview.tsx ===
import { Table2, X } from "lucide-react";
import { analyzeNode, formatRows, previewRows } from "@/lib/etl-schema";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";

/** Cassetto Data preview: si apre solo su comando e si chiude con la X. */
export function DataPreview({
  workflow,
  node,
  open,
  inset,
  onClose,
}: {
  workflow: EtlWorkflow;
  node: EtlNode | null;
  open: boolean;
  inset?: boolean;
  onClose: () => void;
}) {
  if (!open) return null;

  const analysis = node ? analyzeNode(workflow, node) : null;
  const columns = analysis?.columns ?? [];
  const rows = node ? previewRows(columns, node.id) : [];

  return (
    <section
      className={`glass-panel absolute bottom-3 left-3 z-40 flex max-h-[42%] flex-col rounded-3xl ${
        inset ? "right-[21.5rem]" : "right-3"
      }`}
      style={{ background: "color-mix(in srgb, var(--background) 92%, transparent)" }}
    >
      <div className="flex items-center gap-2 px-4 py-2.5">
        <Table2 className="size-4 text-muted-foreground" />
        <span className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
          Data preview
        </span>
        <span className="glass-chip truncate rounded-full px-2.5 py-1 text-[10px] text-muted-foreground">
          {node ? node.title : "nessun nodo selezionato"}
        </span>
        {analysis && (
          <span className="glass-chip hidden rounded-full px-2.5 py-1 text-[10px] text-muted-foreground sm:inline">
            {formatRows(analysis.rows)} rows · {columns.length} cols
          </span>
        )}
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi data preview"
          className="glass-chip ml-auto flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>
      </div>

      <div className="scroll-slim min-h-0 flex-1 overflow-auto border-t border-border/60 px-4 py-3">
        {!node ? (
          <p className="text-xs text-muted-foreground">
            Seleziona un nodo nel canvas per ispezionarne i dati.
          </p>
        ) : columns.length === 0 ? (
          <p className="text-xs text-muted-foreground">
            Completa la configurazione del nodo nell'Inspector per definire lo schema in uscita.
          </p>
        ) : (
          <>
            <div className="overflow-x-auto rounded-2xl border border-border/60">
              <table className="w-full text-left text-xs">
                <thead>
                  <tr>
                    {columns.map((c) => (
                      <th key={c.name} className="whitespace-nowrap px-3 py-2 font-semibold">
                        {c.name}
                        <span className="ml-1.5 text-[9px] font-normal text-muted-foreground">
                          {c.type}
                        </span>
                      </th>
                    ))}
                  </tr>
                </thead>
                <tbody>
                  {rows.map((row, i) => (
                    <tr key={i} className="border-t border-border/60">
                      {row.map((cell, j) => (
                        <td
                          key={j}
                          className="whitespace-nowrap px-3 py-1.5 font-mono text-[11px] text-muted-foreground"
                        >
                          {cell}
                        </td>
                      ))}
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
            <p className="mt-2 text-[10px] text-muted-foreground">
              Anteprima di authoring: i dati reali compariranno quando il motore di esecuzione sarà
              collegato.
            </p>
          </>
        )}
      </div>
    </section>
  );
}


=== FILE: src/components/isa/etl/inspector.tsx ===
import { AlertTriangle, Table2, Trash2, X } from "lucide-react";
import type { ColumnDef } from "@/lib/etl-catalog";
import { categoryAccent, nodeDef } from "@/lib/etl-catalog";
import { analyzeNode, formatRows } from "@/lib/etl-schema";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";

/** Cassetto Inspector: si apre solo su comando e si chiude con la X. */
export function Inspector({
  workflow,
  node,
  open,
  onClose,
  onChangeTitle,
  onChangeConfig,
  onRemove,
  onPreview,
}: {
  workflow: EtlWorkflow;
  node: EtlNode | null;
  open: boolean;
  onClose: () => void;
  onChangeTitle: (value: string) => void;
  onChangeConfig: (key: string, value: string) => void;
  onRemove: () => void;
  onPreview: () => void;
}) {
  if (!open) return null;

  const def = node ? nodeDef(node.type) : undefined;
  const analysis = node ? analyzeNode(workflow, node) : null;

  return (
    <aside
      className="glass-panel absolute bottom-3 right-3 top-14 z-40 flex w-[min(20rem,calc(100%-1.5rem))] flex-col rounded-3xl p-4"
      style={{ background: "color-mix(in srgb, var(--background) 92%, transparent)" }}
    >
      <div className="flex items-center gap-2">
        <span className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
          Inspector
        </span>
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi inspector"
          className="glass-chip ml-auto flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>
      </div>

      {!node || !def || !analysis ? (
        <p className="mt-3 text-xs text-muted-foreground">
          Seleziona un nodo nel canvas per configurarne i parametri.
        </p>
      ) : (
        <div className="scroll-slim mt-3 min-h-0 flex-1 space-y-4 overflow-y-auto pr-1">
          <div className="flex items-center gap-2">
            <span
              className={`${categoryAccent(def.category)} flex size-8 shrink-0 items-center justify-center rounded-xl`}
            >
              <def.Icon className="size-4" />
            </span>
            <span className="min-w-0">
              <span className="block truncate text-sm font-semibold">{def.label}</span>
              <span className="block truncate text-[11px] text-muted-foreground">
                {def.description}
              </span>
            </span>
          </div>

          <div className="flex flex-wrap gap-1.5">
            <span className="glass-chip rounded-full px-2.5 py-1 text-[10px] text-muted-foreground">
              {formatRows(analysis.rows)} rows
            </span>
            <span className="glass-chip rounded-full px-2.5 py-1 text-[10px] text-muted-foreground">
              {analysis.columns.length} cols
            </span>
            <span
              className="glass-chip rounded-full px-2.5 py-1 text-[10px] font-semibold"
              style={{ color: analysis.errors.length ? "var(--destructive)" : "var(--success)" }}
            >
              {analysis.errors.length ? "Non valido" : "Valido"}
            </span>
          </div>

          {analysis.errors.length > 0 && (
            <ul className="glass-chip space-y-1.5 rounded-2xl p-3">
              {analysis.errors.map((err) => (
                <li key={err} className="flex gap-2 text-[11px] text-destructive">
                  <AlertTriangle className="mt-0.5 size-3.5 shrink-0" />
                  {err}
                </li>
              ))}
            </ul>
          )}

          <div>
            <label htmlFor="isa-node-title" className="text-xs font-medium">
              Nome nodo
            </label>
            <input
              id="isa-node-title"
              value={node.title}
              onChange={(e) => onChangeTitle(e.target.value)}
              className="glass-chip mt-1.5 h-10 w-full rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
            />
          </div>

          {def.fields.map((field) => {
            const id = `isa-field-${field.key}`;
            const value = node.config[field.key] ?? "";
            return (
              <div key={field.key}>
                <label htmlFor={id} className="text-xs font-medium">
                  {field.label}
                  {field.required && <span className="text-destructive"> *</span>}
                </label>
                {field.kind === "textarea" ? (
                  <textarea
                    id={id}
                    value={value}
                    rows={3}
                    placeholder={field.placeholder}
                    onChange={(e) => onChangeConfig(field.key, e.target.value)}
                    className="glass-chip mt-1.5 w-full resize-none rounded-2xl px-3 py-2 font-mono text-xs outline-none focus:ring-2 focus:ring-ring"
                  />
                ) : field.kind === "select" ? (
                  <select
                    id={id}
                    value={value}
                    onChange={(e) => onChangeConfig(field.key, e.target.value)}
                    className="glass-chip mt-1.5 h-10 w-full rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
                  >
                    {field.options?.map((opt) => (
                      <option key={opt} value={opt}>
                        {opt}
                      </option>
                    ))}
                  </select>
                ) : (
                  <input
                    id={id}
                    value={value}
                    placeholder={field.placeholder}
                    onChange={(e) => onChangeConfig(field.key, e.target.value)}
                    className="glass-chip mt-1.5 h-10 w-full rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
                  />
                )}
              </div>
            );
          })}

          <div>
            <span className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
              Schema evolution
            </span>
            <div className="mt-2 grid gap-2 sm:grid-cols-2">
              <SchemaList title="Input" columns={analysis.inputColumns} />
              <SchemaList title="Output" columns={analysis.columns} />
            </div>
          </div>

          <button
            type="button"
            onClick={onPreview}
            className="glass-chip flex h-10 w-full items-center justify-center gap-2 rounded-full text-sm font-medium transition hover:brightness-105"
          >
            <Table2 className="size-4" />
            Preview data
          </button>

          <button
            type="button"
            onClick={onRemove}
            className="glass-chip flex h-10 w-full items-center justify-center gap-2 rounded-full text-sm font-medium text-destructive transition hover:brightness-105"
          >
            <Trash2 className="size-4" />
            Elimina nodo
          </button>
        </div>
      )}
    </aside>
  );
}

function SchemaList({ title, columns }: { title: string; columns: ColumnDef[] }) {
  return (
    <div className="glass-chip rounded-2xl p-2.5">
      <span className="text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
        {title}
      </span>
      {columns.length === 0 ? (
        <p className="mt-1.5 text-[11px] text-muted-foreground">—</p>
      ) : (
        <ul className="scroll-slim mt-1.5 max-h-40 space-y-1 overflow-y-auto pr-1">
          {columns.map((c) => (
            <li key={c.name} className="flex items-baseline gap-1.5">
              <span className="min-w-0 flex-1 truncate text-[11px] font-medium">{c.name}</span>
              <span className="text-[9px] text-muted-foreground">{c.type}</span>
              {c.nullable && <span className="text-[9px] text-muted-foreground/70">null</span>}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}


=== FILE: src/components/isa/etl/isa-context-menu.tsx ===
import {
  Copy,
  Grid3x3,
  LayoutGrid,
  Maximize2,
  Plus,
  Trash2,
  Unlink,
} from "lucide-react";
import {
  useEffect,
  useLayoutEffect,
  useRef,
  useState,
} from "react";
import { createPortal } from "react-dom";

type ContextTarget = {
  nodeId: string | null;
};

type IsaContextMenuProps = {
  open: boolean;
  x: number;
  y: number;
  boundaryRef: React.RefObject<HTMLElement | null>;
  target: ContextTarget;
  grid: boolean;
  onAddDataset: () => void;
  onFitView: () => void;
  onToggleGrid: () => void;
  onAutoLayout: () => void;
  onDuplicateNode?: (id: string) => void;
  onRemoveNode?: (id: string) => void;
  onUnlinkNode?: (id: string) => void;
  onClose: () => void;
};

const MENU_WIDTH = 224;
const MENU_MARGIN = 8;

export function IsaContextMenu({
  open,
  x,
  y,
  boundaryRef,
  target,
  grid,
  onAddDataset,
  onFitView,
  onToggleGrid,
  onAutoLayout,
  onDuplicateNode,
  onRemoveNode,
  onUnlinkNode,
  onClose,
}: IsaContextMenuProps) {
  const menuRef =
    useRef<HTMLDivElement>(null);

  const [position, setPosition] = useState({
    left: x,
    top: y,
  });

  useLayoutEffect(() => {
    if (!open) return;

    const boundary =
      boundaryRef.current;

    const menu =
      menuRef.current;

    if (!boundary || !menu) return;

    const boundaryRect =
      boundary.getBoundingClientRect();

    const menuRect =
      menu.getBoundingClientRect();

    const maxLeft =
      Math.max(
        boundaryRect.left + MENU_MARGIN,
        boundaryRect.right -
          menuRect.width -
          MENU_MARGIN,
      );

    const maxTop =
      Math.max(
        boundaryRect.top + MENU_MARGIN,
        boundaryRect.bottom -
          menuRect.height -
          MENU_MARGIN,
      );

    setPosition({
      left: Math.min(
        Math.max(
          x,
          boundaryRect.left +
            MENU_MARGIN,
        ),
        maxLeft,
      ),
      top: Math.min(
        Math.max(
          y,
          boundaryRect.top +
            MENU_MARGIN,
        ),
        maxTop,
      ),
    });
  }, [
    open,
    x,
    y,
    boundaryRef,
    target.nodeId,
  ]);

  useEffect(() => {
    if (!open) return;

    const handlePointerDown = (
      event: PointerEvent,
    ) => {
      if (
        !menuRef.current?.contains(
          event.target as Node,
        )
      ) {
        onClose();
      }
    };

    const handleKeyDown = (
      event: KeyboardEvent,
    ) => {
      if (event.key === "Escape") {
        onClose();
      }
    };

    document.addEventListener(
      "pointerdown",
      handlePointerDown,
    );

    document.addEventListener(
      "keydown",
      handleKeyDown,
    );

    return () => {
      document.removeEventListener(
        "pointerdown",
        handlePointerDown,
      );

      document.removeEventListener(
        "keydown",
        handleKeyDown,
      );
    };
  }, [open, onClose]);

  if (
    !open ||
    typeof document ===
      "undefined"
  ) {
    return null;
  }

  const hasNode =
    Boolean(target.nodeId);

  return createPortal(
    <div
      ref={menuRef}
      role="menu"
      aria-label="Menu contestuale IsA"
      className="fixed z-100 w-56 overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl"
      style={{
        left: position.left,
        top: position.top,
      }}
      onPointerDown={(event) =>
        event.stopPropagation()
      }
      onContextMenu={(event) =>
        event.preventDefault()
      }
    >
      {hasNode ? (
        <>
          <button
            type="button"
            role="menuitem"
            className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
            onClick={() => {
              if (
                target.nodeId &&
                onDuplicateNode
              ) {
                onDuplicateNode(
                  target.nodeId,
                );
              }
              onClose();
            }}
          >
            <Copy className="size-4" />
            Duplica nodo
          </button>

          <button
            type="button"
            role="menuitem"
            className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
            onClick={() => {
              if (
                target.nodeId &&
                onUnlinkNode
              ) {
                onUnlinkNode(
                  target.nodeId,
                );
              }
              onClose();
            }}
          >
            <Unlink className="size-4" />
            Scollega input
          </button>

          <button
            type="button"
            role="menuitem"
            className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left text-destructive transition hover:bg-muted"
            onClick={() => {
              if (
                target.nodeId &&
                onRemoveNode
              ) {
                onRemoveNode(
                  target.nodeId,
                );
              }
              onClose();
            }}
          >
            <Trash2 className="size-4" />
            Elimina nodo
          </button>

          <span className="my-1 block h-px bg-border" />
        </>
      ) : null}

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onAddDataset();
          onClose();
        }}
      >
        <Plus className="size-4" />
        Aggiungi dataset
      </button>

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onAutoLayout();
          onClose();
        }}
      >
        <LayoutGrid className="size-4" />
        Disponi automaticamente
      </button>

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onFitView();
          onClose();
        }}
      >
        <Maximize2 className="size-4" />
        Adatta vista
      </button>

      <button
        type="button"
        role="menuitem"
        className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted"
        onClick={() => {
          onToggleGrid();
          onClose();
        }}
      >
        <Grid3x3 className="size-4" />
        {grid
          ? "Nascondi griglia"
          : "Mostra griglia"}
      </button>
    </div>,
    document.body,
  );
}

=== FILE: src/components/isa/etl/settings-panels/aggregate-panel.tsx ===
import { nodeDef } from "@/lib/etl-catalog";
import { getAggregateFieldSpecs } from "@/lib/etl-node-config";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";
import {
  PanelChipToggleList,
  PanelLabel,
  PanelSelect,
} from "@/components/isa/etl/settings-panels/panel-controls";

/**
 * Pannello impostazioni per la famiglia "aggregate" (fase 4, punto 3):
 * Group By / Aggregate / Pivot condividono lo stesso principio —
 * selettori (singoli o multipli) dallo schema in ingresso reale,
 * invece dei campi di testo libero del catalogo generico.
 */
export function AggregatePanel({
  workflow,
  node,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const fieldSpecs = getAggregateFieldSpecs(
    workflow,
    node,
  );

  const aggregationOptions =
    nodeDef(node.type)?.fields.find(
      (field) =>
        field.key === "aggregation",
    )?.options ?? [];

  return (
    <div className="space-y-3">
      {fieldSpecs.map((spec) => {
        if (spec.kind === "multi") {
          const selected = (
            node.config[spec.key] ?? ""
          )
            .split(",")
            .map((v) => v.trim())
            .filter(Boolean);

          const values =
            spec.options.map(
              (c) => c.name,
            );

          return (
            <div key={spec.key}>
              <PanelLabel>
                {spec.label}
              </PanelLabel>

              <PanelChipToggleList
                values={values}
                selected={selected}
                onToggle={(value) => {
                  const next =
                    selected.includes(
                      value,
                    )
                      ? selected.filter(
                          (v) =>
                            v !==
                            value,
                        )
                      : [
                          ...selected,
                          value,
                        ];

                  onConfigChange({
                    [spec.key]:
                      next.join(
                        ", ",
                      ),
                  });
                }}
              />
            </div>
          );
        }

        return (
          <div key={spec.key}>
            <PanelLabel>
              {spec.label}
            </PanelLabel>

            <PanelSelect
              value={
                node.config[
                  spec.key
                ] ?? ""
              }
              onChange={(value) =>
                onConfigChange({
                  [spec.key]: value,
                })
              }
              options={spec.options}
            />
          </div>
        );
      })}

      {aggregationOptions.length > 0 && (
        <div>
          <PanelLabel>
            Funzione
          </PanelLabel>

          <select
            value={
              node.config[
                "aggregation"
              ] ?? ""
            }
            onChange={(event) =>
              onConfigChange({
                aggregation:
                  event.target
                    .value,
              })
            }
            className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
          >
            {aggregationOptions.map(
              (option) => (
                <option
                  key={option}
                  value={option}
                >
                  {option}
                </option>
              ),
            )}
          </select>
        </div>
      )}
    </div>
  );
}


=== FILE: src/components/isa/etl/settings-panels/combine-panel.tsx ===
import { nodeDef } from "@/lib/etl-catalog";
import {
  getCombineInputs,
  getFilterColumnOptions,
  getJoinTypeOptions,
  joinPairConfigKeys,
} from "@/lib/etl-node-config";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";
import {
  PanelEmptyState,
  PanelLabel,
  PanelSelect,
} from "@/components/isa/etl/settings-panels/panel-controls";

/**
 * Pannello impostazioni per la famiglia "combine" (fase 4, punto 2).
 * Il contenuto dipende dal tipo di nodo combine (join/union/lookup —
 * modalità GIA' esistenti come EtlNodeDef distinti nel catalogo,
 * nessun nuovo tipo introdotto qui): la modalità "join" ottiene il
 * form dinamico completo richiesto dal brief; union/lookup restano
 * con i loro campi esistenti, resi dinamici dove lo schema lo
 * permette, così tutte le modalità della famiglia condividono lo
 * stesso pannello invece di UI separate e duplicate.
 */
export function CombinePanel({
  workflow,
  node,
  memberIds,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  /** Id dei membri della bubble del nodo (fase 2), incluso `node.id`; assente/singolo = nodo non raggruppato. */
  memberIds?: readonly string[] | undefined;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  if (node.type === "combine.join") {
    return (
      <JoinFields
        workflow={workflow}
        node={node}
        memberIds={memberIds}
        onConfigChange={onConfigChange}
      />
    );
  }

  if (node.type === "combine.lookup") {
    return (
      <LookupFields
        workflow={workflow}
        node={node}
        onConfigChange={onConfigChange}
      />
    );
  }

  return (
    <UnionFields
      node={node}
      onConfigChange={onConfigChange}
    />
  );
}

function JoinFields({
  workflow,
  node,
  memberIds,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  memberIds?: readonly string[] | undefined;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const inputs = getCombineInputs(
    workflow,
    node,
    memberIds,
  );

  const joinTypeOptions =
    getJoinTypeOptions();

  const joinType =
    node.config["how"] ||
    joinTypeOptions[0] ||
    "inner";

  if (inputs.length < 2) {
    return (
      <PanelEmptyState>
        Collega almeno 2 dataset per
        configurare la join (attualmente{" "}
        {inputs.length}
        ).
      </PanelEmptyState>
    );
  }

  const pairs = inputs
    .slice(0, -1)
    .map((left, index) => ({
      index,
      left,
      right: inputs[index + 1]!,
    }));

  return (
    <div className="space-y-3">
      <div>
        <PanelLabel>
          Tipo di join
        </PanelLabel>

        <select
          value={joinType}
          onChange={(event) =>
            onConfigChange({
              how: event.target
                .value,
            })
          }
          className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
        >
          {joinTypeOptions.map(
            (option) => (
              <option
                key={option}
                value={option}
              >
                {option}
              </option>
            ),
          )}
        </select>
      </div>

      {pairs.map(
        ({ index, left, right }) => {
          const keys =
            joinPairConfigKeys(index);

          return (
            <div
              key={`${left.nodeId}-${right.nodeId}`}
              className="glass-chip rounded-xl p-2.5"
            >
              <span className="block truncate text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                {left.title} ⋈{" "}
                {right.title}
              </span>

              <div className="mt-1.5 grid grid-cols-2 gap-2">
                <div>
                  <PanelLabel>
                    {left.title}
                  </PanelLabel>

                  <PanelSelect
                    value={
                      node.config[
                        keys.left
                      ] ?? ""
                    }
                    onChange={(
                      value,
                    ) =>
                      onConfigChange({
                        [keys.left]:
                          value,
                      })
                    }
                    options={
                      left.columns
                    }
                  />
                </div>

                <div>
                  <PanelLabel>
                    {right.title}
                  </PanelLabel>

                  <PanelSelect
                    value={
                      node.config[
                        keys.right
                      ] ?? ""
                    }
                    onChange={(
                      value,
                    ) =>
                      onConfigChange({
                        [keys.right]:
                          value,
                      })
                    }
                    options={
                      right.columns
                    }
                  />
                </div>
              </div>
            </div>
          );
        },
      )}
    </div>
  );
}

function LookupFields({
  workflow,
  node,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const inputColumns =
    getFilterColumnOptions(
      workflow,
      node,
    );

  return (
    <div className="space-y-3">
      <div>
        <PanelLabel>
          Chiave di lookup
        </PanelLabel>

        <PanelSelect
          value={
            node.config["key"] ?? ""
          }
          onChange={(value) =>
            onConfigChange({
              key: value,
            })
          }
          options={inputColumns}
        />
      </div>

      <div>
        <PanelLabel>
          Nuova colonna da aggiungere
        </PanelLabel>

        <input
          value={
            node.config["column"] ??
            ""
          }
          placeholder="category"
          onChange={(event) =>
            onConfigChange({
              column:
                event.target.value,
            })
          }
          className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
        />
      </div>
    </div>
  );
}

function UnionFields({
  node,
  onConfigChange,
}: {
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const modeOptions =
    nodeDef("combine.union")?.fields.find(
      (field) => field.key === "mode",
    )?.options ?? [];

  return (
    <div>
      <PanelLabel>
        Modalità
      </PanelLabel>

      <select
        value={
          node.config["mode"] ?? ""
        }
        onChange={(event) =>
          onConfigChange({
            mode: event.target.value,
          })
        }
        className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring"
      >
        {modeOptions.map((option) => (
          <option
            key={option}
            value={option}
          >
            {option}
          </option>
        ))}
      </select>
    </div>
  );
}


=== FILE: src/components/isa/etl/settings-panels/filter-panel.tsx ===
import {
  getColumnDomainValues,
  getFilterColumnOptions,
} from "@/lib/etl-node-config";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";
import {
  PanelChipToggleList,
  PanelLabel,
  PanelSelect,
} from "@/components/isa/etl/settings-panels/panel-controls";

/**
 * Pannello impostazioni per i nodi "Filter" (fase 4, punto 1):
 * colonna dallo schema in ingresso reale, output "solo colonna" vs
 * "intero dataset", valori da multi-select sul dominio della colonna
 * — mai testo libero.
 *
 * Config scritta: `column` (esistente, riusata), `value` (nuovo
 * significato: elenco dei valori selezionati, comma-joined — resta
 * compatibile con `nodeSummary`/Inspector, che lo trattano già come
 * stringa opaca), `outputMode` ("column" | "dataset", nuova chiave).
 */
export function FilterPanel({
  workflow,
  node,
  onConfigChange,
}: {
  workflow: EtlWorkflow;
  node: EtlNode;
  onConfigChange: (
    patch: Record<string, string>,
  ) => void;
}) {
  const columns = getFilterColumnOptions(
    workflow,
    node,
  );

  const selectedColumnName =
    node.config["column"] ?? "";

  const selectedColumn = columns.find(
    (c) => c.name === selectedColumnName,
  );

  const outputMode =
    node.config["outputMode"] === "column"
      ? "column"
      : "dataset";

  const selectedValues = (
    node.config["value"] ?? ""
  )
    .split(",")
    .map((v) => v.trim())
    .filter(Boolean);

  const domainValues = selectedColumn
    ? getColumnDomainValues(
        selectedColumn,
        `${node.id}:${selectedColumn.name}`,
      )
    : [];

  const toggleValue = (value: string) => {
    const next = selectedValues.includes(
      value,
    )
      ? selectedValues.filter(
          (v) => v !== value,
        )
      : [...selectedValues, value];

    onConfigChange({
      value: next.join(", "),
    });
  };

  return (
    <div className="space-y-3">
      <div>
        <PanelLabel>
          Colonna da filtrare
        </PanelLabel>

        <PanelSelect
          value={selectedColumnName}
          onChange={(value) =>
            onConfigChange({
              column: value,
              /* Cambiare colonna invalida i valori scelti sulla precedente. */
              value: "",
            })
          }
          options={columns}
          placeholder={
            columns.length
              ? "Seleziona colonna…"
              : "Nessuno schema in ingresso"
          }
        />
      </div>

      <div>
        <PanelLabel>
          Output
        </PanelLabel>

        <div className="mt-1.5 flex gap-1.5">
          {(
            [
              {
                key: "dataset",
                label: "Intero dataset filtrato",
              },
              {
                key: "column",
                label: "Solo quella colonna",
              },
            ] as const
          ).map((option) => (
            <button
              key={option.key}
              type="button"
              aria-pressed={
                outputMode ===
                option.key
              }
              onClick={() =>
                onConfigChange({
                  outputMode:
                    option.key,
                })
              }
              className={`flex-1 rounded-xl border px-2.5 py-1.5 text-[11px] font-medium transition ${
                outputMode ===
                option.key
                  ? "border-transparent gradient-brand text-brand-foreground"
                  : "glass-chip border-transparent text-muted-foreground hover:text-foreground"
              }`}
            >
              {option.label}
            </button>
          ))}
        </div>
      </div>

      <div>
        <PanelLabel>
          Valori da mantenere
        </PanelLabel>

        {selectedColumn ? (
          <PanelChipToggleList
            values={domainValues}
            selected={selectedValues}
            onToggle={toggleValue}
          />
        ) : (
          <p className="mt-1.5 text-[11px] text-muted-foreground">
            Scegli prima una colonna.
          </p>
        )}
      </div>
    </div>
  );
}


=== FILE: src/components/isa/etl/settings-panels/panel-controls.tsx ===
import { Check } from "lucide-react";
import type { ColumnDef } from "@/lib/etl-catalog";

/**
 * Controlli condivisi dai pannelli impostazioni della fase 4
 * (FilterPanel, CombineJoinPanel, AggregatePanel): stile coerente con
 * il resto del popover (`glass-chip`), niente testo libero dove il
 * brief chiede selettori guidati dallo schema.
 */

export function PanelLabel({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <span className="block text-[11px] font-medium text-muted-foreground">
      {children}
    </span>
  );
}

export function PanelSelect({
  value,
  onChange,
  options,
  placeholder = "Seleziona colonna…",
  disabled,
}: {
  value: string;
  onChange: (value: string) => void;
  options: ColumnDef[];
  placeholder?: string;
  disabled?: boolean;
}) {
  return (
    <select
      value={value}
      disabled={disabled}
      onChange={(event) =>
        onChange(event.target.value)
      }
      className="glass-chip mt-1 h-9 w-full rounded-xl px-2.5 text-xs outline-none focus:ring-2 focus:ring-ring disabled:opacity-50"
    >
      <option value="">
        {placeholder}
      </option>
      {options.map((column) => (
        <option
          key={column.name}
          value={column.name}
        >
          {column.name} · {column.type}
        </option>
      ))}
    </select>
  );
}

/** Lista di chip selezionabili (multi-select) invece di testo libero. */
export function PanelChipToggleList({
  values,
  selected,
  onToggle,
}: {
  values: string[];
  selected: string[];
  onToggle: (value: string) => void;
}) {
  if (values.length === 0) {
    return (
      <p className="mt-1.5 text-[11px] text-muted-foreground">
        Nessun valore disponibile.
      </p>
    );
  }

  return (
    <div className="mt-1.5 flex flex-wrap gap-1.5">
      {values.map((value) => {
        const isSelected =
          selected.includes(value);

        return (
          <button
            key={value}
            type="button"
            onClick={() =>
              onToggle(value)
            }
            aria-pressed={isSelected}
            className={`flex items-center gap-1 rounded-full border px-2.5 py-1 text-[11px] transition ${
              isSelected
                ? "border-transparent gradient-brand text-brand-foreground"
                : "glass-chip border-transparent text-muted-foreground hover:text-foreground"
            }`}
          >
            {isSelected && (
              <Check className="size-3" />
            )}
            {value}
          </button>
        );
      })}
    </div>
  );
}

export function PanelEmptyState({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <p className="px-1 py-2 text-[11px] leading-relaxed text-muted-foreground">
      {children}
    </p>
  );
}


=== FILE: src/components/isa/etl/tool-palette.tsx ===
import {
  AlignEndHorizontal,
  AlignEndVertical,
  AlignStartHorizontal,
  AlignStartVertical,
  Boxes,
  GripVertical,
  List,
  Shrink,
} from "lucide-react";
import { useRef, useState } from "react";
import { createPortal } from "react-dom";
import { IsaMenu, IsaMenuItem } from "@/components/isa/ui/isa-menu";
import type { EtlCategory, EtlNodeDef } from "@/lib/etl-catalog";
import { ETL_CATEGORIES, ETL_NODES, categoryAccent } from "@/lib/etl-catalog";
import { CATEGORY_ICONS } from "@/lib/etl-display";

export type Dock = "top" | "bottom" | "left" | "right";

function NodeChip({
  node,
  onAdd,
}: {
  node: EtlNodeDef;
  onAdd: (type: string) => void;
}) {
  const { type, label, description, Icon, category } = node;

  const handleDragStart = (event: React.DragEvent<HTMLButtonElement>) => {
    event.stopPropagation();

    event.dataTransfer.effectAllowed = "copy";
    event.dataTransfer.setData("application/isa-node", type);
    event.dataTransfer.setData("text/plain", `isa-node:${type}`);
  };

  const handleDragEnd = (event: React.DragEvent<HTMLButtonElement>) => {
    event.stopPropagation();
  };

  return (
    <button
      type="button"
      draggable={true}
      onDragStart={handleDragStart}
      onDragEnd={handleDragEnd}
      onClick={() => onAdd(type)}
      title={description}
      aria-label={`Aggiungi ${label}`}
      className="flex shrink-0 select-none items-center gap-1.5 rounded-lg px-2.5 py-1.5 text-left text-[12px] font-medium text-muted-foreground transition hover:bg-muted hover:text-foreground"
      style={{
        userSelect: "none",
        touchAction: "none",
      }}
    >
      <span
        className={`${categoryAccent(category)} flex size-5 shrink-0 items-center justify-center rounded-md`}
      >
        <Icon className="pointer-events-none size-3" />
      </span>

      <span className="pointer-events-none truncate">{label}</span>
    </button>
  );
}

export function ToolPalette({
  dock,
  onDockChange,
  onAdd,
}: {
  dock: Dock;
  onDockChange: (dock: Dock) => void;
  onAdd: (type: string) => void;
}) {
  const [open, setOpen] = useState<EtlCategory | null>(null);
  const [expanded, setExpanded] = useState(false);
  const [ghost, setGhost] = useState<{ x: number; y: number } | null>(null);
  const ref = useRef<HTMLDivElement>(null);
  const vertical = dock === "left" || dock === "right";
  const secondLevel = expanded || open !== null;
  const dragging = ghost !== null;

  const startDrag = (event: React.PointerEvent) => {
    event.preventDefault();
    event.stopPropagation();
    const workspace = ref.current?.closest<HTMLElement>("[data-palette-workspace]");
    if (!workspace) return;

    setGhost({ x: event.clientX, y: event.clientY });

    const move = (pointer: PointerEvent) => {
      setGhost({ x: pointer.clientX, y: pointer.clientY });
    };
    const finish = (pointer: PointerEvent) => {
      window.removeEventListener("pointermove", move);
      window.removeEventListener("pointerup", finish);
      setGhost(null);
      const rect = workspace.getBoundingClientRect();
      const distances: Array<[Dock, number]> = [
        ["left", Math.abs(pointer.clientX - rect.left)],
        ["right", Math.abs(rect.right - pointer.clientX)],
        ["top", Math.abs(pointer.clientY - rect.top)],
        ["bottom", Math.abs(rect.bottom - pointer.clientY)],
      ];
      distances.sort((a, b) => a[1] - b[1]);
      const nearest = distances[0]?.[0];
      if (nearest && nearest !== dock) onDockChange(nearest);
    };
    window.addEventListener("pointermove", move);
    window.addEventListener("pointerup", finish);
  };

  const categories = expanded
    ? ETL_CATEGORIES
    : ETL_CATEGORIES.filter((category) => category.key === open);

  const mainBar = (
    <div
      className={`glass-panel flex shrink-0 items-center gap-1 rounded-full p-1.5 shadow-lg transition-all duration-200 ${
        vertical ? "flex-col" : "flex-row"
      } ${dragging ? "scale-75 opacity-30" : "animate-scale-in"}`}
    >
      <button
        type="button"
        onPointerDown={startDrag}
        title="Trascina verso un bordo"
        aria-label="Sposta la barra delle risorse"
        className="flex size-8 cursor-grab items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground active:cursor-grabbing"
        style={{ touchAction: "none" }}
      >
        <GripVertical className={`size-4 ${vertical ? "" : "rotate-90"}`} />
      </button>

      {ETL_CATEGORIES.map((category) => {
        const Icon = CATEGORY_ICONS[category.key];
        const active = !expanded && open === category.key;
        return (
          <button
            key={category.key}
            type="button"
            onClick={() => {
              setExpanded(false);
              setOpen((current) => (current === category.key ? null : category.key));
            }}
            title={category.label}
            aria-label={category.label}
            aria-pressed={active}
            className={`flex size-8 items-center justify-center rounded-full transition ${
              active ? `${categoryAccent(category.key)} scale-105` : "text-muted-foreground hover:text-foreground"
            }`}
          >
            <Icon className="size-4" />
          </button>
        );
      })}

      <button
        type="button"
        onClick={() => {
          setExpanded((value) => !value);
          setOpen(null);
        }}
        aria-pressed={expanded}
        title={expanded ? "Riduci la barra" : "Espandi tutte le risorse"}
        aria-label={expanded ? "Riduci la barra" : "Espandi tutte le risorse"}
        className={`flex size-8 items-center justify-center rounded-full transition ${
          expanded ? "text-foreground" : "text-muted-foreground hover:text-foreground"
        }`}
      >
        {expanded ? <Shrink className="size-4" /> : <List className="size-4" />}
      </button>

      <span className={`bg-border ${vertical ? "my-0.5 h-px w-4" : "mx-0.5 h-4 w-px"}`} />

      <IsaMenu label="Posizione barra" Icon={vertical ? AlignStartVertical : AlignStartHorizontal}>
        {(close) => (
          <>
            <IsaMenuItem Icon={AlignStartHorizontal} label="Aggancia in alto" onClick={() => { onDockChange("top"); close(); }} />
            <IsaMenuItem Icon={AlignEndHorizontal} label="Aggancia in basso" onClick={() => { onDockChange("bottom"); close(); }} />
            <IsaMenuItem Icon={AlignStartVertical} label="Aggancia a sinistra" onClick={() => { onDockChange("left"); close(); }} />
            <IsaMenuItem Icon={AlignEndVertical} label="Aggancia a destra" onClick={() => { onDockChange("right"); close(); }} />
          </>
        )}
      </IsaMenu>
    </div>
  );

  const resources = secondLevel && !dragging ? (
    <div
      className={`glass-panel scroll-slim min-h-0 min-w-0 rounded-2xl p-1.5 shadow-lg ${
        vertical ? "max-h-[26rem] overflow-y-auto" : "max-w-full overflow-x-auto"
      }`}
    >
      <div className={vertical ? "flex flex-col gap-2" : "flex min-w-max items-start gap-3"}>
        {categories.map((category) => {
          const CategoryIcon = CATEGORY_ICONS[category.key];
          const nodes = ETL_NODES.filter((node) => node.category === category.key);
          return (
            <div key={category.key} className="min-w-0">
              {expanded && (
                <span className="flex items-center gap-1.5 px-2.5 py-1 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                  <CategoryIcon className="size-3" />
                  {category.label}
                </span>
              )}
              <div className={vertical ? "flex flex-col gap-0.5" : "flex items-center gap-0.5"}>
                {nodes.map((node) => <NodeChip key={node.type} node={node} onAdd={onAdd} />)}
              </div>
            </div>
          );
        })}
      </div>
    </div>
  ) : null;

  const reverse = dock === "bottom" || dock === "right";

  return (
    <>
      <aside
        key={`${dock}-${String(secondLevel)}`}
        ref={ref}
        aria-label="Risorse ETL"
        onPointerDown={(event) => event.stopPropagation()}
        className={`relative z-40 flex shrink-0 items-center justify-center gap-1.5 ${
          vertical
            ? `${reverse ? "flex-row-reverse" : "flex-row"} max-w-[19rem]`
            : `${reverse ? "flex-col-reverse" : "flex-col"} max-w-full`
        } p-1.5`}
      >
        {mainBar}
        {resources}
      </aside>

      {ghost &&
        createPortal(
          <div
            aria-hidden
            className="glass-panel animate-scale-in pointer-events-none fixed z-50 flex size-11 items-center justify-center rounded-full shadow-xl"
            style={{ left: ghost.x - 22, top: ghost.y - 22 }}
          >
            <Boxes className="size-4 text-muted-foreground" />
          </div>,
          document.body,
        )}
    </>
  );
}


=== FILE: src/components/isa/etl/workflow-canvas.tsx ===
import {
  Copy,
  Database,
  Eye,
  Grid3x3,
  LayoutGrid,
  Maximize2,
  MoreHorizontal,
  Move,
  Plus,
  Trash2,
  Ungroup,
  Unlink,
  ZoomIn,
  ZoomOut,
} from "lucide-react";
import {
  useCallback,
  useEffect,
  useMemo,
  useRef,
  useState,
} from "react";
import { createPortal } from "react-dom";

import { ToolPalette } from "@/components/isa/etl/tool-palette";
import type { Dock } from "@/components/isa/etl/tool-palette";
import { AggregatePanel } from "@/components/isa/etl/settings-panels/aggregate-panel";
import { CombinePanel } from "@/components/isa/etl/settings-panels/combine-panel";
import { FilterPanel } from "@/components/isa/etl/settings-panels/filter-panel";
import {
  IsaMenu,
  IsaMenuCheckItem,
  IsaMenuItem,
} from "@/components/isa/ui/isa-menu";
import {
  categoryAccent,
  nodeDef,
  nodeSummary,
} from "@/lib/etl-catalog";
import type { EtlDisplaySettings } from "@/lib/etl-display";
import { getSettingsPanelKind } from "@/lib/etl-node-config";
import {
  DEFAULT_DISPLAY,
  DISPLAY_OPTIONS,
  getTransformCardLayout,
} from "@/lib/etl-display";
import type { BubbleGeometry } from "@/lib/etl-bubble";
import { computeBubbles } from "@/lib/etl-bubble";
import {
  BUBBLE_AUTO_MOVE_TRANSITION,
  CARD_AUTO_MOVE_TRANSITION,
  EDGE_AUTO_MOVE_TRANSITION,
} from "@/lib/etl-motion";
import {
  analyzeNode,
  formatRows,
} from "@/lib/etl-schema";
import type {
  EtlNode,
  EtlWorkflow,
  LayoutMode,
  NodeStatus,
} from "@/lib/etl-workflow";

export const NODE_W = 128;
export const NODE_H = 128;

const MIN_NODE_SIZE = 112;
const MIN_NODE_HEIGHT = 88;
const MAX_NODE_SIZE = 248;
const ROUTE_GAP = 24;

/*
 * Raggio (px, coordinate superficie) con cui vengono arrotondati i
 * vertici del path SVG delle frecce, puramente in fase di rendering —
 * i waypoint restano quelli calcolati da `getBestRoute`, qui si
 * smussa solo il tratto disegnato. Tenuto sotto PORT_STUB così
 * l'arrotondamento non "mangia" mai lo stub rettilineo di aggancio.
 */
const EDGE_CORNER_RADIUS = 10;

/*
 * Tratto rettilineo obbligatorio con cui una freccia entra/esce da una
 * card: garantisce che l'aggancio sia sempre perpendicolare al lato e
 * che la linea non corra mai lungo il bordo del box.
 */
const PORT_STUB = 20;

/* Penalità per un percorso che appoggia/attraversa una card. */
const ROUTE_BLOCKED = 100_000;

/* Spazio minimo lasciato tra due card quando si "respingono". */
const COLLISION_GAP = 8;

/* Margine di sicurezza attorno a una card "ostacolo" nel routing frecce. */
const OBSTACLE_MARGIN = 8;

/*
 * Raggio (in px, coordinate superficie) entro cui una card viene
 * considerata un possibile ostacolo per il routing di un edge: evita di
 * testare l'intersezione contro OGNI nodo del canvas quando ce ne sono
 * molti, limitandosi a quelli realmente vicini al percorso diretto.
 */
const OBSTACLE_SEARCH_MARGIN = 160;

/*
 * Due card "transform" entrano in modalità combine quando il gap tra i
 * loro bounding box scende sotto questa soglia (in px, coordinate
 * superficie).
 */
const COMBINE_GAP = 24;

/*
 * Trascinando un nodo che fa parte di un gruppo, superata questa
 * distanza (in px) dalla sua posizione di partenza il nodo si stacca
 * dal gruppo invece di trascinarlo rigidamente con sé.
 */
const DETACH_THRESHOLD = 48;

/* Padding visivo del contenitore che racchiude un gruppo di card. */
const GROUP_PADDING = 14;

type Side =
  | "top"
  | "right"
  | "bottom"
  | "left";

type Point = {
  x: number;
  y: number;
};

type Anchor = Point & {
  side: Side;
};

type Pending = {
  node: string;
  port: string;
  x: number;
  y: number;
  targetNode: string | null;
} | null;

type NodeSize = {
  width: number;
  height: number;
};

/** Snapshot di una card stazionaria catturato all'inizio di un drag. */
type BaseNode = Point &
  NodeSize & {
    id: string;
  };

type NodeGeometry = EtlNode & {
  width: number;
  height: number;
};

type Rect = {
  x: number;
  y: number;
  width: number;
  height: number;
};

type ContextMenuState = {
  open: boolean;
  x: number;
  y: number;
  nodeId: string | null;
};

const STATUS: Record<
  NodeStatus,
  {
    label: string;
    color: string;
  }
> = {
  ready: {
    label: "Ready",
    color: "var(--muted-foreground)",
  },
  running: {
    label: "Running",
    color: "var(--brand)",
  },
  succeeded: {
    label: "Succeeded",
    color: "var(--success)",
  },
  error: {
    label: "Error",
    color: "var(--destructive)",
  },
};

/* -------------------------------------------------------------------------- */
/*                            CARD DIMENSION                                  */
/* -------------------------------------------------------------------------- */

/**
 * Altezza della card per un dato `width` e un dato `showDetail` (mostrare
 * o no la riga di dettaglio/formula). Isolata dal resto di
 * `estimateNodeSize` perché serve calcolarla due volte: una con la
 * regola di visibilità "reale" del nodo, una con quella dei nodi
 * `sources` (per il lato del quadrato dei nodi transform, vedi sotto).
 */
function estimateCardHeight(
  node: EtlNode,
  display: EtlDisplaySettings,
  width: number,
  detail: string,
  showDetail: boolean,
): number {
  const CARD_PADDING = 12;
  const BODY_GAP = 12;

  /*
   * Header: icona (36px) affiancata a titolo + label.
   * Il titolo può andare a capo: stimiamo le righe in base
   * allo spazio orizzontale realmente disponibile.
   */
  const headerTextWidth = Math.max(
    40,
    width - 36 - 8 - 28 - CARD_PADDING * 2,
  );

  const titleLines = Math.min(
    3,
    Math.max(
      1,
      Math.ceil(
        (node.title.length * 6.4) /
          headerTextWidth,
      ),
    ),
  );

  const HEADER_HEIGHT = Math.max(
    36,
    titleLines * 15 + 12,
  );

  let bodyHeight = 0;

  if (showDetail && detail) {
    const charsPerLine = Math.max(
      1,
      Math.floor(
        (width -
          CARD_PADDING * 2 -
          20) /
          5.2,
      ),
    );

    const lines = Math.min(
      4,
      Math.max(
        1,
        Math.ceil(
          detail.length / charsPerLine,
        ),
      ),
    );

    bodyHeight += 16 + lines * 14;
  }

  if (display.metrics) {
    bodyHeight +=
      (bodyHeight > 0 ? 8 : 0) + 24;
  }

  if (display.status) {
    bodyHeight +=
      (bodyHeight > 0 ? 6 : 0) + 20;
  }

  if (bodyHeight === 0) {
    /* Solo la label di fallback. */
    bodyHeight = 16;
  }

  return Math.min(
    MAX_NODE_SIZE,
    Math.max(
      MIN_NODE_HEIGHT,
      Math.ceil(
        (CARD_PADDING * 2 +
          HEADER_HEIGHT +
          BODY_GAP +
          bodyHeight) /
          8,
      ) * 8,
    ),
  );
}

function estimateNodeSize(
  node: EtlNode,
  workflow: EtlWorkflow,
  display: EtlDisplaySettings,
): NodeSize {
  const def = nodeDef(node.type);

  if (!def) {
    return {
      width: NODE_W,
      height: MIN_NODE_HEIGHT,
    };
  }

  const analysis = analyzeNode(
    workflow,
    node,
  );

  const detail = nodeSummary(
    node.type,
    node.config,
  );

  const isSource =
    def.category === "sources";

  const showDetail =
    (display.source && isSource) ||
    (display.formula && !isSource);

  const metricsText = display.metrics
    ? `${formatRows(analysis.rows)} rows · ${analysis.columns.length} cols`
    : "";

  /* ------------------------------------------------------------------ */
  /*  LARGHEZZA — guidata dal testo più lungo (header / metriche).      */
  /* ------------------------------------------------------------------ */

  const longestText = Math.max(
    node.title.length,
    def.label.length,
    metricsText.length,
    8,
  );

  const textWidth = Math.min(
    210,
    Math.max(
      90,
      longestText * 6.2,
    ),
  );

  /*
   * Header: icona (size-9) + gap + testo + menu (size-7) + padding card.
   */
  const headerWidth =
    textWidth + 36 + 8 + 28 + 24;

  const width = Math.min(
    MAX_NODE_SIZE,
    Math.max(
      MIN_NODE_SIZE,
      Math.ceil(headerWidth / 8) * 8,
    ),
  );

  /* ------------------------------------------------------------------ */
  /*  ALTEZZA — si adatta al contenuto reale, senza spazio in eccesso.  */
  /* ------------------------------------------------------------------ */

  if (!isSource) {
    /*
     * Card "transform" (ogni categoria diversa da sources): sempre
     * quadrata, con lato pari all'altezza che avrebbe una card Dataset
     * con lo stesso livello di dettaglio visualizzato — non un valore
     * fisso, ma la stessa `estimateCardHeight` usata per i nodi
     * sources, così i due tipi restano visivamente allineati anche se
     * cambiano i display settings.
     */
    const side = estimateCardHeight(
      node,
      display,
      width,
      detail,
      display.source,
    );

    return {
      width: side,
      height: side,
    };
  }

  const height = estimateCardHeight(
    node,
    display,
    width,
    detail,
    showDetail,
  );

  return {
    width,
    height,
  };
}

/* -------------------------------------------------------------------------- */
/*                               GEOMETRY                                     */
/* -------------------------------------------------------------------------- */

function getAnchor(
  node: NodeGeometry,
  side: Side,
): Anchor {
  const halfW = node.width / 2;
  const halfH = node.height / 2;

  switch (side) {
    case "top":
      return {
        side,
        x: node.x + halfW,
        y: node.y,
      };

    case "right":
      return {
        side,
        x: node.x + node.width,
        y: node.y + halfH,
      };

    case "bottom":
      return {
        side,
        x: node.x + halfW,
        y: node.y + node.height,
      };

    case "left":
      return {
        side,
        x: node.x,
        y: node.y + halfH,
      };
  }
}

/* -------------------------------------------------------------------------- */
/*                             COLLISION                                      */
/* -------------------------------------------------------------------------- */

function rectsOverlap(
  a: Rect,
  b: Rect,
  gap = COLLISION_GAP,
): boolean {
  return (
    a.x < b.x + b.width + gap &&
    a.x + a.width + gap > b.x &&
    a.y < b.y + b.height + gap &&
    a.y + a.height + gap > b.y
  );
}

/**
 * Spinge `moving` fuori da `obstacle` lungo l'asse di minima penetrazione.
 * Restituisce `null` se non c'è sovrapposizione (nessuno spostamento
 * necessario), altrimenti la nuova posizione di `moving`.
 *
 * A differenza della vecchia `avoidCollisions`, qui è sempre `moving` a
 * cedere il passo: la card trascinata (l'obstacle, quando la si chiama
 * durante il drag) resta libera di seguire il puntatore senza mai
 * fermarsi.
 */
function pushOutOfOverlap(
  moving: Rect,
  obstacle: Rect,
  gap = COLLISION_GAP,
): Point | null {
  if (
    !rectsOverlap(
      moving,
      obstacle,
      gap,
    )
  ) {
    return null;
  }

  const pushRight =
    obstacle.x +
    obstacle.width +
    gap -
    moving.x;

  const pushLeft =
    moving.x +
    moving.width +
    gap -
    obstacle.x;

  const pushDown =
    obstacle.y +
    obstacle.height +
    gap -
    moving.y;

  const pushUp =
    moving.y +
    moving.height +
    gap -
    obstacle.y;

  const minPush = Math.min(
    pushLeft,
    pushRight,
    pushUp,
    pushDown,
  );

  if (minPush === pushLeft) {
    return {
      x:
        obstacle.x -
        moving.width -
        gap,
      y: moving.y,
    };
  }

  if (minPush === pushRight) {
    return {
      x:
        obstacle.x +
        obstacle.width +
        gap,
      y: moving.y,
    };
  }

  if (minPush === pushUp) {
    return {
      x: moving.x,
      y:
        obstacle.y -
        moving.height -
        gap,
    };
  }

  return {
    x: moving.x,
    y:
      obstacle.y +
      obstacle.height +
      gap,
  };
}

function distance(
  a: Point,
  b: Point,
): number {
  return (
    Math.abs(a.x - b.x) +
    Math.abs(a.y - b.y)
  );
}

/** Il più piccolo rettangolo che racchiude tutti i `rects`. */
function unionRect(
  rects: Rect[],
): Rect {
  const first = rects[0];

  if (!first) {
    return {
      x: 0,
      y: 0,
      width: 0,
      height: 0,
    };
  }

  let minX = first.x;
  let minY = first.y;
  let maxX = first.x + first.width;
  let maxY = first.y + first.height;

  for (const rect of rects) {
    minX = Math.min(minX, rect.x);
    minY = Math.min(minY, rect.y);
    maxX = Math.max(
      maxX,
      rect.x + rect.width,
    );
    maxY = Math.max(
      maxY,
      rect.y + rect.height,
    );
  }

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}

/**
 * Tra i nodi `candidates` (già filtrati a card "transform" non
 * coinvolte nel drag in corso), quello il cui bounding box è a meno di
 * `gap` px da `movingRect` — cioè il candidato di "combine" più vicino
 * al gruppo che si sta trascinando. `null` se nessuno è abbastanza
 * vicino.
 */
function pickCombineCandidate(
  movingRect: Rect,
  candidates: NodeGeometry[],
  gap: number,
): NodeGeometry | null {
  let best: NodeGeometry | null =
    null;

  let bestDistance =
    Number.POSITIVE_INFINITY;

  const movingCenter = {
    x:
      movingRect.x +
      movingRect.width / 2,
    y:
      movingRect.y +
      movingRect.height / 2,
  };

  for (const candidate of candidates) {
    if (
      !rectsOverlap(
        movingRect,
        candidate,
        gap,
      )
    ) {
      continue;
    }

    const candidateDistance =
      distance(movingCenter, {
        x:
          candidate.x +
          candidate.width / 2,
        y:
          candidate.y +
          candidate.height / 2,
      });

    if (candidateDistance < bestDistance) {
      best = candidate;
      bestDistance = candidateDistance;
    }
  }

  return best;
}

function simplifyPath(
  points: Point[],
): Point[] {
  const result: Point[] = [];

  for (const point of points) {
    const previous =
      result[result.length - 1];

    if (
      previous &&
      previous.x === point.x &&
      previous.y === point.y
    ) {
      continue;
    }

    const beforePrevious =
      result[result.length - 2];

    if (
      beforePrevious &&
      previous &&
      beforePrevious.x ===
        previous.x &&
      previous.x === point.x
    ) {
      result[result.length - 1] =
        point;
      continue;
    }

    if (
      beforePrevious &&
      previous &&
      beforePrevious.y ===
        previous.y &&
      previous.y === point.y
    ) {
      result[result.length - 1] =
        point;
      continue;
    }

    result.push(point);
  }

  return result;
}

function pathFromPoints(
  points: Point[],
): string {
  return points
    .map(
      (point, index) =>
        index === 0
          ? `M ${point.x} ${point.y}`
          : `L ${point.x} ${point.y}`,
    )
    .join(" ");
}

/**
 * Stessa sequenza di waypoint di `pathFromPoints`, ma con i vertici
 * interni smussati (quadratic bezier) invece che ad angolo vivo: solo
 * resa visiva più morbida del path già calcolato, nessun ricalcolo del
 * routing/anchor. Primo e ultimo punto (gli anchor sui bordi delle
 * card) restano invariati.
 */
function smoothPathFromPoints(
  points: Point[],
  radius: number = EDGE_CORNER_RADIUS,
): string {
  const first = points[0];

  if (!first || points.length < 3) {
    return pathFromPoints(points);
  }

  let d = `M ${first.x} ${first.y}`;

  for (
    let i = 1;
    i < points.length - 1;
    i += 1
  ) {
    const prev = points[i - 1]!;
    const curr = points[i]!;
    const next = points[i + 1]!;

    const toPrev = {
      x: prev.x - curr.x,
      y: prev.y - curr.y,
    };

    const toNext = {
      x: next.x - curr.x,
      y: next.y - curr.y,
    };

    const lenPrev = Math.hypot(
      toPrev.x,
      toPrev.y,
    );

    const lenNext = Math.hypot(
      toNext.x,
      toNext.y,
    );

    if (
      lenPrev < 0.01 ||
      lenNext < 0.01
    ) {
      d += ` L ${curr.x} ${curr.y}`;
      continue;
    }

    const r = Math.min(
      radius,
      lenPrev / 2,
      lenNext / 2,
    );

    const enter = {
      x:
        curr.x +
        (toPrev.x / lenPrev) * r,
      y:
        curr.y +
        (toPrev.y / lenPrev) * r,
    };

    const exit = {
      x:
        curr.x +
        (toNext.x / lenNext) * r,
      y:
        curr.y +
        (toNext.y / lenNext) * r,
    };

    d += ` L ${enter.x} ${enter.y} Q ${curr.x} ${curr.y} ${exit.x} ${exit.y}`;
  }

  const last =
    points[points.length - 1]!;

  d += ` L ${last.x} ${last.y}`;

  return d;
}

/** Versore uscente, perpendicolare al lato. */
function sideNormal(
  side: Side,
): Point {
  switch (side) {
    case "top":
      return { x: 0, y: -1 };
    case "right":
      return { x: 1, y: 0 };
    case "bottom":
      return { x: 0, y: 1 };
    case "left":
      return { x: -1, y: 0 };
  }
}

/** Punto a `dist` dall'anchor, uscente perpendicolare al lato. */
function stubPoint(
  anchor: Anchor,
  dist: number,
): Point {
  const n = sideNormal(anchor.side);

  return {
    x: anchor.x + n.x * dist,
    y: anchor.y + n.y * dist,
  };
}

/** Il segmento [a,b] (ortogonale) interseca il rettangolo del nodo? */
function segmentHitsNode(
  a: Point,
  b: Point,
  node: NodeGeometry,
  pad: number,
): boolean {
  const left = node.x - pad;
  const right = node.x + node.width + pad;
  const top = node.y - pad;
  const bottom = node.y + node.height + pad;

  const minX = Math.min(a.x, b.x);
  const maxX = Math.max(a.x, b.x);
  const minY = Math.min(a.y, b.y);
  const maxY = Math.max(a.y, b.y);

  return (
    minX < right &&
    maxX > left &&
    minY < bottom &&
    maxY > top
  );
}

/**
 * Card diverse da `from`/`to` che si trovano abbastanza vicine al
 * rettangolo che racchiude i due nodi dell'edge da poter finire in
 * mezzo al percorso. Filtrare qui evita di testare l'intersezione
 * contro OGNI nodo del canvas quando i nodi sono molti.
 */
function nearbyObstacles(
  from: NodeGeometry,
  to: NodeGeometry,
  allNodes: NodeGeometry[],
  /*
   * Id extra da escludere dagli ostacoli oltre a from.id/to.id — serve
   * quando from/to sono in realtà il rettangolo sintetico di una
   * bubble (fase 2): gli id reali delle card membro vanno esclusi
   * esplicitamente, perché from.id/to.id sono un id sintetico
   * (`bubble:<groupId>`) che non combacia con nessuna card reale.
   */
  excludeIds?: ReadonlySet<string>,
): NodeGeometry[] {
  const minX =
    Math.min(from.x, to.x) -
    OBSTACLE_SEARCH_MARGIN;

  const maxX =
    Math.max(
      from.x + from.width,
      to.x + to.width,
    ) + OBSTACLE_SEARCH_MARGIN;

  const minY =
    Math.min(from.y, to.y) -
    OBSTACLE_SEARCH_MARGIN;

  const maxY =
    Math.max(
      from.y + from.height,
      to.y + to.height,
    ) + OBSTACLE_SEARCH_MARGIN;

  return allNodes.filter(
    (node) =>
      node.id !== from.id &&
      node.id !== to.id &&
      !excludeIds?.has(node.id) &&
      node.x < maxX &&
      node.x + node.width > minX &&
      node.y < maxY &&
      node.y + node.height > minY,
  );
}

/**
 * Tra gli `obstacles`, quello il cui bounding box interseca
 * l'ingombro rettangolare del segmento [a,b]: è la card che più
 * probabilmente sta bloccando il percorso diretto. In caso di più
 * candidati si sceglie quello con il centro più vicino al punto
 * medio del segmento.
 */
function pickBlockingObstacle(
  a: Point,
  b: Point,
  obstacles: NodeGeometry[],
): NodeGeometry | null {
  const minX = Math.min(a.x, b.x);
  const maxX = Math.max(a.x, b.x);
  const minY = Math.min(a.y, b.y);
  const maxY = Math.max(a.y, b.y);

  const overlapping = obstacles.filter(
    (node) =>
      node.x < maxX &&
      node.x + node.width > minX &&
      node.y < maxY &&
      node.y + node.height > minY,
  );

  if (overlapping.length === 0) {
    return null;
  }

  const midX = (a.x + b.x) / 2;
  const midY = (a.y + b.y) / 2;

  return overlapping.reduce(
    (closest, node) => {
      const nodeDist = distance(
        {
          x: node.x + node.width / 2,
          y: node.y + node.height / 2,
        },
        { x: midX, y: midY },
      );

      const closestDist = distance(
        {
          x:
            closest.x +
            closest.width / 2,
          y:
            closest.y +
            closest.height / 2,
        },
        { x: midX, y: midY },
      );

      return nodeDist < closestDist
        ? node
        : closest;
    },
  );
}

/**
 * Punti di deviazione (in stile "U") che portano il percorso attorno
 * al bounding box di `obstacle`, uscendo sopra/sotto/a-sinistra/a-
 * destra di esso. Non è un vero pathfinding: è un set aggiuntivo di
 * candidati manhattan usato solo quando le shape "dirette" finiscono
 * tutte per attraversare una card.
 */
function detourShapesAround(
  s: Point,
  e: Point,
  obstacle: NodeGeometry,
): Point[][] {
  const left =
    obstacle.x - OBSTACLE_MARGIN;

  const right =
    obstacle.x +
    obstacle.width +
    OBSTACLE_MARGIN;

  const top =
    obstacle.y - OBSTACLE_MARGIN;

  const bottom =
    obstacle.y +
    obstacle.height +
    OBSTACLE_MARGIN;

  return [
    /* Sopra l'ostacolo. */
    [
      { x: s.x, y: top },
      { x: e.x, y: top },
    ],
    /* Sotto l'ostacolo. */
    [
      { x: s.x, y: bottom },
      { x: e.x, y: bottom },
    ],
    /* A sinistra dell'ostacolo. */
    [
      { x: left, y: s.y },
      { x: left, y: e.y },
    ],
    /* A destra dell'ostacolo. */
    [
      { x: right, y: s.y },
      { x: right, y: e.y },
    ],
  ];
}

/**
 * Calcola un percorso ortogonale tra due anchor.
 *
 * La freccia esce sempre PERPENDICOLARE dal lato di partenza e arriva
 * PERPENDICOLARE al lato di destinazione: i primi/ultimi `PORT_STUB` px
 * sono un tratto dritto obbligato, così la linea non si appoggia mai sul
 * bordo del box. Nessuna curva, nessun arrowhead: solo segmenti H/V.
 *
 * `obstacles` sono le altre card (né `from` né `to`) da evitare: se
 * tutte le shape "dirette" finiscono per attraversarne una, si prova
 * anche un set di percorsi che deviano attorno all'ostacolo più
 * vicino al tragitto.
 */
function routeCandidate(
  from: NodeGeometry,
  to: NodeGeometry,
  fromSide: Side,
  toSide: Side,
  obstacles: NodeGeometry[] = [],
) {
  const start = getAnchor(
    from,
    fromSide,
  );

  const end = getAnchor(
    to,
    toSide,
  );

  const s = stubPoint(
    start,
    PORT_STUB,
  );

  const e = stubPoint(
    end,
    PORT_STUB,
  );

  const middleX = Math.round(
    (s.x + e.x) / 2,
  );

  const middleY = Math.round(
    (s.y + e.y) / 2,
  );

  const shapes: Point[][] = [
    [{ x: e.x, y: s.y }],
    [{ x: s.x, y: e.y }],
    [
      { x: middleX, y: s.y },
      { x: middleX, y: e.y },
    ],
    [
      { x: s.x, y: middleY },
      { x: e.x, y: middleY },
    ],
  ];

  /*
   * Se il lato scelto non "guarda" verso l'altro nodo, la freccia
   * dovrebbe girare attorno alla card: percorso da scartare.
   */
  const fromCenter = {
    x: from.x + from.width / 2,
    y: from.y + from.height / 2,
  };

  const toCenter = {
    x: to.x + to.width / 2,
    y: to.y + to.height / 2,
  };

  const fromN = sideNormal(fromSide);
  const toN = sideNormal(toSide);

  const fromFacesTarget =
    fromN.x *
      (toCenter.x - fromCenter.x) +
    fromN.y *
      (toCenter.y - fromCenter.y);

  const toFacesSource =
    toN.x *
      (fromCenter.x - toCenter.x) +
    toN.y *
      (fromCenter.y - toCenter.y);

  const orientationPenalty =
    (fromFacesTarget < 0
      ? ROUTE_BLOCKED
      : 0) +
    (toFacesSource < 0
      ? ROUTE_BLOCKED
      : 0);

  const evaluate = (
    rawPoints: Point[],
  ) => {
    const points =
      simplifyPath(rawPoints);

    let length = 0;
    let blocked = 0;
    let obstacleHits = 0;

    for (
      let i = 1;
      i < points.length;
      i += 1
    ) {
      const previousPoint = points[i - 1];
      const currentPoint = points[i];

      if (!previousPoint || !currentPoint) {
        continue;
      }

      length += distance(
        previousPoint,
        currentPoint,
      );

      /*
       * Un segmento non deve attraversare una card. Si escludono i
       * due tratti-stub, che toccano per forza il proprio nodo.
       */
      const isFirst = i === 1;
      const isLast =
        i === points.length - 1;

      if (
        !isFirst &&
        segmentHitsNode(
          previousPoint,
          currentPoint,
          from,
          2,
        )
      ) {
        blocked += ROUTE_BLOCKED;
      }

      if (
        !isLast &&
        segmentHitsNode(
          previousPoint,
          currentPoint,
          to,
          2,
        )
      ) {
        blocked += ROUTE_BLOCKED;
      }

      /*
       * Un segmento non deve nemmeno attraversare (con un margine di
       * sicurezza) una card terza che si trova in mezzo al percorso.
       */
      for (const obstacle of obstacles) {
        if (
          segmentHitsNode(
            previousPoint,
            currentPoint,
            obstacle,
            OBSTACLE_MARGIN,
          )
        ) {
          obstacleHits += 1;
        }
      }
    }

    const bends =
      Math.max(
        0,
        points.length - 2,
      );

    return {
      points,
      obstacleHits,
      score:
        length +
        bends * ROUTE_GAP +
        blocked +
        obstacleHits *
          ROUTE_BLOCKED +
        orientationPenalty,
    };
  };

  let scored = shapes
    .map((mid) => [
      start,
      s,
      ...mid,
      e,
      end,
    ])
    .map(evaluate);

  const anyObstacleFree =
    scored.some(
      (candidate) =>
        candidate.obstacleHits === 0,
    );

  if (
    !anyObstacleFree &&
    obstacles.length > 0
  ) {
    const blocking =
      pickBlockingObstacle(
        s,
        e,
        obstacles,
      ) ??
      pickBlockingObstacle(
        start,
        end,
        obstacles,
      );

    if (blocking) {
      const detourCandidates =
        detourShapesAround(
          s,
          e,
          blocking,
        ).map((mid) => [
          start,
          s,
          ...mid,
          e,
          end,
        ]);

      scored = scored.concat(
        detourCandidates.map(
          evaluate,
        ),
      );
    }
  }

  scored.sort(
    (a, b) =>
      a.score - b.score,
  );

  const best = scored[0];

  if (!best) {
    return {
      fromSide,
      toSide,
      points: [
        getAnchor(from, fromSide),
        getAnchor(to, toSide),
      ],
      score: Number.POSITIVE_INFINITY,
    };
  }

  return {
    fromSide,
    toSide,
    points: best.points,
    score: best.score,
  };
}

function getBestRoute(
  from: NodeGeometry,
  to: NodeGeometry,
  allNodes: NodeGeometry[] = [],
  excludeIds?: ReadonlySet<string>,
) {
  const sides: Side[] = [
    "top",
    "right",
    "bottom",
    "left",
  ];

  const obstacles = nearbyObstacles(
    from,
    to,
    allNodes,
    excludeIds,
  );

  const routes: Array<
    ReturnType<typeof routeCandidate>
  > = [];

  for (const fromSide of sides) {
    for (const toSide of sides) {
      routes.push(
        routeCandidate(
          from,
          to,
          fromSide,
          toSide,
          obstacles,
        ),
      );
    }
  }

  routes.sort(
    (a, b) =>
      a.score - b.score,
  );

  return routes[0];
}

/**
 * Adatta una card membro + la geometria della sua bubble (fase 2) in
 * un NodeGeometry sintetico che rappresenta l'intera bubble ai fini
 * del routing: stessa forma di un nodo singolo, ma x/y/width/height
 * presi dal bounding box della bubble, espanso dello stesso
 * GROUP_PADDING con cui è disegnato il contenitore — così la freccia
 * tocca visivamente il bordo disegnato, non il bounding box "nudo"
 * dei soli membri. I campi non geometrici (type/title/config/status)
 * sono presi dal membro passato: servono solo a soddisfare il tipo
 * NodeGeometry, getAnchor/getBestRoute leggono solo
 * x/y/width/height/id.
 */
function bubbleNodeGeometry(
  member: NodeGeometry,
  bubble: BubbleGeometry,
): NodeGeometry {
  return {
    ...member,
    id: `bubble:${bubble.groupId}`,
    x:
      bubble.rect.x -
      GROUP_PADDING,
    y:
      bubble.rect.y -
      GROUP_PADDING,
    width:
      bubble.rect.width +
      GROUP_PADDING * 2,
    height:
      bubble.rect.height +
      GROUP_PADDING * 2,
  };
}

/**
 * Punto visuale mostrato quando il cursore
 * entra in una card durante il linking.
 *
 * Preferenza: angolo alto a destra.
 * Se troppo vicino al bordo destro:
 * angolo alto a sinistra.
 */
function getDropIndicatorPoint(
  node: NodeGeometry,
  surfaceW: number,
  surfaceH: number,
): Point {
  const margin = 12;

  let x =
    node.x +
    node.width -
    margin;

  let y =
    node.y + margin;

  if (
    x + 7 >
    surfaceW
  ) {
    x =
      node.x + margin;
  }

  if (
    y - 7 <
    0
  ) {
    y =
      node.y +
      node.height -
      margin;
  }

  if (
    x - 7 <
    0
  ) {
    x =
      node.x +
      node.width -
      margin;
  }

  if (
    y + 7 >
    surfaceH
  ) {
    y =
      node.y + margin;
  }

  return {
    x,
    y,
  };
}


/**
 * Distribuzione dei port su uno specifico lato.
 */
function portOffset(
  index: number,
  count: number,
  size: number,
): number {
  if (count <= 1) {
    return size / 2;
  }

  const padding = 18;

  return (
    padding +
    index *
      ((size -
        padding * 2) /
        (count - 1))
  );
}

/* -------------------------------------------------------------------------- */
/*                         CONTEXT MENU                                       */
/* -------------------------------------------------------------------------- */

function CanvasContextMenu({
  state,
  boundaryRef,
  grid,
  onClose,
  onAddDataset,
  onFitView,
  onToggleGrid,
  onAutoLayout,
  onDuplicateNode,
  onRemoveNode,
  onUnlinkNode,
}: {
  state: ContextMenuState;
  boundaryRef: React.RefObject<HTMLDivElement | null>;
  grid: boolean;
  onClose: () => void;
  onAddDataset: () => void;
  onFitView: () => void;
  onToggleGrid: () => void;
  onAutoLayout: () => void;
  onDuplicateNode: (
    id: string,
  ) => void;
  onRemoveNode: (
    id: string,
  ) => void;
  onUnlinkNode: (
    id: string,
  ) => void;
}) {
  const menuRef =
    useRef<HTMLDivElement>(null);

  const [
    position,
    setPosition,
  ] = useState<Point>({
    x: state.x,
    y: state.y,
  });

  useEffect(() => {
    if (!state.open) {
      return;
    }

    const handleOutsidePointer =
      (event: PointerEvent) => {
        if (
          !menuRef.current?.contains(
            event.target as Node,
          )
        ) {
          onClose();
        }
      };

    const handleEscape = (
      event: KeyboardEvent,
    ) => {
      if (
        event.key === "Escape"
      ) {
        onClose();
      }
    };

    document.addEventListener(
      "pointerdown",
      handleOutsidePointer,
    );

    document.addEventListener(
      "keydown",
      handleEscape,
    );

    return () => {
      document.removeEventListener(
        "pointerdown",
        handleOutsidePointer,
      );

      document.removeEventListener(
        "keydown",
        handleEscape,
      );
    };
  }, [
    state.open,
    onClose,
  ]);

  useEffect(() => {
    if (!state.open) {
      return;
    }

    const boundary =
      boundaryRef.current;

    const menu =
      menuRef.current;

    if (!boundary || !menu) {
      return;
    }

    const boundaryRect =
      boundary.getBoundingClientRect();

    const menuRect =
      menu.getBoundingClientRect();

    const margin = 8;

    const maxX =
      Math.max(
        boundaryRect.left + margin,
        boundaryRect.right -
          menuRect.width -
          margin,
      );

    const maxY =
      Math.max(
        boundaryRect.top + margin,
        boundaryRect.bottom -
          menuRect.height -
          margin,
      );

    setPosition({
      x: Math.min(
        Math.max(
          state.x,
          boundaryRect.left +
            margin,
        ),
        maxX,
      ),
      y: Math.min(
        Math.max(
          state.y,
          boundaryRect.top +
            margin,
        ),
        maxY,
      ),
    });
  }, [
    state.open,
    state.x,
    state.y,
    boundaryRef,
  ]);

  if (
    !state.open ||
    typeof document ===
      "undefined"
  ) {
    return null;
  }

  return createPortal(
    <div
      ref={menuRef}
      role="menu"
      aria-label="Menu contestuale IsA"
      className="fixed z-100 w-56 overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl"
      style={{
        left: position.x,
        top: position.y,
      }}
      onContextMenu={(event) =>
        event.preventDefault()
      }
      onPointerDown={(event) =>
        event.stopPropagation()
      }
    >
      {state.nodeId ? (
        <>
          <IsaMenuItem
            Icon={Copy}
            label="Duplica nodo"
            onClick={() => {
              onDuplicateNode(
                state.nodeId!,
              );
              onClose();
            }}
          />

          <IsaMenuItem
            Icon={Unlink}
            label="Scollega input"
            onClick={() => {
              onUnlinkNode(
                state.nodeId!,
              );
              onClose();
            }}
          />

          <IsaMenuItem
            Icon={Trash2}
            label="Elimina nodo"
            danger
            onClick={() => {
              onRemoveNode(
                state.nodeId!,
              );
              onClose();
            }}
          />

          <span className="my-1 block h-px bg-border" />
        </>
      ) : null}

      <IsaMenuItem
        Icon={Plus}
        label="Aggiungi dataset"
        onClick={() => {
          onAddDataset();
          onClose();
        }}
      />

      <IsaMenuItem
        Icon={LayoutGrid}
        label="Disponi automaticamente"
        onClick={() => {
          onAutoLayout();
          onClose();
        }}
      />

      <IsaMenuItem
        Icon={Maximize2}
        label="Adatta vista"
        onClick={() => {
          onFitView();
          onClose();
        }}
      />

      <IsaMenuItem
        Icon={Grid3x3}
        label={
          grid
            ? "Nascondi griglia"
            : "Mostra griglia"
        }
        onClick={() => {
          onToggleGrid();
          onClose();
        }}
      />
    </div>,
    document.body,
  );
}

/* -------------------------------------------------------------------------- */
/*                         MAIN COMPONENT                                     */
/* -------------------------------------------------------------------------- */

export function WorkflowCanvas({
  workflow,
  selectedId,
  onSelect,
  onMove,
  onAdd,
  onAddAt,
  onConnect,
  onRemoveNode,
  onDuplicateNode,
  onGroupNodes,
  onUngroupNode,
  onRemoveEdge,
  onAddDataset,
  onLayoutChange,
  onUpdateNodeConfig,
}: {
  workflow: EtlWorkflow;
  selectedId: string | null;
  onSelect: (
    id: string | null,
  ) => void;
  onMove: (
    id: string,
    x: number,
    y: number,
    commit?: boolean,
  ) => void;
  onAdd: (
    type: string,
  ) => void;
  onAddAt: (
    type: string,
    x: number,
    y: number,
  ) => void;
  onConnect: (
    fromNode: string,
    fromPort: string,
    toNode: string,
    toPort: string,
  ) => void;
  onRemoveNode: (
    id: string,
  ) => void;
  onDuplicateNode: (
    id: string,
  ) => void;
  onGroupNodes: (
    ids: string[],
  ) => void;
  onUngroupNode: (
    id: string,
  ) => void;
  onRemoveEdge: (
    id: string,
  ) => void;
  onAddDataset: () => void;
  onLayoutChange: (
    mode: LayoutMode,
  ) => void;
  /** Fase 4: scrive nel config del nodo dai pannelli impostazioni (merge, non replace). */
  onUpdateNodeConfig: (
    id: string,
    patch: Record<string, string>,
  ) => void;
}) {
  const boxRef =
    useRef<HTMLDivElement>(null);

  const surfaceRef =
    useRef<HTMLDivElement>(null);

  const [
    pending,
    setPending,
  ] = useState<Pending>(null);

  const [
    contextMenu,
    setContextMenu,
  ] =
    useState<ContextMenuState>({
      open: false,
      x: 0,
      y: 0,
      nodeId: null,
    });

  const [
    zoom,
    setZoom,
  ] = useState(1);

  const [
    hoverEdge,
    setHoverEdge,
  ] = useState<string | null>(
    null,
  );

  const [
    grid,
    setGrid,
  ] = useState(true);

  const [
    display,
    setDisplay,
  ] =
    useState<EtlDisplaySettings>(
      DEFAULT_DISPLAY,
    );

  const [
    box,
    setBox,
  ] = useState({
    w: 1200,
    h: 720,
  });

  const [
    paletteDock,
    setPaletteDock,
  ] = useState<Dock>("top");

  const paletteRef =
    useRef<HTMLDivElement>(
      null,
    );

  const [
    paletteBox,
    setPaletteBox,
  ] = useState({
    w: 0,
    h: 0,
  });

  const dragRef =
    useRef<{
      id: string;
      pointerId: number;
      offsetX: number;
      offsetY: number;
      width: number;
      height: number;
      element: HTMLDivElement | null;
      /* Posizione della card trascinata all'inizio del drag: serve a
       * misurare quanto si è allontanata, per la soglia di distacco
       * dal gruppo. */
      primaryStart: Point;
      /* groupId del nodo trascinato all'inizio del drag (se ne aveva
       * uno). Non cambia durante il drag: il distacco è per-sessione. */
      groupId: string | undefined;
      /* Altri membri dello stesso gruppo, con offset RIGIDO rispetto
       * alla card trascinata: finché il gruppo non si stacca, si
       * spostano insieme ad essa di questo stesso delta. */
      memberOffsets: Map<
        string,
        {
          dx: number;
          dy: number;
          width: number;
          height: number;
        }
      >;
      /* Una volta staccato dal gruppo (soglia superata), resta
       * staccato per il resto di questa sessione di drag. */
      detached: boolean;
      /* Snapshot di TUTTE le altre card (inclusi gli altri membri del
       * gruppo), preso all'inizio del drag: è la base immutabile da
       * cui ricalcolare gli spostamenti anti-sovrapposizione a ogni
       * frame (così tornano al proprio posto non appena non servono
       * più). Finché un membro del gruppo si muove rigidamente con la
       * card trascinata viene escluso dal calcolo anti-sovrapposizione;
       * torna a essere un "ostacolo" normale appena si stacca.
       */
      basePositions: BaseNode[];
      /* Ultima posizione che QUESTO drag ha applicato a ciascuna delle
       * altre card (sia per l'anti-sovrapposizione sia per il
       * movimento rigido del gruppo): serve solo a capire, frame per
       * frame, quali onMove vanno effettivamente emessi. */
      livePositions: Map<string, Point>;
    } | null>(null);

  const [
    combinePreview,
    setCombinePreview,
  ] = useState<{
    movingIds: string[];
    targetId: string;
  } | null>(null);

  useEffect(() => {
    const el =
      paletteRef.current;

    if (!el) {
      return;
    }

    const ro =
      new ResizeObserver(() =>
        setPaletteBox({
          w: el.offsetWidth,
          h: el.offsetHeight,
        }),
      );

    ro.observe(el);

    setPaletteBox({
      w: el.offsetWidth,
      h: el.offsetHeight,
    });

    return () =>
      ro.disconnect();
  }, []);

  useEffect(() => {
    const el =
      boxRef.current;

    if (!el) {
      return;
    }

    const ro =
      new ResizeObserver(() =>
        setBox({
          w: el.clientWidth,
          h: el.clientHeight,
        }),
      );

    ro.observe(el);

    setBox({
      w: el.clientWidth,
      h: el.clientHeight,
    });

    return () =>
      ro.disconnect();
  }, []);

  const surfaceW =
    Math.max(
      320,
      box.w / zoom,
    );

  const surfaceH =
    Math.max(
      280,
      box.h / zoom,
    );

  /*
   * Banda occupata dalla palette (in coordinate superficie): serve a
   * tenere le card fuori dall'ingombro della toolbar.
   */
  const paletteBand = useMemo(() => {
    const vertical =
      paletteDock === "left" ||
      paletteDock === "right";

    return {
      w: vertical
        ? (paletteBox.w + 12) / zoom
        : 0,
      h: vertical
        ? 0
        : (paletteBox.h + 12) / zoom,
    };
  }, [
    paletteDock,
    paletteBox.w,
    paletteBox.h,
    zoom,
  ]);

  /*
   * Posizione "reale" di una card: un solo punto di verità usato sia
   * per il rendering della card sia per gli endpoint delle frecce e per
   * l'hit-test dei collegamenti. Clampa dentro la superficie ed esclude
   * la banda della palette, così box e frecce non possono desincronizzarsi.
   */
  const placeNode = useCallback(
    (
      x: number,
      y: number,
      width: number,
      height: number,
    ): Point => {
      let nx = Math.min(
        Math.max(0, x),
        Math.max(0, surfaceW - width),
      );

      let ny = Math.min(
        Math.max(0, y),
        Math.max(0, surfaceH - height),
      );

      if (paletteDock === "left") {
        nx = Math.max(
          nx,
          Math.min(
            paletteBand.w,
            Math.max(0, surfaceW - width),
          ),
        );
      } else if (
        paletteDock === "right"
      ) {
        nx = Math.min(
          nx,
          Math.max(
            0,
            surfaceW -
              width -
              paletteBand.w,
          ),
        );
      } else if (
        paletteDock === "top"
      ) {
        ny = Math.max(
          ny,
          Math.min(
            paletteBand.h,
            Math.max(0, surfaceH - height),
          ),
        );
      } else if (
        paletteDock === "bottom"
      ) {
        ny = Math.min(
          ny,
          Math.max(
            0,
            surfaceH -
              height -
              paletteBand.h,
          ),
        );
      }

      return { x: nx, y: ny };
    },
    [
      surfaceW,
      surfaceH,
      paletteDock,
      paletteBand,
    ],
  );

  const toLocal = useCallback(
    (
      clientX: number,
      clientY: number,
    ): Point => {
      const rect =
        surfaceRef.current?.getBoundingClientRect();

      return {
        x:
          (clientX -
            (rect?.left ??
              0)) /
          zoom,
        y:
          (clientY -
            (rect?.top ??
              0)) /
          zoom,
      };
    },
    [zoom],
  );

  const nodeSizes =
    useMemo(
      () => {
        const sizes: Record<
          string,
          NodeSize
        > = {};

        for (
          const node of workflow.nodes
        ) {
          sizes[node.id] =
            estimateNodeSize(
              node,
              workflow,
              display,
            );
        }

        return sizes;
      },
      [
        workflow,
        display,
      ],
    );

  const getSize =
    useCallback(
      (id: string): NodeSize =>
        nodeSizes[id] ?? {
          width: NODE_W,
          height: MIN_NODE_HEIGHT,
        },
      [nodeSizes],
    );

  /*
   * Geometria corrente delle card. Ricalcolata a ogni render dalle
   * posizioni CORRENTI di workflow.nodes: è l'unica fonte di verità sia
   * per il rendering delle card sia per il routing delle frecce, quindi
   * le frecce seguono sempre la posizione reale (clampata) del nodo.
   */
  const visibleNodes =
    useMemo(
      () =>
        workflow.nodes.map(
          (node): NodeGeometry => {
            const {
              width,
              height,
            } = getSize(node.id);

            const { x, y } =
              placeNode(
                node.x,
                node.y,
                width,
                height,
              );

            return {
              ...node,
              width,
              height,
              x,
              y,
            };
          },
        ),
      [
        workflow.nodes,
        getSize,
        placeNode,
      ],
    );

  /*
   * Specchio della geometria in un ref, sempre aggiornato al render più
   * recente. I listener globali di startLink (pointermove/pointerup)
   * sopravvivono ai render: devono leggere QUI, non da un array
   * catturato nella closure alla creazione del listener, altrimenti i
   * nodi creati dopo non risultano agganciabili.
   */
  const nodesRef =
    useRef<NodeGeometry[]>(
      visibleNodes,
    );

  nodesRef.current = visibleNodes;

  const nodeById = useCallback(
    (id: string) =>
      visibleNodes.find(
        (node) =>
          node.id === id,
      ),
    [visibleNodes],
  );

  const liveNodeById = useCallback(
    (id: string) =>
      nodesRef.current.find(
        (node) =>
          node.id === id,
      ),
    [],
  );

  /**
   * Dato il rettangolo della card trascinata e uno snapshot delle altre
   * card (preso all'inizio del drag), calcola dove ognuna di esse deve
   * spostarsi in questo istante per non sovrapporsi né alla card
   * trascinata né, a cascata, alle altre card già spostate.
   *
   * Ricalcolando sempre a partire dallo snapshot iniziale (non dalla
   * posizione del frame precedente) il risultato è una funzione pura
   * della posizione corrente del puntatore: le card spostate tornano
   * esattamente al loro posto originale non appena smettono di essere
   * in collisione, senza accumulare deriva frame dopo frame.
   */
  const resolveDisplacedPositions =
    useCallback(
      (
        draggedRect: Rect,
        baseNodes: BaseNode[],
      ): Map<string, Point> => {
        const positions = new Map<
          string,
          Point
        >();

        for (const n of baseNodes) {
          positions.set(n.id, {
            x: n.x,
            y: n.y,
          });
        }

        /* Passo 1: spinge fuori dalla card trascinata. */
        for (const n of baseNodes) {
          const pos =
            positions.get(n.id)!;

          const rect: Rect = {
            x: pos.x,
            y: pos.y,
            width: n.width,
            height: n.height,
          };

          const pushed =
            pushOutOfOverlap(
              rect,
              draggedRect,
            );

          if (pushed) {
            positions.set(
              n.id,
              placeNode(
                pushed.x,
                pushed.y,
                n.width,
                n.height,
              ),
            );
          }
        }

        /*
         * Passo 2: propaga a catena le collisioni che restano tra le
         * card stazionarie spostate. Un numero limitato di iterazioni
         * (al più una per nodo) evita loop infiniti: non è una vera
         * simulazione fisica, ma risolve il caso comune di più card
         * spinte in cascata.
         */
        for (
          let pass = 0;
          pass < baseNodes.length;
          pass += 1
        ) {
          let changed = false;

          for (const a of baseNodes) {
            const aPos =
              positions.get(a.id)!;

            const aRect: Rect = {
              x: aPos.x,
              y: aPos.y,
              width: a.width,
              height: a.height,
            };

            for (const b of baseNodes) {
              if (a.id === b.id) {
                continue;
              }

              const bPos =
                positions.get(
                  b.id,
                )!;

              const bRect: Rect = {
                x: bPos.x,
                y: bPos.y,
                width: b.width,
                height: b.height,
              };

              const pushed =
                pushOutOfOverlap(
                  bRect,
                  aRect,
                );

              if (!pushed) {
                continue;
              }

              const clamped =
                placeNode(
                  pushed.x,
                  pushed.y,
                  b.width,
                  b.height,
                );

              if (
                clamped.x !==
                  bPos.x ||
                clamped.y !== bPos.y
              ) {
                positions.set(
                  b.id,
                  clamped,
                );

                changed = true;
              }
            }
          }

          if (!changed) {
            break;
          }
        }

        return positions;
      },
      [placeNode],
    );

  /* ---------------------------------------------------------------------- */
  /*                              FIT VIEW                                   */
  /* ---------------------------------------------------------------------- */

  const fitView = useCallback(
    () => {
      if (
        workflow.nodes.length ===
        0
      ) {
        setZoom(1);
        return;
      }

      const maxX =
        Math.max(
          ...visibleNodes.map(
            (node) =>
              node.x +
              node.width,
          ),
        ) + 40;

      const maxY =
        Math.max(
          ...visibleNodes.map(
            (node) =>
              node.y +
              node.height,
          ),
        ) + 40;

      setZoom(
        Math.min(
          1.4,
          Math.max(
            0.4,
            Math.min(
              box.w / maxX,
              box.h / maxY,
            ),
          ),
        ),
      );
    },
    [
      workflow.nodes.length,
      visibleNodes,
      box.w,
      box.h,
    ],
  );

  /* ---------------------------------------------------------------------- */
  /*                              NODE DRAG                                  */
  /* ---------------------------------------------------------------------- */

  type DragState = NonNullable<
    (typeof dragRef)["current"]
  >;

  /**
   * Un frame di drag: dove va la card trascinata, quali altri membri
   * del gruppo la seguono ancora rigidamente, come si risolvono le
   * collisioni anti-sovrapposizione contro tutte le altre card (i
   * membri ancora rigidi NON sono ostacoli), e se c'è un candidato
   * "combine" abbastanza vicino da mostrare/confermare.
   *
   * Muta `drag.detached` quando la soglia di distacco viene superata:
   * una volta staccato, resta staccato per il resto del drag.
   */
  const computeDragFrame = useCallback(
    (
      drag: DragState,
      point: Point,
    ) => {
      /*
       * La card trascinata segue sempre liberamente il puntatore:
       * nessun anti-sovrapposizione applicato a lei, solo il clamp ai
       * bordi del canvas già gestito da placeNode.
       */
      const primaryDesired = placeNode(
        point.x - drag.offsetX,
        point.y - drag.offsetY,
        drag.width,
        drag.height,
      );

      if (
        !drag.detached &&
        drag.groupId &&
        Math.hypot(
          primaryDesired.x -
            drag.primaryStart.x,
          primaryDesired.y -
            drag.primaryStart.y,
        ) > DETACH_THRESHOLD
      ) {
        drag.detached = true;
      }

      const rigidOtherIds =
        drag.groupId &&
        !drag.detached
          ? Array.from(
              drag.memberOffsets.keys(),
            )
          : [];

      const movingRects: Rect[] = [
        {
          x: primaryDesired.x,
          y: primaryDesired.y,
          width: drag.width,
          height: drag.height,
        },
        ...rigidOtherIds.map(
          (id) => {
            const off =
              drag.memberOffsets.get(
                id,
              )!;

            return {
              x:
                primaryDesired.x +
                off.dx,
              y:
                primaryDesired.y +
                off.dy,
              width: off.width,
              height: off.height,
            };
          },
        ),
      ];

      const draggedUnion =
        unionRect(movingRects);

      const rigidSet = new Set(
        rigidOtherIds,
      );

      const obstaclesForPush =
        drag.basePositions.filter(
          (n) =>
            !rigidSet.has(n.id),
        );

      const resolvedObstacles =
        obstaclesForPush.length > 0
          ? resolveDisplacedPositions(
              draggedUnion,
              obstaclesForPush,
            )
          : new Map<string, Point>();

      /*
       * Candidato "combine": solo tra card non-sources ("transform"),
       * e solo tra quelle NON già in movimento rigido con questa.
       */
      const primaryNode = nodeById(
        drag.id,
      );

      const primaryDef = primaryNode
        ? nodeDef(primaryNode.type)
        : undefined;

      const isTransformLike =
        !!primaryDef &&
        primaryDef.category !==
          "sources";

      let combineTarget: NodeGeometry | null =
        null;

      if (isTransformLike) {
        const movingIdsNow = new Set([
          drag.id,
          ...rigidOtherIds,
        ]);

        const candidates =
          nodesRef.current.filter(
            (n) => {
              if (
                movingIdsNow.has(
                  n.id,
                )
              ) {
                return false;
              }

              const def = nodeDef(
                n.type,
              );

              return (
                !!def &&
                def.category !==
                  "sources"
              );
            },
          );

        combineTarget =
          pickCombineCandidate(
            draggedUnion,
            candidates,
            COMBINE_GAP,
          );
      }

      return {
        primaryDesired,
        rigidOtherIds,
        obstaclesForPush,
        resolvedObstacles,
        combineTarget,
      };
    },
    [
      placeNode,
      resolveDisplacedPositions,
      nodeById,
    ],
  );

  const startDragNode = (
    event: React.PointerEvent<HTMLDivElement>,
    nodeId: string,
  ) => {
    if (
      event.button !== 0
    ) {
      return;
    }

    const target =
      event.target as HTMLElement;

    if (
      target.closest(
        "[data-node-control]",
      ) ||
      target.closest(
        "[data-in-port]",
      ) ||
      target.closest(
        "[data-out-port]",
      )
    ) {
      return;
    }

    const node =
      nodeById(nodeId);

    if (!node) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    closeContextMenu();

    onSelect(nodeId);

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    const basePositions: BaseNode[] =
      nodesRef.current
        .filter(
          (other) =>
            other.id !== nodeId,
        )
        .map((other) => ({
          id: other.id,
          x: other.x,
          y: other.y,
          width: other.width,
          height: other.height,
        }));

    const livePositions = new Map(
      basePositions.map(
        (other) => [
          other.id,
          { x: other.x, y: other.y },
        ],
      ),
    );

    const memberOffsets = new Map<
      string,
      {
        dx: number;
        dy: number;
        width: number;
        height: number;
      }
    >();

    if (node.groupId) {
      for (const other of nodesRef.current) {
        if (
          other.id === nodeId ||
          other.groupId !==
            node.groupId
        ) {
          continue;
        }

        memberOffsets.set(
          other.id,
          {
            dx: other.x - node.x,
            dy: other.y - node.y,
            width: other.width,
            height: other.height,
          },
        );
      }
    }

    dragRef.current = {
      id: nodeId,
      pointerId:
        event.pointerId,
      offsetX:
        point.x - node.x,
      offsetY:
        point.y - node.y,
      width: node.width,
      height: node.height,
      element:
        event.currentTarget,
      primaryStart: {
        x: node.x,
        y: node.y,
      },
      groupId: node.groupId,
      memberOffsets,
      detached: false,
      basePositions,
      livePositions,
    };

    try {
      event.currentTarget.setPointerCapture(
        event.pointerId,
      );
    } catch {
      /* Pointer Capture non disponibile. */
    }
  };

  const moveDragNode = (
    event: React.PointerEvent<HTMLDivElement>,
  ) => {
    const drag =
      dragRef.current;

    if (
      !drag ||
      drag.pointerId !==
        event.pointerId
    ) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    const frame = computeDragFrame(
      drag,
      point,
    );

    onMove(
      drag.id,
      frame.primaryDesired.x,
      frame.primaryDesired.y,
    );

    for (const id of frame.rigidOtherIds) {
      const off =
        drag.memberOffsets.get(id)!;

      const x =
        frame.primaryDesired.x +
        off.dx;

      const y =
        frame.primaryDesired.y +
        off.dy;

      drag.livePositions.set(id, {
        x,
        y,
      });

      onMove(id, x, y);
    }

    for (const entry of frame.obstaclesForPush) {
      const pos =
        frame.resolvedObstacles.get(
          entry.id,
        );

      const live =
        drag.livePositions.get(
          entry.id,
        );

      if (!pos || !live) {
        continue;
      }

      if (
        pos.x !== live.x ||
        pos.y !== live.y
      ) {
        drag.livePositions.set(
          entry.id,
          pos,
        );

        onMove(
          entry.id,
          pos.x,
          pos.y,
        );
      }
    }

    setCombinePreview(
      (current) => {
        if (!frame.combineTarget) {
          return current
            ? null
            : current;
        }

        const movingIds = [
          drag.id,
          ...frame.rigidOtherIds,
        ];

        if (
          current &&
          current.targetId ===
            frame.combineTarget.id &&
          current.movingIds
            .length ===
            movingIds.length &&
          current.movingIds.every(
            (id, index) =>
              id ===
              movingIds[index],
          )
        ) {
          return current;
        }

        return {
          movingIds,
          targetId:
            frame.combineTarget.id,
        };
      },
    );
  };

  const endDragNode = (
    event: React.PointerEvent<HTMLDivElement>,
  ) => {
    const drag =
      dragRef.current;

    if (
      !drag ||
      drag.pointerId !==
        event.pointerId
    ) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    const frame = computeDragFrame(
      drag,
      point,
    );

    onMove(
      drag.id,
      frame.primaryDesired.x,
      frame.primaryDesired.y,
      true,
    );

    for (const id of frame.rigidOtherIds) {
      const off =
        drag.memberOffsets.get(id)!;

      onMove(
        id,
        frame.primaryDesired.x +
          off.dx,
        frame.primaryDesired.y +
          off.dy,
        true,
      );
    }

    /*
     * Commit anche per le card spostate per effetto domino, ma solo
     * quelle davvero finite fuori dalla loro posizione di partenza:
     * evita voci di history superflue per i nodi mai toccati.
     */
    for (const entry of frame.obstaclesForPush) {
      const pos =
        frame.resolvedObstacles.get(
          entry.id,
        ) ?? {
          x: entry.x,
          y: entry.y,
        };

      if (
        pos.x !== entry.x ||
        pos.y !== entry.y
      ) {
        onMove(
          entry.id,
          pos.x,
          pos.y,
          true,
        );
      }
    }

    /*
     * Distacco dal gruppo di partenza: va risolto PRIMA di un eventuale
     * nuovo combine, altrimenti groupNodes vedrebbe ancora il vecchio
     * groupId di questo nodo e vi trascinerebbe dentro anche i membri
     * da cui ci si è appena staccati.
     */
    if (drag.detached && drag.groupId) {
      onUngroupNode(drag.id);
    }

    if (frame.combineTarget) {
      const movingIds = [
        drag.id,
        ...frame.rigidOtherIds,
      ];

      const targetGroupId =
        frame.combineTarget.groupId;

      const targetGroupMembers =
        targetGroupId
          ? nodesRef.current
              .filter(
                (n) =>
                  n.groupId ===
                  targetGroupId,
              )
              .map((n) => n.id)
          : [];

      const fullIds = Array.from(
        new Set([
          ...movingIds,
          frame.combineTarget.id,
          ...targetGroupMembers,
        ]),
      );

      onGroupNodes(fullIds);
    }

    setCombinePreview(null);

    try {
      if (
        event.currentTarget.hasPointerCapture(
          event.pointerId,
        )
      ) {
        event.currentTarget.releasePointerCapture(
          event.pointerId,
        );
      }
    } catch {
      /* no-op */
    }

    dragRef.current =
      null;
  };

  /* ---------------------------------------------------------------------- */
  /*                              LINK DRAG                                  */
  /* ---------------------------------------------------------------------- */

  const startLink = (
    event: React.PointerEvent<HTMLSpanElement>,
    nodeId: string,
    port: string,
  ) => {
    if (
      event.button !== 0
    ) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    closeContextMenu();

    try {
      event.currentTarget.setPointerCapture(
        event.pointerId,
      );
    } catch {
      /* no-op */
    }

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    setPending({
      node: nodeId,
      port,
      x: point.x,
      y: point.y,
      targetNode: null,
    });

    const move = (
      moveEvent: PointerEvent,
    ) => {
      const next =
        toLocal(
          moveEvent.clientX,
          moveEvent.clientY,
        );

      const element =
        document.elementFromPoint(
          moveEvent.clientX,
          moveEvent.clientY,
        );

      const card =
        element?.closest<HTMLElement>(
          "[data-node-card]",
        );

      const targetId =
        card?.dataset["nodeId"];

      const target =
        targetId
          ? liveNodeById(targetId)
          : undefined;

      if (
        target &&
        target.id !== nodeId
      ) {
        setPending(
          (current) =>
            current
              ? {
                  ...current,
                  x: next.x,
                  y: next.y,
                  targetNode:
                    target.id,
                }
              : current,
        );

        return;
      }

      setPending(
        (current) =>
          current
            ? {
                ...current,
                x: next.x,
                y: next.y,
                targetNode: null,
              }
            : current,
      );
    };

    const up = (
      upEvent: PointerEvent,
    ) => {
      const element =
        document.elementFromPoint(
          upEvent.clientX,
          upEvent.clientY,
        );

      /*
       * Non è più necessario rilasciare esattamente sopra la porta:
       * basta essere sopra una qualsiasi card valida. Il collegamento
       * usa la porta di input più vicina al punto di rilascio.
       */
      const card =
        element?.closest<HTMLElement>(
          "[data-node-card]",
        );

      const targetId =
        card?.dataset["nodeId"];

      const target = targetId
        ? liveNodeById(targetId)
        : undefined;

      if (
        target &&
        target.id !== nodeId
      ) {
        const targetDef = nodeDef(
          target.type,
        );

        const inputs =
          targetDef?.inputs ?? [];

        if (inputs.length > 0) {
          const drop = toLocal(
            upEvent.clientX,
            upEvent.clientY,
          );

          let inPort = inputs[0]!;

          if (inputs.length > 1) {
            const ratio =
              (drop.y - target.y) /
              Math.max(
                1,
                target.height,
              );

            const index = Math.min(
              inputs.length - 1,
              Math.max(
                0,
                Math.round(
                  ratio *
                    (inputs.length -
                      1),
                ),
              ),
            );

            inPort =
              inputs[index] ??
              inPort;
          }

          onConnect(
            nodeId,
            port,
            target.id,
            inPort,
          );
        }
      }

      window.removeEventListener(
        "pointermove",
        move,
      );

      window.removeEventListener(
        "pointerup",
        up,
      );

      window.removeEventListener(
        "pointercancel",
        up,
      );

      setPending(null);
    };

    window.addEventListener(
      "pointermove",
      move,
    );

    window.addEventListener(
      "pointerup",
      up,
    );

    window.addEventListener(
      "pointercancel",
      up,
    );
  };

  /* ---------------------------------------------------------------------- */
  /*                          CONTEXT MENU STATE                             */
  /* ---------------------------------------------------------------------- */

  const closeContextMenu =
    useCallback(() => {
      setContextMenu(
        (current) => ({
          ...current,
          open: false,
        }),
      );
    }, []);

  const openContextMenu = (
    event: React.MouseEvent,
  ) => {
    event.preventDefault();
    event.stopPropagation();

    const card =
      (event.target as HTMLElement).closest<HTMLElement>(
        "[data-node-card]",
      );

    const nodeId =
      card?.dataset["nodeId"] ??
      null;

    if (nodeId) {
      onSelect(nodeId);
    }

    setContextMenu({
      open: true,
      x: event.clientX,
      y: event.clientY,
      nodeId,
    });
  };

  const unlinkNode =
    useCallback(
      (nodeId: string) => {
        workflow.edges
          .filter(
            (edge) =>
              edge.toNode ===
              nodeId,
          )
          .forEach(
            (edge) =>
              onRemoveEdge(
                edge.id,
              ),
          );
      },
      [
        workflow.edges,
        onRemoveEdge,
      ],
    );

  /* ---------------------------------------------------------------------- */
  /*                              PALETTE                                    */
  /* ---------------------------------------------------------------------- */

  const dockPosition: Record<
    Dock,
    string
  > = {
    top:
      "left-1/2 top-0 -translate-x-1/2",
    bottom:
      "bottom-0 left-1/2 -translate-x-1/2",
    left:
      "left-0 top-1/2 -translate-y-1/2",
    right:
      "right-0 top-1/2 -translate-y-1/2",
  };

  const palette = (
    <div
      ref={paletteRef}
      className={`absolute z-40 ${dockPosition[paletteDock]}`}
    >
      <ToolPalette
        dock={paletteDock}
        onDockChange={
          setPaletteDock
        }
        onAdd={onAdd}
      />
    </div>
  );

  /* ---------------------------------------------------------------------- */
  /*                            PENDING ROUTE                                */
  /* ---------------------------------------------------------------------- */

  const pendingRoute =
    useMemo(() => {
      if (!pending) {
        return null;
      }

      const source =
        nodeById(
          pending.node,
        );

      if (!source) {
        return null;
      }

      if (
        pending.targetNode
      ) {
        const target =
          nodeById(
            pending.targetNode,
          );

        if (target) {
          const route =
            getBestRoute(
              source,
              target,
              visibleNodes,
            );

          const indicator =
            getDropIndicatorPoint(
              target,
              surfaceW,
              surfaceH,
            );

          return {
            path: route
              ? pathFromPoints(
                  route.points,
                )
              : pathFromPoints([
                  getAnchor(
                    source,
                    "right",
                  ),
                  indicator,
                ]),
            indicator,
          };
        }
      }

      /*
       * Trascinamento libero (nessun target): la linea esce comunque
       * perpendicolare dal lato della card rivolto verso il puntatore.
       */
      const cx =
        source.x + source.width / 2;
      const cy =
        source.y + source.height / 2;

      const dx = pending.x - cx;
      const dy = pending.y - cy;

      const sourceSide: Side =
        Math.abs(dx) >= Math.abs(dy)
          ? dx >= 0
            ? "right"
            : "left"
          : dy >= 0
            ? "bottom"
            : "top";

      const sourceAnchor =
        getAnchor(
          source,
          sourceSide,
        );

      const stub = stubPoint(
        sourceAnchor,
        PORT_STUB,
      );

      const horizontalFirst =
        sourceSide === "left" ||
        sourceSide === "right";

      return {
        path: pathFromPoints(
          simplifyPath([
            sourceAnchor,
            stub,
            horizontalFirst
              ? {
                  x: pending.x,
                  y: stub.y,
                }
              : {
                  x: stub.x,
                  y: pending.y,
                },
            {
              x: pending.x,
              y: pending.y,
            },
          ]),
        ),
        indicator: null,
      };
    }, [
      pending,
      nodeById,
      visibleNodes,
      surfaceW,
      surfaceH,
    ]);

  /* ---------------------------------------------------------------------- */
  /*                         GROUP CONTAINERS                                */
  /* ---------------------------------------------------------------------- */

  /*
   * Geometria di ogni bubble (gruppo con 2+ membri), indicizzata per
   * groupId — ricalcolata a ogni render dalle posizioni CORRENTI dei
   * nodi (fase 2, punto 4: bounding box/orientamento sempre coerenti
   * con l'ultima disposizione, senza bisogno di gestire esplicitamente
   * "ingresso/uscita dalla bubble" come evento a parte). Usata sia per
   * disegnare il contenitore sia, più sotto, per instradare le frecce
   * esterne sul suo perimetro invece che sui singoli nodi interni.
   */
  const bubbles = useMemo(
    () => computeBubbles(visibleNodes),
    [visibleNodes],
  );

  /** Un box per ogni bubble esistente, per il contenitore visivo. */
  const groupBoxes = useMemo(
    () =>
      Array.from(
        bubbles.values(),
      ).map((bubble) => ({
        groupId: bubble.groupId,
        rect: bubble.rect,
        orientation:
          bubble.orientation,
        memberIds:
          bubble.memberIds,
      })),
    [bubbles],
  );

  /*
   * Fase 3: distingue un movimento "automatico" (anti-sovrapposizione,
   * ingresso/uscita da una bubble, cambio di orientamento/resize) da
   * un movimento originato dal drag diretto dell'utente — solo il
   * primo va animato con transizione, il secondo resta sempre 1:1 col
   * puntatore. Un nodo è "user-driven" se è quello sotto il cursore
   * (drag.id) o se sta seguendo rigidamente il gruppo del nodo sotto
   * il cursore (membro del gruppo, non ancora staccato). Usata sia per
   * le card sia — indirettamente, tramite i membri della bubble — per
   * il contenitore della bubble e per le frecce agganciate a un suo
   * perimetro (vedi il loop degli edge più sopra).
   */
  const isNodeUserDriven = (
    nodeId: string,
  ): boolean => {
    const drag = dragRef.current;

    if (!drag) {
      return false;
    }

    if (drag.id === nodeId) {
      return true;
    }

    return (
      !drag.detached &&
      drag.memberOffsets.has(
        nodeId,
      )
    );
  };

  /**
   * Il contenitore di una bubble non deve avere lag quando uno dei
   * suoi membri è sotto il drag diretto dell'utente (altrimenti il box
   * "insegue" con ritardo la card che lo sta facendo cambiare forma in
   * tempo reale) — ma deve animarsi morbidamente per qualunque altro
   * cambiamento (resize, cambio di orientamento, ingresso/uscita di un
   * membro).
   */
  const isBubbleUserDriven = (
    memberIds: readonly string[],
  ): boolean =>
    memberIds.some(
      isNodeUserDriven,
    );

  /**
   * Anteprima del box combinato mostrata durante il drag: racchiude le
   * card in movimento e quella candidata, finché restano abbastanza
   * vicine da poter essere unite al rilascio.
   */
  const combinePreviewBox =
    useMemo(() => {
      if (!combinePreview) {
        return null;
      }

      const target = nodeById(
        combinePreview.targetId,
      );

      const movers =
        combinePreview.movingIds
          .map(nodeById)
          .filter(
            (
              node,
            ): node is NodeGeometry =>
              !!node,
          );

      if (!target || movers.length === 0) {
        return null;
      }

      return unionRect([
        target,
        ...movers,
      ]);
    }, [
      combinePreview,
      nodeById,
    ]);

  /*
   * Statico e indipendente dal singolo nodo: calcolato una volta per
   * render invece che dentro il map delle card. Vedi
   * lib/etl-display.ts per le assunzioni implementative.
   */
  const transformCardLayout =
    getTransformCardLayout();

  /* ---------------------------------------------------------------------- */
  /*                               RENDER                                    */
  /* ---------------------------------------------------------------------- */

  return (
    <section
      ref={boxRef}
      data-palette-workspace
      className="glass-soft relative min-h-0 min-w-0 flex-1 overflow-hidden rounded-3xl select-none"
      style={{
        userSelect: "none",
        WebkitUserSelect: "none",
      }}
      onContextMenu={
        openContextMenu
      }
      onDragEnter={(event) => {
        event.preventDefault();
        event.stopPropagation();
      }}
      onDragOver={(event) => {
        event.preventDefault();
        event.stopPropagation();
        event.dataTransfer.dropEffect =
          "copy";
      }}
      onDrop={(event) => {
        event.preventDefault();
        event.stopPropagation();

        const type =
          event.dataTransfer.getData(
            "application/isa-node",
          );

        if (!type) {
          return;
        }

        const point =
          toLocal(
            event.clientX,
            event.clientY,
          );

        const dropped = placeNode(
          point.x - NODE_W / 2,
          point.y - NODE_H / 2,
          NODE_W,
          NODE_H,
        );

        onAddAt(
          type,
          dropped.x,
          dropped.y,
        );
      }}
    >
      <CanvasContextMenu
        state={contextMenu}
        boundaryRef={boxRef}
        grid={grid}
        onClose={
          closeContextMenu
        }
        onAddDataset={
          onAddDataset
        }
        onFitView={
          fitView
        }
        onToggleGrid={() =>
          setGrid(
            (value) =>
              !value,
          )
        }
        onAutoLayout={() =>
          onLayoutChange(
            "auto",
          )
        }
        onDuplicateNode={
          onDuplicateNode
        }
        onRemoveNode={
          onRemoveNode
        }
        onUnlinkNode={
          unlinkNode
        }
      />

      {palette}

      {/* --------------------------------------------------------------- */}
      {/* Canvas controls                                                  */}
      {/* --------------------------------------------------------------- */}

      <div
        className="absolute right-3 top-3 z-30 flex items-center gap-1.5"
        onPointerDown={(event) =>
          event.stopPropagation()
        }
      >
        <span className="glass-chip flex h-8 items-center rounded-full px-2.5 text-[10px] text-muted-foreground">
          {Math.round(
            zoom * 100,
          )}
          %
        </span>

        <button
          type="button"
          onClick={() =>
            setZoom(
              (z) =>
                Math.max(
                  0.4,
                  +(
                    z - 0.1
                  ).toFixed(2),
                ),
            )
          }
          aria-label="Zoom out"
          className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <ZoomOut className="size-4" />
        </button>

        <button
          type="button"
          onClick={() =>
            setZoom(
              (z) =>
                Math.min(
                  1.8,
                  +(
                    z + 0.1
                  ).toFixed(2),
                ),
            )
          }
          aria-label="Zoom in"
          className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <ZoomIn className="size-4" />
        </button>

        <button
          type="button"
          onClick={fitView}
          aria-label="Fit view"
          className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <Maximize2 className="size-4" />
        </button>

        <button
          type="button"
          onClick={() =>
            setGrid(
              (value) =>
                !value,
            )
          }
          aria-pressed={grid}
          aria-label={
            grid
              ? "Nascondi griglia"
              : "Mostra griglia"
          }
          title={
            grid
              ? "Nascondi griglia"
              : "Mostra griglia"
          }
          className={`glass-chip flex size-8 items-center justify-center rounded-full transition ${
            grid
              ? "text-foreground"
              : "text-muted-foreground/50 hover:text-foreground"
          }`}
        >
          <Grid3x3 className="size-4" />
        </button>

        <IsaMenu
          label="Disposizione delle card"
          Icon={
            workflow.layout ===
            "auto"
              ? LayoutGrid
              : Move
          }
        >
          {(close) => (
            <>
              <span className="block px-3 py-1.5 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                Disposizione
              </span>

              <IsaMenuCheckItem
                label="Automatica"
                checked={
                  workflow.layout ===
                  "auto"
                }
                onToggle={() => {
                  onLayoutChange(
                    "auto",
                  );
                  close();
                }}
              />

              <IsaMenuCheckItem
                label="Trascinamento libero"
                checked={
                  workflow.layout !==
                  "auto"
                }
                onToggle={() => {
                  onLayoutChange(
                    "manual",
                  );
                  close();
                }}
              />
            </>
          )}
        </IsaMenu>

        <IsaMenu
          label="Visualizza sulle card"
          Icon={Eye}
        >
          {() => (
            <>
              <span className="block px-3 py-1.5 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                Visualizza
              </span>

              {DISPLAY_OPTIONS.map(
                (option) => (
                  <IsaMenuCheckItem
                    key={
                      option.key
                    }
                    label={
                      option.label
                    }
                    checked={
                      display[
                        option.key
                      ]
                    }
                    onToggle={() =>
                      setDisplay(
                        (
                          current,
                        ) => ({
                          ...current,
                          [option.key]:
                            !current[
                              option.key
                            ],
                        }),
                      )
                    }
                  />
                ),
              )}
            </>
          )}
        </IsaMenu>
      </div>

      {/* --------------------------------------------------------------- */}
      {/* Canvas surface                                                  */}
      {/* --------------------------------------------------------------- */}

      <div
        className="relative h-full w-full overflow-hidden"
        style={{
          backgroundImage: grid
            ? "radial-gradient(circle, color-mix(in oklab, var(--foreground) 22%, transparent) 1px, transparent 1px)"
            : undefined,
          backgroundSize: grid
            ? `${Math.round(
                24 * zoom,
              )}px ${Math.round(
                24 * zoom,
              )}px`
            : undefined,
          userSelect: "none",
          WebkitUserSelect:
            "none",
        }}
        onPointerDown={() => {
          closeContextMenu();
          onSelect(null);
        }}
      >
        <div
          ref={surfaceRef}
          className="relative origin-top-left"
          style={{
            width: surfaceW,
            height: surfaceH,
            transform: `scale(${zoom})`,
            userSelect: "none",
            WebkitUserSelect:
              "none",
          }}
        >
          {/* ---------------------------------------------------------- */}
          {/* Edges                                                      */}
          {/* ---------------------------------------------------------- */}

          <svg className="pointer-events-none absolute inset-0 size-full overflow-visible">
            {workflow.edges.map(
              (edge, edgeIndex) => {
                const from =
                  nodeById(
                    edge.fromNode,
                  );

                const to =
                  nodeById(
                    edge.toNode,
                  );

                if (
                  !from ||
                  !to
                ) {
                  return null;
                }

                const fromBubble =
                  from.groupId
                    ? bubbles.get(
                        from.groupId,
                      )
                    : undefined;

                const toBubble =
                  to.groupId
                    ? bubbles.get(
                        to.groupId,
                      )
                    : undefined;

                /*
                 * Fase 2, punto 2: nessuna freccia interna — entrambi
                 * gli endpoint nella STESSA bubble non vengono
                 * disegnati (il collegamento resta nel modello dati,
                 * solo non è renderizzato).
                 */
                if (
                  fromBubble &&
                  toBubble &&
                  fromBubble.groupId ===
                    toBubble.groupId
                ) {
                  return null;
                }

                /*
                 * Fase 2, punto 3: un endpoint che appartiene a una
                 * bubble si aggancia sul perimetro del suo bounding
                 * box (stesso identico algoritmo di routing di un
                 * nodo singolo, solo con un rettangolo diverso), non
                 * sulla card interna specifica. I membri della bubble
                 * vanno esclusi dagli ostacoli, altrimenti il routing
                 * proverebbe a evitare le proprie card interne.
                 */
                const fromGeometry =
                  fromBubble
                    ? bubbleNodeGeometry(
                        from,
                        fromBubble,
                      )
                    : from;

                const toGeometry =
                  toBubble
                    ? bubbleNodeGeometry(
                        to,
                        toBubble,
                      )
                    : to;

                const excludeIds =
                  fromBubble ||
                  toBubble
                    ? new Set<string>([
                        ...(fromBubble?.memberIds ??
                          []),
                        ...(toBubble?.memberIds ??
                          []),
                      ])
                    : undefined;

                const route =
                  getBestRoute(
                    fromGeometry,
                    toGeometry,
                    visibleNodes,
                    excludeIds,
                  );

                if (!route) {
                  return null;
                }

                /*
                 * `route.points` va sempre da from (edge.fromNode) a to
                 * (edge.toNode), perché `getBestRoute(from, to, ...)` è
                 * chiamata così qui sopra: la direzione della luce di
                 * flusso, animata lungo questo stesso path da offset
                 * 0% a 100%, segue quindi il verso reale del
                 * collegamento nel modello dati — non un'euristica sul
                 * tipo di nodo (es. "i Dataset sono sempre origine").
                 * Una card transform con più archi in ingresso e in
                 * uscita mostra quindi ogni luce nel verso corretto
                 * per il proprio arco, indipendentemente dagli altri.
                 */
                const path =
                  smoothPathFromPoints(
                    route.points,
                  );

                const active =
                  edge.fromNode ===
                    selectedId ||
                  edge.toNode ===
                    selectedId ||
                  hoverEdge ===
                    edge.id;

                /*
                 * Fase 3, punto 3: la freccia segue la STESSA regola
                 * "user-driven" della card/bubble a cui è agganciata —
                 * se l'endpoint è dentro una bubble, basta che UNO dei
                 * suoi membri sia sotto drag diretto (la bubble intera
                 * si sta muovendo dal vivo), non necessariamente lo
                 * specifico nodo `edge.fromNode`/`edge.toNode`.
                 */
                const edgeIsUserDriven =
                  (fromBubble
                    ? fromBubble.memberIds.some(
                        isNodeUserDriven,
                      )
                    : isNodeUserDriven(
                        edge.fromNode,
                      )) ||
                  (toBubble
                    ? toBubble.memberIds.some(
                        isNodeUserDriven,
                      )
                    : isNodeUserDriven(
                        edge.toNode,
                      ));

                const edgeTransition =
                  edgeIsUserDriven
                    ? "none"
                    : EDGE_AUTO_MOVE_TRANSITION;

                const rows =
                  formatRows(
                    analyzeNode(
                      workflow,
                      from,
                    ).rows,
                  );

                const middlePoint =
                  route.points[
                    Math.floor(
                      route.points
                        .length /
                        2,
                    )
                  ];

                return (
                  <g
                    key={
                      edge.id
                    }
                  >
                    {/* Hit area */}
                    <path
                      d={path}
                      fill="none"
                      stroke="transparent"
                      strokeWidth={14}
                      style={{
                        transition:
                          edgeTransition,
                      }}
                      className="pointer-events-auto cursor-pointer"
                      onPointerEnter={() =>
                        setHoverEdge(
                          edge.id,
                        )
                      }
                      onPointerLeave={() =>
                        setHoverEdge(
                          (current) =>
                            current ===
                            edge.id
                              ? null
                              : current,
                        )
                      }
                      onDoubleClick={() =>
                        onRemoveEdge(
                          edge.id,
                        )
                      }
                    />

                    {/* Solo linea, nessuna freccia */}
                    <path
                      d={path}
                      fill="none"
                      stroke="var(--brand)"
                      strokeWidth={
                        active
                          ? 2.8
                          : 2
                      }
                      strokeOpacity={
                        active
                          ? 0.95
                          : 0.42
                      }
                      strokeLinecap="round"
                      strokeLinejoin="round"
                      style={{
                        transition:
                          edgeTransition,
                      }}
                      className={`pointer-events-none ${
                        active
                          ? "edge-flow"
                          : ""
                      }`}
                    />

                    {/*
                     * Luce di flusso direzionale: piccolo punto che
                     * percorre lo stesso path (da from a to, vedi
                     * commento sopra) via CSS offset-path, in loop.
                     * Nessun ricalcolo per frame: il browser interpola
                     * offset-distance, niente rAF.
                     */}
                    <circle
                      r={
                        active
                          ? 2.6
                          : 2.1
                      }
                      fill="var(--brand-glow)"
                      className="edge-flow-dot pointer-events-none"
                      style={
                        {
                          offsetPath: `path("${path}")`,
                          animationDelay: `${-(
                            (edgeIndex %
                              6) *
                            0.45
                          )}s`,
                          /*
                           * Letto dalla keyframe come picco di
                           * opacità: un valore statico su `opacity`
                           * qui verrebbe ignorato, perché
                           * un'animazione CSS sovrascrive sempre lo
                           * stile inline della stessa proprietà.
                           */
                          "--edge-flow-peak":
                            active
                              ? 0.9
                              : 0.55,
                        } as React.CSSProperties
                      }
                    />

                    {hoverEdge ===
                      edge.id &&
                      middlePoint && (
                        <g className="pointer-events-none">
                          <rect
                            x={
                              middlePoint.x -
                              34
                            }
                            y={
                              middlePoint.y -
                              20
                            }
                            width={68}
                            height={18}
                            rx={4}
                            fill="var(--background)"
                            stroke="var(--glass-border)"
                          />

                          <text
                            x={
                              middlePoint.x
                            }
                            y={
                              middlePoint.y -
                              7
                            }
                            textAnchor="middle"
                            fontSize={10}
                            fill="var(--muted-foreground)"
                          >
                            {rows}{" "}
                            rows
                          </text>
                        </g>
                      )}
                  </g>
                );
              },
            )}

            {/* -------------------------------------------------------- */}
            {/* Linking preview                                          */}
            {/* -------------------------------------------------------- */}

            {pendingRoute && (
              <g>
                <path
                  d={
                    pendingRoute.path
                  }
                  fill="none"
                  stroke="var(--brand)"
                  strokeWidth={2}
                  strokeDasharray="5 5"
                  strokeLinecap="round"
                  strokeLinejoin="round"
                />

                {pendingRoute.indicator && (
                  <circle
                    cx={
                      pendingRoute
                        .indicator
                        .x
                    }
                    cy={
                      pendingRoute
                        .indicator
                        .y
                    }
                    r={7}
                    fill="var(--background)"
                    stroke="var(--brand)"
                    strokeWidth={2}
                  />
                )}
              </g>
            )}
          </svg>

          {/* ---------------------------------------------------------- */}
          {/* Group containers                                           */}
          {/* ---------------------------------------------------------- */}

          {groupBoxes.map(
            ({
              groupId,
              rect,
              orientation,
              memberIds,
            }) => (
              <div
                key={groupId}
                data-bubble-orientation={
                  orientation
                }
                /*
                 * L'orientamento è esposto qui (nessun effetto visivo
                 * oltre alla forma naturale del bounding box in questa
                 * fase): una futura fase lo userà per riallineare i
                 * membri lungo l'asse della bubble con un'animazione.
                 */
                className="glass-soft pointer-events-none absolute rounded-3xl"
                style={{
                  left:
                    rect.x -
                    GROUP_PADDING,
                  top:
                    rect.y -
                    GROUP_PADDING,
                  width:
                    rect.width +
                    GROUP_PADDING * 2,
                  height:
                    rect.height +
                    GROUP_PADDING * 2,
                  transition:
                    isBubbleUserDriven(
                      memberIds,
                    )
                      ? "none"
                      : BUBBLE_AUTO_MOVE_TRANSITION,
                  opacity: 0.55,
                }}
              />
            ),
          )}

          {combinePreviewBox && (
            <div
              className="gradient-brand pointer-events-none absolute rounded-3xl border-2 border-dashed border-brand"
              style={{
                left:
                  combinePreviewBox.x -
                  GROUP_PADDING,
                top:
                  combinePreviewBox.y -
                  GROUP_PADDING,
                width:
                  combinePreviewBox.width +
                  GROUP_PADDING * 2,
                height:
                  combinePreviewBox.height +
                  GROUP_PADDING * 2,
                opacity: 0.22,
              }}
            />
          )}

          {/* ---------------------------------------------------------- */}
          {/* Cards                                                      */}
          {/* ---------------------------------------------------------- */}

          {visibleNodes.map(
            (node) => {
              const def =
                nodeDef(
                  node.type,
                );

              if (!def) {
                return null;
              }

              const analysis =
                analyzeNode(
                  workflow,
                  node,
                );

              const incoming =
                workflow.edges.filter(
                  (edge) =>
                    edge.toNode ===
                    node.id,
                );

              const outgoing =
                workflow.edges.filter(
                  (edge) =>
                    edge.fromNode ===
                    node.id,
                );

              const selected =
                node.id ===
                selectedId;

              const hasIncoming =
                incoming.length >
                0;

              const hasOutgoing =
                outgoing.length >
                0;

              const status =
                STATUS[
                  analysis.errors
                    .length >
                    0 &&
                  node.status !==
                    "running"
                    ? "error"
                    : node.status
                ];

              const detail =
                nodeSummary(
                  node.type,
                  node.config,
                );

              const isSource =
                def.category ===
                "sources";

              /*
               * Fase 4: quale pannello impostazioni dedicato mostrare
               * dal trigger tre puntini (null = tipo non ancora
               * coperto, resta il menu azioni generico esistente). Se
               * il nodo appartiene a una bubble (fase 2), gli id dei
               * membri servono al CombinePanel per raccogliere gli
               * input esterni collegati all'intera bubble.
               */
              const settingsPanelKind =
                getSettingsPanelKind(
                  node.type,
                );

              const bubbleMemberIds =
                node.groupId
                  ? bubbles.get(
                      node.groupId,
                    )?.memberIds
                  : undefined;

              const showSource =
                isSource &&
                display.source;

              const metricsTooltip =
                display.metrics
                  ? `${formatRows(
                      analysis.rows,
                    )} rows · ${
                      analysis
                        .columns
                        .length
                    } cols`
                  : undefined;
              /*
               * Scegliamo il lato visuale dei port
               * in funzione dei collegamenti esistenti.
               *
               * Default:
               * input a sinistra
               * output a destra
               */
              const firstIncoming =
                incoming[0];

              const firstOutgoing =
                outgoing[0];

              const incomingNode =
                firstIncoming
                  ? nodeById(
                      firstIncoming.fromNode,
                    )
                  : undefined;

              const outgoingNode =
                firstOutgoing
                  ? nodeById(
                      firstOutgoing.toNode,
                    )
                  : undefined;

              const inputSide: Side =
                incomingNode
                  ? getBestRoute(
                      incomingNode,
                      node,
                      visibleNodes,
                    )?.toSide ??
                    "left"
                  : "left";

              const outputSide: Side =
                outgoingNode
                  ? getBestRoute(
                      node,
                      outgoingNode,
                      visibleNodes,
                    )?.fromSide ??
                    "right"
                  : "right";

              const inputCount =
                def.inputs.length;

              const outputCount =
                def.outputs.length;

              const isDragging =
                dragRef.current?.id ===
                node.id;

              /*
               * Fase 3, punto 2: il drag diretto dell'utente resta
               * SEMPRE 1:1 col puntatore, senza transizione — sia per
               * la card sotto il cursore, sia per gli altri membri
               * dello stesso gruppo che la seguono rigidamente finché
               * non si staccano (isNodeUserDriven, sopra). Qualsiasi
               * ALTRA card che si sposta (spinta dall'anti-
               * sovrapposizione, o per un cambio non originato da
               * questo drag) è invece un movimento "automatico" e usa
               * la transizione di lib/etl-motion.ts.
               */
              const isUserDriven =
                isNodeUserDriven(
                  node.id,
                );

              return (
                <div
                  key={node.id}
                  data-node-card
                  data-node-id={
                    node.id
                  }
                  className={`glass-panel absolute flex flex-col overflow-visible rounded-2xl p-3 transition-shadow select-none ${
                    selected
                      ? "node-selected"
                      : ""
                  } ${
                    pending?.targetNode ===
                    node.id
                      ? "node-link-target"
                      : ""
                  }`}
                  style={{
                    left: node.x,
                    top: node.y,
                    width: node.width,
                    height: node.height,
                    transition:
                      isUserDriven
                        ? "none"
                        : CARD_AUTO_MOVE_TRANSITION,
                    cursor: isDragging
                      ? "grabbing"
                      : "grab",
                    touchAction:
                      "none",
                    userSelect:
                      "none",
                    WebkitUserSelect:
                      "none",
                  }}
                  onPointerDown={(
                    event,
                  ) =>
                    startDragNode(
                      event,
                      node.id,
                    )
                  }
                  onPointerMove={
                    moveDragNode
                  }
                  onPointerUp={
                    endDragNode
                  }
                  onPointerCancel={
                    endDragNode
                  }
                >
                  {isSource ? (
                    <>
                      {/* ------------------------------------------------ */}
                      {/* Header (Dataset)                                 */}
                      {/* ------------------------------------------------ */}

                      <div className="flex min-w-0 items-start gap-2 whitespace-nowrap">
                        <span
                          className={`${categoryAccent(
                            def.category,
                          )} flex size-9 shrink-0 items-center justify-center rounded-lg`}
                        >
                          <def.Icon className="size-5" />
                        </span>

                        <span className="min-w-0 flex-1 pt-0.5">
                          <span className="block whitespace-normal break-normal text-[12px] font-semibold leading-tight">
                            {
                              node.title
                            }
                          </span>

                          <span className="mt-0.5 block whitespace-normal break-normal text-[9px] leading-tight text-muted-foreground">
                            {
                              def.label
                            }
                          </span>
                        </span>

                        <span
                          data-node-control
                          onPointerDown={(
                            event,
                          ) =>
                            event.stopPropagation()
                          }
                        >
                          <IsaMenu
                            label={`Azioni per ${node.title}`}
                            triggerClassName="size-7 rounded-lg border border-border/60 text-foreground"
                          >
                            {(close) => (
                              <>
                                <IsaMenuItem
                                  Icon={
                                    Copy
                                  }
                                  label="Duplica nodo"
                                  onClick={() => {
                                    onDuplicateNode(
                                      node.id,
                                    );
                                    close();
                                  }}
                                />

                                {hasIncoming && (
                                  <IsaMenuItem
                                    Icon={
                                      Unlink
                                    }
                                    label="Scollega input"
                                    onClick={() => {
                                      unlinkNode(
                                        node.id,
                                      );
                                      close();
                                    }}
                                  />
                                )}

                                {node.groupId && (
                                  <IsaMenuItem
                                    Icon={
                                      Ungroup
                                    }
                                    label="Rimuovi dal gruppo"
                                    onClick={() => {
                                      onUngroupNode(
                                        node.id,
                                      );
                                      close();
                                    }}
                                  />
                                )}

                                <IsaMenuItem
                                  Icon={
                                    Trash2
                                  }
                                  label="Elimina nodo"
                                  danger
                                  onClick={() => {
                                    onRemoveNode(
                                      node.id,
                                    );
                                    close();
                                  }}
                                />

                                <span className="my-1 block h-px bg-border" />

                                <span className="block px-3 py-1 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                                  Visualizza
                                </span>

                                {DISPLAY_OPTIONS.map(
                                  (
                                    option,
                                  ) => (
                                    <IsaMenuCheckItem
                                      key={
                                        option.key
                                      }
                                      label={
                                        option.label
                                      }
                                      checked={
                                        display[
                                          option.key
                                        ]
                                      }
                                      onToggle={() =>
                                        setDisplay(
                                          (
                                            current,
                                          ) => ({
                                            ...current,
                                            [option.key]:
                                              !current[
                                                option.key
                                              ],
                                          }),
                                        )
                                      }
                                    />
                                  ),
                                )}
                              </>
                            )}
                          </IsaMenu>
                        </span>
                      </div>

                      {/* ------------------------------------------------ */}
                      {/* Body (Dataset)                                   */}
                      {/* ------------------------------------------------ */}

                      <div className="mt-3 min-h-0 flex-1 overflow-hidden">
                        {showSource ? (
                          <div className="rounded-xl border border-border/50 bg-background/20 px-2.5 py-2">
                            <span className="block wrap-break-word text-[10px] leading-relaxed text-foreground">
                              {
                                detail
                              }
                            </span>
                          </div>
                        ) : null}

                        {display.metrics && (
                          <div className="mt-2 flex flex-wrap gap-1.5">
                            <span className="glass-chip rounded-full px-2 py-1 text-[9px] text-muted-foreground">
                              {formatRows(
                                analysis.rows,
                              )}{" "}
                              rows
                            </span>

                            <span className="glass-chip rounded-full px-2 py-1 text-[9px] text-muted-foreground">
                              {
                                analysis
                                  .columns
                                  .length
                              }{" "}
                              cols
                            </span>
                          </div>
                        )}

                        {!showSource &&
                          !display.metrics && (
                            <span className="block wrap-break-word text-[10px] leading-relaxed text-muted-foreground">
                              {
                                def.label
                              }
                            </span>
                          )}

                        {display.status && (
                          <span
                            title={
                              status.label
                            }
                            className="mt-2 inline-flex items-center gap-1 rounded-full border border-border/50 bg-background/20 px-2 py-0.5 text-[9px] font-medium text-muted-foreground"
                          >
                            <span
                              className="size-1.5 shrink-0 rounded-full"
                              style={{
                                background:
                                  status.color,
                              }}
                            />
                            {
                              status.label
                            }
                          </span>
                        )}
                      </div>
                    </>
                  ) : (
                    <>
                      {/* ------------------------------------------------ */}
                      {/* Transform card — icona centrale, titolo sopra,   */}
                      {/* menu impostazioni in basso al centro.            */}
                      {/* ------------------------------------------------ */}

                      <span
                        className="block truncate px-1 text-center text-[11px] font-semibold leading-tight"
                        title={
                          node.title
                        }
                      >
                        {
                          node.title
                        }
                      </span>

                      {/*
                       * Body permanente (dettagli/metriche testuali)
                       * rimosso dal nuovo layout: il riepilogo resta
                       * disponibile come tooltip nativo on-hover sul
                       * riquadro icona, così non serve un pannello
                       * dedicato solo per questo in questa fase.
                       */}
                      <div
                        className="flex min-h-0 flex-1 items-center justify-center"
                        title={
                          metricsTooltip ??
                          detail
                        }
                      >
                        <span
                          className={`${categoryAccent(
                            def.category,
                          )} flex shrink-0 items-center justify-center rounded-2xl`}
                          style={{
                            width:
                              transformCardLayout.iconBoxSize,
                            height:
                              transformCardLayout.iconBoxSize,
                          }}
                        >
                          <def.Icon
                            style={{
                              width:
                                transformCardLayout.iconGlyphSize,
                              height:
                                transformCardLayout.iconGlyphSize,
                            }}
                          />
                        </span>
                      </div>

                      <div className="flex justify-center">
                        <span
                          data-node-control
                          onPointerDown={(
                            event,
                          ) =>
                            event.stopPropagation()
                          }
                        >
                          {/*
                           * Fase 4: per i tipi coperti da un pannello
                           * dedicato (Filter / famiglia Combine /
                           * famiglia Aggregate) il trigger apre quel
                           * pannello, guidato dallo schema effettivo
                           * (analyzeNode), al posto del menu azioni
                           * generico. Duplica/Elimina/Scollega restano
                           * comunque raggiungibili dal menu contestuale
                           * del canvas (click destro sul nodo), quindi
                           * non è una perdita di funzionalità. I tipi
                           * transform non ancora coperti (Select,
                           * Rename, Sort, Dedupe, Fill Missing Values,
                           * Formula) mantengono il menu generico
                           * com'era in fase 1.
                           */}
                          <IsaMenu
                            label={`Impostazioni per ${node.title}`}
                            variant="bare"
                            Icon={
                              MoreHorizontal
                            }
                            width={
                              settingsPanelKind
                                ? 320
                                : undefined
                            }
                            triggerClassName="size-7 text-muted-foreground hover:text-foreground"
                          >
                            {(close) =>
                              settingsPanelKind ? (
                                <div className="max-h-[24rem] overflow-y-auto p-2">
                                  <div className="mb-2 flex items-center gap-2 px-1">
                                    <span
                                      className={`${categoryAccent(
                                        def.category,
                                      )} flex size-7 shrink-0 items-center justify-center rounded-lg`}
                                    >
                                      <def.Icon className="size-3.5" />
                                    </span>

                                    <span className="min-w-0 flex-1">
                                      <span className="block truncate text-xs font-semibold">
                                        {
                                          node.title
                                        }
                                      </span>
                                      <span className="block truncate text-[10px] text-muted-foreground">
                                        {
                                          def.label
                                        }
                                      </span>
                                    </span>
                                  </div>

                                  {settingsPanelKind ===
                                    "filter" && (
                                    <FilterPanel
                                      workflow={
                                        workflow
                                      }
                                      node={
                                        node
                                      }
                                      onConfigChange={(
                                        patch,
                                      ) =>
                                        onUpdateNodeConfig(
                                          node.id,
                                          patch,
                                        )
                                      }
                                    />
                                  )}

                                  {settingsPanelKind ===
                                    "combine" && (
                                    <CombinePanel
                                      workflow={
                                        workflow
                                      }
                                      node={
                                        node
                                      }
                                      memberIds={
                                        bubbleMemberIds
                                      }
                                      onConfigChange={(
                                        patch,
                                      ) =>
                                        onUpdateNodeConfig(
                                          node.id,
                                          patch,
                                        )
                                      }
                                    />
                                  )}

                                  {settingsPanelKind ===
                                    "aggregate" && (
                                    <AggregatePanel
                                      workflow={
                                        workflow
                                      }
                                      node={
                                        node
                                      }
                                      onConfigChange={(
                                        patch,
                                      ) =>
                                        onUpdateNodeConfig(
                                          node.id,
                                          patch,
                                        )
                                      }
                                    />
                                  )}
                                </div>
                              ) : (
                                <>
                                  <IsaMenuItem
                                    Icon={
                                      Copy
                                    }
                                    label="Duplica nodo"
                                    onClick={() => {
                                      onDuplicateNode(
                                        node.id,
                                      );
                                      close();
                                    }}
                                  />

                                  {hasIncoming && (
                                    <IsaMenuItem
                                      Icon={
                                        Unlink
                                      }
                                      label="Scollega input"
                                      onClick={() => {
                                        unlinkNode(
                                          node.id,
                                        );
                                        close();
                                      }}
                                    />
                                  )}

                                  {node.groupId && (
                                    <IsaMenuItem
                                      Icon={
                                        Ungroup
                                      }
                                      label="Rimuovi dal gruppo"
                                      onClick={() => {
                                        onUngroupNode(
                                          node.id,
                                        );
                                        close();
                                      }}
                                    />
                                  )}

                                  <IsaMenuItem
                                    Icon={
                                      Trash2
                                    }
                                    label="Elimina nodo"
                                    danger
                                    onClick={() => {
                                      onRemoveNode(
                                        node.id,
                                      );
                                      close();
                                    }}
                                  />

                                  <span className="my-1 block h-px bg-border" />

                                  <span className="block px-3 py-1 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                                    Visualizza
                                  </span>

                                  {DISPLAY_OPTIONS.map(
                                    (
                                      option,
                                    ) => (
                                      <IsaMenuCheckItem
                                        key={
                                          option.key
                                        }
                                        label={
                                          option.label
                                        }
                                        checked={
                                          display[
                                            option.key
                                          ]
                                        }
                                        onToggle={() =>
                                          setDisplay(
                                            (
                                              current,
                                            ) => ({
                                              ...current,
                                              [option.key]:
                                                !current[
                                                  option.key
                                                ],
                                            }),
                                          )
                                        }
                                      />
                                    ),
                                  )}
                                </>
                              )
                            }
                          </IsaMenu>
                        </span>
                      </div>
                    </>
                  )}

                  {/* ------------------------------------------------ */}
                  {/* Input ports                                       */}
                  {/* ------------------------------------------------ */}

                  {def.inputs.map(
                    (
                      port,
                      index,
                    ) => {
                      const offset =
                        portOffset(
                          index,
                          inputCount,
                          inputSide ===
                            "left" ||
                            inputSide ===
                              "right"
                            ? node.height
                            : node.width,
                        );

                      const style =
                        getPortStyle(
                          inputSide,
                          offset,
                        );

                      return (
                        <span
                          key={`in-${port}`}
                          data-node-control
                          data-in-port={
                            port
                          }
                          data-node-id={
                            node.id
                          }
                          title={`Input ${port}`}
                          className="absolute z-20 flex size-5 -translate-x-1/2 -translate-y-1/2 items-center justify-center"
                          style={style}
                          onPointerDown={(
                            event,
                          ) =>
                            event.stopPropagation()
                          }
                        >
                          <span className="size-2.5 rounded-full border border-border bg-background" />
                        </span>
                      );
                    },
                  )}

                  {/* ------------------------------------------------ */}
                  {/* Output ports                                      */}
                  {/* ------------------------------------------------ */}

                  {def.outputs.map(
                    (
                      port,
                      index,
                    ) => {
                      const offset =
                        portOffset(
                          index,
                          outputCount,
                          outputSide ===
                            "left" ||
                            outputSide ===
                              "right"
                            ? node.height
                            : node.width,
                        );

                      const style =
                        getPortStyle(
                          outputSide,
                          offset,
                        );

                      return (
                        <span
                          key={`out-${port}`}
                          data-node-control
                          data-out-port={
                            port
                          }
                          title={`Output ${port} — trascina su un input`}
                          className="absolute z-20 flex size-5 -translate-x-1/2 -translate-y-1/2 cursor-crosshair items-center justify-center"
                          style={style}
                          onPointerDown={(
                            event,
                          ) =>
                            startLink(
                              event,
                              node.id,
                              port,
                            )
                          }
                        >
                          <span className="gradient-brand size-2.5 rounded-full" />
                        </span>
                      );
                    },
                  )}
                </div>
              );
            },
          )}
        </div>

        {/* -------------------------------------------------------------- */}
        {/* Empty state                                                    */}
        {/* -------------------------------------------------------------- */}

        {workflow.nodes.length ===
          0 && (
          <div className="pointer-events-none absolute inset-0 flex flex-col items-center justify-center gap-3 px-6 text-center select-none">
            <span className="gradient-brand flex size-12 items-center justify-center rounded-2xl text-brand-foreground">
              <Database className="size-6" />
            </span>

            <p className="text-base font-semibold">
              Build your data workflow
            </p>

            <p className="max-w-sm text-sm text-muted-foreground">
              Inizia con un dataset e
              collega trasformazioni per
              creare la tua pipeline.
            </p>

            <button
              type="button"
              onClick={
                onAddDataset
              }
              className="gradient-brand pointer-events-auto flex h-10 items-center gap-2 rounded-full px-4 text-sm font-semibold text-brand-foreground transition hover:brightness-110"
            >
              <Database className="size-4" />
              Add dataset
            </button>
          </div>
        )}
      </div>
    </section>
  );
}

/* -------------------------------------------------------------------------- */
/*                              PORT POSITION                                 */
/* -------------------------------------------------------------------------- */

function getPortStyle(
  side: Side,
  offset: number,
): React.CSSProperties {
  switch (side) {
    case "top":
      return {
        left: offset,
        top: 0,
      };

    case "right":
      return {
        left: "100%",
        top: offset,
      };

    case "bottom":
      return {
        left: offset,
        top: "100%",
      };

    case "left":
      return {
        left: 0,
        top: offset,
      };
  }
}
