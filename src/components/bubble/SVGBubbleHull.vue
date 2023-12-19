<script setup lang="ts">
import {computed, ref, watch} from 'vue';

export type Rect = {
  x: number,
  y: number,
  width: number,
  height: number,
}
export type Point = {
  x: number,
  y: number,
}

function pointToStr(point: Point): string {
  return `${point.x}-${point.y}`;
}

function pointDiff(p1: Point, p2: Point): Point {
  return {
    x: p2.x - p1.x,
    y: p2.y - p1.y,
  }
}

function pointDist(p1: Point, p2: Point): Point {
  const diff = pointDiff(p1, p2);
  return {
    x: Math.abs(diff.x),
    y: Math.abs(diff.y),
  };
}


// list of merged rectangles per row
/**
 * Takes a list of bounding boxes and merges them into a list of bounding boxes per row
 * The y-axis is transformed to a row index
 * @param rects List of bounding boxes
 * @param epsilonMerge Distance under which two rectangles are merged horizontally
 */
function mergeRects(rects: Rect[], epsilonMerge: number): [Rect[][], Map<number, number>] {
  const mergedRectsPerRow: Rect[][] = [];
  const rowToY = new Map<number, number>();
  let lastTop = undefined;
  let lastBottom = undefined;
  let rowRects: Rect[] = [];
  for (let rect of rects) {
    let top = rect.y;
    let bottom = rect.y + rect.height;
    if (lastTop !== undefined && top != lastTop) {
      mergedRectsPerRow.push(rowRects);
      rowToY.set(mergedRectsPerRow.length - 1, lastTop);
      rowRects = [];
      if (lastBottom !== undefined && top != lastBottom) {
        mergedRectsPerRow.push([]);
        rowToY.set(mergedRectsPerRow.length - 1, lastBottom);
      }
    }
    if (rowRects.length > 0) {
      const lastRect = rowRects[rowRects.length - 1];
      const distWithLastRect = rect.x - (lastRect.x + lastRect.width);
      if (distWithLastRect < epsilonMerge) {
        lastRect.width += rect.width;
      }
      else {
        rowRects.push(rect);
      }
    }
    else {
      rowRects.push(rect);
    }
    lastTop = top;
    lastBottom = bottom;
  }
  mergedRectsPerRow.push(rowRects);
  rowToY.set(mergedRectsPerRow.length - 1, lastTop!);
  rowToY.set(mergedRectsPerRow.length, lastBottom!);
  console.log("ROWTOY\n", rowToY)
  return [mergedRectsPerRow, rowToY];
}

/**
 * Displaces the rectangles that have matching corners in diagonal with some rectangle in the row bellow
 * @param rects List of merged rectangles per row
 */
function displaceMatchingCorners(rects: Rect[][]) {
  for (let y = 0; y < rects.length - 1; y++) {
    const row1Rects = rects[y];
    const row2Rects = rects[y + 1];

    for (let rect1 of row1Rects) {
      for (let rect2 of row2Rects) {
        const delta = 0.01 * (y + 1);
        let dXLeft = 0;
        let dXRight = 0;
        if (rect1.x == rect2.x + rect2.width) {
          dXLeft = delta;
        } else if (rect2.x == rect1.x + rect1.width) {
          dXRight = delta;
        }
        rect1.x -= dXLeft;
        rect1.width += dXLeft + dXRight;
      }
    }
  }
}

// list of points per row
/**
 * Takes a list of merged rectangles per row and returns a list of points per row
 * @param mergedRectsPerRow List of merged rectangles per row
 */
function pointsPerRow(mergedRectsPerRow: Rect[][]): number[][] {
  const points: number[][] = [];
  for (let rowRects of mergedRectsPerRow) {
    const rowPoints = [];
    const mapping = new Map();
    rowPoints.push(...rowRects.flatMap(rect => [rect.x, rect.x + rect.width]));
    rowRects.forEach(rect => mapping.set(rect.x, rect.x + rect.width))
    points.push(rowPoints)
  }
  return points;
}


