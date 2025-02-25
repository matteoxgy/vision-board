<template>
  <div class="app">
    <VisionBoard />
  </div>
</template>

<script setup>
import VisionBoard from './components/VisionBoard.vue'
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.app {
  width: 100%;
  min-height: 100vh;
  background-color: #f5f5f5;
}
</style> 