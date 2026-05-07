<script setup>
import { computed } from 'vue'

const props = defineProps({
  tasks: Array,
  year: Number
})

// Groepeer taken per maand
const tasksByMonth = computed(() => {
  const months = Array(12).fill(null).map((_, i) => ({
    month: i + 1,
    monthName: new Date(props.year, i, 1).toLocaleDateString('nl-NL', { month: 'long' }),
    tasks: []
  }))
  
  props.tasks.forEach(task => {
    const startDate = new Date(task.startDate)
    const endDate = new Date(task.endDate)
    const startMonth = startDate.getMonth()
    const endMonth = endDate.getMonth()
    
    // Voeg taak toe aan alle maanden waarin het actief is
    for (let i = startMonth; i <= endMonth; i++) {
      if (!months[i].tasks.find(t => t.id === task.id)) {
        months[i].tasks.push(task)
      }
    }
  })
  
  return months.filter(m => m.tasks.length > 0)
})

// Format title zonder \n
const formatTitle = (title) => {
  return title.replace(/\\n/g, ' ')
}
</script>

<template>
  <div class="print-view">
    <div class="print-header">
      <h1>Jaarplanning {{ year }} - Todo's & Progress Tracker</h1>
      <p class="print-subtitle">Print deze pagina om je voortgang handmatig bij te houden. Vink af en maak notities tijdens je werk!</p>
    </div>
    
    <div v-for="monthGroup in tasksByMonth" :key="monthGroup.month" class="month-section">
      <h2 class="month-title">{{ monthGroup.monthName }} {{ year }}</h2>
      
      <div v-for="task in monthGroup.tasks" :key="task.id" class="task-print-card">
        <div class="task-print-header" :style="{ backgroundColor: task.color }">
          <h3 class="task-print-title">{{ formatTitle(task.title) }}</h3>
          <div class="task-print-dates">
            {{ new Date(task.startDate).toLocaleDateString('nl-NL') }} - 
            {{ new Date(task.endDate).toLocaleDateString('nl-NL') }}
          </div>
        </div>
        
        <div class="task-print-body">
          <!-- Checklist items (indien aanwezig) -->
          <div v-if="task.checklist && task.checklist.length > 0" class="checklist-print">
            <h4>Checklist:</h4>
            <div v-for="item in task.checklist" :key="item.id" class="checklist-print-item">
              <div class="checkbox-box">☐</div>
              <span>{{ item.text }}</span>
            </div>
          </div>
          
          <!-- Standaard lege checkboxes als er geen checklist is -->
          <div v-else class="checklist-print">
            <h4>Checklist:</h4>
            <div class="checklist-print-item">
              <div class="checkbox-box">☐</div>
              <span class="empty-line">_______________________________________</span>
            </div>
            <div class="checklist-print-item">
              <div class="checkbox-box">☐</div>
              <span class="empty-line">_______________________________________</span>
            </div>
            <div class="checklist-print-item">
              <div class="checkbox-box">☐</div>
              <span class="empty-line">_______________________________________</span>
            </div>
          </div>
          
          <!-- Extra ruimte voor notities -->
          <div class="notes-section">
            <h4>Notities & Voortgang:</h4>
            <div class="notes-lines">
              <div class="note-line"></div>
              <div class="note-line"></div>
              <div class="note-line"></div>
              <div class="note-line"></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Print view is altijd aanwezig maar alleen zichtbaar bij printen of in print-view mode */
.print-view {
  display: none;
}

/* Screen view - toon wanneer expliciet geselecteerd */
.print-view:not(.screen-hidden) {
  display: block;
  background: #f5f5f5;
  padding: 30px;
  max-width: 800px;
  margin: 0 auto;
}

