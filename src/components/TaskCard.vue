<script setup>
import { computed, ref } from 'vue'

const props = defineProps({
  task: Object
})

const emit = defineEmits(['edit', 'delete', 'click', 'dragstart', 'dragend', 'resizeStart'])

const isResizing = ref(false)

// Format de titel met line breaks
const formattedTitle = computed(() => {
  return props.task.title.replace(/\\n/g, '\n')
})

function handleEdit() {
  emit('edit')
}

function handleDelete() {
  if (confirm(`Weet je zeker dat je "${props.task.title}" wilt verwijderen?`)) {
    emit('delete')
  }
}

function handleDragStart(event) {
  emit('dragstart', event)
}

function handleDragEnd(event) {
  emit('dragend', event)
}

function handleResizeStart(direction, event) {
  event.preventDefault()
  event.stopPropagation()
  isResizing.value = true
  emit('resizeStart', props.task, direction, event)
  
  // Reset na resize
  setTimeout(() => {
    isResizing.value = false
  }, 100)
}
</script>

<template>
  <div 
    class="task-card" 
    :style="{ backgroundColor: task.color }"
    :draggable="!isResizing"
    @click="emit('click')"
    @dragstart="!isResizing && handleDragStart($event)"
    @dragend="handleDragEnd"
  >
    <div 
      class="resize-handle resize-left"
      @mousedown.prevent.stop="handleResizeStart('left', $event)"
      @click.prevent.stop
      title="Sleep om startdatum te wijzigen"
    ></div>
    <div class="task-content">
      <div class="task-title">{{ formattedTitle }}</div>
      <div class="task-actions">
        <button @click.stop="handleEdit" class="task-btn" title="Bewerken">✏️</button>
        <button @click.stop="handleDelete" class="task-btn" title="Verwijderen">🗑️</button>
      </div>
    </div>
    <div 
      class="resize-handle resize-right"
      @mousedown.prevent.stop="handleResizeStart('right', $event)"
      @click.prevent.stop
      title="Sleep om einddatum te wijzigen"
    ></div>
  </div>
</template>

<style scoped>
.task-card {
  height: 100px;
  border-radius: 6px;
  padding: 10px 12px;
  cursor: move;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  border: 1px solid rgba(0,0,0,0.15);
  overflow: visible;
  display: flex;
  align-items: center;
  background-color: v-bind('task.color');
  box-sizing: border-box;
  position: relative;
}

@media print {
  .task-card {
    overflow: hidden !important;
    background-color: v-bind('task.color') !important;
    -webkit-print-color-adjust: exact !important;
    print-color-adjust: exact !important;
    color-adjust: exact !important;
  }
}

.task-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0,0,0,0.15);
  z-index: 10;
}

.task-card:active {
  cursor: grabbing;
  opacity: 0.7;
  transform: scale(1.02);
}

.task-content {
  position: relative;
  z-index: 2;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  width: 100%;
}

.task-title {
  font-size: 16px;
  font-weight: 600;
  color: #222;
  white-space: pre-line;
  line-height: 1.4;
  text-align: center;
}

.task-actions {
  position: absolute;
  top: 50%;
  right: -40px;
  transform: translateY(-50%);
  display: flex;
  gap: 5px;
  opacity: 0;
  transition: all 0.2s;
  z-index: 30;
}

.task-card:hover .task-actions {
  right: 8px;
  opacity: 1;
}

.task-btn {
  background: rgba(255, 255, 255, 0.9);
  border: 1px solid rgba(0,0,0,0.1);
  border-radius: 4px;
  width: 28px;
  height: 28px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  transition: all 0.2s;
  pointer-events: all;
}

.task-btn:hover {
  background: white;
  transform: scale(1.1);
}

.resize-handle {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 15px;
  cursor: ew-resize;
  background: transparent;
  transition: all 0.2s;
  z-index: 20;
}

.resize-handle:hover {
  background: rgba(76, 175, 80, 0.3);
}

.resize-handle:active {
  background: rgba(76, 175, 80, 0.5);
}

.resize-left {
  left: 0;
  border-top-left-radius: 6px;
  border-bottom-left-radius: 6px;
}

.resize-right {
  right: 0;
  border-top-right-radius: 6px;
  border-bottom-right-radius: 6px;
}

.task-card:hover .resize-handle {
  background: rgba(0, 0, 0, 0.15);
}
</style>
