<script setup lang="ts">
import katex from 'katex'
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

/* ════════════════════════════════════════════════════════════════════════════
   RbfMatrix: pedagogical 2D RBF kernel similarity table & query classifier
   k(x, z) = exp(−γ · ||x − z||²), where ||x − z||² = (x₁ − z₁)² + (x₂ − z₂)²
   ════════════════════════════════════════════════════════════════════════════ */

/* ───────────────────────────── 1. SETTINGS ───────────────────────────── */

export interface DataPoint {
  name: string
  tex?: string         // optional LaTeX label (defaults to A1 -> A_1, B2 -> B_2, or \text{...})
  x: [number, number]
  cls: 'A' | 'B' | 'Q' // 'A' = Class A, 'B' = Class B, 'Q' = Query / Unlabeled point
}

const props = withDefaults(defineProps<{
  full?: boolean       // false (default): upper triangle for symmetric training pairs; true: full matrix
  showQuery?: boolean  // true: include unlabeled query point ('Q') to classify
}>(), {
  full: false,
  showQuery: true,
})

const POINTS: DataPoint[] = [
  { name: 'A1', tex: 'A_1', x: [3, 3], cls: 'A' },
  { name: 'A2', tex: 'A_2', x: [4, 2], cls: 'A' },
  { name: 'B1', tex: 'B_1', x: [1, 1], cls: 'B' },
  { name: 'B2', tex: 'B_2', x: [0, 2], cls: 'B' },
  { name: 'New', tex: '\\text{New}', x: [3, 2], cls: 'Q' },
]

const FEATURE_LABELS: [string, string] = ['Strength', 'Body']
const CLASS_NAMES = {
  A: 'Class A',
  B: 'Class B',
}

const GAMMA = 0.25
const DECIMALS = 4
const SHOW_DIAGONAL = true

const COLORS = {
  classA:     'var(--c-orange, #c2410c)',
  classB:     'var(--c-blue, #0284c7)',
  query:      'var(--c-dark, #0f172a)',
  similar:    'var(--c-green, #22c55e)',
  dissimilar: 'var(--c-red, #ef4444)',
  surface:    'var(--c-light, #f8fafc)',
  text:       'var(--c-dark, #0f172a)',
  lines:      'rgba(15, 23, 42, 0.13)',
}

const TINT = {
  neutralAt: 0.35,
  maxStrength: 0.24,
  curve: 0.8,
}

const STYLE = {
  fontSize: '0.95rem',
  headerWeight: 700,
  cellPadding: '0.45rem 1rem',
  radius: '0.85rem',
  tableRadius: '0.5rem',
}

/* ───────────────────────────── 2. MACHINERY ───────────────────────────── */

function tex(expr: string, displayMode = false): string {
  return katex.renderToString(expr, {
    throwOnError: false,
    displayMode,
  })
}

function pointTex(p: DataPoint): string {
  if (p.tex) return p.tex
  const match = p.name.match(/^([A-Za-z]+)(\d+)$/)
  if (match) return `${match[1]}_{${match[2]}}`
  return `\\text{${p.name}}`
}

const kernelFormulaTex = computed(() =>
  tex('k(\\mathbf{x}, \\mathbf{z}) = \\exp\\!\\left(-\\gamma \\,\\|\\mathbf{x} - \\mathbf{z}\\|^2\\right)'),
)

const gammaTex = computed(() => tex(`\\gamma = ${GAMMA}`))

const activePoints = computed(() =>
  props.showQuery ? POINTS : POINTS.filter((p) => p.cls !== 'Q'),
)

function sqDist2D(a: [number, number], b: [number, number]): number {
  const dx = a[0] - b[0]
  const dy = a[1] - b[1]
  return dx * dx + dy * dy
}

const kernel = (a: [number, number], b: [number, number]) =>
  Math.exp(-GAMMA * sqDist2D(a, b))

function classColor(cls: DataPoint['cls']): string {
  if (cls === 'A') return COLORS.classA
  if (cls === 'B') return COLORS.classB
  return COLORS.query
}

function tint(score: number): string {
  const n = TINT.neutralAt
  const up = score >= n
  const ratio = up ? (score - n) / (1 - n) : (n - score) / n
  const alpha = TINT.maxStrength * Math.pow(Math.max(0, Math.min(1, ratio)), TINT.curve)
  return `color-mix(in srgb, ${up ? COLORS.similar : COLORS.dissimilar} ${(alpha * 100).toFixed(1)}%, transparent)`
}

