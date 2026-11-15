<script setup lang="ts">
import { ref, computed, useSlots, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'

type Kind = 'ring' | 'xor'
const N = 64          // number of dots
const LIFT_SECONDS = 4.5

const host = ref<HTMLDivElement>()
const choice = ref<Kind | null>(null)
const lifted = ref(false)
const noGL = ref(false)

// ---------- helpers ----------
const clamp01 = (x: number) => Math.min(1, Math.max(0, x))
const smoother = (x: number) => { x = clamp01(x); return x * x * x * (x * (x * 6 - 15) + 10) }
const win = (p: number, a: number, b: number) => smoother((p - a) / (b - a))   // eased progress inside [a, b]
const outBack = (x: number) => { x = clamp01(x); const c1 = 1.70158, c3 = c1 + 1; return 1 + c3 * (x - 1) ** 3 + c1 * (x - 1) ** 2 }
const lerp = (a: number, b: number, t: number) => a + (b - a) * t
function mulberry(seed: number) {
  return () => { seed = (seed + 0x6d2b79f5) | 0; let a = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    a = (a + Math.imul(a ^ (a >>> 7), 61 | a)) ^ a; return ((a ^ (a >>> 14)) >>> 0) / 4294967296 }
}

// ---------- data: one distribution + one kernel per option ----------
type Dataset = { x: number[]; y: number[]; c: (1 | -1)[]; phi: (x: number, y: number) => number; plane: number }
function build(kind: Kind): Dataset {
  const r = mulberry(kind === 'ring' ? 5 : 9)
  const x: number[] = [], y: number[] = [], c: (1 | -1)[] = []
  if (kind === 'ring') {
    for (let i = 0; i < N; i++) {
      const inner = i < 22, a = r() * Math.PI * 2, d = inner ? Math.sqrt(r()) * 0.38 : 0.72 + r() * 0.22
      x.push(d * Math.cos(a)); y.push(d * Math.sin(a)); c.push(inner ? 1 : -1)
    }
  } else {
    const cs = [[.55, .55, 1], [-.55, -.55, 1], [.55, -.55, -1], [-.55, .55, -1]] as const
    for (let i = 0; i < N; i++) {
      const [mx, my, k] = cs[i % 4]
      x.push(mx + (r() - .5) * .4); y.push(my + (r() - .5) * .4); c.push(k)
    }
  }
  // RBF drawn as ONE feature (similarity to the origin): the true RBF feature space is infinite-dimensional.
  const phi = kind === 'ring' ? (a: number, b: number) => Math.exp(-2 * (a * a + b * b)) : (a: number, b: number) => 1.6 * a * b
  const z = x.map((_, i) => phi(x[i], y[i]))
  const minA = Math.min(...z.filter((_, i) => c[i] === 1)), maxB = Math.max(...z.filter((_, i) => c[i] === -1))
  return { x, y, c, phi, plane: (minA + maxB) / 2 }   // max-margin horizontal plane z = c
}
const DATA: Record<Kind, Dataset> = { ring: build('ring'), xor: build('xor') }

// ---------- per-dot state ----------
const rnd = mulberry(77)
const ux = Float32Array.from({ length: N }, () => rnd() * 2 - 1)    // uniform "scatter" start
const uy = Float32Array.from({ length: N }, () => rnd() * 2 - 1)
const delay = Float32Array.from({ length: N }, () => rnd())          // stagger for every animation
const fx = new Float32Array(ux), fy = new Float32Array(uy), tx = new Float32Array(ux), ty = new Float32Array(uy)
const curX = new Float32Array(ux), curY = new Float32Array(uy)
const fromC = new Float32Array(N * 3), toC = new Float32Array(N * 3), curC = new Float32Array(N * 3)
const kz = new Float32Array(N)       // smoothed kernel feature z (so switching distributions never pops)

let disposeAll = () => {}
let startLift: (up: boolean) => void = () => {}
let retarget: (k: Kind | null) => void = () => {}
let replayIntro = () => {}

function choose(k: Kind) { if (choice.value === k) return; choice.value = k; retarget(k) }
function toggleLift() { if (!choice.value) return; lifted.value = !lifted.value; startLift(lifted.value) }
function replay() { lifted.value = false; choice.value = null; startLift(false); replayIntro() }

// Info box content comes from the slide: <template #default>, #ring, #xor (scoped: { choice, lifted })
const slots = useSlots()
const hasInfo = !!(slots.default || slots.ring || slots.xor)
const slotName = computed(() => (choice.value && slots[choice.value] ? choice.value : 'default'))

onMounted(() => {
  const el = host.value!
  const cs = getComputedStyle(el), css = (n: string) => cs.getPropertyValue(n).trim() || '#000'
  const axisCol = cs.getPropertyValue('--c-axis').trim() || cs.getPropertyValue('--c-light').trim() || '#f8fafc'
  const COL = { axis: axisCol, dark: css('--c-dark'), blue: css('--c-blue'), orange: css('--c-orange'), green: css('--c-green'), hl: css('--c-blue-highlight') }
  const cNeutral = new THREE.Color(COL.axis), cBlue = new THREE.Color(COL.blue), cOrange = new THREE.Color(COL.orange)

  let renderer: THREE.WebGLRenderer
  try { renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true }) }
  catch (err) { noGL.value = true; console.error('KernelLift3D: WebGL unavailable', err); return }
  renderer.setClearColor(0x000000, 0)
  el.appendChild(renderer.domElement)
  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(35, 1.6, 0.1, 50)
  scene.add(new THREE.AmbientLight(0xffffff, 1.7))
  const sun = new THREE.DirectionalLight(0xffffff, 1.4); sun.position.set(-3, -5, 8); scene.add(sun)

  // ground plane (z = 0), grid, outline
  const fade: { m: THREE.Material & { opacity: number }; base: number }[] = []
  const track = <T extends THREE.Material & { opacity: number }>(m: T, base: number) => { m.transparent = true; fade.push({ m, base }); return m }
  const ground = new THREE.Mesh(new THREE.PlaneGeometry(2.4, 2.4), track(new THREE.MeshBasicMaterial({ color: COL.axis, side: THREE.DoubleSide, depthWrite: false }), 0.07))
  const grid = new THREE.GridHelper(2.4, 12, COL.axis, COL.axis); grid.rotateX(Math.PI / 2); grid.position.z = 0.002
  track(grid.material as THREE.LineBasicMaterial, 0.22)
  const sq = (h: number) => [[-h, -h, 0], [h, -h, 0], [h, h, 0], [-h, h, 0]].map(p => new THREE.Vector3(...p))
  const outline = new THREE.LineLoop(new THREE.BufferGeometry().setFromPoints(sq(1.2)), track(new THREE.LineBasicMaterial({ color: COL.axis }), 0.55))
  scene.add(ground, grid, outline)

  // axes: thick cylinders + cone heads + text labels
  function label(text: string) {
    const c = document.createElement('canvas'); c.width = c.height = 256
    const g = c.getContext('2d')!; g.fillStyle = COL.axis; g.font = 'italic 600 176px serif'; g.textAlign = 'center'; g.textBaseline = 'middle'; g.fillText(text, 128, 140)
    const tex = new THREE.CanvasTexture(c); tex.colorSpace = THREE.SRGBColorSpace; tex.anisotropy = 8
    const s = new THREE.Sprite(new THREE.SpriteMaterial({ map: tex, transparent: true, depthTest: false })); s.scale.set(0.3, 0.3, 1); return s
  }
  const axisMat = new THREE.MeshBasicMaterial({ color: COL.axis })
  function makeAxis(dir: THREE.Vector3, a: number, b: number, text: string) {
    const q = new THREE.Quaternion().setFromUnitVectors(new THREE.Vector3(0, 1, 0), dir)
    const shaft = new THREE.Mesh(new THREE.CylinderGeometry(0.012, 0.012, 1, 32), axisMat)
    const head = new THREE.Mesh(new THREE.ConeGeometry(0.05, 0.16, 32), axisMat)
    const tag = label(text)
    shaft.quaternion.copy(q); head.quaternion.copy(q)
    scene.add(shaft, head, tag)
    return (p: number) => {
      const end = a + (b - a) * p, len = Math.max(end - a, 1e-4)
      shaft.position.copy(dir).multiplyScalar(a + len / 2); shaft.scale.set(1, len, 1); shaft.visible = p > 0.001
      head.position.copy(dir).multiplyScalar(end + 0.08); head.visible = p > 0.02
      tag.position.copy(dir).multiplyScalar(end + 0.3); tag.material.opacity = win(p, 0.8, 1)
    }
  }
  const axX = makeAxis(new THREE.Vector3(1, 0, 0), -1.45, 1.45, 'x')
  const axY = makeAxis(new THREE.Vector3(0, 1, 0), -1.45, 1.45, 'y')
  const axZ = makeAxis(new THREE.Vector3(0, 0, 1), 0, 1.5, 'z')

  // dots (instanced spheres) + drop lines
  const dots = new THREE.InstancedMesh(new THREE.SphereGeometry(1, 48, 32), new THREE.MeshLambertMaterial(), N)
  dots.instanceMatrix.setUsage(THREE.DynamicDrawUsage)
  for (let i = 0; i < N; i++) dots.setColorAt(i, cNeutral)
  scene.add(dots)
  const dropPos = new Float32Array(N * 6)
  const dropGeo = new THREE.BufferGeometry(); dropGeo.setAttribute('position', new THREE.BufferAttribute(dropPos, 3))
  const drops = new THREE.LineSegments(dropGeo, track(new THREE.LineBasicMaterial({ color: COL.axis }), 0.3)); drops.frustumCulled = false
  scene.add(drops)

  // hyperplane z = c, plus the decision boundary it leaves on the 2D plane (ring only)
  const planeGroup = new THREE.Group()
  const planeMat = track(new THREE.MeshBasicMaterial({ color: COL.green, side: THREE.DoubleSide, depthWrite: false }), 0.28)
  planeGroup.add(new THREE.Mesh(new THREE.PlaneGeometry(2.4, 2.4), planeMat),
    new THREE.LineLoop(new THREE.BufferGeometry().setFromPoints(sq(1.2)), track(new THREE.LineBasicMaterial({ color: COL.green }), 1)))
  scene.add(planeGroup)
  const boundary = new THREE.Mesh(new THREE.TorusGeometry(1, 0.012, 16, 256), track(new THREE.MeshBasicMaterial({ color: COL.green }), 1))
  scene.add(boundary)

  // ---------- animation state ----------
  let introT = 0, morphS = 99, p = 0, pTarget = 0
  let planeCur = 0, planeTarget = 0, phi: (x: number, y: number) => number = () => 0
  let dragAz = 0, dragEl = 0, autoRot = true, dragging = false, lastX = 0, lastY = 0
  const dummy = new THREE.Object3D(), tmp = new THREE.Color()

  retarget = (k) => {
    for (let i = 0; i < N; i++) {
      fx[i] = curX[i]; fy[i] = curY[i]
      for (let j = 0; j < 3; j++) fromC[i * 3 + j] = curC[i * 3 + j]
      const d = k ? DATA[k] : null
      tx[i] = d ? d.x[i] : ux[i]; ty[i] = d ? d.y[i] : uy[i]
      tmp.copy(!d ? cNeutral : d.c[i] === 1 ? cBlue : cOrange); toC[i * 3] = tmp.r; toC[i * 3 + 1] = tmp.g; toC[i * 3 + 2] = tmp.b
    }
    morphS = 0
    if (k) { phi = DATA[k].phi; planeTarget = DATA[k].plane } else { phi = () => 0; planeTarget = 0 }
  }
  startLift = (up) => { pTarget = up ? 1 : 0 }
  replayIntro = () => { introT = 0; fx.set(ux); fy.set(uy); tx.set(ux); ty.set(uy); curX.set(ux); curY.set(uy); morphS = 99
    for (let i = 0; i < N; i++) { curC[i * 3] = fromC[i * 3] = toC[i * 3] = cNeutral.r; curC[i * 3 + 1] = fromC[i * 3 + 1] = toC[i * 3 + 1] = cNeutral.g; curC[i * 3 + 2] = fromC[i * 3 + 2] = toC[i * 3 + 2] = cNeutral.b }
    retarget(null) }
  replayIntro()

  const AZ_END = -0.65, EL_START = 89.5 * Math.PI / 180, EL_END = 22 * Math.PI / 180, R = 5.6

  let raf = 0, last = performance.now(), frameNo = 0
  function frame(now: number) {
    raf = requestAnimationFrame(frame)
    const dt = Math.min((now - last) / 1000, 0.05); last = now
    introT += dt; morphS += dt
    if (p !== pTarget) { const st = dt / LIFT_SECONDS; p = Math.abs(pTarget - p) <= st ? pTarget : p + Math.sign(pTarget - p) * st }
    if (p === 0) { dragAz = 0; dragEl = 0; autoRot = true }
    if (p === 1 && autoRot && !dragging) dragAz += dt * 0.12

    // camera: from straight above (z points at the viewer) to a three-quarter view
    const e = win(p, 0.06, 0.75)
    const el = lerp(EL_START, EL_END, e) + dragEl * e, az = lerp(0, AZ_END, e) + dragAz * e
    const tgt = new THREE.Vector3(0, 0, 0.42 * e)
    camera.up.set(0, 0, 1)
    camera.position.set(tgt.x + R * Math.sin(az) * Math.cos(el), tgt.y - R * Math.cos(az) * Math.cos(el), tgt.z + R * Math.sin(el))
    camera.lookAt(tgt)

    // intro: ground and x/y axes draw themselves, then the dots rain onto the plane
    const gI = win(introT, 0, 1.1)
    for (const f of fade) f.m.opacity = f.base * gI * (f.m === planeMat ? win(p, 0.8, 1) : 1)
    axX(win(introT, 0.1, 1.3)); axY(win(introT, 0.3, 1.5)); axZ(win(p, 0.02, 0.38))

    // morph to the chosen distribution + colours
    const kk = 1 - Math.exp(-dt * 6)
    for (let i = 0; i < N; i++) {
      const m = smoother((morphS - delay[i] * 0.6) / 1.0)
      curX[i] = lerp(fx[i], tx[i], m); curY[i] = lerp(fy[i], ty[i], m)
      for (let j = 0; j < 3; j++) curC[i * 3 + j] = lerp(fromC[i * 3 + j], toC[i * 3 + j], m)
      kz[i] += (phi(curX[i], curY[i]) - kz[i]) * kk
      const a = clamp01((introT - 0.9 - delay[i] * 1.0) / 0.55)
      const lift = win(p, 0.34 + delay[i] * 0.14, 0.68 + delay[i] * 0.14)
      const z = lift * kz[i] + (1 - a) * 0.7 + 0.035
      dummy.position.set(curX[i], curY[i], z); dummy.scale.setScalar(0.055 * outBack(a)); dummy.updateMatrix()
      dots.setMatrixAt(i, dummy.matrix)
      tmp.setRGB(curC[i * 3], curC[i * 3 + 1], curC[i * 3 + 2]); dots.setColorAt(i, tmp)
      dropPos.set([curX[i], curY[i], 0.002, curX[i], curY[i], z], i * 6)
    }
    dots.instanceMatrix.needsUpdate = true; dots.instanceColor!.needsUpdate = true
    dropGeo.attributes.position.needsUpdate = true
    drops.visible = p > 0.3

    // hyperplane and its trace on the 2D plane
    planeCur += (planeTarget - planeCur) * kk
    const pl = win(p, 0.8, 1)
    planeGroup.visible = pl > 0.001; planeGroup.position.z = planeCur; planeGroup.scale.set(lerp(0.7, 1, pl), lerp(0.7, 1, pl), 1)
    const ring = choice.value === 'ring' && planeCur > 0.02 && planeCur < 0.98
    boundary.visible = ring && win(p, 0.9, 1) > 0.001
    if (ring) { const rc = Math.sqrt(-Math.log(planeCur) / 2); boundary.scale.set(rc, rc, 1); boundary.position.z = 0.004 }
    if (++frameNo % 30 === 0) fit()   // catches Slidev scale changes (fullscreen, window resize)
    renderer.render(scene, camera)
  }
  raf = requestAnimationFrame(frame)

  // ---------- resize + drag-to-rotate ----------
  // Slidev scales slides with a CSS transform, so clientWidth is NOT the on-screen size:
  // measure the real scale and render at that resolution, with 1.5x supersampling on top.
  let fitKey = ''
  function fit() {
    const w = el.clientWidth, h = el.clientHeight
    if (!w || !h) return
    const scale = el.getBoundingClientRect().width / w
    const pr = Math.min(Math.max(window.devicePixelRatio * scale, 1) * 1.5, 4)
    const key = `${w}x${h}@${pr.toFixed(2)}`
    if (key === fitKey) return
    fitKey = key
    renderer.setPixelRatio(pr); renderer.setSize(w, h, false)
    camera.aspect = w / h; camera.updateProjectionMatrix()
  }
  const ro = new ResizeObserver(fit)
  window.addEventListener('resize', fit)
  ro.observe(el)
  const down = (ev: PointerEvent) => { if (p < 0.95) return; dragging = true; autoRot = false; lastX = ev.clientX; lastY = ev.clientY; el.setPointerCapture(ev.pointerId) }
  const move = (ev: PointerEvent) => { if (!dragging) return
    dragAz += (ev.clientX - lastX) * 0.008; dragEl = Math.max(-0.35, Math.min(0.6, dragEl + (ev.clientY - lastY) * 0.006)); lastX = ev.clientX; lastY = ev.clientY }
  const up = () => { dragging = false }
  el.addEventListener('pointerdown', down); el.addEventListener('pointermove', move); el.addEventListener('pointerup', up)

  disposeAll = () => {
    cancelAnimationFrame(raf); ro.disconnect(); window.removeEventListener('resize', fit)
    el.removeEventListener('pointerdown', down); el.removeEventListener('pointermove', move); el.removeEventListener('pointerup', up)
    scene.traverse((o: any) => { o.geometry?.dispose(); const m = o.material; (Array.isArray(m) ? m : m ? [m] : []).forEach((x: any) => { x.map?.dispose(); x.dispose() }) })
    renderer.dispose(); renderer.domElement.remove()
  }
})
onBeforeUnmount(() => disposeAll())
</script>

