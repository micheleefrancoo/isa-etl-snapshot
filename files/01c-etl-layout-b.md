# 01c-etl-layout-b.md

File in questo blocco:

- `src/etl-layout/__tests__/golden/08-spostamento.json`
- `src/etl-layout/__tests__/golden/09-catena-riordino.json`
- `src/etl-layout/__tests__/golden/10-riordino-isolati.json`
- `src/etl-layout/__tests__/golden/10b-riordino-colonna-fitta.json`
- `src/etl-layout/__tests__/golden/11-organizzato.json`
- `src/etl-layout/__tests__/golden/12-output-generato.json`
- `src/etl-layout/__tests__/golden/13-output-organizzato.json`
- `src/etl-layout/__tests__/golden/corretti/06-corsie.json`
- `src/etl-layout/__tests__/golden/corretti/09-catena-riordino.json`
- `src/etl-layout/__tests__/properties.test.ts`

---

### `src/etl-layout/__tests__/golden/08-spostamento.json`

142 righe

```json
{
  "name": "08-spostamento",
  "description": "Un nodo spostato di poco (il cavo conserva il percorso) e di molto (lo cambia).",
  "type": "routes",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 312
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 416,
        "y": 442
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ],
    "moves": [
      {
        "id": "B",
        "dx": 8,
        "dy": -6
      },
      {
        "id": "B",
        "dx": -390,
        "dy": 260
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "B",
            "portA": 0,
            "portB": -1.5707963267948966,
            "shape": "L",
            "pts": [
              {
                "x": 192,
                "y": 356
              },
              {
                "x": 460,
                "y": 356
              },
              {
                "x": 460,
                "y": 442
              }
            ],
            "d": "M 192.00 356.00 L 449.00 356.00 Q 460.00 356.00 460.00 367.00 L 460.00 442.00"
          }
        ]
      },
      {
        "move": {
          "id": "B",
          "dx": 8,
          "dy": -6
        },
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "B",
            "portA": 0,
            "portB": -1.5707963267948966,
            "shape": "L",
            "pts": [
              {
                "x": 192,
                "y": 356
              },
              {
                "x": 468,
                "y": 356
              },
              {
                "x": 468,
                "y": 436
              }
            ],
            "d": "M 192.00 356.00 L 457.00 356.00 Q 468.00 356.00 468.00 367.00 L 468.00 436.00"
          }
        ]
      },
      {
        "move": {
          "id": "B",
          "dx": -390,
          "dy": 260
        },
        "passes": 2,
        "routes": [
          {
            "from": "A",
            "to": "B",
            "portA": 3.141592653589793,
            "portB": -1.5707963267948966,
            "shape": "L",
            "pts": [
              {
                "x": 104,
                "y": 356
              },
              {
                "x": 78,
                "y": 356
              },
              {
                "x": 78,
                "y": 696
              }
            ],
            "d": "M 104.00 356.00 L 89.00 356.00 Q 78.00 356.00 78.00 367.00 L 78.00 696.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/09-catena-riordino.json`

531 righe

