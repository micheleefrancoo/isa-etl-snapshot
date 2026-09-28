# STATUS.md

Generato: 2026-09-28T21:08:21Z (UTC)

## Type check

Nessuno script "typecheck" in package.json: eseguito il comando diretto.

Comando: `npx tsc --noEmit`

Esito: OK (exit 0)
Durata: 10s

Ultime 60 righe di output:
```
```

## Lint (npm run lint)

Comando: `npm run lint`

Esito: OK (exit 0)
Durata: 8s

Ultime 60 righe di output:
```

> lint
> eslint .


/workspaces/isa-glass-platform/src/canvas/components/CanvasContainer.tsx
  23:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/canvas/store/canvasStore.tsx
   31:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components
   63:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components
   85:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components
  114:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/components/isa/app-shell.tsx
  9:14  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/components/ui/badge.tsx
  32:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/components/ui/button.tsx
  49:18  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/components/ui/form.tsx
  163:3  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/components/ui/navigation-menu.tsx
  111:3  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/components/ui/sidebar.tsx
  743:3  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/components/ui/toggle.tsx
  42:18  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/lib/solutions-store.tsx
  327:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/lib/theme.tsx
  29:14  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

✖ 14 problems (0 errors, 14 warnings)

```

## Test (npm test / vitest run)

Comando: `npm test`

Esito: OK (exit 0)
Durata: 3s

Ultime 60 righe di output:
```

> test
> vitest run

The plugin "vite-tsconfig-paths" is detected. Vite now supports tsconfig paths resolution natively via the resolve.tsconfigPaths option. You can remove the plugin and set resolve.tsconfigPaths: true in your Vite config instead.

 RUN  v5.0.1 /workspaces/isa-glass-platform


 Test Files  11 passed (11)
      Tests  118 passed (118)
   Start at  21:08:40
   Duration  2.00s (transform 49%, import 28%, tests 14%, worker 9%)

    Isolate  11 workers spawned · ~111ms startup each (spawn + environment, per file)
             at least ~1.11s faster with isolate: false — reuses workers across files instead of one per file

```

## Build (npm run build)

Comando: `npm run build`

Esito: OK (exit 0)
Durata: 7s

Ultime 60 righe di output:
```
.output/server/_ssr/favorites-CyGrO8TZ.mjs                          0.57 kB │ gzip:   0.38 kB
.output/server/_ssr/trash-BZw2hxgU.mjs                              0.58 kB │ gzip:   0.38 kB
.output/server/_chunks/ssr-renderer.mjs                             0.60 kB │ gzip:   0.36 kB
.output/server/_ssr/shared-Bq5kHv-K.mjs                             0.60 kB │ gzip:   0.39 kB
.output/server/_ssr/templates-DIZhedAs.mjs                          0.61 kB │ gzip:   0.39 kB
.output/server/_ssr/solutions._solutionId.dashboard-CujVFx5K.mjs    0.86 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.model-D5GqwrKQ.mjs        0.88 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.etl-BPFLdwn6.mjs          0.90 kB │ gzip:   0.49 kB
.output/server/_libs/hookable.mjs                                   1.16 kB │ gzip:   0.51 kB
.output/server/_ssr/theme-ovWAvoLq.mjs                              1.23 kB │ gzip:   0.57 kB
.output/server/_ssr/start-RKGGYzjZ.mjs                              1.53 kB │ gzip:   0.70 kB
.output/server/_runtime.mjs                                         1.61 kB │ gzip:   0.74 kB
.output/server/_libs/react-is.mjs                                   1.67 kB │ gzip:   0.65 kB
.output/server/_libs/prop-types.mjs                                 2.27 kB │ gzip:   0.84 kB
.output/server/_ssr/createCsrfMiddleware-B2To0gPJ.mjs               3.11 kB │ gzip:   1.01 kB
.output/server/_libs/d3-path.mjs                                    3.25 kB │ gzip:   1.16 kB
.output/server/_ssr/activity-DmLzxlkY.mjs                           3.45 kB │ gzip:   0.97 kB
.output/server/_tanstack-start-manifest_v-DpCJYIEN.mjs              3.66 kB │ gzip:   0.86 kB
.output/server/_ssr/solutions._solutionId.model-DN1m8FSz.mjs        4.05 kB │ gzip:   1.32 kB
.output/server/_ssr/ssr.mjs                                         4.55 kB │ gzip:   1.92 kB
.output/server/_libs/d3-interpolate.mjs                             5.02 kB │ gzip:   1.61 kB
.output/server/_ssr/solutions._solutionId.dashboard-6paFuEf7.mjs    5.05 kB │ gzip:   1.41 kB
.output/server/_ssr/solutions._solutionId-DQgrqfr2.mjs              5.89 kB │ gzip:   1.63 kB
.output/server/_ssr/solutions-store-ceMrv_vy.mjs                    6.12 kB │ gzip:   2.06 kB
.output/server/_libs/d3-array.mjs                                   8.05 kB │ gzip:   2.25 kB
.output/server/_libs/eventemitter3.mjs                              8.31 kB │ gzip:   2.05 kB
.output/server/_ssr/isa-menu-CW0cymx5.mjs                           9.56 kB │ gzip:   2.97 kB
.output/server/_libs/h3-v2+rou3+srvx.mjs                            9.87 kB │ gzip:   3.06 kB
.output/server/_libs/d3-color.mjs                                  10.38 kB │ gzip:   3.57 kB
.output/server/_libs/d3-format.mjs                                 10.56 kB │ gzip:   3.02 kB
.output/server/_libs/tanstack__history.mjs                         12.09 kB │ gzip:   3.48 kB
.output/server/_ssr/router-BP7XxGMH.mjs                            14.19 kB │ gzip:   3.57 kB
.output/server/index.mjs                                           14.82 kB │ gzip:   4.32 kB
.output/server/_libs/fast-equals.mjs                               15.91 kB │ gzip:   4.13 kB
.output/server/_libs/h3+rou3+srvx.mjs                              17.30 kB │ gzip:   4.98 kB
.output/server/_libs/react+tanstack__react-query.mjs               18.47 kB │ gzip:   4.82 kB
.output/server/_libs/decimal.js-light.mjs                          23.31 kB │ gzip:   6.89 kB
.output/server/_ssr/app-shell-BNrG4YMF.mjs                         24.55 kB │ gzip:   5.63 kB
.output/server/_libs/d3-shape.mjs                                  24.56 kB │ gzip:   5.02 kB
.output/server/_ssr/routes-CzXoXnQ3.mjs                            24.84 kB │ gzip:   4.48 kB
.output/server/_libs/lucide-react.mjs                              37.25 kB │ gzip:   7.17 kB
.output/server/_libs/react-smooth.mjs                              37.83 kB │ gzip:   8.14 kB
.output/server/_libs/tanstack__query-core.mjs                      49.84 kB │ gzip:  11.12 kB
.output/server/_ssr/server--Ho9n_qd.mjs                            54.77 kB │ gzip:  14.38 kB
.output/server/_libs/d3-scale+[...].mjs                            58.22 kB │ gzip:  12.11 kB
.output/server/_libs/@tanstack/router-core+[...].mjs              125.35 kB │ gzip:  26.53 kB
.output/server/_libs/lodash.mjs                                   161.80 kB │ gzip:  29.45 kB
.output/server/_ssr/solutions._solutionId.etl-B9H4LDib.mjs        169.47 kB │ gzip:  42.03 kB
.output/server/_libs/recharts+[...].mjs                           515.72 kB │ gzip:  96.80 kB
.output/server/_libs/@tanstack/react-router+[...].mjs             681.94 kB │ gzip: 143.15 kB

✓ built in 1.28s
[nitro] ℹ Using auto generated worker name: micheleefrancoo-isa-glass-platform
ℹ Generated .output/server/wrangler.json
ℹ Generated .wrangler/deploy/config.json
ℹ Generated .output/public/_headers
ℹ Generated .output/nitro.json

[nitro] ✔ You can preview this build using npx vite preview
[nitro] ✔ You can deploy this build using npx nitro deploy --prebuilt
```

