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
      measurementDiv: null
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
      if ((direction === 'horizontal' || direction === 'both') && !this.content.multiLine) {
        if (this.inputWidth !== null) {
          style.width = `${this.inputWidth}px`
        }
      }

      // Handle vertical resizing
      if ((direction === 'vertical' || direction === 'both') && this.content.multiLine) {
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
    this.createMeasurementDiv()
    this.$nextTick(() => {
      this.updateSize()
    })
  },
  beforeUnmount() {
    if (this.measurementDiv && this.measurementDiv.parentNode) {
      this.measurementDiv.parentNode.removeChild(this.measurementDiv)
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
    createMeasurementDiv() {
      // Create a hidden div for measuring text dimensions
      this.measurementDiv = document.createElement('div')
      this.measurementDiv.style.position = 'absolute'
      this.measurementDiv.style.top = '-9999px'
      this.measurementDiv.style.left = '-9999px'
      this.measurementDiv.style.visibility = 'hidden'
      this.measurementDiv.style.whiteSpace = this.content.multiLine ? 'pre-wrap' : 'pre'
      this.measurementDiv.style.wordWrap = 'break-word'
      this.measurementDiv.style.overflow = 'hidden'
      
      // Copy text styles
      this.updateMeasurementDivStyles()
      
      document.body.appendChild(this.measurementDiv)
    },
    updateMeasurementDivStyles() {
      if (!this.measurementDiv) return
      
      // Copy all relevant styles for accurate measurement
      this.measurementDiv.style.fontSize = this.content.fontSize || '16px'
      this.measurementDiv.style.fontFamily = this.content.fontFamily || 'inherit'
      this.measurementDiv.style.fontWeight = this.content.fontWeight || 'normal'
      this.measurementDiv.style.lineHeight = this.content.lineHeight || 'normal'
      this.measurementDiv.style.padding = this.content.padding || '8px 12px'
      this.measurementDiv.style.borderWidth = this.content.borderWidth || '1px'
      this.measurementDiv.style.borderStyle = 'solid'
      this.measurementDiv.style.boxSizing = 'border-box'
    },
    updateSize() {
      const direction = this.content.autoResizeDirection || 'horizontal'
      if (direction === 'none') return

      const inputEl = this.$refs.inputElement
      if (!inputEl || !this.measurementDiv) return

      // Update measurement div content
      const textContent = this.content.value || this.content.placeholder || ''
      this.measurementDiv.textContent = textContent

      // For empty input, use placeholder for width calculation
      if (!this.content.value && this.content.placeholder) {
        this.measurementDiv.textContent = this.content.placeholder
      }

      // Handle horizontal resizing
      if ((direction === 'horizontal' || direction === 'both') && !this.content.multiLine) {
        // Add some padding to prevent text cutoff
        const horizontalPadding = this.content.horizontalPadding || 5
        const measuredWidth = this.measurementDiv.scrollWidth + horizontalPadding
        
        // Apply min/max constraints
        const minWidth = parseInt(this.content.minWidth || '100')
        const maxWidth = this.getMaxWidthInPixels()
        
        this.inputWidth = Math.min(Math.max(measuredWidth, minWidth), maxWidth)
      }

      // Handle vertical resizing
      if ((direction === 'vertical' || direction === 'both') && this.content.multiLine) {
        // Set width for proper text wrapping calculation
        const inputWidth = inputEl.offsetWidth
        this.measurementDiv.style.width = `${inputWidth}px`
        
        // Measure height
        const measuredHeight = this.measurementDiv.scrollHeight
        
        // Apply min/max constraints
        const minHeight = parseInt(this.content.minHeight || '40')
        const maxHeight = parseInt(this.content.maxHeight || '200')
        
        this.inputHeight = Math.min(Math.max(measuredHeight, minHeight), maxHeight)
      }
    },
    getMaxWidthInPixels() {
      // Convert percentage or other units to pixels
      const maxWidth = this.content.maxWidth || '100%'
      
      if (maxWidth.endsWith('%')) {
        const percentage = parseFloat(maxWidth) / 100
        const parentWidth = this.$el.parentElement ? this.$el.parentElement.offsetWidth : window.innerWidth
        return parentWidth * percentage
      } else if (maxWidth.endsWith('vw')) {
        const vw = parseFloat(maxWidth) / 100
        return window.innerWidth * vw
      } else {
        return parseInt(maxWidth) || 9999
      }
    }
  },
  watch: {
    'content.value'() {
      this.$nextTick(() => {
        this.updateSize()
      })
    },
    'content.fontSize'() {
      this.updateMeasurementDivStyles()
      this.updateSize()
    },
    'content.fontFamily'() {
      this.updateMeasurementDivStyles()
      this.updateSize()
    },
    'content.fontWeight'() {
      this.updateMeasurementDivStyles()
      this.updateSize()
    },
    'content.padding'() {
      this.updateMeasurementDivStyles()
      this.updateSize()
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