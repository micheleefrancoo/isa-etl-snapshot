# STATUS.md

Generato: 2026-09-29T15:34:45Z (UTC)

## Type check

Nessuno script "typecheck" in package.json: eseguito il comando diretto.

Comando: `npx tsc --noEmit`

Esito: OK (exit 0)
Durata: 12s

Ultime 60 righe di output:
```
```

## Lint (npm run lint)

Comando: `npm run lint`

Esito: OK (exit 0)
Durata: 11s

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
Durata: 36s

Ultime 60 righe di output:
```

> test
> vitest run

The plugin "vite-tsconfig-paths" is detected. Vite now supports tsconfig paths resolution natively via the resolve.tsconfigPaths option. You can remove the plugin and set resolve.tsconfigPaths: true in your Vite config instead.

 RUN  v5.0.1 /workspaces/isa-glass-platform


 Test Files  25 passed (25)
      Tests  335 passed (335)
   Start at  15:35:08
   Duration  36.15s (tests 92%, import 4%, transform 4%)

    Isolate  25 workers spawned · ~119ms startup each (spawn + environment, per file)
             at least ~2.86s faster with isolate: false — reuses workers across files instead of one per file

```

## Build (npm run build)

Comando: `npm run build`

Esito: OK (exit 0)
Durata: 8s

Ultime 60 righe di output:
```
.output/server/_ssr/favorites-CyGrO8TZ.mjs                          0.57 kB │ gzip:   0.38 kB
.output/server/_ssr/trash-BZw2hxgU.mjs                              0.58 kB │ gzip:   0.38 kB
.output/server/_chunks/ssr-renderer.mjs                             0.60 kB │ gzip:   0.36 kB
.output/server/_ssr/shared-Bq5kHv-K.mjs                             0.60 kB │ gzip:   0.39 kB
.output/server/_ssr/templates-DIZhedAs.mjs                          0.61 kB │ gzip:   0.39 kB
.output/server/_ssr/solutions._solutionId.dashboard-CujVFx5K.mjs    0.86 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.model-D5GqwrKQ.mjs        0.88 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.etl-xFv0rMtU.mjs          1.09 kB │ gzip:   0.57 kB
.output/server/_libs/hookable.mjs                                   1.16 kB │ gzip:   0.51 kB
.output/server/_ssr/theme-ovWAvoLq.mjs                              1.23 kB │ gzip:   0.57 kB
.output/server/_ssr/start-RKGGYzjZ.mjs                              1.53 kB │ gzip:   0.70 kB
.output/server/_runtime.mjs                                         1.61 kB │ gzip:   0.74 kB
.output/server/_libs/react-is.mjs                                   1.67 kB │ gzip:   0.65 kB
.output/server/_libs/prop-types.mjs                                 2.27 kB │ gzip:   0.84 kB
.output/server/_ssr/createCsrfMiddleware-B2To0gPJ.mjs               3.11 kB │ gzip:   1.01 kB
.output/server/_libs/d3-path.mjs                                    3.25 kB │ gzip:   1.16 kB
.output/server/_ssr/activity-DmLzxlkY.mjs                           3.45 kB │ gzip:   0.97 kB
.output/server/_tanstack-start-manifest_v-DNArJWGJ.mjs              3.71 kB │ gzip:   0.87 kB
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
.output/server/_ssr/router-DoBIgYaI.mjs                            14.19 kB │ gzip:   3.57 kB
.output/server/_libs/fast-equals.mjs                               15.91 kB │ gzip:   4.13 kB
.output/server/index.mjs                                           16.40 kB │ gzip:   4.68 kB
.output/server/_libs/h3+rou3+srvx.mjs                              17.30 kB │ gzip:   4.98 kB
.output/server/_libs/react+tanstack__react-query.mjs               18.47 kB │ gzip:   4.82 kB
.output/server/_libs/decimal.js-light.mjs                          23.31 kB │ gzip:   6.89 kB
.output/server/_ssr/app-shell-BNrG4YMF.mjs                         24.55 kB │ gzip:   5.63 kB
.output/server/_libs/d3-shape.mjs                                  24.56 kB │ gzip:   5.02 kB
.output/server/_ssr/routes-CzXoXnQ3.mjs                            24.84 kB │ gzip:   4.48 kB
.output/server/_libs/lucide-react.mjs                              37.25 kB │ gzip:   7.17 kB
.output/server/_libs/react-smooth.mjs                              37.83 kB │ gzip:   8.14 kB
.output/server/_libs/tanstack__query-core.mjs                      49.84 kB │ gzip:  11.12 kB
.output/server/_ssr/server-SRA2yJLg.mjs                            54.77 kB │ gzip:  14.38 kB
.output/server/_libs/d3-scale+[...].mjs                            58.22 kB │ gzip:  12.11 kB
.output/server/_libs/@tanstack/router-core+[...].mjs              125.35 kB │ gzip:  26.53 kB
.output/server/_libs/lodash.mjs                                   161.80 kB │ gzip:  29.45 kB
.output/server/_ssr/solutions._solutionId.etl-uETzYmTM.mjs        315.82 kB │ gzip:  83.72 kB
.output/server/_libs/recharts+[...].mjs                           515.72 kB │ gzip:  96.80 kB
.output/server/_libs/@tanstack/react-router+[...].mjs             681.94 kB │ gzip: 143.15 kB

✓ built in 1.20s
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
8b7cb52 Fase 4a: il canvas visibile (src/etl-canvas/)
5ef51d1 Fase 3.1: raggruppamento della cronologia, registro senza vista, inspector
c5696b6 Fase 3: stato dell'applicazione, cronologia e registro (src/etl-store/)
951207c Report Fase 2.1: validazione ripetuta dopo npm ci pulito
4ac5954 Report Fase 2.1: formattazione
dca3b87 Fase 2.1: convergenza dei cavi e correzioni puntuali alla geometria
fad990a Fase 2: geometria del canvas in TypeScript puro (src/etl-layout/)
b215503 etl-core: normalizeValuesField divide solo sul separatore registrato; esporta splitTokens
46e0718 Fase 1.1: invariante output su tutte le mutazioni, separatori, test dei requisiti
a70acf7 Fase 1.1: correzioni intenzionali al dominio ETL
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
```

