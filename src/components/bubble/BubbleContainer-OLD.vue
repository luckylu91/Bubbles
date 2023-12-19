<script setup lang="ts">
import {onMounted, ref} from 'vue';

const items = ref(['Soudain', 'un', 'mec', 'éternue', 'et', 'tout', 'le', 'monde', 'décède', 'de', 'mort',])
const itemsOutlined = ref([true, true, false, true, false, true, true, false, false, true, true,])
const containerRef = ref<HTMLDivElement>();

onMounted(() => {
  if (!containerRef.value)
    return;
  const container = containerRef.value!;
  let lastTop = undefined;
  const line = 0;
  for (let i = 0; i < container.children.length; i++) {
    const child = container.children.item(i) as HTMLDivElement;
    const top = child.getBoundingClientRect();
    if (top != lastTop) {
      child.classList.add('first-in-row');
      if (i > 0) {
        container.children.item(i - 1)?.classList.add('last-in-row');
      }
      if (i == container.children.length - 1) {
        child.classList.add('last-in-row');
      }
    }
    lastTop = top;
  }
})

</script>

<template>
  <div id="container" ref="containerRef">
    <div
      :class="[
        'item',
        {outlined: itemsOutlined[index]},
        {'before-outlined': index < items.length && itemsOutlined[index + 1]}
      ]"
      @click="itemsOutlined[index] = !itemsOutlined[index]"
      v-for="(item, index) of items"
      :key="item"
    >
      <div class="bubble"/>
      <div class="content">{{ item }}</div>
    </div>
  </div>
</template>

<style scoped lang="scss">
#container {
  width: 100%;
  max-width: 20em;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  z-index: 0;
  //column-gap: 1em;

  .item {
    position: relative;
    padding: .5em;

    .bubble {
      position: absolute;
      z-index: -1;
      left: 0;
      width: 100%;
      border-radius: 9999em;

      transition: height .5s, top .5s;
    }

    &:not(.outlined) {
      .bubble {
        height: 0;
        top: 50%;
      }
    }

    &.outlined {
      .bubble {
        background-color: #31CCEC;
        height: 100%;
        top: 0;
      }

      & + .outlined {
        .bubble {
          border-top-left-radius: 0;
          border-bottom-left-radius: 0;
        }
      }

      &.before-outlined {
        .bubble {
          border-top-right-radius: 0;
          border-bottom-right-radius: 0;
        }
      }
    }

    .first-in-row:before {//}, .last-in-row:after {
      border-radius: 9999em;
      height: 100%;
      width: 100%;
      background-color: #aaaaaa;
      content: " ";
    }
  }
}
</style>
