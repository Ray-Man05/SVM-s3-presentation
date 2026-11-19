<script setup lang="ts">
import { ref, computed, useSlots, watch, unref, onBeforeUnmount } from 'vue'
import { useNav, useSlideContext } from '@slidev/client'

/* ════════════════════════════════════════════════════════════════════════════
   SvmSteps: step-by-step SVM demo.   Everything you may want to change is in
   section 1. Sections 2-8 are the machinery and should not need editing.

   Coordinates: data units, origin at the centre of the plot, x to the right,
   y UP. With the default VIEW the visible area is roughly x ∈ [-4.1, 4.1] and
   y ∈ [-2.5, 2.5].

   Controls: play/pause, a slider (drag to scrub the line continuously, it snaps to
   whole steps on release; click a step number to jump), and subtle arrows for the
   previous / next SLIDE of the presentation.
   ════════════════════════════════════════════════════════════════════════════ */

/* ───────────────────────────── 1. SETTINGS ───────────────────────────── */

// 1a. PROPS, set on the tag in slides.md:  <SvmSteps :steps="6" :speed="500">
const props = withDefaults(defineProps<{
  clicks?: boolean   // true: Space / arrow keys drive the steps (one click = one step). Off by default so Space goes straight to the next slide
  nav?: boolean      // show the previous / next slide arrows
  steps?: number     // how many adjustments between the bad start line and the final line
  decay?: number     // step sizes: 1 = all equal, >1 = big moves first and small ones last
  hold?: number      // Play: ms to pause on each step before moving on
  speed?: number     // ms one step takes (jumping several steps at once takes up to 1.5x as long)
}>(), { clicks: false, nav: true, steps: 5, decay: 1.1, hold: 450, speed: 650 })

// 1b. DATA: two clusters of points. Blue is class +1, orange is class −1.
const SEED = 41                                  // changes the random scatter (try other numbers)
const BLUE   = { x:  1.0, y: -0.8, spread: 0.62, count: 14 }   // centre, how spread out, how many points
const ORANGE = { x: 0.0, y:  0.5, spread: 0.62, count: 14 }
//   Move a cluster by changing its x / y (e.g. orange further down: y: 0.2).
//   Bigger `spread` = looser, more overlapping cluster.
const MIN_GAP = 0.1        // empty corridor kept between the classes, so they are always separable. Raise it for easier data.
const KEEP_ON_SCREEN = true   // true: points that would fall outside the plot are re-drawn

// 1c. STARTING LINE: the bad guess, defined relative to the final (optimal) line.
const START_LINE = {
  rotateDeg: -48,   // degrees to turn it away from the optimum (sign = direction). 0 = same angle
  shift: 0.5,       // distance (data units) to slide it sideways (sign = which side). 0 = same position
  widen: 0.5,       // margin width relative to the final one: <1 starts narrower, >1 starts wider
}
//   After moving clusters, the number of initial errors changes: adjust rotateDeg / shift until it reads well.

// 1d. WHICH POINTS GET A RING: points whose "score" is at most this (1 = exactly on the margin edge;
//     points inside the margin or on the wrong side score lower). Slightly above 1 hides rounding noise.
const RING_THRESHOLD = 1.01

// 1e. COLOURS: any CSS colour ('#f80', 'tomato', 'var(--c-red)' …)
const COLORS = {
  blueClass:     'var(--c-blue)',        // blue points and the +1 margin line
  orangeClass:   'var(--c-orange)',      // orange points and the −1 margin line
  line:          'var(--c-axis, var(--c-light, #f8fafc))',  // the decision line while training
  lineFinal:     'var(--c-green)',       // the decision line once it has converged
  band:          'var(--c-blue-highlight)',   // shaded margin band
  ringNear:      'var(--c-blue-highlight)',   // ring: touches / sits inside the margin
  ringWrong:     'var(--c-red)',              // ring: point on the wrong side
  ringWrongFill: 'var(--c-red-highlight)',    // soft fill behind wrong-side rings
  axes:          'var(--c-axis, var(--c-light, #f8fafc))',  // axes, grid, point outlines
  controls:      'var(--c-axis, var(--c-light, #f8fafc))',  // play button, slider, step numbers, slide arrows
  infoBackground:'var(--c-light)',             // info box background
  infoText:      'var(--c-dark)',              // info box text
}

