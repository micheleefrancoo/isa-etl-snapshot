# INDEX.md

Generato: 2026-09-28T19:33:55Z (UTC)
Repository sorgente: isa-glass-platform, branch `main`, commit `98f83661ac04a55c78465ba5ebf79974f290eb66`
Working tree del repository sorgente: modifiche non committate presenti (23 file):

- `package.json`
- `src/components/isa/etl/data-preview.tsx`
- `src/components/isa/etl/inspector.tsx`
- `src/components/isa/etl/tool-palette.tsx`
- `src/components/isa/etl/workflow-canvas.tsx`
- `src/components/isa/ui/isa-menu.tsx`
- `src/lib/etl-display.ts`
- `src/lib/etl-workflow.tsx`
- `src/routes/solutions.$solutionId.etl.tsx`
- `src/styles.css`
- `.devcontainer/`
- `.pw-tmp/`
- `docs/`
- `package-lock.json`
- `scripts/`
- `src/canvas/`
- `src/components/isa/etl/isa-context-menu.tsx`
- `src/components/isa/etl/settings-panels/`
- `src/lib/etl-bubble.ts`
- `src/lib/etl-motion.ts`
- `src/lib/etl-node-config.ts`
- `src/lib/etl-node-size.ts`
- `vitest.config.ts`

Questo indice è fissato al commit `0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb` del repository snapshot (isa-etl-snapshot): tutti gli URL sotto puntano a quel commit e restano validi anche dopo aggiornamenti futuri.

## Da leggere per primi

1. [STATUS.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/STATUS.md) — stato di type check, lint, test, build
2. [ENV.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/ENV.md) — configurazione completa (package.json, tsconfig, vite, eslint, CSS)
3. [TREE.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/TREE.md) — albero completo del repository
4. I blocchi in `files/`, in ordine, elencati sotto.

## Blocchi (files/)

### [files/01-canvas.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/01-canvas.md)

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

### [files/02-isa-etl-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/02-isa-etl-a.md)

55.5 KB. File sorgente contenuti:

- `src/components/isa/etl/data-preview.tsx`
- `src/components/isa/etl/inspector.tsx`
- `src/components/isa/etl/isa-context-menu.tsx`
- `src/components/isa/etl/settings-panels/aggregate-panel.tsx`
- `src/components/isa/etl/settings-panels/combine-panel.tsx`
- `src/components/isa/etl/settings-panels/filter-panel.tsx`
- `src/components/isa/etl/settings-panels/panel-controls.tsx`
- `src/components/isa/etl/tool-palette.tsx`

### [files/02-isa-etl-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/02-isa-etl-b.md)

49.9 KB. File sorgente contenuti:

- `src/components/isa/etl/workflow-canvas.tsx`

### [files/02-isa-etl-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/02-isa-etl-c.md)

49.9 KB. File sorgente contenuti:

- `src/components/isa/etl/workflow-canvas.tsx`

### [files/02-isa-etl-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/02-isa-etl-d.md)

27.0 KB. File sorgente contenuti:

- `src/components/isa/etl/workflow-canvas.tsx`

### [files/03-components-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/03-components-a.md)

59.0 KB. File sorgente contenuti:

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

### [files/03-components-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/03-components-b.md)

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

### [files/03-components-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/03-components-c.md)

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

### [files/03-components-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/03-components-d.md)

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

### [files/04-lib-hooks-store-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/04-lib-hooks-store-a.md)

49.2 KB. File sorgente contenuti:

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

### [files/04-lib-hooks-store-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/04-lib-hooks-store-b.md)

28.1 KB. File sorgente contenuti:

- `src/lib/etl-workflow.tsx`
- `src/lib/lovable-error-reporting.ts`
- `src/lib/modules.ts`
- `src/lib/solutions-store.tsx`
- `src/lib/theme.tsx`
- `src/lib/utils.ts`

### [files/05-app-pages.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/05-app-pages.md)

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

### [files/06-styles.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/06-styles.md)

11.7 KB. File sorgente contenuti:

- `src/styles.css`

### [files/08-scripts-config.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/08-scripts-config.md)

37.7 KB. File sorgente contenuti:

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
- `scripts/generate-index.mjs`
- `scripts/generate-snapshot.mjs`
- `scripts/sync-snapshot.sh`
- `tsconfig.json`
- `vite.config.ts`
- `vitest.config.ts`

### [files/09-docs.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/09-docs.md)

6.3 KB. File sorgente contenuti:

- `.lovable/plan/barra-risorse-etl-ancorata-al-canvas-2026-09-10.md`
- `AGENTS.md`
- `README.md`
- `roadmap.md`

### [files/10-prototype-a.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/10-prototype-a.md)

49.8 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-b.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/10-prototype-b.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-c.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/10-prototype-c.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-d.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/10-prototype-d.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-e.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/10-prototype-e.md)

49.9 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/10-prototype-f.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/10-prototype-f.md)

11.1 KB. File sorgente contenuti:

- `docs/prototype/isa-fusion-prototype.html`

### [files/11-inventory.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/files/11-inventory.md)

42.7 KB. File sorgente contenuti:

- `docs/inventory/INVENTARIO_2026-09-28T19-29-07Z.md`

## Report (reports/)

- [VALIDATION_REPORT_2026-09-19T10-24-53Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/reports/VALIDATION_REPORT_2026-09-19T10-24-53Z.md)
- [VALIDATION_REPORT_2026-09-19T11-38-16Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/reports/VALIDATION_REPORT_2026-09-19T11-38-16Z.md)
- [VALIDATION_REPORT_2026-09-28T18-45-29Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/reports/VALIDATION_REPORT_2026-09-28T18-45-29Z.md)
- [VALIDATION_REPORT_2026-09-28T19-33-26Z.md](https://raw.githubusercontent.com/micheleefrancoo/isa-etl-snapshot/0831a14ea30d6e76aeac35f6d5a7d23fe9b8a9eb/reports/VALIDATION_REPORT_2026-09-28T19-33-26Z.md)

## File esclusi dallo snapshot

(elencati per riferimento in TREE.md, contenuto non incluso in files/)

- `.claude/scheduled_tasks.lock` — motivo: out-of-scope-dir
- `bun.lock` — motivo: lockfile
- `package-lock.json` — motivo: lockfile
- `public/favicon.ico` — motivo: binary
- `public/robots.txt` — motivo: out-of-scope-dir