```json
{
  "name": "09-catena-riordino",
  "description": "Catena dataset → filtro → join → ordina → esporta con un secondo dataset sul join, prima e dopo il riordino automatico.",
  "type": "autoLayout",
  "input": {
    "cards": [
      {
        "id": "D1",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 520,
        "y": 600
      },
      {
        "id": "F",
        "kind": "op",
        "components": ["filter"],
        "x": 130,
        "y": 130
      },
      {
        "id": "OF",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 780,
        "y": 390,
        "isOutput": true,
        "capacity": 1,
        "filled": 1
      },
      {
        "id": "D2",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 60,
        "y": 700
      },
      {
        "id": "J",
        "kind": "op",
        "components": ["join"],
        "x": 910,
        "y": 130
      },
      {
        "id": "OJ",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 300,
        "y": 450,
        "isOutput": true,
        "capacity": 2,
        "filled": 2
      },
      {
        "id": "S",
        "kind": "op",
        "components": ["sort"],
        "x": 1100,
        "y": 600
      },
      {
        "id": "OS",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 650,
        "y": 100,
        "isOutput": true,
        "capacity": 1,
        "filled": 1
      },
      {
        "id": "E",
        "kind": "op",
        "components": ["exportOp"],
        "x": 400,
        "y": 260
      }
    ],
    "links": [
      {
        "from": "D1",
        "to": "F"
      },
      {
        "from": "F",
        "to": "OF"
      },
      {
        "from": "OF",
        "to": "J"
      },
      {
        "from": "D2",
        "to": "J"
      },
      {
        "from": "J",
        "to": "OJ"
      },
      {
        "from": "OJ",
        "to": "S"
      },
      {
        "from": "S",
        "to": "OS"
      },
      {
        "from": "OS",
        "to": "E"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "before": {
      "passes": 8,
      "routes": [
        {
          "from": "D1",
          "to": "F",
          "portA": 3.141592653589793,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 520,
              "y": 644
            },
            {
              "x": 166.5,
              "y": 644
            },
            {
              "x": 166.5,
              "y": 218
            }
          ],
          "d": "M 520.00 644.00 L 177.50 644.00 Q 166.50 644.00 166.50 633.00 L 166.50 218.00"
        },
        {
          "from": "F",
          "to": "OF",
          "portA": 1.5707963267948966,
          "portB": 3.141592653589793,
          "shape": "L",
          "pts": [
            {
              "x": 181.5,
              "y": 218
            },
            {
              "x": 181.5,
              "y": 434
            },
            {
              "x": 780,
              "y": 434
            }
          ],
          "d": "M 181.50 218.00 L 181.50 423.00 Q 181.50 434.00 192.50 434.00 L 780.00 434.00"
        },
        {
          "from": "OF",
          "to": "J",
          "portA": -1.5707963267948966,
          "portB": 3.141592653589793,
          "shape": "L",
          "pts": [
            {
              "x": 824,
              "y": 390
            },
            {
              "x": 824,
              "y": 174
            },
            {
              "x": 910,
              "y": 174
            }
          ],
          "d": "M 824.00 390.00 L 824.00 185.00 Q 824.00 174.00 835.00 174.00 L 910.00 174.00"
        },
        {
          "from": "D2",
          "to": "J",
          "portA": -1.5707963267948966,
          "portB": -1.5707963267948966,
          "shape": "Z",
          "pts": [
            {
              "x": 104,
              "y": 700
            },
            {
              "x": 104,
              "y": 77
            },
            {
              "x": 946.5,
              "y": 77
            },
            {
              "x": 946.5,
              "y": 130
            }
          ],
          "d": "M 104.00 700.00 L 104.00 88.00 Q 104.00 77.00 115.00 77.00 L 935.50 77.00 Q 946.50 77.00 946.50 88.00 L 946.50 130.00"
        },
        {
          "from": "J",
          "to": "OJ",
          "portA": -1.5707963267948966,
          "portB": -1.5707963267948966,
          "shape": "Z",
          "pts": [
            {
              "x": 961.5,
              "y": 130
            },
            {
              "x": 961.5,
              "y": 90
            },
            {
              "x": 344,
              "y": 90
            },
            {
              "x": 344,
              "y": 450
            }
          ],
          "d": "M 961.50 130.00 L 961.50 101.00 Q 961.50 90.00 950.50 90.00 L 355.00 90.00 Q 344.00 90.00 344.00 101.00 L 344.00 450.00"
        },
        {
          "from": "OJ",
          "to": "S",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "Z",
          "pts": [
            {
              "x": 388,
              "y": 494
            },
            {
              "x": 744,
              "y": 494
            },
            {
              "x": 744,
              "y": 644
            },
            {
              "x": 1100,
              "y": 644
            }
          ],
          "d": "M 388.00 494.00 L 733.00 494.00 Q 744.00 494.00 744.00 505.00 L 744.00 633.00 Q 744.00 644.00 755.00 644.00 L 1100.00 644.00"
        },
        {
          "from": "S",
          "to": "OS",
          "portA": -1.5707963267948966,
          "portB": 0,
          "shape": "Z",
          "pts": [
            {
              "x": 1144,
              "y": 600
            },
            {
              "x": 1144,
              "y": 107
            },
            {
              "x": 770,
              "y": 107
            },
            {
              "x": 770,
              "y": 144
            },
            {
              "x": 738,
              "y": 144
            }
          ],
          "d": "M 1144.00 600.00 L 1144.00 118.00 Q 1144.00 107.00 1133.00 107.00 L 781.00 107.00 Q 770.00 107.00 770.00 118.00 L 770.00 133.00 Q 770.00 144.00 759.00 144.00 L 738.00 144.00"
        },
        {
          "from": "OS",
          "to": "E",
          "portA": 3.141592653589793,
          "portB": -1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 650,
              "y": 144
            },
            {
              "x": 444,
              "y": 144
            },
            {
              "x": 444,
              "y": 260
            }
          ],
          "d": "M 650.00 144.00 L 455.00 144.00 Q 444.00 144.00 444.00 155.00 L 444.00 260.00"
        }
      ]
    },
    "positions": [
      {
        "id": "D1",
        "x": 12,
        "y": 205
      },
      {
        "id": "F",
        "x": 178,
        "y": 205
      },
      {
        "id": "OF",
        "x": 344,
        "y": 136
      },
      {
        "id": "D2",
        "x": 344,
        "y": 274
      },
      {
        "id": "J",
        "x": 510,
        "y": 205
      },
      {
        "id": "OJ",
        "x": 676,
        "y": 205
      },
      {
        "id": "S",
        "x": 842,
        "y": 205
      },
      {
        "id": "OS",
        "x": 1008,
        "y": 205
      },
      {
        "id": "E",
        "x": 1174,
        "y": 205
      }
    ],
    "after": {
      "passes": 2,
      "routes": [
        {
          "from": "D1",
          "to": "F",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 100,
              "y": 249
            },
            {
              "x": 178,
              "y": 249
            }
          ],
          "d": "M 100.00 249.00 L 178.00 249.00"
        },
        {
          "from": "F",
          "to": "OF",
          "portA": -1.5707963267948966,
          "portB": 3.141592653589793,
          "shape": "L",
          "pts": [
            {
              "x": 222,
              "y": 205
            },
            {
              "x": 222,
              "y": 180
            },
            {
              "x": 344,
              "y": 180
            }
          ],
          "d": "M 222.00 205.00 L 222.00 191.00 Q 222.00 180.00 233.00 180.00 L 344.00 180.00"
        },
        {
          "from": "OF",
          "to": "J",
          "portA": 1.5707963267948966,
          "portB": 3.141592653589793,
          "shape": "L",
          "pts": [
            {
              "x": 388,
              "y": 224
            },
            {
              "x": 388,
              "y": 249
            },
            {
              "x": 510,
              "y": 249
            }
          ],
          "d": "M 388.00 224.00 L 388.00 238.00 Q 388.00 249.00 399.00 249.00 L 510.00 249.00"
        },
        {
          "from": "D2",
          "to": "J",
          "portA": 0,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 432,
              "y": 318
            },
            {
              "x": 554,
              "y": 318
            },
            {
              "x": 554,
              "y": 293
            }
          ],
          "d": "M 432.00 318.00 L 543.00 318.00 Q 554.00 318.00 554.00 307.00 L 554.00 293.00"
        },
        {
          "from": "J",
          "to": "OJ",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 598,
              "y": 249
            },
            {
              "x": 676,
              "y": 249
            }
          ],
          "d": "M 598.00 249.00 L 676.00 249.00"
        },
        {
          "from": "OJ",
          "to": "S",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 764,
              "y": 249
            },
            {
              "x": 842,
              "y": 249
            }
          ],
          "d": "M 764.00 249.00 L 842.00 249.00"
        },
        {
          "from": "S",
          "to": "OS",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 930,
              "y": 249
            },
            {
              "x": 1008,
              "y": 249
            }
          ],
          "d": "M 930.00 249.00 L 1008.00 249.00"
        },
        {
          "from": "OS",
          "to": "E",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 1096,
              "y": 249
            },
            {
              "x": 1174,
              "y": 249
            }
          ],
          "d": "M 1096.00 249.00 L 1174.00 249.00"
        }
      ]
    }
  }
}
```

