<template>
  <div>
    <div class="canvas-wrap" ref="canvasWrapRef">
      <div class="tool-box-out" v-show="isControllerVisiable">
        <ToolBox
          @updateSetting="updateSetting"
          :step="step"
          :history-list-length="historyListLength"
          :change-tool="changeTool"
        />
      </div>
      <canvas class="whiteboard-canvas" id="whiteboard"></canvas>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, provide, Ref, ref, onUnmounted } from 'vue';
import {
  ToolSettings,
  DrawingTool,
  toolCursorStyleMap,
} from '../Whiteboard/type';
import FabricCanvas from './core';
import HistoryManager from './core/historyManager';
import ToolBox from './ToolBox/index.vue';

import { getUrlParam } from '../../utils/utils';
import logger from '../../../TUIRoom/utils/common/logger';

const isAnnotationWin: Ref<boolean> = ref(false);
const isAnnotationStarted: Ref<boolean> = ref(false);
const canvas = ref<FabricCanvas>();
const canvasWrapRef = ref<HTMLDivElement>();
let upperCanvas = null as HTMLCanvasElement | null;
const step = ref<number>(0);
const historyListLength = ref<number>(0);
const changeTool = ref<string>('');
let historyManager: HistoryManager;
// eslint-disable-next-line no-undef
let whiteboardIntervalId: number;

// const canvas1 = document.getElementById('whiteboard-test');
// const ctx = canvas1.getContext("2d");
// ctx.fillStyle = "green";
// ctx.fillRect(10, 10, 150, 100);

provide('whiteboard', canvas);

const isControllerVisiable = computed(
  () => !isAnnotationWin.value || isAnnotationStarted.value
);

function init() {
  const fabricCanvas = new FabricCanvas('whiteboard');
  canvas.value = fabricCanvas;
  historyManager = new HistoryManager(canvas.value);
}

const toolActiveMap = {
  [DrawingTool.Image]: () => insertImage(),
  [DrawingTool.Clear]: () => historyManager.clear(),
  [DrawingTool.Redo]: () => historyManager.redo(),
  [DrawingTool.Undo]: () => historyManager.undo(),
  [DrawingTool.Download]: () => downloadCanvas(),
  [DrawingTool.Retract]: () => stopWhiteboard(),
  [DrawingTool.Select]: () => canvas.value?.setDrawingTool(DrawingTool.Select),
};

function stopWhiteboard() {
  canvas.value?.discardActiveObject();
  canvas.value?.renderAll();
  let isAnnotationWin = false;
  const annotationParam = getUrlParam('isAnnotationWin');
  if (annotationParam && annotationParam === 'true') {
    isAnnotationWin = true;
  }
  if (isAnnotationWin) {
    //ipcRenderer.send('annotation:stop-from-annotation-window');
  }
}

function resizeCanvas() {
  canvas.value.setWidth(canvasWrapRef.value?.offsetWidth);
  canvas.value.setHeight(canvasWrapRef.value?.offsetHeight);
}

function updateSetting(toolSetting: ToolSettings) {
  const behavior =
    toolActiveMap[toolSetting.drawingTool as keyof typeof toolActiveMap];
  if (behavior) {
    behavior();
    if (changeTool.value === DrawingTool.Select) {
      return;
    }
    changeTool.value = '';
    return;
  }
  const currentClass =
    toolCursorStyleMap[
      toolSetting.drawingTool as keyof typeof toolCursorStyleMap
    ];
  Object.values(toolCursorStyleMap).forEach(className => {
    if (className) {
      upperCanvas?.classList.remove(className);
    }
  });
  if (currentClass) {
    upperCanvas?.classList.add(currentClass);
    canvas.value?.setDrawingTool(toolSetting.drawingTool);
    changeTool.value = '';
    return;
  }
  canvas.value?.setDrawingTool(toolSetting.drawingTool);
  canvas.value?.setOptions(toolSetting);
  changeTool.value = '';
}

