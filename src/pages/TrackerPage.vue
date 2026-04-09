<script setup>
import { computed, onMounted, reactive, ref } from 'vue';
import GoalCard from '../components/GoalCard.vue';
import MetricCard from '../components/MetricCard.vue';

const DB_STORAGE_KEY_WEB = 'tracker-db-v1';
const DB_STORAGE_KEY_TAURI = 'tracker-db-v1-tauri';
const THEME_STORAGE_KEY_WEB = 'tracker-theme';
const THEME_STORAGE_KEY_TAURI = 'tracker-theme-tauri';

const loading = ref(false);
const savingGoal = ref(false);
const error = ref('');
const theme = ref('light');
const goals = ref([]);
const notesByGoal = ref({});
const summary = ref({
  totalGoals: 0,
  completedGoals: 0,
  activeGoals: 0,
  completionRate: 0,
  portfolioProgress: 0,
  weekProgress: 0,
  weekCheckins: 0,
  monthProgress: 0,
  monthCheckins: 0,
});

const goalForm = reactive({
  title: '',
  description: '',
  target: 10,
  deadline: '',
});

const doneGoals = computed(() => goals.value.filter((g) => g.isDone).length);

const summaryCards = computed(() => [
  { title: 'Всего целей', value: summary.value.totalGoals },
  { title: 'Выполнено', value: summary.value.completedGoals },
  { title: 'Активные', value: summary.value.activeGoals },
  { title: 'Процент закрытия', value: `${summary.value.completionRate}%` },
]);

function applyTheme(nextTheme) {
  theme.value = nextTheme;
  document.documentElement.setAttribute('data-theme', nextTheme);
  localStorage.setItem(getThemeStorageKey(), nextTheme);
}

function toggleTheme() {
  applyTheme(theme.value === 'dark' ? 'light' : 'dark');
}

function toNumber(value, fallback = 0) {
  const num = Number(value);
  return Number.isFinite(num) ? num : fallback;
}

function isTauriRuntime() {
  if (typeof window === 'undefined') return false;
  const hasGlobalTauri = typeof window.__TAURI__ !== 'undefined';
  const hasInternalBridge = typeof window.__TAURI_INTERNALS__ !== 'undefined';
  const isTauriProtocol = window.location?.protocol === 'tauri:';
  const isTauriUserAgent = typeof navigator !== 'undefined' && navigator.userAgent.includes('Tauri');
  return hasGlobalTauri || hasInternalBridge || isTauriProtocol || isTauriUserAgent;
}

function getDbStorageKey() {
  return isTauriRuntime() ? DB_STORAGE_KEY_TAURI : DB_STORAGE_KEY_WEB;
}

function getThemeStorageKey() {
  return isTauriRuntime() ? THEME_STORAGE_KEY_TAURI : THEME_STORAGE_KEY_WEB;
}

function readDb() {
  const raw = localStorage.getItem(getDbStorageKey());
  if (!raw) {
    return {
      goals: [],
      checkins: [],
      nextGoalId: 1,
      nextCheckinId: 1,
    };
  }

  try {
    const parsed = JSON.parse(raw);
    return {
      goals: Array.isArray(parsed.goals) ? parsed.goals : [],
      checkins: Array.isArray(parsed.checkins) ? parsed.checkins : [],
      nextGoalId: Math.max(1, Math.floor(toNumber(parsed.nextGoalId, 1))),
      nextCheckinId: Math.max(1, Math.floor(toNumber(parsed.nextCheckinId, 1))),
    };
  } catch (_err) {
    return {
      goals: [],
      checkins: [],
      nextGoalId: 1,
      nextCheckinId: 1,
    };
  }
}

function writeDb(db) {
  localStorage.setItem(getDbStorageKey(), JSON.stringify(db));
}

