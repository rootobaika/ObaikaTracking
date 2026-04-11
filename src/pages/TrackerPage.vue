<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue';
import GoalCard from '../components/GoalCard.vue';
import MetricCard from '../components/MetricCard.vue';

const DB_STORAGE_KEY_WEB = 'tracker-db-v1';
const DB_STORAGE_KEY_TAURI = 'tracker-db-v1-tauri';
const THEME_STORAGE_KEY_WEB = 'tracker-theme';
const THEME_STORAGE_KEY_TAURI = 'tracker-theme-tauri';
const HWID_STORAGE_KEY_WEB = 'tracker-hwid-v1';
const HWID_STORAGE_KEY_TAURI = 'tracker-hwid-v1-tauri';
const AUTH_TOKEN_KEY_WEB = 'tracker-auth-token-v1';
const AUTH_TOKEN_KEY_TAURI = 'tracker-auth-token-v1-tauri';
const API_BASE_OVERRIDE_KEY_WEB = 'tracker-api-base-override-v1';
const API_BASE_OVERRIDE_KEY_TAURI = 'tracker-api-base-override-v1-tauri';
const DEFAULT_API_BASE_URL = String(import.meta.env.VITE_API_BASE_URL || 'https://backtodo.obaika.fun').replace(/\/$/, '');

const loading = ref(false);
const savingGoal = ref(false);
const error = ref('');
const theme = ref('light');
const hwid = ref('');
const syncMode = ref('offline');
const syncInitialized = ref(false);
const isAuthenticated = ref(false);
const authLoading = ref(false);
const authMode = ref('login');
const authError = ref('');
const authForm = reactive({
  password: '',
  confirmPassword: '',
});
const currentApiBaseUrl = ref(DEFAULT_API_BASE_URL);
const settingsForm = reactive({
  apiBaseUrl: DEFAULT_API_BASE_URL,
});
const settingsMessage = ref('');
const settingsError = ref('');
const importingData = ref(false);
const changingPassword = ref(false);
const importFileInput = ref(null);
const settingsOpen = ref(false);
const settingsSection = ref('core');
const settingsSubsection = ref('server');
const passwordForm = reactive({
  oldPassword: '',
  newPassword: '',
  confirmPassword: '',
});
const goalTitleInput = ref(null);
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
const hwidMasked = computed(() => {
  if (!hwid.value) return '...';
  if (hwid.value.length <= 12) return hwid.value;
  return `${hwid.value.slice(0, 8)}...${hwid.value.slice(-4)}`;
});

const summaryCards = computed(() => [
  { title: 'Всего целей', value: summary.value.totalGoals },
  { title: 'Выполнено', value: summary.value.completedGoals },
  { title: 'Активные', value: summary.value.activeGoals },
  { title: 'Процент закрытия', value: `${summary.value.completionRate}%` },
]);

const sectionSubsections = {
  core: [
    { id: 'server', label: 'Сервер', icon: 'globe' },
    { id: 'appearance', label: 'Внешний вид', icon: 'palette' },
    { id: 'quick', label: 'Быстрый старт', icon: 'zap' },
  ],
  sync: [
    { id: 'status', label: 'Статус', icon: 'wifi' },
    { id: 'manual', label: 'Ручной режим', icon: 'refresh' },
  ],
  data: [
    { id: 'backup', label: 'Бэкап', icon: 'download' },
    { id: 'storage', label: 'Хранилище', icon: 'database' },
  ],
  security: [
    { id: 'auth', label: 'Вход', icon: 'log-in' },
    { id: 'password', label: 'Пароль', icon: 'lock' },
  ],
};

const activeSubsections = computed(() => sectionSubsections[settingsSection.value] || []);

const sectionIcons = {
  core: 'settings',
  sync: 'refresh-cw',
  data: 'hard-drive',
  security: 'shield',
};

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

function getHwidStorageKey() {
  return isTauriRuntime() ? HWID_STORAGE_KEY_TAURI : HWID_STORAGE_KEY_WEB;
}