// 1f. SIZES AND STYLE
const VIEW = {
  width: 640, height: 400,   // internal drawing size (only the ratio matters; the plot scales to fit)
  pixelsPerUnit: 78,         // zoom: bigger = points further apart, smaller visible area
  showGrid: true, gridHalfWidth: 4, gridHalfHeight: 2,   // grid lines at whole units, this many each side of the origin
  showAxes: true,
}
const STYLE = {
  pointRadius: 6.5, pointOutline: 1.2,
  ringRadius: 13, ringWidth: 3, ringPulseSeconds: 1.2, ringFadeSeconds: 0.15,   // pulsing rings
  decisionLineWidth: 3.5, marginLineWidth: 2, marginDash: '6 5',
  axisWidth: 1.6, axisOpacity: 0.6, gridWidth: 1, gridOpacity: 0.12, bandOpacity: 0.16,
  playButtonSize: '2.1rem', sliderTrack: '4px', sliderThumb: '16px', slideArrowOpacity: 0.45,
}

// 1g. PAGE LAYOUT
const LAYOUT = {
  plotShare: 3, panelShare: 2,   // width ratio between the plot (left) and the controls + info box (right)
  infoMinHeight: '14rem', infoFontSize: '1rem', infoPadding: '1rem 1.3rem', infoRadius: '1rem',
}

/* ───────────────────────────── 2. TYPES & HELPERS ───────────────────────────── */

type Pt = { x: number; y: number; c: 1 | -1 }            // c: class, +1 blue, −1 orange
type Line = { phi: number; d: number; m: number }
//  A line is stored as: unit normal at angle `phi` (pointing toward blue), distance `d` from the origin,
//  and margin half-width `m`.  Decision line: n·p = d.   Margin edges: n·p = d ± m.

const lerp = (a: number, b: number, t: number) => a + (b - a) * t
const easeInOut = (u: number) => (u < 0.5 ? 2 * u * u : 1 - (-2 * u + 2) ** 2 / 2)   // quadratic: snappier than cubic
const sleep = (ms: number) => new Promise<void>(r => setTimeout(r, ms))
const range = (a: number, b: number) => Array.from({ length: b - a + 1 }, (_, k) => a + k)

/* ───────────────────────────── 3. DATA ───────────────────────────── */

function makeRng(seed: number) {   // small deterministic random generator: same seed, same points
  return () => { seed |= 0; seed = (seed + 0x6d2b79f5) | 0; let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t; return ((t ^ (t >>> 14)) >>> 0) / 4294967296 }
}

function generatePoints(): Pt[] {
  const rand = makeRng(SEED)
  const gauss = () => Math.sqrt(-2 * Math.log(1 - rand())) * Math.cos(2 * Math.PI * rand())
  // the corridor lies between the clusters: perpendicular to the blue→orange direction, through their midpoint
  const dx = BLUE.x - ORANGE.x, dy = BLUE.y - ORANGE.y, len = Math.hypot(dx, dy) || 1
  const ux = dx / len, uy = dy / len, mx = (BLUE.x + ORANGE.x) / 2, my = (BLUE.y + ORANGE.y) / 2
  const maxX = VIEW.width / 2 / VIEW.pixelsPerUnit - 0.3, maxY = VIEW.height / 2 / VIEW.pixelsPerUnit - 0.3
  const out: Pt[] = []
  for (const [c, cl] of [[1, BLUE], [-1, ORANGE]] as const) for (let n = 0; n < cl.count; n++) {
    let x = 0, y = 0, tries = 0
    while (true) {                                       // redraw until the point is allowed (give up after 200 tries)
      x = cl.x + cl.spread * gauss(); y = cl.y + cl.spread * gauss(); tries++
      const side = c * ((x - mx) * ux + (y - my) * uy)   // distance into this class's half
      const offScreen = KEEP_ON_SCREEN && (Math.abs(x) > maxX || Math.abs(y) > maxY)
      if ((side >= MIN_GAP && !offScreen) || tries >= 200) {
        if (side < MIN_GAP) { x += (MIN_GAP - side) * c * ux; y += (MIN_GAP - side) * c * uy }   // last resort: push into place
        break
      }
    }
    out.push({ x, y, c })
  }
  return out
}
const pts = generatePoints()

/* ───────────────────────────── 4. SVM SOLVER ───────────────────────────── */
// Exact linear SVM by dual coordinate descent. Only used to find the FINAL line.

