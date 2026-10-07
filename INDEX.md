# INDEX.md

Generato: 2026-10-07T11:48:41Z (UTC)
Repository sorgente: isa-glass-platform, branch `feat/inspector-conditions`, commit `852be600e89afc9d35eea8d412b724e6a0e4c4b2`
Working tree del repository sorgente: pulito (nessuna modifica non committata).

Questo indice è fissato al commit `f1745fa75954c07472385d17ecf86fe652ef773c` del repository snapshot (isa-etl-snapshot): tutti gli URL sotto puntano a quel commit e restano validi anche dopo aggiornamenti futuri.

## Da leggere per primi

1. [STATUS.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/STATUS.md) — stato di type check, lint, test, build
2. [ENV.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/ENV.md) — configurazione completa (package.json, tsconfig, vite, eslint, CSS)
3. [TREE.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/TREE.md) — albero completo del repository
4. I blocchi in `files/`, in ordine, elencati sotto.

## Blocchi (files/)

### [files/01-canvas.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01-canvas.md)

55.6 KB. File sorgente contenuti:

- `src/canvas/FUNCTIONAL_CHECKS.md`
- `src/canvas/README.md`
- `src/canvas/__tests__/panelPositioning.test.ts`
- `src/canvas/components/CanvasContainer.tsx`
- `src/canvas/hooks/useCanvasBounds.ts`
- `src/canvas/hooks/usePanelState.ts`
- `src/canvas/layout/__tests__/canvasBounds.test.ts`
- `src/canvas/layout/__tests__/dropZones.test.ts`
- `src/canvas/layout/__tests__/panelRegistry.test.ts`
- `src/canvas/layout/canvasBounds.ts`
- `src/canvas/layout/dropZones.ts`
- `src/canvas/layout/panelRegistry.ts`
- `src/canvas/layout/surfacePanels.ts`
- `src/canvas/store/canvasStore.tsx`

### [files/01b-etl-core-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01b-etl-core-a.md)

57.3 KB. File sorgente contenuti:

- `src/etl-core/NOTE_DIVERGENZE.md`
- `src/etl-core/README.md`
- `src/etl-core/__tests__/csv.test.ts`
- `src/etl-core/__tests__/expressions.test.ts`
- `src/etl-core/__tests__/fase11-requisiti.test.ts`
- `src/etl-core/__tests__/fase11.test.ts`
- `src/etl-core/__tests__/helpers.ts`
- `src/etl-core/__tests__/join-tables.test.ts`

### [files/01b-etl-core-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01b-etl-core-b.md)

51.9 KB. File sorgente contenuti:

- `src/etl-core/__tests__/multi-columns.test.ts`
- `src/etl-core/__tests__/mutations.test.ts`
- `src/etl-core/__tests__/params.test.ts`
- `src/etl-core/__tests__/relations.test.ts`
- `src/etl-core/__tests__/schema.test.ts`
- `src/etl-core/__tests__/state.test.ts`
- `src/etl-core/catalog/icons.ts`
- `src/etl-core/catalog/operations.ts`

### [files/01b-etl-core-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01b-etl-core-c.md)

53.2 KB. File sorgente contenuti:

- `src/etl-core/catalog/params.ts`
- `src/etl-core/data/csv.ts`
- `src/etl-core/index.ts`
- `src/etl-core/logic/expressions.ts`
- `src/etl-core/model/graph.ts`
- `src/etl-core/model/types.ts`

### [files/01b-etl-core-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01b-etl-core-d.md)

29.3 KB. File sorgente contenuti:

- `src/etl-core/rules/mutations.ts`
- `src/etl-core/rules/relations.ts`
- `src/etl-core/rules/state.ts`
- `src/etl-core/schema/schema.ts`

### [files/01c-etl-layout-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01c-etl-layout-a.md)

56.7 KB. File sorgente contenuti:

- `src/etl-layout/NOTE_DIVERGENZE.md`
- `src/etl-layout/README.md`
- `src/etl-layout/__tests__/golden.test.ts`
- `src/etl-layout/__tests__/golden/01-dritto-allineati.json`
- `src/etl-layout/__tests__/golden/02-dritto-scorrimento.json`
- `src/etl-layout/__tests__/golden/03-oltre-scorrimento.json`
- `src/etl-layout/__tests__/golden/04-ostacolo.json`
- `src/etl-layout/__tests__/golden/05-incrocio.json`
- `src/etl-layout/__tests__/golden/06-corsie.json`
- `src/etl-layout/__tests__/golden/07-join-output-parziale.json`

### [files/01c-etl-layout-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01c-etl-layout-b.md)

51.2 KB. File sorgente contenuti:

