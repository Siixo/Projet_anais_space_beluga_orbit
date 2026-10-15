<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const container = ref(null)
const showHint = ref(true)
let renderer, frameId, handleResize

onMounted(() => {
  if (!container.value) return

  const scene = new THREE.Scene()
  const width = container.value.clientWidth || window.innerWidth
  const height = container.value.clientHeight || window.innerHeight

  const loader = new THREE.TextureLoader()
  const map = loader.load('/2k_earth_nightmap.jpg')
  map.colorSpace = THREE.SRGBColorSpace

  const camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 100)
  camera.position.set(0, 0, 8)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false })
  renderer.setSize(width, height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.domElement.style.display = 'block'
  container.value.appendChild(renderer.domElement)

  // Lighting
  scene.add(new THREE.AmbientLight(0xffffff, 1.3))
  scene.add(new THREE.HemisphereLight(0xffffff, 0x4466aa, 1.2))
  scene.background = new THREE.Color(0x050816)

  const sun = new THREE.DirectionalLight(0xffffff, 2.2)
  sun.position.set(6, 4, 6)
  scene.add(sun)

  // Earth
  const earth = new THREE.Mesh(
    new THREE.SphereGeometry(2, 48, 48),
    new THREE.MeshBasicMaterial({ map })
  )
  scene.add(earth)

  // Atmosphere Glow
  const atmosphereGeo = new THREE.SphereGeometry(2.07, 48, 48)
  const atmosphereMat = new THREE.ShaderMaterial({
    vertexShader: `
      varying vec3 vNormal;
      void main() {
        vNormal = normalize(normalMatrix * normal);
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      varying vec3 vNormal;
      void main() {
        float intensity = pow(0.68 - dot(vNormal, vec3(0, 0, 1.0)), 2.8);
        gl_FragColor = vec4(0.35, 0.7, 1.0, 1.0) * intensity;
      }
    `,
    blending: THREE.AdditiveBlending,
    side: THREE.BackSide,
    transparent: true,
  })
  const atmosphere = new THREE.Mesh(atmosphereGeo, atmosphereMat)
  scene.add(atmosphere)

  // Starfield
  const STAR_COUNT = 2000
  const starPositions = new Float32Array(STAR_COUNT * 3)
  for (let i = 0; i < STAR_COUNT; i++) {
    const u = Math.random() * 2 - 1
    const theta = Math.random() * Math.PI * 2
    const r = Math.sqrt(1 - u * u)
    const radius = 30 + Math.random() * 20
    starPositions[i * 3] = radius * r * Math.cos(theta)
    starPositions[i * 3 + 1] = radius * u
    starPositions[i * 3 + 2] = radius * r * Math.sin(theta)
  }
  const starGeometry = new THREE.BufferGeometry()
  starGeometry.setAttribute('position', new THREE.BufferAttribute(starPositions, 3))
  const starMaterial = new THREE.PointsMaterial({
    color: 0xffffff,
    size: 0.15,
    sizeAttenuation: true,
    transparent: true,
    opacity: 0.85,
    depthWrite: false,
  })
  const stars = new THREE.Points(starGeometry, starMaterial)
  scene.add(stars)

  // Orbit Hierarchy
  const ORBIT_RADIUS = 2.7
  const orbitTilt = new THREE.Group()
  orbitTilt.rotation.x = 0.35 // Fixed orbit inclination from the first version
  scene.add(orbitTilt)

  const pivot = new THREE.Group()
  orbitTilt.add(pivot)

  const belugaRoot = new THREE.Group()
  const orient = new THREE.Group()
  pivot.add(belugaRoot)
  belugaRoot.add(orient)
  belugaRoot.position.set(0, 0, ORBIT_RADIUS)

  // Nose forward along the circular trajectory
  orient.rotation.y = -Math.PI / 2
  orient.rotation.z = -0.15

  // Glowing Orbital Guide Ring
  const ringCurve = new THREE.EllipseCurve(0, 0, ORBIT_RADIUS, ORBIT_RADIUS, 0, Math.PI * 2, false, 0)
  const ringPoints = ringCurve.getPoints(140).map(p => new THREE.Vector3(p.x, 0, p.y))
  const ringGeometry = new THREE.BufferGeometry().setFromPoints(ringPoints)
  const ringMaterial = new THREE.LineBasicMaterial({
    color: 0x4fc3f7,
    transparent: true,
    opacity: 0.25,
    blending: THREE.AdditiveBlending,
  })
  const orbitRing = new THREE.LineLoop(ringGeometry, ringMaterial)
  orbitTilt.add(orbitRing)

  // Glowing Jet Contrail
  const TRAIL_LENGTH = 80
  const trailPositions = new Float32Array(TRAIL_LENGTH * 3)
  for (let i = 0; i < TRAIL_LENGTH; i++) {
    trailPositions[i * 3] = 0
    trailPositions[i * 3 + 1] = 0
    trailPositions[i * 3 + 2] = ORBIT_RADIUS
  }
  const trailGeometry = new THREE.BufferGeometry()
  trailGeometry.setAttribute('position', new THREE.BufferAttribute(trailPositions, 3))
  const trailMaterial = new THREE.LineBasicMaterial({
    color: 0x5cd9ff,
    transparent: true,
    opacity: 0.6,
    blending: THREE.AdditiveBlending,
  })
  const trailLine = new THREE.Line(trailGeometry, trailMaterial)
  scene.add(trailLine)

  // Load Beluga 3D Model
  new GLTFLoader().load(
    '/models/beluga.glb',
    (gltf) => {
      console.log('Beluga model loaded successfully!', gltf)
      const beluga = gltf.scene

      const box = new THREE.Box3().setFromObject(beluga)
      const size = box.getSize(new THREE.Vector3())
      const maxDim = Math.max(size.x, size.y, size.z) || 1
      beluga.scale.setScalar(0.85 / maxDim)

      const scaledBox = new THREE.Box3().setFromObject(beluga)
      const center = scaledBox.getCenter(new THREE.Vector3())
      beluga.position.sub(center)

      orient.add(beluga)

      beluga.traverse((node) => {
        if (node.isMesh && node.material) {
          node.material.side = THREE.DoubleSide
          if ('metalness' in node.material) node.material.metalness = 0.25
          if ('roughness' in node.material) node.material.roughness = 0.35
          node.material.needsUpdate = true
        }
      })
    },
    null,
    (err) => console.error('Error loading beluga:', err)
  )

  // Interactive Hammer Throw Physics State
  const mouse = new THREE.Vector2(0, 0)
  let prevClientX = null
  let prevClientY = null
  let angularVelocity = 0.008 // Current orbital speed (rad/frame)
  const CRUISE_SPEED = 0.008
  const MAX_SPEED = 0.22
  let currentAngle = 0
  let currentBank = -0.15
  let rollAngle = 0
  let targetRoll = 0
  let boostTimer = 0

  // Progressive inclination tilt state
  const BASE_INCLINATION = 0.35
  const MAX_TILT_OFFSET = 0.18 // Subtle, very progressive angle shift (~10 deg)
  let targetInclination = BASE_INCLINATION

  const TWO_PI = Math.PI * 2
  const belugaWorldPos = new THREE.Vector3()

  // Trigger playful barrel roll stunt
  const triggerBarrelRoll = () => {
    targetRoll += Math.PI * 2
    boostTimer = 25
    angularVelocity = Math.min(MAX_SPEED, angularVelocity + 0.006)
    if (showHint.value) showHint.value = false
  }

  // Pointer event listeners
  const onPointerDown = () => {
    triggerBarrelRoll()
  }

  const onPointerMove = (e) => {
    if (!container.value) return
    const rect = container.value.getBoundingClientRect()
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1
    mouse.y = -(((e.clientY - rect.top) / rect.height) * 2 - 1)

    // Vertical input shifts the orbital inclination very subtly and progressively
    targetInclination = BASE_INCLINATION + mouse.y * MAX_TILT_OFFSET

    if (showHint.value) {
      setTimeout(() => { showHint.value = false }, 3500)
    }

    if (prevClientX !== null && prevClientY !== null) {
      const dx = e.clientX - prevClientX
      const dy = e.clientY - prevClientY

      // Position relative to screen center (Earth center)
      const relX = (e.clientX - rect.left) - rect.width * 0.5
      const relY = (e.clientY - rect.top) - rect.height * 0.5

      // 1. Horizontal swipe impulse: moving left-to-right (dx > 0) accelerates, right-to-left (dx < 0) decelerates
      const horizontalImpulse = dx * 0.000075

      // 2. Circular swirl torque (cooperative swirling adds momentum, counter swirling brakes)
      const swirlImpulse = -(relX * dy - relY * dx) * 0.0000055

      // Combined signed torque clamped per event to enforce physical mass
      const netTorque = THREE.MathUtils.clamp(horizontalImpulse + swirlImpulse, -0.002, 0.0016)

      // Apply signed torque: speed up or slow down smoothly towards CRUISE_SPEED
      angularVelocity = THREE.MathUtils.clamp(angularVelocity + netTorque, CRUISE_SPEED, MAX_SPEED)
    }

    prevClientX = e.clientX
    prevClientY = e.clientY
  }

  const onPointerLeave = () => {
    prevClientX = null
    prevClientY = null
  }

  window.addEventListener('pointerdown', onPointerDown)
  window.addEventListener('pointermove', onPointerMove)
  window.addEventListener('pointerleave', onPointerLeave)

  handleResize = () => {
    if (!container.value || !renderer) return
    const w = container.value.clientWidth
    const h = container.value.clientHeight
    camera.aspect = w / h
    camera.updateProjectionMatrix()
    renderer.setSize(w, h)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  }
  window.addEventListener('resize', handleResize)

  const loop = () => {
    earth.rotation.y += 0.002
    stars.rotation.y += 0.0003

    // 1. Smoothly update orbital inclination with very high damping for liquid progression
    orbitTilt.rotation.x = THREE.MathUtils.lerp(orbitTilt.rotation.x, targetInclination, 0.015)

    // 2. Beluga world position (for contrail trail)
    belugaRoot.getWorldPosition(belugaWorldPos)

    // 3. Momentum & Inertia Physics (Olympic Hammer Throw model)
    // Boost bonus from barrel roll clicks
    let boostBonus = 0
    if (boostTimer > 0) {
      boostBonus = (boostTimer / 25) * 0.004
      boostTimer--
    }

    // Natural momentum decay: coast with heavy kinetic inertia smoothly back to cruise speed
    angularVelocity = THREE.MathUtils.lerp(angularVelocity, CRUISE_SPEED, 0.008)

    // Apply angular momentum step to advance along the circular orbit
    const step = angularVelocity + boostBonus
    currentAngle = (currentAngle + step) % TWO_PI
    pivot.rotation.y = currentAngle

    // 3. Dynamic G-Force Banking & Contrail Intensity
    // As velocity builds up, centrifugal G-force makes the Beluga bank deeper into the turn!
    const velocityRatio = THREE.MathUtils.clamp((angularVelocity - CRUISE_SPEED) / (MAX_SPEED - CRUISE_SPEED), 0, 1)
    const gForceBank = -0.15 - velocityRatio * 0.75 // Bank up to ~52 degrees under heavy momentum
    currentBank = THREE.MathUtils.lerp(currentBank, gForceBank, 0.1)

    // Smooth barrel roll interpolation
    rollAngle = THREE.MathUtils.lerp(rollAngle, targetRoll, 0.12)
    orient.rotation.z = currentBank + rollAngle
    orient.rotation.x = boostBonus > 0 ? 0.04 : 0

    // Dynamic contrail glow based on kinetic energy
    trailMaterial.opacity = 0.45 + velocityRatio * 0.45

    // 4. Subtle camera parallax for depth
    camera.position.x = THREE.MathUtils.lerp(camera.position.x, mouse.x * 0.4, 0.04)
    camera.position.y = THREE.MathUtils.lerp(camera.position.y, mouse.y * 0.3, 0.04)
    camera.lookAt(0, 0, 0)

    // 5. Update jet contrail trail
    const posAttr = trailGeometry.attributes.position
    const arr = posAttr.array
    for (let i = TRAIL_LENGTH - 1; i > 0; i--) {
      arr[i * 3] = arr[(i - 1) * 3]
      arr[i * 3 + 1] = arr[(i - 1) * 3 + 1]
      arr[i * 3 + 2] = arr[(i - 1) * 3 + 2]
    }
    arr[0] = belugaWorldPos.x
    arr[1] = belugaWorldPos.y
    arr[2] = belugaWorldPos.z
    posAttr.needsUpdate = true

    renderer.render(scene, camera)
    frameId = requestAnimationFrame(loop)
  }
  loop()
})

