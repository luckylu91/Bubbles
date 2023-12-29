<script setup lang="ts">
import SVGBubbleHull from 'components/domain/bubble/SVGBubbleHull.vue';
import {ref, watch} from 'vue';
import {Rect} from 'components/domain/bubble/types';

type Props = {
  // rects: Rect[],
  items: string[],
  borderRadius: number,
  // epsilonMerge?: number,
  // showDebug?: boolean,
  // stroke?: string,
  // fill?: string,
}
const props = defineProps<Props>();

const containerRef = ref<HTMLDivElement | null>(null);
const containerRect = ref({x: 0, y: 0, width: 0, height: 0});
watch(([props.rects, containerRef]), () => {
  if (!containerRef.value) {
    return;
  }
  containerRect.value = containerRef.value.getBoundingClientRect();
}, {immediate: true});

</script>

<template>
  <div class="bubble-container" ref="containerRef">
    <div class="content">
      <div class="item"></div>
    </div>
    <SVGBubbleHull
      class="bubble-hull"
      :rects="$props.rects"
      :border-radius="$props.borderRadius"
      :stroke="$props.stroke"
      :show-debug="$props.showDebug"
      :width="containerRect.width"
      :height="containerRect.height"
    />
  </div>
</template>

<style scoped lang="scss">
.bubble-container {
  position: relative;

  .content {
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
}

</style>
