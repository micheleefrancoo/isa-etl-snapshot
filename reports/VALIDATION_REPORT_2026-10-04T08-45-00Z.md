# Validazione — Fase 6a.1: i pannelli orizzontali sottraggono altezza al canvas

Branch `feat/panels-fix`, da `main` (37cd559). Due commit: la baseline visiva
rigenerata (solo PNG, nessun codice) e la correzione.

## Causa confermata

`Dock.tsx` dava a `.ec-workspace` `height: calc(100% + <altezza dei pannelli in alto e in basso>)` (regola `fitWorkspace` del prototipo, righe 4819-4824, pensata per una pagina che scorre) dentro un contenitore alto quanto la finestra e con `overflow` nascosto o a scorrimento interno. L'area di lavoro superava così il contenitore: il pannello in basso finiva fuori finestra e, con il pannello in alto, il canvas scendeva e trascinava fuori minimappa e controlli di zoom.

Una seconda causa, emersa durante la correzione: il canvas aveva comunque un'altezza minima di 520 px (stile in linea dell'host in `EtlCanvas.tsx`, `canvas.css` e `min-h-[520px]` nella rotta), che gli impediva di restringersi e lo faceva sovrapporre al pannello (a 1440 × 900 il canvas restava a 520 invece di 516; a 1280 × 600 la pagina scorreva già senza pannelli).

## Prima e dopo (bounding box in px, asse y)

Finestra 1440 × 900, cassetta aperta:

| Bordo | Misura           | Prima                  | Dopo                  |
| ----- | ---------------- | ---------------------- | --------------------- |
| basso | area di lavoro   | 146→1106 (960)         | 146→884 (738)         |
| basso | pannello         | 884→1106, taglio a 900 | 662→884 (222), dentro |
| basso | canvas           | 146→884 (738)          | 146→662 (516)         |
| basso | minimappa / zoom | 768→872 / 834→872      | 546→650 / 612→650     |
| alto  | area di lavoro   | 146→1106 (960)         | 146→884 (738)         |
| alto  | canvas           | 368→1106 (738)         | 368→884 (516)         |
| alto  | minimappa / zoom | 990→1094 / 1056→1094   | 768→872 / 834→872     |

Finestra 1280 × 600:

| Bordo | Misura           | Prima             | Dopo              |
| ----- | ---------------- | ----------------- | ----------------- |
| basso | area di lavoro   | 146→888 (742)     | 146→584 (438)     |
| basso | pannello         | 666→888           | 362→584 (222)     |
| basso | canvas           | 146→666 (520)     | 146→362 (216)     |
| basso | minimappa / zoom | 550→654 / 616→654 | 246→350 / 312→350 |
| alto  | canvas           | 368→888 (520)     | 368→584 (216)     |
| alto  | minimappa / zoom | 772→876 / 838→876 | 468→572 / 534→572 |

Prima, con il pannello in alto, minimappa e zoom finivano oltre la finestra (1094 su 900; 876 su 600). Dopo, tutto sta sotto 884 (900) e sotto 584 (600).

## La correzione

- `layout.ts`: `viewCompensation` restituisce `{ dx, dy }` (sinistra compensa x, alto compensa y; destra e basso nulla); `compensate` aggiorna x e y; `workspaceExtra` rimosso; nuova costante `MIN_CANVAS_HEIGHT` (200). Commenti aggiornati.
- `actions.ts`: invia `setView` con x e y.
- `Dock.tsx` / `panels.css`: l'area di lavoro ha `height: 100%`, senza transizione di altezza (resta solo quella del pannello); riga centrale `minmax(var(--ec-canvas-min-h), 1fr)` con il valore da `MIN_CANVAS_HEIGHT`; il contenitore scorre (`overflow-y: auto`) sotto la soglia.
- `EtlCanvas.tsx`: proprietà `minHeight` (predefinito 520, invariato per il canvas isolato); lo spazio di lavoro passa 0. Nella rotta il contenitore del nuovo canvas non ha più `min-h-[520px]` (la rotta con `?canvas=v1` è invariata).
- Schede in alto o in basso: stessa regola, altezza maggiore dei due (già in `panelSize`).
- Annotato in `NOTE_DIVERGENZE.md` (§ 8).

## Esiti

| Controllo                       | Esito                                                                                                  |
| ------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `npx tsc --noEmit`              | nessun errore                                                                                          |
| `npm run lint`                  | 0 errori (14 avvisi preesistenti)                                                                      |
| `npm test`                      | 45 file, 820 test, tutti superati                                                                      |
| `npm run build`                 | riuscita                                                                                               |
| `node scripts/check-tokens.mjs` | 0 violazioni (debito preesistente invariato: 26)                                                       |
| `scripts/e2e-fase6a.mjs`        | 67 prove su 67                                                                                         |
| `scripts/e2e-fase5.mjs`         | 34 prove su 34                                                                                         |
| `visual-fase4` e `4b`           | `v2-*` identici alla baseline; il resto entro il rumore (0-356 px, anche i ritagli del solo prototipo) |

Prove nel browser (finestra 1440 × 900, bounding box confrontati con la finestra, non solo schermate): con la cassetta in basso, in alto e come scheda in basso, pannello, minimappa e controlli di zoom sono dentro la finestra, l'area di lavoro ha l'altezza del contenitore e la pagina non scorre; i nodi restano fermi sullo schermo prima e dopo l'apertura (basso, alto con `view.y` compensata, schede); con un pannello in alto e uno in basso il canvas resta ≥ 200 px. Finestra 1280 × 600 con un pannello in basso: tutto dentro; con due pannelli (alto e basso) scorre il contenitore e non la pagina.

Test unitari: `viewCompensation` (alto compensa y, basso/destra/sinistra non toccano y, alto e basso non toccano x), schede in alto e in basso con l'altezza maggiore e compensazione sulla differenza, `MIN_CANVAS_HEIGHT`, azioni con y.

## Schermate (`docs/visual/fase6a/`)

Rigenerate con gli stessi nomi; nuove: `cassetta-alto-e-basso.png` (un pannello in alto e uno in basso) e `cassetta-basso-finestra-bassa.png` (1280 × 600). I PNG di `fase4`, `fase4b` e `fase5` rigenerati per il confronto non sono stati committati.