/**
 * Find the cycle edges that are horizontal in the aggregation of rectangles
 * @param points
 * @param rects
 */
function horizontalMappings(points: number[][], rects: Rect[][]): Map<string, Point> {
  // const mappings: Map<number, number>[] = [];
  function groupByPair(values: number[], takeFirst: boolean): [number, number][] {
    const pairs = [];
    let i = takeFirst ? 0 : 1;
    for (i; i < values.length - 1; i += 2) {
      const x1 = values[i];
      const x2 = values[i + 1];
      pairs.push([x1, x2] as [number, number]);
    }
    return pairs;
  }

  function findLeft(xRight: number, rowPoints: number[], xLeft: number, takeFirst: boolean): [number, number][] {
    const points = [xRight, ...[...rowPoints].reverse().filter((x2) => x2 < xRight && x2 > xLeft), xLeft];
    return groupByPair(points, takeFirst);
  }
  function findRight(xLeft: number, rowPoints: number[], xRight: number, takeFirst: boolean):  [number, number][] {
    const points = [xLeft, ...rowPoints.filter((x2) => x2 < xRight && x2 > xLeft), xRight];
    return groupByPair(points, takeFirst);
  }

  const mapping = new Map<string, Point>();
  for (let y = 0; y < rects.length - 1; y++) {
    const row1 = rects[y];
    const row2 = rects[y+1];
    const rowPoints1 = points[y];
    const rowPoints2 = points[y + 1];

    for (let rect1 of row1) {
      const xleft = rect1.x;
      const xright = rect1.x + rect1.width;
      const iright = rowPoints2.findIndex(x => x > xright);
      const isAboveRect = xright >= rowPoints2[0] && xright <= rowPoints2[rowPoints2.length - 1] && iright % 2 === 1;
      const lefts = findLeft(xright, rowPoints2, xleft, !isAboveRect);
      for (let [x1, x2] of lefts) {
        mapping.set(pointToStr({x: x1, y: y + 1}), {x: x2, y: y + 1});
      }
    }
    for (let rect2 of row2) {
      const xleft = rect2.x;
      const xright = rect2.x + rect2.width;
      const ileft = rowPoints1.findIndex(x => x > xleft);
      const isBelowRect = xleft >= rowPoints1[0] && xleft <= rowPoints1[rowPoints1.length - 1] && ileft % 2 === 1;
      const rights = findRight(xleft, rowPoints1, xright, !isBelowRect);
      for (let [x1, x2] of rights) {
        mapping.set(pointToStr({x: x1, y: y + 1}), {x: x2, y: y + 1});
      }
    }
  }


  // first row
  let row = rects[0];
  for (let rect of row) {
    const xleft = rect.x;
    const xright = rect.x + rect.width;
    mapping.set(pointToStr({x: xleft, y: 0}), {x: xright, y: 0});
  }

  // last row
  row = rects[rects.length - 1];
  for (let rect of row) {
    const xleft = rect.x;
    const xright = rect.x + rect.width;
    mapping.set(pointToStr({x: xright, y: rects.length}), {x: xleft, y: rects.length});
  }
  console.log("HORIZONTAL\n", mapping)
  return mapping;
}

/**
 * Find the cycle edges that are vertical in the aggregation of rectangles
 * @param rects
 */
function verticalMappings(rects: Rect[][]): Map<string, Point> {
  const mapping = new Map<string, Point>();
  for (let [index, rowRects] of rects.entries()) {
    for (let rect of rowRects) {
      const xleft = rect.x;
      const xright = rect.x + rect.width;
      const p1top = {x: xleft, y: index};
      const p1bottom = {x: xleft, y: index + 1};
      const p2top = {x: xright, y: index};
      const p2bottom = {x: xright, y: index + 1};
      mapping.set(pointToStr(p1bottom), p1top);
      mapping.set(pointToStr(p2top), p2bottom);
    }
  }
  console.log("VERTICAL\n", mapping)
  return mapping;
}


