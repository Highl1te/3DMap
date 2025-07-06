<template>
  <div class="map-view">
    <BabylonMap />
  </div>
</template>

<script setup lang="ts">
import BabylonMap from '../components/BabylonMap.vue'
</script>

<style scoped>
.map-view {
  width: 100%;
  height: 100vh;
  overflow: hidden;
}
</style> 