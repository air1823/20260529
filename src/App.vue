<script setup>
import { ref, computed, onMounted } from 'vue';

// 請在 .env 檔中設定 VITE_GAS_URL=你的 Apps Script Web App URL
const GAS_URL = import.meta.env.VITE_GAS_URL || 'https://script.google.com/macros/s/AKfycbxKhGPtLfU-VJVkOBED5AiqI4DmNfd5ob9ps6xWblSkm_-NbXdRlTKL5nPrxNzLZFKg/exec';

const events = ref([]);
const newEvent = ref({
  summary: '',
  start: '',
  end: '',
  category: '',
  description: ''
});

const statusMsg = ref('');
const currentViewDate = ref(new Date(2026, 7, 1)); // 預設顯示 2026年8月 (ICS資料主要月份)
const categoryOptions = ref([
  { name: '重要截止', color: '#ff4d4f' },
  { name: '學期課程', color: '#1890ff' },
  { name: '行政會議', color: '#fa8c16' },
  { name: '假日', color: '#52c41a' },
  { name: '教學活動', color: '#722ed1' },
  { name: '學生活動', color: '#eb2f96' },
  { name: '招生相關', color: '#13c2c2' }
]);
const newCategory = ref({ name: '', color: '#4a90e2' });

const getCategoryColor = (cat) => {
  const found = categoryOptions.value.find(item => item.name === cat);
  return found ? found.color : '#8c8c8c';
};

// 月曆翻頁邏輯
const prevMonth = () => {
  currentViewDate.value = new Date(currentViewDate.value.getFullYear(), currentViewDate.value.getMonth() - 1, 1);
};

const nextMonth = () => {
  currentViewDate.value = new Date(currentViewDate.value.getFullYear(), currentViewDate.value.getMonth() + 1, 1);
};

