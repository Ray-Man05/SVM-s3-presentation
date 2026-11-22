<script setup lang="ts">
import { ref, computed, watch, unref } from 'vue'
import { useSlideContext } from '@slidev/client'

/* ════════════════════════════════════════════════════════════════════════════
   SupportVectors: what support vectors are, and their types.
   Everything you may want to change is in section 1. Sections 2-5 are the machinery.

   A real soft-margin SVM is trained on the data; each point is then classified by its score
   s = y·f(x)  (1 = exactly on its margin edge, 0 = on the line, negative = wrong side):
     s > 1       not a support vector: correct and clear of the margin  → greyed out
     s ≈ 1       support vector: correct, touching the margin
     0 < s < 1   support vector: correct, but inside the margin
     -1 < s < 0  support vector: MISCLASSIFIED, inside the margin
     s < -1      support vector: MISCLASSIFIED, outside the margin (far on the wrong side)

   Four stages (prop `stage`, or Slidev clicks with `clicks`):
     0  data, line and margin    1  non-support vectors greyed out
     2  support vectors ringed (colour = type)    3  callout labels
   ════════════════════════════════════════════════════════════════════════════ */

/* ───────────────────────────── 1. SETTINGS ───────────────────────────── */

// 1a. PROPS, set on the tag in slides.md:  <SupportVectors :stage="2" />   or   <SupportVectors clicks />
const props = withDefaults(defineProps<{
  stage?: number         // 0 data only · 1 + grey out non-support vectors · 2 + rings · 3 + callout labels (everything)
  clicks?: boolean       // true: Slidev clicks reveal the stages one by one (3 clicks), instead of `stage`
  greyOut?: boolean      // grey out the points that are not support vectors
  labels?: boolean       // show the callout labels (stage 3)
  caption?: string       // text in the top-left corner of the figure
}>(), { stage: 3, clicks: false, greyOut: true, labels: true, caption: '' })

// 1b. DATA. Blue is class +1 (right), orange is class −1 (left).
const SEED = 21                                           // random scatter of the two groups
const BLUE = { x: 1.4, y: 0 }, ORANGE = { x: -1.4, y: 0 }   // cluster centres
const SPREAD = { x: 0.5, y: 0.85 }                        // how spread out each cluster is
const KEEP_WITHIN = { x: 3.8, y: 2.2 }                    // a point that would land outside ±these is re-drawn
const PER_CLASS = 10
const GAP = 0.4                                           // empty corridor kept around the ideal boundary
// Two hand-placed points that create the two kinds of misclassification:
const OUTSIDE_WRONG = { x: 1.4, y: 0.4, c: -1 as const }  // an orange point deep in blue territory
const INSIDE_WRONG  = { x: -0.6, y: 0.0, c: 1 as const }  // a blue point just over the line, inside the margin
// If you change any of the above, the support-vector types change too: check that each one still occurs.

// 1c. THE MODEL
const C = 1                  // soft-margin strength (smaller = softer, wider margin)
const ON_MARGIN_TOLERANCE = 0.04   // how close to the margin edge counts as "touching" it

// 1d. TYPES: ring colour and label for each kind of support vector. Any CSS colour.
const COLORS = {
  blueClass:     'var(--c-blue)',
  orangeClass:   'var(--c-orange)',
  greyedOut:     'var(--c-axis, var(--c-light, #f8fafc))',   // fill of non-support vectors (drawn at greyOpacity)
  line:          'var(--c-axis, var(--c-light, #f8fafc))',   // the decision line
  band:          'var(--c-blue-highlight)',                  // shaded margin band
  axes:          'var(--c-axis, var(--c-light, #f8fafc))',   // axes, grid, point outlines, caption
  labelHalo:     'var(--c-dark)',                            // outline behind labels so they read over dots: match your slide background, or '' for none
}
const TYPES = {
  onMargin:      { label: 'Correct, touches the margin',        color: 'var(--c-green)' },
  insideCorrect: { label: 'Correct, inside the margin',         color: 'var(--c-blue-highlight)' },
  insideWrong:   { label: 'Misclassified, inside the margin',   color: 'var(--c-red)' },
  outsideWrong:  { label: 'Misclassified, outside the margin',  color: 'var(--c-red-highlight)' },
}
type Kind = keyof typeof TYPES

