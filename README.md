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
    {
  "name": "EcoTrack - Carbon Footprint",
  "short_name": "EcoTrack",
  "description": "Track and reduce your carbon footprint with personalized insights",
  "start_url": "/index.html",
  "display": "standalone",
  "background_color": "#0a0f1e",
  "theme_color": "#22c55e",
  "orientation": "portrait",
  "icons": [
    {
      "src": "icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "categories": ["health", "lifestyle", "utilities"],
  "screenshots": []
}

/* ============================================================
   ECOTRACK — Full Design System
   Dark Glassmorphism + Neon Green Accent
   ============================================================ */

@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;500;600;700;800&display=swap');

/* ─── CSS VARIABLES ──────────────────────────────────────── */
:root {
  --bg-base:      #070d1a;
  --bg-surface:   #0d1526;
  --bg-card:      rgba(255,255,255,0.04);
  --bg-card-hover:rgba(255,255,255,0.07);
  --border:       rgba(255,255,255,0.08);
  --border-active:rgba(34,197,94,0.4);

  --green:        #22c55e;
  --green-dim:    #16a34a;
  --green-glow:   rgba(34,197,94,0.25);
  --blue:         #3b82f6;
  --orange:       #f97316;
  --yellow:       #eab308;
  --purple:       #a855f7;
  --teal:         #14b8a6;
  --red:          #ef4444;

  --text-primary:   #f1f5f9;
  --text-secondary: #94a3b8;
  --text-muted:     #475569;

  --nav-height:   72px;
  --radius-sm:    10px;
  --radius-md:    16px;
  --radius-lg:    22px;
  --radius-xl:    30px;

  --shadow-card:  0 4px 24px rgba(0,0,0,0.4);
  --shadow-glow:  0 0 24px rgba(34,197,94,0.2);
  --transition:   all 0.25s cubic-bezier(0.4,0,0.2,1);
}

/* ─── RESET & BASE ───────────────────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html, body {
  width: 100%; height: 100%;
  font-family: 'Outfit', sans-serif;
  background: var(--bg-base);
  color: var(--text-primary);
  overflow-x: hidden;
  -webkit-tap-highlight-color: transparent;
  -webkit-font-smoothing: antialiased;
}

body {
  background: radial-gradient(ellipse 80% 50% at 50% -10%, rgba(34,197,94,0.12), transparent),
              radial-gradient(ellipse 60% 40% at 80% 110%, rgba(59,130,246,0.08), transparent),
              var(--bg-base);
  min-height: 100dvh;
}

/* ─── SCROLLBAR ──────────────────────────────────────────── */
::-webkit-scrollbar { width: 4px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: var(--border); border-radius: 2px; }

/* ─── APP SHELL ──────────────────────────────────────────── */
#app {
  max-width: 480px;
  margin: 0 auto;
  min-height: 100dvh;
  display: flex;
  flex-direction: column;
  position: relative;
}

/* ─── HEADER ─────────────────────────────────────────────── */
.app-header {
  position: sticky;
  top: 0;
  z-index: 100;
  padding: 16px 20px 12px;
  background: rgba(7,13,26,0.85);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-bottom: 1px solid var(--border);
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.header-logo {
  display: flex;
  align-items: center;
  gap: 10px;
}

.logo-icon {
  width: 36px; height: 36px;
  background: linear-gradient(135deg, #22c55e, #16a34a);
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  box-shadow: 0 0 16px rgba(34,197,94,0.4);
  animation: pulse-glow 3s ease-in-out infinite;
}

.logo-text {
  font-size: 20px;
  font-weight: 800;
  background: linear-gradient(135deg, #22c55e, #84cc16);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  letter-spacing: -0.5px;
}

.header-date {
  font-size: 12px;
  color: var(--text-muted);
  font-weight: 500;
}

#install-btn {
  display: none;
  align-items: center;
  gap: 6px;
  background: var(--green-glow);
  border: 1px solid var(--border-active);
  color: var(--green);
  font-family: 'Outfit', sans-serif;
  font-size: 13px;
  font-weight: 600;
  padding: 7px 14px;
  border-radius: 20px;
  cursor: pointer;
  transition: var(--transition);
}
#install-btn:hover { background: var(--green); color: #000; }

/* ─── PAGE CONTENT ───────────────────────────────────────── */
.page-content {
  flex: 1;
  overflow-y: auto;
  padding: 16px 16px calc(var(--nav-height) + 16px);
  scroll-behavior: smooth;
}

.page { display: none; }
.page.active { display: block; }

/* ─── GLASS CARD ─────────────────────────────────────────── */
.glass-card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  padding: 18px;
  transition: var(--transition);
  box-shadow: var(--shadow-card);
}
.glass-card:hover {
  background: var(--bg-card-hover);
  border-color: rgba(255,255,255,0.12);
  transform: translateY(-1px);
}

/* ─── SECTION HEADER ─────────────────────────────────────── */
.section-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}
.section-title {
  font-size: 15px;
  font-weight: 700;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.8px;
}
.section-badge {
  font-size: 11px;
  padding: 4px 10px;
  border-radius: 20px;
  background: var(--bg-card);
  border: 1px solid var(--border);
  color: var(--text-muted);
}

/* ─── DASHBOARD PAGE ─────────────────────────────────────── */
.dash-hero {
  background: linear-gradient(135deg, rgba(34,197,94,0.1), rgba(59,130,246,0.08));
  border: 1px solid var(--border-active);
  border-radius: var(--radius-xl);
  padding: 24px;
  margin-bottom: 16px;
  text-align: center;
  position: relative;
  overflow: hidden;
}
.dash-hero::before {
  content: '';
  position: absolute;
  inset: 0;
  background: radial-gradient(circle at 50% 0%, rgba(34,197,94,0.15), transparent 60%);
  pointer-events: none;
}

#dash-total {
  font-size: 13px;
  color: var(--text-secondary);
  margin-bottom: 4px;
}
.total-num {
  font-size: 48px;
  font-weight: 800;
  display: block;
  line-height: 1;
  margin-bottom: 4px;
  background: linear-gradient(135deg, var(--green), #84cc16);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}
.total-unit {
  font-size: 14px;
  color: var(--text-secondary);
  font-weight: 500;
}

.dash-meta {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  margin-top: 14px;
  flex-wrap: wrap;
}

#dash-rating {
  font-size: 12px;
  font-weight: 700;
  padding: 6px 14px;
  border-radius: 20px;
  letter-spacing: 0.3px;
}

#dash-streak {
  font-size: 12px;
  color: var(--orange);
  background: rgba(249,115,22,0.1);
  border: 1px solid rgba(249,115,22,0.3);
  padding: 6px 12px;
  border-radius: 20px;
  font-weight: 600;
}

/* Doughnut chart */
.doughnut-wrap {
  position: relative;
  width: 200px;
  height: 200px;
  margin: 20px auto;
}
#doughnut-center-text {
  position: absolute;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  pointer-events: none;
}
.center-value {
  display: block;
  font-size: 22px;
  font-weight: 800;
  color: var(--text-primary);
}
.center-unit {
  font-size: 10px;
  color: var(--text-secondary);
  font-weight: 500;
}

