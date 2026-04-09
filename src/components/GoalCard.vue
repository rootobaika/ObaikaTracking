<script setup>
import { reactive, watch } from 'vue';

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

const emit = defineEmits(['add-checkin', 'toggle-done', 'remove-goal']);

const checkinForm = reactive({
  amount: 1,
  note: '',
});

watch(
  () => props.goal.id,
  () => {
    checkinForm.amount = 1;
    checkinForm.note = '';
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
        <li v-for="noteItem in props.notes" :key="noteItem.id">
          <p>{{ noteItem.note }}</p>
          <small>{{ formatDate(noteItem.createdAt) }} • {{ noteItem.amount > 0 ? '+' : '' }}{{ noteItem.amount }}</small>
        </li>
      </ul>
    </div>
  </article>
</template>
