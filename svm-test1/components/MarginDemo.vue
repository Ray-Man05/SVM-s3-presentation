<script setup lang="ts">
import { ref, computed, watch, useSlots, unref, onBeforeUnmount } from 'vue'
import { useNav, useSlideContext } from '@slidev/client'

/* ════════════════════════════════════════════════════════════════════════════
   MarginDemo: why we need the soft margin.   Everything you may want to change is in
   section 1. Sections 2-8 are the machinery and should not need editing.

   The story (4 stages, 0-3):
     0  training data, which contains one OUTLIER (an orange point deep in blue territory)
     1  the fitted line appears and its margin band opens up
     2  TEST data (hollow dots) is overlaid
     3  the misclassified points are highlighted
   Hard margin: the outlier drags the line, and many test points are wrong.
   Soft margin: the outlier is allowed to be wrong, the line stays sensible, no test mistakes.

   Use it animated (with controls), or as STATIC figures, e.g. side by side:
     <div class="grid grid-cols-2 gap-6">
       <MarginDemo static margin="hard" caption="Hard margin" show-count />
       <MarginDemo static margin="soft" caption="Soft margin" show-count />
     </div>
   ════════════════════════════════════════════════════════════════════════════ */

/* ───────────────────────────── 1. SETTINGS ───────────────────────────── */

// 1a. PROPS, set on the tag in slides.md
const props = withDefaults(defineProps<{
  margin?: 'hard' | 'soft'    // which version to show (animated mode can switch with the on-screen toggle)
  static?: boolean            // true: a still figure. No animation, no controls, no info box
  stage?: number              // static only: 0 training data · 1 + fitted line · 2 + test data · 3 + mistakes highlighted
  highlightErrors?: boolean   // ring the misclassified points (the outlier in training, wrong test points)
  markOutlier?: boolean       // dashed ring + label on the outlier
  caption?: string            // static: text in the top-left corner of the figure, e.g. "Hard margin"
  showCount?: boolean         // add "· N mistakes" to the caption (counts test points once they are shown, else training points)
  nav?: boolean               // animated: previous / next slide arrows
  clicks?: boolean            // animated: Space / arrow keys drive the stages (one click = one stage)
  speed?: number              // animated: ms one stage takes
  hold?: number               // animated: Play pauses this long on each stage (ms)
}>(), { margin: 'hard', static: false, stage: 3, highlightErrors: true, markOutlier: true, caption: '', showCount: false,
        nav: true, clicks: false, speed: 650, hold: 800 })

// 1b. DATA. Blue is class +1 (right), orange is class −1 (left). Both clusters sit at the same height,
//     so the ideal separating line is vertical.
const TRAIN_SEED = 118, TEST_SEED = 1001                // random scatter of the training / test points
const BLUE = { x: 1.4, y: 0 }, ORANGE = { x: -1.4, y: 0 }   // cluster centres
const SPREAD = { x: 0.5, y: 0.85 }                     // how spread out each cluster is: narrow across, tall up and down
const KEEP_WITHIN = { x: 3.8, y: 2.2 }                 // a point that would land outside ±these is re-drawn
const TRAIN_PER_CLASS = 14, TEST_PER_CLASS = 12
const TRAIN_GAP = 0.35, TEST_GAP = 0.30                // empty corridor kept around the ideal boundary (larger = easier data)
const OUTLIER = { x: 1.0, y: 0.2 }                       // an ORANGE training point in the middle of the blue group
//   If you change the data or the outlier, check the result: the outlier must still leave the training set
//   separable by a straight line (otherwise there is no hard margin), and it should still cause test mistakes.

// 1c. HOW SOFT IS "SOFT": the SVM's C. Large C punishes training mistakes hard, small C tolerates them.
const C_HARD = 10000    // effectively "no mistakes allowed"
const C_SOFT = 0.5      // lower = softer (and a wider margin)

