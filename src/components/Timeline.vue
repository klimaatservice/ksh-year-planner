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
const originalStartMonth = ref(null)
const originalEndMonth = ref(null)
const hoverRow = ref(null)

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
  // Check of we deze taak aan het resizen zijn (match ook gesplitste IDs)
  const resizingTaskId = resizingTask.value?.originalId || resizingTask.value?.id
  const currentTaskId = task.originalId || task.id
  
  // Strip year suffix voor matching (bijv. "1-2026" wordt "1")
  const getBaseId = (id) => {
    if (typeof id === 'string' && id.includes('-')) {
      const parts = id.split('-')
      return !isNaN(parts[0]) ? parseInt(parts[0]) : id
    }
    return id
  }
  
  const isThisTaskResizing = isResizing.value && resizingTask.value && 
    getBaseId(resizingTaskId) === getBaseId(currentTaskId)
  
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
  
  // Detecteer welke rij (Y positie)
  const timelineGrid = document.querySelector('.timeline-grid')
  if (timelineGrid) {
    const rect = timelineGrid.getBoundingClientRect()
    const mouseY = event.clientY - rect.top
    const rowHeight = 120 // grid-auto-rows: 100px + 20px gap
    const detectedRow = Math.max(0, Math.floor(mouseY / rowHeight))
    hoverRow.value = detectedRow
  }
}

function handleDragLeave() {
  hoverMonth.value = null
}

function handleDrop(event, targetMonth) {
  event.preventDefault()
  event.stopPropagation()
  
  console.log('Drop on month:', targetMonth + 1, 'year:', props.year)
  
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
  
  // Nieuwe startdatum = eerste dag van target maand in target jaar
  const newStartDate = new Date(props.year, targetMonth, 1)
  // Nieuwe einddatum = laatste dag van (target maand + duur)
  const endDateCalc = new Date(props.year, targetMonth + durationInMonths + 1, 0)
  
  // Detecteer nieuwe rij
  const newRow = hoverRow.value !== null ? hoverRow.value : task.row
  
  console.log('Updating:', task.title)
  console.log('  To:', newStartDate.toLocaleDateString(), '-', endDateCalc.toLocaleDateString())
  console.log('  Row:', task.row, '->', newRow)
  
  emit('updateTask', {
    ...task,
    startDate: newStartDate.toISOString().split('T')[0],
    endDate: endDateCalc.toISOString().split('T')[0],
    row: newRow
  })
  
  draggingTask.value = null
  hoverMonth.value = null
  hoverRow.value = null
}

// Resize handlers met live preview
function handleResizeStart(task, direction, event) {
  event.stopPropagation()
  event.preventDefault()
  
  console.log('=== RESIZE START ===')
  console.log('Task:', task.title)
  console.log('Direction:', direction)
  
  resizingTask.value = task
  resizeDirection.value = direction
  isResizing.value = true
  
  // Lees de HUIDIGE datums
  const startDate = new Date(task.startDate)
  const endDate = new Date(task.endDate)
  originalStartMonth.value = startDate.getMonth()
  originalEndMonth.value = endDate.getMonth()
  previewStartMonth.value = originalStartMonth.value
  previewEndMonth.value = originalEndMonth.value
  
  console.log('Original months:', originalStartMonth.value, 'to', originalEndMonth.value)
  console.log('Original dates:', task.startDate, 'to', task.endDate)
  
  document.addEventListener('mousemove', handleResizeMove)
  document.addEventListener('mouseup', handleResizeEnd)
  document.body.style.cursor = 'ew-resize'
}

function handleResizeMove(event) {
  if (!resizingTask.value || !isResizing.value) return
  
  const tasksGrid = document.querySelector('.tasks-grid')
  if (!tasksGrid) return
  
  const rect = tasksGrid.getBoundingClientRect()
  const mouseX = Math.max(0, Math.min(rect.width, event.clientX - rect.left))
  const gridWidth = rect.width
  
  // Bereken welke maand (0-11) - gebruik de ratio voor betere precisie
  let targetMonth = Math.floor((mouseX / gridWidth) * 12)
  targetMonth = Math.max(0, Math.min(11, targetMonth))
  
  if (resizeDirection.value === 'left') {
    // Resize startdatum - einddatum blijft VAST op originele positie
    if (targetMonth <= originalEndMonth.value) {
      previewStartMonth.value = targetMonth
      previewEndMonth.value = originalEndMonth.value
    }
  } else if (resizeDirection.value === 'right') {
    // Resize einddatum - startdatum blijft VAST op originele positie  
    if (targetMonth >= originalStartMonth.value) {
      previewStartMonth.value = originalStartMonth.value
      previewEndMonth.value = targetMonth
    }
  }
}

function handleResizeEnd(event) {
  console.log('=== RESIZE END ===')
  
  if (!resizingTask.value || !isResizing.value) {
    console.log('No resize task, cleanup')
    cleanup()
    return
  }
  
  const task = resizingTask.value
  const startMonth = previewStartMonth.value
  const endMonth = previewEndMonth.value
  
  console.log('Preview months:', startMonth, 'to', endMonth)
  console.log('Preview month names:', months[startMonth], 'to', months[endMonth])
  
  if (startMonth === null || endMonth === null) {
    console.log('Invalid months, cleanup')
    cleanup()
    return
  }
  
  // Maak nieuwe datums - SIMPEL
  // Start: eerste dag van startmaand
  const newStartYear = props.year
  const newStartMonth = startMonth  // 0-indexed
  const newStartDay = 1
  
  // Eind: laatste dag van eindmaand
  const newEndYear = props.year
  const newEndMonth = endMonth + 1  // +1 omdat we dag 0 pakken van VOLGENDE maand
  const newEndDay = 0  // Dag 0 = laatste dag van vorige maand
  
  const startDateObj = new Date(newStartYear, newStartMonth, newStartDay)
  const endDateObj = new Date(newEndYear, newEndMonth, newEndDay)
  
  const newStartDate = startDateObj.toISOString().split('T')[0]
  const newEndDate = endDateObj.toISOString().split('T')[0]
  
  console.log('New dates:', newStartDate, 'to', newEndDate)
  console.log('Emitting updateTask for task:', task.id, task.originalId)
  
  emit('updateTask', {
    ...task,
    startDate: newStartDate,
    endDate: newEndDate
  })
  
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
  originalStartMonth.value = null
  originalEndMonth.value = null
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
        ></div>
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
  position: relative;
}

.drop-zone-hover::after {
  content: 'Drop hier';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: #4CAF50;
  color: white;
  padding: 8px 16px;
  border-radius: 6px;
  font-weight: 600;
  font-size: 14px;
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
