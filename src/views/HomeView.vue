<script setup lang="ts">
import { ref } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const selectedDifficulty = ref<'easy' | 'medium' | 'hard'>('easy')

const difficulties = [
  { key: 'easy', label: '簡單' },
  { key: 'medium', label: '中等' },
  { key: 'hard', label: '困難' },
] as const

function startGame() {
  router.push(`/game?difficulty=${selectedDifficulty.value}`)
}
</script>

<template>
  <div class="home">
    <div class="snake-decoration top"></div>
    <div class="snake-decoration bottom"></div>

    <div class="content">
      <div class="logo">
        <span class="snake-icon">🐍</span>
      </div>

      <h1 class="title">貪吃蛇</h1>
      <p class="subtitle">經典懷舊小遊戲</p>

      <div class="menu">
        <button class="btn-start" @click="startGame">
          <span class="btn-icon">▶</span>
          開始遊戲
        </button>

        <div class="difficulty">
          <span class="diff-label">難度</span>
          <div class="diff-options">
            <button
              v-for="d in difficulties"
              :key="d.key"
              class="diff-btn"
              :class="{ active: selectedDifficulty === d.key }"
              @click="selectedDifficulty = d.key"
            >
              {{ d.label }}
            </button>
          </div>
        </div>
      </div>

    </div>

    <div class="controls-hint">
      <p>使用 <kbd>←</kbd> <kbd>↑</kbd> <kbd>↓</kbd> <kbd>→</kbd> 控制方向</p>
    </div>
  </div>
</template>

<style scoped>
.home {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #0a0a1a 0%, #1a1a3e 50%, #0a0a1a 100%);
  position: relative;
  overflow: hidden;
  font-family: 'Segoe UI', system-ui, sans-serif;
}

.snake-decoration {
  position: absolute;
  width: 200%;
  height: 4px;
  background: repeating-linear-gradient(
    90deg,
    #00ff88 0px,
    #00ff88 20px,
    transparent 20px,
    transparent 40px
  );
  opacity: 0.3;
  animation: scroll 3s linear infinite;
}

.snake-decoration.top {
  top: 60px;
}

.snake-decoration.bottom {
  bottom: 60px;
  animation-direction: reverse;
}

@keyframes scroll {
  from { transform: translateX(0); }
  to { transform: translateX(-50%); }
}

.content {
  text-align: center;
  z-index: 1;
}

.logo {
  margin-bottom: 16px;
}

.snake-icon {
  font-size: 80px;
  display: inline-block;
  animation: bounce 2s ease-in-out infinite;
  filter: drop-shadow(0 0 20px rgba(0, 255, 136, 0.5));
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-12px); }
}

.title {
  font-size: 64px;
  font-weight: 800;
  color: #00ff88;
  margin: 0;
  text-shadow: 0 0 30px rgba(0, 255, 136, 0.5), 0 0 60px rgba(0, 255, 136, 0.2);
  letter-spacing: 8px;
}

.subtitle {
  font-size: 18px;
  color: #8899bb;
  margin: 8px 0 48px;
  letter-spacing: 4px;
}

.menu {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
}

.btn-start {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 16px 56px;
  font-size: 22px;
  font-weight: 700;
  color: #0a0a1a;
  background: linear-gradient(135deg, #00ff88, #00cc66);
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 20px rgba(0, 255, 136, 0.3);
  letter-spacing: 2px;
}

.btn-start:hover {
  transform: translateY(-3px) scale(1.02);
  box-shadow: 0 8px 30px rgba(0, 255, 136, 0.5);
}

.btn-start:active {
  transform: translateY(0) scale(0.98);
}

.btn-icon {
  font-size: 18px;
}

.difficulty {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}

.diff-label {
  color: #667799;
  font-size: 14px;
  letter-spacing: 2px;
}

.diff-options {
  display: flex;
  gap: 8px;
}

.diff-btn {
  padding: 8px 24px;
  font-size: 14px;
  font-weight: 600;
  color: #667799;
  background: transparent;
  border: 2px solid #334466;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.2s;
}

.diff-btn:hover {
  border-color: #00ff88;
  color: #00ff88;
}

.diff-btn.active {
  background: #00ff88;
  border-color: #00ff88;
  color: #0a0a1a;
}

.controls-hint {
  position: absolute;
  bottom: 90px;
  color: #445566;
  font-size: 13px;
}

.controls-hint kbd {
  display: inline-block;
  padding: 2px 8px;
  background: #1a1a3e;
  border: 1px solid #334466;
  border-radius: 4px;
  font-family: inherit;
  color: #8899bb;
  font-size: 12px;
}

@media (max-width: 480px) {
  .title {
    font-size: 40px;
    letter-spacing: 4px;
  }

  .snake-icon {
    font-size: 56px;
  }

  .btn-start {
    padding: 14px 40px;
    font-size: 18px;
  }
}
</style>