// 1e. CALLOUTS: which types get a label, where the label goes, and which example point it points at.
//     at: label position in data units · anchor: which part of the text sits at `at` · pick: which point of that type (0 = first)
const CALLOUTS: { kind: Kind; at: { x: number; y: number }; anchor: 'start' | 'middle' | 'end'; pick?: number }[] = [
  { kind: 'onMargin',     at: { x: 0.9,  y: -2.05 }, anchor: 'start',  pick: 1 },
  { kind: 'insideWrong',  at: { x: -0.7, y: 1.9 },   anchor: 'end' },
  { kind: 'outsideWrong', at: { x: 2.65, y: 2.2 },   anchor: 'middle' },
]
// (a type with no points in the data, e.g. insideCorrect, is simply not shown)

// 1f. SIZES AND STYLE
const VIEW = {
  width: 640, height: 400,   // internal drawing size (only the ratio matters; the plot scales to fit)
  pixelsPerUnit: 78,         // zoom
  showGrid: true, gridHalfWidth: 4, gridHalfHeight: 2, showAxes: true,
}
const STYLE = {
  pointRadius: 6.5, pointOutline: 1.2, greyOpacity: 0.28,
  ringRadius: 13, ringWidth: 3,
  labelSize: 15, labelWeight: 600, leaderWidth: 1.8, captionSize: 20,
  decisionLineWidth: 3.5, marginLineWidth: 2, marginDash: '6 5',
  axisWidth: 1.6, axisOpacity: 0.6, gridWidth: 1, gridOpacity: 0.12, bandOpacity: 0.16,
  fadeSeconds: 0.5,          // how fast things fade between stages
  maxHeight: '27rem',        // the figure never grows taller than this
}

/* ───────────────────────────── 2. TYPES & HELPERS ───────────────────────────── */

type Pt = { x: number; y: number; c: 1 | -1 }
type Line = { phi: number; d: number; m: number }
//  A line: unit normal at angle `phi` (pointing toward blue), distance `d` from the origin, margin half-width `m`.

const range = (a: number, b: number) => Array.from({ length: b - a + 1 }, (_, k) => a + k)
const K = 3                                               // last stage

/* ───────────────────────────── 3. DATA, MODEL, TYPES ───────────────────────────── */

function makeRng(seed: number) {
  return () => { seed |= 0; seed = (seed + 0x6d2b79f5) | 0; let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296 }
}
function makeCluster(): Pt[] {
  const rand = makeRng(SEED)
  const gauss = () => Math.sqrt(-2 * Math.log(1 - rand())) * Math.cos(2 * Math.PI * rand())
  const dx = BLUE.x - ORANGE.x, dy = BLUE.y - ORANGE.y, len = Math.hypot(dx, dy) || 1       // ideal boundary: perpendicular bisector
  const ux = dx / len, uy = dy / len, mx = (BLUE.x + ORANGE.x) / 2, my = (BLUE.y + ORANGE.y) / 2
  const out: Pt[] = []
  for (const [c, cl] of [[1, BLUE], [-1, ORANGE]] as const) for (let n = 0; n < PER_CLASS; n++) {
    let x = 0, y = 0, tries = 0
    do { x = cl.x + SPREAD.x * gauss(); y = cl.y + SPREAD.y * gauss(); tries++ }
    while ((c * ((x - mx) * ux + (y - my) * uy) < GAP || Math.abs(x) > KEEP_WITHIN.x || Math.abs(y) > KEEP_WITHIN.y) && tries < 200)
    out.push({ x, y, c })
  }
  return out
}
const pts: Pt[] = [...makeCluster(), OUTSIDE_WRONG, INSIDE_WRONG]

