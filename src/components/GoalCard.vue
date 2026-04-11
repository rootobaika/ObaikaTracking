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

    <div v-if="props.notes.length" class="notes">
      <h4>Пометки</h4>
      <ul>
        <li v-for="noteItem in props.notes" :key="noteItem.id" class="note-item">
          <template v-if="editingNoteId === noteItem.id">
            <div class="note-edit">
              <input v-model="editingNoteText" type="text" @keyup.enter="saveEditNote(noteItem)" @keyup.escape="cancelEditNote" />
              <div class="note-edit-actions">
                <button class="save-btn" type="button" @click="saveEditNote(noteItem)">Сохранить</button>
                <button class="cancel-btn" type="button" @click="cancelEditNote">Отмена</button>
              </div>
            </div>
          </template>
          <template v-else>
            <div class="note-content">
              <p>{{ noteItem.note }}</p>
              <small>{{ formatDate(noteItem.createdAt) }} • {{ noteItem.amount > 0 ? '+' : '' }}{{ noteItem.amount }}</small>
            </div>
            <div class="note-actions">
              <button class="note-action edit" type="button" @click="startEditNote(noteItem)" title="Редактировать">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/></svg>
              </button>
              <button class="note-action delete" type="button" @click="deleteNote(noteItem.id)" title="Удалить">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"/><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/></svg>
              </button>
            </div>
          </template>
        </li>
      </ul>
    </div>
  </article>
</template>
