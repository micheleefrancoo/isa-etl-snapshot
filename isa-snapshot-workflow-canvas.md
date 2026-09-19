# ISA ETL Snapshot

Generated: 2026-09-19T10:28:53Z

## Index
- src/components/isa/etl/workflow-canvas.tsx


=== FILE: src/components/isa/etl/workflow-canvas.tsx ===
import {
  Copy,
  Database,
  Eye,
  Grid3x3,
  LayoutGrid,
  Maximize2,
  MoreHorizontal,
  Move,
  Plus,
  Trash2,
  Ungroup,
  Unlink,
  ZoomIn,
  ZoomOut,
} from "lucide-react";
import {
  useCallback,
  useEffect,
  useMemo,
  useRef,
  useState,
} from "react";
import { createPortal } from "react-dom";

import { ToolPalette } from "@/components/isa/etl/tool-palette";
import type { Dock } from "@/components/isa/etl/tool-palette";
import { AggregatePanel } from "@/components/isa/etl/settings-panels/aggregate-panel";
import { CombinePanel } from "@/components/isa/etl/settings-panels/combine-panel";
import { FilterPanel } from "@/components/isa/etl/settings-panels/filter-panel";
import {
  IsaMenu,
  IsaMenuCheckItem,
  IsaMenuItem,
} from "@/components/isa/ui/isa-menu";
import {
  categoryAccent,
  nodeDef,
  nodeSummary,
} from "@/lib/etl-catalog";
import type { EtlDisplaySettings } from "@/lib/etl-display";
import { getSettingsPanelKind } from "@/lib/etl-node-config";
import {
  DEFAULT_DISPLAY,
  DISPLAY_OPTIONS,
  getCardIconLayout,
} from "@/lib/etl-display";
import type { BubbleGeometry } from "@/lib/etl-bubble";
import { computeBubbles } from "@/lib/etl-bubble";
import {
  BUBBLE_AUTO_MOVE_TRANSITION,
  CARD_AUTO_MOVE_TRANSITION,
  EDGE_AUTO_MOVE_TRANSITION,
} from "@/lib/etl-motion";
import {
  analyzeNode,
  formatRows,
} from "@/lib/etl-schema";
import type {
  EtlNode,
  EtlWorkflow,
  LayoutMode,
  NodeStatus,
} from "@/lib/etl-workflow";
import type { NodeSize } from "@/lib/etl-node-size";
import {
  MIN_NODE_HEIGHT,
  NODE_H,
  NODE_W,
  ROUTE_GAP,
  estimateNodeSize,
} from "@/lib/etl-node-size";

/*
 * Raggio (px, coordinate superficie) con cui vengono arrotondati i
 * vertici del path SVG delle frecce, puramente in fase di rendering —
 * i waypoint restano quelli calcolati da `getBestRoute`, qui si
 * smussa solo il tratto disegnato. Tenuto sotto PORT_STUB così
 * l'arrotondamento non "mangia" mai lo stub rettilineo di aggancio.
 */
const EDGE_CORNER_RADIUS = 10;

/*
 * Tratto rettilineo obbligatorio con cui una freccia entra/esce da una
 * card: garantisce che l'aggancio sia sempre perpendicolare al lato e
 * che la linea non corra mai lungo il bordo del box.
 */
const PORT_STUB = 20;

/* Penalità per un percorso che appoggia/attraversa una card. */
const ROUTE_BLOCKED = 100_000;

/* Spazio minimo lasciato tra due card quando si "respingono". */
const COLLISION_GAP = 8;

/*
 * Spazio minimo lasciato tra una card e l'ingombro reale della barra
 * delle risorse (bug 1.2): una card può stare comunque accanto alla
 * palette, non deve solo evitare di finirci esattamente sotto.
 */
const PALETTE_GAP = 12;

/* Margine di sicurezza attorno a una card "ostacolo" nel routing frecce. */
const OBSTACLE_MARGIN = 8;

/*
 * Raggio (in px, coordinate superficie) entro cui una card viene
 * considerata un possibile ostacolo per il routing di un edge: evita di
 * testare l'intersezione contro OGNI nodo del canvas quando ce ne sono
 * molti, limitandosi a quelli realmente vicini al percorso diretto.
 */
const OBSTACLE_SEARCH_MARGIN = 160;

/*
 * Due card "transform" entrano in modalità combine quando il gap tra i
 * loro bounding box scende sotto questa soglia (in px, coordinate
 * superficie).
 */
const COMBINE_GAP = 24;

/*
 * Trascinando un nodo che fa parte di un gruppo, superata questa
 * distanza (in px) dalla sua posizione di partenza il nodo si stacca
 * dal gruppo invece di trascinarlo rigidamente con sé.
 */
const DETACH_THRESHOLD = 48;

/* Padding visivo del contenitore che racchiude un gruppo di card. */
const GROUP_PADDING = 14;

type Side =
  | "top"
  | "right"
  | "bottom"
  | "left";

type Point = {
  x: number;
  y: number;
};

type Anchor = Point & {
  side: Side;
};

type Pending = {
  node: string;
  port: string;
  x: number;
  y: number;
  targetNode: string | null;
} | null;

/** Snapshot di una card stazionaria catturato all'inizio di un drag. */
type BaseNode = Point &
  NodeSize & {
    id: string;
  };

type NodeGeometry = EtlNode & {
  width: number;
  height: number;
};

type Rect = {
  x: number;
  y: number;
  width: number;
  height: number;
};

type ContextMenuState = {
  open: boolean;
  x: number;
  y: number;
  nodeId: string | null;
};

const STATUS: Record<
  NodeStatus,
  {
    label: string;
    color: string;
  }
> = {
  ready: {
    label: "Ready",
    color: "var(--muted-foreground)",
  },
  running: {
    label: "Running",
    color: "var(--brand)",
  },
  succeeded: {
    label: "Succeeded",
    color: "var(--success)",
  },
  error: {
    label: "Error",
    color: "var(--destructive)",
  },
};

/* -------------------------------------------------------------------------- */
/*                               GEOMETRY                                     */
/* -------------------------------------------------------------------------- */

function getAnchor(
  node: NodeGeometry,
  side: Side,
): Anchor {
  const halfW = node.width / 2;
  const halfH = node.height / 2;

  switch (side) {
    case "top":
      return {
        side,
        x: node.x + halfW,
        y: node.y,
      };

    case "right":
      return {
        side,
        x: node.x + node.width,
        y: node.y + halfH,
      };

    case "bottom":
      return {
        side,
        x: node.x + halfW,
        y: node.y + node.height,
      };

    case "left":
      return {
        side,
        x: node.x,
        y: node.y + halfH,
      };
  }
}

/* -------------------------------------------------------------------------- */
/*                             COLLISION                                      */
/* -------------------------------------------------------------------------- */

function rectsOverlap(
  a: Rect,
  b: Rect,
  gap = COLLISION_GAP,
): boolean {
  return (
    a.x < b.x + b.width + gap &&
    a.x + a.width + gap > b.x &&
    a.y < b.y + b.height + gap &&
    a.y + a.height + gap > b.y
  );
}

/**
 * Spinge `moving` fuori da `obstacle` lungo l'asse di minima penetrazione.
 * Restituisce `null` se non c'è sovrapposizione (nessuno spostamento
 * necessario), altrimenti la nuova posizione di `moving`.
 *
 * A differenza della vecchia `avoidCollisions`, qui è sempre `moving` a
 * cedere il passo: la card trascinata (l'obstacle, quando la si chiama
 * durante il drag) resta libera di seguire il puntatore senza mai
 * fermarsi.
 */
function pushOutOfOverlap(
  moving: Rect,
  obstacle: Rect,
  gap = COLLISION_GAP,
): Point | null {
  if (
    !rectsOverlap(
      moving,
      obstacle,
      gap,
    )
  ) {
    return null;
  }

  const pushRight =
    obstacle.x +
    obstacle.width +
    gap -
    moving.x;

  const pushLeft =
    moving.x +
    moving.width +
    gap -
    obstacle.x;

  const pushDown =
    obstacle.y +
    obstacle.height +
    gap -
    moving.y;

  const pushUp =
    moving.y +
    moving.height +
    gap -
    obstacle.y;

  const minPush = Math.min(
    pushLeft,
    pushRight,
    pushUp,
    pushDown,
  );

  if (minPush === pushLeft) {
    return {
      x:
        obstacle.x -
        moving.width -
        gap,
      y: moving.y,
    };
  }

  if (minPush === pushRight) {
    return {
      x:
        obstacle.x +
        obstacle.width +
        gap,
      y: moving.y,
    };
  }

  if (minPush === pushUp) {
    return {
      x: moving.x,
      y:
        obstacle.y -
        moving.height -
        gap,
    };
  }

  return {
    x: moving.x,
    y:
      obstacle.y +
      obstacle.height +
      gap,
  };
}

function distance(
  a: Point,
  b: Point,
): number {
  return (
    Math.abs(a.x - b.x) +
    Math.abs(a.y - b.y)
  );
}

/** Il più piccolo rettangolo che racchiude tutti i `rects`. */
function unionRect(
  rects: Rect[],
): Rect {
  const first = rects[0];

  if (!first) {
    return {
      x: 0,
      y: 0,
      width: 0,
      height: 0,
    };
  }

  let minX = first.x;
  let minY = first.y;
  let maxX = first.x + first.width;
  let maxY = first.y + first.height;

  for (const rect of rects) {
    minX = Math.min(minX, rect.x);
    minY = Math.min(minY, rect.y);
    maxX = Math.max(
      maxX,
      rect.x + rect.width,
    );
    maxY = Math.max(
      maxY,
      rect.y + rect.height,
    );
  }

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}

/**
 * Tra i nodi `candidates` (già filtrati a card "transform" non
 * coinvolte nel drag in corso), quello il cui bounding box è a meno di
 * `gap` px da `movingRect` — cioè il candidato di "combine" più vicino
 * al gruppo che si sta trascinando. `null` se nessuno è abbastanza
 * vicino.
 */
function pickCombineCandidate(
  movingRect: Rect,
  candidates: NodeGeometry[],
  gap: number,
): NodeGeometry | null {
  let best: NodeGeometry | null =
    null;

  let bestDistance =
    Number.POSITIVE_INFINITY;

  const movingCenter = {
    x:
      movingRect.x +
      movingRect.width / 2,
    y:
      movingRect.y +
      movingRect.height / 2,
  };

  for (const candidate of candidates) {
    if (
      !rectsOverlap(
        movingRect,
        candidate,
        gap,
      )
    ) {
      continue;
    }

    const candidateDistance =
      distance(movingCenter, {
        x:
          candidate.x +
          candidate.width / 2,
        y:
          candidate.y +
          candidate.height / 2,
      });

    if (candidateDistance < bestDistance) {
      best = candidate;
      bestDistance = candidateDistance;
    }
  }

  return best;
}

