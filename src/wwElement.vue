<template>
  <div>
    <input 
      :value="content.value || ''" 
      @input="$emit('update:content', { ...content, value: $event.target.value })"
      :placeholder="content.placeholder || 'Type here...'"
    />
  </div>
</template>

<script>
export default {
  props: ['content']
}
</script>