/* Category breakdown bars */
.cat-breakdown { margin-bottom: 16px; }
.cat-row {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}
.cat-row-icon { font-size: 16px; width: 22px; text-align: center; }
.cat-row-label { font-size: 12px; color: var(--text-secondary); width: 70px; flex-shrink: 0; font-weight: 500; }
.cat-bar-track {
  flex: 1;
  height: 6px;
  background: rgba(255,255,255,0.06);
  border-radius: 3px;
  overflow: hidden;
}
.cat-bar-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 0.8s cubic-bezier(0.4,0,0.2,1);
  width: 0%;
}
.cat-row-val { font-size: 11px; color: var(--text-muted); width: 64px; text-align: right; flex-shrink: 0; }
.cat-row-pct { font-size: 10px; color: var(--text-muted); width: 34px; text-align: right; flex-shrink: 0; }

/* Annual projection */
#annual-projection {
  text-align: center;
  font-size: 22px;
  font-weight: 800;
  margin: 8px 0 4px;
}
#annual-projection small { font-size: 12px; font-weight: 500; color: var(--text-secondary); }

/* Weekly insight */
#weekly-insight p {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.6;
}
#weekly-insight b { color: var(--text-primary); }

/* Chart containers */
.chart-container { position: relative; height: 220px; }
.chart-container-sm { position: relative; height: 180px; }

/* ─── LOG PAGE ───────────────────────────────────────────── */
.cat-tabs-scroll {
  display: flex;
  gap: 8px;
  overflow-x: auto;
  padding-bottom: 4px;
  margin-bottom: 16px;
  scrollbar-width: none;
}
.cat-tabs-scroll::-webkit-scrollbar { display: none; }

.cat-tab {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 10px 14px;
  border-radius: var(--radius-sm);
  background: var(--bg-card);
  border: 1px solid var(--border);
  color: var(--text-secondary);
  font-size: 11px;
  font-weight: 600;
  font-family: 'Outfit', sans-serif;
  cursor: pointer;
  white-space: nowrap;
  transition: var(--transition);
  flex-shrink: 0;
}
.cat-tab .tab-icon { font-size: 18px; }
.cat-tab:hover { background: var(--bg-card-hover); color: var(--text-primary); }
.cat-tab.active {
  background: linear-gradient(135deg, rgba(34,197,94,0.15), rgba(34,197,94,0.05));
  border-color: var(--border-active);
  color: var(--green);
  box-shadow: 0 0 12px rgba(34,197,94,0.15);
}

/* Log category header */
.log-category-header {
  text-align: center;
  padding: 16px 0 8px;
}
.cat-icon-big { font-size: 40px; }
.log-category-header h2 {
  font-size: 22px;
  font-weight: 700;
  margin: 6px 0 4px;
}
.cat-desc {
  font-size: 13px;
  color: var(--text-secondary);
}

/* Log items */
.log-item {
  margin-bottom: 10px;
  transition: var(--transition);
}
.log-item-info {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
}
.log-icon { font-size: 24px; flex-shrink: 0; }
.log-label {
  font-size: 15px;
  font-weight: 600;
  color: var(--text-primary);
}
.log-unit {
  font-size: 11px;
  color: var(--text-muted);
  margin-top: 2px;
}

.log-controls {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}
.qty-btn {
  width: 40px; height: 40px;
  border-radius: 12px;
  border: 1px solid var(--border);
  background: var(--bg-card);
  color: var(--text-primary);
  font-size: 20px;
  font-family: 'Outfit', sans-serif;
  cursor: pointer;
  transition: var(--transition);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.qty-btn.plus { border-color: rgba(34,197,94,0.4); color: var(--green); }
.qty-btn.minus { border-color: rgba(239,68,68,0.3); color: #f87171; }
.qty-btn:hover { transform: scale(1.08); background: var(--bg-card-hover); }
.qty-btn:active { transform: scale(0.95); }

.qty-input {
  flex: 1;
  height: 40px;
  background: rgba(255,255,255,0.05);
  border: 1px solid var(--border);
  border-radius: 10px;
  color: var(--text-primary);
  font-size: 18px;
  font-weight: 700;
  font-family: 'Outfit', sans-serif;
  text-align: center;
  outline: none;
  transition: var(--transition);
}
.qty-input:focus {
  border-color: var(--border-active);
  box-shadow: 0 0 0 3px var(--green-glow);
}
.qty-input::-webkit-outer-spin-button,
.qty-input::-webkit-inner-spin-button { -webkit-appearance: none; }

.log-item-emission {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-secondary);
  text-align: right;
  transition: color 0.3s;
}

.log-category-total {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 8px;
  font-size: 15px;
  font-weight: 600;
  border: 1px solid;
  background: rgba(34,197,94,0.04);
}

.save-log-btn {
  width: 100%;
  margin-top: 16px;
  padding: 16px;
  background: linear-gradient(135deg, var(--green), var(--green-dim));
  border: none;
  border-radius: var(--radius-md);
  color: #000;
  font-size: 16px;
  font-weight: 700;
  font-family: 'Outfit', sans-serif;
  cursor: pointer;
  transition: var(--transition);
  box-shadow: 0 4px 20px rgba(34,197,94,0.4);
  letter-spacing: 0.3px;
}
.save-log-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 32px rgba(34,197,94,0.5);
}
.save-log-btn:active { transform: scale(0.98); }

/* ─── HISTORY PAGE ───────────────────────────────────────── */
.history-row {
  display: grid;
  grid-template-columns: 90px 1fr 80px;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
  padding: 12px 16px;
}
.hdate-label { font-size: 13px; font-weight: 600; color: var(--text-primary); }
.hdate-key { font-size: 10px; color: var(--green); font-weight: 600; }
.history-bar-wrap {
  height: 6px;
  background: rgba(255,255,255,0.06);
  border-radius: 3px;
  overflow: hidden;
}
.history-bar-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 0.6s ease;
}
.history-total { font-size: 13px; font-weight: 700; text-align: right; }

#history-empty {
  text-align: center;
  padding: 60px 20px;
  color: var(--text-muted);
}
#history-empty .empty-icon { font-size: 48px; margin-bottom: 12px; }

/* ─── SUGGESTIONS PAGE ───────────────────────────────────── */
.suggestion-card {
  margin-bottom: 12px;
  animation: slideUp 0.4s ease both;
}
.suggestion-badge {
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
  color: var(--text-secondary);
}
.suggestion-card p {
  font-size: 14px;
  color: var(--text-primary);
  line-height: 1.65;
}
.suggestion-card p b { color: var(--green); }

.insight-chip {
  background: rgba(59,130,246,0.1);
  border: 1px solid rgba(59,130,246,0.25);
  border-radius: var(--radius-sm);
  padding: 12px 16px;
  font-size: 13px;
  color: var(--text-secondary);
  margin-bottom: 10px;
  line-height: 1.6;
}
.insight-chip b { color: var(--text-primary); }

