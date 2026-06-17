// ============================================================
// APP.JS — Core Logic, Data Management, Routing
// ============================================================

// ─── EMISSION FACTORS (kg CO₂e per unit) ────────────────────
const EMISSION_FACTORS = {
  transportation: {
    car:    { factor: 0.21,  unit: 'km',   label: 'Car Travel',   icon: '🚗' },
    bike:   { factor: 0.11,  unit: 'km',   label: 'Motorbike',    icon: '🏍️' },
    bus:    { factor: 0.089, unit: 'km',   label: 'Bus',          icon: '🚌' },
    train:  { factor: 0.041, unit: 'km',   label: 'Train',        icon: '🚆' },
    flights:{ factor: 0.255, unit: 'km',   label: 'Flight',       icon: '✈️' }
  },
  food: {
    vegetarian: { factor: 1.5,  unit: 'meals', label: 'Vegetarian Meal', icon: '🥗' },
    vegan:      { factor: 0.8,  unit: 'meals', label: 'Vegan Meal',      icon: '🌿' },
    chicken:    { factor: 3.5,  unit: 'meals', label: 'Chicken Meal',    icon: '🍗' },
    beef:       { factor: 13.3, unit: 'meals', label: 'Beef Meal',       icon: '🥩' }
  },
  energy: {
    electricity: { factor: 0.82, unit: 'kWh',   label: 'Electricity',  icon: '⚡' },
    ac:          { factor: 1.2,  unit: 'hours',  label: 'AC / Heater',  icon: '❄️' },
    gas:         { factor: 0.44, unit: 'hours',  label: 'Cooking Gas',  icon: '🔥' }
  },
  shopping: {
    clothing:   { factor: 10.0, unit: 'items', label: 'Clothing',         icon: '👕' },
    electronics:{ factor: 70.0, unit: 'items', label: 'Electronics',      icon: '📱' },
    household:  { factor: 15.0, unit: 'items', label: 'Household Goods',  icon: '🏠' }
  },
  waste: {
    recycling:  { factor: -0.5, unit: 'kg', label: 'Recycling',   icon: '♻️' },
    composting: { factor: -0.3, unit: 'kg', label: 'Composting',  icon: '🌱' },
    trash:      { factor: 0.57, unit: 'kg', label: 'Trash',       icon: '🗑️' }
  }
};

const CATEGORY_META = {
  transportation: { label: 'Transportation', icon: '🚗', color: '#3b82f6' },
  food:           { label: 'Food & Diet',    icon: '🍽️', color: '#f97316' },
  energy:         { label: 'Home Energy',    icon: '⚡', color: '#eab308' },
  shopping:       { label: 'Shopping',       icon: '🛍️', color: '#a855f7' },
  waste:          { label: 'Waste',          icon: '🗑️', color: '#14b8a6' }
};

// ─── STATE ───────────────────────────────────────────────────
let appState = {
  currentPage: 'dashboard',
  currentDate: getTodayKey(),
  logData: {},      // { 'YYYY-MM-DD': { transportation: {car:0,...,total:0}, food:{...}, ... } }
  streak: 0,
  lastLogDate: null
};

// ─── UTILS ───────────────────────────────────────────────────
function getTodayKey() {
  return new Date().toISOString().split('T')[0];
}

function saveState() {
  localStorage.setItem('ecotrack_data', JSON.stringify(appState.logData));
  localStorage.setItem('ecotrack_streak', appState.streak);
  localStorage.setItem('ecotrack_lastlog', appState.lastLogDate || '');
}

function loadState() {
  const data = localStorage.getItem('ecotrack_data');
  if (data) appState.logData = JSON.parse(data);
  appState.streak = parseInt(localStorage.getItem('ecotrack_streak') || '0');
  appState.lastLogDate = localStorage.getItem('ecotrack_lastlog') || null;
  updateStreak();
}

function updateStreak() {
  const today = getTodayKey();
  const yesterday = new Date(); yesterday.setDate(yesterday.getDate() - 1);
  const yKey = yesterday.toISOString().split('T')[0];

  if (appState.lastLogDate === today) return;
  if (appState.lastLogDate === yKey) {
    appState.streak += 1;
  } else if (appState.lastLogDate && appState.lastLogDate !== today) {
    appState.streak = 0;
  }
}

function getTodayData() {
  const key = getTodayKey();
  if (!appState.logData[key]) {
    appState.logData[key] = {
      transportation: { car:0, bike:0, bus:0, train:0, flights:0, total:0 },
      food:           { vegetarian:0, vegan:0, chicken:0, beef:0, total:0 },
      energy:         { electricity:0, ac:0, gas:0, total:0 },
      shopping:       { clothing:0, electronics:0, household:0, total:0 },
      waste:          { recycling:0, composting:0, trash:0, total:0 }
    };
  }
  return appState.logData[key];
}

