# 12-docs-other-e.md

File in questo blocco:

- `docs/visual/fase6b2/misure.json`

---

### `docs/visual/fase6b2/misure.json` (parte 2/2)

2057 righe totali

```json
      "dettaglio": "460,312 300×306"
    },
    {
      "prova": "1280x720 top filtro: «+ Condizione nel gruppo» aggiunge una voce dentro il gruppo",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top filtro: un gruppo che resta con una sola condizione si scioglie da sé",
      "esito": "ok",
      "dettaglio": "[null,null]"
    },
    {
      "prova": "1280x720 top canvas: i nodi richiesti restano dentro l'area con l'Inspector aperto",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top join: colonna = colonna (cliente = cliente) con eventi veri",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"col\",\"lval\":\"\",\"rval\":\"\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1280x720 top join: con una colonna = colonna non c'è l'avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top join: le colonne del lato sinistro vengono dalla tabella sinistra, quelle del destro dalla destra",
      "esito": "ok",
      "dettaglio": "sinistra: id|cliente|regione|categoria|stato|quantita|importo|data — destra: cliente|agente|sconto"
    },
    {
      "prova": "1280x720 top join: solo disuguaglianze → avviso di prestazioni",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top join: tornando a «=» l'avviso sparisce",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top join: lato destro «Valore» con i valori della colonna dell'altro lato (cliente = “Acme”)",
      "esito": "ok",
      "dettaglio": "{\"left\":\"cliente\",\"right\":\"cliente\",\"op\":\"=\",\"lmode\":\"col\",\"rmode\":\"val\",\"lval\":\"\",\"rval\":\"Acme\",\"rlist\":{\"mode\":\"list\",\"values\":[],\"text\":\"\",\"sep\":\",\"}}"
    },
    {
      "prova": "1280x720 top join: colonna = valore non è un'uguaglianza tra colonne → avviso",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top join: con una lista a destra il confronto passa a «è uno di»",
      "esito": "ok",
      "dettaglio": "[\"list\",\"è uno di\"]"
    },
    {
      "prova": "1280x720 top join: lista di valori dal dominio della colonna dell'altro lato (Acme, Borealis)",
      "esito": "ok",
      "dettaglio": "[\"Acme\",\"Borealis\"]"
    },
    {
      "prova": "1280x720 top join confronto: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "976,342 267×146"
    },
    {
      "prova": "1280x720 top join valori della lista: il menu dista ≥ 16 px dai bordi, non copre il campo ed è nel portale",
      "esito": "ok",
      "dettaglio": "689,48 267×276"
    },
    {
      "prova": "1280x720 top join: uscendo dalla lista il confronto torna «=»",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top join: freccia destra sul radiogroup sceglie «Valore» e sposta il focus (tabindex 0 solo sulla scelta)",
      "esito": "ok",
      "dettaglio": "{\"on\":\"Valore\",\"focused\":true,\"tab\":\"0\"}"
    },
    {
      "prova": "1280x720 top join: Fine sceglie «Lista»",
      "esito": "ok",
      "dettaglio": "Lista"
    },
    {
      "prova": "1280x720 top join: Home sceglie «Colonna»",
      "esito": "ok",
      "dettaglio": "Colonna"
    },
    {
      "prova": "1280x720 top join: il tipo di join si sceglie dalle impostazioni (valori del catalogo)",
      "esito": "ok",
      "dettaglio": "left"
    },
    {
      "prova": "1280x720 top layout: a 1280 e 1440, sui bordi alto e basso, tre colonne",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top layout: larghezze da token (Impostazioni 240, centrale 310) e il dettaglio prende il resto",
      "esito": "ok",
      "dettaglio": "{\"general\":240,\"master\":310,\"detail\":571,\"root\":1121}"
    },
    {
      "prova": "1280x720 top layout: intestazioni «Impostazioni» e «Condizioni e gruppi»",
      "esito": "ok",
      "dettaglio": "Impostazioni|Condizioni e gruppi"
    },
    {
      "prova": "1280x720 top layout: nessun overflow orizzontale",
      "esito": "ok",
      "dettaglio": "{\"insp\":0,\"c3\":0,\"page\":0}"
    },
    {
      "prova": "1280x720 top layout: la colonna centrale scorre per conto suo, le altre restano ferme",
      "esito": "ok",
      "dettaglio": "{\"master\":80,\"detail\":0,\"general\":0,\"headY\":217}"
    },
    {
      "prova": "1280x720 top layout: l'intestazione della colonna resta ferma mentre il contenuto scorre",
      "esito": "ok",
      "dettaglio": "217 → 217"
    },
    {
      "prova": "1280x720 top layout: il clic su «Condizione 4» la mostra nel dettaglio, evidenziata e senza comprimerla; il titolo ripete numero e riassunto",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"sum\":\"categoria ≠ Hardware\",\"rowSum\":\"categoria ≠ Hardware\",\"active\":\"3\",\"bodiesInMaster\":0}"
    },
    {
      "prova": "1280x720 top layout: dopo una modifica la voce attiva e la posizione di scorrimento sono quelle di prima, e il riassunto è aggiornato",
      "esito": "ok",
      "dettaglio": "{\"title\":\"Condizione 4\",\"active\":\"3\",\"master\":239,\"sum\":\"categoria contiene Hardware\"}"
    },
    {
      "prova": "1280x720 top elenco: l'operazione a voci ha «Elenco» come colonna centrale",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top ordina: una maniglia per criterio, con nome accessibile",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top ordina: Alt+↓ sposta il primo criterio in seconda posizione",
      "esito": "ok",
      "dettaglio": "importo,regione,stato"
    },
    {
      "prova": "1280x720 top ordina: il focus segue la riga spostata e lo spostamento è annunciato",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "1280x720 top ordina: Alt+↑ lo riporta in prima posizione",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 top ordina: annullare uno spostamento da tastiera lo riporta com'era, in un passo",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 top ordina: trascinare la maniglia del primo criterio in fondo lo porta per ultimo",
      "esito": "ok",
      "dettaglio": "importo,stato,regione"
    },
    {
      "prova": "1280x720 top ordina: un solo annullamento riporta il trascinamento com'era",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720 top ordina: Esc durante il trascinamento lo annulla",
      "esito": "ok",
      "dettaglio": "regione,importo,stato"
    },
    {
      "prova": "1280x720: sotto la larghezza minima (token, 900 px) tornano le colonne CSS della 6b.1",
      "esito": "ok",
      "dettaglio": "{\"cols3\":0,\"columnWidth\":\"280px\",\"rootWidth\":701,\"rows\":7}"
    },
    {
      "prova": "1280x720: ripristinata la larghezza tornano le tre colonne",
      "esito": "ok",
      "dettaglio": ""
    },
    {
      "prova": "nessun errore in console",
      "esito": "ok",
      "dettaglio": ""
    }
  ],
  "misure": [
    {
      "prova": "1440 right connettore",
      "menu": {
        "x": 1124,
        "y": 231.390625,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 right colonna",
      "menu": {
        "x": 1150,
        "y": 444.1875,
        "w": 240,
        "h": 410.796875
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 right operatore",
      "menu": {
        "x": 1150,
        "y": 16,
        "w": 240,
        "h": 469.78125
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 right valori",
      "menu": {
        "x": 1150,
        "y": 296.578125,
        "w": 240,
        "h": 266
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 right connettore esterno",
      "menu": {
        "x": 1124,
        "y": 317.390625,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 right join confronto",
      "menu": {
        "x": 1150,
        "y": 637.171875,
        "w": 240,
        "h": 146
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 right join valori della lista",
      "menu": {
        "x": 1150,
        "y": 293.96875,
        "w": 240,
        "h": 276
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 bottom connettore",
      "menu": {
        "x": 454.765625,
        "y": 517,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 bottom colonna",
      "menu": {
        "x": 689,
        "y": 262.78125,
        "w": 224.65625,
        "h": 410.796875
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 bottom operatore",
      "menu": {
        "x": 933.65625,
        "y": 167.78125,
        "w": 224.671875,
        "h": 506
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 bottom valori",
      "menu": {
        "x": 1178.328125,
        "y": 407.78125,
        "w": 224.671875,
        "h": 266
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 bottom connettore esterno",
      "menu": {
        "x": 459.515625,
        "y": 371,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 bottom join confronto",
      "menu": {
        "x": 933.65625,
        "y": 527.78125,
        "w": 224.671875,
        "h": 146
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 bottom join valori della lista",
      "menu": {
        "x": 1178.328125,
        "y": 441.78125,
        "w": 224.671875,
        "h": 276
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 top connettore",
      "menu": {
        "x": 454.765625,
        "y": 129,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 top colonna",
      "menu": {
        "x": 689,
        "y": 341.78125,
        "w": 224.65625,
        "h": 410.796875
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 top operatore",
      "menu": {
        "x": 933.65625,
        "y": 341.78125,
        "w": 224.671875,
        "h": 506
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 top valori",
      "menu": {
        "x": 1178.328125,
        "y": 395.78125,
        "w": 224.671875,
        "h": 266
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 top connettore esterno",
      "menu": {
        "x": 459.515625,
        "y": 337,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 top join confronto",
      "menu": {
        "x": 933.65625,
        "y": 341.78125,
        "w": 224.671875,
        "h": 146
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1440 top join valori della lista",
      "menu": {
        "x": 1178.328125,
        "y": 439.78125,
        "w": 224.671875,
        "h": 276
      },
      "finestra": {
        "w": 1440,
        "h": 900
      }
    },
    {
      "prova": "1280x720 right connettore",
      "menu": {
        "x": 964,
        "y": 120.390625,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 right colonna",
      "menu": {
        "x": 990,
        "y": 287.1875,
        "w": 240,
        "h": 410.796875
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 right operatore",
      "menu": {
        "x": 990,
        "y": 384.78125,
        "w": 240,
        "h": 319.21875
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 right valori",
      "menu": {
        "x": 990,
        "y": 139.578125,
        "w": 240,
        "h": 266
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 right connettore esterno",
      "menu": {
        "x": 964,
        "y": 120.390625,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 right join confronto",
      "menu": {
        "x": 990,
        "y": 509.171875,
        "w": 240,
        "h": 146
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 right join valori della lista",
      "menu": {
        "x": 990,
        "y": 113.96875,
        "w": 240,
        "h": 276
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 bottom connettore",
      "menu": {
        "x": 454.765625,
        "y": 264,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 bottom colonna",
      "menu": {
        "x": 689,
        "y": 126.78125,
        "w": 267,
        "h": 410.796875
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 bottom operatore",
      "menu": {
        "x": 976,
        "y": 31.78125,
        "w": 267,
        "h": 506
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 bottom valori",
      "menu": {
        "x": 689,
        "y": 366.375,
        "w": 267,
        "h": 266
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 bottom connettore esterno",
      "menu": {
        "x": 459.515625,
        "y": 215,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 bottom join confronto",
      "menu": {
        "x": 976,
        "y": 396.78125,
        "w": 267,
        "h": 146
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 bottom join valori della lista",
      "menu": {
        "x": 689,
        "y": 304.578125,
        "w": 267,
        "h": 276
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 top connettore",
      "menu": {
        "x": 454.765625,
        "y": 361,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 top colonna",
      "menu": {
        "x": 689,
        "y": 336.78125,
        "w": 267,
        "h": 367.21875
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 top operatore",
      "menu": {
        "x": 976,
        "y": 336.78125,
        "w": 267,
        "h": 367.21875
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 top valori",
      "menu": {
        "x": 689,
        "y": 109.375,
        "w": 267,
        "h": 266
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 top connettore esterno",
      "menu": {
        "x": 459.515625,
        "y": 312,
        "w": 300,
        "h": 306
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 top join confronto",
      "menu": {
        "x": 976,
        "y": 341.78125,
        "w": 267,
        "h": 146
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    },
    {
      "prova": "1280x720 top join valori della lista",
      "menu": {
        "x": 689,
        "y": 47.578125,
        "w": 267,
        "h": 276
      },
      "finestra": {
        "w": 1280,
        "h": 720
      }
    }
  ]
}
```

