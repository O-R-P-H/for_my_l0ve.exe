<template>
  <div class="container" @touchstart="start" @touchend="end" @touchcancel="end">

    <!-- Статические эмодзи под наклоном -->
    <div class="static-emoji emoji-1">❤️</div>
    <div class="static-emoji emoji-2">💖</div>
    <div class="static-emoji emoji-3">💗</div>
    <div class="static-emoji emoji-4">💓</div>
    <div class="static-emoji emoji-5">💕</div>
    <div class="static-emoji emoji-6">💘</div>
    <div class="static-emoji emoji-7">💝</div>
    <div class="static-emoji emoji-8">✨</div>
    <div class="static-emoji emoji-9">⭐</div>
    <div class="static-emoji emoji-10">🌟</div>
    <div class="static-emoji emoji-11">🔥</div>
    <div class="static-emoji emoji-12">🌸</div>
    <div class="static-emoji emoji-13">🫶</div>
    <div class="static-emoji emoji-14">💞</div>
    <div class="static-emoji emoji-15">💋</div>
    <div class="static-emoji emoji-16">💌</div>

    <!-- Искры и эмодзи (динамические) -->
    <div class="particles">
      <div v-for="p in particles" :key="p.id" class="particle" :class="{ 'is-emoji': p.isEmoji }" :style="{
        left: p.x + 'px',
        top: p.y + 'px',
        '--dx': (p.dx * 0.5) + 'px',
        '--dy': (p.dy * 0.5) + 'px',
        '--scale-start': p.scaleStart,
        '--scale-end': p.scaleEnd,
        '--rotate-start': p.rotateStart + 'deg',
        '--rotate-end': p.rotateEnd + 'deg',
        fontSize: p.size + 'px',
        color: p.color,
        textShadow: p.glow ? `0 0 ${p.glow}px ${p.color}, 0 0 ${p.glow * 2}px rgba(255,255,255,0.5)` : 'none',
        background: p.isEmoji ? 'none' : p.color,
        boxShadow: p.isEmoji ? 'none' : `0 0 ${p.glow}px ${p.color}, 0 0 ${p.glow * 2}px rgba(255,255,255,0.3)`,
        opacity: p.opacity,
        zIndex: p.isEmoji ? 25 : 20,
        filter: p.isEmoji ? 'drop-shadow(0 0 8px currentColor)' : 'none'
      }">
        <span v-if="p.isEmoji">{{ p.emoji }}</span>
      </div>
    </div>

    <!-- BPM -->
    <div class="bpm" :class="{ 'fade-out': showFinal || showSad }">
      <span>❤️</span>
      <span class="bpm-value">{{ Math.round(bpm) }}</span>
      <span>BPM</span>
    </div>

    <!-- Статус - ИСПРАВЛЕН -->
    <div class="status" :class="{ 'status-touching': touching && !showFinal && !showSad }">{{ statusText }}</div>

    <!-- СЕРДЦЕ -->
    <div class="heart-wrapper" :class="{ 'fade-out': showFinal || showSad }">
      <div class="heart"
           ref="heart"
           :style="{
             transform: `rotate(45deg) scale(${heartScale})`,
             filter: `drop-shadow(0 0 ${glowIntensity}px rgba(255, 51, 102, 0.8))`
           }">
        <div class="heart-inner"></div>
      </div>
    </div>

    <!-- ГРУСТНЫЙ ЭКРАН -->
    <transition name="sad-slide">
      <div v-if="showSad" class="sad-screen" @touchstart.stop @touchend.stop @click.stop>
        <div class="sad-content">
          <div class="sad-image">
            <!-- 👇👇👇 ВСТАВЬ СВОЕ ГРУСТНОЕ ФОТО СЮДА 👇👇👇 -->
            <img src="./img/me_sad.png" alt="грустно" class="sad-img">
          </div>
          <h2 class="sad-title">Не отпускай! 😢</h2>
          <p class="sad-message">Держи сердечко крепче, чтобы оно забилось быстрее...</p>
          <button class="sad-reset" @touchstart.stop.prevent="reset" @click.stop.prevent="reset">Попробовать снова ❤️</button>
        </div>
      </div>
    </transition>

    <!-- ФИНАЛЬНЫЙ ЭКРАН -->
    <transition name="final-slide">
      <div v-if="showFinal" class="final" @touchstart.stop @touchend.stop @click.stop>
        <div class="final-content">
          <h2>Ты — мой ритм ❤️</h2>

          <!-- 👇👇👇 КОЛЛАЖ С ФОТО - ВСТАВЬ СВОИ ПУТИ В src 👇👇👇 -->
          <div class="collage">
            <div class="photo">
              <img src="./img/IMG_7572.JPG" alt="фото 1" class="photo-img">
            </div>
            <div class="photo">
              <img src="./img/IMG_7573.JPG" alt="фото 2" class="photo-img">
            </div>
            <div class="photo">
              <img src="./img/IMG_7543.PNG" alt="фото 4" class="photo-img">
            </div>
            <div class="photo">
              <img src="./img/IMG_7493.jpg" alt="фото 3" class="photo-img">
            </div>
          </div>

          <p class="message">{{ message }}</p>
          <button class="reset" @touchstart.stop.prevent="reset" @click.stop.prevent="reset">Ещё раз</button>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, nextTick } from 'vue'

