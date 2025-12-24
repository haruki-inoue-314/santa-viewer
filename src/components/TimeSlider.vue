<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

interface Props {
  minTime: number
  maxTime: number
  modelValue: number
}

interface Emits {
  (e: 'update:modelValue', value: number): void
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

const currentTime = ref(props.modelValue)
const isPlaying = ref(false)
let animationFrameId: number | null = null
const playbackSpeed = ref(1) // 1秒 = 実際の1分

const formatTime = (timestamp: number): string => {
  // JST (UTC+9) で表示
  const date = new Date(timestamp)
  const jstOffset = 9 * 60 * 60 * 1000 // 9時間をミリ秒に変換
  const jstDate = new Date(date.getTime() + jstOffset)

  const month = jstDate.getUTCMonth() + 1
  const day = jstDate.getUTCDate()
  const hours = String(jstDate.getUTCHours()).padStart(2, '0')
  const minutes = String(jstDate.getUTCMinutes()).padStart(2, '0')
  const seconds = String(jstDate.getUTCSeconds()).padStart(2, '0')
  return `${month}月${day}日 ${hours}:${minutes}:${seconds}`
}

const formattedTime = computed(() => formatTime(currentTime.value))
const progress = computed(() => {
  const range = props.maxTime - props.minTime
  return ((currentTime.value - props.minTime) / range) * 100
})

const updateTime = (value: number) => {
  currentTime.value = value
  emit('update:modelValue', value)
}

const handleSliderChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  updateTime(Number(target.value))
}

const play = () => {
  if (currentTime.value >= props.maxTime) {
    currentTime.value = props.minTime
  }
  isPlaying.value = true
  animate()
}

const pause = () => {
  isPlaying.value = false
  if (animationFrameId !== null) {
    cancelAnimationFrame(animationFrameId)
    animationFrameId = null
  }
  lastTimestamp = 0
}

const reset = () => {
  pause()
  updateTime(props.minTime)
}

let lastTimestamp = 0
const animate = () => {
  if (!isPlaying.value) return

  const now = performance.now()
  if (lastTimestamp === 0) {
    lastTimestamp = now
  }

  const delta = now - lastTimestamp
  lastTimestamp = now

  // playbackSpeed: 1 = 1秒で実際の60秒進む
  const increment = (delta / 1000) * 60 * 1000 * playbackSpeed.value

  const newTime = currentTime.value + increment

  if (newTime >= props.maxTime) {
    updateTime(props.maxTime)
    pause()
    lastTimestamp = 0
  } else {
    updateTime(newTime)
    animationFrameId = requestAnimationFrame(animate)
  }
}

const togglePlay = () => {
  if (isPlaying.value) {
    pause()
  } else {
    play()
  }
}

onMounted(() => {
  updateTime(props.modelValue)
})
</script>

<template>
  <div class="time-slider">
    <div class="time-display">
      <h2>{{ formattedTime }} (JST)</h2>
    </div>

    <div class="slider-container">
      <input
        type="range"
        :min="minTime"
        :max="maxTime"
        :value="currentTime"
        :step="1000"
        class="slider"
        @input="handleSliderChange"
      />
      <div class="progress-bar" :style="{ width: `${progress}%` }"></div>
    </div>

    <div class="controls">
      <button @click="reset" class="control-button" title="リセット">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor">
          <path d="M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8" />
          <path d="M21 3v5h-5" />
          <path d="M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16" />
        </svg>
      </button>

      <button
        @click="togglePlay"
        class="control-button play-button"
        :title="isPlaying ? '一時停止' : '再生'"
      >
        <svg v-if="!isPlaying" width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
          <path d="M8 5v14l11-7z" />
        </svg>
        <svg v-else width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
          <path d="M6 4h4v16H6V4zm8 0h4v16h-4V4z" />
        </svg>
      </button>

      <div class="speed-control">
        <label for="speed">速度: {{ playbackSpeed }}x</label>
        <input
          id="speed"
          v-model.number="playbackSpeed"
          type="range"
          min="0.5"
          max="10"
          step="0.5"
          class="speed-slider"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
.time-slider {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 255, 255, 0.95);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  min-width: 600px;
  z-index: 1000;
}

.time-display {
  text-align: center;
  margin-bottom: 15px;
}

.time-display h2 {
  margin: 0;
  font-size: 24px;
  font-weight: 600;
  color: #c41e3a;
}

.slider-container {
  position: relative;
  height: 40px;
  margin-bottom: 15px;
}

.slider {
  position: absolute;
  width: 100%;
  height: 8px;
  top: 50%;
  transform: translateY(-50%);
  -webkit-appearance: none;
  appearance: none;
  background: transparent;
  outline: none;
  z-index: 2;
  cursor: pointer;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  background: #c41e3a;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  background: #c41e3a;
  border-radius: 50%;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.progress-bar {
  position: absolute;
  height: 8px;
  top: 50%;
  transform: translateY(-50%);
  background: linear-gradient(90deg, #c41e3a, #ff6b6b);
  border-radius: 4px;
  transition: width 0.1s ease;
  pointer-events: none;
}

.slider-container::before {
  content: '';
  position: absolute;
  width: 100%;
  height: 8px;
  top: 50%;
  transform: translateY(-50%);
  background: #e0e0e0;
  border-radius: 4px;
}

.controls {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 15px;
}

.control-button {
  background: #c41e3a;
  color: white;
  border: none;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.control-button:hover {
  background: #a01728;
  transform: scale(1.05);
}

.control-button:active {
  transform: scale(0.95);
}

.play-button {
  width: 50px;
  height: 50px;
}

.speed-control {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}

.speed-control label {
  font-size: 12px;
  color: #666;
  white-space: nowrap;
}

.speed-slider {
  width: 100px;
  height: 4px;
  -webkit-appearance: none;
  appearance: none;
  background: #e0e0e0;
  border-radius: 2px;
  outline: none;
}

.speed-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  background: #666;
  border-radius: 50%;
  cursor: pointer;
}

.speed-slider::-moz-range-thumb {
  width: 12px;
  height: 12px;
  background: #666;
  border-radius: 50%;
  cursor: pointer;
  border: none;
}
</style>
