<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const COOKIE_START = 'noeats_start'
const COOKIE_END = 'noeats_end'

// ── State ─────────────────────────────────────────────────────────────────────
const fastingStart = ref(null)
const fastingEnd   = ref(null)
const now          = ref(Date.now())

const inputMode       = ref('duration')   // 'duration' | 'datetime'
const durationHours   = ref(16)
const durationMinutes = ref(0)
const endDateTime     = ref('')
const errorMsg        = ref('')

let timer = null

// ── Cookie helpers ────────────────────────────────────────────────────────────
function setCookie(name, value) {
  const exp = new Date(Date.now() + 30 * 86400000).toUTCString()
  document.cookie = `${name}=${encodeURIComponent(value)};expires=${exp};path=/;SameSite=Strict`
}

function getCookie(name) {
  const escaped = name.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
  const m = document.cookie.match(new RegExp('(?:^|; )' + escaped + '=([^;]*)'))
  return m ? decodeURIComponent(m[1]) : null
}

function deleteCookie(name) {
  document.cookie = `${name}=;expires=Thu, 01 Jan 1970 00:00:00 GMT;path=/;SameSite=Strict`
}

// ── Computed ──────────────────────────────────────────────────────────────────
const isActive = computed(() =>
  fastingStart.value !== null &&
  fastingEnd.value   !== null &&
  now.value < fastingEnd.value
)

const isComplete = computed(() =>
  fastingStart.value !== null &&
  fastingEnd.value   !== null &&
  now.value >= fastingEnd.value
)

const progress = computed(() => {
  if (!fastingStart.value || !fastingEnd.value) return 0
  const total   = fastingEnd.value - fastingStart.value
  const elapsed = now.value - fastingStart.value
  return Math.min(100, Math.max(0, (elapsed / total) * 100))
})

const timeElapsed = computed(() => {
  if (!fastingStart.value) return '00:00:00'
  return formatDuration(Math.max(0, now.value - fastingStart.value))
})

const timeRemaining = computed(() => {
  if (!fastingEnd.value) return '00:00:00'
  return formatDuration(Math.max(0, fastingEnd.value - now.value))
})

// ── Formatters ────────────────────────────────────────────────────────────────
function formatDuration(ms) {
  const s   = Math.floor(ms / 1000)
  const h   = Math.floor(s / 3600)
  const m   = Math.floor((s % 3600) / 60)
  const sec = s % 60
  return [h, m, sec].map(n => String(n).padStart(2, '0')).join(':')
}

function formatDateTime(ts) {
  if (!ts) return '—'
  return new Date(ts).toLocaleString(undefined, {
    month: 'short', day: 'numeric',
    hour: '2-digit', minute: '2-digit'
  })
}

// ── Actions ───────────────────────────────────────────────────────────────────
function startFasting() {
  errorMsg.value = ''
  clearLogs()
  const start = Date.now()
  let end

  if (inputMode.value === 'duration') {
    const h  = Math.max(0, Number(durationHours.value)   || 0)
    const m  = Math.max(0, Number(durationMinutes.value) || 0)
    const ms = (h * 3600 + m * 60) * 1000
    if (ms <= 0) { errorMsg.value = 'Please enter a duration greater than 0.'; return }
    end = start + ms
  } else {
    if (!endDateTime.value) { errorMsg.value = 'Please select an end date and time.'; return }
    end = new Date(endDateTime.value).getTime()
    if (isNaN(end) || end <= start) { errorMsg.value = 'End time must be in the future.'; return }
  }

  fastingStart.value = start
  fastingEnd.value   = end
  setCookie(COOKIE_START, start)
  setCookie(COOKIE_END, end)
}

function stopFasting() {
  fastingStart.value = null
  fastingEnd.value   = null
  deleteCookie(COOKIE_START)
  deleteCookie(COOKIE_END)
  clearLogs()
}

function loadFromCookies() {
  const s = getCookie(COOKIE_START)
  const e = getCookie(COOKIE_END)
  if (s && e) {
    const start = parseInt(s, 10)
    const end   = parseInt(e, 10)
    if (!isNaN(start) && !isNaN(end)) {
      fastingStart.value = start
      fastingEnd.value   = end
    }
  }
}

// ── Presets ───────────────────────────────────────────────────────────────────
const presets = [
  { label: '12h', hours: 12, minutes: 0 },
  { label: '16h', hours: 16, minutes: 0 },
  { label: '18h', hours: 18, minutes: 0 },
  { label: '20h', hours: 20, minutes: 0 },
  { label: '24h', hours: 24, minutes: 0 },
  { label: '36h', hours: 36, minutes: 0 },
]

function applyPreset(p) {
  durationHours.value   = p.hours
  durationMinutes.value = p.minutes
}

