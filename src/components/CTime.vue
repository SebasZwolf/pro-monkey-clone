<script lang="ts" setup>

const p = withDefaults(defineProps<{
	date?: Date;
	format? : Intl.DateTimeFormatOptions;
}>(), {
  date : () => new Date(),
});


</script>

<template>
  <time :datetime="date.toISOString()" v-text="p.date.toLocaleDateString('es', Object.assign({
    year: 'numeric',
    month: 'short',
    day: 'numeric',
  }, p.format))"></time>
</template>