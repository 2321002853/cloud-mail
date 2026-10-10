<template>
  <div class="bloom-bg" aria-hidden="true">
    <canvas ref="canvasRef" class="bloom-canvas"></canvas>
    <div ref="particlesRef" class="bloom-particles"></div>
  </div>
</template>

<script setup>
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { useUiStore } from '@/store/ui.js'

const canvasRef = ref(null)
const particlesRef = ref(null)
const uiStore = useUiStore()

const motion = [
  { x: 0.45, y: 0.55, rx: 0.45, ry: 0.3, speed: 0.0003, phase: 0 },
  { x: 0.65, y: 0.40, rx: 0.3, ry: 0.4, speed: 0.0004, phase: 1.2 },
  { x: 0.30, y: 0.35, rx: 0.35, ry: 0.25, speed: 0.00025, phase: 2.5 },
  { x: 0.55, y: 0.50, rx: 0.15, ry: 0.12, speed: 0.0005, phase: 3.8 },
  { x: 0.20, y: 0.70, rx: 0.25, ry: 0.2, speed: 0.00035, phase: 5.0 },
  { x: 0.75, y: 0.65, rx: 0.2, ry: 0.28, speed: 0.00045, phase: 0.7 },
]

const lightColors = [
  ['rgba(126,232,196,0.82)', 'rgba(126,232,196,0)'],
  ['rgba(126,206,255,0.78)', 'rgba(126,206,255,0)'],
  ['rgba(186,214,92,0.72)', 'rgba(186,214,92,0)'],
  ['rgba(255,255,255,0.5)', 'rgba(255,255,255,0)'],
  ['rgba(120,214,214,0.7)', 'rgba(120,214,214,0)'],
  ['rgba(255,214,120,0.72)', 'rgba(255,214,120,0)'],
]

const darkColors = [
  ['rgba(48,168,148,0.72)', 'rgba(48,168,148,0)'],
  ['rgba(46,132,196,0.68)', 'rgba(46,132,196,0)'],
  ['rgba(132,164,64,0.55)', 'rgba(132,164,64,0)'],
  ['rgba(186,230,220,0.22)', 'rgba(186,230,220,0)'],
  ['rgba(42,150,160,0.6)', 'rgba(42,150,160,0)'],
  ['rgba(196,156,58,0.55)', 'rgba(196,156,58,0)'],
]

let width = 0
let height = 0
let frame = 0
let ctx = null

function resize() {
  const canvas = canvasRef.value
  if (!canvas) return
  width = canvas.width = window.innerWidth
  height = canvas.height = window.innerHeight
}

function currentLeaves() {
  const colors = uiStore.dark ? darkColors : lightColors
  return motion.map((leaf, index) => ({
    ...leaf,
    color1: colors[index][0],
    color2: colors[index][1],
  }))
}

function drawLeaves(time) {
  ctx.fillStyle = uiStore.dark ? '#0c1a1e' : '#e4f6f4'
  ctx.fillRect(0, 0, width, height)

  currentLeaves().forEach(leaf => {
    const dx = Math.sin(time * leaf.speed + leaf.phase) * 25
    const dy = Math.cos(time * leaf.speed * 0.7 + leaf.phase * 1.3) * 20
    const cx = leaf.x * width + dx
    const cy = leaf.y * height + dy
    const rx = leaf.rx * width
    const ry = leaf.ry * height
    const grad = ctx.createRadialGradient(cx, cy, 0, cx, cy, Math.max(rx, ry))
    grad.addColorStop(0, leaf.color1)
    grad.addColorStop(0.5, leaf.color1.replace(/[\d.]+\)$/, '0.3)'))
    grad.addColorStop(1, leaf.color2)

    ctx.fillStyle = grad
    ctx.beginPath()
    ctx.ellipse(cx, cy, rx, ry, Math.sin(time * 0.0001 + leaf.phase) * 0.3, 0, Math.PI * 2)
    ctx.fill()
  })
}

function draw(time) {
  ctx.filter = 'blur(50px)'
  drawLeaves(time)
  ctx.filter = 'none'
  frame = requestAnimationFrame(draw)
}

function paintParticles() {
  const root = particlesRef.value
  if (!root) return
  const hues = [158, 174, 196, 86, 46]
  const light = uiStore.dark ? 74 : 46
  const nodes = root.children
  for (let i = 0; i < nodes.length; i++) {
    const hue = hues[i % hues.length]
    nodes[i].style.background = `hsla(${hue}, 72%, ${light}%, 0.8)`
    nodes[i].style.boxShadow = `0 0 8px hsla(${hue}, 72%, ${light}%, 0.5)`
  }
}

function spawnParticles() {
  const root = particlesRef.value
  if (!root) return
  for (let i = 0; i < 40; i++) {
    const particle = document.createElement('span')
    particle.className = 'bloom-particle'
    const size = Math.random() * 4 + 2
    particle.style.left = Math.random() * 100 + '%'
    particle.style.width = size + 'px'
    particle.style.height = size + 'px'
    particle.style.opacity = String(Math.random() * 0.45 + 0.25)
    particle.style.animationDuration = (Math.random() * 15 + 10) + 's'
    particle.style.animationDelay = (Math.random() * 15) + 's'
    root.appendChild(particle)
  }
  paintParticles()
}

function onVisibility() {
  if (document.hidden) {
    cancelAnimationFrame(frame)
    return
  }
  frame = requestAnimationFrame(draw)
}

watch(() => uiStore.dark, paintParticles)

onMounted(() => {
  ctx = canvasRef.value.getContext('2d')
  resize()
  spawnParticles()
  window.addEventListener('resize', resize)
  document.addEventListener('visibilitychange', onVisibility)
  frame = requestAnimationFrame(draw)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(frame)
  window.removeEventListener('resize', resize)
  document.removeEventListener('visibilitychange', onVisibility)
})
</script>

<style>
.bloom-bg {
  position: fixed;
  inset: 0;
  z-index: 0;
  pointer-events: none;
  overflow: hidden;
}

.bloom-canvas,
.bloom-particles {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}

.bloom-particles {
  z-index: 1;
  overflow: hidden;
}

.bloom-particle {
  position: absolute;
  top: 110vh;
  border-radius: 50%;
  animation: bloom-float-up linear infinite;
  will-change: transform;
}

@keyframes bloom-float-up {
  0% {
    transform: translateY(0) translateX(0) scale(0);
    opacity: 0;
  }
  10% {
    opacity: 1;
    transform: translateY(-15vh) translateX(10px) scale(1);
  }
  50% {
    transform: translateY(-60vh) translateX(-15px) scale(1.1);
  }
  90% {
    opacity: 0.5;
  }
  100% {
    transform: translateY(-120vh) translateX(20px) scale(0.3);
    opacity: 0;
  }
}
</style>