function mapGoal(goal) {
  const target = Math.max(1, toNumber(goal.target, 1));
  const current = Math.max(0, toNumber(goal.current, 0));
  const progress = Math.min(100, Math.round((current / target) * 100));

  return {
    ...goal,
    target,
    current,
    progress,
    isDone: goal.status === 'done' || current >= target,
  };
}

function getSummary(db) {
  const now = Date.now();
  const weekAgo = now - 7 * 24 * 60 * 60 * 1000;
  const monthAgo = now - 30 * 24 * 60 * 60 * 1000;

  const totals = db.goals.reduce(
    (acc, goal) => {
      const target = Math.max(1, toNumber(goal.target, 1));
      const current = Math.max(0, toNumber(goal.current, 0));
      const isDone = goal.status === 'done' || current >= target;
      const isActive = goal.status === 'active' && current < target;

      acc.totalGoals += 1;
      acc.completedGoals += isDone ? 1 : 0;
      acc.activeGoals += isActive ? 1 : 0;
      acc.totalCurrent += current;
      acc.totalTarget += target;
      return acc;
    },
    { totalGoals: 0, completedGoals: 0, activeGoals: 0, totalCurrent: 0, totalTarget: 0 }
  );

  const checkinTotals = db.checkins.reduce(
    (acc, checkin) => {
      const time = new Date(checkin.createdAt).getTime();
      if (Number.isNaN(time)) return acc;

      if (time >= weekAgo) {
        acc.weekProgress += toNumber(checkin.amount, 0);
        acc.weekCheckins += 1;
      }
      if (time >= monthAgo) {
        acc.monthProgress += toNumber(checkin.amount, 0);
        acc.monthCheckins += 1;
      }
      return acc;
    },
    { weekProgress: 0, weekCheckins: 0, monthProgress: 0, monthCheckins: 0 }
  );

  const completionRate = totals.totalGoals
    ? Math.round((totals.completedGoals / Math.max(1, totals.totalGoals)) * 100)
    : 0;
  const portfolioProgress = totals.totalTarget
    ? Math.round((totals.totalCurrent / Math.max(1, totals.totalTarget)) * 100)
    : 0;

  return {
    totalGoals: totals.totalGoals,
    completedGoals: totals.completedGoals,
    activeGoals: totals.activeGoals,
    completionRate,
    portfolioProgress: Math.min(100, portfolioProgress),
    weekProgress: checkinTotals.weekProgress,
    weekCheckins: checkinTotals.weekCheckins,
    monthProgress: checkinTotals.monthProgress,
    monthCheckins: checkinTotals.monthCheckins,
  };
}