- `src/etl-layout/__tests__/golden/08-spostamento.json`
- `src/etl-layout/__tests__/golden/09-catena-riordino.json`
- `src/etl-layout/__tests__/golden/10-riordino-isolati.json`
- `src/etl-layout/__tests__/golden/10b-riordino-colonna-fitta.json`
- `src/etl-layout/__tests__/golden/11-organizzato.json`
- `src/etl-layout/__tests__/golden/12-output-generato.json`
- `src/etl-layout/__tests__/golden/13-output-organizzato.json`
- `src/etl-layout/__tests__/golden/corretti/06-corsie.json`
- `src/etl-layout/__tests__/golden/corretti/09-catena-riordino.json`
- `src/etl-layout/__tests__/properties.test.ts`

### [files/01c-etl-layout-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01c-etl-layout-c.md)

54.2 KB. File sorgente contenuti:

- `src/etl-layout/__tests__/unit.test.ts`
- `src/etl-layout/autoLayout.ts`
- `src/etl-layout/constants.ts`
- `src/etl-layout/free.ts`
- `src/etl-layout/hitTest.ts`
- `src/etl-layout/index.ts`
- `src/etl-layout/links.ts`
- `src/etl-layout/nodes.ts`
- `src/etl-layout/path.ts`

### [files/01c-etl-layout-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01c-etl-layout-d.md)

30.5 KB. File sorgente contenuti:

- `src/etl-layout/placement.ts`
- `src/etl-layout/routing.ts`
- `src/etl-layout/slots.ts`
- `src/etl-layout/types.ts`

### [files/01d-etl-store-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01d-etl-store-a.md)

50.2 KB. File sorgente contenuti:

- `src/etl-store/README.md`
- `src/etl-store/__tests__/grouping.test.ts`
- `src/etl-store/__tests__/helpers.ts`
- `src/etl-store/__tests__/persistence.test.ts`
- `src/etl-store/__tests__/react.test.ts`

### [files/01d-etl-store-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01d-etl-store-b.md)

46.3 KB. File sorgente contenuti:

- `src/etl-store/__tests__/reduce.test.ts`
- `src/etl-store/__tests__/save-versions.test.ts`
- `src/etl-store/__tests__/store.test.ts`
- `src/etl-store/derived.ts`
- `src/etl-store/index.ts`
- `src/etl-store/persistence.ts`
- `src/etl-store/react.ts`

### [files/01d-etl-store-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01d-etl-store-c.md)

58.7 KB. File sorgente contenuti:

- `src/etl-store/reduce.ts`
- `src/etl-store/serialize.ts`
- `src/etl-store/state.ts`
- `src/etl-store/store.ts`
- `src/etl-store/types.ts`

### [files/01e-etl-canvas-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-a.md)

41.9 KB. File sorgente contenuti:

- `src/etl-canvas/EtlCanvas.tsx`
- `src/etl-canvas/Links.tsx`
- `src/etl-canvas/Minimap.tsx`
- `src/etl-canvas/NOTE_DIVERGENZE.md`
- `src/etl-canvas/Node.tsx`

### [files/01e-etl-canvas-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-b.md)

51.6 KB. File sorgente contenuti:

- `src/etl-canvas/README.md`
- `src/etl-canvas/__tests__/animator.test.ts`

### [files/01e-etl-canvas-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-c.md)

53.6 KB. File sorgente contenuti:

- `src/etl-canvas/__tests__/autofit.test.ts`
- `src/etl-canvas/__tests__/conditions-ui.test.tsx`
- `src/etl-canvas/__tests__/conditions.test.ts`
- `src/etl-canvas/__tests__/controlbar.test.tsx`
- `src/etl-canvas/__tests__/drop.test.ts`

### [files/01e-etl-canvas-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-d.md)

58.6 KB. File sorgente contenuti:

- `src/etl-canvas/__tests__/engine.test.ts`
- `src/etl-canvas/__tests__/fake-env.ts`
- `src/etl-canvas/__tests__/flow.test.ts`
- `src/etl-canvas/__tests__/gesture-render.test.ts`
- `src/etl-canvas/__tests__/helpers.ts`
- `src/etl-canvas/__tests__/inspector-logic.test.ts`
- `src/etl-canvas/__tests__/inspector-rules.test.ts`
- `src/etl-canvas/__tests__/inspector.test.tsx`

### [files/01e-etl-canvas-e.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-e.md)

50.2 KB. File sorgente contenuti:

- `src/etl-canvas/__tests__/interaction.test.ts`
- `src/etl-canvas/__tests__/keyboard.test.ts`
- `src/etl-canvas/__tests__/loop.test.ts`
- `src/etl-canvas/__tests__/menu.test.ts`
- `src/etl-canvas/__tests__/no-reroute.test.ts`
- `src/etl-canvas/__tests__/overlay-layout.test.ts`

