<template>
  <div class="remark">
    <div 
      class="resize-handler"
      @mousedown="$event => resize($event)"
    ></div>
    <textarea
      :value="remark"
      placeholder="点击输入演讲者备注"
      @input="$event => handleInput($event)"
    ></textarea>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent } from 'vue'
import { storeToRefs } from 'pinia'
import { useSlidesStore } from '@/store'

export default defineComponent({
  name: 'remark',
  emits: ['update:height'],
  props: {
    height: {
      type: Number,
      required: true,
    },
  },
  setup(props, { emit }) {
    const slidesStore = useSlidesStore()
    const { currentSlide } = storeToRefs(slidesStore)
    
    const remark = computed(() => currentSlide.value?.remark || '')

    const handleInput = (e: Event) => {
      const value = (e.target as HTMLTextAreaElement).value
      slidesStore.updateSlide({ remark: value })
    }

    const resize = (e: MouseEvent) => {
      let isMouseDown = true
      const startPageY = e.pageY
      const originHeight = props.height

      document.onmousemove = e => {
        if (!isMouseDown) return

        const currentPageY = e.pageY

        const moveY = currentPageY - startPageY
        let newHeight = -moveY + originHeight

        if (newHeight < 40) newHeight = 40
        if (newHeight > 120) newHeight = 120

        emit('update:height', newHeight)
      }

      document.onmouseup = () => {
        isMouseDown = false
        document.onmousemove = null
        document.onmouseup = null
      }
    }

    return {
      remark,
      handleInput,
      resize,
    }
  },
})
</script>

<style lang="scss" scoped>
.remark {
  position: relative;
  border-top: 1px solid $borderColor;
  background-color: $lightGray;

  textarea {
    width: 100%;
    height: 100%;
    overflow-y: auto;
    resize: none;
    border: 0;
    outline: 0;
    padding: 8px;
    font-size: 12px;
    background-color: transparent;

    @include absolute-0();
  }
}
.resize-handler {
  height: 7px;
  position: absolute;
  top: -3px;
  left: 0;
  right: 0;
  cursor: n-resize;
  z-index: 2;
}
</style>