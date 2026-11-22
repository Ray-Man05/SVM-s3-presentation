
<script setup lang="ts">
import { ref, computed, watch, unref, onBeforeUnmount } from 'vue'
import { useSlideContext } from '@slidev/client'

/* ════════════════════════════════════════════════════════════════════════════
   MarginShift: one outlier drags a hard-margin line.

   Before training: original line, wide margin.
   After training:  line skewed by the outlier, smaller margin.
   Testing:         overlay unseen test points and highlight errors.
   ════════════════════════════════════════════════════════════════════════════ */

/* ───────────────────────────── 1. SETTINGS ───────────────────────────── */

// 1a. PROPS
const props = withDefaults(defineProps<{
  state?: 'before' | 'after' | 'testing'
  controls?: boolean
  clicks?: boolean
  markOutlier?: boolean
  caption?: string
  speed?: number
}>(), {
  state: 'before',
  controls: true,
  clicks: false,
  markOutlier: true,
  caption: '',
  speed: 900
})

const LABELS = {
  before: 'Before Training',
  after: 'After Training',
  testing: 'Testing'
}

// 1b. DATA. UNCHANGED.
const TRAIN_SEED = 118
const BLUE = { x: 1.4, y: 0 }
const ORANGE = { x: -1.4, y: 0 }
const SPREAD = { x: 0.5, y: 0.85 }
const KEEP_WITHIN = { x: 3.8, y: 2.2 }
const PER_CLASS = 14
const GAP = 0.35

const OUTLIER = { x: 1.0, y: 0.2 }

const TEST_SEED = 2026
const TEST_PER_CLASS = 24

// 1c. THE TWO LINES
const BEFORE_HALF_WIDTH = 0.9
const C_HARD = 10000

// 1d. COLOURS
const COLORS = {
  blueClass: 'var(--c-blue)',
  orangeClass: 'var(--c-orange)',
  line: 'var(--c-axis, var(--c-light, #f8fafc))',
  band: 'var(--c-blue-highlight)',
  ringWrong: 'var(--c-red)',
  ringWrongFill: 'var(--c-red-highlight)',
  outlierMark: 'var(--c-axis, var(--c-light, #f8fafc))',
  axes: 'var(--c-axis, var(--c-light, #f8fafc))',
  controls: 'var(--c-axis, var(--c-light, #f8fafc))',
  halo: 'var(--c-dark)',
}

// 1e. SIZES AND STYLE
const VIEW = {
  width: 640,
  height: 400,
  pixelsPerUnit: 78,
  showGrid: true,
  gridHalfWidth: 4,
  gridHalfHeight: 2,
  showAxes: true,
}

const STYLE = {
  pointRadius: 6.5,
  pointOutline: 1.2,

  testRadius: 6.5,
  testOutline: 2.4,
  testFillOpacity: 0.25,

  ringRadius: 13,
  ringWidth: 3,
  ringPulseSeconds: 1.2,

  outlierPointRadius: 8.5,
  outlierRadius: 16,
  outlierRingWidth: 2.2,
  outlierLabel: 'outlier',
  outlierLabelSize: 15,
  outlierLabelHalo: 'var(--c-dark)',

  decisionLineWidth: 3.5,
  marginLineWidth: 2,
  marginDash: '6 5',

  axisWidth: 1.6,
  axisOpacity: 0.6,
  gridWidth: 1,
  gridOpacity: 0.12,
  bandOpacity: 0.16,

  captionSize: 20,
  switchFontSize: '0.78rem',
  maxHeight: '25rem',
}

/* ───────────────────────────── 2. TYPES & HELPERS ───────────────────────────── */

type Pt = {
  x: number
  y: number
  c: 1 | -1
}

type TestPt = Pt & {
  error: boolean
}

type Line = {
  phi: number
  d: number
  m: number
}

const clamp01 = (x: number) =>
  Math.min(1, Math.max(0, x))

const lerp = (a: number, b: number, t: number) =>
  a + (b - a) * t

const easeInOut = (u: number) =>
  u < 0.5
    ? 2 * u * u
    : 1 - (-2 * u + 2) ** 2 / 2

const smooth = (x: number) => {
  x = clamp01(x)
  return x * x * (3 - 2 * x)
}

const win = (p: number, a: number, b: number) =>
  smooth((p - a) / (b - a))