function insertImage() {
  if (!canvas.value || !canvas.value) {
    logger.error('Canvas is not available or not initialized.');
    return;
  }
  const input = document.createElement('input');
  input.type = 'file';
  const mimeTypes = ['image/png', 'image/jpeg', 'image/jpg', 'image/bmp'].join(
    ','
  );
  input.accept = mimeTypes;
  input.onchange = event => {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) {
      logger.info('No file selected.');
      return;
    }
    const validTypes = mimeTypes.split(',').map(type => type.trim());
    if (!validTypes.includes(file.type)) {
      logger.error(
        'Invalid file type. Please upload a PNG, JPEG, JPG, or BMP image.'
      );
      return;
    }
    const reader = new FileReader();
    reader.onload = event => {
      const img = new Image();
      img.onload = () => {
        canvas.value?.insertImage(img.src);
      };
      img.src = event.target?.result as string;
    };
    reader.onerror = () => {
      logger.error('Failed to read file.');
    };
    reader.readAsDataURL(file);
  };
  input.onerror = () => {
    logger.error('Failed to create file input.');
  };
  input.click();
}

function generateImageName() {
  const date = new Date();
  const year = date.getFullYear();
  const month = date.getMonth() + 1;
  const day = date.getDate();
  const hours = date.getHours();
  const minutes = date.getMinutes();
  const seconds = date.getSeconds();
  const formattedMonth = month < 10 ? `0${month}` : month;
  const formattedDay = day < 10 ? `0${day}` : day;
  const formattedHours = hours < 10 ? `0${hours}` : hours;
  const formattedMinutes = minutes < 10 ? `0${minutes}` : minutes;
  const formattedSeconds = seconds < 10 ? `0${seconds}` : seconds;
  return `${year}-${formattedMonth}-${formattedDay}-${formattedHours}.${formattedMinutes}.${formattedSeconds}.png`;
}

function downloadCanvas() {
  if (!canvas.value || !canvas.value) {
    return;
  }
  const dataURL = canvas.value.toDataURL({
    format: 'png',
  } as any);
  const link = document.createElement('a');
  link.style.display = 'none';
  link.href = dataURL;
  link.download = generateImageName();
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  //ipcRenderer.send('whiteboard:save-from-whiteboard-window');
}

function handleStepChange(stepNumber: number) {
  step.value = stepNumber;
}

function handleHistoryChange(historyListLengthNumber: number) {
  historyListLength.value = historyListLengthNumber;
}

function handleToolChange() {
  changeTool.value = DrawingTool.Select;
}

onMounted(() => {
  init();

  const annotationParam = getUrlParam('isAnnotationWin');
  if (annotationParam && annotationParam === 'true') {
    isAnnotationWin.value = true;
  }
  if (!isAnnotationWin.value) {
    canvas.value?.setBackgroundColor('white');
  }
  upperCanvas = document.querySelector(
    '.upper-canvas.whiteboard-canvas'
  ) as HTMLCanvasElement | null;

  historyManager.on('change-step', handleStepChange);
  historyManager.on('change-history', handleHistoryChange);
  canvas.value?.on('insert-images', handleToolChange);
  canvas.value.setWidth(canvasWrapRef.value?.offsetWidth);
  canvas.value.setHeight(canvasWrapRef.value?.offsetHeight);
  window.addEventListener('resize', resizeCanvas);

  whiteboardIntervalId = setInterval(() => {
    canvas?.value?.renderAll();
  }, 500);
});

onUnmounted(() => {
  historyManager.off('change-step', handleStepChange);
  historyManager.off('change-history', handleHistoryChange);
  canvas.value?.off('insert-images', handleToolChange);
  window.removeEventListener('resize', resizeCanvas);
  if (whiteboardIntervalId) {
    clearInterval(whiteboardIntervalId);
  }
});

</script>

<style lang="scss">
.eraser-cursor {
  cursor: url('./resources/editor/eraser-cursor.svg'), auto !important;
}

.laser-cursor {
  cursor: url('./resources/editor/laser-cursor.svg'), auto !important;
}

.canvas-wrap {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  margin: 0 auto;

  .whiteboard-canvas {
    flex: 1;
    width: 100%;
    height: 100%;
  }

  .tool-box-out {
    position: relative;;
    left: 8px;
    z-index: 3;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 0;
    height: 100%;
  }
}
</style>