function simplifyPath(
  points: Point[],
): Point[] {
  const result: Point[] = [];

  for (const point of points) {
    const previous =
      result[result.length - 1];

    if (
      previous &&
      previous.x === point.x &&
      previous.y === point.y
    ) {
      continue;
    }

    const beforePrevious =
      result[result.length - 2];

    if (
      beforePrevious &&
      previous &&
      beforePrevious.x ===
        previous.x &&
      previous.x === point.x
    ) {
      result[result.length - 1] =
        point;
      continue;
    }

    if (
      beforePrevious &&
      previous &&
      beforePrevious.y ===
        previous.y &&
      previous.y === point.y
    ) {
      result[result.length - 1] =
        point;
      continue;
    }

    result.push(point);
  }

  return result;
}

function pathFromPoints(
  points: Point[],
): string {
  return points
    .map(
      (point, index) =>
        index === 0
          ? `M ${point.x} ${point.y}`
          : `L ${point.x} ${point.y}`,
    )
    .join(" ");
}

/**
 * Stessa sequenza di waypoint di `pathFromPoints`, ma con i vertici
 * interni smussati (quadratic bezier) invece che ad angolo vivo: solo
 * resa visiva più morbida del path già calcolato, nessun ricalcolo del
 * routing/anchor. Primo e ultimo punto (gli anchor sui bordi delle
 * card) restano invariati.
 */
function smoothPathFromPoints(
  points: Point[],
  radius: number = EDGE_CORNER_RADIUS,
): string {
  const first = points[0];

  if (!first || points.length < 3) {
    return pathFromPoints(points);
  }

  let d = `M ${first.x} ${first.y}`;

  for (
    let i = 1;
    i < points.length - 1;
    i += 1
  ) {
    const prev = points[i - 1]!;
    const curr = points[i]!;
    const next = points[i + 1]!;

    const toPrev = {
      x: prev.x - curr.x,
      y: prev.y - curr.y,
    };

    const toNext = {
      x: next.x - curr.x,
      y: next.y - curr.y,
    };

    const lenPrev = Math.hypot(
      toPrev.x,
      toPrev.y,
    );

    const lenNext = Math.hypot(
      toNext.x,
      toNext.y,
    );

    if (
      lenPrev < 0.01 ||
      lenNext < 0.01
    ) {
      d += ` L ${curr.x} ${curr.y}`;
      continue;
    }

    const r = Math.min(
      radius,
      lenPrev / 2,
      lenNext / 2,
    );

    const enter = {
      x:
        curr.x +
        (toPrev.x / lenPrev) * r,
      y:
        curr.y +
        (toPrev.y / lenPrev) * r,
    };

    const exit = {
      x:
        curr.x +
        (toNext.x / lenNext) * r,
      y:
        curr.y +
        (toNext.y / lenNext) * r,
    };

    d += ` L ${enter.x} ${enter.y} Q ${curr.x} ${curr.y} ${exit.x} ${exit.y}`;
  }

  const last =
    points[points.length - 1]!;

  d += ` L ${last.x} ${last.y}`;

  return d;
}

/** Versore uscente, perpendicolare al lato. */
function sideNormal(
  side: Side,
): Point {
  switch (side) {
    case "top":
      return { x: 0, y: -1 };
    case "right":
      return { x: 1, y: 0 };
    case "bottom":
      return { x: 0, y: 1 };
    case "left":
      return { x: -1, y: 0 };
  }
}

/** Punto a `dist` dall'anchor, uscente perpendicolare al lato. */
function stubPoint(
  anchor: Anchor,
  dist: number,
): Point {
  const n = sideNormal(anchor.side);

  return {
    x: anchor.x + n.x * dist,
    y: anchor.y + n.y * dist,
  };
}

/** Il segmento [a,b] (ortogonale) interseca il rettangolo del nodo? */
function segmentHitsNode(
  a: Point,
  b: Point,
  node: NodeGeometry,
  pad: number,
): boolean {
  const left = node.x - pad;
  const right = node.x + node.width + pad;
  const top = node.y - pad;
  const bottom = node.y + node.height + pad;

  const minX = Math.min(a.x, b.x);
  const maxX = Math.max(a.x, b.x);
  const minY = Math.min(a.y, b.y);
  const maxY = Math.max(a.y, b.y);

  return (
    minX < right &&
    maxX > left &&
    minY < bottom &&
    maxY > top
  );
}

/**
 * Card diverse da `from`/`to` che si trovano abbastanza vicine al
 * rettangolo che racchiude i due nodi dell'edge da poter finire in
 * mezzo al percorso. Filtrare qui evita di testare l'intersezione
 * contro OGNI nodo del canvas quando i nodi sono molti.
 */
function nearbyObstacles(
  from: NodeGeometry,
  to: NodeGeometry,
  allNodes: NodeGeometry[],
  /*
   * Id extra da escludere dagli ostacoli oltre a from.id/to.id — serve
   * quando from/to sono in realtà il rettangolo sintetico di una
   * bubble (fase 2): gli id reali delle card membro vanno esclusi
   * esplicitamente, perché from.id/to.id sono un id sintetico
   * (`bubble:<groupId>`) che non combacia con nessuna card reale.
   */
  excludeIds?: ReadonlySet<string>,
): NodeGeometry[] {
  const minX =
    Math.min(from.x, to.x) -
    OBSTACLE_SEARCH_MARGIN;

  const maxX =
    Math.max(
      from.x + from.width,
      to.x + to.width,
    ) + OBSTACLE_SEARCH_MARGIN;

  const minY =
    Math.min(from.y, to.y) -
    OBSTACLE_SEARCH_MARGIN;

  const maxY =
    Math.max(
      from.y + from.height,
      to.y + to.height,
    ) + OBSTACLE_SEARCH_MARGIN;

  return allNodes.filter(
    (node) =>
      node.id !== from.id &&
      node.id !== to.id &&
      !excludeIds?.has(node.id) &&
      node.x < maxX &&
      node.x + node.width > minX &&
      node.y < maxY &&
      node.y + node.height > minY,
  );
}

/**
 * Tra gli `obstacles`, quello il cui bounding box interseca
 * l'ingombro rettangolare del segmento [a,b]: è la card che più
 * probabilmente sta bloccando il percorso diretto. In caso di più
 * candidati si sceglie quello con il centro più vicino al punto
 * medio del segmento.
 */
function pickBlockingObstacle(
  a: Point,
  b: Point,
  obstacles: NodeGeometry[],
): NodeGeometry | null {
  const minX = Math.min(a.x, b.x);
  const maxX = Math.max(a.x, b.x);
  const minY = Math.min(a.y, b.y);
  const maxY = Math.max(a.y, b.y);

  const overlapping = obstacles.filter(
    (node) =>
      node.x < maxX &&
      node.x + node.width > minX &&
      node.y < maxY &&
      node.y + node.height > minY,
  );

  if (overlapping.length === 0) {
    return null;
  }

  const midX = (a.x + b.x) / 2;
  const midY = (a.y + b.y) / 2;

  return overlapping.reduce(
    (closest, node) => {
      const nodeDist = distance(
        {
          x: node.x + node.width / 2,
          y: node.y + node.height / 2,
        },
        { x: midX, y: midY },
      );

      const closestDist = distance(
        {
          x:
            closest.x +
            closest.width / 2,
          y:
            closest.y +
            closest.height / 2,
        },
        { x: midX, y: midY },
      );

      return nodeDist < closestDist
        ? node
        : closest;
    },
  );
}

/**
 * Punti di deviazione (in stile "U") che portano il percorso attorno
 * al bounding box di `obstacle`, uscendo sopra/sotto/a-sinistra/a-
 * destra di esso. Non è un vero pathfinding: è un set aggiuntivo di
 * candidati manhattan usato solo quando le shape "dirette" finiscono
 * tutte per attraversare una card.
 */
function detourShapesAround(
  s: Point,
  e: Point,
  obstacle: NodeGeometry,
): Point[][] {
  const left =
    obstacle.x - OBSTACLE_MARGIN;

  const right =
    obstacle.x +
    obstacle.width +
    OBSTACLE_MARGIN;

  const top =
    obstacle.y - OBSTACLE_MARGIN;

  const bottom =
    obstacle.y +
    obstacle.height +
    OBSTACLE_MARGIN;

  return [
    /* Sopra l'ostacolo. */
    [
      { x: s.x, y: top },
      { x: e.x, y: top },
    ],
    /* Sotto l'ostacolo. */
    [
      { x: s.x, y: bottom },
      { x: e.x, y: bottom },
    ],
    /* A sinistra dell'ostacolo. */
    [
      { x: left, y: s.y },
      { x: left, y: e.y },
    ],
    /* A destra dell'ostacolo. */
    [
      { x: right, y: s.y },
      { x: right, y: e.y },
    ],
  ];
}

/**
 * Calcola un percorso ortogonale tra due anchor.
 *
 * La freccia esce sempre PERPENDICOLARE dal lato di partenza e arriva
 * PERPENDICOLARE al lato di destinazione: i primi/ultimi `PORT_STUB` px
 * sono un tratto dritto obbligato, così la linea non si appoggia mai sul
 * bordo del box. Nessuna curva, nessun arrowhead: solo segmenti H/V.
 *
 * `obstacles` sono le altre card (né `from` né `to`) da evitare: se
 * tutte le shape "dirette" finiscono per attraversarne una, si prova
 * anche un set di percorsi che deviano attorno all'ostacolo più
 * vicino al tragitto.
 */
