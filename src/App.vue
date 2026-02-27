<template>
  <div id="app">
    <div class="dashboard">
      <!-- Top Bar -->
      <header class="top-bar">
        <div class="search-area">
          <div class="search-container">
            <svg class="search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="11" cy="11" r="8"/><path d="m21 21-4.35-4.35"/>
            </svg>
            <input
              id="city-search"
              type="text"
              class="search-bar"
              placeholder="Buscar ciudad..."
              v-model="query"
              @keypress.enter="searchCity"
              @input="onQueryInput"
            />
            <button id="btn-locate" class="btn-locate" @click="fetchWeatherByLocation" title="Mi ubicación">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <circle cx="12" cy="12" r="3"/><path d="M12 2v4M12 18v4M2 12h4M18 12h4"/>
              </svg>
            </button>
          </div>
          <!-- Autocomplete -->
          <ul class="suggestions" v-if="suggestions.length > 0">
            <li v-for="(s, i) in suggestions" :key="i" @click="selectSuggestion(s)">
              <span class="suggestion-name">{{ s.name }}</span>
              <span class="suggestion-detail">{{ s.admin1 ? s.admin1 + ', ' : '' }}{{ s.country }}</span>
            </li>
          </ul>
        </div>
        <div class="location-date" v-if="hasWeather">
          <span class="location-text">📍 {{ cityName }}</span>
          <span class="date-text">{{ dateBuilder() }}</span>
        </div>
      </header>

      <!-- Loading -->
      <div class="state-message" v-if="loading">
        <div class="spinner"></div>
        <p>Obteniendo datos del clima...</p>
      </div>

      <!-- Error -->
      <div class="state-message" v-else-if="error">
        <span class="state-emoji">⚠️</span>
        <p>{{ error }}</p>
        <button class="btn-retry" @click="fetchWeatherByLocation">Reintentar</button>
      </div>

      <!-- Main Content -->
      <div class="main-content" v-else-if="hasWeather">
        <div class="left-panel">
          <!-- Temperature -->
          <div class="temp-block">
            <div class="temp-value">{{ Math.round(currentWeather.temperature_2m) }}<span class="temp-unit">°c</span></div>
            <div class="weather-desc">{{ weatherInfo.description }}</div>
          </div>
          <!-- Stats -->
          <div class="stats-row">
            <div class="stat">
              <svg class="stat-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5"><path d="M9.59 4.59A2 2 0 1 1 11 8H2m10.59 11.41A2 2 0 1 0 14 16H2m15.73-8.27A2.5 2.5 0 1 1 19.5 12H2"/></svg>
              <div class="stat-info">
                <span class="stat-label">Wind</span>
                <span class="stat-value">{{ Math.round(currentWeather.wind_speed_10m) }}km/h</span>
              </div>
            </div>
            <div class="stat">
              <svg class="stat-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5"><path d="M12 2.69l5.66 5.66a8 8 0 1 1-11.31 0z"/></svg>
              <div class="stat-info">
                <span class="stat-label">Humidity</span>
                <span class="stat-value">{{ currentWeather.relative_humidity_2m }}%</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Center: Weather Icon -->
        <div class="center-panel">
          <div class="weather-emoji-big">{{ weatherInfo.icon }}</div>
        </div>

        <!-- Right: Weekly Forecast -->
        <div class="right-panel" v-if="dailyWeather">
          <div class="forecast-card">
            <div
              class="forecast-day"
              v-for="(day, i) in dailyForecast"
              :key="i"
              :class="{ today: i === 0 }"
            >
              <span class="forecast-icon">{{ day.icon }}</span>
              <div class="forecast-info">
                <span class="forecast-name">{{ day.name }}</span>
                <span class="forecast-condition">{{ day.condition }}</span>
              </div>
              <span class="forecast-temp">{{ day.maxTemp }}°</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Hourly Forecast Bar -->
      <div class="hourly-bar" v-if="hasWeather && hourlyForecast.length > 0">
        <div class="hourly-scroll">
          <div class="hourly-item" v-for="(h, i) in hourlyForecast" :key="i" :class="{ 'now': i === 0 }">
            <span class="hourly-temp">{{ h.temp }}°</span>
            <span class="hourly-icon">{{ h.icon }}</span>
            <span class="hourly-time">{{ h.time }}</span>
          </div>
        </div>
      </div>

      <!-- Empty -->
      <div class="state-message" v-if="!loading && !error && !hasWeather">
        <span class="state-emoji">🔍</span>
        <p>Busca una ciudad o permite el acceso a tu ubicación</p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'app',
  data() {
    return {
      query: '',
      cityName: '',
      currentWeather: null,
      dailyWeather: null,
      hourlyData: null,
      loading: false,
      error: null,
      suggestions: [],
      searchTimeout: null
    }
  },
  computed: {
    hasWeather() {
      return this.currentWeather !== null;
    },
    weatherInfo() {
      if (!this.hasWeather) return { icon: '', description: '' };
      return this.getWeatherInfo(this.currentWeather.weather_code, this.currentWeather.is_day);
    },
    dailyForecast() {
      if (!this.dailyWeather) return [];
      const dayNames = ['Domingo', 'Lunes', 'Martes', 'Miércoles', 'Jueves', 'Viernes', 'Sábado'];
      return this.dailyWeather.time.map((t, i) => {
        const d = new Date(t + 'T12:00:00');
        const info = this.getWeatherInfo(this.dailyWeather.weather_code[i], 1);
        return {
          name: i === 0 ? 'Hoy' : dayNames[d.getDay()],
          condition: info.description,
          icon: info.icon,
          maxTemp: Math.round(this.dailyWeather.temperature_2m_max[i])
        };
      });
    },
    hourlyForecast() {
      if (!this.hourlyData) return [];
      const now = new Date();
      const currentHour = now.getHours();
      const items = [];
      for (let i = currentHour; i < Math.min(currentHour + 12, this.hourlyData.time.length); i++) {
        const info = this.getWeatherInfo(this.hourlyData.weather_code[i], i > 6 && i < 20 ? 1 : 0);
        const h = new Date(this.hourlyData.time[i]);
        items.push({
          temp: Math.round(this.hourlyData.temperature_2m[i]),
          icon: info.icon,
          time: i === currentHour ? 'Ahora' : String(h.getHours()).padStart(2, '0') + ':00'
        });
      }
      return items;
    }
  },
  mounted() {
    this.fetchWeatherByLocation();
  },
  methods: {
    onQueryInput() {
      clearTimeout(this.searchTimeout);
      if (this.query.trim().length < 2) { this.suggestions = []; return; }
      this.searchTimeout = setTimeout(() => { this.fetchSuggestions(); }, 350);
    },
    fetchSuggestions() {
      const q = this.query.trim();
      if (q.length < 2) return;
      fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(q)}&count=5&language=es`)
        .then(r => r.json())
        .then(d => { this.suggestions = d.results || []; })
        .catch(() => { this.suggestions = []; });
    },
    selectSuggestion(s) {
      this.suggestions = [];
      this.query = s.name;
      const name = s.admin1 ? `${s.name}, ${s.admin1}, ${s.country}` : `${s.name}, ${s.country}`;
      this.fetchWeather(s.latitude, s.longitude, name);
    },
    searchCity() {
      if (!this.query.trim()) return;
      this.suggestions = [];
      this.loading = true;
      this.error = null;
      fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(this.query.trim())}&count=1&language=es`)
        .then(r => r.json())
        .then(d => {
          if (!d.results || d.results.length === 0) { this.loading = false; this.error = 'Ciudad no encontrada. Intenta de nuevo.'; return; }
          const r = d.results[0];
          const name = r.admin1 ? `${r.name}, ${r.admin1}, ${r.country}` : `${r.name}, ${r.country}`;
          this.fetchWeather(r.latitude, r.longitude, name);
        })
        .catch(() => { this.loading = false; this.error = 'Error de conexión.'; });
    },
    fetchWeatherByLocation() {
      if (!navigator.geolocation) { this.error = 'Tu navegador no soporta geolocalización.'; return; }
      this.loading = true; this.error = null; this.suggestions = [];
      navigator.geolocation.getCurrentPosition(
        (pos) => {
          const { latitude, longitude } = pos.coords;
          fetch(`https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${latitude}&longitude=${longitude}&localityLanguage=es`)
            .then(r => r.json())
            .then(g => {
              const city = g.city || g.locality || g.principalSubdivision || 'Tu ubicación';
              const country = g.countryName || '';
              this.fetchWeather(latitude, longitude, country ? `${city}, ${country}` : city);
            })
            .catch(() => { this.fetchWeather(latitude, longitude, 'Tu ubicación'); });
        },
        () => { this.loading = false; this.error = 'No se pudo obtener tu ubicación. Busca una ciudad.'; },
        { timeout: 10000 }
      );
    },
    fetchWeather(lat, lon, name) {
      this.loading = true; this.error = null;
      const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,weather_code,wind_speed_10m,wind_direction_10m,is_day&daily=temperature_2m_max,temperature_2m_min,weather_code&hourly=temperature_2m,weather_code&timezone=auto&forecast_days=7`;
      fetch(url)
        .then(r => { if (!r.ok) throw new Error(); return r.json(); })
        .then(d => {
          this.currentWeather = d.current;
          this.dailyWeather = d.daily;
          this.hourlyData = d.hourly;
          this.cityName = name;
          this.loading = false;
        })
        .catch(() => { this.loading = false; this.error = 'Error al obtener datos del clima.'; });
    },
    getWeatherInfo(code, isDay) {
      const m = {
        0:  { icon: isDay ? '☀️' : '🌙', description: 'Cielo despejado' },
        1:  { icon: isDay ? '🌤️' : '🌙', description: 'Mayormente despejado' },
        2:  { icon: '⛅', description: 'Parcialmente nublado' },
        3:  { icon: '☁️', description: 'Nublado' },
        45: { icon: '🌫️', description: 'Niebla' },
        48: { icon: '🌫️', description: 'Niebla helada' },
        51: { icon: '🌦️', description: 'Llovizna ligera' },
        53: { icon: '🌦️', description: 'Llovizna moderada' },
        55: { icon: '🌧️', description: 'Llovizna densa' },
        56: { icon: '🌧️', description: 'Llovizna helada' },
        57: { icon: '🌧️', description: 'Llovizna helada densa' },
        61: { icon: '🌧️', description: 'Lluvia ligera' },
        63: { icon: '🌧️', description: 'Lluvia moderada' },
        65: { icon: '🌧️', description: 'Lluvia fuerte' },
        66: { icon: '🌧️', description: 'Lluvia helada' },
        67: { icon: '🌧️', description: 'Lluvia helada fuerte' },
        71: { icon: '🌨️', description: 'Nevada ligera' },
        73: { icon: '🌨️', description: 'Nevada moderada' },
        75: { icon: '❄️', description: 'Nevada fuerte' },
        77: { icon: '❄️', description: 'Granizo fino' },
        80: { icon: '🌦️', description: 'Chubascos ligeros' },
        81: { icon: '🌧️', description: 'Chubascos moderados' },
        82: { icon: '⛈️', description: 'Chubascos violentos' },
        85: { icon: '🌨️', description: 'Chubascos de nieve' },
        86: { icon: '🌨️', description: 'Nieve fuerte' },
        95: { icon: '⛈️', description: 'Tormenta eléctrica' },
        96: { icon: '⛈️', description: 'Tormenta con granizo' },
        99: { icon: '⛈️', description: 'Tormenta con granizo fuerte' }
      };
      return m[code] || { icon: '🌡️', description: 'Desconocido' };
    },
    dateBuilder() {
      const d = new Date();
      const days = ['Domingo', 'Lunes', 'Martes', 'Miércoles', 'Jueves', 'Viernes', 'Sábado'];
      const months = ['Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo', 'Junio', 'Julio', 'Agosto', 'Septiembre', 'Octubre', 'Noviembre', 'Diciembre'];
      return `(${days[d.getDay()]}, ${d.getDate()} de ${months[d.getMonth()]})`;
    }
  }
}
</script>

<style>
/* ===== RESET ===== */
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: 'Outfit', sans-serif;
  -webkit-font-smoothing: antialiased;
  background: linear-gradient(135deg, #0D4671, #1a6e8e, #92E1E2);
  min-height: 100vh;
}

#app {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}

/* ===== DASHBOARD CONTAINER ===== */
.dashboard {
  width: 100%;
  max-width: 1100px;
  background: rgba(10, 18, 30, 0.85);
  backdrop-filter: blur(24px);
  -webkit-backdrop-filter: blur(24px);
  border-radius: 32px;
  border: 1px solid rgba(255, 255, 255, 0.06);
  padding: 32px 36px;
  box-shadow:
    0 24px 80px rgba(0, 0, 0, 0.4),
    inset 0 1px 0 rgba(255, 255, 255, 0.05);
  min-height: 560px;
  display: flex;
  flex-direction: column;
}

/* ===== TOP BAR ===== */
.top-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: 36px;
  flex-wrap: wrap;
}

.search-area {
  position: relative;
  flex: 0 1 360px;
}

.search-container {
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.07);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 14px;
  padding: 3px;
  transition: all 0.3s ease;
}

.search-container:focus-within {
  background: rgba(255, 255, 255, 0.12);
  border-color: rgba(146, 225, 226, 0.3);
}

.search-icon {
  width: 18px; height: 18px;
  color: rgba(255,255,255,0.35);
  margin-left: 12px;
  flex-shrink: 0;
}

.search-bar {
  flex: 1;
  padding: 10px;
  color: #fff;
  font-family: 'Outfit', sans-serif;
  font-size: 14px;
  border: none; outline: none; background: none;
}

.search-bar::placeholder { color: rgba(255,255,255,0.3); }

.btn-locate {
  display: flex; align-items: center; justify-content: center;
  width: 36px; height: 36px;
  border: none; border-radius: 10px;
  background: rgba(146, 225, 226, 0.15);
  color: #92E1E2;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.btn-locate:hover {
  background: rgba(146, 225, 226, 0.25);
  transform: scale(1.05);
}

.btn-locate svg { width: 16px; height: 16px; }

/* Suggestions */
.suggestions {
  position: absolute; top: 100%; left: 0; right: 0;
  z-index: 100; list-style: none;
  background: rgba(10, 18, 30, 0.97);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 12px;
  margin-top: 4px;
  overflow: hidden;
  box-shadow: 0 12px 40px rgba(0,0,0,0.5);
  animation: dropIn 0.2s ease;
}

@keyframes dropIn { from { opacity:0; transform:translateY(-4px); } to { opacity:1; transform:translateY(0); } }

.suggestions li {
  padding: 10px 16px;
  cursor: pointer;
  display: flex; justify-content: space-between; align-items: center;
  transition: background 0.15s;
  border-bottom: 1px solid rgba(255,255,255,0.04);
}

.suggestions li:last-child { border-bottom: none; }
.suggestions li:hover { background: rgba(146,225,226,0.08); }
.suggestion-name { color: #fff; font-size: 14px; font-weight: 500; }
.suggestion-detail { color: rgba(255,255,255,0.35); font-size: 12px; }

/* Location & Date */
.location-date {
  display: flex; align-items: center; gap: 16px;
  flex-wrap: wrap;
}

.location-text {
  color: rgba(255,255,255,0.85);
  font-size: 14px; font-weight: 500;
}

.date-text {
  color: rgba(255,255,255,0.4);
  font-size: 13px; font-weight: 300;
}

/* ===== STATE MESSAGES ===== */
.state-message {
  flex: 1;
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  gap: 14px; text-align: center;
}

.state-message p {
  color: rgba(255,255,255,0.6);
  font-size: 15px; font-weight: 300;
  max-width: 300px; line-height: 1.5;
}

.state-emoji { font-size: 48px; }

.spinner {
  width: 40px; height: 40px;
  border: 3px solid rgba(255,255,255,0.1);
  border-top-color: #92E1E2;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin { to { transform: rotate(360deg); } }

.btn-retry {
  padding: 8px 24px;
  border: 1px solid rgba(146,225,226,0.3);
  border-radius: 10px;
  background: rgba(146,225,226,0.1);
  color: #92E1E2;
  font-family: 'Outfit', sans-serif;
  font-size: 13px; cursor: pointer;
  transition: all 0.2s;
}

.btn-retry:hover { background: rgba(146,225,226,0.2); }

/* ===== MAIN CONTENT GRID ===== */
.main-content {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr auto 280px;
  gap: 24px;
  align-items: center;
  animation: fadeIn 0.5s ease;
}

@keyframes fadeIn { from { opacity:0; transform:translateY(12px); } to { opacity:1; transform:translateY(0); } }

/* Left Panel */
.left-panel {
  display: flex;
  flex-direction: column;
  gap: 32px;
}

.temp-block { }

.temp-value {
  color: #fff;
  font-size: 96px;
  font-weight: 200;
  line-height: 1;
  letter-spacing: -4px;
}

.temp-unit {
  font-size: 36px;
  font-weight: 300;
  vertical-align: super;
  letter-spacing: 0;
  opacity: 0.7;
}

.weather-desc {
  color: rgba(255,255,255,0.65);
  font-size: 22px;
  font-weight: 400;
  margin-top: 4px;
}

.stats-row {
  display: flex; gap: 28px;
}

.stat {
  display: flex; align-items: center; gap: 10px;
}

.stat-icon {
  width: 22px; height: 22px;
  color: rgba(146,225,226,0.6);
  flex-shrink: 0;
}

.stat-info {
  display: flex; flex-direction: column;
}

.stat-label {
  color: rgba(255,255,255,0.35);
  font-size: 12px; font-weight: 400;
}

.stat-value {
  color: #fff;
  font-size: 16px; font-weight: 600;
}

/* Center Panel */
.center-panel {
  display: flex;
  align-items: center;
  justify-content: center;
}

.weather-emoji-big {
  font-size: 120px;
  filter: drop-shadow(0 8px 24px rgba(0,0,0,0.3));
  animation: float 4s ease-in-out infinite;
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-12px); }
}

/* Right Panel: Forecast Card */
.right-panel { }

.forecast-card {
  background: rgba(255,255,255,0.04);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 20px;
  padding: 8px 0;
  overflow: hidden;
}

.forecast-day {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 18px;
  transition: background 0.15s;
}

.forecast-day:hover { background: rgba(255,255,255,0.04); }

.forecast-day.today {
  background: rgba(146,225,226,0.06);
  border-left: 2px solid #92E1E2;
}

.forecast-icon { font-size: 20px; flex-shrink: 0; }

.forecast-info {
  flex: 1;
  display: flex; flex-direction: column;
}

.forecast-name {
  color: #fff;
  font-size: 13px; font-weight: 500;
}

.forecast-condition {
  color: rgba(255,255,255,0.35);
  font-size: 11px; font-weight: 300;
}

.forecast-temp {
  color: #fff;
  font-size: 15px; font-weight: 600;
  flex-shrink: 0;
}

/* ===== HOURLY BAR ===== */
.hourly-bar {
  margin-top: 24px;
  border-top: 1px solid rgba(255,255,255,0.06);
  padding-top: 20px;
  animation: fadeIn 0.5s ease 0.2s both;
}

.hourly-scroll {
  display: flex;
  gap: 4px;
  overflow-x: auto;
  padding-bottom: 4px;
  scrollbar-width: thin;
  scrollbar-color: rgba(146,225,226,0.2) transparent;
}

.hourly-scroll::-webkit-scrollbar { height: 4px; }
.hourly-scroll::-webkit-scrollbar-track { background: transparent; }
.hourly-scroll::-webkit-scrollbar-thumb { background: rgba(146,225,226,0.2); border-radius: 4px; }

.hourly-item {
  flex: 0 0 auto;
  display: flex; flex-direction: column;
  align-items: center; gap: 6px;
  padding: 10px 16px;
  border-radius: 12px;
  transition: background 0.15s;
  min-width: 72px;
}

.hourly-item:hover { background: rgba(255,255,255,0.04); }

.hourly-item.now {
  background: rgba(146,225,226,0.08);
  border: 1px solid rgba(146,225,226,0.15);
}

.hourly-temp {
  color: #fff;
  font-size: 15px; font-weight: 600;
}

.hourly-icon { font-size: 18px; }

.hourly-time {
  color: rgba(255,255,255,0.35);
  font-size: 11px; font-weight: 400;
}

/* ===== RESPONSIVE ===== */
@media (max-width: 900px) {
  .main-content {
    grid-template-columns: 1fr;
    text-align: center;
  }

  .left-panel { align-items: center; }
  .stats-row { justify-content: center; }

  .center-panel { order: -1; }

  .weather-emoji-big { font-size: 80px; }

  .temp-value { font-size: 72px; }
  .weather-desc { font-size: 18px; }

  .right-panel { width: 100%; }

  .forecast-card {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }

  .forecast-day.today { border-left: none; border-top: 2px solid #92E1E2; }
}

@media (max-width: 600px) {
  #app { padding: 12px; }

  .dashboard {
    border-radius: 24px;
    padding: 20px 18px;
    min-height: auto;
  }

  .top-bar { flex-direction: column; align-items: stretch; gap: 12px; }
  .search-area { flex: 1; }
  .location-date { justify-content: center; }

  .temp-value { font-size: 56px; }

  .forecast-card { grid-template-columns: 1fr; }
}
</style>
