<script setup lang="ts">
import { Icon } from '@iconify/vue'
import { useEventListener, useIntervalFn, useLocalStorage } from '@vueuse/core'
import { computed, onUnmounted, ref, watch } from 'vue'

// ─── Timer Modes ───────────────────────────────────────────
type TimerMode = 'pomodoro' | 'shortBreak' | 'longBreak'

const DURATIONS: Record<TimerMode, number> = {
  pomodoro: 25 * 60,
  shortBreak: 5 * 60,
  longBreak: 15 * 60,
}

const MODE_LABELS: Record<TimerMode, string> = {
  pomodoro: 'Tập trung',
  shortBreak: 'Nghỉ ngắn',
  longBreak: 'Nghỉ dài',
}

const MODE_ICONS: Record<TimerMode, string> = {
  pomodoro: 'lucide:brain',
  shortBreak: 'lucide:coffee',
  longBreak: 'lucide:sunset',
}

const mode = ref<TimerMode>('pomodoro')
const timeLeft = ref(DURATIONS.pomodoro)
const isRunning = ref(false)
const pomodoroCount = ref(0)
const showModeTransition = ref(true)

const displayTime = computed(() => {
  const m = Math.floor(timeLeft.value / 60)
  const s = timeLeft.value % 60
  return `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`
})

const progress = computed(() => {
  const total = DURATIONS[mode.value]
  return ((total - timeLeft.value) / total) * 100
})

// SVG circular progress
const RADIUS = 120
const CIRCUMFERENCE = 2 * Math.PI * RADIUS
const strokeDashoffset = computed(
  () => CIRCUMFERENCE - (progress.value / 100) * CIRCUMFERENCE,
)

function switchMode(newMode: TimerMode) {
  showModeTransition.value = false
  setTimeout(() => {
    mode.value = newMode
    timeLeft.value = DURATIONS[newMode]
    isRunning.value = false
    showModeTransition.value = true
  }, 150)
}

function getModeIcon(key: string): string {
  return MODE_ICONS[key as TimerMode]
}

function handleModeClick(key: string) {
  ensureSoundsInit()
  switchMode(key as TimerMode)
}

function handleSliderInput(sound: AmbientSound, e: Event) {
  ensureSoundsInit()
  const target = e.target as HTMLInputElement
  sound.volume = Number(target.value)
  onVolumeChange(sound)
}

function toggleTimer() {
  // Resume AudioContext on user gesture (browser autoplay policy)
  if (audioCtx && audioCtx.state === 'suspended') {
    audioCtx.resume()
  }
  isRunning.value = !isRunning.value
}

function resetTimer() {
  isRunning.value = false
  timeLeft.value = DURATIONS[mode.value]
}

function playNotificationBeep() {
  if (!audioCtx) return
  const osc = audioCtx.createOscillator()
  const gain = audioCtx.createGain()
  osc.connect(gain)
  gain.connect(audioCtx.destination)
  osc.frequency.value = 830
  osc.type = 'sine'
  gain.gain.setValueAtTime(0.3, audioCtx.currentTime)
  gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.8)
  osc.start(audioCtx.currentTime)
  osc.stop(audioCtx.currentTime + 0.8)
}

function onTimerComplete() {
  isRunning.value = false
  playNotificationBeep()

  if (mode.value === 'pomodoro') {
    pomodoroCount.value++
    // Every 4 pomodoros → long break
    if (pomodoroCount.value % 4 === 0) {
      switchMode('longBreak')
    } else {
      switchMode('shortBreak')
    }
  } else {
    switchMode('pomodoro')
  }
}

const { pause: pauseInterval, resume: resumeInterval } = useIntervalFn(
  () => {
    if (timeLeft.value > 0) {
      timeLeft.value--
    } else {
      onTimerComplete()
    }
  },
  1000,
  { immediate: false },
)

watch(isRunning, (val) => {
  if (val) {
    resumeInterval()
  } else {
    pauseInterval()
  }
})

// ─── Ambient Sounds (Web Audio API) ───────────────────────
interface AmbientSound {
  id: string
  name: string
  icon: string
  volume: number
  gainNode: GainNode | null
  sourceNode: AudioNode | null
  isPlaying: boolean
}