### `src/etl-layout/__tests__/golden/10-riordino-isolati.json`

217 righe

```json
{
  "name": "10-riordino-isolati",
  "description": "Riordino con nodi isolati: colonna di parcheggio a destra del flusso.",
  "type": "autoLayout",
  "input": {
    "cards": [
      {
        "id": "I1",
        "kind": "op",
        "components": ["sort"],
        "x": 700,
        "y": 80
      },
      {
        "id": "D",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 300,
        "y": 500
      },
      {
        "id": "I2",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 90,
        "y": 90
      },
      {
        "id": "F",
        "kind": "op",
        "components": ["filter"],
        "x": 90,
        "y": 400
      },
      {
        "id": "O",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 900,
        "y": 600,
        "isOutput": true,
        "capacity": 1,
        "filled": 1
      },
      {
        "id": "I3",
        "kind": "op",
        "components": ["aggregate"],
        "x": 500,
        "y": 300
      },
      {
        "id": "I4",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 620,
        "y": 520
      },
      {
        "id": "I5",
        "kind": "op",
        "components": ["rename"],
        "x": 250,
        "y": 250
      }
    ],
    "links": [
      {
        "from": "D",
        "to": "F"
      },
      {
        "from": "F",
        "to": "O"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "before": {
      "passes": 2,
      "routes": [
        {
          "from": "D",
          "to": "F",
          "portA": 3.141592653589793,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 300,
              "y": 544
            },
            {
              "x": 134,
              "y": 544
            },
            {
              "x": 134,
              "y": 488
            }
          ],
          "d": "M 300.00 544.00 L 145.00 544.00 Q 134.00 544.00 134.00 533.00 L 134.00 488.00"
        },
        {
          "from": "F",
          "to": "O",
          "portA": 0,
          "portB": -1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 178,
              "y": 444
            },
            {
              "x": 944,
              "y": 444
            },
            {
              "x": 944,
              "y": 600
            }
          ],
          "d": "M 178.00 444.00 L 933.00 444.00 Q 944.00 444.00 944.00 455.00 L 944.00 600.00"
        }
      ]
    },
    "positions": [
      {
        "id": "I1",
        "x": 561,
        "y": 6
      },
      {
        "id": "D",
        "x": 63,
        "y": 205
      },
      {
        "id": "I2",
        "x": 561,
        "y": 144.3621535540758
      },
      {
        "id": "F",
        "x": 229,
        "y": 205
      },
      {
        "id": "O",
        "x": 395,
        "y": 205
      },
      {
        "id": "I3",
        "x": 561,
        "y": 419.42191406303857
      },
      {
        "id": "I4",
        "x": 561,
        "y": 557.580855639096
      },
      {
        "id": "I5",
        "x": 561,
        "y": 282.42191406303857
      }
    ],
    "after": {
      "passes": 2,
      "routes": [
        {
          "from": "D",
          "to": "F",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 151,
              "y": 249
            },
            {
              "x": 229,
              "y": 249
            }
          ],
          "d": "M 151.00 249.00 L 229.00 249.00"
        },
        {
          "from": "F",
          "to": "O",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 317,
              "y": 249
            },
            {
              "x": 395,
              "y": 249
            }
          ],
          "d": "M 317.00 249.00 L 395.00 249.00"
        }
      ]
    }
  }
}
```

### `src/etl-layout/__tests__/golden/10b-riordino-colonna-fitta.json`

159 righe

```json
{
  "name": "10b-riordino-colonna-fitta",
  "description": "Riordino con cinque nodi nella stessa colonna in uno stage alto 636 px: la distanza tra le righe scende al minimo (CARD + LABEL_H + 18).",
  "type": "autoLayout",
  "input": {
    "stageH": 636,
    "cards": [
      {
        "id": "D",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 300,
        "y": 500
      },
      {
        "id": "F",
        "kind": "op",
        "components": ["filter"],
        "x": 90,
        "y": 400
      },
      {
        "id": "I1",
        "kind": "op",
        "components": ["sort"],
        "x": 700,
        "y": 80
      },
      {
        "id": "I2",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 90,
        "y": 90
      },
      {
        "id": "I3",
        "kind": "op",
        "components": ["aggregate"],
        "x": 500,
        "y": 300
      },
      {
        "id": "I4",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 620,
        "y": 520
      },
      {
        "id": "I5",
        "kind": "op",
        "components": ["rename"],
        "x": 250,
        "y": 250
      }
    ],
    "links": [
      {
        "from": "D",
        "to": "F"
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 636
    },
    "before": {
      "passes": 2,
      "routes": [
        {
          "from": "D",
          "to": "F",
          "portA": 3.141592653589793,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 300,
              "y": 544
            },
            {
              "x": 134,
              "y": 544
            },
            {
              "x": 134,
              "y": 488
            }
          ],
          "d": "M 300.00 544.00 L 145.00 544.00 Q 134.00 544.00 134.00 533.00 L 134.00 488.00"
        }
      ]
    },
    "positions": [
      {
        "id": "D",
        "x": 146,
        "y": 263
      },
      {
        "id": "F",
        "x": 312,
        "y": 263
      },
      {
        "id": "I1",
        "x": 478,
        "y": 7
      },
      {
        "id": "I2",
        "x": 478,
        "y": 135
      },
      {
        "id": "I3",
        "x": 478,
        "y": 391
      },
      {
        "id": "I4",
        "x": 478,
        "y": 519
      },
      {
        "id": "I5",
        "x": 478,
        "y": 263
      }
    ],
    "after": {
      "passes": 2,
      "routes": [
        {
          "from": "D",
          "to": "F",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 234,
              "y": 307
            },
            {
              "x": 312,
              "y": 307
            }
          ],
          "d": "M 234.00 307.00 L 312.00 307.00"
        }
      ]
    }
  }
}
```

