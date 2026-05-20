<script setup>
import { reactive, onMounted, ref } from 'vue'
import Landscape from './components/Landscape.vue'

const landscapeData = ref(null)
const landscapeId = ref(null)

const files = [
  'item1.json',
  'item2.json',
]
const items = reactive({})

onMounted(async () => {
  files.forEach(async (file, index) => {
    const response = await fetch(`/data/${file}`)
    const data = await response.json()
    items[index] = data
  })
})

const handleClick = (index) => {
  landscapeId.value = index
  landscapeData.value = items[index]
}
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="index">
     <a href="#" @click="handleClick(index)" :class="index == landscapeId ?  'active underline' : ''"> {{ item.name }}</a>
    </li>
  </ul>
  <div
    class="box-border flex min-h-screen flex-col bg-[#f0ebe3] antialiased selection:bg-neutral-900/15"
  >
    <main class="flex flex-1 items-center justify-center p-4 box-border">
      <Landscape v-if="landscapeId" :landscapeData="landscapeData" />
    </main>
  </div>
</template>