### [files/01e-etl-canvas-f.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-f.md)

58.5 KB. File sorgente contenuti:

- `src/etl-canvas/__tests__/panels-actions.test.ts`
- `src/etl-canvas/__tests__/panels-layout.test.ts`
- `src/etl-canvas/__tests__/render.test.ts`
- `src/etl-canvas/__tests__/reorder.test.ts`
- `src/etl-canvas/__tests__/segmented.test.tsx`
- `src/etl-canvas/__tests__/ssr.test.tsx`
- `src/etl-canvas/__tests__/tokens.test.ts`
- `src/etl-canvas/__tests__/toolbox-drop.test.ts`
- `src/etl-canvas/__tests__/toolbox.test.tsx`
- `src/etl-canvas/__tests__/transitions.test.ts`

### [files/01e-etl-canvas-g.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-g.md)

53.9 KB. File sorgente contenuti:

- `src/etl-canvas/__tests__/view.test.ts`
- `src/etl-canvas/actions.ts`
- `src/etl-canvas/canvas.css`
- `src/etl-canvas/contrast.ts`
- `src/etl-canvas/drop.ts`
- `src/etl-canvas/engine.ts`
- `src/etl-canvas/flow.ts`
- `src/etl-canvas/icons.tsx`
- `src/etl-canvas/index.ts`
- `src/etl-canvas/inspector/ActionMenu.tsx`
- `src/etl-canvas/inspector/BlockedNotice.tsx`

### [files/01e-etl-canvas-h.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-h.md)

58.6 KB. File sorgente contenuti:

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

### [files/01e-etl-canvas-i.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-i.md)

59.0 KB. File sorgente contenuti:

- `src/etl-canvas/inspector/ListRow.tsx`
- `src/etl-canvas/inspector/Menu.tsx`
- `src/etl-canvas/inspector/MultiList.tsx`
- `src/etl-canvas/inspector/NameInput.tsx`
- `src/etl-canvas/inspector/Segmented.tsx`
- `src/etl-canvas/inspector/StepList.tsx`
- `src/etl-canvas/inspector/StyledSelect.tsx`
- `src/etl-canvas/inspector/ValuePicker.tsx`
- `src/etl-canvas/inspector/conditions.ts`

### [files/01e-etl-canvas-j.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-j.md)

56.5 KB. File sorgente contenuti:

- `src/etl-canvas/inspector/copy.ts`
- `src/etl-canvas/inspector/family.ts`
- `src/etl-canvas/inspector/icons.tsx`
- `src/etl-canvas/inspector/inspector.css`
- `src/etl-canvas/inspector/joinKeys.ts`
- `src/etl-canvas/inspector/joinSides.ts`
- `src/etl-canvas/inspector/logic.ts`
- `src/etl-canvas/inspector/masterDetail.ts`
- `src/etl-canvas/inspector/menu.ts`
- `src/etl-canvas/inspector/params.ts`
- `src/etl-canvas/inspector/useActiveSchema.ts`

### [files/01e-etl-canvas-k.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-k.md)

55.1 KB. File sorgente contenuti:

- `src/etl-canvas/inspector/useReorder.ts`
- `src/etl-canvas/interaction.ts`
- `src/etl-canvas/loop.ts`
- `src/etl-canvas/model.ts`
- `src/etl-canvas/motion.tsx`
- `src/etl-canvas/panels/ControlBar.tsx`
- `src/etl-canvas/panels/Dock.tsx`

### [files/01e-etl-canvas-l.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-l.md)

59.5 KB. File sorgente contenuti:

- `src/etl-canvas/panels/EtlWorkspace.tsx`
- `src/etl-canvas/panels/InspectorShell.tsx`
- `src/etl-canvas/panels/Toolbox.tsx`
- `src/etl-canvas/panels/actions.ts`
- `src/etl-canvas/panels/animator.ts`
- `src/etl-canvas/panels/autoFit.ts`
- `src/etl-canvas/panels/csv.ts`
- `src/etl-canvas/panels/dockArea.ts`
- `src/etl-canvas/panels/families.ts`
- `src/etl-canvas/panels/layout.ts`
- `src/etl-canvas/panels/overlayLayout.ts`

### [files/01e-etl-canvas-m.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/01e-etl-canvas-m.md)

36.9 KB. File sorgente contenuti:

- `src/etl-canvas/panels/panels.css`
- `src/etl-canvas/panels/ui-icons.tsx`
- `src/etl-canvas/seed.ts`
- `src/etl-canvas/tokens.css`
- `src/etl-canvas/transitions.ts`
- `src/etl-canvas/view.ts`

### [files/02-isa-etl-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/02-isa-etl-a.md)

52.7 KB. File sorgente contenuti:

