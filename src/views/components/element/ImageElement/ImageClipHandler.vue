<template>
  <div 
    class="image-clip-handler" 
    :style="clipWrapperPositionStyle" 
    v-click-outside="handleClip"
  >
    <img 
      class="bottom-img" 
      :src="src" 
      :draggable="false" 
      alt="" 
      :style="bottomImgPositionStyle" 
    />

    <div 
      class="top-image-content" 
      :style="{
        ...topImgWrapperPositionStyle,
        clipPath,
      }"
    >
      <img 
        class="top-img" 
        :src="src" 
        :draggable="false" 
        alt="" 
        :style="topImgPositionStyle" 
      />
    </div>

    <div 
      class="operate" 
      :style="topImgWrapperPositionStyle" 
      @mousedown.stop="$event => moveClipRange($event)"
    >
      <div 
        :class="['clip-point', point, rotateClassName]"
        v-for="point in ['left-top', 'right-top', 'left-bottom', 'right-bottom']" 
        :key="point" 
        @mousedown.stop="$event => scaleClipRange($event, point)"
      >
        <svg width="16" height="16" fill="#fff" stroke="#333">
          <path
            stroke-width="0.3" 
            shape-rendering="crispEdges"
            d="M 16 0 L 0 0 L 0 16 L 4 16 L 4 4 L 16 4 L 16 0 Z"
          ></path>
        </svg>
      </div>
      <div 
        :class="['clip-point', point, rotateClassName]"
        v-for="point in ['top', 'bottom', 'left', 'right']" 
        :key="point" 
        @mousedown.stop="$event => scaleClipRange($event, point)"
      >
        <svg width="16" height="16" fill="#fff" stroke="#333">
          <path
            stroke-width="0.3" 
            shape-rendering="crispEdges"
            d="M 16 0 L 0 0 L 0 4 L 16 4 Z"
          ></path>
        </svg>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, onMounted, onUnmounted, PropType, reactive, ref } from 'vue'
import { storeToRefs } from 'pinia'
import { useMainStore, useKeyboardStore } from '@/store'
import { KEYS } from '@/configs/hotkey'
import { ImageClipData, ImageClipDataRange, ImageClipedEmitData, OperateResizeHandlers } from '@/types/edit'

