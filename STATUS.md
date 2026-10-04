# STATUS.md

Generato: 2026-10-04T07:51:59Z (UTC)

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
  54:14  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

✖ 14 problems (0 errors, 14 warnings)

```

## Test (npm test / vitest run)

Comando: `npm test`

Esito: OK (exit 0)
Durata: 38s

Ultime 60 righe di output:
```

> test
> vitest run

The plugin "vite-tsconfig-paths" is detected. Vite now supports tsconfig paths resolution natively via the resolve.tsconfigPaths option. You can remove the plugin and set resolve.tsconfigPaths: true in your Vite config instead.

 RUN  v5.0.1 /workspaces/isa-glass-platform


 Test Files  45 passed (45)
      Tests  814 passed (814)
   Start at  07:52:21
   Duration  37.11s (tests 89%, import 6%, transform 4%, worker 1%)

    Isolate  45 workers spawned · ~101ms startup each (spawn + environment, per file)
             at least ~4.46s faster with isolate: false — reuses workers across files instead of one per file

```

## Build (npm run build)

Comando: `npm run build`

Esito: OK (exit 0)
Durata: 6s

Ultime 60 righe di output:
```
.output/server/_ssr/favorites-C8Z5TnLg.mjs                          0.57 kB │ gzip:   0.38 kB
.output/server/_ssr/trash-Cy2PattI.mjs                              0.58 kB │ gzip:   0.37 kB
.output/server/_chunks/ssr-renderer.mjs                             0.60 kB │ gzip:   0.36 kB
.output/server/_ssr/shared-Bbcfg4ps.mjs                             0.60 kB │ gzip:   0.39 kB
.output/server/_ssr/templates-BxgvM8qO.mjs                          0.61 kB │ gzip:   0.39 kB
.output/server/_ssr/solutions._solutionId.dashboard-CujVFx5K.mjs    0.86 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.model-D5GqwrKQ.mjs        0.88 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.etl-DADefv2-.mjs          1.09 kB │ gzip:   0.58 kB
.output/server/_libs/hookable.mjs                                   1.16 kB │ gzip:   0.51 kB
.output/server/_ssr/start-RKGGYzjZ.mjs                              1.53 kB │ gzip:   0.70 kB
.output/server/_runtime.mjs                                         1.61 kB │ gzip:   0.74 kB
.output/server/_libs/react-is.mjs                                   1.67 kB │ gzip:   0.65 kB
.output/server/_libs/prop-types.mjs                                 2.27 kB │ gzip:   0.84 kB
.output/server/_ssr/createCsrfMiddleware-B2To0gPJ.mjs               3.11 kB │ gzip:   1.01 kB
.output/server/_libs/d3-path.mjs                                    3.25 kB │ gzip:   1.16 kB
.output/server/_ssr/activity-C06WSQVD.mjs                           3.45 kB │ gzip:   0.97 kB
.output/server/_tanstack-start-manifest_v-CSemw1G6.mjs              3.71 kB │ gzip:   0.87 kB
.output/server/_ssr/solutions._solutionId.model-DN1m8FSz.mjs        4.05 kB │ gzip:   1.32 kB
.output/server/_ssr/ssr.mjs                                         4.55 kB │ gzip:   1.92 kB
.output/server/_libs/d3-interpolate.mjs                             5.02 kB │ gzip:   1.61 kB
.output/server/_ssr/solutions._solutionId.dashboard-6paFuEf7.mjs    5.05 kB │ gzip:   1.41 kB
.output/server/_ssr/solutions._solutionId-DCSdbmJH.mjs              5.89 kB │ gzip:   1.63 kB
.output/server/_ssr/solutions-store-ceMrv_vy.mjs                    6.12 kB │ gzip:   2.06 kB
.output/server/_libs/d3-array.mjs                                   8.05 kB │ gzip:   2.25 kB
.output/server/_libs/eventemitter3.mjs                              8.31 kB │ gzip:   2.05 kB
.output/server/_ssr/isa-menu-CW0cymx5.mjs                           9.56 kB │ gzip:   2.97 kB
.output/server/_libs/d3-color.mjs                                  10.38 kB │ gzip:   3.57 kB
.output/server/_libs/d3-format.mjs                                 10.56 kB │ gzip:   3.02 kB
.output/server/_libs/tanstack__history.mjs                         12.09 kB │ gzip:   3.48 kB
.output/server/_ssr/router-rHxp9gR6.mjs                            14.47 kB │ gzip:   3.89 kB
.output/server/_ssr/theme-Dujx4GV7.mjs                             15.18 kB │ gzip:   6.06 kB
.output/server/_libs/fast-equals.mjs                               15.91 kB │ gzip:   4.13 kB
.output/server/_libs/h3-v2+rou3+srvx+unenv.mjs                     15.92 kB │ gzip:   4.39 kB
.output/server/index.mjs                                           16.41 kB │ gzip:   4.69 kB
.output/server/_libs/h3+rou3+srvx.mjs                              17.30 kB │ gzip:   4.98 kB
.output/server/_libs/react+tanstack__react-query.mjs               18.47 kB │ gzip:   4.82 kB
.output/server/_libs/decimal.js-light.mjs                          23.31 kB │ gzip:   6.89 kB
.output/server/_ssr/app-shell-DCrQJ3mI.mjs                         24.55 kB │ gzip:   5.63 kB
.output/server/_libs/d3-shape.mjs                                  24.56 kB │ gzip:   5.02 kB
.output/server/_ssr/routes-BP81naRf.mjs                            24.84 kB │ gzip:   4.48 kB
.output/server/_libs/lucide-react.mjs                              37.25 kB │ gzip:   7.17 kB
.output/server/_libs/react-smooth.mjs                              37.83 kB │ gzip:   8.14 kB
.output/server/_libs/tanstack__query-core.mjs                      49.84 kB │ gzip:  11.12 kB
.output/server/_ssr/server-BniDSYZs.mjs                            54.77 kB │ gzip:  14.39 kB
.output/server/_libs/d3-scale+[...].mjs                            58.22 kB │ gzip:  12.11 kB
.output/server/_libs/@tanstack/router-core+[...].mjs              125.35 kB │ gzip:  26.53 kB
.output/server/_libs/lodash.mjs                                   161.80 kB │ gzip:  29.45 kB
.output/server/_ssr/solutions._solutionId.etl-j3fQU8Hr.mjs        386.55 kB │ gzip: 103.61 kB
.output/server/_libs/recharts+[...].mjs                           515.72 kB │ gzip:  96.80 kB
.output/server/_libs/@tanstack/react-router+[...].mjs             681.94 kB │ gzip: 143.15 kB

