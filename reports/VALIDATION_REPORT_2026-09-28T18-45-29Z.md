# VALIDATION_REPORT — ricostruzione snapshot isa-etl-snapshot

Generato: 2026-09-28T18:45:29Z (UTC)

Contesto: ricostruzione completa dello snapshot pubblico
`micheleefrancoo/isa-etl-snapshot` con nuova struttura (INDEX.md, STATUS.md,
ENV.md, TREE.md, `files/NN-<area>.md`, `reports/`), sostituendo i vecchi
`isa-snapshot.md` e `isa-snapshot-workflow-canvas.md` con rimandi a
INDEX.md. Nessun codice applicativo è stato modificato: solo
`scripts/sync-snapshot.sh` (riscritto) e due nuovi helper,
`scripts/generate-snapshot.mjs` e `scripts/generate-index.mjs`.

Commit finale verificato del repository snapshot:
`583ebbd0501662be871f28263bc5d57176e2444f` (contenuto) +
`3398dc34a85b51c2e2559385c74642b0b0ef45d8` (INDEX.md).

## 1. Doppia esecuzione consecutiva

`./scripts/sync-snapshot.sh` è stato eseguito due volte di fila (via
`bash`, redirigendo l'output su file):

- Esecuzione 1: exit code 0. Commit content `fa3a5eaa5d741dfafb0293dc94f693c6c147b4e8`, commit INDEX.md `1873f8056063e48d3672213eb2f70ddd150ba714`.
- Esecuzione 2: exit code 0. Commit content `583ebbd0501662be871f28263bc5d57176e2444f`, commit INDEX.md `3398dc34a85b51c2e2559385c74642b0b0ef45d8`.

Nessun errore in entrambe le esecuzioni. **Esito: PASS.**

Nota: la seconda esecuzione produce comunque un nuovo commit (non è un
no-op) perché STATUS.md e INDEX.md incorporano il timestamp di
generazione e la durata dei check, che cambiano ad ogni run; questo è
atteso e non è un errore.

## 2. Verifica HTTP (curl) di ogni URL in INDEX.md

Tutti i 20 URL elencati nell'INDEX.md pinnato al commit
`583ebbd0501662be871f28263bc5d57176e2444f` sono stati verificati con
`curl -s -o /dev/null -w "%{http_code}"`: **20/20 → 200**.

```
200  .../583ebbd0.../ENV.md
200  .../583ebbd0.../STATUS.md
200  .../583ebbd0.../TREE.md
200  .../583ebbd0.../files/01-canvas.md
200  .../583ebbd0.../files/02-isa-etl-a.md
200  .../583ebbd0.../files/02-isa-etl-b.md
200  .../583ebbd0.../files/02-isa-etl-c.md
200  .../583ebbd0.../files/02-isa-etl-d.md
200  .../583ebbd0.../files/03-components-a.md
200  .../583ebbd0.../files/03-components-b.md
200  .../583ebbd0.../files/03-components-c.md
200  .../583ebbd0.../files/03-components-d.md
200  .../583ebbd0.../files/04-lib-hooks-store-a.md
200  .../583ebbd0.../files/04-lib-hooks-store-b.md
200  .../583ebbd0.../files/05-app-pages.md
200  .../583ebbd0.../files/06-styles.md
200  .../583ebbd0.../files/08-scripts-config.md
200  .../583ebbd0.../files/09-docs.md
200  .../583ebbd0.../reports/VALIDATION_REPORT_2026-09-19T10-24-53Z.md
200  .../583ebbd0.../reports/VALIDATION_REPORT_2026-09-19T11-38-16Z.md
```

Nota sul link "su main" (`.../main/INDEX.md`, non pinnato): risponde 200
ma può servire per qualche minuto una versione leggermente precedente a
causa della cache CDN di raw.githubusercontent.com — comportamento noto e
non un difetto della generazione. Gli URL pinnati allo SHA sono
immutabili e non soggetti a questo effetto una volta popolata la cache.

**Esito: PASS.**

## 3. Dimensione di ogni blocco (limite: 60 000 byte)

| Blocco | Byte | KB |
|---|---|---|
| files/09-docs.md | 6 342 | 6.3 |
| files/06-styles.md | 11 705 | 11.7 |
| files/02-isa-etl-d.md | 27 040 | 27.0 |
| files/04-lib-hooks-store-b.md | 28 139 | 28.1 |
| files/08-scripts-config.md | 37 503 | 37.5 |
| files/04-lib-hooks-store-a.md | 49 191 | 49.2 |
| files/03-components-d.md | 49 194 | 49.2 |
| files/02-isa-etl-b.md | 49 916 | 49.9 |
| files/02-isa-etl-c.md | 49 923 | 49.9 |
| files/03-components-b.md | 55 294 | 55.3 |
| files/02-isa-etl-a.md | 55 467 | 55.5 |
| files/03-components-c.md | 55 545 | 55.5 |
| files/01-canvas.md | 55 577 | 55.6 |
| files/05-app-pages.md | 57 241 | 57.2 |
| files/03-components-a.md | 58 975 | 59.0 |

Massimo: 58 975 B, sotto il limite di 60 000 B ("60 KB" trattato come
60 000 byte decimali, per evitare ambiguità con KiB). Nessun blocco supera
il limite. **Esito: PASS.**

## 4. Copertura file → blocchi (uno e un solo blocco)

Verifica automatica (script Node ad-hoc sul `manifest.json` pubblicato
nel commit finale): 141 file in-scope raccolti, 141 file scritti in
`files/`. Un solo file compare in più di un blocco:
`src/components/isa/etl/workflow-canvas.tsx`, diviso in 3 parti
consecutive (`02-isa-etl-b.md`, `02-isa-etl-c.md`, `02-isa-etl-d.md`)
perché supera da solo la soglia di split — comportamento esplicitamente
previsto dalle istruzioni ("un singolo file che da solo supera 60 KB va
diviso in parti consecutive"). Riconcatenando le 3 parti si riottiene
byte-per-byte il file originale (verificato programmaticamente: lunghezza
e contenuto identici). Tutti gli altri 140 file compaiono in esattamente
un blocco. **Esito: PASS.**

## 5. Segreti

Scansione (token GitHub/OpenAI/Anthropic/AWS/Slack, chiavi private PEM,
URL con credenziali, assegnazioni a chiavi password/secret/api_key/token)
eseguita su tutti i file candidati prima della scrittura. Nessun segreto
trovato: `redactions: 0`. Verifica indipendente con `grep -rnE` sui file
effettivamente pubblicati: nessun match. `.env*` esclusi per policy fissa
(nessuno presente nel repository comunque). **Esito: PASS (nessuna
redazione necessaria).**

## 6. File esclusi dallo snapshot (con motivo)

| File | Motivo |
|---|---|
| `.claude/scheduled_tasks.lock` | fuori dall'ambito definito (non è src/**, scripts/**, doc o config di radice); listato solo in TREE.md |
| `bun.lock` | lockfile — solo nome/dimensione in TREE.md |
| `package-lock.json` | lockfile — solo nome/dimensione in TREE.md |
| `public/favicon.ico` | binario — solo in TREE.md |
| `public/robots.txt` | fuori dall'ambito definito (non è src/**, scripts/**, doc o config di radice); listato solo in TREE.md |

Directory escluse a priori (mai scansionate): `node_modules`, `.git`,
`dist`, `build`, `coverage`, `.output`, `.wrangler`, `.pw-tmp`, `.nitro`,
`.vinxi`. Nessun file `.env*` presente nel repository sorgente.

`src/canvas/.reports/*.md` non sono duplicati in `files/`: sono copiati
in `reports/` come richiesto, e non contati come "esclusi".

## Riepilogo STATUS.md (fotografia, nessuna correzione applicata)

- Type check (`npx tsc --noEmit`, nessuno script dedicato in
  package.json): **OK**.
- Lint (`npm run lint`): **FALLITO** — 2040 errori (quasi tutti
  `prettier/prettier`, formattazione) + 14 warning
  `react-refresh/only-export-components`. Nessuna correzione applicata,
  come richiesto: è una fotografia dello stato attuale.
- Test (`npm test` / `vitest run`): **OK** — 41 test passati su 4 file,
  0 falliti.
- Build (`npm run build`): **OK**.

Dettaglio completo (comandi, durate, ultime 60 righe di output) in
STATUS.md nello snapshot pubblicato.

## Struttura finale pubblicata

15 blocchi in `files/`, 141 file sorgente coperti, 2 report in
`reports/`, totale `files/` ≈ 647 KB. `isa-snapshot.md` e
`isa-snapshot-workflow-canvas.md` sostituiti da un breve rimando a
INDEX.md (link stabile su `main`, non pinnato, così i link già
esistenti restano validi anche dopo run futuri).
