# 09-docs.md

File in questo blocco:

- `AGENTS.md`
- `README.md`
- `roadmap.md`

---

### `AGENTS.md`

5 righe

```md
> [!IMPORTANT]
> Evita di riscrivere la cronologia git già pubblicata (force push, rebase,
> amend o squash di commit già pushati). Tieni sempre il branch principale in
> uno stato funzionante.
```

### `README.md`

27 righe

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
Icone Lucide e font Manrope (self-hosted, senza richieste a server esterni).

## Development

Servono Node.js e npm ([installazione con nvm](https://github.com/nvm-sh/nvm#installing-and-updating)).

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

