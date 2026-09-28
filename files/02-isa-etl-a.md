# 02-isa-etl-a.md

File in questo blocco:

- `src/components/isa/etl/data-preview.tsx`
- `src/components/isa/etl/inspector.tsx`
- `src/components/isa/etl/isa-context-menu.tsx`
- `src/components/isa/etl/settings-panels/aggregate-panel.tsx`
- `src/components/isa/etl/settings-panels/combine-panel.tsx`
- `src/components/isa/etl/settings-panels/filter-panel.tsx`
- `src/components/isa/etl/settings-panels/panel-controls.tsx`
- `src/components/isa/etl/tool-palette.tsx`

---

### `src/components/isa/etl/data-preview.tsx`

166 righe

```tsx
import { useEffect } from "react";
import { Table2, X } from "lucide-react";
import { useCanvasBoundsContext } from "@/canvas/components/CanvasContainer";
import { usePanelState } from "@/canvas/hooks/usePanelState";
import { computeSurfacePanelRect, surfacePanelStyle } from "@/canvas/layout/surfacePanels";
import { analyzeNode, formatRows, previewRows } from "@/lib/etl-schema";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";

/**
 * Fase 2A: Data Preview vive dentro la superficie zoomata (vedi
 * workflow-canvas.tsx / src/canvas/layout/surfacePanels.ts). Evita
 * l'Inspector automaticamente — tramite i bounds condivisi in
 * canvasStore, non più tramite il vecchio prop `inset` cablato a mano
 * con una larghezza magica (21.5rem).
 */

/** Altezza fisica: stessa proporzione (42% dell'altezza del canvas) della vecchia classe Tailwind max-h-[42%], ma calcolata (non un CSS max-height) perché ora è il canvas a doverne conoscere l'ingombro per i bounds. */
const PREVIEW_HEIGHT_RATIO = 0.42;
const PREVIEW_HEIGHT_MIN = 160;

/* Margini puramente estetici (stessi bottom-3/left-3/right-3 di prima) — non entrano nel calcolo dei bounds condivisi. */
const PREVIEW_INSET_LEFT = 12;
const PREVIEW_INSET_RIGHT = 12;
const PREVIEW_INSET_BOTTOM = 12;

function previewPhysicalHeight(containerHeight: number, zoom: number): number {
  return Math.max(PREVIEW_HEIGHT_MIN, Math.round(containerHeight * zoom * PREVIEW_HEIGHT_RATIO));
}

/** Cassetto Data preview: si apre solo su comando e si chiude con la X. */
export function DataPreview({
  workflow,
  node,
  open,
  onClose,
}: {
  workflow: EtlWorkflow;
  node: EtlNode | null;
  open: boolean;
  onClose: () => void;
}) {
  const { openPanel, closePanel, setSize } = usePanelState("data-preview");
  const { container, panels, zoom } = useCanvasBoundsContext();

  useEffect(() => {
    if (open) {
      openPanel();
    } else {
      closePanel();
    }
  }, [open, openPanel, closePanel]);

  useEffect(() => {
    const physicalHeight = previewPhysicalHeight(container.height, zoom);
    setSize({ width: 0, height: physicalHeight / zoom });
  }, [container.height, zoom, setSize]);

  if (!open) return null;

  const analysis = node ? analyzeNode(workflow, node) : null;
  const columns = analysis?.columns ?? [];
  const rows = node ? previewRows(columns, node.id) : [];

  const rect = computeSurfacePanelRect(
    container,
    panels,
    "data-preview",
    {
      side: "bottom",
      anchor: "stretch",
      physicalSize: { width: 0, height: previewPhysicalHeight(container.height, zoom) },
    },
    zoom,
  );

  const style = surfacePanelStyle(rect, zoom);

  return (
    <section
      className="glass-panel absolute z-40 flex flex-col rounded-3xl"
      style={{
        left: style.left + PREVIEW_INSET_LEFT / zoom,
        top: style.top,
        width: Math.max(0, style.width - PREVIEW_INSET_LEFT - PREVIEW_INSET_RIGHT),
        height: Math.max(0, style.height - PREVIEW_INSET_BOTTOM),
        transform: `scale(${1 / zoom})`,
        transformOrigin: "top left",
        transition: "left 0.2s ease, top 0.2s ease, width 0.2s ease",
        background: "color-mix(in srgb, var(--background) 92%, transparent)",
      }}
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
```

### `src/components/isa/etl/inspector.tsx`

273 righe