onBeforeUnmount(() => {
  if (handleResize) {
    window.removeEventListener('resize', handleResize)
  }
  window.removeEventListener('pointerdown', onPointerDown)
  window.removeEventListener('pointermove', onPointerMove)
  window.removeEventListener('pointerleave', onPointerLeave)
  cancelAnimationFrame(frameId)
  renderer?.dispose()
})
</script>

<template>
  <div ref="container" class="scene-container">
    <transition name="fade">
      <div v-if="showHint" class="flight-hint">
        <span>✈️ Swipe vers la gauche pour accélérer • Swipe vers la droite pour ralentir • Cliquez pour faire un tonneau</span>
      </div>
    </transition>
  </div>
</template>

<style scoped>
.scene-container {
  width: 100%;
  height: 100%;
  position: absolute;
  inset: 0;
  overflow: hidden;
  cursor: grab;
  user-select: none;
}

.scene-container:active {
  cursor: grabbing;
}

.flight-hint {
  position: absolute;
  bottom: 28px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(15, 23, 42, 0.65);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.12);
  color: #e2e8f0;
  padding: 10px 22px;
  border-radius: 9999px;
  font-family: system-ui, -apple-system, sans-serif;
  font-size: 13px;
  font-weight: 500;
  letter-spacing: 0.3px;
  pointer-events: none;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.8s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>