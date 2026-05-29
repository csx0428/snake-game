<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

const difficulty = (route.query.difficulty as string) || 'easy'

const speedMap: Record<string, number> = {
  easy: 1000,
  medium: 400,
  hard: 400,
}

const moveInterval = speedMap[difficulty] ?? 1000
const maxFruits = difficulty === 'medium' ? 2 : 1

const shrinkThresholds: Record<string, number> = {
  easy: 3,
  medium: 7,
  hard: 5,
}
const shrinkThreshold = shrinkThresholds[difficulty] ?? 5

const obstacleSets: Point[][] = [
  [
    { x: 1, y: 3 },
    { x: 3, y: 1 },
  ],
  [
    { x: 1, y: 1 },
    { x: 3, y: 3 },
  ],
]

function pickObstacles(): Point[] {
  return obstacleSets[Math.floor(Math.random() * obstacleSets.length)]!.map((p) => ({ ...p }))
}

const obstacles = ref<Point[]>(difficulty === 'hard' ? pickObstacles() : [])

interface Point {
  x: number
  y: number
}

interface Fruit extends Point {
  type: string
  emoji: string
}

type Direction = 'UP' | 'DOWN' | 'LEFT' | 'RIGHT'

const direction = ref<Direction>('UP')
const nextDirection = ref<Direction>('UP')

const snake = ref<Point[]>([
  { x: 2, y: 2 },
  { x: 2, y: 3 },
  { x: 2, y: 4 },
])

const fruitTypes = [
  { type: 'apple', emoji: '🍎' },
  { type: 'orange', emoji: '🍊' },
  { type: 'lemon', emoji: '🍋' },
  { type: 'grape', emoji: '🍇' },
  { type: 'watermelon', emoji: '🍉' },
] as const

const fruits = ref<Fruit[]>([])
const rottenFruit = ref<Point | null>(null)
const shrinkFruit = ref<Point | null>(null)
const gameOver = ref(false)
const started = ref(false)
const paused = ref(false)
const score = ref(0)
let timer: ReturnType<typeof setInterval> | null = null

function initFruits() {
  const placed: Fruit[] = []
  const occupied = new Set(snake.value.map((p) => `${p.x},${p.y}`))
  obstacles.value.forEach((o) => occupied.add(`${o.x},${o.y}`))
  const shuffled = [...fruitTypes].sort(() => Math.random() - 0.5)

  for (let x = 0; x < 5; x++) {
    const pos = `${x},0`
    if (!occupied.has(pos)) {
      const ft = shuffled[x]!
      placed.push({ x, y: 0, type: ft.type, emoji: ft.emoji })
    }
  }
  fruits.value = placed
}

initFruits()

const fruitEmojis = ['🍎', '🍊', '🍋', '🍇', '🍉']

function frontCell(): Point | null {
  const head = snake.value[0]
  if (!head) return null
  const front: Point = { x: head.x, y: head.y }
  switch (direction.value) {
    case 'UP':    front.y--; break
    case 'DOWN':  front.y++; break
    case 'LEFT':  front.x--; break
    case 'RIGHT': front.x++; break
  }
  if (front.x < 0 || front.x >= 5 || front.y < 0 || front.y >= 5) return null
  return front
}

function spawnFruit() {
  while (fruits.value.length < maxFruits) {
    const occupied = new Set(snake.value.map((p) => `${p.x},${p.y}`))
    fruits.value.forEach((f) => occupied.add(`${f.x},${f.y}`))
    obstacles.value.forEach((o) => occupied.add(`${o.x},${o.y}`))
    if (rottenFruit.value) occupied.add(`${rottenFruit.value.x},${rottenFruit.value.y}`)
    if (shrinkFruit.value) occupied.add(`${shrinkFruit.value.x},${shrinkFruit.value.y}`)
    const fc = frontCell()
    if (fc) occupied.add(`${fc.x},${fc.y}`)
    let pos: Point
    do {
      pos = {
        x: Math.floor(Math.random() * 5),
        y: Math.floor(Math.random() * 5),
      }
    } while (occupied.has(`${pos.x},${pos.y}`))
    fruits.value.push({ ...pos, type: 'fruit', emoji: fruitEmojis[Math.floor(Math.random() * fruitEmojis.length)]! })
  }
}