// 1d. COLOURS: any CSS colour ('#f80', 'tomato', 'var(--c-red)' …)
const COLORS = {
  blueClass:     'var(--c-blue)',        // blue points and the +1 margin line
  orangeClass:   'var(--c-orange)',      // orange points and the −1 margin line
  line:          'var(--c-axis, var(--c-light, #f8fafc))',  // the decision line
  band:          'var(--c-blue-highlight)',   // shaded margin band
  ringWrong:     'var(--c-red)',              // ring around misclassified points
  ringWrongFill: 'var(--c-red-highlight)',    // soft fill behind those rings
  outlierMark:   'var(--c-axis, var(--c-light, #f8fafc))',  // dashed ring and label on the outlier
  axes:          'var(--c-axis, var(--c-light, #f8fafc))',  // axes, grid, outlines, caption, legend
  controls:      'var(--c-axis, var(--c-light, #f8fafc))',  // play button, slider, switches, arrows
  infoBackground:'var(--c-light)',             // info box background
  infoText:      'var(--c-dark)',              // info box text
}

// 1e. SIZES AND STYLE
const VIEW = {
  width: 640, height: 400,   // internal drawing size (only the ratio matters; the plot scales to fit)
  pixelsPerUnit: 78,         // zoom
  showGrid: true, gridHalfWidth: 4, gridHalfHeight: 2, showAxes: true,
  showLegend: true,          // "● training  ○ test" key, shown once test data is visible
}
const STYLE = {
  pointRadius: 6.5, pointOutline: 1.2,
  testRadius: 6.5, testOutline: 2.4, testFillOpacity: 0.25,   // test points are hollow-ish so they differ from training points
  ringRadius: 13, ringWidth: 3, ringPulseSeconds: 1.2,
  outlierPointRadius: 8.5, outlierRadius: 16, outlierRingWidth: 2.2,       // the outlier is drawn bigger, inside a dashed ring
  outlierLabel: 'outlier', outlierLabelSize: 19,                          // set outlierLabel to '' for no text
  outlierLabelHalo: 'var(--c-dark)',                                      // outline behind the label so it reads over dots: match your slide background, or '' for none
  captionSize: 20, legendSize: 13,
  decisionLineWidth: 3.5, marginLineWidth: 2, marginDash: '6 5',
  axisWidth: 1.6, axisOpacity: 0.6, gridWidth: 1, gridOpacity: 0.12, bandOpacity: 0.16,
  playButtonSize: '2.1rem', sliderTrack: '4px', sliderThumb: '16px', slideArrowOpacity: 0.45,
}
const STAGE_LABELS = ['Data', 'Fit', 'Test', 'Mistakes']   // names under the slider (4 stages: 0-3)

// 1f. PAGE LAYOUT (animated mode)
const LAYOUT = {
  plotShare: 3, panelShare: 2,   // width ratio between the plot and the controls + info box
  infoMinHeight: '14rem', infoFontSize: '1rem', infoPadding: '1rem 1.3rem', infoRadius: '1rem',
}

/* ───────────────────────────── 2. TYPES & HELPERS ───────────────────────────── */

type Pt = { x: number; y: number; c: 1 | -1 }            // c: class, +1 blue, −1 orange
type Line = { phi: number; d: number; m: number }
//  A line: unit normal at angle `phi` (pointing toward blue), distance `d` from the origin, margin half-width `m`.
//  Decision line: n·p = d.   Margin edges: n·p = d ± m.

const K = 3                                               // last stage
const clamp01 = (x: number) => Math.min(1, Math.max(0, x))
const lerp = (a: number, b: number, t: number) => a + (b - a) * t
const smooth = (x: number) => { x = clamp01(x); return x * x * (3 - 2 * x) }
const win = (p: number, a: number, b: number) => smooth((p - a) / (b - a))      // eased progress inside [a, b]
const easeInOut = (u: number) => (u < 0.5 ? 2 * u * u : 1 - (-2 * u + 2) ** 2 / 2)
const outBack = (x: number) => { x = clamp01(x); const c1 = 1.70158, c3 = c1 + 1; return 1 + c3 * (x - 1) ** 3 + c1 * (x - 1) ** 2 }
const sleep = (ms: number) => new Promise<void>(r => setTimeout(r, ms))
const range = (a: number, b: number) => Array.from({ length: b - a + 1 }, (_, k) => a + k)
const lerpLine = (a: Line, b: Line, t: number): Line => ({ phi: lerp(a.phi, b.phi, t), d: lerp(a.d, b.d, t), m: lerp(a.m, b.m, t) })

/* ───────────────────────────── 3. DATA ───────────────────────────── */

