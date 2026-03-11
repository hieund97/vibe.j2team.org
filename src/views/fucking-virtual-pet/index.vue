<script setup lang="ts">
import { Icon } from '@iconify/vue'
import { useIntervalFn } from '@vueuse/core'
import { computed, onUnmounted, ref } from 'vue'

// ─── State ────────────────────────────────────────────────
const hunger = ref(80)
const energy = ref(80)
const pokeDialogue = ref('')
const pokeTimeout = ref<ReturnType<typeof setTimeout> | null>(null)
const isShaking = ref(false)

// ─── Auto decay every 3 seconds ──────────────────────────
const { pause } = useIntervalFn(
  () => {
    hunger.value = Math.max(0, hunger.value - 5)
    energy.value = Math.max(0, energy.value - 5)
  },
  3000,
  { immediate: true },
)
onUnmounted(() => pause())

// ─── Computed states ─────────────────────────────────────
const isDropped = computed(() => hunger.value < 30)
const isBloated = computed(() => hunger.value > 100)
const isSleepy = computed(() => energy.value < 20)

const petScale = computed(() => {
  if (isBloated.value) {
    const extra = (hunger.value - 100) / 50
    return 1.5 + extra * 0.3
  }
  return 1
})

// ─── Dialogue system ─────────────────────────────────────
const normalDialogues = [
  'Bay lơ lửng chill phết 🌙',
  'Code đi, nhìn cái gì? 👀',
  'Không gian vũ trụ chill thế 🚀',
  'Ê, đưa miếng cá đây coi 🐟',
  'Mày tắm chưa mà ngồi code? 🤔',
  'Sao tao biết bay mà mày không? 😏',
]

const hungryDialogues = [
  'Đói rã ruột rồi, tính bỏ đói tôi à? 😤',
  'Cho xin miếng cơm đi con sen! 🍚',
  'Mày ăn KFC mà không chia à? 😡',
  'Đói quá bay hết nổi rồi... 💀',
]

const bloatedDialogues = [
  'No quá trương phình rồi, mớm nữa là nổ đấy! 🎈',
  'Bụng tròn như cái bánh bao! 😵',
  'Tao sắp nổ như bom tấn! 💣',
]

const sleepyDialogues = [
  'Buồn ngủ rũ cả mắt... 😴',
  'Mở nhạc Lofi lên rồi tắt đèn hộ cái 🎧',
  'Zzz... đừng... chọc... Zzz 💤',
]

const pokeDialogues = [
  'Đừng có đụng vào cái body ngọc ngà này! 😾',
  'Rảnh rỗi sinh nông nổi à? 🙄',
  'Chọc nữa tao cào cho sấp mặt! 🐾',
  'HANDS OFF, con người! 🖐️',
]

function pickRandom(arr: string[]): string {
  return arr[Math.floor(Math.random() * arr.length)] ?? arr[0] ?? ''
}

const currentDialogue = computed(() => {
  if (pokeDialogue.value) return pokeDialogue.value
  if (isBloated.value) return pickRandom(bloatedDialogues)
  if (isDropped.value) return pickRandom(hungryDialogues)
  if (isSleepy.value) return pickRandom(sleepyDialogues)
  return pickRandom(normalDialogues)
})

// ─── Actions ─────────────────────────────────────────────
function feed() {
  hunger.value = Math.min(150, hunger.value + 15)
}

function sleep() {
  energy.value = Math.min(100, energy.value + 20)
}

function poke() {
  energy.value = Math.max(0, energy.value - 10)
  pokeDialogue.value = pickRandom(pokeDialogues)
  isShaking.value = true

  if (pokeTimeout.value) clearTimeout(pokeTimeout.value)
  pokeTimeout.value = setTimeout(() => {
    pokeDialogue.value = ''
    isShaking.value = false
    pokeTimeout.value = null
  }, 2000)
}

