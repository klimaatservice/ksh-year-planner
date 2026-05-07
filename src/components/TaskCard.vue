<script setup>
import { computed } from 'vue'

const props = defineProps({
  task: Object
})

const emit = defineEmits(['edit', 'delete', 'click'])

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
</script>

<template>
  <div 
    class="task-card" 
    :style="{ backgroundColor: task.color }"
    @click="emit('click')"
  >
    <div class="task-content">
      <div class="task-title">{{ formattedTitle }}</div>
      <div class="task-actions">
        <button @click.stop="handleEdit" class="task-btn" title="Bewerken">✏️</button>
        <button @click.stop="handleDelete" class="task-btn" title="Verwijderen">🗑️</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.task-card {
  height: 100px;
  border-radius: 6px;
  padding: 10px 12px;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  border: 1px solid rgba(0,0,0,0.15);
  overflow: hidden;
  display: flex;
  align-items: center;
  background-color: v-bind('task.color');
  box-sizing: border-box;
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
}

.task-btn:hover {
  background: white;
  transform: scale(1.1);
}
</style>
