<script setup>
import { ref, computed } from 'vue'
import TaskCard from './TaskCard.vue'

const props = defineProps({
  year: Number,
  tasks: Array
})

const emit = defineEmits(['edit', 'delete', 'click', 'updateTask'])

const draggingTask = ref(null)
const resizingTask = ref(null)
const resizeDirection = ref(null)
const previewStartMonth = ref(null)
const previewEndMonth = ref(null)
const isResizing = ref(false)
const hoverMonth = ref(null)

const months = [
  'Januari', 'Februari', 'Maart', 'April', 'Mei', 'Juni',
  'Juli', 'Augustus', 'September', 'Oktober', 'November', 'December'
]

// Bereken het aantal dagen per maand voor dit jaar
const monthLengths = computed(() => {
  return months.map((_, index) => {
    const date = new Date(props.year, index + 1, 0) // Laatste dag van de maand
    return date.getDate()
  })
})

// Bereken grid positie voor elke taak
function getTaskStyle(task) {
  // Als we deze taak aan het resizen zijn, gebruik preview waarden
  const isThisTaskResizing = isResizing.value && resizingTask.value && 
    (resizingTask.value.id === task.id || resizingTask.value.originalId === task.id)
  
  let startMonth, endMonth
  
  if (isThisTaskResizing && previewStartMonth.value !== null && previewEndMonth.value !== null) {
    startMonth = previewStartMonth.value + 1 // Convert to 1-based
    endMonth = previewEndMonth.value + 1
  } else {
    const startDate = new Date(task.startDate)
    const endDate = new Date(task.endDate)
    
    // Bepaal start en eind maand (1-based)
    startMonth = startDate.getMonth() + 1
    endMonth = endDate.getMonth() + 1
    const startYear = startDate.getFullYear()
    const endYear = endDate.getFullYear()
    
    // Als task in dit jaar begint maar doorloopt naar volgend jaar
    if (startYear === props.year && endYear > props.year) {
      startMonth = startMonth
      endMonth = 12
    }
    // Als task vanuit vorig jaar doorloopt
    else if (startYear < props.year && endYear === props.year) {
      startMonth = 1
      endMonth = endMonth
    }
  }
  
  const style = {
    gridColumnStart: startMonth,
    gridColumnEnd: endMonth + 1,
    gridRow: task.row + 1
  }
  
  // Voeg visuele feedback toe tijdens resize
  if (isThisTaskResizing) {
    style.opacity = '0.8'
    style.border = '3px solid #4CAF50'
    style.boxShadow = '0 4px 12px rgba(76, 175, 80, 0.4)'
  }
  
  return style
}

// Bereken maand markeringen
const monthMarkers = computed(() => {
  return months.map((month, index) => {
    const date = new Date(props.year, index, 1)
    const yearStart = new Date(props.year, 0, 1)
    const yearEnd = new Date(props.year + 1, 0, 1)
    const totalDays = (yearEnd - yearStart) / (1000 * 60 * 60 * 24)
    const dayOfYear = (date - yearStart) / (1000 * 60 * 60 * 24)
    
    return {
      month,
      position: (dayOfYear / totalDays) * 100
    }
  })
})

// Groepeer taken per rij
const tasksByRow = computed(() => {
  const rows = {}
  props.tasks.forEach(task => {
    if (!rows[task.row]) {
      rows[task.row] = []
    }
    rows[task.row].push(task)
  })
  return rows
})

const maxRow = computed(() => {
  return Math.max(...props.tasks.map(t => t.row), 0)
})

const hasNextYear = computed(() => {
  // Check of er taken zijn die doorlopen naar volgend jaar
  return props.tasks.some(task => {
    const endYear = new Date(task.endDate).getFullYear()
    return endYear > props.year
  })
})

function scrollToNextYear() {
  // Scroll naar de volgende tijdlijn sectie
  const currentSection = document.querySelector(`[data-year="${props.year}"]`)
  if (currentSection) {
    const nextSection = currentSection.nextElementSibling
    if (nextSection) {
      nextSection.scrollIntoView({ behavior: 'smooth', block: 'start' })
    }
  }
}

// Drag and Drop handlers - werkende versie
function handleDragStart(task, event) {
  console.log('Drag start:', task.title)
  draggingTask.value = task
  event.dataTransfer.effectAllowed = 'move'
  event.dataTransfer.setData('text/plain', task.id)
}

function handleDragEnd(event) {
  console.log('Drag end')
  draggingTask.value = null
  hoverMonth.value = null
}

function handleDragOver(event, monthIndex) {
  if (!draggingTask.value) return
  event.preventDefault()
  event.dataTransfer.dropEffect = 'move'
  hoverMonth.value = monthIndex
}

function handleDragLeave() {
  hoverMonth.value = null
}