function calcCategoryTotal(category, values) {
  let total = 0;
  Object.entries(values).forEach(([key, qty]) => {
    if (key === 'total') return;
    const ef = EMISSION_FACTORS[category]?.[key];
    if (ef) total += qty * ef.factor;
  });
  return Math.max(total, 0); // Waste can go negative but we floor at 0 for display
}

function calcRawCategoryTotal(category, values) {
  let total = 0;
  Object.entries(values).forEach(([key, qty]) => {
    if (key === 'total') return;
    const ef = EMISSION_FACTORS[category]?.[key];
    if (ef) total += qty * ef.factor;
  });
  return total; // Allow negative for waste offsets
}

function getAllCategoryTotals(dayData) {
  const result = {};
  Object.keys(EMISSION_FACTORS).forEach(cat => {
    result[cat] = Math.max(calcRawCategoryTotal(cat, dayData[cat] || {}), 0);
  });
  return result;
}

function getGrandTotal(dayData) {
  return Object.keys(EMISSION_FACTORS).reduce((sum, cat) => {
    return sum + calcRawCategoryTotal(cat, dayData[cat] || {});
  }, 0);
}

function getLast7Days() {
  const result = [];
  for (let i = 6; i >= 0; i--) {
    const d = new Date(); d.setDate(d.getDate() - i);
    const key = d.toISOString().split('T')[0];
    result.push(appState.logData[key] || {
      transportation:{total:0}, food:{total:0}, energy:{total:0}, shopping:{total:0}, waste:{total:0}
    });
  }
  return result;
}

function getHistoryDays(n) {
  const result = [];
  const keys = Object.keys(appState.logData).sort().slice(-n);
  keys.forEach(k => result.push(appState.logData[k]));
  return result;
}

function getCarbonRating(totalKg) {
  if (totalKg <= 5)  return { label: 'Excellent', color: '#22c55e', emoji: '🌟' };
  if (totalKg <= 10) return { label: 'Good',      color: '#84cc16', emoji: '✅' };
  if (totalKg <= 20) return { label: 'Average',   color: '#eab308', emoji: '⚠️' };
  if (totalKg <= 35) return { label: 'High',      color: '#f97316', emoji: '🔴' };
  return              { label: 'Critical',  color: '#ef4444', emoji: '🚨' };
}

// ─── NAVIGATION ──────────────────────────────────────────────
function navigateTo(page) {
  document.querySelectorAll('.page').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active'));

  const target = document.getElementById(`page-${page}`);
  if (target) {
    target.classList.add('active');
    // animate in
    target.style.opacity = '0';
    target.style.transform = 'translateY(12px)';
    setTimeout(() => {
      target.style.transition = 'opacity 0.3s ease, transform 0.3s ease';
      target.style.opacity = '1';
      target.style.transform = 'translateY(0)';
    }, 10);
  }

  const navBtn = document.getElementById(`nav-${page}`);
  if (navBtn) navBtn.classList.add('active');

  appState.currentPage = page;

  // Render page-specific content
  switch(page) {
    case 'dashboard':  renderDashboard(); break;
    case 'log':        renderLogPage(); break;
    case 'history':    renderHistoryPage(); break;
    case 'suggestions':renderSuggestionsPage(); break;
    case 'settings':   renderSettingsPage(); break;
    case 'install':    break; // static page, no render needed
  }
}

// ─── DASHBOARD ───────────────────────────────────────────────
function renderDashboard() {
  const today = getTodayData();
  const catTotals = getAllCategoryTotals(today);
  const grand = getGrandTotal(today);
  const rating = getCarbonRating(grand);
  const weekData = getLast7Days();

  // Grand total
  const el = document.getElementById('dash-total');
  if (el) {
    el.innerHTML = `<span class="total-num">${grand.toFixed(2)}</span><span class="total-unit">kg CO₂e today</span>`;
    el.style.color = rating.color;
  }

  // Rating badge
  const badge = document.getElementById('dash-rating');
  if (badge) {
    badge.textContent = `${rating.emoji} ${rating.label}`;
    badge.style.background = rating.color + '22';
    badge.style.color = rating.color;
    badge.style.border = `1px solid ${rating.color}55`;
  }

  // Streak
  const streakEl = document.getElementById('dash-streak');
  if (streakEl) streakEl.textContent = `🔥 ${appState.streak} day streak`;

  // Doughnut
  renderDoughnutChart('dashboard-doughnut', catTotals);

  // Category bars
  renderCategoryBars(catTotals);

  // Weekly bar chart
  const week7 = weekData.map(d => {
    const out = {};
    Object.keys(CATEGORY_META).forEach(cat => {
      out[cat] = { total: calcRawCategoryTotal(cat, d[cat] || {}) };
    });
    return out;
  });
  renderBarChart('weekly-bar-chart', week7);

  // Weekly insight
  const weekInsights = getWeeklyInsight(week7);
  const insightEl = document.getElementById('weekly-insight');
  if (insightEl && weekInsights.length > 0) {
    insightEl.innerHTML = weekInsights.map(i => `<p>${i}</p>`).join('');
  }

  // Annual projection
  const days = Object.keys(appState.logData).length || 1;
  const totalAll = Object.values(appState.logData).reduce((sum, d) => sum + getGrandTotal(d), 0);
  const dailyAvg = totalAll / days;
  const annualKg = dailyAvg * 365;
  const projEl = document.getElementById('annual-projection');
  if (projEl) {
    projEl.innerHTML = `<span>${(annualKg / 1000).toFixed(2)}</span> <small>tons CO₂e / year projected</small>`;
    projEl.style.color = annualKg > 8000 ? '#ef4444' : annualKg > 4700 ? '#f97316' : '#22c55e';
  }

  // Update GPS permission badge
  const permBadge = document.getElementById('tracker-perm-badge');
  if (permBadge && typeof Tracker !== 'undefined') {
    const ts = Tracker.getState();
    const pMap = {
      granted:     { text: '📍 GPS Active',   color: '#22c55e' },
      denied:      { text: '🚫 GPS Denied',   color: '#ef4444' },
      unavailable: { text: '⚠️ GPS N/A',      color: '#f97316' },
      unknown:     { text: '🛰 GPS Ready',    color: '#94a3b8' }
    };
    const p = pMap[ts.permissionState] || pMap.unknown;
    permBadge.textContent = p.text;
    permBadge.style.color = p.color;
    permBadge.style.borderColor = p.color + '44';
  }

  // Last-updated timestamp on dashboard
  const tsEl = document.getElementById('dash-last-updated');
  if (tsEl) {
    tsEl.textContent = 'Updated ' + new Date().toLocaleTimeString('en-IN', { hour:'2-digit', minute:'2-digit' });
  }
}

