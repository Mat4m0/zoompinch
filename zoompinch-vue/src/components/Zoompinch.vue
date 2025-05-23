<template>
  <div
    ref="zoompinchRef"
    class="zoompinch"
    :style="{
      '--canvas-width': `${width}px`,
      '--canvas-height': `${height}px`,
      '--canvas-ratio': width / height,
      '--rendering-scale': renderingScale,
      '--translate-x': `${renderingTranslate[0]}px`,
      '--translate-y': `${renderingTranslate[1]}px`,
      '--rotate': `${radiansToDegrees(renderingRotate)}deg`,
      '--offset-top': `${props.offset.top}px`,
      '--offset-right': `${props.offset.right}px`,
      '--offset-bottom': `${props.offset.bottom}px`,
      '--offset-left': `${props.offset.left}px`,
      '--transition-duration': `${transitionDuration}s`,
    }"
    :class="{
      'transition-enabled': transitionEnabled,
    }"
    @touchstart="touchstartProxy"
    @mousedown="mousedownProxy"
    @gesturestart="gesturestartProxy"
  >
    <div ref="canvasRef" class="canvas">
      <slot name="canvas" />
    </div>
    <div ref="matrixRef" class="matrix">
      <slot
        name="matrix"
        :compose-point="composePoint"
        :compose="compose"
        :clientCoordinatesToCanvasCoordinates="clientCoordinatesToCanvasCoordinates"
        :normalizeMatrixCoordinates="normalizeMatrixCoordinates"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { useZoom } from '../controllers/zoom';
import { ref, defineProps, toRef, computed, onMounted, watch, toRefs, onUnmounted, reactive } from 'vue';
import { radiansToDegrees } from '../controllers/helpers';
// import { detectTrackpad } from '../controllers/wheel';
import { WheelEventData, WheelEventState, WheelGestures } from '../controllers/wheel-gestures';

export type Transform = {
  x: number;
  y: number;
  scale: number;
  rotate: number;
};

const props = withDefaults(
  defineProps<{
    transform?: Transform;
    width: number;
    height: number;
    offset?: {
      top: number;
      right: number;
      bottom: number;
      left: number;
    };
    maxScale?: number;
    minScale?: number;
    bounds?: boolean;
    rotation?: boolean;
    mouse?: boolean;
    wheel?: boolean;
    touch?: boolean;
    gesture?: boolean;
  }>(),
  {
    transform: () => ({ x: 0, y: 0, scale: 1, rotate: 0 }),
    offset: () => ({ left: 0, top: 0, right: 0, bottom: 0 }),
    minScale: 0.5,
    maxScale: 10,
    bounds: false,
    rotation: true,
    mouse: true,
    wheel: true,
    touch: true,
    gesture: true,
  }
);
const emit = defineEmits<{
  'update:transform': [transform: Transform];
  dragGestureStart: [event: MouseEvent | TouchEvent | WheelEventData | WheelEvent];
  dragGestureEnd: [event: MouseEvent | TouchEvent | WheelEventData | WheelEvent];
}>();

console.log('!!!!!! ZOOMPINCH !!!!!!!', props);

const zoompinchRef = ref<HTMLElement>();
const canvasRef = ref<HTMLElement>();
const matrixRef = ref<HTMLElement>();

const canvasNaturalWidth = toRef(props, 'width');
const canvasNaturalHeight = toRef(props, 'height');
const offset = toRef(props, 'offset');

const rotationEnabled = computed(() => {
  return !props.bounds && props.rotation;
});

const {
  renderingTranslate,
  renderingRotate,
  renderingScale,
  composePoint,
  handleWheel,
  handleMousedown,
  handleMousemove,
  handleMouseup,
  handleTouchstart,
  handleTouchmove,
  handleTouchend,
  handleGesturestart,
  handleGesturechange,
  handleGestureend,
  applyTransform,
  rotateCanvas,
  compose,
  exposedTransform,
  calcProjectionTranslate,
  clientCoordinatesToCanvasCoordinates,
  normalizeMatrixCoordinates,
  transitionEnabled,
  transitionDuration,
  wrapperX,
  wrapperY,
  wrapperWidth,
  wrapperHeight,
} = useZoom({
  wrapperElementRef: zoompinchRef,
  canvasNaturalWidth,
  canvasNaturalHeight,
  offset,
  bounds: toRef(props, 'bounds'),
  rotationEnabled,
  minScale: toRef(props, 'minScale'),
  maxScale: toRef(props, 'maxScale'),
});

watch(exposedTransform, () => {
  if (
    props.transform.rotate !== exposedTransform.value.rotate ||
    props.transform.scale !== exposedTransform.value.scale ||
    props.transform.x !== exposedTransform.value.x ||
    props.transform.y !== exposedTransform.value.y
  ) {
    emit('update:transform', exposedTransform.value);
  }
});

watch(
  () => props.transform,
  () => {
    exposedTransform.value = {
      x: props.transform.x,
      y: props.transform.y,
      scale: props.transform.scale,
      rotate: props.transform.rotate,
    };
  },
  { deep: true }
);
onMounted(() => {
  // exposedTransform.value = {
  //   x: props.transform.x,
  //   y: props.transform.y,
  //   scale: props.transform.scale,
  //   rotate: props.transform.rotate,
  // };
});

const wheelGestures = WheelGestures();
wheelGestures.on('wheel', (wheelEventState) => {
  if (wheelEventState.isStart) {
    emit('dragGestureStart', wheelEventState.event);
  }
  handleWheel(wheelEventState.event as any);
  if (wheelEventState.isEnding) {
    emit('dragGestureEnd', wheelEventState.event);
  }
});
watch([zoompinchRef, () => props.wheel], () => {
  if (zoompinchRef.value) {
    if (props.wheel) {
      wheelGestures.observe(zoompinchRef.value);
    } else {
      wheelGestures.disconnect();
    }
  }
});
onUnmounted(() => {
  wheelGestures.disconnect();
});

