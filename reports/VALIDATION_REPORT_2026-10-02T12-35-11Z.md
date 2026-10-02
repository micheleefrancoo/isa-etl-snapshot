# Report di validazione — Fase T: sistema di temi

**Stato: COMPLETO** sul branch `feat/theme-system`. Con il tema predefinito l'app e il canvas sono identici a prima (vedi «Identità visiva»). Una sola coppia di contrasto del tema predefinito resta sotto soglia, per il vincolo di identità: è dichiarata e bloccata da un test (vedi «Contrasti»).

## Esiti

| Controllo | Esito |
| --- | --- |
| `npx tsc --noEmit` | nessun errore |
| `npm run lint` | 0 errori (14 avvisi preesistenti) |
| `npm test` | 590/590 (37 file; prima della fase: 400 in 30 file) |
| `npm run build` | riuscita |
| `node scripts/check-tokens.mjs` | 0 violazioni nei file controllati; debito preesistente: 26 valori (6 ombre, 7 raggi, 13 colori), non corretti |

### Test per file

| File | Superati |
| --- | --- |
| `src/canvas/__tests__/panelPositioning.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/canvasBounds.test.ts` | 13/13 |
| `src/canvas/layout/__tests__/dropZones.test.ts` | 9/9 |
| `src/canvas/layout/__tests__/panelRegistry.test.ts` | 10/10 |
| `src/etl-canvas/__tests__/engine.test.ts` | 20/20 |
| `src/etl-canvas/__tests__/flow.test.ts` | 16/16 |
| `src/etl-canvas/__tests__/loop.test.ts` | 10/10 |
| `src/etl-canvas/__tests__/no-reroute.test.ts` | 2/2 |
| `src/etl-canvas/__tests__/render.test.ts` | 14/14 |
| `src/etl-canvas/__tests__/ssr.test.tsx` | 4/4 |
| `src/etl-canvas/__tests__/tokens.test.ts` | 60/60 |
| `src/etl-canvas/__tests__/transitions.test.ts` | 13/13 |
| `src/etl-canvas/__tests__/view.test.ts` | 12/12 |
| `src/etl-core/__tests__/csv.test.ts` | 9/9 |
| `src/etl-core/__tests__/expressions.test.ts` | 7/7 |
| `src/etl-core/__tests__/fase11-requisiti.test.ts` | 30/30 |
| `src/etl-core/__tests__/fase11.test.ts` | 11/11 |
| `src/etl-core/__tests__/mutations.test.ts` | 9/9 |
| `src/etl-core/__tests__/params.test.ts` | 11/11 |
| `src/etl-core/__tests__/relations.test.ts` | 11/11 |
| `src/etl-core/__tests__/schema.test.ts` | 5/5 |
| `src/etl-core/__tests__/state.test.ts` | 26/26 |
| `src/etl-layout/__tests__/golden.test.ts` | 17/17 |
| `src/etl-layout/__tests__/properties.test.ts` | 11/11 |
| `src/etl-layout/__tests__/unit.test.ts` | 22/22 |
| `src/etl-store/__tests__/grouping.test.ts` | 11/11 |
| `src/etl-store/__tests__/persistence.test.ts` | 9/9 |
| `src/etl-store/__tests__/react.test.ts` | 3/3 |
| `src/etl-store/__tests__/reduce.test.ts` | 40/40 |
| `src/etl-store/__tests__/store.test.ts` | 13/13 |
| `src/lib/theme.test.tsx` | 1/1 |
| `src/theme/__tests__/check-tokens.test.ts` | 4/4 |
| `src/theme/__tests__/contrast.test.ts` | 104/104 |
| `src/theme/__tests__/derive.test.ts` | 10/10 |
| `src/theme/__tests__/readme.test.ts` | 1/1 |
| `src/theme/__tests__/runtime.test.ts` | 20/20 |
| `src/theme/__tests__/themes.test.ts` | 14/14 |

## Architettura

