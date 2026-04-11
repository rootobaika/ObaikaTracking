<script setup>
import { reactive, ref, watch } from 'vue';

const props = defineProps({
  goal: {
    type: Object,
    required: true,
  },
  notes: {
    type: Array,
    default: () => [],
  },
});

const emit = defineEmits(['add-checkin', 'toggle-done', 'remove-goal', 'edit-note', 'delete-note']);

const checkinForm = reactive({
  amount: 1,
  note: '',
});

const editingNoteId = ref(null);
const editingNoteText = ref('');

watch(
  () => props.goal.id,
  () => {
    checkinForm.amount = 1;
    checkinForm.note = '';
    editingNoteId.value = null;
  }
);

function onAddCheckin() {
  emit('add-checkin', {
    goalId: props.goal.id,
    amount: Number(checkinForm.amount),
    note: checkinForm.note,
  });

  checkinForm.amount = 1;
  checkinForm.note = '';
}

function formatDate(dateString) {
  const date = new Date(dateString);
  return new Intl.DateTimeFormat('ru-RU', {
    day: '2-digit',
    month: 'short',
    hour: '2-digit',
    minute: '2-digit',
  }).format(date);
}

function startEditNote(note) {
  editingNoteId.value = note.id;
  editingNoteText.value = note.note || '';
}

function saveEditNote(note) {
  if (editingNoteText.value.trim()) {
    emit('edit-note', {
      goalId: props.goal.id,
      checkinId: note.id,
      note: editingNoteText.value.trim(),
    });
  }
  cancelEditNote();
}

function cancelEditNote() {
  editingNoteId.value = null;
  editingNoteText.value = '';
}

function deleteNote(checkinId) {
  emit('delete-note', {
    goalId: props.goal.id,
    checkinId: checkinId,
  });
}
</script>

<template>
  <article class="card goal-card">
    <div class="goal-head">
      <div>
        <h3>{{ props.goal.title }}</h3>
        <p>{{ props.goal.description || 'Описание не добавлено' }}</p>
        <small v-if="props.goal.deadline">Дедлайн: {{ props.goal.deadline }}</small>
      </div>
      <div class="actions">
        <button class="secondary" @click="emit('toggle-done', props.goal)">
          {{ props.goal.isDone ? 'Вернуть в активные' : 'Отметить выполненной' }}
        </button>
        <button class="danger" @click="emit('remove-goal', props.goal.id)">Удалить</button>
      </div>
    </div>

    <div class="progress-row">
      <div class="bar"><span :style="{ width: `${props.goal.progress}%` }" /></div>
      <strong>{{ props.goal.current }} / {{ props.goal.target }} ({{ props.goal.progress }}%)</strong>
    </div>

    <form class="checkin-row" @submit.prevent="onAddCheckin">
      <input v-model.number="checkinForm.amount" type="number" placeholder="+1" />
      <input v-model="checkinForm.note" type="text" placeholder="Короткая заметка" />
      <button type="submit">Добавить прогресс</button>
    </form>

    <div v-if="props.notes.length" class="notes-list">
      <h4>Пометки</h4>
      <div v-for="noteItem in props.notes" :key="noteItem.id" class="note-item">
        <template v-if="editingNoteId === noteItem.id">
          <input class="note-input" v-model="editingNoteText" type="text" @keyup.enter="saveEditNote(noteItem)" @keyup.escape="cancelEditNote" />
          <div class="note-btns">
            <button class="note-btn save" type="button" @click="saveEditNote(noteItem)">✓</button>
            <button class="note-btn cancel" type="button" @click="cancelEditNote">✕</button>
          </div>
        </template>
        <template v-else>
          <span class="note-text">{{ noteItem.note }}</span>
          <span class="note-meta">{{ formatDate(noteItem.createdAt) }} • +{{ noteItem.amount }}</span>
          <div class="note-btns">
            <button class="note-btn" type="button" @click="startEditNote(noteItem)" title="Ред.">✎</button>
            <button class="note-btn del" type="button" @click="deleteNote(noteItem.id)" title="Удал.">✕</button>
          </div>
        </template>
      </div>
    </div>
  </article>
</template>
