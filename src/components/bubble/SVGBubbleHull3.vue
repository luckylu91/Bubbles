<script setup lang="ts">
import {computed, ref} from 'vue';

type Rect = {
  x: number,
  y: number,
  width: number,
  height: number,
}
type Point = {
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

function pointFromStr(str: string): Point {
  const [x, y] = str.split("-").map(s => parseInt(s));
  return {x, y};
}


// list of merged rectangles per row
/**
 * Takes a list of bounding boxes and merges them into a list of bounding boxes per row
 * The y-axis is transformed to a row index
 * @param rects List of bounding boxes
 */
function mergeRects(rects: Rect[]): [Rect[][], Map<number, number>] {
  const mergedRectsPerRow: Rect[][] = [];
  const rowToY = new Map<number, number>();
  let lastTop = undefined;
  let lastBottom = undefined;
  let rowRects: Rect[] = [];
  for (let rect of rects) {
    let top = rect.y;
    let bottom = rect.y + rect.height;
    if (lastTop !== undefined && top != lastTop) {
      if (lastBottom !== undefined && top != lastBottom) {
        mergedRectsPerRow.push([]);
        rowRects = [];
        // rowToY.set(mergedRectsPerRow.length - 1, top);
      }
      mergedRectsPerRow.push(rowRects);
      rowRects = [];
      rowToY.set(mergedRectsPerRow.length - 1, top);
    }
    if (rowRects.length > 0 && rowRects[rowRects.length - 1].x + rowRects[rowRects.length - 1].width === rect.x) {
      const lastRect = rowRects[rowRects.length - 1];
      lastRect.width += rect.width;
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
  console.log("rowToY", rowToY)
  console.log("mergedRectsPerRow", mergedRectsPerRow)
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
  console.log("rectsToHull")
  // const mergedRectsPerRow = mergeRects(rects);
  const points = pointsPerRow(mergedRectsPerRow);
  const horizontalMap = horizontalMappings(points, mergedRectsPerRow);
  const verticalMap = verticalMappings(mergedRectsPerRow);
  const cycles: Point[][] = [];

  let cycle: Point[] = [];
  // while (globalMapping.size > 0 && k++ < 100) {
  while (horizontalMap.size > 0 && verticalMap.size > 0) {
    let pstart: Point = horizontalMap.values().next().value;
    let pv = verticalMap.get(pointToStr(pstart))!;
    verticalMap.delete(pointToStr(pstart));

    if (!pv || !horizontalMap.has(pointToStr(pv))) {
      console.log("pstart", pstart);
      console.log("pv", pv);
      throw Error();
    }

    let ph = horizontalMap.get(pointToStr(pv))!;
    horizontalMap.delete(pointToStr(pv));
    cycle.push(pv, ph);

    while (ph != pstart && ph !== undefined && pv !== undefined) {
      // console.log("\tpv", pv);
      // console.log("\tph", ph);
      pv = verticalMap.get(pointToStr(ph))!;
      verticalMap.delete(pointToStr(ph));
      ph = horizontalMap.get(pointToStr(pv))!;
      horizontalMap.delete(pointToStr(pv));
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

  return cycles;
}
//
// function addPointsToCycle(cycle: Point[]) {
//   const newCycle = [];
//   for (let i = 0; i < cycle.length; i++) {
//     const p1 = cycle[i];
//     const p2 = i < cycle.length - 1 ?  cycle[i + 1] : cycle[0];
//     const p1p2 = {
//       x: p2.x - p1.x,
//       y: p2.y - p1.y,
//     };
//     const distX = Math.abs(p1.x - p2.x);
//     const distY = Math.abs(p1.y - p2.y);
//     const dist = distX + distY;
//     if (dist < 50) {
//       // newCycle.push(p1);
//       newCycle.push({
//         x: p1.x + p1p2.x / 2,
//         y: p1.y + p1p2.y / 2,
//       });
//     }
//     else {
//       // newCycle.push(p1);
//       newCycle.push({
//         x: p1.x + p1p2.x / dist * 25,
//         y: p1.y + p1p2.y / dist * 25,
//       });
//       newCycle.push({
//         x: p2.x - p1p2.x / dist * 25,
//         y: p2.y - p1p2.y / dist * 25,
//       });
//     }
//   }
//   // newCycle.push(cycle[cycle.length - 1])
//   return newCycle;
// }

/**
 * Takes a cycle and returns a path and the list of points that are used to draw the path
 * @param cycle
 */
function cycleToPath(cycle: Point[]): [string, Point[]] {
  const points = [];
  let p1 = cycle[0];
  let p2 = cycle[1];
  const {x: dist12x, y: dist12y} = pointDist(p1, p2);
  const diff12 = pointDiff(p1, p2);
  const dist12 = dist12x + dist12y;
  let p12;
  if (dist12 < 50) {
    p12 = {
      x: p1.x + diff12.x / 2,
      y: p1.y + diff12.y / 2,
    };
  }
  else {
    p12 = {
      x: p2.x - diff12.x / dist12 * 25,
      y: p2.y - diff12.y / dist12 * 25,
    };
  }
  let path = `M ${p12.x} ${p12.y} `;
  points.push(p12);
  for (let i = 0; i < cycle.length; i++) {
    const p1 = cycle[i];
    const p2 = i < cycle.length - 1 ?  cycle[i + 1] : cycle[i - cycle.length + 1];
    const p3 = i < cycle.length - 2 ? cycle[i + 2] : cycle[i - cycle.length + 2];
    const {x: dist12x, y: dist12y} = pointDist(p1, p2);
    const diff12 = pointDiff(p1, p2);
    const dist12 = dist12x + dist12y;
    if (dist12 > 50) {
      path += `L ${p2.x - diff12.x / dist12 * 25} ${p2.y - diff12.y / dist12 * 25} `;
      points.push({
        x: p2.x - diff12.x / dist12 * 25,
        y: p2.y - diff12.y / dist12 * 25,
      });
    }
    const {x: dist23x, y: dist23y} = pointDist(p2, p3);
    const diff23 = pointDiff(p2, p3);
    const dist23 = dist23x + dist23y;
    let p23;
    if (dist23 < 50) {
      p23 = {
        x: p2.x + diff23.x / 2,
        y: p2.y + diff23.y / 2,
      };
    }
    else {
      p23 = {
        x: p2.x + diff23.x / dist23 * 25,
        y: p2.y + diff23.y / dist23 * 25,
      };
    }
    path += `Q ${p2.x} ${p2.y}, ${p23.x} ${p23.y} `;
    points.push(p23);
  }
  return [path, points];
}

const rects = ref<Rect[]>([
  {x: 0, y: 0, width: 300, height: 100},
  {x: 100, y: 100, width: 100, height: 100},
  {x: 300, y: 100, width: 200, height: 100},
  {x: 200, y: 200, width: 200, height: 100},
]);

const cycles = computed(() => {
  let [mergedRectsPerRow, rowToY] = mergeRects(rects.value);
  displaceMatchingCorners(mergedRectsPerRow);
  const cycles = rectsToHull(mergedRectsPerRow);
  console.log("cycles", cycles.map(cycle => cycle.map(point => ({x: point.x, y: rowToY.get(point.y)!}))))
  console.log("cycles BABA", cycles.map(cycle => cycle.map(point => ({x: point.x, y: 10 + 100 *point.y}))));
  return cycles.map(cycle => cycle.map(point => ({x: point.x, y: 10 + 100 *point.y})));
});

const pathAndPoints = computed(() => cycles.value.map(cycle => cycleToPath(cycle)));

// let displacedRects = computed(() => {
//   let [mergedRectsPerRow, rowToY] = mergeRects(rects.value);
//   mergedRectsPerRow = displaceMatchingCorners(mergedRectsPerRow);
//   return mergedRectsPerRow;
// })


// let time = 0;
// setInterval(() => {
//   const x = 300 + Math.sin((time + 245) / 1000) * 300;
//   rects.value[2].x = x;
//   time += 60;
// }, 60)

const rectColors = [
  "red",
  "green",
  "blue",
  "yellow",
  "purple",
  "cyan"
]

</script>

<template>
  <div class="container">
    <svg>
<!--      <template v-for="(rect, index) of displacedRects.flat()" :key="index">-->
<!--        <rect :x="rect.x" :width="rect.width" :y="rect.y" :height="rect.height" fill="black" ></rect>-->
<!--      </template>-->
<!--      <template v-for="(rect, index) of mergedRectsPerRow.flat()" :key="index">-->
<!--        <rect :x="rect.x + 10" :y="100*rect.y + 10" :width="rect.width" :height="rect.height" :fill="rectColors[index]"/>-->
<!--      </template>-->
      <template v-for="(pathXPoints, k) of pathAndPoints" :key="k">
        <circle v-for="{x, y} of pathXPoints[1]" :key="`${x}-${y}`" r="2" :cx="x" :cy="y"/>
<!--        <template v-for="(point, i) of cycle" :key="i">-->
<!--&lt;!&ndash;          <line v-if="i < cycle.length - 1" :x1="10 + cycle[i].x" :y1="10 + 100 *cycle[i].y" :x2="10 + cycle[i+1].x" :y2="10 + 100 *cycle[i+1].y" :stroke="`rgb(${80 * k + i*2}, ${ i*4}, ${255 - 80 * k + i*4})`" :stroke-width="10" />&ndash;&gt;-->
<!--        </template>-->
<!--        <path :d="path" fill="red" rx=""></path>-->
      </template>
<!--      <template v-for="(cycle, k) of cycles" :key="k">-->
<!--        <path :d="pathAndPoints[0]" fill="red"></path>-->
<!--      </template>-->
      <path :d="pathAndPoints.map(([path, points]) => path).reduce((s1, s2) => s1 + ' ' + s2)" fill="red"></path>
    </svg>
  </div>
</template>

<style scoped lang="scss">
.container {
  width: 1000px;
  height: 1000px;

  svg {

    width: 1000px;
    height: 1000px;
  }
}
</style>