```tsx
import { useEffect } from "react";
import { AlertTriangle, Table2, Trash2, X } from "lucide-react";
import { useCanvasBoundsContext } from "@/canvas/components/CanvasContainer";
import { usePanelState } from "@/canvas/hooks/usePanelState";
import { computeSurfacePanelRect, surfacePanelStyle } from "@/canvas/layout/surfacePanels";
import type { ColumnDef } from "@/lib/etl-catalog";
import { categoryAccent, nodeDef } from "@/lib/etl-catalog";
import { analyzeNode, formatRows } from "@/lib/etl-schema";
import type { EtlNode, EtlWorkflow } from "@/lib/etl-workflow";

/**
 * Fase 2A: l'Inspector vive dentro la superficie zoomata del canvas
 * (vedi workflow-canvas.tsx) ma applica un controscale per restare a
 * dimensione fisica costante sullo schermo — vedi
 * src/canvas/layout/surfacePanels.ts per la tecnica e il perché.
 */

/** Larghezza fisica costante (px schermo, indipendente da zoom) — prima classe Tailwind w-[min(20rem,...)]. */
const INSPECTOR_WIDTH = 320;

/* Margini puramente estetici (spazio per i controlli zoom/griglia in alto, un po' d'aria in basso) — non entrano nel calcolo dei bounds condivisi. */
const INSPECTOR_INSET_TOP = 56;
const INSPECTOR_INSET_BOTTOM = 12;

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
  const { openPanel, closePanel, setSize } = usePanelState("inspector");
  const { container, panels, zoom } = useCanvasBoundsContext();

  /*
   * `open` (prop) resta la fonte di verità per "l'Inspector è aperto":
   * qui la specchiamo in canvasStore, così Data Preview (o un futuro
   * pannello) può sapere che deve farle spazio senza che nessuno gliela
   * passi esplicitamente (bug del vecchio `inset` prop, rimosso).
   */
  useEffect(() => {
    if (open) {
      openPanel();
    } else {
      closePanel();
    }
  }, [open, openPanel, closePanel]);

  /*
   * La larghezza è fisica-costante (non dipende dal contenuto), quindi
   * non serve un ResizeObserver: solo ricalcolare la sua controparte in
   * unità superficie (÷ zoom) quando zoom o lo spazio disponibile
   * cambiano.
   */
  useEffect(() => {
    const physicalWidth = Math.max(0, Math.min(INSPECTOR_WIDTH, container.width * zoom - 24));
    setSize({ width: physicalWidth / zoom, height: 0 });
  }, [container.width, zoom, setSize]);

  if (!open) return null;

  const def = node ? nodeDef(node.type) : undefined;
  const analysis = node ? analyzeNode(workflow, node) : null;

  const rect = computeSurfacePanelRect(
    container,
    panels,
    "inspector",
    { side: "right", anchor: "stretch", physicalSize: { width: INSPECTOR_WIDTH, height: 0 } },
    zoom,
  );

  const style = surfacePanelStyle(rect, zoom);

  return (
    <aside
      className="glass-panel absolute z-40 flex flex-col rounded-3xl p-4"
      style={{
        left: style.left,
        top: style.top + INSPECTOR_INSET_TOP / zoom,
        width: style.width,
        height: Math.max(0, style.height - INSPECTOR_INSET_TOP - INSPECTOR_INSET_BOTTOM),
        transform: `scale(${1 / zoom})`,
        transformOrigin: "top left",
        transition: "left 0.2s ease, top 0.2s ease, height 0.2s ease",
        background: "color-mix(in srgb, var(--background) 92%, transparent)",
      }}
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
```

### `src/components/isa/etl/isa-context-menu.tsx`

321 righe

```tsx
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
```

### `src/components/isa/etl/settings-panels/aggregate-panel.tsx`

150 righe

```tsx
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
```

### `src/components/isa/etl/settings-panels/combine-panel.tsx`

322 righe

```tsx
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
```

### `src/components/isa/etl/settings-panels/filter-panel.tsx`

168 righe

```tsx
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
```

### `src/components/isa/etl/settings-panels/panel-controls.tsx`

120 righe

```tsx
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
```

### `src/components/isa/etl/tool-palette.tsx`

357 righe

