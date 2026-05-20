<script setup>
import {
  computed,
  nextTick,
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  toRaw,
  watch,
} from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import {
  CSS2DRenderer,
  CSS2DObject,
} from 'three/addons/renderers/CSS2DRenderer.js'

const props = defineProps({
  /** { name?, title?, gradientColors: string[], items: { title?, name?, height: number }[] } */
  landscapeData: {
    type: Object,
    required: true,
  },
})

const MAX_COLORS = 12

const sceneWrap = ref(null)
const sceneMount = ref(null)
const labelsMount = ref(null)

const state = reactive({
  landscape: null,
  positions: [],
  baseWidth: 28,
  maxHeight: 40,
  peakProportionRatio: 5,
  colorOffset: 0,
  bumpAmount: 10,
  bumpTerrain: 15,
  noiseSeed: Math.random() * 1000,
  planeSize: 220,
  segments: 192,
})

const displayTitle = computed(() =>
  (state.landscape?.name || state.landscape?.title || 'landscape').toUpperCase(),
)

function rotateArray(arr, offset) {
  if (!arr?.length) return arr
  const n = ((offset % arr.length) + arr.length) % arr.length
  return arr.slice(n).concat(arr.slice(0, n))
}

const rotatedPalette = computed(() =>
  state.landscape?.gradientColors
    ? rotateArray(state.landscape.gradientColors, state.colorOffset)
    : [],
)

const colorOffsetMax = computed(() =>
  Math.max(0, (state.landscape?.gradientColors?.length || 1) - 1),
)

let terrainGeom
let terrain
let terrainUniforms
let terrainMat
let water
let labelGroup
/** @type {CSS2DObject[]} */
let labelObjects = []

let renderer
let labelRenderer
let scene
let camera
let controls
let animationId = 0
let resizeObserver

const defaultCamPos = new THREE.Vector3(140, 110, 170)

function hash2(ix, iz) {
  const s =
    Math.sin(ix * 12.9898 + iz * 78.233 + state.noiseSeed * 17.31) * 43758.5453
  return (s - Math.floor(s)) * 2 - 1
}

function smooth(t) {
  return t * t * (3 - 2 * t)
}

function valueNoise(x, z) {
  const ix = Math.floor(x)
  const iz = Math.floor(z)
  const fx = x - ix
  const fz = z - iz
  const a = hash2(ix, iz)
  const b = hash2(ix + 1, iz)
  const c = hash2(ix, iz + 1)
  const d = hash2(ix + 1, iz + 1)
  const u = smooth(fx)
  const v = smooth(fz)
  return a + (b - a) * u + (c - a) * v + (a - b - c + d) * u * v
}

function fbm(x, z) {
  let total = 0
  let amp = 1
  let max = 0
  let fx = x * 0.18
  let fz = z * 0.18
  for (let i = 0; i < 4; i++) {
    total += valueNoise(fx, fz) * amp
    max += amp
    amp *= 0.5
    fx *= 2.07
    fz *= 2.03
  }
  return total / max
}

function fbmTerrain(x, z) {
  let total = 0
  let amp = 1
  let max = 0
  let fx = x * 0.048 + 71.17
  let fz = z * 0.048 - 23.41
  for (let i = 0; i < 3; i++) {
    total += valueNoise(fx, fz) * amp
    max += amp
    amp *= 0.52
    fx *= 2.05
    fz *= 2.02
  }
  return total / max
}

function shufflePositions(items) {
  const radius = state.planeSize * 0.32
  return items.map(() => {
    const a = Math.random() * Math.PI * 2
    const r = Math.sqrt(Math.random()) * radius
    return {
      x: Math.cos(a) * r,
      z: Math.sin(a) * r,
      widthJitter: 0.75 + Math.random() * 0.5,
      shape: 0.85 + Math.random() * 0.5,
    }
  })
}

function bumpAt(x, z, peaks) {
  let h = 0
  for (const p of peaks) {
    const dx = x - p.x
    const dz = z - p.z
    const r = Math.sqrt(dx * dx + dz * dz)
    if (r > p.halfWidth) continue
    const t = r / p.halfWidth
    const k = p.shape
    const profile = Math.cos((Math.PI * t) / 2) ** (1.6 * k)
    const local = p.height * profile
    if (local > h) h = local
  }
  return h
}

