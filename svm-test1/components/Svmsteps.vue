<script setup lang="ts">
import { ref, reactive, computed, useSlots, watch, unref, onBeforeUnmount } from 'vue'
import { useSlideContext } from '@slidev/client'

const props = withDefaults(defineProps<{ clicks?: boolean; batch?: number; maxSteps?: number; hold?: number; speed?: number }>(),
  { clicks: false, batch: 1, maxSteps: 7, hold: 1500, speed: 1500 })
// clicks: advance with Space | batch: violators added per class per step (smaller = more steps) | hold/speed: ms

type Pt = { x: number; y: number; c: 1 | -1 }
type Line = { w1: number; w2: number; b: number }

// ---------- data: two separable clusters (blue = +1, orange = -1) ----------
function rng(seed: number) {
  return () => { seed |= 0; seed = (seed + 0x6d2b79f5) | 0; let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296 }
}
const rand = rng(11)
const gauss = () => Math.sqrt(-2 * Math.log(1 - rand())) * Math.cos(2 * Math.PI * rand())
const pts: Pt[] = []
{ const ux = 1.3 / Math.hypot(1.3, 1), uy = 1 / Math.hypot(1.3, 1)
  for (const c of [1, -1] as const) for (let i = 0; i < 15; i++) {
    let x = 0, y = 0
    do { x = c * 1.3 + 0.75 * gauss(); y = c * 1.0 + 0.75 * gauss() } while (c * (x * ux + y * uy) < 0.35)   // guarantees a gap
    pts.push({ x, y, c })
  } }

// ---------- a tiny linear SVM (dual coordinate descent) ----------
const f = (L: Line, p: Pt) => L.w1 * p.x + L.w2 * p.y + L.b
function train(idx: number[]): Line {
  const BIAS = 5, C = 1000, X = idx.map(i => [pts[i].x, pts[i].y, BIAS]), Y = idx.map(i => pts[i].c)
  const Q = X.map(x => x[0] ** 2 + x[1] ** 2 + x[2] ** 2), a = X.map(() => 0), w = [0, 0, 0]
  for (let it = 0; it < 3000; it++) for (let i = 0; i < X.length; i++) {
    const G = Y[i] * (w[0] * X[i][0] + w[1] * X[i][1] + w[2] * X[i][2]) - 1
    const an = Math.min(Math.max(a[i] - G / Q[i], 0), C), d = an - a[i]
    if (d) { for (let j = 0; j < 3; j++) w[j] += d * Y[i] * X[i][j]; a[i] = an }
  }
  return { w1: w[0], w2: w[1], b: w[2] * BIAS }
}

// ---------- the sequence of lines: random start -> ... -> optimum ----------
// Chunking: fix the worst violators of the current line, retrain on that working set, repeat.
const all = pts.map((_, i) => i)
const yf = (L: Line, i: number) => pts[i].c * f(L, pts[i])
let keys: Line[] = [{ w1: 1, w2: -1.1, b: 0.6 }]                 // deliberately bad starting line
{ const W = new Set<number>()
  for (let it = 0; it < 12; it++) {
    const L = keys[keys.length - 1]
    const viol = all.filter(i => !W.has(i) && yf(L, i) < 1 - 1e-3).sort((a, b) => yf(L, a) - yf(L, b))
    if (!viol.length) break
    for (const c of [1, -1]) viol.filter(i => pts[i].c === c).slice(0, props.batch).forEach(i => W.add(i))
    keys.push(train([...W]))
  }
  const fin = train(all), last = keys[keys.length - 1]
  if (Math.hypot(fin.w1 - last.w1, fin.w2 - last.w2, fin.b - last.b) > 0.02) keys.push(fin); else keys[keys.length - 1] = fin
  if (keys.length - 1 > props.maxSteps) keys = [...keys.slice(0, props.maxSteps), fin] }
const K = keys.length - 1                                          // number of adjustments
const info = keys.map(L => ({
  sv: all.filter(i => yf(L, i) <= 1.05),                            // on / inside the margin, or misclassified
  err: all.filter(i => yf(L, i) <= 0),
  margin: 2 / Math.hypot(L.w1, L.w2),
}))

