
<script setup lang="ts">
import { ref, computed, watch, unref } from 'vue'
import { useSlideContext } from '@slidev/client'

/*
  SimpleLine: two nearby, linearly separable classes and their separator.

  Stages:
    0  Two clusters
    1  + Separating line
    2  + Actual equation of the line
*/

// ───────────────────────────── 1. SETTINGS ─────────────────────────────

// PROPS
const props = withDefaults(defineProps<{
  stage?: number
  clicks?: boolean
  equation?: string
  numeric?: boolean
  axisLabels?: boolean
  caption?: string
}>(), {
  stage: 2,
  clicks: false,
  equation: '',
  numeric: true,
  axisLabels: true,
  caption: ''
})

// DATA
const SEED = 3

// Direction perpendicular to the separating line.
// The two cluster centres lie on opposite sides of it.
const ANGLE = -25 * Math.PI / 180

const SEPARATION = 0.58
const TANGENT_SPREAD = 0.82
const NORMAL_SPREAD = 0.30
const MIN_CLEARANCE = 0.08

const PER_CLASS = 12

// Keep points within the visible plotting area.
const KEEP_WITHIN = { x: 3.6, y: 2.2 }

// THE LINE
const LINE = {
  rotateDeg: 0,
  shift: 0
}

// EQUATION POSITION AND STYLE
const EQUATION = {
  alongLineY: 1.75,
  offset: { x: -4.8, y: 0 },
  at: null as null | { x: number; y: number },
  anchor: 'start' as 'start' | 'middle' | 'end',
  size: 26,
  font: "'Times New Roman', Georgia, serif",
  italic: true
}

// COLOURS
const COLORS = {
  blueClass: 'var(--c-blue)',
  orangeClass: 'var(--c-orange)',
  line: 'var(--c-axis, var(--c-light, #f8fafc))',
  equation: 'var(--c-axis, var(--c-light, #f8fafc))',
  axes: 'var(--c-axis, var(--c-light, #f8fafc))',
  halo: 'var(--c-dark)'
}

// SIZES AND STYLE
const VIEW = {
  width: 640,
  height: 400,
  pixelsPerUnit: 78,
  showGrid: true,
  gridHalfWidth: 4,
  gridHalfHeight: 2,
  showAxes: true
}

const STYLE = {
  pointRadius: 7,
  pointOutline: 1.3,
  lineWidth: 4,
  axisWidth: 1.6,
  axisOpacity: 0.6,
  gridWidth: 1,
  gridOpacity: 0.12,
  axisLabelSize: 22,
  captionSize: 20,
  fadeSeconds: 0.6,
  drawSeconds: 0.9,
  maxHeight: '27rem'
}

// ───────────────────────────── 2. TYPES & HELPERS ─────────────────────────────

type Pt = { x: number; y: number; c: 1 | -1 }

const K = 2

const range = (a: number, b: number) =>
  Array.from({ length: b - a + 1 }, (_, k) => a + k)

const sx = (x: number) =>
  VIEW.width / 2 + VIEW.pixelsPerUnit * x

const sy = (y: number) =>
  VIEW.height / 2 - VIEW.pixelsPerUnit * y

const halfW = VIEW.width / 2 / VIEW.pixelsPerUnit
const halfH = VIEW.height / 2 / VIEW.pixelsPerUnit

// ───────────────────────────── 3. DATA AND LINE ─────────────────────────────

