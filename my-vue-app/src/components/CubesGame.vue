<script setup lang="ts">
import {onMounted, onUnmounted, ref} from 'vue'
import * as THREE from 'three'

const canvasContainer = ref<HTMLElement | null>(null)

onMounted(() => {
  if (!canvasContainer.value) return

  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(75, 800 / 600, 0.1, 1000)
  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(800, 600)
  canvasContainer.value.appendChild(renderer.domElement)

  // 光源
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5)
  scene.add(ambientLight)
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(5, 5, 5)
  scene.add(directionalLight)

  // 魔方（红色，标准材质）
  const cube = new THREE.Mesh(
      new THREE.BoxGeometry(1, 1, 1),
      new THREE.MeshStandardMaterial({ color: 0xff0000 })
  )
  cube.position.set(0, 0, -3)
  scene.add(cube)

  // 动画
  const animate = () => {
    requestAnimationFrame(animate)
    cube.rotation.x += 0.01
    cube.rotation.y += 0.01
    renderer.render(scene, camera)
  }
  animate()

  // 窗口大小
  const handleResize = () => {
    camera.aspect = 800 / 600
    camera.updateProjectionMatrix()
    renderer.setSize(800, 600)
  }
  window.addEventListener('resize', handleResize)
  onUnmounted(() => window.removeEventListener('resize', handleResize))
})
</script>

<template>
  <div class="puzzle-game">
    <h2>魔方游戏</h2>
    <div ref="canvasContainer" class="canvas-container"></div>
  </div>
</template>

<style scoped>
.canvas-container {
  width: 800px;
  height: 600px;
  margin: 0 auto;
}
</style>
