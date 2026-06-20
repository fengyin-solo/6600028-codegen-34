<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { useFluidStore } from '../store/fluid'

const store = useFluidStore()
const canvas = ref<HTMLCanvasElement | null>(null)

const W = 800
const H = 500
const mouseX = ref(0)
const mouseY = ref(0)
const isMouseDown = ref(false)
const isMouseInCanvas = ref(false)

function velocityToColor(speed: number): string {
  const maxSpeed = 200
  const t = Math.min(speed / maxSpeed, 1)
  const hue = (1 - t) * 240
  const sat = 80
  const light = 40 + t * 20
  return `hsl(${hue}, ${sat}%, ${light}%)`
}

function getCanvasCoords(e: MouseEvent) {
  if (!canvas.value) return { x: 0, y: 0 }
  const rect = canvas.value.getBoundingClientRect()
  const scaleX = W / rect.width
  const scaleY = H / rect.height
  return {
    x: (e.clientX - rect.left) * scaleX,
    y: (e.clientY - rect.top) * scaleY
  }
}

function drawVortexIndicator(ctx: CanvasRenderingContext2D, cx: number, cy: number, active: boolean) {
  const radius = store.vortexRadius
  const direction = store.vortexClockwise ? 1 : -1

  // Outer glow
  const gradient = ctx.createRadialGradient(cx, cy, 0, cx, cy, radius)
  gradient.addColorStop(0, active ? 'rgba(168, 85, 247, 0.3)' : 'rgba(168, 85, 247, 0.1)')
  gradient.addColorStop(0.7, active ? 'rgba(168, 85, 247, 0.15)' : 'rgba(168, 85, 247, 0.05)')
  gradient.addColorStop(1, 'rgba(168, 85, 247, 0)')
  ctx.fillStyle = gradient
  ctx.beginPath()
  ctx.arc(cx, cy, radius, 0, Math.PI * 2)
  ctx.fill()

  // Vortex circle
  ctx.strokeStyle = active ? 'rgba(192, 132, 252, 0.8)' : 'rgba(192, 132, 252, 0.4)'
  ctx.lineWidth = 2
  ctx.setLineDash([6, 4])
  ctx.beginPath()
  ctx.arc(cx, cy, radius, 0, Math.PI * 2)
  ctx.stroke()
  ctx.setLineDash([])

  // Rotating arrow indicators
  const arrowCount = 4
  const time = performance.now() / 1000
  const rotationSpeed = active ? 3 : 1
  const baseAngle = time * rotationSpeed * direction

  for (let i = 0; i < arrowCount; i++) {
    const angle = baseAngle + (i / arrowCount) * Math.PI * 2
    const r = radius * 0.7
    const ax = cx + Math.cos(angle) * r
    const ay = cy + Math.sin(angle) * r

    const tangentAngle = angle + Math.PI / 2 * direction
    const arrowLength = 12

    ctx.strokeStyle = active ? 'rgba(216, 180, 254, 0.9)' : 'rgba(216, 180, 254, 0.5)'
    ctx.lineWidth = 2
    ctx.beginPath()
    ctx.moveTo(ax, ay)
    ctx.lineTo(
      ax + Math.cos(tangentAngle) * arrowLength,
      ay + Math.sin(tangentAngle) * arrowLength
    )
    ctx.stroke()

    // Arrow head
    const headLength = 5
    ctx.beginPath()
    ctx.moveTo(
      ax + Math.cos(tangentAngle) * arrowLength,
      ay + Math.sin(tangentAngle) * arrowLength
    )
    ctx.lineTo(
      ax + Math.cos(tangentAngle - 0.5) * headLength + Math.cos(tangentAngle) * (arrowLength - headLength),
      ay + Math.sin(tangentAngle - 0.5) * headLength + Math.sin(tangentAngle) * (arrowLength - headLength)
    )
    ctx.moveTo(
      ax + Math.cos(tangentAngle) * arrowLength,
      ay + Math.sin(tangentAngle) * arrowLength
    )
    ctx.lineTo(
      ax + Math.cos(tangentAngle + 0.5) * headLength + Math.cos(tangentAngle) * (arrowLength - headLength),
      ay + Math.sin(tangentAngle + 0.5) * headLength + Math.sin(tangentAngle) * (arrowLength - headLength)
    )
    ctx.stroke()
  }

  // Center point
  ctx.fillStyle = active ? '#c084fc' : 'rgba(192, 132, 252, 0.5)'
  ctx.beginPath()
  ctx.arc(cx, cy, 4, 0, Math.PI * 2)
  ctx.fill()
}