✓ built in 978ms
[nitro] ℹ Using auto generated worker name: micheleefrancoo-isa-glass-platform
ℹ Generated .output/server/wrangler.json
ℹ Generated .wrangler/deploy/config.json
ℹ Generated .output/public/_headers
ℹ Generated .output/nitro.json

[nitro] ✔ You can preview this build using npx vite preview
[nitro] ✔ You can deploy this build using npx nitro deploy --prebuilt
```

## Disciplina dei token

Colori, raggi e ombre letterali nei file controllati (scripts/check-tokens.mjs).

Comando: `node scripts/check-tokens.mjs`

Esito: OK (exit 0)
Durata: 1s

Ultime 60 righe di output:
```
check-tokens: 0 violazioni nei file controllati; debito preesistente: 26 (6 ombra, 7 raggio, 13 colore).
```

## Git log (ultimi 20 commit)

```
37cd559 Fase 6a: pannelli agganciabili e cassetta degli strumenti
a2dd012 Clic su un cavo per eliminarlo (deleteLink), come nel prototipo
199cdfe Soglia di trascinamento: DRAG_THRESHOLD_PX = 5 in etl-layout (prototipo, riga 1982)
a4788a3 Fase 5: interazioni (trascinamento, fusione, collegamento, porte, selezione, tastiera)
1ac7de4 Fase T: deroghe di contrasto allineate ai valori misurati (4,3153 e 2,1476), bidirezionali
07a0a0e Fase T: sistema di temi (token a tre livelli, tema notte, accento derivato, contrasti garantiti)
1e3f6a1 Fase T: schermate di riferimento del tema predefinito (prima del sistema di temi)
313a602 Fondazione: vite.config.ts esplicito (cloudflare-module) e primitive --isa-* condivise
48de751 Fondazione (parziale): Manrope unico, derivazioni a valore identico, pulizia Lovable
41a04c4 Fase 4b: animazioni del canvas (flusso, attesa, transizioni dei cavi)
9663abe Il nuovo canvas diventa quello predefinito della pagina ETL
81831f0 Test della rotta: distingue il canvas nuovo dal pulsante che lo apre
ea2d57c Collegamenti tra canvas vecchio e nuovo nella pagina ETL
8b7cb52 Fase 4a: il canvas visibile (src/etl-canvas/)
5ef51d1 Fase 3.1: raggruppamento della cronologia, registro senza vista, inspector
c5696b6 Fase 3: stato dell'applicazione, cronologia e registro (src/etl-store/)
951207c Report Fase 2.1: validazione ripetuta dopo npm ci pulito
4ac5954 Report Fase 2.1: formattazione
dca3b87 Fase 2.1: convergenza dei cavi e correzioni puntuali alla geometria
fad990a Fase 2: geometria del canvas in TypeScript puro (src/etl-layout/)
```

## Branch

```
  feat/design-tokens-unify
  feat/etl-canvas
  feat/etl-canvas-motion
  feat/etl-core
  feat/etl-layout
  feat/etl-layout-fix
  feat/etl-store
  feat/etl-store-fix
  feat/interactions
  feat/link-click-delete
  feat/panels
  feat/theme-system
  fix/drag-threshold
