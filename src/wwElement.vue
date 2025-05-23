<template>
  <div class="autoresize-wrapper" :style="wrapperStyle">
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
    <div 
      v-if="content.autoResizeDirection === 'horizontal' && !content.multiLine"
      ref="measurer"
      class="text-measurer"
      :style="measurerStyle"
    >{{ content.value || content.placeholder || 'W' }}</div>
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
      measuredWidth: 0,
      measuredHeight: 0
    }
  },
  computed: {
    componentType() {
      return this.content.multiLine ? 'textarea' : 'input'
    },
    wrapperStyle() {
      return {
        display: 'inline-block',
        width: '100%'
      }
    },
    measurerStyle() {
      return {
        position: 'absolute',
        top: '-9999px',
        left: '-9999px',
        visibility: 'hidden',
        whiteSpace: 'pre',
        fontSize: this.content.fontSize || '16px',
        fontFamily: this.content.fontFamily || 'inherit',
        fontWeight: this.content.fontWeight || 'normal',
        lineHeight: this.content.lineHeight || 'normal',
        padding: this.content.padding || '8px 12px',
        borderWidth: this.content.borderWidth || '1px',
        borderStyle: 'solid',
        boxSizing: 'border-box'
      }
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
        if (this.measuredWidth > 0) {
          const minWidth = parseInt(this.content.minWidth) || 100
          const maxWidthValue = this.content.maxWidth || '100%'
          let maxWidth = 9999
          
          if (maxWidthValue.includes('%') && this.$el) {
            const parent = this.$el.parentElement
            if (parent) {
              maxWidth = (parseInt(maxWidthValue) / 100) * parent.offsetWidth
            }
          } else {
            maxWidth = parseInt(maxWidthValue) || 9999
          }
          
          const finalWidth = Math.min(Math.max(this.measuredWidth, minWidth), maxWidth)
          style.width = `${finalWidth}px`
        }
      }

      // Handle vertical resizing
      if (direction === 'vertical' && this.content.multiLine) {
        style.minHeight = this.content.minHeight || '40px'
        style.maxHeight = this.content.maxHeight || '200px'
        
        if (this.measuredHeight > 0) {
          const minHeight = parseInt(this.content.minHeight) || 40
          const maxHeight = parseInt(this.content.maxHeight) || 200
          const finalHeight = Math.min(Math.max(this.measuredHeight, minHeight), maxHeight)
          style.height = `${finalHeight}px`
        }
      } else if (!this.content.multiLine) {
        // Single line inputs should have a fixed height
        style.height = this.content.inputHeight || '40px'
      }

      return style
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.measureText()
    })
  },
  updated() {
    this.$nextTick(() => {
      this.measureText()
    })
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
        this.measureText()
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
    measureText() {
      const direction = this.content.autoResizeDirection || 'horizontal'
      
      if (direction === 'none') return

      // Handle horizontal resizing
      if (direction === 'horizontal' && !this.content.multiLine) {
        const measurer = this.$refs.measurer
        if (measurer) {
          this.measuredWidth = measurer.offsetWidth + (parseInt(this.content.horizontalPadding) || 10)
        }
      }

      // Handle vertical resizing
      if (direction === 'vertical' && this.content.multiLine) {
        const textContent = this.content.value || ''
        const lines = textContent.split('\n')
        const lineCount = Math.max(lines.length, 1)
        
        const fontSize = parseInt(this.content.fontSize) || 16
        const lineHeight = this.content.lineHeight === 'normal' ? fontSize * 1.2 : 
                         (parseFloat(this.content.lineHeight) || 1.2) * fontSize
        
        const padding = this.extractPadding()
        const border = parseInt(this.content.borderWidth || '1px') * 2
        
        this.measuredHeight = (lineCount * lineHeight) + padding.vertical + border
      }
    },
    extractPadding() {
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
  },
  watch: {
    'content.value': {
      handler() {
        this.$nextTick(() => {
          this.measureText()
        })
      },
      immediate: true
    },
    'content.fontSize'() {
      this.$nextTick(() => {
        this.measureText()
      })
    },
    'content.fontFamily'() {
      this.$nextTick(() => {
        this.measureText()
      })
    },
    'content.fontWeight'() {
      this.$nextTick(() => {
        this.measureText()
      })
    },
    'content.padding'() {
      this.$nextTick(() => {
        this.measureText()
      })
    },
    'content.multiLine'() {
      this.$nextTick(() => {
        this.measureText()
      })
    },
    'content.autoResizeDirection'() {
      this.$nextTick(() => {
        this.measureText()
      })
    }
  }
}
</script>

<style scoped>
.autoresize-wrapper {
  position: relative;
  display: inline-block;
  width: 100%;
}

.autoresize-input {
  font-family: inherit;
  width: 100%;
}

.autoresize-input::placeholder {
  color: #999;
  opacity: 1;
}

.autoresize-input:disabled {
  -webkit-text-fill-color: currentColor;
  opacity: 1;
}

.text-measurer {
  white-space: pre;
  word-wrap: break-word;
}
</style>