// ========== TELEGRAM WEBAPP ==========
const tg = window.Telegram?.WebApp

// ========== СОСТОЯНИЕ ==========
const touching = ref(false)
const touchStartTime = ref(0)
const bpm = ref(65)
const heartScale = ref(1)
const heart = ref(null)
const showFinal = ref(false)
const showSad = ref(false)
const particles = ref([])
const glowIntensity = ref(20)
const message = ref('Спасибо, что держишь моё сердце. С тобой оно бьется чаще.')

let particleId = 0
let animationFrame = null
let beatAnimationFrame = null
let lastBeatTime = 0
let targetBPM = 65
let currentBPM = 65
let beatPhase = 0
let sadTimeout = null

// ========== КОНСТАНТЫ ==========
const MAX_BPM = 180
const FINAL_TIME = 40
const BASE_BPM = 65
const MAX_PARTICLES = 80
const SAD_DELAY = 500

// Эмодзи для частиц
const EMOJIS = ['❤️', '💖', '💗', '💓', '💕', '💘', '💝', '✨', '⭐', '🌟', '🔥', '🌸', '🫶']

// ========== HAPTIC FEEDBACK ==========
const hapticImpact = (style = 'light') => {
  if (tg?.HapticFeedback) {
    try {
      tg.HapticFeedback.impactOccurred(style)
    } catch (e) {}
  }
}

const hapticNotification = (type = 'success') => {
  if (tg?.HapticFeedback) {
    try {
      tg.HapticFeedback.notificationOccurred(type)
    } catch (e) {}
  }
}

const hapticSelection = () => {
  if (tg?.HapticFeedback) {
    try {
      tg.HapticFeedback.selectionChanged()
    } catch (e) {}
  }
}

const heartbeatHaptic = () => {
  if (!tg?.HapticFeedback) return

  if (currentBPM < 100) {
    hapticImpact('light')
  } else if (currentBPM < 140) {
    hapticImpact('medium')
    setTimeout(() => hapticImpact('light'), 30)
  } else {
    hapticImpact('heavy')
    setTimeout(() => hapticImpact('medium'), 20)
    setTimeout(() => hapticImpact('light'), 40)
  }
}

const touchHaptic = () => {
  if (tg?.HapticFeedback) {
    hapticImpact('light')
    hapticSelection()
  }
}

const finalHaptic = () => {
  if (tg?.HapticFeedback) {
    hapticNotification('success')
    setTimeout(() => hapticNotification('success'), 200)
    setTimeout(() => hapticNotification('success'), 400)
  }
}

const sadHaptic = () => {
  if (tg?.HapticFeedback) {
    hapticNotification('warning')
    hapticImpact('heavy')
  }
}

// ========== ВЫЧИСЛЯЕМЫЕ СВОЙСТВА ==========
const statusText = computed(() => {
  if (showFinal.value) return '❤️ Спасибо ❤️'
  if (showSad.value) return 'Не отпускай... 😢'
  if (!touching.value) return '👆 Коснись'

  // Когда касаются
  if (currentBPM < 85) return '😌 Спокойно'
  if (currentBPM < 110) return '💓 Чаще'
  if (currentBPM < 140) return '💗 Быстрее'
  return '💖 Сильнее'
})