function makeRng(seed: number) {   // small deterministic random generator: same seed, same points
  return () => { seed |= 0; seed = (seed + 0x6d2b79f5) | 0; let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296 }
}
// the ideal boundary: perpendicular to the blue→orange direction, through the midpoint of the cluster centres
const idealNormal = (() => { const dx = BLUE.x - ORANGE.x, dy = BLUE.y - ORANGE.y, l = Math.hypot(dx, dy) || 1; return { x: dx / l, y: dy / l } })()
const idealMid = { x: (BLUE.x + ORANGE.x) / 2, y: (BLUE.y + ORANGE.y) / 2 }

function makeCluster(seed: number, perClass: number, gap: number): Pt[] {
  const rand = makeRng(seed)
  const gauss = () => Math.sqrt(-2 * Math.log(1 - rand())) * Math.cos(2 * Math.PI * rand())
  const out: Pt[] = []
  for (const [c, cl] of [[1, BLUE], [-1, ORANGE]] as const) for (let n = 0; n < perClass; n++) {
    let x = 0, y = 0, tries = 0
    do { x = cl.x + SPREAD.x * gauss(); y = cl.y + SPREAD.y * gauss(); tries++ }       // redraw until allowed (max 200 tries)
    while ((c * ((x - idealMid.x) * idealNormal.x + (y - idealMid.y) * idealNormal.y) < gap || Math.abs(x) > KEEP_WITHIN.x || Math.abs(y) > KEEP_WITHIN.y) && tries < 200)
    out.push({ x, y, c })
  }
  return out
}
const trainPts: Pt[] = [...makeCluster(TRAIN_SEED, TRAIN_PER_CLASS, TRAIN_GAP), { x: OUTLIER.x, y: OUTLIER.y, c: -1 }]
const testPts: Pt[] = makeCluster(TEST_SEED, TEST_PER_CLASS, TEST_GAP)
const outlierIndex = trainPts.length - 1
const rngDelay = makeRng(99)
const trainDelay = trainPts.map(() => rngDelay()), testDelay = testPts.map(() => rngDelay())   // stagger for the appear animations

/* ───────────────────────────── 4. SVM SOLVER ───────────────────────────── */
// Linear SVM by dual coordinate descent, with soft-margin parameter C.

function trainSvm(pts: Pt[], C: number): Line {
  const BIAS = 5, X = pts.map(p => [p.x, p.y, BIAS]), Y = pts.map(p => p.c)
  const Q = X.map(x => x[0] ** 2 + x[1] ** 2 + x[2] ** 2), a = X.map(() => 0), w = [0, 0, 0]
  for (let sweep = 0; sweep < 4000; sweep++) for (let i = 0; i < X.length; i++) {
    const G = Y[i] * (w[0] * X[i][0] + w[1] * X[i][1] + w[2] * X[i][2]) - 1
    const next = Math.min(Math.max(a[i] - G / Q[i], 0), C), delta = next - a[i]
    if (delta) { for (let j = 0; j < 3; j++) w[j] += delta * Y[i] * X[i][j]; a[i] = next }
  }
  const norm = Math.hypot(w[0], w[1]), b = w[2] * BIAS
  return { phi: Math.atan2(w[1], w[0]), d: -b / norm, m: 1 / norm }
}

/* ───────────────────────────── 5. THE TWO FITS ───────────────────────────── */

const fits: Record<'hard' | 'soft', Line> = { hard: trainSvm(trainPts, C_HARD), soft: trainSvm(trainPts, C_SOFT) }
const wrongSide = (L: Line, p: Pt) => p.c * (Math.cos(L.phi) * p.x + Math.sin(L.phi) * p.y - L.d) <= 0
const mistakes = (L: Line, pts: Pt[]) => pts.map((_, i) => i).filter(i => wrongSide(L, pts[i]))
const results = {
  hard: { train: mistakes(fits.hard, trainPts), test: mistakes(fits.hard, testPts) },
  soft: { train: mistakes(fits.soft, trainPts), test: mistakes(fits.soft, testPts) },
}

/* ───────────────────────────── 6. PLAYBACK ───────────────────────────── */
// pos: 0…3 is the stage (fractions are in between). blend: 0 = hard margin, 1 = soft margin (animated when switched).
// intro: 0…1, the training points dropping in.