### `src/etl-layout/__tests__/golden/11-organizzato.json`

184 righe

```json
{
  "name": "11-organizzato",
  "description": "Modalità Organizzato: assegnazione iniziale delle postazioni e scambio di posto.",
  "type": "grid",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 40,
        "y": 30
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 170,
        "y": 40
      },
      {
        "id": "C",
        "kind": "op",
        "components": ["sort"],
        "x": 150,
        "y": 170
      },
      {
        "id": "D",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 30,
        "y": 180
      },
      {
        "id": "E",
        "kind": "op",
        "components": ["aggregate"],
        "x": 420,
        "y": 300
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ],
    "drops": [
      {
        "id": "E",
        "x": 150,
        "y": 20
      },
      {
        "id": "A",
        "x": 700,
        "y": 700
      }
    ]
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "drop": null,
        "positions": [
          {
            "id": "A",
            "x": 34,
            "y": 14,
            "slot": 0
          },
          {
            "id": "B",
            "x": 162,
            "y": 14,
            "slot": 1
          },
          {
            "id": "C",
            "x": 162,
            "y": 156,
            "slot": 21
          },
          {
            "id": "D",
            "x": 34,
            "y": 156,
            "slot": 20
          },
          {
            "id": "E",
            "x": 418,
            "y": 298,
            "slot": 43
          }
        ]
      },
      {
        "drop": {
          "id": "E",
          "x": 150,
          "y": 20
        },
        "positions": [
          {
            "id": "A",
            "x": 34,
            "y": 14,
            "slot": 0
          },
          {
            "id": "B",
            "x": 418,
            "y": 298,
            "slot": 43
          },
          {
            "id": "C",
            "x": 162,
            "y": 156,
            "slot": 21
          },
          {
            "id": "D",
            "x": 34,
            "y": 156,
            "slot": 20
          },
          {
            "id": "E",
            "x": 162,
            "y": 14,
            "slot": 1
          }
        ]
      },
      {
        "drop": {
          "id": "A",
          "x": 700,
          "y": 700
        },
        "positions": [
          {
            "id": "A",
            "x": 674,
            "y": 724,
            "slot": 105
          },
          {
            "id": "B",
            "x": 418,
            "y": 298,
            "slot": 43
          },
          {
            "id": "C",
            "x": 162,
            "y": 156,
            "slot": 21
          },
          {
            "id": "D",
            "x": 34,
            "y": 156,
            "slot": 20
          },
          {
            "id": "E",
            "x": 162,
            "y": 14,
            "slot": 1
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/12-output-generato.json`

67 righe

```json
{
  "name": "12-output-generato",
  "description": "Posizione dell'output generato in modalità Libero, con un nodo già nel posto ideale.",
  "type": "spawn",
  "input": {
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 104,
        "y": 312
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 312,
        "y": 312
      },
      {
        "id": "X",
        "kind": "op",
        "components": ["sort"],
        "x": 520,
        "y": 312
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ],
    "boxId": "B"
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "outputId": "out-0",
    "positions": [
      {
        "id": "A",
        "x": 104,
        "y": 312
      },
      {
        "id": "B",
        "x": 312,
        "y": 312
      },
      {
        "id": "X",
        "x": 520,
        "y": 312
      },
      {
        "id": "out-0",
        "x": 634,
        "y": 312
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/13-output-organizzato.json`

72 righe

```json
{
  "name": "13-output-organizzato",
  "description": "Posizione dell'output generato in modalità Organizzato: postazione a destra del box.",
  "type": "spawn",
  "input": {
    "mode": "grid",
    "cards": [
      {
        "id": "A",
        "kind": "dataset",
        "components": ["dataset"],
        "x": 40,
        "y": 300
      },
      {
        "id": "B",
        "kind": "op",
        "components": ["filter"],
        "x": 300,
        "y": 300
      },
      {
        "id": "X",
        "kind": "op",
        "components": ["sort"],
        "x": 430,
        "y": 300
      }
    ],
    "links": [
      {
        "from": "A",
        "to": "B"
      }
    ],
    "boxId": "B"
  },
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "outputId": "out-1",
    "positions": [
      {
        "id": "A",
        "x": 34,
        "y": 298,
        "slot": 40
      },
      {
        "id": "B",
        "x": 290,
        "y": 298,
        "slot": 42
      },
      {
        "id": "X",
        "x": 418,
        "y": 298,
        "slot": 43
      },
      {
        "id": "out-1",
        "x": 546,
        "y": 298,
        "slot": 44
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/corretti/06-corsie.json`