function buildPeaks() {
  const items = state.landscape.items || []
  const pos = state.positions
  const pr = Math.max(1, Math.min(10, state.peakProportionRatio))
  const alpha = (pr - 1) / 9
  return items.map((item, i) => {
    const p = pos[i] || { x: 0, z: 0, widthJitter: 1, shape: 1 }
    const heightNorm = Math.max(1, Math.min(10, item.height)) / 10
    const baseHalf = (state.baseWidth * p.widthJitter) / 2
    const stretchCoupled = 0.35 + 1.25 * heightNorm
    const stretch = 1 + alpha * (stretchCoupled - 1)
    return {
      name: item.title || item.name || `item ${i + 1}`,
      x: p.x,
      z: p.z,
      halfWidth: baseHalf * stretch,
      height: heightNorm * state.maxHeight,
      shape: p.shape,
    }
  })
}

function rebuildTerrain() {
  if (!terrainGeom) return
  const peaks = buildPeaks()
  const pos = terrainGeom.attributes.position
  const bumpAmt = state.bumpAmount
  const bumpTer = state.bumpTerrain
  const heightRef = Math.max(0.001, state.maxHeight * 0.1)
  const bumpStart = heightRef * 0.35
  const bumpFade = heightRef * 0.55
  let maxY = 0
  for (let i = 0; i < pos.count; i++) {
    const x = pos.getX(i)
    const z = pos.getZ(i)
    const h = bumpAt(x, z, peaks)
    const factor =
      h <= bumpStart
        ? 0
        : Math.min(
            1,
            (h - bumpStart) / Math.max(0.001, bumpFade - bumpStart),
          )
    const terrainWt = 1 - factor
    const nPeak = fbm(x, z) * bumpAmt * factor
    const nBase = fbmTerrain(x, z) * bumpTer * terrainWt
    const y = Math.max(0, h + nBase + nPeak)
    pos.setY(i, y)
    if (y > maxY) maxY = y
  }
  pos.needsUpdate = true
  terrainGeom.computeVertexNormals()
  terrainUniforms.uMinY.value = 0
  terrainUniforms.uMaxY.value = Math.max(0.01, maxY)
  updateLabels(peaks)
}

function applyGradient() {
  if (!terrainUniforms) return
  const colors = rotateArray(
    state.landscape.gradientColors || ['#888'],
    state.colorOffset,
  )
  const count = Math.min(colors.length, MAX_COLORS)
  for (let i = 0; i < count; i++) {
    terrainUniforms.uColors.value[i].set(colors[i])
  }
  terrainUniforms.uColorCount.value = count
}

function updateLabels(peaks) {
  if (!labelGroup) return
  for (const obj of labelObjects) {
    labelGroup.remove(obj)
    if (obj.element?.parentNode)
      obj.element.parentNode.removeChild(obj.element)
  }
  labelObjects = []
  const wrapClass =
    'relative flex -translate-x-1/2 -translate-y-full flex-col items-center'
  const labelClass =
    'whitespace-nowrap px-[8.4px] py-[1.4px] font-mono font-medium text-[15.4px] text-white/95 tracking-wide [text-shadow:0_1px_0_rgba(0,0,0,0.45),0_0_6px_rgba(0,0,0,0.35)]'
  const lineClass = 'mt-px h-[25.2px] w-px shrink-0 bg-white/80'
  for (const peak of peaks) {
    const wrap = document.createElement('div')
    wrap.className = wrapClass
    const span = document.createElement('span')
    span.className = labelClass
    span.textContent = peak.name
    const line = document.createElement('span')
    line.className = lineClass
    line.setAttribute('aria-hidden', 'true')
    wrap.appendChild(span)
    wrap.appendChild(line)
    const obj = new CSS2DObject(wrap)
    obj.position.set(peak.x, peak.height + 6, peak.z)
    labelGroup.add(obj)
    labelObjects.push(obj)
  }
}