- `src/components/isa/etl/data-preview.tsx`
- `src/components/isa/etl/inspector.tsx`
- `src/components/isa/etl/isa-context-menu.tsx`
- `src/components/isa/etl/settings-panels/aggregate-panel.tsx`
- `src/components/isa/etl/settings-panels/combine-panel.tsx`
- `src/components/isa/etl/settings-panels/filter-panel.tsx`
- `src/components/isa/etl/settings-panels/panel-controls.tsx`
- `src/components/isa/etl/tool-palette.tsx`

### [files/02-isa-etl-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/02-isa-etl-b.md)

49.9 KB. File sorgente contenuti:

- `src/components/isa/etl/workflow-canvas.tsx`

### [files/02-isa-etl-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/02-isa-etl-c.md)

54.2 KB. File sorgente contenuti:

- `src/components/isa/etl/workflow-canvas.tsx`

### [files/03-components-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/03-components-a.md)

56.3 KB. File sorgente contenuti:

- `src/components/isa/app-shell.tsx`
- `src/components/isa/back-button.tsx`
- `src/components/isa/header.tsx`
- `src/components/isa/logo.tsx`
- `src/components/isa/mini-chart.tsx`
- `src/components/isa/module-picker-modal.tsx`
- `src/components/isa/new-solution-modal.tsx`
- `src/components/isa/share-modal.tsx`
- `src/components/isa/sidebar.tsx`
- `src/components/isa/solution-card.tsx`
- `src/components/isa/solution-row.tsx`
- `src/components/isa/ui/isa-menu.tsx`
- `src/components/isa/ui/isa-modal.tsx`

### [files/03-components-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/03-components-b.md)

55.3 KB. File sorgente contenuti:

- `src/components/isa/widget-panel.tsx`
- `src/components/ui/accordion.tsx`
- `src/components/ui/alert-dialog.tsx`
- `src/components/ui/alert.tsx`
- `src/components/ui/aspect-ratio.tsx`
- `src/components/ui/avatar.tsx`
- `src/components/ui/badge.tsx`
- `src/components/ui/breadcrumb.tsx`
- `src/components/ui/button.tsx`
- `src/components/ui/calendar.tsx`
- `src/components/ui/card.tsx`
- `src/components/ui/carousel.tsx`
- `src/components/ui/chart.tsx`
- `src/components/ui/checkbox.tsx`
- `src/components/ui/collapsible.tsx`
- `src/components/ui/command.tsx`

### [files/03-components-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/03-components-c.md)

55.5 KB. File sorgente contenuti:

- `src/components/ui/context-menu.tsx`
- `src/components/ui/dialog.tsx`
- `src/components/ui/drawer.tsx`
- `src/components/ui/dropdown-menu.tsx`
- `src/components/ui/form.tsx`
- `src/components/ui/hover-card.tsx`
- `src/components/ui/input-otp.tsx`
- `src/components/ui/input.tsx`
- `src/components/ui/label.tsx`
- `src/components/ui/menubar.tsx`
- `src/components/ui/navigation-menu.tsx`
- `src/components/ui/pagination.tsx`
- `src/components/ui/popover.tsx`
- `src/components/ui/progress.tsx`
- `src/components/ui/radio-group.tsx`
- `src/components/ui/resizable.tsx`
- `src/components/ui/scroll-area.tsx`

### [files/03-components-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/03-components-d.md)

49.2 KB. File sorgente contenuti:

- `src/components/ui/select.tsx`
- `src/components/ui/separator.tsx`
- `src/components/ui/sheet.tsx`
- `src/components/ui/sidebar.tsx`
- `src/components/ui/skeleton.tsx`
- `src/components/ui/slider.tsx`
- `src/components/ui/sonner.tsx`
- `src/components/ui/switch.tsx`
- `src/components/ui/table.tsx`
- `src/components/ui/tabs.tsx`
- `src/components/ui/textarea.tsx`
- `src/components/ui/toggle-group.tsx`
- `src/components/ui/toggle.tsx`
- `src/components/ui/tooltip.tsx`

### [files/04-lib-hooks-store-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/04-lib-hooks-store-a.md)

49.8 KB. File sorgente contenuti:

- `src/hooks/use-mobile.tsx`
- `src/lib/error-capture.ts`
- `src/lib/error-page.ts`
- `src/lib/etl-bubble.ts`
- `src/lib/etl-catalog.ts`
- `src/lib/etl-display.ts`
- `src/lib/etl-motion.ts`
- `src/lib/etl-node-config.ts`
- `src/lib/etl-node-size.ts`
- `src/lib/etl-schema.ts`

### [files/04-lib-hooks-store-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/04-lib-hooks-store-b.md)

27.8 KB. File sorgente contenuti:

