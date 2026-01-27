<template>
  <div class="puzzle-game">
    <h2>拼图游戏</h2>
    <input type="file" @change="handleFileUpload" accept="image/*" />
    <div v-if="imageSrc" class="puzzle-board">
      <div v-for="(row, rowIndex) in board" :key="rowIndex" class="puzzle-row">
        <div
          v-for="(cell, colIndex) in row"
          :key="colIndex"
          :class="['puzzle-cell', { 'empty': cell === 0 }]"
          @click="move(rowIndex, colIndex)"
          :style="{
            backgroundImage: `url(${imageSrc})`,
            backgroundPosition: `${-colIndex * cellSize}px ${-rowIndex * cellSize}px`,
            backgroundSize: `${size * cellSize}px ${size * cellSize}px`
          }"
        ></div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';

interface Position {
  x: number;
  y: number;
}

const size = 10; // 拼图的大小
const cellSize = 100; // 每个格子的大小

// 初始化拼图板
const board = ref<number[][]>(Array.from({ length: size }, () => Array(size).fill(0)));

// 生成拼图
function generatePuzzle() {
  const numbers = Array.from({ length: size * size - 1 }, (_, i) => i + 1);
  numbers.push(0); // 空格
  shuffle(numbers);

  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      if (board.value && board.value[i]) {
        board.value[i][j] = numbers[i * size + j];
      }
    }
  }

  // 找到空格的位置
  for (let i = 0; i < size; i++) {
    if (board.value && board.value[i]) {
      for (let j = 0; j < size; j++) {
        if (board.value[i][j] === 0) {
          emptyPos.value = { x: i, y: j };
          return;
        }
      }
    }
  }
}

// Fisher-Yates 洗牌算法
function shuffle(array: number[]) {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
}

const emptyPos = ref<Position>({ x: 0, y: 0 });

function move(row: number, col: number) {
  if (!board.value || !emptyPos.value) return;

  if (Math.abs(emptyPos.value.x - row) + Math.abs(emptyPos.value.y - col) !== 1) return;

  const temp = board.value[emptyPos.value.x][emptyPos.value.y];
  board.value[emptyPos.value.x][emptyPos.value.y] = board.value[row][col];
  board.value[row][col] = temp;
  emptyPos.value = { x: row, y: col };
}

onMounted(() => {
  generatePuzzle();
});

// 图片上传处理
const imageSrc = ref<string | null>(null);

function handleFileUpload(event: Event) {
  const input = event.target as HTMLInputElement;
  if (input.files && input.files[0]) {
    const file = input.files[0];
    const reader = new FileReader();
    reader.onload = (e: ProgressEvent<FileReader>) => {
      if (e.target && e.target.result) {
        imageSrc.value = e.target.result as string;
        generatePuzzle(); // 重新生成拼图
      }
    };
    reader.readAsDataURL(file);
  }
}
</script>

<style scoped>
.puzzle-game {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.puzzle-board {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.puzzle-row {
  display: flex;
}

.puzzle-cell {
  width: 100px; /* 每个格子的宽度 */
  height: 100px; /* 每个格子的高度 */
  border: 1px solid #000;
  cursor: pointer;
  background-size: cover;
  background-repeat: no-repeat;
}

.empty {
  background-color: #fff;
}
</style>
