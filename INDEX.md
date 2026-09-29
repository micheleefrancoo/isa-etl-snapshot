# INDEX.md

Generato: 2026-09-29T08:58:40Z (UTC)
Repository sorgente: isa-glass-platform, branch `main`, commit `4ac5954f04aa1431de4ddb5228136d0efa1a2fb7`
Working tree del repository sorgente: pulito (nessuna modifica non committata).

Questo indice è fissato al commit `6add0301efb94efc3d8c3b8e9937c0090ccf8f1c` del repository snapshot (isa-etl-snapshot): tutti gli URL sotto puntano a quel commit e restano validi anche dopo aggiornamenti futuri.

## Da leggere per primi

1. [STATUS.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/STATUS.md) — stato di type check, lint, test, build
2. [ENV.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/ENV.md) — configurazione completa (package.json, tsconfig, vite, eslint, CSS)
3. [TREE.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/TREE.md) — albero completo del repository
4. I blocchi in `files/`, in ordine, elencati sotto.

## Blocchi (files/)

### [files/01-canvas.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01-canvas.md)

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

### [files/01b-etl-core-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01b-etl-core-a.md)

56.8 KB. File sorgente contenuti:

- `src/etl-core/NOTE_DIVERGENZE.md`
- `src/etl-core/README.md`
- `src/etl-core/__tests__/csv.test.ts`
- `src/etl-core/__tests__/expressions.test.ts`
- `src/etl-core/__tests__/fase11-requisiti.test.ts`
- `src/etl-core/__tests__/fase11.test.ts`
- `src/etl-core/__tests__/helpers.ts`
- `src/etl-core/__tests__/mutations.test.ts`

### [files/01b-etl-core-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01b-etl-core-b.md)

54.2 KB. File sorgente contenuti:

- `src/etl-core/__tests__/params.test.ts`
- `src/etl-core/__tests__/relations.test.ts`
- `src/etl-core/__tests__/schema.test.ts`
- `src/etl-core/__tests__/state.test.ts`
- `src/etl-core/catalog/icons.ts`
- `src/etl-core/catalog/operations.ts`
- `src/etl-core/catalog/params.ts`
- `src/etl-core/data/csv.ts`
- `src/etl-core/index.ts`

### [files/01b-etl-core-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01b-etl-core-c.md)

43.8 KB. File sorgente contenuti:

- `src/etl-core/logic/expressions.ts`
- `src/etl-core/model/graph.ts`
- `src/etl-core/model/types.ts`
- `src/etl-core/rules/mutations.ts`
- `src/etl-core/rules/relations.ts`
- `src/etl-core/rules/state.ts`
- `src/etl-core/schema/schema.ts`

### [files/01c-etl-layout-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01c-etl-layout-a.md)

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

### [files/01c-etl-layout-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01c-etl-layout-b.md)

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

### [files/01c-etl-layout-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01c-etl-layout-c.md)

53.9 KB. File sorgente contenuti:

- `src/etl-layout/__tests__/unit.test.ts`
- `src/etl-layout/autoLayout.ts`
- `src/etl-layout/constants.ts`
- `src/etl-layout/free.ts`
- `src/etl-layout/hitTest.ts`
- `src/etl-layout/index.ts`
- `src/etl-layout/links.ts`
- `src/etl-layout/nodes.ts`
- `src/etl-layout/path.ts`

### [files/01c-etl-layout-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/01c-etl-layout-d.md)

30.5 KB. File sorgente contenuti:

- `src/etl-layout/placement.ts`
- `src/etl-layout/routing.ts`
- `src/etl-layout/slots.ts`
- `src/etl-layout/types.ts`

### [files/02-isa-etl-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/02-isa-etl-a.md)

52.7 KB. File sorgente contenuti:

- `src/components/isa/etl/data-preview.tsx`
- `src/components/isa/etl/inspector.tsx`
- `src/components/isa/etl/isa-context-menu.tsx`
- `src/components/isa/etl/settings-panels/aggregate-panel.tsx`
- `src/components/isa/etl/settings-panels/combine-panel.tsx`
- `src/components/isa/etl/settings-panels/filter-panel.tsx`
- `src/components/isa/etl/settings-panels/panel-controls.tsx`
- `src/components/isa/etl/tool-palette.tsx`