const outBack = (x: number) => {
  x = clamp01(x)
  const c1 = 1.70158
  const c3 = c1 + 1
  return 1 + c3 * (x - 1) ** 3 + c1 * (x - 1) ** 2
}

const range = (a: number, b: number) =>
  Array.from({ length: b - a + 1 }, (_, k) => a + k)

const sx = (x: number) =>
  VIEW.width / 2 + VIEW.pixelsPerUnit * x

const sy = (y: number) =>
  VIEW.height / 2 - VIEW.pixelsPerUnit * y

const clipId = 'ms-' + Math.random().toString(36).slice(2, 8)

/* ───────────────────────────── 3. DATA AND THE TWO LINES ───────────────────────────── */

function makeRng(seed: number) {
  return () => {
    seed |= 0
    seed = (seed + 0x6d2b79f5) | 0

    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t

    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}

const dx = BLUE.x - ORANGE.x
const dy = BLUE.y - ORANGE.y
const len = Math.hypot(dx, dy) || 1

const idealNormal = {
  x: dx / len,
  y: dy / len
}

const idealMid = {
  x: (BLUE.x + ORANGE.x) / 2,
  y: (BLUE.y + ORANGE.y) / 2
}

// Original training points. Generation is unchanged.
function makePoints(): Pt[] {
  const rand = makeRng(TRAIN_SEED)

  const gauss = () =>
    Math.sqrt(-2 * Math.log(1 - rand())) *
    Math.cos(2 * Math.PI * rand())

  const out: Pt[] = []

  for (const [c, cl] of [
    [1, BLUE],
    [-1, ORANGE]
  ] as const) {
    for (let n = 0; n < PER_CLASS; n++) {
      let x = 0
      let y = 0
      let tries = 0

      do {
        x = cl.x + SPREAD.x * gauss()
        y = cl.y + SPREAD.y * gauss()
        tries++
      } while (
        (
          c * (
            (x - idealMid.x) * idealNormal.x +
            (y - idealMid.y) * idealNormal.y
          ) < GAP ||
          Math.abs(x) > KEEP_WITHIN.x ||
          Math.abs(y) > KEEP_WITHIN.y
        ) &&
        tries < 200
      )

      out.push({ x, y, c })
    }
  }

  return out
}

const pts: Pt[] = [
  ...makePoints(),
  { x: OUTLIER.x, y: OUTLIER.y, c: -1 }
]

const outlierIndex = pts.length - 1

// Train a linear SVM using dual coordinate descent.
function trainSvm(C: number): Line {
  const BIAS = 5

  const X = pts.map(p => [p.x, p.y, BIAS])
  const Y = pts.map(p => p.c)

  const Q = X.map(
    x => x[0] ** 2 + x[1] ** 2 + x[2] ** 2
  )

  const a = X.map(() => 0)
  const w = [0, 0, 0]

  for (let sweep = 0; sweep < 4000; sweep++) {
    for (let i = 0; i < X.length; i++) {
      const G =
        Y[i] * (
          w[0] * X[i][0] +
          w[1] * X[i][1] +
          w[2] * X[i][2]
        ) - 1

      const next = Math.min(
        Math.max(a[i] - G / Q[i], 0),
        C
      )

      const delta = next - a[i]

      if (delta) {
        for (let j = 0; j < 3; j++) {
          w[j] += delta * Y[i] * X[i][j]
        }

        a[i] = next
      }
    }
  }

  const norm = Math.hypot(w[0], w[1])
  const b = w[2] * BIAS

  return {
    phi: Math.atan2(w[1], w[0]),
    d: -b / norm,
    m: 1 / norm
  }
}

// Initial line: slightly tilted.
const before: Line = {
  phi: Math.atan2(-0.2, 1.0),
  d: 0.1,
  m: BEFORE_HALF_WIDTH
}

// Final line: fitted using the original training data, including the outlier.
const after: Line = trainSvm(C_HARD)

// Original independent test dataset. Generation is unchanged.
function makeTestPoints(): Pt[] {
  const rand = makeRng(TEST_SEED)

  const gauss = () =>
    Math.sqrt(-2 * Math.log(1 - rand())) *
    Math.cos(2 * Math.PI * rand())

  const out: Pt[] = []

  for (const [c, cl] of [
    [1, BLUE],
    [-1, ORANGE]
  ] as const) {
    for (let n = 0; n < TEST_PER_CLASS; n++) {
      let x = 0
      let y = 0
      let tries = 0

      do {
        x = cl.x + SPREAD.x * gauss()
        y = cl.y + SPREAD.y * gauss()
        tries++
      } while (
        (
          c * (
            (x - idealMid.x) * idealNormal.x +
            (y - idealMid.y) * idealNormal.y
          ) < GAP ||
          Math.abs(x) > KEEP_WITHIN.x ||
          Math.abs(y) > KEEP_WITHIN.y
        ) &&
        tries < 200
      )

      out.push({ x, y, c })
    }
  }

  return out
}

// Predict using the trained SVM decision boundary.
function predict(p: Pt, L: Line): 1 | -1 {
  const score =
    Math.cos(L.phi) * p.x +
    Math.sin(L.phi) * p.y -
    L.d

  return score >= 0 ? 1 : -1
}

// Evaluate every test point against the trained SVM.
const testPts: TestPt[] = makeTestPoints().map(p => ({
  ...p,
  error: predict(p, after) !== p.c
}))

const testMistakes = testPts
  .map((p, i) => p.error ? i : -1)
  .filter(i => i !== -1)

// Same deterministic stagger convention as MarginDemo.
const rngDelay = makeRng(99)
const trainDelay = pts.map(() => rngDelay())
const testDelay = testPts.map(() => rngDelay())

/* ───────────────────────────── 4. STATE AND ANIMATION ───────────────────────────── */

type State = 'before' | 'after' | 'testing'

const current = ref<State>(props.state)
const t = ref(props.state === 'before' ? 0 : 1)

// Points appear independently of the line animation.
const trainIntro = ref(0)
const testIntro = ref(props.state === 'testing' ? 1 : 0)

let raf = 0
let trainRaf = 0
let testRaf = 0
let dead = false

function animateTo(target: number) {
  cancelAnimationFrame(raf)

  const from = t.value
  const t0 = performance.now()

  const tick = () => {
    if (dead) return

    const u = Math.min(
      (performance.now() - t0) / props.speed,
      1
    )

    t.value = lerp(from, target, easeInOut(u))

    if (u < 1) {
      raf = requestAnimationFrame(tick)
    }
  }

  raf = requestAnimationFrame(tick)
}

function animateReveal(
  target: { value: number },
  to: number,
  duration: number,
  kind: 'train' | 'test'
) {
  if (kind === 'train') cancelAnimationFrame(trainRaf)
  else cancelAnimationFrame(testRaf)

  const from = target.value
  const t0 = performance.now()

  const tick = () => {
    if (dead) return

    const u = Math.min((performance.now() - t0) / duration, 1)
    target.value = lerp(from, to, easeInOut(u))

    if (u < 1) {
      const id = requestAnimationFrame(tick)
      if (kind === 'train') trainRaf = id
      else testRaf = id
    }
  }

  const id = requestAnimationFrame(tick)
  if (kind === 'train') trainRaf = id
  else testRaf = id
}

function go(s: State) {
  current.value = s
  animateTo(s === 'before' ? 0 : 1)
  animateReveal(testIntro, s === 'testing' ? 1 : 0, 450, 'test')
}

const dropFocus = (e: Event) =>
  (e.currentTarget as HTMLElement).blur()

watch(
  () => props.state,
  s => go(s)
)

onBeforeUnmount(() => {
  dead = true
  cancelAnimationFrame(raf)
  cancelAnimationFrame(trainRaf)
  cancelAnimationFrame(testRaf)
})

// Slidev click progression:
// 0 clicks = Before Training
// 1 click  = After Training
// 2 clicks = Testing
if (props.clicks) {
  const ctx: any = useSlideContext()

  current.value = 'before'
  t.value = 0
  testIntro.value = 0

  watch(
    () => Number(unref(unref(ctx)?.$clicks) ?? 0),
    c => go(c >= 2 ? 'testing' : c >= 1 ? 'after' : 'before')
  )
}

// Animate the original training points into view on mount.
animateReveal(trainIntro, 1, 1600, 'train')

/* ───────────────────────────── 5. DRAWING ───────────────────────────── */

// Same line representation and lineAt geometry as MarginDemo.
const line = computed<Line>(() => ({
  phi: lerp(before.phi, after.phi, t.value),
  d: lerp(before.d, after.d, t.value),
  m: lerp(before.m, after.m, t.value)
}))

function lineAt(offset: number) {
  const L = line.value
  const c = Math.cos(L.phi)
  const s = Math.sin(L.phi)
  const px = (L.d + offset) * c
  const py = (L.d + offset) * s
  const far = 7

  return {
    x1: sx(px + far * s),
    y1: sy(py - far * c),
    x2: sx(px - far * s),
    y2: sy(py + far * c)
  }
}

const lines = computed(() => ({
  mid: lineAt(0),
  lo: lineAt(-line.value.m),
  hi: lineAt(line.value.m)
}))

const bandPoints = computed(() => {
  const { lo, hi } = lines.value
  return `${lo.x1},${lo.y1} ${lo.x2},${lo.y2} ${hi.x2},${hi.y2} ${hi.x1},${hi.y1}`
})

// Staggered point appearance, following MarginDemo.
const appear = (delay: number, progress: number) =>
  clamp01((progress - delay * 0.5) / 0.5)

const trainA = (i: number) =>
  appear(trainDelay[i], trainIntro.value)

const testA = (i: number) =>
  clamp01((testIntro.value - testDelay[i] * 0.6) / 0.4)

const testRingA = computed(() =>
  win(testIntro.value, 0.65, 1)
)

const gridX = range(
  -VIEW.gridHalfWidth,
  VIEW.gridHalfWidth
)

const gridY = range(
  -VIEW.gridHalfHeight,
  VIEW.gridHalfHeight
)

const cssVars = {
  '--ax': COLORS.controls,
  '--switch-size': STYLE.switchFontSize,
  '--max-h': STYLE.maxHeight,
  '--ring-w': String(STYLE.ringWidth),
  '--ring-pulse': `${STYLE.ringPulseSeconds}s`
}
</script>

<template>
  <div class="wrap" :style="cssVars">
    <svg
      class="plot"
      :viewBox="`0 0 ${VIEW.width} ${VIEW.height}`"
    >
      <defs>
        <clipPath :id="clipId">
          <rect
            x="0"
            y="0"
            :width="VIEW.width"
            :height="VIEW.height"
          />
        </clipPath>
      </defs>

      <!-- Grid -->
      <g
        v-if="VIEW.showGrid"
        :stroke="COLORS.axes"
        :stroke-width="STYLE.gridWidth"
        :opacity="STYLE.gridOpacity"
      >
        <line
          v-for="g in gridX"
          :key="'gx' + g"
          :x1="sx(g)"
          y1="0"
          :x2="sx(g)"
          :y2="VIEW.height"
        />

        <line
          v-for="g in gridY"
          :key="'gy' + g"
          x1="0"
          :y1="sy(g)"
          :x2="VIEW.width"
          :y2="sy(g)"
        />
      </g>

      <!-- Axes -->
      <g
        v-if="VIEW.showAxes"
        :stroke="COLORS.axes"
        :stroke-width="STYLE.axisWidth"
        :opacity="STYLE.axisOpacity"
      >
        <line
          x1="0"
          :y1="sy(0)"
          :x2="VIEW.width"
          :y2="sy(0)"
        />

        <line
          :x1="sx(0)"
          y1="0"
          :x2="sx(0)"
          :y2="VIEW.height"
        />
      </g>

      <!-- Margin band and separator -->
      <g :clip-path="`url(#${clipId})`">
        <polygon
          :points="bandPoints"
          :fill="COLORS.band"
          :opacity="STYLE.bandOpacity"
        />

        <line
          v-bind="lines.lo"
          :stroke="COLORS.orangeClass"
          :stroke-width="STYLE.marginLineWidth"
          :stroke-dasharray="STYLE.marginDash"
        />

        <line
          v-bind="lines.hi"
          :stroke="COLORS.blueClass"
          :stroke-width="STYLE.marginLineWidth"
          :stroke-dasharray="STYLE.marginDash"
        />

        <line
          v-bind="lines.mid"
          :stroke="COLORS.line"
          :stroke-width="STYLE.decisionLineWidth"
          stroke-linecap="round"
        />
      </g>

      <!-- Training points -->
      <circle
        v-for="(p, i) in pts"
        :key="'tr' + i"
        :cx="sx(p.x)"
        :cy="sy(p.y)"
        :r="(i === outlierIndex
          ? STYLE.outlierPointRadius
          : STYLE.pointRadius) * outBack(trainA(i))"
        :opacity="trainA(i)"
        :stroke="COLORS.axes"
        :stroke-width="i === outlierIndex ? 2.4 : STYLE.pointOutline"
        :fill="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass"
      />

      <!-- Test points: hollow-ish, appear in Testing -->
      <template v-if="current === 'testing'">
        <circle
          v-for="(p, i) in testPts"
          :key="'te' + i"
          :cx="sx(p.x)"
          :cy="sy(p.y)"
          :r="STYLE.testRadius * outBack(testA(i))"
          :opacity="testA(i)"
          :stroke="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass"
          :stroke-width="STYLE.testOutline"
          :fill="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass"
          :fill-opacity="STYLE.testFillOpacity"
        />
      </template>

      <!-- Misclassified test points -->
      <g v-if="current === 'testing'">
        <circle
          v-for="i in testMistakes"
          :key="'err' + i"
          class="ring"
          :opacity="testRingA * testA(i)"
          :cx="sx(testPts[i].x)"
          :cy="sy(testPts[i].y)"
          :r="STYLE.ringRadius"
          :stroke="COLORS.ringWrong"
          :fill="COLORS.ringWrongFill"
          fill-opacity="0.3"
        />
      </g>

      <!-- Outlier mark -->
      <g v-if="markOutlier" :opacity="trainA(outlierIndex)">
        <circle
          :cx="sx(OUTLIER.x)"
          :cy="sy(OUTLIER.y)"
          :r="STYLE.outlierRadius"
          fill="none"
          :stroke="COLORS.outlierMark"
          :stroke-width="STYLE.outlierRingWidth"
          stroke-dasharray="4 3"
        />

        <text
          v-if="STYLE.outlierLabel"
          :x="sx(OUTLIER.x) + STYLE.outlierRadius + 4"
          :y="sy(OUTLIER.y) - STYLE.outlierRadius - 2"
          :font-size="STYLE.outlierLabelSize"
          :fill="COLORS.outlierMark"
          font-style="italic"
          font-weight="600"
          :stroke="STYLE.outlierLabelHalo || 'none'"
          stroke-width="5"
          stroke-linejoin="round"
          paint-order="stroke"
        >
          {{ STYLE.outlierLabel }}
        </text>
      </g>

      <!-- Caption -->
      <text
        v-if="caption"
        x="18"
        :y="18 + STYLE.captionSize"
        :font-size="STYLE.captionSize"
        font-weight="600"
        :fill="COLORS.axes"
      >
        {{ caption }}
      </text>
    </svg>

    <!-- Three-way state switch -->
    <div
      v-if="controls"
      class="switch"
      role="group"
      aria-label="Training state"
    >
      <button
        :class="{ on: current === 'before' }"
        @click="go('before'); dropFocus($event)"
      >
        {{ LABELS.before }}
      </button>

      <button
        :class="{ on: current === 'after' }"
        @click="go('after'); dropFocus($event)"
      >
        {{ LABELS.after }}
      </button>

      <button
        :class="{ on: current === 'testing' }"
        @click="go('testing'); dropFocus($event)"
      >
        {{ LABELS.testing }}
      </button>
    </div>

    <template v-if="clicks">
      <span v-click hidden />
      <span v-click hidden />
    </template>
  </div>
</template>

<style scoped>
.wrap {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: .7rem;
}

.plot {
  width: 100%;
  max-height: var(--max-h);
  display: block;
}

.ring {
  stroke-width: var(--ring-w);
  transform-box: fill-box;
  transform-origin: center;
  animation: pulse var(--ring-pulse) ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { transform: scale(.9); }
  50% { transform: scale(1.2); }
}

.switch {
  display: inline-flex;
  border: 1.5px solid var(--ax);
  border-radius: 999px;
  overflow: hidden;
  color: var(--ax);
}

.switch button {
  background: none;
  border: 0;
  color: inherit;
  font: inherit;
  font-size: var(--switch-size);
  padding: .22rem .85rem;
  cursor: pointer;
  transition: background .15s, color .15s;
  white-space: nowrap;
}

.switch button:hover:not(.on) {
  background: color-mix(in srgb, var(--ax) 18%, transparent);
}

.switch button.on {
  background: var(--ax);
  color: var(--c-dark);
}
</style>