115 righe

```json
{
  "name": "06-corsie",
  "description": "Più cavi nello stesso corridoio (snodi ammessi: 2, perché nascano forme a Z).",
  "motivo": "Convergenza dei cavi (Fase 2.1): aggiornamento sequenziale con miglioramento stretto. Il prototipo aggiorna i cavi in simultanea e qui non converge o converge altrove.",
  "confronto": [
    {
      "risultato": "iniziale",
      "prototipo": {
        "passate": 8,
        "incroci": 1,
        "snodi": 6,
        "lunghezza": 2366
      },
      "corretto": {
        "passate": 2,
        "incroci": 0,
        "snodi": 6,
        "lunghezza": 2232
      }
    }
  ],
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "steps": [
      {
        "move": null,
        "passes": 2,
        "routes": [
          {
            "from": "A1",
            "to": "B1",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 148
              },
              {
                "x": 369,
                "y": 148
              },
              {
                "x": 369,
                "y": 538
              },
              {
                "x": 546,
                "y": 538
              }
            ],
            "d": "M 192.00 148.00 L 358.00 148.00 Q 369.00 148.00 369.00 159.00 L 369.00 527.00 Q 369.00 538.00 380.00 538.00 L 546.00 538.00"
          },
          {
            "from": "A2",
            "to": "B2",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 278
              },
              {
                "x": 215,
                "y": 278
              },
              {
                "x": 215,
                "y": 668
              },
              {
                "x": 546,
                "y": 668
              }
            ],
            "d": "M 192.00 278.00 L 204.00 278.00 Q 215.00 278.00 215.00 289.00 L 215.00 657.00 Q 215.00 668.00 226.00 668.00 L 546.00 668.00"
          },
          {
            "from": "A3",
            "to": "B3",
            "portA": 0,
            "portB": 3.141592653589793,
            "shape": "Z",
            "pts": [
              {
                "x": 192,
                "y": 408
              },
              {
                "x": 201,
                "y": 408
              },
              {
                "x": 201,
                "y": 798
              },
              {
                "x": 546,
                "y": 798
              }
            ],
            "d": "M 192.00 408.00 L 196.50 408.00 Q 201.00 408.00 201.00 412.50 L 201.00 787.00 Q 201.00 798.00 212.00 798.00 L 546.00 798.00"
          }
        ]
      }
    ]
  }
}
```

### `src/etl-layout/__tests__/golden/corretti/09-catena-riordino.json`

445 righe

