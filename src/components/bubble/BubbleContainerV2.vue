<script setup lang="ts">
import {onMounted, ref, watch} from 'vue';

const startWords = ['Soudain', 'un', 'mec', 'éternue', 'et', 'tout', 'le', 'monde', 'décède', 'de', 'mort',];
const items = ref(startWords.map(word => ({
  content: word,
  outlined: false,
  cornerOutlined: {
    topLeft: false,
    topRight: false,
    bottomLeft: false,
    bottomRight: false,
  },
  firstInRow: false,
  lastInRow: false,
  itemLeftAbove: undefined,
  itemLeftBelow: undefined,
  itemRightAbove: undefined,
  itemRightBelow: undefined,
})));
const itemsLeftRightAndLine = ref<{line: number, left: number, right: number}[]>(items.value.map(_ => ({left: 0, right: 0, line: 0})));
const boundariesPerLine = ref(new Map());
const lineOffsets = ref(new Map());
const lineEndOffsets = ref(new Map());
const nLines = ref(0);

const cornerOutlinedDefault = {
  topLeft: false,
  topRight: false,
  bottomLeft: false,
  bottomRight: false,
};

const updateLTopologicData = () => {
  if (!containerRef.value)
    return;
  const container = containerRef.value!;
  let lastTop = undefined;
  let line = -1;
  for (let i = 0; i < container.children.length; i++) {
    const child = container.children.item(i) as HTMLDivElement;
    const {top, left, right} = child.getBoundingClientRect();
    if (top != lastTop) {
      // New Line
      line += 1;
      boundariesPerLine.value.set(line, []);
      lineOffsets.value.set(line, i);
      if (line > 0) {
        boundariesPerLine.value.get(line - 1).push(right)
        lineEndOffsets.value.set(line - 1, i);
      }
      items.value[i].firstInRow = true;
      if (i > 0) {
        items.value[i - 1].lastInRow = true;
      }
    }
    if (i == container.children.length - 1) {
      items.value[i].lastInRow = true;
    }
    boundariesPerLine.value.get(line)!.push(left)
    if (i == containerRef.value.children.length - 1) {
      boundariesPerLine.value.get(line)!.push(right);
    }
    itemsLeftRightAndLine.value[i] = {left, right, line};
    lastTop = top;
  }
  nLines.value = line;
  lineEndOffsets.value.set(line, container.children.length);

  for (let i = 0; i < container.children.length; i++) {
    let {left, right, line} = itemsLeftRightAndLine.value[i];
    console.log("IIIII", i)
    console.log("-----", line)
    if (line > 0) {
      const lineBoundaries = boundariesPerLine.value.get(line - 1);
      const aboveLeft = findInSorted(left, lineBoundaries);
      if (aboveLeft >= 0 && aboveLeft < lineBoundaries.length) {
        items.value[i].itemLeftAbove = aboveLeft + lineOffsets.value.get(line - 1);
      }
      const aboveRight = findInSorted(right, lineBoundaries);
      if (aboveRight >= 0 && aboveRight < lineBoundaries.length) {
        items.value[i].itemRightAbove = aboveRight + lineOffsets.value.get(line - 1);
      }
    }
    if (line < nLines.value - 1) {
      const lineBoundaries = boundariesPerLine.value.get(line + 1);
      const bellowLeft = findInSorted(left, lineBoundaries);
      if (bellowLeft >= 0 && bellowLeft < lineBoundaries.length) {
        items.value[i].itemLeftBelow = bellowLeft + lineOffsets.value.get(line + 1);
      }
      const bellowRight = findInSorted(right, lineBoundaries);
      if (bellowRight >= 0 && bellowRight < lineBoundaries.length) {
        items.value[i].itemRightBelow = bellowRight + lineOffsets.value.get(line + 1);
      }
    }
  }
}

// const items = ref(['Soudain', 'un', 'mec', 'éternue', 'et', 'tout', 'le', 'monde', 'décède', 'de', 'mort',])
// const itemsOutlined = ref([true, true, false, true, false, true, true, false, false, true, true,])
// const itemsFirstInRow = ref([false, false, false, false, false, false, false, false, false, false, false,])
// const itemsLastInRow = ref([false, false, false, false, false, false, false, false, false, false, false,])
// const itemsCornerOutlined = ref([false, false, false, false, false, false, false, false, false, false, false,])
const containerRef = ref<HTMLDivElement>();

const findInSorted = (value: number, array: number[]) => {
  if (value < array[0]) {
    return -1;
  }
  for (let i = 0; i < array.length; i++) {
    if (value < array[i]) {
      return i - 1;
    }
  }
  return array.length;
}

