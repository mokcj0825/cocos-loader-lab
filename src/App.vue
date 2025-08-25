<script setup>
import { onMounted, onUnmounted, ref } from 'vue'

let touchStartY = 0
let touchEndY = 0
let swipeCount = 0
let lastSwipeTime = 0
let lastSwipeDirection = ''
let isAlertTriggered = false

const iframeRef = ref(null)
let attachedIframeDoc = null
let iframeLoadHandler = null

const handleDoubleSwipe = (event) => {
  if (!isAlertTriggered) {
    const direction = event?.detail?.direction || ''
    console.log('Triggering double swipe alert with direction:', direction)
    isAlertTriggered = true
    alert(`Swipe ${direction}`)
    // Reset flag after a short delay
    setTimeout(() => {
      isAlertTriggered = false
    }, 1000)
  } else {
    console.log('Alert already triggered, ignoring duplicate')
  }
}

const handleTouchStart = (e) => {
  let y = null
  if (e.touches && e.touches[0] && typeof e.touches[0].clientY === 'number') {
    y = e.touches[0].clientY
  } else if (typeof e.clientY === 'number') {
    y = e.clientY
  }
  if (y === null) return
  console.log('Touch/Pointer start detected:', y)
  touchStartY = y
}

const handleTouchEnd = (e) => {
  let y = null
  if (e.changedTouches && e.changedTouches[0] && typeof e.changedTouches[0].clientY === 'number') {
    y = e.changedTouches[0].clientY
  } else if (typeof e.clientY === 'number') {
    y = e.clientY
  }
  if (y === null) return
  console.log('Touch/Pointer end detected:', y)
  touchEndY = y
  const currentTime = Date.now()
  const swipeDistance = touchStartY - touchEndY
  const minSwipeDistance = 30
  const timeWindow = 1200
  
  console.log('Swipe distance:', swipeDistance, 'Min distance:', minSwipeDistance)
  
  if (Math.abs(swipeDistance) > minSwipeDistance) {
    const direction = swipeDistance > 0 ? 'up' : 'down'
    console.log('Swipe direction:', direction)
    
    // Check if this is a valid second swipe
    if (swipeCount === 1 && 
        direction === lastSwipeDirection && 
        (currentTime - lastSwipeTime) < timeWindow) {
      
      // This is the SECOND swipe - trigger alert without blocking events
      console.log('SECOND SWIPE DETECTED - TRIGGERING ALERT')
      handleDoubleSwipe({ detail: { direction } })
      
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
      
      // Allow the swipe to pass through to the iframe by not intercepting
    }
  } else {
    console.log('Swipe distance too small, ignoring')
  }
}

onMounted(() => {
  console.log('Adding touch event listeners')
  // Fallback listeners on parent window for swipes outside iframe
  window.addEventListener('touchstart', handleTouchStart, { passive: true, capture: true })
  window.addEventListener('touchend', handleTouchEnd, { passive: true, capture: true })

  // Attach passive listeners inside the iframe document (same-origin only)
  const attachToIframe = () => {
    try {
      if (!iframeRef.value) return
      const doc = iframeRef.value.contentDocument || iframeRef.value.contentWindow?.document
      if (!doc || attachedIframeDoc === doc) return
      // Detach from previous doc if any
      if (attachedIframeDoc) {
        attachedIframeDoc.removeEventListener('touchstart', handleTouchStart)
        attachedIframeDoc.removeEventListener('touchend', handleTouchEnd)
      }
      doc.addEventListener('touchstart', handleTouchStart, { passive: true, capture: true })
      doc.addEventListener('touchend', handleTouchEnd, { passive: true, capture: true })
      attachedIframeDoc = doc
      console.log('Attached touch listeners inside iframe document')
    } catch (err) {
      console.warn('Unable to access iframe document (likely cross-origin):', err)
    }
  }

  iframeLoadHandler = attachToIframe
  if (iframeRef.value) {
    iframeRef.value.addEventListener('load', iframeLoadHandler)
    // Try immediate attach in case iframe is already loaded
    attachToIframe()
  }
})

onUnmounted(() => {
  console.log('Removing touch event listeners')
  window.removeEventListener('touchstart', handleTouchStart, { capture: true })
  window.removeEventListener('touchend', handleTouchEnd, { capture: true })
  if (iframeRef.value && iframeLoadHandler) {
    iframeRef.value.removeEventListener('load', iframeLoadHandler)
  }
  if (attachedIframeDoc) {
    attachedIframeDoc.removeEventListener('touchstart', handleTouchStart, { capture: true })
    attachedIframeDoc.removeEventListener('touchend', handleTouchEnd, { capture: true })
    attachedIframeDoc = null
  }
})
</script>

<template>
  <main>
    <!-- Transparent overlay positioned above the iframe -->
    <div 
      class="touch-overlay"
      >
    </div>
    
    <iframe 
        class="game-frame"
        ref="iframeRef"
        src="/game/index.html"
        title="Game Frame"
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
  pointer-events: none; /* Start with pointer events disabled */
}

.game-frame {
  width: 100%;
  height: 100vh;
  border: none;
  position: relative;
  z-index: 1;
}
</style>