// start cycle with first segment of first row
/**
 * Takes a list of merged rectangles per row and returns a list of cycles corresponding to the hull
 * @param mergedRectsPerRow List of merged rectangles per row
 */
function rectsToHull(mergedRectsPerRow: Rect[][]): Point[][] {
  const points = pointsPerRow(mergedRectsPerRow);
  const horizontalMap = horizontalMappings(points, mergedRectsPerRow);
  const verticalMap = verticalMappings(mergedRectsPerRow);
  const cycles: Point[][] = [];

  let cycle: Point[] = [];
  while (horizontalMap.size > 0 && verticalMap.size > 0) {
    let pstart: Point = horizontalMap.values().next().value;
    let pv = verticalMap.get(pointToStr(pstart))!;
    verticalMap.delete(pointToStr(pstart));

    if (!pv || !horizontalMap.has(pointToStr(pv))) {
      console.error("Un cas innatendu s'est produit dans la fonction rectsToHull:");
      console.error("Le point de départ n'a pas de point vertical associé " +
        "ou le point vertical n'a pas de point horizontal associé");
      console.error("pstart", pstart);
      console.error("pv", pv);
      return [];
    }

    let ph = horizontalMap.get(pointToStr(pv))!;
    horizontalMap.delete(pointToStr(pv));
    cycle.push(pv, ph);

    while (ph != pstart && ph !== undefined && pv !== undefined) {
      pv = verticalMap.get(pointToStr(ph))!;
      verticalMap.delete(pointToStr(ph));
      ph = horizontalMap.get(pointToStr(pv))!;
      horizontalMap.delete(pointToStr(pv));
      // console.log("\tpv", pv);
      // console.log("\tph", ph);
      // console.log("\thorizontalMap", horizontalMap);
      // console.log("\tverticalMap", verticalMap);
      if (pv !== undefined) {
        cycle.push(pv);
        if (ph !== undefined) {
          cycle.push(ph);
        }
      }
    }
    cycles.push(cycle);
    cycle = [];
  }
  console.log("CYCLES\n", cycles)
  return cycles;
}

function findTangentLine(center1: Point, center2: Point, radius: number, rotationIsCCW1: boolean, rotationIsCCW2: boolean): [Point, Point] {
  const dx = center2.x - center1.x;
  const dy = center2.y - center1.y;
  const d = Math.sqrt(dx * dx + dy * dy);
  const alpha = Math.atan2(dy, dx);
  if ((rotationIsCCW1 && rotationIsCCW2) || (!rotationIsCCW1 && !rotationIsCCW2)) {
    const d1 = rotationIsCCW1 ? -radius : radius;
    const d2 = rotationIsCCW2 ? -radius : radius;
    const x1 = center1.x + d1 * Math.sin(alpha);
    const y1 = center1.y - d1 * Math.cos(alpha);
    const x2 = center2.x + d2 * Math.sin(alpha);
    const y2 = center2.y - d2 * Math.cos(alpha);
    return [{x: x1, y: y1}, {x: x2, y: y2}];
  }
  else if (d <= 2 * radius) {
    const middle = {
      x: center1.x + dx / 2,
      y: center1.y + dy / 2,
    };
    return [middle, middle];
  }
  else {
    const beta = Math.asin((radius * 2) / d);
    const gamma = alpha + (rotationIsCCW1 ? -beta : beta);
    const d1 = rotationIsCCW1 ? -radius : radius;
    const x1 = center1.x + d1 * Math.sin(gamma);
    const y1 = center1.y - d1 * Math.cos(gamma);
    const x2 = center2.x - d1 * Math.sin(gamma);
    const y2 = center2.y + d1 * Math.cos(gamma);
    return [{x: x1, y: y1}, {x: x2, y: y2}];
  }
}


