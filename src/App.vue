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

// ── Lifecycle ─────────────────────────────────────────────────────────────────
onMounted(() => {
  loadFromCookies()
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

    <!-- ── Progress View ──────────────────────────────────────────────────── -->
    <main v-else class="card">
      <div :class="['status-badge', isComplete ? 'badge-complete' : 'badge-active']">
        {{ isComplete ? '🎉 Fast Complete!' : '⚡ Fasting Active' }}
      </div>

      <!-- Progress bar -->
      <div class="progress-wrap">
        <div class="progress-track">
          <div class="progress-fill" :style="{ width: progress + '%' }"></div>
        </div>
        <span class="progress-pct">{{ Math.floor(progress) }}%</span>
      </div>

      <!-- Stats grid -->
      <div class="stats">
        <div class="stat">
          <span class="stat-label">Time Fasting</span>
          <span class="stat-mono">{{ timeElapsed }}</span>
        </div>
        <div class="stat">
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

<style>
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

:root {
  --bg:     #0d1117;
  --card:   #161b22;
  --surf:   #21262d;
  --border: #30363d;
  --text:   #e6edf3;
  --muted:  #8b949e;
  --green:  #3fb950;
  --blue:   #58a6ff;
  --red:    #f85149;
}

html { color-scheme: dark; }

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  background: var(--bg);
  color: var(--text);
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

/* ── App shell ─────────────────────────────────────────────────────────────── */
.app {
  width: 100%;
  max-width: 460px;
}

/* ── Header ────────────────────────────────────────────────────────────────── */
.header {
  text-align: center;
  margin-bottom: 28px;
}

.logo {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  margin-bottom: 6px;
}

.logo-icon { font-size: 2rem; }

h1 {
  font-size: 2.4rem;
  font-weight: 800;
  background: linear-gradient(120deg, var(--green), var(--blue));
  -webkit-background-clip: text;
  background-clip: text;
  -webkit-text-fill-color: transparent;
  letter-spacing: -0.02em;
}

.tagline {
  color: var(--muted);
  font-size: 0.9rem;
}

/* ── Card ──────────────────────────────────────────────────────────────────── */
.card {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 16px;
  padding: 28px;
}

h2 {
  font-size: 1.15rem;
  font-weight: 600;
  margin-bottom: 20px;
}

.section-label {
  font-size: 0.73rem;
  color: var(--muted);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-weight: 500;
  margin-bottom: 10px;
}

/* ── Mode toggle ───────────────────────────────────────────────────────────── */
.mode-toggle {
  display: flex;
  background: var(--surf);
  border-radius: 10px;
  padding: 3px;
  gap: 3px;
  margin-bottom: 22px;
}

.toggle-btn {
  flex: 1;
  padding: 8px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.875rem;
  font-weight: 500;
  font-family: inherit;
  background: transparent;
  color: var(--muted);
  transition: background 0.15s, color 0.15s;
}

.toggle-btn.active {
  background: var(--card);
  color: var(--text);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.35);
}

/* ── Presets ───────────────────────────────────────────────────────────────── */
.presets {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
  margin-bottom: 22px;
}

.preset-btn {
  padding: 10px 6px;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--surf);
  color: var(--muted);
  cursor: pointer;
  font-size: 0.9rem;
  font-weight: 600;
  font-family: inherit;
  transition: border-color 0.15s, color 0.15s, background 0.15s;
}

.preset-btn:hover  { border-color: var(--green); color: var(--green); }
.preset-btn.active {
  border-color: var(--green);
  background: rgba(63, 185, 80, 0.12);
  color: var(--green);
}

/* ── Duration inputs ───────────────────────────────────────────────────────── */
.duration-inputs {
  display: flex;
  gap: 10px;
}

.num-input {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 8px;
  background: var(--surf);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 10px 14px;
  transition: border-color 0.15s;
}

.num-input:focus-within { border-color: var(--blue); }

