# 01c-etl-layout-b.md

File in questo blocco:

- `src/etl-layout/__tests__/golden/09-catena-riordino.json`
- `src/etl-layout/__tests__/golden/10-riordino-isolati.json`
- `src/etl-layout/__tests__/golden/10b-riordino-colonna-fitta.json`
- `src/etl-layout/__tests__/golden/11-organizzato.json`
- `src/etl-layout/__tests__/golden/12-output-generato.json`
- `src/etl-layout/__tests__/golden/13-output-organizzato.json`
- `src/etl-layout/__tests__/properties.test.ts`
- `src/etl-layout/__tests__/unit.test.ts`
- `src/etl-layout/autoLayout.ts`
- `src/etl-layout/constants.ts`

---

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

### `src/etl-layout/__tests__/properties.test.ts`

218 righe

```ts
import { describe, expect, it } from "vitest";
import { defaultParams } from "../../etl-core";
import type { Card, ComponentId, Graph, Link } from "../../etl-core";
import {
  CARD,
  EPS,
  STRAIGHT_EPS,
  autoLayout,
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

  it("scostamenti simmetrici: un percorso dritto resta un solo segmento (o uno scalino sotto STRAIGHT_EPS)", () => {
    let straight = 0;
    for (const seed of SEEDS) {
      const { routes } = settleLinks(randomGraph(seed, 8, 9));
      for (const r of Object.values(routes).filter((x: LinkRoute) => x.shape.kind === "straight")) {
        straight++;
        if (r.pts.length === 2) continue;
        // NOTE_DIVERGENZE.md: agganci disallineati di meno di STRAIGHT_EPS non scorrono
        const a = r.pts[0] as Point;
        const b = r.pts[r.pts.length - 1] as Point;
        const jog = Math.min(Math.abs(a.x - b.x), Math.abs(a.y - b.y));
        expect(jog, `seed ${seed} ${r.from}->${r.to}`).toBeLessThan(STRAIGHT_EPS);
      }
    }
    expect(straight).toBeGreaterThan(20);
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

### `src/etl-layout/__tests__/unit.test.ts`

306 righe

```ts
import { describe, expect, it } from "vitest";
import { defaultParams, detachStep, insertOnLink } from "../../etl-core";
import type { Card, ComponentId, Graph } from "../../etl-core";
import {
  CARD,
  GRID,
  LABEL_H,
  PORTS,
  WORLD_H,
  WORLD_W,
  anyOverlap,
  assignSlots,
  borderPoint,
  clampPoint,
  computeSlots,
  detachPositionFn,
  displace,
  distanceToRoute,
  dropFree,
  dropInSlot,
  freeSpot,
  insertPosition,
  linkAt,
  moveNode,
  nodeAt,
  nodePorts,
  outputPositionFn,
  relocateAfterMerge,
  resolveOverlaps,
  roundedPath,
  separateWhileDragging,
  settleLinks,
  settleNewNode,
  snapToGrid,
} from "..";

function card(
  id: string,
  comp: ComponentId,
  x: number,
  y: number,
  extra: Partial<Card> = {},
): Card {
  return {
    id,
    kind: comp === "dataset" ? "dataset" : "op",
    components: [comp],
    params: [defaultParams(comp)],
    name: id,
    x,
    y,
    ...extra,
  };
}

function graphOf(cards: Card[], links: { from: string; to: string }[] = []): Graph {
  const rec: Record<string, Card> = {};
  for (const c of cards) rec[c.id] = c;
  return { cards: rec, links };
}

describe("porte", () => {
  it("quattro porte al centro dei lati: destra, sotto, sinistra, sopra", () => {
    const ports = nodePorts({ x: 100, y: 200 });
    expect(ports.map((p) => ({ x: Math.round(p.anchor.x), y: Math.round(p.anchor.y) }))).toEqual([
      { x: 188, y: 244 },
      { x: 144, y: 288 },
      { x: 100, y: 244 },
      { x: 144, y: 200 },
    ]);
    expect(ports.map((p) => p.axis)).toEqual([
      { x: 1, y: 0 },
      { x: 0, y: 1 },
      { x: -1, y: 0 },
      { x: 0, y: -1 },
    ]);
  });

  it("borderPoint proietta sul bordo del quadrato anche in diagonale", () => {
    const p = borderPoint(0, 0, 44, Math.PI / 4);
    expect(p.x).toBeCloseTo(44, 9);
    expect(p.y).toBeCloseTo(44, 9);
  });
});