// ─── LOG PAGE ────────────────────────────────────────────────
let activeLogCategory = 'transportation';

function renderLogPage() {
  // Highlight active category tab
  document.querySelectorAll('.cat-tab').forEach(t => t.classList.remove('active'));
  const activeTab = document.querySelector(`.cat-tab[data-cat="${activeLogCategory}"]`);
  if (activeTab) activeTab.classList.add('active');

  renderLogFields(activeLogCategory);
}

function renderLogFields(category) {
  const container = document.getElementById('log-fields');
  if (!container) return;

  const today = getTodayData();
  const catData = today[category] || {};
  const factors = EMISSION_FACTORS[category];
  const meta = CATEGORY_META[category];

  container.innerHTML = `
    <div class="log-category-header">
      <span class="cat-icon-big">${meta.icon}</span>
      <h2>${meta.label}</h2>
      <p class="cat-desc">${getCategoryDesc(category)}</p>
    </div>
    ${Object.entries(factors).map(([key, info]) => `
      <div class="log-item glass-card" id="logitem-${key}">
        <div class="log-item-info">
          <span class="log-icon">${info.icon}</span>
          <div>
            <div class="log-label">${info.label}</div>
            <div class="log-unit">Factor: ${info.factor} kg CO₂e / ${info.unit}</div>
          </div>
        </div>
        <div class="log-controls">
          <button class="qty-btn minus" onclick="adjustQty('${category}','${key}',-1)">−</button>
          <input type="number" class="qty-input" id="qty-${key}" 
            value="${catData[key] || 0}" min="0" step="${info.unit === 'km' ? 5 : 1}"
            onchange="setQty('${category}','${key}',this.value)"
            oninput="setQty('${category}','${key}',this.value)">
          <button class="qty-btn plus" onclick="adjustQty('${category}','${key}',1)">+</button>
        </div>
        <div class="log-item-emission" id="emission-${key}">
          ${((catData[key] || 0) * info.factor).toFixed(2)} kg CO₂e
        </div>
      </div>
    `).join('')}
    <div class="log-category-total glass-card" style="border-color: ${meta.color}44;">
      <span>Category Total</span>
      <span id="cat-total-display" style="color:${meta.color}; font-weight:700;">
        ${calcCategoryTotal(category, catData).toFixed(2)} kg CO₂e
      </span>
    </div>
    <button class="save-log-btn" onclick="saveLog()">💾 Save Today's Log</button>
  `;
}

function getCategoryDesc(cat) {
  const descs = {
    transportation: 'Enter distances traveled by each mode today',
    food: 'Log meals consumed today',
    energy: 'Track home energy usage today',
    shopping: 'Record items purchased today',
    waste: 'Track waste generated and managed today'
  };
  return descs[cat] || '';
}

function adjustQty(category, key, delta) {
  const today = getTodayData();
  const info = EMISSION_FACTORS[category][key];
  const step = info.unit === 'km' ? 5 : 1;
  const current = today[category][key] || 0;
  const newVal = Math.max(0, current + delta * step);
  today[category][key] = newVal;

  const input = document.getElementById(`qty-${key}`);
  if (input) input.value = newVal;

  updateEmissionDisplay(category, key, newVal, info.factor);
  updateCatTotal(category);
  saveState();
  bounceElement(`logitem-${key}`);
}

