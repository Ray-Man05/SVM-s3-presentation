<script setup lang="ts">
import { computed } from 'vue'

const props = withDefaults(defineProps<{
  layout?: 'vertical' | 'horizontal'
}>(), {
  layout: 'vertical',
})

/* ───────────────────────────── SETTINGS ───────────────────────────── */

type Pt = {
  x: number
  y: number
  c: 1 | -1
}

const pts: Pt[] = [
  // Blue class (+1)
  { x: -2.1, y: 1.6, c: 1 },
  { x: -1.5, y: 1.9, c: 1 },
  { x: -1.8, y: 0.9, c: 1 },
  { x: -0.9, y: 1.3, c: 1 },
  { x: -1.2, y: 0.5, c: 1 },
  { x: -0.4, y: 1.0, c: 1 },
  { x: -0.5, y: 0.3, c: 1 },
  { x: -2.3, y: 0.5, c: 1 },

  // Orange class (-1)
  { x: 1.7, y: -1.6, c: -1 },
  { x: 1.3, y: -0.9, c: -1 },
  { x: 2.1, y: -0.7, c: -1 },
  { x: 0.9, y: -1.5, c: -1 },
  { x: 0.6, y: -0.6, c: -1 },
  { x: 1.5, y: 0.0, c: -1 },
  { x: 0.5, y: -1.9, c: -1 },
  { x: 2.3, y: -1.8, c: -1 },
]

const K = 5
const QUERY = { x: 0.1, y: 0.7 }

const DISTANCE_POWER = 1
const DISTANCE_EPSILON = 0.0001

const SVM_C = 10
const SVM_BIAS = 5
const SVM_SWEEPS = 5000
const SV_THRESHOLD = 0.0001

const COLORS = {
  blue: 'var(--c-blue)',
  orange: 'var(--c-orange)',
  green: 'var(--c-green)',
  axis: 'var(--c-axis, var(--c-light, #f8fafc))',
}

const VIEW = {
  width: 560,
  height: 400,
  pixelsPerUnit: 76,
  gridHalfWidth: 3,
  gridHalfHeight: 2,
}

const STYLE = {
  pointRadius: 7,
  queryRadius: 9,
  pointOutline: 1.5,
  inactiveOpacity: 0.22,
  edgeWidth: 1.6,
  edgeOpacity: 0.45,
  ringWidth: 2.5,
  decisionWidth: 3,
  marginWidth: 1.7,
  marginDash: '6 5',
}

/* ───────────────────────────── HELPERS ───────────────────────────── */

type WeightedPoint = {
  index: number
  distance: number
  weight: number
  normalized: number
}

type SvmResult = {
  w: number[]
  b: number
  alpha: number[]
  supportVectors: number[]
}

const clamp = (x: number, min: number, max: number) =>
  Math.max(min, Math.min(max, x))

const range = (min: number, max: number) =>
  Array.from({ length: max - min + 1 }, (_, i) => min + i)

const distance = (
  a: { x: number; y: number },
  b: { x: number; y: number },
) => Math.hypot(a.x - b.x, a.y - b.y)

/* ───────────────────────────── KNN ───────────────────────────── */

const neighbors = computed<WeightedPoint[]>(() => {
  const selected = pts
    .map((p, index) => ({
      index,
      distance: distance(p, QUERY),
    }))
    .sort((a, b) => a.distance - b.distance)
    .slice(0, K)

  const exact = selected.filter(
    p => p.distance < DISTANCE_EPSILON,
  )

  const raw = selected.map(p => ({
    ...p,
    weight: exact.length
      ? (p.distance < DISTANCE_EPSILON ? 1 : 0)
      : 1 / Math.pow(p.distance, DISTANCE_POWER),
  }))

  const maxWeight = Math.max(...raw.map(p => p.weight))

  return raw.map(p => ({
    ...p,
    normalized: maxWeight > 0 ? p.weight / maxWeight : 0,
  }))
})

const neighborIds = computed(
  () => new Set(neighbors.value.map(p => p.index)),
)

/* ───────────────────────────── SVM ───────────────────────────── */