function trainSvm(): Line {
  const BIAS = 5, C = 1000, X = pts.map(p => [p.x, p.y, BIAS]), Y = pts.map(p => p.c)
  const Q = X.map(x => x[0] ** 2 + x[1] ** 2 + x[2] ** 2), a = X.map(() => 0), w = [0, 0, 0]
  for (let sweep = 0; sweep < 3000; sweep++) for (let i = 0; i < X.length; i++) {
    const G = Y[i] * (w[0] * X[i][0] + w[1] * X[i][1] + w[2] * X[i][2]) - 1
    const next = Math.min(Math.max(a[i] - G / Q[i], 0), C), delta = next - a[i]
    if (delta) { for (let j = 0; j < 3; j++) w[j] += delta * Y[i] * X[i][j]; a[i] = next }
  }
  const norm = Math.hypot(w[0], w[1]), b = w[2] * BIAS
  return { phi: Math.atan2(w[1], w[0]), d: -b / norm, m: 1 / norm }
}

/* ───────────────────────────── 5. KEYFRAMES ───────────────────────────── */
// Walk from the bad start line to the optimum in `steps` moves. The final line is exact; the lines in
// between are an illustration of convergence (the rings at every step are computed exactly for that line).

const K = Math.max(1, props.steps)
const finalLine = trainSvm()
const startLine: Line = {
  phi: finalLine.phi + (START_LINE.rotateDeg * Math.PI) / 180,
  d: finalLine.d + START_LINE.shift,
  m: finalLine.m * START_LINE.widen,
}
const keys: Line[] = Array.from({ length: K + 1 }, (_, k) => {
  const remaining = Math.pow(1 - k / K, props.decay)      // 1 at the start → 0 at the end
  return { phi: lerp(finalLine.phi, startLine.phi, remaining), d: lerp(finalLine.d, startLine.d, remaining), m: lerp(finalLine.m, startLine.m, remaining) }
})
keys[K] = finalLine

const score = (L: Line, p: Pt) => (p.c * (Math.cos(L.phi) * p.x + Math.sin(L.phi) * p.y - L.d)) / L.m   // 1 = on margin edge, <0 = wrong side
const keyInfo = keys.map(L => {
  const idx = pts.map((_, i) => i)
  return { ringed: idx.filter(i => score(L, pts[i]) <= RING_THRESHOLD), wrong: idx.filter(i => score(L, pts[i]) <= 0) }
})

/* ───────────────────────────── 6. PLAYBACK ───────────────────────────── */
// One number drives everything: `pos`. 0 = start line, K = final line, 1.5 = halfway between steps 1 and 2.
// Play, the slider, the step numbers and Slidev clicks all just move `pos`.

const pos = ref(0)
const playing = ref(false)
let epoch = 0, playRun = 0, raf = 0, dead = false

const shown = computed(() => Math.round(pos.value))                           // the whole step we are at (or closest to)
const settled = computed(() => Math.abs(pos.value - shown.value) < 0.03)      // sitting on a step, not between two
const converged = computed(() => Math.abs(pos.value - K) < 0.03)
const line = computed<Line>(() => {                                           // the line to draw at the current position
  const k = Math.min(Math.floor(pos.value), K - 1), f = pos.value - k, a = keys[k], b = keys[k + 1]
  return { phi: lerp(a.phi, b.phi, f), d: lerp(a.d, b.d, f), m: lerp(a.m, b.m, f) }
})

