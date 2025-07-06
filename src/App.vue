<script setup lang="ts">
import MapView from './views/MapView.vue'
</script>

<template>
  <MapView />
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  overflow: hidden;
  background-color: #000000;
}

body {
  font-family: Arial, sans-serif;
}

#app {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  outline: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

*:focus {
  outline: none !important;
}
</style>