// ========== ПЛАВНОЕ ОБНОВЛЕНИЕ BPM ==========
const updateBPMSmoothly = () => {
  if (!touching.value && !showFinal.value && !showSad.value) {
    targetBPM = BASE_BPM
  } else if (touching.value && !showFinal.value && !showSad.value) {
    const holdTime = (Date.now() - touchStartTime.value) / 1000
    const progress = Math.min(holdTime / FINAL_TIME, 1)
    targetBPM = BASE_BPM + (MAX_BPM - BASE_BPM) * Math.pow(progress, 1.1)

    if (holdTime >= FINAL_TIME) {
      final()
    }
  }

  const diff = targetBPM - currentBPM
  if (Math.abs(diff) > 0.1) {
    currentBPM += diff * 0.03
  } else {
    currentBPM = targetBPM
  }

  bpm.value = currentBPM
  glowIntensity.value = 20 + (currentBPM - BASE_BPM) * 1.2

  animationFrame = requestAnimationFrame(updateBPMSmoothly)
}

// ========== ПЛАВНАЯ ПУЛЬСАЦИЯ ==========
const updateBeatAnimation = () => {
  if (!heart.value || showFinal.value || showSad.value) {
    beatAnimationFrame = requestAnimationFrame(updateBeatAnimation)
    return
  }

  const now = Date.now()
  const beatInterval = 60000 / currentBPM
  const timeSinceLastBeat = now - lastBeatTime

  if (timeSinceLastBeat < beatInterval) {
    beatPhase = timeSinceLastBeat / beatInterval

    let scaleFactor

    if (beatPhase < 0.15) {
      const t = beatPhase / 0.15
      scaleFactor = 1 + (t * 0.28)
    } else if (beatPhase < 0.4) {
      scaleFactor = 1.28
    } else {
      const t = (beatPhase - 0.4) / 0.6
      scaleFactor = 1.28 - (t * 0.28)
    }

    if (touching.value) {
      heartScale.value = 1 + (scaleFactor - 1) * 1.4
    } else {
      heartScale.value = scaleFactor
    }

    if (touching.value && beatPhase < 0.05) {
      heartbeatHaptic()
    }

    if (touching.value && beatPhase > 0.1 && beatPhase < 0.2 && particles.value.length < MAX_PARTICLES) {
      const particleCount = Math.min(Math.floor(currentBPM / 15) + 2, 6)
      for (let i = 0; i < particleCount; i++) {
        setTimeout(() => createParticle(), i * 10)
      }
    }
  } else {
    beatPhase = 0
    heartScale.value = 1
    lastBeatTime = now
  }

  beatAnimationFrame = requestAnimationFrame(updateBeatAnimation)
}

// ========== СОЗДАНИЕ ЧАСТИЦ ==========
const createParticle = (explosion = false) => {
  if (particles.value.length >= MAX_PARTICLES) return
  if (!heart.value) return

  const rect = heart.value.getBoundingClientRect()
  const container = document.querySelector('.container').getBoundingClientRect()

  const centerX = rect.left + rect.width / 2 - container.left
  const centerY = rect.top + rect.height / 2 - container.top

  const angle = Math.random() * Math.PI * 2
  const distance = explosion ? Math.random() * 150 : 10 + Math.random() * 60

  const startX = centerX + Math.cos(angle) * distance * 0.2
  const startY = centerY + Math.sin(angle) * distance * 0.2

  const baseSpeed = explosion ? 350 : 150 + (currentBPM - BASE_BPM) * 1.5
  const speed = baseSpeed + Math.random() * 200
  const dx = Math.cos(angle) * speed
  const dy = Math.sin(angle) * speed

  const isEmoji = Math.random() > 0.5

  const id = particleId++

  if (isEmoji) {
    const emoji = EMOJIS[Math.floor(Math.random() * EMOJIS.length)]
    const size = 22 + Math.random() * 28

    particles.value.push({
      id,
      x: startX,
      y: startY,
      dx,
      dy,
      isEmoji: true,
      emoji,
      size,
      scaleStart: 0.7 + Math.random() * 0.8,
      scaleEnd: 0,
      rotateStart: Math.random() * 360,
      rotateEnd: (Math.random() * 540) - 270,
      color: `hsl(${Math.random() * 30 + 330}, 80%, 70%)`,
      glow: 12 + Math.random() * 15,
      opacity: 0.8 + Math.random() * 0.3
    })
  } else {
    const colors = ['#ff99aa', '#ffb3c6', '#ffd700', '#ff88aa', '#ff66aa', '#ff4488', '#ff3366']
    const color = colors[Math.floor(Math.random() * colors.length)]
    const size = 6 + Math.random() * 14

    particles.value.push({
      id,
      x: startX,
      y: startY,
      dx,
      dy,
      isEmoji: false,
      size,
      scaleStart: 0.8 + Math.random() * 0.7,
      scaleEnd: 0,
      rotateStart: 0,
      rotateEnd: 0,
      color,
      glow: 12 + Math.random() * 18,
      opacity: 0.7 + Math.random() * 0.4
    })
  }

  setTimeout(() => {
    particles.value = particles.value.filter(p => p.id !== id)
  }, 900)
}

