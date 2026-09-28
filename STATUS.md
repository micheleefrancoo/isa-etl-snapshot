# STATUS.md

Generato: 2026-09-28T18:46:51Z (UTC)

## Type check

Nessuno script "typecheck" in package.json: eseguito il comando diretto.

Comando: `npx tsc --noEmit`

Esito: OK (exit 0)
Durata: 9s

Ultime 60 righe di output:
```
```

## Lint (npm run lint)

Comando: `npm run lint`

Esito: FALLITO (exit 1)
Durata: 12s

Ultime 60 righe di output:
```
  280:1   error  Delete `⏎`                                                                                                                                                                                                                    prettier/prettier
  363:17  error  Replace `·e.fromNode·===·fromNode·&&·e.fromPort·===·fromPort·&&·e.toNode·===·toNode·&&` with `⏎············e.fromNode·===·fromNode·&&⏎············e.fromPort·===·fromPort·&&⏎············e.toNode·===·toNode·&&⏎···········`  prettier/prettier

/workspaces/isa-glass-platform/src/lib/modules.ts
  9:6  error  Replace `·"/solutions/$solutionId/etl"·|·"/solutions/$solutionId/model"` with `⏎····|·"/solutions/$solutionId/etl"⏎····|·"/solutions/$solutionId/model"⏎···`  prettier/prettier

/workspaces/isa-glass-platform/src/lib/solutions-store.tsx
   40:1   error    Delete `⏎`                                                                                                                                                prettier/prettier
  102:71  error    Delete `⏎`                                                                                                                                                prettier/prettier
  239:43  error    Replace `⏎················sh.id·===·shareId·?·{·...sh,·permission·}·:·sh,⏎··············` with `·(sh.id·===·shareId·?·{·...sh,·permission·}·:·sh)`        prettier/prettier
  255:1   error    Replace `⏎··const·updateParameter:·Store["updateParameter"]·=·useCallback(⏎····` with `··const·updateParameter:·Store["updateParameter"]·=·useCallback(`  prettier/prettier
  258:1   error    Delete `··`                                                                                                                                               prettier/prettier
  259:7   error    Delete `··`                                                                                                                                               prettier/prettier
  260:1   error    Delete `··`                                                                                                                                               prettier/prettier
  261:1   error    Replace `············` with `··········`                                                                                                                  prettier/prettier
  262:1   error    Delete `··`                                                                                                                                               prettier/prettier
  263:15  error    Delete `··`                                                                                                                                               prettier/prettier
  264:1   error    Delete `··`                                                                                                                                               prettier/prettier
  265:1   error    Delete `··`                                                                                                                                               prettier/prettier
  266:15  error    Delete `··`                                                                                                                                               prettier/prettier
  267:1   error    Delete `··`                                                                                                                                               prettier/prettier
  268:1   error    Replace `········` with `······`                                                                                                                          prettier/prettier
  269:1   error    Delete `··`                                                                                                                                               prettier/prettier
  270:1   error    Replace `····},⏎····[],⏎··` with `··},·[]`                                                                                                                prettier/prettier
  330:5   error    Delete `⏎`                                                                                                                                                prettier/prettier
  336:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components                            react-refresh/only-export-components

/workspaces/isa-glass-platform/src/lib/theme.tsx
  24:30  error    Replace `⏎····()·=>·setTheme((t)·=>·(t·===·"dark"·?·"light"·:·"dark")),⏎····[],⏎··` with `()·=>·setTheme((t)·=>·(t·===·"dark"·?·"light"·:·"dark")),·[]`                                               prettier/prettier
  29:9   error    Replace `·(⏎····<ThemeContext.Provider·value={{·theme,·toggle·}}>{children}</ThemeContext.Provider>⏎··)` with `·<ThemeContext.Provider·value={{·theme,·toggle·}}>{children}</ThemeContext.Provider>`  prettier/prettier
  34:14  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components                                                                        react-refresh/only-export-components

/workspaces/isa-glass-platform/src/routes/activity.tsx
  61:65  error  Replace `⏎··················{Math.round(j.progress)}%⏎················` with `{Math.round(j.progress)}%`  prettier/prettier

/workspaces/isa-glass-platform/src/routes/index.tsx
  37:14  error  Replace `⏎······query={query}⏎······onQueryChange={setQuery}⏎······view={view}⏎······onViewChange={setView}⏎····` with `·query={query}·onQueryChange={setQuery}·view={view}·onViewChange={setView}`  prettier/prettier

/workspaces/isa-glass-platform/src/routes/solutions.$solutionId.dashboard.tsx
  22:17  error  Delete `⏎·········`  prettier/prettier

/workspaces/isa-glass-platform/src/routes/solutions.$solutionId.etl.tsx
   84:27  error  Replace `node.type,·node.x·+·36,·node.y·+·36,·node.config,·`${node.title}·(copia)`` with `⏎······node.type,⏎······node.x·+·36,⏎······node.y·+·36,⏎······node.config,⏎······`${node.title}·(copia)`,⏎····`  prettier/prettier
   96:27  error  Replace `⏎········setTimeout(()·=>·setStatuses({·[node.id]:·"running"·}),·i·*·420),⏎······` with `setTimeout(()·=>·setStatuses({·[node.id]:·"running"·}),·i·*·420)`                                        prettier/prettier
  100:20  error  Insert `⏎··········`                                                                                                                                                                                       prettier/prettier
  101:1   error  Insert `··`                                                                                                                                                                                                prettier/prettier
  102:1   error  Replace `··········` with `············`                                                                                                                                                                   prettier/prettier
  103:1   error  Replace `········},·i·*·420·+·380` with `··········},⏎··········i·*·420·+·380,⏎········`                                                                                                                   prettier/prettier
  221:45  error  Replace `⏎············updateNode(id,·{·config:·patch·})⏎··········` with `·updateNode(id,·{·config:·patch·})`                                                                                              prettier/prettier

/workspaces/isa-glass-platform/src/routes/solutions.$solutionId.model.tsx
  85:1   error  Insert `··········`                                                                                 prettier/prettier
  89:65  error  Replace `⏎············Fattore·{factor.toFixed(2)}x⏎··········` with `Fattore·{factor.toFixed(2)}x`  prettier/prettier

/workspaces/isa-glass-platform/src/routes/solutions.$solutionId.tsx
  53:1  error  Delete `⏎`  prettier/prettier

✖ 2054 problems (2040 errors, 14 warnings)
  2040 errors and 0 warnings potentially fixable with the `--fix` option.

```

