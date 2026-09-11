# ISA ETL Snapshot

Generated: 2026-09-11T21:53:31Z

## Index
- src/components/isa/etl/data-preview.tsx
- src/components/isa/etl/inspector.tsx
- src/components/isa/etl/isa-context-menu.tsx
- src/components/isa/etl/settings-panels/aggregate-panel.tsx
- src/components/isa/etl/settings-panels/combine-panel.tsx
- src/components/isa/etl/settings-panels/filter-panel.tsx
- src/components/isa/etl/settings-panels/panel-controls.tsx
- src/components/isa/etl/tool-palette.tsx
- src/components/isa/ui/isa-menu.tsx
- src/lib/etl-bubble.ts
- src/lib/etl-catalog.ts
- src/lib/etl-display.ts
- src/lib/etl-motion.ts
- src/lib/etl-node-config.ts
- src/lib/etl-schema.ts
- src/lib/etl-workflow.tsx


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


=== FILE: src/components/isa/ui/isa-menu.tsx ===
import type { Pencil } from "lucide-react";
import {
  Check,
  MoreHorizontal,
} from "lucide-react";
import { createPortal } from "react-dom";
import {
  useCallback,
  useEffect,
  useLayoutEffect,
  useRef,
  useState,
} from "react";

type MenuSide =
  | "top"
  | "bottom"
  | "left"
  | "right";

type MenuPlacement =
  | "auto"
  | MenuSide;

type MenuPosition = {
  top: number;
  left: number;
};

const MENU_WIDTH = 208;
const MENU_GAP = 8;
const MENU_MARGIN = 8;