function generateCoherentPalette6() {
  const tmp = new THREE.Color()
  const hex = []
  const h0 = Math.random()
  const clockwise = Math.random() < 0.5 ? 1 : -1
  const hueArc = clockwise * (0.032 + Math.random() * 0.11)
  const sLow = THREE.MathUtils.clamp(
    0.28 + Math.random() * 0.38,
    0.22,
    0.82,
  )
  const sHigh = THREE.MathUtils.clamp(
    sLow + (Math.random() - 0.35) * 0.35,
    0.18,
    0.92,
  )
  const lDark = THREE.MathUtils.clamp(
    0.1 + Math.random() * 0.14,
    0.06,
    0.28,
  )
  const lLight = THREE.MathUtils.clamp(
    0.78 + Math.random() * 0.14,
    0.72,
    0.96,
  )

  for (let i = 0; i < 6; i++) {
    const t = i / 5
    const te = t * t * (3 - 2 * t)
    let h = h0 + hueArc * te
    h -= Math.floor(h)
    const s = THREE.MathUtils.clamp(
      THREE.MathUtils.lerp(sLow, sHigh, te) + Math.sin(te * Math.PI) * 0.07,
      0.14,
      1,
    )
    const l = THREE.MathUtils.clamp(
      THREE.MathUtils.lerp(lDark, lLight, te),
      0.05,
      0.97,
    )
    tmp.setHSL(h, s, l)
    hex.push('#' + tmp.getHexString())
  }
  return hex
}

function onRandomPalette() {
  state.landscape.gradientColors = generateCoherentPalette6()
  state.colorOffset = 0
  if (terrainUniforms) applyGradient()
}

/**
 * Snapshot prop into plain nested objects (structuredClone rejects Vue reactive proxies).
 */
function cloneLandscapeFromProp(input) {
  return JSON.parse(JSON.stringify(toRaw(input)))
}

function syncLandscapeFromProp() {
  state.landscape = cloneLandscapeFromProp(props.landscapeData)
  state.positions = shufflePositions(state.landscape.items || [])
  state.colorOffset = 0
  if (!terrainGeom || !terrainUniforms) return
  applyGradient()
  rebuildTerrain()
}

function getViewportSize() {
  const el = sceneWrap.value
  if (!el) return { w: 960, h: 700 }
  return { w: el.clientWidth, h: el.clientHeight }
}

function disposeLabels() {
  if (!labelGroup) {
    labelObjects = []
    return
  }
  for (const obj of labelObjects) {
    labelGroup.remove(obj)
    if (obj.element?.parentNode)
      obj.element.parentNode.removeChild(obj.element)
  }
  labelObjects = []
}

function disposeThree() {
  cancelAnimationFrame(animationId)
  animationId = 0
  resizeObserver?.disconnect()
  resizeObserver = undefined

  disposeLabels()

  if (scene && terrain) {
    scene.remove(terrain)
  }
  terrainGeom?.dispose?.()
  terrainGeom = undefined
  terrainMat?.dispose?.()
  terrainMat = undefined
  terrainUniforms = undefined
  terrain = undefined

  if (scene && water) {
    scene.remove(water)
    water.geometry?.dispose?.()
    water.material?.dispose?.()
  }
  water = undefined

  controls?.dispose?.()
  controls = undefined

  if (renderer?.domElement?.parentNode)
    renderer.domElement.parentNode.removeChild(renderer.domElement)
  renderer?.dispose?.()
  renderer = undefined
  labelRenderer = undefined
  scene = undefined
  camera = undefined
  labelGroup = undefined
}