function animateTo(target: number): Promise<void> {      // glide pos to `target`; starting a new glide cancels the old one
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
function stop() { playing.value = false; playRun++; epoch++ }                // any manual control stops Play and any glide
function jump(step: number) { stop(); animateTo(Math.max(0, Math.min(K, step))) }

async function togglePlay() {
  if (playing.value) { stop(); return }
  const run = ++playRun
  playing.value = true
  if (pos.value > K - 0.01) { pos.value = 0; await sleep(250) }              // at the end: start over
  while (playing.value && run === playRun && !dead && pos.value < K - 0.01) {
    await animateTo(Math.floor(pos.value + 1e-3) + 1)                        // glide to the next whole step…
    if (run !== playRun) return
    await sleep(props.hold)                                                  // …and rest there
  }
  if (run === playRun) playing.value = false
}

function onScrub(e: Event) {                                                 // slider dragged: follow the thumb, stick to steps
  stop()
  const v = Number((e.target as HTMLInputElement).value), whole = Math.round(v)
  pos.value = Math.abs(v - whole) < 0.08 ? whole : v
}
function onRelease(e: Event) { animateTo(Math.round(pos.value)); (e.target as HTMLElement).blur() }   // let go: snap to the nearest step
const dropFocus = (e: Event) => (e.currentTarget as HTMLElement).blur()      // so Space keeps meaning "next slide", not "press the button again"

const { nextSlide, prevSlide } = useNav()                                    // previous / next slide of the presentation

onBeforeUnmount(() => { dead = true; stop(); cancelAnimationFrame(raf) })

if (props.clicks) {                            // optional: Slidev clicks drive the steps
  const ctx: any = useSlideContext()
  watch(() => Number(unref(unref(ctx)?.$clicks) ?? 0), c => jump(c))
}

/* ───────────────────────────── 7. DRAWING ───────────────────────────── */

const sx = (x: number) => VIEW.width / 2 + VIEW.pixelsPerUnit * x     // data → pixel
const sy = (y: number) => VIEW.height / 2 - VIEW.pixelsPerUnit * y
function lineAt(offset: number) {              // the line n·p = d + offset, as two far-apart pixel points
  const L = line.value, c = Math.cos(L.phi), s = Math.sin(L.phi), px = (L.d + offset) * c, py = (L.d + offset) * s, far = 20
  return { x1: sx(px + far * s), y1: sy(py - far * c), x2: sx(px - far * s), y2: sy(py + far * c) }
}
const lines = computed(() => ({ mid: lineAt(0), lo: lineAt(-line.value.m), hi: lineAt(line.value.m) }))
const bandPoints = computed(() => { const { lo, hi } = lines.value; return `${lo.x1},${lo.y1} ${lo.x2},${lo.y2} ${hi.x2},${hi.y2} ${hi.x1},${hi.y1}` })
const gridX = range(-VIEW.gridHalfWidth, VIEW.gridHalfWidth), gridY = range(-VIEW.gridHalfHeight, VIEW.gridHalfHeight)
const stepNumbers = range(0, K)

/* ───────────────────────────── 8. INFO BOX (slots) ───────────────────────────── */
// In slides.md:  <template #default="{ step, steps, errors, nSV, margin, final }"> … </template>
// or per step:   <template #step0> … </template>  (#step0 … #stepN win over #default)

const slots = useSlots()
const hasInfo = !!Object.keys(slots).length
const slotName = computed(() => (slots[`step${shown.value}`] ? `step${shown.value}` : 'default'))
const slotProps = computed(() => {
  const L = keys[shown.value], info = keyInfo[shown.value]
  return { step: shown.value, steps: K, moving: !settled.value, final: converged.value,
    errors: info.wrong.length, nSV: info.ringed.length, margin: +(2 * L.m).toFixed(2),
    w: [+(Math.cos(L.phi) / L.m).toFixed(2), +(Math.sin(L.phi) / L.m).toFixed(2)], b: +(-L.d / L.m).toFixed(2) }
})

const cssVars = {   // settings handed to the stylesheet below
  '--ax': COLORS.controls, '--info-bg': COLORS.infoBackground, '--info-fg': COLORS.infoText,
  '--cols': `${LAYOUT.plotShare}fr ${LAYOUT.panelShare}fr`, '--info-min': LAYOUT.infoMinHeight, '--info-size': LAYOUT.infoFontSize,
  '--info-pad': LAYOUT.infoPadding, '--info-radius': LAYOUT.infoRadius,
  '--ring-w': String(STYLE.ringWidth), '--ring-pulse': `${STYLE.ringPulseSeconds}s`, '--ring-fade': `${STYLE.ringFadeSeconds}s`,
  '--play-size': STYLE.playButtonSize, '--track': STYLE.sliderTrack, '--thumb': STYLE.sliderThumb, '--arrow-opacity': String(STYLE.slideArrowOpacity),
}
const fill = computed(() => `${(pos.value / K) * 100}%`)   // how much of the slider track is filled
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
      <line v-bind="lines.mid" class="decision" :stroke="converged ? COLORS.lineFinal : COLORS.line" :stroke-width="STYLE.decisionLineWidth" stroke-linecap="round" />

      <g class="rings" :style="{ opacity: settled ? 1 : 0 }">
        <circle v-for="n in keyInfo[shown].ringed" :key="'ring' + n" class="ring" :cx="sx(pts[n].x)" :cy="sy(pts[n].y)" :r="STYLE.ringRadius"
                :stroke="keyInfo[shown].wrong.includes(n) ? COLORS.ringWrong : COLORS.ringNear"
                :fill="keyInfo[shown].wrong.includes(n) ? COLORS.ringWrongFill : 'none'"
                :fill-opacity="keyInfo[shown].wrong.includes(n) ? 0.3 : 0" />
      </g>

      <circle v-for="(p, n) in pts" :key="n" :cx="sx(p.x)" :cy="sy(p.y)" :r="STYLE.pointRadius" :stroke="COLORS.axes" :stroke-width="STYLE.pointOutline"
              :fill="p.c === 1 ? COLORS.blueClass : COLORS.orangeClass" />
    </svg>

    <div class="panel">
      <div class="controls">
        <button class="play" :aria-label="playing ? 'Pause' : 'Play'" :title="playing ? 'Pause' : 'Play'" @click="togglePlay(); dropFocus($event)">
          <svg v-if="!playing" viewBox="0 0 24 24"><path d="M8 5.5v13l11-6.5z" fill="currentColor" /></svg>
          <svg v-else viewBox="0 0 24 24"><path d="M7 5h4v14H7zM13 5h4v14h-4z" fill="currentColor" /></svg>
        </button>

        <div class="scrub">
          <input type="range" min="0" :max="K" step="any" :value="pos" :style="{ '--fill': fill }" aria-label="Animation step"
                 @input="onScrub" @change="onRelease" />
          <div class="ticks">
            <button v-for="n in stepNumbers" :key="n" class="tick" :class="{ on: n === shown && settled }"
                    :style="{ left: `calc(var(--thumb) / 2 + (100% - var(--thumb)) * ${n / K})` }" @click="jump(n); dropFocus($event)">{{ n }}</button>
          </div>
        </div>

        <div v-if="nav" class="slide-nav">
          <button title="Previous slide" aria-label="Previous slide" @click="prevSlide(); dropFocus($event)"><svg viewBox="0 0 24 24"><path d="M15 5l-7 7 7 7" /></svg></button>
          <button title="Next slide" aria-label="Next slide" @click="nextSlide(); dropFocus($event)"><svg viewBox="0 0 24 24"><path d="M9 5l7 7-7 7" /></svg></button>
        </div>
      </div>
      <div v-if="hasInfo" class="info"><slot :name="slotName" v-bind="slotProps" /></div>
    </div>
    <template v-if="clicks"><span v-for="n in K" :key="n" v-click hidden /></template>
  </div>
</template>

<style scoped>
.wrap { display: grid; grid-template-columns: var(--cols); gap: 1.4rem; align-items: center; height: 100%; }
.plot { width: 100%; height: auto; }
.decision { transition: stroke .4s; }
.rings { transition: opacity var(--ring-fade); }
.ring { stroke-width: var(--ring-w); transform-box: fill-box; transform-origin: center; animation: pulse var(--ring-pulse) ease-in-out infinite; }
@keyframes pulse { 0%, 100% { transform: scale(.9); opacity: .95; } 50% { transform: scale(1.22); opacity: .55; } }

/* controls: [play] [slider + step numbers] [slide arrows] */
.panel { display: flex; flex-direction: column; gap: .9rem; }
.controls { display: flex; align-items: flex-start; gap: .8rem; color: var(--ax); }
.controls button { background: none; border: 0; padding: 0; color: inherit; cursor: pointer; font: inherit; }
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
.tick { position: absolute; top: 0; transform: translateX(-50%); font-size: .72rem !important; opacity: .5; padding: 0 .25rem !important; }
.tick:hover, .tick.on { opacity: 1; }
.tick.on { font-weight: 700; }

.slide-nav { flex: none; display: flex; gap: .1rem; height: var(--play-size); align-items: center; opacity: var(--arrow-opacity); transition: opacity .2s; }
.slide-nav:hover { opacity: 1; }
.slide-nav svg { width: 1.3rem; height: 1.3rem; fill: none; stroke: currentColor; stroke-width: 2.2; stroke-linecap: round; stroke-linejoin: round; }

.info { background: var(--info-bg); color: var(--info-fg); border-radius: var(--info-radius); padding: var(--info-pad); font-size: var(--info-size);
  line-height: 1.5; min-height: var(--info-min); display: flex; flex-direction: column; justify-content: center; }
.info :deep(p) { margin: .4em 0; }
.info :deep(.katex-display) { margin: .6em 0; }
.info :deep(.katex) { font-size: 1.1em; }
</style>