const minDateTime = computed(() => {
  return new Date(Date.now() + 60000).toISOString().slice(0, 16)
})

// ── Mood & hunger log ─────────────────────────────────────────────────────────
const moodEmojis   = ['😫', '😕', '😐', '🙂', '😄']
const hungerEmojis = ['😌', '😐', '😕', '😩', '🤤']

const selectedMood   = ref(null)
const selectedHunger = ref(null)
const logs           = ref([])   // [{ time, mood: 1-5, hunger: 1-5 }]

function logEntry() {
  if (selectedMood.value === null || selectedHunger.value === null) return
  logs.value.push({ time: Date.now(), mood: selectedMood.value, hunger: selectedHunger.value })
  localStorage.setItem('noeats_logs', JSON.stringify(logs.value))
  selectedMood.value   = null
  selectedHunger.value = null
}

function loadLogs() {
  try {
    const raw = localStorage.getItem('noeats_logs')
    logs.value = raw ? JSON.parse(raw) : []
  } catch { logs.value = [] }
}

function clearLogs() {
  logs.value = []
  localStorage.removeItem('noeats_logs')
}

// ── Chart ─────────────────────────────────────────────────────────────────────
const CW = 400, CH = 160, CPL = 28, CPR = 12, CPT = 14, CPB = 10

const chartData = computed(() => {
  if (!fastingStart.value || logs.value.length === 0) return null
  const plotW = CW - CPL - CPR
  const plotH = CH - CPT - CPB
  const endT  = fastingEnd.value ?? now.value
  const xOf   = t => CPL + Math.max(0, Math.min(1, (t - fastingStart.value) / (endT - fastingStart.value))) * plotW
  const yOf   = v => CPT + plotH - ((v - 1) / 4) * plotH
  const pts   = logs.value.map(l => ({ x: xOf(l.time), my: yOf(l.mood), hy: yOf(l.hunger) }))
  return {
    gridYs:     [1, 2, 3, 4, 5].map(v => ({ v, y: yOf(v) })),
    pts,
    moodLine:   pts.map(p => `${p.x},${p.my}`).join(' '),
    hungerLine: pts.map(p => `${p.x},${p.hy}`).join(' '),
    nowX:       xOf(now.value),
    x1: CPL,  x2: CW - CPR,
    yTop: CPT, yBot: CPT + plotH,
  }
})

// ── Lifecycle ─────────────────────────────────────────────────────────────────
onMounted(() => {
  loadFromCookies()
  loadLogs()
  timer = setInterval(() => { now.value = Date.now() }, 1000)
})

onUnmounted(() => clearInterval(timer))
</script>