onUnmounted(() => {
  if (pokeTimeout.value) clearTimeout(pokeTimeout.value)
})

// ─── Status bar helpers ──────────────────────────────────
function statusColor(value: number): string {
  if (value > 60) return 'bg-emerald-400'
  if (value > 30) return 'bg-amber-400'
  return 'bg-red-400'
}

function statusEmoji(type: 'hunger' | 'energy', value: number): string {
  if (type === 'hunger') {
    if (value > 100) return '🤢'
    if (value > 60) return '😋'
    if (value > 30) return '😐'
    return '😩'
  }
  if (value > 60) return '⚡'
  if (value > 30) return '😮‍💨'
  return '😴'
}
</script>

<template>
  <div class="pet-page">
    <!-- Starfield Background -->
    <div class="stars-container">
      <div
        v-for="n in 60"
        :key="n"
        class="star"
        :style="{
          left: `${Math.random() * 100}%`,
          top: `${Math.random() * 100}%`,
          animationDelay: `${Math.random() * 3}s`,
          animationDuration: `${2 + Math.random() * 3}s`,
          width: `${1 + Math.random() * 2}px`,
          height: `${1 + Math.random() * 2}px`,
        }"
      />
    </div>

    <!-- Header -->
    <div class="pet-header animate-fade-up">
      <RouterLink
        to="/"
        class="back-link"
      >
        <Icon
          icon="lucide:arrow-left"
          class="size-4"
        />
        Trang chủ
      </RouterLink>
      <h1 class="page-title">
        <span class="title-emoji">🐱</span>
        Thú ảo Mỏ Hỗn
      </h1>
    </div>

    <!-- Pet Arena -->
    <div class="arena animate-fade-up animate-delay-1">
      <!-- Speech Bubble -->
      <div
        class="speech-bubble"
        :class="{ dropped: isDropped }"
      >
        <span class="bubble-text">{{ currentDialogue }}</span>
        <div class="bubble-tail" />
      </div>

      <!-- The Pet -->
      <div
        class="pet-wrapper"
        :class="{
          floating: !isDropped,
          dropped: isDropped,
          shaking: isShaking,
          bloated: isBloated,
        }"
        :style="{ transform: `scale(${petScale})` }"
      >
        <span class="pet-emoji">🐱</span>
      </div>

      <!-- Ground line when dropped -->
      <Transition name="fade">
        <div
          v-if="isDropped"
          class="ground-line"
        />
      </Transition>
    </div>

    <!-- Status Bars -->
    <div class="status-panel animate-fade-up animate-delay-2">
      <div class="stat-row">
        <span class="stat-label">
          {{ statusEmoji('hunger', hunger) }} Độ no
        </span>
        <div class="stat-bar-bg">
          <div
            class="stat-bar-fill"
            :class="statusColor(Math.min(hunger, 100))"
            :style="{ width: `${Math.min(hunger, 100)}%` }"
          />
        </div>
        <span class="stat-value">{{ hunger }}</span>
      </div>
      <div class="stat-row">
        <span class="stat-label">
          {{ statusEmoji('energy', energy) }} Năng lượng
        </span>
        <div class="stat-bar-bg">
          <div
            class="stat-bar-fill"
            :class="statusColor(energy)"
            :style="{ width: `${energy}%` }"
          />
        </div>
        <span class="stat-value">{{ energy }}</span>
      </div>
    </div>

    <!-- Action Buttons -->
    <div class="actions animate-fade-up animate-delay-3">
      <button
        class="action-btn feed-btn"
        @click="feed"
      >
        <span class="btn-icon">🐟</span>
        <span class="btn-label">Cho ăn</span>
      </button>
      <button
        class="action-btn sleep-btn"
        @click="sleep"
      >
        <span class="btn-icon">💤</span>
        <span class="btn-label">Đi ngủ</span>
      </button>
      <button
        class="action-btn poke-btn"
        @click="poke"
      >
        <span class="btn-icon">👈</span>
        <span class="btn-label">Chọc phá</span>
      </button>
    </div>
  </div>