describe("percorso SVG", () => {
  it("due punti: solo M e L", () => {
    expect(
      roundedPath(
        [
          { x: 0, y: 0 },
          { x: 10, y: 0 },
        ],
        11,
      ),
    ).toBe("M 0.00 0.00 L 10.00 0.00");
  });

  it("uno snodo: raccordo Q con raggio limitato a metà del tratto più corto", () => {
    expect(
      roundedPath(
        [
          { x: 0, y: 0 },
          { x: 100, y: 0 },
          { x: 100, y: 10 },
        ],
        11,
      ),
    ).toBe("M 0.00 0.00 L 95.00 0.00 Q 100.00 0.00 100.00 5.00 L 100.00 10.00");
  });
});

describe("individuazione", () => {
  const g = graphOf([card("A", "dataset", 100, 100), card("B", "filter", 140, 120)]);

  it("nodeAt: vince il nodo creato dopo; l'etichetta conta; ignoreId salta il nodo in mano", () => {
    expect(nodeAt(g, { x: 150, y: 150 })).toBe("B");
    expect(nodeAt(g, { x: 150, y: 150 }, "B")).toBe("A");
    expect(nodeAt(g, { x: 102, y: 105 })).toBe("A");
    expect(nodeAt(g, { x: 184, y: 120 + CARD + 12 })).toBe("B");
    expect(nodeAt(g, { x: 20, y: 20 })).toBeNull();
    // etichetta: 8 px sotto il quadrato, larga 96 px centrata (da x - 4 a x + 92)
    const solo = graphOf([card("S", "dataset", 500, 500)]);
    expect(nodeAt(solo, { x: 544, y: 500 + CARD + 7.5 })).toBeNull();
    expect(nodeAt(solo, { x: 544, y: 500 + CARD + 8.5 })).toBe("S");
    expect(nodeAt(solo, { x: 500 - 3.75, y: 600 })).toBe("S");
    expect(nodeAt(solo, { x: 500 - 4.25, y: 600 })).toBeNull();
  });

  it("linkAt: entro 8 px dal percorso, anche sulla curva del raccordo", () => {
    const two = graphOf(
      [card("A", "dataset", 100, 300), card("B", "filter", 400, 300)],
      [{ from: "A", to: "B" }],
    );
    const { routes } = settleLinks(two);
    expect(linkAt(routes, { x: 300, y: 344 + 7 })).toBe("A|B");
    expect(linkAt(routes, { x: 300, y: 344 + 8.25 })).toBeNull();
    const l = settleLinks(
      graphOf(
        [card("A", "dataset", 104, 312), card("B", "filter", 416, 442)],
        [{ from: "A", to: "B" }],
      ),
    ).routes;
    const route = l["A|B"];
    expect(route?.shape.kind).toBe("L");
    // vertice dello snodo a L (460, 356): la curva passa a ~3,2 px dal vertice
    expect(distanceToRoute({ x: 460, y: 356 }, route as NonNullable<typeof route>)).toBeLessThan(4);
  });
});