function setQty(category, key, val) {
  const today = getTodayData();
  const num = Math.max(0, parseFloat(val) || 0);
  today[category][key] = num;
  const info = EMISSION_FACTORS[category][key];
  updateEmissionDisplay(category, key, num, info.factor);
  updateCatTotal(category);
  saveState();
}

function updateEmissionDisplay(category, key, qty, factor) {
  const el = document.getElementById(`emission-${key}`);
  if (el) {
    const co2 = qty * factor;
    el.textContent = `${co2.toFixed(2)} kg CO₂e`;
    el.style.color = co2 > 10 ? '#ef4444' : co2 > 5 ? '#f97316' : co2 < 0 ? '#22c55e' : '#94a3b8';
  }
}

function updateCatTotal(category) {
  const today = getTodayData();
  const total = calcCategoryTotal(category, today[category]);
  const el = document.getElementById('cat-total-display');
  if (el) el.textContent = `${total.toFixed(2)} kg CO₂e`;
  today[category].total = total;
}

function saveLog() {
  const today = getTodayData();
  Object.keys(EMISSION_FACTORS).forEach(cat => {
    today[cat].total = calcCategoryTotal(cat, today[cat]);
  });
  appState.lastLogDate = getTodayKey();
  appState.streak = Math.max(1, appState.streak);
  saveState();
  showToast('✅ Log saved! Great job tracking your footprint.');
  renderDashboard();
}

function bounceElement(id) {
  const el = document.getElementById(id);
  if (!el) return;
  el.style.transform = 'scale(1.02)';
  setTimeout(() => el.style.transform = '', 150);
}

// ─── HISTORY PAGE ────────────────────────────────────────────
function renderHistoryPage() {
  const history = getHistoryDays(30);
  if (history.length === 0) {
    document.getElementById('history-empty').style.display = 'block';
    return;
  }
  document.getElementById('history-empty').style.display = 'none';
  renderTrendChart('trend-chart', history);

  // Populate history list
  const list = document.getElementById('history-list');
  if (!list) return;
  const keys = Object.keys(appState.logData).sort().slice(-30).reverse();
  list.innerHTML = keys.map(key => {
    const d = appState.logData[key];
    const total = getGrandTotal(d);
    const rating = getCarbonRating(total);
    const date = new Date(key);
    const label = date.toLocaleDateString('en-IN', { weekday:'short', month:'short', day:'numeric' });
    return `
      <div class="history-row glass-card">
        <div class="history-date">
          <span class="hdate-label">${label}</span>
          <span class="hdate-key">${key === getTodayKey() ? 'Today' : ''}</span>
        </div>
        <div class="history-bar-wrap">
          <div class="history-bar-fill" style="width:${Math.min((total/50)*100, 100)}%; background:${rating.color}"></div>
        </div>
        <div class="history-total" style="color:${rating.color}">
          ${rating.emoji} ${total.toFixed(1)} kg
        </div>
      </div>
    `;
  }).join('');
}

// ─── SUGGESTIONS PAGE ────────────────────────────────────────
function renderSuggestionsPage() {
  const today = getTodayData();
  const container = document.getElementById('suggestions-list');
  if (!container) return;

  const data = { today: {} };
  Object.keys(EMISSION_FACTORS).forEach(cat => {
    data.today[cat] = today[cat] || {};
  });
  // Flatten for suggestions engine
  data.today.transportation = today.transportation;
  data.today.food = today.food;
  data.today.energy = today.energy;
  data.today.shopping = today.shopping;
  data.today.waste = today.waste;

  const suggestions = generateSuggestions(data);

  const typeStyles = {
    warning:  { bg: '#ef444411', border: '#ef444433', badge: '⚠️ Action Needed' },
    info:     { bg: '#3b82f611', border: '#3b82f633', badge: '💡 Tip' },
    positive: { bg: '#22c55e11', border: '#22c55e33', badge: '✅ Keep It Up' },
    general:  { bg: '#a855f711', border: '#a855f733', badge: '🌍 Did You Know?' }
  };

  container.innerHTML = suggestions.map((s, i) => {
    const style = typeStyles[s.type] || typeStyles.general;
    return `
      <div class="suggestion-card glass-card" style="background:${style.bg}; border-color:${style.border}; animation-delay:${i * 0.08}s">
        <div class="suggestion-badge">${style.badge}</div>
        <p>${s.tip}</p>
      </div>
    `;
  }).join('');

  // Weekly insights section
  const weekData = getLast7Days().map(d => {
    const out = {};
    Object.keys(CATEGORY_META).forEach(cat => {
      out[cat] = { total: calcRawCategoryTotal(cat, d[cat] || {}) };
    });
    return out;
  });
  const insights = getWeeklyInsight(weekData);
  const insEl = document.getElementById('weekly-insights-sugg');
  if (insEl) {
    insEl.innerHTML = insights.map(i => `<div class="insight-chip">${i}</div>`).join('');
  }
}

