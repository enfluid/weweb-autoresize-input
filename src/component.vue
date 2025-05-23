<template>
  <component :is="multiLine ? 'textarea' : 'input'"
    ref="inputElement"
    :value="value"
    @input="handleInput"
    :placeholder="placeholder"
    :style="computedStyle"
    :class="inputClass"
    :disabled="disabled"
    :readonly="readonly"
    :maxlength="maxLength"
    :rows="multiLine ? rows : undefined"
  />
</template>

<script>
export default {
  props: {
    value: {
      type: String,
      default: ''
    },
    placeholder: {
      type: String,
      default: ''
    },
    multiLine: {
      type: Boolean,
      default: false
    },
    autoResizeDirection: {
      type: String,
      default: 'horizontal', // 'horizontal', 'vertical', 'both', 'none'
      validator: (value) => ['horizontal', 'vertical', 'both', 'none'].includes(value)
    },
    minWidth: {
      type: String,
      default: '100px'
    },
    maxWidth: {
      type: String,
      default: '100%'
    },
    minHeight: {
      type: String,
      default: '40px'
    },
    maxHeight: {
      type: String,
      default: '200px'
    },
    fontSize: {
      type: String,
      default: '16px'
    },
    fontFamily: {
      type: String,
      default: 'inherit'
    },
    padding: {
      type: String,
      default: '8px 12px'
    },
    borderRadius: {
      type: String,
      default: '4px'
    },
    borderColor: {
      type: String,
      default: '#e0e0e0'
    },
    borderWidth: {
      type: String,
      default: '1px'
    },
    backgroundColor: {
      type: String,
      default: '#ffffff'
    },
    textColor: {
      type: String,
      default: '#333333'
    },
    focusBorderColor: {
      type: String,
      default: '#4CAF50'
    },
    inputClass: {
      type: String,
      default: ''
    },
    disabled: {
      type: Boolean,
      default: false
    },
    readonly: {
      type: Boolean,
      default: false
    },
    maxLength: {
      type: Number,
      default: null
    },
    rows: {
      type: Number,
      default: 1
    },
    horizontalPadding: {
      type: Number,
      default: 5
    }
  },
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
        fontSize: this.fontSize,
        fontFamily: this.fontFamily,
        padding: this.padding,
        borderRadius: this.borderRadius,
        border: `${this.borderWidth} solid ${this.isFocused ? this.focusBorderColor : this.borderColor}`,
        backgroundColor: this.backgroundColor,
        color: this.textColor,
        transition: 'all 0.2s ease',
        resize: 'none',
        overflow: 'hidden',
        boxSizing: 'border-box'
      };

      // Apply auto-resize dimensions
      if (this.autoResizeDirection === 'horizontal' || this.autoResizeDirection === 'both') {
        baseStyle.width = this.width ? `${this.width}px` : this.minWidth;
        baseStyle.minWidth = this.minWidth;
        baseStyle.maxWidth = this.maxWidth;
      }

      if (this.autoResizeDirection === 'vertical' || this.autoResizeDirection === 'both') {
        baseStyle.height = this.height ? `${this.height}px` : this.minHeight;
        baseStyle.minHeight = this.minHeight;
        baseStyle.maxHeight = this.maxHeight;
      }

      return baseStyle;
    }
  },
  mounted() {
    this.createHiddenDiv();
    this.updateSize();
    
    // Add focus/blur listeners
    this.$refs.inputElement.addEventListener('focus', this.handleFocus);
    this.$refs.inputElement.addEventListener('blur', this.handleBlur);
  },
  beforeDestroy() {
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
      this.$emit('input', event.target.value);
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
      this.hiddenDiv.style.whiteSpace = this.multiLine ? 'pre-wrap' : 'pre';
      this.hiddenDiv.style.wordWrap = 'break-word';
      this.hiddenDiv.style.fontSize = this.fontSize;
      this.hiddenDiv.style.fontFamily = this.fontFamily;
      this.hiddenDiv.style.padding = this.padding;
      this.hiddenDiv.style.border = `${this.borderWidth} solid transparent`;
      this.hiddenDiv.style.boxSizing = 'border-box';
      document.body.appendChild(this.hiddenDiv);
    },
    updateSize() {
      if (this.autoResizeDirection === 'none') return;

      const inputEl = this.$refs.inputElement;
      if (!inputEl || !this.hiddenDiv) return;

      // Update hidden div content
      this.hiddenDiv.textContent = this.value || this.placeholder || '';
      
      // Handle horizontal resizing
      if (this.autoResizeDirection === 'horizontal' || this.autoResizeDirection === 'both') {
        if (!this.multiLine) {
          // For single-line inputs, add some padding to prevent text cutoff
          this.width = this.hiddenDiv.scrollWidth + this.horizontalPadding;
        } else {
          // For multi-line, set the width constraint for proper height calculation
          this.hiddenDiv.style.width = this.maxWidth;
          this.width = Math.min(
            Math.max(this.hiddenDiv.scrollWidth, parseInt(this.minWidth)),
            parseInt(this.maxWidth)
          );
        }
      }

      // Handle vertical resizing
      if (this.autoResizeDirection === 'vertical' || this.autoResizeDirection === 'both') {
        if (this.multiLine) {
          // Set width for proper text wrapping calculation
          if (this.autoResizeDirection === 'vertical') {
            this.hiddenDiv.style.width = `${inputEl.offsetWidth}px`;
          }
          
          this.height = Math.min(
            Math.max(this.hiddenDiv.scrollHeight, parseInt(this.minHeight)),
            parseInt(this.maxHeight)
          );
        }
      }
    }
  },
  watch: {
    value() {
      this.$nextTick(() => {
        this.updateSize();
      });
    },
    fontSize() {
      if (this.hiddenDiv) {
        this.hiddenDiv.style.fontSize = this.fontSize;
        this.updateSize();
      }
    },
    fontFamily() {
      if (this.hiddenDiv) {
        this.hiddenDiv.style.fontFamily = this.fontFamily;
        this.updateSize();
      }
    },
    padding() {
      if (this.hiddenDiv) {
        this.hiddenDiv.style.padding = this.padding;
        this.updateSize();
      }
    }
  }
};
</script>