/* ─── SETTINGS PAGE ─────────────────────────────────────── */
.settings-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 16px;
}
.stat-tile {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  padding: 18px 14px;
  text-align: center;
}
.stat-tile-val {
  font-size: 24px;
  font-weight: 800;
  color: var(--green);
  display: block;
  margin-bottom: 4px;
}
.stat-tile-label {
  font-size: 11px;
  color: var(--text-muted);
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.settings-action-btn {
  width: 100%;
  padding: 15px;
  border-radius: var(--radius-md);
  border: 1px solid var(--border);
  background: var(--bg-card);
  color: var(--text-primary);
  font-family: 'Outfit', sans-serif;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: var(--transition);
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
}
.settings-action-btn:hover { background: var(--bg-card-hover); transform: translateY(-1px); }
.settings-action-btn.danger {
  border-color: rgba(239,68,68,0.3);
  color: #f87171;
}
.settings-action-btn.danger:hover { background: rgba(239,68,68,0.08); }

.emission-factors-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;
}
.emission-factors-table th {
  color: var(--text-muted);
  font-weight: 600;
  padding: 8px 10px;
  text-align: left;
  border-bottom: 1px solid var(--border);
}
.emission-factors-table td {
  padding: 8px 10px;
  color: var(--text-secondary);
  border-bottom: 1px solid rgba(255,255,255,0.03);
}
.emission-factors-table tr:hover td { background: rgba(255,255,255,0.02); }

.ef-badge {
  font-size: 10px;
  padding: 2px 8px;
  border-radius: 10px;
  font-weight: 700;
}
.ef-badge.high { background: rgba(239,68,68,0.12); color: #f87171; }
.ef-badge.med  { background: rgba(249,115,22,0.12); color: #fb923c; }
.ef-badge.low  { background: rgba(34,197,94,0.12);  color: #4ade80; }
.ef-badge.neg  { background: rgba(20,184,166,0.12); color: #2dd4bf; }

/* ─── BOTTOM NAV ─────────────────────────────────────────── */
.bottom-nav {
  position: fixed;
  bottom: 0; left: 50%;
  transform: translateX(-50%);
  width: 100%;
  max-width: 480px;
  height: var(--nav-height);
  background: rgba(7,13,26,0.92);
  backdrop-filter: blur(24px);
  -webkit-backdrop-filter: blur(24px);
  border-top: 1px solid var(--border);
  display: flex;
  align-items: center;
  justify-content: space-around;
  padding: 0 8px;
  z-index: 200;
}

.nav-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  flex: 1;
  padding: 10px 4px;
  background: none;
  border: none;
  color: var(--text-muted);
  font-family: 'Outfit', sans-serif;
  font-size: 10px;
  font-weight: 600;
  cursor: pointer;
  transition: var(--transition);
  border-radius: 12px;
  letter-spacing: 0.2px;
  position: relative;
}
.nav-icon { font-size: 22px; transition: transform 0.2s; }
.nav-item:hover { color: var(--text-secondary); }
.nav-item:hover .nav-icon { transform: translateY(-2px); }
.nav-item.active { color: var(--green); }
.nav-item.active .nav-icon { filter: drop-shadow(0 0 6px rgba(34,197,94,0.6)); }
.nav-item.active::before {
  content: '';
  position: absolute;
  top: 0; left: 20%; right: 20%;
  height: 2px;
  background: var(--green);
  border-radius: 0 0 2px 2px;
  animation: nav-indicator 0.2s ease;
}

/* ─── PAGE TITLES ────────────────────────────────────────── */
.page-title {
  font-size: 24px;
  font-weight: 800;
  margin-bottom: 16px;
  letter-spacing: -0.5px;
}

/* ─── TOAST ──────────────────────────────────────────────── */
#toast {
  position: fixed;
  bottom: calc(var(--nav-height) + 16px);
  left: 50%;
  transform: translateX(-50%) translateY(20px);
  background: rgba(17,24,39,0.96);
  border: 1px solid var(--border);
  color: var(--text-primary);
  padding: 12px 20px;
  border-radius: var(--radius-lg);
  font-size: 14px;
  font-weight: 500;
  z-index: 999;
  opacity: 0;
  transition: all 0.3s cubic-bezier(0.34,1.56,0.64,1);
  white-space: nowrap;
  max-width: 90vw;
  box-shadow: var(--shadow-card);
}
#toast.show {
  opacity: 1;
  transform: translateX(-50%) translateY(0);
}

/* ─── ANIMATIONS ─────────────────────────────────────────── */
@keyframes pulse-glow {
  0%, 100% { box-shadow: 0 0 16px rgba(34,197,94,0.4); }
  50%       { box-shadow: 0 0 28px rgba(34,197,94,0.7); }
}

@keyframes slideUp {
  from { opacity: 0; transform: translateY(16px); }
  to   { opacity: 1; transform: translateY(0); }
}

@keyframes nav-indicator {
  from { opacity: 0; transform: scaleX(0); }
  to   { opacity: 1; transform: scaleX(1); }
}

@keyframes fadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}

@keyframes countUp {
  from { transform: scale(0.8); opacity: 0; }
  to   { transform: scale(1); opacity: 1; }
}

/* ─── RESPONSIVE ─────────────────────────────────────────── */
@media (min-width: 480px) {
  #app { box-shadow: 0 0 80px rgba(0,0,0,0.6); }
}

@media (prefers-color-scheme: light) {
  /* App is dark-mode only by design */
}

/* Safe area for notched phones */
@supports (padding-bottom: env(safe-area-inset-bottom)) {
  .bottom-nav {
    height: calc(var(--nav-height) + env(safe-area-inset-bottom));
    padding-bottom: env(safe-area-inset-bottom);
  }
  .page-content {
    padding-bottom: calc(var(--nav-height) + env(safe-area-inset-bottom) + 16px);
  }
}

/* ─── SETTINGS ABOUT ─────────────────────────────────────── */
.about-card {
  text-align: center;
  padding: 24px;
  margin-top: 16px;
}
.about-card .about-logo { font-size: 40px; margin-bottom: 8px; }
.about-card h3 { font-size: 18px; font-weight: 700; margin-bottom: 4px; }
.about-card p { font-size: 12px; color: var(--text-muted); line-height: 1.7; }
.about-card .version { display: inline-block; margin-top: 10px; font-size: 11px; padding: 4px 12px; border-radius: 20px; background: var(--bg-card); border: 1px solid var(--border); color: var(--text-muted); }

/* ─── LIVE TRACKER CARD ──────────────────────────────────── */
.tracker-card {
  padding: 18px;
  border-color: rgba(34,197,94,0.2);
  background: linear-gradient(135deg, rgba(34,197,94,0.05), rgba(59,130,246,0.03));
  position: relative;
  overflow: hidden;
}
.tracker-card::before {
  content: '';
  position: absolute;
  top: -30px; right: -30px;
  width: 100px; height: 100px;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(34,197,94,0.12), transparent 70%);
  pointer-events: none;
}

