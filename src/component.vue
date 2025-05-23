<template>
  <component 
    :is="componentType"
    ref="inputElement"
    :value="value"
    @input="updateValue"
    :placeholder="content.placeholder"
    :style="inputStyle"
    class="autoresize-input"
  />
</template>

<script>
export default {
  name: 'AutoresizeInput',
  props: {
    content: {
      type: Object,
      required: true
    }
  },
  emits: ['update:content'],
  data() {
    return {
      inputWidth: null,
      inputHeight: null
    }
  },
  computed: {
    value() {
      return this.content.value || ''
    },
    componentType() {
      return this.content.multiLine ? 'textarea' : 'input'
    },
    inputStyle() {
      const style = {
        fontSize: '16px',
        padding: '8px 12px',
        border: '1px solid #e0e0e0',
        borderRadius: '4px',
        backgroundColor: '#ffffff',
        color: '#333333',
        resize: 'none',
        overflow: 'hidden',
        transition: 'all 0.2s ease',
        boxSizing: 'border-box'
      }

      const direction = this.content.autoResizeDirection || 'horizontal'

      if (direction === 'horizontal' || direction === 'both') {
        style.width = this.inputWidth ? `${this.inputWidth}px` : '100px'
        style.minWidth = '100px'
        style.maxWidth = '100%'
      } else {
        style.width = '100%'
      }

      if ((direction === 'vertical' || direction === 'both') && this.content.multiLine) {
        style.height = this.inputHeight ? `${this.inputHeight}px` : '40px'
        style.minHeight = '40px'
        style.maxHeight = '200px'
      }

      return style
    }
  },
  mounted() {
    this.updateSize()
  },
  methods: {
    updateValue(event) {
      const newValue = event.target.value
      this.$emit('update:content', { ...this.content, value: newValue })
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    updateSize() {
      const direction = this.content.autoResizeDirection || 'horizontal'
      if (direction === 'none') return

      const el = this.$refs.inputElement
      if (!el) return

      // Create a hidden element to measure text
      const hiddenEl = document.createElement(this.componentType)
      hiddenEl.style.position = 'absolute'
      hiddenEl.style.visibility = 'hidden'
      hiddenEl.style.whiteSpace = this.content.multiLine ? 'pre-wrap' : 'pre'
      hiddenEl.style.fontSize = '16px'
      hiddenEl.style.padding = '8px 12px'
      hiddenEl.style.border = '1px solid transparent'
      hiddenEl.style.boxSizing = 'border-box'
      hiddenEl.value = this.value || this.content.placeholder || ''
      
      document.body.appendChild(hiddenEl)

      if (direction === 'horizontal' || direction === 'both') {
        this.inputWidth = hiddenEl.scrollWidth + 5
      }

      if ((direction === 'vertical' || direction === 'both') && this.content.multiLine) {
        hiddenEl.style.width = `${el.offsetWidth}px`
        this.inputHeight = hiddenEl.scrollHeight
      }

      document.body.removeChild(hiddenEl)
    }
  },
  watch: {
    'content.value'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    }
  }
}
</script>

<style scoped>
.autoresize-input:focus {
  outline: none;
  border-color: #4CAF50;
}
</style>