describe("modalità Libero", () => {
  it("limiti del mondo 2600 × 1600 con margine 6 (valori letterali)", () => {
    expect(clampPoint({ x: 99999, y: 99999 })).toEqual({ x: 2506, y: 1484 });
    expect(clampPoint({ x: -1, y: -1 })).toEqual({ x: 6, y: 6 });
  });

  it("clampPoint e snapToGrid", () => {
    expect(clampPoint({ x: -50, y: 99999 })).toEqual({ x: 6, y: WORLD_H - CARD - LABEL_H - 6 });
    expect(clampPoint({ x: 99999, y: 0 })).toEqual({ x: WORLD_W - CARD - 6, y: 6 });
    expect(snapToGrid({ x: 40, y: 38 })).toEqual({ x: 2 * GRID, y: GRID });
  });

  it("resolveOverlaps separa due nodi sovrapposti; il nodo fisso non si muove", () => {
    const g = graphOf([card("A", "dataset", 300, 300), card("B", "dataset", 320, 310)]);
    const out = resolveOverlaps(g, { fixedId: "A" });
    expect(out.cards["A"]).toMatchObject({ x: 300, y: 300 });
    expect(anyOverlap(out)).toBe(false);
  });

  it("durante il trascinamento si scansano solo le coppie incompatibili", () => {
    const g = graphOf([
      card("D", "dataset", 300, 300),
      card("E", "dataset", 330, 300),
      card("F", "filter", 280, 310),
    ]);
    const out = separateWhileDragging(g, "D");
    expect(out.cards["D"]).toMatchObject({ x: 300, y: 300 });
    expect(out.cards["E"]?.x).not.toBe(330);
    expect(out.cards["F"]).toMatchObject({ x: 280, y: 310 });
  });

  it("dropFree riallinea alla griglia tutti tranne il nodo rilasciato", () => {
    const g = graphOf([card("A", "dataset", 301, 299), card("B", "dataset", 600, 601)]);
    const out = dropFree(g, "A");
    expect(out.cards["A"]).toMatchObject({ x: 301, y: 299 });
    expect(out.cards["B"]).toMatchObject({ x: 598, y: 598 });
  });

  it("freeSpot cerca a spirale un punto libero", () => {
    const g = graphOf([card("A", "dataset", 300, 300)]);
    expect(freeSpot(g, 800, 800)).toEqual({ x: 800, y: 800 });
    expect(freeSpot(g, 300, 300)).toEqual({ x: 300 + CARD + 26, y: 300 });
  });

  it("displace spinge via il nodo sotto quello trascinato", () => {
    const g = graphOf([card("D", "dataset", 300, 300), card("E", "dataset", 340, 300)]);
    const out = displace(g, "D", "E");
    expect(out.cards["D"]).toMatchObject({ x: 300, y: 300 });
    expect((out.cards["E"] as Card).x).toBeGreaterThanOrEqual(300 + CARD + 28);
  });
});

describe("modalità Organizzato", () => {
  it("assegnazione: dall'alto a sinistra, la postazione libera più vicina", () => {
    const g = graphOf([card("A", "dataset", 40, 30), card("B", "filter", 60, 40)]);
    const out = assignSlots(g);
    const slots = computeSlots();
    expect(out.cards["A"]?.slot).toBe(0);
    expect(out.cards["B"]?.slot).toBe(1);
    expect(out.cards["B"]).toMatchObject(slots[1] as object);
  });

  it("scambio di posto: chi occupa la postazione prende quella lasciata libera", () => {
    const g = assignSlots(graphOf([card("A", "dataset", 40, 30), card("B", "filter", 170, 40)]));
    const slots = computeSlots();
    const target = slots[1] as { x: number; y: number };
    const out = dropInSlot(g, "A", target.x + 5, target.y + 5);
    expect(out.cards["A"]?.slot).toBe(1);
    expect(out.cards["B"]?.slot).toBe(0);
    expect(out.cards["B"]).toMatchObject(slots[0] as object);
  });
});

describe("posizionamento dei nodi generati", () => {
  it("passaggio sganciato: al punto di rilascio, oppure vicino al box", () => {
    const g = graphOf([card("B", "filter", 300, 300)]);
    expect(detachPositionFn({ dropPoint: { x: 700, y: 500 } })(g, "B")).toEqual({
      x: 700 - CARD / 2,
      y: 500 - CARD / 2,
    });
    // box.y + CARD + 34 dista 122 px dal box, meno dei 128 di overlapsAny: freeSpot
    // sposta sempre il punto di un passo a destra (NOTE_DIVERGENZE.md)
    expect(detachPositionFn()(g, "B")).toEqual({ x: 300 + CARD + 26, y: 300 + CARD + 34 });
    // punto di rilascio in fondo al mondo: limite 1600 - 88 - 26 (riga 2176)
    expect(detachPositionFn({ dropPoint: { x: 700, y: 99999 } })(g, "B")).toEqual({
      x: 656,
      y: 1486,
    });
  });

  it("detachStep di etl-core accetta detachPositionFn", () => {
    const g = graphOf([
      {
        ...card("B", "filter", 300, 300),
        components: ["filter", "sort"],
        params: [defaultParams("filter"), defaultParams("sort")],
      },
    ]);
    const r = detachStep(g, "B", 1, () => "S", detachPositionFn());
    expect(r?.graph.cards["S"]).toMatchObject({ x: 300 + CARD + 26, y: 300 + CARD + 34 });
  });

  it("nodo inserito su un cavo: a metà strada, poi etl-core crea il suo output a destra", () => {
    let g = graphOf(
      [card("A", "dataset", 100, 300), card("B", "filter", 700, 300), card("X", "sort", 50, 800)],
      [{ from: "A", to: "B" }],
    );
    const link = g.links[0] as { from: string; to: string };
    const mid = insertPosition(g, link);
    expect(mid).toEqual({ x: 400, y: 300 });
    g = moveNode(g, "X", mid as { x: number; y: number });
    let n = 0;
    const ids = (): string => "O" + ++n;
    const next = insertOnLink(g, link, "X", ids, outputPositionFn());
    // l'output di X (O1): a destra di 200 px, 598 sulla griglia, cadrebbe su B: primo punto libero
    expect(next?.cards["O1"]).toMatchObject(freeSpot(g, Math.round(600 / GRID) * GRID, 300));
    expect(next?.links).toContainEqual({ from: "X", to: "O1" });
    expect(next?.links).toContainEqual({ from: "O1", to: "B" });
    const settled = settleNewNode(next as Graph, "X");
    expect(anyOverlap(settled)).toBe(false);
  });

  it("dopo una fusione il box va a valle del suo primo ingresso", () => {
    const g = graphOf(
      [card("A", "dataset", 500, 300), card("B", "filter", 100, 300)],
      [{ from: "A", to: "B" }],
    );
    const out = relocateAfterMerge(g, "B");
    expect(out.cards["B"]).toMatchObject({ x: 500 + CARD + 78, y: 300 });
  });
});

