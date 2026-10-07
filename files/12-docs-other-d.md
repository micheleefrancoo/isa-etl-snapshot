# 12-docs-other-d.md

File in questo blocco:

- `docs/visual/fase6b2/misure.json`

---

### `docs/visual/fase6b2/misure.json` (parte 1/2)

2057 righe totali

```json
{
  "risultati": [
    {
      "prova": "1440 right filtro: tre condizioni create con eventi veri (regione = Nord, importo > 100, stato = Chiuso)",
      "esito": "ok",
      "dettaglio": "[[\"regione\",\"=\",[\"Nord\"],\"\"],[\"importo\",\">\",[],\"100\"],[\"stato\",\"=\",[\"Chiuso\"],\"\"]]"
    },
    {
      "prova": "1440 right filtro: ogni condizione ha un connettore tra sé e la precedente, nessun selettore globale E/O",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right filtro: anteprima senza gruppi, da sinistra a destra",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) AND stato = Chiuso"
    },
    {
      "prova": "1440 right filtro: «Raggruppa» sulle ultime due crea un gruppo (riquadro «Gruppo» con «Sciogli»)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right filtro: dopo aver raggruppato il focus non si perde (resta sul pulsante del connettore)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right connettore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1124,231 300×306"
    },
    {
      "prova": "1440 right filtro: il connettore interno diventa OR e l'anteprima è «regione = Nord AND (importo > 100 OR stato = Chiuso)»",
      "esito": "ok",
      "dettaglio": "regione = Nord AND (importo > 100 OR stato = Chiuso)"
    },
    {
      "prova": "1440 right filtro: le ultime due condizioni stanno nello stesso gruppo, la prima no",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right filtro: scrivere nel campo non fa perdere il focus",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right filtro: un solo annullamento riporta il campo a prima della digitazione",
      "esito": "ok",
      "dettaglio": "250 → 100"
    },
    {
      "prova": "1440 right filtro: «Sciogli» toglie il gruppo, le condizioni restano e l'anteprima segue",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1440 right filtro: raggruppa di nuovo (prime due) e l'anteprima porta le parentesi sul gruppo",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1440 right colonna: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1150,444 240×411"
    },
    {
      "prova": "1440 right operatore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1150,16 240×470"
    },
    {
      "prova": "1440 right valori: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1150,297 240×266"
    },
    {
      "prova": "1440 right connettore esterno: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1124,317 300×306"
    },
    {
      "prova": "1440 right filtro: «+ Condizione nel gruppo» aggiunge una voce dentro il gruppo",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right filtro: un gruppo che resta con una sola condizione si scioglie da sé",
      "esito": "ok",
      "dettaglio": "[null,null]"
    },
    {
      "prova": "1440 right canvas: i nodi richiesti restano dentro l'area con l'Inspector aperto",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right join: colonna = colonna (cliente = cliente) con eventi veri",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"col\",\"lval\":\"\",\"rval\":\"\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1440 right join: con una colonna = colonna non c'è l'avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right join: le colonne del lato sinistro vengono dalla tabella sinistra, quelle del destro dalla destra",
      "esito": "ok",
      "dettaglio": "sinistra: id|cliente|regione|categoria|stato|quantita|importo|data — destra: cliente|agente|sconto"
    },
    {
      "prova": "1440 right join: solo disuguaglianze → avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right join: tornando a «=» l'avviso sparisce",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right join: lato destro «Valore» con i valori della colonna dell'altro lato (cliente = “Acme”)",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"val\",\"lval\":\"\",\"rval\":\"Acme\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1440 right join: colonna = valore non è un'uguaglianza tra colonne → avviso",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right join: con una lista a destra il confronto passa a «è uno di»",
      "esito": "ok",
      "dettaglio": "[\"list\",\"è uno di\"]"
    },
    {
      "prova": "1440 right join: lista di valori dal dominio della colonna dell'altro lato (Acme, Borealis)",
      "esito": "ok",
      "dettaglio": "[\"Acme\",\"Borealis\"]"
    },
    {
      "prova": "1440 right join confronto: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1150,637 240×146"
    },
    {
      "prova": "1440 right join valori della lista: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1150,294 240×276"
    },
    {
      "prova": "1440 right join: uscendo dalla lista il confronto torna «=»",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right join: freccia destra sul radiogroup sceglie «Valore» e sposta il focus (tabindex 0 solo sulla scelta)",
      "esito": "ok",
      "dettaglio": "{\"on\":\"Valore\",\"focused\":true,\"tab\":\"0\"}"
    },
    {
      "prova": "1440 right join: Fine sceglie «Lista»",
      "esito": "ok",
      "dettaglio": "Lista"
    },
    {
      "prova": "1440 right join: Home sceglie «Colonna»",
      "esito": "ok",
      "dettaglio": "Colonna"
    },
    {
      "prova": "1440 right join: il tipo di join si sceglie dalle impostazioni (valori del catalogo)",
      "esito": "ok",
      "dettaglio": "left"
    },
    {
      "prova": "1440 right layout: sul bordo laterale le parti si impilano con le righe comprimibili (nessuna tre colonne)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right ordina: una maniglia per criterio, con nome accessibile",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right ordina: Alt+↓ sposta il primo criterio in seconda posizione",
      "esito": "ok",
      "dettaglio": "importo,regione,stato"
    },
    {
      "prova": "1440 right ordina: il focus segue la riga spostata e lo spostamento è annunciato",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 right ordina: Alt+↑ lo riporta in prima posizione",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 right ordina: annullare uno spostamento da tastiera lo riporta com'era, in un passo",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 right ordina: trascinare la maniglia del primo criterio in fondo lo porta per ultimo",
      "esito": "ok",
      "dettaglio": "importo,stato,regione"
    },
    {
      "prova": "1440 right ordina: un solo annullamento riporta il trascinamento com'era",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 right ordina: Esc durante il trascinamento lo annulla",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 bottom filtro: tre condizioni create con eventi veri (regione = Nord, importo > 100, stato = Chiuso)",
      "esito": "ok",
      "dettaglio": "[[\"regione\",\"=\",[\"Nord\"],\"\"],[\"importo\",\">\",[],\"100\"],[\"stato\",\"=\",[\"Chiuso\"],\"\"]]"
    },
    {
      "prova": "1440 bottom filtro: ogni condizione ha un connettore tra sé e la precedente, nessun selettore globale E/O",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom filtro: anteprima senza gruppi, da sinistra a destra",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) AND stato = Chiuso"
    },
    {
      "prova": "1440 bottom filtro: «Raggruppa» sulle ultime due crea un gruppo (riquadro «Gruppo» con «Sciogli»)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom filtro: dopo aver raggruppato il focus non si perde (resta sul pulsante del connettore)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom connettore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "455,517 300×306"
    },
    {
      "prova": "1440 bottom filtro: il connettore interno diventa OR e l'anteprima è «regione = Nord AND (importo > 100 OR stato = Chiuso)»",
      "esito": "ok",
      "dettaglio": "regione = Nord AND (importo > 100 OR stato = Chiuso)"
    },
    {
      "prova": "1440 bottom filtro: le ultime due condizioni stanno nello stesso gruppo, la prima no",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom filtro: scrivere nel campo non fa perdere il focus",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom filtro: un solo annullamento riporta il campo a prima della digitazione",
      "esito": "ok",
      "dettaglio": "250 → 100"
    },
    {
      "prova": "1440 bottom filtro: «Sciogli» toglie il gruppo, le condizioni restano e l'anteprima segue",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1440 bottom filtro: raggruppa di nuovo (prime due) e l'anteprima porta le parentesi sul gruppo",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1440 bottom colonna: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,263 225×411"
    },
    {
      "prova": "1440 bottom operatore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "934,168 225×506"
    },
    {
      "prova": "1440 bottom valori: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1178,408 225×266"
    },
    {
      "prova": "1440 bottom connettore esterno: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "460,371 300×306"
    },
    {
      "prova": "1440 bottom filtro: «+ Condizione nel gruppo» aggiunge una voce dentro il gruppo",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom filtro: un gruppo che resta con una sola condizione si scioglie da sé",
      "esito": "ok",
      "dettaglio": "[null,null]"
    },
    {
      "prova": "1440 bottom canvas: i nodi richiesti restano dentro l'area con l'Inspector aperto",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom join: colonna = colonna (cliente = cliente) con eventi veri",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"col\",\"lval\":\"\",\"rval\":\"\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1440 bottom join: con una colonna = colonna non c'è l'avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom join: le colonne del lato sinistro vengono dalla tabella sinistra, quelle del destro dalla destra",
      "esito": "ok",
      "dettaglio": "sinistra: id|cliente|regione|categoria|stato|quantita|importo|data — destra: cliente|agente|sconto"
    },
    {
      "prova": "1440 bottom join: solo disuguaglianze → avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom join: tornando a «=» l'avviso sparisce",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom join: lato destro «Valore» con i valori della colonna dell'altro lato (cliente = “Acme”)",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"val\",\"lval\":\"\",\"rval\":\"Acme\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1440 bottom join: colonna = valore non è un'uguaglianza tra colonne → avviso",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom join: con una lista a destra il confronto passa a «è uno di»",
      "esito": "ok",
      "dettaglio": "[\"list\",\"è uno di\"]"
    },
    {
      "prova": "1440 bottom join: lista di valori dal dominio della colonna dell'altro lato (Acme, Borealis)",
      "esito": "ok",
      "dettaglio": "[\"Acme\",\"Borealis\"]"
    },
    {
      "prova": "1440 bottom join confronto: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "934,528 225×146"
    },
    {
      "prova": "1440 bottom join valori della lista: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1178,442 225×276"
    },
    {
      "prova": "1440 bottom join: uscendo dalla lista il confronto torna «=»",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom join: freccia destra sul radiogroup sceglie «Valore» e sposta il focus (tabindex 0 solo sulla scelta)",
      "esito": "ok",
      "dettaglio": "{\"on\":\"Valore\",\"focused\":true,\"tab\":\"0\"}"
    },
    {
      "prova": "1440 bottom join: Fine sceglie «Lista»",
      "esito": "ok",
      "dettaglio": "Lista"
    },
    {
      "prova": "1440 bottom join: Home sceglie «Colonna»",
      "esito": "ok",
      "dettaglio": "Colonna"
    },
    {
      "prova": "1440 bottom join: il tipo di join si sceglie dalle impostazioni (valori del catalogo)",
      "esito": "ok",
      "dettaglio": "left"
    },
    {
      "prova": "1440 bottom layout: a 1280 e 1440, sui bordi alto e basso, tre colonne",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom layout: larghezze da token (Impostazioni 240, centrale 310) e il dettaglio prende il resto",
      "esito": "ok",
      "dettaglio": "{\"general\":240,\"master\":310,\"detail\":731,\"root\":1281}"
    },
    {
      "prova": "1440 bottom layout: intestazioni «Impostazioni» e «Condizioni e gruppi»",
      "esito": "ok",
      "dettaglio": "Impostazioni|Condizioni e gruppi"
    },
    {
      "prova": "1440 bottom layout: nessun overflow orizzontale",
      "esito": "ok",
      "dettaglio": "{\"insp\":0,\"c3\":0,\"page\":0}"
    },
    {
      "prova": "1440 bottom layout: la colonna centrale scorre per conto suo, le altre restano ferme",
      "esito": "ok",
      "dettaglio": "{\"master\":80,\"detail\":0,\"general\":0,\"headY\":605}"
    },
    {
      "prova": "1440 bottom layout: l'intestazione della colonna resta ferma mentre il contenuto scorre",
      "esito": "ok",
      "dettaglio": "605 → 605"
    },
    {
      "prova": "1440 bottom layout: il clic su «Condizione 4» la mostra nel dettaglio, evidenziata e senza comprimerla; il titolo ripete numero e riassunto",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"sum\":\"categoria ≠ Hardware\",\"rowSum\":\"categoria ≠ Hardware\",\"active\":\"3\",\"bodiesInMaster\":0}"
    },
    {
      "prova": "1440 bottom layout: dopo una modifica la voce attiva e la posizione di scorrimento sono quelle di prima, e il riassunto è aggiornato",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"active\":\"3\",\"master\":214,\"sum\":\"categoria contiene Hardware\"}"
    },
    {
      "prova": "1440 bottom elenco: l'operazione a voci ha «Elenco» come colonna centrale",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom ordina: una maniglia per criterio, con nome accessibile",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom ordina: Alt+↓ sposta il primo criterio in seconda posizione",
      "esito": "ok",
      "dettaglio": "importo,regione,stato"
    },
    {
      "prova": "1440 bottom ordina: il focus segue la riga spostata e lo spostamento è annunciato",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 bottom ordina: Alt+↑ lo riporta in prima posizione",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 bottom ordina: annullare uno spostamento da tastiera lo riporta com'era, in un passo",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 bottom ordina: trascinare la maniglia del primo criterio in fondo lo porta per ultimo",
      "esito": "ok",
      "dettaglio": "importo,stato,regione"
    },
    {
      "prova": "1440 bottom ordina: un solo annullamento riporta il trascinamento com'era",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 bottom ordina: Esc durante il trascinamento lo annulla",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 top filtro: tre condizioni create con eventi veri (regione = Nord, importo > 100, stato = Chiuso)",
      "esito": "ok",
      "dettaglio": "[[\"regione\",\"=\",[\"Nord\"],\"\"],[\"importo\",\">\",[],\"100\"],[\"stato\",\"=\",[\"Chiuso\"],\"\"]]"
    },
    {
      "prova": "1440 top filtro: ogni condizione ha un connettore tra sé e la precedente, nessun selettore globale E/O",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top filtro: anteprima senza gruppi, da sinistra a destra",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) AND stato = Chiuso"
    },
    {
      "prova": "1440 top filtro: «Raggruppa» sulle ultime due crea un gruppo (riquadro «Gruppo» con «Sciogli»)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top filtro: dopo aver raggruppato il focus non si perde (resta sul pulsante del connettore)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top connettore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "455,129 300×306"
    },
    {
      "prova": "1440 top filtro: il connettore interno diventa OR e l'anteprima è «regione = Nord AND (importo > 100 OR stato = Chiuso)»",
      "esito": "ok",
      "dettaglio": "regione = Nord AND (importo > 100 OR stato = Chiuso)"
    },
    {
      "prova": "1440 top filtro: le ultime due condizioni stanno nello stesso gruppo, la prima no",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top filtro: scrivere nel campo non fa perdere il focus",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top filtro: un solo annullamento riporta il campo a prima della digitazione",
      "esito": "ok",
      "dettaglio": "250 → 100"
    },
    {
      "prova": "1440 top filtro: «Sciogli» toglie il gruppo, le condizioni restano e l'anteprima segue",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1440 top filtro: raggruppa di nuovo (prime due) e l'anteprima porta le parentesi sul gruppo",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1440 top colonna: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,342 225×411"
    },
    {
      "prova": "1440 top operatore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "934,342 225×506"
    },
    {
      "prova": "1440 top valori: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1178,396 225×266"
    },
    {
      "prova": "1440 top connettore esterno: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "460,337 300×306"
    },
    {
      "prova": "1440 top filtro: «+ Condizione nel gruppo» aggiunge una voce dentro il gruppo",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top filtro: un gruppo che resta con una sola condizione si scioglie da sé",
      "esito": "ok",
      "dettaglio": "[null,null]"
    },
    {
      "prova": "1440 top canvas: i nodi richiesti restano dentro l'area con l'Inspector aperto",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top join: colonna = colonna (cliente = cliente) con eventi veri",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"col\",\"lval\":\"\",\"rval\":\"\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1440 top join: con una colonna = colonna non c'è l'avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top join: le colonne del lato sinistro vengono dalla tabella sinistra, quelle del destro dalla destra",
      "esito": "ok",
      "dettaglio": "sinistra: id|cliente|regione|categoria|stato|quantita|importo|data — destra: cliente|agente|sconto"
    },
    {
      "prova": "1440 top join: solo disuguaglianze → avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top join: tornando a «=» l'avviso sparisce",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top join: lato destro «Valore» con i valori della colonna dell'altro lato (cliente = “Acme”)",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"val\",\"lval\":\"\",\"rval\":\"Acme\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1440 top join: colonna = valore non è un'uguaglianza tra colonne → avviso",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top join: con una lista a destra il confronto passa a «è uno di»",
      "esito": "ok",
      "dettaglio": "[\"list\",\"è uno di\"]"
    },
    {
      "prova": "1440 top join: lista di valori dal dominio della colonna dell'altro lato (Acme, Borealis)",
      "esito": "ok",
      "dettaglio": "[\"Acme\",\"Borealis\"]"
    },
    {
      "prova": "1440 top join confronto: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "934,342 225×146"
    },
    {
      "prova": "1440 top join valori della lista: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "1178,440 225×276"
    },
    {
      "prova": "1440 top join: uscendo dalla lista il confronto torna «=»",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top join: freccia destra sul radiogroup sceglie «Valore» e sposta il focus (tabindex 0 solo sulla scelta)",
      "esito": "ok",
      "dettaglio": "{\"on\":\"Valore\",\"focused\":true,\"tab\":\"0\"}"
    },
    {
      "prova": "1440 top join: Fine sceglie «Lista»",
      "esito": "ok",
      "dettaglio": "Lista"
    },
    {
      "prova": "1440 top join: Home sceglie «Colonna»",
      "esito": "ok",
      "dettaglio": "Colonna"
    },
    {
      "prova": "1440 top join: il tipo di join si sceglie dalle impostazioni (valori del catalogo)",
      "esito": "ok",
      "dettaglio": "left"
    },
    {
      "prova": "1440 top layout: a 1280 e 1440, sui bordi alto e basso, tre colonne",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top layout: larghezze da token (Impostazioni 240, centrale 310) e il dettaglio prende il resto",
      "esito": "ok",
      "dettaglio": "{\"general\":240,\"master\":310,\"detail\":731,\"root\":1281}"
    },
    {
      "prova": "1440 top layout: intestazioni «Impostazioni» e «Condizioni e gruppi»",
      "esito": "ok",
      "dettaglio": "Impostazioni|Condizioni e gruppi"
    },
    {
      "prova": "1440 top layout: nessun overflow orizzontale",
      "esito": "ok",
      "dettaglio": "{\"insp\":0,\"c3\":0,\"page\":0}"
    },
    {
      "prova": "1440 top layout: la colonna centrale scorre per conto suo, le altre restano ferme",
      "esito": "ok",
      "dettaglio": "{\"master\":80,\"detail\":0,\"general\":0,\"headY\":217}"
    },
    {
      "prova": "1440 top layout: l'intestazione della colonna resta ferma mentre il contenuto scorre",
      "esito": "ok",
      "dettaglio": "217 → 217"
    },
    {
      "prova": "1440 top layout: il clic su «Condizione 4» la mostra nel dettaglio, evidenziata e senza comprimerla; il titolo ripete numero e riassunto",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"sum\":\"categoria ≠ Hardware\",\"rowSum\":\"categoria ≠ Hardware\",\"active\":\"3\",\"bodiesInMaster\":0}"
    },
    {
      "prova": "1440 top layout: dopo una modifica la voce attiva e la posizione di scorrimento sono quelle di prima, e il riassunto è aggiornato",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"active\":\"3\",\"master\":214,\"sum\":\"categoria contiene Hardware\"}"
    },
    {
      "prova": "1440 top elenco: l'operazione a voci ha «Elenco» come colonna centrale",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top ordina: una maniglia per criterio, con nome accessibile",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top ordina: Alt+↓ sposta il primo criterio in seconda posizione",
      "esito": "ok",
      "dettaglio": "importo,regione,stato"
    },
    {
      "prova": "1440 top ordina: il focus segue la riga spostata e lo spostamento è annunciato",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 top ordina: Alt+↑ lo riporta in prima posizione",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 top ordina: annullare uno spostamento da tastiera lo riporta com'era, in un passo",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 top ordina: trascinare la maniglia del primo criterio in fondo lo porta per ultimo",
      "esito": "ok",
      "dettaglio": "importo,stato,regione"
    },
    {
      "prova": "1440 top ordina: un solo annullamento riporta il trascinamento com'era",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440 top ordina: Esc durante il trascinamento lo annulla",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1440: sotto la larghezza minima (token, 900 px) tornano le colonne CSS della 6b.1",
      "esito": "ok",
      "dettaglio": "{\"cols3\":0,\"columnWidth\":\"280px\",\"rootWidth\":701,\"rows\":7}"
    },
    {
      "prova": "1440: ripristinata la larghezza tornano le tre colonne",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right filtro: tre condizioni create con eventi veri (regione = Nord, importo > 100, stato = Chiuso)",
      "esito": "ok",
      "dettaglio": "[[\"regione\",\"=\",[\"Nord\"],\"\"],[\"importo\",\">\",[],\"100\"],[\"stato\",\"=\",[\"Chiuso\"],\"\"]]"
    },
    {
      "prova": "1280x720 right filtro: ogni condizione ha un connettore tra sé e la precedente, nessun selettore globale E/O",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right filtro: anteprima senza gruppi, da sinistra a destra",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) AND stato = Chiuso"
    },
    {
      "prova": "1280x720 right filtro: «Raggruppa» sulle ultime due crea un gruppo (riquadro «Gruppo» con «Sciogli»)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right filtro: dopo aver raggruppato il focus non si perde (resta sul pulsante del connettore)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right connettore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "964,120 300×306"
    },
    {
      "prova": "1280x720 right filtro: il connettore interno diventa OR e l'anteprima è «regione = Nord AND (importo > 100 OR stato = Chiuso)»",
      "esito": "ok",
      "dettaglio": "regione = Nord AND (importo > 100 OR stato = Chiuso)"
    },
    {
      "prova": "1280x720 right filtro: le ultime due condizioni stanno nello stesso gruppo, la prima no",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right filtro: scrivere nel campo non fa perdere il focus",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right filtro: un solo annullamento riporta il campo a prima della digitazione",
      "esito": "ok",
      "dettaglio": "250 → 100"
    },
    {
      "prova": "1280x720 right filtro: «Sciogli» toglie il gruppo, le condizioni restano e l'anteprima segue",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1280x720 right filtro: raggruppa di nuovo (prime due) e l'anteprima porta le parentesi sul gruppo",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1280x720 right colonna: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "990,287 240×411"
    },
    {
      "prova": "1280x720 right operatore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "990,385 240×319"
    },
    {
      "prova": "1280x720 right valori: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "990,140 240×266"
    },
    {
      "prova": "1280x720 right connettore esterno: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "964,120 300×306"
    },
    {
      "prova": "1280x720 right filtro: «+ Condizione nel gruppo» aggiunge una voce dentro il gruppo",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right filtro: un gruppo che resta con una sola condizione si scioglie da sé",
      "esito": "ok",
      "dettaglio": "[null,null]"
    },
    {
      "prova": "1280x720 right canvas: i nodi richiesti restano dentro l'area con l'Inspector aperto",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right join: colonna = colonna (cliente = cliente) con eventi veri",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"col\",\"lval\":\"\",\"rval\":\"\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1280x720 right join: con una colonna = colonna non c'è l'avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right join: le colonne del lato sinistro vengono dalla tabella sinistra, quelle del destro dalla destra",
      "esito": "ok",
      "dettaglio": "sinistra: id|cliente|regione|categoria|stato|quantita|importo|data — destra: cliente|agente|sconto"
    },
    {
      "prova": "1280x720 right join: solo disuguaglianze → avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right join: tornando a «=» l'avviso sparisce",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right join: lato destro «Valore» con i valori della colonna dell'altro lato (cliente = “Acme”)",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"val\",\"lval\":\"\",\"rval\":\"Acme\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1280x720 right join: colonna = valore non è un'uguaglianza tra colonne → avviso",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right join: con una lista a destra il confronto passa a «è uno di»",
      "esito": "ok",
      "dettaglio": "[\"list\",\"è uno di\"]"
    },
    {
      "prova": "1280x720 right join: lista di valori dal dominio della colonna dell'altro lato (Acme, Borealis)",
      "esito": "ok",
      "dettaglio": "[\"Acme\",\"Borealis\"]"
    },
    {
      "prova": "1280x720 right join confronto: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "990,509 240×146"
    },
    {
      "prova": "1280x720 right join valori della lista: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "990,114 240×276"
    },
    {
      "prova": "1280x720 right join: uscendo dalla lista il confronto torna «=»",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right join: freccia destra sul radiogroup sceglie «Valore» e sposta il focus (tabindex 0 solo sulla scelta)",
      "esito": "ok",
      "dettaglio": "{\"on\":\"Valore\",\"focused\":true,\"tab\":\"0\"}"
    },
    {
      "prova": "1280x720 right join: Fine sceglie «Lista»",
      "esito": "ok",
      "dettaglio": "Lista"
    },
    {
      "prova": "1280x720 right join: Home sceglie «Colonna»",
      "esito": "ok",
      "dettaglio": "Colonna"
    },
    {
      "prova": "1280x720 right join: il tipo di join si sceglie dalle impostazioni (valori del catalogo)",
      "esito": "ok",
      "dettaglio": "left"
    },
    {
      "prova": "1280x720 right layout: sul bordo laterale le parti si impilano con le righe comprimibili (nessuna tre colonne)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right ordina: una maniglia per criterio, con nome accessibile",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right ordina: Alt+↓ sposta il primo criterio in seconda posizione",
      "esito": "ok",
      "dettaglio": "importo,regione,stato"
    },
    {
      "prova": "1280x720 right ordina: il focus segue la riga spostata e lo spostamento è annunciato",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 right ordina: Alt+↑ lo riporta in prima posizione",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 right ordina: annullare uno spostamento da tastiera lo riporta com'era, in un passo",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 right ordina: trascinare la maniglia del primo criterio in fondo lo porta per ultimo",
      "esito": "ok",
      "dettaglio": "importo,stato,regione"
    },
    {
      "prova": "1280x720 right ordina: un solo annullamento riporta il trascinamento com'era",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 right ordina: Esc durante il trascinamento lo annulla",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 bottom filtro: tre condizioni create con eventi veri (regione = Nord, importo > 100, stato = Chiuso)",
      "esito": "ok",
      "dettaglio": "[[\"regione\",\"=\",[\"Nord\"],\"\"],[\"importo\",\">\",[],\"100\"],[\"stato\",\"=\",[\"Chiuso\"],\"\"]]"
    },
    {
      "prova": "1280x720 bottom filtro: ogni condizione ha un connettore tra sé e la precedente, nessun selettore globale E/O",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom filtro: anteprima senza gruppi, da sinistra a destra",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) AND stato = Chiuso"
    },
    {
      "prova": "1280x720 bottom filtro: «Raggruppa» sulle ultime due crea un gruppo (riquadro «Gruppo» con «Sciogli»)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom filtro: dopo aver raggruppato il focus non si perde (resta sul pulsante del connettore)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom connettore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "455,264 300×306"
    },
    {
      "prova": "1280x720 bottom filtro: il connettore interno diventa OR e l'anteprima è «regione = Nord AND (importo > 100 OR stato = Chiuso)»",
      "esito": "ok",
      "dettaglio": "regione = Nord AND (importo > 100 OR stato = Chiuso)"
    },
    {
      "prova": "1280x720 bottom filtro: le ultime due condizioni stanno nello stesso gruppo, la prima no",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom filtro: scrivere nel campo non fa perdere il focus",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom filtro: un solo annullamento riporta il campo a prima della digitazione",
      "esito": "ok",
      "dettaglio": "250 → 100"
    },
    {
      "prova": "1280x720 bottom filtro: «Sciogli» toglie il gruppo, le condizioni restano e l'anteprima segue",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1280x720 bottom filtro: raggruppa di nuovo (prime due) e l'anteprima porta le parentesi sul gruppo",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1280x720 bottom colonna: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,127 267×411"
    },
    {
      "prova": "1280x720 bottom operatore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "976,32 267×506"
    },
    {
      "prova": "1280x720 bottom valori: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,366 267×266"
    },
    {
      "prova": "1280x720 bottom connettore esterno: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "460,215 300×306"
    },
    {
      "prova": "1280x720 bottom filtro: «+ Condizione nel gruppo» aggiunge una voce dentro il gruppo",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom filtro: un gruppo che resta con una sola condizione si scioglie da sé",
      "esito": "ok",
      "dettaglio": "[null,null]"
    },
    {
      "prova": "1280x720 bottom canvas: i nodi richiesti restano dentro l'area con l'Inspector aperto",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom join: colonna = colonna (cliente = cliente) con eventi veri",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"col\",\"lval\":\"\",\"rval\":\"\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1280x720 bottom join: con una colonna = colonna non c'è l'avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom join: le colonne del lato sinistro vengono dalla tabella sinistra, quelle del destro dalla destra",
      "esito": "ok",
      "dettaglio": "sinistra: id|cliente|regione|categoria|stato|quantita|importo|data — destra: cliente|agente|sconto"
    },
    {
      "prova": "1280x720 bottom join: solo disuguaglianze → avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom join: tornando a «=» l'avviso sparisce",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom join: lato destro «Valore» con i valori della colonna dell'altro lato (cliente = “Acme”)",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"val\",\"lval\":\"\",\"rval\":\"Acme\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1280x720 bottom join: colonna = valore non è un'uguaglianza tra colonne → avviso",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom join: con una lista a destra il confronto passa a «è uno di»",
      "esito": "ok",
      "dettaglio": "[\"list\",\"è uno di\"]"
    },
    {
      "prova": "1280x720 bottom join: lista di valori dal dominio della colonna dell'altro lato (Acme, Borealis)",
      "esito": "ok",
      "dettaglio": "[\"Acme\",\"Borealis\"]"
    },
    {
      "prova": "1280x720 bottom join confronto: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "976,397 267×146"
    },
    {
      "prova": "1280x720 bottom join valori della lista: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,305 267×276"
    },
    {
      "prova": "1280x720 bottom join: uscendo dalla lista il confronto torna «=»",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom join: freccia destra sul radiogroup sceglie «Valore» e sposta il focus (tabindex 0 solo sulla scelta)",
      "esito": "ok",
      "dettaglio": "{\"on\":\"Valore\",\"focused\":true,\"tab\":\"0\"}"
    },
    {
      "prova": "1280x720 bottom join: Fine sceglie «Lista»",
      "esito": "ok",
      "dettaglio": "Lista"
    },
    {
      "prova": "1280x720 bottom join: Home sceglie «Colonna»",
      "esito": "ok",
      "dettaglio": "Colonna"
    },
    {
      "prova": "1280x720 bottom join: il tipo di join si sceglie dalle impostazioni (valori del catalogo)",
      "esito": "ok",
      "dettaglio": "left"
    },
    {
      "prova": "1280x720 bottom layout: a 1280 e 1440, sui bordi alto e basso, tre colonne",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom layout: larghezze da token (Impostazioni 240, centrale 310) e il dettaglio prende il resto",
      "esito": "ok",
      "dettaglio": "{\"general\":240,\"master\":310,\"detail\":571,\"root\":1121}"
    },
    {
      "prova": "1280x720 bottom layout: intestazioni «Impostazioni» e «Condizioni e gruppi»",
      "esito": "ok",
      "dettaglio": "Impostazioni|Condizioni e gruppi"
    },
    {
      "prova": "1280x720 bottom layout: nessun overflow orizzontale",
      "esito": "ok",
      "dettaglio": "{\"insp\":0,\"c3\":0,\"page\":0}"
    },
    {
      "prova": "1280x720 bottom layout: la colonna centrale scorre per conto suo, le altre restano ferme",
      "esito": "ok",
      "dettaglio": "{\"master\":80,\"detail\":0,\"general\":0,\"headY\":474}"
    },
    {
      "prova": "1280x720 bottom layout: l'intestazione della colonna resta ferma mentre il contenuto scorre",
      "esito": "ok",
      "dettaglio": "474 → 474"
    },
    {
      "prova": "1280x720 bottom layout: il clic su «Condizione 4» la mostra nel dettaglio, evidenziata e senza comprimerla; il titolo ripete numero e riassunto",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"sum\":\"categoria ≠ Hardware\",\"rowSum\":\"categoria ≠ Hardware\",\"active\":\"3\",\"bodiesInMaster\":0}"
    },
    {
      "prova": "1280x720 bottom layout: dopo una modifica la voce attiva e la posizione di scorrimento sono quelle di prima, e il riassunto è aggiornato",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"active\":\"3\",\"master\":239,\"sum\":\"categoria contiene Hardware\"}"
    },
    {
      "prova": "1280x720 bottom elenco: l'operazione a voci ha «Elenco» come colonna centrale",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom ordina: una maniglia per criterio, con nome accessibile",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom ordina: Alt+↓ sposta il primo criterio in seconda posizione",
      "esito": "ok",
      "dettaglio": "importo,regione,stato"
    },
    {
      "prova": "1280x720 bottom ordina: il focus segue la riga spostata e lo spostamento è annunciato",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 bottom ordina: Alt+↑ lo riporta in prima posizione",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 bottom ordina: annullare uno spostamento da tastiera lo riporta com'era, in un passo",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 bottom ordina: trascinare la maniglia del primo criterio in fondo lo porta per ultimo",
      "esito": "ok",
      "dettaglio": "importo,stato,regione"
    },
    {
      "prova": "1280x720 bottom ordina: un solo annullamento riporta il trascinamento com'era",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 bottom ordina: Esc durante il trascinamento lo annulla",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 top filtro: tre condizioni create con eventi veri (regione = Nord, importo > 100, stato = Chiuso)",
      "esito": "ok",
      "dettaglio": "[[\"regione\",\"=\",[\"Nord\"],\"\"],[\"importo\",\">\",[],\"100\"],[\"stato\",\"=\",[\"Chiuso\"],\"\"]]"
    },
    {
      "prova": "1280x720 top filtro: ogni condizione ha un connettore tra sé e la precedente, nessun selettore globale E/O",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top filtro: anteprima senza gruppi, da sinistra a destra",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) AND stato = Chiuso"
    },
    {
      "prova": "1280x720 top filtro: «Raggruppa» sulle ultime due crea un gruppo (riquadro «Gruppo» con «Sciogli»)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top filtro: dopo aver raggruppato il focus non si perde (resta sul pulsante del connettore)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top connettore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "455,361 300×306"
    },
    {
      "prova": "1280x720 top filtro: il connettore interno diventa OR e l'anteprima è «regione = Nord AND (importo > 100 OR stato = Chiuso)»",
      "esito": "ok",
      "dettaglio": "regione = Nord AND (importo > 100 OR stato = Chiuso)"
    },
    {
      "prova": "1280x720 top filtro: le ultime due condizioni stanno nello stesso gruppo, la prima no",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top filtro: scrivere nel campo non fa perdere il focus",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top filtro: un solo annullamento riporta il campo a prima della digitazione",
      "esito": "ok",
      "dettaglio": "250 → 100"
    },
    {
      "prova": "1280x720 top filtro: «Sciogli» toglie il gruppo, le condizioni restano e l'anteprima segue",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1280x720 top filtro: raggruppa di nuovo (prime due) e l'anteprima porta le parentesi sul gruppo",
      "esito": "ok",
      "dettaglio": "(regione = Nord AND importo > 100) OR stato = Chiuso"
    },
    {
      "prova": "1280x720 top colonna: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,337 267×367"
    },
    {
      "prova": "1280x720 top operatore: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "976,337 267×367"
    },
    {
      "prova": "1280x720 top valori: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,109 267×266"
    },
    {
      "prova": "1280x720 top connettore esterno: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
```

