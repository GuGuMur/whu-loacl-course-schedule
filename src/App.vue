<template>
  <n-config-provider :theme-overrides="themeOverrides">
    <n-message-provider>
      <CourseSchedule />
    </n-message-provider>
  </n-config-provider>
</template>

<script setup>
import { NConfigProvider, NMessageProvider } from 'naive-ui'
import CourseSchedule from './components/CourseSchedule.vue'

const themeOverrides = {
  common: { primaryColor: '#18a058' }
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
  font-family: v-sans, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}
</style>