Tre livelli (dettaglio in `src/theme/README.md`): primitive `--isa-p-*` (`src/theme/primitives.css`, OKLCH), semantici (`src/theme/themes/prototipo.css` e `notte.css`, per modo), di componente (`--ec-*`, che leggono solo semantici). Selezione con `data-theme` + classe `dark` su `<html>`; `setTheme`, `setMode`, `setAccentHue` in `src/theme/runtime.ts`, preferenza in `localStorage['isa.theme.v1']`; `deriveAccent` in `src/theme/derive.ts` (puro); script di avvio in `__root.tsx`; `?theme=notte` solo in sviluppo.

Scelte da conoscere:
- Nel tema `prototipo` i valori storici sono scritti **direttamente** nell'assegnazione semantica, non via primitive: è ciò che la specifica chiede («gli attuali token diventano l'assegnazione semantica») e garantisce l'identità. Le primitive le usa il tema `notte`. Le due tavolozze non sono unificate.
- I nomi canvas `--isa-bg`, `--isa-ink`, `--isa-muted`, … sono diventati ruoli (`--isa-surface-base`, `--isa-text`, `--isa-text-secondary/-muted`, …); `--isa-link` resta il colore dei **cavi** del canvas, il collegamento ipertestuale è `--isa-text-link`.
- `tokens.css` del canvas non ha più il blocco `.dark .etl-canvas`: i token semantici variano col modo da soli. I nodi lavorazione hanno `data-family` (filtra-ordina, trasforma, merge-union, output) e leggono `--isa-op-*`; nel tema predefinito le quattro famiglie coincidono con la tinta unica di prima.
- Per lo script di avvio `setAccentHue` salva anche i token d'accento già derivati per i due modi; lo script non ricalcola nulla.
- Il modo scuro resta il predefinito se non c'è nulla di salvato (come prima); il vecchio `isa-theme` si legge ancora (mai scritto).

## Difetto di ThemeProvider

Corretto: la preferenza vive in uno store che **non scrive mai in lettura**; `ThemeProvider` si limita a leggerlo (`useSyncExternalStore`) e a riapplicarlo a `<html>`. Test di regressione in `src/theme/__tests__/runtime.test.ts`: scelto il chiaro, dopo il ricaricamento resta chiaro (anche con il doppio montaggio di StrictMode, e verificando che non ci sia nessun `setItem` in lettura). Verificato anche nel browser reale: `setMode('light')` → ricaricamento → classe `dark` assente, nessun avviso di idratazione né errore in console.

## Contrasti

Verificate **100 combinazioni sui temi** (2 temi × 2 modi × 3 superfici × testo, testo secondario, accento, anello di focus, 4 famiglie; più testo su accento) e **4032 combinazioni di `deriveAccent`** (72 tinte da 0 a 355 a passi di 5 × 2 modi × 2 temi, ciascuna su tre superfici). Tutte rispettano la soglia tranne la deroga sotto. Il margine minimo sulle altre è 1.011× la soglia (≥ 1 = a norma).

**Deroga (tema predefinito, chiaro e scuro): testo su accento.** Bianco su `#6c63ff` = **4,3153:1** (4,32 arrotondato; soglia 4,5). È il colore storico del prototipo e del canvas: cambiarlo altera i pixel (icone sui dataset, pulsanti), contro il vincolo critico. Registrata in `KNOWN_EXCEPTIONS` (`src/theme/__tests__/checks.ts`) con soglia minima 4,3153 (il valore misurato): il test fallisce se peggiora, e fallisce anche se torna a 4,5 (la deroga va tolta). **Decisione per te:** nel restyling portare il testo su accento a ≥ 4,5 (per esempio accento `#5f56f0`, 5,15:1) o accettare la deroga. Con `deriveAccent` e col tema `notte` la soglia è rispettata.

Altra deroga, solo canvas del tema `notte` chiaro: il puntino d'avviso `#F59E0B` (colore richiesto dalla specifica) dà 2,1476:1 sul fondo (`etl-canvas/__tests__/tokens.test.ts`), con la stessa disciplina bidirezionale (soglia minima 2,1476; fallisce se peggiora e se torna a 3:1 senza che la deroga sia tolta). **Nota: rivedere nella revisione di stile dopo la Fase 6.**