async function api(path, options) {
  const method = String(options?.method || 'GET').toUpperCase();
  const body = options?.body ? JSON.parse(options.body) : {};
  const now = new Date().toISOString();
  const db = readDb();

  if (path === '/api/goals' && method === 'GET') {
    return db.goals
      .slice()
      .sort((a, b) => String(b.createdAt).localeCompare(String(a.createdAt)))
      .map(mapGoal);
  }

  if (path === '/api/goals' && method === 'POST') {
    const title = String(body.title ?? '').trim();
    if (!title) throw new Error('Goal title is required.');

    const description = String(body.description ?? '').trim();
    const target = Math.max(1, Math.floor(toNumber(body.target, 1)));
    const deadline = String(body.deadline ?? '').trim() || null;

    const goal = {
      id: db.nextGoalId,
      title,
      description: description || null,
      target,
      current: 0,
      deadline,
      status: 'active',
      createdAt: now,
      updatedAt: now,
    };

    db.goals.push(goal);
    db.nextGoalId += 1;
    writeDb(db);
    return mapGoal(goal);
  }

  if (path === '/api/summary' && method === 'GET') {
    return getSummary(db);
  }

  const goalCheckinsMatch = path.match(/^\/api\/goals\/(\d+)\/checkins$/);
  if (goalCheckinsMatch) {
    const goalId = Number(goalCheckinsMatch[1]);
    const goal = db.goals.find((item) => item.id === goalId);
    if (!goal) throw new Error('Goal not found.');

    if (method === 'GET') {
      return db.checkins
        .filter((item) => item.goalId === goalId)
        .sort((a, b) => String(b.createdAt).localeCompare(String(a.createdAt)))
        .slice(0, 20);
    }

    if (method === 'POST') {
      const amount = Math.floor(toNumber(body.amount, 0));
      if (!amount) throw new Error('Check-in amount must be non-zero.');

      const note = String(body.note ?? '').trim();
      const checkin = {
        id: db.nextCheckinId,
        goalId,
        amount,
        note: note || null,
        createdAt: now,
      };

      db.nextCheckinId += 1;
      db.checkins.push(checkin);

      const target = Math.max(1, toNumber(goal.target, 1));
      const current = Math.max(0, toNumber(goal.current, 0));
      const newCurrent = Math.min(target, Math.max(0, current + amount));
      goal.current = newCurrent;
      goal.status = newCurrent >= target ? 'done' : goal.status;
      goal.updatedAt = now;

      writeDb(db);
      return mapGoal(goal);
    }
  }

  const goalMatch = path.match(/^\/api\/goals\/(\d+)$/);
  if (goalMatch) {
    const id = Number(goalMatch[1]);
    const goal = db.goals.find((item) => item.id === id);
    if (!goal) throw new Error('Goal not found.');

    if (method === 'PATCH') {
      const title = body.title === undefined ? goal.title : String(body.title).trim();
      const description =
        body.description === undefined ? goal.description : String(body.description).trim() || null;
      const target =
        body.target === undefined
          ? toNumber(goal.target, 1)
          : Math.max(1, Math.floor(toNumber(body.target, goal.target)));
      const current =
        body.current === undefined
          ? toNumber(goal.current, 0)
          : Math.max(0, Math.floor(toNumber(body.current, goal.current)));

      let status = body.status === undefined ? goal.status : String(body.status).trim();
      if (!['active', 'done', 'archived'].includes(status)) status = goal.status;
      const deadline =
        body.deadline === undefined ? goal.deadline : String(body.deadline ?? '').trim() || null;

      goal.title = title || goal.title;
      goal.description = description;
      goal.target = target;
      goal.current = current;
      goal.status = status;
      goal.deadline = deadline;
      goal.updatedAt = now;

      writeDb(db);
      return mapGoal(goal);
    }

    if (method === 'DELETE') {
      db.goals = db.goals.filter((item) => item.id !== id);
      db.checkins = db.checkins.filter((item) => item.goalId !== id);
      writeDb(db);
      return null;
    }
  }

  throw new Error('Request failed');
}

async function loadNotesForGoals(goalIds) {
  if (!goalIds.length) {
    notesByGoal.value = {};
    return;
  }

  const entries = await Promise.all(
    goalIds.map(async (goalId) => {
      try {
        const checkins = await api(`/api/goals/${goalId}/checkins`);
        return [goalId, checkins.filter((item) => item.note && item.note.trim())];
      } catch (_err) {
        return [goalId, []];
      }
    })
  );

  notesByGoal.value = Object.fromEntries(entries);
}

async function loadDashboard() {
  loading.value = true;
  error.value = '';
  try {
    const [goalsRes, summaryRes] = await Promise.all([api('/api/goals'), api('/api/summary')]);
    goals.value = goalsRes;
    summary.value = summaryRes;
    await loadNotesForGoals(goalsRes.map((goal) => goal.id));
  } catch (err) {
    error.value = err.message;
  } finally {
    loading.value = false;
  }
}

async function createGoal() {
  if (!goalForm.title.trim()) {
    error.value = 'Введите название цели';
    return;
  }

  savingGoal.value = true;
  error.value = '';
  try {
    await api('/api/goals', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(goalForm),
    });
    goalForm.title = '';
    goalForm.description = '';
    goalForm.target = 10;
    goalForm.deadline = '';
    await loadDashboard();
  } catch (err) {
    error.value = err.message;
  } finally {
    savingGoal.value = false;
  }
}