function getAuthTokenStorageKey() {
  return isTauriRuntime() ? AUTH_TOKEN_KEY_TAURI : AUTH_TOKEN_KEY_WEB;
}

function getApiBaseOverrideStorageKey() {
  return isTauriRuntime() ? API_BASE_OVERRIDE_KEY_TAURI : API_BASE_OVERRIDE_KEY_WEB;
}

function normalizeApiBaseUrl(value) {
  return String(value || '').trim().replace(/\/$/, '');
}

function getFriendlyNetworkError() {
  const apiUrl = String(currentApiBaseUrl.value || '');
  const isHttpsPage = typeof window !== 'undefined' && window.location?.protocol === 'https:';
  if (isHttpsPage && apiUrl.startsWith('http://')) {
    return 'Network unavailable: сайт открыт по HTTPS, а backend указан через HTTP. Используй https:// адрес сервера.';
  }
  return 'Network unavailable';
}

function loadApiBaseUrl() {
  const override = normalizeApiBaseUrl(localStorage.getItem(getApiBaseOverrideStorageKey()));
  if (!override) return DEFAULT_API_BASE_URL;
  return override;
}

function setApiBaseUrl(url) {
  const normalized = normalizeApiBaseUrl(url);
  currentApiBaseUrl.value = normalized || DEFAULT_API_BASE_URL;
  settingsForm.apiBaseUrl = currentApiBaseUrl.value;
  const key = getApiBaseOverrideStorageKey();
  if (!normalized || normalized === DEFAULT_API_BASE_URL) {
    localStorage.removeItem(key);
    return;
  }
  localStorage.setItem(key, normalized);
}

function getAuthToken() {
  return String(localStorage.getItem(getAuthTokenStorageKey()) ?? '').trim();
}

function setAuthToken(token) {
  const key = getAuthTokenStorageKey();
  if (!token) {
    localStorage.removeItem(key);
    return;
  }
  localStorage.setItem(key, token);
}

function createHwid() {
  if (typeof crypto !== 'undefined' && typeof crypto.randomUUID === 'function') {
    return `hwid-${crypto.randomUUID()}`;
  }
  const random = Math.random().toString(36).slice(2);
  return `hwid-${Date.now().toString(36)}-${random}`;
}

function getOrCreateHwid() {
  const key = getHwidStorageKey();
  const existing = String(localStorage.getItem(key) ?? '').trim();
  if (existing) return existing;
  const next = createHwid();
  localStorage.setItem(key, next);
  return next;
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

function isBrowserOnline() {
  if (typeof navigator === 'undefined') return false;
  return navigator.onLine;
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

async function localApi(path, options) {
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

async function remoteApi(path, options) {
  const timeoutMs = 4500;
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);
  const method = String(options?.method || 'GET').toUpperCase();
  const headers = {
    ...(options?.headers || {}),
    'X-HWID': hwid.value,
  };
  const authToken = getAuthToken();
  if (authToken) {
    headers['X-Auth-Token'] = authToken;
  }

  try {
    let response;
    try {
      response = await fetch(`${currentApiBaseUrl.value}${path}`, {
        method,
        headers,
        body: options?.body,
        signal: controller.signal,
      });
    } catch (err) {
      const networkErr = new Error(getFriendlyNetworkError());
      networkErr.isNetworkError = true;
      throw networkErr;
    }

    if (!response.ok) {
      let message = 'Request failed';
      try {
        const payload = await response.json();
        message = payload.error || message;
      } catch (_err) {
        // Keep fallback message if body is not JSON.
      }
      const httpErr = new Error(message);
      httpErr.status = response.status;
      throw httpErr;
    }

    if (response.status === 204) return null;
    return await response.json();
  } finally {
    clearTimeout(timeout);
  }
}

async function detectOnlineSync() {
  if (!isBrowserOnline()) {
    syncMode.value = 'offline';
    return false;
  }

  try {
    await remoteApi('/api/health');
    syncMode.value = 'online';
    return true;
  } catch (_err) {
    syncMode.value = 'offline';
    return false;
  }
}

async function checkAuthStatus() {
  if (syncMode.value !== 'online') {
    isAuthenticated.value = false;
    return false;
  }

  const token = getAuthToken();
  if (!token) {
    isAuthenticated.value = false;
    return false;
  }

  try {
    await remoteApi('/api/auth/status');
    isAuthenticated.value = true;
    return true;
  } catch (_err) {
    setAuthToken('');
    isAuthenticated.value = false;
    return false;
  }
}

async function registerWithPassword() {
  authError.value = '';
  if (authForm.password.length < 6) {
    authError.value = 'Минимум 6 символов.';
    return;
  }
  if (authForm.password !== authForm.confirmPassword) {
    authError.value = 'Пароли не совпадают.';
    return;
  }

  authLoading.value = true;
  try {
    const res = await remoteApi('/api/auth/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ password: authForm.password }),
    });
    setAuthToken(res.token);
    isAuthenticated.value = true;
    authForm.password = '';
    authForm.confirmPassword = '';
    await loadDashboard();
  } catch (err) {
    authError.value = err.message;
  } finally {
    authLoading.value = false;
  }
}

async function loginWithPassword() {
  authError.value = '';
  if (!authForm.password) {
    authError.value = 'Введите пароль.';
    return;
  }

  authLoading.value = true;
  try {
    const res = await remoteApi('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ password: authForm.password }),
    });
    setAuthToken(res.token);
    isAuthenticated.value = true;
    authForm.password = '';
    authForm.confirmPassword = '';
    await loadDashboard();
  } catch (err) {
    authError.value = err.message;
  } finally {
    authLoading.value = false;
  }
}

