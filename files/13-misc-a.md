# 13-misc-a.md

File in questo blocco:

- `src/theme/README.md`
- `src/theme/__tests__/check-tokens.test.ts`
- `src/theme/__tests__/checks.ts`
- `src/theme/__tests__/contrast.test.ts`
- `src/theme/__tests__/derive.test.ts`
- `src/theme/__tests__/readme.test.ts`
- `src/theme/__tests__/runtime.test.ts`
- `src/theme/__tests__/support.ts`
- `src/theme/__tests__/themes.test.ts`

---

### `src/theme/README.md`

226 righe

```md
# Sistema di temi

Architettura che permetterà all'utente di personalizzare l'interfaccia entro limiti definiti: un tema principale e la tinta dell'accento. In questa fase c'è **solo l'infrastruttura** (nessuna interfaccia di scelta), due temi e le garanzie di contrasto.

## Tre livelli di token

| Livello          | Prefisso                                                                                 | Dove                                                    | Chi lo usa                                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 1. Primitive     | `--isa-p-*`                                                                              | `primitives.css`                                        | solo i temi. **Mai** i componenti. Tavolozze grezze in OKLCH, indipendenti da tema e modo.                                       |
| 2. Semantici     | `--isa-*` (canvas e ruoli condivisi) e i token dell'app (`--background`, `--primary`, …) | `themes/<tema>.css`                                     | i componenti, direttamente o tramite il livello 3. Un tema è l'assegnazione di tutti i token semantici, per modo chiaro e scuro. |
| 3. Di componente | `--ec-*` e simili                                                                        | accanto al componente (es. `src/etl-canvas/tokens.css`) | solo il componente. Usano **soltanto** token semantici.                                                                          |

Le due tavolozze storiche (token OKLCH dell'app in `styles.css`, primitive `--isa-*` del canvas) **non sono state unificate**: sono messe sotto la stessa architettura. Nel tema `prototipo` i loro valori sono scritti direttamente nell'assegnazione semantica (sono quelli storici e devono restare identici); il tema `notte` invece passa dalle primitive.

## Selezione di tema e modo

Due attributi su `<html>`:

- modo: la classe `dark` (come prima, la legge anche `@custom-variant dark` di Tailwind);
- tema: `data-theme="prototipo" | "notte"`.

Cascata, a specificità crescente: `:root` (prototipo chiaro) < `.dark` (prototipo scuro) < `:root[data-theme="X"]` < `:root.dark[data-theme="X"]`. Un tema diverso dal predefinito **assegna tutti i token** (nessuno ricade sul predefinito): lo verifica `themes.test.ts`.

Per aggiungere un tema: `themes/<nome>.css` con i due blocchi, importarlo in `index.css`, aggiungere il nome a `THEME_NAMES` in `runtime.ts`. I test dei contrasti lo coprono da soli (`THEMES` in `__tests__/support.ts` legge i file).

## API (nessuna interfaccia utente)

```ts
import { setTheme, setMode, setAccentHue } from "@/theme";

setTheme("notte"); // tema
setMode("light"); // modo (l'header usa toggle() di useTheme)
setAccentHue(140); // tinta OKLCH 0–360; null torna all'accento del tema
```

La preferenza è in `localStorage["isa.theme.v1"]`: `{ v, theme, mode, accentHue, accent }`. `accent` contiene i token d'accento **già derivati** per i due modi, così lo script di avvio li applica senza ricalcolare. Se la chiave manca si usa il vecchio `isa-theme` (solo lettura) per il modo; in assenza di tutto: tema `prototipo`, scuro, come l'app si è sempre aperta.

**Leggere non scrive mai.** Si scrive solo in `setTheme` / `setMode` / `setAccentHue`. È la correzione del difetto di `ThemeProvider`, che scriveva «dark» in localStorage prima di leggere il valore salvato (in sviluppo, con StrictMode), per cui il chiaro non sopravviveva a un ricaricamento. Test di regressione: `__tests__/runtime.test.ts`.

### Nessun lampo al caricamento

`boot.ts` produce uno script autoeseguito inserito nella testa della pagina (`src/routes/__root.tsx`, prima di qualunque modulo). Applica classe `dark`, `data-theme` e i token d'accento salvati **prima del primo disegno**. `<html>` ha `suppressHydrationWarning` perché lo script ne cambia gli attributi prima dell'idratazione. Solo in sviluppo (`import.meta.env.DEV`), `?theme=notte` forza il tema senza salvarlo.

## Tinta dell'accento (`derive.ts`)

`deriveAccent(hue, mode)` è una funzione pura (nessun DOM). L'utente sceglie solo la tinta; luminosità e croma le decide il sistema:

1. si parte da una luminosità e una croma «di gusto» (chiaro: L 0,60, C 0,19; scuro: L 0,68, C 0,17);
2. se per quella tinta il contrasto richiesto non basta, la **luminosità si sposta a passi di 0,005** (verso lo scuro nel chiaro, verso il chiaro nello scuro) finché la soglia è rispettata, con 0,05 di margine sopra il minimo;
3. se il colore esce dal gamut sRGB si riduce la croma (il valore è identico su ogni schermo);
4. il contrasto si misura sul valore già arrotondato che viene emesso.

La funzione non conosce il tema, quindi garantisce le soglie contro due fondi di riferimento più sfavorevoli di qualunque superficie reale: grigio OKLCH L 0,90 (chiaro) e L 0,32 (scuro). I test lo verificano sulle superfici vere di ogni tema.

`accentCssVars(hue, mode)` traduce il risultato nelle proprietà CSS scritte su `<html>` (stile inline, che batte i temi): i token d'accento semantici, quelli del canvas che ne dipendono (cavi, minimappa, fette, selezione, riempimento dei dataset) e quelli dell'app (`--primary`, `--brand`, `--ring`, …).

## Contrasti garantiti (test automatici)

Per ogni tema e modo (`__tests__/contrast.test.ts`):

- testo principale e secondario sulle tre superfici (base, rialzata, sovrapposta): ≥ 4,5:1;
- testo su accento: ≥ 4,5:1;
- accento, colori delle quattro famiglie di operazioni e anello di focus sulle superfici: ≥ 3:1.

Per `deriveAccent`, per ogni tinta 0–355 a passi di 5, in ogni modo e su ogni tema: testo su accento (`base` e `active`), accento, anello di focus, accento come testo (≥ 4,5:1), icona su chip tinto, selezione e icona sulla fetta vuota. Il test fallisce se anche una sola combinazione non rispetta la soglia.

**Deroga nota, nel tema predefinito.** Il testo bianco su `#6c63ff` (l'accento del prototipo) dà 4,3153:1 (4,32), sotto 4,5:1, sia in chiaro sia in scuro. Il valore è quello storico e il vincolo «identico al pixel» impedisce di cambiarlo: è registrato in `KNOWN_EXCEPTIONS` (`__tests__/checks.ts`) con 4,3153 come soglia minima. Il test fallisce se peggiora e anche se la coppia torna a rispettare 4,5 (la deroga va tolta). Da risolvere nel restyling della palette; con `deriveAccent` e col tema `notte` la soglia è rispettata.

