<script setup lang="ts">
const mode = useColorMode()
const { t } = useI18n()
const dark = computed(() => mode.value === 'dark')
</script>

<template>
  <ClientOnly>
    <UButton color="neutral" variant="ghost" :icon="dark ? 'i-lucide-sun' : 'i-lucide-moon'" :aria-label="t(dark ? 'theme.light' : 'theme.dark')" :title="t(dark ? 'theme.light' : 'theme.dark')" class="theme-toggle" @click="mode.preference = dark ? 'light' : 'dark'" />
    <template #fallback><span class="inline-block size-8" aria-hidden="true" /></template>
  </ClientOnly>
</template>