const pos = ref(props.static ? Math.min(K, Math.max(0, props.stage)) : 0)
const blend = ref(props.margin === 'soft' ? 1 : 0)
const intro = ref(props.static ? 1 : 0)
const playing = ref(false)
let epoch = 0, playRun = 0, raf = 0, blendRaf = 0, introRaf = 0, dead = false

const shown = computed(() => Math.round(pos.value))
const mode = computed<'hard' | 'soft'>(() => (blend.value >= 0.5 ? 'soft' : 'hard'))

function tweenRef(target: typeof pos, to: number, ms: number, ease = easeInOut, rafKey?: 'blend' | 'intro'): Promise<void> {
  const from = target.value
  return new Promise<void>(done => {
    const t0 = performance.now()
    const tick = () => {
      if (dead) return done()
      const u = Math.min((performance.now() - t0) / ms, 1)
      target.value = lerp(from, to, ease(u))
      if (u < 1) { const id = requestAnimationFrame(tick); if (rafKey === 'blend') blendRaf = id; else if (rafKey === 'intro') introRaf = id } else done()
    }
    requestAnimationFrame(tick)
  })
}
function animateTo(target: number): Promise<void> {          // glide pos to `target`; a new glide cancels the old one
  const myEpoch = ++epoch, from = pos.value, dist = Math.abs(target - from)
  if (dist < 1e-4) { pos.value = target; return Promise.resolve() }
  const duration = props.speed * Math.min(Math.max(dist, 0.5), 1.5), t0 = performance.now()
  return new Promise<void>(done => {
    const tick = () => {
      if (myEpoch !== epoch || dead) return done()
      const u = Math.min((performance.now() - t0) / duration, 1)
      pos.value = lerp(from, target, easeInOut(u))
      u < 1 ? (raf = requestAnimationFrame(tick)) : done()
    }
    raf = requestAnimationFrame(tick)
  })
}
function stop() { playing.value = false; playRun++; epoch++ }
function jump(stage: number) { stop(); animateTo(Math.max(0, Math.min(K, stage))) }
async function togglePlay() {
  if (playing.value) { stop(); return }
  const run = ++playRun
  playing.value = true
  if (pos.value > K - 0.01) { pos.value = 0; await sleep(300) }
  while (playing.value && run === playRun && !dead && pos.value < K - 0.01) {
    await animateTo(Math.floor(pos.value + 1e-3) + 1)
    if (run !== playRun) return
    await sleep(props.hold)
  }
  if (run === playRun) playing.value = false
}
function onScrub(e: Event) {
  stop()
  const v = Number((e.target as HTMLInputElement).value), whole = Math.round(v)
  pos.value = Math.abs(v - whole) < 0.08 ? whole : v
}
function onRelease(e: Event) { animateTo(Math.round(pos.value)); (e.target as HTMLElement).blur() }
const dropFocus = (e: Event) => (e.currentTarget as HTMLElement).blur()   // so Space keeps meaning "next slide"

function setMode(m: 'hard' | 'soft') { if (!props.static) tweenRef(blend, m === 'soft' ? 1 : 0, 700, easeInOut, 'blend') }
function replay() { stop(); pos.value = 0; intro.value = 0; tweenRef(intro, 1, 1600, easeInOut, 'intro') }
watch(() => props.margin, m => { if (props.static) blend.value = m === 'soft' ? 1 : 0; else setMode(m) })
watch(() => props.stage, s => { if (props.static) pos.value = Math.min(K, Math.max(0, s)) })

const { nextSlide, prevSlide } = useNav()
if (!props.static) tweenRef(intro, 1, 1600, easeInOut, 'intro')
onBeforeUnmount(() => { dead = true; stop(); cancelAnimationFrame(raf); cancelAnimationFrame(blendRaf); cancelAnimationFrame(introRaf) })
if (props.clicks && !props.static) {                         // optional: Slidev clicks drive the stages
  const ctx: any = useSlideContext()
  watch(() => Number(unref(unref(ctx)?.$clicks) ?? 0), c => jump(c))
}

/* ───────────────────────────── 7. DRAWING ───────────────────────────── */