### [files/02-isa-etl-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/02-isa-etl-b.md)

49.9 KB. File sorgente contenuti:

- `src/components/isa/etl/workflow-canvas.tsx`

### [files/02-isa-etl-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/02-isa-etl-c.md)

54.2 KB. File sorgente contenuti:

- `src/components/isa/etl/workflow-canvas.tsx`

### [files/03-components-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/03-components-a.md)

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

### [files/03-components-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/03-components-b.md)

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

### [files/03-components-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/03-components-c.md)

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

### [files/03-components-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/03-components-d.md)

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

### [files/04-lib-hooks-store-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/04-lib-hooks-store-a.md)

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

### [files/04-lib-hooks-store-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/04-lib-hooks-store-b.md)

28.1 KB. File sorgente contenuti:

- `src/lib/etl-workflow.tsx`
- `src/lib/lovable-error-reporting.ts`
- `src/lib/modules.ts`
- `src/lib/solutions-store.tsx`
- `src/lib/theme.tsx`
- `src/lib/utils.ts`

### [files/05-app-pages.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/05-app-pages.md)

57.2 KB. File sorgente contenuti:

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
- `src/start.ts`

### [files/06-styles.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/06-styles.md)

11.7 KB. File sorgente contenuti:

- `src/styles.css`

### [files/08-scripts-config.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/08-scripts-config.md)

51.8 KB. File sorgente contenuti:

- `.claude/settings.local.json`
- `.devcontainer/devcontainer.json`
- `.gitignore`
- `.lovable/project.json`
- `.prettierignore`
- `.prettierrc`
- `.vscode/settings.json`
- `bunfig.toml`
- `components.json`
- `eslint.config.js`
- `package.json`
- `scripts/extract-golden.mjs`
- `scripts/generate-index.mjs`
- `scripts/generate-snapshot.mjs`
- `scripts/sync-snapshot.sh`
- `tsconfig.json`
- `vite.config.ts`
- `vitest.config.ts`

### [files/09-docs.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/09-docs.md)

6.3 KB. File sorgente contenuti:

- `.lovable/plan/barra-risorse-etl-ancorata-al-canvas-2026-09-10.md`
- `AGENTS.md`
- `README.md`
- `roadmap.md`

### [files/10-prototype-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/10-prototype-a.md)

49.8 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/10-prototype-b.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/10-prototype-c.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/10-prototype-d.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-e.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/10-prototype-e.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-f.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/10-prototype-f.md)

11.1 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/11-inventory.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/files/11-inventory.md)

52.7 KB. File sorgente contenuti:

- `docs/inventory/INVENTARIO_2026-09-28T19-40-33Z.md`
- `docs/inventory/lovable-1e72955.diff`

## Report (reports/)

- [VALIDATION_REPORT_2026-09-19T10-24-53Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-19T10-24-53Z.md)
- [VALIDATION_REPORT_2026-09-19T11-38-16Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-19T11-38-16Z.md)
- [VALIDATION_REPORT_2026-09-28T18-45-29Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-28T18-45-29Z.md)
- [VALIDATION_REPORT_2026-09-28T19-44-43Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-28T19-44-43Z.md)
- [VALIDATION_REPORT_2026-09-28T20-14-48Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-28T20-14-48Z.md)
- [VALIDATION_REPORT_2026-09-28T21-06-48Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-28T21-06-48Z.md)
- [VALIDATION_REPORT_2026-09-29T07-20-03Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-29T07-20-03Z.md)
- [VALIDATION_REPORT_2026-09-29T08-33-03Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-29T08-33-03Z.md)
- [VALIDATION_REPORT_2026-09-29T08-57-43Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/6add0301efb94efc3d8c3b8e9937c0090ccf8f1c/reports/VALIDATION_REPORT_2026-09-29T08-57-43Z.md)

## File esclusi dallo snapshot

(elencati per riferimento in TREE.md, contenuto non incluso in files/)

- `.claude/scheduled_tasks.lock` — motivo: out-of-scope-dir
- `bun.lock` — motivo: lockfile
- `package-lock.json` — motivo: lockfile
- `public/favicon.ico` — motivo: binary
- `public/robots.txt` — motivo: out-of-scope-dir

