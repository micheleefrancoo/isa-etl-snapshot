# 02-isa-etl-c.md

File in questo blocco:

- `src/components/isa/etl/workflow-canvas.tsx`

---

### `src/components/isa/etl/workflow-canvas.tsx` (parte 2/3)

3226 righe totali

```tsx
          continue;
        }

        memberOffsets.set(other.id, {
          dx: other.x - node.x,
          dy: other.y - node.y,
          width: other.width,
          height: other.height,
        });
      }
    }

    dragRef.current = {
      id: nodeId,
      pointerId: event.pointerId,
      offsetX: point.x - node.x,
      offsetY: point.y - node.y,
      width: node.width,
      height: node.height,
      element: event.currentTarget,
      primaryStart: {
        x: node.x,
        y: node.y,
      },
      groupId: node.groupId,
      memberOffsets,
      detached: false,
      basePositions,
      livePositions,
    };

    try {
      event.currentTarget.setPointerCapture(event.pointerId);
    } catch {
      /* Pointer Capture non disponibile. */
    }
  };

  const moveDragNode = (event: React.PointerEvent<HTMLDivElement>) => {
    const drag = dragRef.current;

    if (!drag || drag.pointerId !== event.pointerId) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    const point = toLocal(event.clientX, event.clientY);

    const frame = computeDragFrame(drag, point);

    onMove(drag.id, frame.primaryDesired.x, frame.primaryDesired.y);

    for (const id of frame.rigidOtherIds) {
      const off = drag.memberOffsets.get(id)!;

      const x = frame.primaryDesired.x + off.dx;

      const y = frame.primaryDesired.y + off.dy;

      drag.livePositions.set(id, {
        x,
        y,
      });

      onMove(id, x, y);
    }

    for (const entry of frame.obstaclesForPush) {
      const pos = frame.resolvedObstacles.get(entry.id);

      const live = drag.livePositions.get(entry.id);

      if (!pos || !live) {
        continue;
      }

      if (pos.x !== live.x || pos.y !== live.y) {
        drag.livePositions.set(entry.id, pos);

        onMove(entry.id, pos.x, pos.y);
      }
    }

    setCombinePreview((current) => {
      if (!frame.combineTarget) {
        return current ? null : current;
      }

      const movingIds = [drag.id, ...frame.rigidOtherIds];

      if (
        current &&
        current.targetId === frame.combineTarget.id &&
        current.movingIds.length === movingIds.length &&
        current.movingIds.every((id, index) => id === movingIds[index])
      ) {
        return current;
      }

      return {
        movingIds,
        targetId: frame.combineTarget.id,
      };
    });
  };

  const endDragNode = (event: React.PointerEvent<HTMLDivElement>) => {
    const drag = dragRef.current;

    if (!drag || drag.pointerId !== event.pointerId) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    const point = toLocal(event.clientX, event.clientY);

    const frame = computeDragFrame(drag, point);

    onMove(drag.id, frame.primaryDesired.x, frame.primaryDesired.y, true);

    for (const id of frame.rigidOtherIds) {
      const off = drag.memberOffsets.get(id)!;

      onMove(id, frame.primaryDesired.x + off.dx, frame.primaryDesired.y + off.dy, true);
    }

    /*
     * Commit anche per le card spostate per effetto domino, ma solo
     * quelle davvero finite fuori dalla loro posizione di partenza:
     * evita voci di history superflue per i nodi mai toccati.
     */
    for (const entry of frame.obstaclesForPush) {
      const pos = frame.resolvedObstacles.get(entry.id) ?? {
        x: entry.x,
        y: entry.y,
      };

      if (pos.x !== entry.x || pos.y !== entry.y) {
        onMove(entry.id, pos.x, pos.y, true);
      }
    }

    /*
     * Distacco dal gruppo di partenza: va risolto PRIMA di un eventuale
     * nuovo combine, altrimenti groupNodes vedrebbe ancora il vecchio
     * groupId di questo nodo e vi trascinerebbe dentro anche i membri
     * da cui ci si è appena staccati.
     */
    if (drag.detached && drag.groupId) {
      onUngroupNode(drag.id);
    }

    if (frame.combineTarget) {
      const movingIds = [drag.id, ...frame.rigidOtherIds];

      const targetGroupId = frame.combineTarget.groupId;

      const targetGroupMembers = targetGroupId
        ? nodesRef.current.filter((n) => n.groupId === targetGroupId).map((n) => n.id)
        : [];

      const fullIds = Array.from(
        new Set([...movingIds, frame.combineTarget.id, ...targetGroupMembers]),
      );

      onGroupNodes(fullIds);
    }

    setCombinePreview(null);

    try {
      if (event.currentTarget.hasPointerCapture(event.pointerId)) {
        event.currentTarget.releasePointerCapture(event.pointerId);
      }
    } catch {
      /* no-op */
    }

    dragRef.current = null;
  };

  /* ---------------------------------------------------------------------- */
  /*                              LINK DRAG                                  */
  /* ---------------------------------------------------------------------- */

  const startLink = (event: React.PointerEvent<HTMLSpanElement>, nodeId: string, port: string) => {
    if (event.button !== 0) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    closeContextMenu();

    try {
      event.currentTarget.setPointerCapture(event.pointerId);
    } catch {
      /* no-op */
    }

    const point = toLocal(event.clientX, event.clientY);

    setPending({
      node: nodeId,
      port,
      x: point.x,
      y: point.y,
      targetNode: null,
    });

    const move = (moveEvent: PointerEvent) => {
      const next = toLocal(moveEvent.clientX, moveEvent.clientY);

      const element = document.elementFromPoint(moveEvent.clientX, moveEvent.clientY);

      const card = element?.closest<HTMLElement>("[data-node-card]");

      const targetId = card?.dataset["nodeId"];

      const target = targetId ? liveNodeById(targetId) : undefined;

      if (target && target.id !== nodeId) {
        setPending((current) =>
          current
            ? {
                ...current,
                x: next.x,
                y: next.y,
                targetNode: target.id,
              }
            : current,
        );

        return;
      }

      setPending((current) =>
        current
          ? {
              ...current,
              x: next.x,
              y: next.y,
              targetNode: null,
            }
          : current,
      );
    };

    const up = (upEvent: PointerEvent) => {
      const element = document.elementFromPoint(upEvent.clientX, upEvent.clientY);

      /*
       * Non è più necessario rilasciare esattamente sopra la porta:
       * basta essere sopra una qualsiasi card valida. Il collegamento
       * usa la porta di input più vicina al punto di rilascio.
       */
      const card = element?.closest<HTMLElement>("[data-node-card]");

      const targetId = card?.dataset["nodeId"];

      const target = targetId ? liveNodeById(targetId) : undefined;

      if (target && target.id !== nodeId) {
        const targetDef = nodeDef(target.type);

        const inputs = targetDef?.inputs ?? [];

        if (inputs.length > 0) {
          const drop = toLocal(upEvent.clientX, upEvent.clientY);

          let inPort = inputs[0]!;

          if (inputs.length > 1) {
            const ratio = (drop.y - target.y) / Math.max(1, target.height);

            const index = Math.min(
              inputs.length - 1,
              Math.max(0, Math.round(ratio * (inputs.length - 1))),
            );

            inPort = inputs[index] ?? inPort;
          }

          onConnect(nodeId, port, target.id, inPort);
        }
      }

      window.removeEventListener("pointermove", move);

      window.removeEventListener("pointerup", up);

      window.removeEventListener("pointercancel", up);

      setPending(null);
    };

    window.addEventListener("pointermove", move);

    window.addEventListener("pointerup", up);

    window.addEventListener("pointercancel", up);
  };

  /* ---------------------------------------------------------------------- */
  /*                          CONTEXT MENU STATE                             */
  /* ---------------------------------------------------------------------- */

  const closeContextMenu = useCallback(() => {
    setContextMenu((current) => ({
      ...current,
      open: false,
    }));
  }, []);

  const openContextMenu = (event: React.MouseEvent) => {
    event.preventDefault();
    event.stopPropagation();

    const card = (event.target as HTMLElement).closest<HTMLElement>("[data-node-card]");

    const nodeId = card?.dataset["nodeId"] ?? null;

    if (nodeId) {
      onSelect(nodeId);
    }

    setContextMenu({
      open: true,
      x: event.clientX,
      y: event.clientY,
      nodeId,
    });
  };

  const unlinkNode = useCallback(
    (nodeId: string) => {
      workflow.edges
        .filter((edge) => edge.toNode === nodeId)
        .forEach((edge) => onRemoveEdge(edge.id));
    },
    [workflow.edges, onRemoveEdge],
  );

  /* ---------------------------------------------------------------------- */
  /*                              PALETTE                                    */
  /* ---------------------------------------------------------------------- */

  const dockPosition: Record<Dock, string> = {
    top: "left-1/2 top-0 -translate-x-1/2",
    bottom: "bottom-0 left-1/2 -translate-x-1/2",
    left: "left-0 top-1/2 -translate-y-1/2",
    right: "right-0 top-1/2 -translate-y-1/2",
  };

  const palette = (
    <div ref={paletteRef} className={`absolute z-40 ${dockPosition[paletteDock]}`}>
      <ToolPalette
        dock={paletteDock}
        onDockChange={setPaletteDock}
        onAdd={handlePaletteDoubleClick}
        onDragAdd={handlePaletteDrop}
      />
    </div>
  );

  /* ---------------------------------------------------------------------- */
  /*                            PENDING ROUTE                                */
  /* ---------------------------------------------------------------------- */

  const pendingRoute = useMemo(() => {
    if (!pending) {
      return null;
    }

    const source = nodeById(pending.node);

    if (!source) {
      return null;
    }

    if (pending.targetNode) {
      const target = nodeById(pending.targetNode);

      if (target) {
        const route = getBestRoute(source, target, visibleNodes);

        const indicator = getDropIndicatorPoint(target, surfaceW, surfaceH);

        return {
          path: route
            ? pathFromPoints(route.points)
            : pathFromPoints([getAnchor(source, "right"), indicator]),
          indicator,
        };
      }
    }

    /*
     * Trascinamento libero (nessun target): la linea esce comunque
     * perpendicolare dal lato della card rivolto verso il puntatore.
     */
    const cx = source.x + source.width / 2;
    const cy = source.y + source.height / 2;

    const dx = pending.x - cx;
    const dy = pending.y - cy;

    const sourceSide: Side =
      Math.abs(dx) >= Math.abs(dy) ? (dx >= 0 ? "right" : "left") : dy >= 0 ? "bottom" : "top";

    const sourceAnchor = getAnchor(source, sourceSide);

    const stub = stubPoint(sourceAnchor, PORT_STUB);

    const horizontalFirst = sourceSide === "left" || sourceSide === "right";

    return {
      path: pathFromPoints(
        simplifyPath([
          sourceAnchor,
          stub,
          horizontalFirst
            ? {
                x: pending.x,
                y: stub.y,
              }
            : {
                x: stub.x,
                y: pending.y,
              },
          {
            x: pending.x,
            y: pending.y,
          },
        ]),
      ),
      indicator: null,
    };
  }, [pending, nodeById, visibleNodes, surfaceW, surfaceH]);

  /* ---------------------------------------------------------------------- */
  /*                         GROUP CONTAINERS                                */
  /* ---------------------------------------------------------------------- */

  /*
   * Geometria di ogni bubble (gruppo con 2+ membri), indicizzata per
   * groupId — ricalcolata a ogni render dalle posizioni CORRENTI dei
   * nodi (fase 2, punto 4: bounding box/orientamento sempre coerenti
   * con l'ultima disposizione, senza bisogno di gestire esplicitamente
   * "ingresso/uscita dalla bubble" come evento a parte). Usata sia per
   * disegnare il contenitore sia, più sotto, per instradare le frecce
   * esterne sul suo perimetro invece che sui singoli nodi interni.
   */
  const bubbles = useMemo(() => computeBubbles(visibleNodes), [visibleNodes]);

  /** Un box per ogni bubble esistente, per il contenitore visivo. */
  const groupBoxes = useMemo(
    () =>
      Array.from(bubbles.values()).map((bubble) => ({
        groupId: bubble.groupId,
        rect: bubble.rect,
        orientation: bubble.orientation,
        memberIds: bubble.memberIds,
      })),
    [bubbles],
  );

  /*
   * Fase 3: distingue un movimento "automatico" (anti-sovrapposizione,
   * ingresso/uscita da una bubble, cambio di orientamento/resize) da
   * un movimento originato dal drag diretto dell'utente — solo il
   * primo va animato con transizione, il secondo resta sempre 1:1 col
   * puntatore. Un nodo è "user-driven" se è quello sotto il cursore
   * (drag.id) o se sta seguendo rigidamente il gruppo del nodo sotto
   * il cursore (membro del gruppo, non ancora staccato). Usata sia per
   * le card sia — indirettamente, tramite i membri della bubble — per
   * il contenitore della bubble e per le frecce agganciate a un suo
   * perimetro (vedi il loop degli edge più sopra).
   */
  const isNodeUserDriven = (nodeId: string): boolean => {
    const drag = dragRef.current;

    if (!drag) {
      return false;
    }

    if (drag.id === nodeId) {
      return true;
    }

    return !drag.detached && drag.memberOffsets.has(nodeId);
  };

  /**
   * Il contenitore di una bubble non deve avere lag quando uno dei
   * suoi membri è sotto il drag diretto dell'utente (altrimenti il box
   * "insegue" con ritardo la card che lo sta facendo cambiare forma in
   * tempo reale) — ma deve animarsi morbidamente per qualunque altro
   * cambiamento (resize, cambio di orientamento, ingresso/uscita di un
   * membro).
   */
  const isBubbleUserDriven = (memberIds: readonly string[]): boolean =>
    memberIds.some(isNodeUserDriven);

  /**
   * Anteprima del box combinato mostrata durante il drag: racchiude le
   * card in movimento e quella candidata, finché restano abbastanza
   * vicine da poter essere unite al rilascio.
   */
  const combinePreviewBox = useMemo(() => {
    if (!combinePreview) {
      return null;
    }

    const target = nodeById(combinePreview.targetId);

    const movers = combinePreview.movingIds
      .map(nodeById)
      .filter((node): node is NodeGeometry => !!node);

    if (!target || movers.length === 0) {
      return null;
    }

    return unionRect([target, ...movers]);
  }, [combinePreview, nodeById]);

  /* ---------------------------------------------------------------------- */
  /*                               RENDER                                    */
  /* ---------------------------------------------------------------------- */

  return (
    <CanvasStoreProvider>
      <CanvasContainer containerSize={{ width: surfaceW, height: surfaceH }} zoom={zoom}>
        <section
          ref={boxRef}
          data-palette-workspace
          className="glass-soft relative min-h-0 min-w-0 flex-1 overflow-hidden rounded-3xl select-none"
          style={{
            userSelect: "none",
            WebkitUserSelect: "none",
          }}
          onContextMenu={openContextMenu}
        >
          <CanvasContextMenu
            state={contextMenu}
            boundaryRef={boxRef}
            grid={grid}
            onClose={closeContextMenu}
            onAddDataset={onAddDataset}
            onFitView={fitView}
            onToggleGrid={() => setGrid((value) => !value)}
            onAutoLayout={() => onLayoutChange("auto")}
            onDuplicateNode={onDuplicateNode}
            onRemoveNode={onRemoveNode}
            onUnlinkNode={unlinkNode}
          />

          {palette}

          {/* --------------------------------------------------------------- */}
          {/* Canvas controls                                                  */}
          {/* --------------------------------------------------------------- */}

          <div
            className="absolute right-3 top-3 z-30 flex items-center gap-1.5"
            onPointerDown={(event) => event.stopPropagation()}
          >
            <span className="glass-chip flex h-8 items-center rounded-full px-2.5 text-[10px] text-muted-foreground">
              {Math.round(zoom * 100)}%
            </span>

            <button
              type="button"
              onClick={() => setZoom((z) => Math.max(0.4, +(z - 0.1).toFixed(2)))}
              aria-label="Zoom out"
              className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
            >
              <ZoomOut className="size-4" />
            </button>

            <button
              type="button"
              onClick={() => setZoom((z) => Math.min(1.8, +(z + 0.1).toFixed(2)))}
              aria-label="Zoom in"
              className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
            >
              <ZoomIn className="size-4" />
            </button>

            <button
              type="button"
              onClick={fitView}
              aria-label="Fit view"
              className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
            >
              <Maximize2 className="size-4" />
            </button>

            <button
              type="button"
              onClick={() => setGrid((value) => !value)}
              aria-pressed={grid}
              aria-label={grid ? "Nascondi griglia" : "Mostra griglia"}
              title={grid ? "Nascondi griglia" : "Mostra griglia"}
              className={`glass-chip flex size-8 items-center justify-center rounded-full transition ${
                grid ? "text-foreground" : "text-muted-foreground/50 hover:text-foreground"
              }`}
            >
              <Grid3x3 className="size-4" />
            </button>

            <IsaMenu
              label="Disposizione delle card"
              Icon={workflow.layout === "auto" ? LayoutGrid : Move}
              boundaryRef={boxRef}
              placement="auto"
            >
              {(close) => (
                <>
                  <span className="block px-3 py-1.5 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                    Disposizione
                  </span>

                  <IsaMenuCheckItem
                    label="Automatica"
                    checked={workflow.layout === "auto"}
                    onToggle={() => {
                      onLayoutChange("auto");
                      close();
                    }}
                  />

                  <IsaMenuCheckItem
                    label="Trascinamento libero"
                    checked={workflow.layout !== "auto"}
                    onToggle={() => {
                      onLayoutChange("manual");
                      close();
                    }}
                  />
                </>
              )}
            </IsaMenu>

            <IsaMenu label="Visualizza sulle card" Icon={Eye} boundaryRef={boxRef} placement="auto">
              {() => (
                <>
                  <span className="block px-3 py-1.5 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                    Visualizza
                  </span>

                  {DISPLAY_OPTIONS.map((option) => (
                    <IsaMenuCheckItem
                      key={option.key}
                      label={option.label}
                      checked={display[option.key]}
                      onToggle={() =>
                        setDisplay((current) => ({
                          ...current,
                          [option.key]: !current[option.key],
                        }))
                      }
                    />
                  ))}
                </>
              )}
            </IsaMenu>
          </div>

          {/* --------------------------------------------------------------- */}
          {/* Canvas surface                                                  */}
          {/* --------------------------------------------------------------- */}

          <div
            className="relative h-full w-full overflow-hidden"
            style={{
              backgroundImage: grid
                ? "radial-gradient(circle, color-mix(in oklab, var(--foreground) 22%, transparent) 1px, transparent 1px)"
                : undefined,
              backgroundSize: grid
                ? `${Math.round(24 * zoom)}px ${Math.round(24 * zoom)}px`
                : undefined,
              userSelect: "none",
              WebkitUserSelect: "none",
            }}
            onPointerDown={() => {
              closeContextMenu();
              onSelect(null);
            }}
          >
            <div
              ref={surfaceRef}
              className="relative origin-top-left"
              style={{
                width: surfaceW,
                height: surfaceH,
                transform: `scale(${zoom})`,
                userSelect: "none",
                WebkitUserSelect: "none",
              }}
            >
              {/* ---------------------------------------------------------- */}
              {/* Edges                                                      */}
              {/* ---------------------------------------------------------- */}

              <svg className="pointer-events-none absolute inset-0 size-full overflow-visible">
                {workflow.edges.map((edge, edgeIndex) => {
                  const from = nodeById(edge.fromNode);

                  const to = nodeById(edge.toNode);

                  if (!from || !to) {
                    return null;
                  }

                  const fromBubble = from.groupId ? bubbles.get(from.groupId) : undefined;

                  const toBubble = to.groupId ? bubbles.get(to.groupId) : undefined;

                  /*
                   * Fase 2, punto 2: nessuna freccia interna — entrambi
                   * gli endpoint nella STESSA bubble non vengono
                   * disegnati (il collegamento resta nel modello dati,
                   * solo non è renderizzato).
                   */
                  if (fromBubble && toBubble && fromBubble.groupId === toBubble.groupId) {
                    return null;
                  }

                  /*
                   * Fase 2, punto 3: un endpoint che appartiene a una
                   * bubble si aggancia sul perimetro del suo bounding
                   * box (stesso identico algoritmo di routing di un
                   * nodo singolo, solo con un rettangolo diverso), non
                   * sulla card interna specifica. I membri della bubble
                   * vanno esclusi dagli ostacoli, altrimenti il routing
                   * proverebbe a evitare le proprie card interne.
                   */
                  const fromGeometry = fromBubble ? bubbleNodeGeometry(from, fromBubble) : from;

                  const toGeometry = toBubble ? bubbleNodeGeometry(to, toBubble) : to;

                  const excludeIds =
                    fromBubble || toBubble
                      ? new Set<string>([
                          ...(fromBubble?.memberIds ?? []),
                          ...(toBubble?.memberIds ?? []),
                        ])
                      : undefined;

                  const route = getBestRoute(fromGeometry, toGeometry, visibleNodes, excludeIds);

                  if (!route) {
                    return null;
                  }

                  /*
                   * `route.points` va sempre da from (edge.fromNode) a to
                   * (edge.toNode), perché `getBestRoute(from, to, ...)` è
                   * chiamata così qui sopra: la direzione della luce di
                   * flusso, animata lungo questo stesso path da offset
                   * 0% a 100%, segue quindi il verso reale del
                   * collegamento nel modello dati — non un'euristica sul
                   * tipo di nodo (es. "i Dataset sono sempre origine").
                   * Una card transform con più archi in ingresso e in
                   * uscita mostra quindi ogni luce nel verso corretto
                   * per il proprio arco, indipendentemente dagli altri.
                   */
                  const path = smoothPathFromPoints(route.points);

                  const active =
                    edge.fromNode === selectedId ||
                    edge.toNode === selectedId ||
                    hoverEdge === edge.id;

                  /*
                   * Fase 3, punto 3: la freccia segue la STESSA regola
                   * "user-driven" della card/bubble a cui è agganciata —
                   * se l'endpoint è dentro una bubble, basta che UNO dei
                   * suoi membri sia sotto drag diretto (la bubble intera
                   * si sta muovendo dal vivo), non necessariamente lo
                   * specifico nodo `edge.fromNode`/`edge.toNode`.
                   */
                  const edgeIsUserDriven =
                    (fromBubble
                      ? fromBubble.memberIds.some(isNodeUserDriven)
                      : isNodeUserDriven(edge.fromNode)) ||
                    (toBubble
                      ? toBubble.memberIds.some(isNodeUserDriven)
                      : isNodeUserDriven(edge.toNode));

                  const edgeTransition = edgeIsUserDriven ? "none" : EDGE_AUTO_MOVE_TRANSITION;

                  const rows = formatRows(analyzeNode(workflow, from).rows);

                  const middlePoint = route.points[Math.floor(route.points.length / 2)];

                  return (
                    <g key={edge.id}>
                      {/* Hit area */}
                      <path
                        d={path}
                        fill="none"
                        stroke="transparent"
                        strokeWidth={14}
                        style={{
                          transition: edgeTransition,
                        }}
                        className="pointer-events-auto cursor-pointer"
                        onPointerEnter={() => setHoverEdge(edge.id)}
                        onPointerLeave={() =>
                          setHoverEdge((current) => (current === edge.id ? null : current))
                        }
                        onDoubleClick={() => onRemoveEdge(edge.id)}
                      />

                      {/* Solo linea, nessuna freccia */}
                      <path
                        d={path}
                        fill="none"
                        stroke="var(--brand)"
                        strokeWidth={active ? 2.8 : 2}
                        strokeOpacity={active ? 0.95 : 0.42}
                        strokeLinecap="round"
                        strokeLinejoin="round"
                        style={{
                          transition: edgeTransition,
                        }}
                        className={`pointer-events-none ${active ? "edge-flow" : ""}`}
                      />

                      {/*
                       * Luce di flusso direzionale: piccolo punto che
                       * percorre lo stesso path (da from a to, vedi
                       * commento sopra) via CSS offset-path, in loop.
                       * Nessun ricalcolo per frame: il browser interpola
                       * offset-distance, niente rAF.
                       */}
                      <circle
                        r={active ? 2.6 : 2.1}
                        fill="var(--brand-glow)"
                        className="edge-flow-dot pointer-events-none"
                        style={
                          {
                            offsetPath: `path("${path}")`,
                            animationDelay: `${-((edgeIndex % 6) * 0.45)}s`,
                            /*
                             * Letto dalla keyframe come picco di
                             * opacità: un valore statico su `opacity`
                             * qui verrebbe ignorato, perché
                             * un'animazione CSS sovrascrive sempre lo
                             * stile inline della stessa proprietà.
                             */
                            "--edge-flow-peak": active ? 0.9 : 0.55,
                          } as React.CSSProperties
                        }
                      />

                      {hoverEdge === edge.id && middlePoint && (
                        <g className="pointer-events-none">
                          <rect
                            x={middlePoint.x - 34}
                            y={middlePoint.y - 20}
                            width={68}
                            height={18}
                            rx={4}
                            fill="var(--background)"
                            stroke="var(--glass-border)"
                          />

                          <text
                            x={middlePoint.x}
                            y={middlePoint.y - 7}
                            textAnchor="middle"
                            fontSize={10}
                            fill="var(--muted-foreground)"
                          >
                            {rows} rows
                          </text>
                        </g>
                      )}
                    </g>
                  );
                })}

                {/* -------------------------------------------------------- */}
                {/* Linking preview                                          */}
                {/* -------------------------------------------------------- */}

                {pendingRoute && (
                  <g>
                    <path
                      d={pendingRoute.path}
                      fill="none"
                      stroke="var(--brand)"
                      strokeWidth={2}
                      strokeDasharray="5 5"
                      strokeLinecap="round"
                      strokeLinejoin="round"
                    />

                    {pendingRoute.indicator && (
                      <circle
                        cx={pendingRoute.indicator.x}
                        cy={pendingRoute.indicator.y}
                        r={7}
                        fill="var(--background)"
                        stroke="var(--brand)"
                        strokeWidth={2}
                      />
                    )}
                  </g>
                )}
              </svg>

              {/* ---------------------------------------------------------- */}
              {/* Group containers                                           */}
              {/* ---------------------------------------------------------- */}

              {groupBoxes.map(({ groupId, rect, orientation, memberIds }) => (
                <div
                  key={groupId}
                  data-bubble-orientation={orientation}
                  /*
                   * L'orientamento è esposto qui (nessun effetto visivo
                   * oltre alla forma naturale del bounding box in questa
                   * fase): una futura fase lo userà per riallineare i
                   * membri lungo l'asse della bubble con un'animazione.
                   */
                  className="glass-soft pointer-events-none absolute rounded-3xl"
                  style={{
                    left: rect.x - GROUP_PADDING,
                    top: rect.y - GROUP_PADDING,
                    width: rect.width + GROUP_PADDING * 2,
                    height: rect.height + GROUP_PADDING * 2,
                    transition: isBubbleUserDriven(memberIds)
                      ? "none"
                      : BUBBLE_AUTO_MOVE_TRANSITION,
                    opacity: 0.55,
                  }}
                />
              ))}

              {combinePreviewBox && (
                <div
                  className="gradient-brand pointer-events-none absolute rounded-3xl border-2 border-dashed border-brand"
                  style={{
                    left: combinePreviewBox.x - GROUP_PADDING,
                    top: combinePreviewBox.y - GROUP_PADDING,
                    width: combinePreviewBox.width + GROUP_PADDING * 2,
                    height: combinePreviewBox.height + GROUP_PADDING * 2,
                    opacity: 0.22,
                  }}
                />
              )}

              {/* ---------------------------------------------------------- */}
              {/* Cards                                                      */}
              {/* ---------------------------------------------------------- */}

              {visibleNodes.map((node) => {
                const def = nodeDef(node.type);

                if (!def) {
                  return null;
                }

                const analysis = analyzeNode(workflow, node);

                const incoming = workflow.edges.filter((edge) => edge.toNode === node.id);

                const outgoing = workflow.edges.filter((edge) => edge.fromNode === node.id);

                const selected = node.id === selectedId;

                const hasIncoming = incoming.length > 0;

                const hasOutgoing = outgoing.length > 0;

                const status =
                  STATUS[
                    analysis.errors.length > 0 && node.status !== "running" ? "error" : node.status
                  ];

                const detail = nodeSummary(node.type, node.config);

                /*
                 * Fase 4: quale pannello impostazioni dedicato mostrare
                 * dal trigger tre puntini (null = tipo non ancora
                 * coperto, resta il menu azioni generico esistente). Se
                 * il nodo appartiene a una bubble (fase 2), gli id dei
                 * membri servono al CombinePanel per raccogliere gli
                 * input esterni collegati all'intera bubble.
                 */
                const settingsPanelKind = getSettingsPanelKind(node.type);

                const bubbleMemberIds = node.groupId
                  ? bubbles.get(node.groupId)?.memberIds
                  : undefined;

                /*
                 * Redesign "la card e' l'icona": layout icona centrale
                 * per TUTTE le categorie -- calcolato sul lato MINORE
                 * della card cosi' l'icona resta quadrata anche sulle
                 * card sources (non forzate quadrate come le transform,
                 * vedi estimateNodeSize).
                 */
                const iconLayout = getCardIconLayout(node.width, node.height);

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
                    ? `${formatRows(analysis.rows)} rows · ${analysis.columns.length} cols`
                    : null,
                  display.status ? status.label : null,
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
                const firstIncoming = incoming[0];

                const firstOutgoing = outgoing[0];

                const incomingNode = firstIncoming ? nodeById(firstIncoming.fromNode) : undefined;

                const outgoingNode = firstOutgoing ? nodeById(firstOutgoing.toNode) : undefined;

                const inputSide: Side = incomingNode
                  ? (getBestRoute(incomingNode, node, visibleNodes)?.toSide ?? "left")
                  : "left";

                const outputSide: Side = outgoingNode
                  ? (getBestRoute(node, outgoingNode, visibleNodes)?.fromSide ?? "right")
                  : "right";

                const inputCount = def.inputs.length;

                const outputCount = def.outputs.length;

                const isDragging = dragRef.current?.id === node.id;

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
                const isUserDriven = isNodeUserDriven(node.id);

                return (
                  <div
                    key={node.id}
                    data-node-card
                    data-node-id={node.id}
                    className={`glass-panel ${categoryAccent(def.category)} absolute flex flex-col overflow-visible rounded-2xl p-3 transition-shadow select-none ${
                      selected ? "node-selected" : ""
                    } ${pending?.targetNode === node.id ? "node-link-target" : ""} ${
                      freshNodeIds.has(node.id) ? "node-pending" : ""
                    }`}
                    style={{
                      left: node.x,
                      top: node.y,
                      width: node.width,
                      height: node.height,
                      transition: isUserDriven ? "none" : CARD_AUTO_MOVE_TRANSITION,
                      cursor: isDragging ? "grabbing" : "grab",
                      touchAction: "none",
                      userSelect: "none",
                      WebkitUserSelect: "none",
                    }}
                    onPointerDown={(event) => {
                      if (freshNodeIds.has(node.id)) {
                        setFreshNodeIds((current) => {
                          const next = new Set(current);
                          next.delete(node.id);
                          return next;
                        });
                      }
                      startDragNode(event, node.id);
                    }}
                    onPointerMove={moveDragNode}
                    onPointerUp={endDragNode}
                    onPointerCancel={endDragNode}
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
                        title={node.title}
                      >
                        {node.title}
                      </span>

                      {/*
                       * Body permanente (dettagli/metriche/stato)
                       * rimosso dal nuovo layout per tutte le
                       * categorie: il riepilogo resta disponibile
                       * come tooltip nativo on-hover sull'icona.
                       */}
                      <div
                        className="flex min-h-0 flex-1 items-center justify-center"
                        title={cardTooltip}
                      >
                        <span
                          className="flex shrink-0 items-center justify-center rounded-2xl"
                          style={{
                            width: iconLayout.iconBoxSize,
                            height: iconLayout.iconBoxSize,
                          }}
                        >
                          <def.Icon
                            style={{
                              width: iconLayout.iconGlyphSize,
                              height: iconLayout.iconGlyphSize,
                            }}
                          />
                        </span>
                      </div>

                      <div className="flex justify-center">
                        <span data-node-control onPointerDown={(event) => event.stopPropagation()}>
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
                            Icon={MoreHorizontal}
                            width={settingsPanelKind ? 320 : undefined}
                            triggerClassName="size-7 text-muted-foreground hover:text-foreground"
                            boundaryRef={boxRef}
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
                                        {node.title}
                                      </span>
                                      <span className="block truncate text-[10px] text-muted-foreground">
                                        {def.label}
                                      </span>
                                    </span>
                                  </div>

                                  {settingsPanelKind === "filter" && (
                                    <FilterPanel
                                      workflow={workflow}
                                      node={node}
                                      onConfigChange={(patch) => onUpdateNodeConfig(node.id, patch)}
                                    />
                                  )}

                                  {settingsPanelKind === "combine" && (
                                    <CombinePanel
                                      workflow={workflow}
                                      node={node}
                                      memberIds={bubbleMemberIds}
                                      onConfigChange={(patch) => onUpdateNodeConfig(node.id, patch)}
                                    />
                                  )}

                                  {settingsPanelKind === "aggregate" && (
                                    <AggregatePanel
                                      workflow={workflow}
                                      node={node}
                                      onConfigChange={(patch) => onUpdateNodeConfig(node.id, patch)}
                                    />
                                  )}
                                </div>
                              ) : (
                                <>
                                  <IsaMenuItem
                                    Icon={Copy}
                                    label="Duplica nodo"
                                    onClick={() => {
                                      onDuplicateNode(node.id);
                                      close();
                                    }}
                                  />

                                  {hasIncoming && (
                                    <IsaMenuItem
                                      Icon={Unlink}
                                      label="Scollega input"
                                      onClick={() => {
                                        unlinkNode(node.id);
                                        close();
                                      }}
                                    />
                                  )}

                                  {node.groupId && (
                                    <IsaMenuItem
                                      Icon={Ungroup}
                                      label="Rimuovi dal gruppo"
                                      onClick={() => {
                                        onUngroupNode(node.id);
                                        close();
                                      }}
                                    />
                                  )}

                                  <IsaMenuItem
                                    Icon={Trash2}
                                    label="Elimina nodo"
                                    danger
                                    onClick={() => {
                                      onRemoveNode(node.id);
                                      close();
                                    }}
                                  />

                                  <span className="my-1 block h-px bg-border" />

                                  <span className="block px-3 py-1 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                                    Visualizza
                                  </span>

                                  {DISPLAY_OPTIONS.map((option) => (
                                    <IsaMenuCheckItem
                                      key={option.key}
                                      label={option.label}
                                      checked={display[option.key]}
                                      onToggle={() =>
                                        setDisplay((current) => ({
                                          ...current,
                                          [option.key]: !current[option.key],
                                        }))
                                      }
                                    />
                                  ))}
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

                    {def.inputs.map((port, index) => {
                      const offset = portOffset(
                        index,
                        inputCount,
                        inputSide === "left" || inputSide === "right" ? node.height : node.width,
                      );

                      const style = getPortStyle(inputSide, offset);

                      return (
                        <span
                          key={`in-${port}`}
                          data-node-control
                          data-in-port={port}
                          data-node-id={node.id}
                          title={`Input ${port}`}
                          className="absolute z-20 flex size-5 -translate-x-1/2 -translate-y-1/2 items-center justify-center"
                          style={style}
```