type CircleWithDirection = {
  center: Point,
  cross: number,
}
type CycleToPathResult = {
  path: string,
  debugPoints: Point[],
  debugCircles: CircleWithDirection[],
}
/**
 * Takes a cycle and returns a path and the list of points that are used to draw the path
 * @param cycle
 */
function cycleToPath(cycle: Point[], borderRadius: number): CycleToPathResult {
  type CycleStepData = {
    p: Point,
    diff: Point,
    dist: number,
    segmentIsShort: boolean,
    cross: number,
    circleCenter?: Point,
  }
  const getArrayElementCyclic = <T>(arr: T[], index: number): T => arr[(index + arr.length) % arr.length];

  console.log("cycle", cycle)
  let cycleStepData: CycleStepData[] = Array.from({length: cycle.length}, (_, i) => {
    const p1 = cycle[i];
    const p2 = getArrayElementCyclic(cycle, i + 1);
    const pm1 = getArrayElementCyclic(cycle, i - 1);
    const distVec12 = pointDist(p1, p2);
    const dist12 = distVec12.x + distVec12.y;
    const distVecm11 = pointDist(pm1, p1);
    const distm11 = distVecm11.x + distVecm11.y;
    const diff12 = pointDiff(p1, p2);
    const diffm11 = pointDiff(pm1, p1);
    const crossm112 = diffm11.x * diff12.y - diffm11.y * diff12.x;
    const segmentIsShort = dist12 < borderRadius;
    let circleCenter;
    if (!segmentIsShort) {
      console.log("p1", p1)
      console.log("pm1", pm1)
      console.log("p2", p2)
      circleCenter = {
        x: p1.x + (diff12.x / dist12 - diffm11.x / distm11) * borderRadius,
        y: p1.y + (diff12.y / dist12 - diffm11.y / distm11) * borderRadius,
      };
      console.log("circleCenter", circleCenter)
    }
    return {
      p: p1,
      diff: diff12,
      dist: dist12,
      segmentIsShort,
      cross: crossm112,
      circleCenter: circleCenter,
    };
  });

  cycleStepData = cycleStepData.filter(
    (data1, index) => !data1.segmentIsShort && !getArrayElementCyclic(cycleStepData, index - 1).segmentIsShort
  )

  if (cycleStepData.length < 2) {
    const centerOfCycle = cycle.reduce((p1, p2) => ({x: p1.x + p2.x / cycle.length, y: p1.y + p2.y / cycle.length}), {x: 0, y: 0});
    const pathOfReplacementCircle = `M ${centerOfCycle.x - borderRadius} ${centerOfCycle.y} A ${borderRadius} ${borderRadius} 0 0 1 ${centerOfCycle.x + borderRadius} ${centerOfCycle.y} A ${borderRadius} ${borderRadius} 0 0 1 ${centerOfCycle.x - borderRadius} ${centerOfCycle.y}`;
    return {
      path: pathOfReplacementCircle,
      debugPoints: [centerOfCycle],
      debugCircles: [],
    };
  }

  let pathParts = [];
  const pathPoints = [];
  for (let i = 0; i < cycleStepData.length; i++) {
    const cycleStep1 = getArrayElementCyclic(cycleStepData, i);
    const cycleStep2 = getArrayElementCyclic(cycleStepData, i + 1);
    const [pt1, pt2] = findTangentLine(
      cycleStep1.circleCenter!,
      cycleStep2.circleCenter!,
      borderRadius,
      cycleStep1.cross < 0,
      cycleStep2.cross < 0
    );

    pathParts.push(`A ${borderRadius} ${borderRadius} 0 0 ${cycleStep1.cross > 0 ? 1 : 0} ${pt1.x} ${pt1.y}`);
    pathParts.push(`L ${pt2.x} ${pt2.y}`);
    pathPoints.push(pt1);
    pathPoints.push(pt2);
  }
  const lastPoint = pathPoints[pathPoints.length - 1];
  pathParts.splice(0, 0, `M ${lastPoint.x} ${lastPoint.y} `);
  // return [pathParts.join(' '), pathPoints];
  return {
    path: pathParts.join(' '),
    debugPoints: pathPoints,
    debugCircles: cycleStepData.map(data => ({center: data.circleCenter!, cross: data.cross})),
  };
}

