<script setup lang="ts">

import {computed, onMounted, ref, watch} from 'vue';

type Circle = {
  x: number,
  y: number,
}

const circle1 = ref<Circle>({x: 100, y: 100});
const circle2 = ref<Circle>({x: 300, y: 500});
const rotationIsCW1 = ref(false);
const rotationIsCW2 = ref(false);


type Point = {
  x: number,
  y: number,
}
/**
 * Find the tangent line between two circles.
 * The segment can then be used as part of a path that also comprise arcs on the circles
 * that rotate in the given directions
 * The circles have a fixed radius
 * @param center1
 * @param center2
 * @param radius
 * @param rotationIsCW1
 * @param rotationIsCW2
 */
function findTangentLine(center1: Point, center2: Point, radius: number, rotationIsCW1: boolean, rotationIsCW2: boolean) {
  const dx = center2.x - center1.x;
  const dy = center2.y - center1.y;
  const d = Math.sqrt(dx * dx + dy * dy);
  const alpha = Math.atan2(dy, dx);
  if ((rotationIsCW1 && rotationIsCW2) || (!rotationIsCW1 && !rotationIsCW2)) {
    const d1 = rotationIsCW1 ? 50 : -50;
    const d2 = rotationIsCW2 ? 50 : -50;
    const x1 = center1.x + d1 * Math.sin(alpha);
    const y1 = center1.y - d1 * Math.cos(alpha);
    const x2 = center2.x + d2 * Math.sin(alpha);
    const y2 = center2.y - d2 * Math.cos(alpha);
    return [[x1, y1], [x2, y2]];
  }
  else {
    const beta = Math.asin((50 * 2) / d);
    const d1 = rotationIsCW1 ? 50 : -50;
    const gamma = alpha + (rotationIsCW1 ? beta : -beta);
    const x1 = center1.x + d1 * Math.sin(gamma);
    const y1 = center1.y - d1 * Math.cos(gamma);
    const x2 = center2.x - d1 * Math.sin(gamma);
    const y2 = center2.y + d1 * Math.cos(gamma);
    return [[x1, y1], [x2, y2]];
  }
}

const lineEnds = computed(() => findTangentLine(circle1.value, circle2.value, 50, rotationIsCW1.value, rotationIsCW2.value,));

watch(lineEnds, (value) => {
//  console.log(value);
}, {immediate: true, deep: true});

// mouse move listener that updates the position  of circle1
const mouseMoveListener = (event: MouseEvent) => {
  circle1.value.x = event.clientX;
  circle1.value.y = event.clientY;
};

// mouse up listener that cycles through all possible combinaisons of rotationIsCW1 and rotationIsCW2
const mouseUpListener = () => {
  if (!rotationIsCW1.value && !rotationIsCW2.value) {
    rotationIsCW1.value = false;
    rotationIsCW2.value = true;
  } else if (!rotationIsCW1.value && rotationIsCW2.value) {
    rotationIsCW1.value = true;
    rotationIsCW2.value = true;
  } else if (rotationIsCW1.value && rotationIsCW2.value) {
    rotationIsCW1.value = true;
    rotationIsCW2.value = false;
  } else if (rotationIsCW1.value && !rotationIsCW2.value) {
    rotationIsCW1.value = false;
    rotationIsCW2.value = false;
  }
//  console.log(rotationIsCW1.value, rotationIsCW2.value);
};
//register the listeners
onMounted(() => {
  window.addEventListener('mousemove', mouseMoveListener);
  window.addEventListener('mouseup', mouseUpListener);
});

</script>




<template>
  <div id="container">
    <svg>
      <circle :cx="circle1.x" :cy="circle1.y" :r="50"></circle>
      <circle :cx="circle2.x" :cy="circle2.y" :r="50"></circle>
      <line :x1="lineEnds[0][0]" :y1="lineEnds[0][1]" :x2="lineEnds[1][0]" :y2="lineEnds[1][1]" stroke="black"></line>
    </svg>
  </div>
</template>

<style scoped lang="scss">
#container {
  position: relative;
  width: 1000px;
  height: 1000px;

  svg {
    width: 1000px;
    height: 1000px;
  }
}
</style>