export default defineComponent({
  name: 'image-clip-handler',
  emits: ['clip'],
  props: {
    src: {
      type: String,
      required: true,
    },
    clipData: {
      type: Object as PropType<ImageClipData>,
    },
    clipPath: {
      type: String,
      required: true,
    },
    width: {
      type: Number,
      required: true,
    },
    height: {
      type: Number,
      required: true,
    },
    top: {
      type: Number,
      required: true,
    },
    left: {
      type: Number,
      required: true,
    },
    rotate: {
      type: Number,
      required: true,
    },
  },
  setup(props, { emit }) {
    const { canvasScale } = storeToRefs(useMainStore())
    const { ctrlOrShiftKeyActive } = storeToRefs(useKeyboardStore())

    const clipWrapperPositionStyle = reactive({
      top: '0',
      left: '0',
    })
    const isSettingClipRange = ref(false)
    const currentRange = ref<ImageClipDataRange | null>(null)

    // 获取裁剪区域信息（裁剪区域占原图的宽高比例，处在原图中的位置）
    const getClipDataTransformInfo = () => {
      const [start, end] = props.clipData ? props.clipData.range : [[0, 0], [100, 100]]

      const widthScale = (end[0] - start[0]) / 100
      const heightScale = (end[1] - start[1]) / 100
      const left = start[0] / widthScale
      const top = start[1] / heightScale

      return { widthScale, heightScale, left, top }
    }
    
    // 底层图片位置大小（遮罩区域图片）
    const imgPosition = computed(() => {
      const { widthScale, heightScale, left, top } = getClipDataTransformInfo()
      return {
        left: -left,
        top: -top,
        width: 100 / widthScale,
        height: 100 / heightScale,
      }
    })

    // 底层图片位置大小样式（遮罩区域图片）
    const bottomImgPositionStyle = computed(() => {
      return {
        top: imgPosition.value.top + '%',
        left: imgPosition.value.left + '%',
        width: imgPosition.value.width + '%',
        height: imgPosition.value.height + '%',
      }
    })

    // 顶层图片容器位置大小（裁剪高亮区域）
    const topImgWrapperPosition = reactive({
      top: 0,
      left: 0,
      width: 0,
      height: 0,
    })

    // 顶层图片容器位置大小样式（裁剪高亮区域）
    const topImgWrapperPositionStyle = computed(() => {
      return {
        top: topImgWrapperPosition.top + '%',
        left: topImgWrapperPosition.left + '%',
        width: topImgWrapperPosition.width + '%',
        height: topImgWrapperPosition.height + '%',
      }
    })

    // 顶层图片位置大小样式（裁剪区域图片）
    const topImgPositionStyle = computed(() => {
      const bottomWidth = imgPosition.value.width
      const bottomHeight = imgPosition.value.height
      
      const topLeft = topImgWrapperPosition.left
      const topTop = topImgWrapperPosition.top
      const topWidth = topImgWrapperPosition.width
      const topHeight = topImgWrapperPosition.height
      
      return {
        left: -topLeft * (100 / topWidth) + '%',
        top: -topTop * (100 / topHeight) + '%',
        width: bottomWidth / topWidth * 100 + '%',
        height: bottomHeight / topHeight * 100 + '%',
      }
    })

    // 初始化裁剪位置信息
    const initClipPosition = () => {
      const { left, top } = getClipDataTransformInfo()
      topImgWrapperPosition.left = left
      topImgWrapperPosition.top = top
      topImgWrapperPosition.width = 100
      topImgWrapperPosition.height = 100
      
      clipWrapperPositionStyle.top = -top + '%'
      clipWrapperPositionStyle.left = -left + '%'
    }

    // 执行裁剪：计算裁剪后的图片位置大小和裁剪信息，并将数据同步出去
    const handleClip = () => {
      if (isSettingClipRange.value) return

      if (!currentRange.value) {
        emit('clip', null)
        return
      }

      const { left, top } = getClipDataTransformInfo()

      const position = {
        left: (topImgWrapperPosition.left - left) / 100 * props.width,
        top: (topImgWrapperPosition.top - top) / 100 * props.height,
        width: (topImgWrapperPosition.width - 100) / 100 * props.width,
        height: (topImgWrapperPosition.height - 100) / 100 * props.height,
      }

      const clipedEmitData: ImageClipedEmitData = {
        range: currentRange.value,
        position,
      }
      emit('clip', clipedEmitData)
    }

    // 快捷键监听：回车确认裁剪
    const keyboardListener = (e: KeyboardEvent) => {
      const key = e.key.toUpperCase()
      if (key === KEYS.ENTER) handleClip()
    }

    onMounted(() => {
      initClipPosition()
      document.addEventListener('keydown', keyboardListener)
    })
    onUnmounted(() => {
      document.removeEventListener('keydown', keyboardListener)
    })

    // 计算并更新裁剪区域范围数据
    const updateRange = () => {
      const retPosition = {
        left: parseInt(topImgPositionStyle.value.left),
        top: parseInt(topImgPositionStyle.value.top),
        width: parseInt(topImgPositionStyle.value.width),
        height: parseInt(topImgPositionStyle.value.height),
      }

      const widthScale = 100 / retPosition.width
      const heightScale = 100 / retPosition.height

      const start: [number, number] = [
        -retPosition.left * widthScale,
        -retPosition.top * heightScale,
      ]
      const end: [number, number] = [
        widthScale * 100 + start[0],
        heightScale * 100 + start[1],
      ]

      currentRange.value = [start, end]
    }

    // 移动裁剪区域
    const moveClipRange = (e: MouseEvent) => {
      isSettingClipRange.value = true
      let isMouseDown = true

      const startPageX = e.pageX
      const startPageY = e.pageY
      const bottomPosition = imgPosition.value
      const originPositopn = {
        left: topImgWrapperPosition.left,
        top: topImgWrapperPosition.top,
        width: topImgWrapperPosition.width,
        height: topImgWrapperPosition.height,
      }

      document.onmousemove = e => {
        if (!isMouseDown) return

        const currentPageX = e.pageX
        const currentPageY = e.pageY

        let moveX = (currentPageX - startPageX) / canvasScale.value / props.width * 100
        let moveY = (currentPageY - startPageY) / canvasScale.value / props.height * 100

        if (props.rotate > 45 && props.rotate < 135) {
          moveX = (currentPageY - startPageY) / canvasScale.value / props.width * 100
          moveY = -(currentPageX - startPageX) / canvasScale.value / props.height * 100
        }
        if ((props.rotate >= 135 && props.rotate <= 180) || (props.rotate >= -180 && props.rotate <= -135)) {
          moveX = -moveX
          moveY = -moveY
        }
        if (props.rotate > -135 && props.rotate < -45) {
          moveX = -(currentPageY - startPageY) / canvasScale.value / props.width * 100
          moveY = (currentPageX - startPageX) / canvasScale.value / props.height * 100
        }

        let targetLeft = originPositopn.left + moveX
        let targetTop = originPositopn.top + moveY

        if (targetLeft < 0) targetLeft = 0
        else if (targetLeft + originPositopn.width > bottomPosition.width) {
          targetLeft = bottomPosition.width - originPositopn.width
        }
        if (targetTop < 0) targetTop = 0
        else if (targetTop + originPositopn.height > bottomPosition.height) {
          targetTop = bottomPosition.height - originPositopn.height
        }
        
        topImgWrapperPosition.left = targetLeft
        topImgWrapperPosition.top = targetTop
      }

      document.onmouseup = () => {
        isMouseDown = false
        document.onmousemove = null
        document.onmouseup = null

        updateRange()

        setTimeout(() => {
          isSettingClipRange.value = false
        }, 0)
      }
    }

    // 缩放裁剪区域
    const scaleClipRange = (e: MouseEvent, type: OperateResizeHandlers) => {
      isSettingClipRange.value = true
      let isMouseDown = true

      const minWidth = 50 / props.width * 100
      const minHeight = 50 / props.height * 100
      
      const startPageX = e.pageX
      const startPageY = e.pageY
      const bottomPosition = imgPosition.value
      const originPositopn = {
        left: topImgWrapperPosition.left,
        top: topImgWrapperPosition.top,
        width: topImgWrapperPosition.width,
        height: topImgWrapperPosition.height,
      }

      const aspectRatio = topImgWrapperPosition.width / topImgWrapperPosition.height

      document.onmousemove = e => {
        if (!isMouseDown) return

        const currentPageX = e.pageX
        const currentPageY = e.pageY

        let moveX = (currentPageX - startPageX) / canvasScale.value / props.width * 100
        let moveY = (currentPageY - startPageY) / canvasScale.value / props.height * 100

        if (props.rotate > 45 && props.rotate < 135) {
          moveX = (currentPageY - startPageY) / canvasScale.value / props.width * 100
          moveY = -(currentPageX - startPageX) / canvasScale.value / props.height * 100
        }
        if ((props.rotate >= 135 && props.rotate <= 180) || (props.rotate >= -180 && props.rotate <= -135)) {
          moveX = -moveX
          moveY = -moveY
        }
        if (props.rotate > -135 && props.rotate < -45) {
          moveX = -(currentPageY - startPageY) / canvasScale.value / props.width * 100
          moveY = (currentPageX - startPageX) / canvasScale.value / props.height * 100
        }

        if (ctrlOrShiftKeyActive.value) {
          if (type === OperateResizeHandlers.RIGHT_BOTTOM || type === OperateResizeHandlers.LEFT_TOP) moveY = moveX / aspectRatio
          if (type === OperateResizeHandlers.LEFT_BOTTOM || type === OperateResizeHandlers.RIGHT_TOP) moveY = -moveX / aspectRatio
        }

        let targetLeft, targetTop, targetWidth, targetHeight

        if (type === OperateResizeHandlers.LEFT_TOP) {
          if (originPositopn.left + moveX < 0) {
            moveX = -originPositopn.left
          }
          if (originPositopn.top + moveY < 0) {
            moveY = -originPositopn.top
          }
          if (originPositopn.width - moveX < minWidth) {
            moveX = originPositopn.width - minWidth
          }
          if (originPositopn.height - moveY < minHeight) {
            moveY = originPositopn.height - minHeight
          }
          targetWidth = originPositopn.width - moveX
          targetHeight = originPositopn.height - moveY
          targetLeft = originPositopn.left + moveX
          targetTop = originPositopn.top + moveY
        }
        else if (type === OperateResizeHandlers.RIGHT_TOP) {
          if (originPositopn.left + originPositopn.width + moveX > bottomPosition.width) {
            moveX = bottomPosition.width - (originPositopn.left + originPositopn.width)
          }
          if (originPositopn.top + moveY < 0) {
            moveY = -originPositopn.top
          }
          if (originPositopn.width + moveX < minWidth) {
            moveX = minWidth - originPositopn.width
          }
          if (originPositopn.height - moveY < minHeight) {
            moveY = originPositopn.height - minHeight
          }
          targetWidth = originPositopn.width + moveX
          targetHeight = originPositopn.height - moveY
          targetLeft = originPositopn.left
          targetTop = originPositopn.top + moveY
        }
        else if (type === OperateResizeHandlers.LEFT_BOTTOM) {
          if (originPositopn.left + moveX < 0) {
            moveX = -originPositopn.left
          }
          if (originPositopn.top + originPositopn.height + moveY > bottomPosition.height) {
            moveY = bottomPosition.height - (originPositopn.top + originPositopn.height)
          }
          if (originPositopn.width - moveX < minWidth) {
            moveX = originPositopn.width - minWidth
          }
          if (originPositopn.height + moveY < minHeight) {
            moveY = minHeight - originPositopn.height
          }
          targetWidth = originPositopn.width - moveX
          targetHeight = originPositopn.height + moveY
          targetLeft = originPositopn.left + moveX
          targetTop = originPositopn.top
        }
        else if (type === OperateResizeHandlers.RIGHT_BOTTOM) {
          if (originPositopn.left + originPositopn.width + moveX > bottomPosition.width) {
            moveX = bottomPosition.width - (originPositopn.left + originPositopn.width)
          }
          if (originPositopn.top + originPositopn.height + moveY > bottomPosition.height) {
            moveY = bottomPosition.height - (originPositopn.top + originPositopn.height)
          }
          if (originPositopn.width + moveX < minWidth) {
            moveX = minWidth - originPositopn.width
          }
          if (originPositopn.height + moveY < minHeight) {
            moveY = minHeight - originPositopn.height
          }
          targetWidth = originPositopn.width + moveX
          targetHeight = originPositopn.height + moveY
          targetLeft = originPositopn.left
          targetTop = originPositopn.top
        }
        else if (type === OperateResizeHandlers.TOP) {
          if (originPositopn.top + moveY < 0) {
            moveY = -originPositopn.top
          }
          if (originPositopn.height - moveY < minHeight) {
            moveY = originPositopn.height - minHeight
          }
          targetWidth = originPositopn.width
          targetHeight = originPositopn.height - moveY
          targetLeft = originPositopn.left
          targetTop = originPositopn.top + moveY
        }
        else if (type === OperateResizeHandlers.BOTTOM) {
          if (originPositopn.top + originPositopn.height + moveY > bottomPosition.height) {
            moveY = bottomPosition.height - (originPositopn.top + originPositopn.height)
          }
          if (originPositopn.height + moveY < minHeight) {
            moveY = minHeight - originPositopn.height
          }
          targetWidth = originPositopn.width
          targetHeight = originPositopn.height + moveY
          targetLeft = originPositopn.left
          targetTop = originPositopn.top
        }
        else if (type === OperateResizeHandlers.LEFT) {
          if (originPositopn.left + moveX < 0) {
            moveX = -originPositopn.left
          }
          if (originPositopn.width - moveX < minWidth) {
            moveX = originPositopn.width - minWidth
          }
          targetWidth = originPositopn.width - moveX
          targetHeight = originPositopn.height
          targetLeft = originPositopn.left + moveX
          targetTop = originPositopn.top
        }
        else {
          if (originPositopn.left + originPositopn.width + moveX > bottomPosition.width) {
            moveX = bottomPosition.width - (originPositopn.left + originPositopn.width)
          }
          if (originPositopn.width + moveX < minWidth) {
            moveX = minWidth - originPositopn.width
          }
          targetHeight = originPositopn.height
          targetWidth = originPositopn.width + moveX
          targetLeft = originPositopn.left
          targetTop = originPositopn.top
        }
        
        topImgWrapperPosition.left = targetLeft
        topImgWrapperPosition.top = targetTop
        topImgWrapperPosition.width = targetWidth
        topImgWrapperPosition.height = targetHeight
      }

      document.onmouseup = () => {
        isMouseDown = false
        document.onmousemove = null
        document.onmouseup = null

        updateRange()

        setTimeout(() => isSettingClipRange.value = false, 0)
      }
    }

    const rotateClassName = computed(() => {
      const prefix = 'rotate-'
      const rotate = props.rotate
      if (rotate > -22.5 && rotate <= 22.5) return prefix + 0
      else if (rotate > 22.5 && rotate <= 67.5) return prefix + 45
      else if (rotate > 67.5 && rotate <= 112.5) return prefix + 90
      else if (rotate > 112.5 && rotate <= 157.5) return prefix + 135
      else if (rotate > 157.5 || rotate <= -157.5) return prefix + 0
      else if (rotate > -157.5 && rotate <= -112.5) return prefix + 45
      else if (rotate > -112.5 && rotate <= -67.5) return prefix + 90
      else if (rotate > -67.5 && rotate <= -22.5) return prefix + 135
      return prefix + 0
    })

    return {
      clipWrapperPositionStyle,
      bottomImgPositionStyle,
      topImgWrapperPositionStyle,
      topImgPositionStyle,
      rotateClassName,
      handleClip,
      moveClipRange,
      scaleClipRange,
    }
  },
})
</script>

