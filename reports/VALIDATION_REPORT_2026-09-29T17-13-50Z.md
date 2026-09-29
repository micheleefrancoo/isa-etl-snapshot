# Report di validazione — Fase di fondazione: un solo sistema di design

**Stato: PARZIALE, non fuso in `main`.** Due punti prescrivono di fermarsi e riportare prima di procedere (sotto: §A e §B). Tutto il resto è fatto e validato sul branch `feat/design-tokens-unify`.

## Esiti (sul branch)

| Controllo | Esito |
| --- | --- |
| `npx tsc --noEmit` | nessun errore |
| `npm run lint` | nessun errore (solo avvisi preesistenti) |
| `npm test` | 400/400 (30 file), come prima |
| `npm run build` | riuscita; nel CSS e negli asset costruiti nessun riferimento a Poppins o a Google Fonts; Manrope servito dai file dell'app |
| `node scripts/visual-fase4.mjs`, `node scripts/visual-fase4b.mjs` | riusciti, nessun errore né avviso in console |

## Fatto

### 1. Un solo carattere: Manrope
- `src/styles.css`: `@import "@fontsource-variable/manrope/wght.css"` e `--font-sans: "Manrope Variable", "Manrope", system-ui, sans-serif` al posto di Poppins.
- `src/routes/__root.tsx`: tolti i due `preconnect` e il foglio di stile Google Fonts di Poppins. Nessuna richiesta a Google Fonts da nessuna pagina.
- `src/etl-canvas/canvas.css`: non importa più il carattere (lo fa l'app).
- `grep -i poppins` fuori da documenti storici: nessuno. Restano riferimenti a `fonts.googleapis.com` solo in `scripts/visual-fase4*.mjs`, che intercettano la richiesta del **prototipo** (`docs/prototype/`, che li usa) e la servono da file locale: non sono dell'app.

### 2. Token: [REDATTO] dove il valore è identico
| Ruolo | Prima (`etl-canvas/tokens.css`) | Ora | Valore calcolato |
| --- | --- | --- | --- |
| Carattere | `"Manrope Variable", "Manrope", system-ui, sans-serif` (letterale) | `var(--font-sans)` dell'app | identico |
| Raggio dello stage | `20px` (letterale) | `calc(var(--radius) + 4px)` | 20 px, identico |
| Raggio della minimappa | `14px` (letterale) | `calc(var(--radius) - 2px)` | 14 px, identico |

Unica altra modifica di token: nell'app `--font-sans` da Poppins a Manrope.

### 3. Pulizia di Lovable (parte non condizionata)
Rimossi: la cartella `.lovable/` (`project.json`, `plan/…`), `src/lib/lovable-error-reporting.ts` e il suo uso in `__root.tsx` (l'`errorComponent` continua a fare `console.error` e a offrire «Try again»: la segnalazione andava solo all'editor di Lovable). Aggiornati `README.md` (tolte le sezioni su Lovable e «Poppins»), `AGENTS.md` (resta solo l'avvertenza generica di non riscrivere la cronologia pubblicata) e `scripts/generate-snapshot.mjs` (tolti i percorsi `.lovable/`).

## Identità visiva del canvas: confermata

Confronto pixel per pixel (`scripts/visual-compare.mjs`, nuovo, senza dipendenze) delle schermate rigenerate con quelle committate, sulla regione del canvas (l'intestazione dell'app cambia carattere, com'è giusto):

| Immagini | Esito |
| --- | --- |
| Prototipo (t0/t250/t500) | 0 pixel diversi |
| `v2-chiaro.png`, `v2-scuro.png` (4a, scena statica) | 0 pixel diversi |
| 4b: t250 chiaro e scuro, t500 scuro, movimento ridotto | 0 pixel diversi |
| 4b: t0 chiaro / scuro, t500 chiaro | 11 / 3 / 282 pixel su 1.039.104, scarto massimo 26 / 4 / 1 |
| ritagli 4a con il flusso (dataset, lavorazione, output, cavi) | pochi pixel: i ritagli includono un tratto di cavo con il tubo in movimento e l'orologio è reale |

Gli scarti restano nel rumore: due esecuzioni consecutive dello **stesso** codice differiscono di 6 / 2 / 280 / 15 pixel (scarto massimo 19), cioè i bordi antialias del tubo. Le misure esatte del DOM (`misure.json`, posizione, dimensione, colore, raggio, carattere di ogni nodo, cavo, controllo, minimappa e dei quattro tubi) sono **identiche** prima e dopo, a parte il carattere del corpo pagina (Poppins → Manrope) e un conteggio di frame (61 → 60). Per non generare differenze di cronologia le immagini che differiscono solo per rumore non sono state ricommitate; è stato aggiornato solo `docs/visual/fase4/misure.json` (non cita più Poppins).

## §A — Fermata: `@lovable.dev/vite-tanstack-config` NON è solo cosmetico

Non l'ho rimosso, come prescritto. `vite.config.ts` è una sola chiamata a `defineConfig` di quel pacchetto, che **costruisce l'intera configurazione di build**. Cosa fa prima di ogni build e avvio (versione 2.21.0):

| Funzione | Note |
| --- | --- |
| Catena di plugin | `@tailwindcss/vite`, `vite-tsconfig-paths` (`./tsconfig.json`), `@tanstack/react-start` (`tanstackStart`, con `importProtection`: errore se il client importa `**/server/**` o `server-only`), `@vitejs/plugin-react`, e in sviluppo i TanStack devtools |
| Deploy | plugin `nitro` **solo in build**, preset `cloudflare-module` (`nodeCompat: true`, `deployConfig: true`) se non se ne sceglie un altro |
| Alias e dipendenze | alias `@` → `./src`; `resolve.dedupe` di react, react-dom, jsx-runtime, react-query, query-core; `optimizeDeps.include` di react e react-dom |
| CSS | trasformatore `lightningcss` |
| Ambiente | `import.meta.env.VITE_*` iniettato con `define` |
| Server di sviluppo | `host "::"`, **porta 8080** |
| Solo dev | registrazione degli errori SSR e delle server function |
| Solo ambiente Lovable | bridge del server, «HMR gate», proxy degli asset, diagnostica di build, cartelle di output di pubblicazione: qui inattivi |

Dipendenze che il pacchetto porta con sé e che andrebbero dichiarate a mano: `@tanstack/devtools-vite` e `lightningcss`. `nitro`, `@tailwindcss/vite`, `vite-tsconfig-paths`, `@vitejs/plugin-react` e `@tanstack/react-start` sono già in `package.json`.

**Proposta**: sostituire il pacchetto con un `vite.config.ts` esplicito che riproduce solo la parte usata fuori da Lovable (plugin nell'ordine sopra, alias, dedupe, `optimizeDeps`, `lightningcss`, `define` di `VITE_*`, porta 8080, `nitro` in build con lo stesso preset), aggiungere `@tanstack/devtools-vite` e `lightningcss` alle dipendenze, e verificare che `npm run build`, `npm run dev` e i test diano gli stessi risultati (stessi file in `.output`, stesso CSS). Punto da decidere: il **preset di deploy Cloudflare**, oggi scelto in automatico dal pacchetto. Se la destinazione non è Cloudflare va scelto quello giusto.

## §B — Fermata: i token del canvas e quelli dell'app non coincidono

La consegna chiede di derivare dall'app i ruoli comuni **e** di lasciare identici i valori del canvas. Per i ruoli che contano le due richieste si escludono: l'app e il canvas usano due tavolozze diverse (app: grigi freddi con accento blu-grigio; prototipo: grigi caldi con accento viola). Derivare dall'app cambierebbe visibilmente il canvas, quindi non l'ho fatto.

| Ruolo | Canvas chiaro / scuro | App chiaro / scuro (OKLCH convertito in sRGB) |
| --- | --- | --- |
| Accento | `#6C63FF` / `#6C63FF` (testo `#A8A3FF`) | `--primary` ≈ `#5363A8` / ≈ `#8592DC` |
| Testo | `#262420` / `#F1F2F5` | `--foreground` ≈ `#1B2026` / ≈ `#F0F2F4` |
| Testo secondario | `#847E74` / `#A9ABB3` | `--muted-foreground` ≈ `#676C73` / ≈ `#ABAEB3` |
| Fondo pagina | `#F5F3EE` / `#17181D` | `--background` ≈ `#F6F8FA` / ≈ `#121417` |
| Ambra (avviso) | `#E0A23B` / `#E8B34F` | `--warning` ≈ `#DDA552` / ≈ `#E6B55D` |
| Superficie del vetro | `rgba(255,255,255,.92)` / `rgba(36,37,45,.92)` | `--glass-strong` bianco 66 % / bianco 10 % |
| Bordo del vetro | `rgba(38,36,32,.06)` / `rgba(255,255,255,.11)` | `--glass-border` bianco 55 % / bianco 11 % (uguale solo nel scuro) |
| Ombra del vetro | `0 10px 24px -14px …` | `--shadow-glass` `0 18px 45px -22px …` |
| Sfocatura del vetro | 16 px | 40 / 28 / 22 px |
| Raggi dei nodi | 22 / 26 px | scala 12–28 px senza 22 né 26 |

Due strade, da scegliere:
1. **Promuovere i valori del canvas a primitive condivise** in `styles.css` (per esempio `--isa-accent`, `--isa-ink`, …) e far derivare il canvas da quelle: il canvas non cambia al pixel; i token OKLCH dell'app restano com'erano finché non li si riporta sulle primitive nella fase di restyling, quando la tavolozza dell'app cambierà.
2. **Adottare i valori dell'app nel canvas**: una sola tavolozza subito, ma il canvas cambia (colori e ombre) e va deciso da chi ne cura il design.

Consiglio la 1: dà un'unica definizione per ogni valore del canvas senza rischio, e il restyling dell'app diventa un cambio di puntamento delle primitive.

## Cosa resta di Lovable (`grep -i lovable` fuori da questo report)
Solo `vite.config.ts` (commento e `import`) e `package.json` (dipendenza `@lovable.dev/vite-tanstack-config`), cioè §A. I riferimenti nei report di validazione precedenti sono cronologia.