function trainSvm(): SvmResult {
  const X = pts.map(p => [p.x, p.y, SVM_BIAS])
  const Y = pts.map(p => p.c)

  const Q = X.map(x =>
    x[0] ** 2 + x[1] ** 2 + x[2] ** 2,
  )

  const alpha = pts.map(() => 0)
  const w = [0, 0, 0]

  for (let sweep = 0; sweep < SVM_SWEEPS; sweep++) {
    let changed = 0

    for (let i = 0; i < X.length; i++) {
      const dot =
        w[0] * X[i][0] +
        w[1] * X[i][1] +
        w[2] * X[i][2]

      const gradient = Y[i] * dot - 1

      const next = clamp(
        alpha[i] - gradient / Q[i],
        0,
        SVM_C,
      )

      const delta = next - alpha[i]

      if (Math.abs(delta) > 1e-10) {
        for (let j = 0; j < 3; j++) {
          w[j] += delta * Y[i] * X[i][j]
        }

        alpha[i] = next
        changed++
      }
    }

    if (!changed) break
  }

  const supportVectors = alpha
    .map((a, i) => a > SV_THRESHOLD ? i : -1)
    .filter(i => i !== -1)

  return {
    w: [w[0], w[1]],
    b: w[2] * SVM_BIAS,
    alpha,
    supportVectors,
  }
}

const svm = trainSvm()

const svmSupportIds = computed(
  () => new Set(svm.supportVectors),
)

const maxAlpha = Math.max(...svm.alpha)

const svmWeights = computed(() =>
  svm.alpha.map(a => ({
    alpha: a,
    normalized: maxAlpha > 0 ? a / maxAlpha : 0,
  })),
)

/* ───────────────────────────── DRAWING ───────────────────────────── */

const sx = (x: number) =>
  VIEW.width / 2 + VIEW.pixelsPerUnit * x

const sy = (y: number) =>
  VIEW.height / 2 - VIEW.pixelsPerUnit * y

const gridX = range(-VIEW.gridHalfWidth, VIEW.gridHalfWidth)
const gridY = range(-VIEW.gridHalfHeight, VIEW.gridHalfHeight)

function lineAt(level: number) {
  const [a, b] = svm.w
  const c = level - svm.b

  const hits: { x: number; y: number }[] = []

  const min = -VIEW.width / 2 / VIEW.pixelsPerUnit
  const max = VIEW.width / 2 / VIEW.pixelsPerUnit
  const ymin = -VIEW.height / 2 / VIEW.pixelsPerUnit
  const ymax = VIEW.height / 2 / VIEW.pixelsPerUnit

  if (Math.abs(b) > 1e-8) {
    for (const x of [min, max]) {
      const y = (c - a * x) / b
      if (y >= ymin && y <= ymax) hits.push({ x, y })
    }
  }

  if (Math.abs(a) > 1e-8) {
    for (const y of [ymin, ymax]) {
      const x = (c - b * y) / a
      if (x >= min && x <= max) hits.push({ x, y })
    }
  }

  const unique = hits.filter((p, i) =>
    hits.findIndex(q =>
      Math.hypot(p.x - q.x, p.y - q.y) < 1e-5,
    ) === i,
  )

  if (unique.length < 2) return null

  return {
    x1: sx(unique[0].x),
    y1: sy(unique[0].y),
    x2: sx(unique[1].x),
    y2: sy(unique[1].y),
  }
}

const decisionLine = computed(() => lineAt(0))
const marginPositive = computed(() => lineAt(1))
const marginNegative = computed(() => lineAt(-1))
</script>

