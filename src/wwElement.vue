<template>
  <div 
    class="autoresize-wrapper"
    :class="{ 
      'horizontal-resize': effectiveAutoResizeDirection === 'horizontal',
      'vertical-resize': effectiveAutoResizeDirection === 'vertical' 
    }"
    :data-value="effectiveAutoResizeDirection === 'horizontal' ? (content.value || content.placeholder || '') : ''"
    :style="wrapperStyle"
  >
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
    wrapperStyle() {
      // Pass styles as CSS custom properties for the pseudo-element
      if (this.effectiveAutoResizeDirection === 'horizontal') {
        return {
          '--font-size': this.content.fontSize || '16px',
          '--font-family': this.content.fontFamily || 'inherit',
          '--font-weight': this.content.fontWeight || 'normal',
          '--line-height': this.content.lineHeight || 'normal',
          '--padding': this.content.padding || '8px 12px',
          '--border-width': this.content.borderWidth || '1px'
        }
      }
      return {}
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
        // Let CSS Grid handle the sizing
        style.width = '100%'
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
        }
      }, 50)
    })
  },
  watch: {
    'content.value'() {
      this.$nextTick(() => {
        if (this.effectiveAutoResizeDirection === 'vertical') {
          this.adjustTextareaHeight()
        }
      })
    },
    'effectiveAutoResizeDirection'() {
      this.$nextTick(() => {
        if (this.effectiveAutoResizeDirection === 'vertical') {
          this.adjustTextareaHeight()
        }
      })
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
        }
      })
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

/* Horizontal auto-sizing using CSS Grid */
.autoresize-wrapper.horizontal-resize {
  display: inline-grid;
  align-items: center;
  width: auto;
  position: relative;
}

.autoresize-wrapper.horizontal-resize::after {
  content: attr(data-value);
  visibility: hidden;
  white-space: pre;
  grid-area: 1 / 1 / 2 / 2;
  
  /* Copy text styles from input */
  font: inherit;
  font-size: var(--font-size, 16px);
  font-family: var(--font-family, inherit);
  font-weight: var(--font-weight, normal);
  line-height: var(--line-height, normal);
  
  /* Copy spacing styles */
  padding: var(--padding, 8px 12px);
  padding-right: calc(var(--padding, 8px 12px) + 5px); /* Extra space for cursor */
  border: var(--border-width, 1px) solid transparent;
  box-sizing: border-box;
  
  /* Ensure proper sizing */
  width: auto;
  min-width: 0;
  overflow: hidden;
}

.autoresize-wrapper.horizontal-resize .autoresize-input {
  grid-area: 1 / 1 / 2 / 2;
  width: 100%;
  min-width: 0;
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