<template>
  <div 
    class="autoresize-wrapper"
    :class="{ 
      'horizontal-resize': effectiveAutoResizeDirection === 'horizontal',
      'vertical-resize': effectiveAutoResizeDirection === 'vertical' 
    }"
    :data-value="content.value || content.placeholder || ''"
  >
    <div 
      v-if="effectiveAutoResizeDirection === 'horizontal'"
      class="size-detector"
      :style="sizeDetectorStyle"
      ref="sizeDetector"
    >{{ content.value || content.placeholder || '' }}</div>
    
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
      isFocused: false
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
    sizeDetectorStyle() {
      return {
        fontSize: this.content.fontSize || '16px',
        fontFamily: this.content.fontFamily || 'inherit',
        fontWeight: this.content.fontWeight || 'normal',
        lineHeight: this.content.lineHeight || 'normal',
        padding: this.content.padding || '8px 12px',
        borderWidth: this.content.borderWidth || '1px',
        borderStyle: 'solid',
        borderColor: 'transparent',
        boxSizing: 'border-box',
        whiteSpace: 'pre',
        visibility: 'hidden',
        position: 'absolute',
        top: '0',
        left: '0',
        zIndex: '-1',
        minWidth: this.content.minWidth || '100px',
        maxWidth: this.content.maxWidth || '100%',
        pointerEvents: 'none'
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

      // Handle horizontal auto-resizing with CSS grid approach
      if (this.effectiveAutoResizeDirection === 'horizontal') {
        // Let CSS handle the sizing through grid
        style.gridArea = '1 / 1'
        style.width = 'auto'
        style.minWidth = this.content.minWidth || '100px'
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
    if (this.effectiveAutoResizeDirection === 'vertical') {
      this.$nextTick(() => {
        this.adjustTextareaHeight()
      })
    }
  },
  watch: {
    'content.value'() {
      if (this.effectiveAutoResizeDirection === 'vertical') {
        this.$nextTick(() => {
          this.adjustTextareaHeight()
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
      
      // Handle vertical auto-resize
      if (this.effectiveAutoResizeDirection === 'vertical') {
        this.$nextTick(() => {
          this.adjustTextareaHeight()
        })
      }
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

/* CSS Grid approach for horizontal auto-sizing */
.autoresize-wrapper.horizontal-resize {
  display: inline-grid;
  align-items: center;
  width: auto;
  min-width: 100px;
  max-width: 100%;
}

.autoresize-wrapper.horizontal-resize::after {
  content: attr(data-value) ' ';
  visibility: hidden;
  white-space: pre;
  grid-area: 1 / 1;
  font: inherit;
  padding: inherit;
  border: 1px solid transparent;
  box-sizing: border-box;
}

.autoresize-wrapper.horizontal-resize .autoresize-input {
  grid-area: 1 / 1;
  width: 100%;
  min-width: 0;
}

.autoresize-wrapper.horizontal-resize .size-detector {
  grid-area: 1 / 1;
  width: auto;
  white-space: pre;
  overflow: hidden;
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

.size-detector {
  white-space: pre;
  word-wrap: break-word;
  overflow: hidden;
  pointer-events: none;
  user-select: none;
  color: transparent;
  background: transparent;
  border-color: transparent;
}
</style>