async function logout() {
  try {
    if (syncMode.value === 'online' && getAuthToken()) {
      await remoteApi('/api/auth/logout', { method: 'POST' });
    }
  } catch (_err) {
    // Ignore logout network issues and still clear local auth state.
  } finally {
    setAuthToken('');
    isAuthenticated.value = false;
    authMode.value = 'login';
  }
}

function openSettings() {
  settingsOpen.value = true;
  clearSettingsMessages();
}

function closeSettings() {
  settingsOpen.value = false;
}

function setSettingsSection(nextSection) {
  settingsSection.value = nextSection;
}

function focusCreateGoal() {
  const target = goalTitleInput.value;
  if (!target) return;
  target.scrollIntoView({ behavior: 'smooth', block: 'center' });
  target.focus();
}

function handleGlobalKeydown(event) {
  if (event.key === 'Escape' && settingsOpen.value) {
    closeSettings();
  }
}

watch(
  () => settingsSection.value,
  (nextSection) => {
    const next = sectionSubsections[nextSection]?.[0]?.id;
    settingsSubsection.value = next || 'server';
  },
  { immediate: true }
);

function clearSettingsMessages() {
  settingsMessage.value = '';
  settingsError.value = '';
}

async function testConnection() {
  clearSettingsMessages();
  const probeUrl = normalizeApiBaseUrl(settingsForm.apiBaseUrl);
  if (!probeUrl) {
    settingsError.value = 'Введите адрес сервера.';
    return;
  }

  try {
    const response = await fetch(`${probeUrl}/api/health`, { method: 'GET' });
    if (!response.ok) throw new Error('Сервер недоступен.');
    settingsMessage.value = 'Соединение успешно.';
  } catch (_err) {
    settingsError.value = 'Не удалось подключиться к серверу.';
  }
}

async function saveApiSettings() {
  clearSettingsMessages();
  const nextUrl = normalizeApiBaseUrl(settingsForm.apiBaseUrl);
  if (!nextUrl) {
    settingsError.value = 'Введите адрес сервера.';
    return;
  }

  setApiBaseUrl(nextUrl);
  syncMode.value = 'offline';
  isAuthenticated.value = false;
  setAuthToken('');
  syncInitialized.value = false;

  const isOnline = await detectOnlineSync();
  if (isOnline) {
    await checkAuthStatus();
  }
  await loadDashboard();
  settingsMessage.value = 'Настройки сервера сохранены.';
}