function initThree() {
  disposeThree()

  const sceneEl = sceneMount.value
  const labelsEl = labelsMount.value
  if (!sceneEl || !labelsEl) return

  const { w, h } = getViewportSize()

  renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: false,
    logarithmicDepthBuffer: true,
  })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(w, h)
  renderer.outputColorSpace = THREE.SRGBColorSpace
  sceneEl.appendChild(renderer.domElement)

  labelRenderer = new CSS2DRenderer({ element: labelsEl })
  labelRenderer.setSize(w, h)

  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xefe7d8)
  scene.fog = new THREE.Fog(0xefe7d8, 220, 460)

  camera = new THREE.PerspectiveCamera(38, w / h, 0.1, 2000)
  camera.position.copy(defaultCamPos)
  camera.lookAt(0, 0, 0)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.08
  controls.minDistance = 60
  controls.maxDistance = 380
  controls.maxPolarAngle = Math.PI / 2.05
  controls.target.set(0, 5, 0)

  scene.add(new THREE.AmbientLight(0xfff3df, 0.6))
  const sun = new THREE.DirectionalLight(0xffe7b0, 1.15)
  sun.position.set(80, 140, 60)
  scene.add(sun)
  const back = new THREE.DirectionalLight(0xb6c8e0, 0.35)
  back.position.set(-90, 60, -120)
  scene.add(back)

  const waterUniforms = {
    uColorA: { value: new THREE.Color('#efe7d8') },
    uColorB: { value: new THREE.Color('#c9bfa9') },
    uRingColor: { value: new THREE.Color('#ffffff') },
  }
  const waterMat = new THREE.ShaderMaterial({
    uniforms: waterUniforms,
    transparent: true,
    vertexShader: /* glsl */ `
      varying vec2 vUv;
      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: /* glsl */ `
      varying vec2 vUv;
      uniform vec3 uColorA;
      uniform vec3 uColorB;
      uniform vec3 uRingColor;
      void main() {
        vec2 p = (vUv - 0.5) * 2.0;
        float d = length(p);
        vec3 base = mix(uColorA, uColorB, smoothstep(0.0, 1.0, d));
        float rings = smoothstep(0.45, 0.5, fract(d * 14.0)) - smoothstep(0.5, 0.55, fract(d * 14.0));
        float ringFade = smoothstep(1.1, 0.05, d);
        vec3 col = mix(base, uRingColor, rings * 0.35 * ringFade);
        float vignette = smoothstep(1.25, 0.2, d);
        col = mix(uColorA * 0.92, col, vignette);
        gl_FragColor = vec4(col, 1.0);
      }
    `,
  })
  water = new THREE.Mesh(
    new THREE.PlaneGeometry(
      state.planeSize * 2.6,
      state.planeSize * 2.6,
      1,
      1,
    ),
    waterMat,
  )
  water.rotation.x = -Math.PI / 2
  water.position.y = -0.35
  scene.add(water)

  terrainUniforms = {
    uColors: {
      value: new Array(MAX_COLORS).fill(0).map(() => new THREE.Color('#888')),
    },
    uColorCount: { value: 1 },
    uMinY: { value: 0 },
    uMaxY: { value: 1 },
    uLightDir: { value: new THREE.Vector3(0.5, 0.8, 0.3).normalize() },
  }

  terrainMat = new THREE.ShaderMaterial({
    uniforms: terrainUniforms,
    vertexShader: /* glsl */ `
      varying vec3 vWorldPos;
      varying vec3 vNormal;
      void main() {
        vec4 wp = modelMatrix * vec4(position, 1.0);
        vWorldPos = wp.xyz;
        vNormal = normalize(normalMatrix * normal);
        gl_Position = projectionMatrix * viewMatrix * wp;
      }
    `,
    fragmentShader: /* glsl */ `
      uniform vec3 uColors[${MAX_COLORS}];
      uniform int uColorCount;
      uniform float uMinY;
      uniform float uMaxY;
      uniform vec3 uLightDir;
      varying vec3 vWorldPos;
      varying vec3 vNormal;

      vec3 sampleGradient(float t) {
        t = clamp(t, 0.0, 1.0);
        float scaled = t * float(uColorCount - 1);
        int idx = int(floor(scaled));
        float f = fract(scaled);
        int nextIdx = idx + 1;
        if (nextIdx > uColorCount - 1) nextIdx = uColorCount - 1;
        vec3 a = uColors[0];
        vec3 b = uColors[0];
        for (int i = 0; i < ${MAX_COLORS}; i++) {
          if (i == idx) a = uColors[i];
          if (i == nextIdx) b = uColors[i];
        }
        return mix(a, b, f);
      }

      void main() {
        float t = (vWorldPos.y - uMinY) / max(0.0001, (uMaxY - uMinY));
        float gradT = clamp((t - 0.07) / 0.93, 0.0, 1.0);
        vec3 col = sampleGradient(gradT);
        vec3 N = normalize(vNormal);
        vec3 L = normalize(uLightDir);
        float shadeBlend = smoothstep(0.0, 0.28, t);
        shadeBlend = shadeBlend * shadeBlend;
        vec3 Nsh = normalize(mix(vec3(0.0, 1.0, 0.0), N, shadeBlend));
        float ndotl = max(dot(Nsh, L), 0.0);
        float shadeAmt = shadeBlend * shadeBlend;
        float lum = mix(
          0.88 + 0.12 * ndotl,
          0.38 + 0.62 * ndotl,
          shadeAmt
        );
        col *= lum;
        float ao = smoothstep(-0.2, 1.0, Nsh.y);
        col *= mix(0.97, 0.78 + 0.22 * ao, shadeAmt);
        float rimAmt = shadeBlend * shadeBlend;
        float rim = pow(1.0 - max(dot(Nsh, vec3(0.0,1.0,0.0)), 0.0), 2.0);
        col += rim * 0.04 * rimAmt;
        gl_FragColor = vec4(col, 1.0);
      }
    `,
  })

  terrainGeom = new THREE.PlaneGeometry(
    state.planeSize,
    state.planeSize,
    state.segments,
    state.segments,
  )
  terrainGeom.rotateX(-Math.PI / 2)
  terrain = new THREE.Mesh(terrainGeom, terrainMat)
  scene.add(terrain)

  labelGroup = new THREE.Group()
  scene.add(labelGroup)

  applyGradient()
  rebuildTerrain()

  function tick() {
    animationId = requestAnimationFrame(tick)
    controls.update()
    renderer.render(scene, camera)
    labelRenderer.render(scene, camera)
  }
  tick()

  resizeObserver = new ResizeObserver(() => {
    const { w: nw, h: nh } = getViewportSize()
    renderer.setSize(nw, nh)
    labelRenderer.setSize(nw, nh)
    camera.aspect = nw / nh
    camera.updateProjectionMatrix()
  })
  resizeObserver.observe(sceneWrap.value)
}

function onReshuffle() {
  state.positions = shufflePositions(state.landscape.items || [])
  state.noiseSeed = Math.random() * 1000
  rebuildTerrain()
}

function onResetView() {
  camera?.position.copy(defaultCamPos)
  if (controls) {
    controls.target.set(0, 5, 0)
    controls.update()
  }
}

watch(
  () => props.landscapeData,
  () => syncLandscapeFromProp(),
  { deep: true },
)

watch(
  () => state.colorOffset,
  () => {
    applyGradient()
  },
)

onMounted(async () => {
  syncLandscapeFromProp()
  await nextTick()
  initThree()
})

onBeforeUnmount(() => {
  disposeThree()
})
</script>

<template>
  <div class="relative">
    <div
      v-if="state.landscape?.items?.length"
      ref="sceneWrap"
      class="relative mx-auto aspect-16/10 w-[min(92vw,1200px)] max-h-[min(76vh,720px)] rounded-xl bg-[#efe7d8] shadow-[0_12px_40px_rgba(0,0,0,0.12),0_0_0_1px_rgba(0,0,0,0.06)]"
    >
      <div ref="sceneMount" class="absolute inset-0" />
      <div ref="labelsMount" class="pointer-events-none absolute inset-0" />

      <header
        class="pointer-events-none absolute left-0 top-0 z-20 p-4 px-5"
      >
        <h1
          class="m-0 font-mono text-[1.375rem] font-bold tracking-tight text-white mix-blend-difference"
        >
          {{ displayTitle }}
        </h1>
        <p
          class="mt-1.5 m-0 font-mono text-[10px] uppercase tracking-[0.28em] text-neutral-100 mix-blend-difference"
        >
          3D Landscape Generator
        </p>
      </header>

      <aside
        class="absolute right-3 top-3 z-30 w-60 max-w-[42vw] rounded-2xl border border-black/10 bg-white/90 p-4 shadow-xl backdrop-blur-md"
        @click.stop
      >
        <div class="mb-4">
          <h2 class="m-0 font-mono text-sm font-semibold uppercase tracking-[0.2em] text-neutral-900">
            Options
          </h2>
        </div>

        <div class="mb-4">
          <div
            class="mb-1 flex items-baseline justify-between font-mono text-[11px]"
          >
            <label class="tracking-wide uppercase text-neutral-700" for="lw-baseWidth"
              >Base width (avg)</label
            >
            <span class="text-xs text-neutral-900">{{ state.baseWidth }}</span>
          </div>
          <input
            id="lw-baseWidth"
            v-model.number="state.baseWidth"
            type="range"
            min="6"
            max="80"
            step="1"
            class="w-full accent-[#f1b814]"
            @input="rebuildTerrain"
          />
        </div>

        <div class="mb-4">
          <div
            class="mb-1 flex items-baseline justify-between font-mono text-[11px]"
          >
            <label class="tracking-wide uppercase text-neutral-700" for="lw-maxHeight"
              >Max peak height</label
            >
            <span class="text-xs text-neutral-900">{{ state.maxHeight }}</span>
          </div>
          <input
            id="lw-maxHeight"
            v-model.number="state.maxHeight"
            type="range"
            min="5"
            max="120"
            step="1"
            class="w-full accent-[#f1b814]"
            @input="rebuildTerrain"
          />
        </div>

        <div class="mb-4">
          <div
            class="mb-1 flex items-baseline justify-between font-mono text-[11px]"
          >
            <label class="tracking-wide uppercase text-neutral-700" for="lw-peakRatio"
              >Peak proportion ratio</label
            >
            <span class="text-xs text-neutral-900">{{
              state.peakProportionRatio
            }}</span>
          </div>
          <input
            id="lw-peakRatio"
            v-model.number="state.peakProportionRatio"
            type="range"
            min="1"
            max="10"
            step="1"
            class="w-full accent-[#f1b814]"
            @input="rebuildTerrain"
          />
        </div>

        <div class="mb-4">
          <div
            class="mb-1 flex items-baseline justify-between font-mono text-[11px]"
          >
            <label class="tracking-wide uppercase text-neutral-700" for="lw-co"
              >Color offset</label
            >
            <span class="text-xs text-neutral-900">{{ state.colorOffset }}</span>
          </div>
          <input
            id="lw-co"
            v-model.number="state.colorOffset"
            type="range"
            min="0"
            :max="colorOffsetMax"
            step="1"
            class="w-full accent-[#f1b814]"
          />
        </div>

        <div class="mb-4">
          <div
            class="mb-1 flex items-baseline justify-between font-mono text-[11px]"
          >
            <label class="tracking-wide uppercase text-neutral-700" for="lw-bump"
              >Bump amount</label
            >
            <span class="text-xs text-neutral-900">{{
              state.bumpAmount.toFixed(2)
            }}</span>
          </div>
          <input
            id="lw-bump"
            v-model.number="state.bumpAmount"
            type="range"
            min="0"
            max="20"
            step="0.05"
            class="w-full accent-[#f1b814]"
            @input="rebuildTerrain"
          />
        </div>

        <div class="mb-4">
          <div
            class="mb-1 flex items-baseline justify-between font-mono text-[11px]"
          >
            <label class="tracking-wide uppercase text-neutral-700" for="lw-bumpt"
              >Bump terrain</label
            >
            <span class="text-xs text-neutral-900">{{
              state.bumpTerrain.toFixed(2)
            }}</span>
          </div>
          <input
            id="lw-bumpt"
            v-model.number="state.bumpTerrain"
            type="range"
            min="0"
            max="30"
            step="0.05"
            class="w-full accent-[#f1b814]"
            @input="rebuildTerrain"
          />
        </div>

        <div class="flex gap-1.5 pt-1">
          <button
            type="button"
            class="flex-1 cursor-pointer rounded-lg border border-neutral-300 bg-neutral-50 py-2 font-mono text-[11px] font-medium uppercase tracking-wider text-neutral-900 hover:border-amber-500 hover:text-amber-800"
            @click="onReshuffle"
          >
            Reshuffle
          </button>
          <button
            type="button"
            class="flex-1 cursor-pointer rounded-lg border border-neutral-300 bg-neutral-50 py-2 font-mono text-[11px] font-medium uppercase tracking-wider text-neutral-900 hover:border-amber-500 hover:text-amber-800"
            @click="onResetView"
          >
            Reset view
          </button>
        </div>

        <div class="mt-4 border-t border-neutral-200 pt-4">
          <p class="mb-2 m-0 font-mono text-[10px] uppercase tracking-[0.22em] text-neutral-500">
            Palette
          </p>
          <div class="flex flex-wrap gap-1.5">
            <div
              v-for="(c, i) in rotatedPalette"
              :key="i"
              class="h-7 w-7 rounded-md border border-black/20"
              :style="{ background: c }"
              :title="`${i}: ${c}`"
            />
          </div>
          <button
            type="button"
            class="mt-3 w-full cursor-pointer rounded-lg border border-neutral-300 bg-neutral-50 py-2 font-mono text-[11px] font-medium uppercase tracking-wider text-neutral-900 hover:border-violet-400 hover:text-violet-800"
            @click="onRandomPalette"
          >
            Random coherent palette
          </button>
        </div>
      </aside>
    </div>
    <p v-else class="p-8 text-center font-sans text-neutral-600">
      Landscape JSON missing
      <code class="rounded bg-neutral-200/80 px-1 font-mono text-sm text-neutral-800"
        >items</code
      >.
    </p>
  </div>
</template>