<template>
  <div class="wrap" :class="`layout-${props.layout}`">

    <!-- KNN -->

    <section class="algorithm">
      <div class="algorithm-name">KNN</div>

      <svg
        class="plot"
        :viewBox="`0 0 ${VIEW.width} ${VIEW.height}`"
      >
        <!-- Grid -->
        <g
          stroke="var(--c-axis, var(--c-light, #f8fafc))"
          stroke-width="1"
          opacity=".12"
        >
          <line
            v-for="g in gridX"
            :key="'knn-gx' + g"
            :x1="sx(g)"
            y1="0"
            :x2="sx(g)"
            :y2="VIEW.height"
          />
          <line
            v-for="g in gridY"
            :key="'knn-gy' + g"
            x1="0"
            :y1="sy(g)"
            :x2="VIEW.width"
            :y2="sy(g)"
          />
        </g>

        <!-- Axes -->
        <g
          :stroke="COLORS.axis"
          stroke-width="1.5"
          opacity=".5"
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

        <!-- Lines to the five nearest neighbours -->
        <g>
          <line
            v-for="p in neighbors"
            :key="'edge' + p.index"
            :x1="sx(QUERY.x)"
            :y1="sy(QUERY.y)"
            :x2="sx(pts[p.index].x)"
            :y2="sy(pts[p.index].y)"
            :stroke="pts[p.index].c === 1 ? COLORS.blue : COLORS.orange"
            :stroke-width="STYLE.edgeWidth * (.45 + p.normalized * .55)"
            :opacity="STYLE.edgeOpacity + p.normalized * .35"
            stroke-linecap="round"
          />
        </g>

        <!-- Training points -->
        <g v-for="(p, i) in pts" :key="'knn' + i">
          <circle
            v-if="neighborIds.has(i)"
            :cx="sx(p.x)"
            :cy="sy(p.y)"
            :r="STYLE.pointRadius + 5 * neighbors.find(n => n.index === i)!.normalized"
            :stroke="p.c === 1 ? COLORS.blue : COLORS.orange"
            fill="none"
            :stroke-width="STYLE.ringWidth"
            opacity=".85"
          />

          <circle
            :cx="sx(p.x)"
            :cy="sy(p.y)"
            :r="STYLE.pointRadius + (neighborIds.has(i) ? 2 * neighbors.find(n => n.index === i)!.normalized : 0)"
            :fill="p.c === 1 ? COLORS.blue : COLORS.orange"
            :stroke="COLORS.axis"
            :stroke-width="STYLE.pointOutline"
            :opacity="neighborIds.has(i) ? 1 : STYLE.inactiveOpacity"
          />
        </g>

        <!-- Query point -->
        <g>
          <circle
            :cx="sx(QUERY.x)"
            :cy="sy(QUERY.y)"
            :r="STYLE.queryRadius + 6"
            fill="none"
            :stroke="COLORS.axis"
            stroke-width="1.5"
            opacity=".55"
          />
          <circle
            :cx="sx(QUERY.x)"
            :cy="sy(QUERY.y)"
            :r="STYLE.queryRadius"
            :fill="COLORS.green"
            :stroke="COLORS.axis"
            stroke-width="2"
          />
          <path
            :d="`M${sx(QUERY.x)-4} ${sy(QUERY.y)}h8 M${sx(QUERY.x)} ${sy(QUERY.y)-4}v8`"
            stroke="var(--c-dark)"
            stroke-width="1.8"
          />
        </g>
      </svg>

  
    </section>

    <!-- SVM -->

    <section class="algorithm">
      <div class="algorithm-name">SVM</div>

      <svg
        class="plot"
        :viewBox="`0 0 ${VIEW.width} ${VIEW.height}`"
      >
        <!-- Grid -->
        <g
          stroke="var(--c-axis, var(--c-light, #f8fafc))"
          stroke-width="1"
          opacity=".12"
        >
          <line
            v-for="g in gridX"
            :key="'svm-gx' + g"
            :x1="sx(g)"
            y1="0"
            :x2="sx(g)"
            :y2="VIEW.height"
          />
          <line
            v-for="g in gridY"
            :key="'svm-gy' + g"
            x1="0"
            :y1="sy(g)"
            :x2="VIEW.width"
            :y2="sy(g)"
          />
        </g>

        <!-- Axes -->
        <g
          :stroke="COLORS.axis"
          stroke-width="1.5"
          opacity=".5"
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

        <!-- Margins -->
        <line
          v-if="marginPositive"
          v-bind="marginPositive"
          :stroke="COLORS.blue"
          :stroke-width="STYLE.marginWidth"
          :stroke-dasharray="STYLE.marginDash"
          opacity=".75"
        />

        <line
          v-if="marginNegative"
          v-bind="marginNegative"
          :stroke="COLORS.orange"
          :stroke-width="STYLE.marginWidth"
          :stroke-dasharray="STYLE.marginDash"
          opacity=".75"
        />

        <!-- Decision boundary -->
        <line
          v-if="decisionLine"
          v-bind="decisionLine"
          :stroke="COLORS.green"
          :stroke-width="STYLE.decisionWidth"
          stroke-linecap="round"
        />

        <!-- Training points -->
        <g v-for="(p, i) in pts" :key="'svm' + i">
          <circle
            v-if="svmSupportIds.has(i)"
            :cx="sx(p.x)"
            :cy="sy(p.y)"
            :r="STYLE.pointRadius + 4 + 5 * svmWeights[i].normalized"
            :stroke="COLORS.green"
            fill="none"
            :stroke-width="STYLE.ringWidth"
            opacity=".9"
          />

          <circle
            :cx="sx(p.x)"
            :cy="sy(p.y)"
            :r="STYLE.pointRadius + (svmSupportIds.has(i) ? 2 * svmWeights[i].normalized : 0)"
            :fill="p.c === 1 ? COLORS.blue : COLORS.orange"
            :stroke="COLORS.axis"
            :stroke-width="STYLE.pointOutline"
            :opacity="svmSupportIds.has(i) ? 1 : STYLE.inactiveOpacity"
          />
        </g>

        <!-- Query point -->
        <g>
          <circle
            :cx="sx(QUERY.x)"
            :cy="sy(QUERY.y)"
            :r="STYLE.queryRadius + 6"
            fill="none"
            :stroke="COLORS.axis"
            stroke-width="1.5"
            opacity=".55"
          />
          <circle
            :cx="sx(QUERY.x)"
            :cy="sy(QUERY.y)"
            :r="STYLE.queryRadius"
            :fill="COLORS.green"
            :stroke="COLORS.axis"
            stroke-width="2"
          />
          <path
            :d="`M${sx(QUERY.x)-4} ${sy(QUERY.y)}h8 M${sx(QUERY.x)} ${sy(QUERY.y)-4}v8`"
            stroke="var(--c-dark)"
            stroke-width="1.8"
          />
        </g>
      </svg>


    </section>

  </div>
