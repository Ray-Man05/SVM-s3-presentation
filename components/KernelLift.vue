<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

type Pt = { x: number; y: number; c: 1 | -1 }
const W = 640, H = 400, S = 140, Z = 170

// Explicit feature maps z = φ(x, y). RBF is shown as a 1-D proxy ("similarity to the origin"):
// the true RBF feature space is infinite-dimensional.
const kernels = {
  linear: { label: 'Linear', z: (_x: number, _y: number) => 0 },
  poly: { label: 'Polynomial  z = x² + y²', z: (x: number, y: number) => x * x + y * y },
  rbf: { label: 'RBF  z = e^(−2r²)', z: (x: number, y: number) => Math.exp(-2 * (x * x + y * y)) },
  prod: { label: 'Product  z = xy', z: (x: number, y: number) => x * y },
} as const
type KName = keyof typeof kernels

const kernel = ref<KName>('poly')
const cls = ref<1 | -1>(1)
const pts = ref<Pt[]>([])
const canvas = ref<HTMLCanvasElement>()
let t = 0, target = 0, theta = 0.6, raf = 0, dragging = false, lastX = 0
const lifted = ref(false)
const col: Record<string, string> = {}

function mulberry(seed: number) {
  return () => { seed = (seed + 0x6d2b79f5) | 0; let a = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    a = (a + Math.imul(a ^ (a >>> 7), 61 | a)) ^ a; return ((a ^ (a >>> 14)) >>> 0) / 4294967296 }
}
function preset(name: 'rings' | 'xor' | 'clear') {
  const r = mulberry(3), out: Pt[] = []
  if (name === 'rings') {
    for (let i = 0; i < 18; i++) { const a = r() * 6.283, d = r() * 0.4; out.push({ x: d * Math.cos(a), y: d * Math.sin(a), c: 1 }) }
    for (let i = 0; i < 26; i++) { const a = r() * 6.283, d = 0.7 + r() * 0.25; out.push({ x: d * Math.cos(a), y: d * Math.sin(a), c: -1 }) }
  } else if (name === 'xor') {
    for (const [mx, my, c] of [[.55, .55, 1], [-.55, -.55, 1], [.55, -.55, -1], [-.55, .55, -1]] as const)
      for (let i = 0; i < 9; i++) out.push({ x: mx + (r() - .5) * .4, y: my + (r() - .5) * .4, c })
  }
  pts.value = out
}

function project(x: number, y: number, z: number) {
  const th = theta * t, ph = ((90 - 55 * t) * Math.PI) / 180
  const xr = x * Math.cos(th) - y * Math.sin(th), yr = x * Math.sin(th) + y * Math.cos(th)
  return { X: W / 2 + S * xr, Y: 200 + 55 * t - (S * yr * Math.sin(ph) + Z * z * Math.cos(ph)), d: yr * Math.cos(ph) - z * Math.sin(ph) }
}

function draw() {
  const el = canvas.value; if (!el) return
  const ctx = el.getContext('2d')!, dpr = window.devicePixelRatio || 1
  if (el.width !== W * dpr) { el.width = W * dpr; el.height = H * dpr }
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, W, H)
  ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, W, H)

  const poly = (ps: number[][]) => { ctx.beginPath(); ps.forEach(([x, y, z], i) => { const p = project(x, y, z); i ? ctx.lineTo(p.X, p.Y) : ctx.moveTo(p.X, p.Y) }); ctx.closePath() }
  poly([[-1, -1, 0], [1, -1, 0], [1, 1, 0], [-1, 1, 0]])
  ctx.fillStyle = col.blueHl + '66'; ctx.fill()
  ctx.strokeStyle = '#94a3b8'; ctx.lineWidth = 1
  for (const g of [-0.5, 0, 0.5]) {
    const a = project(g, -1, 0), b = project(g, 1, 0), c = project(-1, g, 0), d = project(1, g, 0)
    ctx.beginPath(); ctx.moveTo(a.X, a.Y); ctx.lineTo(b.X, b.Y); ctx.moveTo(c.X, c.Y); ctx.lineTo(d.X, d.Y); ctx.stroke()
  }
  poly([[-1, -1, 0], [1, -1, 0], [1, 1, 0], [-1, 1, 0]])
  ctx.strokeStyle = col.dark; ctx.lineWidth = 2; ctx.stroke()

  const k = kernels[kernel.value]
  const zs = pts.value.map(p => t * k.z(p.x, p.y))

  // illustrative separating plane: halfway between the two class means in z
  const A = pts.value.map((p, i) => (p.c === 1 ? zs[i] : null)).filter(v => v !== null) as number[]
  const B = pts.value.map((p, i) => (p.c === -1 ? zs[i] : null)).filter(v => v !== null) as number[]
  if (kernel.value !== 'linear' && t > 0.9 && A.length && B.length) {
    const c = (A.reduce((s, v) => s + v, 0) / A.length + B.reduce((s, v) => s + v, 0) / B.length) / 2
    poly([[-1, -1, c], [1, -1, c], [1, 1, c], [-1, 1, c]])
    ctx.fillStyle = col.green + '33'; ctx.fill(); ctx.strokeStyle = col.green; ctx.lineWidth = 2; ctx.stroke()
  }

  const order = pts.value.map((p, i) => ({ p, z: zs[i], pr: project(p.x, p.y, zs[i]) })).sort((a, b) => b.pr.d - a.pr.d)
  for (const { p, z, pr } of order) {
    if (t > 0.02) { const g = project(p.x, p.y, 0); ctx.strokeStyle = '#94a3b855'; ctx.beginPath(); ctx.moveTo(g.X, g.Y); ctx.lineTo(pr.X, pr.Y); ctx.stroke() }
    ctx.beginPath(); ctx.arc(pr.X, pr.Y, 7, 0, 6.283)
    ctx.fillStyle = p.c === 1 ? col.blue : col.orange; ctx.fill()
    ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.5; ctx.stroke()
  }
}