- `src/lib/etl-workflow.tsx`
- `src/lib/modules.ts`
- `src/lib/solutions-store.tsx`
- `src/lib/theme.test.tsx`
- `src/lib/theme.tsx`
- `src/lib/utils.ts`

### [files/05-app-pages-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/05-app-pages-a.md)

59.0 KB. File sorgente contenuti:

- `src/routeTree.gen.ts`
- `src/router.tsx`
- `src/routes/README.md`
- `src/routes/__root.tsx`
- `src/routes/activity.tsx`
- `src/routes/favorites.tsx`
- `src/routes/index.tsx`
- `src/routes/settings.tsx`
- `src/routes/shared.tsx`
- `src/routes/solutions.$solutionId.dashboard.tsx`
- `src/routes/solutions.$solutionId.etl.tsx`
- `src/routes/solutions.$solutionId.index.tsx`
- `src/routes/solutions.$solutionId.model.tsx`
- `src/routes/solutions.$solutionId.tsx`
- `src/routes/teams.tsx`
- `src/routes/templates.tsx`
- `src/routes/trash.tsx`
- `src/routes/users.tsx`
- `src/server.ts`

### [files/05-app-pages-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/05-app-pages-b.md)

1.1 KB. File sorgente contenuti:

- `src/start.ts`

### [files/06-styles.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/06-styles.md)

7.5 KB. File sorgente contenuti:

- `src/styles.css`

### [files/08-scripts-config-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/08-scripts-config-a.md)

58.7 KB. File sorgente contenuti:

- `.devcontainer/devcontainer.json`
- `.gitignore`
- `.prettierignore`
- `.prettierrc`
- `components.json`
- `eslint.config.js`
- `package.json`
- `scripts/check-tokens.mjs`
- `scripts/e2e-fase5.mjs`
- `scripts/e2e-fase6a.mjs`

### [files/08-scripts-config-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/08-scripts-config-b.md)

37.8 KB. File sorgente contenuti:

- `scripts/e2e-fase6b1.mjs`

### [files/08-scripts-config-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/08-scripts-config-c.md)

33.6 KB. File sorgente contenuti:

- `scripts/e2e-fase6b11.mjs`

### [files/08-scripts-config-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/08-scripts-config-d.md)

57.1 KB. File sorgente contenuti:

- `scripts/e2e-fase6b2.mjs`
- `scripts/extract-golden.mjs`
- `scripts/generate-index.mjs`

### [files/08-scripts-config-e.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/08-scripts-config-e.md)

57.9 KB. File sorgente contenuti:

- `scripts/generate-snapshot.mjs`
- `scripts/snapshot-lib.mjs`
- `scripts/snapshot-lib.test.mjs`
- `scripts/sync-snapshot.sh`
- `scripts/theme-map.mjs`
- `scripts/token-legacy-files.txt`
- `scripts/visual-compare.mjs`
- `scripts/visual-fase4.mjs`

### [files/08-scripts-config-f.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/08-scripts-config-f.md)

23.6 KB. File sorgente contenuti:

- `scripts/visual-fase4b.mjs`
- `scripts/visual-lib.mjs`
- `scripts/visual-temi.mjs`
- `tsconfig.json`
- `vite.config.ts`
- `vitest.config.ts`

### [files/09-docs.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/09-docs.md)

3.6 KB. File sorgente contenuti:

- `AGENTS.md`
- `README.md`
- `roadmap.md`

### [files/10-prototype-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/10-prototype-a.md)

49.8 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/10-prototype-b.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/10-prototype-c.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/10-prototype-d.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-e.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/10-prototype-e.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-f.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/10-prototype-f.md)

11.1 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/11-inventory.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/11-inventory.md)

52.7 KB. File sorgente contenuti:

- `docs/inventory/INVENTARIO_2026-09-28T19-40-33Z.md`
- `docs/inventory/lovable-1e72955.diff`

### [files/12-docs-other-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/12-docs-other-a.md)

42.2 KB. File sorgente contenuti:

- `docs/theme-debt.md`
- `docs/visual/fase4/console.txt`
- `docs/visual/fase4/misure.json`
- `docs/visual/fase4b/console.txt`
- `docs/visual/fase4b/misure.json`
- `docs/visual/fase6a/posizioni-nodi.json`
- `docs/visual/fase6b11/REPORT.md`

### [files/12-docs-other-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/12-docs-other-b.md)

49.8 KB. File sorgente contenuti:

- `docs/visual/fase6b11/misure.json`

### [files/12-docs-other-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/12-docs-other-c.md)

28.0 KB. File sorgente contenuti:

- `docs/visual/fase6b11/misure.json`

### [files/12-docs-other-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/12-docs-other-d.md)

49.9 KB. File sorgente contenuti:

- `docs/visual/fase6b2/misure.json`

### [files/12-docs-other-e.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/12-docs-other-e.md)