const sx = (x: number) => VIEW.width / 2 + VIEW.pixelsPerUnit * x
const sy = (y: number) => VIEW.height / 2 - VIEW.pixelsPerUnit * y
const gridX = range(-VIEW.gridHalfWidth, VIEW.gridHalfWidth), gridY = range(-VIEW.gridHalfHeight, VIEW.gridHalfHeight)

// the line on screen: it sits at its fitted position and only its margin band opens up (stage 0→1).
// Hard and soft cross-fade by `blend`, so switching shows how the margin changes the line.
const line = computed<Line>(() => {
  const L = lerpLine(fits.hard, fits.soft, blend.value)
  return { ...L, m: L.m * easeInOut(clamp01(pos.value)) }
})
const lineOpacity = computed(() => clamp01(pos.value / 0.3))
function lineAt(offset: number) {
  const L = line.value, c = Math.cos(L.phi), s = Math.sin(L.phi),
      px = (L.d + offset) * c, py = (L.d + offset) * s,
      far = 7
  return { x1: sx(px + far * s), y1: sy(py - far * c), x2: sx(px - far * s), y2: sy(py + far * c) }
}
const lines = computed(() => ({ mid: lineAt(0), lo: lineAt(-line.value.m), hi: lineAt(line.value.m) }))
const bandPoints = computed(() => { const { lo, hi } = lines.value; return `${lo.x1},${lo.y1} ${lo.x2},${lo.y2} ${hi.x2},${hi.y2} ${hi.x1},${hi.y1}` })

const appear = (delay: number, t: number) => clamp01((t - delay * 0.5) / 0.5)           // per-point staggered 0→1
const trainA = (i: number) => appear(trainDelay[i], intro.value)
const testA = (i: number) => clamp01((pos.value - 1 - testDelay[i] * 0.6) / 0.4)
const trainRingA = computed(() => (props.highlightErrors ? win(pos.value, 0.7, 1.1) : 0))   // training mistakes (the outlier, soft margin)
const testRingA = computed(() => (props.highlightErrors ? win(pos.value, 2, 2.6) : 0))      // test mistakes
// hard / soft rings are drawn together, each weighted by `blend`, so switching cross-fades them
const trainRings = computed(() => (['hard', 'soft'] as const).map(m => ({ m, idx: results[m].train, o: trainRingA.value * (m === 'soft' ? blend.value : 1 - blend.value) })))
const testRings = computed(() => (['hard', 'soft'] as const).map(m => ({ m, idx: results[m].test, o: testRingA.value * (m === 'soft' ? blend.value : 1 - blend.value) })))

const testVisible = computed(() => pos.value > 1.05)
const countText = computed(() => {
  const r = results[mode.value], n = testVisible.value ? r.test.length : r.train.length
  return `${n} mistake${n === 1 ? '' : 's'}`
})
const captionText = computed(() => [props.caption, props.showCount ? countText.value : ''].filter(Boolean).join('  ·  '))

/* ───────────────────────────── 8. INFO BOX (slots) ───────────────────────────── */
// In slides.md:  <template #default="{ step, mode, testErrors, trainErrors, margin }"> … </template>
// or per stage:  <template #step0> … </template> … <template #step3> … </template>   (#stepN wins over #default)

const slots = useSlots()
const hasInfo = !props.static && !!Object.keys(slots).length
const slotName = computed(() => (slots[`step${shown.value}`] ? `step${shown.value}` : 'default'))
const slotProps = computed(() => {
  const r = results[mode.value], L = fits[mode.value]
  return { step: shown.value, steps: K, mode: mode.value, trainErrors: r.train.length, testErrors: r.test.length,
    nTrain: trainPts.length, nTest: testPts.length, margin: +(2 * L.m).toFixed(2) }
})

const cssVars = {
  '--ax': COLORS.controls, '--info-bg': COLORS.infoBackground, '--info-fg': COLORS.infoText,
  '--cols': `${LAYOUT.plotShare}fr ${LAYOUT.panelShare}fr`, '--info-min': LAYOUT.infoMinHeight, '--info-size': LAYOUT.infoFontSize,
  '--info-pad': LAYOUT.infoPadding, '--info-radius': LAYOUT.infoRadius,
  '--ring-w': String(STYLE.ringWidth), '--ring-pulse': `${STYLE.ringPulseSeconds}s`,
  '--play-size': STYLE.playButtonSize, '--track': STYLE.sliderTrack, '--thumb': STYLE.sliderThumb, '--arrow-opacity': String(STYLE.slideArrowOpacity),
}
const fill = computed(() => `${(pos.value / K) * 100}%`)
</script>