describe("purezza", () => {
  it("nessuna funzione modifica il grafo che riceve", () => {
    const g = graphOf(
      [
        card("A", "dataset", 300, 300),
        card("B", "filter", 320, 310),
        card("C", "sort", 700, 300, { slot: 3 }),
      ],
      [{ from: "A", to: "B" }],
    );
    const snapshot = JSON.stringify(g);
    settleLinks(g);
    resolveOverlaps(g, { snap: true });
    separateWhileDragging(g, "A");
    displace(g, "A", "B");
    assignSlots(g);
    dropInSlot(g, "A", 0, 0);
    settleNewNode(g, "B", { mode: "grid" });
    relocateAfterMerge(g, "B");
    outputPositionFn()(g, "B");
    expect(JSON.stringify(g)).toBe(snapshot);
  });
});
```

### `src/etl-layout/autoLayout.ts`

171 righe

```ts
/**
 * Riordino automatico. Prototipo, righe 4220-4327 (`autoLayout`):
 * colonne per profondità del flusso, ordine per baricentro dei vicini,
 * simmetria verticale, colonna di parcheggio per i nodi isolati.
 */
import type { Card, Graph } from "../etl-core";
import {
  AUTO_AVAIL_MARGIN,
  AUTO_BARY_PASSES,
  AUTO_COL,
  AUTO_ROW_MAX,
  AUTO_ROW_MIN,
  AUTO_SOURCE_PASSES,
  AUTO_SYMMETRY_PASSES,
  AUTO_X0_MIN,
  AUTO_Y0_MIN,
  CARD,
  LABEL_H,
} from "./constants";
import { DEFAULT_WORLD, anyOverlap, clampPoint, resolveOverlaps, withPositions } from "./free";
import { assignSlots } from "./slots";
import type { LayoutMode, Size } from "./types";

export interface AutoLayoutOptions {
  /**
   * Area visibile del canvas (`stage.clientWidth/clientHeight` nel
   * prototipo): colonne e righe vengono centrate lì.
   */
  readonly viewport: Size;
  readonly world?: Size;
  readonly mode?: LayoutMode;
}