## Test (npm test / vitest run)

Comando: `npm test`

Esito: OK (exit 0)
Durata: 1s

Ultime 60 righe di output:
```

> test
> vitest run

The plugin "vite-tsconfig-paths" is detected. Vite now supports tsconfig paths resolution natively via the resolve.tsconfigPaths option. You can remove the plugin and set resolve.tsconfigPaths: true in your Vite config instead.

 RUN  v5.0.1 /workspaces/isa-glass-platform


 Test Files  4 passed (4)
      Tests  41 passed (41)
   Start at  18:47:12
   Duration  645ms (transform 51%, import 27%, tests 14%, worker 7%)

    Isolate  4 workers spawned · ~99ms startup each (spawn + environment, per file)
             at least ~297ms faster with isolate: false — reuses workers across files instead of one per file

```

## Build (npm run build)

Comando: `npm run build`

Esito: OK (exit 0)
Durata: 6s

Ultime 60 righe di output:
```
.output/server/_ssr/favorites-CyGrO8TZ.mjs                          0.57 kB │ gzip:   0.38 kB
.output/server/_ssr/trash-BZw2hxgU.mjs                              0.58 kB │ gzip:   0.38 kB
.output/server/_chunks/ssr-renderer.mjs                             0.60 kB │ gzip:   0.36 kB
.output/server/_ssr/shared-Bq5kHv-K.mjs                             0.60 kB │ gzip:   0.39 kB
.output/server/_ssr/templates-DIZhedAs.mjs                          0.61 kB │ gzip:   0.39 kB
.output/server/_ssr/solutions._solutionId.dashboard-CujVFx5K.mjs    0.86 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.model-D5GqwrKQ.mjs        0.88 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.etl-D-pkNJxv.mjs          0.90 kB │ gzip:   0.49 kB
.output/server/_libs/hookable.mjs                                   1.16 kB │ gzip:   0.51 kB
.output/server/_ssr/theme-ovWAvoLq.mjs                              1.23 kB │ gzip:   0.57 kB
.output/server/_ssr/start-RKGGYzjZ.mjs                              1.53 kB │ gzip:   0.70 kB
.output/server/_runtime.mjs                                         1.61 kB │ gzip:   0.74 kB
.output/server/_libs/react-is.mjs                                   1.67 kB │ gzip:   0.65 kB
.output/server/_libs/prop-types.mjs                                 2.27 kB │ gzip:   0.84 kB
.output/server/_ssr/createCsrfMiddleware-B2To0gPJ.mjs               3.11 kB │ gzip:   1.01 kB
.output/server/_libs/d3-path.mjs                                    3.25 kB │ gzip:   1.16 kB
.output/server/_ssr/activity-DmLzxlkY.mjs                           3.45 kB │ gzip:   0.97 kB
.output/server/_tanstack-start-manifest_v-DxyMn2-X.mjs              3.66 kB │ gzip:   0.85 kB
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
.output/server/_ssr/router-D3G03G_w.mjs                            14.19 kB │ gzip:   3.57 kB
.output/server/index.mjs                                           14.82 kB │ gzip:   4.31 kB
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
.output/server/_ssr/server-DIgwlAuZ.mjs                            54.77 kB │ gzip:  14.38 kB
.output/server/_libs/d3-scale+[...].mjs                            58.22 kB │ gzip:  12.11 kB
.output/server/_libs/@tanstack/router-core+[...].mjs              125.35 kB │ gzip:  26.53 kB
.output/server/_libs/lodash.mjs                                   161.80 kB │ gzip:  29.45 kB
.output/server/_ssr/solutions._solutionId.etl-C0HjIavK.mjs        169.54 kB │ gzip:  42.04 kB
.output/server/_libs/recharts+[...].mjs                           515.72 kB │ gzip:  96.80 kB
.output/server/_libs/@tanstack/react-router+[...].mjs             681.94 kB │ gzip: 143.15 kB

✓ built in 1.07s
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
e61af43 Changes
c6f039f Changes
344ccc6 Changes
b4e5d94 Changes
23c864a Changes
f9b06be Changes
18f2c22 Separato border da canvas
ff1b8f0 Changes
```

## Branch

```
* main
  remotes/origin/HEAD -> origin/main
  remotes/origin/main
```

## Branch diversi da main

Nessun branch locale diverso da `main`.