const cycles = computed(() => {
  // Arrange rectangles per row and merge adjacent ones
  let [mergedRectsPerRow, rowToY] = mergeRects(props.rects, props.epsilonMerge);
  // Displace corners in the (edge) case of rectangles whose corners touch diagonally
  displaceMatchingCorners(mergedRectsPerRow);
  // Compute the hulls
  const cycles = rectsToHull(mergedRectsPerRow);
  // Put back the right y-axis scale
  return cycles.map(cycle => cycle.map(point => ({x: point.x, y: rowToY.get(point.y)!})));
});

const cycleToPathResults = computed(() => {
  return cycles.value.map(cycle => {
    return cycleToPath(cycle, props.borderRadius);
  });
});


type Props = {
  rects: Rect[],
  borderRadius: number,
  epsilonMerge?: number,
  showDebug?: boolean,
  stroke?: string,
  fill?: string,
  parentRect: Rect,
}
const props = withDefaults(defineProps<Props>(), {
  borderRadius: 20,
  epsilonMerge: 1,
  showDebug: false,
});

</script>

<template>
    <svg width="100%" :viewBox="`${0} ${0} ${parentRect.width} ${parentRect.height}`">
<!--      <template v-for="(rect, index) of displacedRects.flat()" :key="index">-->
<!--        <rect :x="rect.x" :width="rect.width" :y="rect.y" :height="rect.height" fill="black" ></rect>-->
<!--      </template>-->
<!--      <template v-for="(rect, index) of mergedRectsPerRow.flat()" :key="index">-->
<!--        <rect :x="rect.x + 10" :y="100*rect.y + 10" :width="rect.width" :height="rect.height" :fill="rectColors[index]"/>-->
<!--      </template>-->
      <template v-for="(pathXPoints, k) of cycleToPathResults" :key="k">
<!--        <circle v-for="{x, y} of pathXPoints[1]" :key="`${x}-${y}`" r="2" :cx="x" :cy="y"/>-->
<!--        <template v-for="(point, i) of cycle" :key="i">-->
<!--&lt;!&ndash;          <line v-if="i < cycle.length - 1" :x1="10 + cycle[i].x" :y1="10 + 100 *cycle[i].y" :x2="10 + cycle[i+1].x" :y2="10 + 100 *cycle[i+1].y" :stroke="`rgb(${80 * k + i*2}, ${ i*4}, ${255 - 80 * k + i*4})`" :stroke-width="10" />&ndash;&gt;-->
<!--        </template>-->
<!--        <path :d="path" fill="red" rx=""></path>-->
      </template>
      <template v-if="$props.showDebug">
        <circle
          v-for="({center, cross}, k) of cycleToPathResults.flatMap(({debugCircles}) => debugCircles)"
          :key="k" :r="$props.borderRadius"
          :cx="center.x"
          :cy="center.y"
          :fill="cross > 0 ? 'rgba(0,0,0,0.3)' : 'rgba(255,0,0,0.3)'"
        />
      </template>
      <path
        v-if="cycleToPathResults.length > 0"
        :d="cycleToPathResults.map(({path}) => path).reduce((s1, s2) => s1 + ' ' + s2)"
        :stroke="props.showDebug ? 'green' : props.stroke ?? 'none'"
        :fill="props.showDebug ? 'none' : (props.fill ?? 'none')"
      />
  </svg>
</template>

<style scoped lang="scss">
  svg {
    width: 100%;
    height: 100%;
  }
</style>
