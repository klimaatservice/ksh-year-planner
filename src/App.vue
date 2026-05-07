<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import Timeline from './components/Timeline.vue'
import TaskCard from './components/TaskCard.vue'
import PrintView from './components/PrintView.vue'

const year = ref(new Date().getFullYear())

// Default taken
const defaultTasks = [
  {
    id: 1,
    title: 'Meldingen\nplatform',
    startDate: '2026-03-15',
    endDate: '2026-04-01',
    color: '#E6D5FF',
    row: 0
  },
  {
    id: 2,
    title: 'Vinea versiebeheer',
    startDate: '2026-04-01',
    endDate: '2026-05-01',
    color: '#C8E6C9',
    row: 0
  },
  {
    id: 3,
    title: 'MP documentatie',
    startDate: '2026-04-01',
    endDate: '2026-05-01',
    color: '#E1BEE7',
    row: 1
  },
  {
    id: 4,
    title: 'Power automate\nregeltechniek',
    startDate: '2026-04-01',
    endDate: '2026-05-01',
    color: '#E1BEE7',
    row: 2
  },
  {
    id: 5,
    title: 'Dashboard\n+ Acto',
    startDate: '2026-05-01',
    endDate: '2026-06-01',
    color: '#FFECB3',
    row: 0
  },
  {
    id: 6,
    title: 'Dashboard testen\nbugfixes',
    startDate: '2026-06-01',
    endDate: '2026-07-01',
    color: '#FFCCBC',
    row: 0
  },
  {
    id: 7,
    title: 'Dashboard\nlive',
    startDate: '2026-07-01',
    endDate: '2026-09-01',
    color: '#FFECB3',
    row: 0
  },
  {
    id: 8,
    title: 'Afmaken\nversiebeheer',
    startDate: '2026-09-01',
    endDate: '2026-12-31',
    color: '#C8E6C9',
    row: 0
  },
  {
    id: 9,
    title: 'Div.\nzaken afronden',
    startDate: '2027-01-01',
    endDate: '2027-03-01',
    color: '#E6D5FF',
    row: 0
  }
]

// Laad taken uit localStorage of gebruik defaults
const tasks = ref([])

onMounted(() => {
  const saved = localStorage.getItem('yearPlannerTasks')
  if (saved) {
    try {
      tasks.value = JSON.parse(saved)
    } catch (e) {
      tasks.value = [...defaultTasks]
    }
  } else {
    tasks.value = [...defaultTasks]
  }
})

// Sla taken automatisch op bij elke wijziging
watch(tasks, (newTasks) => {
  localStorage.setItem('yearPlannerTasks', JSON.stringify(newTasks))
}, { deep: true })

// Groepeer taken per jaar
const tasksByYear = computed(() => {
  const grouped = {}
  tasks.value.forEach(task => {
    const startDate = new Date(task.startDate)
    const endDate = new Date(task.endDate)
    const startYear = startDate.getFullYear()
    const endYear = endDate.getFullYear()
    
    // Als taak binnen één jaar blijft
    if (startYear === endYear) {
      if (!grouped[startYear]) grouped[startYear] = []
      grouped[startYear].push(task)
    } else {
      // Splits de taak over meerdere jaren
      // Deel 1: van start tot 31 december van startjaar
      if (!grouped[startYear]) grouped[startYear] = []
      grouped[startYear].push({
        ...task,
        id: `${task.id}-${startYear}`, // Unieke ID voor deze split
        endDate: `${startYear}-12-31`,
        originalId: task.id
      })
      
      // Deel 2: van 1 januari van eindjaar tot eind
      if (!grouped[endYear]) grouped[endYear] = []
      grouped[endYear].push({
        ...task,
        id: `${task.id}-${endYear}`, // Unieke ID voor deze split
        startDate: `${endYear}-01-01`,
        originalId: task.id
      })
    }
  })
  
  // Sorteer jaren
  return Object.keys(grouped)
    .sort()
    .map(year => ({
      year: parseInt(year),
      tasks: grouped[year]
    }))
})

