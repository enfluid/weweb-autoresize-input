<template>
  <div 
    class="autoresize-wrapper"
    :class="{ 
      'horizontal-resize': effectiveAutoResizeDirection === 'horizontal',
      'vertical-resize': effectiveAutoResizeDirection === 'vertical' 
    }"
  >
    <!-- Hidden span for measuring text width -->
    <span 
      v-if="effectiveAutoResizeDirection === 'horizontal'"
      class="size-detector"
      :style="sizeDetectorStyle"
      ref="sizeDetector"
    >{{ displayText }}</span>
    
    <component 
      :is="componentType"
      ref="inputElement"
      :value="content.value || ''"
      @input="handleInput"
      @focus="handleFocus"
      @blur="handleBlur"
      :placeholder="content.placeholder || 'Enter text...'"
      :style="inputStyle"
      :class="['autoresize-input', content.inputClass]"
      :disabled="content.disabled"
      :readonly="content.readonly"
      :maxlength="content.maxLength"
      :rows="componentType === 'textarea' ? (content.rows || 1) : undefined"
    />
  </div>
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
      isFocused: false,
      computedWidth: this.content.minWidth || '100px'
    }
  },
  computed: {
    effectiveAutoResizeDirection() {
      // Automatically determine resize direction based on multiLine setting
      if (this.content.autoResizeDirection === 'none') return 'none'
      if (this.content.autoResizeDirection === 'horizontal') return 'horizontal'
      if (this.content.autoResizeDirection === 'vertical') return 'vertical'
      return 'none'
    },
    componentType() {
      // Use textarea for vertical resize, input for horizontal
      if (this.effectiveAutoResizeDirection === 'vertical') return 'textarea'
      if (this.effectiveAutoResizeDirection === 'horizontal') return 'input'
      return this.content.multiLine ? 'textarea' : 'input'
    },
    displayText() {
      // Text to display in the size detector
      return this.content.value || this.content.placeholder || ''
    },
    sizeDetectorStyle() {
      // Style for the hidden size detector span
      return {
        position: 'absolute',
        visibility: 'hidden',
        height: 'auto',
        width: 'auto',
        whiteSpace: 'pre',
        fontSize: this.content.fontSize || '16px',
        fontFamily: this.content.fontFamily || 'inherit',
        fontWeight: this.content.fontWeight || 'normal',
        lineHeight: this.content.lineHeight || 'normal',
        padding: this.content.padding || '8px 12px',
        border: `${this.content.borderWidth || '1px'} solid transparent`,
        boxSizing: 'border-box',
        overflow: 'hidden'
      }
    },
    inputStyle() {
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
        transition: 'border-color 0.2s ease',
        resize: 'none',
        overflow: 'hidden',
        boxSizing: 'border-box',
        outline: 'none',
        width: '100%',
        minWidth: this.content.minWidth || '0px',
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

      // Handle horizontal auto-resizing
      if (this.effectiveAutoResizeDirection === 'horizontal') {
        style.width = this.computedWidth
        style.minWidth = this.content.minWidth || '0px'
        style.maxWidth = this.content.maxWidth || '100%'
      } else {
        style.width = '100%'
      }

      // Handle vertical auto-resizing for textareas
      if (this.effectiveAutoResizeDirection === 'vertical') {
        style.minHeight = this.content.minHeight || '40px'
        style.maxHeight = this.content.maxHeight || '200px'
        style.resize = 'none'
        style.overflow = 'auto'
        style.overflowX = 'hidden'
      } else if (this.effectiveAutoResizeDirection === 'horizontal') {
        style.height = this.content.inputHeight || '40px'
      } else if (!this.content.multiLine) {
        style.height = this.content.inputHeight || '40px'
      }

      return style
    }
  },
  mounted() {
    // Handle initial sizing with a delay to ensure DOM is ready in preview mode
    this.$nextTick(() => {
      // Additional timeout for WeWeb preview mode compatibility
      setTimeout(() => {
        if (this.effectiveAutoResizeDirection === 'vertical') {
          this.adjustTextareaHeight()
        } else if (this.effectiveAutoResizeDirection === 'horizontal') {
          this.updateInputWidth()
        }
      }, 50)
    })
  },
  watch: {
    'content.value'() {
      this.$nextTick(() => {
        if (this.effectiveAutoResizeDirection === 'vertical') {
          this.adjustTextareaHeight()
        } else if (this.effectiveAutoResizeDirection === 'horizontal') {
          this.updateInputWidth()
        }
      })
    },
    'effectiveAutoResizeDirection'() {
      this.$nextTick(() => {
        if (this.effectiveAutoResizeDirection === 'vertical') {
          this.adjustTextareaHeight()
        } else if (this.effectiveAutoResizeDirection === 'horizontal') {
          this.updateInputWidth()
        }
      })
    },
    displayText() {
      // Watch for changes in the display text
      if (this.effectiveAutoResizeDirection === 'horizontal') {
        this.$nextTick(() => {
          this.updateInputWidth()
        })
      }
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
      
      // Handle auto-resize
      this.$nextTick(() => {
        if (this.effectiveAutoResizeDirection === 'vertical') {
          this.adjustTextareaHeight()
        } else if (this.effectiveAutoResizeDirection === 'horizontal') {
          this.updateInputWidth()
        }
      })
    },
    updateInputWidth() {
      if (this.effectiveAutoResizeDirection !== 'horizontal') return
      
      // Small delay to ensure DOM is ready
      setTimeout(() => {
        const detector = this.$refs.sizeDetector
        if (!detector) {
          console.warn('Size detector not found')
          this.computedWidth = 'auto'
          return
        }
        
        // Force layout calculation
        const rect = detector.getBoundingClientRect()
        const width = rect.width || detector.offsetWidth || detector.scrollWidth
        
        if (width === 0) {
          console.warn('Size detector has 0 width')
          // Try alternative measurement
          const tempSpan = document.createElement('span')
          Object.assign(tempSpan.style, this.sizeDetectorStyle)
          tempSpan.style.position = 'absolute'
          tempSpan.style.visibility = 'hidden'
          tempSpan.style.height = 'auto'
          tempSpan.style.width = 'auto'
          tempSpan.textContent = this.displayText
          document.body.appendChild(tempSpan)
          const measuredWidth = tempSpan.offsetWidth
          document.body.removeChild(tempSpan)
          
          if (measuredWidth > 0) {
            const extraPadding = parseInt(this.content.horizontalPadding) || 5
            const totalWidth = measuredWidth + extraPadding
            
            // Apply constraints
            const minWidth = parseInt(this.content.minWidth) || 0
            const maxWidth = parseInt(this.content.maxWidth) || 9999
            
            const finalWidth = Math.min(Math.max(totalWidth, minWidth), maxWidth)
            this.computedWidth = `${finalWidth}px`
          } else {
            this.computedWidth = 'auto'
          }
          return
        }
        
        const extraPadding = parseInt(this.content.horizontalPadding) || 5
        const totalWidth = width + extraPadding
        
        // Apply constraints
        const minWidth = parseInt(this.content.minWidth) || 0
        const maxWidth = parseInt(this.content.maxWidth) || 9999
        
        const finalWidth = Math.min(Math.max(totalWidth, minWidth), maxWidth)
        this.computedWidth = `${finalWidth}px`
      }, 10)
    },
    adjustTextareaHeight() {
      const textarea = this.$refs.inputElement
      if (!textarea || this.effectiveAutoResizeDirection !== 'vertical') return
      
      // Reset height to auto to get the correct scrollHeight
      textarea.style.height = 'auto'
      
      // Calculate the new height
      const minHeight = parseInt(this.content.minHeight) || 40
      const maxHeight = parseInt(this.content.maxHeight) || 200
      const scrollHeight = textarea.scrollHeight
      
      // Set the new height within bounds
      const newHeight = Math.min(Math.max(scrollHeight, minHeight), maxHeight)
      textarea.style.height = `${newHeight}px`
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
    calculatePadding() {
      const padding = this.content.padding || '8px 12px'
      const parts = padding.split(' ').map(p => parseInt(p) || 0)
      
      if (parts.length === 1) {
        return { vertical: parts[0] * 2, horizontal: parts[0] * 2 }
      } else if (parts.length === 2) {
        return { vertical: parts[0] * 2, horizontal: parts[1] * 2 }
      } else if (parts.length === 4) {
        return { vertical: parts[0] + parts[2], horizontal: parts[1] + parts[3] }
      }
      
      return { vertical: 16, horizontal: 24 }
    }
  }
}
</script>

<style scoped>
.autoresize-wrapper {
  display: inline-block;
  width: 100%;
  position: relative;
}

/* Horizontal auto-sizing */
.autoresize-wrapper.horizontal-resize {
  display: inline-block;
  width: auto;
  position: relative;
}

.autoresize-wrapper.horizontal-resize .autoresize-input {
  display: inline-block;
}

.autoresize-wrapper.horizontal-resize .size-detector {
  position: fixed;
  top: 0;
  left: 0;
  visibility: hidden;
  height: auto;
  width: auto;
  white-space: pre;
  pointer-events: none;
  z-index: -1;
}

.autoresize-wrapper.vertical-resize {
  display: block;
  width: 100%;
}

.autoresize-input {
  font-family: inherit;
  box-sizing: border-box;
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