// ---------- animation state ----------
const W = 640, H = 400, S = 64
const sx = (x: number) => W / 2 + S * x
const sy = (y: number) => H / 2 - S * y
const cur = reactive<Line>({ ...keys[0] })
const i = ref(0), hl = ref(true), moving = ref(false), playing = ref(false)
let epoch = 0, desired = 0, driver: Promise<void> | null = null, raf = 0, dead = false
const lerp = (a: number, b: number, t: number) => a + (b - a) * t
const ease = (u: number) => (u < 0.5 ? 4 * u ** 3 : 1 - (-2 * u + 2) ** 3 / 2)
const sleep = (ms: number) => new Promise<void>(r => setTimeout(r, ms))

function tween(a: Line, b: Line) {
  const ep = epoch, t0 = performance.now()
  return new Promise<void>(res => {
    const tick = () => {
      if (ep !== epoch || dead) return res()
      const u = Math.min((performance.now() - t0) / props.speed, 1), e = ease(u)
      cur.w1 = lerp(a.w1, b.w1, e); cur.w2 = lerp(a.w2, b.w2, e); cur.b = lerp(a.b, b.b, e)
      u < 1 ? (raf = requestAnimationFrame(tick)) : res()
    }
    raf = requestAnimationFrame(tick)
  })
}
async function move(dir: 1 | -1) {
  const ep = epoch, from = i.value, to = from + dir
  hl.value = false; moving.value = true
  await sleep(250)                                                  // let the highlight rings fade out
  await tween(keys[from], keys[to])
  if (ep !== epoch) return
  i.value = to; moving.value = false; hl.value = true
}
function setDesired(n: number) {
  desired = Math.max(0, Math.min(K, n))
  if (!driver) { const ep = epoch
    driver = (async () => { while (ep === epoch && !dead && i.value !== desired) await move(i.value < desired ? 1 : -1) })()
      .finally(() => { if (ep === epoch) driver = null }) }
  return driver
}
const next = () => setDesired(desired + 1), back = () => setDesired(desired - 1)
function reset() { epoch++; playing.value = false; driver = null; desired = 0; i.value = 0; moving.value = false; hl.value = true; Object.assign(cur, keys[0]) }
async function play() {
  if (playing.value) { playing.value = false; return }
  if (i.value >= K) reset()
  playing.value = true
  while (playing.value && i.value < K && !dead) { await sleep(props.hold); if (!playing.value) break; await next() }
  playing.value = false
}
onBeforeUnmount(() => { dead = true; epoch++; cancelAnimationFrame(raf) })

// optional: drive the steps with Slidev clicks (Space / arrow keys)
if (props.clicks) {
  const ctx: any = useSlideContext()
  watch(() => Number(unref(unref(ctx)?.$clicks) ?? 0), c => setDesired(c))
}

// ---------- geometry ----------
const norm = computed(() => Math.hypot(cur.w1, cur.w2))
function line(c: number) {
  const n2 = norm.value ** 2, k = (c - cur.b) / n2, n = norm.value, L = 20
  const px = k * cur.w1, py = k * cur.w2, dx = -cur.w2 / n, dy = cur.w1 / n
  return { x1: sx(px - L * dx), y1: sy(py - L * dy), x2: sx(px + L * dx), y2: sy(py + L * dy) }
}
const lines = computed(() => ({ mid: line(0), lo: line(-1), hi: line(1) }))
const band = computed(() => { const { lo, hi } = lines.value; return `${lo.x1},${lo.y1} ${lo.x2},${lo.y2} ${hi.x2},${hi.y2} ${hi.x1},${hi.y1}` })
const final = computed(() => i.value === K && !moving.value)

// ---------- info box (slot) ----------
const slots = useSlots()
const hasInfo = !!Object.keys(slots).length
const slotName = computed(() => (slots[`step${i.value}`] ? `step${i.value}` : 'default'))
const slotProps = computed(() => {
  const L = keys[i.value], d = info[i.value]
  return { step: i.value, steps: K, moving: moving.value, final: final.value, errors: d.err.length, nSV: d.sv.length,
    margin: +d.margin.toFixed(2), w: [+L.w1.toFixed(2), +L.w2.toFixed(2)], b: +L.b.toFixed(2) }
})
</script>

