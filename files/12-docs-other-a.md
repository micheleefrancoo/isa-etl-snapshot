# 12-docs-other-a.md

File in questo blocco:

- `docs/theme-debt.md`
- `docs/visual/fase4/console.txt`
- `docs/visual/fase4/misure.json`
- `docs/visual/fase4b/console.txt`
- `docs/visual/fase4b/misure.json`
- `docs/visual/fase6a/posizioni-nodi.json`
- `docs/visual/fase6b11/REPORT.md`

---

### `docs/theme-debt.md`

57 righe

```md
# Debito dei token: [REDATTO] scritti a mano

Generato da `node scripts/check-tokens.mjs --write-debt`. Elenca i colori, i raggi e le ombre letterali dei file di `src/` preesistenti alla Fase T (`scripts/token-legacy-files.txt`), da portare sui token semantici nel restyling. Il controllo non li fa fallire.

**Totale: 26** (6 ombra, 7 raggio, 13 colore) in 8 file.

## `src/components/isa/sidebar.tsx` (1)

- riga 102 — ombra: `shadow-[inset_0_1px_0_var(--glass-border)]`

## `src/components/isa/ui/isa-menu.tsx` (1)

- riga 409 — raggio: `rounded-[5px]`

## `src/components/ui/chart.tsx` (7)

- riga 51 — colore: `#ccc`
- riga 51 — colore: `#fff`
- riga 51 — colore: `#ccc`
- riga 51 — colore: `#ccc`
- riga 51 — colore: `#fff`
- riga 193 — raggio: `rounded-[2px]`
- riga 283 — raggio: `rounded-[2px]`

## `src/components/ui/drawer.tsx` (1)

- riga 41 — raggio: `rounded-t-[10px]`

## `src/components/ui/sidebar.tsx` (2)

- riga 510 — ombra: `shadow-[0_0_0_1px_var(--sidebar-border)]`
- riga 510 — ombra: `shadow-[0_0_0_1px_var(--sidebar-accent)]`

## `src/lib/error-page.ts` (8)

- riga 9 — colore: `#fafafa`
- riga 9 — colore: `#111`
- riga 12 — colore: `#4b5563`
- riga 15 — colore: `#111`
- riga 15 — colore: `#fff`
- riga 16 — colore: `#fff`
- riga 16 — colore: `#111`
- riga 16 — colore: `#d1d5db`

## `src/routes/solutions.$solutionId.dashboard.tsx` (1)

- riga 62 — raggio: `borderRadius: 16`

## `src/styles.css` (5)

- riga 149 — raggio: `border-radius: 9999px`
- riga 165 — raggio: `border-radius: 9999px`
- riga 217 — ombra: `filter: drop-shadow(0 0 3px var(--brand-glow))`
- riga 224 — ombra: `box-shadow: 0 0 0 1px color-mix(in oklab, var(--brand) 45%, transparent), 0 0 24px color-mix(in oklab, var(--brand) 28%, transparent)`
- riga 233 — ombra: `box-shadow: 0 0 0 2px var(--brand), 0 0 26px color-mix(in oklab, var(--brand) 35%, transparent)`