Un'altra deroga, solo nel tema `notte` chiaro: il puntino d'avviso del canvas (`#F59E0B`, richiesto dalla specifica) sul fondo dà 2,1476:1 (`etl-canvas/__tests__/tokens.test.ts`), con la stessa disciplina bidirezionale (soglia minima 2,1476; fallisce anche se torna a 3:1 senza toglierla). Nota: rivedere nella revisione di stile dopo la Fase 6.

## Disciplina dei token

`node scripts/check-tokens.mjs` (anche in `npm test`, `npm run check:tokens` e in `scripts/sync-snapshot.sh`) fallisce se trova colori (`#hex`, `rgb()`, `hsl()`, `oklch()`…), raggi o ombre letterali in `src/etl-canvas/` e in ogni file nuovo sotto `src/`, fuori da `src/theme/`. I file preesistenti sono elencati in `scripts/token-legacy-files.txt` e **non** vengono corretti: i loro valori scritti a mano sono il debito da saldare nel restyling (`docs/theme-debt.md`, con file e riga; rigenerabile con `node scripts/check-tokens.mjs --write-debt docs/theme-debt.md`).

## Verifica visiva

`node scripts/visual-temi.mjs prototipo|notte|tinte` salva le schermate in `docs/visual/temi/`. Le quattro `prototipo-*` sono i riferimenti presi **prima** di questa fase: `node scripts/visual-compare.mjs docs/visual/temi/prototipo-*.png` le confronta al pixel con quelle committate.

## Mappa dei token

Token semantico → valore assegnato, per tema e modo. Dove il valore è una primitiva si indica anche il suo esadecimale. I token del livello 3 (`--ec-*`) non compaiono qui: leggono questi.