/* Status row */
.tracker-status-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
}
.tracker-dot {
  width: 10px; height: 10px;
  border-radius: 50%;
  background: var(--text-muted);
  flex-shrink: 0;
  transition: var(--transition);
}
.tracker-dot.active {
  background: var(--green);
  box-shadow: 0 0 0 3px rgba(34,197,94,0.25);
  animation: tracker-pulse 1.4s ease-in-out infinite;
}
@keyframes tracker-pulse {
  0%,100% { box-shadow: 0 0 0 3px rgba(34,197,94,0.25); }
  50%      { box-shadow: 0 0 0 7px rgba(34,197,94,0.05); }
}
.tracker-status-text {
  flex: 1;
  font-size: 12px;
  color: var(--text-muted);
  font-weight: 500;
  transition: color 0.3s;
}
.tracker-accuracy {
  font-size: 11px;
  font-weight: 600;
  padding: 3px 8px;
  border-radius: 10px;
  background: rgba(255,255,255,0.05);
  color: var(--text-muted);
}

/* Live readings row */
.tracker-live-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 10px;
  margin-bottom: 14px;
  padding: 14px;
  background: rgba(0,0,0,0.2);
  border-radius: var(--radius-sm);
  border: 1px solid var(--border);
}
.tracker-live-cell {
  text-align: center;
}
.tlc-icon {
  font-size: 26px;
  margin-bottom: 3px;
  transition: all 0.4s ease;
}
.tlc-value {
  font-size: 16px;
  font-weight: 800;
  color: var(--text-primary);
  font-variant-numeric: tabular-nums;
  transition: color 0.3s;
}
.tlc-label {
  font-size: 13px;
  font-weight: 700;
  transition: color 0.3s;
}
.tlc-sub {
  font-size: 10px;
  color: var(--text-muted);
  margin-top: 2px;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.4px;
}

/* Session summary */
.tracker-session {
  margin-bottom: 12px;
  min-height: 28px;
}
.session-stats {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 6px;
}
.sess-chip {
  font-size: 11px;
  padding: 4px 10px;
  border-radius: 12px;
  background: rgba(255,255,255,0.05);
  border: 1px solid var(--border);
  color: var(--text-secondary);
  font-weight: 600;
}
.session-modes {
  font-size: 12px;
  color: var(--text-muted);
  line-height: 1.6;
}

/* Tracker toggle button */
.tracker-toggle-btn {
  width: 100%;
  padding: 13px;
  border-radius: var(--radius-md);
  border: 1px solid var(--border-active);
  background: rgba(34,197,94,0.1);
  color: var(--green);
  font-family: 'Outfit', sans-serif;
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
  transition: var(--transition);
  letter-spacing: 0.3px;
  margin-bottom: 10px;
}
.tracker-toggle-btn:hover {
  background: rgba(34,197,94,0.2);
  transform: translateY(-1px);
  box-shadow: 0 4px 16px rgba(34,197,94,0.2);
}
.tracker-toggle-btn.tracking {
  background: rgba(239,68,68,0.1);
  border-color: rgba(239,68,68,0.4);
  color: #f87171;
}
.tracker-toggle-btn.tracking:hover {
  background: rgba(239,68,68,0.2);
  box-shadow: 0 4px 16px rgba(239,68,68,0.2);
}
.tracker-toggle-btn:active { transform: scale(0.98); }

.tracker-note {
  font-size: 11px;
  color: var(--text-muted);
  text-align: center;
  line-height: 1.5;
}

/* Speed level indicator bar */
.speed-level-bar {
  height: 4px;
  border-radius: 2px;
  background: var(--border);
  margin: 8px 0;
  overflow: hidden;
}
.speed-level-fill {
  height: 100%;
  border-radius: 2px;
  transition: width 0.6s ease, background 0.4s;
}

/* ─── LOGIN OVERLAY ──────────────────────────────────────── */
#login-overlay {
  position: fixed;
  inset: 0;
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg-base);
  padding: 20px;
  overflow-y: auto;
}

.login-bg-glow {
  position: fixed;
  inset: 0;
  background:
    radial-gradient(ellipse 70% 50% at 30% 20%, rgba(34,197,94,0.15), transparent),
    radial-gradient(ellipse 60% 50% at 70% 80%, rgba(59,130,246,0.1), transparent);
  pointer-events: none;
}

.login-card {
  width: 100%;
  max-width: 380px;
  animation: slideUp 0.5s cubic-bezier(0.34,1.56,0.64,1);
  position: relative;
  z-index: 1;
}

/* Logo */
.login-logo {
  text-align: center;
  margin-bottom: 32px;
}
.login-logo-icon {
  font-size: 52px;
  margin-bottom: 8px;
  filter: drop-shadow(0 0 20px rgba(34,197,94,0.5));
  animation: pulse-glow 3s ease-in-out infinite;
  display: inline-block;
}
.login-app-name {
  font-size: 32px;
  font-weight: 800;
  background: linear-gradient(135deg, #22c55e, #84cc16);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  margin: 0 0 4px;
  letter-spacing: -1px;
}
.login-tagline {
  font-size: 13px;
  color: var(--text-muted);
  font-weight: 500;
  letter-spacing: 2px;
  text-transform: uppercase;
}

/* Google button */
.google-signin-btn {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 14px;
  background: #fff;
  border: none;
  border-radius: var(--radius-md);
  color: #333;
  font-family: 'Outfit', sans-serif;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: var(--transition);
  box-shadow: 0 2px 12px rgba(0,0,0,0.3);
  margin-bottom: 16px;
}
.google-signin-btn:hover {
  background: #f8f8f8;
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.35);
}
.google-signin-btn:active { transform: scale(0.98); }

.g-spinner {
  width: 16px; height: 16px;
  border: 2px solid #ccc;
  border-top-color: #4285F4;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  display: inline-block;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* Divider */
.login-divider {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
  color: var(--text-muted);
  font-size: 12px;
}
.login-divider::before, .login-divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: var(--border);
}

/* Form */
.login-form { display: flex; flex-direction: column; gap: 12px; }
.login-field { display: flex; flex-direction: column; gap: 6px; }
.login-label { font-size: 12px; font-weight: 600; color: var(--text-secondary); }
.login-input {
  padding: 13px 16px;
  background: rgba(255,255,255,0.06);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  color: var(--text-primary);
  font-family: 'Outfit', sans-serif;
  font-size: 15px;
  outline: none;
  transition: var(--transition);
}
.login-input::placeholder { color: var(--text-muted); }
.login-input:focus {
  border-color: var(--border-active);
  background: rgba(34,197,94,0.05);
  box-shadow: 0 0 0 3px var(--green-glow);
}

/* Submit button */
.login-submit-btn {
  width: 100%;
  padding: 15px;
  background: linear-gradient(135deg, var(--green), var(--green-dim));
  border: none;
  border-radius: var(--radius-md);
  color: #000;
  font-family: 'Outfit', sans-serif;
  font-size: 16px;
  font-weight: 800;
  cursor: pointer;
  transition: var(--transition);
  box-shadow: 0 4px 20px rgba(34,197,94,0.4);
  margin-top: 4px;
  letter-spacing: 0.3px;
}
.login-submit-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 28px rgba(34,197,94,0.5);
}
.login-submit-btn:active { transform: scale(0.98); }

/* Privacy note */
.login-privacy {
  text-align: center;
  font-size: 11px;
  color: var(--text-muted);
  margin-top: 14px;
  line-height: 1.5;
}