function spawnRottenFruit() {
  if (difficulty !== 'hard') return
  const occupied = new Set(snake.value.map((p) => `${p.x},${p.y}`))
  fruits.value.forEach((f) => occupied.add(`${f.x},${f.y}`))
  obstacles.value.forEach((o) => occupied.add(`${o.x},${o.y}`))
  if (shrinkFruit.value) occupied.add(`${shrinkFruit.value.x},${shrinkFruit.value.y}`)
  const fc = frontCell()
  if (fc) occupied.add(`${fc.x},${fc.y}`)
  let pos: Point
  do {
    pos = {
      x: Math.floor(Math.random() * 5),
      y: Math.floor(Math.random() * 5),
    }
  } while (occupied.has(`${pos.x},${pos.y}`))
  rottenFruit.value = pos
}

function spawnShrinkFruit() {
  const bodyLength = snake.value.length - 1
  if (bodyLength < shrinkThreshold) {
    shrinkFruit.value = null
    return
  }
  const occupied = new Set(snake.value.map((p) => `${p.x},${p.y}`))
  fruits.value.forEach((f) => occupied.add(`${f.x},${f.y}`))
  obstacles.value.forEach((o) => occupied.add(`${o.x},${o.y}`))
  if (rottenFruit.value) occupied.add(`${rottenFruit.value.x},${rottenFruit.value.y}`)
  const fc = frontCell()
  if (fc) occupied.add(`${fc.x},${fc.y}`)
  let pos: Point
  do {
    pos = {
      x: Math.floor(Math.random() * 5),
      y: Math.floor(Math.random() * 5),
    }
  } while (occupied.has(`${pos.x},${pos.y}`))
  shrinkFruit.value = pos
}

function move() {
  if (gameOver.value) return

  direction.value = nextDirection.value
  const head = snake.value[0]
  if (!head) return

  const newHead: Point = { x: head.x, y: head.y }

  switch (direction.value) {
    case 'UP':    newHead.y--; break
    case 'DOWN':  newHead.y++; break
    case 'LEFT':  newHead.x--; break
    case 'RIGHT': newHead.x++; break
  }

  if (
    newHead.x < 0 || newHead.x >= 5 ||
    newHead.y < 0 || newHead.y >= 5
  ) {
    endGame()
    return
  }

  if (obstacles.value.some((o) => o.x === newHead.x && o.y === newHead.y)) {
    endGame()
    return
  }

  if (snake.value.some((s) => s.x === newHead.x && s.y === newHead.y)) {
    endGame()
    return
  }

  snake.value.unshift(newHead)

  const eatenShrink =
    shrinkFruit.value !== null &&
    shrinkFruit.value.x === newHead.x &&
    shrinkFruit.value.y === newHead.y

  if (eatenShrink) {
    shrinkFruit.value = null
    snake.value.pop()
    if (snake.value.length > 3) {
      snake.value.pop()
    }
    spawnShrinkFruit()
    return
  }

  const eatenRotten =
    rottenFruit.value !== null &&
    rottenFruit.value.x === newHead.x &&
    rottenFruit.value.y === newHead.y

  if (eatenRotten) {
    rottenFruit.value = null
    score.value = Math.max(0, score.value - 2)
    snake.value.pop()
    spawnRottenFruit()
    return
  }

  const eaten = fruits.value.findIndex((f) => f.x === newHead.x && f.y === newHead.y)
  if (eaten !== -1) {
    fruits.value.splice(eaten, 1)
    score.value++
    spawnShrinkFruit()
    if (fruits.value.length === 0) {
      spawnFruit()
      spawnRottenFruit()
    }
  } else {
    snake.value.pop()
  }
}

function endGame() {
  gameOver.value = true
  if (timer !== null) {
    clearInterval(timer)
    timer = null
  }
}

function changeDirection(dir: Direction) {
  const opposites: Record<Direction, Direction> = {
    UP: 'DOWN', DOWN: 'UP', LEFT: 'RIGHT', RIGHT: 'LEFT',
  }
  if (dir !== opposites[direction.value]) {
    nextDirection.value = dir
  }
}

function restart() {
  gameOver.value = false
  started.value = false
  paused.value = false
  score.value = 0
  obstacles.value = difficulty === 'hard' ? pickObstacles() : []
  rottenFruit.value = null
  shrinkFruit.value = null
  direction.value = 'UP'
  nextDirection.value = 'UP'
  snake.value = [
    { x: 2, y: 2 },
    { x: 2, y: 3 },
    { x: 2, y: 4 },
  ]
  initFruits()
  if (timer !== null) {
    clearInterval(timer)
    timer = null
  }
}

