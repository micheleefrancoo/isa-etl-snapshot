# Validazione — Fase 6a: pannelli agganciabili e cassetta degli strumenti

Branch `feat/panels`, da `main` (a2dd012, allineato con `origin/main`).

## Esiti

| Controllo                       | Esito                                                                 |
| ------------------------------- | --------------------------------------------------------------------- |
| `npx tsc --noEmit`              | nessun errore                                                         |
| `npm run lint`                  | 0 errori (14 avvisi preesistenti di `react-refresh`)                  |
| `npm test`                      | 45 file, 814 test, tutti superati                                     |
| `npm run build`                 | riuscita                                                              |
| `node scripts/check-tokens.mjs` | 0 violazioni nei file controllati (debito preesistente invariato: 26) |
| `node scripts/e2e-fase6a.mjs`   | 36 prove su 36, nessun errore in console                              |
| Colori letterali nei file nuovi | nessuno (`grep` su `src/etl-canvas/panels/`)                          |

Test nuovi: `panels-layout`, `panels-actions`, `toolbox`, `toolbox-drop`.

## Correzioni fatte durante la validazione

1. `etl-canvas/contrast.ts` (`readBlock`) cercava la stringa esatta `.etl-canvas {`: con il selettore in lista (`.etl-canvas, .ec-workspace {` in `tokens.css`) `tokens.test.ts` falliva. Ora riconosce il selettore anche seguito da altri.
2. `panels.css`: gli angoli delle tacche usavano la scorciatoia `border-radius: 0 var(--ec-r-minimap) …`, in cui lo `0` letterale era segnalato da `check-tokens.mjs`. Ora sono proprietà per angolo, solo con token.
3. Prettier sulla riga CSS aggiunta a `visual-fase4.mjs` e `visual-temi.mjs`.

## Le voci della cassetta derivano dal catalogo

`Toolbox.tsx` costruisce sezioni e voci da `etl-core/catalog/operations.ts` (19 operazioni più Dataset), senza elenchi scritti a mano. `toolbox.test.tsx` («la cassetta deriva dal catalogo di etl-core») confronta una a una le voci della cassetta con le chiavi del catalogo, in entrambe le direzioni (nessuna mancante, nessuna inventata). L'e2e conferma nel browser: cinque sezioni nell'ordine del catalogo, 19 operazioni presenti.

## Canvas nudo: nessuna alterazione (visual-fase4 e 4b)

I due script non producono file identici nemmeno su `main` pulito: i PNG committati in `docs/visual/fase4/` differiscono già di migliaia di pixel da quelli rigenerati su `main` (`cavi-*`, `v2-*`: 6631-9787 px; anche i ritagli del solo prototipo variano di 4-90 px). Per questo il confronto è stato fatto tra `main` rigenerato e `feat/panels` rigenerata, nello stesso ambiente:

- `v2-chiaro` 2771 px, `v2-scuro` 447 px, `cavi-chiaro` 2790 px, `cavi-scuro` 455 px: la sola differenza visibile è la coppia di tacche nuove sui bordi sinistro e destro (UI nuova, a pannelli chiusi). Nodi, cavi, minimappa e comandi di zoom coincidono.
- I ritagli (`crop-*`) differiscono di 0-90 px (anche quelli del solo prototipo), il rumore tra due esecuzioni.
- Due esecuzioni consecutive su `feat/panels` danno `v2-*` identici (0 px).

I PNG di `docs/visual/fase4/` e `fase4b/` non sono stati committati: restano quelli già nel repository. `visual-fase4.mjs`, `visual-fase4b.mjs` e `visual-temi.mjs` chiudono la cassetta nello store prima dello scatto (il canvas nudo ha i pannelli chiusi) e disattivano la transizione.

## Schermate nuove (`docs/visual/fase6a/`)

`cassetta-sinistra`, `cassetta-destra`, `cassetta-alto`, `cassetta-basso` (fascia orizzontale in basso, anche `cassetta-basso-orizzontale`), `schede-stesso-bordo`, `trascinamento-dalla-cassetta` (anteprima visibile) e `trascinamento-dalla-cassetta-su-cavo`.

## Rimandato alla Fase 6b

- Contenuto dell'Inspector (campi, layout a tre colonne, selettori di valori, connettori logici): il guscio mostra solo il nome del nodo.
- I pulsanti di eliminazione ed espansione sul nodo (già rimandati nella Fase 5).
- Pulsante «Funzionalità»: non portato (Fase T).
- Baseline di `docs/visual/fase4/`: i PNG committati sono datati rispetto a `main` e andrebbero rigenerati in un commit a parte, quando si decide una baseline stabile.