<template>
  <div class="wrap" :class="{ still: $props.static }" :style="cssVars">
    <svg class="plot" :viewBox="`0 0 ${VIEW.width} ${VIEW.height}`">
      <g v-if="VIEW.showGrid" :stroke="COLORS.axes" :stroke-width="STYLE.gridWidth" :opacity="STYLE.gridOpacity">
        <line v-for="g in gridX" :key="'gx' + g" :x1="sx(g)" y1="0" :x2="sx(g)" :y2="VIEW.height" />
        <line v-for="g in gridY" :key="'gy' + g" x1="0" :y1="sy(g)" :x2="VIEW.width" :y2="sy(g)" />
      </g>
      <g v-if="VIEW.showAxes" :stroke="COLORS.axes" :stroke-width="STYLE.axisWidth" :opacity="STYLE.axisOpacity">
        <line x1="0" :y1="sy(0)" :x2="VIEW.width" :y2="sy(0)" />
        <line :x1="sx(0)" y1="0" :x2="sx(0)" :y2="VIEW.height" />
      </g>

      <g :opacity="lineOpacity">
        <polygon :points="bandPoints" :fill="COLORS.band" :opacity="STYLE.bandOpacity" />
        <line v-bind="lines.lo" :stroke="COLORS.orangeClass" :stroke-width="STYLE.marginLineWidth" :stroke-dasharray="STYLE.marginDash" />
        <line v-bind="lines.hi" :stroke="COLORS.blueClass" :stroke-width="STYLE.marginLineWidth" :stroke-dasharray="STYLE.marginDash" />
        <line v-bind="lines.mid" :stroke="COLORS.line" :stroke-width="STYLE.decisionLineWidth" stroke-linecap="round" />
      </g>

      <!-- training points -->
      <circle v-for="(p, i) in trainPts" :key="'tr' + i" :cx="sx(p.x)" :cy="sy(p.y)" :r="(i === outlierIndex ? STYLE.outlierPointRadius : STYLE.pointRadius) * outBack(trainA(i))" :opacity="trainA(i)"
              :stroke="COLORS.axes" :stroke-width="i === outlierIndex ? 2.4 : STYLE.pointOutline" :fill="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass" />

      <!-- test points: hollow-ish, appear in stage 2 -->
      <template v-if="pos > 1">
        <circle v-for="(p, i) in testPts" :key="'te' + i" :cx="sx(p.x)" :cy="sy(p.y)" :r="STYLE.testRadius * outBack(testA(i))" :opacity="testA(i)"
                :stroke="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass" :stroke-width="STYLE.testOutline"
                :fill="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass" :fill-opacity="STYLE.testFillOpacity" />
      </template>

      <!-- mistake rings -->
      <template v-for="r in trainRings" :key="'trr' + r.m">
        <circle v-for="i in r.idx" :key="'trr' + r.m + i" class="ring" :opacity="r.o" :cx="sx(trainPts[i].x)" :cy="sy(trainPts[i].y)" :r="STYLE.ringRadius"
                :stroke="COLORS.ringWrong" :fill="COLORS.ringWrongFill" fill-opacity="0.3" />
      </template>
      <template v-for="r in testRings" :key="'ter' + r.m">
        <circle v-for="i in r.idx" :key="'ter' + r.m + i" class="ring" :opacity="r.o" :cx="sx(testPts[i].x)" :cy="sy(testPts[i].y)" :r="STYLE.ringRadius"
                :stroke="COLORS.ringWrong" :fill="COLORS.ringWrongFill" fill-opacity="0.3" />
      </template>

      <!-- outlier mark -->
      <g v-if="markOutlier" :opacity="trainA(outlierIndex)">
        <circle :cx="sx(OUTLIER.x)" :cy="sy(OUTLIER.y)" :r="STYLE.outlierRadius" fill="none" :stroke="COLORS.outlierMark" :stroke-width="STYLE.outlierRingWidth" stroke-dasharray="4 3" />
        <text v-if="STYLE.outlierLabel" :x="sx(OUTLIER.x) + STYLE.outlierRadius + 4" :y="sy(OUTLIER.y) - STYLE.outlierRadius - 2"
              :font-size="STYLE.outlierLabelSize" :fill="COLORS.outlierMark" font-style="italic" font-weight="600"
              :stroke="STYLE.outlierLabelHalo || 'none'" stroke-width="5" stroke-linejoin="round" paint-order="stroke">{{ STYLE.outlierLabel }}</text>
      </g>

      <text v-if="captionText" x="18" :y="18 + STYLE.captionSize" :font-size="STYLE.captionSize" font-weight="600" :fill="COLORS.axes">{{ captionText }}</text>

      <g v-if="VIEW.showLegend && pos > 1" :opacity="win(pos, 1, 1.5)" :fill="COLORS.axes" :font-size="STYLE.legendSize">
        <circle cx="20" :cy="VIEW.height - 40" r="5.5" :fill="COLORS.axes" :opacity="0.9" />
        <text x="32" :y="VIEW.height - 35">training</text>
        <circle cx="20" :cy="VIEW.height - 20" r="5.5" fill="none" :stroke="COLORS.axes" stroke-width="2" />
        <text x="32" :y="VIEW.height - 15">test</text>
      </g>
    </svg>

    <div v-if="!$props.static" class="panel">
      <div class="controls">
        <div class="row">
          <span class="cap">Margin</span>
          <div class="seg">
            <button :class="{ on: mode === 'hard' }" @click="setMode('hard'); dropFocus($event)">Hard</button>
            <button :class="{ on: mode === 'soft' }" @click="setMode('soft'); dropFocus($event)">Soft</button>
          </div>
          <span class="spacer" />
          <button class="icon" title="Replay from the start" aria-label="Replay" @click="replay(); dropFocus($event)">
            <svg viewBox="0 0 24 24"><path d="M4 12a8 8 0 1 0 2.6-5.9M4 4v4.5h4.5" /></svg>
          </button>
          <div v-if="nav" class="slide-nav">
            <button title="Previous slide" aria-label="Previous slide" @click="prevSlide(); dropFocus($event)"><svg viewBox="0 0 24 24"><path d="M15 5l-7 7 7 7" /></svg></button>
            <button title="Next slide" aria-label="Next slide" @click="nextSlide(); dropFocus($event)"><svg viewBox="0 0 24 24"><path d="M9 5l7 7-7 7" /></svg></button>
          </div>
        </div>

        <div class="row">
          <button class="play" :aria-label="playing ? 'Pause' : 'Play'" :title="playing ? 'Pause' : 'Play'" @click="togglePlay(); dropFocus($event)">
            <svg v-if="!playing" viewBox="0 0 24 24"><path d="M8 5.5v13l11-6.5z" fill="currentColor" /></svg>
            <svg v-else viewBox="0 0 24 24"><path d="M7 5h4v14H7zM13 5h4v14h-4z" fill="currentColor" /></svg>
          </button>
          <div class="scrub">
            <input type="range" min="0" :max="K" step="any" :value="pos" :style="{ '--fill': fill }" aria-label="Stage" @input="onScrub" @change="onRelease" />
            <div class="ticks">
              <button v-for="n in range(0, K)" :key="n" class="tick" :class="{ on: n === shown }"
                      :style="{ left: `calc(var(--thumb) / 2 + (100% - var(--thumb)) * ${n / K})` }" @click="jump(n); dropFocus($event)">{{ STAGE_LABELS[n] ?? n }}</button>
            </div>
          </div>
        </div>
      </div>
      <div v-if="hasInfo" class="info"><slot :name="slotName" v-bind="slotProps" /></div>
    </div>
    <template v-if="clicks && !$props.static"><span v-for="n in K" :key="n" v-click hidden /></template>
  </div>