// ─── SETTINGS PAGE ───────────────────────────────────────────
function renderSettingsPage() {
  const totalDays = Object.keys(appState.logData).length;
  const allTotals = Object.values(appState.logData).map(d => getGrandTotal(d));
  const avg = allTotals.length > 0 ? allTotals.reduce((a, b) => a + b, 0) / allTotals.length : 0;
  const best = allTotals.length > 0 ? Math.min(...allTotals) : 0;

  document.getElementById('stat-days').textContent = totalDays;
  document.getElementById('stat-avg').textContent = avg.toFixed(2) + ' kg';
  document.getElementById('stat-best').textContent = best.toFixed(2) + ' kg';
  document.getElementById('stat-streak').textContent = appState.streak + ' days';

  // Render profile card
  if (typeof Auth !== 'undefined') Auth.renderProfileCard();
}

function clearAllData() {
  if (confirm('⚠️ This will permanently delete all your tracking data. Are you sure?')) {
    localStorage.clear();
    appState.logData = {};
    appState.streak = 0;
    appState.lastLogDate = null;
    showToast('🗑️ All data cleared.');
    navigateTo('dashboard');
  }
}

function exportData() {
  const json = JSON.stringify(appState.logData, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `ecotrack-export-${getTodayKey()}.json`;
  a.click();
  URL.revokeObjectURL(url);
  showToast('📥 Data exported successfully!');
}

// ─── TOAST ───────────────────────────────────────────────────
function showToast(msg) {
  let toast = document.getElementById('toast');
  if (!toast) {
    toast = document.createElement('div');
    toast.id = 'toast';
    document.body.appendChild(toast);
  }
  toast.innerHTML = msg;
  toast.classList.add('show');
  setTimeout(() => toast.classList.remove('show'), 3200);
}

// ─── SHOW INSTALL GUIDE ───────────────────────────────────────────────
function showInstallGuide() {
  // If app is visible, navigate to install page
  const app = document.getElementById('app');
  if (app && app.style.display !== 'none') {
    navigateTo('install');
    return;
  }
  // Otherwise hide login and show install page
  const overlay = document.getElementById('login-overlay');
  if (overlay) overlay.style.display = 'none';
  if (app) {
    app.style.display = 'flex';
    app.style.flexDirection = 'column';
  }
  navigateTo('install');
}

// ─── INIT ────────────────────────────────────────────────────
document.addEventListener('DOMContentLoaded', () => {
  loadState();
  getTodayData(); // ensure today's slot exists

  // ── Inject last-updated stamp into header ──────────────────
  const hdrRight = document.getElementById('header-right');
  if (hdrRight) {
    const tsSpan = document.createElement('span');
    tsSpan.id = 'dash-last-updated';
    tsSpan.style.cssText = 'font-size:10px;color:var(--text-muted);margin-left:6px;';
    hdrRight.appendChild(tsSpan);
  }

  // Nav buttons
  document.querySelectorAll('.nav-item').forEach(btn => {
    btn.addEventListener('click', () => navigateTo(btn.dataset.page));
  });

  // Category tabs in log page
  document.querySelectorAll('.cat-tab').forEach(tab => {
    tab.addEventListener('click', () => {
      activeLogCategory = tab.dataset.cat;
      renderLogPage();
    });
  });

  // Register service worker
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js').catch(() => {});
  }

  // ── Initialise Auth (login check) ────────────────────────
  const isLoggedIn = (typeof Auth !== 'undefined') ? Auth.init() : true;

  // Only proceed with app init if already logged in
  // (if not logged in, Auth.init() shows the login screen)
  if (!isLoggedIn) return;

  // ── Initialise GPS sensor tracker ─────────────────────────
  if (typeof Tracker !== 'undefined') {
    Tracker.init();
  }

  // ── Load dashboard IMMEDIATELY with all saved data ─────────
  navigateTo('dashboard');

  // ── Auto-refresh dashboard every 30 s (live data) ──────────
  setInterval(() => {
    if (appState.currentPage === 'dashboard') {
      renderDashboard();
    }
  }, 30000);

  // ── Refresh on page visibility change (app re-open) ────────
  document.addEventListener('visibilitychange', () => {
    if (!document.hidden) {
      // Day may have rolled over
      const newKey = getTodayKey();
      if (newKey !== appState.currentDate) {
        appState.currentDate = newKey;
        getTodayData();
        showToast('🌅 New day! Fresh tracking started.');
      }
      if (appState.currentPage === 'dashboard') renderDashboard();
    }
  });

  // Add install prompt for PWA
  let deferredPrompt;
  window.addEventListener('beforeinstallprompt', e => {
    e.preventDefault();
    deferredPrompt = e;
    const installBtn = document.getElementById('install-btn');
    if (installBtn) {
      installBtn.style.display = 'flex';
      installBtn.onclick = () => {
        deferredPrompt.prompt();
        deferredPrompt.userChoice.then(() => { installBtn.style.display = 'none'; });
      };
    }
  });
});

