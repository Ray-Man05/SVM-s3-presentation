<script setup lang="ts">
import { ref, computed, onBeforeUnmount } from 'vue'

type Pt = { x: number; y: number; c: 1 | -1 }

// Seeded RNG so the dataset is identical on every load
function rng(seed: number) {
  return () => {
    seed |= 0; seed = (seed + 0x6d2b79f5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
const rand = rng(7)
const gauss = () => Math.sqrt(-2 * Math.log(1 - rand())) * Math.cos(2 * Math.PI * rand())
const clamp = (v: number, a: number) => Math.max(-a, Math.min(a, v))

const pts: Pt[] = []
for (const [c, mx, my] of [[1, 1.6, 1.3], [-1, -1.6, -1.3]] as const)
  for (let i = 0; i < 14; i++)
    pts.push({ x: clamp(mx + 0.8 * gauss(), 4.6), y: clamp(my + 0.8 * gauss(), 2.9), c })

// SVG geometry: data -> pixels
const W = 640, H = 400, S = 64
const sx = (x: number) => W / 2 + S * x
const sy = (y: number) => H / 2 - S * y

const w1 = ref(1), w2 = ref(1), b = ref(0), C = ref(1)
const norm = computed(() => Math.hypot(w1.value, w2.value))
const f = (p: Pt) => w1.value * p.x + w2.value * p.y + b.value
const ym = computed(() => pts.map(p => p.c * f(p)))     // functional margins y_i f(x_i)
const isSV = (i: number) => ym.value[i] <= 1.0001       // on / inside margin, or misclassified
const nSV = computed(() => pts.filter((_, i) => isSV(i)).length)
const hinge = computed(() => ym.value.reduce((s, m) => s + Math.max(0, 1 - m), 0))
const objective = computed(() => 0.5 * norm.value ** 2 + C.value * hinge.value)
const margin = computed(() => (norm.value > 1e-9 ? 2 / norm.value : Infinity))

// Line {w·x + b = c}, drawn long enough to be clipped by the SVG
function line(c: number) {
  const n2 = norm.value ** 2
  if (n2 < 1e-9) return null
  const k = (c - b.value) / n2, n = norm.value, L = 20
  const px = k * w1.value, py = k * w2.value, dx = -w2.value / n, dy = w1.value / n
  return { x1: sx(px - L * dx), y1: sy(py - L * dy), x2: sx(px + L * dx), y2: sy(py + L * dy) }
}
const lines = computed(() => ({ mid: line(0), lo: line(-1), hi: line(1) }))
const band = computed(() => {
  const { lo, hi } = lines.value
  return lo && hi ? `${lo.x1},${lo.y1} ${lo.x2},${lo.y2} ${hi.x2},${hi.y2} ${hi.x1},${hi.y1}` : ''
})

// Full-batch Pegasos on  J = ½‖w‖² + C Σ max(0, 1 − y f(x))  (step 1/k)
let raf = 0, k = 0
function step() {
  for (let i = 0; i < 6; i++) {
    k++
    let gx = 0, gy = 0, gb = 0
    for (const p of pts) if (p.c * f(p) < 1) { gx += p.c * p.x; gy += p.c * p.y; gb += p.c }
    const a = 1 / k
    w1.value = (1 - a) * w1.value + C.value * a * gx
    w2.value = (1 - a) * w2.value + C.value * a * gy
    b.value += C.value * a * gb
  }
  if (k < 600) raf = requestAnimationFrame(step)
}
function train() { cancelAnimationFrame(raf); k = 0; raf = requestAnimationFrame(step) }
function reset() { cancelAnimationFrame(raf); w1.value = 1; w2.value = 1; b.value = 0 }
onBeforeUnmount(() => cancelAnimationFrame(raf))
const fmt = (v: number, d = 2) => (Number.isFinite(v) ? v.toFixed(d) : '∞')
</script>

<template>
  <div class="wrap">
    <svg class="figure plot" :viewBox="`0 0 ${W} ${H}`">
      <defs><clipPath id="clip"><rect width="640" height="400" rx="16" /></clipPath></defs>
      <g clip-path="url(#clip)">
        <rect width="640" height="400" fill="white" />
        <line :x1="0" :y1="H / 2" :x2="W" :y2="H / 2" stroke="#cbd5e1" />
        <line :x1="W / 2" :y1="0" :x2="W / 2" :y2="H" stroke="#cbd5e1" />
        <polygon v-if="band" :points="band" fill="var(--c-blue-highlight)" opacity=".45" />
        <template v-if="lines.mid">
          <line v-bind="lines.lo!" stroke="var(--c-orange)" stroke-width="2" stroke-dasharray="6 5" />
          <line v-bind="lines.hi!" stroke="var(--c-blue)" stroke-width="2" stroke-dasharray="6 5" />
          <line v-bind="lines.mid!" stroke="var(--c-dark)" stroke-width="3" />
        </template>
        <g v-for="(p, i) in pts" :key="i">
          <circle v-if="isSV(i)" :cx="sx(p.x)" :cy="sy(p.y)" r="13"
                  :fill="ym[i] < 0 ? 'var(--c-red-highlight)' : 'var(--c-blue-highlight)'"
                  :stroke="ym[i] < 0 ? 'var(--c-red)' : 'var(--c-dark)'" stroke-width="2" />
          <circle :cx="sx(p.x)" :cy="sy(p.y)" r="6.5" stroke="white" stroke-width="1.5"
                  :fill="p.c === 1 ? 'var(--c-blue)' : 'var(--c-orange)'" />
        </g>
      </g>
    </svg>

    <div class="panel">
      <label>w₁ = {{ fmt(w1) }}<input type="range" min="-3" max="3" step="0.05" v-model.number="w1" /></label>
      <label>w₂ = {{ fmt(w2) }}<input type="range" min="-3" max="3" step="0.05" v-model.number="w2" /></label>
      <label>b = {{ fmt(b) }}<input type="range" min="-8" max="8" step="0.05" v-model.number="b" /></label>
      <label>C = {{ fmt(C, 1) }}<input type="range" min="0.1" max="10" step="0.1" v-model.number="C" /></label>
      <div class="btn-row">
        <button class="btn on" @click="train">Optimise</button>
        <button class="btn" @click="reset">Reset</button>
      </div>
      <div class="stats">
        ‖w‖ = {{ fmt(norm) }}<br />
        margin 2/‖w‖ = {{ fmt(margin) }}<br />
        support vectors: {{ nSV }} / {{ pts.length }}<br />
        hinge loss = {{ fmt(hinge) }}<br />
        objective = {{ fmt(objective) }}
      </div>
    </div>
  </div>
</template>

<style scoped>
.wrap { display: grid; grid-template-columns: 2fr 1fr; gap: 1.2rem; align-items: center; height: 100%; }
.plot { width: 100%; height: auto; border: 2px solid var(--c-dark); }
</style>