<template>
  <div class="app">
    <!-- Header -->
    <header class="header">
      <div class="logo">
        <span class="logo-icon">🍽️</span>
        <h1>NoEats</h1>
      </div>
      <p class="tagline">Intermittent Fasting Tracker</p>
    </header>

    <!-- ── Setup View ─────────────────────────────────────────────────────── -->
    <main v-if="!isActive && !isComplete" class="card">
      <h2>Start a New Fast</h2>

      <div class="mode-toggle">
        <button :class="['toggle-btn', { active: inputMode === 'duration' }]"
                @click="inputMode = 'duration'">Duration</button>
        <button :class="['toggle-btn', { active: inputMode === 'datetime' }]"
                @click="inputMode = 'datetime'">End Date &amp; Time</button>
      </div>

      <!-- Duration mode -->
      <template v-if="inputMode === 'duration'">
        <p class="section-label">Presets</p>
        <div class="presets">
          <button
            v-for="p in presets"
            :key="p.label"
            :class="['preset-btn', { active: durationHours === p.hours && durationMinutes === p.minutes }]"
            @click="applyPreset(p)"
          >{{ p.label }}</button>
        </div>

        <p class="section-label">Custom Duration</p>
        <div class="duration-inputs">
          <div class="num-input">
            <input type="number" v-model.number="durationHours" min="0" max="999" />
            <span>hr</span>
          </div>
          <div class="num-input">
            <input type="number" v-model.number="durationMinutes" min="0" max="59" />
            <span>min</span>
          </div>
        </div>
      </template>

      <!-- End date/time mode -->
      <template v-if="inputMode === 'datetime'">
        <p class="section-label">Fast Ends At</p>
        <input
          type="datetime-local"
          v-model="endDateTime"
          :min="minDateTime"
          class="datetime-input"
        />
      </template>

      <p v-if="errorMsg" class="error-msg">{{ errorMsg }}</p>
      <button class="btn-start" @click="startFasting">🚀 Start Fasting</button>
    </main>

    <!-- -- Progress View ---------------------------------------------------- -->
    <main v-else class="card">
      <div :class="['status-badge', isComplete ? 'badge-complete' : 'badge-active']">
        {{ isComplete ? '🎉 Fast Complete!' : '⚡ Fasting Active' }}
      </div>

      <!-- Progress bar (main feature) -->
      <div class="progress-hero">
        <div class="progress-track-hero">
          <div class="progress-fill-hero" :style="{ width: progress + '%' }"></div>
        </div>
        <div class="progress-pct-hero">{{ Math.floor(progress) }}%</div>
      </div>

      <!-- Time Fasting (second feature) -->
      <div class="elapsed-hero">
        <span class="elapsed-label">Time Fasting</span>
        <span class="elapsed-time">{{ timeElapsed }}</span>
      </div>

      <!-- Log entry -->
      <div v-if="isActive" class="log-section">
        <p class="section-label">How do you feel?</p>
        <div class="log-row">
          <span class="log-row-label">Mood</span>
          <div class="picker">
            <button
              v-for="(emoji, i) in moodEmojis" :key="i"
              :class="['pick-btn', { 'pick-active-mood': selectedMood === i + 1 }]"
              @click="selectedMood = selectedMood === i + 1 ? null : i + 1"
            >{{ emoji }}</button>
          </div>
        </div>
        <div class="log-row">
          <span class="log-row-label">Hunger</span>
          <div class="picker">
            <button
              v-for="(emoji, i) in hungerEmojis" :key="i"
              :class="['pick-btn', { 'pick-active-hunger': selectedHunger === i + 1 }]"
              @click="selectedHunger = selectedHunger === i + 1 ? null : i + 1"
            >{{ emoji }}</button>
          </div>
        </div>
        <button
          class="btn-log"
          :disabled="selectedMood === null || selectedHunger === null"
          @click="logEntry"
        >+ Log</button>
      </div>

      <!-- Chart -->
      <div v-if="logs.length > 0 && chartData" class="chart-wrap">
        <div class="chart-legend">
          <span class="legend-mood">&#9679; Mood</span>
          <span class="legend-hunger">&#9679; Hunger</span>
        </div>
        <svg class="chart-svg" viewBox="0 0 400 160" xmlns="http://www.w3.org/2000/svg">
          <!-- Grid lines -->
          <line v-for="g in chartData.gridYs" :key="g.v"
            :x1="chartData.x1" :x2="chartData.x2"
            :y1="g.y" :y2="g.y"
            stroke="var(--border)" stroke-width="1" />
          <!-- Y labels -->
          <text v-for="g in chartData.gridYs" :key="'l' + g.v"
            :x="CPL - 4" :y="g.y + 4"
            text-anchor="end" font-size="9" fill="var(--muted)">{{ g.v }}</text>
          <!-- Now marker -->
          <line v-if="isActive"
            :x1="chartData.nowX" :x2="chartData.nowX"
            :y1="chartData.yTop" :y2="chartData.yBot"
            stroke="var(--muted)" stroke-width="1" stroke-dasharray="3,3" />
          <!-- Mood line -->
          <polyline v-if="chartData.pts.length > 1"
            :points="chartData.moodLine"
            fill="none" stroke="var(--blue)" stroke-width="2"
            stroke-linejoin="round" stroke-linecap="round" />
          <!-- Hunger line -->
          <polyline v-if="chartData.pts.length > 1"
            :points="chartData.hungerLine"
            fill="none" stroke="var(--orange)" stroke-width="2"
            stroke-linejoin="round" stroke-linecap="round" />
          <!-- Mood dots -->
          <circle v-for="(p, i) in chartData.pts" :key="'m' + i"
            :cx="p.x" :cy="p.my" r="3.5" fill="var(--blue)" />
          <!-- Hunger dots -->
          <circle v-for="(p, i) in chartData.pts" :key="'h' + i"
            :cx="p.x" :cy="p.hy" r="3.5" fill="var(--orange)" />
        </svg>
      </div>

      <!-- Secondary stats -->
      <div class="stats">
        <div class="stat stat-full">
          <span class="stat-label">{{ isComplete ? 'Goal Reached' : 'Time Remaining' }}</span>
          <span class="stat-mono">{{ isComplete ? '✓' : timeRemaining }}</span>
        </div>
        <div class="stat">
          <span class="stat-label">Started</span>
          <span class="stat-date">{{ formatDateTime(fastingStart) }}</span>
        </div>
        <div class="stat">
          <span class="stat-label">{{ isComplete ? 'Ended' : 'Goal' }}</span>
          <span class="stat-date">{{ formatDateTime(fastingEnd) }}</span>
        </div>
      </div>

      <button class="btn-stop" @click="stopFasting">
        {{ isComplete ? '✅ Start New Fast' : '⏹ Stop Fast' }}
      </button>
    </main>
  </div>
</template>