export function autoLayout(graph: Graph, opts: AutoLayoutOptions): Graph {
  const world = opts.world ?? DEFAULT_WORLD;
  const stageW = opts.viewport.w;
  const stageH = opts.viewport.h;
  const links = graph.links;
  const all = Object.keys(graph.cards);
  if (!all.length) return graph;
  const card = (id: string): Card => graph.cards[id] as Card;

  // un nodo senza collegamenti non appartiene al flusso
  const isolated = all.filter((id) => !links.some((l) => l.from === id || l.to === id));
  const ids = all.filter((id) => !isolated.includes(id));

  // 1. profondità = percorso più lungo dalle sorgenti (righe 4228-4239)
  const depth: Record<string, number> = {};
  for (const id of ids) depth[id] = 0;
  let changed = true;
  let guard = 0;
  while (changed && guard++ < ids.length + 6) {
    changed = false;
    for (const l of links) {
      if (
        graph.cards[l.from] &&
        graph.cards[l.to] &&
        (depth[l.to] as number) < (depth[l.from] as number) + 1
      ) {
        depth[l.to] = (depth[l.from] as number) + 1;
        changed = true;
      }
    }
  }
  // una sorgente si avvicina al suo consumatore più vicino (righe 4240-4250)
  for (let pass = 0; pass < AUTO_SOURCE_PASSES; pass++) {
    for (const id of ids) {
      const hasInput = links.some((l) => l.to === id);
      if (hasInput) continue;
      const succ = links.filter((l) => l.from === id).map((l) => depth[l.to] as number);
      if (!succ.length) continue;
      depth[id] = Math.max(0, Math.min(...succ) - 1);
    }
  }
  const minD = Math.min(...ids.map((id) => depth[id] as number));
  for (const id of ids) depth[id] = (depth[id] as number) - minD;

  const layers: (string[] | undefined)[] = [];
  for (const id of ids) {
    const d = depth[id] as number;
    (layers[d] = layers[d] ?? []).push(id);
  }
  const cols: string[][] = layers.filter((l): l is string[] => !!l);
  if (isolated.length) cols.push(isolated);
  if (!cols.length) return graph;

  // posizioni di lavoro, modificate sul posto come nel prototipo
  const pos = new Map<string, { x: number; y: number }>();
  for (const id of all) pos.set(id, { x: card(id).x, y: card(id).y });
  const P = (id: string): { x: number; y: number } => pos.get(id) as { x: number; y: number };

  // 2. ordine dentro la colonna: baricentro dei vicini (righe 4256-4278)
  const posIn: Record<string, number> = {};
  const reindex = (): void => {
    for (const layer of cols) layer.forEach((id, i) => (posIn[id] = i));
  };
  for (const layer of cols) layer.sort((a, b) => P(a).y - P(b).y);
  reindex();
  for (let it = 0; it < AUTO_BARY_PASSES; it++) {
    for (const layer of cols) {
      const bary: Record<string, number> = {};
      for (const id of layer) {
        const nb: number[] = [];
        for (const l of links) {
          if (l.to === id && posIn[l.from] !== undefined) nb.push(posIn[l.from] as number);
          if (l.from === id && posIn[l.to] !== undefined) nb.push(posIn[l.to] as number);
        }
        bary[id] = nb.length ? nb.reduce((a, b) => a + b, 0) / nb.length : (posIn[id] as number);
      }
      layer.sort((a, b) => (bary[a] as number) - (bary[b] as number));
      reindex();
    }
  }

  // 3. griglia di partenza: colonne equidistanti, ogni colonna centrata (righe 4280-4295)
  const maxCount = Math.max(...cols.map((c) => c.length));
  const avail = stageH - AUTO_AVAIL_MARGIN - (CARD + LABEL_H);
  const ROW =
    maxCount > 1
      ? Math.max(AUTO_ROW_MIN, Math.min(AUTO_ROW_MAX, avail / (maxCount - 1)))
      : AUTO_ROW_MAX;
  const totalW = (cols.length - 1) * AUTO_COL + CARD;
  const x0 = Math.max(AUTO_X0_MIN, (stageW - totalW) / 2);
  cols.forEach((layer, li) => {
    const h = (layer.length - 1) * ROW + CARD + LABEL_H;
    const y0 = Math.max(AUTO_Y0_MIN, (stageH - h) / 2);
    layer.forEach((id, i) => {
      P(id).x = x0 + li * AUTO_COL;
      P(id).y = y0 + i * ROW;
    });
  });

  // 4. simmetria: ogni nodo tende al baricentro dei vicini (righe 4297-4316)
  for (let pass = 0; pass < AUTO_SYMMETRY_PASSES; pass++) {
    for (const layer of cols) {
      for (const id of layer) {
        const nb: number[] = [];
        for (const l of links) {
          if (l.to === id && graph.cards[l.from]) nb.push(P(l.from).y);
          if (l.from === id && graph.cards[l.to]) nb.push(P(l.to).y);
        }
        if (nb.length) P(id).y = (P(id).y + nb.reduce((a, b) => a + b, 0) / nb.length) / 2;
      }
      layer.sort((a, b) => P(a).y - P(b).y);
      for (let i = 1; i < layer.length; i++) {
        const prev = P(layer[i - 1] as string);
        const cur = P(layer[i] as string);
        if (cur.y - prev.y < ROW) cur.y = prev.y + ROW;
      }
      const first = P(layer[0] as string).y;
      const last = P(layer[layer.length - 1] as string).y;
      const off = (stageH - (last - first + CARD + LABEL_H)) / 2 - first;
      for (const id of layer) P(id).y += off;
    }
  }

  // arrotondamento e limiti del mondo (riga 4319)
  for (const id of all) {
    const p = P(id);
    const q = clampPoint({ x: Math.round(p.x), y: Math.round(p.y) }, world);
    p.x = q.x;
    p.y = q.y;
  }
  const placed = withPositions(graph, pos);

  // righe 4321-4323
  if (opts.mode === "grid") return assignSlots(placed, world);
  if (anyOverlap(placed)) return resolveOverlaps(placed, { snap: false, world });
  return placed;
}
```

### `src/etl-layout/constants.ts`

142 righe

```ts
/**
 * Costanti numeriche della geometria, IDENTICHE al prototipo
 * (docs/prototype/isa-fusion-prototype.html). Ogni costante riporta la
 * riga da cui proviene.
 */