// linear soft-margin SVM by dual coordinate descent
function trainSvm(): Line {
  const BIAS = 5, X = pts.map(p => [p.x, p.y, BIAS]), Y = pts.map(p => p.c)
  const Q = X.map(x => x[0] ** 2 + x[1] ** 2 + x[2] ** 2), a = X.map(() => 0), w = [0, 0, 0]
  for (let sweep = 0; sweep < 3000; sweep++) for (let i = 0; i < X.length; i++) {
    const G = Y[i] * (w[0] * X[i][0] + w[1] * X[i][1] + w[2] * X[i][2]) - 1
    const next = Math.min(Math.max(a[i] - G / Q[i], 0), C), delta = next - a[i]
    if (delta) { for (let j = 0; j < 3; j++) w[j] += delta * Y[i] * X[i][j]; a[i] = next }
  }
  const norm = Math.hypot(w[0], w[1]), b = w[2] * BIAS
  return { phi: Math.atan2(w[1], w[0]), d: -b / norm, m: 1 / norm }
}
const model = trainSvm()

// score s = y·f(x), with f scaled so the margin edges are at ±1
const score = (p: Pt) => (p.c * (Math.cos(model.phi) * p.x + Math.sin(model.phi) * p.y - model.d)) / model.m
function kindOf(s: number): Kind | null {                  // null = not a support vector
  if (s > 1 + ON_MARGIN_TOLERANCE) return null
  if (Math.abs(s - 1) <= ON_MARGIN_TOLERANCE) return 'onMargin'
  if (s > 0) return 'insideCorrect'
  if (s > -1) return 'insideWrong'
  return 'outsideWrong'
}
const kinds = pts.map(p => kindOf(score(p)))
const callouts = CALLOUTS.flatMap(co => {                  // resolve each callout to its target point
  const idx = kinds.map((k, i) => (k === co.kind ? i : -1)).filter(i => i >= 0)
  const target = idx[Math.min(co.pick ?? 0, idx.length - 1)]
  return target === undefined ? [] : [{ ...co, target }]
})

/* ───────────────────────────── 4. STAGES ───────────────────────────── */

const stage = ref(Math.min(K, Math.max(0, props.stage)))
watch(() => props.stage, s => { if (!props.clicks) stage.value = Math.min(K, Math.max(0, s)) })
if (props.clicks) {                                        // Slidev clicks reveal the stages
  const ctx: any = useSlideContext()
  stage.value = 0
  watch(() => Number(unref(unref(ctx)?.$clicks) ?? 0), c => { stage.value = Math.min(K, Math.max(0, c)) })
}
const isGrey = (i: number) => props.greyOut && stage.value >= 1 && kinds[i] === null
const ringsOn = computed(() => stage.value >= 2)
const labelsOn = computed(() => stage.value >= 3 && props.labels)

/* ───────────────────────────── 5. DRAWING ───────────────────────────── */

const sx = (x: number) => VIEW.width / 2 + VIEW.pixelsPerUnit * x
const sy = (y: number) => VIEW.height / 2 - VIEW.pixelsPerUnit * y
const gridX = range(-VIEW.gridHalfWidth, VIEW.gridHalfWidth), gridY = range(-VIEW.gridHalfHeight, VIEW.gridHalfHeight)
function lineAt(offset: number) {
  const c = Math.cos(model.phi), s = Math.sin(model.phi), px = (model.d + offset) * c, py = (model.d + offset) * s, far = 20
  return { x1: sx(px + far * s), y1: sy(py - far * c), x2: sx(px - far * s), y2: sy(py + far * c) }
}
const lines = { mid: lineAt(0), lo: lineAt(-model.m), hi: lineAt(model.m) }
const bandPoints = `${lines.lo.x1},${lines.lo.y1} ${lines.lo.x2},${lines.lo.y2} ${lines.hi.x2},${lines.hi.y2} ${lines.hi.x1},${lines.hi.y1}`

const leaders = callouts.map(co => {                       // label position and a leader line that stops at the ring
  const lx = sx(co.at.x), ly = sy(co.at.y), p = pts[co.target], px = sx(p.x), py = sy(p.y)
  const y0 = py > ly ? ly + 6 : ly - STYLE.labelSize - 2                           // leave from the near edge of the text
  const dx = px - lx, dy = py - y0, dist = Math.hypot(dx, dy) || 1, stop = STYLE.ringRadius + 4
  return { ...co, lx, ly, x1: lx, y1: y0, x2: px - (dx / dist) * stop, y2: py - (dy / dist) * stop }
})
const cssVars = { '--fade': `${STYLE.fadeSeconds}s`, '--max-h': STYLE.maxHeight }
</script>

