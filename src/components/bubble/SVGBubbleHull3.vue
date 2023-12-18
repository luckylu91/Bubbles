<script setup lang="ts">
import {computed, ref} from 'vue';
// list of rectangles per row

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

// const rects = ref<Rect[]>([
//   {x: 0, y: 0, width: 300, height: 100},
//   {x: 100, y: 1, width: 200, height: 100},
//   {x: 300, y: 1, width: 200, height: 100},
//   {x: 200, y: 2, width: 200, height: 100},
// ]);
const mergedRectsPerRow = ref<Rect[][]>([
  [{x: 200, y: 0, width: 500, height: 100}],
  [{x: 100, y: 1, width: 200, height: 100}, {x: 350, y: 1, width: 200, height: 100}],
  [{x: 200, y: 2, width: 200, height: 100}],
]);

// list of merged rectangles per row
function mergeRects(rects: Rect[]): Rect[][] {
  console.log("mergeRects")
  return [];
}

// list of points per row + mapping
function pointsPerRow(mergedRectsPerRow: Rect[][]): number[][] {
  console.log("pointsPerRow")
  const points: number[][] = [];
  for (let [y, rowRects] of mergedRectsPerRow.entries()) {
    const rowPoints = [];
    const mapping = new Map();
    rowPoints.push(...rowRects.flatMap(rect => [rect.x, rect.x + rect.width]));
    rowRects.forEach(rect => mapping.set(rect.x, rect.x + rect.width))
    points.push(rowPoints)
  }
  return points;
}


// mapping inter-row {x1: x2}
function interRowMappings(points: number[][], rects: Rect[][]): Map<number, number>[] {
  console.log("interRowMappings")
  const mappings: Map<number, number>[] = [];
  for (let y = 0; y < rects.length - 1; y++) {
    const row1 = rects[y];
    const row2 = rects[y+1];
    const rowPoints1 = points[y];
    const rowPoints2 = points[y + 1];

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
      console.log("findLeft")
      const points = [xRight, ...[...rowPoints].reverse().filter((x2) => x2 < xRight && x2 > xLeft), xLeft];
      return groupByPair(points, takeFirst);
    }
    function findRight(xLeft: number, rowPoints: number[], xRight: number, takeFirst: boolean):  [number, number][] {
      console.log("findRight")
      const points = [xLeft, ...rowPoints.filter((x2) => x2 < xRight && x2 > xLeft), xRight];
      return groupByPair(points, takeFirst);
    }

    const mapping = new Map();

    for (let rect1 of row1) {
      const xleft = rect1.x;
      const xright = rect1.x + rect1.width;
      const iright = rowPoints2.findIndex(x => x > xright);
      const isAboveRect = xright >= rowPoints2[0] && xright <= rowPoints2[rowPoints2.length - 1] && iright % 2 === 1;
      const lefts = findLeft(xright, rowPoints2, xleft, !isAboveRect);
      for (let [x1, x2] of lefts) {
        mapping.set(x1, x2);
      }
    }
    for (let rect2 of row2) {
      const xleft = rect2.x;
      const xright = rect2.x + rect2.width;
      const ileft = rowPoints1.findIndex(x => x > xleft);
      const isBelowRect = xleft >= rowPoints1[0] && xleft <= rowPoints1[rowPoints1.length - 1] && ileft % 2 === 1;
      const rights = findRight(xleft, rowPoints1, xright, !isBelowRect);
      for (let [x1, x2] of rights) {
        mapping.set(x1, x2);
      }
    }
    mappings.push(mapping);
  }


  // first row
  let row = rects[0];
  let mapping = new Map();
  for (let rect of row) {
    const xleft = rect.x;
    const xright = rect.x + rect.width;
    mapping.set(xleft, xright);
  }
  mappings.splice(0, 0, mapping);

  // last row
  row = rects[rects.length - 1];
  mapping = new Map();
  for (let rect of row) {
    const xleft = rect.x;
    const xright = rect.x + rect.width;
    mapping.set(xright, xleft);
  }
  mappings.push(mapping);



  return mappings;
}

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
  return mapping;
}