* main
  wip/stato-2026-09-28
  remotes/origin/HEAD -> origin/main
  remotes/origin/feat/design-tokens-unify
  remotes/origin/feat/etl-canvas
  remotes/origin/feat/etl-canvas-motion
  remotes/origin/feat/etl-core
  remotes/origin/feat/etl-layout
  remotes/origin/feat/etl-layout-fix
  remotes/origin/feat/etl-store
  remotes/origin/feat/etl-store-fix
  remotes/origin/feat/interactions
  remotes/origin/feat/link-click-delete
  remotes/origin/feat/panels
  remotes/origin/feat/theme-system
  remotes/origin/fix/drag-threshold
  remotes/origin/main
  remotes/origin/wip/stato-2026-09-28
```

## Branch diversi da main

### `feat/design-tokens-unify`

Ultimo commit:
```
313a602 Fondazione: vite.config.ts esplicito (cloudflare-module) e primitive --isa-* condivise
```

Diff stat rispetto a main:
```
```

### `feat/etl-canvas`

Ultimo commit:
```
8b7cb52 Fase 4a: il canvas visibile (src/etl-canvas/)
```

Diff stat rispetto a main:
```
```

### `feat/etl-canvas-motion`

Ultimo commit:
```
41a04c4 Fase 4b: animazioni del canvas (flusso, attesa, transizioni dei cavi)
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

### `feat/interactions`

Ultimo commit:
```
a4788a3 Fase 5: interazioni (trascinamento, fusione, collegamento, porte, selezione, tastiera)
```

Diff stat rispetto a main:
```
```

### `feat/link-click-delete`

Ultimo commit:
```
a2dd012 Clic su un cavo per eliminarlo (deleteLink), come nel prototipo
```

Diff stat rispetto a main:
```
```

### `feat/panels`

Ultimo commit:
```
37cd559 Fase 6a: pannelli agganciabili e cassetta degli strumenti
```

Diff stat rispetto a main:
```
```

### `feat/theme-system`

Ultimo commit:
```
07a0a0e Fase T: sistema di temi (token a tre livelli, tema notte, accento derivato, contrasti garantiti)
```

Diff stat rispetto a main:
```
```

### `fix/drag-threshold`

Ultimo commit:
```
199cdfe Soglia di trascinamento: DRAG_THRESHOLD_PX = 5 in etl-layout (prototipo, riga 1982)
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