```tsx
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
import { useCallback, useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";
import { IsaMenu, IsaMenuItem } from "@/components/isa/ui/isa-menu";
import type { EtlCategory, EtlNodeDef } from "@/lib/etl-catalog";
import { ETL_CATEGORIES, ETL_NODES, categoryAccent } from "@/lib/etl-catalog";
import { CATEGORY_ICONS } from "@/lib/etl-display";

export type Dock = "top" | "bottom" | "left" | "right";

/*
 * Bug 1.1: sotto una soglia minima di spostamento un pointerdown+up
 * sul chip è un click/doppio click, non un drag — evita di far
 * scattare un fantasma di drag per una semplice pressione.
 */
const RESOURCE_DRAG_THRESHOLD = 4;

type ResourceDrag = {
  node: EtlNodeDef;
  x: number;
  y: number;
};

function NodeChip({
  node,
  onAdd,
  onDragStart,
}: {
  node: EtlNodeDef;
  /** Doppio click (PARTE B): niente più singolo click per aggiungere. */
  onAdd: (type: string, clientX: number, clientY: number) => void;
  /**
   * Bug 1.1: avvio del drag-to-canvas via Pointer Events, non più HTML5
   * Drag & Drop (in conflitto con i Pointer Events usati ovunque altro
   * sul canvas — card, palette, porte). Lo stato del drag vive nel
   * genitore ToolPalette (un solo ghost alla volta, coerente con la
   * palette stessa).
   */
  onDragStart: (
    node: EtlNodeDef,
    event: React.PointerEvent<HTMLButtonElement>,
  ) => void;
}) {
  const { label, description, Icon, category } = node;

  const handleDoubleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
    event.stopPropagation();
    onAdd(node.type, event.clientX, event.clientY);
  };

  return (
    <button
      type="button"
      onPointerDown={(event) => onDragStart(node, event)}
      onDoubleClick={handleDoubleClick}
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
  onDragAdd,
}: {
  dock: Dock;
  onDockChange: (dock: Dock) => void;
  /** Doppio click su un NodeChip (PARTE B): coordinate schermo del click, la conversione in coordinate canvas la fa WorkflowCanvas. */
  onAdd: (type: string, clientX: number, clientY: number) => void;
  /**
   * Bug 1.1: drag-and-drop di un NodeChip sul canvas, coordinate
   * schermo del punto di rilascio (stessa conversione di `onAdd`, fatta
   * da WorkflowCanvas). Chiamato solo se il rilascio avviene
   * effettivamente sopra il canvas.
   */
  onDragAdd: (type: string, clientX: number, clientY: number) => void;
}) {
  const [open, setOpen] = useState<EtlCategory | null>(null);
  const [expanded, setExpanded] = useState(false);
  const [ghost, setGhost] = useState<{ x: number; y: number } | null>(null);
  const [resourceDrag, setResourceDrag] = useState<ResourceDrag | null>(null);
  const ref = useRef<HTMLDivElement>(null);
  const workspaceRef = useRef<HTMLElement | null>(null);
  const vertical = dock === "left" || dock === "right";
  const secondLevel = expanded || open !== null;
  const dragging = ghost !== null;

  useEffect(() => {
    workspaceRef.current =
      ref.current?.closest<HTMLElement>("[data-palette-workspace]") ?? null;
  }, []);

  const startResourceDrag = useCallback(
    (node: EtlNodeDef, event: React.PointerEvent<HTMLButtonElement>) => {
      if (event.button !== 0) return;

      const startX = event.clientX;
      const startY = event.clientY;
      let dragStarted = false;

      const move = (pointer: PointerEvent) => {
        if (!dragStarted) {
          const moved = Math.hypot(pointer.clientX - startX, pointer.clientY - startY);
          if (moved < RESOURCE_DRAG_THRESHOLD) return;
          dragStarted = true;
        }
        setResourceDrag({ node, x: pointer.clientX, y: pointer.clientY });
      };

      const cleanup = () => {
        window.removeEventListener("pointermove", move);
        window.removeEventListener("pointerup", finish);
        window.removeEventListener("pointercancel", cleanup);
      };

      const finish = (pointer: PointerEvent) => {
        cleanup();
        setResourceDrag(null);

        if (!dragStarted) return;

        // Il rilascio conta solo se avviene sopra il canvas: fuori da
        // lì (es. sulla stessa palette) il drag viene semplicemente
        // annullato, coerente col comportamento di un vero drag & drop.
        const dropTarget = document.elementFromPoint(pointer.clientX, pointer.clientY);
        if (dropTarget?.closest("[data-palette-workspace]")) {
          onDragAdd(node.type, pointer.clientX, pointer.clientY);
        }
      };

      window.addEventListener("pointermove", move);
      window.addEventListener("pointerup", finish);
      window.addEventListener("pointercancel", cleanup);
    },
    [onDragAdd],
  );

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

      <IsaMenu
        label="Posizione barra"
        Icon={vertical ? AlignStartVertical : AlignStartHorizontal}
        boundaryRef={workspaceRef}
        placement="auto"
      >
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
                {nodes.map((node) => (
                  <NodeChip
                    key={node.type}
                    node={node}
                    onAdd={onAdd}
                    onDragStart={startResourceDrag}
                  />
                ))}
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

      {resourceDrag &&
        createPortal(
          <div
            aria-hidden
            className="glass-panel animate-scale-in pointer-events-none fixed z-50 flex items-center gap-1.5 rounded-lg px-2.5 py-1.5 text-[12px] font-medium text-foreground shadow-xl"
            style={{ left: resourceDrag.x + 14, top: resourceDrag.y + 14 }}
          >
            <span
              className={`${categoryAccent(resourceDrag.node.category)} flex size-5 shrink-0 items-center justify-center rounded-md`}
            >
              <resourceDrag.node.Icon className="size-3" />
            </span>
            <span className="truncate">{resourceDrag.node.label}</span>
          </div>,
          document.body,
        )}
    </>
  );
}
```