AUTH FILE
// ============================================================
// AUTH.JS — User Login, Profile, Session Management
// ============================================================

const Auth = (() => {

  const STORAGE_KEY = 'ecotrack_user';

  // ─── SAVE / LOAD ─────────────────────────────────────────
  function saveUser(user) {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(user));
  }

  function loadUser() {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      return raw ? JSON.parse(raw) : null;
    } catch { return null; }
  }

  function clearUser() {
    localStorage.removeItem(STORAGE_KEY);
  }

  // ─── AVATAR INITIALS ──────────────────────────────────────
  function getInitials(name) {
    return name.trim().split(/\s+/)
      .slice(0, 2)
      .map(w => w[0].toUpperCase())
      .join('');
  }

  function getAvatarColor(name) {
    const colors = [
      '#22c55e','#3b82f6','#a855f7','#f97316',
      '#14b8a6','#ec4899','#eab308','#06b6d4'
    ];
    let hash = 0;
    for (let c of name) hash = (hash * 31 + c.charCodeAt(0)) & 0xffffffff;
    return colors[Math.abs(hash) % colors.length];
  }

  // ─── RENDER LOGIN SCREEN ──────────────────────────────────
  function showLoginScreen() {
    const overlay = document.getElementById('login-overlay');
    if (overlay) {
      overlay.style.display = 'flex';
      overlay.style.animation = 'fadeIn 0.4s ease';
    }
    // Hide main app
    const app = document.getElementById('app');
    if (app) app.style.display = 'none';

    // Bind form submit
    const form = document.getElementById('login-form');
    if (form) {
      form.onsubmit = handleLogin;
    }

    // Google button
    const gBtn = document.getElementById('google-signin-btn');
    if (gBtn) {
      gBtn.onclick = handleGoogleSignin;
    }
  }

  function hideLoginScreen() {
    const overlay = document.getElementById('login-overlay');
    if (overlay) {
      overlay.style.opacity = '0';
      overlay.style.transform = 'scale(0.96)';
      setTimeout(() => { overlay.style.display = 'none'; }, 300);
    }
    const app = document.getElementById('app');
    if (app) {
      app.style.display = 'flex';
      app.style.flexDirection = 'column';
      app.style.animation = 'fadeIn 0.4s ease';
    }
  }

  // ─── LOGIN HANDLERS ───────────────────────────────────────
  function handleLogin(e) {
    e.preventDefault();
    const name  = document.getElementById('login-name').value.trim();
    const email = document.getElementById('login-email').value.trim();
    const errEl = document.getElementById('login-error');

    // Validate
    if (!name || name.length < 2) {
      showError('Please enter your full name (min 2 characters).');
      return;
    }
    if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      showError('Please enter a valid Gmail address.');
      return;
    }
    if (!email.toLowerCase().endsWith('@gmail.com') && !email.includes('@')) {
      showError('Please use a Gmail address (e.g. name@gmail.com).');
      return;
    }

    const user = {
      name,
      email,
      initials: getInitials(name),
      avatarColor: getAvatarColor(name),
      loginTime: new Date().toISOString(),
      provider: 'email'
    };

    saveUser(user);
    completeLogin(user);
  }

  function handleGoogleSignin() {
    // Animate button to show loading
    const btn = document.getElementById('google-signin-btn');
    if (btn) {
      btn.innerHTML = '<span class="g-spinner"></span> Signing in…';
      btn.disabled = true;
    }

    // Show the email/name fields after 1s (simulate OAuth redirect return)
    setTimeout(() => {
      if (btn) {
        btn.innerHTML = `<img src="https://www.google.com/favicon.ico" width="18" style="border-radius:2px"> Sign in with Google`;
        btn.disabled = false;
      }
      // Show manual fields with pre-filled hint
      const formSection = document.getElementById('login-manual-section');
      if (formSection) {
        formSection.style.display = 'block';
        formSection.style.animation = 'slideUp 0.3s ease';
        document.getElementById('login-email').placeholder = 'your@gmail.com';
        document.getElementById('login-email').focus();
        showError('Enter your Gmail and name to continue ↓', 'info');
      }
    }, 1200);
  }

  function showError(msg, type = 'error') {
    const el = document.getElementById('login-error');
    if (!el) return;
    el.textContent = msg;
    el.style.color = type === 'info' ? '#22c55e' : '#f87171';
    el.style.display = 'block';
  }

  function completeLogin(user) {
    hideLoginScreen();
    updateHeaderProfile(user);
    if (typeof showToast === 'function') {
      showToast(`👋 Welcome, ${user.name.split(' ')[0]}!`);
    }
    if (typeof navigateTo === 'function') {
      navigateTo('dashboard');
    }
  }

  // ─── UPDATE HEADER PROFILE ────────────────────────────────
  function updateHeaderProfile(user) {
    const avatarEl = document.getElementById('header-avatar');
    if (avatarEl) {
      avatarEl.textContent  = user.initials;
      avatarEl.style.background = user.avatarColor;
      avatarEl.title = `${user.name}\n${user.email}`;
      avatarEl.style.display = 'flex';
    }

    const nameEl = document.getElementById('header-username');
    if (nameEl) {
      nameEl.textContent = user.name.split(' ')[0]; // First name only
    }
  }

  // ─── LOGOUT ───────────────────────────────────────────────
  function logout() {
    if (!confirm('👋 Sign out of EcoTrack? Your tracking data will be preserved.')) return;
    clearUser();
    // Reset header
    const avatarEl = document.getElementById('header-avatar');
    if (avatarEl) avatarEl.style.display = 'none';
    const nameEl = document.getElementById('header-username');
    if (nameEl) nameEl.textContent = '';
    showLoginScreen();
    if (typeof showToast === 'function') showToast('👋 Signed out successfully.');
  }

  // ─── UPDATE SETTINGS PROFILE CARD ────────────────────────
  function renderProfileCard() {
    const user = loadUser();
    const el = document.getElementById('settings-profile-card');
    if (!el || !user) return;

    el.innerHTML = `
      <div class="profile-avatar-lg" style="background:${user.avatarColor};">${user.initials}</div>
      <div class="profile-info">
        <div class="profile-name">${user.name}</div>
        <div class="profile-email">${user.email}</div>
        <div class="profile-since">Member since ${new Date(user.loginTime).toLocaleDateString('en-IN', { month: 'long', year: 'numeric' })}</div>
      </div>
    `;
  }

  // ─── INIT ─────────────────────────────────────────────────
  function init() {
    const user = loadUser();
    if (user) {
      // Already logged in — show app directly
      const overlay = document.getElementById('login-overlay');
      if (overlay) overlay.style.display = 'none';
      const app = document.getElementById('app');
      if (app) app.style.display = 'flex';
      updateHeaderProfile(user);
      return true; // logged in
    } else {
      showLoginScreen();
      return false; // not logged in
    }
  }

  return {
    init,
    logout,
    loadUser,
    renderProfileCard,
    updateHeaderProfile
  };

})();