// 計算當月網格資料
const calendarDays = computed(() => {
  const year = currentViewDate.value.getFullYear();
  const month = currentViewDate.value.getMonth();
  const firstDay = new Date(year, month, 1).getDay();
  const daysInMonth = new Date(year, month + 1, 0).getDate();
  
  const days = [];
  // 填充上個月空白
  for (let i = 0; i < firstDay; i++) {
    days.push(null);
  }
  
  // 填充當月日期
  for (let d = 1; d <= daysInMonth; d++) {
    const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(d).padStart(2, '0')}`;
    // 篩選出當天的事件
    const dayEvents = events.value.filter(e => {
      return e.start === dateStr;
    });
    days.push({ day: d, dateStr, dayEvents });
  }
  return days;
});

// 格式化 ICS 日期 (YYYYMMDD) 為 HTML input 使用的格式 (YYYY-MM-DD)
const formatIcsToInput = (icsDate) => {
  if (!icsDate) return '';
  const dateOnly = icsDate.split(':')[1] || icsDate;
  return `${dateOnly.substring(0, 4)}-${dateOnly.substring(4, 6)}-${dateOnly.substring(6, 8)}`;
};

// 格式化 Input 日期為 ICS 格式 (YYYYMMDD)
const formatInputToIcs = (dateStr) => {
  return dateStr.replace(/-/g, '');
};

// 解析 ICS 檔案
const handleImport = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    const content = e.target.result;
    const lines = content.split(/\r?\n/);
    const parsedEvents = [];
    let current = null;

    lines.forEach(line => {
      if (line.startsWith('BEGIN:VEVENT')) {
        current = {};
      } else if (line.startsWith('END:VEVENT')) {
        parsedEvents.push(current);
        current = null;
      } else if (current) {
        if (line.includes('SUMMARY:')) current.summary = line.split('SUMMARY:')[1];
        if (line.includes('CATEGORIES:')) current.category = line.split('CATEGORIES:')[1];
        if (line.includes('DESCRIPTION:')) current.description = line.split('DESCRIPTION:')[1];
        if (line.includes('DTSTART')) current.start = formatIcsToInput(line.split('DTSTART')[1]);
        if (line.includes('DTEND')) current.end = formatIcsToInput(line.split('DTEND')[1]);
      }
    });
    
    events.value = parsedEvents;
    statusMsg.value = `成功讀取 ${parsedEvents.length} 筆事項，點擊「儲存至雲端」上傳。`;
  };
  reader.readAsText(file);
};

// 新增單一事項
const addEvent = () => {
  if (!newEvent.value.summary || !newEvent.value.start) {
    alert('請填寫名稱與日期');
    return;
  }
  events.value.push({ ...newEvent.value });
  newEvent.value = { summary: '', start: '', end: '', category: '', description: '' };
};

const addCategoryItem = () => {
  const name = newCategory.value.name.trim();
  if (!name) {
    alert('請輸入分類名稱');
    return;
  }

  const exists = categoryOptions.value.some(item => item.name === name);
  if (exists) {
    alert('這個分類已存在');
    return;
  }

  categoryOptions.value.push({
    name,
    color: newCategory.value.color
  });

  newCategory.value = { name: '', color: '#4a90e2' };
  statusMsg.value = `已新增分類「${name}」`;
};

// 從 Google Sheets 獲取資料 (同步)
const fetchFromSheets = async () => {
  if (!GAS_URL || GAS_URL.includes('YOUR_GOOGLE_APPS_SCRIPT_URL_HERE')) {
    statusMsg.value = '請先設定 VITE_GAS_URL';
    return;
  }

  statusMsg.value = '從雲端同步中...';
  try {
    const response = await fetch(GAS_URL, { method: 'GET' });
    if (!response.ok) throw new Error('讀取失敗');

    const data = await response.json();
    events.value = (data || []).map(row => ({
      summary: row[0] || '',
      start: row[1] || '',
      end: row[2] || '',
      category: row[3] || '',
      description: row[4] || ''
    }));
    statusMsg.value = '同步完成';
  } catch (err) {
    statusMsg.value = '同步失敗：' + (err.message || '未知錯誤');
  }
};

// 匯出為 ICS 檔案
const exportIcs = () => {
  let ics = "BEGIN:VCALENDAR\nVERSION:2.0\nPRODID:-//My Calendar//ZH\n";
  events.value.forEach(ev => {
    ics += "BEGIN:VEVENT\n";
    ics += `SUMMARY:${ev.summary}\n`;
    ics += `CATEGORIES:${ev.category}\n`;
    ics += `DESCRIPTION:${ev.description || ''}\n`;
    ics += `DTSTART;VALUE=DATE:${formatInputToIcs(ev.start)}\n`;
    ics += `DTEND;VALUE=DATE:${formatInputToIcs(ev.end || ev.start)}\n`;
    ics += "END:VEVENT\n";
  });
  ics += "END:VCALENDAR";

  const blob = new Blob([ics], { type: 'text/calendar' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = 'calendar-export.ics';
  link.click();
};

// 儲存至 Google Sheets
const saveToSheets = async () => {
  if (!GAS_URL || GAS_URL.includes('YOUR_GOOGLE_APPS_SCRIPT_URL_HERE')) {
    statusMsg.value = '請先設定 VITE_GAS_URL';
    return;
  }

  if (events.value.length === 0) {
    statusMsg.value = '目前沒有可同步的資料';
    return;
  }

  statusMsg.value = '儲存中...';

  try {
    const response = await fetch(GAS_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(events.value)
    });

    if (!response.ok) {
      const text = await response.text();
      throw new Error(text || '儲存失敗');
    }

    statusMsg.value = '資料已同步至 Google Sheets';
  } catch (err) {
    statusMsg.value = '儲存失敗：' + (err.message || '未知錯誤');
  }
};

onMounted(() => {
  fetchFromSheets(); // 初始化時讀取
});
</script>

<template>
  <div class="calendar-app">
    <h1 class="title">📅 雲端行事曆管理</h1>

    <div class="main-layout">
      <aside class="control-panel">
        <div class="card">
          <h3>新增/分類事項</h3>
          <div class="form-group">
            <input v-model="newEvent.summary" placeholder="事項名稱" />
            <label class="field-label">分類</label>
            <select v-model="newEvent.category">
              <option value="">請選擇分類</option>
              <option v-for="item in categoryOptions" :key="item.name" :value="item.name">
                {{ item.name }}
              </option>
            </select>
            <div class="date-row">
              <label>開始：<input type="date" v-model="newEvent.start" /></label>
              <label>結束：<input type="date" v-model="newEvent.end" /></label>
            </div>
            <textarea v-model="newEvent.description" placeholder="備註描述"></textarea>
            <button @click="addEvent">加入清單</button>
          </div>
        </div>

        <div class="actions">
          <label class="btn-file">
            匯入 ICS 檔案
            <input type="file" @change="handleImport" accept=".ics" hidden />
          </label>
          <button @click="saveToSheets" class="btn-save">儲存至 Google Sheets</button>
          <button @click="exportIcs" class="btn-export">匯出 ICS</button>
        </div>

        <p v-if="statusMsg" class="status">{{ statusMsg }}</p>
      </aside>

      <section class="calendar-view">
        <div class="card category-legend">
          <h3>分類顏色說明</h3>
          <div class="legend-grid">
            <div v-for="item in categoryOptions" :key="item.name" class="legend-item">
              <span class="legend-swatch" :style="{ backgroundColor: item.color }"></span>
              <span>{{ item.name }}</span>
            </div>
          </div>
        </div>

        <div class="card add-category-card">
          <h3>新增分類</h3>
          <div class="form-group compact-form">
            <input v-model="newCategory.name" placeholder="分類名稱" />
            <label class="field-label">代表顏色</label>
            <input v-model="newCategory.color" type="color" />
            <button @click="addCategoryItem">新增分類</button>
          </div>
        </div>

        <div class="calendar-container card">
          <div class="calendar-header">
            <button @click="prevMonth" class="btn-nav">◀</button>
            <h2>{{ currentViewDate.getFullYear() }} 年 {{ currentViewDate.getMonth() + 1 }} 月</h2>
            <button @click="nextMonth" class="btn-nav">▶</button>
          </div>
          <div class="calendar-grid">
            <div v-for="d in ['日','一','二','三','四','五','六']" :key="d" class="weekday">{{ d }}</div>
            <div v-for="(dayObj, idx) in calendarDays" :key="idx" class="day-cell" :class="{ empty: !dayObj }">
              <template v-if="dayObj">
                <div class="day-number">{{ dayObj.day }}</div>
                <div class="event-tags">
                  <div v-for="(ev, eIdx) in dayObj.dayEvents" :key="eIdx"
                       class="event-tag" :style="{ backgroundColor: getCategoryColor(ev.category) }"
                       :title="ev.summary + (ev.description ? ': ' + ev.description : '')">
                    {{ ev.summary }}
                  </div>
                </div>
              </template>
            </div>
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
.calendar-app {
  max-width: 1400px;
  width: 100%;
  margin: 0 auto;
  font-family: 'PingFang TC', 'Microsoft JhengHei', sans-serif;
  padding: 20px 16px 40px;
  box-sizing: border-box;
  color: #333;
}

.title {
  text-align: center;
  color: #2c3e50;
  margin-bottom: 24px;
}

.main-layout {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.control-panel {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.card {
  background: white;
  padding: 18px;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.form-group input,
.form-group textarea,
.form-group select {
  width: 100%;
  box-sizing: border-box;
  padding: 10px;
  border: 1px solid #d5dbe3;
  border-radius: 6px;
  font: inherit;
}

.date-row {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.date-row label {
  flex: 1 1 180px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 0.95rem;
  color: #4b5563;
}

.date-row input {
  width: 100%;
  box-sizing: border-box;
  padding: 10px;
  border: 1px solid #d5dbe3;
  border-radius: 6px;
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.calendar-view {
  min-width: 0;
}

.calendar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  margin-bottom: 12px;
}

.calendar-header h2 {
  margin: 0;
  font-size: 1.05rem;
  text-align: center;
}

.btn-nav {
  background: #eee;
  color: #333;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex: 0 0 40px;
}

.calendar-grid {
  display: grid;
  grid-template-columns: repeat(7, minmax(0, 1fr));
  border: 1px solid #eee;
  border-radius: 8px;
  overflow: hidden;
}

.weekday {
  background: #f8f9fa;
  padding: 10px;
  text-align: center;
  font-weight: bold;
  border-bottom: 1px solid #eee;
}

.day-cell {
  min-height: 110px;
  border: 0.5px solid #f0f0f0;
  padding: 6px;
  background: #fff;
}

.day-cell.empty {
  background: #fafafa;
}

.day-number {
  font-size: 0.9em;
  color: #666;
  font-weight: bold;
  margin-bottom: 4px;
}

.event-tags {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.event-tag {
  font-size: 11px;
  color: white;
  padding: 2px 5px;
  border-radius: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  cursor: help;
}

.legend-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.95rem;
  color: #334155;
}

.legend-swatch {
  width: 14px;
  height: 14px;
  border-radius: 999px;
  border: 1px solid rgba(0, 0, 0, 0.08);
  flex: 0 0 14px;
}

.field-label {
  font-size: 0.95rem;
  color: #4b5563;
}

.compact-form {
  gap: 8px;
}

button,
.btn-file {
  padding: 10px 15px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background: #4a90e2;
  color: white;
  font-weight: bold;
}

.btn-save {
  background: #47af50;
}

.btn-export {
  background: #7f8c8d;
}

.status {
  color: #d32f2f;
  font-weight: bold;
  text-align: center;
  margin: 0;
}

@media (min-width: 1100px) {
  .main-layout {
    flex-direction: row;
    align-items: flex-start;
  }

  .control-panel {
    flex: 0 0 380px;
    position: sticky;
    top: 16px;
  }

  .calendar-view {
    flex: 1 1 auto;
  }

  .actions {
    flex-direction: column;
    align-items: stretch;
  }

  .legend-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 640px) {
  .calendar-app {
    padding: 16px 12px 28px;
  }

  .calendar-header {
    align-items: flex-start;
  }

  .day-cell {
    min-height: 92px;
  }

  .event-tag {
    font-size: 10px;
  }
}
</style>