const showForm = ref(false)
const showDetailsModal = ref(false)
const showPrintView = ref(false)
const editingTask = ref(null)
const viewingTask = ref(null)
const newTask = ref({
  title: '',
  startDate: '',
  endDate: '',
  color: '#C8E6C9',
  row: 0,
  checklist: [],
  comments: []
})

const newChecklistItem = ref('')
const newComment = ref('')

const colors = [
  '#E6D5FF', // Paars
  '#C8E6C9', // Groen
  '#FFECB3', // Geel
  '#FFCCBC', // Oranje
  '#E1BEE7', // Lila
  '#B3E5FC', // Blauw
  '#F8BBD0', // Roze
  '#DCEDC8'  // Lichtgroen
]

function addTask() {
  if (editingTask.value) {
    const index = tasks.value.findIndex(t => t.id === editingTask.value.id)
    if (index !== -1) {
      tasks.value[index] = { ...newTask.value, id: editingTask.value.id }
    }
    editingTask.value = null
  } else {
    tasks.value.push({
      ...newTask.value,
      id: Date.now()
    })
  }
  
  newTask.value = {
    title: '',
    startDate: '',
    endDate: '',
    color: '#C8E6C9',
    row: 0
  }
  showForm.value = false
}

function editTask(task) {
  editingTask.value = task
  newTask.value = { 
    ...task,
    checklist: task.checklist || [],
    comments: task.comments || []
  }
  showForm.value = true
}

function deleteTask(taskId) {
  tasks.value = tasks.value.filter(t => t.id !== taskId)
}

function updateTask(updatedTask) {
  // Zoek de originele taak (niet de gesplitste versie)
  const taskId = updatedTask.originalId || updatedTask.id
  
  // Als het een gesplitste taak ID is (bijv. "1-2026"), haal dan het originele ID eruit
  let realTaskId = taskId
  if (typeof taskId === 'string' && taskId.includes('-')) {
    const parts = taskId.split('-')
    if (parts.length === 2 && !isNaN(parts[0])) {
      realTaskId = parseInt(parts[0])
    }
  }
  
  const index = tasks.value.findIndex(t => t.id === realTaskId)
  
  if (index !== -1) {
    console.log('Updating task:', tasks.value[index].title, 'naar', updatedTask.startDate, '-', updatedTask.endDate)
    // Update de taak met de nieuwe datums, behoud andere eigenschappen
    tasks.value[index] = {
      ...tasks.value[index],
      startDate: updatedTask.startDate,
      endDate: updatedTask.endDate
    }
  } else {
    console.error('Task not found:', realTaskId, taskId)
  }
}

function viewTaskDetails(task) {
  viewingTask.value = task
  // Zorg dat checklist en comments arrays bestaan
  if (!viewingTask.value.checklist) viewingTask.value.checklist = []
  if (!viewingTask.value.comments) viewingTask.value.comments = []
  showDetailsModal.value = true
}

function addChecklistItem() {
  if (newChecklistItem.value.trim() && viewingTask.value) {
    if (!viewingTask.value.checklist) viewingTask.value.checklist = []
    viewingTask.value.checklist.push({
      id: Date.now(),
      text: newChecklistItem.value.trim(),
      completed: false
    })
    newChecklistItem.value = ''
  }
}

function toggleChecklistItem(itemId) {
  if (viewingTask.value) {
    const item = viewingTask.value.checklist.find(i => i.id === itemId)
    if (item) item.completed = !item.completed
  }
}

function removeChecklistItem(itemId) {
  if (viewingTask.value) {
    viewingTask.value.checklist = viewingTask.value.checklist.filter(i => i.id !== itemId)
  }
}

function addComment() {
  if (newComment.value.trim() && viewingTask.value) {
    if (!viewingTask.value.comments) viewingTask.value.comments = []
    viewingTask.value.comments.push({
      id: Date.now(),
      text: newComment.value.trim(),
      date: new Date().toISOString()
    })
    newComment.value = ''
  }
}