</template>

<style scoped>
/* ─── Page ─────────────────────────────────────────────── */
.pet-page {
  min-height: 100vh;
  background: var(--color-bg-deep);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1.5rem 1rem 2rem;
  position: relative;
  overflow: hidden;
}

/* ─── Starfield ────────────────────────────────────────── */
.stars-container {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 0;
}

.star {
  position: absolute;
  border-radius: 50%;
  background: #f0ede6;
  animation: twinkle ease-in-out infinite alternate;
}

@keyframes twinkle {
  0% {
    opacity: 0.15;
  }
  100% {
    opacity: 0.7;
  }
}

/* ─── Header ───────────────────────────────────────────── */
.pet-header {
  position: relative;
  z-index: 10;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  width: 100%;
  max-width: 480px;
  margin-bottom: 1rem;
}

.back-link {
  align-self: flex-start;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  color: var(--color-text-dim);
  font-family: var(--font-display);
  font-size: 0.8rem;
  letter-spacing: 0.05em;
  text-decoration: none;
  transition: color 0.2s;
}
.back-link:hover {
  color: var(--color-text-primary);
}

.page-title {
  font-family: var(--font-display);
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--color-text-primary);
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
.title-emoji {
  font-size: 1.8rem;
}

/* ─── Arena ────────────────────────────────────────────── */
.arena {
  position: relative;
  z-index: 10;
  width: 100%;
  max-width: 480px;
  height: 320px;
  border: 1px solid var(--color-border-default);
  background: rgba(22, 34, 50, 0.6);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin-bottom: 1.5rem;
  overflow: hidden;
}

/* ─── Speech Bubble ────────────────────────────────────── */
.speech-bubble {
  position: absolute;
  top: 24px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 16px;
  padding: 0.6rem 1rem;
  max-width: 85%;
  text-align: center;
  z-index: 20;
  transition: all 0.5s ease;
}

.speech-bubble.dropped {
  top: auto;
  bottom: 100px;
}

.bubble-text {
  font-family: var(--font-body);
  font-size: 0.85rem;
  color: var(--color-text-primary);
  white-space: nowrap;
}

.bubble-tail {
  position: absolute;
  bottom: -8px;
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-top: 8px solid rgba(255, 255, 255, 0.1);
}

/* ─── Pet ──────────────────────────────────────────────── */
.pet-wrapper {
  transition:
    transform 0.5s ease,
    bottom 0.8s cubic-bezier(0.68, -0.55, 0.27, 1.55);
  position: absolute;
  z-index: 15;
}

.pet-emoji {
  font-size: 5rem;
  display: block;
  filter: drop-shadow(0 0 20px rgba(255, 184, 48, 0.3));
}

/* Floating animation */
.floating {
  animation: float 6s ease-in-out infinite;
}

@keyframes float {
  0%,
  100% {
    transform: translate(0, 0) rotate(0deg);
  }
  15% {
    transform: translate(30px, -25px) rotate(5deg);
  }
  30% {
    transform: translate(-20px, -40px) rotate(-3deg);
  }
  45% {
    transform: translate(25px, -15px) rotate(4deg);
  }
  60% {
    transform: translate(-30px, 10px) rotate(-5deg);
  }
  75% {
    transform: translate(15px, -30px) rotate(2deg);
  }
  90% {
    transform: translate(-10px, 5px) rotate(-2deg);
  }
}

/* Dropped to bottom */
.dropped {
  animation: none !important;
  bottom: 16px !important;
  top: auto !important;
}

/* Shake animation */
.shaking {
  animation: shake 0.4s ease-in-out !important;
}