/* Android install hint */
.login-install-hint {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  margin-top: 16px;
  padding: 10px 16px;
  background: rgba(255,255,255,0.04);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  font-size: 12px;
  color: var(--text-muted);
}

/* ─── HEADER AVATAR ──────────────────────────────────────── */
.header-avatar {
  width: 34px; height: 34px;
  border-radius: 50%;
  display: none;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  font-weight: 800;
  color: #000;
  cursor: pointer;
  transition: var(--transition);
  border: 2px solid rgba(255,255,255,0.2);
  flex-shrink: 0;
}
.header-avatar:hover {
  transform: scale(1.08);
  border-color: var(--green);
  box-shadow: 0 0 12px rgba(34,197,94,0.4);
}

/* ─── SETTINGS PROFILE CARD ──────────────────────────────── */
.profile-avatar-lg {
  width: 60px; height: 60px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 22px;
  font-weight: 800;
  color: #000;
  flex-shrink: 0;
  box-shadow: 0 4px 16px rgba(0,0,0,0.3);
}
.profile-info { flex: 1; }
.profile-name { font-size: 17px; font-weight: 700; margin-bottom: 3px; }
.profile-email { font-size: 12px; color: var(--text-secondary); margin-bottom: 4px; }
.profile-since { font-size: 11px; color: var(--text-muted); }

/* ─── ANDROID INSTALL PAGE ───────────────────────────────── */
.install-tag {
  font-size: 11px;
  padding: 5px 12px;
  border-radius: 20px;
  background: rgba(34,197,94,0.1);
  border: 1px solid rgba(34,197,94,0.25);
  color: var(--green);
  font-weight: 600;
}

.install-steps { display: flex; flex-direction: column; gap: 10px; margin-bottom: 8px; }

.install-step {
  display: flex;
  align-items: flex-start;
  gap: 14px;
  padding: 16px;
}
.step-num {
  width: 32px; height: 32px;
  border-radius: 50%;
  background: rgba(255,255,255,0.08);
  border: 2px solid var(--border);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  font-weight: 800;
  color: var(--text-secondary);
  flex-shrink: 0;
}
.step-body { flex: 1; }
.step-title { font-size: 14px; font-weight: 700; margin-bottom: 4px; }
.step-desc { font-size: 12px; color: var(--text-secondary); line-height: 1.6; }
.step-desc b { color: var(--text-primary); }

.share-platform {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 14px 16px;
  cursor: pointer;
  text-align: left;
}
.share-platform:hover { transform: translateY(-1px); border-color: var(--border-active); }



// ============================================================
// SUGGESTIONS ENGINE — Personalized carbon reduction tips
// ============================================================

const SUGGESTIONS_DB = {
  transportation: {
    car: [
      { threshold: 50, tip: "🚗 You drove <b>{val} km</b> today. Try carpooling to cut your car emissions by 50%.", priority: 3 },
      { threshold: 20, tip: "🚲 Short trips under 5 km? Switch to cycling — saves ~2 kg CO₂ per trip.", priority: 2 },
      { threshold: 0,  tip: "✅ Great job minimizing car use! Keep it up and consider an EV for your next vehicle.", priority: 1 }
    ],
    flights: [
      { threshold: 500, tip: "✈️ Flights are your #1 emission source. Consider a train for trips under 600 km — it's 6× less carbon.", priority: 5 },
      { threshold: 200, tip: "✈️ For medium-distance travel, direct flights emit 20% less than connecting flights.", priority: 4 },
      { threshold: 0,   tip: "✈️ When you must fly, offset your carbon via certified programs like Gold Standard.", priority: 2 }
    ],
    bike: [
      { threshold: 0, tip: "🏍️ Motorbikes emit less than cars but more than cycles. Can you switch short trips to a bicycle?", priority: 1 }
    ],
    bus: [
      { threshold: 0, tip: "🚌 Buses are 3× greener than solo car trips. You're on the right track!", priority: 1 }
    ],
    train: [
      { threshold: 0, tip: "🚆 Trains are among the greenest ways to travel. Great choice!", priority: 1 }
    ]
  },
  food: {
    beef: [
      { threshold: 1, tip: "🥩 Beef is your largest food emission. Swapping just 1 beef meal/week for chicken saves ~50 kg CO₂/month.", priority: 5 },
      { threshold: 0, tip: "🌱 Even reducing beef to once a week puts you below average food emissions. Try plant-based Mondays!", priority: 3 }
    ],
    chicken: [
      { threshold: 3, tip: "🍗 Chicken has 4× less carbon than beef. Try swapping some meals with vegetarian options to go even lower.", priority: 2 },
      { threshold: 0, tip: "🍗 Good choice over beef! Replacing 2 chicken meals/week with vegetarian saves ~18 kg CO₂/month.", priority: 1 }
    ],
    vegetarian: [
      { threshold: 0, tip: "🥗 Vegetarian meals are a great choice! You're saving ~2 kg CO₂ vs a beef meal every time.", priority: 1 }
    ],
    vegan: [
      { threshold: 0, tip: "🌿 Vegan meals have the lowest footprint of all. You're a climate champion!", priority: 1 }
    ]
  },
  energy: {
    electricity: [
      { threshold: 20, tip: "⚡ Your electricity use is high. Switch to LED bulbs — they use 75% less energy.", priority: 4 },
      { threshold: 10, tip: "⚡ Unplug devices on standby — they account for 10% of home electricity bills.", priority: 3 },
      { threshold: 0,  tip: "⚡ Consider rooftop solar panels to offset your electricity footprint entirely.", priority: 1 }
    ],
    ac: [
      { threshold: 8, tip: "❄️ AC/Heating is a major emitter. Set your thermostat 2°C closer to outdoor temp to save 10% energy.", priority: 4 },
      { threshold: 4, tip: "❄️ Use ceiling fans with AC — allows setting thermostat 4°C higher with same comfort.", priority: 3 },
      { threshold: 0, tip: "❄️ Regular AC servicing improves efficiency by up to 15%. Schedule a check-up!", priority: 1 }
    ],
    gas: [
      { threshold: 4, tip: "🔥 Cooking gas burns clean but still emits CO₂. Use a pressure cooker to cut cooking time by 70%.", priority: 2 },
      { threshold: 0, tip: "🔥 Consider an induction cooktop powered by renewable electricity as a long-term upgrade.", priority: 1 }
    ]
  },
  shopping: {
    electronics: [
      { threshold: 1, tip: "📱 Electronics carry ~70 kg CO₂ each. Before buying new, check refurbished markets — same quality, 80% less carbon.", priority: 5 },
      { threshold: 0, tip: "📱 Extend your device lifespan by 1 year — it cuts its lifetime footprint by 30%.", priority: 2 }
    ],
    clothing: [
      { threshold: 3, tip: "👕 Fast fashion is a major emitter. Try thrift stores or clothes swaps — same style, near-zero carbon.", priority: 4 },
      { threshold: 1, tip: "👕 Choose quality over quantity. One durable item beats 5 cheap ones in carbon terms.", priority: 2 },
      { threshold: 0, tip: "👕 Washing clothes in cold water reduces laundry energy use by 90%.", priority: 1 }
    ],
    household: [
      { threshold: 2, tip: "🏠 Before buying household goods, check if rental, secondhand, or repair is an option.", priority: 3 },
      { threshold: 0, tip: "🏠 When buying new, look for energy-star certified products — they're greener by design.", priority: 1 }
    ]
  },
  waste: {
    trash: [
      { threshold: 2, tip: "🗑️ High trash output? Try the 5R rule: Refuse → Reduce → Reuse → Recycle → Rot (compost).", priority: 4 },
      { threshold: 1, tip: "🗑️ Use reusable bags, bottles, and containers to eliminate single-use plastic waste.", priority: 3 },
      { threshold: 0, tip: "🗑️ Great waste management! Food waste accounts for 30% of trash — try meal planning.", priority: 1 }
    ],
    recycling: [
      { threshold: 0, tip: "♻️ You're recycling! Recycling aluminium saves 95% of the energy needed to produce new aluminium.", priority: 1 }
    ],
    composting: [
      { threshold: 0, tip: "🌱 Composting diverts food from landfill where it would produce methane — 25× more potent than CO₂!", priority: 1 }
    ]
  }
};

