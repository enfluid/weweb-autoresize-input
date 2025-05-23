<template>
  <component 
    :is="content.multiLine ? 'textarea' : 'input'"
    ref="inputElement"
    :value="content.value"
    @input="handleInput"
    :placeholder="content.placeholder"
    :style="computedStyle"
    :class="content.inputClass"
    :disabled="content.disabled"
    :readonly="content.readonly"
    :maxlength="content.maxLength"
    :rows="content.multiLine ? content.rows : undefined"
  />
</template>

<script>
export default {
  props: {
    content: {
      type: Object,
      required: true
    },
    uid: {
      type: String,
      required: true
    }
  },
  emits: ['update:content'],
  data() {
    return {
      width: null,
      height: null,
      hiddenDiv: null,
      isFocused: false
    };
  },
  computed: {
    computedStyle() {
      const baseStyle = {
        fontSize: this.content.fontSize || '16px',
        fontFamily: this.content.fontFamily || 'inherit',
        padding: this.content.padding || '8px 12px',
        borderRadius: this.content.borderRadius || '4px',
        border: `${this.content.borderWidth || '1px'} solid ${this.isFocused ? (this.content.focusBorderColor || '#4CAF50') : (this.content.borderColor || '#e0e0e0')}`,
        backgroundColor: this.content.backgroundColor || '#ffffff',
        color: this.content.textColor || '#333333',
        transition: 'all 0.2s ease',
        resize: 'none',
        overflow: 'hidden',
        boxSizing: 'border-box',
        width: '100%'
      };

      const direction = this.content.autoResizeDirection || 'horizontal';

      // Apply auto-resize dimensions
      if (direction === 'horizontal' || direction === 'both') {
        baseStyle.width = this.width ? `${this.width}px` : (this.content.minWidth || '100px');
        baseStyle.minWidth = this.content.minWidth || '100px';
        baseStyle.maxWidth = this.content.maxWidth || '100%';
      }

      if (direction === 'vertical' || direction === 'both') {
        baseStyle.height = this.height ? `${this.height}px` : (this.content.minHeight || '40px');
        baseStyle.minHeight = this.content.minHeight || '40px';
        baseStyle.maxHeight = this.content.maxHeight || '200px';
      }

      return baseStyle;
    }
  },
  mounted() {
    this.createHiddenDiv();
    this.updateSize();
    
    // Add focus/blur listeners
    if (this.$refs.inputElement) {
      this.$refs.inputElement.addEventListener('focus', this.handleFocus);
      this.$refs.inputElement.addEventListener('blur', this.handleBlur);
    }
  },
  beforeUnmount() {
    if (this.hiddenDiv && this.hiddenDiv.parentNode) {
      this.hiddenDiv.parentNode.removeChild(this.hiddenDiv);
    }
    
    // Remove focus/blur listeners
    if (this.$refs.inputElement) {
      this.$refs.inputElement.removeEventListener('focus', this.handleFocus);
      this.$refs.inputElement.removeEventListener('blur', this.handleBlur);
    }
  },
  methods: {
    handleInput(event) {
      const newValue = event.target.value;
      this.$emit('update:content', { ...this.content, value: newValue });
      this.$nextTick(() => {
        this.updateSize();
      });
    },
    handleFocus() {
      this.isFocused = true;
    },
    handleBlur() {
      this.isFocused = false;
    },
    createHiddenDiv() {
      this.hiddenDiv = document.createElement('div');
      this.hiddenDiv.style.position = 'absolute';
      this.hiddenDiv.style.top = '-9999px';
      this.hiddenDiv.style.left = '-9999px';
      this.hiddenDiv.style.visibility = 'hidden';
      this.hiddenDiv.style.whiteSpace = this.content.multiLine ? 'pre-wrap' : 'pre';
      this.hiddenDiv.style.wordWrap = 'break-word';
      this.hiddenDiv.style.fontSize = this.content.fontSize || '16px';
      this.hiddenDiv.style.fontFamily = this.content.fontFamily || 'inherit';
      this.hiddenDiv.style.padding = this.content.padding || '8px 12px';
      this.hiddenDiv.style.border = `${this.content.borderWidth || '1px'} solid transparent`;
      this.hiddenDiv.style.boxSizing = 'border-box';
      document.body.appendChild(this.hiddenDiv);
    },
    updateSize() {
      const direction = this.content.autoResizeDirection || 'horizontal';
      if (direction === 'none') return;

      const inputEl = this.$refs.inputElement;
      if (!inputEl || !this.hiddenDiv) return;

      // Update hidden div content
      this.hiddenDiv.textContent = this.content.value || this.content.placeholder || '';
      
      // Handle horizontal resizing
      if (direction === 'horizontal' || direction === 'both') {
        if (!this.content.multiLine) {
          // For single-line inputs, add some padding to prevent text cutoff
          this.width = this.hiddenDiv.scrollWidth + (this.content.horizontalPadding || 5);
        } else {
          // For multi-line, set the width constraint for proper height calculation
          this.hiddenDiv.style.width = this.content.maxWidth || '100%';
          this.width = Math.min(
            Math.max(this.hiddenDiv.scrollWidth, parseInt(this.content.minWidth || '100')),
            parseInt(this.content.maxWidth || '9999')
          );
        }
      }

      // Handle vertical resizing
      if (direction === 'vertical' || direction === 'both') {
        if (this.content.multiLine) {
          // Set width for proper text wrapping calculation
          if (direction === 'vertical') {
            this.hiddenDiv.style.width = `${inputEl.offsetWidth}px`;
          }
          
          this.height = Math.min(
            Math.max(this.hiddenDiv.scrollHeight, parseInt(this.content.minHeight || '40')),
            parseInt(this.content.maxHeight || '200')
          );
        }
      }
    }
  },
  watch: {
    'content.value'() {
      this.$nextTick(() => {
        this.updateSize();
      });
    },
    'content.fontSize'() {
      if (this.hiddenDiv) {
        this.hiddenDiv.style.fontSize = this.content.fontSize || '16px';
        this.updateSize();
      }
    },
    'content.fontFamily'() {
      if (this.hiddenDiv) {
        this.hiddenDiv.style.fontFamily = this.content.fontFamily || 'inherit';
        this.updateSize();
      }
    },
    'content.padding'() {
      if (this.hiddenDiv) {
        this.hiddenDiv.style.padding = this.content.padding || '8px 12px';
        this.updateSize();
      }
    }
  }
};
</script>