const rows = computed(() => {
  const pts = activePoints.value
  return pts.map((p, i) => ({
    ...p,
    labelHtml: tex(pointTex(p)),
    cells: pts.map((q, j) => {
      const involvesQuery = p.cls === 'Q' || q.cls === 'Q'
      const shown = props.full || involvesQuery || j > i || (SHOW_DIAGONAL && j === i)
      const val = kernel(p.x, q.x)
      return {
        shown,
        text: val.toFixed(DECIMALS),
        background: shown ? tint(val) : 'transparent',
      }
    }),
  }))
})

const queryClassification = computed(() => {
  const queryPt = activePoints.value.find((p) => p.cls === 'Q')
  if (!queryPt) return null

  const labeled = activePoints.value.filter((p) => p.cls === 'A' || p.cls === 'B')
  const scoreA = labeled
    .filter((p) => p.cls === 'A')
    .reduce((acc, p) => acc + kernel(queryPt.x, p.x), 0)
  const scoreB = labeled
    .filter((p) => p.cls === 'B')
    .reduce((acc, p) => acc + kernel(queryPt.x, p.x), 0)

  const predicted: 'A' | 'B' = scoreA >= scoreB ? 'A' : 'B'

  return {
    name: queryPt.name,
    labelHtml: tex(pointTex(queryPt)),
    coords: queryPt.x,
    scoreA,
    scoreB,
    predicted,
    predictedLabel: CLASS_NAMES[predicted],
  }
})

// Uniform aspect-ratio-preserving scaling inside available stage bounds
const stageRef = ref<HTMLElement | null>(null)
const cardRef = ref<HTMLElement | null>(null)
const naturalSize = ref({ width: 0, height: 0 })
const scale = ref(1)

let resizeObserver: ResizeObserver | null = null

function measureAndFit() {
  const stage = stageRef.value
  const card = cardRef.value
  if (!stage || !card) return

  const prevTransform = card.style.transform
  card.style.transform = 'none'
  const natW = card.offsetWidth
  const natH = card.offsetHeight
  card.style.transform = prevTransform

  if (!natW || !natH) return
  naturalSize.value = { width: natW, height: natH }

  const availW = stage.clientWidth
  const availH = stage.clientHeight
  if (!availW || !availH) return

  scale.value = Math.min(availW / natW, availH / natH)
}

watch(() => [props.full, props.showQuery], async () => {
  await nextTick()
  measureAndFit()
})

onMounted(async () => {
  await nextTick()
  measureAndFit()
  if (typeof ResizeObserver !== 'undefined') {
    resizeObserver = new ResizeObserver(() => measureAndFit())
    if (stageRef.value) resizeObserver.observe(stageRef.value)
    if (cardRef.value) resizeObserver.observe(cardRef.value)
  }
})

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
})

const scaledBoxStyle = computed(() => ({
  width: naturalSize.value.width ? `${naturalSize.value.width * scale.value}px` : 'auto',
  height: naturalSize.value.height ? `${naturalSize.value.height * scale.value}px` : 'auto',
}))

const cardTransformStyle = computed(() => ({
  transform: `scale(${scale.value})`,
  transformOrigin: 'top left',
}))

const cssVars = {
  '--card-bg': COLORS.surface,
  '--text': COLORS.text,
  '--lines': COLORS.lines,
  '--font': STYLE.fontSize,
  '--pad': STYLE.cellPadding,
  '--radius': STYLE.radius,
  '--table-radius': STYLE.tableRadius,
  '--head-weight': String(STYLE.headerWeight),
}
</script>

