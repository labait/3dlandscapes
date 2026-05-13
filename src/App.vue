<script setup>
import { reactive } from 'vue'
import Landscape from './components/Landscape.vue'

const demoLandscape = reactive({
  name: 'bauhaus',
  gradientColors: [
    '#C5B597',
    '#88704B',
    '#1D4E89',
    '#E63946',
    '#F1B814',
    '#E76F51',
    '#F1FAEE',
  ],
  items: Array.from({ length: 10 }, (_, i) => ({
    title: `item ${i + 1}`,
    height: 1 + Math.floor(Math.random() * 10),
  })),
})
</script>

<template>
  <div class="app-layout">
    <main class="app-main">
      <Landscape :landscape-json="demoLandscape" />
    </main>
  </div>
</template>

<style scoped>
.app-layout {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: var(--canvas-bg, #f0ebe3);
}

.app-main {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
  box-sizing: border-box;
}
</style>