.num-input input {
  flex: 1;
  min-width: 0;
  background: transparent;
  border: none;
  outline: none;
  color: var(--text);
  font-size: 1.2rem;
  font-weight: 700;
  font-family: inherit;
}

/* Hide number spinners */
.num-input input::-webkit-outer-spin-button,
.num-input input::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }
.num-input input[type=number]               { -moz-appearance: textfield; }

.num-input span {
  color: var(--muted);
  font-size: 0.85rem;
  font-weight: 500;
}

/* ── Datetime input ────────────────────────────────────────────────────────── */
.datetime-input {
  width: 100%;
  background: var(--surf);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 11px 14px;
  color: var(--text);
  font-size: 0.95rem;
  font-family: inherit;
  outline: none;
  transition: border-color 0.15s;
}

.datetime-input:focus { border-color: var(--blue); }

/* ── Error ─────────────────────────────────────────────────────────────────── */
.error-msg {
  color: var(--red);
  font-size: 0.85rem;
  margin-top: 14px;
}

/* ── Start button ──────────────────────────────────────────────────────────── */
.btn-start {
  width: 100%;
  margin-top: 24px;
  padding: 14px;
  background: linear-gradient(135deg, #3fb950 0%, #2ea043 100%);
  border: none;
  border-radius: 10px;
  color: #fff;
  font-size: 1rem;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  letter-spacing: 0.01em;
  transition: opacity 0.15s, transform 0.1s;
}

.btn-start:hover  { opacity: 0.88; }
.btn-start:active { transform: scale(0.99); }

/* ── Status badge ──────────────────────────────────────────────────────────── */
.status-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 5px 14px;
  border-radius: 20px;
  font-size: 0.85rem;
  font-weight: 600;
  margin-bottom: 24px;
}

.badge-active {
  background: rgba(63, 185, 80, 0.12);
  border: 1px solid rgba(63, 185, 80, 0.35);
  color: var(--green);
}

.badge-complete {
  background: rgba(88, 166, 255, 0.12);
  border: 1px solid rgba(88, 166, 255, 0.35);
  color: var(--blue);
}

/* ── Progress bar ──────────────────────────────────────────────────────────── */
.progress-wrap {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 24px;
}

.progress-track {
  flex: 1;
  height: 14px;
  background: var(--surf);
  border-radius: 7px;
  overflow: hidden;
  border: 1px solid var(--border);
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #3fb950 0%, #58a6ff 100%);
  border-radius: 7px;
  transition: width 0.8s cubic-bezier(0.4, 0, 0.2, 1);
}

.progress-pct {
  min-width: 38px;
  text-align: right;
  font-size: 0.9rem;
  font-weight: 700;
  color: var(--muted);
}

/* ── Stats grid ────────────────────────────────────────────────────────────── */
.stats {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  margin-bottom: 24px;
}

.stat {
  background: var(--surf);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 14px 16px;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.stat-label {
  font-size: 0.72rem;
  color: var(--muted);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-weight: 500;
}

.stat-mono {
  font-size: 1.35rem;
  font-weight: 700;
  font-variant-numeric: tabular-nums;
  letter-spacing: -0.01em;
  line-height: 1.2;
}

.stat-date {
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--text);
  line-height: 1.3;
}

/* ── Stop button ───────────────────────────────────────────────────────────── */
.btn-stop {
  width: 100%;
  padding: 12px;
  background: transparent;
  border: 1px solid var(--border);
  border-radius: 10px;
  color: var(--muted);
  font-size: 0.95rem;
  font-weight: 500;
  font-family: inherit;
  cursor: pointer;
  transition: border-color 0.15s, color 0.15s;
}

.btn-stop:hover {
  border-color: var(--red);
  color: var(--red);
}

/* ── Responsive ────────────────────────────────────────────────────────────── */
@media (max-width: 400px) {
  .card { padding: 20px 16px; }
  h1    { font-size: 2rem; }
  .stat-mono { font-size: 1.1rem; }
}
</style>