// General tips shown when emissions are very low
const GENERAL_TIPS = [
  "🌍 You're below the global average of 4.7 tons CO₂/year. Keep it up!",
  "☀️ Advocate for renewable energy in your community — collective action matters most.",
  "🚶 Walking just 2 km/day instead of driving saves ~150 kg CO₂ per year.",
  "🌳 Plant a tree! A single tree absorbs ~21 kg of CO₂ per year.",
  "💧 Reducing hot water use by 2 minutes/shower saves ~50 kg CO₂/year.",
  "🥡 Choosing local, seasonal food cuts your food transport emissions by up to 50%.",
  "📦 Consolidate online deliveries — batch orders to reduce delivery trips.",
  "💡 Share your EcoTrack insights with friends — collective behavior change is powerful!"
];

function generateSuggestions(data) {
  const suggestions = [];
  const today = data.today || {};

  // Analyse each category
  const transport = today.transportation || {};
  const food = today.food || {};
  const energy = today.energy || {};
  const shopping = today.shopping || {};
  const waste = today.waste || {};

  // Transportation suggestions
  checkSuggestions(suggestions, SUGGESTIONS_DB.transportation.car, transport.car || 0, 'car');
  checkSuggestions(suggestions, SUGGESTIONS_DB.transportation.flights, transport.flights || 0, 'flights');
  if ((transport.bike || 0) > 0) checkSuggestions(suggestions, SUGGESTIONS_DB.transportation.bike, transport.bike, 'bike');

  // Food suggestions
  checkSuggestions(suggestions, SUGGESTIONS_DB.food.beef, food.beef || 0, 'beef');
  checkSuggestions(suggestions, SUGGESTIONS_DB.food.chicken, food.chicken || 0, 'chicken');

  // Energy suggestions
  checkSuggestions(suggestions, SUGGESTIONS_DB.energy.electricity, energy.electricity || 0, 'electricity');
  checkSuggestions(suggestions, SUGGESTIONS_DB.energy.ac, energy.ac || 0, 'ac');

  // Shopping suggestions
  checkSuggestions(suggestions, SUGGESTIONS_DB.shopping.electronics, shopping.electronics || 0, 'electronics');
  checkSuggestions(suggestions, SUGGESTIONS_DB.shopping.clothing, shopping.clothing || 0, 'clothing');

  // Waste suggestions
  checkSuggestions(suggestions, SUGGESTIONS_DB.waste.trash, waste.trash || 0, 'trash');

  // Positive reinforcement for good habits
  if ((waste.recycling || 0) > 0) suggestions.push({ tip: SUGGESTIONS_DB.waste.recycling[0].tip, priority: 1, type: 'positive' });
  if ((waste.composting || 0) > 0) suggestions.push({ tip: SUGGESTIONS_DB.waste.composting[0].tip, priority: 1, type: 'positive' });
  if ((food.vegan || 0) > 0) suggestions.push({ tip: SUGGESTIONS_DB.food.vegan[0].tip, priority: 1, type: 'positive' });
  if ((food.vegetarian || 0) > 0) suggestions.push({ tip: SUGGESTIONS_DB.food.vegetarian[0].tip, priority: 1, type: 'positive' });
  if ((transport.bus || 0) > 0) suggestions.push({ tip: SUGGESTIONS_DB.transportation.bus[0].tip, priority: 1, type: 'positive' });
  if ((transport.train || 0) > 0) suggestions.push({ tip: SUGGESTIONS_DB.transportation.train[0].tip, priority: 1, type: 'positive' });

  // Sort by priority descending
  suggestions.sort((a, b) => b.priority - a.priority);

  // Add general tips if fewer than 3 suggestions
  if (suggestions.length < 3) {
    const shuffled = [...GENERAL_TIPS].sort(() => Math.random() - 0.5);
    shuffled.slice(0, 3 - suggestions.length).forEach(tip => {
      suggestions.push({ tip, priority: 0, type: 'general' });
    });
  }

  return suggestions.slice(0, 8); // Return top 8
}

function checkSuggestions(list, db, value, key) {
  for (const entry of db) {
    if (value >= entry.threshold) {
      list.push({
        tip: entry.tip.replace('{val}', value),
        priority: entry.priority,
        type: entry.priority >= 4 ? 'warning' : entry.priority >= 2 ? 'info' : 'positive'
      });
      break;
    }
  }
}

function getWeeklyInsight(weeklyData) {
  const insights = [];
  const totals = { transportation: 0, food: 0, energy: 0, shopping: 0, waste: 0 };

  weeklyData.forEach(day => {
    Object.keys(totals).forEach(cat => {
      totals[cat] += (day[cat] || {}).total || 0;
    });
  });

  const sorted = Object.entries(totals).sort((a, b) => b[1] - a[1]);
  const top = sorted[0];

  if (top && top[1] > 0) {
    const catNames = {
      transportation: '🚗 Transportation',
      food: '🍽️ Food',
      energy: '⚡ Home Energy',
      shopping: '🛍️ Shopping',
      waste: '🗑️ Waste'
    };
    insights.push(`Your biggest emission source this week is <b>${catNames[top[0]]}</b> at ${top[1].toFixed(1)} kg CO₂e. Focus here for maximum impact.`);
  }

  const total = Object.values(totals).reduce((a, b) => a + b, 0);
  const daily = total / 7;
  const annualProjection = daily * 365;

  if (annualProjection > 8000) {
    insights.push(`⚠️ Your current pace projects to <b>${(annualProjection/1000).toFixed(1)} tons CO₂/year</b> — above the global 4.7-ton average.`);
  } else if (annualProjection > 4700) {
    insights.push(`📊 You're projected at <b>${(annualProjection/1000).toFixed(1)} tons CO₂/year</b> — close to global average. A few changes can bring you below!`);
  } else {
    insights.push(`🌟 Excellent! You're on track for just <b>${(annualProjection/1000).toFixed(1)} tons CO₂/year</b> — well below the global average!`);
  }

  return insights;
}