<template>
  <div ref="stageRef" class="wrap" :style="cssVars">
    <div class="scaled-box" :style="scaledBoxStyle">
      <div ref="cardRef" class="card" :style="cardTransformStyle">
        <div class="card-header">
          <div class="title-group">
            <span class="badge">2D RBF Similarity</span>
            <h3 class="formula" v-html="kernelFormulaTex"></h3>
          </div>
          <span class="legend">
            <i :style="{ color: COLORS.classA }">{{ CLASS_NAMES.A }}</i>
            ·
            <i :style="{ color: COLORS.classB }">{{ CLASS_NAMES.B }}</i>
          </span>
        </div>

        <div class="table-shell">
          <table>
            <thead>
              <tr>
                <th class="corner">Point</th>
                <th
                  v-for="r in rows"
                  :key="r.name"
                  :style="{ color: classColor(r.cls) }"
                >
                  <div class="pt-head">
                    <span class="pt-tex" v-html="r.labelHtml"></span>
                    <small class="coords">({{ r.x[0] }}, {{ r.x[1] }})</small>
                  </div>
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="r in rows" :key="r.name">
                <th class="side" :style="{ color: classColor(r.cls) }">
                  <div class="pt-head">
                    <span class="pt-tex" v-html="r.labelHtml"></span>
                    <small class="coords">({{ r.x[0] }}, {{ r.x[1] }})</small>
                  </div>
                </th>
                <td
                  v-for="(c, j) in r.cells"
                  :key="j"
                  :class="{ empty: !c.shown }"
                  :style="{ background: c.background }"
                >
                  {{ c.shown ? c.text : '' }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="footer-bar">
          <div v-if="queryClassification" class="pred-inline">
            <span>
              <i :style="{ color: COLORS.classA }">{{ CLASS_NAMES.A }}</i>:
              <strong>{{ queryClassification.scoreA.toFixed(DECIMALS) }}</strong>
            </span>
            <span>
              <i :style="{ color: COLORS.classB }">{{ CLASS_NAMES.B }}</i>:
              <strong>{{ queryClassification.scoreB.toFixed(DECIMALS) }}</strong>
            </span>
            <span class="verdict-wrap">
              <span class="pt-tex" v-html="queryClassification.labelHtml"></span>
              <span
                class="verdict-pill"
                :style="{
                  color: classColor(queryClassification.predicted),
                  background: `color-mix(in srgb, ${classColor(queryClassification.predicted)} 16%, transparent)`,
                }"
              >
                → {{ queryClassification.predictedLabel }}
              </span>
            </span>
          </div>

          <div class="meta-inline">
            <span class="gamma-tex" v-html="gammaTex"></span>
            <!-- <small>({{ props.full ? 'full' : 'upper half' }} · {{ DECIMALS }} dec)</small> -->
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.wrap {
  width: 100%;
  height: 100%;
  min-width: 0;
  min-height: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  box-sizing: border-box;
}

.scaled-box {
  position: relative;
  flex: none;
}

.card {
  width: max-content;
  background: var(--card-bg);
  color: var(--text);
  border-radius: var(--radius);
  padding: 0.85rem 1.15rem;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.07);
  border: 1px solid rgba(0, 0, 0, 0.07);
  box-sizing: border-box;
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.title-group {
  display: flex;
  align-items: center;
  gap: 0.55rem;
  white-space: nowrap;
}

.badge {
  font-size: 0.62rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  padding: 0.14rem 0.48rem;
  border-radius: 999px;
  background: color-mix(in srgb, var(--c-green, #22c55e) 20%, transparent);
  color: #15803d;
}

.formula {
  margin: 0;
  font-size: 0.98rem;
  line-height: 1.25;
}

.formula :deep(.katex) {
  font-size: 1em;
}

.legend {
  font-size: 0.82rem;
  font-weight: 600;
  white-space: nowrap;
}

.legend i {
  font-style: normal;
}

.table-shell {
  border: 1px solid var(--lines);
  border-radius: var(--table-radius);
  overflow: hidden;
  background: rgba(0, 0, 0, 0.015);
}

table {
  border-collapse: collapse;
  font-size: var(--font);
  color: var(--text);
  font-variant-numeric: tabular-nums;
}

th,
td {
  padding: var(--pad);
  text-align: center;
  border: 1px solid var(--lines);
}

th {
  font-weight: var(--head-weight);
  background: rgba(0, 0, 0, 0.028);
}

.pt-head {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.1rem;
  line-height: 1.15;
}

.pt-tex :deep(.katex) {
  font-size: 0.98em;
}

.coords {
  font-size: 0.72em;
  font-weight: 500;
  opacity: 0.72;
}

.corner {
  font-size: 0.82em;
  opacity: 0.75;
}

tr > :first-child {
  border-left: 0;
}

tr > :last-child {
  border-right: 0;
}

thead tr:first-child > * {
  border-top: 0;
}

tbody tr:last-child > * {
  border-bottom: 0;
}

td.empty {
  background: transparent;
}

.footer-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1.2rem;
  padding: 0.42rem 0.68rem;
  border-radius: 0.5rem;
  background: rgba(0, 0, 0, 0.04);
  font-size: 0.82rem;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.pred-inline {
  display: flex;
  align-items: center;
  gap: 0.9rem;
}

.pred-inline i {
  font-style: normal;
  font-weight: 600;
}

.verdict-wrap {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
}

.verdict-pill {
  font-weight: 700;
  font-size: 0.92em;
  padding: 0.12rem 0.5rem;
  border-radius: 999px;
}

.meta-inline {
  display: inline-flex;
  align-items: baseline;
  gap: 0.35rem;
  margin-left: auto;
  font-weight: 600;
}

.gamma-tex :deep(.katex) {
  font-size: 0.98em;
}

.meta-inline small {
  opacity: 0.68;
  font-size: 0.84em;
  font-weight: 500;
}
</style>