<style lang="scss" scoped>
.image-clip-handler {
  width: 100%;
  height: 100%;
  position: relative;

  .bottom-img {
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    opacity: .5;
  }

  img {
    width: 100%;
    height: 100%;
  }

  .top-image-content {
    position: absolute;
    overflow: hidden;

    img {
      position: absolute;
    }
  }
}

.operate {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  cursor: move;
}

.clip-point {
  position: absolute;
  width: 16px;
  height: 16px;

  svg {
    overflow: visible;
  }

  &.left-top {
    left: 0;
    top: 0;
  }
  &.right-top {
    left: 100%;
    top: 0;
    transform: rotate(90deg);
    transform-origin: 0 0;
  }
  &.left-bottom {
    left: 0;
    top: 100%;
    transform: rotate(-90deg);
    transform-origin: 0 0;
  }
  &.right-bottom {
    left: 100%;
    top: 100%;
    transform: rotate(180deg);
    transform-origin: 0 0;
  }
  &.top {
    left: 50%;
    top: 0;
    margin-left: -8px;
  }
  &.bottom {
    left: 50%;
    bottom: 0;
    margin-left: -8px;
    transform: rotate(180deg);
  }
  &.left {
    left: 0;
    top: 50%;
    margin-top: -8px;
    transform: rotate(-90deg);
  }
  &.right {
    right: 0;
    top: 50%;
    margin-top: -8px;
    transform: rotate(90deg);
  }

  &.left-top.rotate-0,
  &.right-bottom.rotate-0,
  &.left.rotate-45,
  &.right.rotate-45,
  &.left-bottom.rotate-90,
  &.right-top.rotate-90,
  &.top.rotate-135,
  &.bottom.rotate-135 {
    cursor: nwse-resize;
  }
  &.top.rotate-0,
  &.bottom.rotate-0,
  &.left-top.rotate-45,
  &.right-bottom.rotate-45,
  &.left.rotate-90,
  &.right.rotate-90,
  &.left-bottom.rotate-135,
  &.right-top.rotate-135 {
    cursor: ns-resize;
  }
  &.left-bottom.rotate-0,
  &.right-top.rotate-0,
  &.top.rotate-45,
  &.bottom.rotate-45,
  &.left-top.rotate-90,
  &.right-bottom.rotate-90,
  &.left.rotate-135,
  &.right.rotate-135 {
    cursor: nesw-resize;
  }
  &.left.rotate-0,
  &.right.rotate-0,
  &.left-bottom.rotate-45,
  &.right-top.rotate-45,
  &.top.rotate-90,
  &.bottom.rotate-90,
  &.left-top.rotate-135,
  &.right-bottom.rotate-135 {
    cursor: ew-resize;
  }
}
</style>