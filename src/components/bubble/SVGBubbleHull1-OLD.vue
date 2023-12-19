<script setup lang="ts">

import {computed, ref} from 'vue';

type Rect = {
  x: number,
  y: number,
  width: number,
  height: number,
}
const rects = ref<Rect[]>([
  {x: 0, y: 0, width: 300, height: 100},
  {x: 100, y: 100, width: 200, height: 100},
  {x: 300, y: 100, width: 200, height: 100},
  {x: 200, y: 200, width: 200, height: 100},
]);

const points = computed(() => {
  return rects.value.flatMap(({x, y, width, height}) => [
    [x + 20, y],
    [x, y + 20],
    [x + width - 20, y],
    [x + width, y + 20],
    [x + width - 20, y + height],
    [x + width, y + height - 20],
    [x + 20, y + height],
    [x, y + height - 20],
  ] as [number, number][])
})

function find(start: [number, number], direction: 'top'|'right'|'left'|'bottom', set: 'br'|'bl'|'tr'|'tl') {
  let condition: (point: [number, number]) => boolean;
  switch (set) {
    case 'br':
      condition = ([x, y]) => x > start[0] &&  y > start[1];
      break;
    case 'bl':
      condition = ([x, y]) => x < start[0] &&  y > start[1];
      break;
    case 'tr':
      condition = ([x, y]) => x > start[0] &&  y < start[1];
      break;
    case 'tl':
      condition = ([x, y]) => x < start[0] &&  y < start[1];
      break;
  }
  let compFunc: (point1: [number, number], point2: [number, number]) => bool;
  switch (direction) {
    case 'top':
      compFunc = ([x1, y1], [x2, y2]) => y1 <= y2;
      break;
    case 'right':
      compFunc = ([x1, y1], [x2, y2]) => x1 >= x2;
      break;
    case 'left':
      compFunc = ([x1, y1], [x2, y2]) => x1 <= x2;
      break;
    case 'bottom':
      compFunc = ([x1, y1], [x2, y2]) => y1 >= y2;
      break;
  }

  return points.value
    .map((point, index) => [point, index] as [[number, number], number])
    .filter(([point, index]) => condition(point))
    .reduce(([point1, index1], [point2, index2]) => compFunc(point1, point2) ? [point1, index1] : [point2, index2])
}

const orderedPoints = computed(() => {
  function compFunc([x1, y2]: [number, number], [x1, y2]: [number, number]) {

  }
  const [startPoint, startIndex] = points.value
    .map((point, index) => [point, index] as [[number, number], number])
    .reduce(([[x1, y1], index1], [[x2, y2], index2]) => ? [point1, index1] : [point2, index2])
})

</script>

<template>
  <div class="container">
    <svg>
      <circle v-for="[x, y] of points" :key="`${x}-${y}`" r="2" :cx="x+ 10" :cy="y + 10"/>
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
