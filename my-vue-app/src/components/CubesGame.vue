<script setup lang="ts">
import { onMounted, onUnmounted, ref, reactive } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const canvasContainer = ref<HTMLElement | null>(null)

// 游戏状态
const gameState = reactive({
  score: 0,
  time: 0,
  isSolving: false,
  moves: 0
})

// 魔方配置
const cubeConfig = {
  size: 3, // 3x3x3魔方
  cubeletSize: 1,
  gap: 0.1 // 小方块之间的间隙
}

// 颜色配置（标准魔方颜色）
const colors = {
  front: 0xff0000,   // 红
  back: 0xff8800,    // 橙
  right: 0x0000ff,   // 蓝
  left: 0x00ff00,    // 绿
  top: 0xffffff,     // 白
  bottom: 0xffff00   // 黄
}

// 存储所有小方块的引用
let cubelets: THREE.Group[] = []
let scene: THREE.Scene, camera: THREE.PerspectiveCamera, renderer: THREE.WebGLRenderer, controls: OrbitControls

// 初始化Three.js场景
const initScene = () => {
  if (!canvasContainer.value) return

  // 1. 创建基础场景
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf0f0f0)

  camera = new THREE.PerspectiveCamera(75, canvasContainer.value.clientWidth / canvasContainer.value.clientHeight, 0.1, 1000)
  camera.position.set(8, 8, 8)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight)
  renderer.shadowMap.enabled = true
  canvasContainer.value.appendChild(renderer.domElement)

  // 2. 添加光源
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(10, 20, 10)
  directionalLight.castShadow = true
  scene.add(directionalLight)

  // 3. 添加控制器
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.minDistance = 5
  controls.maxDistance = 20
  controls.target.set(0, 0, 0)
  controls.update()

  // 4. 添加辅助工具（开发时可用）
  const axesHelper = new THREE.AxesHelper(5)
  scene.add(axesHelper)

  const gridHelper = new THREE.GridHelper(10, 10)
  scene.add(gridHelper)
}

// 创建单个小方块
const createCubelet = (x: number, y: number, z: number) => {
  const group = new THREE.Group()

  // 根据位置确定哪些面需要着色
  const materials = []

  // 6个面
  for (let i = 0; i < 6; i++) {
    let color = 0x333333 // 默认黑色（内部不可见面）

    // 根据位置确定颜色
    if (z === cubeConfig.size - 1) materials.push(new THREE.MeshBasicMaterial({ color: colors.front }))    // 前
    else if (z === 0) materials.push(new THREE.MeshBasicMaterial({ color: colors.back }))                  // 后
    else if (x === cubeConfig.size - 1) materials.push(new THREE.MeshBasicMaterial({ color: colors.right })) // 右
    else if (x === 0) materials.push(new THREE.MeshBasicMaterial({ color: colors.left }))                  // 左
    else if (y === cubeConfig.size - 1) materials.push(new THREE.MeshBasicMaterial({ color: colors.top }))   // 上
    else if (y === 0) materials.push(new THREE.MeshBasicMaterial({ color: colors.bottom }))                // 下
    else materials.push(new THREE.MeshBasicMaterial({ color: 0x333333 }))
  }

  const geometry = new THREE.BoxGeometry(
      cubeConfig.cubeletSize - cubeConfig.gap,
      cubeConfig.cubeletSize - cubeConfig.gap,
      cubeConfig.cubeletSize - cubeConfig.gap
  )

  const cube = new THREE.Mesh(geometry, materials)
  group.add(cube)

  // 添加黑色边框
  const edges = new THREE.EdgesGeometry(geometry)
  const line = new THREE.LineSegments(edges, new THREE.LineBasicMaterial({ color: 0x000000 }))
  group.add(line)

  // 设置位置
  const offset = (cubeConfig.size - 1) * cubeConfig.cubeletSize / 2
  group.position.set(
      x * cubeConfig.cubeletSize - offset,
      y * cubeConfig.cubeletSize - offset,
      z * cubeConfig.cubeletSize - offset
  )

  // 存储位置信息
  group.userData = { originalPosition: { x, y, z }, currentPosition: { x, y, z } }

  return group
}

// 创建完整魔方
const createRubiksCube = () => {
  cubelets = []

  for (let x = 0; x < cubeConfig.size; x++) {
    for (let y = 0; y < cubeConfig.size; y++) {
      for (let z = 0; z < cubeConfig.size; z++) {
        // 跳过中心块（3x3魔方只有26个可见方块）
        if (x === 1 && y === 1 && z === 1) continue

        const cubelet = createCubelet(x, y, z)
        scene.add(cubelet)
        cubelets.push(cubelet)
      }
    }
  }
}

