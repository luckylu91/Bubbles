<script setup lang="ts">
import {computed, onMounted, onUnmounted, ref, watch, watchEffect} from 'vue';
import {Rect} from 'components/domain/bubble/types';
import SVGBubbleHull from 'components/domain/bubble/SVGBubbleHull.vue';

const containerRef = ref<HTMLDivElement | null>(null);
const selection = ref<Selection | null>(null);
const updatedSelection = ref(false);

const containerRect = computed(() => {
  if (!containerRef.value) {
    return {x: 0, y: 0, width: 0, height: 0};
  }
  return containerRef.value.getBoundingClientRect();
});

const props = defineProps<{
  borderRadius: number,
  fill?: string,
  stroke?: string,
  showDebug: boolean,
  content: string,
}>();

const getSelection = (event: Event) => {
  if (!window.getSelection()) {
    return;
  }
  selection.value = window.getSelection()!;
//  console.log(selection.value.rangeCount);
  updatedSelection.value = !updatedSelection.value;
}
// watch(selection, () => {
//   console.log("selection", selection.value)
// }, {immediate: true});
const selectionRects = computed(() => {
  updatedSelection.value;
  if (!selection.value) {
    return [];
  }
  const rects = [];
  const snapDistance = 20;

  const bottoms = new Set<number>();
  for (let i = 0; i < selection.value!.rangeCount; i++) {
    const range = selection.value!.getRangeAt(i);
    const rangeRects = range.getClientRects();
    for (let j = 0; j < rangeRects.length; j++) {
      if (rangeRects.item(j) === null) {
        continue;
      }
      const rect = rangeRects.item(j)!;
      if (rect.width === 0 || rect.height === 0) {
        continue;
      }
      if (bottoms.size > 0) {
        const bottom = Array.from(bottoms).find((bottom) => Math.abs(bottom - rect.top) < snapDistance);
        if (bottom) {
          rect.height += rect.top - bottom;
          rect.y = bottom;
        }
      }
      rects.push(rect);
      bottoms.add(rect.bottom);
    }
  }
  return rects.map((rect) => {
    return {
      x: rect.x - containerRect.value.x,
      y: rect.y - containerRect.value.y,
      width: rect.width,
      height: rect.height,
    }
  });
})


watchEffect( () => {
// console.log('selection', selectionRects.value.length);
});

onMounted(() => {
  window.addEventListener('mousemove', getSelection);
});
onUnmounted(() => {
  window.removeEventListener('mousemove', getSelection);
});

</script>

<template>
  <div class="content-and-bubble" ref="containerRef">
    <div class="text-selection-bubble-content">
      {{content}}
    </div>
    <SVGBubbleHull
      class="bubble-hull"
      :rects="selectionRects"
      :border-radius="borderRadius"
      :fill="fill"
      :stroke="stroke"
      :show-debug="showDebug"
      :width="containerRect.width"
      :height="containerRect.height"
    />
  </div>
</template>

<style scoped lang="scss">
.content-and-bubble {
  position: relative;

  .text-selection-bubble-content {
    position: relative;
    z-index: 1;
  }

  .bubble-hull {
    position: absolute;
    z-index: 0;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }

  :deep(.text-selection-bubble-content) {
    width: 100%;
    height: 100%;
    max-width: 12em;
    word-break: break-all;
    font-size: 36px;

    &::selection {
      background-color: transparent;
    }
    &::-moz-selection {
      background-color: transparent;
    }
  }

}

</style>