/* Screen styling voor preview */
@media screen {
  .print-view:not(.screen-hidden) .print-header {
    text-align: center;
    margin-bottom: 30px;
    padding-bottom: 20px;
    border-bottom: 3px solid #333;
  }
  
  .print-view:not(.screen-hidden) .print-header h1 {
    font-size: 28px;
    margin: 0 0 10px 0;
    color: #333;
  }
  
  .print-view:not(.screen-hidden) .print-subtitle {
    font-size: 14px;
    color: #666;
    margin: 0;
  }
  
  .print-view:not(.screen-hidden) .month-section {
    margin-bottom: 30px;
  }
  
  .print-view:not(.screen-hidden) .month-title {
    font-size: 22px;
    color: #333;
    margin: 0 0 15px 0;
    padding: 10px 15px;
    background: #e8e8e8;
    border-left: 4px solid #333;
    border-radius: 4px;
  }
  
  .print-view:not(.screen-hidden) .task-print-card {
    margin-bottom: 20px;
    border: 2px solid #333;
    border-radius: 8px;
    overflow: hidden;
    background: white;
  }
  
  .print-view:not(.screen-hidden) .task-print-header {
    padding: 15px 20px;
    -webkit-print-color-adjust: exact;
    print-color-adjust: exact;
  }
  
  .print-view:not(.screen-hidden) .task-print-title {
    font-size: 18px;
    margin: 0 0 8px 0;
    color: #222;
    font-weight: 600;
  }
  
  .print-view:not(.screen-hidden) .task-print-dates {
    font-size: 13px;
    color: #444;
    font-weight: 500;
  }
  
  .print-view:not(.screen-hidden) .task-print-body {
    padding: 20px;
  }
  
  .print-view:not(.screen-hidden) .checklist-print {
    margin-bottom: 20px;
  }
  
  .print-view:not(.screen-hidden) .checklist-print h4 {
    font-size: 14px;
    margin: 0 0 12px 0;
    color: #333;
    font-weight: 600;
  }
  
  .print-view:not(.screen-hidden) .checklist-print-item {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    margin-bottom: 10px;
    font-size: 14px;
  }
  
  .print-view:not(.screen-hidden) .checkbox-box {
    font-size: 20px;
    line-height: 1;
    color: #333;
    min-width: 24px;
  }
  
  .print-view:not(.screen-hidden) .empty-line {
    color: #999;
  }
  
  .print-view:not(.screen-hidden) .notes-section {
    margin-top: 20px;
  }
  
  .print-view:not(.screen-hidden) .notes-section h4 {
    font-size: 14px;
    margin: 0 0 12px 0;
    color: #333;
    font-weight: 600;
  }
  
  .print-view:not(.screen-hidden) .notes-lines {
    display: flex;
    flex-direction: column;
    gap: 15px;
  }
  
  .print-view:not(.screen-hidden) .note-line {
    height: 2px;
    background: #ddd;
    width: 100%;
  }
}

/* Print stijlen */
@media print {
  /* Bij printen: toon altijd de print view */
  .print-view {
    display: block !important;
    background: white;
    padding: 0;
    margin: 0;
  }
  
  @page {
    size: A4 portrait;
    margin: 15mm;
  }
  
  .print-header {
    text-align: center;
    margin-bottom: 20px;
    padding-bottom: 15px;
    border-bottom: 3px solid #333;
  }
  
  .print-header h1 {
    font-size: 24pt;
    margin: 0 0 8px 0;
    color: #333;
  }
  
  .print-subtitle {
    font-size: 11pt;
    color: #666;
    margin: 0;
  }
  
  .month-section {
    page-break-inside: avoid;
    margin-bottom: 25px;
  }
  
  .month-title {
    font-size: 18pt;
    color: #333;
    margin: 0 0 12px 0;
    padding: 8px 12px;
    background: #f0f0f0;
    border-left: 4px solid #333;
  }
  
  .task-print-card {
    page-break-inside: avoid;
    margin-bottom: 20px;
    border: 2px solid #333;
    border-radius: 8px;
    overflow: hidden;
  }
  
  .task-print-header {
    padding: 12px 15px;
    background-color: #e0e0e0;
    -webkit-print-color-adjust: exact !important;
    print-color-adjust: exact !important;
    color-adjust: exact !important;
  }
  
  .task-print-title {
    font-size: 14pt;
    margin: 0 0 6px 0;
    color: #222;
    font-weight: 600;
  }
  
  .task-print-dates {
    font-size: 10pt;
    color: #444;
    font-weight: 500;
  }
  
  .task-print-body {
    padding: 15px;
    background: white;
  }
  
  .checklist-print {
    margin-bottom: 15px;
  }
  
  .checklist-print h4 {
    font-size: 11pt;
    margin: 0 0 10px 0;
    color: #333;
    font-weight: 600;
  }
  
  .checklist-print-item {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    margin-bottom: 8px;
    font-size: 10pt;
  }
  
  .checkbox-box {
    font-size: 16pt;
    line-height: 1;
    color: #333;
    min-width: 20px;
  }
  
  .empty-line {
    color: #999;
  }
  
  .notes-section {
    margin-top: 15px;
  }
  
  .notes-section h4 {
    font-size: 11pt;
    margin: 0 0 10px 0;
    color: #333;
    font-weight: 600;
  }
  
  .notes-lines {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }
  
  .note-line {
    height: 1px;
    background: #ccc;
    width: 100%;
    position: relative;
  }
  
  .note-line::before {
    content: '';
    position: absolute;
    left: 0;
    top: 0;
    bottom: 0;
    width: 100%;
    border-bottom: 1px solid #999;
  }
}
</style>
