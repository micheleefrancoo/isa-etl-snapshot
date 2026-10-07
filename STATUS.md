# STATUS.md

Generato: 2026-10-07T11:48:41Z (UTC)

## Type check

Nessuno script "typecheck" in package.json: eseguito il comando diretto.

Comando: `npx tsc --noEmit`

Esito: OK (exit 0)
Durata: 16s

Ultime 60 righe di output:
```
```

## Lint (npm run lint)

Comando: `npm run lint`

Esito: FALLITO (exit 1)
Durata: 17s

Ultime 60 righe di output:
```
  722:1   error  Replace `····]·}` with `········],⏎······},⏎···`                                                                                                                                                                                                                                                                                                                                                                                             prettier/prettier
  727:11  error  Replace ``${tag}·elenco:·l'operazione·a·voci·ha·«Elenco»·come·colonna·centrale`,·three·&&·(await·page.locator(".ei-col-master·.ei-col-head").textContent())·===·"Elenco"` with `⏎······`${tag}·elenco:·l'operazione·a·voci·ha·«Elenco»·come·colonna·centrale`,⏎······three·&&·(await·page.locator(".ei-col-master·.ei-col-head").textContent())·===·"Elenco",⏎····`                                                                          prettier/prettier
  733:9   error  Replace ``${tag}·ordina:·una·maniglia·per·criterio,·con·nome·accessibile`,·(await·page.locator(".ec-insp·.ei-row-grip").count())·===·3·&&·(await·grip(0).getAttribute("aria-label"))·===·"Sposta·Criterio·1"` with `⏎····`${tag}·ordina:·una·maniglia·per·criterio,·con·nome·accessibile`,⏎····(await·page.locator(".ec-insp·.ei-row-grip").count())·===·3·&&⏎······(await·grip(0).getAttribute("aria-label"))·===·"Sposta·Criterio·1",⏎··`  prettier/prettier
  739:9   error  Replace ``${tag}·ordina:·Alt+↓·sposta·il·primo·criterio·in·seconda·posizione`,·(await·order())·===·"importo,regione,stato",·await·order()` with `⏎····`${tag}·ordina:·Alt+↓·sposta·il·primo·criterio·in·seconda·posizione`,⏎····(await·order())·===·"importo,regione,stato",⏎····await·order(),⏎··`                                                                                                                                          prettier/prettier
  742:96  error  Insert `⏎·····`                                                                                                                                                                                                                                                                                                                                                                                                                              prettier/prettier
  748:9   error  Replace ``${tag}·ordina:·Alt+↑·lo·riporta·in·prima·posizione`,·(await·order())·===·"regione,importo,stato",·await·order()` with `⏎····`${tag}·ordina:·Alt+↑·lo·riporta·in·prima·posizione`,⏎····(await·order())·===·"regione,importo,stato",⏎····await·order(),⏎··`                                                                                                                                                                          prettier/prettier
  755:9   error  Replace ``${tag}·ordina:·annullare·uno·spostamento·da·tastiera·lo·riporta·com'era,·in·un·passo`,·(await·order())·===·"regione,importo,stato",·await·order()` with `⏎····`${tag}·ordina:·annullare·uno·spostamento·da·tastiera·lo·riporta·com'era,·in·un·passo`,⏎····(await·order())·===·"regione,importo,stato",⏎····await·order(),⏎··`                                                                                                      prettier/prettier
  760:35  error  Replace `".ec-insp·.ei-list·>·[data-row=\"2\"]"` with `'.ec-insp·.ei-list·>·[data-row="2"]'`                                                                                                                                                                                                                                                                                                                                                 prettier/prettier
  766:27  error  Replace `g0.x·+·g0.width·/·2,·g0.y·+·g0.height·/·2·+·((targetY·-·(g0.y·+·g0.height·/·2))·*·i)·/·steps` with `⏎······g0.x·+·g0.width·/·2,⏎······g0.y·+·g0.height·/·2·+·((targetY·-·(g0.y·+·g0.height·/·2))·*·i)·/·steps,⏎····`                                                                                                                                                                                                                prettier/prettier
  772:9   error  Replace ``${tag}·ordina:·trascinare·la·maniglia·del·primo·criterio·in·fondo·lo·porta·per·ultimo`,·(await·order())·===·"importo,stato,regione",·await·order()` with `⏎····`${tag}·ordina:·trascinare·la·maniglia·del·primo·criterio·in·fondo·lo·porta·per·ultimo`,⏎····(await·order())·===·"importo,stato,regione",⏎····await·order(),⏎··`                                                                                                    prettier/prettier
  775:9   error  Replace ``${tag}·ordina:·un·solo·annullamento·riporta·il·trascinamento·com'era`,·(await·order())·===·"regione,importo,stato",·await·order()` with `⏎····`${tag}·ordina:·un·solo·annullamento·riporta·il·trascinamento·com'era`,⏎····(await·order())·===·"regione,importo,stato",⏎····await·order(),⏎··`                                                                                                                                      prettier/prettier
  785:9   error  Replace ``${tag}·ordina:·Esc·durante·il·trascinamento·lo·annulla`,·(await·order())·===·"regione,importo,stato",·await·order()` with `⏎····`${tag}·ordina:·Esc·durante·il·trascinamento·lo·annulla`,⏎····(await·order())·===·"regione,importo,stato",⏎····await·order(),⏎··`                                                                                                                                                                  prettier/prettier
  817:15  error  Replace `resolve(OUT,·"misure.json"),·JSON.stringify({·risultati:·results,·misure:·measures·},·null,·2)` with `⏎··resolve(OUT,·"misure.json"),⏎··JSON.stringify({·risultati:·results,·misure:·measures·},·null,·2),⏎`                                                                                                                                                                                                                        prettier/prettier
  820:13  error  Replace ``\nprove:·${results.length},·fallite:·${failed},·secondi·clic·sulle·tendine:·${retries}`` with `⏎··`\nprove:·${results.length},·fallite:·${failed},·secondi·clic·sulle·tendine:·${retries}`,⏎`                                                                                                                                                                                                                                      prettier/prettier
  823:21  error  Replace `·Math.min(m.menu.x,·m.menu.y,·m.finestra.w·-·(m.menu.x·+·m.menu.w),·m.finestra.h·-·(m.menu.y·+·m.menu.h)` with `⏎····Math.min(⏎······m.menu.x,⏎······m.menu.y,⏎······m.finestra.w·-·(m.menu.x·+·m.menu.w),⏎······m.finestra.h·-·(m.menu.y·+·m.menu.h),⏎····`                                                                                                                                                                        prettier/prettier
  824:15  error  Replace ``tendine·misurate:·${menus.length},·distanza·minima·dai·bordi:·${Math.min(...menus.map(gap)).toFixed(1)}·px`` with `⏎····`tendine·misurate:·${menus.length},·distanza·minima·dai·bordi:·${Math.min(...menus.map(gap)).toFixed(1)}·px`,⏎··`                                                                                                                                                                                          prettier/prettier

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

/workspaces/isa-glass-platform/src/etl-canvas/inspector/Menu.tsx
  47:99  error  Insert `⏎···`    prettier/prettier
  52:90  error  Insert `⏎·····`  prettier/prettier

/workspaces/isa-glass-platform/src/lib/solutions-store.tsx
  327:17  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

/workspaces/isa-glass-platform/src/lib/theme.tsx
  54:14  warning  Fast refresh only works when a file only exports components. Use a new file to share constants or functions between components  react-refresh/only-export-components

✖ 113 problems (99 errors, 14 warnings)
  99 errors and 0 warnings potentially fixable with the `--fix` option.

```

## Test (npm test / vitest run)

Comando: `npm test`

Esito: OK (exit 0)
Durata: 46s

Ultime 60 righe di output:
```

> test
> vitest run

The plugin "vite-tsconfig-paths" is detected. Vite now supports tsconfig paths resolution natively via the resolve.tsconfigPaths option. You can remove the plugin and set resolve.tsconfigPaths: true in your Vite config instead.

 RUN  v5.0.1 /workspaces/isa-glass-platform


 Test Files  61 passed (61)
      Tests  1239 passed (1239)
   Start at  11:49:14
   Duration  46.29s (tests 87%, import 8%, transform 4%, worker 1%)

    Isolate  61 workers spawned · ~110ms startup each (spawn + environment, per file)
             at least ~6.63s faster with isolate: false — reuses workers across files instead of one per file

```

## Build (npm run build)

Comando: `npm run build`

Esito: OK (exit 0)
Durata: 9s

Ultime 60 righe di output:
```
.output/server/_ssr/favorites-C8Z5TnLg.mjs                          0.57 kB │ gzip:   0.38 kB
.output/server/_ssr/trash-Cy2PattI.mjs                              0.58 kB │ gzip:   0.37 kB
.output/server/_chunks/ssr-renderer.mjs                             0.60 kB │ gzip:   0.36 kB
.output/server/_ssr/shared-Bbcfg4ps.mjs                             0.60 kB │ gzip:   0.39 kB
.output/server/_ssr/templates-BxgvM8qO.mjs                          0.61 kB │ gzip:   0.39 kB
.output/server/_ssr/solutions._solutionId.dashboard-CujVFx5K.mjs    0.86 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.model-D5GqwrKQ.mjs        0.88 kB │ gzip:   0.47 kB
.output/server/_ssr/solutions._solutionId.etl-Dz4HNvhk.mjs          1.09 kB │ gzip:   0.58 kB
.output/server/_libs/hookable.mjs                                   1.16 kB │ gzip:   0.51 kB
.output/server/_ssr/start-RKGGYzjZ.mjs                              1.53 kB │ gzip:   0.70 kB
.output/server/_runtime.mjs                                         1.61 kB │ gzip:   0.74 kB
.output/server/_libs/react-is.mjs                                   1.67 kB │ gzip:   0.65 kB
.output/server/_libs/prop-types.mjs                                 2.27 kB │ gzip:   0.84 kB
.output/server/_ssr/createCsrfMiddleware-B2To0gPJ.mjs               3.11 kB │ gzip:   1.01 kB
.output/server/_libs/d3-path.mjs                                    3.25 kB │ gzip:   1.16 kB
.output/server/_ssr/activity-C06WSQVD.mjs                           3.45 kB │ gzip:   0.97 kB
.output/server/_tanstack-start-manifest_v-DR3GzHiF.mjs              3.71 kB │ gzip:   0.87 kB
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
.output/server/_ssr/router-BwDGZJDF.mjs                            14.47 kB │ gzip:   3.90 kB
.output/server/_ssr/theme-Dujx4GV7.mjs                             15.18 kB │ gzip:   6.06 kB
.output/server/_libs/fast-equals.mjs                               15.91 kB │ gzip:   4.13 kB
.output/server/_libs/h3-v2+rou3+srvx+unenv.mjs                     15.92 kB │ gzip:   4.39 kB
.output/server/index.mjs                                           16.41 kB │ gzip:   4.69 kB
.output/server/_libs/h3+rou3+srvx.mjs                              17.30 kB │ gzip:   4.98 kB
.output/server/_libs/react+tanstack__react-query.mjs               18.47 kB │ gzip:   4.82 kB
.output/server/_libs/decimal.js-light.mjs                          23.31 kB │ gzip:   6.89 kB
.output/server/_ssr/app-shell-DCrQJ3mI.mjs                         24.55 kB │ gzip:   5.63 kB
.output/server/_libs/d3-shape.mjs                                  24.56 kB │ gzip:   5.02 kB
.output/server/_ssr/routes-CyxR9BVE.mjs                            24.84 kB │ gzip:   4.48 kB
.output/server/_libs/lucide-react.mjs                              37.25 kB │ gzip:   7.17 kB
.output/server/_libs/react-smooth.mjs                              37.83 kB │ gzip:   8.14 kB
.output/server/_libs/tanstack__query-core.mjs                      49.84 kB │ gzip:  11.12 kB
.output/server/_ssr/server-D6VL85kT.mjs                            54.77 kB │ gzip:  14.39 kB
.output/server/_libs/d3-scale+[...].mjs                            58.22 kB │ gzip:  12.11 kB
.output/server/_libs/@tanstack/router-core+[...].mjs              125.35 kB │ gzip:  26.53 kB
.output/server/_libs/lodash.mjs                                   161.80 kB │ gzip:  29.45 kB
.output/server/_libs/recharts+[...].mjs                           515.72 kB │ gzip:  96.80 kB
.output/server/_ssr/solutions._solutionId.etl-C-FJe3rQ.mjs        560.37 kB │ gzip: 149.38 kB
.output/server/_libs/@tanstack/react-router+[...].mjs             681.94 kB │ gzip: 143.15 kB

✓ built in 1.11s
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
Durata: 0s

Ultime 60 righe di output:
```
check-tokens: 0 violazioni nei file controllati; debito preesistente: 26 (6 ombra, 7 raggio, 13 colore).
```

## Git log (ultimi 20 commit)

```
852be60 Fase 6b.2: verifica e2e di condizioni, layout a tre colonne e riordino, con le schermate
28c5a8a Fase 6b.2: Alt+frecce non muovono i nodi e il menu segue il campo quando il pannello scorre
ba05eb9 Fase 6b.2: riordino dei criteri di Ordina con la maniglia e test dei componenti
cbfd632 Fase 6b.2: condizioni di filtro e join, gruppi, anteprima e layout a tre colonne
ca41077 Fase 6b.2: Segmented, ConnectorSelect e transizioni delle condizioni
1b0f681 Fase 6b.2: righe del prototipo da portare (README di etl-canvas)
c8535cd Fase 6b.2, Passo 0c: le tacche dei pannelli stanno nell'area sicura
17a9cb2 Fase 6b.2, Passo 0b: regola delle tabelle del join assenti nei parametri
89576d3 Fase 6b.2, Passo 0a: colonne non presenti nei dati in ingresso
94f3974 Snapshot: i file vengono da git (tracciati e non ignorati), non dalla visita del working tree
9affb87 Snapshot: esclude i binari multimediali, suffisso dei blocchi oltre la 'z'
f9e1cfb Fase 6b.1.1: report di validazione, registro delle richieste di rete fallite
78cb9ee Fase 6b.1.1: verifica nel browser dello zoom automatico e correzione della misura a fine animazione
681c72a Fase 6b.1.1: test dell'animatore, easing a seno (pendenza ≤ π/2) per rispettare R4
aa9fb9c Fase 6b.1.1: animatore unico di pannelli e vista, al posto di keepVisible e della transizione CSS
6c3fb88 Fase 6b.1.1: modulo puro autoFit (nodi richiesti, bersaglio, ripristino, interpolazione, easing)
5dad418 Fase 6b.1: schermate dell'Inspector e report di validazione
6840ff1 Fase 6b.1: documentazione (README, note di divergenza, token di forma)
8dec8e2 Fase 6b.1: e2e, notte in due schermate, selettori dei pulsanti circoscritti alle conferme
738aa1e Fase 6b.1: e2e (colonne, valori, riordino), correzione del riordino per trascinamento delle etichette
```

## Branch

```
  feat/auto-zoom
  feat/design-tokens-unify
  feat/etl-canvas
  feat/etl-canvas-motion
  feat/etl-core
  feat/etl-layout
  feat/etl-layout-fix
  feat/etl-store
  feat/etl-store-fix
* feat/inspector-conditions
  feat/inspector-core
  feat/interactions
  feat/link-click-delete
  feat/multi-columns
  feat/panels
  feat/panels-exclusive
  feat/panels-fix
  feat/theme-system
  fix/drag-threshold
  main
  wip/stato-2026-09-28
  remotes/origin/HEAD -> origin/main
  remotes/origin/feat/auto-zoom
  remotes/origin/feat/design-tokens-unify
  remotes/origin/feat/etl-canvas
  remotes/origin/feat/etl-canvas-motion
  remotes/origin/feat/etl-core
  remotes/origin/feat/etl-layout
  remotes/origin/feat/etl-layout-fix
  remotes/origin/feat/etl-store
  remotes/origin/feat/etl-store-fix
  remotes/origin/feat/inspector-core
  remotes/origin/feat/interactions
  remotes/origin/feat/link-click-delete
  remotes/origin/feat/multi-columns
  remotes/origin/feat/panels
  remotes/origin/feat/panels-exclusive
  remotes/origin/feat/panels-fix
  remotes/origin/feat/theme-system
  remotes/origin/fix/drag-threshold
  remotes/origin/main
  remotes/origin/wip/stato-2026-09-28
```

## Branch diversi da main

### `feat/auto-zoom`

Ultimo commit:
```
f9e1cfb Fase 6b.1.1: report di validazione, registro delle richieste di rete fallite
```

Diff stat rispetto a main:
```
```

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

### `feat/inspector-conditions`

Ultimo commit:
```
852be60 Fase 6b.2: verifica e2e di condizioni, layout a tre colonne e riordino, con le schermate
```

Diff stat rispetto a main:
```
 ...etta-a-sinistra-scena-densa-1280x720-chiaro.png |  Bin 163561 -> 167404 bytes
 ...setta-a-sinistra-scena-densa-1280x720-scuro.png |  Bin 128678 -> 132608 bytes
 ...cassetta-a-sinistra-scena-densa-1440-chiaro.png |  Bin 158644 -> 157426 bytes
 .../cassetta-a-sinistra-scena-densa-1440-scuro.png |  Bin 132042 -> 131120 bytes
 .../inspector-in-basso-1280x720-dopo-chiaro.png    |  Bin 145955 -> 143981 bytes
 .../inspector-in-basso-1280x720-dopo-scuro.png     |  Bin 105820 -> 104068 bytes
 .../inspector-in-basso-1440-dopo-chiaro.png        |  Bin 132428 -> 134879 bytes
 .../inspector-in-basso-1440-dopo-scuro.png         |  Bin 104015 -> 106689 bytes
 docs/visual/fase6b11/misure.json                   |  850 +++++---
 .../striscia-apertura-inspector-in-basso.png       |  Bin 164873 -> 155066 bytes
 .../striscia-chiusura-inspector-in-basso.png       |  Bin 170578 -> 161567 bytes
 docs/visual/fase6b2/anteprima-chiaro.png           |  Bin 0 -> 165834 bytes
 docs/visual/fase6b2/anteprima-scuro.png            |  Bin 0 -> 139840 bytes
 .../fase6b2/condizioni-filtro-gruppi-chiaro.png    |  Bin 0 -> 159765 bytes
 .../condizioni-filtro-gruppi-notte-chiaro.png      |  Bin 0 -> 143219 bytes
 .../fase6b2/condizioni-filtro-gruppi-scuro.png     |  Bin 0 -> 135188 bytes
 docs/visual/fase6b2/connettore-aperto-chiaro.png   |  Bin 0 -> 164623 bytes
 docs/visual/fase6b2/connettore-aperto-scuro.png    |  Bin 0 -> 138007 bytes
 .../inspector-tre-colonne-1280x720-chiaro.png      |  Bin 0 -> 161507 bytes
 .../inspector-tre-colonne-1280x720-scuro.png       |  Bin 0 -> 123570 bytes
 .../fase6b2/inspector-tre-colonne-1440-chiaro.png  |  Bin 0 -> 160288 bytes
 .../inspector-tre-colonne-1440-notte-scuro.png     |  Bin 0 -> 111486 bytes
 .../fase6b2/inspector-tre-colonne-1440-scuro.png   |  Bin 0 -> 134675 bytes
 .../fase6b2/inspector-tre-colonne-alto-chiaro.png  |  Bin 0 -> 160240 bytes
 .../fase6b2/inspector-tre-colonne-alto-scuro.png   |  Bin 0 -> 135119 bytes
 .../fase6b2/join-avviso-prestazioni-chiaro.png     |  Bin 0 -> 159007 bytes
 .../fase6b2/join-avviso-prestazioni-scuro.png      |  Bin 0 -> 132875 bytes
 ...join-condizioni-colonna-valore-lista-chiaro.png |  Bin 0 -> 155223 bytes
 .../join-condizioni-colonna-valore-lista-scuro.png |  Bin 0 -> 130666 bytes
 docs/visual/fase6b2/misure.json                    | 2057 ++++++++++++++++++++
 .../operazione-a-voci-tre-colonne-chiaro.png       |  Bin 0 -> 158621 bytes
 .../operazione-a-voci-tre-colonne-scuro.png        |  Bin 0 -> 133375 bytes
 docs/visual/fase6b2/ordina-riordino-chiaro.png     |  Bin 0 -> 159990 bytes
 docs/visual/fase6b2/ordina-riordino-scuro.png      |  Bin 0 -> 133995 bytes
 scripts/e2e-fase6b11.mjs                           |   53 +-
 scripts/e2e-fase6b2.mjs                            |  826 ++++++++
 scripts/generate-snapshot.mjs                      |   21 +-
 scripts/snapshot-lib.mjs                           |   16 +
 scripts/snapshot-lib.test.mjs                      |   21 +-
 src/etl-canvas/EtlCanvas.tsx                       |    2 +
 src/etl-canvas/NOTE_DIVERGENZE.md                  |   26 +
 src/etl-canvas/README.md                           |   29 +
 src/etl-canvas/__tests__/autofit.test.ts           |   58 +
 src/etl-canvas/__tests__/conditions-ui.test.tsx    |  464 +++++
 src/etl-canvas/__tests__/conditions.test.ts        |  184 ++
 src/etl-canvas/__tests__/inspector-rules.test.ts   |   14 +
 src/etl-canvas/__tests__/inspector.test.tsx        |   90 +-
 src/etl-canvas/__tests__/overlay-layout.test.ts    |   63 +
 src/etl-canvas/__tests__/reorder.test.ts           |  110 ++
 src/etl-canvas/__tests__/segmented.test.tsx        |   81 +
 src/etl-canvas/inspector/Columns3.tsx              |   63 +
 src/etl-canvas/inspector/ConditionList.tsx         |  178 ++
 src/etl-canvas/inspector/ConnectorSelect.tsx       |   42 +
 src/etl-canvas/inspector/ExpressionPreview.tsx     |   30 +
 src/etl-canvas/inspector/FilterCondition.tsx       |  140 ++
 src/etl-canvas/inspector/GroupFrame.tsx            |   34 +
 src/etl-canvas/inspector/Inspector.tsx             |  218 ++-
 src/etl-canvas/inspector/JoinCondition.tsx         |  205 ++
 src/etl-canvas/inspector/JoinSettings.tsx          |   79 +
 src/etl-canvas/inspector/ListRow.tsx               |  122 ++
 src/etl-canvas/inspector/Menu.tsx                  |   21 +-
 src/etl-canvas/inspector/MultiList.tsx             |  254 ++-
 src/etl-canvas/inspector/Segmented.tsx             |   71 +
 src/etl-canvas/inspector/StyledSelect.tsx          |   18 +-
 src/etl-canvas/inspector/conditions.ts             |  102 +
 src/etl-canvas/inspector/copy.ts                   |   56 +-
 src/etl-canvas/inspector/inspector.css             |  270 +++
 src/etl-canvas/inspector/joinKeys.ts               |   25 +
 src/etl-canvas/inspector/joinSides.ts              |   86 +
 src/etl-canvas/inspector/logic.ts                  |   68 +
 src/etl-canvas/inspector/masterDetail.ts           |   42 +
 src/etl-canvas/inspector/params.ts                 |   19 +
 src/etl-canvas/inspector/useReorder.ts             |  149 ++
 src/etl-canvas/panels/InspectorShell.tsx           |    2 +-
 src/etl-canvas/panels/autoFit.ts                   |   24 +-
 src/etl-canvas/panels/overlayLayout.ts             |   28 +-
 src/etl-core/README.md                             |   30 +
 src/etl-core/__tests__/join-tables.test.ts         |   67 +
 src/etl-core/__tests__/multi-columns.test.ts       |   44 +-
 src/etl-core/__tests__/schema.test.ts              |   28 +-
 src/etl-core/index.ts                              |    2 +-
 src/etl-core/rules/state.ts                        |   29 +-
 src/etl-core/schema/schema.ts                      |   17 +
 src/theme/layout-tokens.css                        |    5 +
 84 files changed, 7027 insertions(+), 506 deletions(-)
```

### `feat/inspector-core`

Ultimo commit:
```
5dad418 Fase 6b.1: schermate dell'Inspector e report di validazione
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

### `feat/multi-columns`

Ultimo commit:
```
d560fdd Fase 6b.0: dominio, colonne multiple nelle voci
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

### `feat/panels-exclusive`

Ultimo commit:
```
8db8087 Fase 6a.2: report di validazione
```

Diff stat rispetto a main:
```
```

### `feat/panels-fix`

Ultimo commit:
```
cf6e294 Fase 6a.1: i pannelli in alto e in basso sottraggono altezza al canvas
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