@keyframes shake {
  0%,
  100% {
    transform: translateX(0) rotate(0deg);
  }
  10% {
    transform: translateX(-12px) rotate(-8deg);
  }
  20% {
    transform: translateX(12px) rotate(8deg);
  }
  30% {
    transform: translateX(-10px) rotate(-6deg);
  }
  40% {
    transform: translateX(10px) rotate(6deg);
  }
  50% {
    transform: translateX(-6px) rotate(-3deg);
  }
  60% {
    transform: translateX(6px) rotate(3deg);
  }
  70% {
    transform: translateX(-3px) rotate(-1deg);
  }
  80% {
    transform: translateX(3px) rotate(1deg);
  }
}

/* Bloated */
.bloated .pet-emoji {
  filter: drop-shadow(0 0 30px rgba(255, 107, 74, 0.6));
}

/* Ground line */
.ground-line {
  position: absolute;
  bottom: 12px;
  left: 10%;
  right: 10%;
  height: 2px;
  background: linear-gradient(
    90deg,
    transparent,
    var(--color-accent-coral),
    transparent
  );
  opacity: 0.4;
}

/* ─── Status Panel ─────────────────────────────────────── */
.status-panel {
  position: relative;
  z-index: 10;
  width: 100%;
  max-width: 480px;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
  border: 1px solid var(--color-border-default);
  background: var(--color-bg-surface);
  padding: 1rem 1.25rem;
}

.stat-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.stat-label {
  font-family: var(--font-display);
  font-size: 0.8rem;
  font-weight: 600;
  color: var(--color-text-secondary);
  min-width: 110px;
  white-space: nowrap;
}

.stat-bar-bg {
  flex: 1;
  height: 10px;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 100px;
  overflow: hidden;
}

.stat-bar-fill {
  height: 100%;
  border-radius: 100px;
  transition: width 0.4s ease;
}

.stat-value {
  font-family: var(--font-display);
  font-size: 0.8rem;
  font-weight: 700;
  color: var(--color-text-primary);
  min-width: 28px;
  text-align: right;
}

/* ─── Action Buttons ───────────────────────────────────── */
.actions {
  position: relative;
  z-index: 10;
  display: flex;
  gap: 0.75rem;
  width: 100%;
  max-width: 480px;
}

.action-btn {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.35rem;
  padding: 0.85rem 0.5rem;
  border: 1px solid var(--color-border-default);
  background: var(--color-bg-surface);
  color: var(--color-text-primary);
  cursor: pointer;
  transition: all 0.25s ease;
  font-family: var(--font-display);
}

.action-btn:hover {
  border-color: var(--color-accent-coral);
  background: var(--color-bg-elevated);
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(255, 107, 74, 0.1);
}
.action-btn:active {
  transform: translateY(0) scale(0.97);
}

.btn-icon {
  font-size: 1.5rem;
}

.btn-label {
  font-size: 0.75rem;
  font-weight: 600;
  letter-spacing: 0.03em;
}

.feed-btn:hover {
  border-color: var(--color-accent-amber);
  box-shadow: 0 4px 16px rgba(255, 184, 48, 0.15);
}

.sleep-btn:hover {
  border-color: var(--color-accent-sky);
  box-shadow: 0 4px 16px rgba(56, 189, 248, 0.15);
}

.poke-btn:hover {
  border-color: var(--color-accent-coral);
  box-shadow: 0 4px 16px rgba(255, 107, 74, 0.15);
}

/* ─── Transitions ──────────────────────────────────────── */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.4s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* ─── Responsive ───────────────────────────────────────── */
@media (max-width: 480px) {
  .arena {
    height: 280px;
  }

  .pet-emoji {
    font-size: 4rem;
  }

  .bubble-text {
    font-size: 0.75rem;
    white-space: normal;
  }

  .action-btn {
    padding: 0.7rem 0.35rem;
  }

  .btn-icon {
    font-size: 1.3rem;
  }

  .btn-label {
    font-size: 0.7rem;
  }
}
</style>