// ========== ПОТОК ЧАСТИЦ ==========
let particleFlowInterval = null
const startParticleFlow = () => {
  if (particleFlowInterval) clearInterval(particleFlowInterval)

  particleFlowInterval = setInterval(() => {
    if (touching.value && !showFinal.value && !showSad.value && particles.value.length < MAX_PARTICLES) {
      const count = Math.min(Math.floor(currentBPM / 20) + 2, 5)
      for (let i = 0; i < count; i++) {
        setTimeout(() => createParticle(), i * 15)
      }
    }
  }, 200)
}

// ========== ОБРАБОТЧИКИ ==========
const start = (e) => {
  e.preventDefault()
  if (showFinal.value || showSad.value) return

  if (sadTimeout) {
    clearTimeout(sadTimeout)
    sadTimeout = null
  }

  touching.value = true
  touchStartTime.value = Date.now()

  touchHaptic()

  for (let i = 0; i < 25; i++) {
    setTimeout(() => createParticle(true), i * 2)
  }
}

const end = (e) => {
  e.preventDefault()

  if (touching.value && !showFinal.value && !showSad.value) {
    const holdTime = (Date.now() - touchStartTime.value) / 1000

    if (holdTime < FINAL_TIME) {
      sadTimeout = setTimeout(() => {
        showSad.value = true
        touching.value = false
        sadHaptic()
      }, SAD_DELAY)
    }
  }

  touching.value = false
}

// ========== ФИНАЛ ==========
const final = () => {
  if (sadTimeout) {
    clearTimeout(sadTimeout)
    sadTimeout = null
  }

  showFinal.value = true
  touching.value = false
  showSad.value = false

  for (let i = 0; i < 60; i++) {
    setTimeout(() => createParticle(true), i * 1.5)
  }

  finalHaptic()
}

// ========== СБРОС ==========
const reset = (e) => {
  if (e) {
    e.preventDefault()
    e.stopPropagation()
  }

  // Сбрасываем все состояния
  showFinal.value = false
  showSad.value = false
  touching.value = false

  // Сбрасываем BPM
  targetBPM = BASE_BPM
  currentBPM = BASE_BPM
  bpm.value = BASE_BPM

  // Сбрасываем визуальные эффекты
  heartScale.value = 1
  glowIntensity.value = 20

  // Сбрасываем тайминги
  lastBeatTime = Date.now()
  beatPhase = 0

  // Очищаем частицы
  particles.value = []

  // Очищаем таймер
  if (sadTimeout) {
    clearTimeout(sadTimeout)
    sadTimeout = null
  }
}

// ========== LIFECYCLE ==========
onMounted(() => {
  if (tg) {
    tg.ready()
    tg.expand()
  }

  nextTick(() => {
    lastBeatTime = Date.now()
    updateBPMSmoothly()
    updateBeatAnimation()
    startParticleFlow()

    setInterval(() => {
      if (!touching.value && !showFinal.value && !showSad.value && Math.random() > 0.8 && particles.value.length < MAX_PARTICLES / 2) {
        createParticle()
      }
    }, 1200)
  })
})

onUnmounted(() => {
  cancelAnimationFrame(animationFrame)
  cancelAnimationFrame(beatAnimationFrame)
  clearInterval(particleFlowInterval)
  if (sadTimeout) {
    clearTimeout(sadTimeout)
  }
})
</script>

<style scoped>
.container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: radial-gradient(circle at 50% 50%, #000000, #0a0812);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  -webkit-tap-highlight-color: transparent;
  touch-action: pan-y;
  user-select: none;
  overflow: hidden;
}

