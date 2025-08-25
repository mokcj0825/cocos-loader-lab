<script setup>
import { onMounted, onUnmounted } from 'vue'

let touchStartY = 0
let touchEndY = 0
let swipeCount = 0
let lastSwipeTime = 0
let lastSwipeDirection = ''
let isTouchActive = false
let isAlertTriggered = false

const handleDoubleSwipe = (event) => {
  if (!isAlertTriggered) {
    console.log('Triggering double swipe alert')
    isAlertTriggered = true
    alert('swipe up/down')
    // Reset flag after a short delay
    setTimeout(() => {
      isAlertTriggered = false
    }, 1000)
  } else {
    console.log('Alert already triggered, ignoring duplicate')
  }
}

// Listen for the custom onDoubleSwipe event
window.addEventListener('onDoubleSwipe', handleDoubleSwipe)

const handleTouchStart = (e) => {
  if (isTouchActive) return // Prevent multiple touch starts
  
  console.log('Touch start detected:', e.touches[0].clientY)
  touchStartY = e.touches[0].clientY
  isTouchActive = true
}

const handleTouchEnd = (e) => {
  if (!isTouchActive) return // Only process if touch was active
  
  console.log('Touch end detected:', e.changedTouches[0].clientY)
  touchEndY = e.changedTouches[0].clientY
  const currentTime = Date.now()
  const swipeDistance = touchStartY - touchEndY
  const minSwipeDistance = 50
  const timeWindow = 800
  
  console.log('Swipe distance:', swipeDistance, 'Min distance:', minSwipeDistance)
  
  if (Math.abs(swipeDistance) > minSwipeDistance) {
    const direction = swipeDistance > 0 ? 'up' : 'down'
    console.log('Swipe direction:', direction)
    
    // Check if this is a valid second swipe
    if (swipeCount === 1 && 
        direction === lastSwipeDirection && 
        (currentTime - lastSwipeTime) < timeWindow) {
      
      // This is the SECOND swipe - trigger alert and block event
      console.log('SECOND SWIPE DETECTED - TRIGGERING ALERT')
      e.preventDefault()
      e.stopPropagation()
      
      console.log('Dispatching onDoubleSwipe event')
      const event = new CustomEvent('onDoubleSwipe', {
        detail: { direction: direction }
      })
      window.dispatchEvent(event)
      
      // Reset for next sequence
      swipeCount = 0
      lastSwipeDirection = ''
      lastSwipeTime = 0
      
    } else {
      // This is the FIRST swipe or a new sequence - let it pass through
      console.log('FIRST SWIPE - LETTING IT PASS THROUGH')
      swipeCount = 1
      lastSwipeDirection = direction
      lastSwipeTime = currentTime
      
      // Allow the first swipe to pass through to the iframe
      // by not calling preventDefault() or stopPropagation()
    }
  } else {
    console.log('Swipe distance too small, ignoring')
  }
  
  // Reset touch active state
  isTouchActive = false
}

onMounted(() => {
  console.log('Adding touch event listeners')
  // Only use the overlay template listeners to avoid multiple event handling
})

onUnmounted(() => {
  console.log('Removing touch event listeners')
  // No need to remove listeners since they're template-bound
})
</script>

<template>
  <main>
    <!-- Transparent overlay positioned above the iframe -->
    <div 
      class="touch-overlay"
      @touchstart="handleTouchStart"
      @touchend="handleTouchEnd">
    </div>
    
    <iframe 
        class="game-frame"
        src="http://localhost:8000/index.html"
        frameborder="0"
        allowfullscreen>
    </iframe>
  </main>
</template>

<style scoped>
main {
  overflow: hidden;
  height: 100vh;
  position: relative;
}

.touch-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10;
  background: transparent;
  pointer-events: auto;
}

.game-frame {
  width: 100%;
  height: 100vh;
  border: none;
  position: relative;
  z-index: 1;
}
</style>