function removeComment(commentId) {
  if (viewingTask.value) {
    viewingTask.value.comments = viewingTask.value.comments.filter(c => c.id !== commentId)
  }
}

function closeDetailsModal() {
  showDetailsModal.value = false
  viewingTask.value = null
}

function cancelForm() {
  showForm.value = false
  editingTask.value = null
  newTask.value = {
    title: '',
    startDate: '',
    endDate: '',
    color: '#C8E6C9',
    row: 0,
    checklist: [],
    comments: []
  }
}

function exportJSON() {
  const data = {
    year: year.value,
    tasks: tasks.value
  }
  const json = JSON.stringify(data, null, 2)
  const blob = new Blob([json], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `jaarplanning-${year.value}.json`
  a.click()
  URL.revokeObjectURL(url)
}

function importJSON(event) {
  const file = event.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (e) => {
      try {
        const data = JSON.parse(e.target.result)
        if (data.year) year.value = data.year
        if (data.tasks) tasks.value = data.tasks
      } catch (error) {
        alert('Fout bij het importeren van het bestand')
      }
    }
    reader.readAsText(file)
  }
}

function printTimeline() {
  window.print()
}

function togglePrintView() {
  showPrintView.value = !showPrintView.value
}

function clearAllTasks() {
  if (confirm('Weet je zeker dat je alle taken wilt verwijderen? Dit kan niet ongedaan worden gemaakt.')) {
    tasks.value = []
    localStorage.removeItem('yearPlannerTasks')
  }
}
</script>

