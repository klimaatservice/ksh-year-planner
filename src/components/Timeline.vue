<script setup>
import { ref, computed } from 'vue'
import TaskCard from './TaskCard.vue'

const props = defineProps({
  year: Number,
  tasks: Array
})

const emit = defineEmits(['edit', 'delete', 'click'])

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
  const startDate = new Date(task.startDate)
  const endDate = new Date(task.endDate)
  
  // Bepaal start en eind maand (1-based)
  const startMonth = startDate.getMonth() + 1
  const endMonth = endDate.getMonth() + 1
  const startYear = startDate.getFullYear()
  const endYear = endDate.getFullYear()
  
  let columnStart = startMonth
  let columnEnd = endMonth + 1 // +1 omdat grid-column-end exclusief is
  
  // Als task in dit jaar begint maar doorloopt naar volgend jaar
  if (startYear === props.year && endYear > props.year) {
    columnStart = startMonth
    columnEnd = 13 // Tot einde van het jaar (31 december)
  }
  // Als task vanuit vorig jaar doorloopt
  else if (startYear < props.year && endYear === props.year) {
    columnStart = 1
    columnEnd = endMonth + 1
  }
  // Als task volledig dit jaar beslaat
  else if (startYear === props.year && endYear === props.year) {
    columnStart = startMonth
    columnEnd = endMonth + 1
  }
  
  return {
    gridColumnStart: columnStart,
    gridColumnEnd: columnEnd,
    gridRow: task.row + 1
  }
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

.grid-background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: grid;
  grid-template-columns: 31fr 28fr 31fr 30fr 31fr 30fr 31fr 31fr 30fr 31fr 30fr 31fr;
  gap: 0;
}

.grid-column {
  border-right: 1px solid #eee;
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
}

.tasks-grid > * {
  box-sizing: border-box;
}
</style>