17.1 KB. File sorgente contenuti:

- `docs/visual/fase6b2/misure.json`

### [files/13-misc-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/13-misc-a.md)

59.0 KB. File sorgente contenuti:

- `src/theme/README.md`
- `src/theme/__tests__/check-tokens.test.ts`
- `src/theme/__tests__/checks.ts`
- `src/theme/__tests__/contrast.test.ts`
- `src/theme/__tests__/derive.test.ts`
- `src/theme/__tests__/readme.test.ts`
- `src/theme/__tests__/runtime.test.ts`
- `src/theme/__tests__/support.ts`

### [files/13-misc-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/files/13-misc-b.md)

56.6 KB. File sorgente contenuti:

- `src/theme/__tests__/themes.test.ts`
- `src/theme/boot.ts`
- `src/theme/color.ts`
- `src/theme/derive.ts`
- `src/theme/index.css`
- `src/theme/index.ts`
- `src/theme/layout-tokens.css`
- `src/theme/primitives.css`
- `src/theme/runtime.ts`
- `src/theme/themes/notte.css`
- `src/theme/themes/prototipo.css`

## Report (reports/)

- [VALIDATION_REPORT_2026-09-19T10-24-53Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-19T10-24-53Z.md)
- [VALIDATION_REPORT_2026-09-19T11-38-16Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-19T11-38-16Z.md)
- [VALIDATION_REPORT_2026-09-28T18-45-29Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-28T18-45-29Z.md)
- [VALIDATION_REPORT_2026-09-28T19-44-43Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-28T19-44-43Z.md)
- [VALIDATION_REPORT_2026-09-28T20-14-48Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-28T20-14-48Z.md)
- [VALIDATION_REPORT_2026-09-28T21-06-48Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-28T21-06-48Z.md)
- [VALIDATION_REPORT_2026-09-29T07-20-03Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T07-20-03Z.md)
- [VALIDATION_REPORT_2026-09-29T08-33-03Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T08-33-03Z.md)
- [VALIDATION_REPORT_2026-09-29T08-57-43Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T08-57-43Z.md)
- [VALIDATION_REPORT_2026-09-29T09-28-26Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T09-28-26Z.md)
- [VALIDATION_REPORT_2026-09-29T09-39-17Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T09-39-17Z.md)
- [VALIDATION_REPORT_2026-09-29T15-34-16Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T15-34-16Z.md)
- [VALIDATION_REPORT_2026-09-29T16-50-41Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T16-50-41Z.md)
- [VALIDATION_REPORT_2026-09-29T17-13-50Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T17-13-50Z.md)
- [VALIDATION_REPORT_2026-09-29T19-51-32Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-09-29T19-51-32Z.md)
- [VALIDATION_REPORT_2026-10-02T12-35-11Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-10-02T12-35-11Z.md)
- [VALIDATION_REPORT_2026-10-02T13-25-09Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-10-02T13-25-09Z.md)
- [VALIDATION_REPORT_2026-10-04T07-50-59Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-10-04T07-50-59Z.md)
- [VALIDATION_REPORT_2026-10-04T08-45-00Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-10-04T08-45-00Z.md)
- [VALIDATION_REPORT_2026-10-04T10-30-00Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-10-04T10-30-00Z.md)
- [VALIDATION_REPORT_2026-10-04T11-15-00Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-10-04T11-15-00Z.md)
- [VALIDATION_REPORT_2026-10-04T14-30-00Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/f1745fa75954c07472385d17ecf86fe652ef773c/reports/VALIDATION_REPORT_2026-10-04T14-30-00Z.md)

## File esclusi dallo snapshot

(elencati per riferimento in TREE.md, contenuto non incluso in files/)