async function resetApiSettings() {
  clearSettingsMessages();
  setApiBaseUrl(DEFAULT_API_BASE_URL);
  syncMode.value = 'offline';
  isAuthenticated.value = false;
  setAuthToken('');
  syncInitialized.value = false;
  await detectOnlineSync();
  await checkAuthStatus();
  await loadDashboard();
  settingsMessage.value = 'Сервер сброшен на значение по умолчанию.';
}

async function forceSyncNow() {
  clearSettingsMessages();
  const online = await detectOnlineSync();
  if (!online) {
    settingsError.value = 'Сеть недоступна, синхронизация невозможна.';
    return;
  }
  const authed = await checkAuthStatus();
  if (!authed) {
    settingsError.value = 'Для синхронизации нужна авторизация.';
    return;
  }

  syncInitialized.value = false;
  await syncLocalToRemoteIfNeeded();
  await loadDashboard();
  settingsMessage.value = 'Синхронизация завершена.';
}

function exportLocalData() {
  clearSettingsMessages();
  try {
    const backup = {
      exportedAt: new Date().toISOString(),
      hwid: hwid.value,
      db: readDb(),
    };
    const blob = new Blob([JSON.stringify(backup, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const anchor = document.createElement('a');
    anchor.href = url;
    anchor.download = `tracker-backup-${Date.now()}.json`;
    document.body.appendChild(anchor);
    anchor.click();
    document.body.removeChild(anchor);
    URL.revokeObjectURL(url);
    settingsMessage.value = 'Локальный бэкап экспортирован.';
  } catch (_err) {
    settingsError.value = 'Не удалось экспортировать данные.';
  }
}

function requestImportData() {
  importFileInput.value?.click();
}

async function handleImportData(event) {
  const file = event?.target?.files?.[0];
  if (!file) return;
  clearSettingsMessages();
  importingData.value = true;

  try {
    const text = await file.text();
    const parsed = JSON.parse(text);
    const db = parsed?.db ?? parsed;
    if (!db || !Array.isArray(db.goals) || !Array.isArray(db.checkins)) {
      throw new Error('Invalid backup format');
    }

    writeDb({
      goals: db.goals,
      checkins: db.checkins,
      nextGoalId: Math.max(1, Math.floor(toNumber(db.nextGoalId, db.goals.length + 1))),
      nextCheckinId: Math.max(1, Math.floor(toNumber(db.nextCheckinId, db.checkins.length + 1))),
    });

    syncInitialized.value = false;
    await loadDashboard();
    settingsMessage.value = 'Бэкап успешно импортирован.';
  } catch (_err) {
    settingsError.value = 'Не удалось импортировать файл.';
  } finally {
    importingData.value = false;
    if (event?.target) event.target.value = '';
  }
}

async function clearLocalData() {
  clearSettingsMessages();
  const confirmed = window.confirm('Очистить локальные данные на этом устройстве?');
  if (!confirmed) return;

  writeDb({
    goals: [],
    checkins: [],
    nextGoalId: 1,
    nextCheckinId: 1,
  });
  syncInitialized.value = false;
  await loadDashboard();
  settingsMessage.value = 'Локальные данные очищены.';
}

async function changePassword() {
  clearSettingsMessages();
  if (!isAuthenticated.value || syncMode.value !== 'online') {
    settingsError.value = 'Смена пароля доступна только онлайн после входа.';
    return;
  }
  if (!passwordForm.oldPassword || !passwordForm.newPassword) {
    settingsError.value = 'Заполните текущий и новый пароль.';
    return;
  }
  if (passwordForm.newPassword.length < 6) {
    settingsError.value = 'Новый пароль должен быть минимум 6 символов.';
    return;
  }
  if (passwordForm.newPassword !== passwordForm.confirmPassword) {
    settingsError.value = 'Новый пароль и подтверждение не совпадают.';
    return;
  }

  changingPassword.value = true;
  try {
    await remoteApi('/api/auth/change-password', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        oldPassword: passwordForm.oldPassword,
        newPassword: passwordForm.newPassword,
      }),
    });
    passwordForm.oldPassword = '';
    passwordForm.newPassword = '';
    passwordForm.confirmPassword = '';
    settingsMessage.value = 'Пароль успешно обновлён.';
  } catch (err) {
    settingsError.value = err.message || 'Не удалось сменить пароль.';
  } finally {
    changingPassword.value = false;
  }
}