### `src/components/isa/etl/workflow-canvas.tsx` (parte 3/3)

3226 righe totali

```tsx
                          onPointerDown={(event) => event.stopPropagation()}
                        >
                          <span className="size-2.5 rounded-full border border-border bg-background" />
                        </span>
                      );
                    })}

                    {/* ------------------------------------------------ */}
                    {/* Output ports                                      */}
                    {/* ------------------------------------------------ */}

                    {def.outputs.map((port, index) => {
                      const offset = portOffset(
                        index,
                        outputCount,
                        outputSide === "left" || outputSide === "right" ? node.height : node.width,
                      );

                      const style = getPortStyle(outputSide, offset);

                      return (
                        <span
                          key={`out-${port}`}
                          data-node-control
                          data-out-port={port}
                          title={`Output ${port} — trascina su un input`}
                          className="absolute z-20 flex size-5 -translate-x-1/2 -translate-y-1/2 cursor-crosshair items-center justify-center"
                          style={style}
                          onPointerDown={(event) => startLink(event, node.id, port)}
                        >
                          <span className="gradient-brand size-2.5 rounded-full" />
                        </span>
                      );
                    })}
                  </div>
                );
              })}

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

            {workflow.nodes.length === 0 && (
              <div className="pointer-events-none absolute inset-0 flex flex-col items-center justify-center gap-3 px-6 text-center select-none">
                <span className="gradient-brand flex size-12 items-center justify-center rounded-2xl text-brand-foreground">
                  <Database className="size-6" />
                </span>

                <p className="text-base font-semibold">Build your data workflow</p>

                <p className="max-w-sm text-sm text-muted-foreground">
                  Inizia con un dataset e collega trasformazioni per creare la tua pipeline.
                </p>

                <button
                  type="button"
                  onClick={onAddDataset}
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

function getPortStyle(side: Side, offset: number): React.CSSProperties {
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