<template>
  <div class="app">
    <header>
      <h1>Tijdlijn {{ year }}</h1>
      <div class="header-actions">
        <button @click="showForm = true" class="btn-primary">+ Nieuwe Taak</button>
        <button @click="togglePrintView" class="btn-secondary">
          {{ showPrintView ? '📅 Tijdlijn' : '📝 Todo Print' }}
        </button>
        <button @click="printTimeline" class="btn-secondary">🖨️ Afdrukken</button>
        <button @click="exportJSON" class="btn-secondary">Exporteer JSON</button>
        <label class="btn-secondary">
          Importeer JSON
          <input type="file" accept=".json" @change="importJSON" style="display: none" />
        </label>
        <button @click="clearAllTasks" class="btn-danger">🗑️ Wis Alles</button>
      </div>
    </header>

    <div v-if="showForm" class="modal-overlay" @click.self="cancelForm">
      <div class="modal">
        <h2>{{ editingTask ? 'Taak Bewerken' : 'Nieuwe Taak' }}</h2>
        <form @submit.prevent="addTask">
          <div class="form-group">
            <label>Titel:</label>
            <textarea v-model="newTask.title" required rows="3"></textarea>
          </div>
          <div class="form-group">
            <label>Startdatum:</label>
            <input type="date" v-model="newTask.startDate" required />
          </div>
          <div class="form-group">
            <label>Einddatum:</label>
            <input type="date" v-model="newTask.endDate" required />
          </div>
          <div class="form-group">
            <label>Rij (0-3):</label>
            <input type="number" v-model.number="newTask.row" min="0" max="3" required />
          </div>
          <div class="form-group">
            <label>Kleur:</label>
            <div class="color-picker">
              <button 
                v-for="color in colors" 
                :key="color"
                type="button"
                class="color-option"
                :class="{ selected: newTask.color === color }"
                :style="{ backgroundColor: color }"
                @click="newTask.color = color"
              ></button>
            </div>
          </div>
          <div class="form-actions">
            <button type="submit" class="btn-primary">{{ editingTask ? 'Opslaan' : 'Toevoegen' }}</button>
            <button type="button" @click="cancelForm" class="btn-secondary">Annuleren</button>
          </div>
        </form>
      </div>
    </div>

    <div v-if="showDetailsModal && viewingTask" class="modal-overlay" @click.self="closeDetailsModal">
      <div class="modal details-modal">
        <div class="modal-header">
          <h2>{{ viewingTask.title }}</h2>
          <button @click="closeDetailsModal" class="close-btn">✕</button>
        </div>
        
        <div class="task-info">
          <div class="info-item">
            <strong>Periode:</strong> 
            {{ new Date(viewingTask.startDate).toLocaleDateString('nl-NL') }} - 
            {{ new Date(viewingTask.endDate).toLocaleDateString('nl-NL') }}
          </div>
          <div class="info-item">
            <strong>Rij:</strong> {{ viewingTask.row }}
          </div>
        </div>

        <div class="details-section">
          <h3>Checklist</h3>
          <div class="checklist">
            <div v-for="item in viewingTask.checklist" :key="item.id" class="checklist-item">
              <input 
                type="checkbox" 
                :checked="item.completed" 
                @change="toggleChecklistItem(item.id)"
                class="checkbox"
              />
              <span :class="{ completed: item.completed }">{{ item.text }}</span>
              <button @click="removeChecklistItem(item.id)" class="remove-btn">🗑️</button>
            </div>
          </div>
          <div class="add-item">
            <input 
              v-model="newChecklistItem" 
              @keyup.enter="addChecklistItem"
              placeholder="Nieuw checklist item..."
              class="input-field"
            />
            <button @click="addChecklistItem" class="btn-primary btn-small">Toevoegen</button>
          </div>
        </div>

        <div class="details-section">
          <h3>Opmerkingen</h3>
          <div class="comments">
            <div v-for="comment in viewingTask.comments" :key="comment.id" class="comment">
              <div class="comment-header">
                <span class="comment-date">{{ new Date(comment.date).toLocaleString('nl-NL') }}</span>
                <button @click="removeComment(comment.id)" class="remove-btn">🗑️</button>
              </div>
              <p>{{ comment.text }}</p>
            </div>
          </div>
          <div class="add-item">
            <textarea 
              v-model="newComment" 
              placeholder="Nieuwe opmerking..."
              rows="3"
              class="input-field"
            ></textarea>
            <button @click="addComment" class="btn-primary btn-small">Toevoegen</button>
          </div>
        </div>

        <div class="modal-actions">
          <button @click="() => { closeDetailsModal(); editTask(viewingTask); }" class="btn-secondary">
            ✏️ Bewerken
          </button>
          <button @click="closeDetailsModal" class="btn-primary">Sluiten</button>
        </div>
      </div>
    </div>

    <div v-if="!showPrintView">
      <Timeline 
        v-for="yearGroup in tasksByYear" 
        :key="yearGroup.year"
        :year="yearGroup.year" 
        :tasks="yearGroup.tasks" 
        @edit="editTask" 
        @delete="deleteTask"
        @click="viewTaskDetails"
        @updateTask="updateTask"
        class="timeline-section"
      />
    </div>
    
    <div v-else>
      <PrintView 
        v-for="yearGroup in tasksByYear" 
        :key="`print-${yearGroup.year}`"
        :year="yearGroup.year" 
        :tasks="yearGroup.tasks"
      />
    </div>
    
    <!-- PrintView altijd in DOM voor print functionaliteit -->
    <div v-show="false" class="print-only-container">
      <PrintView 
        v-for="yearGroup in tasksByYear" 
        :key="`print-hidden-${yearGroup.year}`"
        :year="yearGroup.year" 
        :tasks="yearGroup.tasks"
      />
    </div>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html {
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: #f5f5f5;
  margin: 0;
  padding: 0;
  overflow-x: hidden;
}

#app {
  margin: 0;
  padding: 0;
  width: 100vw;
}