// --- Nodi --------------------------------------------------------------------

/** Lato del quadrato di un nodo (riga 909; CSS `.icon-wrap`, riga 630). */
export const CARD = 88;
/** Spazio occupato dall'etichetta sotto il nodo (riga 1524). */
export const LABEL_H = 22;
/** Distanza tra il quadrato e l'etichetta (CSS `.card { gap:8px }`, riga 626). */
export const LABEL_GAP = 8;
/** Larghezza massima dell'etichetta (CSS `.label { max-width:96px }`, riga 682). */
export const LABEL_MAX_W = 96;

// --- Mondo -------------------------------------------------------------------

/** Dimensioni del mondo con pan/zoom attivo (riga 936; `FEATURES.panZoom` è `true`, riga 919). */
export const WORLD_W = 2600;
export const WORLD_H = 1600;
/** Margine dal bordo del mondo per `clampCard` (righe 1538-1539). */
export const WORLD_MARGIN = 6;

// --- Porte e cavi ------------------------------------------------------------

/** Angoli delle quattro porte: destra, sotto, sinistra, sopra (riga 1018). */
export const PORTS: readonly number[] = [0, Math.PI / 2, Math.PI, -Math.PI / 2];
/** Tratto rettilineo in uscita da una porta prima dello snodo (riga 1020). */
export const STUB = 32;
/** Raggio dei raccordi arrotondati (riga 1020). */
export const ELBOW_R = 11;
/** Snodi ammessi per collegamento (riga 1021). */
export const MAX_BENDS = 1;
/** Margine attorno a ogni nodo trattato come ostacolo (riga 1055). */
export const OBST_PAD = 9;
/** Rientro del rettangolo dei due nodi collegati usati come ostacolo (riga 1245). */
export const SELF_OBSTACLE_INSET = 2;
/** Scorrimento massimo del punto di aggancio: `half - 14` (riga 1257). */
export const PORT_SLACK_INSET = 14;
/** Sotto questa differenza i due agganci sono già allineati (righe 1218, 1223). */
export const STRAIGHT_EPS = 1.5;
/** Distanza degli snodi candidati dai bordi di un ostacolo (righe 1234-1235). */
export const KNOB_OBSTACLE_MARGIN = 14;
/** Passo e numero degli snodi candidati attorno a quello centrale (riga 1237). */
export const KNOB_STEP = 24;
export const KNOB_STEPS = 8;
/** Tolleranza sui confronti tra coordinate (righe 1057, 1090, 1102, 1180). */
export const EPS = 0.5;
/** `segCross`: margine sulle estremità (riga 1101). */
export const CROSS_MARGIN = 3;
/** `segCross`: due tratti paralleli sono sullo stesso binario sotto questa distanza (righe 1109, 1114). */
export const CROSS_SAME_TRACK = 4;
/** `segCross`: sovrapposizione minima per contare due tratti paralleli (righe 1112, 1117). */
export const CROSS_MIN_OVERLAP = 8;