<template>
  <div class="wrap" :style="cssVars">
    <svg class="plot" :viewBox="`0 0 ${VIEW.width} ${VIEW.height}`">
      <g v-if="VIEW.showGrid" :stroke="COLORS.axes" :stroke-width="STYLE.gridWidth" :opacity="STYLE.gridOpacity">
        <line v-for="g in gridX" :key="'gx' + g" :x1="sx(g)" y1="0" :x2="sx(g)" :y2="VIEW.height" />
        <line v-for="g in gridY" :key="'gy' + g" x1="0" :y1="sy(g)" :x2="VIEW.width" :y2="sy(g)" />
      </g>
      <g v-if="VIEW.showAxes" :stroke="COLORS.axes" :stroke-width="STYLE.axisWidth" :opacity="STYLE.axisOpacity">
        <line x1="0" :y1="sy(0)" :x2="VIEW.width" :y2="sy(0)" />
        <line :x1="sx(0)" y1="0" :x2="sx(0)" :y2="VIEW.height" />
      </g>

      <polygon :points="bandPoints" :fill="COLORS.band" :opacity="STYLE.bandOpacity" />
      <line v-bind="lines.lo" :stroke="COLORS.orangeClass" :stroke-width="STYLE.marginLineWidth" :stroke-dasharray="STYLE.marginDash" />
      <line v-bind="lines.hi" :stroke="COLORS.blueClass" :stroke-width="STYLE.marginLineWidth" :stroke-dasharray="STYLE.marginDash" />
      <line v-bind="lines.mid" :stroke="COLORS.line" :stroke-width="STYLE.decisionLineWidth" stroke-linecap="round" />

      <!-- points: non-support vectors fade to grey -->
      <circle v-for="(p, i) in pts" :key="i" class="pt" :cx="sx(p.x)" :cy="sy(p.y)" :r="STYLE.pointRadius"
              :style="{ fill: isGrey(i) ? COLORS.greyedOut : (p.c === 1 ? COLORS.blueClass : COLORS.orangeClass), opacity: isGrey(i) ? STYLE.greyOpacity : 1 }"
              :stroke="COLORS.axes" :stroke-width="STYLE.pointOutline" />

      <!-- rings around the support vectors, coloured by type -->
      <g class="fade" :style="{ opacity: ringsOn ? 1 : 0 }">
        <template v-for="(k, i) in kinds" :key="'r' + i">
          <circle v-if="k" :cx="sx(pts[i].x)" :cy="sy(pts[i].y)" :r="STYLE.ringRadius" fill="none" :stroke="TYPES[k].color" :stroke-width="STYLE.ringWidth" />
        </template>
      </g>

      <!-- callouts -->
      <g class="fade" :style="{ opacity: labelsOn ? 1 : 0 }">
        <g v-for="l in leaders" :key="'l' + l.kind">
          <line :x1="l.x1" :y1="l.y1" :x2="l.x2" :y2="l.y2" :stroke="TYPES[l.kind].color" :stroke-width="STYLE.leaderWidth" stroke-linecap="round" />
          <text :x="l.lx" :y="l.ly" :text-anchor="l.anchor" :font-size="STYLE.labelSize" :font-weight="STYLE.labelWeight" :fill="TYPES[l.kind].color"
                :stroke="COLORS.labelHalo || 'none'" stroke-width="5" stroke-linejoin="round" paint-order="stroke">{{ TYPES[l.kind].label }}</text>
        </g>
      </g>

      <text v-if="caption" x="18" :y="18 + STYLE.captionSize" :font-size="STYLE.captionSize" font-weight="600" :fill="COLORS.axes">{{ caption }}</text>
    </svg>
    <template v-if="clicks"><span v-for="n in K" :key="n" v-click hidden /></template>
  </div>
</template>

<style scoped>
.wrap { width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; }
.plot { width: 100%; max-height: var(--max-h); display: block; }
.pt { transition: fill var(--fade), opacity var(--fade); }
.fade { transition: opacity var(--fade); }
</style>