let audioCtx: AudioContext | null = null

function getAudioContext(): AudioContext {
  if (!audioCtx) {
    audioCtx = new AudioContext()
  }
  return audioCtx
}

// Persisted volumes
const savedVolumes = useLocalStorage<Record<string, number>>(
  'lofi-pomodoro-volumes',
  {
    rain: 0,
    cafe: 0,
    fireplace: 0,
    birds: 0,
    library: 0,
    train: 0,
    ocean: 0,
    keyboard: 0,
  },
)

const sounds = ref<AmbientSound[]>([
  {
    id: 'rain',
    name: 'Mưa rào',
    icon: 'lucide:cloud-rain',
    volume: savedVolumes.value.rain ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
  {
    id: 'cafe',
    name: 'Quán cà phê',
    icon: 'lucide:cup-soda',
    volume: savedVolumes.value.cafe ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
  {
    id: 'fireplace',
    name: 'Lửa ấm',
    icon: 'lucide:flame',
    volume: savedVolumes.value.fireplace ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
  {
    id: 'birds',
    name: 'Chim hót',
    icon: 'lucide:bird',
    volume: savedVolumes.value.birds ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
  {
    id: 'library',
    name: 'Thư viện',
    icon: 'lucide:book-open',
    volume: savedVolumes.value.library ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
  {
    id: 'train',
    name: 'Tàu hỏa',
    icon: 'lucide:train-front',
    volume: savedVolumes.value.train ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
  {
    id: 'ocean',
    name: 'Sóng biển',
    icon: 'lucide:waves',
    volume: savedVolumes.value.ocean ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
  {
    id: 'keyboard',
    name: 'Bàn phím',
    icon: 'lucide:keyboard',
    volume: savedVolumes.value.keyboard ?? 0,
    gainNode: null,
    sourceNode: null,
    isPlaying: false,
  },
])

// Noise generators
function createWhiteNoise(ctx: AudioContext): AudioBufferSourceNode {
  const bufferSize = 2 * ctx.sampleRate
  const buffer = ctx.createBuffer(1, bufferSize, ctx.sampleRate)
  const data = buffer.getChannelData(0)
  for (let i = 0; i < bufferSize; i++) {
    data[i] = Math.random() * 2 - 1
  }
  const source = ctx.createBufferSource()
  source.buffer = buffer
  source.loop = true
  return source
}

function createRainSound(ctx: AudioContext, gainNode: GainNode): AudioNode {
  const noise = createWhiteNoise(ctx)
  const bandpass = ctx.createBiquadFilter()
  bandpass.type = 'bandpass'
  bandpass.frequency.value = 800
  bandpass.Q.value = 0.5

  const highpass = ctx.createBiquadFilter()
  highpass.type = 'highpass'
  highpass.frequency.value = 400

  noise.connect(bandpass)
  bandpass.connect(highpass)
  highpass.connect(gainNode)
  noise.start()
  return noise
}

function createCafeSound(ctx: AudioContext, gainNode: GainNode): AudioNode {
  const noise = createWhiteNoise(ctx)
  const lowpass = ctx.createBiquadFilter()
  lowpass.type = 'lowpass'
  lowpass.frequency.value = 500

  const lfoGain = ctx.createGain()
  lfoGain.gain.value = 0.7

  // Subtle volume modulation for "murmur" effect
  const lfo = ctx.createOscillator()
  lfo.frequency.value = 0.3
  lfo.type = 'sine'
  const lfoDepth = ctx.createGain()
  lfoDepth.gain.value = 0.15
  lfo.connect(lfoDepth)
  lfoDepth.connect(lfoGain.gain)
  lfo.start()

  noise.connect(lowpass)
  lowpass.connect(lfoGain)
  lfoGain.connect(gainNode)
  noise.start()
  return noise
}

function createFireplaceSound(
  ctx: AudioContext,
  gainNode: GainNode,
): AudioNode {
  const noise = createWhiteNoise(ctx)
  const lowpass = ctx.createBiquadFilter()
  lowpass.type = 'lowpass'
  lowpass.frequency.value = 300

  const highpass = ctx.createBiquadFilter()
  highpass.type = 'highpass'
  highpass.frequency.value = 80

  // Add subtle crackling via gain modulation
  const crackleGain = ctx.createGain()
  crackleGain.gain.value = 0.8
  const crackle = ctx.createOscillator()
  crackle.type = 'sawtooth'
  crackle.frequency.value = 2.5
  const crackleDepth = ctx.createGain()
  crackleDepth.gain.value = 0.3
  crackle.connect(crackleDepth)
  crackleDepth.connect(crackleGain.gain)
  crackle.start()

  noise.connect(lowpass)
  lowpass.connect(highpass)
  highpass.connect(crackleGain)
  crackleGain.connect(gainNode)
  noise.start()
  return noise
}

let birdIntervalId: ReturnType<typeof setInterval> | null = null

function createBirdSound(ctx: AudioContext, gainNode: GainNode): AudioNode {
  // Use a persistent node we can disconnect later
  const masterGain = ctx.createGain()
  masterGain.gain.value = 1
  masterGain.connect(gainNode)

  function chirp() {
    const osc = ctx.createOscillator()
    const chirpGain = ctx.createGain()
    osc.connect(chirpGain)
    chirpGain.connect(masterGain)

    const baseFreq = 2000 + Math.random() * 2500
    osc.frequency.setValueAtTime(baseFreq, ctx.currentTime)
    osc.frequency.exponentialRampToValueAtTime(
      baseFreq * (0.6 + Math.random() * 0.8),
      ctx.currentTime + 0.1,
    )
    osc.frequency.exponentialRampToValueAtTime(
      baseFreq * (0.8 + Math.random() * 0.4),
      ctx.currentTime + 0.2,
    )
    osc.type = 'sine'

    chirpGain.gain.setValueAtTime(0, ctx.currentTime)
    chirpGain.gain.linearRampToValueAtTime(0.08, ctx.currentTime + 0.02)
    chirpGain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.25)

    osc.start(ctx.currentTime)
    osc.stop(ctx.currentTime + 0.25)
  }

  // Random chirps
  birdIntervalId = setInterval(
    () => {
      if (Math.random() > 0.4) chirp()
      // Sometimes double chirp
      if (Math.random() > 0.7) {
        setTimeout(chirp, 80 + Math.random() * 120)
      }
    },
    600 + Math.random() * 1200,
  )

  return masterGain
}

function createLibrarySound(ctx: AudioContext, gainNode: GainNode): AudioNode {
  const noise = createWhiteNoise(ctx)
  const lowpass = ctx.createBiquadFilter()
  lowpass.type = 'lowpass'
  lowpass.frequency.value = 400

  // Very subtle page turning noise periodically
  const pageGain = ctx.createGain()
  pageGain.gain.value = 0.05
  noise.connect(lowpass)
  lowpass.connect(pageGain)
  pageGain.connect(gainNode)
  noise.start()
  return noise
}

function createTrainSound(ctx: AudioContext, gainNode: GainNode): AudioNode {
  const noise = createWhiteNoise(ctx)
  const lowpass = ctx.createBiquadFilter()
  lowpass.type = 'lowpass'
  lowpass.frequency.value = 150

  // Periodic rhythmic "clack-clack" via gain modulation
  const rhythmicGain = ctx.createGain()
  rhythmicGain.gain.value = 0.5
  const oscillator = ctx.createOscillator()
  oscillator.type = 'square'
  oscillator.frequency.value = 1.2 // Train rhythm
  const oscGain = ctx.createGain()
  oscGain.gain.value = 0.2
  oscillator.connect(oscGain)
  oscGain.connect(rhythmicGain.gain)
  oscillator.start()

  noise.connect(lowpass)
  lowpass.connect(rhythmicGain)
  rhythmicGain.connect(gainNode)
  noise.start()
  return noise
}

function createOceanSound(ctx: AudioContext, gainNode: GainNode): AudioNode {
  const noise = createWhiteNoise(ctx)
  const lowpass = ctx.createBiquadFilter()
  lowpass.type = 'lowpass'
  lowpass.frequency.value = 1000

  // Wave swelling via LFO
  const swellGain = ctx.createGain()
  const lfo = ctx.createOscillator()
  lfo.type = 'sine'
  lfo.frequency.value = 0.1 // 10 second wave cycle
  const lfoDepth = ctx.createGain()
  lfoDepth.gain.value = 0.4
  lfo.connect(lfoDepth)
  lfoDepth.connect(swellGain.gain)
  swellGain.gain.value = 0.5
  lfo.start()

  noise.connect(lowpass)
  lowpass.connect(swellGain)
  swellGain.connect(gainNode)
  noise.start()
  return noise
}

let keyboardIntervalId: ReturnType<typeof setInterval> | null = null

function createKeyboardSound(ctx: AudioContext, gainNode: GainNode): AudioNode {
  const masterGain = ctx.createGain()
  masterGain.connect(gainNode)

  function typeKey() {
    const osc = ctx.createOscillator()
    const clickGain = ctx.createGain()
    osc.connect(clickGain)
    clickGain.connect(masterGain)

    osc.type = 'triangle'
    osc.frequency.setValueAtTime(150 + Math.random() * 50, ctx.currentTime)
    osc.frequency.exponentialRampToValueAtTime(10, ctx.currentTime + 0.05)

    clickGain.gain.setValueAtTime(0.1, ctx.currentTime)
    clickGain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.05)

    osc.start(ctx.currentTime)
    osc.stop(ctx.currentTime + 0.05)
  }

  keyboardIntervalId = setInterval(
    () => {
      if (Math.random() > 0.3) {
        typeKey()
        // Occasional burst of typing
        if (Math.random() > 0.8) {
          setTimeout(typeKey, 100)
          setTimeout(typeKey, 200)
        }
      }
    },
    200 + Math.random() * 800,
  )

  return masterGain
}

type SoundCreator = (ctx: AudioContext, gain: GainNode) => AudioNode

const soundCreators: Record<string, SoundCreator> = {
  rain: createRainSound,
  cafe: createCafeSound,
  fireplace: createFireplaceSound,
  birds: createBirdSound,
  library: createLibrarySound,
  train: createTrainSound,
  ocean: createOceanSound,
  keyboard: createKeyboardSound,
}

function startSound(sound: AmbientSound) {
  const ctx = getAudioContext()
  if (ctx.state === 'suspended') ctx.resume()

  const gainNode = ctx.createGain()
  gainNode.gain.value = sound.volume / 100
  gainNode.connect(ctx.destination)

  const creator = soundCreators[sound.id]
  if (!creator) return

  const sourceNode = creator(ctx, gainNode)
  sound.gainNode = gainNode
  sound.sourceNode = sourceNode
  sound.isPlaying = true
}

function stopSound(sound: AmbientSound) {
  if (sound.sourceNode) {
    try {
      // Disconnect the source
      sound.sourceNode.disconnect()
      // For BufferSourceNode, stop it
      if ('stop' in sound.sourceNode) {
        ;(sound.sourceNode as AudioBufferSourceNode).stop()
      }
    } catch {
      // Already stopped
    }
  }
  if (sound.gainNode) {
    sound.gainNode.disconnect()
  }
  if (sound.id === 'birds' && birdIntervalId) {
    clearInterval(birdIntervalId)
    birdIntervalId = null
  }
  if (sound.id === 'keyboard' && keyboardIntervalId) {
    clearInterval(keyboardIntervalId)
    keyboardIntervalId = null
  }
  sound.sourceNode = null
  sound.gainNode = null
  sound.isPlaying = false
}

function onVolumeChange(sound: AmbientSound) {
  // Persist
  savedVolumes.value[sound.id] = sound.volume

  if (sound.volume > 0) {
    if (!sound.isPlaying) {
      startSound(sound)
    } else if (sound.gainNode) {
      sound.gainNode.gain.value = sound.volume / 100
    }
  } else {
    if (sound.isPlaying) {
      stopSound(sound)
    }
  }
}

// Initialize sounds with saved volumes on mount
function initSavedSounds() {
  for (const sound of sounds.value) {
    if (sound.volume > 0) {
      startSound(sound)
    }
  }
}

// Delay init until user interaction
let soundsInitialized = false
function ensureSoundsInit() {
  if (!soundsInitialized) {
    soundsInitialized = true
    initSavedSounds()
  }
}

// ─── Keyboard Shortcuts ───────────────────────────────────
useEventListener(document, 'keydown', (e: KeyboardEvent) => {
  // Don't trigger shortcuts when typing in inputs
  if (
    e.target instanceof HTMLInputElement ||
    e.target instanceof HTMLTextAreaElement
  )
    return

  if (e.code === 'Space') {
    e.preventDefault()
    ensureSoundsInit()
    toggleTimer()
  }
  if (e.key === 'r' || e.key === 'R') {
    resetTimer()
  }
})

// ─── YouTube Background ───────────────────────────────────
const YOUTUBE_VIDEO_ID = 'jfKfPfyJRdk' // lofi hip hop radio

// ─── Cleanup ──────────────────────────────────────────────
onUnmounted(() => {
  pauseInterval()
  for (const sound of sounds.value) {
    stopSound(sound)
  }
  if (birdIntervalId) {
    clearInterval(birdIntervalId)
  }
  if (keyboardIntervalId) {
    clearInterval(keyboardIntervalId)
  }
  if (audioCtx) {
    audioCtx.close()
    audioCtx = null
  }
})
</script>

<template>
  <div class="lofi-page">
    <!-- YouTube Background -->
    <div class="yt-bg">
      <iframe
        :src="`https://www.youtube.com/embed/${YOUTUBE_VIDEO_ID}?autoplay=1&mute=1&loop=1&controls=0&showinfo=0&playlist=${YOUTUBE_VIDEO_ID}&modestbranding=1&rel=0`"
        frameborder="0"
        allow="autoplay; encrypted-media"
        allowfullscreen
        title="Lofi background"
      />
    </div>

    <!-- Gradient overlay -->
    <div class="yt-overlay" />

    <!-- Main content -->
    <div class="lofi-content">
      <!-- Header -->
      <div class="lofi-header animate-fade-up">
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
        <div class="session-badge">
          <span class="tomato-icon">🍅</span>
          <span class="session-count">× {{ pomodoroCount }}</span>
        </div>
      </div>

      <!-- Timer Card (Glassmorphism) -->
      <div class="glass-card animate-fade-up animate-delay-1">
        <!-- Mode Tabs -->
        <div class="mode-tabs">
          <button
            v-for="(label, key) in MODE_LABELS"
            :key="key"
            class="mode-tab"
            :class="{ active: mode === key }"
            @click="handleModeClick(key)"
          >
            <Icon
              :icon="getModeIcon(key)"
              class="size-4"
            />
            {{ label }}
          </button>
        </div>

        <!-- Circular Timer -->
        <div class="timer-container">
          <svg
            class="progress-ring"
            viewBox="0 0 260 260"
          >
            <!-- Background circle -->
            <circle
              cx="130"
              cy="130"
              :r="RADIUS"
              fill="none"
              stroke="rgba(255,255,255,0.08)"
              stroke-width="6"
            />
            <!-- Progress circle -->
            <circle
              cx="130"
              cy="130"
              :r="RADIUS"
              fill="none"
              class="progress-arc"
              :class="{
                'stroke-coral': mode === 'pomodoro',
                'stroke-sky': mode === 'shortBreak',
                'stroke-amber': mode === 'longBreak',
              }"
              stroke-width="6"
              stroke-linecap="round"
              :stroke-dasharray="CIRCUMFERENCE"
              :stroke-dashoffset="strokeDashoffset"
              transform="rotate(-90 130 130)"
            />
          </svg>

          <div class="timer-inner">
            <Transition
              name="fade"
              mode="out-in"
            >
              <div
                v-if="showModeTransition"
                :key="mode"
                class="mode-label"
                :class="{
                  'text-accent-coral': mode === 'pomodoro',
                  'text-accent-sky': mode === 'shortBreak',
                  'text-accent-amber': mode === 'longBreak',
                }"
              >
                <Icon
                  :icon="MODE_ICONS[mode]"
                  class="size-5"
                />
                {{ MODE_LABELS[mode] }}
              </div>
            </Transition>
            <div
              class="timer-digits"
              :class="{ 'is-break': mode !== 'pomodoro' }"
            >
              {{ displayTime }}
            </div>
          </div>
        </div>

        <!-- Controls -->
        <div class="timer-controls">
          <button
            class="ctrl-btn ctrl-reset"
            title="Đặt lại (R)"
            @click="resetTimer()"
          >
            <Icon
              icon="lucide:rotate-ccw"
              class="size-5"
            />
          </button>
          <button
            class="ctrl-btn ctrl-play"
            :class="{
              'bg-coral': mode === 'pomodoro',
              'bg-sky': mode === 'shortBreak',
              'bg-amber': mode === 'longBreak',
            }"
            :title="isRunning ? 'Tạm dừng (Space)' : 'Bắt đầu (Space)'"
            @click="
              () => {
                ensureSoundsInit()
                toggleTimer()
              }
            "
          >
            <Icon
              :icon="isRunning ? 'lucide:pause' : 'lucide:play'"
              class="size-7"
            />
          </button>
          <button
            class="ctrl-btn ctrl-skip"
            title="Bỏ qua"
            @click="onTimerComplete()"
          >
            <Icon
              icon="lucide:skip-forward"
              class="size-5"
            />
          </button>
        </div>

        <!-- Keyboard hints -->
        <div class="kbd-hints">
          <span><kbd>Space</kbd> Bắt đầu / Dừng</span>
          <span><kbd>R</kbd> Đặt lại</span>
        </div>
      </div>

      <!-- Sound Mixer (Glassmorphism) -->
      <div class="glass-card sound-mixer animate-fade-up animate-delay-3">
        <h3 class="mixer-title">
          <Icon
            icon="lucide:headphones"
            class="size-5"
          />
          Âm thanh môi trường
        </h3>
        <div class="sound-grid">
          <div
            v-for="sound in sounds"
            :key="sound.id"
            class="sound-item"
            :class="{ active: sound.volume > 0 }"
          >
            <button
              class="sound-icon-btn"
              :class="{ active: sound.volume > 0 }"
              @click="
                () => {
                  ensureSoundsInit()
                  if (sound.volume > 0) {
                    sound.volume = 0
                    onVolumeChange(sound)
                  } else {
                    sound.volume = 50
                    onVolumeChange(sound)
                  }
                }
              "
            >
              <Icon
                :icon="sound.icon"
                class="size-6"
              />
            </button>
            <span class="sound-name">{{ sound.name }}</span>
            <input
              type="range"
              min="0"
              max="100"
              :value="sound.volume"
              class="sound-slider"
              @input="handleSliderInput(sound, $event)"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* ─── Page Layout ──────────────────────────────────────────── */
.lofi-page {
  position: relative;
  min-height: 100vh;
  overflow: hidden;
  background: linear-gradient(135deg, #0f1923 0%, #1a1a2e 50%, #16213e 100%);
}

/* ─── YouTube Background ───────────────────────────────────── */
.yt-bg {
  position: fixed;
  inset: 0;
  z-index: 0;
  pointer-events: none;
  overflow: hidden;
}

.yt-bg iframe {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 180vw;
  height: 180vh;
  transform: translate(-50%, -50%);
  border: none;
  opacity: 0.35;
}

.yt-overlay {
  position: fixed;
  inset: 0;
  z-index: 1;
  background: radial-gradient(
      ellipse at center,
      transparent 0%,
      rgba(15, 25, 35, 0.5) 60%,
      rgba(15, 25, 35, 0.85) 100%
    ),
    linear-gradient(to top, rgba(15, 25, 35, 0.7) 0%, transparent 40%);
  pointer-events: none;
}

/* ─── Content ──────────────────────────────────────────────── */
.lofi-content {
  position: relative;
  z-index: 10;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-height: 100vh;
  padding: 1.5rem 1rem 3rem;
  gap: 1.5rem;
}

/* ─── Header ───────────────────────────────────────────────── */
.lofi-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  max-width: 480px;
}

.back-link {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  color: rgba(240, 237, 230, 0.6);
  font-family: var(--font-display);
  font-size: 0.8rem;
  letter-spacing: 0.05em;
  text-decoration: none;
  transition: color 0.2s;
}
.back-link:hover {
  color: #f0ede6;
}

.session-badge {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(8px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  padding: 0.35rem 0.75rem;
  border-radius: 100px;
  font-family: var(--font-display);
  font-size: 0.85rem;
  font-weight: 600;
}
.tomato-icon {
  font-size: 1rem;
}
.session-count {
  color: #f0ede6;
}

/* ─── Glassmorphism Card ───────────────────────────────────── */
.glass-card {
  width: 100%;
  max-width: 480px;
  background: rgba(255, 255, 255, 0.07);
  backdrop-filter: blur(24px);
  -webkit-backdrop-filter: blur(24px);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 24px;
  padding: 2rem 1.5rem;
  box-shadow:
    0 8px 32px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

/* ─── Mode Tabs ────────────────────────────────────────────── */
.mode-tabs {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
  margin-bottom: 1.5rem;
}

.mode-tab {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  padding: 0.5rem 1rem;
  border-radius: 100px;
  font-family: var(--font-display);
  font-size: 0.75rem;
  font-weight: 600;
  letter-spacing: 0.03em;
  color: rgba(240, 237, 230, 0.5);
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid transparent;
  cursor: pointer;
  transition: all 0.25s ease;
}
.mode-tab:hover {
  color: rgba(240, 237, 230, 0.8);
  background: rgba(255, 255, 255, 0.1);
}
.mode-tab.active {
  color: #f0ede6;
  background: rgba(255, 255, 255, 0.12);
  border-color: rgba(255, 255, 255, 0.2);
}

/* ─── Timer ────────────────────────────────────────────────── */
.timer-container {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1.5rem;
  width: 260px;
  height: 260px;
}

.progress-ring {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}

.progress-arc {
  transition: stroke-dashoffset 0.5s ease;
}
.stroke-coral {
  stroke: #ff6b4a;
  filter: drop-shadow(0 0 8px rgba(255, 107, 74, 0.4));
}
.stroke-sky {
  stroke: #38bdf8;
  filter: drop-shadow(0 0 8px rgba(56, 189, 248, 0.4));
}
.stroke-amber {
  stroke: #ffb830;
  filter: drop-shadow(0 0 8px rgba(255, 184, 48, 0.4));
}

.timer-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.25rem;
}

.mode-label {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  font-family: var(--font-display);
  font-size: 0.8rem;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
}

.timer-digits {
  font-family: var(--font-display);
  font-size: 4rem;
  font-weight: 800;
  letter-spacing: 0.04em;
  color: #f0ede6;
  line-height: 1;
  text-shadow:
    0 0 30px rgba(255, 107, 74, 0.3),
    0 0 60px rgba(255, 107, 74, 0.1);
  transition: text-shadow 0.4s ease;
}
.timer-digits.is-break {
  text-shadow:
    0 0 30px rgba(56, 189, 248, 0.3),
    0 0 60px rgba(56, 189, 248, 0.1);
}

/* ─── Controls ─────────────────────────────────────────────── */
.timer-controls {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1.25rem;
}

.ctrl-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
  color: #f0ede6;
}

.ctrl-reset,
.ctrl-skip {
  width: 44px;
  height: 44px;
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.12);
}
.ctrl-reset:hover,
.ctrl-skip:hover {
  background: rgba(255, 255, 255, 0.15);
  transform: scale(1.08);
}

.ctrl-play {
  width: 64px;
  height: 64px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}
.ctrl-play:hover {
  transform: scale(1.08);
  box-shadow: 0 6px 28px rgba(0, 0, 0, 0.4);
}
.ctrl-play:active {
  transform: scale(0.96);
}

.bg-coral {
  background: #ff6b4a;
  box-shadow: 0 4px 20px rgba(255, 107, 74, 0.35);
}
.bg-coral:hover {
  box-shadow: 0 6px 28px rgba(255, 107, 74, 0.5);
}
.bg-sky {
  background: #38bdf8;
  box-shadow: 0 4px 20px rgba(56, 189, 248, 0.35);
}
.bg-sky:hover {
  box-shadow: 0 6px 28px rgba(56, 189, 248, 0.5);
}
.bg-amber {
  background: #ffb830;
  box-shadow: 0 4px 20px rgba(255, 184, 48, 0.35);
}
.bg-amber:hover {
  box-shadow: 0 6px 28px rgba(255, 184, 48, 0.5);
}

/* ─── Keyboard Hints ───────────────────────────────────────── */
.kbd-hints {
  display: flex;
  justify-content: center;
  gap: 1.25rem;
  margin-top: 1.25rem;
  font-size: 0.7rem;
  color: rgba(240, 237, 230, 0.3);
  font-family: var(--font-body);
}
.kbd-hints kbd {
  display: inline-block;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.1);
  font-family: var(--font-display);
  font-size: 0.65rem;
  margin-right: 0.25rem;
}