function draw() {
  const ctx = canvas.value?.getContext('2d')
  if (!ctx) return

  // Clear
  ctx.fillStyle = '#0c1222'
  ctx.fillRect(0, 0, W, H)

  // Draw boundary walls
  ctx.strokeStyle = '#475569'
  ctx.lineWidth = 3
  ctx.strokeRect(2, 2, W - 4, H - 4)

  // Draw grid (faint)
  ctx.strokeStyle = '#1e293b'
  ctx.lineWidth = 0.3
  for (let x = 0; x < W; x += 50) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, H)
    ctx.stroke()
  }
  for (let y = 0; y < H; y += 50) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(W, y)
    ctx.stroke()
  }

  if (!store.engine) return

  // Draw density heatmap background (low-res)
  const gridSize = 20
  const gw = Math.ceil(W / gridSize)
  const gh = Math.ceil(H / gridSize)
  const densityGrid = new Float32Array(gw * gh)
  for (const p of store.engine.particles) {
    const gx = Math.floor(p.x / gridSize)
    const gy = Math.floor(p.y / gridSize)
    if (gx >= 0 && gx < gw && gy >= 0 && gy < gh) {
      densityGrid[gy * gw + gx] += p.density
    }
  }
  const maxDens = Math.max(...densityGrid, 1)
  for (let gy = 0; gy < gh; gy++) {
    for (let gx = 0; gx < gw; gx++) {
      const d = densityGrid[gy * gw + gx]
      if (d > 0) {
        const alpha = Math.min(d / maxDens * 0.15, 0.15)
        ctx.fillStyle = `rgba(59, 130, 246, ${alpha})`
        ctx.fillRect(gx * gridSize, gy * gridSize, gridSize, gridSize)
      }
    }
  }

  // Draw particles
  const particles = store.engine.particles
  for (const p of particles) {
    const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy)
    const color = velocityToColor(speed)
    const radius = 4

    ctx.beginPath()
    ctx.arc(p.x, p.y, radius, 0, Math.PI * 2)
    ctx.fillStyle = color
    ctx.fill()
  }

  // Draw active vortex from store (drag mode)
  if (store.vortexActive && store.interactionMode === 'vortex') {
    drawVortexIndicator(ctx, store.vortexX, store.vortexY, true)
  }

  // Draw hover vortex indicator
  if (isMouseInCanvas.value && !store.vortexActive && store.interactionMode === 'vortex') {
    drawVortexIndicator(ctx, mouseX.value, mouseY.value, false)
  }

  // FPS overlay
  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)'
  ctx.fillRect(W - 80, 5, 75, 22)
  ctx.fillStyle = '#22c55e'
  ctx.font = 'bold 12px monospace'
  ctx.fillText(`FPS: ${store.fps}`, W - 74, 20)

  // Frame count
  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)'
  ctx.fillRect(W - 120, 30, 115, 22)
  ctx.fillStyle = '#94a3b8'
  ctx.font = '11px monospace'
  ctx.fillText(`Frame: ${store.frameCount}`, W - 114, 44)

  // Mode indicator
  if (store.interactionMode === 'vortex') {
    ctx.fillStyle = 'rgba(147, 51, 234, 0.6)'
    ctx.fillRect(5, 5, 75, 22)
    ctx.fillStyle = '#e9d5ff'
    ctx.font = 'bold 11px monospace'
    ctx.fillText('涡旋模式', 12, 20)
  }
}

let raf: number | null = null
function animate() {
  draw()
  raf = requestAnimationFrame(animate)
}

function onMouseMove(e: MouseEvent) {
  const coords = getCanvasCoords(e)
  mouseX.value = coords.x
  mouseY.value = coords.y
  if (isMouseDown.value && store.interactionMode === 'vortex') {
    store.updateVortexPosition(coords.x, coords.y)
  }
}

function onMouseDown(e: MouseEvent) {
  if (!store.engine || !canvas.value) return
  const coords = getCanvasCoords(e)
  isMouseDown.value = true

  if (store.interactionMode === 'vortex') {
    store.startVortex(coords.x, coords.y)
  } else {
    store.engine.applyImpulse(coords.x, coords.y, 300)
  }
}

function onMouseUp() {
  isMouseDown.value = false
  if (store.interactionMode === 'vortex') {
    store.stopVortex()
  }
}

function onMouseLeave() {
  isMouseInCanvas.value = false
  isMouseDown.value = false
  if (store.vortexActive) {
    store.stopVortex()
  }
}

function onMouseEnter() {
  isMouseInCanvas.value = true
}

onMounted(() => {
  animate()
})

onUnmounted(() => {
  if (raf) cancelAnimationFrame(raf)
})
</script>

<template>
  <div class="relative">
    <canvas
      ref="canvas"
      :width="W"
      :height="H"
      class="rounded-lg border border-gray-700 w-full max-w-[800px] select-none"
      :class="store.interactionMode === 'vortex' ? 'cursor-grab active:cursor-grabbing' : 'cursor-crosshair'"
      @mousedown="onMouseDown"
      @mouseup="onMouseUp"
      @mousemove="onMouseMove"
      @mouseleave="onMouseLeave"
      @mouseenter="onMouseEnter"
    />
  </div>
</template>
