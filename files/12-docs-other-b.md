# 12-docs-other-b.md

File in questo blocco:

- `docs/visual/fase6b11/misure.json`

---

### `docs/visual/fase6b11/misure.json` (parte 1/2)

2133 righe totali

```json
{
  "risultati": [
    {
      "prova": "1440: la scena densa ha 32 nodi e tutti sono richiesti a pannelli chiusi",
      "esito": "ok",
      "dettaglio": "32 richiesti"
    },
    {
      "prova": "1440: la scena densa a pannelli chiusi dista dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440: nessuna transizione CSS di larghezza o altezza sui pannelli (un solo orologio)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri cassetta left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri cassetta left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440 apri cassetta left: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri cassetta left: R3 zoom monotono (1.000 → 0.763) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri cassetta left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 24.18 px, media 15.21 px"
    },
    {
      "prova": "1440 chiudi cassetta left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi cassetta left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440 chiudi cassetta left: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi cassetta left: R3 zoom monotono (0.763 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi cassetta left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 24.18 px, media 15.21 px"
    },
    {
      "prova": "1440 cassetta left: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1440 apri cassetta right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri cassetta right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1440 apri cassetta right: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri cassetta right: R3 zoom monotono (1.000 → 0.739) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri cassetta right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 0.61 px, media 0.38 px"
    },
    {
      "prova": "1440 chiudi cassetta right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi cassetta right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440 chiudi cassetta right: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi cassetta right: R3 zoom monotono (0.739 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi cassetta right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.01 px, media 1.90 px"
    },
    {
      "prova": "1440 cassetta right: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1440 apri cassetta top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri cassetta top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 232.14 px"
    },
    {
      "prova": "1440 apri cassetta top: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri cassetta top: R3 zoom monotono (1.000 → 0.634) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri cassetta top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.53 px, media 2.22 px"
    },
    {
      "prova": "1440 chiudi cassetta top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi cassetta top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440 chiudi cassetta top: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi cassetta top: R3 zoom monotono (0.634 → 0.974) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi cassetta top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.28 px, media 2.06 px"
    },
    {
      "prova": "1440 cassetta top: R6 apri → chiudi, la vista finale è quella iniziale (area finale più stretta per la tacca: la vista si adatta, R1 vale)",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":20.541,\"y\":16.634,\"zoom\":0.9736}"
    },
    {
      "prova": "1440 apri cassetta bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri cassetta bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 232.14 px"
    },
    {
      "prova": "1440 apri cassetta bottom: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri cassetta bottom: R3 zoom monotono (1.000 → 0.634) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri cassetta bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 1.04 px, media 0.66 px"
    },
    {
      "prova": "1440 chiudi cassetta bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi cassetta bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440 chiudi cassetta bottom: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi cassetta bottom: R3 zoom monotono (0.634 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi cassetta bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 8.52 px, media 5.36 px"
    },
    {
      "prova": "1440 cassetta bottom: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1440 apri Inspector left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri Inspector left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1440 apri Inspector left: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri Inspector left: R3 zoom monotono (1.000 → 0.739) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri Inspector left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.32 px, media 2.09 px"
    },
    {
      "prova": "1440 chiudi Inspector left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi Inspector left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 164.00 px"
    },
    {
      "prova": "1440 chiudi Inspector left: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi Inspector left: R3 zoom monotono (0.739 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi Inspector left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.32 px, media 2.09 px"
    },
    {
      "prova": "1440 Inspector left: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1440 apri Inspector right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri Inspector right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 164.00 px"
    },
    {
      "prova": "1440 apri Inspector right: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri Inspector right: R3 zoom monotono (1.000 → 0.739) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri Inspector right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 0.61 px, media 0.38 px"
    },
    {
      "prova": "1440 chiudi Inspector right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi Inspector right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440 chiudi Inspector right: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi Inspector right: R3 zoom monotono (0.739 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi Inspector right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.01 px, media 1.90 px"
    },
    {
      "prova": "1440 Inspector right: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1440 apri Inspector top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri Inspector top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 360.55 px"
    },
    {
      "prova": "1440 apri Inspector top: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri Inspector top: R3 zoom monotono (1.000 → 0.535) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri Inspector top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 27.02 px, media 17.00 px"
    },
    {
      "prova": "1440 chiudi Inspector top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi Inspector top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1440 chiudi Inspector top: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi Inspector top: R3 zoom monotono (0.535 → 0.974) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi Inspector top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 26.53 px, media 16.70 px"
    },
    {
      "prova": "1440 Inspector top: R6 apri → chiudi, la vista finale è quella iniziale (area finale più stretta per la tacca: la vista si adatta, R1 vale)",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":28.541,\"y\":16.634,\"zoom\":0.9736}"
    },
    {
      "prova": "1440 apri Inspector bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 apri Inspector bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 360.55 px"
    },
    {
      "prova": "1440 apri Inspector bottom: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 apri Inspector bottom: R3 zoom monotono (1.000 → 0.535) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 apri Inspector bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 4.30 px, media 2.70 px"
    },
    {
      "prova": "1440 chiudi Inspector bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 chiudi Inspector bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 34.00 px"
    },
    {
      "prova": "1440 chiudi Inspector bottom: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 chiudi Inspector bottom: R3 zoom monotono (0.535 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 chiudi Inspector bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 4.30 px, media 2.70 px"
    },
    {
      "prova": "1440 Inspector bottom: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1440 cambio scheda (cassetta → Inspector, stesso bordo): R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 cambio scheda (cassetta → Inspector, stesso bordo): R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1440 cambio scheda (cassetta → Inspector, stesso bordo): R2 a ogni frame (5) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 cambio scheda (cassetta → Inspector, stesso bordo): R3 la vista resta ferma (0.535 → 0.535) e transizione di 1 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=1"
    },
    {
      "prova": "1440 cambio scheda (cassetta → Inspector, stesso bordo): R4 nessun salto (rapporto massimo 0.00 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 0.00 px, media 0.00 px"
    },
    {
      "prova": "1440: il cambio scheda non sposta il canvas (stessa area, vista invariata)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 cambio scheda (Inspector → cassetta): R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 cambio scheda (Inspector → cassetta): R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1440 cambio scheda (Inspector → cassetta): R2 a ogni frame (5) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 cambio scheda (Inspector → cassetta): R3 la vista resta ferma (0.535 → 0.535) e transizione di 1 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=1"
    },
    {
      "prova": "1440 cambio scheda (Inspector → cassetta): R4 nessun salto (rapporto massimo 0.00 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 0.00 px, media 0.00 px"
    },
    {
      "prova": "1440 dalla cassetta a sinistra all'Inspector a destra: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 dalla cassetta a sinistra all'Inspector a destra: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 164.00 px"
    },
    {
      "prova": "1440 dalla cassetta a sinistra all'Inspector a destra: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 dalla cassetta a sinistra all'Inspector a destra: R3 zoom monotono (0.763 → 0.739) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1440 dalla cassetta a sinistra all'Inspector a destra: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 25.59 px, media 16.10 px"
    },
    {
      "prova": "1440 sposta l'Inspector su bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 sposta l'Inspector su bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 360.55 px"
    },
    {
      "prova": "1440 sposta l'Inspector su bottom: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 sposta l'Inspector su bottom: R3 zoom monotono (0.739 → 0.535) e transizione di 20 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=20"
    },
    {
      "prova": "1440 sposta l'Inspector su bottom: R4 nessun salto (rapporto massimo 1.67 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 6.98 px, media 4.17 px"
    },
    {
      "prova": "1440: dopo lo spostamento su bottom non resta nessun guscio",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 sposta l'Inspector su top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 sposta l'Inspector su top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 360.55 px"
    },
    {
      "prova": "1440 sposta l'Inspector su top: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 sposta l'Inspector su top: R3 zoom monotono (0.535 → 0.535) e transizione di 20 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=20"
    },
    {
      "prova": "1440 sposta l'Inspector su top: R4 nessun salto (rapporto massimo 1.67 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 26.42 px, media 15.80 px"
    },
    {
      "prova": "1440: dopo lo spostamento su top non resta nessun guscio",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 sposta l'Inspector su left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 sposta l'Inspector su left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1440 sposta l'Inspector su left: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1440 sposta l'Inspector su left: R3 zoom monotono (0.535 → 0.739) e transizione di 20 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=20"
    },
    {
      "prova": "1440 sposta l'Inspector su left: R4 nessun salto (rapporto massimo 1.67 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 31.44 px, media 18.79 px"
    },
    {
      "prova": "1440: dopo lo spostamento su left non resta nessun guscio",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 ridimensionamento: a ogni passo (12) i nodi richiesti restano dentro l'area sicura",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 ridimensionamento: tornando alla dimensione di prima la vista è quella di prima",
      "esito": "ok",
      "dettaglio": "{\"x\":284.039603960396,\"y\":11.168316831683168,\"zoom\":0.5346534653465347} → {\"x\":284.039603960396,\"y\":11.168316831683168,\"zoom\":0.5346534653465347}"
    },
    {
      "prova": "1440 rotella a metà transizione: la vista è continua (solo lo scorrimento della rotella, 40 px)",
      "esito": "ok",
      "dettaglio": "{\"x\":65.92178666159676,\"y\":2.5920167092261974,\"zoom\":0.8919993037822418} → {\"x\":65.92178666159676,\"y\":-37.4079832907738,\"zoom\":0.8919993037822418}"
    },
    {
      "prova": "1440 rotella a metà transizione: l'animazione della vista è annullata (resta dov'è) e il pannello finisce",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1440 due cambi ravvicinati (apri e subito chiudi): nessun salto (rapporto 1.72 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 25.85 px"
    },
    {
      "prova": "1440 due cambi ravvicinati: alla fine la vista è tornata a quella iniziale",
      "esito": "ok",
      "dettaglio": "{\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1440: i frame dell'animazione usano solo il ciclo condiviso (rAF per passo ≤ 2: 1.63)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720: la scena densa ha 32 nodi e tutti sono richiesti a pannelli chiusi",
      "esito": "ok",
      "dettaglio": "32 richiesti"
    },
    {
      "prova": "1280x720: la scena densa a pannelli chiusi dista dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720: nessuna transizione CSS di larghezza o altezza sui pannelli (un solo orologio)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri cassetta left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri cassetta left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720 apri cassetta left: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri cassetta left: R3 zoom monotono (1.000 → 0.725) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri cassetta left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 23.93 px, media 15.05 px"
    },
    {
      "prova": "1280x720 chiudi cassetta left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi cassetta left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta left: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta left: R3 zoom monotono (0.725 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi cassetta left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 23.93 px, media 15.05 px"
    },
    {
      "prova": "1280x720 cassetta left: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1280x720 apri cassetta right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri cassetta right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1280x720 apri cassetta right: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri cassetta right: R3 zoom monotono (1.000 → 0.698) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri cassetta right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 2.60 px, media 1.64 px"
    },
    {
      "prova": "1280x720 chiudi cassetta right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi cassetta right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta right: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta right: R3 zoom monotono (0.698 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi cassetta right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 2.60 px, media 1.64 px"
    },
    {
      "prova": "1280x720 cassetta right: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1280x720 apri cassetta top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri cassetta top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 319.07 px"
    },
    {
      "prova": "1280x720 apri cassetta top: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri cassetta top: R3 zoom monotono (1.000 → 0.559) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri cassetta top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 7.33 px, media 4.61 px"
    },
    {
      "prova": "1280x720 chiudi cassetta top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi cassetta top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta top: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta top: R3 zoom monotono (0.559 → 0.962) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi cassetta top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 6.79 px, media 4.27 px"
    },
    {
      "prova": "1280x720 cassetta top: R6 apri → chiudi, la vista finale è quella iniziale (area finale più stretta per la tacca: la vista si adatta, R1 vale)",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":26.216,\"y\":16.901,\"zoom\":0.9624}"
    },
    {
      "prova": "1280x720 apri cassetta bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri cassetta bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 319.07 px"
    },
    {
      "prova": "1280x720 apri cassetta bottom: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri cassetta bottom: R3 zoom monotono (1.000 → 0.559) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri cassetta bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 8.16 px, media 5.13 px"
    },
    {
      "prova": "1280x720 chiudi cassetta bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi cassetta bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta bottom: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi cassetta bottom: R3 zoom monotono (0.559 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi cassetta bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 2.48 px, media 1.56 px"
    },
    {
      "prova": "1280x720 cassetta bottom: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1280x720 apri Inspector left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri Inspector left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1280x720 apri Inspector left: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri Inspector left: R3 zoom monotono (1.000 → 0.698) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri Inspector left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.57 px, media 2.24 px"
    },
    {
      "prova": "1280x720 chiudi Inspector left: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi Inspector left: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 164.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector left: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector left: R3 zoom monotono (0.698 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi Inspector left: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 3.57 px, media 2.24 px"
    },
    {
      "prova": "1280x720 Inspector left: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1280x720 apri Inspector right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri Inspector right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 164.00 px"
    },
    {
      "prova": "1280x720 apri Inspector right: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri Inspector right: R3 zoom monotono (1.000 → 0.698) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri Inspector right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 2.60 px, media 1.64 px"
    },
    {
      "prova": "1280x720 chiudi Inspector right: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi Inspector right: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector right: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector right: R3 zoom monotono (0.698 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi Inspector right: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 2.60 px, media 1.64 px"
    },
    {
      "prova": "1280x720 Inspector right: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1280x720 apri Inspector top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri Inspector top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 290.94 px"
    },
    {
      "prova": "1280x720 apri Inspector top: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri Inspector top: R3 zoom monotono (1.000 → 0.453) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri Inspector top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 8.04 px, media 5.05 px"
    },
    {
      "prova": "1280x720 chiudi Inspector top: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi Inspector top: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 16.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector top: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector top: R3 zoom monotono (0.453 → 0.962) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi Inspector top: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 8.00 px, media 5.03 px"
    },
    {
      "prova": "1280x720 Inspector top: R6 apri → chiudi, la vista finale è quella iniziale (area finale più stretta per la tacca: la vista si adatta, R1 vale)",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":34.216,\"y\":16.901,\"zoom\":0.9624}"
    },
    {
      "prova": "1280x720 apri Inspector bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 apri Inspector bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 300.94 px"
    },
    {
      "prova": "1280x720 apri Inspector bottom: R2 a ogni frame (22) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 apri Inspector bottom: R3 zoom monotono (1.000 → 0.453) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 apri Inspector bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 6.80 px, media 4.28 px"
    },
    {
      "prova": "1280x720 chiudi Inspector bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 chiudi Inspector bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 34.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector bottom: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 chiudi Inspector bottom: R3 zoom monotono (0.453 → 1.000) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 chiudi Inspector bottom: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 0.71 px, media 0.45 px"
    },
    {
      "prova": "1280x720 Inspector bottom: R6 apri → chiudi, la vista finale è quella iniziale",
      "esito": "ok",
      "dettaglio": "iniziale {\"x\":0,\"y\":0,\"zoom\":1} finale {\"x\":0,\"y\":0,\"zoom\":1}"
    },
    {
      "prova": "1280x720 cambio scheda (cassetta → Inspector, stesso bordo): R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 cambio scheda (cassetta → Inspector, stesso bordo): R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1280x720 cambio scheda (cassetta → Inspector, stesso bordo): R2 a ogni frame (5) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 cambio scheda (cassetta → Inspector, stesso bordo): R3 la vista resta ferma (0.453 → 0.453) e transizione di 1 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=1"
    },
    {
      "prova": "1280x720 cambio scheda (cassetta → Inspector, stesso bordo): R4 nessun salto (rapporto massimo 0.00 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 0.00 px, media 0.00 px"
    },
    {
      "prova": "1280x720: il cambio scheda non sposta il canvas (stessa area, vista invariata)",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 cambio scheda (Inspector → cassetta): R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 cambio scheda (Inspector → cassetta): R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima nessuna tacca px"
    },
    {
      "prova": "1280x720 cambio scheda (Inspector → cassetta): R2 a ogni frame (5) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 cambio scheda (Inspector → cassetta): R3 la vista resta ferma (0.453 → 0.453) e transizione di 1 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=1"
    },
    {
      "prova": "1280x720 cambio scheda (Inspector → cassetta): R4 nessun salto (rapporto massimo 0.00 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 0.00 px, media 0.00 px"
    },
    {
      "prova": "1280x720 dalla cassetta a sinistra all'Inspector a destra: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 dalla cassetta a sinistra all'Inspector a destra: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 164.00 px"
    },
    {
      "prova": "1280x720 dalla cassetta a sinistra all'Inspector a destra: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 dalla cassetta a sinistra all'Inspector a destra: R3 zoom monotono (0.725 → 0.698) e transizione di 19 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=19"
    },
    {
      "prova": "1280x720 dalla cassetta a sinistra all'Inspector a destra: R4 nessun salto (rapporto massimo 1.59 ≤ 2,5)",
      "esito": "ok",
      "dettaglio": "passo massimo 25.57 px, media 16.09 px"
    },
    {
      "prova": "1280x720 sposta l'Inspector su bottom: R1 a riposo i 32 nodi richiesti sono dentro l'area sicura e non toccano pannelli né widget",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 sposta l'Inspector su bottom: R7 a riposo i nodi richiesti distano dalle tacche almeno 16 px",
      "esito": "ok",
      "dettaglio": "distanza minima 300.94 px"
    },
    {
      "prova": "1280x720 sposta l'Inspector su bottom: R2 a ogni frame (23) i nodi richiesti sono dentro il canvas di quel frame",
      "esito": "ok",
      "dettaglio": "violazioni 0, scarto massimo 0.00 px"
    },
    {
      "prova": "1280x720 sposta l'Inspector su bottom: R3 zoom monotono (0.698 → 0.453) e transizione di 20 frame",
      "esito": "ok",
      "dettaglio": "monotono=true limitato=true frame attivi=20"
    },
    {
```

