# 09-docs.md

File in questo blocco:

- `.lovable/plan/barra-risorse-etl-ancorata-al-canvas-2026-09-10.md`
- `AGENTS.md`
- `README.md`
- `roadmap.md`

---

### `.lovable/plan/barra-risorse-etl-ancorata-al-canvas-2026-09-10.md`

25 righe

```md
# Barra risorse ETL ancorata al canvas

## Obiettivo
Rendere la barra delle risorse parte strutturale dell’area ETL: può vivere solo sui quattro bordi, occupa spazio reale e non si sovrappone mai al workflow.

## Modifiche
- Eliminare la posizione libera e qualsiasi posizionamento assoluto della barra sopra il canvas.
- Gestire quattro agganci consentiti: alto, basso, sinistra, destra.
- Durante il trascinamento, determinare il bordo più vicino e agganciare lì la barra, mantenendola sempre entro il perimetro ETL.
- Adattare automaticamente l’orientamento:
  - orizzontale in alto o in basso;
  - verticale a sinistra o a destra.
- Fare della barra e del canvas due aree adiacenti nello stesso layout, così la barra riduce sempre lo spazio disponibile al canvas.
- All’apertura di una categoria o di tutte le risorse, mostrare un secondo livello interno all’area della barra, mai come menu o livello sovrapposto.
- Quando la barra orizzontale superiore si espande, il canvas sottostante si abbassa e tutte le card si spostano visivamente con esso senza alterare le coordinate salvate.
- Quando la barra laterale si espande, il canvas si restringe lateralmente; le card restano contenute grazie ai limiti già presenti.
- Conservare aggiunta con click/drag, categorie, stato espanso/ridotto e stile isa esistente.

## Dettagli tecnici
- Lo stato di aggancio sarà controllato dal contenitore del workflow.
- Il layout userà una struttura flex ordinata in base al bordo scelto, senza `absolute` per la palette.
- Il drag della barra sarà limitato al rettangolo dell’area ETL e al rilascio selezionerà uno dei quattro bordi.
- Saranno rimossi il comando e il tipo `free`.
- Verranno verificati orientamento, espansione, aggiunta risorse e contenimento delle card nel canvas.
```

### `AGENTS.md`

11 righe

```md
<!-- LOVABLE:BEGIN -->
> [!IMPORTANT]
> This project is connected to [Lovable](https://lovable.dev). Avoid rewriting
> published git history — force pushing, or rebasing/amending/squashing commits
> that are already pushed — as it rewrites history on Lovable's side and the
> user will likely lose their project history.
>
> Commits you push to the connected branch sync back to Lovable and show up in
> the editor, so keep the branch in a working state.
<!-- LOVABLE:END -->
```

### `README.md`

37 righe

```md
# Isa Glass Platform

Crea la piattaforma di calcolo "isa" usando il design system glassmorphism fornito:
- Tema glassmorphism light e dark completo (sfondi con blob colorati sfocati, pannelli e chip con backdrop-blur, gradienti)
- Logo personalizzato SVG "isa"
- Sidebar di navigazione con sezioni primarie (Solutions, Shared with me, Favorites, Templates, Activity, Trash) e secondarie (Teams, Users, Settings), oltre al profilo utente
- Header con saluto utente, toggle dark/light mode, barra di ricerca, notifiche e pulsante "New solution"
- Sezione principale "Solutions" con grid di card di soluzione (con anteprime dinamiche/grafici a barre o linee, badge di stato Draft/Live, data aggiornamento, eliminazione) e card per creare nuova soluzione
- Modale per creare una nuova soluzione
- Pannello laterale widget modulare e collassabile/trascinabile con:
  - Team Chat interattiva con selezione del team (Fleet Operations, Financial Planning, Inventory Manager)
  - Recent Activity
  - Batch Jobs con barre di progresso calcolo
- Predisponi le viste/rotte future per la piattaforma di calcolo: dettaglio calcolo/soluzione con configuratore parametri, esecuzione job e visualizzazione dati.
Includi Lucide icons e Poppins come font.

This project was built with [Lovable](https://lovable.dev).

## Build with Lovable

Continue developing this project in the [Lovable editor](https://lovable.dev/projects/2a10b01f-cdfc-4c19-93f0-8d63f283723f).

- **Ship faster**: describe what you want to build and Lovable handles the code.
- **Stay in sync**: every change made in Lovable is committed straight to this repository.
- **Full ownership**: this code is yours. Push to `main` on GitHub and your changes sync back into Lovable, ready for your next prompt.

## Development

Prefer working locally? You need Node.js and npm — [install with nvm](https://github.com/nvm-sh/nvm#installing-and-updating).

```sh
git clone <this-repository-url>
cd <repository-name>
npm i
npm run dev
```
```

### `roadmap.md`

27 righe

```md
# Roadmap isa

## In corso — Visual ETL Workspace (/solutions/$solutionId/etl)
- [x] Primitivi condivisi riusabili: IsaMenu / IsaMenuItem / IsaModal (estratti dai pattern esistenti)
- [x] Catalogo nodi (Sources, Transform, Combine, Aggregate, Output, Model/Future)
- [x] Stato workflow per soluzione (nodi, edge, config)
- [x] Tool Palette strutturale e riposizionabile sui quattro bordi, con secondo livello che ridimensiona il canvas senza sovrapporsi
- [x] Canvas full-screen con toggle griglia, drag nodi, porte in/out, connessioni ortogonali 90° (Manhattan)
- [x] Inspector contestuale con campi per tipo di nodo
- [x] Inspector e Data Preview come cassetti a comparsa con chiusura a X
- [x] Card nodo minimali + menu "Visualizza" con spunte (metriche, sorgente, formule, stato)
- [x] Top bar workspace: indietro, breadcrumb, saved, undo/redo, moduli, Run Workflow
- [x] Zoom in / out / fit view
- [x] Stati nodo (Ready/Running/Succeeded/Error) con esecuzione simulata a catena
- [x] Metadati volume su nodi e hover sugli edge
- [x] Empty state "Build your data workflow" con "+ Add dataset"
- [x] Nessun workflow finto pre-popolato: ogni nuova soluzione parte da canvas vuoto
- [x] Schema Evolution nell'Inspector

## Vincoli permanenti
- Continuità del design system: riusare i pattern isa esistenti, non creare varianti locali
- Non toccare la Home, Solutions, Solution Detail e le altre route
- Nessun backend / execution engine in questa fase
- [ ] Drag palette: cerchio sotto il puntatore (portale fuori dal canvas scalato) + animazione compressione/espansione visibile
- [ ] Switch tema dark/light visibile anche nel workspace ETL (topbar)
- [ ] Griglia a puntini del canvas visibile anche in light mode
```