function routeCandidate(
  from: NodeGeometry,
  to: NodeGeometry,
  fromSide: Side,
  toSide: Side,
  obstacles: NodeGeometry[] = [],
) {
  const start = getAnchor(
    from,
    fromSide,
  );

  const end = getAnchor(
    to,
    toSide,
  );

  const s = stubPoint(
    start,
    PORT_STUB,
  );

  const e = stubPoint(
    end,
    PORT_STUB,
  );

  const middleX = Math.round(
    (s.x + e.x) / 2,
  );

  const middleY = Math.round(
    (s.y + e.y) / 2,
  );

  const shapes: Point[][] = [
    [{ x: e.x, y: s.y }],
    [{ x: s.x, y: e.y }],
    [
      { x: middleX, y: s.y },
      { x: middleX, y: e.y },
    ],
    [
      { x: s.x, y: middleY },
      { x: e.x, y: middleY },
    ],
  ];

  /*
   * Se il lato scelto non "guarda" verso l'altro nodo, la freccia
   * dovrebbe girare attorno alla card: percorso da scartare.
   */
  const fromCenter = {
    x: from.x + from.width / 2,
    y: from.y + from.height / 2,
  };

  const toCenter = {
    x: to.x + to.width / 2,
    y: to.y + to.height / 2,
  };

  const fromN = sideNormal(fromSide);
  const toN = sideNormal(toSide);

  const fromFacesTarget =
    fromN.x *
      (toCenter.x - fromCenter.x) +
    fromN.y *
      (toCenter.y - fromCenter.y);

  const toFacesSource =
    toN.x *
      (fromCenter.x - toCenter.x) +
    toN.y *
      (fromCenter.y - toCenter.y);

  const orientationPenalty =
    (fromFacesTarget < 0
      ? ROUTE_BLOCKED
      : 0) +
    (toFacesSource < 0
      ? ROUTE_BLOCKED
      : 0);

  const evaluate = (
    rawPoints: Point[],
  ) => {
    const points =
      simplifyPath(rawPoints);

    let length = 0;
    let blocked = 0;
    let obstacleHits = 0;

    for (
      let i = 1;
      i < points.length;
      i += 1
    ) {
      const previousPoint = points[i - 1];
      const currentPoint = points[i];

      if (!previousPoint || !currentPoint) {
        continue;
      }

      length += distance(
        previousPoint,
        currentPoint,
      );

      /*
       * Un segmento non deve attraversare una card. Si escludono i
       * due tratti-stub, che toccano per forza il proprio nodo.
       */
      const isFirst = i === 1;
      const isLast =
        i === points.length - 1;

      if (
        !isFirst &&
        segmentHitsNode(
          previousPoint,
          currentPoint,
          from,
          2,
        )
      ) {
        blocked += ROUTE_BLOCKED;
      }

      if (
        !isLast &&
        segmentHitsNode(
          previousPoint,
          currentPoint,
          to,
          2,
        )
      ) {
        blocked += ROUTE_BLOCKED;
      }

      /*
       * Un segmento non deve nemmeno attraversare (con un margine di
       * sicurezza) una card terza che si trova in mezzo al percorso.
       */
      for (const obstacle of obstacles) {
        if (
          segmentHitsNode(
            previousPoint,
            currentPoint,
            obstacle,
            OBSTACLE_MARGIN,
          )
        ) {
          obstacleHits += 1;
        }
      }
    }

    const bends =
      Math.max(
        0,
        points.length - 2,
      );

    return {
      points,
      obstacleHits,
      score:
        length +
        bends * ROUTE_GAP +
        blocked +
        obstacleHits *
          ROUTE_BLOCKED +
        orientationPenalty,
    };
  };

  let scored = shapes
    .map((mid) => [
      start,
      s,
      ...mid,
      e,
      end,
    ])
    .map(evaluate);

  const anyObstacleFree =
    scored.some(
      (candidate) =>
        candidate.obstacleHits === 0,
    );

  if (
    !anyObstacleFree &&
    obstacles.length > 0
  ) {
    const blocking =
      pickBlockingObstacle(
        s,
        e,
        obstacles,
      ) ??
      pickBlockingObstacle(
        start,
        end,
        obstacles,
      );

    if (blocking) {
      const detourCandidates =
        detourShapesAround(
          s,
          e,
          blocking,
        ).map((mid) => [
          start,
          s,
          ...mid,
          e,
          end,
        ]);

      scored = scored.concat(
        detourCandidates.map(
          evaluate,
        ),
      );
    }
  }

  scored.sort(
    (a, b) =>
      a.score - b.score,
  );

  const best = scored[0];

  if (!best) {
    return {
      fromSide,
      toSide,
      points: [
        getAnchor(from, fromSide),
        getAnchor(to, toSide),
      ],
      score: Number.POSITIVE_INFINITY,
    };
  }

  return {
    fromSide,
    toSide,
    points: best.points,
    score: best.score,
  };
}

function getBestRoute(
  from: NodeGeometry,
  to: NodeGeometry,
  allNodes: NodeGeometry[] = [],
  excludeIds?: ReadonlySet<string>,
) {
  const sides: Side[] = [
    "top",
    "right",
    "bottom",
    "left",
  ];

  const obstacles = nearbyObstacles(
    from,
    to,
    allNodes,
    excludeIds,
  );

  const routes: Array<
    ReturnType<typeof routeCandidate>
  > = [];

  for (const fromSide of sides) {
    for (const toSide of sides) {
      routes.push(
        routeCandidate(
          from,
          to,
          fromSide,
          toSide,
          obstacles,
        ),
      );
    }
  }

  routes.sort(
    (a, b) =>
      a.score - b.score,
  );

  return routes[0];
}

/**
 * Adatta una card membro + la geometria della sua bubble (fase 2) in
 * un NodeGeometry sintetico che rappresenta l'intera bubble ai fini
 * del routing: stessa forma di un nodo singolo, ma x/y/width/height
 * presi dal bounding box della bubble, espanso dello stesso
 * GROUP_PADDING con cui è disegnato il contenitore — così la freccia
 * tocca visivamente il bordo disegnato, non il bounding box "nudo"
 * dei soli membri. I campi non geometrici (type/title/config/status)
 * sono presi dal membro passato: servono solo a soddisfare il tipo
 * NodeGeometry, getAnchor/getBestRoute leggono solo
 * x/y/width/height/id.
 */
function bubbleNodeGeometry(
  member: NodeGeometry,
  bubble: BubbleGeometry,
): NodeGeometry {
  return {
    ...member,
    id: `bubble:${bubble.groupId}`,
    x:
      bubble.rect.x -
      GROUP_PADDING,
    y:
      bubble.rect.y -
      GROUP_PADDING,
    width:
      bubble.rect.width +
      GROUP_PADDING * 2,
    height:
      bubble.rect.height +
      GROUP_PADDING * 2,
  };
}

/**
 * Punto visuale mostrato quando il cursore
 * entra in una card durante il linking.
 *
 * Preferenza: angolo alto a destra.
 * Se troppo vicino al bordo destro:
 * angolo alto a sinistra.
 */
function getDropIndicatorPoint(
  node: NodeGeometry,
  surfaceW: number,
  surfaceH: number,
): Point {
  const margin = 12;

  let x =
    node.x +
    node.width -
    margin;

  let y =
    node.y + margin;

  if (
    x + 7 >
    surfaceW
  ) {
    x =
      node.x + margin;
  }

  if (
    y - 7 <
    0
  ) {
    y =
      node.y +
      node.height -
      margin;
  }

  if (
    x - 7 <
    0
  ) {
    x =
      node.x +
      node.width -
      margin;
  }

  if (
    y + 7 >
    surfaceH
  ) {
    y =
      node.y + margin;
  }

  return {
    x,
    y,
  };
}


/**
 * Distribuzione dei port su uno specifico lato.
 */
function portOffset(
  index: number,
  count: number,
  size: number,
): number {
  if (count <= 1) {
    return size / 2;
  }

  const padding = 18;

  return (
    padding +
    index *
      ((size -
        padding * 2) /
        (count - 1))
  );
}

/* -------------------------------------------------------------------------- */
/*                         CONTEXT MENU                                       */
/* -------------------------------------------------------------------------- */

function CanvasContextMenu({
  state,
  boundaryRef,
  grid,
  onClose,
  onAddDataset,
  onFitView,
  onToggleGrid,
  onAutoLayout,
  onDuplicateNode,
  onRemoveNode,
  onUnlinkNode,
}: {
  state: ContextMenuState;
  boundaryRef: React.RefObject<HTMLDivElement | null>;
  grid: boolean;
  onClose: () => void;
  onAddDataset: () => void;
  onFitView: () => void;
  onToggleGrid: () => void;
  onAutoLayout: () => void;
  onDuplicateNode: (
    id: string,
  ) => void;
  onRemoveNode: (
    id: string,
  ) => void;
  onUnlinkNode: (
    id: string,
  ) => void;
}) {
  const menuRef =
    useRef<HTMLDivElement>(null);

  const [
    position,
    setPosition,
  ] = useState<Point>({
    x: state.x,
    y: state.y,
  });

  useEffect(() => {
    if (!state.open) {
      return;
    }

    const handleOutsidePointer =
      (event: PointerEvent) => {
        if (
          !menuRef.current?.contains(
            event.target as Node,
          )
        ) {
          onClose();
        }
      };

    const handleEscape = (
      event: KeyboardEvent,
    ) => {
      if (
        event.key === "Escape"
      ) {
        onClose();
      }
    };

    document.addEventListener(
      "pointerdown",
      handleOutsidePointer,
    );

    document.addEventListener(
      "keydown",
      handleEscape,
    );

    return () => {
      document.removeEventListener(
        "pointerdown",
        handleOutsidePointer,
      );

      document.removeEventListener(
        "keydown",
        handleEscape,
      );
    };
  }, [
    state.open,
    onClose,
  ]);

  useEffect(() => {
    if (!state.open) {
      return;
    }

    const boundary =
      boundaryRef.current;

    const menu =
      menuRef.current;

    if (!boundary || !menu) {
      return;
    }

    const boundaryRect =
      boundary.getBoundingClientRect();

    const menuRect =
      menu.getBoundingClientRect();

    const margin = 8;

    const maxX =
      Math.max(
        boundaryRect.left + margin,
        boundaryRect.right -
          menuRect.width -
          margin,
      );

    const maxY =
      Math.max(
        boundaryRect.top + margin,
        boundaryRect.bottom -
          menuRect.height -
          margin,
      );

    setPosition({
      x: Math.min(
        Math.max(
          state.x,
          boundaryRect.left +
            margin,
        ),
        maxX,
      ),
      y: Math.min(
        Math.max(
          state.y,
          boundaryRect.top +
            margin,
        ),
        maxY,
      ),
    });
  }, [
    state.open,
    state.x,
    state.y,
    boundaryRef,
  ]);

  if (
    !state.open ||
    typeof document ===
      "undefined"
  ) {
    return null;
  }

  return createPortal(
    <div
      ref={menuRef}
      role="menu"
      aria-label="Menu contestuale IsA"
      className="fixed z-100 w-56 overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl"
      style={{
        left: position.x,
        top: position.y,
      }}
      onContextMenu={(event) =>
        event.preventDefault()
      }
      onPointerDown={(event) =>
        event.stopPropagation()
      }
    >
      {state.nodeId ? (
        <>
          <IsaMenuItem
            Icon={Copy}
            label="Duplica nodo"
            onClick={() => {
              onDuplicateNode(
                state.nodeId!,
              );
              onClose();
            }}
          />

          <IsaMenuItem
            Icon={Unlink}
            label="Scollega input"
            onClick={() => {
              onUnlinkNode(
                state.nodeId!,
              );
              onClose();
            }}
          />

          <IsaMenuItem
            Icon={Trash2}
            label="Elimina nodo"
            danger
            onClick={() => {
              onRemoveNode(
                state.nodeId!,
              );
              onClose();
            }}
          />

          <span className="my-1 block h-px bg-border" />
        </>
      ) : null}

      <IsaMenuItem
        Icon={Plus}
        label="Aggiungi dataset"
        onClick={() => {
          onAddDataset();
          onClose();
        }}
      />

      <IsaMenuItem
        Icon={LayoutGrid}
        label="Disponi automaticamente"
        onClick={() => {
          onAutoLayout();
          onClose();
        }}
      />

      <IsaMenuItem
        Icon={Maximize2}
        label="Adatta vista"
        onClick={() => {
          onFitView();
          onClose();
        }}
      />

      <IsaMenuItem
        Icon={Grid3x3}
        label={
          grid
            ? "Nascondi griglia"
            : "Mostra griglia"
        }
        onClick={() => {
          onToggleGrid();
          onClose();
        }}
      />
    </div>,
    document.body,
  );
}

