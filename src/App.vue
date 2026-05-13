<script setup>
import { reactive } from 'vue'
import Landscape from './components/Landscape.vue'

const demoLandscape = reactive({
  name: 'landscape1',
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
  <div
    class="box-border flex min-h-screen flex-col bg-[#f0ebe3] antialiased selection:bg-neutral-900/15"
  >
    <main class="flex flex-1 items-center justify-center p-4 box-border">
      <Landscape :landscape-json="demoLandscape" />
    </main>
  </div>
</template>