// const wheelProxy = (event: WheelEvent) => {
//   if (props.wheel) {
//     handleWheel(event);
//   }
// };
let touchmoveSinceTouchstart = false;
let touchstartPoint: { x: number; y: number } | null = null;
const touchmoveMinDistance = 10;
const touchstartProxy = (event: TouchEvent) => {
  touchmoveSinceTouchstart = false;
  touchstartPoint = { x: event.touches[0].clientX, y: event.touches[0].clientY };
  if (props.touch) {
    emit('dragGestureStart', event);
    handleTouchstart(event);
  }
};
const touchmoveProxy = (event: TouchEvent) => {
  if (
    touchstartPoint &&
    Math.sqrt(
      Math.pow(event.touches[0].clientX - touchstartPoint.x, 2) + Math.pow(event.touches[0].clientY - touchstartPoint.y, 2)
    ) > touchmoveMinDistance
  ) {
    touchmoveSinceTouchstart = true;
  }
  if (props.touch) {
    handleTouchmove(event);
  }
};
const touchendProxy = (event: TouchEvent) => {
  if (props.touch) {
    handleTouchend(event);
    if (event.composedPath().includes(zoompinchRef.value!) && touchmoveSinceTouchstart) {
      emit('dragGestureEnd', event);
    }
  }
};
let mousemoveSinceMousedown = false;
let mousestartPoint: { x: number; y: number } | null = null;
const mousemoveMinDistance = 10;
const mousedownProxy = (event: MouseEvent) => {
  mousemoveSinceMousedown = false;
  mousestartPoint = { x: event.clientX, y: event.clientY };
  if (props.mouse) {
    emit('dragGestureStart', event);
    handleMousedown(event);
  }
};
const mousemoveProxy = (event: MouseEvent) => {
  if (
    mousestartPoint &&
    Math.sqrt(Math.pow(event.clientX - mousestartPoint.x, 2) + Math.pow(event.clientY - mousestartPoint.y, 2)) >
      mousemoveMinDistance
  ) {
    mousemoveSinceMousedown = true;
  }
  if (props.mouse) {
    handleMousemove(event);
  }
};
const mouseupProxy = (event: MouseEvent) => {
  if (props.mouse) {
    handleMouseup(event);
    if (event.composedPath().includes(zoompinchRef.value!) && mousemoveSinceMousedown) {
      emit('dragGestureEnd', event);
    }
  }
};
function isTouchDevice() {
  return 'ontouchstart' in window || navigator.maxTouchPoints > 0;
}
const gestureEnabled = computed(() => {
  return !isTouchDevice() && props.gesture;
});

const gesturestartProxy = (event: any) => {
  if (gestureEnabled.value) {
    handleGesturestart(event);
  }
};
const gesturechangeProxy = (event: any) => {
  if (gestureEnabled.value) {
    handleGesturechange(event);
  }
};
const gestureendProxy = (event: any) => {
  if (gestureEnabled.value) {
    handleGestureend(event);
  }
};

window.addEventListener('touchmove', touchmoveProxy);
window.addEventListener('touchend', touchendProxy);
window.addEventListener('mouseup', mouseupProxy);
window.addEventListener('mousemove', mousemoveProxy);
window.addEventListener('gesturechange', gesturechangeProxy);
window.addEventListener('gestureend', gestureendProxy);
onUnmounted(() => {
  window.removeEventListener('touchmove', touchmoveProxy);
  window.removeEventListener('touchend', touchendProxy);
  window.removeEventListener('mouseup', mouseupProxy);
  window.removeEventListener('mousemove', mousemoveProxy);
  window.removeEventListener('gesturechange', gesturechangeProxy);
  window.removeEventListener('gestureend', gestureendProxy);
});

const wrapperBounds = computed(() => {
  return {
    x: wrapperX.value,
    y: wrapperY.value,
    width: wrapperWidth.value,
    height: wrapperHeight.value,
  };
});

defineExpose({
  compose,
  composePoint,
  clientCoordinatesToCanvasCoordinates,
  normalizeMatrixCoordinates,
  applyTransform,
  calcProjectionTranslate,
  rotateCanvas,
  wrapperBounds,
  canvas: canvasRef,
  matrix: matrixRef,
});
</script>

<style scoped lang="scss">
.zoompinch {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
  touch-action: none;
  .canvas {
    position: absolute;
    left: 0px;
    top: 0px;
    width: var(--canvas-width);
    height: var(--canvas-height);
    transform-origin: 0% 0%;
    transform: translate(var(--translate-x), var(--translate-y)) scale(var(--rendering-scale)) rotate(var(--rotate));
    user-select: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    user-drag: none;
    -webkit-user-drag: none;
    -moz-user-drag: none;
    ::v-deep(img) {
      pointer-events: none;
    }
  }
  &.transition-enabled {
    .canvas {
      transition: transform var(--transition-duration);
    }
  }

  .matrix {
    left: 0px;
    top: 0px;
    position: absolute;
    width: 100%;
    pointer-events: none;
    height: 100%;
    > ::v-deep(*) {
      width: 100%;
      height: 100%;
    }
    .offset-rect {
      position: absolute;
      background-color: rgba(255, 0, 0, 0.1);
      left: var(--offset-left);
      top: var(--offset-top);
      width: calc(100% - var(--offset-left) - var(--offset-right));
      height: calc(100% - var(--offset-top) - var(--offset-bottom));
    }
  }
}
</style>