function handleDrop(event, targetMonth) {
  event.preventDefault()
  event.stopPropagation()
  
  console.log('Drop on month:', targetMonth + 1)
  
  if (!draggingTask.value) {
    console.log('No dragging task')
    return
  }
  
  const task = draggingTask.value
  const oldStartDate = new Date(task.startDate)
  const oldEndDate = new Date(task.endDate)
  
  // Bereken hoeveel maanden de taak duurt
  const startMonth = oldStartDate.getMonth()
  const endMonth = oldEndDate.getMonth()
  const startYear = oldStartDate.getFullYear()
  const endYear = oldEndDate.getFullYear()
  const durationInMonths = (endYear - startYear) * 12 + (endMonth - startMonth)
  
  // Nieuwe startdatum = eerste dag van target maand
  const newStartDate = new Date(props.year, targetMonth, 1)
  // Nieuwe einddatum = laatste dag van (target maand + duur)
  const newEndMonth = targetMonth + durationInMonths
  const newEndDate = new Date(props.year, newEndMonth + 1, 0) // Laatste dag van de maand
  
  console.log('Updating:', task.title, 'to', newStartDate.toLocaleDateString(), '-', newEndDate.toLocaleDateString())
  
  emit('updateTask', {
    ...task,
    startDate: newStartDate.toISOString().split('T')[0],
    endDate: newEndDate.toISOString().split('T')[0]
  })
  
  draggingTask.value = null
  hoverMonth.value = null
}

// Resize handlers met live preview
function handleResizeStart(task, direction, event) {
  event.stopPropagation()
  event.preventDefault()
  
  resizingTask.value = task
  resizeDirection.value = direction
  isResizing.value = true
  
  // Initiële preview waarden
  const startDate = new Date(task.startDate)
  const endDate = new Date(task.endDate)
  previewStartMonth.value = startDate.getMonth()
  previewEndMonth.value = endDate.getMonth()
  
  document.addEventListener('mousemove', handleResizeMove)
  document.addEventListener('mouseup', handleResizeEnd)
  document.body.style.cursor = 'ew-resize'
}

function handleResizeMove(event) {
  if (!resizingTask.value || !isResizing.value) return
  
  const tasksGrid = document.querySelector('.tasks-grid')
  if (!tasksGrid) return
  
  const rect = tasksGrid.getBoundingClientRect()
  const mouseX = event.clientX - rect.left
  const gridWidth = rect.width
  
  // Bereken welke maand (0-11)
  let targetMonth = Math.floor((mouseX / gridWidth) * 12)
  targetMonth = Math.max(0, Math.min(11, targetMonth))
  
  const task = resizingTask.value
  const currentStartDate = new Date(task.startDate)
  const currentEndDate = new Date(task.endDate)
  const currentStartMonth = currentStartDate.getMonth()
  const currentEndMonth = currentEndDate.getMonth()
  
  if (resizeDirection.value === 'left') {
    // Resize startdatum
    if (targetMonth < currentEndMonth) {
      previewStartMonth.value = targetMonth
      previewEndMonth.value = currentEndMonth
    }
  } else if (resizeDirection.value === 'right') {
    // Resize einddatum
    if (targetMonth > currentStartMonth) {
      previewStartMonth.value = currentStartMonth
      previewEndMonth.value = targetMonth
    }
  }
}

function handleResizeEnd(event) {
  if (!resizingTask.value || !isResizing.value) {
    cleanup()
    return
  }
  
  const task = resizingTask.value
  
  if (previewStartMonth.value !== null && previewEndMonth.value !== null) {
    // Eerste dag van startmaand
    const newStartDate = new Date(props.year, previewStartMonth.value, 1)
    // Laatste dag van eindmaand
    const newEndDate = new Date(props.year, previewEndMonth.value + 1, 0)
    
    console.log('Resize:', task.title, 'van', newStartDate.toLocaleDateString(), 'tot', newEndDate.toLocaleDateString())
    
    emit('updateTask', {
      ...task,
      startDate: newStartDate.toISOString().split('T')[0],
      endDate: newEndDate.toISOString().split('T')[0]
    })
  }
  
  cleanup()
}

function cleanup() {
  document.removeEventListener('mousemove', handleResizeMove)
  document.removeEventListener('mouseup', handleResizeEnd)
  document.body.style.cursor = ''
  resizingTask.value = null
  resizeDirection.value = null
  previewStartMonth.value = null
  previewEndMonth.value = null
  isResizing.value = false
}
</script>