// start cycle with first segment of first row
function rectsToHull(mergedRectsPerRow: Rect[][]): Point[][] {
  console.log("rectsToHull")
  // const mergedRectsPerRow = mergeRects(rects);
  const points = pointsPerRow(mergedRectsPerRow);
  const interRowMaps = interRowMappings(points, mergedRectsPerRow);
  const verticalMap = verticalMappings(mergedRectsPerRow);
  const cycles: Point[][] = [];

  let globalMapping = new Map<string, Point>();
  // for (let [index, mapping] of mappings.entries()) {
  //   for (let [x1, x2] of mapping.entries()) {
  //     const point1 = {x: x1, y: index};
  //     const point2 = {x: x2, y: index};
  //     const point3 = {x: x1, y: index + 1};
  //     const point4 = {x: x2, y: index + 1};
  //     globalMapping.set(point3, point1);
  //     globalMapping.set(point1, point2);
  //     globalMapping.set(point2, point4);
  //   }
  // }
  for (let [p1, p2] of verticalMap.entries()) {
    globalMapping.set(p1, p2);
  }

  for (let [index, mapping] of interRowMaps.entries()) {
    for (let [x1, x2] of mapping.entries()) {
      const point1 = {x: x1, y: index};
      const point2 = {x: x2, y: index};
      globalMapping.set(pointToStr(point1), point2);
    }
  }

  let cycle: Point[] = [];
  let k = 0;
  while (globalMapping.size > 0 && k++ < 100) {
    let pstart: Point = globalMapping.values().next().value;
    console.log("start", pstart)
    // globalMapping.delete(pstart);
    // cycle.push(pstart, p1);
    // let p2 = globalMapping.get(p1);
    // let plast = pstart;
    let p = globalMapping.get(pointToStr(pstart));
    globalMapping.delete(pointToStr(pstart));
    cycle.push(pstart);
    while (p != pstart && p !== undefined) {
      console.log("\tp", p);
      cycle.push(p!);
      const pNew = globalMapping.get(pointToStr(p));
      globalMapping.delete(pointToStr(p));
      p = pNew;
    }
    // cycle.push(pstart);
    cycles.push(cycle);
    cycle = [];
  }

  return cycles;
}

function addPointsToCycle(cycle: Point[]) {
  const newCycle = [];
  for (let i = 0; i < cycle.length; i++) {
    const p1 = cycle[i];
    const p2 = i < cycle.length - 1 ?  cycle[i + 1] : cycle[0];
    const p1p2 = {
      x: p2.x - p1.x,
      y: p2.y - p1.y,
    };
    const distX = Math.abs(p1.x - p2.x);
    const distY = Math.abs(p1.y - p2.y);
    const dist = distX + distY;
    if (dist < 50) {
      // newCycle.push(p1);
      newCycle.push({
        x: p1.x + p1p2.x / 2,
        y: p1.y + p1p2.y / 2,
      });
    }
    else {
      // newCycle.push(p1);
      newCycle.push({
        x: p1.x + p1p2.x / dist * 25,
        y: p1.y + p1p2.y / dist * 25,
      });
      newCycle.push({
        x: p2.x - p1p2.x / dist * 25,
        y: p2.y - p1p2.y / dist * 25,
      });
    }
  }
  // newCycle.push(cycle[cycle.length - 1])
  return newCycle;
}

const cycles = computed(() => {
  const cycles = rectsToHull(mergedRectsPerRow.value);
  return cycles.map(cycle => cycle.map(point => ({x: point.x, y: point.y * 100})));
});


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

const pathAndPoints = computed(() => cycles.value.map(cycle => cycleToPath(cycle)));

const rectColors = [
  "red",
  "green",
  "blue",
  "yellow",
  "purple",
  "cyan"
]

// let time = 0;
// setInterval(() => {
//   // const width = 500 + Math.sin(time / 1000) * 400;
//   // const x = 300 + Math.sin((time + 245) / 1000) * 400;
//   // mergedRectsPerRow.value[1][1].width = width;
//   // mergedRectsPerRow.value[1][1].x = x;
//   time += 60;
// }, 60)

</script>

<template>
  <div class="container">
    <svg>
<!--      <template v-for="(rect, index) of mergedRectsPerRow.flat()" :key="index">-->
<!--        <rect :x="rect.x + 10" :y="100*rect.y + 10" :width="rect.width" :height="rect.height" :fill="rectColors[index]"/>-->
<!--      </template>-->
<!--      <template v-for="([path, points], k) of pathAndPoints" :key="k">-->
<!--        <circle v-for="{x, y} of points" :key="`${x}-${y}`" r="2" :cx="x" :cy="y"/>-->
<!--&lt;!&ndash;        <template v-for="(point, i) of cycle" :key="i">&ndash;&gt;-->
<!--&lt;!&ndash;&lt;!&ndash;          <line v-if="i < cycle.length - 1" :x1="10 + cycle[i].x" :y1="10 + 100 *cycle[i].y" :x2="10 + cycle[i+1].x" :y2="10 + 100 *cycle[i+1].y" :stroke="`rgb(${80 * k + i*2}, ${ i*4}, ${255 - 80 * k + i*4})`" :stroke-width="10" />&ndash;&gt;&ndash;&gt;-->
<!--&lt;!&ndash;        </template>&ndash;&gt;-->
<!--        <path :d="path" fill="red" rx=""></path>-->
<!--      </template>-->
<!--      <template v-for="(cycle, k) of cycles" :key="k">-->
<!--        <path :d="pathAndPoints[0]" fill="red"></path>-->
<!--      </template>-->
      <path :d="pathAndPoints.map(([path, points]) => path).reduce((s1, s2) => s1 + ' ' + s2)"></path>
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