<!-- BEGIN token-map (generata da scripts/theme-map.mjs: non modificare a mano) -->

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
| `--isa-drop-merge` | `var(--isa-select)` | `var(--isa-select)` | `var(--isa-select)` | `var(--isa-select)` |
| `--isa-drop-link` | `oklch(0.5 0.12 155)` | `oklch(0.74 0.12 155)` | `var(--isa-p-green-600) (#16a34a)` | `var(--isa-p-green-400) (#4ade80)` |
| `--isa-drop-link-reverse` | `var(--isa-drop-link)` | `var(--isa-drop-link)` | `var(--isa-drop-link)` | `var(--isa-drop-link)` |
| `--isa-drop-displace` | `oklch(0.52 0.12 65)` | `var(--isa-warning)` | `var(--isa-p-orange-600) (#ea580c)` | `var(--isa-p-orange-400) (#fb923c)` |
| `--isa-drop-reject` | `oklch(0.52 0.19 25)` | `oklch(0.72 0.16 22)` | `var(--isa-p-red-600) (#dc2626)` | `var(--isa-p-red-400) (#f87171)` |
| `--isa-drop-insert` | `var(--isa-select)` | `var(--isa-select)` | `var(--isa-select)` | `var(--isa-select)` |
| `--isa-doomed` | `var(--isa-drop-reject)` | `var(--isa-drop-reject)` | `var(--isa-drop-reject)` | `var(--isa-drop-reject)` |
| `--isa-marquee-line` | `var(--isa-select)` | `var(--isa-select)` | `var(--isa-select)` | `var(--isa-select)` |
| `--isa-marquee-fill` | `var(--isa-mm-view-bg)` | `var(--isa-mm-view-bg)` | `var(--isa-mm-view-bg)` | `var(--isa-mm-view-bg)` |
| `--isa-temp-link` | `var(--isa-accent)` | `var(--isa-accent)` | `var(--isa-accent)` | `var(--isa-accent)` |
| `--isa-temp-link-muted` | `var(--isa-accent-soft-2)` | `var(--isa-accent-soft-2)` | `var(--isa-accent-soft-2)` | `var(--isa-accent-soft-2)` |
| `--isa-port-fill` | `#ffffff` | `var(--isa-surface-overlay)` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-night-800) (#172036)` |
| `--isa-port-line` | `var(--isa-accent)` | `var(--isa-accent)` | `var(--isa-accent)` | `var(--isa-accent)` |
| `--isa-danger` | `#b23a3a` | `#b23a3a` | `var(--isa-p-red-600) (#dc2626)` | `var(--isa-p-red-600) (#dc2626)` |
| `--isa-text-on-danger` | `#ffffff` | `#ffffff` | `var(--isa-p-white) (#ffffff)` | `var(--isa-p-white) (#ffffff)` |
| `--isa-shadow-drag` | `0 18px 30px -14px rgba(38, 36, 32, 0.4)` | `0 18px 30px -14px rgba(0, 0, 0, 0.6)` | `0 12px 24px -12px var(--isa-p-slate-900-a14)` | `0 12px 24px -12px var(--isa-p-black-a40)` |
| `--isa-shadow-overlay` | `0 40px 80px -30px rgba(38, 36, 32, 0.4)` | `0 40px 80px -30px rgba(0, 0, 0, 0.6)` | `0 24px 48px -24px var(--isa-p-slate-900-a14)` | `0 24px 48px -24px var(--isa-p-black-a40)` |
| `--isa-ring-select` | `0 0 0 3px var(--isa-select)` | `0 0 0 3px var(--isa-select)` | `0 0 0 3px var(--isa-select)` | `0 0 0 3px var(--isa-select)` |
| `--isa-ring-drop-merge` | `0 0 0 3px var(--isa-drop-merge)` | `0 0 0 3px var(--isa-drop-merge)` | `0 0 0 3px var(--isa-drop-merge)` | `0 0 0 3px var(--isa-drop-merge)` |
| `--isa-ring-drop-link` | `0 0 0 3px var(--isa-drop-link)` | `0 0 0 3px var(--isa-drop-link)` | `0 0 0 3px var(--isa-drop-link)` | `0 0 0 3px var(--isa-drop-link)` |
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

<!-- END token-map -->
```

### `src/theme/__tests__/check-tokens.test.ts`

57 righe

```ts
import { execFileSync, spawnSync } from "node:child_process";
import { mkdtempSync, writeFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { describe, expect, it } from "vitest";

const SCRIPT = new URL("../../../scripts/check-tokens.mjs", import.meta.url).pathname;

function checkFile(name: string, content: string) {
  const file = join(mkdtempSync(join(tmpdir(), "tokens-")), name);
  writeFileSync(file, content);
  const r = spawnSync("node", [SCRIPT, "--check-file", file], { encoding: "utf8" });
  return { code: r.status, out: r.stderr };
}

describe("scripts/check-tokens.mjs", () => {
  it("i file controllati del repository sono a norma", () => {
    expect(() => execFileSync("node", [SCRIPT], { encoding: "utf8" })).not.toThrow();
  });

  it("rifiuta colori letterali in CSS e in TSX", () => {
    for (const color of [
      "#fff",
      "#6c63ff",
      "rgb(0 0 0)",
      "rgba(1,2,3,.5)",
      "hsl(10 10% 10%)",
      "oklch(0.5 0.1 20)",
    ]) {
      expect(checkFile("a.css", `.a { color: ${color}; }`).code, color).toBe(1);
    }
    expect(checkFile("a.tsx", `export const c = "#6c63ff";`).code).toBe(1);
    expect(checkFile("a.tsx", `export const s = { background: "rgb(1, 2, 3)" };`).code).toBe(1);
  });

  it("rifiuta raggi e ombre letterali", () => {
    expect(checkFile("a.css", `.a { border-radius: 8px; }`).code).toBe(1);
    expect(checkFile("a.css", `.a { box-shadow: 0 0 0 1px var(--x); }`).code).toBe(1);
    expect(checkFile("a.css", `.a { filter: drop-shadow(0 0 3px var(--x)); }`).code).toBe(1);
    expect(checkFile("a.tsx", `export const s = { borderRadius: 12 };`).code).toBe(1);
    expect(
      checkFile("a.tsx", `export const c = "rounded-[12px] shadow-[0_1px_2px_red]";`).code,
    ).toBe(1);
  });

  it("accetta i token e ignora i commenti", () => {
    const ok = `/* #fff rgba(0,0,0,.5) */ .a { color: var(--isa-text); border-radius: var(--ec-r-op); box-shadow: var(--ec-glass-shadow); border-radius: 0; }`;
    expect(checkFile("ok.css", ok).code).toBe(0);
    expect(
      checkFile(
        "ok.tsx",
        `// #fff\nexport const c = "rounded-xl bg-background text-foreground"; /* rgb(0,0,0) */`,
      ).code,
    ).toBe(0);
  });
});
```

### `src/theme/__tests__/checks.ts`

119 righe

```ts
/**
 * Le combinazioni di contrasto richieste (Fase T, punto 4), calcolate sui
 * valori REALI dei temi e di deriveAccent. Usate da contrast.test.ts e, con
 * THEME_REPORT_OUT, per il rapporto di validazione.
 */
import { accentCssVars, deriveAccent } from "../derive";
import type { Mode } from "../derive";
import { contrast, over, parseColor } from "../color";
import type { Rgba } from "../color";
import { MODES, THEMES, resolveTokens, rgb, surfaces } from "./support";
import type { Decls, Theme } from "./support";

export interface Check {
  /** Identificatore stabile, es. `prototipo/light/text-on-accent su accent`. */
  readonly id: string;
  readonly ratio: number;
  readonly min: number;
}

/**
 * Deroghe note. Ognuna è una coppia del tema PREDEFINITO che non raggiunge la
 * soglia perché il vincolo «identico al pixel» impedisce di cambiare il valore.
 * `floor` è il rapporto misurato: il test fallisce se peggiora, e fallisce
 * anche se la coppia torna a rispettare la soglia (la deroga va tolta).
 */
const ON_ACCENT_WHY =
  "Bianco su #6c63ff (l'accento del prototipo) dà 4,3153:1 (4,32 arrotondato), sotto 4,5:1. Il valore è quello del prototipo " +
  "e del canvas attuale: cambiarlo altera i pixel (icona sui dataset, pulsanti). Da risolvere nel restyling " +
  "della palette; con deriveAccent o col tema «notte» la soglia è rispettata.";
export const KNOWN_EXCEPTIONS: Record<string, { floor: number; why: string }> = {
  "prototipo/light/text-on-accent su accent": { floor: 4.3153, why: ON_ACCENT_WHY },
  "prototipo/dark/text-on-accent su accent": { floor: 4.3153, why: ON_ACCENT_WHY },
};

const HUES = Array.from({ length: 72 }, (_, i) => i * 5);

function themeChecks(theme: Theme, mode: Mode): Check[] {
  const t = resolveTokens(theme, mode);
  const surf = surfaces(t);
  const out: Check[] = [];
  const add = (label: string, fg: Rgba, bg: Rgba, min: number) =>
    out.push({ id: `${theme}/${mode}/${label}`, ratio: contrast(over(fg, bg), bg), min });
  const fg = (n: string) => rgb(t, n);

  for (const [sn, s] of Object.entries(surf)) {
    add(`text su ${sn}`, fg("--isa-text"), s, 4.5);
    add(`text-secondary su ${sn}`, fg("--isa-text-secondary"), s, 4.5);
    add(`accent su ${sn}`, fg("--isa-accent"), s, 3);
    add(`focus-ring su ${sn}`, fg("--isa-focus-ring"), s, 3);
    // gesti (Fase 5): contorni degli esiti, cavo da inserire, riquadro, cavo provvisorio, porta, nodi da eliminare
    for (const g of [
      "drop-merge",
      "drop-link",
      "drop-link-reverse",
      "drop-displace",
      "drop-reject",
      "drop-insert",
      "doomed",
      "marquee-line",
      "temp-link",
      "port-line",
    ]) {
      add(`${g} su ${sn}`, fg(`--isa-${g}`), s, 3);
    }
    for (const fam of ["filter", "transform", "merge", "output"]) {
      add(`op-${fam} su ${sn}`, fg(`--isa-op-${fam}`), s, 3);
    }
  }
  const accentBg = over(fg("--isa-accent"), surf["base"] as Rgba);
  add(
    "text-on-danger su danger",
    fg("--isa-text-on-danger"),
    over(fg("--isa-danger"), surf["base"] as Rgba),
    4.5,
  );
  add("text-on-accent su accent", fg("--isa-text-on-accent"), accentBg, 4.5);
  return out;
}

/** Tutte le combinazioni di deriveAccent per ogni tinta 0–355 (passo 5), modo e tema. */
function deriveChecks(): Check[] {
  const out: Check[] = [];
  for (const theme of THEMES) {
    for (const mode of MODES) {
      const t: Decls = resolveTokens(theme, mode);
      const surf = Object.entries(surfaces(t));
      for (const hue of HUES) {
        const a = deriveAccent(hue, mode);
        const c = (s: string) => parseColor(s);
        const id = (label: string) => `derive/${theme}/${mode}/${hue}/${label}`;
        const add = (label: string, f: Rgba, bg: Rgba, min: number) =>
          out.push({ id: id(label), ratio: contrast(over(f, bg), bg), min });
        add("on-accent su base", c(a.onAccent), c(a.base), 4.5);
        add("on-accent su active", c(a.onAccent), c(a.active), 4.5);
        add("tint-ink su tint", c(a.tintInk), over(c(a.tint), surf[0]![1]), 3);
        for (const [sn, s] of surf) {
          add(`base su ${sn}`, c(a.base), s, 3);
          add(`focus-ring su ${sn}`, c(a.focusRing), s, 3);
          add(`text su ${sn}`, c(a.text), s, 4.5);
        }
        const vars = accentCssVars(hue, mode);
        add(
          "split-empty-ink su split-empty",
          c(vars["--isa-split-empty-ink"] as string),
          over(c(vars["--isa-split-empty"] as string), surf[0]![1]),
          3,
        );
        add("select su base", c(vars["--isa-select"] as string), surf[0]![1], 3);
      }
    }
  }
  return out;
}

export function allChecks(): { themes: Check[]; derive: Check[] } {
  const themes = THEMES.flatMap((th) => MODES.flatMap((m) => themeChecks(th, m)));
  return { themes, derive: deriveChecks() };
}
```

### `src/theme/__tests__/contrast.test.ts`

53 righe

```ts
import { writeFileSync } from "node:fs";
import { describe, expect, it } from "vitest";
import { KNOWN_EXCEPTIONS, allChecks } from "./checks";

const { themes, derive } = allChecks();

describe("contrasti dei temi (chiaro e scuro)", () => {
  it("verifica un numero di combinazioni coerente", () => {
    expect(themes.length).toBeGreaterThanOrEqual(2 * 2 * 3 * 7);
  });

  for (const c of themes) {
    const exception = KNOWN_EXCEPTIONS[c.id];
    if (exception) {
      it(`${c.id}: deroga nota, non deve peggiorare`, () => {
        expect(c.ratio, "la coppia ora rispetta la soglia: togliere la deroga").toBeLessThan(c.min);
        expect(c.ratio).toBeGreaterThanOrEqual(exception.floor);
      });
    } else {
      it(`${c.id}: almeno ${c.min}:1`, () => {
        expect(c.ratio).toBeGreaterThanOrEqual(c.min);
      });
    }
  }
});

describe("deriveAccent: ogni tinta 0–355 a passi di 5", () => {
  it("copre 72 tinte × 2 modi × 2 temi", () => {
    expect(new Set(derive.map((c) => c.id.split("/").slice(1, 4).join("/"))).size).toBe(72 * 2 * 2);
  });

  it("nessuna combinazione scende sotto la soglia", () => {
    const failing = derive.filter((c) => c.ratio < c.min);
    expect(failing.map((c) => `${c.id}: ${c.ratio.toFixed(2)} < ${c.min}`)).toEqual([]);
  });
});

if (process.env["THEME_REPORT_OUT"]) {
  describe("rapporto", () => {
    it("salva il riepilogo", () => {
      const summary = {
        themeCombinations: themes.length,
        deriveCombinations: derive.length,
        minThemeRatio: Math.min(...themes.map((c) => c.ratio / c.min)),
        minDeriveRatio: Math.min(...derive.map((c) => c.ratio / c.min)),
        exceptions: Object.entries(KNOWN_EXCEPTIONS).map(([id, e]) => ({ id, ...e })),
        themes: themes.map((c) => ({ id: c.id, ratio: +c.ratio.toFixed(2), min: c.min })),
      };
      writeFileSync(process.env["THEME_REPORT_OUT"] as string, JSON.stringify(summary, null, 1));
    });
  });
}
```

### `src/theme/__tests__/derive.test.ts`

79 righe

```ts
import { describe, expect, it } from "vitest";
import { contrast, inGamut, oklchToRgb, parseColor } from "../color";
import { ACCENT_VAR_NAMES, accentCssVars, deriveAccent } from "../derive";

describe("color", () => {
  it("OKLCH → sRGB coincide con valori noti", () => {
    const [r, g, b] = oklchToRgb(0.6279553606145516, 0.2576833077, 29.2338851923);
    expect([r, g, b].map(Math.round)).toEqual([255, 0, 0]);
    expect(parseColor("oklch(1 0 0)").slice(0, 3).map(Math.round)).toEqual([255, 255, 255]);
    expect(parseColor("oklch(0.5 0.1 200 / 40%)")[3]).toBeCloseTo(0.4);
  });
  it("contrasto bianco su nero = 21", () => {
    expect(contrast(parseColor("#fff"), parseColor("#000"))).toBeCloseTo(21, 5);
  });
  it("inGamut", () => {
    expect(inGamut(0.6, 0.1, 200)).toBe(true);
    expect(inGamut(0.6, 0.4, 200)).toBe(false);
  });
});

describe("deriveAccent", () => {
  it("è deterministico e dipende dalla tinta e dal modo", () => {
    expect(deriveAccent(140, "light")).toEqual(deriveAccent(140, "light"));
    expect(deriveAccent(140, "light").base).not.toBe(deriveAccent(140, "dark").base);
    expect(deriveAccent(140, "light").base).not.toBe(deriveAccent(200, "light").base);
  });
  it("360 equivale a 0 e le tinte negative si riportano in 0–360", () => {
    expect(deriveAccent(360, "dark")).toEqual(deriveAccent(0, "dark"));
    expect(deriveAccent(-90, "light")).toEqual(deriveAccent(270, "light"));
  });
  it("la tinta resta quella scelta", () => {
    expect(deriveAccent(140, "light").base).toMatch(/^oklch\(\S+ \S+ 140\)$/);
  });
  it("nel chiaro l'attivo è più scuro, nello scuro più chiaro", () => {
    const L = (s: string) => parseFloat(/oklch\(([\d.]+)/.exec(s)![1]!);
    expect(L(deriveAccent(250, "light").active)).toBeLessThan(L(deriveAccent(250, "light").base));
    expect(L(deriveAccent(250, "dark").active)).toBeGreaterThan(L(deriveAccent(250, "dark").base));
  });
  it("tutti i colori emessi sono dentro il gamut sRGB", () => {
    for (let h = 0; h < 360; h += 5) {
      for (const mode of ["light", "dark"] as const) {
        for (const v of [
          deriveAccent(h, mode).base,
          deriveAccent(h, mode).text,
          deriveAccent(h, mode).focusRing,
        ]) {
          const m = /oklch\(([\d.]+) ([\d.]+) ([\d.]+)\)/.exec(v)!;
          expect(inGamut(+m[1]!, +m[2]!, +m[3]!)).toBe(true);
        }
      }
    }
  });
});

describe("accentCssVars", () => {
  it("scrive i token d'accento semantici, del canvas e dell'app", () => {
    const v = accentCssVars(140, "light");
    for (const name of [
      "--isa-accent",
      "--isa-accent-active",
      "--isa-accent-soft",
      "--isa-text-on-accent",
      "--isa-focus-ring",
      "--isa-select",
      "--isa-split-bg",
      "--primary",
      "--ring",
      "--brand",
    ]) {
      expect(v, name).toHaveProperty(name);
    }
  });
  it("ACCENT_VAR_NAMES contiene i nomi di entrambi i modi", () => {
    for (const mode of ["light", "dark"] as const) {
      for (const n of Object.keys(accentCssVars(10, mode))) expect(ACCENT_VAR_NAMES).toContain(n);
    }
  });
});
```

### `src/theme/__tests__/readme.test.ts`

15 righe

```ts
import { readFileSync } from "node:fs";
import { describe, expect, it } from "vitest";
// @ts-expect-error modulo .mjs senza tipi
import { BEGIN, END, renderReadmeSection } from "../../../scripts/theme-map.mjs";

describe("src/theme/README.md", () => {
  it("la mappa dei token è aggiornata (node scripts/theme-map.mjs --write)", () => {
    const text = readFileSync(new URL("../README.md", import.meta.url), "utf8");
    const a = text.indexOf(BEGIN as string);
    const b = text.indexOf(END as string);
    expect(a).toBeGreaterThan(-1);
    expect(text.slice(a, b + (END as string).length)).toBe(renderReadmeSection());
  });
});
```

### `src/theme/__tests__/runtime.test.ts`

257 righe

```ts
import { describe, expect, it } from "vitest";
import { themeBootScript } from "../boot";
import {
  DEFAULT_PREFERENCE,
  LEGACY_MODE_KEY,
  THEME_STORAGE_KEY,
  applyPreference,
  createThemeStore,
  normalizeHue,
  parsePreference,
  themeFromSearch,
} from "../runtime";
import type { RootLike, StorageLike } from "../runtime";

/** <html> finto: classi, attributi e proprietà di stile. */
function fakeRoot() {
  const classes = new Set<string>();
  const attrs = new Map<string, string>();
  const style = new Map<string, string>();
  const root: RootLike = {
    classList: {
      toggle(name, force) {
        const on = force ?? !classes.has(name);
        if (on) classes.add(name);
        else classes.delete(name);
        return on;
      },
    },
    setAttribute: (n, v) => void attrs.set(n, v),
    style: {
      setProperty: (n, v) => void style.set(n, v),
      removeProperty: (n) => {
        const old = style.get(n) ?? "";
        style.delete(n);
        return old;
      },
    },
  };
  return { root, classes, attrs, style };
}

/** localStorage finto che registra l'ordine delle operazioni. */
function fakeStorage(initial: Record<string, string> = {}) {
  const data = new Map(Object.entries(initial));
  const log: string[] = [];
  const storage: StorageLike = {
    getItem: (k) => (log.push(`get ${k}`), data.get(k) ?? null),
    setItem: (k, v) => (log.push(`set ${k}`), void data.set(k, v)),
  };
  return { storage, data, log };
}

describe("difetto di ThemeProvider: il chiaro sopravvive a un ricaricamento", () => {
  it("scelto il chiaro, dopo il ricaricamento resta chiaro (store e <html>)", () => {
    const s = fakeStorage();
    const first = fakeRoot();
    const store1 = createThemeStore({ storage: s.storage, root: first.root });
    store1.sync();
    expect(first.classes.has("dark")).toBe(true); // predefinito: scuro
    store1.setMode("light");
    expect(first.classes.has("dark")).toBe(false);

    // ricaricamento: nuovo <html>, nuovo store, stesso localStorage
    const second = fakeRoot();
    const store2 = createThemeStore({ storage: s.storage, root: second.root });
    store2.sync();
    expect(store2.get().mode).toBe("light");
    expect(second.classes.has("dark")).toBe(false);
  });

  it("anche con il montaggio ripetuto di StrictMode (sync due volte) il chiaro resta", () => {
    const s = fakeStorage();
    createThemeStore({ storage: s.storage, root: fakeRoot().root }).setMode("light");
    const r = fakeRoot();
    const store = createThemeStore({ storage: s.storage, root: r.root });
    store.sync();
    store.sync();
    const again = createThemeStore({ storage: s.storage, root: fakeRoot().root });
    expect(again.get().mode).toBe("light");
  });

  it("leggere non scrive mai: creare lo store e sincronizzare non fa nessun setItem", () => {
    const s = fakeStorage({
      [THEME_STORAGE_KEY]: JSON.stringify({ v: 1, theme: "notte", mode: "light", accentHue: null }),
    });
    const store = createThemeStore({ storage: s.storage, root: fakeRoot().root });
    store.sync();
    store.sync();
    expect(s.log.filter((l) => l.startsWith("set"))).toEqual([]);
    expect(s.log[0]).toMatch(/^get /);
  });

  it("lo script di avvio applica il chiaro salvato prima del primo disegno", () => {
    const s = fakeStorage({
      [THEME_STORAGE_KEY]: JSON.stringify({ mode: "light", theme: "prototipo" }),
    });
    const r = fakeRoot();
    r.classes.add("dark"); // se non fa nulla, il chiaro non è applicato
    runBoot(r, s.storage, "");
    expect(r.classes.has("dark")).toBe(false);
  });
});

function runBoot(r: ReturnType<typeof fakeRoot>, storage: StorageLike, search: string, dev = true) {
  const doc = {
    documentElement: {
      classList: r.root.classList,
      setAttribute: r.root.setAttribute,
      style: r.root.style,
    },
  };
  new Function("document", "localStorage", "location", "URLSearchParams", themeBootScript(dev))(
    doc,
    storage,
    { search },
    URLSearchParams,
  );
}

describe("preferenza salvata", () => {
  it("dato mancante o rovinato → predefinito", () => {
    expect(parsePreference(null)).toEqual(DEFAULT_PREFERENCE);
    expect(parsePreference("{non json")).toEqual(DEFAULT_PREFERENCE);
    expect(parsePreference('{"theme":"x","mode":"y","accentHue":"z"}')).toEqual(DEFAULT_PREFERENCE);
  });
  it("ripiega sul vecchio modo (isa-theme) se manca la preferenza nuova", () => {
    expect(parsePreference(null, "light").mode).toBe("light");
    expect(parsePreference('{"mode":"dark"}', "light").mode).toBe("dark");
  });
  it("normalizza la tinta", () => {
    expect(normalizeHue(370)).toBe(10);
    expect(normalizeHue(-10)).toBe(350);
    expect(normalizeHue(NaN)).toBeNull();
    expect(normalizeHue("5")).toBeNull();
  });
});

describe("setTheme, setMode, setAccentHue", () => {
  it("salvano in isa.theme.v1 e applicano a <html>", () => {
    const s = fakeStorage();
    const r = fakeRoot();
    const store = createThemeStore({ storage: s.storage, root: r.root });
    store.setTheme("notte");
    store.setAccentHue(140);
    expect(r.attrs.get("data-theme")).toBe("notte");
    expect(r.style.get("--isa-accent")).toMatch(/^oklch\(/);
    const saved = JSON.parse(s.data.get(THEME_STORAGE_KEY)!);
    expect(saved).toMatchObject({ theme: "notte", accentHue: 140 });
    expect(saved.accent.light["--isa-accent"]).toBeTruthy();
    expect(saved.accent.dark["--isa-accent"]).toBeTruthy();
    store.setAccentHue(null);
    expect(r.style.has("--isa-accent")).toBe(false);
  });
  it("l'accento segue il cambio di modo", () => {
    const r = fakeRoot();
    const store = createThemeStore({ storage: fakeStorage().storage, root: r.root });
    store.setAccentHue(200);
    const light = r.style.get("--isa-accent");
    store.setMode("light");
    expect(r.style.get("--isa-accent")).not.toBe(light);
  });
  it("avvisa chi ascolta e rispetta l'annullamento", () => {
    const store = createThemeStore({ storage: fakeStorage().storage, root: fakeRoot().root });
    let n = 0;
    const off = store.subscribe(() => n++);
    store.toggleMode();
    off();
    store.toggleMode();
    expect(n).toBe(1);
  });
  it("senza archivio funziona per la sessione", () => {
    const store = createThemeStore({ root: fakeRoot().root });
    store.setMode("light");
    expect(store.get().mode).toBe("light");
  });
  it("applyPreference non lascia residui di un accento precedente", () => {
    const r = fakeRoot();
    applyPreference(r.root, { theme: "prototipo", mode: "dark", accentHue: 10 });
    applyPreference(r.root, { theme: "prototipo", mode: "dark", accentHue: null });
    expect(r.style.size).toBe(0);
  });
});

describe("?theme= (solo sviluppo)", () => {
  it("lo store lo onora solo con allowQueryTheme", () => {
    const on = createThemeStore({
      search: "?theme=notte",
      allowQueryTheme: true,
      root: fakeRoot().root,
    });
    const off = createThemeStore({ search: "?theme=notte", root: fakeRoot().root });
    expect(on.get().theme).toBe("notte");
    expect(off.get().theme).toBe("prototipo");
    expect(themeFromSearch("?theme=boh")).toBeNull();
  });
  it("lo script di avvio lo onora solo in sviluppo", () => {
    const dev = fakeRoot();
    runBoot(dev, fakeStorage().storage, "?theme=notte", true);
    expect(dev.attrs.get("data-theme")).toBe("notte");
    const prod = fakeRoot();
    runBoot(prod, fakeStorage().storage, "?theme=notte", false);
    expect(prod.attrs.get("data-theme")).toBe("prototipo");
    expect(themeBootScript(false)).not.toContain("search");
  });
  it("non viene salvato", () => {
    const s = fakeStorage();
    createThemeStore({
      storage: s.storage,
      search: "?theme=notte",
      allowQueryTheme: true,
      root: fakeRoot().root,
    }).sync();
    expect(s.data.has(THEME_STORAGE_KEY)).toBe(false);
  });
});

describe("script di avvio", () => {
  it("predefinito: scuro, tema prototipo", () => {
    const r = fakeRoot();
    runBoot(r, fakeStorage().storage, "");
    expect(r.classes.has("dark")).toBe(true);
    expect(r.attrs.get("data-theme")).toBe("prototipo");
  });
  it("usa il vecchio isa-theme se non c'è isa.theme.v1", () => {
    const r = fakeRoot();
    runBoot(r, fakeStorage({ [LEGACY_MODE_KEY]: "light" }).storage, "");
    expect(r.classes.has("dark")).toBe(false);
  });
  it("applica i token d'accento salvati per il modo corrente", () => {
    const s = fakeStorage();
    createThemeStore({ storage: s.storage, root: fakeRoot().root }).setAccentHue(280);
    createThemeStore({ storage: s.storage, root: fakeRoot().root }).setMode("light");
    const r = fakeRoot();
    runBoot(r, s.storage, "");
    const saved = JSON.parse(s.data.get(THEME_STORAGE_KEY)!);
    expect(r.style.get("--isa-accent")).toBe(saved.accent.light["--isa-accent"]);
  });
  it("sopporta archivio che lancia e dati rovinati", () => {
    const r = fakeRoot();
    const broken: StorageLike = {
      getItem: () => {
        throw new Error("no");
      },
      setItem: () => {},
    };
    expect(() => runBoot(r, broken, "")).not.toThrow();
    expect(() =>
      runBoot(fakeRoot(), fakeStorage({ [THEME_STORAGE_KEY]: "{{" }).storage, ""),
    ).not.toThrow();
  });
  it("è una sola funzione autoeseguita, senza import", () => {
    const src = themeBootScript(true);
    expect(src.trim().startsWith("(function(){")).toBe(true);
    expect(src).not.toMatch(/\bimport\b|\brequire\b/);
  });
});
```

### `src/theme/__tests__/support.ts`

99 righe

```ts
/**
 * Supporto ai test dei temi: legge i file CSS dei temi e risolve i token per
 * (tema, modo) con la stessa cascata del browser, cioè:
 *   :root  <  .dark  <  :root[data-theme=X]  <  :root.dark[data-theme=X]
 * (specificità crescente; il tema predefinito è `:root` / `.dark`).
 */
import { readFileSync } from "node:fs";
import { contrast, over, parseColor } from "../color";
import type { Rgba } from "../color";
import type { Mode } from "../derive";

const file = (p: string) => readFileSync(new URL(`../${p}`, import.meta.url), "utf8");

export const PRIMITIVES_CSS = file("primitives.css");
export const THEME_FILES = {
  prototipo: file("themes/prototipo.css"),
  notte: file("themes/notte.css"),
};
export type Theme = keyof typeof THEME_FILES;
export const THEMES = Object.keys(THEME_FILES) as Theme[];
export const MODES: Mode[] = ["light", "dark"];

export type Decls = Record<string, string>;

/** Blocchi `selettore { --token: [REDATTO]; … }` di primo livello, con i commenti tolti. */
export function parseBlocks(css: string): { selector: string; decls: Decls }[] {
  const clean = css.replace(/\/\*[\s\S]*?\*\//g, "");
  const out: { selector: string; decls: Decls }[] = [];
  for (const m of clean.matchAll(/([^{}]+)\{([^{}]*)\}/g)) {
    const decls: Decls = {};
    for (const d of (m[2] as string).matchAll(/(--[a-z0-9-]+)\s*:\s*([^;]+);/g)) {
      decls[d[1] as string] = (d[2] as string).trim().replace(/\s+/g, " ");
    }
    out.push({ selector: (m[1] as string).trim().replace(/\s+/g, " "), decls });
  }
  return out;
}

export const selectorsOf = (css: string) => parseBlocks(css).map((b) => b.selector);

function declsOf(css: string, selector: string): Decls {
  return Object.assign(
    {},
    ...parseBlocks(css)
      .filter((b) => b.selector === selector)
      .map((b) => b.decls),
  );
}

/** Dichiarazioni (non risolte) di un tema in un modo, cascata inclusa. */
export function rawTokens(theme: Theme, mode: Mode): Decls {
  const prim = declsOf(PRIMITIVES_CSS, ":root");
  if (theme === "prototipo") {
    return {
      ...prim,
      ...declsOf(THEME_FILES.prototipo, ":root"),
      ...(mode === "dark" ? declsOf(THEME_FILES.prototipo, ".dark") : {}),
    };
  }
  // un tema diverso dal predefinito sostituisce per intero (e deve definirli tutti: vedi themes.test.ts)
  return {
    ...prim,
    ...declsOf(THEME_FILES[theme], `:root[data-theme="${theme}"]`),
    ...(mode === "dark" ? declsOf(THEME_FILES[theme], `:root.dark[data-theme="${theme}"]`) : {}),
  };
}

/** Come `rawTokens` ma con ogni `var(--x)` sostituito dal suo valore. */
export function resolveTokens(theme: Theme, mode: Mode): Decls {
  const raw = rawTokens(theme, mode);
  const resolve = (value: string, depth = 0): string => {
    if (depth > 10) throw new Error(`riferimento circolare: ${value}`);
    return value.replace(/var\((--[a-z0-9-]+)\)/g, (_, name: string) => {
      const target = raw[name];
      if (target === undefined) throw new Error(`token non definito: ${name}`);
      return resolve(target, depth + 1);
    });
  };
  return Object.fromEntries(Object.entries(raw).map(([k, v]) => [k, resolve(v)]));
}

export const rgb = (tokens: Decls, name: string): Rgba => {
  const v = tokens[name];
  if (v === undefined) throw new Error(`token mancante: ${name}`);
  return parseColor(v);
};

/** Le tre superfici del tema, rese opache (la rialzata e la sovrapposta stanno sopra la base). */
export function surfaces(tokens: Decls): Record<string, Rgba> {
  const base = over(rgb(tokens, "--isa-surface-base"), [255, 255, 255, 1]);
  return {
    base,
    raised: over(rgb(tokens, "--isa-surface-raised"), base),
    overlay: over(rgb(tokens, "--isa-surface-overlay"), base),
  };
}

export { contrast, over };
```

### `src/theme/__tests__/themes.test.ts`

147 righe

```ts
import { describe, expect, it } from "vitest";
import { parseColor } from "../color";
import {
  MODES,
  PRIMITIVES_CSS,
  THEMES,
  THEME_FILES,
  parseBlocks,
  rawTokens,
  resolveTokens,
  selectorsOf,
} from "./support";

describe("struttura dei file CSS", () => {
  it("le primitive stanno solo in :root e hanno solo nomi --isa-p-*", () => {
    expect(selectorsOf(PRIMITIVES_CSS)).toEqual([":root"]);
    for (const b of parseBlocks(PRIMITIVES_CSS)) {
      for (const name of Object.keys(b.decls)) expect(name).toMatch(/^--isa-p-[a-z0-9-]+$/);
    }
  });

  it("le primitive sono in OKLCH", () => {
    for (const [name, value] of Object.entries(parseBlocks(PRIMITIVES_CSS)[0]!.decls)) {
      expect(value, name).toMatch(/^oklch\(/);
    }
  });

  it("il tema predefinito usa solo :root e .dark; gli altri solo :root[data-theme] (e .dark)", () => {
    expect(selectorsOf(THEME_FILES.prototipo)).toEqual([":root", ".dark"]);
    expect(selectorsOf(THEME_FILES.notte)).toEqual([
      ':root[data-theme="notte"]',
      ':root.dark[data-theme="notte"]',
    ]);
  });
});

describe("completezza dei temi", () => {
  const protoLight = new Set(Object.keys(parseBlocks(THEME_FILES.prototipo)[0]!.decls));
  const protoDark = new Set(Object.keys(parseBlocks(THEME_FILES.prototipo)[1]!.decls));

  it("«notte» assegna ogni token che assegna il tema predefinito, nei due modi", () => {
    const [light, dark] = parseBlocks(THEME_FILES.notte);
    expect(Object.keys(light!.decls).sort()).toEqual([...protoLight].sort());
    // il blocco scuro ridefinisce tutto ciò che cambia col modo nel predefinito
    for (const name of protoDark) expect(dark!.decls, name).toHaveProperty(name);
  });

  for (const theme of THEMES) {
    for (const mode of MODES) {
      it(`${theme}/${mode}: ogni var() si risolve`, () => {
        expect(() => resolveTokens(theme, mode)).not.toThrow();
      });
    }
  }

  it("il tema predefinito ha i ruoli richiesti dal sistema", () => {
    const t = resolveTokens("prototipo", "light");
    const roles = [
      "surface-base",
      "surface-raised",
      "surface-overlay",
      "text",
      "text-secondary",
      "text-muted",
      "text-on-accent",
      "border",
      "border-strong",
      "accent",
      "accent-active",
      "accent-soft",
      "focus-ring",
      "op-filter",
      "op-filter-soft",
      "op-transform",
      "op-transform-soft",
      "op-merge",
      "op-merge-soft",
      "op-output",
      "op-output-soft",
      "dataset-fill",
      "warning",
      "error",
      "success",
      "radius-control",
      "radius-panel",
      "shadow-glass",
      "blur-glass",
      "duration-fast",
      "duration-base",
      "duration-slow",
    ];
    for (const r of roles) expect(t, r).toHaveProperty(`--isa-${r}`);
  });
});

describe("«notte»: i colori richiesti", () => {
  const near = (value: string, hex: string) => {
    const got = parseColor(value);
    const want = parseColor(hex);
    for (let i = 0; i < 3; i++) expect(Math.abs(got[i]! - want[i]!)).toBeLessThanOrEqual(1);
  };
  const t = resolveTokens("notte", "light");

  it("accento, collegamento, superfici, bordi, testo", () => {
    near(t["--isa-accent"]!, "#1e3a6e");
    near(t["--isa-text-link"]!, "#2563eb");
    near(t["--isa-surface-raised"]!, "#ffffff");
    near(t["--isa-surface-base"]!, "#f8fafc");
    near(t["--isa-border"]!, "#e2e8f0");
    near(t["--isa-text"]!, "#0f172a");
    near(t["--isa-text-secondary"]!, "#64748b");
  });

  it("famiglie di operazioni e avviso", () => {
    near(t["--isa-op-filter"]!, "#2563eb");
    near(t["--isa-op-transform"]!, "#7c3aed");
    near(t["--isa-op-merge"]!, "#ea580c");
    near(t["--isa-op-output"]!, "#16a34a");
    near(t["--isa-warning"]!, "#f59e0b");
  });

  it("ogni famiglia ha la propria tinta tenue, diversa dalle altre", () => {
    const softs = ["filter", "transform", "merge", "output"].map((f) => t[`--isa-op-${f}-soft`]);
    expect(new Set(softs).size).toBe(4);
  });

  it("nel tema predefinito le quattro famiglie coincidono (aspetto identico a prima)", () => {
    const p = resolveTokens("prototipo", "light");
    expect(
      new Set(["filter", "transform", "merge", "output"].map((f) => p[`--isa-op-${f}`])).size,
    ).toBe(1);
    expect(p["--isa-op-filter"]).toBe(p["--isa-tint-ink"]);
    expect(p["--isa-op-filter-soft"]).toBe(p["--isa-tint"]);
  });

  it("il tema predefinito non cambia i valori storici", () => {
    const l = rawTokens("prototipo", "light");
    expect(l["--isa-surface-base"]).toBe("#f5f3ee");
    expect(l["--isa-accent"]).toBe("#6c63ff");
    expect(l["--background"]).toBe("oklch(0.978 0.004 250)");
    expect(l["--primary"]).toBe("oklch(0.52 0.11 272)");
    const d = rawTokens("prototipo", "dark");
    expect(d["--background"]).toBe("oklch(0.19 0.008 260)");
    expect(d["--isa-surface-base"]).toBe("#17181d");
  });
});
```