function handleKeydown(e: KeyboardEvent) {
  const keyMap: Record<string, Direction> = {
    ArrowUp: 'UP', ArrowDown: 'DOWN',
    ArrowLeft: 'LEFT', ArrowRight: 'RIGHT',
    w: 'UP', s: 'DOWN', a: 'LEFT', d: 'RIGHT',
  }

  if (gameOver.value) {
    const dir = keyMap[e.key]
    if (dir) {
      e.preventDefault()
      restart()
    }
    return
  }

  const dir = keyMap[e.key]
  if (dir) {
    e.preventDefault()
    if (!started.value) {
      started.value = true
      paused.value = false
      fruits.value = []
      spawnFruit()
      spawnRottenFruit()
      spawnShrinkFruit()
      nextDirection.value = dir
      timer = setInterval(move, moveInterval)
      return
    }
    changeDirection(dir)
    return
  }

  if (e.key === ' ' || e.key === 'Escape') {
    e.preventDefault()
    if (!started.value || gameOver.value) return
    if (paused.value) {
      paused.value = false
      timer = setInterval(move, moveInterval)
    } else {
      paused.value = true
      if (timer !== null) {
        clearInterval(timer)
        timer = null
      }
    }
  }
}

function isSnake(x: number, y: number) {
  return snake.value.some((seg) => seg.x === x && seg.y === y)
}

function isHead(x: number, y: number) {
  const head = snake.value[0]
  return head !== undefined && head.x === x && head.y === y
}

function getFruit(x: number, y: number) {
  return fruits.value.find((f) => f.x === x && f.y === y)
}

function isObstacle(x: number, y: number) {
  return obstacles.value.some((o) => o.x === x && o.y === y)
}

function isRottenFruit(x: number, y: number) {
  return rottenFruit.value !== null && rottenFruit.value.x === x && rottenFruit.value.y === y
}

function isShrinkFruit(x: number, y: number) {
  return shrinkFruit.value !== null && shrinkFruit.value.x === x && shrinkFruit.value.y === y
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  if (timer !== null) clearInterval(timer)
})
</script>

<template>
  <div class="game-screen">
    <header class="game-header">
      <button class="btn-back" @click="router.push('/')">← 離開</button>
      <div class="scoreboard">
        <span class="score-icon">🍎</span>
        <span class="score-num">{{ score }}</span>
      </div>
    </header>

    <main class="game-board-wrapper">
      <div class="game-board">
        <div v-for="row in 5" :key="row" class="row">
          <div
            v-for="col in 5"
            :key="col"
            class="cell"
            :class="{
              snake: isSnake(col - 1, row - 1),
              head: isHead(col - 1, row - 1),
              obstacle: isObstacle(col - 1, row - 1),
              'rotten-fruit': isRottenFruit(col - 1, row - 1),
              'shrink-fruit': isShrinkFruit(col - 1, row - 1),
            }"
          >
            <span v-if="isObstacle(col - 1, row - 1)" class="stone">🪨</span>
            <span v-else-if="isRottenFruit(col - 1, row - 1)" class="rotten">🫐</span>
            <span v-else-if="isShrinkFruit(col - 1, row - 1)" class="shrink">🌸</span>
            <span v-else-if="getFruit(col - 1, row - 1)" class="fruit">
              {{ getFruit(col - 1, row - 1)!.emoji }}
            </span>
          </div>
        </div>
      </div>
      <div v-if="!started && !gameOver" class="overlay">
        <div class="overlay-box">
          <h2>🐍</h2>
          <p class="start-hint">按下方向鍵開始遊戲</p>
        </div>
      </div>
      <div v-if="paused && !gameOver" class="overlay">
        <div class="overlay-box">
          <h2>已暫停</h2>
          <p>按空白鍵繼續</p>
        </div>
      </div>
      <div v-if="gameOver" class="overlay">
        <div class="overlay-box">
          <h2>遊戲結束</h2>
          <p class="final-score">得分：{{ score }}</p>
          <button class="btn-restart" @click="restart()">重新開始</button>
          <button class="btn-home" @click="router.push('/')">回首頁</button>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