const CACHE_NAME = 'ecotrack-v1';
const ASSETS = [
  '/index.html',
  '/styles.css',
  '/app.js',
  '/charts.js',
  '/suggestions.js',
  '/manifest.json'
];

self.addEventListener('install', e => {
  e.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(ASSETS))
  );
  self.skipWaiting();
});

self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', e => {
  e.respondWith(
    caches.match(e.request).then(cached => cached || fetch(e.request))
  );
});

// ============================================================
// TRACKER.JS — Automatic GPS Sensor + Transport Detection
// Real-time carbon tracking engine for EcoTrack
// ============================================================

const Tracker = (() => {

  // ─── STATE ────────────────────────────────────────────────
  let state = {
    isTracking:    false,
    watchId:       null,
    lastPos:       null,
    lastTime:      null,
    currentSpeed:  0,        // km/h
    currentMode:   'stationary',
    sessionKm:     { car: 0, bike: 0, bus: 0, train: 0, flights: 0, walk: 0 },
    sessionStart:  null,
    stepCount:     0,
    motionEnabled: false,
    autoSaveTimer: null,
    dashRefreshTimer: null,
    accuracy:      null,
    latitude:      null,
    longitude:     null,
    permissionState: 'unknown', // 'unknown' | 'granted' | 'denied' | 'unavailable'
  };

  // ─── SPEED → TRANSPORT MODE CLASSIFIER ───────────────────
  // Based on typical speed ranges (km/h)
  const SPEED_MODES = [
    { max: 1.5,  mode: 'stationary', icon: '🧍', label: 'Stationary',  color: '#94a3b8' },
    { max: 8,    mode: 'walk',       icon: '🚶', label: 'Walking',      color: '#22c55e' },
    { max: 25,   mode: 'bike',       icon: '🚲', label: 'Cycling/Bike', color: '#84cc16' },
    { max: 70,   mode: 'car',        icon: '🚗', label: 'Car / Bus',    color: '#3b82f6' },
    { max: 160,  mode: 'train',      icon: '🚆', label: 'Train',        color: '#8b5cf6' },
    { max: 99999,mode: 'flights',    icon: '✈️', label: 'Flight',       color: '#f97316' },
  ];

  function classifyMode(speedKmh) {
    for (const s of SPEED_MODES) {
      if (speedKmh <= s.max) return s;
    }
    return SPEED_MODES[0];
  }

  // ─── HAVERSINE DISTANCE (km) ──────────────────────────────
  function haversine(lat1, lon1, lat2, lon2) {
    const R = 6371;
    const dLat = ((lat2 - lat1) * Math.PI) / 180;
    const dLon = ((lon2 - lon1) * Math.PI) / 180;
    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos((lat1 * Math.PI) / 180) *
        Math.cos((lat2 * Math.PI) / 180) *
        Math.sin(dLon / 2) * Math.sin(dLon / 2);
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
  }

  // ─── GPS POSITION HANDLER ─────────────────────────────────
  function onPosition(pos) {
    const { latitude, longitude, accuracy } = pos.coords;
    const now = Date.now();

    state.latitude  = latitude;
    state.longitude = longitude;
    state.accuracy  = Math.round(accuracy);
    state.permissionState = 'granted';

    // Use browser-provided speed if available (m/s → km/h)
    let speedKmh = 0;
    if (pos.coords.speed !== null && pos.coords.speed >= 0) {
      speedKmh = pos.coords.speed * 3.6;
    } else if (state.lastPos && state.lastTime) {
      const distKm  = haversine(state.lastPos.lat, state.lastPos.lon, latitude, longitude);
      const timeSec = (now - state.lastTime) / 1000;
      if (timeSec > 0) speedKmh = (distKm / timeSec) * 3600;
    }

    // Smooth speed (low-pass filter to remove GPS noise)
    state.currentSpeed = state.currentSpeed * 0.4 + speedKmh * 0.6;

    const modeInfo = classifyMode(state.currentSpeed);
    state.currentMode = modeInfo.mode;

    // Accumulate distance (ignore stationary + walking)
    if (state.lastPos && state.lastTime) {
      const distKm  = haversine(state.lastPos.lat, state.lastPos.lon, latitude, longitude);
      const timeSec = (now - state.lastTime) / 1000;

      // Only count if GPS is reasonably accurate (<80m) and movement is real
      if (accuracy < 80 && distKm > 0.001 && timeSec < 120) {
        if (modeInfo.mode !== 'stationary') {
          state.sessionKm[modeInfo.mode] = (state.sessionKm[modeInfo.mode] || 0) + distKm;
        }
      }
    }

    state.lastPos  = { lat: latitude, lon: longitude };
    state.lastTime = now;

    // Update dashboard live card
    updateTrackerUI(modeInfo);
  }

  function onError(err) {
    const msgs = {
      1: 'Location permission denied. Enable GPS in browser settings.',
      2: 'GPS signal unavailable. Move to an open area.',
      3: 'GPS timed out. Try again.',
    };
    state.permissionState = err.code === 1 ? 'denied' : 'unavailable';
    setTrackerStatus(msgs[err.code] || 'GPS error', 'error');
    stopTracking();
  }

  // ─── START / STOP ─────────────────────────────────────────
  function startTracking() {
    if (!('geolocation' in navigator)) {
      setTrackerStatus('GPS not supported on this device.', 'error');
      state.permissionState = 'unavailable';
      return;
    }

    state.isTracking   = true;
    state.sessionStart = Date.now();
    if (!state.sessionKm) {
      state.sessionKm = { car: 0, bike: 0, bus: 0, train: 0, flights: 0, walk: 0 };
    }

    setTrackerStatus('Acquiring GPS signal…', 'acquiring');

    state.watchId = navigator.geolocation.watchPosition(
      onPosition,
      onError,
      {
        enableHighAccuracy: true,
        timeout: 15000,
        maximumAge: 5000
      }
    );

    // Auto-save every 60 seconds
    state.autoSaveTimer = setInterval(() => {
      autoSaveToLog();
    }, 60000);

    // Refresh dashboard every 5 seconds while tracking
    state.dashRefreshTimer = setInterval(() => {
      if (typeof renderDashboard === 'function' && appState.currentPage === 'dashboard') {
        renderDashboard();
      }
    }, 5000);

    updateTrackBtn(true);
    updateSessionSummary();
  }

  function stopTracking() {
    if (state.watchId !== null) {
      navigator.geolocation.clearWatch(state.watchId);
      state.watchId = null;
    }
    if (state.autoSaveTimer) { clearInterval(state.autoSaveTimer); state.autoSaveTimer = null; }
    if (state.dashRefreshTimer) { clearInterval(state.dashRefreshTimer); state.dashRefreshTimer = null; }

    state.isTracking  = false;
    state.currentSpeed = 0;
    state.currentMode  = 'stationary';

    autoSaveToLog(); // Final save on stop
    updateTrackBtn(false);
    setTrackerStatus('Tracking stopped. Data saved.', 'idle');
    updateSessionSummary();

    if (typeof renderDashboard === 'function') renderDashboard();
    if (typeof showToast === 'function') showToast('📍 Trip ended. Carbon data auto-saved!');
  }

  function toggleTracking() {
    if (state.isTracking) stopTracking();
    else startTracking();
  }

  // ─── AUTO-SAVE SESSION KM TO APP LOG ─────────────────────
  function autoSaveToLog() {
    if (typeof getTodayData !== 'function') return;
    const today = getTodayData();

    // Merge tracked distances (round to 2 decimal)
    const tracked = state.sessionKm;
    ['car', 'bike', 'bus', 'train', 'flights'].forEach(mode => {
      const km = parseFloat((tracked[mode] || 0).toFixed(2));
      if (km > 0) {
        today.transportation[mode] = parseFloat(((today.transportation[mode] || 0) + km).toFixed(2));
      }
    });

    // Reset session accumulators after saving
    state.sessionKm = { car: 0, bike: 0, bus: 0, train: 0, flights: 0, walk: 0 };

    // Recalculate totals
    if (typeof calcCategoryTotal === 'function') {
      today.transportation.total = calcCategoryTotal('transportation', today.transportation);
    }

    if (typeof saveState === 'function') saveState();
    if (typeof renderDashboard === 'function') renderDashboard();
    updateSessionSummary();
  }

  // ─── UI UPDATERS ──────────────────────────────────────────
  function updateTrackerUI(modeInfo) {
    // Speed display
    const speedEl = document.getElementById('tracker-speed');
    if (speedEl) {
      speedEl.textContent = state.currentSpeed.toFixed(1) + ' km/h';
      speedEl.style.color = modeInfo.color;
    }

    // Mode icon + label
    const modeIconEl = document.getElementById('tracker-mode-icon');
    const modeLabelEl = document.getElementById('tracker-mode-label');
    if (modeIconEl) modeIconEl.textContent = modeInfo.icon;
    if (modeLabelEl) {
      modeLabelEl.textContent = modeInfo.label;
      modeLabelEl.style.color = modeInfo.color;
    }

    // Accuracy
    const accEl = document.getElementById('tracker-accuracy');
    if (accEl) {
      accEl.textContent = state.accuracy ? `±${state.accuracy}m` : '—';
      accEl.style.color = state.accuracy < 20 ? '#22c55e' : state.accuracy < 50 ? '#eab308' : '#f97316';
    }

    // Status dot
    const dot = document.getElementById('tracker-dot');
    if (dot) {
      dot.className = 'tracker-dot ' + (state.isTracking ? 'active' : '');
    }

    setTrackerStatus('Live tracking active', 'active');
    updateSessionSummary();
  }

  function updateSessionSummary() {
    const el = document.getElementById('tracker-session-summary');
    if (!el) return;

    const km = state.sessionKm;
    const total = ['car','bike','bus','train','flights'].reduce((s, k) => s + (km[k] || 0), 0);
    const walk  = (km.walk || 0);

    // Duration
    let dur = '';
    if (state.sessionStart) {
      const secs = Math.floor((Date.now() - state.sessionStart) / 1000);
      const m = Math.floor(secs / 60), s = secs % 60;
      dur = `${m}m ${s}s`;
    }

    const parts = [];
    if (km.car    > 0.01) parts.push(`🚗 ${km.car.toFixed(1)} km`);
    if (km.bike   > 0.01) parts.push(`🚲 ${km.bike.toFixed(1)} km`);
    if (km.bus    > 0.01) parts.push(`🚌 ${km.bus.toFixed(1)} km`);
    if (km.train  > 0.01) parts.push(`🚆 ${km.train.toFixed(1)} km`);
    if (km.flights> 0.01) parts.push(`✈️ ${km.flights.toFixed(1)} km`);
    if (walk      > 0.01) parts.push(`🚶 ${walk.toFixed(1)} km`);

    if (parts.length === 0 && state.isTracking) parts.push('Move to start accumulating distance…');

    el.innerHTML = `
      <div class="session-stats">
        ${dur ? `<span class="sess-chip">⏱ ${dur}</span>` : ''}
        ${total > 0 ? `<span class="sess-chip">📍 ${total.toFixed(2)} km tracked</span>` : ''}
        ${walk > 0 ? `<span class="sess-chip" style="color:#22c55e;">🚶 ${walk.toFixed(2)} km walked</span>` : ''}
      </div>
      <div class="session-modes">${parts.join(' &nbsp;·&nbsp; ') || ''}</div>
    `;
  }

  function setTrackerStatus(msg, type) {
    const el = document.getElementById('tracker-status-text');
    if (!el) return;
    el.textContent = msg;
    const colors = { active:'#22c55e', acquiring:'#eab308', error:'#ef4444', idle:'#94a3b8' };
    el.style.color = colors[type] || '#94a3b8';
  }

  function updateTrackBtn(tracking) {
    const btn = document.getElementById('tracker-toggle-btn');
    if (!btn) return;
    if (tracking) {
      btn.textContent = '⏹ Stop Tracking';
      btn.classList.add('tracking');
    } else {
      btn.textContent = '▶ Start Auto-Track';
      btn.classList.remove('tracking');
    }
  }

  // ─── DEVICE MOTION (Step Counter) ────────────────────────
  let lastAccelMag = 0;
  let stepCooldown = false;

  function startMotionTracking() {
    if (!('DeviceMotionEvent' in window)) return;

    // iOS 13+ requires permission
    if (typeof DeviceMotionEvent.requestPermission === 'function') {
      DeviceMotionEvent.requestPermission()
        .then(p => { if (p === 'granted') attachMotion(); })
        .catch(() => {});
    } else {
      attachMotion();
    }
  }

  function attachMotion() {
    window.addEventListener('devicemotion', e => {
      const a = e.acceleration || e.accelerationIncludingGravity;
      if (!a) return;
      const mag = Math.sqrt((a.x||0)**2 + (a.y||0)**2 + (a.z||0)**2);
      const delta = Math.abs(mag - lastAccelMag);
      lastAccelMag = mag;

      // Step detection: significant acceleration change (threshold ~2.5 m/s²)
      if (delta > 2.5 && !stepCooldown) {
        state.stepCount++;
        stepCooldown = true;
        setTimeout(() => stepCooldown = false, 300);
        updateStepDisplay();
      }
    });
    state.motionEnabled = true;
  }

  function updateStepDisplay() {
    const el = document.getElementById('tracker-steps');
    if (el) el.textContent = `${state.stepCount} steps`;
  }

  // ─── CHECK PERMISSION STATUS ──────────────────────────────
  async function checkPermission() {
    if (!navigator.permissions) return;
    try {
      const result = await navigator.permissions.query({ name: 'geolocation' });
      state.permissionState = result.state;
      result.onchange = () => { state.permissionState = result.state; };
    } catch(_) {}
  }

  // ─── PUBLIC API ───────────────────────────────────────────
  return {
    start: startTracking,
    stop: stopTracking,
    toggle: toggleTracking,
    getState: () => state,
    init: () => {
      checkPermission();
      startMotionTracking();
    }
  };
})();


    if (valEl) valEl.textContent = val.toFixed(2) + ' kg';
    const pctEl = document.getElementById(`pct-${cat}`);
    if (pctEl) pctEl.textContent = pct.toFixed(1) + '%';
  });
}

