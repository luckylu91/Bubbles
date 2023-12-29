<script setup lang="ts">

import {ref, watch} from 'vue';
import SVGBubbleHull from 'components/domain/bubble/SVGBubbleHull.vue';
import { type Rect } from 'components/domain/bubble/SVGBubbleHull.vue';

type Props = {
  borderRadius: number,
  fill?: string,
  stroke?: string,
  showDebug: boolean,
}
const props = defineProps<Props>();

const items = ref(['Soudain', 'le', 'mec', 'éternue', 'et', 'tout', 'le', 'monde', 'décède', 'de', 'mort',])
const itemsOutlined = ref([true, true, false, true, false, true, true, false, false, true, true,])
const parentRef = ref<HTMLDivElement | null>(null);
const rects = ref<DOMRect[]>([]);
// const borderRadius = ref(20);
// const fill = ref('#fff');
// const stroke = ref('#000');

const parentRect = ref<Rect>({x: 0, y: 0, width: 0, height: 0});

function updateRects() {
  if (!parentRef.value)
    return;
  parentRect.value = parentRef.value!.getBoundingClientRect();
  let rectsValue = [];
  const container = parentRef.value!;
  for (let i = 0; i < container.children.length; i++) {
    if (!itemsOutlined.value[i]) {
      continue;
    }
    const child = container.children.item(i) as HTMLDivElement;
    const viewRect = child.getBoundingClientRect();
    viewRect.x -= parentRect.value.x;
    viewRect.y -= parentRect.value.y;
    rectsValue.push(viewRect);
  }
  rects.value = rectsValue;
}

watch(
  [parentRef, itemsOutlined],
  updateRects,
  {immediate: true, deep: true}
);

// listener to update rects if the disposition has changed (change of the container's size)
window.addEventListener('resize', updateRects);

</script>

<template>

<div id="container">
  <div id="itemsContainer" ref="parentRef">
    <div
      :class="[
      'item',
      {outlined: itemsOutlined[index]},
    ]"
      @click="itemsOutlined[index] = !itemsOutlined[index]"
      v-for="(item, index) of items"
      :key="item"
    >
      <div class="content">{{ item }}</div>
    </div>
    </div>
    <SVGBubbleHull
      :border-radius="borderRadius"
      :rects="rects"
      :fill="fill"
      :stroke="stroke"
      :show-debug="showDebug"
      :parent-rect="parentRect"
    />
  </div>
</template>

<style scoped lang="scss">
#container {
  position: relative;

  #itemsContainer {
    width: 100%;
    position: absolute;
    top: 0;
    left: 0;
    max-width: 20em;
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
    font-size: 36px;
    //z-index: 0;
    .item {
      position: relative;
      padding-left: .25em;
      padding-right: .25em;
      box-sizing: border-box;
    }
  }
}
</style>
