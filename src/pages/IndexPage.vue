<script setup lang="ts">
import BubbleContainer from 'components/bubble/BubbleContainer.vue';
import {ref} from 'vue';

const borderRadius = ref(20);
const fill = ref('#1976D2');
const stroke = ref('#000');
const fillSelected = ref(true);
const strokeSelected = ref(false);
const showDebug = ref(false);
</script>

<template>
  <q-page class="row items-center justify-evenly bg-grey-1">
    <BubbleContainer
      :borderRadius="borderRadius"
      :fill="fillSelected ? fill : undefined"
      :stroke="strokeSelected ? stroke : undefined"
      :showDebug="showDebug"
    />

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