<template>
  <div class="wrap">
    <svg class="plot" :viewBox="`0 0 ${W} ${H}`">
      <g stroke="var(--ax)" stroke-width="1" opacity=".12">
        <line v-for="g in 9" :key="'v' + g" :x1="sx(g - 5)" y1="0" :x2="sx(g - 5)" :y2="H" />
        <line v-for="g in 7" :key="'h' + g" x1="0" :y1="sy(g - 4)" :x2="W" :y2="sy(g - 4)" />
      </g>
      <line x1="0" :y1="H / 2" :x2="W" :y2="H / 2" stroke="var(--ax)" stroke-width="1.6" opacity=".6" />
      <line :x1="W / 2" y1="0" :x2="W / 2" :y2="H" stroke="var(--ax)" stroke-width="1.6" opacity=".6" />
      <polygon :points="band" fill="var(--c-blue-highlight)" opacity=".16" />
      <line v-bind="lines.lo" stroke="var(--c-orange)" stroke-width="2" stroke-dasharray="6 5" />
      <line v-bind="lines.hi" stroke="var(--c-blue)" stroke-width="2" stroke-dasharray="6 5" />
      <line v-bind="lines.mid" class="decision" :stroke="final ? 'var(--c-green)' : 'var(--ax)'" stroke-width="3.5" stroke-linecap="round" />
      <g class="rings" :style="{ opacity: hl ? 1 : 0 }">
        <circle v-for="n in info[i].sv" :key="'r' + n" class="ring" :cx="sx(pts[n].x)" :cy="sy(pts[n].y)" r="13"
                :stroke="info[i].err.includes(n) ? 'var(--c-red)' : 'var(--c-blue-highlight)'"
                :fill="info[i].err.includes(n) ? 'var(--c-red-highlight)' : 'none'" :fill-opacity="info[i].err.includes(n) ? 0.3 : 0" />
      </g>
      <circle v-for="(p, n) in pts" :key="n" :cx="sx(p.x)" :cy="sy(p.y)" r="6.5" stroke="var(--ax)" stroke-width="1.2"
              :fill="p.c === 1 ? 'var(--c-blue)' : 'var(--c-orange)'" />
    </svg>

    <div class="panel">
      <div class="btn-row">
        <button class="btn" :disabled="i === 0" @click="back">‹ Back</button>
        <button class="btn on" :disabled="i === K" @click="next">Next ›</button>
        <button class="btn" @click="play">{{ playing ? 'Pause' : 'Play' }}</button>
        <button class="btn" @click="reset">Reset</button>
        <span class="count">{{ i }} / {{ K }}</span>
      </div>
      <div v-if="hasInfo" class="info"><slot :name="slotName" v-bind="slotProps" /></div>
    </div>
    <template v-if="clicks"><span v-for="n in K" :key="n" v-click hidden /></template>
  </div>
</template>

<style scoped>
.wrap { --ax: var(--c-axis, var(--c-light, #f8fafc)); display: grid; grid-template-columns: 3fr 2fr; gap: 1.4rem; align-items: center; height: 100%; }
.plot { width: 100%; height: auto; }
.decision { transition: stroke .6s; }
.rings { transition: opacity .35s; }
.ring { stroke-width: 3; transform-box: fill-box; transform-origin: center; animation: pulse 1.2s ease-in-out infinite; }
@keyframes pulse { 0%, 100% { transform: scale(.9); opacity: .95; } 50% { transform: scale(1.22); opacity: .55; } }
.count { align-self: center; font-size: .8rem; opacity: .7; margin-left: .3rem; }
.btn:disabled { opacity: .4; cursor: default; }
/* light box, dark text: the inverse of the rest of the deck */
.info { background: var(--c-light); color: var(--c-dark); border-radius: 1rem; padding: 1rem 1.3rem; font-size: 1rem; line-height: 1.5; min-height: 14rem; display: flex; flex-direction: column; justify-content: center; }
.info :deep(p) { margin: .4em 0; }
.info :deep(.katex-display) { margin: .6em 0; }
.info :deep(.katex) { font-size: 1.1em; }
</style>