/* ─── Sound Mixer ──────────────────────────────────────────── */
.sound-mixer {
  padding: 1.5rem;
}

.mixer-title {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-family: var(--font-display);
  font-size: 0.9rem;
  font-weight: 600;
  color: rgba(240, 237, 230, 0.7);
  margin-bottom: 1.25rem;
  letter-spacing: 0.03em;
}

.sound-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
  max-height: 420px;
  overflow-y: auto;
  padding-right: 8px;
}

.sound-grid::-webkit-scrollbar {
  width: 4px;
}
.sound-grid::-webkit-scrollbar-track {
  background: transparent;
}
.sound-grid::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
}

.sound-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  padding: 1rem 0.5rem;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.06);
  transition: all 0.25s ease;
}
.sound-item.active {
  background: rgba(255, 255, 255, 0.08);
  border-color: rgba(255, 255, 255, 0.15);
}

.sound-icon-btn {
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: rgba(240, 237, 230, 0.5);
  cursor: pointer;
  transition: all 0.25s ease;
}
.sound-icon-btn:hover {
  background: rgba(255, 255, 255, 0.12);
  color: #f0ede6;
  transform: scale(1.06);
}
.sound-icon-btn.active {
  background: rgba(255, 107, 74, 0.2);
  border-color: rgba(255, 107, 74, 0.4);
  color: #ff6b4a;
}

