<template>
  <div class="puzzle-game">
    <h2>拼图游戏</h2>
    <input type="file" @change="handleFileUpload" accept="image/*" />
    <div class="bnt">
      <button @click="resetPuzzle">一键复原</button>
      <button @click="generatePuzzle">打乱</button>
      <div>{{board}}</div>
    </div>
    <div v-if="imageSrc" class="puzzle-board">
      <div v-for="(row, rowIndex) in board" :key="rowIndex" class="puzzle-row">
        <div
          v-for="(cell, colIndex) in row"
          :key="colIndex"
          :class="['puzzle-cell', { 'empty': cell === -1 }]"|
          @click="move(rowIndex, colIndex, cell)"
          :style="{
            backgroundImage: cell === -1 ? 'none' : `url(${imageSrc})`,
            backgroundPosition: cell !== -1 ? `${-initialPositions[cell]?.col * cellSize}px ${-initialPositions[cell]?.row * cellSize}px` : '0 0',
            backgroundSize: `${size * cellSize}px ${size * cellSize}px`
          }"
        >
          <span v-if="cell !== 0" class="cell-number">{{rowIndex}}-{{colIndex}}</span>
        </div>
      </div>
    </div>
    <div v-if="isSolved" class="victory-modal">
      <p>恭喜你，拼图成功！</p>
      <button @click="refreshPage">重新开始</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import {ref, onMounted, computed, watch} from 'vue';

interface Position {
  x: number;
  y: number;
}

const size = 3; // 拼图的大小
const cellSize = 100; // 每个格子的大小
const totle = size * size ;//格子总数

// 初始化拼图板
const board = ref<number[][]>(Array.from({ length: size }, () => Array(size).fill(0)));

// 用于存储打乱后的拼图块位置
const shuffledPositions = ref<Array<{ row: number; col: number } | null>>([]);

// 空格位置
const emptyPos = ref<Position>({ x: 0, y: 0 });

// 图片上传处理
const imageSrc = ref<string | null>(null);

// 是否已解决拼图
const isSolved = ref(false);

// 生成初始位置，重置数组
const initialPositions: Array<{ row: number; col: number } | null> = Object.freeze(Array.from(
    {length: size * size},
    (_, idx) => ({row: Math.floor(idx / size), col: idx % size})
))

// 生成拼图
function generatePuzzle() {
  const numbers = Array.from({length: size * size - 1}, (_, i) => i);
  numbers.push(-1); // 空格
  shuffle(numbers);
  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      if (board.value && board.value[i]) {
        board.value[i][j] = numbers[i * size + j];
      }
    }
  }
  // 找到空格的位置F
  for (let i = 0; i < size; i++) {
    if (board.value && board.value[i]) {
      for (let j = 0; j < size; j++) {
        if (board.value[i][j] === -1) {
          emptyPos.value = { x: i, y: j };
          return;
        }
      }
    }
  }
}

// Fisher-Yates 洗牌算法
function shuffle(array: number[]): number[] {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

// 打乱数组
function shuffleArray<T>(array: T[]): T[] {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

// 移动拼图块
function move(row: number, col: number) {
  if (!board.value || !emptyPos.value) return;

  // 检查点击位置是否与-1相邻
  if (Math.abs(emptyPos.value.x - row) + Math.abs(emptyPos.value.y - col) !== 1) return;

  // 交换两个位置的值
  [board.value[emptyPos.value.x][emptyPos.value.y], board.value[row][col]] =
      [board.value[row][col], board.value[emptyPos.value.x][emptyPos.value.y]];

  // 更新空格位置
  emptyPos.value = { x: row, y: col };
  // 检查是否完成拼图
  checkIfSolved()
}
watch(board, () => {
  checkIfSolved()
});


// 检查拼图是否已解决
function checkIfSolved() {
  const solvedBoard = Array.from({ length: size }, (_, i) => Array(size).fill(0));
  let currentNumber = 0;
  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      if (i === size - 1 && j === size - 1) {
        solvedBoard[i][j] = -1;
      } else {
        solvedBoard[i][j] = currentNumber++;
      }
    }
  }
  if (JSON.stringify(board.value) === JSON.stringify(solvedBoard)) {
    isSolved.value = true;
  }
}

// 重置拼图
function resetPuzzle() {
  const solvedBoard = Array.from({ length: size }, (_, i) => Array(size).fill(0));
  let currentNumber = 0;
  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      if (i === size - 1 && j === size - 1) {
        solvedBoard[i][j] = -1;
      } else {
        solvedBoard[i][j] = currentNumber++;
      }
    }
  }

  board.value = solvedBoard;
  shuffledPositions.value = initialPositions
  for (let i = 0; i < size; i++) {
    if (board.value && board.value[i]) {
      for (let j = 0; j < size; j++) {
        if (board.value[i][j] === -1) {
          emptyPos.value = { x: i, y: j };
          return;
        }
      }
    }
  }
  isSolved.value = false;
}

// 刷新页面
function refreshPage() {
  window.location.reload();
}

onMounted(() => {
  generatePuzzle();
});

// 图片上传处理
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
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
}

.cell-number {
  position: absolute;
  font-size: 24px;
  color: #fff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

.empty {
  background-color: #fff;
}

.victory-modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: #2b2d30;
  padding: 20px;
  border: 1px solid #000;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  z-index: 1000;
}

.victory-modal p {
  margin-bottom: 10px;
}

.victory-modal button {
  padding: 10px 20px;
  background-color: #007bff;
  color: #fff;
  border: none;
  cursor: pointer;
}

.victory-modal button:hover {
  background-color: #0056b3;
}

input[type="file"] {
  margin-bottom: 20px;
}

.bnt{
  display: flex;
  button{
    margin: 5px;
  }
}
</style>