/* -------------------------------------------------------------------------- */
/*                         MAIN COMPONENT                                     */
/* -------------------------------------------------------------------------- */

export function WorkflowCanvas({
  workflow,
  selectedId,
  onSelect,
  onMove,
  onAddAt,
  onConnect,
  onRemoveNode,
  onDuplicateNode,
  onGroupNodes,
  onUngroupNode,
  onRemoveEdge,
  onAddDataset,
  onLayoutChange,
  onUpdateNodeConfig,
}: {
  workflow: EtlWorkflow;
  selectedId: string | null;
  onSelect: (
    id: string | null,
  ) => void;
  onMove: (
    id: string,
    x: number,
    y: number,
    commit?: boolean,
  ) => void;
  /**
   * Ritorna l'id del nodo creato (o `undefined` se il tipo non esiste),
   * così il canvas può marcarlo "in attesa di essere raccolto" quando
   * la creazione arriva da un doppio click sulla palette (PARTE B).
   * `manual` (bug 1.3): quando true, il workflow passa a
   * `layout: "manual"` così l'auto-layout non ricolloca la card appena
   * rilasciata — usato dal drag-and-drop esplicito dalla palette, non
   * dal doppio click (che mantiene il comportamento "l'auto-layout
   * vince sempre").
   */
  onAddAt: (
    type: string,
    x: number,
    y: number,
    manual?: boolean,
  ) => string | undefined;
  onConnect: (
    fromNode: string,
    fromPort: string,
    toNode: string,
    toPort: string,
  ) => void;
  onRemoveNode: (
    id: string,
  ) => void;
  onDuplicateNode: (
    id: string,
  ) => void;
  onGroupNodes: (
    ids: string[],
  ) => void;
  onUngroupNode: (
    id: string,
  ) => void;
  onRemoveEdge: (
    id: string,
  ) => void;
  onAddDataset: () => void;
  onLayoutChange: (
    mode: LayoutMode,
  ) => void;
  /** Fase 4: scrive nel config del nodo dai pannelli impostazioni (merge, non replace). */
  onUpdateNodeConfig: (
    id: string,
    patch: Record<string, string>,
  ) => void;
}) {
  const boxRef =
    useRef<HTMLDivElement>(null);

  const surfaceRef =
    useRef<HTMLDivElement>(null);

  const [
    pending,
    setPending,
  ] = useState<Pending>(null);

  const [
    contextMenu,
    setContextMenu,
  ] =
    useState<ContextMenuState>({
      open: false,
      x: 0,
      y: 0,
      nodeId: null,
    });

  const [
    zoom,
    setZoom,
  ] = useState(1);

  const [
    hoverEdge,
    setHoverEdge,
  ] = useState<string | null>(
    null,
  );

  const [
    grid,
    setGrid,
  ] = useState(true);

  const [
    display,
    setDisplay,
  ] =
    useState<EtlDisplaySettings>(
      DEFAULT_DISPLAY,
    );

  const [
    box,
    setBox,
  ] = useState({
    w: 1200,
    h: 720,
  });

  const [
    paletteDock,
    setPaletteDock,
  ] = useState<Dock>("top");

  /*
   * Id dei nodi creati con doppio click dalla palette (PARTE B) e non
   * ancora "raccolti" con il primo pointerdown sulla card — stato
   * puramente visivo, non persistito nel workflow: sparisce alla prima
   * interazione, quindi non ha senso sopravvivere a reload/localStorage.
   * Nome distinto da `pending` (sopra, stato di un collegamento in
   * corso) per evitare ambiguità: concetti diversi.
   */
  const [
    freshNodeIds,
    setFreshNodeIds,
  ] = useState<Set<string>>(
    () => new Set(),
  );

  const paletteRef =
    useRef<HTMLDivElement>(
      null,
    );

  const [
    paletteBox,
    setPaletteBox,
  ] = useState({
    w: 0,
    h: 0,
  });

  const dragRef =
    useRef<{
      id: string;
      pointerId: number;
      offsetX: number;
      offsetY: number;
      width: number;
      height: number;
      element: HTMLDivElement | null;
      /* Posizione della card trascinata all'inizio del drag: serve a
       * misurare quanto si è allontanata, per la soglia di distacco
       * dal gruppo. */
      primaryStart: Point;
      /* groupId del nodo trascinato all'inizio del drag (se ne aveva
       * uno). Non cambia durante il drag: il distacco è per-sessione. */
      groupId: string | undefined;
      /* Altri membri dello stesso gruppo, con offset RIGIDO rispetto
       * alla card trascinata: finché il gruppo non si stacca, si
       * spostano insieme ad essa di questo stesso delta. */
      memberOffsets: Map<
        string,
        {
          dx: number;
          dy: number;
          width: number;
          height: number;
        }
      >;
      /* Una volta staccato dal gruppo (soglia superata), resta
       * staccato per il resto di questa sessione di drag. */
      detached: boolean;
      /* Snapshot di TUTTE le altre card (inclusi gli altri membri del
       * gruppo), preso all'inizio del drag: è la base immutabile da
       * cui ricalcolare gli spostamenti anti-sovrapposizione a ogni
       * frame (così tornano al proprio posto non appena non servono
       * più). Finché un membro del gruppo si muove rigidamente con la
       * card trascinata viene escluso dal calcolo anti-sovrapposizione;
       * torna a essere un "ostacolo" normale appena si stacca.
       */
      basePositions: BaseNode[];
      /* Ultima posizione che QUESTO drag ha applicato a ciascuna delle
       * altre card (sia per l'anti-sovrapposizione sia per il
       * movimento rigido del gruppo): serve solo a capire, frame per
       * frame, quali onMove vanno effettivamente emessi. */
      livePositions: Map<string, Point>;
    } | null>(null);

  const [
    combinePreview,
    setCombinePreview,
  ] = useState<{
    movingIds: string[];
    targetId: string;
  } | null>(null);

  useEffect(() => {
    const el =
      paletteRef.current;

    if (!el) {
      return;
    }

    const ro =
      new ResizeObserver(() =>
        setPaletteBox({
          w: el.offsetWidth,
          h: el.offsetHeight,
        }),
      );

    ro.observe(el);

    setPaletteBox({
      w: el.offsetWidth,
      h: el.offsetHeight,
    });

    return () =>
      ro.disconnect();
  }, []);

  useEffect(() => {
    const el =
      boxRef.current;

    if (!el) {
      return;
    }

    const ro =
      new ResizeObserver(() =>
        setBox({
          w: el.clientWidth,
          h: el.clientHeight,
        }),
      );

    ro.observe(el);

    setBox({
      w: el.clientWidth,
      h: el.clientHeight,
    });

    return () =>
      ro.disconnect();
  }, []);

  const surfaceW =
    Math.max(
      320,
      box.w / zoom,
    );

  const surfaceH =
    Math.max(
      280,
      box.h / zoom,
    );

  /*
   * Ingombro REALE della palette, in coordinate superficie (bug 1.2:
   * prima si riservava una banda che attraversava tutto il lato di
   * dock, anche dove la palette — centrata sul lato — non c'è
   * visivamente). La palette è renderizzata fuori da `surfaceRef` (non
   * scalata dallo zoom del canvas) ma ancorata/centrata sul bordo
   * `paletteDock` dello stesso `<section>`: la sua posizione in
   * coordinate superficie si può quindi derivare analiticamente dalla
   * stessa regola di centratura usata da `dockPosition` più sotto,
   * senza dover leggere una getBoundingClientRect ad ogni render.
   */
  const paletteRect = useMemo((): Rect => {
    const w = paletteBox.w / zoom;
    const h = paletteBox.h / zoom;

    switch (paletteDock) {
      case "left":
        return {
          x: 0,
          y: (surfaceH - h) / 2,
          width: w,
          height: h,
        };
      case "right":
        return {
          x: Math.max(0, surfaceW - w),
          y: (surfaceH - h) / 2,
          width: w,
          height: h,
        };
      case "bottom":
        return {
          x: (surfaceW - w) / 2,
          y: Math.max(0, surfaceH - h),
          width: w,
          height: h,
        };
      case "top":
      default:
        return {
          x: (surfaceW - w) / 2,
          y: 0,
          width: w,
          height: h,
        };
    }
  }, [
    paletteDock,
    paletteBox.w,
    paletteBox.h,
    zoom,
    surfaceW,
    surfaceH,
  ]);

  /*
   * Posizione "reale" di una card: un solo punto di verità usato sia
   * per il rendering della card sia per gli endpoint delle frecce e per
   * l'hit-test dei collegamenti. Clampa dentro la superficie e, se la
   * card finisce per sovrapporsi all'ingombro REALE della palette (non
   * più una banda a tutta larghezza/altezza, bug 1.2), la spinge fuori
   * lungo l'asse di minima penetrazione — così lo spazio libero
   * accanto a una palette centrata resta utilizzabile.
   */
  const placeNode = useCallback(
    (
      x: number,
      y: number,
      width: number,
      height: number,
    ): Point => {
      const clamp = (px: number, py: number): Point => ({
        x: Math.min(
          Math.max(0, px),
          Math.max(0, surfaceW - width),
        ),
        y: Math.min(
          Math.max(0, py),
          Math.max(0, surfaceH - height),
        ),
      });

      const base = clamp(x, y);

      const pushed = pushOutOfOverlap(
        { x: base.x, y: base.y, width, height },
        paletteRect,
        PALETTE_GAP,
      );

      return pushed ? clamp(pushed.x, pushed.y) : base;
    },
    [
      surfaceW,
      surfaceH,
      paletteRect,
    ],
  );

  const toLocal = useCallback(
    (
      clientX: number,
      clientY: number,
    ): Point => {
      const rect =
        surfaceRef.current?.getBoundingClientRect();

      return {
        x:
          (clientX -
            (rect?.left ??
              0)) /
          zoom,
        y:
          (clientY -
            (rect?.top ??
              0)) /
          zoom,
      };
    },
    [zoom],
  );

  /*
   * PARTE B, doppio click dalla palette: crea il nodo nell'angolo
   * visibile del canvas più vicino al puntatore, invece di un angolo
   * fisso. La palette non conosce zoom/pan del canvas (solo questo
   * componente ha `toLocal`), quindi riceve le coordinate schermo del
   * doppio click e fa qui tutta la conversione.
   */
  const handlePaletteDoubleClick =
    useCallback(
      (
        type: string,
        clientX: number,
        clientY: number,
      ) => {
        const rect =
          boxRef.current?.getBoundingClientRect();

        if (!rect) {
          return;
        }

        const corners: Point[] = [
          {
            x: rect.left,
            y: rect.top,
          },
          {
            x: rect.right,
            y: rect.top,
          },
          {
            x: rect.left,
            y: rect.bottom,
          },
          {
            x: rect.right,
            y: rect.bottom,
          },
        ];

        let nearest = corners[0]!;
        let bestDistance =
          Infinity;

        for (const corner of corners) {
          const distance =
            Math.hypot(
              corner.x - clientX,
              corner.y - clientY,
            );

          if (
            distance <
            bestDistance
          ) {
            bestDistance =
              distance;
            nearest = corner;
          }
        }

        const insetX =
          nearest.x === rect.left
            ? ROUTE_GAP
            : -ROUTE_GAP;

        const insetY =
          nearest.y === rect.top
            ? ROUTE_GAP
            : -ROUTE_GAP;

        const point = toLocal(
          nearest.x + insetX,
          nearest.y + insetY,
        );

        const dropped = placeNode(
          point.x - NODE_W / 2,
          point.y - NODE_H / 2,
          NODE_W,
          NODE_H,
        );

        const id = onAddAt(
          type,
          dropped.x,
          dropped.y,
        );

        if (id) {
          setFreshNodeIds(
            (current) => {
              const next =
                new Set(current);
              next.add(id);
              return next;
            },
          );
        }
      },
      [
        onAddAt,
        placeNode,
        toLocal,
      ],
    );

  /*
   * Bug 1.1: il drag da tool-palette.tsx al canvas usava HTML5 Drag &
   * Drop (`draggable` + `dataTransfer`), un sistema di eventi separato
   * e in conflitto con i Pointer Events usati ovunque altro sul
   * canvas (drag delle card, drag della palette stessa, link tra
   * porte). La palette ora fa un drag basato su Pointer Events
   * identico agli altri (ghost che segue il puntatore via portal, vedi
   * ToolPalette) e chiama questo handler al rilascio, con le
   * coordinate SCHERMO del punto di drop: qui avviene tutta la
   * conversione in coordinate canvas, esattamente come per il doppio
   * click sopra — ma il punto di drop è quello REALE sotto il
   * puntatore, non l'angolo più vicino.
   */
  const handlePaletteDrop =
    useCallback(
      (
        type: string,
        clientX: number,
        clientY: number,
      ) => {
        const point = toLocal(
          clientX,
          clientY,
        );

        const dropped = placeNode(
          point.x - NODE_W / 2,
          point.y - NODE_H / 2,
          NODE_W,
          NODE_H,
        );

        /*
         * Bug 1.3: a differenza del doppio click, un drag esplicito è
         * un'intenzione di posizionamento manuale — il nodo deve
         * restare dove è stato rilasciato anche se il workflow è in
         * layout "auto".
         */
        onAddAt(
          type,
          dropped.x,
          dropped.y,
          true,
        );
      },
      [
        onAddAt,
        placeNode,
        toLocal,
      ],
    );

  const nodeSizes =
    useMemo(
      () => {
        const sizes: Record<
          string,
          NodeSize
        > = {};

        for (
          const node of workflow.nodes
        ) {
          sizes[node.id] =
            estimateNodeSize(
              node,
              workflow,
              display,
            );
        }

        return sizes;
      },
      [
        workflow,
        display,
      ],
    );

  const getSize =
    useCallback(
      (id: string): NodeSize =>
        nodeSizes[id] ?? {
          width: NODE_W,
          height: MIN_NODE_HEIGHT,
        },
      [nodeSizes],
    );

  /*
   * Geometria corrente delle card. Ricalcolata a ogni render dalle
   * posizioni CORRENTI di workflow.nodes: è l'unica fonte di verità sia
   * per il rendering delle card sia per il routing delle frecce, quindi
   * le frecce seguono sempre la posizione reale (clampata) del nodo.
   */
  const visibleNodes =
    useMemo(
      () =>
        workflow.nodes.map(
          (node): NodeGeometry => {
            const {
              width,
              height,
            } = getSize(node.id);

            const { x, y } =
              placeNode(
                node.x,
                node.y,
                width,
                height,
              );

            return {
              ...node,
              width,
              height,
              x,
              y,
            };
          },
        ),
      [
        workflow.nodes,
        getSize,
        placeNode,
      ],
    );

  /*
   * Specchio della geometria in un ref, sempre aggiornato al render più
   * recente. I listener globali di startLink (pointermove/pointerup)
   * sopravvivono ai render: devono leggere QUI, non da un array
   * catturato nella closure alla creazione del listener, altrimenti i
   * nodi creati dopo non risultano agganciabili.
   */
  const nodesRef =
    useRef<NodeGeometry[]>(
      visibleNodes,
    );

  nodesRef.current = visibleNodes;

  const nodeById = useCallback(
    (id: string) =>
      visibleNodes.find(
        (node) =>
          node.id === id,
      ),
    [visibleNodes],
  );

  const liveNodeById = useCallback(
    (id: string) =>
      nodesRef.current.find(
        (node) =>
          node.id === id,
      ),
    [],
  );

  /**
   * Dato il rettangolo della card trascinata e uno snapshot delle altre
   * card (preso all'inizio del drag), calcola dove ognuna di esse deve
   * spostarsi in questo istante per non sovrapporsi né alla card
   * trascinata né, a cascata, alle altre card già spostate.
   *
   * Ricalcolando sempre a partire dallo snapshot iniziale (non dalla
   * posizione del frame precedente) il risultato è una funzione pura
   * della posizione corrente del puntatore: le card spostate tornano
   * esattamente al loro posto originale non appena smettono di essere
   * in collisione, senza accumulare deriva frame dopo frame.
   */
  const resolveDisplacedPositions =
    useCallback(
      (
        draggedRect: Rect,
        baseNodes: BaseNode[],
      ): Map<string, Point> => {
        const positions = new Map<
          string,
          Point
        >();

        for (const n of baseNodes) {
          positions.set(n.id, {
            x: n.x,
            y: n.y,
          });
        }

        /* Passo 1: spinge fuori dalla card trascinata. */
        for (const n of baseNodes) {
          const pos =
            positions.get(n.id)!;

          const rect: Rect = {
            x: pos.x,
            y: pos.y,
            width: n.width,
            height: n.height,
          };

          const pushed =
            pushOutOfOverlap(
              rect,
              draggedRect,
            );

          if (pushed) {
            positions.set(
              n.id,
              placeNode(
                pushed.x,
                pushed.y,
                n.width,
                n.height,
              ),
            );
          }
        }

        /*
         * Passo 2: propaga a catena le collisioni che restano tra le
         * card stazionarie spostate. Un numero limitato di iterazioni
         * (al più una per nodo) evita loop infiniti: non è una vera
         * simulazione fisica, ma risolve il caso comune di più card
         * spinte in cascata.
         */
        for (
          let pass = 0;
          pass < baseNodes.length;
          pass += 1
        ) {
          let changed = false;

          for (const a of baseNodes) {
            const aPos =
              positions.get(a.id)!;

            const aRect: Rect = {
              x: aPos.x,
              y: aPos.y,
              width: a.width,
              height: a.height,
            };

            for (const b of baseNodes) {
              if (a.id === b.id) {
                continue;
              }

              const bPos =
                positions.get(
                  b.id,
                )!;

              const bRect: Rect = {
                x: bPos.x,
                y: bPos.y,
                width: b.width,
                height: b.height,
              };

              const pushed =
                pushOutOfOverlap(
                  bRect,
                  aRect,
                );

              if (!pushed) {
                continue;
              }

              const clamped =
                placeNode(
                  pushed.x,
                  pushed.y,
                  b.width,
                  b.height,
                );

              if (
                clamped.x !==
                  bPos.x ||
                clamped.y !== bPos.y
              ) {
                positions.set(
                  b.id,
                  clamped,
                );

                changed = true;
              }
            }
          }

          if (!changed) {
            break;
          }
        }

        return positions;
      },
      [placeNode],
    );

  /* ---------------------------------------------------------------------- */
  /*                              FIT VIEW                                   */
  /* ---------------------------------------------------------------------- */

  const fitView = useCallback(
    () => {
      if (
        workflow.nodes.length ===
        0
      ) {
        setZoom(1);
        return;
      }

      const maxX =
        Math.max(
          ...visibleNodes.map(
            (node) =>
              node.x +
              node.width,
          ),
        ) + 40;

      const maxY =
        Math.max(
          ...visibleNodes.map(
            (node) =>
              node.y +
              node.height,
          ),
        ) + 40;

      setZoom(
        Math.min(
          1.4,
          Math.max(
            0.4,
            Math.min(
              box.w / maxX,
              box.h / maxY,
            ),
          ),
        ),
      );
    },
    [
      workflow.nodes.length,
      visibleNodes,
      box.w,
      box.h,
    ],
  );

  /* ---------------------------------------------------------------------- */
  /*                              NODE DRAG                                  */
  /* ---------------------------------------------------------------------- */

  type DragState = NonNullable<
    (typeof dragRef)["current"]
  >;

  /**
   * Un frame di drag: dove va la card trascinata, quali altri membri
   * del gruppo la seguono ancora rigidamente, come si risolvono le
   * collisioni anti-sovrapposizione contro tutte le altre card (i
   * membri ancora rigidi NON sono ostacoli), e se c'è un candidato
   * "combine" abbastanza vicino da mostrare/confermare.
   *
   * Muta `drag.detached` quando la soglia di distacco viene superata:
   * una volta staccato, resta staccato per il resto del drag.
   */
  const computeDragFrame = useCallback(
    (
      drag: DragState,
      point: Point,
    ) => {
      /*
       * La card trascinata segue sempre liberamente il puntatore:
       * nessun anti-sovrapposizione applicato a lei, solo il clamp ai
       * bordi del canvas già gestito da placeNode.
       */
      const primaryDesired = placeNode(
        point.x - drag.offsetX,
        point.y - drag.offsetY,
        drag.width,
        drag.height,
      );

      if (
        !drag.detached &&
        drag.groupId &&
        Math.hypot(
          primaryDesired.x -
            drag.primaryStart.x,
          primaryDesired.y -
            drag.primaryStart.y,
        ) > DETACH_THRESHOLD
      ) {
        drag.detached = true;
      }

      const rigidOtherIds =
        drag.groupId &&
        !drag.detached
          ? Array.from(
              drag.memberOffsets.keys(),
            )
          : [];

      const movingRects: Rect[] = [
        {
          x: primaryDesired.x,
          y: primaryDesired.y,
          width: drag.width,
          height: drag.height,
        },
        ...rigidOtherIds.map(
          (id) => {
            const off =
              drag.memberOffsets.get(
                id,
              )!;

            return {
              x:
                primaryDesired.x +
                off.dx,
              y:
                primaryDesired.y +
                off.dy,
              width: off.width,
              height: off.height,
            };
          },
        ),
      ];

      const draggedUnion =
        unionRect(movingRects);

      const rigidSet = new Set(
        rigidOtherIds,
      );

      const obstaclesForPush =
        drag.basePositions.filter(
          (n) =>
            !rigidSet.has(n.id),
        );

      const resolvedObstacles =
        obstaclesForPush.length > 0
          ? resolveDisplacedPositions(
              draggedUnion,
              obstaclesForPush,
            )
          : new Map<string, Point>();

      /*
       * Candidato "combine": solo tra card non-sources ("transform"),
       * e solo tra quelle NON già in movimento rigido con questa.
       */
      const primaryNode = nodeById(
        drag.id,
      );

      const primaryDef = primaryNode
        ? nodeDef(primaryNode.type)
        : undefined;

      const isTransformLike =
        !!primaryDef &&
        primaryDef.category !==
          "sources";

      let combineTarget: NodeGeometry | null =
        null;

      if (isTransformLike) {
        const movingIdsNow = new Set([
          drag.id,
          ...rigidOtherIds,
        ]);

        const candidates =
          nodesRef.current.filter(
            (n) => {
              if (
                movingIdsNow.has(
                  n.id,
                )
              ) {
                return false;
              }

              const def = nodeDef(
                n.type,
              );

              return (
                !!def &&
                def.category !==
                  "sources"
              );
            },
          );

        combineTarget =
          pickCombineCandidate(
            draggedUnion,
            candidates,
            COMBINE_GAP,
          );
      }

      return {
        primaryDesired,
        rigidOtherIds,
        obstaclesForPush,
        resolvedObstacles,
        combineTarget,
      };
    },
    [
      placeNode,
      resolveDisplacedPositions,
      nodeById,
    ],
  );

  const startDragNode = (
    event: React.PointerEvent<HTMLDivElement>,
    nodeId: string,
  ) => {
    if (
      event.button !== 0
    ) {
      return;
    }

    const target =
      event.target as HTMLElement;

    if (
      target.closest(
        "[data-node-control]",
      ) ||
      target.closest(
        "[data-in-port]",
      ) ||
      target.closest(
        "[data-out-port]",
      )
    ) {
      return;
    }

    const node =
      nodeById(nodeId);

    if (!node) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    closeContextMenu();

    onSelect(nodeId);

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    const basePositions: BaseNode[] =
      nodesRef.current
        .filter(
          (other) =>
            other.id !== nodeId,
        )
        .map((other) => ({
          id: other.id,
          x: other.x,
          y: other.y,
          width: other.width,
          height: other.height,
        }));

    const livePositions = new Map(
      basePositions.map(
        (other) => [
          other.id,
          { x: other.x, y: other.y },
        ],
      ),
    );

    const memberOffsets = new Map<
      string,
      {
        dx: number;
        dy: number;
        width: number;
        height: number;
      }
    >();

    if (node.groupId) {
      for (const other of nodesRef.current) {
        if (
          other.id === nodeId ||
          other.groupId !==
            node.groupId
        ) {
          continue;
        }

        memberOffsets.set(
          other.id,
          {
            dx: other.x - node.x,
            dy: other.y - node.y,
            width: other.width,
            height: other.height,
          },
        );
      }
    }

    dragRef.current = {
      id: nodeId,
      pointerId:
        event.pointerId,
      offsetX:
        point.x - node.x,
      offsetY:
        point.y - node.y,
      width: node.width,
      height: node.height,
      element:
        event.currentTarget,
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
      event.currentTarget.setPointerCapture(
        event.pointerId,
      );
    } catch {
      /* Pointer Capture non disponibile. */
    }
  };

  const moveDragNode = (
    event: React.PointerEvent<HTMLDivElement>,
  ) => {
    const drag =
      dragRef.current;

    if (
      !drag ||
      drag.pointerId !==
        event.pointerId
    ) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    const frame = computeDragFrame(
      drag,
      point,
    );

    onMove(
      drag.id,
      frame.primaryDesired.x,
      frame.primaryDesired.y,
    );

    for (const id of frame.rigidOtherIds) {
      const off =
        drag.memberOffsets.get(id)!;

      const x =
        frame.primaryDesired.x +
        off.dx;

      const y =
        frame.primaryDesired.y +
        off.dy;

      drag.livePositions.set(id, {
        x,
        y,
      });

      onMove(id, x, y);
    }

    for (const entry of frame.obstaclesForPush) {
      const pos =
        frame.resolvedObstacles.get(
          entry.id,
        );

      const live =
        drag.livePositions.get(
          entry.id,
        );

      if (!pos || !live) {
        continue;
      }

      if (
        pos.x !== live.x ||
        pos.y !== live.y
      ) {
        drag.livePositions.set(
          entry.id,
          pos,
        );

        onMove(
          entry.id,
          pos.x,
          pos.y,
        );
      }
    }

    setCombinePreview(
      (current) => {
        if (!frame.combineTarget) {
          return current
            ? null
            : current;
        }

        const movingIds = [
          drag.id,
          ...frame.rigidOtherIds,
        ];

        if (
          current &&
          current.targetId ===
            frame.combineTarget.id &&
          current.movingIds
            .length ===
            movingIds.length &&
          current.movingIds.every(
            (id, index) =>
              id ===
              movingIds[index],
          )
        ) {
          return current;
        }

        return {
          movingIds,
          targetId:
            frame.combineTarget.id,
        };
      },
    );
  };

  const endDragNode = (
    event: React.PointerEvent<HTMLDivElement>,
  ) => {
    const drag =
      dragRef.current;

    if (
      !drag ||
      drag.pointerId !==
        event.pointerId
    ) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    const frame = computeDragFrame(
      drag,
      point,
    );

    onMove(
      drag.id,
      frame.primaryDesired.x,
      frame.primaryDesired.y,
      true,
    );

    for (const id of frame.rigidOtherIds) {
      const off =
        drag.memberOffsets.get(id)!;

      onMove(
        id,
        frame.primaryDesired.x +
          off.dx,
        frame.primaryDesired.y +
          off.dy,
        true,
      );
    }

    /*
     * Commit anche per le card spostate per effetto domino, ma solo
     * quelle davvero finite fuori dalla loro posizione di partenza:
     * evita voci di history superflue per i nodi mai toccati.
     */
    for (const entry of frame.obstaclesForPush) {
      const pos =
        frame.resolvedObstacles.get(
          entry.id,
        ) ?? {
          x: entry.x,
          y: entry.y,
        };

      if (
        pos.x !== entry.x ||
        pos.y !== entry.y
      ) {
        onMove(
          entry.id,
          pos.x,
          pos.y,
          true,
        );
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
      const movingIds = [
        drag.id,
        ...frame.rigidOtherIds,
      ];

      const targetGroupId =
        frame.combineTarget.groupId;

      const targetGroupMembers =
        targetGroupId
          ? nodesRef.current
              .filter(
                (n) =>
                  n.groupId ===
                  targetGroupId,
              )
              .map((n) => n.id)
          : [];

      const fullIds = Array.from(
        new Set([
          ...movingIds,
          frame.combineTarget.id,
          ...targetGroupMembers,
        ]),
      );

      onGroupNodes(fullIds);
    }

    setCombinePreview(null);

    try {
      if (
        event.currentTarget.hasPointerCapture(
          event.pointerId,
        )
      ) {
        event.currentTarget.releasePointerCapture(
          event.pointerId,
        );
      }
    } catch {
      /* no-op */
    }

    dragRef.current =
      null;
  };

  /* ---------------------------------------------------------------------- */
  /*                              LINK DRAG                                  */
  /* ---------------------------------------------------------------------- */

  const startLink = (
    event: React.PointerEvent<HTMLSpanElement>,
    nodeId: string,
    port: string,
  ) => {
    if (
      event.button !== 0
    ) {
      return;
    }

    event.preventDefault();
    event.stopPropagation();

    closeContextMenu();

    try {
      event.currentTarget.setPointerCapture(
        event.pointerId,
      );
    } catch {
      /* no-op */
    }

    const point =
      toLocal(
        event.clientX,
        event.clientY,
      );

    setPending({
      node: nodeId,
      port,
      x: point.x,
      y: point.y,
      targetNode: null,
    });

    const move = (
      moveEvent: PointerEvent,
    ) => {
      const next =
        toLocal(
          moveEvent.clientX,
          moveEvent.clientY,
        );

      const element =
        document.elementFromPoint(
          moveEvent.clientX,
          moveEvent.clientY,
        );

      const card =
        element?.closest<HTMLElement>(
          "[data-node-card]",
        );

      const targetId =
        card?.dataset["nodeId"];

      const target =
        targetId
          ? liveNodeById(targetId)
          : undefined;

      if (
        target &&
        target.id !== nodeId
      ) {
        setPending(
          (current) =>
            current
              ? {
                  ...current,
                  x: next.x,
                  y: next.y,
                  targetNode:
                    target.id,
                }
              : current,
        );

        return;
      }

      setPending(
        (current) =>
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

    const up = (
      upEvent: PointerEvent,
    ) => {
      const element =
        document.elementFromPoint(
          upEvent.clientX,
          upEvent.clientY,
        );

      /*
       * Non è più necessario rilasciare esattamente sopra la porta:
       * basta essere sopra una qualsiasi card valida. Il collegamento
       * usa la porta di input più vicina al punto di rilascio.
       */
      const card =
        element?.closest<HTMLElement>(
          "[data-node-card]",
        );

      const targetId =
        card?.dataset["nodeId"];

      const target = targetId
        ? liveNodeById(targetId)
        : undefined;

      if (
        target &&
        target.id !== nodeId
      ) {
        const targetDef = nodeDef(
          target.type,
        );

        const inputs =
          targetDef?.inputs ?? [];

        if (inputs.length > 0) {
          const drop = toLocal(
            upEvent.clientX,
            upEvent.clientY,
          );

          let inPort = inputs[0]!;

          if (inputs.length > 1) {
            const ratio =
              (drop.y - target.y) /
              Math.max(
                1,
                target.height,
              );

            const index = Math.min(
              inputs.length - 1,
              Math.max(
                0,
                Math.round(
                  ratio *
                    (inputs.length -
                      1),
                ),
              ),
            );

            inPort =
              inputs[index] ??
              inPort;
          }

          onConnect(
            nodeId,
            port,
            target.id,
            inPort,
          );
        }
      }

      window.removeEventListener(
        "pointermove",
        move,
      );

      window.removeEventListener(
        "pointerup",
        up,
      );

      window.removeEventListener(
        "pointercancel",
        up,
      );

      setPending(null);
    };

    window.addEventListener(
      "pointermove",
      move,
    );

    window.addEventListener(
      "pointerup",
      up,
    );

    window.addEventListener(
      "pointercancel",
      up,
    );
  };

  /* ---------------------------------------------------------------------- */
  /*                          CONTEXT MENU STATE                             */
  /* ---------------------------------------------------------------------- */

  const closeContextMenu =
    useCallback(() => {
      setContextMenu(
        (current) => ({
          ...current,
          open: false,
        }),
      );
    }, []);

  const openContextMenu = (
    event: React.MouseEvent,
  ) => {
    event.preventDefault();
    event.stopPropagation();

    const card =
      (event.target as HTMLElement).closest<HTMLElement>(
        "[data-node-card]",
      );

    const nodeId =
      card?.dataset["nodeId"] ??
      null;

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

  const unlinkNode =
    useCallback(
      (nodeId: string) => {
        workflow.edges
          .filter(
            (edge) =>
              edge.toNode ===
              nodeId,
          )
          .forEach(
            (edge) =>
              onRemoveEdge(
                edge.id,
              ),
          );
      },
      [
        workflow.edges,
        onRemoveEdge,
      ],
    );

  /* ---------------------------------------------------------------------- */
  /*                              PALETTE                                    */
  /* ---------------------------------------------------------------------- */

  const dockPosition: Record<
    Dock,
    string
  > = {
    top:
      "left-1/2 top-0 -translate-x-1/2",
    bottom:
      "bottom-0 left-1/2 -translate-x-1/2",
    left:
      "left-0 top-1/2 -translate-y-1/2",
    right:
      "right-0 top-1/2 -translate-y-1/2",
  };

  const palette = (
    <div
      ref={paletteRef}
      className={`absolute z-40 ${dockPosition[paletteDock]}`}
    >
      <ToolPalette
        dock={paletteDock}
        onDockChange={
          setPaletteDock
        }
        onAdd={
          handlePaletteDoubleClick
        }
        onDragAdd={
          handlePaletteDrop
        }
      />
    </div>
  );

  /* ---------------------------------------------------------------------- */
  /*                            PENDING ROUTE                                */
  /* ---------------------------------------------------------------------- */

  const pendingRoute =
    useMemo(() => {
      if (!pending) {
        return null;
      }

      const source =
        nodeById(
          pending.node,
        );

      if (!source) {
        return null;
      }

      if (
        pending.targetNode
      ) {
        const target =
          nodeById(
            pending.targetNode,
          );

        if (target) {
          const route =
            getBestRoute(
              source,
              target,
              visibleNodes,
            );

          const indicator =
            getDropIndicatorPoint(
              target,
              surfaceW,
              surfaceH,
            );

          return {
            path: route
              ? pathFromPoints(
                  route.points,
                )
              : pathFromPoints([
                  getAnchor(
                    source,
                    "right",
                  ),
                  indicator,
                ]),
            indicator,
          };
        }
      }

      /*
       * Trascinamento libero (nessun target): la linea esce comunque
       * perpendicolare dal lato della card rivolto verso il puntatore.
       */
      const cx =
        source.x + source.width / 2;
      const cy =
        source.y + source.height / 2;

      const dx = pending.x - cx;
      const dy = pending.y - cy;

      const sourceSide: Side =
        Math.abs(dx) >= Math.abs(dy)
          ? dx >= 0
            ? "right"
            : "left"
          : dy >= 0
            ? "bottom"
            : "top";

      const sourceAnchor =
        getAnchor(
          source,
          sourceSide,
        );

      const stub = stubPoint(
        sourceAnchor,
        PORT_STUB,
      );

      const horizontalFirst =
        sourceSide === "left" ||
        sourceSide === "right";

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
    }, [
      pending,
      nodeById,
      visibleNodes,
      surfaceW,
      surfaceH,
    ]);

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
  const bubbles = useMemo(
    () => computeBubbles(visibleNodes),
    [visibleNodes],
  );

  /** Un box per ogni bubble esistente, per il contenitore visivo. */
  const groupBoxes = useMemo(
    () =>
      Array.from(
        bubbles.values(),
      ).map((bubble) => ({
        groupId: bubble.groupId,
        rect: bubble.rect,
        orientation:
          bubble.orientation,
        memberIds:
          bubble.memberIds,
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
  const isNodeUserDriven = (
    nodeId: string,
  ): boolean => {
    const drag = dragRef.current;

    if (!drag) {
      return false;
    }

    if (drag.id === nodeId) {
      return true;
    }

    return (
      !drag.detached &&
      drag.memberOffsets.has(
        nodeId,
      )
    );
  };

  /**
   * Il contenitore di una bubble non deve avere lag quando uno dei
   * suoi membri è sotto il drag diretto dell'utente (altrimenti il box
   * "insegue" con ritardo la card che lo sta facendo cambiare forma in
   * tempo reale) — ma deve animarsi morbidamente per qualunque altro
   * cambiamento (resize, cambio di orientamento, ingresso/uscita di un
   * membro).
   */
  const isBubbleUserDriven = (
    memberIds: readonly string[],
  ): boolean =>
    memberIds.some(
      isNodeUserDriven,
    );

  /**
   * Anteprima del box combinato mostrata durante il drag: racchiude le
   * card in movimento e quella candidata, finché restano abbastanza
   * vicine da poter essere unite al rilascio.
   */
  const combinePreviewBox =
    useMemo(() => {
      if (!combinePreview) {
        return null;
      }

      const target = nodeById(
        combinePreview.targetId,
      );

      const movers =
        combinePreview.movingIds
          .map(nodeById)
          .filter(
            (
              node,
            ): node is NodeGeometry =>
              !!node,
          );

      if (!target || movers.length === 0) {
        return null;
      }

      return unionRect([
        target,
        ...movers,
      ]);
    }, [
      combinePreview,
      nodeById,
    ]);

  /* ---------------------------------------------------------------------- */
  /*                               RENDER                                    */
  /* ---------------------------------------------------------------------- */

  return (
    <section
      ref={boxRef}
      data-palette-workspace
      className="glass-soft relative min-h-0 min-w-0 flex-1 overflow-hidden rounded-3xl select-none"
      style={{
        userSelect: "none",
        WebkitUserSelect: "none",
      }}
      onContextMenu={
        openContextMenu
      }
    >
      <CanvasContextMenu
        state={contextMenu}
        boundaryRef={boxRef}
        grid={grid}
        onClose={
          closeContextMenu
        }
        onAddDataset={
          onAddDataset
        }
        onFitView={
          fitView
        }
        onToggleGrid={() =>
          setGrid(
            (value) =>
              !value,
          )
        }
        onAutoLayout={() =>
          onLayoutChange(
            "auto",
          )
        }
        onDuplicateNode={
          onDuplicateNode
        }
        onRemoveNode={
          onRemoveNode
        }
        onUnlinkNode={
          unlinkNode
        }
      />

      {palette}

      {/* --------------------------------------------------------------- */}
      {/* Canvas controls                                                  */}
      {/* --------------------------------------------------------------- */}

      <div
        className="absolute right-3 top-3 z-30 flex items-center gap-1.5"
        onPointerDown={(event) =>
          event.stopPropagation()
        }
      >
        <span className="glass-chip flex h-8 items-center rounded-full px-2.5 text-[10px] text-muted-foreground">
          {Math.round(
            zoom * 100,
          )}
          %
        </span>

        <button
          type="button"
          onClick={() =>
            setZoom(
              (z) =>
                Math.max(
                  0.4,
                  +(
                    z - 0.1
                  ).toFixed(2),
                ),
            )
          }
          aria-label="Zoom out"
          className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <ZoomOut className="size-4" />
        </button>

        <button
          type="button"
          onClick={() =>
            setZoom(
              (z) =>
                Math.min(
                  1.8,
                  +(
                    z + 0.1
                  ).toFixed(2),
                ),
            )
          }
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
          onClick={() =>
            setGrid(
              (value) =>
                !value,
            )
          }
          aria-pressed={grid}
          aria-label={
            grid
              ? "Nascondi griglia"
              : "Mostra griglia"
          }
          title={
            grid
              ? "Nascondi griglia"
              : "Mostra griglia"
          }
          className={`glass-chip flex size-8 items-center justify-center rounded-full transition ${
            grid
              ? "text-foreground"
              : "text-muted-foreground/50 hover:text-foreground"
          }`}
        >
          <Grid3x3 className="size-4" />
        </button>

        <IsaMenu
          label="Disposizione delle card"
          Icon={
            workflow.layout ===
            "auto"
              ? LayoutGrid
              : Move
          }
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
                checked={
                  workflow.layout ===
                  "auto"
                }
                onToggle={() => {
                  onLayoutChange(
                    "auto",
                  );
                  close();
                }}
              />

              <IsaMenuCheckItem
                label="Trascinamento libero"
                checked={
                  workflow.layout !==
                  "auto"
                }
                onToggle={() => {
                  onLayoutChange(
                    "manual",
                  );
                  close();
                }}
              />
            </>
          )}
        </IsaMenu>

        <IsaMenu
          label="Visualizza sulle card"
          Icon={Eye}
          boundaryRef={boxRef}
          placement="auto"
        >
          {() => (
            <>
              <span className="block px-3 py-1.5 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                Visualizza
              </span>

              {DISPLAY_OPTIONS.map(
                (option) => (
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
            ? `${Math.round(
                24 * zoom,
              )}px ${Math.round(
                24 * zoom,
              )}px`
            : undefined,
          userSelect: "none",
          WebkitUserSelect:
            "none",
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
            WebkitUserSelect:
              "none",
          }}
        >
          {/* ---------------------------------------------------------- */}
          {/* Edges                                                      */}
          {/* ---------------------------------------------------------- */}

          <svg className="pointer-events-none absolute inset-0 size-full overflow-visible">
            {workflow.edges.map(
              (edge, edgeIndex) => {
                const from =
                  nodeById(
                    edge.fromNode,
                  );

                const to =
                  nodeById(
                    edge.toNode,
                  );

                if (
                  !from ||
                  !to
                ) {
                  return null;
                }

                const fromBubble =
                  from.groupId
                    ? bubbles.get(
                        from.groupId,
                      )
                    : undefined;

                const toBubble =
                  to.groupId
                    ? bubbles.get(
                        to.groupId,
                      )
                    : undefined;

                /*
                 * Fase 2, punto 2: nessuna freccia interna — entrambi
                 * gli endpoint nella STESSA bubble non vengono
                 * disegnati (il collegamento resta nel modello dati,
                 * solo non è renderizzato).
                 */
                if (
                  fromBubble &&
                  toBubble &&
                  fromBubble.groupId ===
                    toBubble.groupId
                ) {
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
                const fromGeometry =
                  fromBubble
                    ? bubbleNodeGeometry(
                        from,
                        fromBubble,
                      )
                    : from;

                const toGeometry =
                  toBubble
                    ? bubbleNodeGeometry(
                        to,
                        toBubble,
                      )
                    : to;

                const excludeIds =
                  fromBubble ||
                  toBubble
                    ? new Set<string>([
                        ...(fromBubble?.memberIds ??
                          []),
                        ...(toBubble?.memberIds ??
                          []),
                      ])
                    : undefined;

                const route =
                  getBestRoute(
                    fromGeometry,
                    toGeometry,
                    visibleNodes,
                    excludeIds,
                  );

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
                const path =
                  smoothPathFromPoints(
                    route.points,
                  );

                const active =
                  edge.fromNode ===
                    selectedId ||
                  edge.toNode ===
                    selectedId ||
                  hoverEdge ===
                    edge.id;

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
                    ? fromBubble.memberIds.some(
                        isNodeUserDriven,
                      )
                    : isNodeUserDriven(
                        edge.fromNode,
                      )) ||
                  (toBubble
                    ? toBubble.memberIds.some(
                        isNodeUserDriven,
                      )
                    : isNodeUserDriven(
                        edge.toNode,
                      ));

                const edgeTransition =
                  edgeIsUserDriven
                    ? "none"
                    : EDGE_AUTO_MOVE_TRANSITION;

                const rows =
                  formatRows(
                    analyzeNode(
                      workflow,
                      from,
                    ).rows,
                  );

                const middlePoint =
                  route.points[
                    Math.floor(
                      route.points
                        .length /
                        2,
                    )
                  ];

                return (
                  <g
                    key={
                      edge.id
                    }
                  >
                    {/* Hit area */}
                    <path
                      d={path}
                      fill="none"
                      stroke="transparent"
                      strokeWidth={14}
                      style={{
                        transition:
                          edgeTransition,
                      }}
                      className="pointer-events-auto cursor-pointer"
                      onPointerEnter={() =>
                        setHoverEdge(
                          edge.id,
                        )
                      }
                      onPointerLeave={() =>
                        setHoverEdge(
                          (current) =>
                            current ===
                            edge.id
                              ? null
                              : current,
                        )
                      }
                      onDoubleClick={() =>
                        onRemoveEdge(
                          edge.id,
                        )
                      }
                    />

                    {/* Solo linea, nessuna freccia */}
                    <path
                      d={path}
                      fill="none"
                      stroke="var(--brand)"
                      strokeWidth={
                        active
                          ? 2.8
                          : 2
                      }
                      strokeOpacity={
                        active
                          ? 0.95
                          : 0.42
                      }
                      strokeLinecap="round"
                      strokeLinejoin="round"
                      style={{
                        transition:
                          edgeTransition,
                      }}
                      className={`pointer-events-none ${
                        active
                          ? "edge-flow"
                          : ""
                      }`}
                    />

                    {/*
                     * Luce di flusso direzionale: piccolo punto che
                     * percorre lo stesso path (da from a to, vedi
                     * commento sopra) via CSS offset-path, in loop.
                     * Nessun ricalcolo per frame: il browser interpola
                     * offset-distance, niente rAF.
                     */}
                    <circle
                      r={
                        active
                          ? 2.6
                          : 2.1
                      }
                      fill="var(--brand-glow)"
                      className="edge-flow-dot pointer-events-none"
                      style={
                        {
                          offsetPath: `path("${path}")`,
                          animationDelay: `${-(
                            (edgeIndex %
                              6) *
                            0.45
                          )}s`,
                          /*
                           * Letto dalla keyframe come picco di
                           * opacità: un valore statico su `opacity`
                           * qui verrebbe ignorato, perché
                           * un'animazione CSS sovrascrive sempre lo
                           * stile inline della stessa proprietà.
                           */
                          "--edge-flow-peak":
                            active
                              ? 0.9
                              : 0.55,
                        } as React.CSSProperties
                      }
                    />

                    {hoverEdge ===
                      edge.id &&
                      middlePoint && (
                        <g className="pointer-events-none">
                          <rect
                            x={
                              middlePoint.x -
                              34
                            }
                            y={
                              middlePoint.y -
                              20
                            }
                            width={68}
                            height={18}
                            rx={4}
                            fill="var(--background)"
                            stroke="var(--glass-border)"
                          />

                          <text
                            x={
                              middlePoint.x
                            }
                            y={
                              middlePoint.y -
                              7
                            }
                            textAnchor="middle"
                            fontSize={10}
                            fill="var(--muted-foreground)"
                          >
                            {rows}{" "}
                            rows
                          </text>
                        </g>
                      )}
                  </g>
                );
              },
            )}

            {/* -------------------------------------------------------- */}
            {/* Linking preview                                          */}
            {/* -------------------------------------------------------- */}

            {pendingRoute && (
              <g>
                <path
                  d={
                    pendingRoute.path
                  }
                  fill="none"
                  stroke="var(--brand)"
                  strokeWidth={2}
                  strokeDasharray="5 5"
                  strokeLinecap="round"
                  strokeLinejoin="round"
                />

                {pendingRoute.indicator && (
                  <circle
                    cx={
                      pendingRoute
                        .indicator
                        .x
                    }
                    cy={
                      pendingRoute
                        .indicator
                        .y
                    }
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

          {groupBoxes.map(
            ({
              groupId,
              rect,
              orientation,
              memberIds,
            }) => (
              <div
                key={groupId}
                data-bubble-orientation={
                  orientation
                }
                /*
                 * L'orientamento è esposto qui (nessun effetto visivo
                 * oltre alla forma naturale del bounding box in questa
                 * fase): una futura fase lo userà per riallineare i
                 * membri lungo l'asse della bubble con un'animazione.
                 */
                className="glass-soft pointer-events-none absolute rounded-3xl"
                style={{
                  left:
                    rect.x -
                    GROUP_PADDING,
                  top:
                    rect.y -
                    GROUP_PADDING,
                  width:
                    rect.width +
                    GROUP_PADDING * 2,
                  height:
                    rect.height +
                    GROUP_PADDING * 2,
                  transition:
                    isBubbleUserDriven(
                      memberIds,
                    )
                      ? "none"
                      : BUBBLE_AUTO_MOVE_TRANSITION,
                  opacity: 0.55,
                }}
              />
            ),
          )}

          {combinePreviewBox && (
            <div
              className="gradient-brand pointer-events-none absolute rounded-3xl border-2 border-dashed border-brand"
              style={{
                left:
                  combinePreviewBox.x -
                  GROUP_PADDING,
                top:
                  combinePreviewBox.y -
                  GROUP_PADDING,
                width:
                  combinePreviewBox.width +
                  GROUP_PADDING * 2,
                height:
                  combinePreviewBox.height +
                  GROUP_PADDING * 2,
                opacity: 0.22,
              }}
            />
          )}

          {/* ---------------------------------------------------------- */}
          {/* Cards                                                      */}
          {/* ---------------------------------------------------------- */}

          {visibleNodes.map(
            (node) => {
              const def =
                nodeDef(
                  node.type,
                );

              if (!def) {
                return null;
              }

              const analysis =
                analyzeNode(
                  workflow,
                  node,
                );

              const incoming =
                workflow.edges.filter(
                  (edge) =>
                    edge.toNode ===
                    node.id,
                );

              const outgoing =
                workflow.edges.filter(
                  (edge) =>
                    edge.fromNode ===
                    node.id,
                );

              const selected =
                node.id ===
                selectedId;

              const hasIncoming =
                incoming.length >
                0;

              const hasOutgoing =
                outgoing.length >
                0;

              const status =
                STATUS[
                  analysis.errors
                    .length >
                    0 &&
                  node.status !==
                    "running"
                    ? "error"
                    : node.status
                ];

              const detail =
                nodeSummary(
                  node.type,
                  node.config,
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