.app {
  padding: 20px;
  width: calc(100vw - 40px);
  margin: 0;
  box-sizing: border-box;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

h1 {
  font-size: 28px;
  color: #333;
}

.header-actions {
  display: flex;
  gap: 10px;
}

.btn-primary {
  background: #4CAF50;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: background 0.2s;
}

.btn-primary:hover {
  background: #45a049;
}

.btn-secondary {
  background: #fff;
  color: #333;
  border: 1px solid #ddd;
  padding: 10px 20px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: background 0.2s;
}

.btn-secondary:hover {
  background: #f5f5f5;
}

.btn-danger {
  background: #DC3545;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: background 0.2s;
}

.btn-danger:hover {
  background: #C82333;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: white;
  padding: 30px;
  border-radius: 12px;
  width: 500px;
  max-width: 90vw;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 10px 40px rgba(0,0,0,0.2);
}

.modal h2 {
  margin-bottom: 20px;
  color: #333;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #555;
}

.form-group input,
.form-group textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  font-family: inherit;
}

.form-group textarea {
  resize: vertical;
}

.color-picker {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.color-option {
  width: 40px;
  height: 40px;
  border: 3px solid transparent;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.color-option:hover {
  transform: scale(1.1);
}

.color-option.selected {
  border-color: #333;
  transform: scale(1.1);
}

.form-actions {
  display: flex;
  gap: 10px;
  margin-top: 25px;
}

.form-actions button {
  flex: 1;
}

.details-modal {
  max-width: 700px;
  max-height: 85vh;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 15px;
  border-bottom: 2px solid #eee;
}

.modal-header h2 {
  margin: 0;
  flex: 1;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #666;
  padding: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-btn:hover {
  color: #333;
}

.task-info {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 6px;
  margin-bottom: 20px;
}

.info-item {
  margin-bottom: 8px;
}

.info-item:last-child {
  margin-bottom: 0;
}

.details-section {
  margin-bottom: 25px;
}

.details-section h3 {
  font-size: 18px;
  margin-bottom: 12px;
  color: #333;
}

.checklist {
  margin-bottom: 10px;
}

.checklist-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  background: #f8f9fa;
  border-radius: 6px;
  margin-bottom: 8px;
}

.checkbox {
  width: 20px;
  height: 20px;
  cursor: pointer;
}

.checklist-item span {
  flex: 1;
  font-size: 14px;
}

.checklist-item span.completed {
  text-decoration: line-through;
  color: #999;
}

.remove-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 16px;
  padding: 4px;
  opacity: 0.6;
  transition: opacity 0.2s;
}

.remove-btn:hover {
  opacity: 1;
}

.comments {
  margin-bottom: 10px;
}

.comment {
  background: #f8f9fa;
  padding: 12px;
  border-radius: 6px;
  margin-bottom: 10px;
}

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.comment-date {
  font-size: 12px;
  color: #666;
}

.comment p {
  margin: 0;
  font-size: 14px;
  white-space: pre-wrap;
}

.add-item {
  display: flex;
  gap: 10px;
  align-items: flex-start;
}

.input-field {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  font-family: inherit;
}

.btn-small {
  padding: 10px 16px;
  white-space: nowrap;
}

.modal-actions {
  display: flex;
  gap: 10px;
  margin-top: 25px;
  padding-top: 20px;
  border-top: 2px solid #eee;
}

.modal-actions button {
  flex: 1;
}

.timeline-section {
  margin-bottom: 40px;
}

.timeline-section:last-of-type {
  margin-bottom: 0;
}

/* Print stijlen */
@media print {
  @page {
    size: A4 portrait;
    margin: 15mm;
  }
  
  * {
    -webkit-print-color-adjust: exact !important;
    print-color-adjust: exact !important;
    color-adjust: exact !important;
  }
  
  body {
    background: white;
  }
  
  .app {
    padding: 0;
    width: 100%;
  }
  
  header {
    display: none !important;
  }
  
  .modal-overlay {
    display: none !important;
  }
  
  /* Verberg de normale timeline view bij printen */
  .timeline-section {
    display: none !important;
  }
  
  /* Toon alleen de print-only container bij printen */
  .print-only-container {
    display: block !important;
  }
  
  /* Verberg de preview versie bij printen */
  .app > div:not(.print-only-container) {
    display: none !important;
  }
}

.print-only-container {
  display: none;
}
</style>
