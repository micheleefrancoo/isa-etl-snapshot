# Report di validazione — Fondazione, completamento (§A e §B risolti)

**Stato: COMPLETO** sul branch `feat/design-tokens-unify`; decisioni confermate: preset di deploy `cloudflare-module`, token secondo l'opzione 1.

## Esiti

| Controllo | Esito |
| --- | --- |
| `npx tsc --noEmit` | nessun errore |
| `npm run lint` | 0 errori (14 avvisi preesistenti) |
| `npm test` | 400/400 (30 file), stesso totale di prima |
| `npm run build` | riuscita; stessi 93 file in `.output` con gli stessi hash, salvo `nitro.json` (data) e `server/index.mjs` (solo `mtime` e ordine delle voci del manifest degli asset: stessi percorsi, dimensioni ed etag) |
| `npm run dev` | risponde 200 su `http://localhost:8080/` |
| `node scripts/visual-fase4.mjs`, `visual-fase4b.mjs` | riusciti, nessun errore né avviso in console |

## §A — `@lovable.dev/vite-tanstack-config` rimosso
- `vite.config.ts` è ora esplicito e riproduce la parte usata fuori da Lovable: plugin nell'ordine devtools (solo `mode === "development"`), `@tailwindcss/vite`, `vite-tsconfig-paths`, `tanstackStart` (entry `server` e `importProtection`), `nitro` (solo build), `@vitejs/plugin-react`; alias `@`, `dedupe`, `optimizeDeps`, trasformatore `lightningcss`, `define` di `VITE_*`, `server` host `::` porta 8080 con debounce del watcher, e in `build --mode development` `process.env.NODE_ENV` = development per il client.
- **Deploy**: preset `cloudflare-module` scritto esplicitamente (`nodeCompat`, `deployConfig`), attivo solo in build. Prima era il preset predefinito del pacchetto; ora una variabile `NITRO_PRESET` non lo sostituisce più.
- `package.json`: tolto `@lovable.dev/vite-tanstack-config`; aggiunte come dipendenze dirette (dev) `@tanstack/devtools-vite` ^0.8.5 e `lightningcss` ^1.33.0.
- Non riprodotti (attivi solo nell'ambiente Lovable o solo diagnostica): bridge del server, HMR gate, proxy degli asset, diagnostica di build, cartelle di pubblicazione, logger degli errori SSR/server function in sviluppo, `esbuild.keepNames` (Vite 8 non usa esbuild: l'opzione non è nei tipi).
- Al termine `grep -i lovable` fuori dai report di validazione (cronologia): nessun risultato.

## §B — Token: [REDATTO] 1 (primitive condivise)
- `src/styles.css`: nuove primitive `--isa-*` in `:root` (tema chiaro) e `.dark` (scuro), con esattamente i valori che il canvas aveva in `etl-canvas/tokens.css` (colori, ombra, e per i temi indipendenti raggi dei nodi 22/26 px e sfocatura 16 px).
- `src/etl-canvas/tokens.css`: ogni `--ec-*` con valore proprio è ora `var(--isa-…)`; i valori derivati dal raggio e dal carattere dell'app (fase precedente) restano. Restano locali solo `--ec-output-opacity` e i derivati `--ec-r-*`.
- I token OKLCH dell'app **non sono stati toccati**: verranno riportati sulle primitive nel restyling.
- `contrast.ts` (`readTokens`) risolve le `var(--isa-*)` con le primitive di `styles.css`; i test dei token e del contrasto WCAG continuano a leggere i valori effettivi.

## Identità visiva del canvas

Confronto pixel per pixel con `scripts/visual-compare.mjs` delle schermate rigenerate con quelle committate, sulla regione del canvas (`16,146,1408,738`, 1.039.104 pixel):

| Immagini | Esito |
| --- | --- |
| `fase4/v2-chiaro.png`, `v2-scuro.png` | 0 pixel diversi |
| 4b: movimento ridotto, t250 e t500 (chiaro e scuro) | 0 pixel diversi |
| 4b: t0 chiaro / scuro | 12 / 11 pixel, scarto massimo 26 (bordi antialias del tubo in movimento, come nel rumore già misurato) |
| Prototipo (`crop-*-prototipo.png`) | 0 pixel diversi; `cavi-prototipo.png` 218 (rumore del prototipo stesso, non toccato) |

Le misure del DOM (`misure.json`: posizione, dimensione, colore, raggio, carattere di ogni elemento) sono **identiche**, salvo un conteggio di frame (61 → 60). Le immagini rigenerate non sono state ricommitate (differenze solo di rumore o di intestazione).