.sound-name {
  font-family: var(--font-body);
  font-size: 0.75rem;
  color: rgba(240, 237, 230, 0.5);
  font-weight: 500;
}

/* ─── Slider ───────────────────────────────────────────────── */
.sound-slider {
  width: 100%;
  height: 4px;
  appearance: none;
  -webkit-appearance: none;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 100px;
  outline: none;
  cursor: pointer;
}

.sound-slider::-webkit-slider-thumb {
  appearance: none;
  -webkit-appearance: none;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #ff6b4a;
  box-shadow: 0 0 8px rgba(255, 107, 74, 0.4);
  cursor: pointer;
  transition: transform 0.15s ease;
}
.sound-slider::-webkit-slider-thumb:hover {
  transform: scale(1.2);
}

.sound-slider::-moz-range-thumb {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #ff6b4a;
  border: none;
  box-shadow: 0 0 8px rgba(255, 107, 74, 0.4);
  cursor: pointer;
}

/* ─── Transitions ──────────────────────────────────────────── */
.fade-enter-active,
.fade-leave-active {
  transition: all 0.2s ease;
}
.fade-enter-from {
  opacity: 0;
  transform: translateY(-6px);
}
.fade-leave-to {
  opacity: 0;
  transform: translateY(6px);
}

/* ─── Responsive ───────────────────────────────────────────── */
@media (max-width: 480px) {
  .glass-card {
    padding: 1.5rem 1rem;
    border-radius: 20px;
  }

  .timer-container {
    width: 220px;
    height: 220px;
  }

  .timer-digits {
    font-size: 3.2rem;
  }

  .mode-tab {
    padding: 0.4rem 0.7rem;
    font-size: 0.7rem;
  }

  .sound-grid {
    grid-template-columns: repeat(2, 1fr);
    max-height: 400px;
    overflow-y: auto;
    padding-right: 4px;
    gap: 0.75rem;
  }

  .sound-grid::-webkit-scrollbar {
    width: 4px;
  }
  .sound-grid::-webkit-scrollbar-track {
    background: transparent;
  }
  .sound-grid::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 10px;
  }

  .kbd-hints {
    display: none;
  }
}

@media (min-width: 640px) {
  .lofi-content {
    justify-content: center;
    padding-top: 2rem;
    padding-bottom: 2rem;
  }
}
</style>