function loop() {
  t += (target - t) * 0.07
  if (Math.abs(target - t) < 0.002) t = target
  draw(); raf = requestAnimationFrame(loop)
}
function toggle() { lifted.value = !lifted.value; target = lifted.value ? 1 : 0 }

function pos(e: PointerEvent) {
  const r = canvas.value!.getBoundingClientRect()          // robust to Slidev's CSS scaling
  return { px: ((e.clientX - r.left) * W) / r.width, py: ((e.clientY - r.top) * H) / r.height }
}
function down(e: PointerEvent) {
  if (t < 0.02) {
    const { px, py } = pos(e), x = (px - W / 2) / S, y = -(py - 200) / S
    if (Math.abs(x) <= 1 && Math.abs(y) <= 1) pts.value = [...pts.value, { x, y, c: cls.value }]
  } else { dragging = true; lastX = e.clientX; canvas.value!.setPointerCapture(e.pointerId) }
}
function move(e: PointerEvent) { if (dragging) { theta += (e.clientX - lastX) * 0.01; lastX = e.clientX } }

onMounted(() => {
  const cs = getComputedStyle(canvas.value!)
  const g = (n: string) => cs.getPropertyValue(n).trim()
  Object.assign(col, { dark: g('--c-dark'), blue: g('--c-blue'), orange: g('--c-orange'), green: g('--c-green'), blueHl: g('--c-blue-highlight') })
  preset('rings'); raf = requestAnimationFrame(loop)
})
onBeforeUnmount(() => cancelAnimationFrame(raf))
</script>

<template>
  <div class="wrap">
    <canvas ref="canvas" class="figure cv" @pointerdown="down" @pointermove="move" @pointerup="dragging = false" />
    <div class="panel">
      <div class="btn-row">
        <button class="btn blue" :class="{ on: cls === 1 }" @click="cls = 1">Class A</button>
        <button class="btn orange" :class="{ on: cls === -1 }" @click="cls = -1">Class B</button>
      </div>
      <div class="btn-row">
        <button v-for="(k, name) in kernels" :key="name" class="btn" :class="{ on: kernel === name }" @click="kernel = name as KName">{{ k.label }}</button>
      </div>
      <div class="btn-row">
        <button class="btn on" @click="toggle">{{ lifted ? 'Back to 2D' : 'Lift to 3D' }}</button>
        <button class="btn" @click="preset('rings')">Rings</button>
        <button class="btn" @click="preset('xor')">XOR</button>
        <button class="btn" @click="preset('clear')">Clear</button>
      </div>
      <div class="stats">{{ lifted ? 'Drag to rotate. The green plane is a linear separator in feature space.' : 'Click the plane to add points.' }}</div>
    </div>
  </div>
</template>

<style scoped>
.wrap { display: grid; grid-template-columns: 2fr 1fr; gap: 1.2rem; align-items: center; height: 100%; }
.cv { width: 100%; aspect-ratio: 640 / 400; border: 2px solid var(--c-dark); touch-action: none; cursor: crosshair; }
</style>