```

### `docs/visual/fase4/console.txt`

2 righe

```
nessun errore né avviso in console
```

### `docs/visual/fase4/misure.json`

1048 righe

```json
{
  "prototipo": {
    "stage": {
      "rect": {
        "w": 712,
        "h": 520
      },
      "style": {
        "backgroundColor": "rgba(255, 255, 255, 0.32)",
        "borderRadius": "20px"
      }
    },
    "nodes": {
      "ds1": {
        "rect": {
          "x": 26,
          "y": 182,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 26,
            "y": 182,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(108, 99, 255)",
            "borderRadius": "26px",
            "color": "rgb(255, 255, 255)",
            "opacity": "1",
            "boxShadow": "none"
          }
        },
        "label": {
          "rect": {
            "x": 34.5,
            "y": 278,
            "w": 71,
            "h": 13.13
          },
          "style": {
            "fontFamily": "Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(108, 99, 255)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": null
      },
      "op-filter": {
        "rect": {
          "x": 260,
          "y": 52,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 52,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "none"
          }
        },
        "label": {
          "rect": {
            "x": 273,
            "y": 148,
            "w": 62,
            "h": 13.13
          },
          "style": {
            "fontFamily": "Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 131,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      },
      "op-join": {
        "rect": {
          "x": 260,
          "y": 182,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 182,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "none"
          }
        },
        "label": {
          "rect": {
            "x": 269,
            "y": 278,
            "w": 70,
            "h": 13.13
          },
          "style": {
            "fontFamily": "Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 261,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      },
      "op-sort": {
        "rect": {
          "x": 260,
          "y": 338,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 338,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "none"
          }
        },
        "label": {
          "rect": {
            "x": 285,
            "y": 434,
            "w": 38,
            "h": 13.13
          },
          "style": {
            "fontFamily": "Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 417,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      },
      "op-export": {
        "rect": {
          "x": 442,
          "y": 338,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 442,
            "y": 338,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "none"
          }
        },
        "label": {
          "rect": {
            "x": 464.5,
            "y": 434,
            "w": 43,
            "h": 13.13
          },
          "style": {
            "fontFamily": "Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 521,
            "y": 417,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      }
    },
    "zoom": {
      "rect": {
        "x": 524,
        "y": 470,
        "w": 176,
        "h": 38
      },
      "fromRight": 12,
      "fromBottom": 12,
      "style": {
        "backgroundColor": "rgba(255, 255, 255, 0.92)",
        "borderRadius": "999px",
        "borderTopWidth": "1px",
        "borderTopColor": "rgba(38, 36, 32, 0.06)",
        "boxShadow": "rgba(38, 36, 32, 0.4) 0px 10px 24px -14px",
        "backdropFilter": "blur(16px)"
      },
      "fit": {
        "color": "rgb(108, 99, 255)",
        "fontSize": "12.5px",
        "fontWeight": "700",
        "height": "28px",
        "minWidth": "28px"
      },
      "text": "100%"
    },
    "minimap": {
      "rect": {
        "x": 12,
        "y": 404,
        "w": 168,
        "h": 104
      },
      "fromLeft": 12,
      "fromBottom": 12,
      "style": {
        "backgroundColor": "rgba(255, 255, 255, 0.92)",
        "borderRadius": "14px",
        "borderTopWidth": "1px",
        "borderTopColor": "rgba(38, 36, 32, 0.06)",
        "boxShadow": "rgba(38, 36, 32, 0.4) 0px 10px 24px -14px",
        "backdropFilter": "blur(16px)"
      }
    },
    "body": {
      "fontFamily": "Manrope, system-ui, sans-serif"
    },
    "cavi": {
      "count": 4,
      "path": {
        "stroke": "rgba(108, 99, 255, 0.34)",
        "strokeWidth": "2.1px",
        "strokeLinecap": "round",
        "strokeLinejoin": "round",
        "fill": "none"
      },
      "dots": 8,
      "dot": {
        "fill": "rgb(108, 99, 255)",
        "r": "2.6"
      },
      "d": [
        "M 114.00 226.00 L 260.00 226.00",
        "M 70.00 182.00 L 70.00 107.00 Q 70.00 96.00 81.00 96.00 L 260.00 96.00",
        "M 348.00 226.00 L 468.00 226.00",
        "M 348.00 96.00 L 468.00 96.00"
      ]
    }
  },
  "v2-chiaro": {
    "stage": {
      "rect": {
        "w": 1408,
        "h": 738
      },
      "style": {
        "backgroundColor": "rgba(255, 255, 255, 0.32)",
        "borderRadius": "20px"
      }
    },
    "nodes": {
      "ds1": {
        "rect": {
          "x": 26,
          "y": 182,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 26,
            "y": 182,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(108, 99, 255)",
            "borderRadius": "26px",
            "color": "rgb(255, 255, 255)",
            "opacity": "1",
            "boxShadow": "none"
          }
        },
        "label": {
          "rect": {
            "x": 34.5,
            "y": 278,
            "w": 71,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(108, 99, 255)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": null
      },
      "op-filter": {
        "rect": {
          "x": 260,
          "y": 52,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 52,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "rgba(0, 0, 0, 0) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 273,
            "y": 148,
            "w": 62,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 131,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      },
      "op-join": {
        "rect": {
          "x": 260,
          "y": 182,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 182,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "rgba(0, 0, 0, 0) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 269,
            "y": 278,
            "w": 70,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 261,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      },
      "op-sort": {
        "rect": {
          "x": 260,
          "y": 338,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 338,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "rgba(0, 0, 0, 0) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 285,
            "y": 434,
            "w": 38,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 417,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      },
      "op-export": {
        "rect": {
          "x": 442,
          "y": 338,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 442,
            "y": 338,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(225, 220, 240)",
            "borderRadius": "22px",
            "color": "rgb(108, 99, 255)",
            "opacity": "1",
            "boxShadow": "rgba(0, 0, 0, 0) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 464.5,
            "y": 434,
            "w": 43,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(38, 36, 32)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 521,
            "y": 417,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(224, 162, 59)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(247, 245, 241)"
          }
        }
      }
    },
    "zoom": {
      "rect": {
        "x": 1220,
        "y": 688,
        "w": 176,
        "h": 38
      },
      "fromRight": 12,
      "fromBottom": 12,
      "style": {
        "backgroundColor": "rgba(255, 255, 255, 0.92)",
        "borderRadius": "999px",
        "borderTopWidth": "1px",
        "borderTopColor": "rgba(38, 36, 32, 0.06)",
        "boxShadow": "rgba(38, 36, 32, 0.4) 0px 10px 24px -14px",
        "backdropFilter": "blur(16px)"
      },
      "fit": {
        "color": "rgb(108, 99, 255)",
        "fontSize": "12.5px",
        "fontWeight": "700",
        "height": "28px",
        "minWidth": "28px"
      },
      "text": "100%"
    },
    "minimap": {
      "rect": {
        "x": 12,
        "y": 622,
        "w": 168,
        "h": 104
      },
      "fromLeft": 12,
      "fromBottom": 12,
      "style": {
        "backgroundColor": "rgba(255, 255, 255, 0.92)",
        "borderRadius": "14px",
        "borderTopWidth": "1px",
        "borderTopColor": "rgba(38, 36, 32, 0.06)",
        "boxShadow": "rgba(38, 36, 32, 0.4) 0px 10px 24px -14px",
        "backdropFilter": "blur(16px)"
      }
    },
    "body": {
      "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif"
    },
    "font": {
      "loaded": [
        "Manrope Variable"
      ]
    },
    "cavi": {
      "count": 4,
      "path": {
        "stroke": "rgba(108, 99, 255, 0.34)",
        "strokeWidth": "2.1px",
        "strokeLinecap": "round",
        "strokeLinejoin": "round",
        "fill": "none"
      },
      "dots": 8,
      "dot": {
        "fill": "rgb(108, 99, 255)",
        "r": "2.6"
      },
      "d": [
        "M 114.00 226.00 L 260.00 226.00",
        "M 348.00 226.00 L 468.00 226.00",
        "M 70.00 182.00 L 70.00 107.00 Q 70.00 96.00 81.00 96.00 L 260.00 96.00",
        "M 348.00 96.00 L 468.00 96.00"
      ]
    }
  },
  "v2-scuro": {
    "stage": {
      "rect": {
        "w": 1408,
        "h": 738
      },
      "style": {
        "backgroundColor": "rgba(255, 255, 255, 0.04)",
        "borderRadius": "20px"
      }
    },
    "nodes": {
      "ds1": {
        "rect": {
          "x": 26,
          "y": 182,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 26,
            "y": 182,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(108, 99, 255)",
            "borderRadius": "26px",
            "color": "rgb(255, 255, 255)",
            "opacity": "1",
            "boxShadow": "none"
          }
        },
        "label": {
          "rect": {
            "x": 34.5,
            "y": 278,
            "w": 71,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(168, 163, 255)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": null
      },
      "op-filter": {
        "rect": {
          "x": 260,
          "y": 52,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 52,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(58, 54, 112)",
            "borderRadius": "22px",
            "color": "rgb(208, 204, 255)",
            "opacity": "1",
            "boxShadow": "rgb(127, 120, 230) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 273,
            "y": 148,
            "w": 62,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(241, 242, 245)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 131,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(232, 179, 79)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(23, 24, 29)"
          }
        }
      },
      "op-join": {
        "rect": {
          "x": 260,
          "y": 182,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 182,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(58, 54, 112)",
            "borderRadius": "22px",
            "color": "rgb(208, 204, 255)",
            "opacity": "1",
            "boxShadow": "rgb(127, 120, 230) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 269,
            "y": 278,
            "w": 70,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(241, 242, 245)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 261,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(232, 179, 79)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(23, 24, 29)"
          }
        }
      },
      "op-sort": {
        "rect": {
          "x": 260,
          "y": 338,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 260,
            "y": 338,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(58, 54, 112)",
            "borderRadius": "22px",
            "color": "rgb(208, 204, 255)",
            "opacity": "1",
            "boxShadow": "rgb(127, 120, 230) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 285,
            "y": 434,
            "w": 38,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(241, 242, 245)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 339,
            "y": 417,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(232, 179, 79)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(23, 24, 29)"
          }
        }
      },
      "op-export": {
        "rect": {
          "x": 442,
          "y": 338,
          "w": 88,
          "h": 109.13
        },
        "wrap": {
          "rect": {
            "x": 442,
            "y": 338,
            "w": 88,
            "h": 88
          },
          "style": {
            "backgroundColor": "rgb(58, 54, 112)",
            "borderRadius": "22px",
            "color": "rgb(208, 204, 255)",
            "opacity": "1",
            "boxShadow": "rgb(127, 120, 230) 0px 0px 0px 1.5px inset"
          }
        },
        "label": {
          "rect": {
            "x": 464.5,
            "y": 434,
            "w": 43,
            "h": 13.13
          },
          "style": {
            "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif",
            "fontSize": "10.5px",
            "fontWeight": "700",
            "color": "rgb(241, 242, 245)",
            "lineHeight": "13.125px"
          }
        },
        "icon": {
          "width": "26px",
          "height": "26px"
        },
        "dot": {
          "rect": {
            "x": 521,
            "y": 417,
            "w": 13,
            "h": 13
          },
          "style": {
            "backgroundColor": "rgb(232, 179, 79)",
            "borderTopWidth": "2px",
            "borderTopColor": "rgb(23, 24, 29)"
          }
        }
      }
    },
    "zoom": {
      "rect": {
        "x": 1220,
        "y": 688,
        "w": 176,
        "h": 38
      },
      "fromRight": 12,
      "fromBottom": 12,
      "style": {
        "backgroundColor": "rgba(36, 37, 45, 0.92)",
        "borderRadius": "999px",
        "borderTopWidth": "1px",
        "borderTopColor": "rgba(255, 255, 255, 0.11)",
        "boxShadow": "rgba(0, 0, 0, 0.6) 0px 10px 24px -14px",
        "backdropFilter": "blur(16px)"
      },
      "fit": {
        "color": "rgb(168, 163, 255)",
        "fontSize": "12.5px",
        "fontWeight": "700",
        "height": "28px",
        "minWidth": "28px"
      },
      "text": "100%"
    },
    "minimap": {
      "rect": {
        "x": 12,
        "y": 622,
        "w": 168,
        "h": 104
      },
      "fromLeft": 12,
      "fromBottom": 12,
      "style": {
        "backgroundColor": "rgba(36, 37, 45, 0.92)",
        "borderRadius": "14px",
        "borderTopWidth": "1px",
        "borderTopColor": "rgba(255, 255, 255, 0.11)",
        "boxShadow": "rgba(0, 0, 0, 0.6) 0px 10px 24px -14px",
        "backdropFilter": "blur(16px)"
      }
    },
    "body": {
      "fontFamily": "\"Manrope Variable\", Manrope, system-ui, sans-serif"
    },
    "font": {
      "loaded": [
        "Manrope Variable"
      ]
    },
    "cavi": {
      "count": 4,
      "path": {
        "stroke": "rgb(127, 120, 230)",
        "strokeWidth": "2.1px",
        "strokeLinecap": "round",
        "strokeLinejoin": "round",
        "fill": "none"
      },
      "dots": 8,
      "dot": {
        "fill": "rgb(108, 99, 255)",
        "r": "2.6"
      },
      "d": [
        "M 114.00 226.00 L 260.00 226.00",
        "M 348.00 226.00 L 468.00 226.00",
        "M 70.00 182.00 L 70.00 107.00 Q 70.00 96.00 81.00 96.00 L 260.00 96.00",
        "M 348.00 96.00 L 468.00 96.00"
      ]
    }
  }
}
```

### `docs/visual/fase4b/console.txt`

2 righe

```
nessun errore né avviso in console
```

### `docs/visual/fase4b/misure.json`

353 righe

```json
{
  "instants": [
    0,
    250,
    500
  ],
  "freezeMs": 100000,
  "prototipo": {
    "t0": {
      "flusso": [
        {
          "x": 348,
          "y": 95,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 69,
          "y": 178.9,
          "w": 2.1,
          "h": 3.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 114,
          "y": 225,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 348,
          "y": 225,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        }
      ],
      "attesa": [
        0.45
      ]
    },
    "t250": {
      "flusso": [
        {
          "x": 348,
          "y": 90.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 64.8,
          "y": 138.9,
          "w": 10.5,
          "h": 43.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 114,
          "y": 220.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 348,
          "y": 220.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(108, 99, 255, 0.6)"
        }
      ],
      "attesa": [
        0.522
      ]
    },
    "t500": {
      "flusso": [
        {
          "x": 350.1,
          "y": 90.5,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 64.5,
          "y": 98.8,
          "w": 10.9,
          "h": 81.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 116.1,
          "y": 220.5,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 350.1,
          "y": 220.5,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(108, 99, 255, 0.6)"
        }
      ],
      "attesa": [
        0.723
      ]
    }
  },
  "v2-chiaro": {
    "t0": {
      "flusso": [
        {
          "x": 348,
          "y": 94.9,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 68.9,
          "y": 178.9,
          "w": 2.1,
          "h": 3.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 114,
          "y": 225,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 348,
          "y": 225,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        }
      ],
      "attesa": [
        0.45
      ]
    },
    "t250": {
      "flusso": [
        {
          "x": 348,
          "y": 90.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 64.8,
          "y": 138.9,
          "w": 10.5,
          "h": 43.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 114,
          "y": 220.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 348,
          "y": 220.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(108, 99, 255, 0.6)"
        }
      ],
      "attesa": [
        0.522
      ]
    },
    "t500": {
      "flusso": [
        {
          "x": 350.1,
          "y": 90.6,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 64.6,
          "y": 98.8,
          "w": 10.9,
          "h": 81.1,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 116.1,
          "y": 220.5,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(108, 99, 255, 0.6)"
        },
        {
          "x": 350.1,
          "y": 220.5,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(108, 99, 255, 0.6)"
        }
      ],
      "attesa": [
        0.723
      ]
    }
  },
  "v2-scuro": {
    "t0": {
      "flusso": [
        {
          "x": 348,
          "y": 94.9,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 68.9,
          "y": 178.9,
          "w": 2.1,
          "h": 3.1,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 114,
          "y": 225,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 348,
          "y": 225,
          "w": 3.1,
          "h": 2.1,
          "fill": "rgba(168, 163, 255, 0.9)"
        }
      ],
      "attesa": [
        0.45
      ]
    },
    "t250": {
      "flusso": [
        {
          "x": 348,
          "y": 90.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 64.8,
          "y": 138.9,
          "w": 10.5,
          "h": 43.1,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 114,
          "y": 220.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 348,
          "y": 220.8,
          "w": 43.1,
          "h": 10.5,
          "fill": "rgba(168, 163, 255, 0.9)"
        }
      ],
      "attesa": [
        0.522
      ]
    },
    "t500": {
      "flusso": [
        {
          "x": 350.1,
          "y": 90.6,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 64.6,
          "y": 98.8,
          "w": 10.9,
          "h": 81.1,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 116.1,
          "y": 220.5,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(168, 163, 255, 0.9)"
        },
        {
          "x": 350.1,
          "y": 220.5,
          "w": 81,
          "h": 10.9,
          "fill": "rgba(168, 163, 255, 0.9)"
        }
      ],
      "attesa": [
        0.723
      ]
    }
  },
  "risparmioEnergetico": {
    "movimentoNormale": {
      "rafInUnSecondo": 60
    },
    "schedaNascosta": {
      "rafInUnSecondo": 0
    },
    "schedaTornataVisibile": {
      "rafInUnSecondo": 61
    },
    "nullaDaAnimare": {
      "rafInUnSecondo": 0,
      "cavi": 0
    },
    "movimentoRidotto": {
      "rafInUnSecondo": 0,
      "flussoStatico": true,
      "tubi": 4,
      "attesa": [
        0.85
      ]
    }
  }
}
```

### `docs/visual/fase6a/posizioni-nodi.json`

107 righe

```json
[
  {
    "bordo": "right",
    "zoom": 1,
    "nodi": {
      "ds1": {
        "prima": "42,378",
        "dopo": "42,378"
      },
      "op-filter": {
        "prima": "276,248",
        "dopo": "276,248"
      },
      "op-join": {
        "prima": "276,378",
        "dopo": "276,378"
      },
      "op-sort": {
        "prima": "276,534",
        "dopo": "276,534"
      },
      "op-export": {
        "prima": "458,534",
        "dopo": "458,534"
      }
    }
  },
  {
    "bordo": "top",
    "zoom": 1,
    "nodi": {
      "ds1": {
        "prima": "42,378",
        "dopo": "42,560"
      },
      "op-filter": {
        "prima": "276,248",
        "dopo": "276,430"
      },
      "op-join": {
        "prima": "276,378",
        "dopo": "276,560"
      },
      "op-sort": {
        "prima": "276,534",
        "dopo": "276,716"
      },
      "op-export": {
        "prima": "458,534",
        "dopo": "458,716"
      }
    }
  },
  {
    "bordo": "bottom",
    "zoom": 1,
    "nodi": {
      "ds1": {
        "prima": "42,338",
        "dopo": "42,338"
      },
      "op-filter": {
        "prima": "276,208",
        "dopo": "276,208"
      },
      "op-join": {
        "prima": "276,338",
        "dopo": "276,338"
      },
      "op-sort": {
        "prima": "276,494",
        "dopo": "276,494"
      },
      "op-export": {
        "prima": "458,494",
        "dopo": "458,494"
      }
    }
  },
  {
    "bordo": "left",
    "zoom": 1,
    "nodi": {
      "ds1": {
        "prima": "42,338",
        "dopo": "322,338"
      },
      "op-filter": {
        "prima": "276,208",
        "dopo": "556,208"
      },
      "op-join": {
        "prima": "276,338",
        "dopo": "556,338"
      },
      "op-sort": {
        "prima": "276,494",
        "dopo": "556,494"
      },
      "op-export": {
        "prima": "458,494",
        "dopo": "738,494"
      }
    }
  }
]
```

### `docs/visual/fase6b11/REPORT.md`

89 righe

```md
# Fase 6b.1.1: zoom automatico, report di validazione

Verifica nel browser reale (`node scripts/e2e-fase6b11.mjs`), orologio controllato
(16 ms per passo, un campione di ogni frame), a 1440×900 e 1280×720 con una scena
densa di 32 nodi che riempie tutta l'area sicura a pannelli chiusi.
Esito: **227 prove, 0 fallite**, ripetuto 11 volte senza differenze (le misure sotto
sono identiche tra le esecuzioni, salvo il numero di frame campionati: 920–924).

Misure complete in `misure.json`.

## Numeri per requisito

44 transizioni (cassetta e Inspector sui 4 bordi, apri e chiudi; cambio scheda;
cassetta → Inspector su un altro bordo; spostamento dell'Inspector su 3 bordi), 905 frame campionati.

| Req. | Cosa | Prove | Esito | Numeri |
|------|------|-------|-------|--------|
| R1 | a riposo ogni nodo richiesto sta nell'area sicura e non tocca pannelli né widget | 46 | 46 ok | 32 nodi su 32 in ogni stato a riposo; nessun avviso attivo |
| R2 | a ogni frame i nodi richiesti stanno nel canvas di quel frame (margine 24 px) | 44 | 44 ok | scarto massimo **0,000 px** su 905 frame (tolleranza 0,5) |
| R3 | zoom monotono, nessun overshoot, almeno 10 frame | 44 | 44 ok | **19 frame attivi** per ogni animazione (≈ 300 ms); zoom minimo **0,453**, massimo **1,000**; cambio scheda: vista ferma (0 px) |
| R4 | nessun salto: passo massimo ≤ 2,5 × la media | 44 | 44 ok | rapporto massimo **1,59** (limite teorico dell'easing a seno: π/2 ≈ 1,57); spostamento massimo per frame **31,24 px** (Inspector spostato su left, 1440) |
| R5 | movimento ridotto: stato finale immediato | 8 | 8 ok | 0 frame di animazione, R1 vale subito, apri → chiudi torna alla vista iniziale |
| R6 | apri → chiudi senza altre azioni: la vista finale è quella iniziale | 16 | 16 ok | scarto ≤ 0,5 px e 1e-3 di zoom (cassetta e Inspector × 4 bordi × 2 finestre) |

Altre prove: ridimensionamento (12 passi per finestra) senza nodi fuori area e,
tornando alla dimensione di prima, vista identica al bit; rotella a metà
transizione: la vista resta dov'è (solo 40 px di scorrimento) e il pannello finisce;
nessun guscio dopo lo spostamento; nessuna transizione CSS di larghezza/altezza;
1,60–1,63 `requestAnimationFrame` per passo (un solo ciclo condiviso).

### Due cambi ravvicinati (apri e, dopo 6 frame, chiudi)

Con la nuova misura (rapporto passo/media calcolato per ciascuna delle due
animazioni): **1,72** a 1440 e a 1280×720, passo massimo 26,48 px (1440) e 28,14 px
(1280×720). La posizione è continua; al momento dell'inversione la velocità cade a
zero e la seconda animazione riparte da ferma (nessun salto di posizione, ma uno
scatto di velocità: scelta di progetto, la seconda animazione parte dalla vista
corrente con la sua curva).

La misura precedente divideva il passo massimo della prima animazione per la media
di entrambe (la seconda è molto più corta): dava 3,59–4,16 senza che ci fosse alcun
salto di posizione. Per questo è stata cambiata, non per far passare la prova.

## MIN_ZOOM

`MIN_ZOOM` = 0,35. Zoom necessario per tenere tutti i 32 nodi richiesti con
l'Inspector in basso: **0,535** a 1440×900 e **0,453** a 1280×720. Entrambi sono
sopra il minimo: nessuno dei due casi arriva al limite, l'avviso «nodi fuori
dall'area» non compare (`avviso: false`) e lo zoom finale coincide con quello
necessario. Il comportamento al limite (zoom fermo a `MIN_ZOOM`, avviso) è
coperto dai test unitari di `autoFit` (`autofit.test.ts`), non da questa scena.

## Il difetto di `clientWidth` (corretto in `Dock.tsx`)

`DockLayout` misura l'area del canvas con `el.clientWidth` / `clientHeight`, che sono
**interi arrotondati**, e li tiene in uno stato React. Durante l'animazione la
larghezza del canvas è frazionaria (per esempio 969,1 px). L'ultima misura
intermedia (969) veniva consegnata a `animator.onMeasure` quando l'animazione
era già finita (l'animatore ignora le misure durante l'animazione, ma quella
arrivava dopo): confrontata con l'area calcolata (968) differiva di 1 px, oltre la
tolleranza di 0,5 px, e la vista veniva ricalcolata sull'area sbagliata; poi
arrivava la misura corretta (968) e la vista veniva ricalcolata di nuovo.

Effetti osservati: per un frame i nodi più a destra stavano a 23 px dal bordo invece
di 24 (R2, scarto di 1,00 px, intermittente), la vista finale oscillava
(zoom 0,7307 → 0,7297) e, dopo un ridimensionamento e il ritorno alla dimensione di
prima, la vista non tornava quella di prima (1280×720: x 236 → 181, zoom 0,453 → 0,531).

Correzione: l'effetto che chiama `onMeasure` legge la misura di adesso con
`getBoundingClientRect()` (frazionaria, come quella calcolata) invece dello stato
intero; lo stato `area` resta intero per la disposizione dei widget.

## Richieste di rete fallite

Una sola esecuzione su 15 ha registrato in console «Failed to load resource:
net::ERR_CONNECTION_REFUSED» (nessun indirizzo: la console non lo dice). Non è
stata riprodotta nelle 14 esecuzioni successive, anche con 3 esecuzioni in
parallelo. Lo script ora registra l'indirizzo di ogni richiesta fallita
(`requestfailed`) e la posizione dell'errore in console, senza filtrare nulla:
se ricompare, la prova «nessun errore in console» riporta l'URL.

## Schermate

- `cassetta-a-sinistra-scena-densa-{1440,1280x720}-{chiaro,scuro}.png`
- `inspector-in-basso-{1440,1280x720}-dopo-{chiaro,scuro}.png`; le schermate «prima»
  sono quelle già committate in `docs/visual/fase6b1/inspector-in-basso-*`.
- `striscia-apertura-inspector-in-basso.png` e `striscia-chiusura-inspector-in-basso.png`:
  sei fotogrammi (0, 20, 40, 60, 80, 100 %), con il riquadro del canvas evidenziato.
```

