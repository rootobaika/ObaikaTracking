<script setup>
const props = defineProps({
  title: {
    type: String,
    required: true,
  },
  value: {
    type: [String, Number],
    default: '',
  },
  wide: {
    type: Boolean,
    default: false,
  },
});
</script>

<template>
  <article class="card metric" :class="{ wide: props.wide }">
    <p>{{ props.title }}</p>
    <strong v-if="props.value !== ''">{{ props.value }}</strong>
    <slot />
  </article>
</template>