/** Pesi della funzione di costo di `chooseRoute` (righe 1267-1268). */
export const SCORE = {
  /** Ogni attraversamento di un nodo. */
  obstacle: 1e6,
  /** Ogni snodo oltre il limite. */
  overBends: 6000,
  /** Ogni incrocio con un altro cavo. */
  crossing: 3500,
  /** Ogni ripiegamento rispetto alla porta. */
  backtrack: 3000,
  /** Uscita e rientro dallo stesso lato (forma a U). */
  sameSide: 1800,
  /** Ogni pixel fuori dal rettangolo che unisce gli agganci. */
  overshoot: 14,
  /** Ogni pixel di lunghezza. */
  length: 1,
  /** Ogni porta diversa da quella precedente. */
  portChange: 0.8,
} as const;

/** Distanza tra cavi che condividono la stessa porta (riga 1355). */
export const PORT_SPREAD = 15;
/** Distanza tra corsie parallele (riga 1380). */
export const LANE_GAP = 14;
/** Due snodi più vicini di così condividono il corridoio (riga 1378). */
export const LANE_NEAR = 12;
/** Spessore dell'area sensibile di un cavo: `stroke-width="16"` (riga 1402). Tolleranza = metà. */
export const LINK_HIT_WIDTH = 16;

// --- Modalità Libero ----------------------------------------------------------

/** Passo della griglia di aggancio (riga 1541). */
export const GRID = 26;
/** Distanza minima tra due nodi in `resolveOverlaps` (riga 1553). */
export const OVERLAP_PAD = 28;
/** Iterazioni massime di `resolveOverlaps` (riga 1556). */
export const OVERLAP_ITERATIONS = 80;
/** Margine di `overlapsAny` e `anyOverlap` (righe 1593, 2635). */
export const OVERLAP_MARGIN = 18;
/** Passi di `freeSpot` (righe 1600-1603). */
export const FREE_SPOT_RADIUS = 8;
export const FREE_SPOT_STEP_X = CARD + 26;
export const FREE_SPOT_STEP_Y = CARD + LABEL_H + 22;
/** Spinta di `displace` e margine inferiore del suo clamp (righe 1946, 1950). */
export const DISPLACE_PUSH = CARD + 36;
export const DISPLACE_BOTTOM = CARD + 26;

// --- Modalità Organizzato -----------------------------------------------------

/** Postazioni: larghezza, altezza, margine (riga 3908). */
export const SLOT_W = 128;
export const SLOT_H = 142;
export const SLOT_M = 14;

// --- Riordino automatico ------------------------------------------------------

/** Distanza tra le colonne (riga 4282). */
export const AUTO_COL = CARD + 78;
/** Margine verticale sottratto all'altezza disponibile (riga 4284). */
export const AUTO_AVAIL_MARGIN = 16;
/** Distanza tra le righe: minima e massima (righe 4285-4287). */
export const AUTO_ROW_MIN = CARD + LABEL_H + 18;
export const AUTO_ROW_MAX = CARD + LABEL_H + 28;
/** Margini minimi a sinistra e in alto (righe 4289, 4292). */
export const AUTO_X0_MIN = 12;
export const AUTO_Y0_MIN = 8;
/** Passate: sorgenti avvicinate, baricentro, simmetria (righe 4242, 4265, 4297). */
export const AUTO_SOURCE_PASSES = 3;
export const AUTO_BARY_PASSES = 6;
export const AUTO_SYMMETRY_PASSES = 5;

// --- Nodi generati ------------------------------------------------------------

/** Output: a destra del box di 200, entro il mondo meno 12 (righe 1743, 1745). */
export const OUTPUT_OFFSET_X = 200;
export const OUTPUT_EDGE_MARGIN = 12;
/** Passaggio sganciato senza punto di rilascio: sotto il box di `CARD + 34` (riga 2178). */
export const DETACH_OFFSET_Y = CARD + 34;
/** `relocateAfter`: passi orizzontali/verticali e tentativi (righe 1678-1681). */
export const RELOCATE_STEP_X = CARD + 78;
export const RELOCATE_STEP_Y = CARD + LABEL_H + 30;
export const RELOCATE_TRIES = 5;
/** `outputSlotFor`: peso dello scarto verticale (riga 1715). */
export const OUTPUT_SLOT_DY_WEIGHT = 3;
```