</template>

<style scoped>
.wrap { display: grid; grid-template-columns: var(--cols); gap: 1.4rem; align-items: center; height: 100%; }
/* .wrap.still { display: block; height: auto; }
 */
.wrap.still {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  width: 100%;
  min-height: 0;
  overflow: hidden;
}

.wrap.still .plot {
  width: 100%;
  height: auto;
  max-height: 100%;
  aspect-ratio: 640 / 400;
  flex: 0 1 auto;
}

.plot {
  width: 100%;
  height: auto;
  display: block;
}


.ring { stroke-width: var(--ring-w); transform-box: fill-box; transform-origin: center; animation: pulse var(--ring-pulse) ease-in-out infinite; }
@keyframes pulse { 0%, 100% { transform: scale(.9); } 50% { transform: scale(1.2); } }

.panel { display: flex; flex-direction: column; gap: .9rem; }
.controls { display: flex; flex-direction: column; gap: .7rem; color: var(--ax); }
.row { display: flex; align-items: flex-start; gap: .8rem; }
.row:first-child { align-items: center; gap: .6rem; }
.cap { flex: none; width: 3.6rem; font-size: .68rem; letter-spacing: .08em; text-transform: uppercase; opacity: .6; }
.spacer { flex: 1; }
.controls button { background: none; border: 0; padding: 0; color: inherit; cursor: pointer; font: inherit; }

