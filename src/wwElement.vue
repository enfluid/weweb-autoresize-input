<template>
  <div 
    class="autoresize-wrapper"
    :class="{ 
      'horizontal-resize': effectiveAutoResizeDirection === 'horizontal',
      'vertical-resize': effectiveAutoResizeDirection === 'vertical'
    }"
  >
    <!-- Use contenteditable for auto-resizing like RichText -->
    <div
      v-if="effectiveAutoResizeDirection === 'horizontal'"
      ref="editableElement"
      class="autoresize-input contenteditable-input"
      :contenteditable="!content.disabled && !content.readonly"
      @input="handleContentEditableInput"
      @focus="handleFocus"
      @blur="handleBlur"
      @paste="handlePaste"
      @keydown="handleKeydown"
      :style="inputStyle"
      :data-placeholder="content.placeholder || 'Enter text...'"
      v-text="content.value || ''"
    ></div>
    
    <!-- Keep regular textarea for vertical resize -->
    <textarea
      v-else-if="effectiveAutoResizeDirection === 'vertical'"
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
    />
    
    <!-- Regular input for no resize -->
    <input
      v-else
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
    inputStyle() {
      const style = {
        fontSize: this.content.fontSize || '16px',
        fontFamily: this.content.fontFamily || 'inherit',
        fontWeight: this.content.fontWeight || 'normal',
        lineHeight: this.content.lineHeight || '1.2',
        padding: this.content.padding || '1px 4px',
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

      // Handle width
      if (this.effectiveAutoResizeDirection === 'horizontal') {
        // For contenteditable, let it auto-size within bounds
        style.display = 'inline-block'
        style.width = 'auto'
        style.minWidth = this.content.minWidth || '50px'
        style.maxWidth = this.content.maxWidth || '100%'
        style.whiteSpace = 'nowrap'
        style.overflowX = 'auto'
        style.overflowY = 'hidden'
        // Hide scrollbar but keep scroll functionality
        style.scrollbarWidth = 'none'
        style.msOverflowStyle = 'none'
      } else {
        style.width = '100%'
      }

      // Handle vertical auto-resizing for textareas
      if (this.effectiveAutoResizeDirection === 'vertical') {
        style.overflow = 'hidden'
        style.resize = 'none'
      } else if (this.effectiveAutoResizeDirection === 'horizontal') {
        // Let padding determine height for horizontal mode
        style.height = 'auto'
      } else if (!this.content.multiLine) {
        // Let padding determine height for single-line inputs
        style.height = 'auto'
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
          // Set initial content for contenteditable
          const editable = this.$refs.editableElement
          if (editable && editable.textContent !== this.content.value) {
            editable.textContent = this.content.value || ''
          }
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
          // Update contenteditable if value changed externally
          const editable = this.$refs.editableElement
          if (editable && editable.textContent !== this.content.value) {
            editable.textContent = this.content.value || ''
          }
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
    handleContentEditableInput(event) {
      // Get plain text content only
      const text = event.target.textContent || ''
      
      // Update the content value
      this.$emit('update:content', { 
        ...this.content, 
        value: text 
      })
      
      // Trigger input event for workflows
      this.$emit('trigger-event', {
        name: 'input',
        event: {
          value: text,
          length: text.length
        }
      })
    },
    handlePaste(event) {
      // Prevent formatted paste - only allow plain text
      event.preventDefault()
      
      const text = (event.clipboardData || window.clipboardData).getData('text/plain')
      
      // Insert plain text at cursor position
      const selection = window.getSelection()
      if (!selection.rangeCount) return
      
      selection.deleteFromDocument()
      selection.getRangeAt(0).insertNode(document.createTextNode(text))
      
      // Move cursor to end of inserted text
      selection.collapseToEnd()
      
      // Trigger input event
      this.handleContentEditableInput({ target: event.target })
    },
    handleKeydown(event) {
      // Prevent Enter key in single-line mode
      if (event.key === 'Enter' && this.effectiveAutoResizeDirection === 'horizontal') {
        event.preventDefault()
        
        // Trigger blur to simulate form submission
        event.target.blur()
        
        // Trigger enter event for workflows
        this.$emit('trigger-event', {
          name: 'enter',
          event: {
            value: event.target.textContent || ''
          }
        })
      }
      
      // Handle max length
      if (this.content.maxLength && event.target.textContent.length >= this.content.maxLength) {
        // Allow backspace, delete, arrow keys
        const allowedKeys = ['Backspace', 'Delete', 'ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown']
        if (!allowedKeys.includes(event.key)) {
          event.preventDefault()
        }
      }
    },
    adjustTextareaHeight() {
      const textarea = this.$refs.inputElement
      if (!textarea || this.effectiveAutoResizeDirection !== 'vertical') return
      
      // Reset height to get accurate scrollHeight
      textarea.style.height = '1px'
      
      // Get the scroll height
      const scrollHeight = textarea.scrollHeight
      
      // Set the height
      textarea.style.height = `${scrollHeight}px`
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
}

/* Horizontal auto-sizing */
.autoresize-wrapper.horizontal-resize {
  display: inline-block;
  width: auto;
  max-width: 100%;
}

.autoresize-wrapper.vertical-resize {
  display: block;
  width: 100%;
  height: auto;
  min-height: 0;
}

.autoresize-input {
  font-family: inherit;
  box-sizing: border-box;
  vertical-align: middle;
  line-height: 1.2;
}

/* Fine-tune vertical centering - shift text up slightly */
.autoresize-input,
.contenteditable-input {
  transform: translateY(0px);
}

/* Ensure textarea has no default height constraints */
textarea.autoresize-input {
  min-height: 1px;
  max-height: none !important;
}

.autoresize-input::placeholder {
  color: #999;
  opacity: 1;
}

.autoresize-input:disabled {
  -webkit-text-fill-color: currentColor;
  opacity: 1;
}

/* Contenteditable styles */
.contenteditable-input {
  outline: none;
  cursor: text;
  max-width: 100%;
  word-break: keep-all;
  line-height: 1.2;
  vertical-align: middle;
}

/* Hide scrollbar for contenteditable but keep scroll functionality */
.contenteditable-input::-webkit-scrollbar {
  display: none;
}

.contenteditable-input:empty:before {
  content: attr(data-placeholder);
  color: #999;
  pointer-events: none;
}

.contenteditable-input[contenteditable="false"] {
  cursor: default;
}

/* Prevent text selection in disabled state */
.contenteditable-input[contenteditable="false"] {
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

</style>