async function addCheckin({ goalId, amount, note }) {
  error.value = '';
  try {
    await api(`/api/goals/${goalId}/checkins`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount, note }),
    });
    await loadDashboard();
  } catch (err) {
    error.value = err.message;
  }
}

async function toggleDone(goal) {
  const nextStatus = goal.isDone ? 'active' : 'done';
  try {
    await api(`/api/goals/${goal.id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status: nextStatus }),
    });
    await loadDashboard();
  } catch (err) {
    error.value = err.message;
  }
}

async function removeGoal(id) {
  error.value = '';
  try {
    await api(`/api/goals/${id}`, { method: 'DELETE' });
    goals.value = goals.value.filter((g) => g.id !== id);
    await loadDashboard();
  } catch (err) {
    error.value = err.message;
  }
}

onMounted(() => {
  const savedTheme = localStorage.getItem(getThemeStorageKey());
  const prefersDark = window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches;
  applyTheme(savedTheme || (prefersDark ? 'dark' : 'light'));
  loadDashboard();
});
</script>

<template>
  <main class="page">
    <section class="hero">
      <div class="hero-top">
        <p class="season">Тема сезона</p>
        <button class="theme-toggle" type="button" @click="toggleTheme">
          {{ theme === 'dark' ? 'Светлая тема' : 'Темная тема' }}
        </button>
      </div>
      <h1>Весенний перезапуск</h1>
      <p class="subtitle">Личный цифровой трекер целей, прогресса и итогов</p>
    </section>

    <section class="summary-grid">
      <MetricCard v-for="card in summaryCards" :key="card.title" :title="card.title" :value="card.value" />

      <MetricCard title="Прогресс портфеля целей" :wide="true">
        <div class="bar"><span :style="{ width: `${summary.portfolioProgress}%` }" /></div>
        <small>{{ summary.portfolioProgress }}%</small>
      </MetricCard>

      <MetricCard title="Итоги" :wide="true">
        <small>
          За 7 дней: +{{ summary.weekProgress }} ({{ summary.weekCheckins }} отметок)<br />
          За 30 дней: +{{ summary.monthProgress }} ({{ summary.monthCheckins }} отметок)
        </small>
      </MetricCard>
    </section>

    <section class="card creator">
      <h2>Добавить цель</h2>
      <form class="goal-form" @submit.prevent="createGoal">
        <input v-model="goalForm.title" type="text" placeholder="Например: Читать каждый день" />
        <textarea
          v-model="goalForm.description"
          rows="2"
          placeholder="Коротко: зачем эта цель и как поймешь результат"
        />
        <div class="row">
          <input v-model.number="goalForm.target" type="number" min="1" placeholder="Целевое значение" />
          <input v-model="goalForm.deadline" type="date" />
          <button :disabled="savingGoal" type="submit">{{ savingGoal ? 'Сохраняю...' : 'Создать цель' }}</button>
        </div>
      </form>
      <p v-if="error" class="error">{{ error }}</p>
    </section>

    <section class="goals">
      <header>
        <h2>Мои цели</h2>
        <p>Выполнено сейчас: {{ doneGoals }} / {{ goals.length }}</p>
      </header>

      <div v-if="loading" class="card empty">Загрузка...</div>
      <div v-else-if="goals.length === 0" class="card empty">
        Пока нет целей. Добавь первую цель, чтобы начать весенний перезапуск.
      </div>

      <GoalCard
        v-for="goal in goals"
        :key="goal.id"
        :goal="goal"
        :notes="notesByGoal[goal.id] || []"
        @add-checkin="addCheckin"
        @toggle-done="toggleDone"
        @remove-goal="removeGoal"
      />
    </section>
  </main>
</template>