// ============================================================
// CHARTS.JS — Chart.js wrappers for EcoTrack visualizations
// ============================================================

let doughnutChart = null;
let barChart = null;
let trendChart = null;

const CATEGORY_COLORS = {
  transportation: '#3b82f6',
  food:           '#f97316',
  energy:         '#eab308',
  shopping:       '#a855f7',
  waste:          '#14b8a6'
};

const CATEGORY_LABELS = {
  transportation: '🚗 Transport',
  food:           '🍽️ Food',
  energy:         '⚡ Energy',
  shopping:       '🛍️ Shopping',
  waste:          '🗑️ Waste'
};

// Default Chart.js global settings
function applyChartDefaults() {
  Chart.defaults.color = '#94a3b8';
  Chart.defaults.borderColor = 'rgba(255,255,255,0.06)';
  Chart.defaults.font.family = "'Outfit', sans-serif";
}

// ─── DASHBOARD DOUGHNUT CHART ────────────────────────────────
function renderDoughnutChart(canvasId, categoryTotals) {
  applyChartDefaults();
  const canvas = document.getElementById(canvasId);
  if (!canvas) return;
  const ctx = canvas.getContext('2d');

  const labels = Object.keys(categoryTotals).map(k => CATEGORY_LABELS[k]);
  const data   = Object.values(categoryTotals);
  const colors = Object.keys(categoryTotals).map(k => CATEGORY_COLORS[k]);

  if (doughnutChart) doughnutChart.destroy();

  const total = data.reduce((a, b) => a + b, 0);

  doughnutChart = new Chart(ctx, {
    type: 'doughnut',
    data: {
      labels,
      datasets: [{
        data,
        backgroundColor: colors,
        borderColor: '#111827',
        borderWidth: 3,
        hoverBorderWidth: 0,
        hoverOffset: 8
      }]
    },
    options: {
      cutout: '72%',
      animation: { animateRotate: true, duration: 900, easing: 'easeInOutQuart' },
      plugins: {
        legend: { display: false },
        tooltip: {
          callbacks: {
            label: ctx => ` ${ctx.label}: ${ctx.parsed.toFixed(2)} kg CO₂e (${total > 0 ? ((ctx.parsed / total) * 100).toFixed(1) : 0}%)`
          },
          backgroundColor: 'rgba(17,24,39,0.95)',
          titleColor: '#f1f5f9',
          bodyColor: '#94a3b8',
          padding: 12,
          cornerRadius: 10
        }
      }
    }
  });

  // Draw centre text
  drawDoughnutCenter(canvas, total);
}

function drawDoughnutCenter(canvas, total) {
  // Using the plugin approach for centre text is done via a custom plugin in Chart.js
  // We'll use a CSS overlay instead (see HTML structure)
  const centerEl = document.getElementById('doughnut-center-text');
  if (centerEl) {
    centerEl.innerHTML = `<span class="center-value">${total.toFixed(1)}</span><span class="center-unit">kg CO₂e</span>`;
  }
}

