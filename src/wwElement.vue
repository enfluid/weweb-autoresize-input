<template>
  <component 
    :is="componentType"
    ref="inputElement"
    :value="content.value || ''"
    @input="handleInput"
    @focus="handleFocus"
    @blur="handleBlur"
    :placeholder="content.placeholder || 'Enter text...'"
    :style="computedStyle"
    :class="['autoresize-input', content.inputClass]"
    :disabled="content.disabled"
    :readonly="content.readonly"
    :maxlength="content.maxLength"
    :rows="componentType === 'textarea' ? (content.rows || 1) : undefined"
  />
</template>

<script>
export default {
  name: 'AutoresizeInput',
  props: {
    content: {
      type: Object,
      required: true
    },
    uid: {
      type: String
    }
  },
  emits: ['update:content', 'trigger-event'],
  data() {
    return {
      inputWidth: null,
      inputHeight: null,
      isFocused: false,
      measurementCanvas: null,
      canvasContext: null
    }
  },
  computed: {
    componentType() {
      return this.content.multiLine ? 'textarea' : 'input'
    },
    computedStyle() {
      const style = {
        fontSize: this.content.fontSize || '16px',
        fontFamily: this.content.fontFamily || 'inherit',
        fontWeight: this.content.fontWeight || 'normal',
        lineHeight: this.content.lineHeight || 'normal',
        padding: this.content.padding || '8px 12px',
        borderRadius: this.content.borderRadius || '4px',
        borderWidth: this.content.borderWidth || '1px',
        borderStyle: 'solid',
        borderColor: this.isFocused 
          ? (this.content.focusBorderColor || '#4CAF50') 
          : (this.content.borderColor || '#e0e0e0'),
        backgroundColor: this.content.backgroundColor || '#ffffff',
        color: this.content.textColor || '#333333',
        transition: 'all 0.2s ease',
        resize: 'none',
        overflow: 'hidden',
        boxSizing: 'border-box',
        outline: 'none',
        width: '100%',
        minWidth: this.content.minWidth || '100px',
        maxWidth: this.content.maxWidth || '100%'
      }

      // Apply disabled styles
      if (this.content.disabled) {
        style.opacity = '0.6'
        style.cursor = 'not-allowed'
        style.backgroundColor = this.content.disabledBackgroundColor || '#f5f5f5'
      }

      // Apply readonly styles
      if (this.content.readonly) {
        style.cursor = 'default'
        style.backgroundColor = this.content.readonlyBackgroundColor || '#fafafa'
      }

      const direction = this.content.autoResizeDirection || 'horizontal'

      // Handle horizontal resizing
      if (direction === 'horizontal' && !this.content.multiLine) {
        if (this.inputWidth !== null) {
          style.width = `${this.inputWidth}px`
        }
      }

      // Handle vertical resizing
      if (direction === 'vertical' && this.content.multiLine) {
        style.minHeight = this.content.minHeight || '40px'
        style.maxHeight = this.content.maxHeight || '200px'
        if (this.inputHeight !== null) {
          style.height = `${this.inputHeight}px`
        }
      } else if (!this.content.multiLine) {
        // Single line inputs should have a fixed height
        style.height = this.content.inputHeight || '40px'
      }

      return style
    }
  },
  mounted() {
    this.createMeasurementCanvas()
    this.$nextTick(() => {
      this.updateSize()
    })
    
    // Add resize observer to handle container size changes
    if (window.ResizeObserver) {
      this.resizeObserver = new ResizeObserver(() => {
        this.$nextTick(() => {
          this.updateSize()
        })
      })
      
      if (this.$el.parentElement) {
        this.resizeObserver.observe(this.$el.parentElement)
      }
    }
  },
  beforeUnmount() {
    // Clean up resize observer
    if (this.resizeObserver) {
      this.resizeObserver.disconnect()
    }
  },
  methods: {
    handleInput(event) {
      const newValue = event.target.value
      
      // Update the content value
      this.$emit('update:content', { 
        ...this.content, 
        value: newValue 
      })
      
      // Trigger input event for workflows
      this.$emit('trigger-event', {
        name: 'input',
        event: {
          value: newValue,
          length: newValue.length
        }
      })
      
      // Update size after value changes
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    handleFocus(event) {
      this.isFocused = true
      this.$emit('trigger-event', {
        name: 'focus',
        event: {
          value: event.target.value
        }
      })
    },
    handleBlur(event) {
      this.isFocused = false
      this.$emit('trigger-event', {
        name: 'blur',
        event: {
          value: event.target.value
        }
      })
    },
    createMeasurementCanvas() {
      // Create a canvas for measuring text dimensions
      this.measurementCanvas = document.createElement('canvas')
      this.canvasContext = this.measurementCanvas.getContext('2d')
    },
    updateCanvasFont() {
      if (!this.canvasContext) return
      
      const fontSize = this.content.fontSize || '16px'
      const fontFamily = this.content.fontFamily || 'Arial, sans-serif'
      const fontWeight = this.content.fontWeight || 'normal'
      
      this.canvasContext.font = `${fontWeight} ${fontSize} ${fontFamily}`
    },
    updateSize() {
      const direction = this.content.autoResizeDirection || 'horizontal'
      if (direction === 'none') return

      const inputEl = this.$refs.inputElement
      if (!inputEl || !this.canvasContext) return

      this.updateCanvasFont()

      // Get the text to measure
      const textContent = this.content.value || this.content.placeholder || ''

      // Handle horizontal resizing for single-line inputs
      if (direction === 'horizontal' && !this.content.multiLine) {
        const textMetrics = this.canvasContext.measureText(textContent)
        const textWidth = textMetrics.width
        
        // Add CSS padding and extra padding for better visual appearance
        const cssPadding = this.getPaddingHorizontal()
        const extraPadding = this.content.horizontalPadding || 10
        const borderWidth = parseInt(this.content.borderWidth || '1px') * 2
        const measuredWidth = textWidth + cssPadding + extraPadding + borderWidth
        
        // Apply min/max constraints
        const minWidth = this.parseLength(this.content.minWidth || '100px')
        const maxWidth = this.parseLength(this.content.maxWidth || '100%', inputEl.parentElement)
        
        this.inputWidth = Math.min(Math.max(measuredWidth, minWidth), maxWidth)
      }

      // Handle vertical resizing for multi-line inputs
      if (direction === 'vertical' && this.content.multiLine) {
        // For textarea, we need to measure based on line breaks and text wrapping
        const lineHeight = this.getLineHeight()
        const padding = this.getPaddingVertical()
        const borderWidth = parseInt(this.content.borderWidth || '1px') * 2
        
        // Calculate number of lines including wrapped lines
        const inputWidth = inputEl.offsetWidth || 200
        const contentWidth = inputWidth - this.getPaddingHorizontal() - borderWidth
        
        let totalLines = 0
        const lines = textContent.split('\n')
        
        for (const line of lines) {
          if (line === '') {
            totalLines += 1
          } else {
            const lineMetrics = this.canvasContext.measureText(line)
            const wrappedLines = Math.ceil(lineMetrics.width / contentWidth) || 1
            totalLines += wrappedLines
          }
        }
        
        // Ensure at least one line for empty content
        if (totalLines === 0) totalLines = 1
        
        // Calculate height based on number of lines
        const measuredHeight = (totalLines * lineHeight) + padding + borderWidth
        
        // Apply min/max constraints
        const minHeight = this.parseLength(this.content.minHeight || '40px')
        const maxHeight = this.parseLength(this.content.maxHeight || '200px')
        
        this.inputHeight = Math.min(Math.max(measuredHeight, minHeight), maxHeight)
      }
    },
    parseLength(value, parentElement = null) {
      if (!value) return 0
      
      if (typeof value === 'number') return value
      
      if (value.endsWith('px')) {
        return parseInt(value)
      } else if (value.endsWith('%')) {
        const percentage = parseFloat(value) / 100
        const parentWidth = parentElement ? parentElement.offsetWidth : (this.$el?.parentElement?.offsetWidth || window.innerWidth)
        return parentWidth * percentage
      } else if (value.endsWith('vw')) {
        const vw = parseFloat(value) / 100
        return window.innerWidth * vw
      } else if (value.endsWith('vh')) {
        const vh = parseFloat(value) / 100
        return window.innerHeight * vh
      } else {
        return parseInt(value) || 0
      }
    },
    getLineHeight() {
      const lineHeight = this.content.lineHeight || '1.2'
      const fontSize = parseInt(this.content.fontSize || '16px')
      
      if (lineHeight === 'normal') {
        return fontSize * 1.2
      } else if (lineHeight.endsWith('px')) {
        return parseInt(lineHeight)
      } else {
        return fontSize * parseFloat(lineHeight)
      }
    },
    getPaddingVertical() {
      const padding = this.content.padding || '8px 12px'
      const parts = padding.split(' ')
      
      if (parts.length === 1) {
        return parseInt(parts[0]) * 2
      } else if (parts.length === 2) {
        return parseInt(parts[0]) * 2
      } else if (parts.length === 4) {
        return parseInt(parts[0]) + parseInt(parts[2])
      }
      
      return 16 // default fallback
    },
    getPaddingHorizontal() {
      const padding = this.content.padding || '8px 12px'
      const parts = padding.split(' ')
      
      if (parts.length === 1) {
        return parseInt(parts[0]) * 2
      } else if (parts.length === 2) {
        return parseInt(parts[1]) * 2
      } else if (parts.length === 4) {
        return parseInt(parts[1]) + parseInt(parts[3])
      }
      
      return 24 // default fallback
    }
  },
  watch: {
    'content.value'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    'content.fontSize'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    'content.fontFamily'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    'content.fontWeight'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    'content.padding'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    'content.multiLine'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    'content.autoResizeDirection'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    }
  }
}
</script>

<style scoped>
.autoresize-input {
  font-family: inherit;
}

.autoresize-input::placeholder {
  color: #999;
  opacity: 1;
}

.autoresize-input:disabled {
  -webkit-text-fill-color: currentColor;
  opacity: 1;
}
</style>