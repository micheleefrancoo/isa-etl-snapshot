# 02-isa-etl-d.md

File in questo blocco:

- `src/components/isa/etl/workflow-canvas.tsx`

---

### `src/components/isa/etl/workflow-canvas.tsx` (parte 3/3)

5038 righe totali

```tsx
                );

              /*
               * Fase 4: quale pannello impostazioni dedicato mostrare
               * dal trigger tre puntini (null = tipo non ancora
               * coperto, resta il menu azioni generico esistente). Se
               * il nodo appartiene a una bubble (fase 2), gli id dei
               * membri servono al CombinePanel per raccogliere gli
               * input esterni collegati all'intera bubble.
               */
              const settingsPanelKind =
                getSettingsPanelKind(
                  node.type,
                );

              const bubbleMemberIds =
                node.groupId
                  ? bubbles.get(
                      node.groupId,
                    )?.memberIds
                  : undefined;

              /*
               * Redesign "la card e' l'icona": layout icona centrale
               * per TUTTE le categorie -- calcolato sul lato MINORE
               * della card cosi' l'icona resta quadrata anche sulle
               * card sources (non forzate quadrate come le transform,
               * vedi estimateNodeSize).
               */
              const iconLayout =
                getCardIconLayout(
                  node.width,
                  node.height,
                );

              /*
               * Redesign "la card è l'icona": dettaglio, metriche e
               * stato esecuzione non sono più mostrati in permanenza
               * sulla card (erano un badge solo per le sources) — restano
               * tutti accessibili in un unico tooltip on-hover, per
               * qualunque categoria.
               */
              const cardTooltip = [
                detail,
                display.metrics
                  ? `${formatRows(
                      analysis.rows,
                    )} rows · ${
                      analysis
                        .columns
                        .length
                    } cols`
                  : null,
                display.status
                  ? status.label
                  : null,
              ]
                .filter(Boolean)
                .join(" · ");

              /*
               * Scegliamo il lato visuale dei port
               * in funzione dei collegamenti esistenti.
               *
               * Default:
               * input a sinistra
               * output a destra
               */
              const firstIncoming =
                incoming[0];

              const firstOutgoing =
                outgoing[0];

              const incomingNode =
                firstIncoming
                  ? nodeById(
                      firstIncoming.fromNode,
                    )
                  : undefined;

              const outgoingNode =
                firstOutgoing
                  ? nodeById(
                      firstOutgoing.toNode,
                    )
                  : undefined;

              const inputSide: Side =
                incomingNode
                  ? getBestRoute(
                      incomingNode,
                      node,
                      visibleNodes,
                    )?.toSide ??
                    "left"
                  : "left";

              const outputSide: Side =
                outgoingNode
                  ? getBestRoute(
                      node,
                      outgoingNode,
                      visibleNodes,
                    )?.fromSide ??
                    "right"
                  : "right";

              const inputCount =
                def.inputs.length;

              const outputCount =
                def.outputs.length;

              const isDragging =
                dragRef.current?.id ===
                node.id;

              /*
               * Fase 3, punto 2: il drag diretto dell'utente resta
               * SEMPRE 1:1 col puntatore, senza transizione — sia per
               * la card sotto il cursore, sia per gli altri membri
               * dello stesso gruppo che la seguono rigidamente finché
               * non si staccano (isNodeUserDriven, sopra). Qualsiasi
               * ALTRA card che si sposta (spinta dall'anti-
               * sovrapposizione, o per un cambio non originato da
               * questo drag) è invece un movimento "automatico" e usa
               * la transizione di lib/etl-motion.ts.
               */
              const isUserDriven =
                isNodeUserDriven(
                  node.id,
                );

              return (
                <div
                  key={node.id}
                  data-node-card
                  data-node-id={
                    node.id
                  }
                  className={`glass-panel ${categoryAccent(def.category)} absolute flex flex-col overflow-visible rounded-2xl p-3 transition-shadow select-none ${
                    selected
                      ? "node-selected"
                      : ""
                  } ${
                    pending?.targetNode ===
                    node.id
                      ? "node-link-target"
                      : ""
                  } ${
                    freshNodeIds.has(
                      node.id,
                    )
                      ? "node-pending"
                      : ""
                  }`}
                  style={{
                    left: node.x,
                    top: node.y,
                    width: node.width,
                    height: node.height,
                    transition:
                      isUserDriven
                        ? "none"
                        : CARD_AUTO_MOVE_TRANSITION,
                    cursor: isDragging
                      ? "grabbing"
                      : "grab",
                    touchAction:
                      "none",
                    userSelect:
                      "none",
                    WebkitUserSelect:
                      "none",
                  }}
                  onPointerDown={(
                    event,
                  ) => {
                    if (
                      freshNodeIds.has(
                        node.id,
                      )
                    ) {
                      setFreshNodeIds(
                        (
                          current,
                        ) => {
                          const next =
                            new Set(
                              current,
                            );
                          next.delete(
                            node.id,
                          );
                          return next;
                        },
                      );
                    }
                    startDragNode(
                      event,
                      node.id,
                    );
                  }}
                  onPointerMove={
                    moveDragNode
                  }
                  onPointerUp={
                    endDragNode
                  }
                  onPointerCancel={
                    endDragNode
                  }
                >
                  <>
                    {/* ------------------------------------------------ */}
                    {/* Card unificata (redesign "la card e' l'icona"):  */}
                    {/* icona centrale, titolo sopra, menu impostazioni  */}
                    {/* in basso al centro -- per TUTTE le categorie,    */}
                    {/* sources incluse (non solo transform).            */}
                    {/* ------------------------------------------------ */}

                      <span
                        className="block truncate px-1 text-center text-[11px] font-semibold leading-tight"
                        title={
                          node.title
                        }
                      >
                        {
                          node.title
                        }
                      </span>

                      {/*
                       * Body permanente (dettagli/metriche/stato)
                       * rimosso dal nuovo layout per tutte le
                       * categorie: il riepilogo resta disponibile
                       * come tooltip nativo on-hover sull'icona.
                       */}
                      <div
                        className="flex min-h-0 flex-1 items-center justify-center"
                        title={
                          cardTooltip
                        }
                      >
                        <span
                          className="flex shrink-0 items-center justify-center rounded-2xl"
                          style={{
                            width:
                              iconLayout.iconBoxSize,
                            height:
                              iconLayout.iconBoxSize,
                          }}
                        >
                          <def.Icon
                            style={{
                              width:
                                iconLayout.iconGlyphSize,
                              height:
                                iconLayout.iconGlyphSize,
                            }}
                          />
                        </span>
                      </div>

                      <div className="flex justify-center">
                        <span
                          data-node-control
                          onPointerDown={(
                            event,
                          ) =>
                            event.stopPropagation()
                          }
                        >
                          {/*
                           * Fase 4: per i tipi coperti da un pannello
                           * dedicato (Filter / famiglia Combine /
                           * famiglia Aggregate) il trigger apre quel
                           * pannello, guidato dallo schema effettivo
                           * (analyzeNode), al posto del menu azioni
                           * generico. Duplica/Elimina/Scollega restano
                           * comunque raggiungibili dal menu contestuale
                           * del canvas (click destro sul nodo), quindi
                           * non è una perdita di funzionalità. I tipi
                           * transform non ancora coperti (Select,
                           * Rename, Sort, Dedupe, Fill Missing Values,
                           * Formula) mantengono il menu generico
                           * com'era in fase 1.
                           */}
                          <IsaMenu
                            label={`Impostazioni per ${node.title}`}
                            variant="bare"
                            Icon={
                              MoreHorizontal
                            }
                            width={
                              settingsPanelKind
                                ? 320
                                : undefined
                            }
                            triggerClassName="size-7 text-muted-foreground hover:text-foreground"
                            boundaryRef={
                              boxRef
                            }
                            placement="auto"
                          >
                            {(close) =>
                              settingsPanelKind ? (
                                <div className="max-h-[24rem] overflow-y-auto p-2">
                                  <div className="mb-2 flex items-center gap-2 px-1">
                                    <span
                                      className={`${categoryAccent(
                                        def.category,
                                      )} flex size-7 shrink-0 items-center justify-center rounded-lg`}
                                    >
                                      <def.Icon className="size-3.5" />
                                    </span>

                                    <span className="min-w-0 flex-1">
                                      <span className="block truncate text-xs font-semibold">
                                        {
                                          node.title
                                        }
                                      </span>
                                      <span className="block truncate text-[10px] text-muted-foreground">
                                        {
                                          def.label
                                        }
                                      </span>
                                    </span>
                                  </div>

                                  {settingsPanelKind ===
                                    "filter" && (
                                    <FilterPanel
                                      workflow={
                                        workflow
                                      }
                                      node={
                                        node
                                      }
                                      onConfigChange={(
                                        patch,
                                      ) =>
                                        onUpdateNodeConfig(
                                          node.id,
                                          patch,
                                        )
                                      }
                                    />
                                  )}

                                  {settingsPanelKind ===
                                    "combine" && (
                                    <CombinePanel
                                      workflow={
                                        workflow
                                      }
                                      node={
                                        node
                                      }
                                      memberIds={
                                        bubbleMemberIds
                                      }
                                      onConfigChange={(
                                        patch,
                                      ) =>
                                        onUpdateNodeConfig(
                                          node.id,
                                          patch,
                                        )
                                      }
                                    />
                                  )}

                                  {settingsPanelKind ===
                                    "aggregate" && (
                                    <AggregatePanel
                                      workflow={
                                        workflow
                                      }
                                      node={
                                        node
                                      }
                                      onConfigChange={(
                                        patch,
                                      ) =>
                                        onUpdateNodeConfig(
                                          node.id,
                                          patch,
                                        )
                                      }
                                    />
                                  )}
                                </div>
                              ) : (
                                <>
                                  <IsaMenuItem
                                    Icon={
                                      Copy
                                    }
                                    label="Duplica nodo"
                                    onClick={() => {
                                      onDuplicateNode(
                                        node.id,
                                      );
                                      close();
                                    }}
                                  />

                                  {hasIncoming && (
                                    <IsaMenuItem
                                      Icon={
                                        Unlink
                                      }
                                      label="Scollega input"
                                      onClick={() => {
                                        unlinkNode(
                                          node.id,
                                        );
                                        close();
                                      }}
                                    />
                                  )}

                                  {node.groupId && (
                                    <IsaMenuItem
                                      Icon={
                                        Ungroup
                                      }
                                      label="Rimuovi dal gruppo"
                                      onClick={() => {
                                        onUngroupNode(
                                          node.id,
                                        );
                                        close();
                                      }}
                                    />
                                  )}

                                  <IsaMenuItem
                                    Icon={
                                      Trash2
                                    }
                                    label="Elimina nodo"
                                    danger
                                    onClick={() => {
                                      onRemoveNode(
                                        node.id,
                                      );
                                      close();
                                    }}
                                  />

                                  <span className="my-1 block h-px bg-border" />

                                  <span className="block px-3 py-1 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                                    Visualizza
                                  </span>

                                  {DISPLAY_OPTIONS.map(
                                    (
                                      option,
                                    ) => (
                                      <IsaMenuCheckItem
                                        key={
                                          option.key
                                        }
                                        label={
                                          option.label
                                        }
                                        checked={
                                          display[
                                            option.key
                                          ]
                                        }
                                        onToggle={() =>
                                          setDisplay(
                                            (
                                              current,
                                            ) => ({
                                              ...current,
                                              [option.key]:
                                                !current[
                                                  option.key
                                                ],
                                            }),
                                          )
                                        }
                                      />
                                    ),
                                  )}
                                </>
                              )
                            }
                          </IsaMenu>
                        </span>
                      </div>
                    </>

                  {/* ------------------------------------------------ */}
                  {/* Input ports                                       */}
                  {/* ------------------------------------------------ */}

                  {def.inputs.map(
                    (
                      port,
                      index,
                    ) => {
                      const offset =
                        portOffset(
                          index,
                          inputCount,
                          inputSide ===
                            "left" ||
                            inputSide ===
                              "right"
                            ? node.height
                            : node.width,
                        );

                      const style =
                        getPortStyle(
                          inputSide,
                          offset,
                        );

                      return (
                        <span
                          key={`in-${port}`}
                          data-node-control
                          data-in-port={
                            port
                          }
                          data-node-id={
                            node.id
                          }
                          title={`Input ${port}`}
                          className="absolute z-20 flex size-5 -translate-x-1/2 -translate-y-1/2 items-center justify-center"
                          style={style}
                          onPointerDown={(
                            event,
                          ) =>
                            event.stopPropagation()
                          }
                        >
                          <span className="size-2.5 rounded-full border border-border bg-background" />
                        </span>
                      );
                    },
                  )}

                  {/* ------------------------------------------------ */}
                  {/* Output ports                                      */}
                  {/* ------------------------------------------------ */}

                  {def.outputs.map(
                    (
                      port,
                      index,
                    ) => {
                      const offset =
                        portOffset(
                          index,
                          outputCount,
                          outputSide ===
                            "left" ||
                            outputSide ===
                              "right"
                            ? node.height
                            : node.width,
                        );

                      const style =
                        getPortStyle(
                          outputSide,
                          offset,
                        );

                      return (
                        <span
                          key={`out-${port}`}
                          data-node-control
                          data-out-port={
                            port
                          }
                          title={`Output ${port} — trascina su un input`}
                          className="absolute z-20 flex size-5 -translate-x-1/2 -translate-y-1/2 cursor-crosshair items-center justify-center"
                          style={style}
                          onPointerDown={(
                            event,
                          ) =>
                            startLink(
                              event,
                              node.id,
                              port,
                            )
                          }
                        >
                          <span className="gradient-brand size-2.5 rounded-full" />
                        </span>
                      );
                    },
                  )}
                </div>
              );
            },
          )}

          {/* ------------------------------------------------------------ */}
          {/* Fase 2A: pannelli ausiliari (Inspector, Data Preview) —       */}
          {/* vivono qui, nella superficie zoomata, non come sibling nel   */}
          {/* file di rotta: vedi src/canvas/README.md                     */}
          {/* ------------------------------------------------------------ */}

          {children}
        </div>

        {/* -------------------------------------------------------------- */}
        {/* Empty state                                                    */}
        {/* -------------------------------------------------------------- */}

        {workflow.nodes.length ===
          0 && (
          <div className="pointer-events-none absolute inset-0 flex flex-col items-center justify-center gap-3 px-6 text-center select-none">
            <span className="gradient-brand flex size-12 items-center justify-center rounded-2xl text-brand-foreground">
              <Database className="size-6" />
            </span>

            <p className="text-base font-semibold">
              Build your data workflow
            </p>

            <p className="max-w-sm text-sm text-muted-foreground">
              Inizia con un dataset e
              collega trasformazioni per
              creare la tua pipeline.
            </p>

            <button
              type="button"
              onClick={
                onAddDataset
              }
              className="gradient-brand pointer-events-auto flex h-10 items-center gap-2 rounded-full px-4 text-sm font-semibold text-brand-foreground transition hover:brightness-110"
            >
              <Database className="size-4" />
              Add dataset
            </button>
          </div>
        )}
      </div>
    </section>
    </CanvasContainer>
    </CanvasStoreProvider>
  );
}

/* -------------------------------------------------------------------------- */
/*                              PORT POSITION                                 */
/* -------------------------------------------------------------------------- */

function getPortStyle(
  side: Side,
  offset: number,
): React.CSSProperties {
  switch (side) {
    case "top":
      return {
        left: offset,
        top: 0,
      };

    case "right":
      return {
        left: "100%",
        top: offset,
      };

    case "bottom":
      return {
        left: offset,
        top: "100%",
      };

    case "left":
      return {
        left: 0,
        top: offset,
      };
  }
}

```