```json
{
  "name": "09-catena-riordino",
  "description": "Catena dataset → filtro → join → ordina → esporta con un secondo dataset sul join, prima e dopo il riordino automatico.",
  "motivo": "Convergenza dei cavi (Fase 2.1): aggiornamento sequenziale con miglioramento stretto. Il prototipo aggiorna i cavi in simultanea e qui non converge o converge altrove.",
  "confronto": [
    {
      "risultato": "prima del riordino",
      "prototipo": {
        "passate": 8,
        "incroci": 4,
        "snodi": 13,
        "lunghezza": 6552
      },
      "corretto": {
        "passate": 3,
        "incroci": 4,
        "snodi": 11,
        "lunghezza": 6310
      }
    },
    {
      "risultato": "dopo il riordino",
      "prototipo": {
        "passate": 2,
        "incroci": 0,
        "snodi": 3,
        "lunghezza": 831
      },
      "corretto": {
        "passate": 2,
        "incroci": 0,
        "snodi": 3,
        "lunghezza": 831
      }
    }
  ],
  "expected": {
    "stage": {
      "w": 712,
      "h": 520
    },
    "before": {
      "passes": 3,
      "routes": [
        {
          "from": "D1",
          "to": "F",
          "portA": 3.141592653589793,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 520,
              "y": 644
            },
            {
              "x": 166.5,
              "y": 644
            },
            {
              "x": 166.5,
              "y": 218
            }
          ],
          "d": "M 520.00 644.00 L 177.50 644.00 Q 166.50 644.00 166.50 633.00 L 166.50 218.00"
        },
        {
          "from": "F",
          "to": "OF",
          "portA": 1.5707963267948966,
          "portB": 3.141592653589793,
          "shape": "L",
          "pts": [
            {
              "x": 181.5,
              "y": 218
            },
            {
              "x": 181.5,
              "y": 434
            },
            {
              "x": 780,
              "y": 434
            }
          ],
          "d": "M 181.50 218.00 L 181.50 423.00 Q 181.50 434.00 192.50 434.00 L 780.00 434.00"
        },
        {
          "from": "OF",
          "to": "J",
          "portA": 0,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 868,
              "y": 434
            },
            {
              "x": 954,
              "y": 434
            },
            {
              "x": 954,
              "y": 218
            }
          ],
          "d": "M 868.00 434.00 L 943.00 434.00 Q 954.00 434.00 954.00 423.00 L 954.00 218.00"
        },
        {
          "from": "D2",
          "to": "J",
          "portA": -1.5707963267948966,
          "portB": -1.5707963267948966,
          "shape": "Z",
          "pts": [
            {
              "x": 104,
              "y": 700
            },
            {
              "x": 104,
              "y": 77
            },
            {
              "x": 954,
              "y": 77
            },
            {
              "x": 954,
              "y": 130
            }
          ],
          "d": "M 104.00 700.00 L 104.00 88.00 Q 104.00 77.00 115.00 77.00 L 943.00 77.00 Q 954.00 77.00 954.00 88.00 L 954.00 130.00"
        },
        {
          "from": "J",
          "to": "OJ",
          "portA": 3.141592653589793,
          "portB": 0,
          "shape": "Z",
          "pts": [
            {
              "x": 910,
              "y": 174
            },
            {
              "x": 757,
              "y": 174
            },
            {
              "x": 757,
              "y": 494
            },
            {
              "x": 388,
              "y": 494
            }
          ],
          "d": "M 910.00 174.00 L 768.00 174.00 Q 757.00 174.00 757.00 185.00 L 757.00 483.00 Q 757.00 494.00 746.00 494.00 L 388.00 494.00"
        },
        {
          "from": "OJ",
          "to": "S",
          "portA": 1.5707963267948966,
          "portB": -1.5707963267948966,
          "shape": "Z",
          "pts": [
            {
              "x": 344,
              "y": 538
            },
            {
              "x": 344,
              "y": 569
            },
            {
              "x": 1144,
              "y": 569
            },
            {
              "x": 1144,
              "y": 600
            }
          ],
          "d": "M 344.00 538.00 L 344.00 558.00 Q 344.00 569.00 355.00 569.00 L 1133.00 569.00 Q 1144.00 569.00 1144.00 580.00 L 1144.00 600.00"
        },
        {
          "from": "S",
          "to": "OS",
          "portA": 3.141592653589793,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 1100,
              "y": 644
            },
            {
              "x": 694,
              "y": 644
            },
            {
              "x": 694,
              "y": 188
            }
          ],
          "d": "M 1100.00 644.00 L 705.00 644.00 Q 694.00 644.00 694.00 633.00 L 694.00 188.00"
        },
        {
          "from": "OS",
          "to": "E",
          "portA": 3.141592653589793,
          "portB": -1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 650,
              "y": 144
            },
            {
              "x": 444,
              "y": 144
            },
            {
              "x": 444,
              "y": 260
            }
          ],
          "d": "M 650.00 144.00 L 455.00 144.00 Q 444.00 144.00 444.00 155.00 L 444.00 260.00"
        }
      ]
    },
    "positions": [
      {
        "id": "D1",
        "x": 12,
        "y": 205
      },
      {
        "id": "F",
        "x": 178,
        "y": 205
      },
      {
        "id": "OF",
        "x": 344,
        "y": 136
      },
      {
        "id": "D2",
        "x": 344,
        "y": 274
      },
      {
        "id": "J",
        "x": 510,
        "y": 205
      },
      {
        "id": "OJ",
        "x": 676,
        "y": 205
      },
      {
        "id": "S",
        "x": 842,
        "y": 205
      },
      {
        "id": "OS",
        "x": 1008,
        "y": 205
      },
      {
        "id": "E",
        "x": 1174,
        "y": 205
      }
    ],
    "after": {
      "passes": 2,
      "routes": [
        {
          "from": "D1",
          "to": "F",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 100,
              "y": 249
            },
            {
              "x": 178,
              "y": 249
            }
          ],
          "d": "M 100.00 249.00 L 178.00 249.00"
        },
        {
          "from": "F",
          "to": "OF",
          "portA": -1.5707963267948966,
          "portB": 3.141592653589793,
          "shape": "L",
          "pts": [
            {
              "x": 222,
              "y": 205
            },
            {
              "x": 222,
              "y": 180
            },
            {
              "x": 344,
              "y": 180
            }
          ],
          "d": "M 222.00 205.00 L 222.00 191.00 Q 222.00 180.00 233.00 180.00 L 344.00 180.00"
        },
        {
          "from": "OF",
          "to": "J",
          "portA": 0,
          "portB": -1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 432,
              "y": 180
            },
            {
              "x": 554,
              "y": 180
            },
            {
              "x": 554,
              "y": 205
            }
          ],
          "d": "M 432.00 180.00 L 543.00 180.00 Q 554.00 180.00 554.00 191.00 L 554.00 205.00"
        },
        {
          "from": "D2",
          "to": "J",
          "portA": 0,
          "portB": 1.5707963267948966,
          "shape": "L",
          "pts": [
            {
              "x": 432,
              "y": 318
            },
            {
              "x": 554,
              "y": 318
            },
            {
              "x": 554,
              "y": 293
            }
          ],
          "d": "M 432.00 318.00 L 543.00 318.00 Q 554.00 318.00 554.00 307.00 L 554.00 293.00"
        },
        {
          "from": "J",
          "to": "OJ",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 598,
              "y": 249
            },
            {
              "x": 676,
              "y": 249
            }
          ],
          "d": "M 598.00 249.00 L 676.00 249.00"
        },
        {
          "from": "OJ",
          "to": "S",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 764,
              "y": 249
            },
            {
              "x": 842,
              "y": 249
            }
          ],
          "d": "M 764.00 249.00 L 842.00 249.00"
        },
        {
          "from": "S",
          "to": "OS",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 930,
              "y": 249
            },
            {
              "x": 1008,
              "y": 249
            }
          ],
          "d": "M 930.00 249.00 L 1008.00 249.00"
        },
        {
          "from": "OS",
          "to": "E",
          "portA": 0,
          "portB": 3.141592653589793,
          "shape": "straight",
          "pts": [
            {
              "x": 1096,
              "y": 249
            },
            {
              "x": 1174,
              "y": 249
            }
          ],
          "d": "M 1096.00 249.00 L 1174.00 249.00"
        }
      ]
    }
  }
}
```

### `src/etl-layout/__tests__/properties.test.ts`

282 righe