.seg { display: inline-flex; border: 2px solid var(--ax); border-radius: 999px; overflow: hidden; }
.seg button { font-size: .82rem; padding: .28rem .95rem; transition: background .15s, color .15s; }
.seg button:hover:not(.on) { background: color-mix(in srgb, var(--ax) 18%, transparent); }
.seg button.on { background: var(--ax); color: var(--c-dark); }

.icon, .slide-nav button { display: grid; place-items: center; width: 1.7rem; height: 1.7rem; }
.icon { opacity: .6; } .icon:hover { opacity: 1; }
.icon svg, .slide-nav svg { width: 1.25rem; height: 1.25rem; fill: none; stroke: currentColor; stroke-width: 2.2; stroke-linecap: round; stroke-linejoin: round; }
.slide-nav { display: flex; opacity: var(--arrow-opacity); transition: opacity .2s; }
.slide-nav:hover { opacity: 1; }

.play { flex: none; width: var(--play-size); height: var(--play-size); border-radius: 50%; border: 2px solid var(--ax) !important; display: grid; place-items: center; }
.play svg { width: 55%; height: 55%; }
.play:hover { background: color-mix(in srgb, var(--ax) 18%, transparent); }
.scrub { position: relative; flex: 1; min-width: 0; height: var(--play-size); display: flex; flex-direction: column; justify-content: center; }
.scrub input { -webkit-appearance: none; appearance: none; width: 100%; height: var(--thumb); margin: 0; background: transparent; cursor: pointer; }
.scrub input::-webkit-slider-runnable-track { height: var(--track); border-radius: 999px;
  background: linear-gradient(to right, var(--ax) var(--fill), color-mix(in srgb, var(--ax) 25%, transparent) var(--fill)); }
.scrub input::-webkit-slider-thumb { -webkit-appearance: none; width: var(--thumb); height: var(--thumb); border-radius: 50%; background: var(--ax); margin-top: calc((var(--track) - var(--thumb)) / 2); border: 0; }
.scrub input::-moz-range-track { height: var(--track); border-radius: 999px; background: color-mix(in srgb, var(--ax) 25%, transparent); }
.scrub input::-moz-range-progress { height: var(--track); border-radius: 999px; background: var(--ax); }
.scrub input::-moz-range-thumb { width: var(--thumb); height: var(--thumb); border-radius: 50%; background: var(--ax); border: 0; }
.ticks { position: relative; height: 1.1rem; margin-top: .1rem; }
.tick { position: absolute; top: 0; transform: translateX(-50%); font-size: .72rem !important; opacity: .5; padding: 0 .25rem !important; white-space: nowrap; }
.tick:first-child { transform: translateX(-20%); } .tick:last-child { transform: translateX(-80%); }
.tick:hover, .tick.on { opacity: 1; }
.tick.on { font-weight: 700; }

.info { background: var(--info-bg); color: var(--info-fg); border-radius: var(--info-radius); padding: var(--info-pad); font-size: var(--info-size);
  line-height: 1.5; min-height: var(--info-min); display: flex; flex-direction: column; justify-content: center; }
.info :deep(p) { margin: .4em 0; }
.info :deep(.katex-display) { margin: .6em 0; }
.info :deep(.katex) { font-size: 1.1em; }
</style>