async function syncLocalToRemoteIfNeeded() {
  if (syncMode.value !== 'online' || !isAuthenticated.value || syncInitialized.value) return;
  syncInitialized.value = true;

  const localDb = readDb();
  if (!localDb.goals.length) return;

  const remoteGoals = await remoteApi('/api/goals');
  if (Array.isArray(remoteGoals) && remoteGoals.length) return;

  const goalIdMap = new Map();
  const sortedGoals = localDb.goals
    .slice()
    .sort((a, b) => String(a.createdAt || '').localeCompare(String(b.createdAt || '')));

  for (const localGoal of sortedGoals) {
    const created = await remoteApi('/api/goals', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title: localGoal.title,
        description: localGoal.description,
        target: Math.max(1, toNumber(localGoal.target, 1)),
        deadline: localGoal.deadline,
      }),
    });
    goalIdMap.set(localGoal.id, created.id);
  }

  const sortedCheckins = localDb.checkins
    .slice()
    .sort((a, b) => String(a.createdAt || '').localeCompare(String(b.createdAt || '')));

  for (const checkin of sortedCheckins) {
    const remoteGoalId = goalIdMap.get(checkin.goalId);
    if (!remoteGoalId) continue;
    await remoteApi(`/api/goals/${remoteGoalId}/checkins`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        amount: Math.floor(toNumber(checkin.amount, 0)),
        note: checkin.note || '',
      }),
    });
  }

  for (const localGoal of sortedGoals) {
    const remoteGoalId = goalIdMap.get(localGoal.id);
    if (!remoteGoalId) continue;
    await remoteApi(`/api/goals/${remoteGoalId}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title: localGoal.title,
        description: localGoal.description,
        target: Math.max(1, Math.floor(toNumber(localGoal.target, 1))),
        current: Math.max(0, Math.floor(toNumber(localGoal.current, 0))),
        status: localGoal.status,
        deadline: localGoal.deadline,
      }),
    });
  }
}

async function api(path, options) {
  if (syncMode.value === 'online') {
    if (!isAuthenticated.value) return localApi(path, options);

    try {
      return await remoteApi(path, options);
    } catch (err) {
      if (err?.status === 401) {
        setAuthToken('');
        isAuthenticated.value = false;
      }
      if (!err?.isNetworkError) throw err;
      syncMode.value = 'offline';
      return localApi(path, options);
    }
  }

  if (isBrowserOnline()) {
    try {
      if (!isAuthenticated.value) {
        return localApi(path, options);
      }
      const remote = await remoteApi(path, options);
      syncMode.value = 'online';
      return remote;
    } catch (err) {
      if (err?.status === 401) {
        setAuthToken('');
        isAuthenticated.value = false;
        throw err;
      }
      if (!err?.isNetworkError) throw err;
      syncMode.value = 'offline';
    }
  }

  return localApi(path, options);
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

const handleOffline = () => {
  syncMode.value = 'offline';
};

const handleOnline = async () => {
  const isOnline = await detectOnlineSync();
  if (!isOnline) return;
  await checkAuthStatus();
  if (!isAuthenticated.value) {
    await loadDashboard();
    return;
  }
  await syncLocalToRemoteIfNeeded();
  await loadDashboard();
};

onMounted(() => {
  setApiBaseUrl(loadApiBaseUrl());
  hwid.value = getOrCreateHwid();
  const savedTheme = localStorage.getItem(getThemeStorageKey());
  const prefersDark = window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches;
  applyTheme(savedTheme || (prefersDark ? 'dark' : 'light'));

  window.addEventListener('offline', handleOffline);
  window.addEventListener('online', handleOnline);
  window.addEventListener('keydown', handleGlobalKeydown);

  detectOnlineSync()
    .then(async (isOnline) => {
      if (isOnline) {
        await checkAuthStatus();
        if (isAuthenticated.value) {
          await syncLocalToRemoteIfNeeded();
        }
      }
    })
    .finally(() => {
      loadDashboard();
    });
});

onBeforeUnmount(() => {
  window.removeEventListener('offline', handleOffline);
  window.removeEventListener('online', handleOnline);
  window.removeEventListener('keydown', handleGlobalKeydown);
});
</script>

<template>
  <main class="page">
    <section class="hero app-topbar">
      <div class="brand">
        <span class="brand-dot" aria-hidden="true"></span>
        <p class="brand-title">Obaika Tracker <small>v0.1.4</small></p>
      </div>

      <div class="hero-actions">
        <p class="sync-status" :class="syncMode">
          {{ syncMode === 'online' ? 'Online' : 'Offline' }}
        </p>
        <button type="button" @click="focusCreateGoal">Новая цель</button>
        <button class="theme-toggle" type="button" @click="toggleTheme">
          {{ theme === 'dark' ? 'Светлая' : 'Темная' }}
        </button>
        <button class="secondary settings-trigger" type="button" @click="openSettings">Настройки</button>
        <button
          v-if="isAuthenticated"
          class="secondary"
          type="button"
          @click="logout"
        >
          Выйти
        </button>
      </div>
    </section>

    <section class="card welcome-card">
      <div>
        <h1>Добро пожаловать в твой трекер</h1>
        <p class="subtitle">Создавай цели, отмечай прогресс и держи фокус на том, что важно каждый день.</p>
      </div>
      <div class="welcome-badges">
        <span>Локально и безопасно</span>
        <span>Синхронизация по HWID</span>
        <span>Бэкап в 1 клик</span>
      </div>
    </section>

    <Teleport to="body">
      <div v-if="settingsOpen" class="settings-overlay" @click.self="closeSettings">
        <section class="settings-modal card">
        <aside class="settings-sidebar">
          <div class="settings-sidebar-header">
            <p>Настройки</p>
          </div>
          <nav class="settings-sidebar-nav">
            <button :class="{ active: settingsSection === 'core' }" type="button" @click="setSettingsSection('core')">
              <svg class="sidebar-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"/></svg>
              Основные
            </button>
            <button :class="{ active: settingsSection === 'sync' }" type="button" @click="setSettingsSection('sync')">
              <svg class="sidebar-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M23 4v6h-6"/><path d="M1 20v-6h6"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/></svg>
              Синхронизация
            </button>
            <button :class="{ active: settingsSection === 'data' }" type="button" @click="setSettingsSection('data')">
              <svg class="sidebar-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M21 12c0 1.66-4 3-9 3s-9-1.34-9-3"/><path d="M3 5v14c0 1.66 4 3 9 3s9-1.34 9-3V5"/></svg>
              Данные
            </button>
            <button :class="{ active: settingsSection === 'security' }" type="button" @click="setSettingsSection('security')">
              <svg class="sidebar-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z"/></svg>
              Безопасность
            </button>
          </nav>
          <div class="settings-version">v0.1.4</div>
        </aside>

        <div class="settings-content">
          <header class="settings-head">
            <div>
              <h2>Параметры</h2>
              <p>Настройте приложение под себя</p>
            </div>
            <button class="secondary" type="button" @click="closeSettings">Закрыть</button>
          </header>

          <div class="settings-subtabs">
            <button
              v-for="sub in activeSubsections"
              :key="sub.id"
              class="subtab-btn"
              :class="{ active: settingsSubsection === sub.id }"
              type="button"
              @click="settingsSubsection = sub.id"
            >
              <svg v-if="sub.icon === 'globe'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="10"/><line x1="2" y1="12" x2="22" y2="12"/><path d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10z"/></svg>
              <svg v-if="sub.icon === 'palette'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="13.5" cy="6.5" r="0.5"/><circle cx="17.5" cy="10.5" r="0.5"/><circle cx="8.5" cy="7.5" r="0.5"/><circle cx="6.5" cy="12.5" r="0.5"/><path d="M12 2C6.5 2 2 6.5 2 12s4.5 10 10 10c.926 0 1.648-.746 1.648-1.688 0-.437-.18-.835-.437-1.125-.29-.289-.438-.652-.438-1.125a1.64 1.64 0 0 1 1.668-1.668h1.996c3.051 0 5.555-2.503 5.555-5.555C21.965 6.012 17.461 2 12 2z"/></svg>
              <svg v-if="sub.icon === 'zap'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2"/></svg>
              <svg v-if="sub.icon === 'wifi'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M5 12.55a11 11 0 0 1 14.08 0"/><path d="M1.42 9a16 16 0 0 1 21.16 0"/><path d="M8.53 16.11a6 6 0 0 1 6.95 0"/><line x1="12" y1="20" x2="12.01" y2="20"/></svg>
              <svg v-if="sub.icon === 'refresh'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M23 4v6h-6"/><path d="M1 20v-6h6"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/></svg>
              <svg v-if="sub.icon === 'download'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/></svg>
              <svg v-if="sub.icon === 'database'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M21 12c0 1.66-4 3-9 3s-9-1.34-9-3"/><path d="M3 5v14c0 1.66 4 3 9 3s9-1.34 9-3V5"/></svg>
              <svg v-if="sub.icon === 'log-in'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M15 3h4a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2h-4"/><polyline points="10 17 15 12 10 7"/><line x1="15" y1="12" x2="3" y2="12"/></svg>
              <svg v-if="sub.icon === 'lock'" class="subtab-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="11" width="18" height="11" rx="2" ry="2"/><path d="M7 11V7a5 5 0 0 1 10 0v4"/></svg>
              {{ sub.label }}
            </button>
          </div>

          <div class="settings-grid">
            <article v-if="settingsSection === 'core' && settingsSubsection === 'server'" class="settings-block">
              <h3>Сервер синхронизации</h3>
              <p class="subtitle">Текущий адрес backend для online режима.</p>
              <input v-model="settingsForm.apiBaseUrl" type="text" placeholder="https://example.com:3000" />
              <div class="settings-actions">
                <button type="button" @click="saveApiSettings">Сохранить сервер</button>
                <button class="secondary" type="button" @click="testConnection">Проверить</button>
                <button class="secondary" type="button" @click="resetApiSettings">По умолчанию</button>
              </div>
            </article>

            <article v-if="settingsSection === 'core' && settingsSubsection === 'appearance'" class="settings-block">
              <h3>Внешний вид</h3>
              <p class="subtitle">Подстрой интерфейс под себя.</p>
              <div class="settings-actions">
                <button class="theme-toggle" type="button" @click="toggleTheme">
                  {{ theme === 'dark' ? 'Светлая тема' : 'Темная тема' }}
                </button>
              </div>
            </article>

            <article v-if="settingsSection === 'core' && settingsSubsection === 'quick'" class="settings-block">
              <h3>Быстрый старт</h3>
              <p class="subtitle">Действия, которые чаще всего нужны каждый день.</p>
              <div class="settings-actions">
                <button type="button" @click="focusCreateGoal">Создать новую цель</button>
                <button class="secondary" type="button" @click="loadDashboard">Обновить дашборд</button>
              </div>
            </article>

            <article v-if="settingsSection === 'sync' && settingsSubsection === 'status'" class="settings-block">
              <h3>Синхронизация</h3>
              <p class="subtitle">Ручной запуск синка и состояние доступа.</p>
              <p class="settings-note">
                Режим: {{ syncMode === 'online' ? 'online' : 'offline' }} ·
                Авторизация: {{ isAuthenticated ? 'выполнена' : 'нет' }}
              </p>
            </article>

            <article v-if="settingsSection === 'sync' && settingsSubsection === 'manual'" class="settings-block">
              <h3>Ручной режим</h3>
              <p class="subtitle">Запусти синхронизацию вручную в любой момент.</p>
              <div class="settings-actions">
                <button type="button" @click="forceSyncNow">Синхронизировать сейчас</button>
              </div>
            </article>

            <article v-if="settingsSection === 'data' && settingsSubsection === 'backup'" class="settings-block">
              <h3>Данные устройства</h3>
              <p class="subtitle">Экспорт и импорт локального бэкапа.</p>
              <input
                ref="importFileInput"
                class="hidden-input"
                type="file"
                accept="application/json"
                @change="handleImportData"
              />
              <div class="settings-actions">
                <button type="button" @click="exportLocalData">Экспорт JSON</button>
                <button class="secondary" type="button" :disabled="importingData" @click="requestImportData">
                  {{ importingData ? 'Импорт...' : 'Импорт JSON' }}
                </button>
                <button class="danger" type="button" @click="clearLocalData">Очистить локальные данные</button>
              </div>
            </article>

            <article v-if="settingsSection === 'data' && settingsSubsection === 'storage'" class="settings-block">
              <h3>Хранилище</h3>
              <p class="subtitle">Идентификатор устройства и текущий сервер.</p>
              <p class="settings-note">HWID: {{ hwidMasked }}</p>
              <p class="settings-note">Backend: {{ currentApiBaseUrl }}</p>
            </article>

            <article v-if="settingsSection === 'security' && settingsSubsection === 'auth'" class="settings-block">
              <h3>Безопасность</h3>
              <p class="subtitle">Авторизация и пароль для текущего HWID.</p>

              <template v-if="!isAuthenticated">
                <form class="goal-form" @submit.prevent="authMode === 'register' ? registerWithPassword() : loginWithPassword()">
                  <input
                    v-model="authForm.password"
                    type="password"
                    minlength="6"
                    placeholder="Введите пароль"
                    autocomplete="current-password"
                  />
                  <input
                    v-if="authMode === 'register'"
                    v-model="authForm.confirmPassword"
                    type="password"
                    minlength="6"
                    placeholder="Повторите пароль"
                    autocomplete="new-password"
                  />
                  <div class="settings-actions">
                    <button :disabled="authLoading" type="submit">
                      {{ authLoading ? 'Проверка...' : authMode === 'register' ? 'Создать пароль' : 'Войти' }}
                    </button>
                    <button
                      class="secondary"
                      type="button"
                      :disabled="authLoading"
                      @click="authMode = authMode === 'register' ? 'login' : 'register'; authError = ''"
                    >
                      {{ authMode === 'register' ? 'У меня уже есть пароль' : 'Первая настройка' }}
                    </button>
                  </div>
                </form>
                <p v-if="authError" class="error">{{ authError }}</p>
              </template>

              <template v-else>
                <p class="settings-note">Авторизация выполнена для этого устройства.</p>
                <div class="settings-actions">
                  <button class="secondary" type="button" @click="logout">Выйти</button>
                </div>
              </template>
            </article>

            <article v-if="settingsSection === 'security' && settingsSubsection === 'password'" class="settings-block">
              <h3>Смена пароля</h3>
              <p class="subtitle">Обнови пароль для текущего HWID.</p>

              <template v-if="isAuthenticated">
                <input v-model="passwordForm.oldPassword" type="password" placeholder="Текущий пароль" />
                <input v-model="passwordForm.newPassword" type="password" placeholder="Новый пароль" />
                <input v-model="passwordForm.confirmPassword" type="password" placeholder="Повтор нового пароля" />
                <div class="settings-actions">
                  <button
                    type="button"
                    :disabled="changingPassword || syncMode !== 'online'"
                    @click="changePassword"
                  >
                    {{ changingPassword ? 'Сохранение...' : 'Сменить пароль' }}
                  </button>
                </div>
              </template>

              <template v-else>
                <p class="settings-note">Сначала войди в подразделе «Вход».</p>
              </template>
            </article>
          </div>

          <p v-if="settingsMessage" class="settings-success">{{ settingsMessage }}</p>
          <p v-if="settingsError" class="error">{{ settingsError }}</p>
        </div>
      </section>
      </div>
    </Teleport>

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
        <input ref="goalTitleInput" v-model="goalForm.title" type="text" placeholder="Например: Читать каждый день" />
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

    <button
      class="mobile-settings-fab"
      type="button"
      @click="openSettings"
    >
      Настройки
    </button>
  </main>
</template>
