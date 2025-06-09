<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const buttonRef = ref<HTMLElement | null>(null)
const isDragging = ref(false)
const posX = ref(0)
const posY = ref(0)
const startX = ref(0)
const startY = ref(0)
const velocityX = ref(0)
const velocityY = ref(0)
const rotation = ref(0)
const angularVelocity = ref(0)
const hasDropped = ref(false)
const lastTime = ref(0)
const animationId = ref(0)

// Физические параметры
const friction = 0.98
const bounce = 0.7
const gravity = 0.5
const buttonWidth = 120
const buttonHeight = 56
const rotationFriction = 0.96
const rotationMultiplier = 0.2

const handleScroll = () => {
  if (!hasDropped.value && window.scrollY > 50) {
    hasDropped.value = true
    posY.value = window.innerHeight - buttonHeight - 20
  }
}

const startDrag = (e: MouseEvent | TouchEvent) => {
  cancelAnimationFrame(animationId.value)
  isDragging.value = true
  velocityX.value = 0
  velocityY.value = 0
  angularVelocity.value = 0

  const clientX = e instanceof MouseEvent ? e.clientX : e.touches[0].clientX
  const clientY = e instanceof MouseEvent ? e.clientY : e.touches[0].clientY

  startX.value = clientX - posX.value
  startY.value = clientY - posY.value
}

const onDrag = (e: MouseEvent | TouchEvent) => {
  if (!isDragging.value) return

  e.preventDefault()
  const clientX = e instanceof MouseEvent ? e.clientX : e.touches[0].clientX
  const clientY = e instanceof MouseEvent ? e.clientY : e.touches[0].clientY

  // Вычисляем скорость для инерции
  const now = performance.now()
  const deltaTime = now - lastTime.value
  if (deltaTime > 0) {
    const newVelocityX = ((clientX - startX.value - posX.value) / deltaTime) * 1000
    const newVelocityY = ((clientY - startY.value - posY.value) / deltaTime) * 1000

    // Добавляем вращение в зависимости от движения
    angularVelocity.value = (newVelocityX - velocityX.value) * rotationMultiplier
    velocityX.value = newVelocityX
    velocityY.value = newVelocityY
  }
  lastTime.value = now

  posX.value = clientX - startX.value
  posY.value = clientY - startY.value
}

const stopDrag = () => {
  isDragging.value = false
  lastTime.value = performance.now()
  animate()
}

const checkBoundaries = () => {
  const windowWidth = window.innerWidth
  const windowHeight = window.innerHeight
  let hitBoundary = false

  // Проверка границ по X
  if (posX.value < 0) {
    posX.value = 0
    velocityX.value = -velocityX.value * bounce
    // Добавляем вращение при ударе о боковые границы
    angularVelocity.value = -velocityY.value * rotationMultiplier
    hitBoundary = true
  } else if (posX.value > windowWidth - buttonWidth) {
    posX.value = windowWidth - buttonWidth
    velocityX.value = -velocityX.value * bounce
    angularVelocity.value = velocityY.value * rotationMultiplier
    hitBoundary = true
  }

  // Проверка границ по Y
  if (posY.value < 0) {
    posY.value = 0
    velocityY.value = -velocityY.value * bounce
    angularVelocity.value = velocityX.value * rotationMultiplier
    hitBoundary = true
  } else if (posY.value > windowHeight - buttonHeight) {
    posY.value = windowHeight - buttonHeight
    velocityY.value = -velocityY.value * bounce
    // Добавляем вращение при ударе о нижнюю границу
    angularVelocity.value = -velocityX.value * rotationMultiplier * 2
    velocityX.value *= friction * 0.9
    hitBoundary = true
  }

  return hitBoundary
}

const animate = () => {
  if (isDragging.value) return

  const now = performance.now()
  const deltaTime = Math.min(now - lastTime.value, 50)
  lastTime.value = now

  // Применяем физику движения
  velocityY.value += gravity
  velocityX.value *= friction
  velocityY.value *= friction

  posX.value += (velocityX.value * deltaTime) / 1000
  posY.value += (velocityY.value * deltaTime) / 1000

  const hitBoundary = checkBoundaries()

  // Физика вращения
  angularVelocity.value *= rotationFriction
  rotation.value += (angularVelocity.value * deltaTime) / 1000

  // Нормализация угла вращения
  rotation.value = rotation.value % 360

  // Если скорость очень мала - останавливаем анимацию
  if (
    Math.abs(velocityX.value) < 0.01 &&
    Math.abs(velocityY.value) < 0.05 &&
    Math.abs(angularVelocity.value) < 0.01 &&
    posY.value >= window.innerHeight - buttonHeight
  ) {
    velocityX.value = 0
    velocityY.value = 0
    angularVelocity.value = 0
    // Выравниваем кнопку при остановке
    if (!hitBoundary) {
      rotation.value = Math.round(rotation.value / 90) * 90
    }
    return
  }

  animationId.value = requestAnimationFrame(animate)
}

onMounted(() => {
  // Начальная позиция по центру вверху
  posX.value = window.innerWidth / 2 - buttonWidth / 2
  posY.value = 20

  window.addEventListener('scroll', handleScroll)
  document.addEventListener('mousemove', onDrag)
  document.addEventListener('touchmove', onDrag, { passive: false })
  document.addEventListener('mouseup', stopDrag)
  document.addEventListener('touchend', stopDrag)
  window.addEventListener('resize', checkBoundaries)
})

onUnmounted(() => {
  cancelAnimationFrame(animationId.value)
  window.removeEventListener('scroll', handleScroll)
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('touchmove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
  document.removeEventListener('touchend', stopDrag)
  window.removeEventListener('resize', checkBoundaries)
})
</script>

<template>
  <div
    ref="buttonRef"
    class="button-drop"
    :style="{
      transform: `translate(${posX}px, ${posY}px) rotate(${rotation}deg)`,
      transition: hasDropped && !isDragging ? 'transform 0.3s ease-out' : 'none',
    }"
    @mousedown="startDrag"
    @touchstart="startDrag"
  >
    <button class="dropbtn">Click me</button>
  </div>
</template>

<style scoped>
.button-drop {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 1000;
  will-change: transform;
  width: 120px;
  transform-origin: center center;
}

.dropbtn {
  background-color: #245326;
  color: white;
  padding: 16px;
  font-size: 16px;
  border: none;
  cursor: grab;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  user-select: none;
  touch-action: none;
  width: 100%;
  text-align: center;
  transition: transform 0.1s;
}

.dropbtn:active {
  cursor: grabbing;
  transform: scale(0.95);
}

@media (max-width: 600px) {
  .button-drop {
    width: 100px;
  }
  .dropbtn {
    padding: 12px;
    font-size: 14px;
  }
}
</style>