/* Статические эмодзи под наклоном */
.static-emoji {
  position: absolute;
  font-size: 24px;
  opacity: 0.15;
  pointer-events: none;
  z-index: 5;
  transform: rotate(-15deg);
  filter: drop-shadow(0 0 5px rgba(255, 51, 102, 0.3));
  animation: staticFloat 8s infinite ease-in-out;
}

@keyframes staticFloat {
  0%, 100% { transform: rotate(-15deg) translateY(0); }
  50% { transform: rotate(-15deg) translateY(-10px); }
}

.emoji-1 { bottom: 5%; left: 5%; font-size: 32px; animation-delay: 0s; }
.emoji-2 { bottom: 8%; left: 15%; font-size: 28px; animation-delay: 0.5s; }
.emoji-3 { bottom: 12%; left: 25%; font-size: 36px; animation-delay: 1s; }
.emoji-4 { bottom: 3%; left: 35%; font-size: 24px; animation-delay: 1.5s; }
.emoji-5 { bottom: 15%; left: 45%; font-size: 40px; animation-delay: 2s; }
.emoji-6 { bottom: 7%; left: 55%; font-size: 30px; animation-delay: 2.5s; }
.emoji-7 { bottom: 10%; left: 65%; font-size: 34px; animation-delay: 3s; }
.emoji-8 { bottom: 4%; left: 75%; font-size: 26px; animation-delay: 3.5s; }
.emoji-9 { bottom: 13%; left: 85%; font-size: 38px; animation-delay: 4s; }
.emoji-10 { bottom: 6%; left: 95%; font-size: 32px; animation-delay: 4.5s; }
.emoji-11 { bottom: 2%; left: 8%; font-size: 44px; animation-delay: 5s; }
.emoji-12 { bottom: 18%; left: 18%; font-size: 28px; animation-delay: 5.5s; }
.emoji-13 { bottom: 9%; left: 28%; font-size: 36px; animation-delay: 6s; }
.emoji-14 { bottom: 14%; left: 38%; font-size: 32px; animation-delay: 6.5s; }
.emoji-15 { bottom: 5%; left: 48%; font-size: 40px; animation-delay: 7s; }
.emoji-16 { bottom: 11%; left: 58%; font-size: 30px; animation-delay: 7.5s; }

.heart-wrapper {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  justify-content: center;
  align-items: center;
  pointer-events: none;
  transition: opacity 0.6s ease;
  z-index: 30;
}

.heart-wrapper.fade-out {
  opacity: 0;
}

.heart {
  position: relative;
  width: clamp(140px, 35vw, 180px);
  height: clamp(140px, 35vw, 180px);
  background: #ff3366;
  transform: rotate(45deg) scale(1);
  border-radius: 20px 20px 0 20px;
  z-index: 30;
  will-change: transform;
  cursor: pointer;
  pointer-events: auto;
  box-shadow: 0 15px 30px rgba(0,0,0,0.5);
  transition: filter 0.2s ease;
}

.heart::before,
.heart::after {
  content: '';
  position: absolute;
  width: 100%;
  height: 100%;
  background: #ff3366;
  border-radius: 50%;
}

.heart::before {
  left: -50%;
  top: 0;
}

.heart::after {
  left: 0;
  top: -50%;
}

.heart-inner {
  position: absolute;
  top: -15%;
  left: -15%;
  width: 130%;
  height: 130%;
  background: radial-gradient(circle at 30% 30%, rgba(255, 255, 255, 0.9), transparent 70%);
  border-radius: 50%;
  opacity: 0.5;
  pointer-events: none;
  mix-blend-mode: overlay;
}

.particles {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 20;
  overflow: hidden;
}

.particle {
  position: absolute;
  pointer-events: none;
  will-change: transform, opacity;
  animation: fly 0.9s cubic-bezier(0.15, 0.7, 0.3, 1) forwards;
  display: flex;
  justify-content: center;
  align-items: center;
  transform-origin: center;
}

.particle.is-emoji {
  background: none !important;
  box-shadow: none !important;
  filter: drop-shadow(0 0 8px currentColor) drop-shadow(0 0 15px rgba(255,255,255,0.3));
}

@keyframes fly {
  0% {
    transform: translate(0, 0) scale(var(--scale-start, 1)) rotate(var(--rotate-start, 0deg));
    opacity: 1;
  }
  100% {
    transform: translate(var(--dx), var(--dy)) scale(var(--scale-end, 0)) rotate(var(--rotate-end, 0deg));
    opacity: 0;
  }
}