<template>
  <div class="timeline-container" :data-year="year">
    <h2 class="timeline-year">Tijdlijn {{ year }}</h2>
    <div class="timeline-header">
      <div class="month-markers">
        <div 
          v-for="marker in monthMarkers" 
          :key="marker.month"
          class="month-marker"
          :style="{ left: marker.position + '%' }"
        >
          <div class="month-line"></div>
          <div class="month-label">{{ marker.month }}</div>
        </div>
        <button v-if="hasNextYear" @click="scrollToNextYear" class="chevron-next" title="Ga naar volgend jaar">
          ❯
        </button>
      </div>
    </div>
    
    <div class="timeline-grid" :style="{ height: (maxRow + 1) * 120 + 150 + 'px' }">
      <!-- Drop zones voor elke maand (over hele hoogte) -->
      <div class="drop-zones">
        <div 
          v-for="(length, index) in monthLengths" 
          :key="'drop-' + index" 
          class="drop-zone"
          :class="{ 'drop-zone-hover': hoverMonth === index && draggingTask }"
          @dragover.prevent="handleDragOver($event, index)"
          @dragleave="handleDragLeave"
          @drop="handleDrop($event, index)"
        >
          <div v-if="hoverMonth === index && draggingTask" class="drop-indicator">
            Drop hier
          </div>
        </div>
      </div>
      
      <!-- Grid achtergrond met maand kolommen -->
      <div class="grid-background">
        <div 
          v-for="(length, index) in monthLengths" 
          :key="index" 
          class="grid-column"
        ></div>
      </div>
      
      <!-- Taken in grid -->
      <div class="tasks-grid">
        <TaskCard
          v-for="task in tasks"
          :key="task.id"
          :task="task"
          :style="getTaskStyle(task)"
          @click="emit('click', task)"
          @edit="emit('edit', task)"
          @delete="emit('delete', task.id)"
          @dragstart="handleDragStart(task, $event)"
          @dragend="handleDragEnd"
          @resize-start="handleResizeStart"
        />
      </div>
    </div>
    
    <div v-if="hasNextYear" class="next-year-container">
      <button @click="scrollToNextYear" class="next-year-btn" title="Ga naar volgend jaar">
        Tijdlijn {{ year + 1 }} →
      </button>
    </div>
  </div>
</template>

<style scoped>
.timeline-container {
  background: white;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  overflow: visible;
  width: 100%;
}

.timeline-year {
  font-size: 24px;
  font-weight: 600;
  color: #333;
  margin-bottom: 20px;
  padding-bottom: 10px;
  border-bottom: 3px solid #4CAF50;
}

.timeline-header {
  position: relative;
  height: 60px;
  margin-bottom: 20px;
  border-bottom: 2px solid #333;
  width: 100%;
}

.month-markers {
  position: relative;
  height: 100%;
  width: 100%;
}

.month-marker {
  position: absolute;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.month-marker:first-child {
  left: 0 !important;
  align-items: flex-start;
}

.month-marker:not(:first-child) {
  transform: translateX(-50%);
}

.month-line {
  width: 2px;
  height: 20px;
  background: #333;
}

.month-label {
  font-size: 14px;
  font-weight: 600;
  color: #333;
  white-space: nowrap;
  margin-top: 5px;
}

.chevron-next {
  position: absolute;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
  background: #4CAF50;
  color: white;
  border: none;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  cursor: pointer;
  font-size: 20px;
  font-weight: bold;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  display: flex;
  align-items: center;
  justify-content: center;
}

.chevron-next:hover {
  background: #45a049;
  transform: translateY(-50%) scale(1.1);
  box-shadow: 0 4px 8px rgba(0,0,0,0.15);
}

.next-year-container {
  display: flex;
  justify-content: flex-end;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 2px solid #eee;
}

.next-year-btn {
  background: #4CAF50;
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 600;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.next-year-btn:hover {
  background: #45a049;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0,0,0,0.15);
}

.month-marker:first-child .month-line {
  margin-left: 0;
}

.month-marker:last-child .month-line {
  margin-right: 0;
}

.timeline-grid {
  position: relative;
  min-height: 400px;
  width: 100%;
  margin-top: 20px;
  overflow: hidden;
}

.drop-zones {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: grid;
  grid-template-columns: 31fr 28fr 31fr 30fr 31fr 30fr 31fr 31fr 30fr 31fr 30fr 31fr;
  gap: 0;
  pointer-events: all;
  z-index: 1;
}

.drop-zone {
  position: relative;
  pointer-events: all;
  transition: background-color 0.2s;
}

.drop-zone-hover {
  background-color: rgba(76, 175, 80, 0.15) !important;
  border: 2px dashed #4CAF50;
  border-radius: 4px;
}

.drop-indicator {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: #4CAF50;
  color: white;
  padding: 12px 24px;
  border-radius: 6px;
  font-weight: 600;
  font-size: 16px;
  box-shadow: 0 4px 12px rgba(76, 175, 80, 0.4);
  pointer-events: none;
  z-index: 100;
}

.grid-background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: grid;
  grid-template-columns: 31fr 28fr 31fr 30fr 31fr 30fr 31fr 31fr 30fr 31fr 30fr 31fr;
  gap: 0;
  pointer-events: none;
  z-index: 0;
}

.grid-column {
  border-right: 1px solid #eee;
  height: 100%;
  min-height: 100%;
}

.grid-column:last-child {
  border-right: none;
}

.tasks-grid {
  position: relative;
  width: 100%;
  height: 100%;
  display: grid;
  grid-template-columns: 31fr 28fr 31fr 30fr 31fr 30fr 31fr 31fr 30fr 31fr 30fr 31fr;
  grid-auto-rows: 100px;
  gap: 8px;
  padding: 4px;
  box-sizing: border-box;
  pointer-events: none;
  z-index: 2;
}

.tasks-grid > * {
  box-sizing: border-box;
  pointer-events: all;
}
</style>