Come `deriveAccent` garantisce le soglie per ogni tinta: parte da luminosità e croma «di gusto» e sposta la luminosità a passi di 0,005 (verso lo scuro nel chiaro, verso il chiaro nello scuro) finché il contrasto, misurato sul valore arrotondato che emette, supera la soglia di 0,05; riduce la croma se il colore esce dal gamut sRGB. Lo fa contro due fondi di riferimento più sfavorevoli di ogni superficie reale (grigio OKLCH L 0,90 e L 0,32).

## Identità visiva del tema predefinito

Riferimenti: le quattro schermate committate **prima** di toccare qualunque cosa (commit «Fase T: schermate di riferimento…»). Rigenerate con `node scripts/visual-temi.mjs prototipo` e confrontate con `node scripts/visual-compare.mjs` sull'intera finestra 1440 × 900 (1.296.000 pixel):

| Immagine | Pixel diversi |
| --- | --- |
| `prototipo-chiaro-soluzioni.png` | 0 |
| `prototipo-chiaro-canvas.png` | 0 |
| `prototipo-scuro-soluzioni.png` | 0 |
| `prototipo-scuro-canvas.png` | 0 |

Ripetuto due volte, stesso esito. In più, un confronto più forte del pixel: gli **stili calcolati** (25 proprietà di pittura — colori, sfondi, bordi, raggi, ombre, filtri, riempimenti — di ogni elemento) del codice di prima e del nuovo, in 4 stati (2 pagine × 2 modi): **640 elementi, 0 differenze**.

Nota sul metodo: nell'elenco soluzioni il modo arriva ora da localStorage (il meccanismo reale); nel canvas del predefinito si fissa con la classe `.dark` come nelle fasi precedenti, perché l'etichetta del pulsante tema (stato React) deve restare quella dei riferimenti. Con il solo cambio di classe l'antialiasing del testo in `chiaro-soluzioni` oscilla di 19–29 pixel (scarto ≤ 2/255) da una esecuzione all'altra: rumore di rasterizzazione, non di colore (gli stili calcolati sono identici).

## Schermate per la tua revisione (`docs/visual/temi/`)

- Tema «notte»: `notte-{chiaro,scuro}-{soluzioni,canvas}.png`.
- `deriveAccent`: `tinta-{0,140,280}-{chiaro,scuro}-canvas.png`.

Osservazioni di design da giudicare a occhio: nel «notte» chiaro i nodi lavorazione hanno un bordo scuro (`slate-500`) richiesto dal test di contrasto del canvas (≥ 3:1); si può ammorbidire se rinunci a quella soglia.

## Debito dei valori scritti a mano (non corretto)

Elenco completo con file e riga in `docs/theme-debt.md` (rigenerabile con `node scripts/check-tokens.mjs --write-debt docs/theme-debt.md`). Totale 26 in 8 file.

- **`src/components/isa/sidebar.tsx` (1)**
  - riga 102 — ombra: `shadow-[inset_0_1px_0_var(--glass-border)]`

- **`src/components/isa/ui/isa-menu.tsx` (1)**
  - riga 409 — raggio: `rounded-[5px]`

- **`src/components/ui/chart.tsx` (7)**
  - riga 51 — colore: `#ccc`
  - riga 51 — colore: `#fff`
  - riga 51 — colore: `#ccc`
  - riga 51 — colore: `#ccc`
  - riga 51 — colore: `#fff`
  - riga 193 — raggio: `rounded-[2px]`
  - riga 283 — raggio: `rounded-[2px]`

- **`src/components/ui/drawer.tsx` (1)**
  - riga 41 — raggio: `rounded-t-[10px]`

- **`src/components/ui/sidebar.tsx` (2)**
  - riga 510 — ombra: `shadow-[0_0_0_1px_var(--sidebar-border)]`
  - riga 510 — ombra: `shadow-[0_0_0_1px_var(--sidebar-accent)]`

- **`src/lib/error-page.ts` (8)**
  - riga 9 — colore: `#fafafa`
  - riga 9 — colore: `#111`
  - riga 12 — colore: `#4b5563`
  - riga 15 — colore: `#111`
  - riga 15 — colore: `#fff`
  - riga 16 — colore: `#fff`
  - riga 16 — colore: `#111`
  - riga 16 — colore: `#d1d5db`