.game-screen {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  background: linear-gradient(135deg, #0a0a1a 0%, #1a1a3e 50%, #0a0a1a 100%);
  font-family: 'Segoe UI', system-ui, sans-serif;
}

.game-header {
  width: 100%;
  max-width: 400px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  position: relative;
}

.btn-back {
  background: transparent;
  border: 2px solid #334466;
  color: #8899bb;
  padding: 6px 14px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
  transition: all 0.2s;
  font-family: inherit;
}

.btn-back:hover {
  border-color: #00ff88;
  color: #00ff88;
}

.scoreboard {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #111133;
  border: 2px solid #334466;
  border-radius: 12px;
  padding: 6px 18px;
}

.score-icon {
  font-size: 22px;
}

.score-num {
  font-size: 28px;
  font-weight: 800;
  color: #00ff88;
  min-width: 24px;
  text-align: center;
}

.game-board-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 0 16px;
  position: relative;
}

.game-board {
  display: flex;
  flex-direction: column;
  gap: 2px;
  padding: 8px;
  background: #111133;
  border: 2px solid #334466;
  border-radius: 8px;
  box-shadow: 0 0 30px rgba(0, 255, 136, 0.1);
}

.row {
  display: flex;
  gap: 2px;
}

.cell {
  width: 60px;
  height: 60px;
  background: #0d0d2b;
  border-radius: 4px;
  transition: background 0.1s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.cell.snake {
  background: #00aa55;
}

.cell.head {
  background: #00ff88;
  box-shadow: 0 0 12px rgba(0, 255, 136, 0.6);
}

.cell.obstacle {
  background: #2a1a0a;
}

.cell.rotten-fruit {
  background: #1a1a2a;
  animation: rottenPulse 2s ease-in-out infinite;
}

@keyframes rottenPulse {
  0%, 100% { box-shadow: inset 0 0 8px rgba(100, 100, 200, 0.2); }
  50% { box-shadow: inset 0 0 16px rgba(100, 100, 200, 0.4); }
}

.stone {
  font-size: 26px;
  pointer-events: none;
}

.rotten {
  font-size: 28px;
  animation: float 2.2s ease-in-out infinite;
  filter: drop-shadow(0 0 4px rgba(100, 100, 200, 0.4));
  pointer-events: none;
}

.cell.shrink-fruit {
  background: #1a0a2a;
  animation: shrinkPulse 2s ease-in-out infinite;
}

@keyframes shrinkPulse {
  0%, 100% { box-shadow: inset 0 0 10px rgba(180, 0, 255, 0.2); }
  50% { box-shadow: inset 0 0 20px rgba(180, 0, 255, 0.5); }
}

.shrink {
  font-size: 28px;
  animation: float 1.8s ease-in-out infinite;
  pointer-events: none;
}

.fruit {
  font-size: 28px;
  filter: drop-shadow(0 0 6px rgba(255, 255, 255, 0.3));
  animation: float 2s ease-in-out infinite;
  pointer-events: none;
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-3px); }
}

.overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(10, 10, 26, 0.85);
  border-radius: 8px;
}

.overlay-box {
  text-align: center;
  padding: 32px 48px;
}

.overlay-box h2 {
  font-size: 36px;
  color: #00ff88;
  margin-bottom: 20px;
  text-shadow: 0 0 20px rgba(0, 255, 136, 0.4);
}

.start-hint {
  color: #8899bb;
  font-size: 18px;
  animation: blink 1.5s ease-in-out infinite;
}

.final-score {
  font-size: 24px;
  font-weight: 700;
  color: #00ff88;
  margin-bottom: 20px;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}

.btn-restart {
  display: block;
  width: 100%;
  padding: 12px 40px;
  margin-bottom: 10px;
  font-size: 18px;
  font-weight: 700;
  color: #0a0a1a;
  background: linear-gradient(135deg, #00ff88, #00cc66);
  border: none;
  border-radius: 50px;
  cursor: pointer;
  font-family: inherit;
  box-shadow: 0 4px 20px rgba(0, 255, 136, 0.3);
}

.btn-restart:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 25px rgba(0, 255, 136, 0.5);
}

.btn-home {
  display: block;
  width: 100%;
  padding: 12px 40px;
  font-size: 18px;
  font-weight: 700;
  color: #8899bb;
  background: transparent;
  border: 2px solid #334466;
  border-radius: 50px;
  cursor: pointer;
  font-family: inherit;
}

.btn-home:hover {
  border-color: #00ff88;
  color: #00ff88;
}
</style>
