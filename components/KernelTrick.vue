
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'

const container = ref<HTMLDivElement | null>(null)
const progress = ref(0)
const playing = ref(false)

let renderer: THREE.WebGLRenderer
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let resizeObserver: ResizeObserver

let animationFrame = 0
let elapsed = 0
let previousTime = 0

const duration = 3500

const target = new THREE.Vector3(2.5, 2.5, 0)
const startPosition = new THREE.Vector3(2.5, 2.5, 9)
const endPosition = new THREE.Vector3(2.5, -6, 8)

const startQuaternion = new THREE.Quaternion()
const endQuaternion = new THREE.Quaternion()

function addLine(
  points: THREE.Vector3[],
  color: number,
  opacity = 1
) {
  const geometry = new THREE.BufferGeometry().setFromPoints(points)
  const material = new THREE.LineBasicMaterial({
    color,
    transparent: opacity < 1,
    opacity
  })

  const line = new THREE.Line(geometry, material)
  scene.add(line)
  return line
}

function addArrow(direction: THREE.Vector3, color: number) {
  const arrow = new THREE.ArrowHelper(
    direction.clone().normalize(),
    new THREE.Vector3(0, 0, 0),
    5.2,
    color,
    0.22,
    0.13
  )

  scene.add(arrow)
}

function createScene() {
  scene = new THREE.Scene()
  scene.background = null

  camera = new THREE.PerspectiveCamera(
    42,
    1,
    0.1,
    100
  )

  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 1.8))

  const light = new THREE.DirectionalLight(0xffffff, 2)
  light.position.set(-3, -4, 10)
  scene.add(light)

  // XY grid: the actual Cartesian plane
  for (let i = 0; i <= 5; i++) {
    addLine(
      [
        new THREE.Vector3(i, 0, 0),
        new THREE.Vector3(i, 5, 0)
      ],
      0x777777
    )

    addLine(
      [
        new THREE.Vector3(0, i, 0),
        new THREE.Vector3(5, i, 0)
      ],
      0xdedede
    )
  }

  // Coordinate axes
    addArrow(new THREE.Vector3(1, 0, 0), 0xffffff)
    addArrow(new THREE.Vector3(0, 1, 0), 0xffffff)
    addArrow(new THREE.Vector3(0, 0, 1), 0xffffff)

  // Deterministic points around (2, 2)
  let seed = 42
  function random() {
    seed = (seed * 16807) % 2147483647
    return (seed - 1) / 2147483646
  }

  function addPoint(x: number, y: number, color: number) {
    const geometry = new THREE.SphereGeometry(0.045, 24, 24)
    const material = new THREE.MeshStandardMaterial({
      color,
      roughness: 0.65
    })

    const point = new THREE.Mesh(geometry, material)
    point.position.set(x, y, 0.035)
    scene.add(point)
  }

  // Inner cluster
  for (let i = 0; i < 20; i++) {
    const angle = random() * Math.PI * 2
    const radius = 0.35 + random() * 0.45

    addPoint(
      2 + Math.cos(angle) * radius,
      2 + Math.sin(angle) * radius,
      0xe87532
    )
  }

  // Outer ring
  for (let i = 0; i < 24; i++) {
    const angle = random() * Math.PI * 2
    const radius = 1.25 + random() * 0.55

    addPoint(
      2 + Math.cos(angle) * radius,
      2 + Math.sin(angle) * radius,
      0x3979c6
    )
  }

  // Calculate the two actual camera orientations.
  camera.position.copy(startPosition)
  camera.up.set(0, 1, 0)
  camera.lookAt(target)
  camera.updateMatrixWorld()
  startQuaternion.copy(camera.quaternion)

  camera.position.copy(endPosition)
  camera.up.set(0, 0, 1)
  camera.lookAt(target)
  camera.updateMatrixWorld()
  endQuaternion.copy(camera.quaternion)

  // Restore the initial top-down view.
  camera.position.copy(startPosition)
  camera.quaternion.copy(startQuaternion)
  camera.updateProjectionMatrix()
}

function render() {
  renderer.render(scene, camera)
}

function updateCamera(value: number) {
  // Smoothstep easing
  const t = value * value * (3 - 2 * value)

  camera.position.lerpVectors(
    startPosition,
    endPosition,
    t
  )

  camera.quaternion.slerpQuaternions(
    startQuaternion,
    endQuaternion,
    t
  )

  render()
}

function animate(time: number) {
  if (!playing.value) return

  if (previousTime === 0) previousTime = time

  elapsed += time - previousTime
  previousTime = time

  progress.value = Math.min(elapsed / duration, 1)
  updateCamera(progress.value)

  if (progress.value < 1) {
    animationFrame = requestAnimationFrame(animate)
  } else {
    playing.value = false
    previousTime = 0
  }
}

function play() {
  if (playing.value) return

  if (progress.value >= 1) {
    elapsed = 0
    progress.value = 0
    updateCamera(0)
  }

  playing.value = true
  previousTime = 0
  animationFrame = requestAnimationFrame(animate)
}

function pause() {
  playing.value = false
  previousTime = 0
  cancelAnimationFrame(animationFrame)
}

function reset() {
  pause()
  elapsed = 0
  progress.value = 0
  updateCamera(0)
}

onMounted(() => {
  if (!container.value) return

    renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true,
    powerPreference: 'high-performance'
    })

    renderer.setPixelRatio(Math.min(window.devicePixelRatio * 1.5, 3))

    renderer.setSize(
    container.value.clientWidth,
    container.value.clientHeight
    )

    renderer.setClearColor(0x000000, 0)
    renderer.outputColorSpace = THREE.SRGBColorSpace
    renderer.toneMapping = THREE.ACESFilmicToneMapping
    renderer.toneMappingExposure = 1.15

  container.value.appendChild(renderer.domElement)

  createScene()
  render()

  resizeObserver = new ResizeObserver(() => {
    if (!container.value) return

    const width = container.value.clientWidth
    const height = container.value.clientHeight

    camera.aspect = width / height
    camera.updateProjectionMatrix()

    renderer.setSize(width, height)
    render()
  })

  resizeObserver.observe(container.value)
})

onBeforeUnmount(() => {
  pause()
  resizeObserver?.disconnect()

  renderer?.dispose()
  scene?.traverse(object => {
    if (object instanceof THREE.Mesh || object instanceof THREE.Line) {
      object.geometry.dispose()

      const materials = Array.isArray(object.material)
        ? object.material
        : [object.material]

      materials.forEach(material => material.dispose())
    }
  })
})
</script>

<template>
  <div class="visualization">
    <div ref="container" class="scene"></div>

    <div class="controls">
      <button @click="play" :disabled="playing">
        Play
      </button>

      <button @click="pause" :disabled="!playing">
        Pause
      </button>

      <button @click="reset">
        Reset
      </button>

      <span>{{ Math.round(progress * 100) }}%</span>
    </div>
  </div>
</template>

<style scoped>
.visualization {
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
}

.scene {
  width: 100%;
  height: 390px;
  overflow: hidden;
}

.scene :deep(canvas) {
  display: block;
  width: 100%;
  height: 100%;
}

.controls {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 8px;
  font-size: 13px;
}

button {
  padding: 5px 12px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: white;
  color: #222;
  cursor: pointer;
}

button:disabled {
  opacity: 0.45;
  cursor: default;
}
</style>