// 旋转魔方的一层
const rotateLayer = (axis: 'x' | 'y' | 'z', layer: number, clockwise: boolean = true) => {
  gameState.moves++

  // 选择要旋转的小方块
  const targetCubelets = cubelets.filter(cubelet => {
    const pos = cubelet.userData.currentPosition
    if (axis === 'x' && pos.x === layer) return true
    if (axis === 'y' && pos.y === layer) return true
    if (axis === 'z' && pos.z === layer) return true
    return false
  })

  // 创建旋转组
  const rotationGroup = new THREE.Group()
  scene.add(rotationGroup)

  targetCubelets.forEach(cubelet => {
    rotationGroup.add(cubelet)
  })

  // 执行旋转动画
  const duration = 300 // 毫秒
  const startTime = Date.now()
  const angle = clockwise ? Math.PI / 2 : -Math.PI / 2

  const animateRotation = () => {
    const elapsed = Date.now() - startTime
    const progress = Math.min(elapsed / duration, 1)

    if (axis === 'x') rotationGroup.rotation.x = angle * progress
    else if (axis === 'y') rotationGroup.rotation.y = angle * progress
    else if (axis === 'z') rotationGroup.rotation.z = angle * progress

    if (progress < 1) {
      requestAnimationFrame(animateRotation)
    } else {
      // 旋转完成后更新位置
      rotationGroup.rotation[axis] = angle

      // 更新小方块的位置数据
      targetCubelets.forEach(cubelet => {
        const pos = cubelet.userData.currentPosition
        let newX = pos.x, newY = pos.y, newZ = pos.z

        if (axis === 'x' && layer === pos.x) {
          // 绕X轴旋转
          if (clockwise) {
            newY = cubeConfig.size - 1 - pos.z
            newZ = pos.y
          } else {
            newY = pos.z
            newZ = cubeConfig.size - 1 - pos.y
          }
        } else if (axis === 'y' && layer === pos.y) {
          // 绕Y轴旋转
          if (clockwise) {
            newZ = cubeConfig.size - 1 - pos.x
            newX = pos.z
          } else {
            newZ = pos.x
            newX = cubeConfig.size - 1 - pos.z
          }
        } else if (axis === 'z' && layer === pos.z) {
          // 绕Z轴旋转
          if (clockwise) {
            newX = cubeConfig.size - 1 - pos.y
            newY = pos.x
          } else {
            newX = pos.y
            newY = cubeConfig.size - 1 - pos.x
          }
        }

        cubelet.userData.currentPosition = { x: newX, y: newY, z: newZ }
      })

      // 从组中移除并重新添加到场景
      targetCubelets.forEach(cubelet => {
        rotationGroup.remove(cubelet)
        scene.add(cubelet)
      })

      scene.remove(rotationGroup)

      // 检查是否完成
      checkCompletion()
    }

    renderer.render(scene, camera)
  }

  animateRotation()
}

// 打乱魔方
const scrambleCube = () => {
  gameState.isSolving = true
  gameState.moves = 0
  gameState.time = 0

  const moves = 20 // 打乱步数

  const performMove = (index: number) => {
    if (index >= moves) {
      gameState.isSolving = false
      return
    }

    const axis = ['x', 'y', 'z'][Math.floor(Math.random() * 3)]
    const layer = Math.floor(Math.random() * cubeConfig.size)
    const clockwise = Math.random() > 0.5

    setTimeout(() => {
      rotateLayer(axis as 'x' | 'y' | 'z', layer, clockwise)
      performMove(index + 1)
    }, 400)
  }

  performMove(0)
}

// 检查是否完成
const checkCompletion = () => {
  if (gameState.isSolving) return

  // 这里可以添加更复杂的完成检查逻辑
  // 简化版：检查每个面的颜色是否一致
  let isSolved = true

  // TODO: 实现完整的完成检查

  if (isSolved) {
    gameState.score += 100
    alert(`恭喜！完成魔方！\n用时: ${gameState.time}秒\n步数: ${gameState.moves}`)
  }
}

// 重置魔方
const resetCube = () => {
  // 移除所有小方块
  cubelets.forEach(cubelet => scene.remove(cubelet))

  // 重置游戏状态
  gameState.score = 0
  gameState.time = 0
  gameState.moves = 0
  gameState.isSolving = false

  // 重新创建魔方
  createRubiksCube()
}

// 动画循环
const animate = () => {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

// 处理窗口大小变化
const handleResize = () => {
  if (!canvasContainer.value) return

  camera.aspect = canvasContainer.value.clientWidth / canvasContainer.value.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight)
}