- `docs/visual/fase4/cavi-chiaro.png` — motivo: binary
- `docs/visual/fase4/cavi-prototipo.png` — motivo: binary
- `docs/visual/fase4/cavi-scuro.png` — motivo: binary
- `docs/visual/fase4/crop-combinato-chiaro.png` — motivo: binary
- `docs/visual/fase4/crop-combinato-prototipo.png` — motivo: binary
- `docs/visual/fase4/crop-combinato-scuro.png` — motivo: binary
- `docs/visual/fase4/crop-dataset-chiaro.png` — motivo: binary
- `docs/visual/fase4/crop-dataset-prototipo.png` — motivo: binary
- `docs/visual/fase4/crop-dataset-scuro.png` — motivo: binary
- `docs/visual/fase4/crop-lavorazione-chiaro.png` — motivo: binary
- `docs/visual/fase4/crop-lavorazione-prototipo.png` — motivo: binary
- `docs/visual/fase4/crop-lavorazione-scuro.png` — motivo: binary
- `docs/visual/fase4/crop-output-parziale-chiaro.png` — motivo: binary
- `docs/visual/fase4/crop-output-parziale-prototipo.png` — motivo: binary
- `docs/visual/fase4/crop-output-parziale-scuro.png` — motivo: binary
- `docs/visual/fase4/crop-output-pieno-chiaro.png` — motivo: binary
- `docs/visual/fase4/crop-output-pieno-prototipo.png` — motivo: binary
- `docs/visual/fase4/crop-output-pieno-scuro.png` — motivo: binary
- `docs/visual/fase4/prototipo.png` — motivo: binary
- `docs/visual/fase4/v2-chiaro.png` — motivo: binary
- `docs/visual/fase4/v2-scuro.png` — motivo: binary
- `docs/visual/fase4b/prototipo-t0.png` — motivo: binary
- `docs/visual/fase4b/prototipo-t250.png` — motivo: binary
- `docs/visual/fase4b/prototipo-t500.png` — motivo: binary
- `docs/visual/fase4b/prototipo.webm` — motivo: binary
- `docs/visual/fase4b/v2-chiaro-movimento-ridotto.png` — motivo: binary
- `docs/visual/fase4b/v2-chiaro-t0.png` — motivo: binary
- `docs/visual/fase4b/v2-chiaro-t250.png` — motivo: binary
- `docs/visual/fase4b/v2-chiaro-t500.png` — motivo: binary
- `docs/visual/fase4b/v2-chiaro.webm` — motivo: binary
- `docs/visual/fase4b/v2-scuro-t0.png` — motivo: binary
- `docs/visual/fase4b/v2-scuro-t250.png` — motivo: binary
- `docs/visual/fase4b/v2-scuro-t500.png` — motivo: binary
- `docs/visual/fase5/conferma-eliminazione.png` — motivo: binary
- `docs/visual/fase5/riquadro-selezione.png` — motivo: binary
- `docs/visual/fase5/trascinamento-collegamento.png` — motivo: binary
- `docs/visual/fase5/trascinamento-fusione.png` — motivo: binary
- `docs/visual/fase5/trascinamento-porta.png` — motivo: binary
- `docs/visual/fase6a/barra-controlli.png` — motivo: binary
- `docs/visual/fase6a/cassetta-alto.png` — motivo: binary
- `docs/visual/fase6a/cassetta-basso-finestra-bassa.png` — motivo: binary
- `docs/visual/fase6a/cassetta-basso-orizzontale.png` — motivo: binary
- `docs/visual/fase6a/cassetta-basso.png` — motivo: binary
- `docs/visual/fase6a/cassetta-destra.png` — motivo: binary
- `docs/visual/fase6a/cassetta-sinistra.png` — motivo: binary
- `docs/visual/fase6a/minimappa-con-pannello-in-basso.png` — motivo: binary
- `docs/visual/fase6a/nodi-spinti-pannello-in-alto.png` — motivo: binary
- `docs/visual/fase6a/schede-stesso-bordo.png` — motivo: binary
- `docs/visual/fase6a/svuota-conferma.png` — motivo: binary
- `docs/visual/fase6a/trascinamento-dalla-cassetta-su-cavo.png` — motivo: binary
- `docs/visual/fase6a/trascinamento-dalla-cassetta.png` — motivo: binary
- `docs/visual/fase6b1/box-combinato-passaggi-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/box-combinato-passaggi-scuro.png` — motivo: binary
- `docs/visual/fase6b1/converti-tipo-due-colonne-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/converti-tipo-due-colonne-notte-scuro.png` — motivo: binary
- `docs/visual/fase6b1/converti-tipo-due-colonne-scuro.png` — motivo: binary
- `docs/visual/fase6b1/inspector-bloccato-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/inspector-bloccato-scuro.png` — motivo: binary
- `docs/visual/fase6b1/inspector-in-basso-1280x720-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/inspector-in-basso-1280x720-scuro.png` — motivo: binary
- `docs/visual/fase6b1/inspector-in-basso-1440-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/inspector-in-basso-1440-scuro.png` — motivo: binary
- `docs/visual/fase6b1/nodo-con-pulsanti-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/nodo-con-pulsanti-scuro.png` — motivo: binary
- `docs/visual/fase6b1/pannello-espanso-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/pannello-espanso-scuro.png` — motivo: binary
- `docs/visual/fase6b1/sostituisci-valori-due-colonne-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/sostituisci-valori-due-colonne-scuro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-colonne-aperta-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-colonne-aperta-notte-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-colonne-aperta-scuro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-in-alto-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-in-alto-scuro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-valori-aperta-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-valori-aperta-scuro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-vicino-al-bordo-1280x720-chiaro.png` — motivo: binary
- `docs/visual/fase6b1/tendina-vicino-al-bordo-1280x720-scuro.png` — motivo: binary
- `docs/visual/fase6b11/cassetta-a-sinistra-scena-densa-1280x720-chiaro.png` — motivo: binary
- `docs/visual/fase6b11/cassetta-a-sinistra-scena-densa-1280x720-scuro.png` — motivo: binary
- `docs/visual/fase6b11/cassetta-a-sinistra-scena-densa-1440-chiaro.png` — motivo: binary
- `docs/visual/fase6b11/cassetta-a-sinistra-scena-densa-1440-scuro.png` — motivo: binary
- `docs/visual/fase6b11/inspector-in-basso-1280x720-dopo-chiaro.png` — motivo: binary
- `docs/visual/fase6b11/inspector-in-basso-1280x720-dopo-scuro.png` — motivo: binary
- `docs/visual/fase6b11/inspector-in-basso-1440-dopo-chiaro.png` — motivo: binary
- `docs/visual/fase6b11/inspector-in-basso-1440-dopo-scuro.png` — motivo: binary
- `docs/visual/fase6b11/striscia-apertura-inspector-in-basso.png` — motivo: binary
- `docs/visual/fase6b11/striscia-chiusura-inspector-in-basso.png` — motivo: binary
- `docs/visual/fase6b2/anteprima-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/anteprima-scuro.png` — motivo: binary
- `docs/visual/fase6b2/condizioni-filtro-gruppi-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/condizioni-filtro-gruppi-notte-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/condizioni-filtro-gruppi-scuro.png` — motivo: binary
- `docs/visual/fase6b2/connettore-aperto-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/connettore-aperto-scuro.png` — motivo: binary
- `docs/visual/fase6b2/inspector-tre-colonne-1280x720-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/inspector-tre-colonne-1280x720-scuro.png` — motivo: binary
- `docs/visual/fase6b2/inspector-tre-colonne-1440-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/inspector-tre-colonne-1440-notte-scuro.png` — motivo: binary
- `docs/visual/fase6b2/inspector-tre-colonne-1440-scuro.png` — motivo: binary
- `docs/visual/fase6b2/inspector-tre-colonne-alto-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/inspector-tre-colonne-alto-scuro.png` — motivo: binary
- `docs/visual/fase6b2/join-avviso-prestazioni-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/join-avviso-prestazioni-scuro.png` — motivo: binary
- `docs/visual/fase6b2/join-condizioni-colonna-valore-lista-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/join-condizioni-colonna-valore-lista-scuro.png` — motivo: binary
- `docs/visual/fase6b2/operazione-a-voci-tre-colonne-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/operazione-a-voci-tre-colonne-scuro.png` — motivo: binary
- `docs/visual/fase6b2/ordina-riordino-chiaro.png` — motivo: binary
- `docs/visual/fase6b2/ordina-riordino-scuro.png` — motivo: binary
- `docs/visual/temi/notte-chiaro-canvas.png` — motivo: binary
- `docs/visual/temi/notte-chiaro-soluzioni.png` — motivo: binary
- `docs/visual/temi/notte-scuro-canvas.png` — motivo: binary
- `docs/visual/temi/notte-scuro-soluzioni.png` — motivo: binary
- `docs/visual/temi/prototipo-chiaro-canvas.png` — motivo: binary
- `docs/visual/temi/prototipo-chiaro-soluzioni.png` — motivo: binary
- `docs/visual/temi/prototipo-scuro-canvas.png` — motivo: binary
- `docs/visual/temi/prototipo-scuro-soluzioni.png` — motivo: binary
- `docs/visual/temi/tinta-0-chiaro-canvas.png` — motivo: binary
- `docs/visual/temi/tinta-0-scuro-canvas.png` — motivo: binary
- `docs/visual/temi/tinta-140-chiaro-canvas.png` — motivo: binary
- `docs/visual/temi/tinta-140-scuro-canvas.png` — motivo: binary
- `docs/visual/temi/tinta-280-chiaro-canvas.png` — motivo: binary
- `docs/visual/temi/tinta-280-scuro-canvas.png` — motivo: binary
- `package-lock.json` — motivo: lockfile
- `public/favicon.ico` — motivo: binary
- `public/robots.txt` — motivo: out-of-scope-dir

## Segreti redatti

- `src/etl-canvas/inspector/inspector.css` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`
- `src/etl-canvas/inspector/logic.ts` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`
- `scripts/check-tokens.mjs` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`
- `docs/theme-debt.md` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`
- `src/theme/__tests__/support.ts` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`
- `src/theme/color.ts` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`
- `src/canvas/.reports/VALIDATION_REPORT_2026-09-29T17-13-50Z.md` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`
- `src/canvas/.reports/VALIDATION_REPORT_2026-09-29T19-51-32Z.md` — pattern: secret-like assignment — valore sostituito con `[REDATTO]`