.bpm {
  position: absolute;
  top: 10%;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(15, 15, 25, 0.8);
  backdrop-filter: blur(15px);
  padding: 14px 28px;
  border-radius: 60px;
  color: white;
  font-size: 1.5rem;
  font-weight: 600;
  border: 1px solid rgba(255, 255, 255, 0.15);
  z-index: 40;
  transition: opacity 0.6s ease, transform 0.6s ease;
  box-shadow: 0 6px 20px rgba(0,0,0,0.3);
}

.bpm.fade-out {
  opacity: 0;
  transform: translateX(-50%) translateY(-20px);
}

.bpm-value {
  font-weight: 700;
  color: #ff6680;
  min-width: 50px;
  text-align: center;
  text-shadow: 0 0 10px rgba(255,102,128,0.5);
}

/* ИСПРАВЛЕННЫЙ СТАТУС */
.status {
  position: absolute;
  top: 22%;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(15, 15, 25, 0.8);
  backdrop-filter: blur(15px);
  padding: 14px 32px;
  border-radius: 60px;
  color: white;
  font-size: 1.3rem;
  border: 1px solid rgba(255, 255, 255, 0.15);
  z-index: 40;
  white-space: nowrap;
  transition: opacity 0.6s ease, transform 0.6s ease, background 0.3s ease, box-shadow 0.3s ease;
  box-shadow: 0 6px 20px rgba(0,0,0,0.3);
  font-weight: 500;
}

.status.fade-out {
  opacity: 0;
  transform: translateX(-50%) translateY(-20px);
}

.status.status-touching {
  background: rgba(255, 51, 102, 0.25);
  border-color: rgba(255, 51, 102, 0.4);
  box-shadow: 0 0 35px rgba(255, 51, 102, 0.25);
}