```ts
import { describe, expect, it } from "vitest";
import { defaultParams } from "../../etl-core";
import type { Card, ComponentId, Graph, Link } from "../../etl-core";
import {
  CARD,
  EPS,
  SETTLE_MAX_PASSES,
  assignSlots,
  autoLayout,
  dropInSlot,
  layoutLinks,
  setSlot,
  chooseRoute,
  nodeCenter,
  obstacleRect,
  routeCandidates,
  routeCost,
  settleLinks,
} from "..";
import type { LinkRoute, Point, Rect } from "..";

/** Generatore pseudo-casuale deterministico (mulberry32). */
function rng(seed: number): () => number {
  let a = seed >>> 0;
  return () => {
    a = (a + 0x6d2b79f5) >>> 0;
    let t = a;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

const OPS: ComponentId[] = ["filter", "sort", "join", "aggregate", "rename"];

/** Un grafo casuale: nodi non sovrapposti su una griglia larga, cavi da un nodo precedente a uno successivo. */
function randomGraph(seed: number, n: number, m: number): Graph {
  const r = rng(seed);
  const cards: Record<string, Card> = {};
  const taken = new Set<string>();
  for (let i = 0; i < n; i++) {
    let cx = 0;
    let cy = 0;
    do {
      cx = Math.floor(r() * 7);
      cy = Math.floor(r() * 5);
    } while (taken.has(cx + ":" + cy));
    taken.add(cx + ":" + cy);
    const isDs = i === 0 || r() < 0.3;
    const comp: ComponentId = isDs ? "dataset" : (OPS[Math.floor(r() * OPS.length)] as ComponentId);
    const id = "n" + i;
    cards[id] = {
      id,
      kind: isDs ? "dataset" : "op",
      components: [comp],
      params: [defaultParams(comp)],
      name: id,
      x: 60 + cx * 190 + Math.round((r() - 0.5) * 40),
      y: 60 + cy * 170 + Math.round((r() - 0.5) * 40),
    };
  }
  const ids = Object.keys(cards);
  const links: Link[] = [];
  for (let k = 0; k < m; k++) {
    const i = Math.floor(r() * (ids.length - 1));
    const j = i + 1 + Math.floor(r() * (ids.length - 1 - i));
    const l = { from: ids[i] as string, to: ids[j] as string };
    if (!links.some((o) => o.from === l.from && o.to === l.to)) links.push(l);
  }
  return { cards, links };
}

const SEEDS = Array.from({ length: 40 }, (_, i) => 1000 + i * 17);

function segments(pts: readonly Point[]): [Point, Point][] {
  const out: [Point, Point][] = [];
  for (let i = 0; i < pts.length - 1; i++) out.push([pts[i] as Point, pts[i + 1] as Point]);
  return out;
}

function obstaclesFor(graph: Graph, l: Link): Rect[] {
  return Object.keys(graph.cards)
    .filter((id) => id !== l.from && id !== l.to)
    .map((id) => obstacleRect(graph.cards[id] as Card));
}

describe("proprietà dei percorsi su scenari generati", () => {
  it("ogni segmento è orizzontale o verticale", { timeout: 60000 }, () => {
    for (const seed of SEEDS) {
      const g = randomGraph(seed, 8, 7);
      for (const maxBends of [0, 1, 2]) {
        const { routes } = settleLinks(g, {}, { maxBends });
        for (const r of Object.values(routes)) {
          for (const [a, b] of segments(r.pts)) {
            const horizontal = Math.abs(a.y - b.y) < EPS;
            const vertical = Math.abs(a.x - b.x) < EPS;
            expect(horizontal || vertical, `seed ${seed} ${r.from}->${r.to}`).toBe(true);
          }
        }
      }
    }
  });

  it("nessuna inversione di direzione lungo un asse e nessun punto superfluo", () => {
    for (const seed of SEEDS) {
      const { routes } = settleLinks(randomGraph(seed, 8, 7));
      for (const r of Object.values(routes)) {
        for (let i = 1; i < r.pts.length - 1; i++) {
          const a = r.pts[i - 1] as Point;
          const b = r.pts[i] as Point;
          const c = r.pts[i + 1] as Point;
          const collinear =
            (Math.abs(a.x - b.x) < EPS && Math.abs(b.x - c.x) < EPS) ||
            (Math.abs(a.y - b.y) < EPS && Math.abs(b.y - c.y) < EPS);
          expect(collinear, `seed ${seed} ${r.from}->${r.to} punto ${i}`).toBe(false);
        }
      }
    }
  });

  it("un percorso dritto ha sempre esattamente 2 punti (correzione Fase 2.1)", () => {
    let straight = 0;
    for (const seed of SEEDS) {
      const { routes } = settleLinks(randomGraph(seed, 8, 9));
      for (const r of Object.values(routes).filter((x: LinkRoute) => x.shape.kind === "straight")) {
        straight++;
        expect(r.pts.length, `seed ${seed} ${r.from}->${r.to}`).toBe(2);
      }
    }
    expect(straight).toBeGreaterThan(20);
  });

  it("un dritto quasi allineato (sotto 1,5 px) è perfettamente dritto (correzione Fase 2.1)", () => {
    for (const dy of [0.2, 0.7, 1, 1.4]) {
      const g = randomGraph(1, 2, 0);
      const [a, b] = Object.values(g.cards) as [Card, Card];
      const two: Graph = {
        cards: { A: { ...a, id: "A", x: 100, y: 300 }, B: { ...b, id: "B", x: 400, y: 300 + dy } },
        links: [{ from: "A", to: "B" }],
      };
      const r = settleLinks(two).routes["A|B"] as LinkRoute;
      expect(r.shape.kind).toBe("straight");
      expect(r.pts).toHaveLength(2);
      expect(Math.abs((r.pts[0] as Point).y - (r.pts[1] as Point).y)).toBeLessThan(1e-9);
    }
  });

  it("chooseRoute non attraversa nodi quando esiste un'alternativa che non li attraversa", () => {
    let checked = 0;
    for (const seed of SEEDS) {
      const g = randomGraph(seed, 9, 6);
      for (const l of g.links) {
        const a = nodeCenter(g.cards[l.from] as Card);
        const b = nodeCenter(g.cards[l.to] as Card);
        const obstacles = obstaclesFor(g, l);
        const cands = routeCandidates(a, b, CARD / 2, obstacles);
        const choice = chooseRoute(a, b, CARD / 2, obstacles);
        if (cands.some((c) => c.cost === 0)) {
          expect(choice?.cost, `seed ${seed} ${l.from}->${l.to}`).toBe(0);
          checked++;
        }
      }
    }
    expect(checked).toBeGreaterThan(50);
  });

  it("un cavo isolato, a regime, non attraversa nodi quando esiste un'alternativa", () => {
    for (const seed of SEEDS) {
      const g = randomGraph(seed, 9, 6);
      for (const l of g.links) {
        const single: Graph = { cards: g.cards, links: [l] };
        const a = nodeCenter(g.cards[l.from] as Card);
        const b = nodeCenter(g.cards[l.to] as Card);
        const obstacles = obstaclesFor(g, l);
        if (!routeCandidates(a, b, CARD / 2, obstacles).some((c) => c.cost === 0)) continue;
        const r = settleLinks(single).routes[l.from + "|" + l.to] as LinkRoute;
        expect(routeCost(r.pts, obstacles), `seed ${seed} ${l.from}->${l.to}`).toBe(0);
      }
    }
  });

  it("il numero di snodi rispetta il limite quando è possibile senza attraversare nodi né ripiegare", () => {
    let checked = 0;
    for (const seed of SEEDS) {
      const g = randomGraph(seed, 9, 6);
      for (const maxBends of [0, 1, 2]) {
        for (const l of g.links) {
          const a = nodeCenter(g.cards[l.from] as Card);
          const b = nodeCenter(g.cards[l.to] as Card);
          const obstacles = obstaclesFor(g, l);
          const cands = routeCandidates(a, b, CARD / 2, obstacles, { maxBends });
          const clean = cands.some((c) => c.cost === 0 && c.bends <= maxBends && c.back === 0);
          if (!clean) continue;
          const choice = chooseRoute(a, b, CARD / 2, obstacles, { maxBends });
          expect(
            choice?.bends,
            `seed ${seed} ${l.from}->${l.to} max ${maxBends}`,
          ).toBeLessThanOrEqual(maxBends);
          checked++;
        }
      }
    }
    expect(checked).toBeGreaterThan(100);
  });
});

describe("convergenza dei cavi (correzione Fase 2.1)", () => {
  const MANY = Array.from({ length: 200 }, (_, i) => 7000 + i * 31);

  it(
    "su 200 scenari, con maxBends 1 e 2, settleLinks termina prima del limite e il risultato è un punto fisso",
    { timeout: 300000 },
    () => {
      for (const maxBends of [1, 2]) {
        for (const seed of MANY) {
          const g = randomGraph(seed, 7, 7);
          const { routes, passes } = settleLinks(g, {}, { maxBends });
          expect(passes, `seed ${seed} maxBends ${maxBends}`).toBeLessThan(SETTLE_MAX_PASSES);
          const again = layoutLinks(g, routes, { maxBends });
          expect(Object.keys(again)).toEqual(Object.keys(routes));
          for (const [k, r] of Object.entries(routes)) {
            const q = again[k] as LinkRoute;
            expect([q.portA, q.portB, q.pts], `seed ${seed} maxBends ${maxBends} ${k}`).toEqual([
              r.portA,
              r.portB,
              r.pts,
            ]);
          }
        }
      }
    },
  );
});

describe("modalità Organizzato (correzione Fase 2.1)", () => {
  it("dopo qualunque dropInSlot ogni nodo ha una postazione e nessuna postazione ha due nodi", () => {
    for (const seed of SEEDS) {
      const r = rng(seed);
      let g = assignSlots(randomGraph(seed, 10, 0));
      const ids = Object.keys(g.cards);
      for (let k = 0; k < 6; k++) {
        const id = ids[Math.floor(r() * ids.length)] as string;
        // a volte il nodo rilasciato è privo di postazione (es. appena creato)
        if (r() < 0.4) g = setSlot(g, id, undefined);
        g = dropInSlot(g, id, r() * 1400, r() * 900);
        const slots = Object.values(g.cards).map((c) => c.slot);
        expect(
          slots.every((x) => x !== undefined),
          `seed ${seed} passo ${k}`,
        ).toBe(true);
        expect(new Set(slots).size, `seed ${seed} passo ${k}`).toBe(slots.length);
      }
    }
  });
});

describe("riordino automatico", () => {
  it("è deterministico e non modifica il grafo che riceve", () => {
    for (const seed of SEEDS) {
      const g = randomGraph(seed, 10, 8);
      const snapshot = JSON.stringify(g);
      const viewport = { w: 712, h: 520 };
      const first = autoLayout(g, { viewport });
      const second = autoLayout(g, { viewport });
      expect(JSON.stringify(second)).toBe(JSON.stringify(first));
      expect(JSON.stringify(g)).toBe(snapshot);
    }
  });

  it("i nodi collegati finiscono in colonne crescenti lungo il flusso", () => {
    for (const seed of SEEDS) {
      const g = randomGraph(seed, 10, 8);
      const out = autoLayout(g, { viewport: { w: 712, h: 520 } });
      for (const l of g.links) {
        const a = out.cards[l.from] as Card;
        const b = out.cards[l.to] as Card;
        expect(b.x, `seed ${seed} ${l.from}->${l.to}`).toBeGreaterThan(a.x);
      }
    }
  });
});
```