// Reproducible pseudo-random number generator.
function makeRng(seed: number) {
  return () => {
    seed |= 0
    seed = (seed + 0x6d2b79f5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}

// Gaussian random variable using Box-Muller.
function makeGaussian(rand: () => number) {
  return () => Math.sqrt(-2 * Math.log(1 - rand()))
    * Math.cos(2 * Math.PI * rand())
}

// Normal vector: perpendicular to the separator.
const nx = Math.cos(ANGLE)
const ny = Math.sin(ANGLE)

// Tangent vector: parallel to the separator.
const tx = -ny
const ty = nx

// Cluster centres, symmetrically positioned around the origin.
const BLUE = { x: 0.85, y: 0.95 }
const ORANGE = { x: -0.35, y: -0.55 }

const mid = {
  x: (BLUE.x + ORANGE.x) / 2,
  y: (BLUE.y + ORANGE.y) / 2
}

function makePoints(): Pt[] {
  const rand = makeRng(SEED)
  const gauss = makeGaussian(rand)
  const out: Pt[] = []

  for (const [c, centre] of [
    [1, BLUE],
    [-1, ORANGE]
  ] as const) {

    for (let n = 0; n < PER_CLASS; n++) {
      let x = 0
      let y = 0
      let tries = 0

      do {
        // Independent variation parallel and perpendicular
        // to the separating line.
        const along = TANGENT_SPREAD * gauss()
        const across = NORMAL_SPREAD * gauss()

        x = centre.x + along * tx + across * nx
        y = centre.y + along * ty + across * ny

        tries++

        // Only reject points on the wrong side of the line,
        // or points outside the plotting area.
      } while (
        (
          c * (x * nx + y * ny) < MIN_CLEARANCE ||
          Math.abs(x) > KEEP_WITHIN.x ||
          Math.abs(y) > KEEP_WITHIN.y
        ) && tries < 200
      )

      out.push({ x, y, c })
    }
  }

  return out
}

const pts = makePoints()

// THE SEPARATING LINE:
//
// a*x₁ + b*x₂ + c = 0
//
// (a,b) is a unit normal vector to the line.
// c determines its position.

const phi = ANGLE + LINE.rotateDeg * Math.PI / 180

const a = Math.cos(phi)
const b = Math.sin(phi)

const c = -(a * mid.x + b * mid.y + LINE.shift)

// Find the visible portion of the line.
function visibleSegment() {
  // A point on the line and its direction vector.
  const x0 = -c * a
  const y0 = -c * b

  const ux = -b
  const uy = a

  let t0 = -1e9
  let t1 = 1e9

  for (const [p0, u, lim] of [
    [x0, ux, halfW + 0.1],
    [y0, uy, halfH + 0.1]
  ] as const) {

    if (Math.abs(u) < 1e-9) continue

    const ta = (-lim - p0) / u
    const tb = (lim - p0) / u

    t0 = Math.max(t0, Math.min(ta, tb))
    t1 = Math.min(t1, Math.max(ta, tb))
  }

  return {
    x1: sx(x0 + t0 * ux),
    y1: sy(y0 + t0 * uy),
    x2: sx(x0 + t1 * ux),
    y2: sy(y0 + t1 * uy),
    tTop: [x0, y0, ux, uy] as const
  }
}

const seg = visibleSegment()

// ───────────────────────────── 4. STAGES ─────────────────────────────

const stage = ref(Math.min(K, Math.max(0, props.stage)))

watch(() => props.stage, s => {
  if (!props.clicks) {
    stage.value = Math.min(K, Math.max(0, s))
  }
})

if (props.clicks) {
  const ctx: any = useSlideContext()
  stage.value = 0

  watch(
    () => Number(unref(unref(ctx)?.$clicks) ?? 0),
    n => {
      stage.value = Math.min(K, Math.max(0, n))
    }
  )
}

// ───────────────────────────── 5. TEXT ─────────────────────────────

// Render subscripts using SVG tspan.
type Piece = { text: string; dy: number; size: number }

function pieces(str: string, size: number): Piece[] {
  const out: Piece[] = []
  let lowered = false

  for (const part of str.split(/(_\w)/).filter(Boolean)) {
    const sub = part.startsWith('_')
    const shift = size * 0.22

    out.push({
      text: sub ? part.slice(1) : part,
      dy: sub
        ? (lowered ? 0 : shift)
        : (lowered ? -shift : 0),
      size: sub ? size * 0.72 : size
    })

    lowered = sub
  }

  return out
}

const fmt = (v: number) =>
  (Math.abs(v) < 0.005 ? 0 : v).toFixed(2)

const sgn = (v: number) => v < 0 ? '−' : '+'

// The default equation always uses the actual line coefficients.
const equationText = computed(() => {
  if (!props.numeric && props.equation) return props.equation

  return `${fmt(a)} x_1 ${sgn(b)} ${fmt(Math.abs(b))} x_2 ${sgn(c)} ${fmt(Math.abs(c))} = 0`
})

const equationPieces = computed(() =>
  pieces(equationText.value, EQUATION.size)
)

const axisX = pieces('x_1', STYLE.axisLabelSize)
const axisY = pieces('x_2', STYLE.axisLabelSize)

// Position the equation near the line.
const equationPos = (() => {
  if (EQUATION.at) {
    return {
      x: sx(EQUATION.at.x),
      y: sy(EQUATION.at.y)
    }
  }

  const [x0, y0, ux, uy] = seg.tTop

  const t = Math.abs(uy) < 1e-9
    ? 0
    : (EQUATION.alongLineY - y0) / uy

  return {
    x: sx(x0 + t * ux + EQUATION.offset.x),
    y: sy(y0 + t * uy + EQUATION.offset.y)
  }
})()

const cssVars = {
  '--fade': `${STYLE.fadeSeconds}s`,
  '--draw': `${STYLE.drawSeconds}s`,
  '--max-h': STYLE.maxHeight
}

const gridX = range(-VIEW.gridHalfWidth, VIEW.gridHalfWidth)
const gridY = range(-VIEW.gridHalfHeight, VIEW.gridHalfHeight)

</script>

<template>
  <div class="wrap" :style="cssVars">
    <svg class="plot" :viewBox="`0 0 ${VIEW.width} ${VIEW.height}`">

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
        <line x1="0" :y1="sy(0)" :x2="VIEW.width" :y2="sy(0)" />
        <line :x1="sx(0)" y1="0" :x2="sx(0)" :y2="VIEW.height" />
      </g>

      <!-- Axis labels -->
      <g
        v-if="axisLabels"
        :fill="COLORS.axes"
        :font-family="EQUATION.font"
        font-style="italic"
        opacity="0.85"
      >
        <text
          :x="VIEW.width - 12"
          :y="sy(0) - 10"
          text-anchor="end"
          :font-size="STYLE.axisLabelSize"
        >
          <tspan
            v-for="(p, i) in axisX"
            :key="i"
            :dy="p.dy"
            :font-size="p.size"
          >{{ p.text }}</tspan>
        </text>

        <text
          :x="sx(0) + 12"
          y="26"
          :font-size="STYLE.axisLabelSize"
        >
          <tspan
            v-for="(p, i) in axisY"
            :key="i"
            :dy="p.dy"
            :font-size="p.size"
          >{{ p.text }}</tspan>
        </text>
      </g>

      <!-- Separating line -->
      <line
        class="draw"
        :x1="seg.x1"
        :y1="seg.y1"
        :x2="seg.x2"
        :y2="seg.y2"
        pathLength="1"
        :stroke="COLORS.line"
        :stroke-width="STYLE.lineWidth"
        stroke-linecap="round"
        :style="{ strokeDashoffset: stage >= 1 ? 0 : 1 }"
      />

      <!-- Data points -->
      <circle
        v-for="(p, i) in pts"
        :key="i"
        :cx="sx(p.x)"
        :cy="sy(p.y)"
        :r="STYLE.pointRadius"
        :stroke="COLORS.axes"
        :stroke-width="STYLE.pointOutline"
        :fill="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass"
      />

      <!-- Equation -->
      <text
        class="fade"
        :x="equationPos.x"
        :y="equationPos.y"
        :text-anchor="EQUATION.anchor"
        :fill="COLORS.equation"
        :font-family="EQUATION.font"
        :font-style="EQUATION.italic ? 'italic' : 'normal'"
        :font-size="EQUATION.size"
        :stroke="COLORS.halo || 'none'"
        stroke-width="6"
        stroke-linejoin="round"
        paint-order="stroke"
        :style="{ opacity: stage >= 2 ? 1 : 0 }"
      >
        <tspan
          v-for="(p, i) in equationPieces"
          :key="i"
          :dy="p.dy"
          :font-size="p.size"
        >{{ p.text }}</tspan>
      </text>

      <!-- Caption -->
      <text
        v-if="caption"
        x="18"
        :y="18 + STYLE.captionSize"
        :font-size="STYLE.captionSize"
        font-weight="600"
        :fill="COLORS.axes"
      >{{ caption }}</text>

    </svg>

    <template v-if="clicks">
      <span v-for="n in K" :key="n" v-click hidden />
    </template>
  </div>
</template>

<style scoped>
.wrap {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.plot {
  width: 100%;
  max-height: var(--max-h);
  display: block;
}

.draw {
  stroke-dasharray: 1;
  transition: stroke-dashoffset var(--draw) ease-in-out;
}

.fade {
  transition: opacity var(--fade);
}
</style>