/* Грустный экран */
.sad-screen {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 20px;
  background: radial-gradient(circle at 50% 50%, #1a1424, #0a0812);
  pointer-events: auto;
  box-sizing: border-box;
  touch-action: manipulation;
}

.sad-content {
  max-width: 400px;
  width: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: white;
  animation: sadPop 0.6s ease;
  pointer-events: auto;
}

@keyframes sadPop {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.sad-image {
  margin-bottom: 30px;
  animation: sadFloat 3s ease infinite;
  pointer-events: none;
}

@keyframes sadFloat {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}

.sad-img {
  width: 150px;
  height: 150px;
  object-fit: contain;
  filter: drop-shadow(0 0 30px rgba(255,51,102,0.3));
  pointer-events: none;
}

.sad-title {
  font-size: 2rem;
  margin-bottom: 20px;
  text-shadow: 0 0 20px #ff3366;
  font-weight: 700;
  text-align: center;
  pointer-events: none;
}

.sad-message {
  font-size: 1.2rem;
  text-align: center;
  background: rgba(255, 51, 102, 0.15);
  backdrop-filter: blur(15px);
  padding: 20px;
  border-radius: 30px;
  margin-bottom: 30px;
  max-width: 320px;
  border: 2px solid rgba(255, 51, 102, 0.2);
  line-height: 1.6;
  pointer-events: none;
}

.sad-reset {
  background: rgba(255, 51, 102, 0.15);
  border: 2px solid #ff3366;
  color: white;
  font-size: 1.2rem;
  padding: 16px 40px;
  border-radius: 60px;
  backdrop-filter: blur(10px);
  cursor: pointer;
  transition: all 0.3s;
  font-weight: 600;
  box-shadow: 0 8px 20px rgba(0,0,0,0.3);
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
  pointer-events: auto;
  position: relative;
  z-index: 1001;
}

.sad-reset:active {
  transform: scale(0.95);
  background: rgba(255, 51, 102, 0.3);
}

/* Финальный экран */
.final-slide-enter-active,
.sad-slide-enter-active {
  transition: all 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}

.final-slide-leave-active,
.sad-slide-leave-active {
  transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
}

.final-slide-enter-from,
.sad-slide-enter-from {
  opacity: 0;
  transform: scale(0.9);
}

.final-slide-enter-to,
.sad-slide-enter-to {
  opacity: 1;
  transform: scale(1);
}

.final-slide-leave-from,
.sad-slide-leave-from {
  opacity: 1;
  transform: scale(1);
}

.final-slide-leave-to,
.sad-slide-leave-to {
  opacity: 0;
  transform: scale(0.9);
}

.final {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 20px;
  background: radial-gradient(circle at 50% 50%, #1a1424, #0a0812);
  pointer-events: auto;
  box-sizing: border-box;
  touch-action: manipulation;
}

.final-content {
  max-width: 400px;
  width: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: white;
  animation: contentPop 0.8s ease 0.2s both;
  pointer-events: auto;
}

@keyframes contentPop {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.final h2 {
  font-size: 2rem;
  margin-bottom: 30px;
  text-shadow: 0 0 30px #ff3366, 0 0 60px rgba(255,51,102,0.5);
  font-weight: 700;
  text-align: center;
  line-height: 1.3;
  animation: titleGlow 2s ease infinite;
  pointer-events: none;
}

@keyframes titleGlow {
  0%, 100% { text-shadow: 0 0 30px #ff3366, 0 0 60px rgba(255,51,102,0.5); }
  50% { text-shadow: 0 0 40px #ff3366, 0 0 80px rgba(255,51,102,0.7); }
}

.collage {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 15px;
  width: 100%;
  max-width: 300px;
  margin-bottom: 30px;
  pointer-events: none;
}

.photo {
  aspect-ratio: 1;
  background: linear-gradient(145deg, #ff3366, #b31f4a);
  border-radius: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 3px solid rgba(255, 255, 255, 0.2);
  animation: photoPop 0.5s ease backwards;
  box-shadow: 0 10px 25px rgba(0,0,0,0.4);
  overflow: hidden;
  padding: 0;
}

.photo-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.photo:nth-child(1) { animation-delay: 0.1s; }
.photo:nth-child(2) { animation-delay: 0.2s; }
.photo:nth-child(3) { animation-delay: 0.3s; }
.photo:nth-child(4) { animation-delay: 0.4s; }

@keyframes photoPop {
  0% {
    opacity: 0;
    transform: scale(0.8) rotate(-5deg);
  }
  100% {
    opacity: 1;
    transform: scale(1) rotate(0deg);
  }
}

.message {
  font-size: 1.3rem;
  text-align: center;
  background: rgba(255, 51, 102, 0.15);
  backdrop-filter: blur(15px);
  padding: 25px;
  border-radius: 30px;
  margin-bottom: 30px;
  max-width: 320px;
  border: 2px solid rgba(255, 51, 102, 0.2);
  line-height: 1.6;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  animation: messageAppear 0.6s ease 0.5s both;
  pointer-events: none;
}

@keyframes messageAppear {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.reset {
  background: rgba(255, 51, 102, 0.15);
  border: 2px solid #ff3366;
  color: white;
  font-size: 1.2rem;
  padding: 16px 40px;
  border-radius: 60px;
  backdrop-filter: blur(10px);
  cursor: pointer;
  transition: all 0.3s;
  font-weight: 600;
  box-shadow: 0 8px 20px rgba(0,0,0,0.3);
  animation: buttonAppear 0.6s ease 0.7s both;
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
  pointer-events: auto;
  position: relative;
  z-index: 1001;
}

@keyframes buttonAppear {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.reset:active {
  transform: scale(0.95);
  background: rgba(255, 51, 102, 0.3);
  box-shadow: 0 5px 15px rgba(255,51,102,0.4);
}

@media (max-width: 480px) {
  .heart {
    width: 120px;
    height: 120px;
  }

  .status {
    font-size: 1rem;
    padding: 10px 24px;
    top: 20%;
  }

  .bpm {
    font-size: 1.2rem;
    padding: 12px 22px;
    top: 8%;
  }

  .final h2 {
    font-size: 1.6rem;
    margin-bottom: 25px;
  }

  .message {
    font-size: 1.1rem;
    padding: 20px;
  }

  .reset {
    font-size: 1rem;
    padding: 14px 32px;
  }

  .collage {
    gap: 12px;
    max-width: 260px;
  }

  .sad-title {
    font-size: 1.6rem;
  }

  .sad-message {
    font-size: 1rem;
    padding: 16px;
  }

  .sad-img {
    width: 120px;
    height: 120px;
  }

  .static-emoji {
    font-size: 20px;
  }
}
</style>