## Git log (ultimi 20 commit)

```
1c7a964 Fase 1: dominio ETL puro (src/etl-core/)
0098c85 sync-snapshot: stampa sempre come ultima riga il link INDEX.md fissato al commit
2f7d695 Aggiunge report di validazione: merge fix Safari e allineamento main
e6e4bfe Formattazione automatica, nessuna modifica funzionale
9b14d46 Merge remote-tracking branch 'origin/main' into wip/stato-2026-09-28
5987c8a Stato di lavoro al 2026-09-28: Fase 1, Fase 2A, snapshot, prototipo e inventario
1e72955 Risolto drag Tool Palette Safari
2ea7e94 Changes
98f8366 Add project README
3e40cd2 Fisso canvas con espansione
c21caaa Changes
3848fb4 Changes
e7c251c Changes
3b8afd2 Changes
e5bdd1a Changes
6012a8a Changes
60544d5 Corretta griglia canvas chiara
a3687c3 Changes
b3fe2f3 Changes
89ff865 Changes
```

## Branch

```
* feat/etl-core
  main
  wip/stato-2026-09-28
  remotes/origin/HEAD -> origin/main
  remotes/origin/feat/etl-core
  remotes/origin/main
  remotes/origin/wip/stato-2026-09-28
```

## Branch diversi da main

### `feat/etl-core`

Ultimo commit:
```
1c7a964 Fase 1: dominio ETL puro (src/etl-core/)
```

Diff stat rispetto a main:
```
 scripts/generate-snapshot.mjs                      |   1 +
 .../VALIDATION_REPORT_2026-09-28T21-06-48Z.md      | 100 +++
 src/etl-core/NOTE_DIVERGENZE.md                    |  59 ++
 src/etl-core/README.md                             | 161 +++++
 src/etl-core/__tests__/csv.test.ts                 |  65 ++
 src/etl-core/__tests__/expressions.test.ts         | 101 +++
 src/etl-core/__tests__/helpers.ts                  |  52 ++
 src/etl-core/__tests__/mutations.test.ts           | 221 ++++++
 src/etl-core/__tests__/params.test.ts              | 123 ++++
 src/etl-core/__tests__/relations.test.ts           | 163 +++++
 src/etl-core/__tests__/schema.test.ts              |  73 ++
 src/etl-core/__tests__/state.test.ts               | 105 +++
 src/etl-core/catalog/icons.ts                      |  42 ++
 src/etl-core/catalog/operations.ts                 |  77 +++
 src/etl-core/catalog/params.ts                     | 768 +++++++++++++++++++++
 src/etl-core/data/csv.ts                           |  81 +++
 src/etl-core/index.ts                              |  98 +++
 src/etl-core/logic/expressions.ts                  | 171 +++++
 src/etl-core/model/graph.ts                        |  97 +++
 src/etl-core/model/types.ts                        | 227 ++++++
 src/etl-core/rules/mutations.ts                    | 473 +++++++++++++
 src/etl-core/rules/relations.ts                    | 115 +++
 src/etl-core/rules/state.ts                        |  95 +++
 src/etl-core/schema/schema.ts                      |  36 +
 24 files changed, 3504 insertions(+)
```

### `wip/stato-2026-09-28`

Ultimo commit:
```
9b14d46 Merge remote-tracking branch 'origin/main' into wip/stato-2026-09-28
```

Diff stat rispetto a main:
```
```