export function IsaMenu({
  label,
  children,
  align = "right",
  Icon = MoreHorizontal,
  triggerClassName = "",
  boundaryRef,
  placement = "bottom",
  variant = "chip",
  width = MENU_WIDTH,
}: {
  label: string;
  children: (
    close: () => void,
  ) => React.ReactNode;
  align?: "left" | "right";
  Icon?: typeof Pencil;
  triggerClassName?: string;
  boundaryRef?: React.RefObject<
    HTMLElement | null
  >;
  placement?: MenuPlacement;
  variant?: "chip" | "bare";
  /** Larghezza del popover in px (default MENU_WIDTH) — es. per i pannelli impostazioni della fase 4, più larghi di un menu azioni. */
  width?: number | undefined;
}) {
  const [open, setOpen] =
    useState(false);

  const [position, setPosition] =
    useState<MenuPosition | null>(
      null,
    );

  const rootRef =
    useRef<HTMLDivElement>(null);

  const triggerRef =
    useRef<HTMLButtonElement>(null);

  const menuRef =
    useRef<HTMLDivElement>(null);

  const constrained =
    Boolean(
      boundaryRef &&
        placement === "auto",
    );

  const close = useCallback(
    () => {
      setOpen(false);
      setPosition(null);
    },
    [],
  );

  const calculatePosition =
    useCallback(() => {
      if (!constrained) {
        return;
      }

      const trigger =
        triggerRef.current;

      const menu =
        menuRef.current;

      const boundary =
        boundaryRef?.current;

      if (
        !trigger ||
        !menu ||
        !boundary
      ) {
        return;
      }

      const triggerRect =
        trigger.getBoundingClientRect();

      const menuRect =
        menu.getBoundingClientRect();

      const boundaryRect =
        boundary.getBoundingClientRect();

      const menuWidth =
        menuRect.width ||
        width;

      const menuHeight =
        menuRect.height;

      const available: Record<
        MenuSide,
        number
      > = {
        top:
          triggerRect.top -
          boundaryRect.top -
          MENU_GAP -
          MENU_MARGIN,

        bottom:
          boundaryRect.bottom -
          triggerRect.bottom -
          MENU_GAP -
          MENU_MARGIN,

        left:
          triggerRect.left -
          boundaryRect.left -
          MENU_GAP -
          MENU_MARGIN,

        right:
          boundaryRect.right -
          triggerRect.right -
          MENU_GAP -
          MENU_MARGIN,
      };

      const fits: Record<
        MenuSide,
        boolean
      > = {
        top:
          available.top >=
          menuHeight,

        bottom:
          available.bottom >=
          menuHeight,

        left:
          available.left >=
          menuWidth,

        right:
          available.right >=
          menuWidth,
      };

      /*
       * Ordine preferenziale:
       * prima sotto, poi sopra, poi lato destro/sinistro.
       *
       * Se un lato non ha spazio sufficiente,
       * viene provato il successivo.
       */
      const order: MenuSide[] = [
        "bottom",
        "top",
        "right",
        "left",
      ];

      let side:
        | MenuSide
        | null = null;

      for (
        const candidate of order
      ) {
        if (fits[candidate]) {
          side = candidate;
          break;
        }
      }

      /*
       * Se il Canvas è troppo piccolo per contenere
       * il menu interamente su un lato, scegliamo il lato
       * con più spazio e facciamo un clamp finale.
       */
      if (!side) {
        const fallback =
          (
            Object.keys(
              available,
            ) as MenuSide[]
          ).sort(
            (a, b) =>
              available[b] -
              available[a],
          );

        side =
          fallback[0] ??
          "bottom";
      }

      let left =
        triggerRect.right -
        menuWidth;

      let top =
        triggerRect.bottom +
        MENU_GAP;

      if (
        side === "bottom"
      ) {
        top =
          triggerRect.bottom +
          MENU_GAP;

        left =
          align === "left"
            ? triggerRect.left
            : triggerRect.right -
              menuWidth;
      }

      if (
        side === "top"
      ) {
        top =
          triggerRect.top -
          menuHeight -
          MENU_GAP;

        left =
          align === "left"
            ? triggerRect.left
            : triggerRect.right -
              menuWidth;
      }

      if (
        side === "right"
      ) {
        left =
          triggerRect.right +
          MENU_GAP;

        top =
          triggerRect.top;
      }

      if (
        side === "left"
      ) {
        left =
          triggerRect.left -
          menuWidth -
          MENU_GAP;

        top =
          triggerRect.top;
      }

      const minLeft =
        boundaryRect.left +
        MENU_MARGIN;

      const maxLeft =
        Math.max(
          minLeft,
          boundaryRect.right -
            menuWidth -
            MENU_MARGIN,
        );

      const minTop =
        boundaryRect.top +
        MENU_MARGIN;

      const maxTop =
        Math.max(
          minTop,
          boundaryRect.bottom -
            menuHeight -
            MENU_MARGIN,
        );

      left = Math.min(
        Math.max(left, minLeft),
        maxLeft,
      );

      top = Math.min(
        Math.max(top, minTop),
        maxTop,
      );

      setPosition({
        left,
        top,
      });
    }, [
      align,
      boundaryRef,
      constrained,
      width,
    ]);

  useLayoutEffect(() => {
    if (
      !open ||
      !constrained
    ) {
      return;
    }

    calculatePosition();

    const frame =
      requestAnimationFrame(
        calculatePosition,
      );

    return () =>
      cancelAnimationFrame(
        frame,
      );
  }, [
    open,
    constrained,
    calculatePosition,
  ]);

  useEffect(() => {
    if (!open) {
      return;
    }

    const handlePointerDown =
      (event: PointerEvent) => {
        const target =
          event.target as Node;

        const insideTrigger =
          rootRef.current?.contains(
            target,
          );

        const insideMenu =
          menuRef.current?.contains(
            target,
          );

        if (
          !insideTrigger &&
          !insideMenu
        ) {
          close();
        }
      };

    const handleKeyDown =
      (event: KeyboardEvent) => {
        if (
          event.key === "Escape"
        ) {
          close();
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
  }, [
    open,
    close,
  ]);

  useEffect(() => {
    if (
      !open ||
      !constrained
    ) {
      return;
    }

    const reposition =
      () => {
        calculatePosition();
      };

    window.addEventListener(
      "resize",
      reposition,
    );

    window.addEventListener(
      "scroll",
      reposition,
      true,
    );

    return () => {
      window.removeEventListener(
        "resize",
        reposition,
      );

      window.removeEventListener(
        "scroll",
        reposition,
        true,
      );
    };
  }, [
    open,
    constrained,
    calculatePosition,
  ]);

  const menu = open ? (
    <div
      ref={menuRef}
      role="menu"
      onPointerDown={(event) =>
        event.stopPropagation()
      }
      className={
        constrained
          ? "fixed z-[80] overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl"
          : `absolute ${
              align === "right"
                ? "right-0"
                : "left-0"
            } top-10 z-30 overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl`
      }
      style={
        constrained
          ? {
              width,
              left:
                position?.left ??
                -10000,

              top:
                position?.top ??
                -10000,

              visibility:
                position
                  ? "visible"
                  : "hidden",
            }
          : { width }
      }
    >
      {children(close)}
    </div>
  ) : null;

  return (
    <div
      ref={rootRef}
      className="relative"
    >
      <button
        ref={triggerRef}
        type="button"
        onClick={() => {
          if (open) {
            close();
          } else {
            setOpen(true);
          }
        }}
        aria-label={label}
        aria-expanded={open}
        className={`flex size-8 items-center justify-center transition hover:text-foreground ${
          variant === "chip"
            ? "glass-chip rounded-full text-muted-foreground"
            : "text-muted-foreground"
        } ${triggerClassName}`}
      >
        <Icon className="size-4" />
      </button>

      {constrained &&
      typeof document !==
        "undefined"
        ? createPortal(
            menu,
            document.body,
          )
        : menu}
    </div>
  );
}

export function IsaMenuItem({
  Icon,
  label,
  onClick,
  danger,
}: {
  Icon: typeof Pencil;
  label: string;
  onClick: () => void;
  danger?: boolean;
}) {
  return (
    <button
      type="button"
      onClick={onClick}
      className={`flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted ${
        danger
          ? "text-destructive"
          : "text-foreground"
      }`}
    >
      <Icon className="size-4" />
      {label}
    </button>
  );
}

export function IsaMenuCheckItem({
  label,
  checked,
  onToggle,
}: {
  label: string;
  checked: boolean;
  onToggle: () => void;
}) {
  return (
    <button
      type="button"
      role="menuitemcheckbox"
      aria-checked={checked}
      onClick={onToggle}
      className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left text-foreground transition hover:bg-muted"
    >
      <span
        className={`flex size-4 shrink-0 items-center justify-center rounded-[5px] border ${
          checked
            ? "gradient-brand border-transparent text-brand-foreground"
            : "border-border"
        }`}
      >
        {checked && (
          <Check className="size-3" />
        )}
      </span>

      <span className="min-w-0 flex-1 truncate text-[13px]">
        {label}
      </span>
    </button>
  );
}

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

