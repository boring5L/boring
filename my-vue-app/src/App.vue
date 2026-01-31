<template>
<PuzzleGame :size="5" :cellSize="30"/>
  <CubesGame />
</template>

<script setup lang="ts">
import PuzzleGame from "./components/PuzzleGame.vue";
import CubesGame from "./components/CubesGame.vue";
</script>

<style scoped>
</style>