// ─── WEEKLY BAR CHART ────────────────────────────────────────
function renderBarChart(canvasId, weeklyData) {
  applyChartDefaults();
  const canvas = document.getElementById(canvasId);
  if (!canvas) return;
  const ctx = canvas.getContext('2d');

  const days = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];
  const categories = ['transportation', 'food', 'energy', 'shopping', 'waste'];

  const datasets = categories.map(cat => ({
    label: CATEGORY_LABELS[cat],
    data: weeklyData.map(d => (d[cat] && d[cat].total) ? parseFloat(d[cat].total.toFixed(2)) : 0),
    backgroundColor: CATEGORY_COLORS[cat] + 'cc',
    borderColor: CATEGORY_COLORS[cat],
    borderWidth: 1.5,
    borderRadius: 6,
    borderSkipped: false
  }));

  if (barChart) barChart.destroy();

  barChart = new Chart(ctx, {
    type: 'bar',
    data: { labels: days.slice(0, weeklyData.length), datasets },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      interaction: { mode: 'index', intersect: false },
      scales: {
        x: {
          stacked: true,
          grid: { display: false },
          ticks: { color: '#94a3b8' }
        },
        y: {
          stacked: true,
          grid: { color: 'rgba(255,255,255,0.05)' },
          ticks: {
            color: '#94a3b8',
            callback: v => v + ' kg'
          }
        }
      },
      plugins: {
        legend: {
          display: true,
          position: 'bottom',
          labels: { boxWidth: 12, padding: 14, color: '#94a3b8', font: { size: 11 } }
        },
        tooltip: {
          backgroundColor: 'rgba(17,24,39,0.95)',
          titleColor: '#f1f5f9',
          bodyColor: '#94a3b8',
          padding: 12,
          cornerRadius: 10,
          callbacks: {
            footer: items => {
              const total = items.reduce((a, i) => a + i.parsed.y, 0);
              return `Total: ${total.toFixed(2)} kg CO₂e`;
            }
          }
        }
      },
      animation: { duration: 700, easing: 'easeInOutQuart' }
    }
  });
}

// ─── TREND LINE CHART ────────────────────────────────────────
function renderTrendChart(canvasId, historyData) {
  applyChartDefaults();
  const canvas = document.getElementById(canvasId);
  if (!canvas) return;
  const ctx = canvas.getContext('2d');

  const labels = historyData.map((_, i) => `Day ${i + 1}`);
  const totals = historyData.map(d => {
    const cats = ['transportation', 'food', 'energy', 'shopping', 'waste'];
    return parseFloat(cats.reduce((s, c) => s + ((d[c] && d[c].total) || 0), 0).toFixed(2));
  });

  if (trendChart) trendChart.destroy();

  const gradient = ctx.createLinearGradient(0, 0, 0, 200);
  gradient.addColorStop(0, 'rgba(34,197,94,0.3)');
  gradient.addColorStop(1, 'rgba(34,197,94,0)');

  trendChart = new Chart(ctx, {
    type: 'line',
    data: {
      labels,
      datasets: [{
        label: 'Daily CO₂e (kg)',
        data: totals,
        borderColor: '#22c55e',
        backgroundColor: gradient,
        borderWidth: 2.5,
        pointBackgroundColor: '#22c55e',
        pointBorderColor: '#111827',
        pointBorderWidth: 2,
        pointRadius: 5,
        tension: 0.4,
        fill: true
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        x: { grid: { display: false }, ticks: { color: '#94a3b8', maxTicksLimit: 8 } },
        y: {
          grid: { color: 'rgba(255,255,255,0.05)' },
          ticks: { color: '#94a3b8', callback: v => v + ' kg' }
        }
      },
      plugins: {
        legend: { display: false },
        tooltip: {
          backgroundColor: 'rgba(17,24,39,0.95)',
          titleColor: '#f1f5f9',
          bodyColor: '#94a3b8',
          padding: 12,
          cornerRadius: 10
        }
      },
      animation: { duration: 700 }
    }
  });
}

// ─── CATEGORY MINI PROGRESS BARS ────────────────────────────
function renderCategoryBars(categoryTotals) {
  const total = Object.values(categoryTotals).reduce((a, b) => a + b, 0);
  Object.entries(categoryTotals).forEach(([cat, val]) => {
    const bar = document.getElementById(`bar-${cat}`);
    const pct = total > 0 ? (val / total) * 100 : 0;
    if (bar) {
      bar.style.width = pct + '%';
      bar.style.background = CATEGORY_COLORS[cat];
    }
    const valEl = document.getElementById(`val-${cat}`);
    if (valEl) valEl.textContent = val.toFixed(2) + ' kg';
    const pctEl = document.getElementById(`pct-${cat}`);
    if (pctEl) pctEl.textContent = pct.toFixed(1) + '%';
  });
}

