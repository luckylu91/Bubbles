<script setup lang="ts">
import {computed, ref} from 'vue/dist/vue';

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
</script>

<template>

</template>

<style scoped lang="scss">

</style>