onMounted(() => {
  initScene()
  createRubiksCube()
  animate()

  // 计时器
  setInterval(() => {
    if (!gameState.isSolving) gameState.time++
  }, 1000)

  window.addEventListener('resize', handleResize)
  handleResize()
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>

<template>
  <div class="rubiks-cube-game">
    <div class="game-header">
      <h1>3D魔方游戏</h1>
      <div class="game-stats">
        <div class="stat">
          <span class="label">分数:</span>
          <span class="value">{{ gameState.score }}</span>
        </div>
        <div class="stat">
          <span class="label">时间:</span>
          <span class="value">{{ gameState.time }}秒</span>
        </div>
        <div class="stat">
          <span class="label">步数:</span>
          <span class="value">{{ gameState.moves }}</span>
        </div>
      </div>
    </div>

    <div class="game-container">
      <div ref="canvasContainer" class="canvas-container"></div>

      <div class="controls-panel">
        <div class="control-section">
          <h3>游戏控制</h3>
          <button @click="scrambleCube" :disabled="gameState.isSolving">
            {{ gameState.isSolving ? '打乱中...' : '打乱魔方' }}
          </button>
          <button @click="resetCube">重置游戏</button>
        </div>

        <div class="control-section">
          <h3>旋转控制</h3>
          <div class="rotation-controls">
            <div class="axis-control">
              <h4>X轴 (左右)</h4>
              <button v-for="i in cubeConfig.size" :key="`x-${i}`"
                      @click="rotateLayer('x', i-1, true)">
                层{{ i }} 顺时针
              </button>
              <button v-for="i in cubeConfig.size" :key="`x-${i}-cc`"
                      @click="rotateLayer('x', i-1, false)">
                层{{ i }} 逆时针
              </button>
            </div>

            <div class="axis-control">
              <h4>Y轴 (上下)</h4>
              <button v-for="i in cubeConfig.size" :key="`y-${i}`"
                      @click="rotateLayer('y', i-1, true)">
                层{{ i }} 顺时针
              </button>
              <button v-for="i in cubeConfig.size" :key="`y-${i}-cc`"
                      @click="rotateLayer('y', i-1, false)">
                层{{ i }} 逆时针
              </button>
            </div>

            <div class="axis-control">
              <h4>Z轴 (前后)</h4>
              <button v-for="i in cubeConfig.size" :key="`z-${i}`"
                      @click="rotateLayer('z', i-1, true)">
                层{{ i }} 顺时针
              </button>
              <button v-for="i in cubeConfig.size" :key="`z-${i}-cc`"
                      @click="rotateLayer('z', i-1, false)">
                层{{ i }} 逆时针
              </button>
            </div>
          </div>
        </div>

        <div class="instructions">
          <h3>操作说明</h3>
          <ul>
            <li>鼠标左键拖动：旋转视角</li>
            <li>鼠标右键拖动：平移视角</li>
            <li>鼠标滚轮：缩放</li>
            <li>点击按钮旋转魔方各层</li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.rubiks-cube-game {
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

.game-header {
  text-align: center;
  margin-bottom: 20px;
}

.game-header h1 {
  color: #333;
  margin-bottom: 10px;
}

.game-stats {
  display: flex;
  justify-content: center;
  gap: 30px;
}

.stat {
  background: #f5f5f5;
  padding: 10px 20px;
  border-radius: 8px;
  min-width: 100px;
}

.stat .label {
  display: block;
  color: #666;
  font-size: 14px;
}

.stat .value {
  display: block;
  font-size: 24px;
  font-weight: bold;
  color: #2196f3;
}

.game-container {
  display: flex;
  gap: 20px;
  max-width: 1400px;
  margin: 0 auto;
}

.canvas-container {
  flex: 1;
  min-height: 600px;
  min-width: 600px;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  overflow: hidden;
}

.controls-panel {
  width: 350px;
  background: #fff;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

.control-section {
  margin-bottom: 30px;
}

.control-section h3 {
  color: #333;
  margin-bottom: 15px;
  padding-bottom: 8px;
  border-bottom: 2px solid #2196f3;
}

button {
  display: block;
  width: 100%;
  padding: 12px;
  margin-bottom: 10px;
  background: #2196f3;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.3s;
}

button:hover:not(:disabled) {
  background: #1976d2;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.rotation-controls {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.axis-control {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 6px;
}

.axis-control h4 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #555;
}

.axis-control button {
  padding: 8px;
  font-size: 14px;
  margin-bottom: 5px;
}

.instructions {
  background: #e8f5e9;
  padding: 15px;
  border-radius: 6px;
}

.instructions h3 {
  color: #388e3c;
  border-bottom-color: #388e3c;
}

.instructions ul {
  padding-left: 20px;
  margin: 0;
}

.instructions li {
  margin-bottom: 8px;
  color: #555;
}
</style>