</template>

<style scoped>
.wrap {
  display: grid;
  gap: 1rem;
  align-items: start;
  width: 100%;
  height: 100%;
  color: var(--c-axis, var(--c-light, #f8fafc));
}

.wrap.layout-vertical {
  grid-template-columns: minmax(0, 1fr);
}

.wrap.layout-horizontal {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.algorithm {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: .45rem;
}

.algorithm-name {
  font-size: .85rem;
  font-weight: 800;
  letter-spacing: .07em;
}

.plot {
  display: block;
  width: 100%;
  height: auto;
  max-height: 260px;
  overflow: visible;
}

.weight-section {
  display: flex;
  flex-direction: column;
  gap: .3rem;
  min-width: 0;
}

.svm-weights {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  column-gap: 1rem;
}

.weight-row {
  display: grid;
  grid-template-columns: 7px 2rem 3.2rem minmax(20px, 1fr) 2rem;
  align-items: center;
  gap: .35rem;
  font-size: .7rem;
  min-width: 0;
}

.weight-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
}

.weight-id {
  opacity: .7;
  font-weight: 700;
}

.weight-value {
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
  opacity: .8;
}

.weight-track {
  height: 6px;
  border-radius: 999px;
  background: color-mix(in srgb, var(--c-axis, var(--c-light, #f8fafc)) 15%, transparent);
  overflow: hidden;
}

.weight-bar {
  height: 100%;
  border-radius: 999px;
  transition: width .2s;
}

.weight-row strong {
  text-align: right;
  font-variant-numeric: tabular-nums;
  font-size: .7rem;
}

@media (max-width: 650px) {
  .wrap.layout-horizontal {
    grid-template-columns: 1fr;
  }

  .plot {
    max-height: 220px;
  }
}
</style>