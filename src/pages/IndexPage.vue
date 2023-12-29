<script setup lang="ts">
import BubbleContainer from 'components/domain/bubble/BubbleFlexContainer.vue';
import {ref} from 'vue';
import {Rect} from 'components/domain/bubble/types';
import {watch} from 'vue';

const borderRadius = ref(27);
const fill = ref('#1976D2');
const stroke = ref('#000');
const fillSelected = ref(true);
const strokeSelected = ref(false);
const showDebug = ref(false);


const items = ref(['Soudain', 'le', 'mec', 'éternue', 'et', 'tout', 'le', 'monde', 'décède', 'de', 'mort',])
const itemsOutlined = ref([true, true, false, true, false, true, true, false, false, true, true,])
const parentRef = ref<HTMLDivElement | null>(null);
const rects = ref<DOMRect[]>([]);
// const borderRadius = ref(20);
// const fill = ref('#fff');
// const stroke = ref('#000');

const parentRect = ref<Rect>({x: 0, y: 0, width: 0, height: 0});
watch(
  [parentRef, itemsOutlined],
  updateRects,
  {immediate: true, deep: true}
);

// listener to update rects if the disposition has changed (change of the container's size)
window.addEventListener('resize', updateRects);

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
//  console.log(rects.value);
}
</script>

<template>
  <q-page class="row items-center justify-evenly bg-grey-1">
    <BubbleContainer
      :borderRadius="borderRadius"
      :fill="fillSelected ? fill : undefined"
      :stroke="strokeSelected ? stroke : undefined"
      :showDebug="showDebug"
      :rects="rects"
    >
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
    </BubbleContainer>

    <div class="actions">
      <q-input type="number" v-model="borderRadius" label="Border radius"/>
      <div class="action-with-checkbox">
        <q-checkbox v-model="fillSelected" />
        <q-input
          filled
          v-model="fill"
          class="color-picker-fill"
          label="Fill color"
        >
          <template v-slot:append>
            <q-icon name="colorize" class="cursor-pointer">
              <q-popup-proxy cover transition-show="scale" transition-hide="scale">
                <q-color v-model="fill" />
              </q-popup-proxy>
            </q-icon>
          </template>
        </q-input>
      </div>
      <div class="action-with-checkbox">
        <q-checkbox v-model="strokeSelected" />
        <q-input
          filled
          v-model="stroke"
          class="color-picker-stroke"
          label="Stroke color"
        >
          <template v-slot:append>
            <q-icon name="colorize" class="cursor-pointer">
              <q-popup-proxy cover transition-show="scale" transition-hide="scale">
                <q-color v-model="stroke" />
              </q-popup-proxy>
            </q-icon>
          </template>
        </q-input>
      </div>
      <div>
        <q-checkbox v-model="showDebug" label="Debug Mode" />
      </div>
    </div>
  </q-page>
</template>

<style scoped lang="scss">
#itemsContainer {
  width: 100%;
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
  .actions {
    margin-top: 2em;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    align-content: stretch;
    & > * {
      margin-bottom: 1em;
      width: 100%;
    }
    .action-with-checkbox {
      display: flex;
      flex-direction: row;
      gap: 1em;
    }
  }
</style>