// const attributeClasses = () => {
//   if (!containerRef.value)
//     return;
//   const container = containerRef.value!;
//   let lastTop = undefined;
//   let line = -1;
//   let leftsAndIndexPerLine = [];
//   let leftsAndIndex = [];
//   for (let i = 0; i < container.children.length; i++) {
//     items.value[i].cornerOutlined = {...cornerOutlinedDefault};
//     const child = container.children.item(i) as HTMLDivElement;
//     const {top, left} = child.getBoundingClientRect();
//     if (top != lastTop) {
//       // New Line
//       line += 1;
//       if (i > 0) {
//         leftsAndIndexPerLine.push(leftsAndIndex);
//       }
//       leftsAndIndex = [];
//       console.log(`line ${line} starts at index ${i}`);
//       items.value[i].firstInRow = true;
//       if (i > 0) {
//         items.value[i - 1].lastInRow = true;
//       }
//     }
//     if (i == container.children.length - 1) {
//       items.value[i].lastInRow = true;
//     }
//     leftsAndIndex.push([left, i]);
//     if (line > 0) {
//       const lastLine = leftsAndIndexPerLine[line - 1];
//       const indexInLastLine = findInSorted(left, lastLine.map(lai => lai[0]));
//       if (
//         indexInLastLine >= 0 &&
//         indexInLastLine < lastLine.length
//       ) {
//         const topIndex = lastLine[indexInLastLine][1];
//         if (
//           items.value[topIndex].outlined &&
//           i < items.value.length &&
//           items.value[i + 1].outlined
//         ) {
//           items.value[i].cornerOutlined.topRight = true;
//           items.value[i + 1].cornerOutlined.topLeft = true;
//         }
//       }
//     }
//     lastTop = top;
//   }
// }
const attributeClasses = () => {
  for (let i = 0; i < items.value.length; i++) {
    const item = items.value[i];
    const {line} = itemsLeftRightAndLine.value[i];
    item.cornerOutlined = {...cornerOutlinedDefault};
    if (line > 0) {
      if (
        item.itemLeftAbove && items.value[item.itemLeftAbove].outlined
      ) {
        if (item.outlined || (i > lineOffsets.value.get(line) && items.value[i - 1].outlined)) {
          items.value[i].cornerOutlined.topLeft = true;
        }
      }
      if (
        item.itemRightAbove && items.value[item.itemRightAbove].outlined
      ) {
        if (item.outlined || (i < lineEndOffsets.value.get(line) && items.value[i + 1].outlined)) {
          items.value[i].cornerOutlined.topRight = true;
        }
      }
    }
    if (line < nLines.value - 1) {
      if (
        item.itemLeftBelow && items.value[item.itemLeftBelow].outlined
      ) {
        if (item.outlined || (i > lineOffsets.value.get(line) && items.value[i - 1].outlined)) {
          items.value[i].cornerOutlined.bottomLeft = true;
        }
      }
      if (
        item.itemRightBelow && items.value[item.itemRightBelow].outlined
      ) {
        if (item.outlined || (i < lineEndOffsets.value.get(line) && items.value[i + 1].outlined)) {
          items.value[i].cornerOutlined.bottomRight = true;
        }
      }
    }
  }
}

onMounted(updateLTopologicData)
onMounted(attributeClasses)
watch([containerRef, items], () => {
  attributeClasses();
  updateLTopologicData();
}, {immediate: true, deep: true})
watch([containerRef, items], () => {
  console.log("WATCH OUT")
  console.log(items.value)
}, {immediate: true, deep: true})

</script>

<template>
  <div id="container" ref="containerRef">
    <div
      :class="[
        'item',
        {outlined: items[index].outlined},
        {'before-outlined': index < items.length - 1 && items[index + 1].outlined},
        {'first-in-row': items[index].firstInRow},
        {'last-in-row': items[index].lastInRow},
        {'outlined-top-right': items[index].cornerOutlined.topRight},
        {'outlined-top-left': items[index].cornerOutlined.topLeft},
        {'outlined-bottom-right': items[index].cornerOutlined.bottomRight},
        {'outlined-bottom-left': items[index].cornerOutlined.bottomLeft},
      ]"
      @click="items[index].outlined = !items[index].outlined"
      v-for="(item, index) of items"
      :key="item"
    >
      <div class="bubble"/>
      <div class="content">{{ item.content }}</div>
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
  background-color: #31CCEC;

  .item {
    position: relative;
    padding: .5em;
    height: 2em;
    background-color: #FFFFFF;
    z-index: 0;

    .content {
      z-index: 2;
    }

    .bubble {
      position: absolute;
      z-index: -1;
      left: 0;
      width: 100%;
      border-radius: 1em;
      transition: height .5s, top .5s;

      background-color: #31CCEC;
    }

    &:not(.outlined) {
      .bubble {
        height: 0;
        top: 50%;
      }
    }

    &.outlined-top-right, &.outlined-top-left, &.outlined-below-right, &.outlined-below-left {
      &:not(.outlined) {
        background-color: #31CCEC;
        .bubble {
          background-color: #FFFFFF;
          height: 100%;
          top: 0;

        }
        &.outlined-top-left {
          .bubble {
            border-radius: 1em 0 0 0;
          }
        }
        &.outlined-top-right {
          .bubble {
            border-radius: 0 1em 0 0;
          }
        }
        &.outlined-below-right {
          .bubble {
            border-radius: 0 0 1em 0;
          }
        }
        &.outlined-below-left {
          .bubble {
            border-radius: 0 0 0 1em;
          }
        }
      }

    }
    :not(.outlined) {
    }

    &.outlined-top-left.outlined {
      .bubble {
        border-top-left-radius: 0 !important;
      }
    }
    &.outlined-top-right.outlined {
      .bubble {
        border-top-right-radius: 0 !important;
      }
    }
    &.outlined-bottom-left.outlined {
      .bubble {
        border-bottom-left-radius: 0 !important;
      }
    }
    &.outlined-bottom-right.outlined {
      .bubble {
        border-bottom-right-radius: 0 !important;
      }
    }

    &.outlined {
      .bubble {
        height: 100%;
        top: 0;
      }

      // outlined to the left
      & + .outlined:not(.first-in-row) {
        .bubble {
          border-top-left-radius: 0;
          border-bottom-left-radius: 0;
        }
      }

      // outlined to the right
      &.before-outlined:not(.last-in-row) {
        .bubble {
          border-top-right-radius: 0;
          border-bottom-right-radius: 0;
        }
      }
    }

    &.first-in-row:before, &.last-in-row:after {
        position: absolute;
        display: inline-block;
        z-index: 4;
        top: 0;
        height: 100%;
        width: 1000em;
        background-color: #FFFFFF;
        content: " ";
    }
    &.first-in-row:before {
      left: -1000em;
    }
    &.last-in-row:after {
      right: -1000em;
    }
  }
}
</style>