- **`src/routes/solutions.$solutionId.dashboard.tsx` (1)**
  - riga 62 — raggio: `borderRadius: 16`

- **`src/styles.css` (5)**
  - riga 149 — raggio: `border-radius: 9999px`
  - riga 165 — raggio: `border-radius: 9999px`
  - riga 217 — ombra: `filter: drop-shadow(0 0 3px var(--brand-glow))`
  - riga 224 — ombra: `box-shadow: 0 0 0 1px color-mix(in oklab, var(--brand) 45%, transparent), 0 0 24px color-mix(in oklab, var(--brand) 28%, transparent)`
  - riga 233 — ombra: `box-shadow: 0 0 0 2px var(--brand), 0 0 26px color-mix(in oklab, var(--brand) 35%, transparent)`

## Mappa dei token

Token semantico → valore per ogni tema e modo (generata da `scripts/theme-map.mjs`, identica a quella di `src/theme/README.md`, e verificata da un test).

#### Token dell'app (livello semantico, famiglia shadcn)

| Token | prototipo chiaro | prototipo scuro | notte chiaro | notte scuro |
|---|---|---|---|---|
| `--radius` | `1rem` | `1rem` | `0.75rem` | `0.75rem` |
| `--background` | `oklch(0.978 0.004 250)` | `oklch(0.19 0.008 260)` | `var(--isa-p-slate-50) (#f8fafc)` | `var(--isa-p-night-950) (#0b1220)` |
| `--foreground` | `oklch(0.24 0.015 255)` | `oklch(0.96 0.004 250)` | `var(--isa-p-slate-900) (#0f172a)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--card` | `oklch(1 0 0 / 70%)` | `oklch(0.3 0.01 260 / 45%)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-900) (#111a2e)` |
| `--card-foreground` | `oklch(0.24 0.015 255)` | `oklch(0.96 0.004 250)` | `var(--isa-p-slate-900) (#0f172a)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--popover` | `oklch(1 0 0 / 99%)` | `oklch(0.23 0.009 260 / 99%)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-800) (#172036)` |
| `--popover-foreground` | `oklch(0.24 0.015 255)` | `oklch(0.96 0.004 250)` | `var(--isa-p-slate-900) (#0f172a)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--primary` | `oklch(0.52 0.11 272)` | `oklch(0.68 0.11 275)` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--primary-foreground` | `oklch(0.99 0.002 250)` | `oklch(0.19 0.008 260)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-950) (#0b1220)` |
| `--secondary` | `oklch(0.95 0.006 250 / 72%)` | `oklch(0.32 0.01 260 / 60%)` | `var(--isa-p-slate-100) (#f1f5f9)` | `var(--isa-p-night-700) (#223049)` |
| `--secondary-foreground` | `oklch(0.32 0.015 255)` | `oklch(0.94 0.004 250)` | `var(--isa-p-slate-900) (#0f172a)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--muted` | `oklch(0.955 0.005 250 / 72%)` | `oklch(0.32 0.008 260 / 55%)` | `var(--isa-p-slate-100) (#f1f5f9)` | `var(--isa-p-night-700) (#223049)` |
| `--muted-foreground` | `oklch(0.53 0.012 255)` | `oklch(0.75 0.008 260)` | `var(--isa-p-slate-500) (#64748b)` | `var(--isa-p-slate-400) (#94a3b8)` |
| `--accent` | `oklch(0.94 0.02 270 / 75%)` | `oklch(0.4 0.035 275 / 55%)` | `var(--isa-p-navy-700-a10)` | `var(--isa-p-accent-300-a16)` |
| `--accent-foreground` | `oklch(0.32 0.03 270)` | `oklch(0.95 0.01 270)` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--destructive` | `oklch(0.6 0.19 22)` | `oklch(0.65 0.18 22)` | `var(--isa-p-red-600) (#dc2626)` | `var(--isa-p-red-400) (#f87171)` |
| `--destructive-foreground` | `oklch(0.99 0.002 250)` | `oklch(0.98 0.002 250)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-950) (#0b1220)` |
| `--border` | `oklch(0.55 0.012 255 / 14%)` | `oklch(1 0 0 / 11%)` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-slate-400-a18)` |
| `--input` | `oklch(0.55 0.012 255 / 18%)` | `oklch(1 0 0 / 15%)` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-slate-400-a32)` |
| `--ring` | `oklch(0.58 0.1 272)` | `oklch(0.68 0.11 275)` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--brand` | `oklch(0.52 0.11 272)` | `oklch(0.66 0.11 275)` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--brand-foreground` | `oklch(0.99 0.002 250)` | `oklch(0.98 0.004 250)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-950) (#0b1220)` |
| `--brand-glow` | `oklch(0.6 0.09 258)` | `oklch(0.68 0.08 250)` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--success` | `oklch(0.64 0.11 155)` | `oklch(0.72 0.11 155)` | `var(--isa-p-green-600) (#16a34a)` | `var(--isa-p-green-400) (#4ade80)` |
| `--warning` | `oklch(0.76 0.12 75)` | `oklch(0.8 0.12 80)` | `var(--isa-p-amber-500) (#f59e0b)` | `var(--isa-p-amber-400) (#fbbf24)` |
| `--glass` | `oklch(1 0 0 / 48%)` | `oklch(1 0 0 / 5%)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-900) (#111a2e)` |
| `--glass-strong` | `oklch(1 0 0 / 66%)` | `oklch(1 0 0 / 10%)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-800) (#172036)` |
| `--glass-border` | `oklch(1 0 0 / 55%)` | `oklch(1 0 0 / 11%)` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-slate-400-a18)` |
| `--chart-1` | `oklch(0.56 0.11 272)` | `oklch(0.68 0.11 275)` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--chart-2` | `oklch(0.66 0.09 225)` | `oklch(0.72 0.09 225)` | `var(--isa-p-violet-600) (#7c3aed)` | `var(--isa-p-violet-400) (#a78bfa)` |
| `--chart-3` | `oklch(0.62 0.06 255)` | `oklch(0.7 0.05 255)` | `var(--isa-p-orange-600) (#ea580c)` | `var(--isa-p-orange-400) (#fb923c)` |
| `--chart-4` | `oklch(0.7 0.1 165)` | `oklch(0.76 0.1 165)` | `var(--isa-p-green-600) (#16a34a)` | `var(--isa-p-green-400) (#4ade80)` |
| `--chart-5` | `oklch(0.76 0.11 85)` | `oklch(0.82 0.11 85)` | `var(--isa-p-amber-500) (#f59e0b)` | `var(--isa-p-amber-400) (#fbbf24)` |
| `--blob-1` | `oklch(0.86 0.035 265)` | `oklch(0.42 0.04 265)` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-night-700) (#223049)` |
| `--blob-2` | `oklch(0.88 0.035 225)` | `oklch(0.4 0.04 230)` | `var(--isa-p-blue-100) (#dbeafe)` | `var(--isa-p-night-800) (#172036)` |
| `--blob-3` | `oklch(0.9 0.02 250)` | `oklch(0.38 0.02 255)` | `var(--isa-p-slate-100) (#f1f5f9)` | `var(--isa-p-night-900) (#111a2e)` |
| `--type-etl-bg` | `oklch(0.93 0.035 215 / 62%)` | `oklch(0.4 0.035 215 / 55%)` | `var(--isa-p-blue-100) (#dbeafe)` | `var(--isa-p-blue-400-a16)` |
| `--type-etl-fg` | `oklch(0.35 0.09 215)` | `oklch(0.86 0.06 215)` | `var(--isa-p-blue-800) (#1e40af)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--type-model-bg` | `oklch(0.93 0.035 272 / 62%)` | `oklch(0.4 0.035 272 / 55%)` | `var(--isa-p-violet-100) (#ede9fe)` | `var(--isa-p-violet-400-a16)` |
| `--type-model-fg` | `oklch(0.35 0.09 272)` | `oklch(0.88 0.06 272)` | `var(--isa-p-violet-800) (#5b21b6)` | `var(--isa-p-violet-400) (#a78bfa)` |
| `--type-dashboard-bg` | `oklch(0.93 0.035 155 / 62%)` | `oklch(0.4 0.035 155 / 55%)` | `var(--isa-p-green-100) (#dcfce7)` | `var(--isa-p-green-400-a16)` |
| `--type-dashboard-fg` | `oklch(0.32 0.09 155)` | `oklch(0.86 0.06 155)` | `var(--isa-p-green-800) (#166534)` | `var(--isa-p-green-400) (#4ade80)` |
| `--shadow-glass` | `0 18px 45px -22px oklch(0.35 0.01 255 / 28%)` | `0 22px 55px -26px oklch(0 0 0 / 55%)` | `0 1px 3px 0 var(--isa-p-slate-900-a8), 0 8px 20px -12px var(--isa-p-slate-900-a14)` | `0 12px 32px -16px var(--isa-p-black-a40)` |
| `--sidebar` | `oklch(1 0 0 / 55%)` | `oklch(1 0 0 / 6%)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-900) (#111a2e)` |
| `--sidebar-foreground` | `oklch(0.3 0.015 255)` | `oklch(0.94 0.004 250)` | `var(--isa-p-slate-900) (#0f172a)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--sidebar-primary` | `oklch(0.52 0.11 272)` | `oklch(0.68 0.11 275)` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--sidebar-primary-foreground` | `oklch(0.99 0.002 250)` | `oklch(0.19 0.008 260)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-950) (#0b1220)` |
| `--sidebar-accent` | `oklch(1 0 0 / 72%)` | `oklch(1 0 0 / 11%)` | `var(--isa-p-slate-100) (#f1f5f9)` | `var(--isa-p-white-a8)` |
| `--sidebar-accent-foreground` | `oklch(0.3 0.03 270)` | `oklch(0.96 0.004 250)` | `var(--isa-p-slate-900) (#0f172a)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--sidebar-border` | `oklch(1 0 0 / 60%)` | `oklch(1 0 0 / 13%)` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-slate-400-a18)` |
| `--sidebar-ring` | `oklch(0.58 0.1 272)` | `oklch(0.68 0.11 275)` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |

#### Token semantici condivisi (`--isa-*`)

| Token | prototipo chiaro | prototipo scuro | notte chiaro | notte scuro |
|---|---|---|---|---|
| `--isa-surface-base` | `#f5f3ee` | `#17181d` | `var(--isa-p-slate-50) (#f8fafc)` | `var(--isa-p-night-950) (#0b1220)` |
| `--isa-surface-raised` | `rgba(255, 255, 255, 0.92)` | `rgba(36, 37, 45, 0.92)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-900) (#111a2e)` |
| `--isa-surface-overlay` | `#ffffff` | `#24252d` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-800) (#172036)` |
| `--isa-text` | `#262420` | `#f1f2f5` | `var(--isa-p-slate-900) (#0f172a)` | `var(--isa-p-slate-100) (#f1f5f9)` |
| `--isa-text-secondary` | `#6a645a` | `#a9abb3` | `var(--isa-p-slate-500) (#64748b)` | `var(--isa-p-slate-400) (#94a3b8)` |
| `--isa-text-muted` | `#847e74` | `#a9abb3` | `var(--isa-p-slate-500) (#64748b)` | `var(--isa-p-slate-400) (#94a3b8)` |
| `--isa-text-on-accent` | `#ffffff` | `#ffffff` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-950) (#0b1220)` |
| `--isa-text-link` | `#6c63ff` | `#a8a3ff` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--isa-border` | `rgba(38, 36, 32, 0.06)` | `rgba(255, 255, 255, 0.11)` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-slate-400-a18)` |
| `--isa-border-strong` | `rgba(38, 36, 32, 0.16)` | `rgba(255, 255, 255, 0.22)` | `var(--isa-p-slate-300) (#cbd5e1)` | `var(--isa-p-slate-400-a32)` |
| `--isa-accent` | `#6c63ff` | `#6c63ff` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--isa-accent-active` | `#5a51e6` | `#8a83ff` | `var(--isa-p-navy-800) (#172e57)` | `var(--isa-p-accent-400) (#6b94e0)` |
| `--isa-accent-soft` | `rgba(108, 99, 255, 0.16)` | `rgba(108, 99, 255, 0.28)` | `var(--isa-p-navy-700-a10)` | `var(--isa-p-accent-300-a16)` |
| `--isa-accent-text` | `#6c63ff` | `#a8a3ff` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--isa-focus-ring` | `#6c63ff` | `#a8a3ff` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--isa-op-filter` | `var(--isa-tint-ink)` | `var(--isa-tint-ink)` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--isa-op-filter-soft` | `var(--isa-tint)` | `var(--isa-tint)` | `var(--isa-p-blue-100) (#dbeafe)` | `var(--isa-p-blue-400-a16)` |
| `--isa-op-transform` | `var(--isa-tint-ink)` | `var(--isa-tint-ink)` | `var(--isa-p-violet-600) (#7c3aed)` | `var(--isa-p-violet-400) (#a78bfa)` |
| `--isa-op-transform-soft` | `var(--isa-tint)` | `var(--isa-tint)` | `var(--isa-p-violet-100) (#ede9fe)` | `var(--isa-p-violet-400-a16)` |
| `--isa-op-merge` | `var(--isa-tint-ink)` | `var(--isa-tint-ink)` | `var(--isa-p-orange-600) (#ea580c)` | `var(--isa-p-orange-400) (#fb923c)` |
| `--isa-op-merge-soft` | `var(--isa-tint)` | `var(--isa-tint)` | `var(--isa-p-orange-100) (#ffedd5)` | `var(--isa-p-orange-400-a16)` |
| `--isa-op-output` | `var(--isa-tint-ink)` | `var(--isa-tint-ink)` | `var(--isa-p-green-600) (#16a34a)` | `var(--isa-p-green-400) (#4ade80)` |
| `--isa-op-output-soft` | `var(--isa-tint)` | `var(--isa-tint)` | `var(--isa-p-green-100) (#dcfce7)` | `var(--isa-p-green-400-a16)` |
| `--isa-dataset-fill` | `#6c63ff` | `#6c63ff` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--isa-warning` | `#e0a23b` | `#e8b34f` | `var(--isa-p-amber-500) (#f59e0b)` | `var(--isa-p-amber-400) (#fbbf24)` |
| `--isa-warning-ring` | `#f7f5f1` | `#17181d` | `var(--isa-p-slate-50) (#f8fafc)` | `var(--isa-p-night-950) (#0b1220)` |
| `--isa-error` | `oklch(0.6 0.19 22)` | `oklch(0.65 0.18 22)` | `var(--isa-p-red-600) (#dc2626)` | `var(--isa-p-red-400) (#f87171)` |
| `--isa-success` | `oklch(0.64 0.11 155)` | `oklch(0.72 0.11 155)` | `var(--isa-p-green-600) (#16a34a)` | `var(--isa-p-green-400) (#4ade80)` |
| `--isa-radius-control` | `calc(var(--radius) - 2px)` | `calc(var(--radius) - 2px)` | `calc(var(--radius) - 2px)` | `calc(var(--radius) - 2px)` |
| `--isa-radius-panel` | `calc(var(--radius) + 4px)` | `calc(var(--radius) + 4px)` | `calc(var(--radius) + 4px)` | `calc(var(--radius) + 4px)` |
| `--isa-radius-node-op` | `22px` | `22px` | `18px` | `18px` |
| `--isa-radius-node-fill` | `26px` | `26px` | `22px` | `22px` |
| `--isa-radius-pill` | `999px` | `999px` | `999px` | `999px` |
| `--isa-radius-xs` | `2px` | `2px` | `2px` | `2px` |
| `--isa-radius-sm` | `4px` | `4px` | `4px` | `4px` |
| `--isa-shadow-glass` | `0 10px 24px -14px rgba(38, 36, 32, 0.4)` | `0 10px 24px -14px rgba(0, 0, 0, 0.6)` | `0 1px 2px 0 var(--isa-p-slate-900-a8), 0 6px 16px -10px var(--isa-p-slate-900-a14)` | `0 10px 24px -14px var(--isa-p-black-a40)` |
| `--isa-shadow-raised` | `0 18px 45px -22px oklch(0.35 0.01 255 / 28%)` | `0 22px 55px -26px oklch(0 0 0 / 55%)` | `0 1px 3px 0 var(--isa-p-slate-900-a8), 0 8px 20px -12px var(--isa-p-slate-900-a14)` | `0 12px 32px -16px var(--isa-p-black-a40)` |
| `--isa-blur-glass` | `16px` | `16px` | `6px` | `6px` |
| `--isa-duration-fast` | `120ms` | `120ms` | `100ms` | `100ms` |
| `--isa-duration-base` | `200ms` | `200ms` | `160ms` | `160ms` |
| `--isa-duration-slow` | `320ms` | `320ms` | `260ms` | `260ms` |
| `--isa-ring-select` | `0 0 0 3px var(--isa-select)` | `0 0 0 3px var(--isa-select)` | `0 0 0 3px var(--isa-select)` | `0 0 0 3px var(--isa-select)` |
| `--isa-outline-node-op` | `inset 0 0 0 1.5px var(--isa-tint-border)` | `inset 0 0 0 1.5px var(--isa-tint-border)` | `inset 0 0 0 1.5px var(--isa-tint-border)` | `inset 0 0 0 1.5px var(--isa-tint-border)` |
| `--isa-stage` | `rgba(255, 255, 255, 0.32)` | `rgba(255, 255, 255, 0.04)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-white-a8)` |
| `--isa-accent-soft-2` | `rgba(108, 99, 255, 0.34)` | `rgba(108, 99, 255, 0.5)` | `var(--isa-p-navy-700-a16)` | `var(--isa-p-accent-300-a28)` |
| `--isa-tint` | `#e1dcf0` | `#3a3670` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-night-700) (#223049)` |
| `--isa-tint-ink` | `#6c63ff` | `#d0ccff` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--isa-tint-border` | `rgba(0, 0, 0, 0)` | `#7f78e6` | `var(--isa-p-slate-500) (#64748b)` | `var(--isa-p-slate-500) (#64748b)` |
| `--isa-split-bg` | `#efedf7` | `#2a2843` | `var(--isa-p-slate-100) (#f1f5f9)` | `var(--isa-p-night-800) (#172036)` |
| `--isa-split-empty` | `#e6e3f5` | `#2e2c4d` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-night-700) (#223049)` |
| `--isa-split-empty-ink` | `#8f88c7` | `#a8a3e6` | `var(--isa-p-slate-600) (#475569)` | `var(--isa-p-slate-400) (#94a3b8)` |
| `--isa-split-line` | `rgba(108, 99, 255, 0.45)` | `rgba(168, 163, 255, 0.55)` | `var(--isa-p-slate-400) (#94a3b8)` | `var(--isa-p-slate-400) (#94a3b8)` |
| `--isa-select` | `#6c63ff` | `#a8a3ff` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--isa-link` | `rgba(108, 99, 255, 0.34)` | `#7f78e6` | `var(--isa-p-slate-500) (#64748b)` | `var(--isa-p-slate-400) (#94a3b8)` |
| `--isa-link-dot-tint` | `#e1dcf0` | `#a8a3ff` | `var(--isa-p-slate-200) (#e2e8f0)` | `var(--isa-p-night-700) (#223049)` |
| `--isa-flow` | `rgba(108, 99, 255, 0.6)` | `rgba(168, 163, 255, 0.9)` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--isa-mm-node` | `#cfc9ef` | `#7b74d9` | `var(--isa-p-slate-500) (#64748b)` | `var(--isa-p-slate-500) (#64748b)` |
| `--isa-mm-node-ds` | `#6c63ff` | `#a8a3ff` | `var(--isa-p-navy-700) (#1e3a6e)` | `var(--isa-p-accent-300) (#7fa3e8)` |
| `--isa-mm-view-line` | `#6c63ff` | `#a8a3ff` | `var(--isa-p-blue-600) (#2563eb)` | `var(--isa-p-blue-400) (#60a5fa)` |
| `--isa-mm-view-bg` | `rgba(108, 99, 255, 0.08)` | `rgba(168, 163, 255, 0.12)` | `var(--isa-p-blue-600-a12)` | `var(--isa-p-blue-400-a16)` |