<template>
  <div class="wrap">
    <div ref="host" class="figure scene"><p v-if="noGL" class="nogl">WebGL is not available in this browser.</p></div>
    <div class="panel">
      <div class="btn-row">
        <button class="btn" :class="{ on: choice === 'ring' }" @click="choose('ring')">Ring + RBF</button>
        <button class="btn" :class="{ on: choice === 'xor' }" @click="choose('xor')">XOR + Product</button>
      </div>
      <div class="btn-row">
        <button class="btn on" :disabled="!choice" @click="toggleLift">{{ lifted ? 'Back to 2D' : 'Lift to 3D' }}</button>
        <button class="btn" @click="replay">Replay</button>
      </div>
      <div v-if="hasInfo" class="info"><slot :name="slotName" :choice="choice" :lifted="lifted" /></div>
    </div>
  </div>
</template>

<style scoped>
.wrap { display: grid; grid-template-columns: 3fr 2fr; gap: 1.4rem; align-items: center; height: 100%; }
.scene { width: 100%; aspect-ratio: 640 / 400; touch-action: none; position: relative; }
.scene :deep(canvas) { display: block; width: 100%; height: 100%; }
.nogl { margin: 0; padding: 2rem; text-align: center; opacity: .7; }
.btn:disabled { opacity: .4; cursor: default; }
/* light box, dark text: the inverse of the rest of the deck */
.info { background: var(--c-light); color: var(--c-dark); border-radius: 1rem; padding: 1rem 1.3rem; font-size: 1rem; line-height: 1.5; min-height: 14rem; display: flex; flex-direction: column; justify-content: center; }
.info :deep(p) { margin: .4em 0; }
.info :deep(.katex-display) { margin: .6em 0; }
.info :deep(.katex) { font-size: 1.1em; }
</style>