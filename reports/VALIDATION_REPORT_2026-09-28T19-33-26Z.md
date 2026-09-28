# VALIDATION REPORT — Inventario canvas ETL vs. prototipo

**Timestamp:** 2026-09-28T19:33:26Z
**Scope:** solo lettura di `src/**`, `docs/prototype/isa-fusion-prototype.html`, `src/canvas/.reports/**`. Nessuna modifica a codice applicativo. Unico artefatto prodotto: `docs/inventory/INVENTARIO_2026-09-28T19-29-07Z.md`.

## 1. Esiti identici a STATUS.md (nulla è stato modificato in `src/`)

```
npx tsc --noEmit    → PASS, 0 errori (identico all'ultimo STATUS.md registrato)
npm run lint        → FALLITO, 2040 errori + 14 warning (identico)
npm test            → 41/41 test passati, 4 file (identico)
```

Confermato: nessun file sotto `src/` è stato letto in modo distruttivo né modificato durante la produzione del rapporto — solo `Read`/`Grep`/`Bash` (grep, wc, find) in modalità lettura. `git status --porcelain` prima e dopo questa sessione mostra le stesse modifiche pre-esistenti su `src/**` (nessuna nuova), più i due nuovi file sotto `docs/`.

## 2. Metodo

Nessun subagent "fork" è stato usabile in questa sessione (il tool ha rifiutato la delega con "Fork is not available inside a forked worker"); l'intera ricognizione è stata svolta direttamente con `Read`/`Bash`/`Grep` sui file elencati nell'incarico: tutto `src/canvas/**`, tutto `src/components/isa/etl/**` (incluso `workflow-canvas.tsx` per intero, 5037 righe, letto in più passate con offset/limit), `src/lib/etl-*.ts(x)`, `src/lib/solutions-store.tsx`, `src/lib/modules.ts`, `src/lib/theme.tsx` (parziale), i file di rotta (`__root.tsx`, `router.tsx`, `solutions.$solutionId.tsx`, `solutions.$solutionId.etl.tsx`), `src/styles.css`, `package.json`, i 4 file di test, i 3 report precedenti in `src/canvas/.reports/`, e `docs/prototype/isa-fusion-prototype.html` per intero (5097 righe, letto a campioni mirati via grep + lettura di intorno alle righe trovate — non riga per riga in sequenza, data la dimensione).

## 3. Cosa NON è stato letto per intero (dichiarato nel rapporto, §9 Domande aperte)

- `src/server.ts` (61 righe) — solo dedotto dal nome/convenzioni, non letto.
- `src/components/isa/etl/settings-panels/combine-panel.tsx` (321 righe) — solo le prime ~20 righe lette.
- `src/lib/etl-catalog.ts` (427 righe) — letto parzialmente (header + porzione con i 19 tipi di nodo), non ogni campo di ogni nodo.
- Il comportamento a runtime (browser) non è stato osservato: nessun `npm run dev` avviato, per restare strettamente in "solo lettura" senza side-effect su `localStorage` o sullo stato del workflow.

## 4. Verifica incrociata delle affermazioni marcate "(dedotto)"

Ogni affermazione nel rapporto che non deriva da una lettura diretta di codice o dall'esecuzione di un comando è marcata `(dedotto)` nel testo — 5 occorrenze in `INVENTARIO_2026-09-28T19-29-07Z.md` (nome del file prototipo, comportamento di `pipelineOrder` su un ciclo non testato, dettaglio di `pickCombineCandidate` non letto per intero, mancata verifica pixel-per-pixel del box combinato, natura Context+useState vs useReducer di `solutions-store.tsx`).

## 5. Esito

- [x] Type check, lint, test identici a STATUS.md
- [x] Nessuna modifica a `src/**`
- [x] Rapporto scritto in `docs/inventory/INVENTARIO_2026-09-28T19-29-07Z.md`
- [x] Ogni deduzione marcata `(dedotto)`
- [x] Nessun piano di implementazione proposto — solo fotografia dello stato e dei divari
- [ ] `./scripts/sync-snapshot.sh` — eseguito **dopo** questo report (vedi messaggio finale in chat per il link INDEX.md aggiornato)