## Branch

```
  feat/etl-canvas
  feat/etl-core
  feat/etl-layout
  feat/etl-layout-fix
  feat/etl-store
  feat/etl-store-fix
* main
  wip/stato-2026-09-28
  remotes/origin/HEAD -> origin/main
  remotes/origin/feat/etl-canvas
  remotes/origin/feat/etl-core
  remotes/origin/feat/etl-layout
  remotes/origin/feat/etl-layout-fix
  remotes/origin/feat/etl-store
  remotes/origin/feat/etl-store-fix
  remotes/origin/main
  remotes/origin/wip/stato-2026-09-28
```

## Branch diversi da main

### `feat/etl-canvas`

Ultimo commit:
```
8b7cb52 Fase 4a: il canvas visibile (src/etl-canvas/)
```

Diff stat rispetto a main:
```
```

### `feat/etl-core`

Ultimo commit:
```
46e0718 Fase 1.1: invariante output su tutte le mutazioni, separatori, test dei requisiti
```

Diff stat rispetto a main:
```
```

### `feat/etl-layout`

Ultimo commit:
```
fad990a Fase 2: geometria del canvas in TypeScript puro (src/etl-layout/)
```

Diff stat rispetto a main:
```
```

### `feat/etl-layout-fix`

Ultimo commit:
```
951207c Report Fase 2.1: validazione ripetuta dopo npm ci pulito
```

Diff stat rispetto a main:
```
```

### `feat/etl-store`

Ultimo commit:
```
c5696b6 Fase 3: stato dell'applicazione, cronologia e registro (src/etl-store/)
```

Diff stat rispetto a main:
```
```

### `feat/etl-store-fix`

Ultimo commit:
```
5ef51d1 Fase 3.1: raggruppamento della cronologia, registro senza vista, inspector
```

Diff stat rispetto a main:
```
```

### `wip/stato-2026-09-28`

Ultimo commit:
```
9b14d46 Merge remote-tracking branch 'origin/main' into wip/stato-2026-09-28
```

Diff stat rispetto a main:
```
```

