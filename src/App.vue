<script setup>
import { ref, computed, onUnmounted, onMounted, watch } from 'vue';
import QRCode from 'qrcode';

// --- 0. 通用設定與點名系統 --- 
const activeTab = ref(localStorage.getItem('activeTab') || 'attendance');

const tabs = [
  { id: 'attendance', name: '🎅點名系統', icon: '🎅' },
  { id: 'grouping', name: '🎄 學生分組', icon: '🎄' },
  { id: 'scoring', name: '🎁 競賽計分', icon: '🎁' },
  { id: 'timer', name: '🔔 報告計時', icon: '🔔' },
  { id: 'trivia', name: '💡 隨堂小問答', icon: '💡' },
  { id: 'quiz', name: '📝 課堂測驗', icon: '📝' }
];

watch(activeTab, (newVal) => {
  localStorage.setItem('activeTab', newVal);
});

const totalStudents = ref(parseInt(localStorage.getItem('totalStudents') || '30'));
const attendanceData = ref(JSON.parse(localStorage.getItem('attendanceData') || '[]'));

// 初始化點名資料
const initAttendance = () => {
  attendanceData.value = Array.from({ length: totalStudents.value }, (_, i) => ({
    seat: i + 1,
    present: false // 預設為未出席
  }));
  saveAttendanceData(); // 初始化後儲存
};

// 儲存點名資料到 localStorage
const saveAttendanceData = () => {
  localStorage.setItem('attendanceData', JSON.stringify(attendanceData.value));
};

// 監聽 totalStudents 變化，自動更新點名表
watch(totalStudents, (newVal) => {
  localStorage.setItem('totalStudents', newVal.toString());
  initAttendance(); // 重新初始化點名表
});

// 全選出席
const markAllPresent = () => {
  attendanceData.value.forEach(student => student.present = true);
  saveAttendanceData();
};

// 手動切換出席狀態
const togglePresence = (index) => {
  attendanceData.value[index].present = !attendanceData.value[index].present;
  saveAttendanceData(); // 每次切換後儲存
};

// 點名統計計算
const attendanceStats = computed(() => {
  const presentCount = attendanceData.value.filter(s => s.present).length;
  const absentList = attendanceData.value.filter(s => !s.present).map(s => s.seat);
  return {
    total: attendanceData.value.length,
    present: presentCount,
    absent: attendanceData.value.length - presentCount,
    absentList: absentList
  };
});

// 匯出缺席名單為文字檔
const exportAbsentList = () => {
  const { total, present, absent, absentList } = attendanceStats.value;
  const now = new Date();
  const dateStr = now.toLocaleDateString();
  const timeStr = now.toLocaleTimeString();
  
  const content = [
    `教學輔助平台 - 點名紀錄`,
    `日期：${dateStr} ${timeStr}`,
    `--------------------------`,
    `應到人數：${total}`,
    `實到人數：${present}`,
    `缺席人數：${absent}`,
    `缺席座號：${absentList.length > 0 ? absentList.join(', ') : '無 (全員到齊)'}`,
    `--------------------------`
  ].join('\n');

  const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = `缺席名單_${dateStr.replace(/[\/\\]/g, '-')}.txt`;
  link.click();
  URL.revokeObjectURL(url);
};

// --- 1. 學生分組與抽籤邏輯 ---
const studentInput = ref(localStorage.getItem('studentInput') || '王小明, 李大華, 張小芬, 趙鐵柱, 孫美美, 錢多多'); // 從 localStorage 讀取
const groupSize = ref(parseInt(localStorage.getItem('groupSize') || '2')); // 從 localStorage 讀取
const groups = ref(JSON.parse(localStorage.getItem('groups') || '[]'));
// 監聽 studentInput 和 groupSize 變化，儲存到 localStorage
watch(studentInput, (newVal) => localStorage.setItem('studentInput', newVal));
watch(groupSize, (newVal) => localStorage.setItem('groupSize', newVal.toString()));

const generateGroups = () => {
  const names = studentInput.value.split(/[,，\s]+/).map(n => n.trim()).filter(n => n);
  if (names.length === 0) {
    alert('請輸入學生姓名！');
    return;
  }
  if (groupSize.value <= 0) {
    alert('每組人數必須大於 0！');
    return;
  }
  if (groupSize.value > names.length) {
    alert('每組人數不能多於總學生人數！');
    return;
  }

  const shuffled = [...names].sort(() => Math.random() - 0.5);
  const result = [];
  for (let i = 0; i < shuffled.length; i += groupSize.value) {
    result.push(shuffled.slice(i, i + groupSize.value));
  }
  groups.value = result;
  localStorage.setItem('groups', JSON.stringify(groups.value)); // 儲存分組結果
  // 自動同步更新計分板的小組數量
  scores.value = result.map((_, i) => ({ name: `第 ${i + 1} 組`, value: 0 }));
};

// --- 2. 小組競賽計分 ---
const scores = ref([]);
const adjustScore = (index, amount) => {
  if (scores.value[index] !== undefined) { // 確保索引存在
    scores.value[index].value += amount;
    localStorage.setItem('scores', JSON.stringify(scores.value)); // 儲存分數
  }
};

// 重置分數
const resetScores = () => {
  if (!confirm('確定要將所有小組分數歸零嗎？')) return; // 增加確認提示
  scores.value.forEach(score => score.value = 0);
  localStorage.setItem('scores', JSON.stringify(scores.value)); // 儲存分數
};


// --- 3. 報告計時管理 ---
const timerMinutes = ref(parseInt(localStorage.getItem('timerMinutes') || '5')); // UI 使用分鐘，從 localStorage 讀取
const timeLeft = ref(timerMinutes.value * 60); // 內部以秒計算
const isRunning = ref(false);
let timerInterval = null;
const timerProgress = computed(() => {
  const totalSeconds = timerMinutes.value * 60;
  return totalSeconds > 0 ? (timeLeft.value / totalSeconds) * 100 : 0; // 避免除以零
});

// 格式化時間顯示
const formattedTime = computed(() => {
  const mins = Math.floor(timeLeft.value / 60);
  const secs = timeLeft.value % 60;
  return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
});

// 監聽分鐘變化
watch(timerMinutes, (newVal) => { // 當使用者改變分鐘設定時
  localStorage.setItem('timerMinutes', newVal.toString()); // 儲存分鐘設定
  if (!isRunning.value) { // 如果計時器沒有在運行，則更新剩餘時間
    timeLeft.value = newVal * 60; // 將分鐘轉換為秒
  }
});

const toggleTimer = () => {
  if (isRunning.value) {
    clearInterval(timerInterval);
  } else {
    timerInterval = setInterval(() => {
      if (timeLeft.value > 0) {
        timeLeft.value--;
      } else {
        clearInterval(timerInterval);
        isRunning.value = false;
        alert('時間到！');
      }
    }, 1000);
  }
  isRunning.value = !isRunning.value;
};

const resetTimer = () => {
  clearInterval(timerInterval);
  isRunning.value = false;
  timeLeft.value = timerMinutes.value * 60; // 重置為設定的秒數
};

// --- 4. 隨堂問答邏輯 ---
const triviaQuestions = [
  { q: "在 Vue 3 中，哪一個函數用來建立響應式變數？", o: ["ref()", "reactive()", "以上皆是", "以上皆非"], a: "以上皆是" },
  { q: "JavaScript 中 '1' + 1 的結果是什麼？", o: ["2", "'11'", "undefined", "NaN"], a: "'11'" },
  { q: "CSS 中的 'rem' 單位是相對於哪一個元素的字體大小？", o: ["父元素", "根元素 (<html>)", "視窗大小", "當前元素"], a: "根元素 (<html>)" },
  { q: "下列哪一個不是常見的 CSS 排版模型？", o: ["Flexbox", "Grid", "Float", "SQL"], a: "SQL" },
  { q: "在 Git 中，哪一個指令用來將遠端儲存庫下載到本地？", o: ["git push", "git clone", "git commit", "git add"], a: "git clone" }
];
const triviaStudent = ref('');
const triviaQuestion = ref(null);
const selectedOption = ref('');
const showTriviaAnswer = ref(false);

const drawTrivia = () => {
  const names = studentInput.value.split(/[,，\s]+/).map(n => n.trim()).filter(n => n);
  if (names.length === 0) {
    alert('請先在分組區輸入學生姓名！');
    return;
  }
  
  // 隨機選學生
  const studentIdx = Math.floor(Math.random() * names.length);
  triviaStudent.value = names[studentIdx];
  
  // 隨機選題目
  const questionIdx = Math.floor(Math.random() * triviaQuestions.length);
  triviaQuestion.value = triviaQuestions[questionIdx];
  
  selectedOption.value = '';
  // 重置答案顯示狀態
  showTriviaAnswer.value = false;
};

// --- 5. 課堂測驗 QR Code 邏輯 ---
const quizLink = ref('https://docs.google.com/forms/d/e/1FAIpQLSfQm1qlzQ0xpjLLbeDZQYrP__CH8cQiu-JwN5-W63thTJIoGA/viewform?usp=publish-editor');
const quizQrImage = ref('');

const generateQuizQRCode = async () => {
  if (!quizLink.value) return;
  try {
    quizQrImage.value = await QRCode.toDataURL(quizLink.value, {
      width: 400,
      margin: 2,
      color: {
        dark: '#f1c40f', // 金色 QR Code
        light: '#ffffff' // 白色背景
      }
    });
  } catch (err) {
    console.error(err);
  }
};

// --- 生命週期鉤子 ---
onMounted(() => {
  // 初始載入時，如果 localStorage 沒有點名資料，或人數不符，則初始化
  if (attendanceData.value.length === 0 || attendanceData.value.length !== totalStudents.value) {
    initAttendance();
  }

  // 載入計分板資料 (如果 localStorage 有，則載入；否則根據分組初始化)
  const savedScores = localStorage.getItem('scores');
  if (savedScores) {
    scores.value = JSON.parse(savedScores);
  } else if (groups.value.length > 0) { // 如果有分組但沒有分數，則根據分組初始化分數
    scores.value = groups.value.map((_, i) => ({ name: `第 ${i + 1} 組`, value: 0 }));
  }

  // 計時器初始時間已在 ref 定義時從 localStorage 載入
  if (!isRunning.value) { // 確保計時器未運行時，timeLeft 與 timerMinutes 同步
    timeLeft.value = timerMinutes.value * 60; 
  }

  // 初始生成預設連結的 QR Code
  generateQuizQRCode();
});

onUnmounted(() => clearInterval(timerInterval));
</script>

<template>
  <div class="teaching-tool-container">
    <header class="app-header">
      <div class="nav-menu">
        <select v-model="activeTab" class="nav-select">
          <option v-for="tab in tabs" :key="tab.id" :value="tab.id">
            {{ tab.name }}
          </option>
        </select>
      </div>
      <h1 class="main-title">🎄 教學輔助聖誕專區</h1>
      <div class="header-spacer"></div>
    </header>

    <!-- 點名系統區 --> 
    <section v-if="activeTab === 'attendance'" class="card attendance-section main-view">
      <div class="card-header">
        <h2>📝 點名系統</h2>
        <div class="setup-row">
          <label>班級人數：</label>
          <input type="number" v-model.number="totalStudents" min="1" max="100" class="input-small">
          <button @click="initAttendance" class="btn-outline">重置</button>
          <button @click="markAllPresent" class="btn-primary">全選出席</button>
        </div>
      </div>

      <div class="attendance-content">
        <div class="seat-grid">
          <div 
            v-for="(student, idx) in attendanceData" 
            :key="idx" 
            class="seat-item"
            :class="{ 'is-present': student.present }"
            @click="togglePresence(idx)"
          >
            {{ student.seat }}
          </div>
        </div>
      </div>

      <!-- 統計資訊 -->
      <div class="attendance-summary" v-if="attendanceData.length > 0">
        <div class="stat-badge info">應到：<strong>{{ attendanceStats.total }}</strong></div>
        <div class="stat-badge success">實到：<strong>{{ attendanceStats.present }}</strong></div>
        <div class="stat-badge danger">未到：<strong>{{ attendanceStats.absent }}</strong></div>
        <button @click="exportAbsentList" class="btn-outline" style="margin-left: auto; font-size: 0.8rem; padding: 5px 10px;">💾 匯出名單</button>
        <div class="absent-numbers" v-if="attendanceStats.absentList.length > 0">
          缺席座號：<span>{{ attendanceStats.absentList.join(', ') }}</span>
        </div>
      </div>
    </section>

    <div class="feature-content">
      <!-- 分組工具區 -->
      <section v-if="activeTab === 'grouping'" class="card main-view">
        <div class="card-header">
          <h2>👥 學生分組</h2>
          <button @click="generateGroups" class="btn-primary">開始分組</button>
        </div>
        <textarea v-model="studentInput" placeholder="輸入名單 (例: 小明, 小華...)"></textarea> 
        <div class="controls row">
          <label>每組人數：</label>
          <input type="number" v-model.number="groupSize" min="1" class="input-small">
        </div>
        
        <div class="group-results">
          <TransitionGroup name="list">
            <div v-for="(group, idx) in groups" :key="idx" class="group-item accent-border">
              <span class="badge">第 {{ idx + 1 }} 組</span> {{ group.join(', ') }}
            </div>
          </TransitionGroup>
        </div>
      </section>

      <!-- 計分工具區 -->
      <section v-if="activeTab === 'scoring'" class="card main-view">
        <div class="card-header">
          <h2>🏆 競賽計分</h2>
          <button @click="resetScores" class="btn-outline" v-if="scores.length > 0">重置</button>
        </div>
        <div v-if="scores.length === 0" class="empty-msg">請先完成分組以產生計分板</div>
        <div v-for="(score, idx) in scores" :key="idx" class="score-item">
          <span class="group-name">{{ score.name }}</span>
          <div class="score-controls">
            <button @click="adjustScore(idx, -1)" class="btn-circle">-</button>
            <span class="score-val">{{ score.value }}</span>
            <button @click="adjustScore(idx, 1)" class="btn-circle">+</button>
          </div>
        </div>
      </section>

      <!-- 計時工具區 -->
      <section v-if="activeTab === 'timer'" class="card timer-card main-view">
        <h2>⏱️ 報告計時</h2>
        <div class="timer-display" :class="{ 'warning': timeLeft < 60 }">
          {{ formattedTime }}
        </div>
        <div class="controls">
          <label>設定時間 (分鐘)：</label>
          <input type="number" v-model.number="timerMinutes" min="1" style="width: 60px;" class="input-number">
          <button @click="toggleTimer">{{ isRunning ? '暫停' : '開始' }}</button>
          <button @click="resetTimer">重置</button>
        </div>
      </section>

      <!-- 隨堂問答區 -->
      <section v-if="activeTab === 'trivia'" class="card main-view">
        <div class="card-header">
          <h2>💡 隨堂問答</h2>
          <button @click="drawTrivia" class="btn-primary">🎲 抽出幸運兒</button>
        </div>
        
        <div v-if="triviaStudent" class="trivia-container">
          <div class="trivia-target">
            🎯 請回答同學：<span class="highlight">{{ triviaStudent }}</span>
          </div>
          
          <div v-if="triviaQuestion" class="trivia-box">
            <h3>{{ triviaQuestion.q }}</h3>
            <div class="options-grid">
              <div 
                v-for="(opt, idx) in triviaQuestion.o" 
                :key="idx" 
                class="option-item"
                :class="{ 
                  'is-selected': selectedOption === opt,
                  'is-correct': showTriviaAnswer && opt === triviaQuestion.a,
                  'is-wrong': showTriviaAnswer && selectedOption === opt && opt !== triviaQuestion.a
                }"
                @click="selectedOption = opt"
              >
                {{ opt }}
              </div>
            </div>
            <div class="answer-section">
              <button @click="showTriviaAnswer = true" class="btn-outline" :disabled="!selectedOption">揭曉答案</button>
              <div v-if="showTriviaAnswer" class="trivia-feedback">
                <span v-if="selectedOption === triviaQuestion.a" class="correct-text">🎉 太棒了！答對了！</span>
                <span v-else class="wrong-text">❌ 可惜了，再接再厲！</span>
              </div>
            </div>
          </div>
        </div>
        <div v-else class="empty-msg">點擊上方按鈕開始隨堂問答</div>
      </section>

      <!-- 課堂測驗區 -->
      <section v-if="activeTab === 'quiz'" class="card main-view">
        <div class="card-header">
          <h2>📝 課堂測驗</h2>
        </div>
        <div class="quiz-container">
          <p class="label-text">請貼上您的 Google 表單或測驗連結：</p>
          <input type="text" v-model="quizLink" placeholder="https://docs.google.com/forms/..." class="quiz-input">
          <button @click="generateQuizQRCode" class="btn-primary full-width" style="margin: 20px 0;">✨ 生成測驗 QR Code</button>
          
          <div v-if="quizQrImage" class="qr-display-box">
            <img :src="quizQrImage" alt="Quiz QR Code">
            <p class="qr-hint">學生掃描上方條碼即可開始測驗</p>
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 30px;
  padding: 10px 0;
}

.nav-select {
  padding: 10px 15px;
  border-radius: 12px;
  border: 2px solid #f1c40f;
  background: white;
  color: #b3000c;
  font-weight: 700;
  font-size: 1rem;
  cursor: pointer;
  outline: none;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}
.nav-select:hover { border-color: #b3000c; }

.main-title {
  margin: 0;
  flex-grow: 1;
  text-align: center;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
}

.header-spacer { width: 150px; } /* 平衡左側選單的寬度，讓標題居中 */

h2 {
  color: #ffffff;
  display: flex;
  align-items: center;
  gap: 8px;
}

.main-view {
  animation: fadeIn 0.3s ease-in-out;
  max-width: 800px;
  margin: 0 auto;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.attendance-section { margin-bottom: 20px; }
.setup-row { margin-bottom: 15px; display: flex; align-items: center; gap: 10px; }
.btn-secondary { background: #6c757d; color: white; border: none; margin-left: 10px; }

.seat-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(50px, 1fr)); gap: 8px; }
.seat-item { 
  height: 50px; display: flex; align-items: center; justify-content: center; 
  border: 1px solid #ccc; border-radius: 4px; cursor: pointer; transition: 0.2s;
}
.seat-item.is-present { background: #42b983; color: white; border-color: #42b983; }

.attendance-summary { 
  margin-top: 20px; padding-top: 15px; border-top: 2px dashed #ddd;
  display: flex; flex-wrap: wrap; gap: 20px; align-items: center;
}
.text-success { color: #42b983; }
.text-danger { color: #e74c3c; }
.absent-numbers { width: 100%; font-size: 0.9em; color: #666; }
.absent-numbers span { color: #e74c3c; font-weight: bold; }

.teaching-tool-container {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  max-width: 1000px;
  margin: 0 auto;
  padding: 40px 20px;
  color: #f8fafc;
  background: linear-gradient(-45deg, #020617, #0f172a, #1e1b4b, #020617);
  background-size: 400% 400%;
  animation: gradientBG 15s ease infinite;
  min-height: 100vh;
  position: relative;
  overflow-x: hidden;
}

/* 閃爍星光層 */
.teaching-tool-container::after {
  content: "";
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background-image: radial-gradient(#ffd700 1px, transparent 1px);
  background-size: 60px 60px;
  opacity: 0.2;
  animation: starsBlink 4s ease-in-out infinite;
  pointer-events: none;
}

h1 { 
  text-align: center; 
  color: #ffffff; 
  font-weight: 800; 
  margin-bottom: 40px; 
  letter-spacing: 0.1em;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5), 0 0 20px rgba(255, 255, 255, 0.2);
  animation: goldGlow 3s ease-in-out infinite alternate;
}

.grid-layout {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 24px;
}
.card {
  border: none;
  border: 1px solid rgba(241, 196, 15, 0.2);
  border-radius: 20px;
  padding: 24px;
  background: rgba(15, 23, 42, 0.8);
  backdrop-filter: blur(10px);
  box-shadow: 0 10px 30px -10px rgba(0, 0, 0, 0.5);
  transition: transform 0.2s ease;
  position: relative;
  z-index: 1;
}
.card:hover { transform: translateY(-2px); }
textarea { width: 100%; height: 80px; margin-bottom: 10px; }
.group-results { margin-top: 15px; font-size: 0.9em; }
.score-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  padding: 12px 20px;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px;
}
.group-name { color: #ffffff; font-weight: 600; }
.score-val { 
  font-weight: 800; 
  font-size: 1.5rem;
  color: #f1c40f; 
  min-width: 40px; 
  text-align: center; 
  text-shadow: 0 0 10px rgba(241, 196, 15, 0.3);
}
.timer-display { font-size: 3rem; text-align: center; margin: 20px 0; font-family: monospace; }
.warning { color: red; }

/* 通用按鈕樣式 */
button { 
  cursor: pointer; 
  padding: 10px 20px; 
  margin: 4px; 
  border-radius: 12px; 
  border: none; 
  background: #edf2f7; 
  color: #4a5568;
  font-weight: 600;
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1);
}
button:hover { background: #e2e8f0; transform: translateY(-1px); box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1); }
button:active { transform: translateY(0); }

.btn-primary { background: #10b981; color: white; }
.btn-primary:hover { background: #059669; color: white; }
.btn-outline { background: transparent; border: 2px solid #e2e8f0; }
.btn-outline:hover { border-color: #cbd5e1; background: #f8fafc; }
.btn-circle { width: 40px; height: 40px; padding: 0; border-radius: 50%; display: flex; align-items: center; justify-content: center; }
.input-number { padding: 6px; border: 1px solid #e2e8f0; border-radius: 8px; text-align: center; }

/* 測驗功能樣式 */
.label-text { color: #e2e8f0; margin-bottom: 12px; }
.quiz-input {
  width: 100%;
  padding: 12px 16px;
  border-radius: 12px;
  border: 1px solid rgba(241, 196, 15, 0.3);
  background: rgba(255, 255, 255, 0.1);
  color: white;
  font-size: 1rem;
}
.qr-display-box { text-align: center; margin-top: 20px; }
.qr-display-box img { border: 10px solid white; border-radius: 16px; box-shadow: 0 0 30px rgba(0,0,0,0.5); max-width: 280px; }
.qr-hint { color: #f1c40f; margin-top: 15px; font-weight: 600; font-size: 1.1rem; }

.full-width { width: 100%; }

/* 隨堂問答樣式 */
.trivia-container { text-align: center; margin-top: 20px; }
.trivia-target { font-size: 1.5rem; color: #ffffff; margin-bottom: 30px; }
.trivia-target .highlight { color: #f1c40f; font-weight: 800; text-decoration: underline; }
.trivia-box { background: rgba(255, 255, 255, 0.05); padding: 30px; border-radius: 16px; border: 1px solid rgba(241, 196, 15, 0.2); }
.trivia-box h3 { color: #f1c40f; margin-bottom: 25px; line-height: 1.4; }
.options-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 15px; margin-bottom: 30px; }
.option-item { 
  background: rgba(255, 255, 255, 0.1); 
  padding: 15px; 
  border-radius: 8px; 
  color: #e2e8f0; 
  font-size: 1.1rem; 
  cursor: pointer;
  transition: all 0.3s ease;
  border: 1px solid transparent;
}
.option-item:hover { background: rgba(255, 255, 255, 0.2); }
.option-item.is-selected { border-color: #f1c40f; background: rgba(241, 196, 15, 0.15); box-shadow: 0 0 15px rgba(241, 196, 15, 0.3); }
.option-item.is-correct { border-color: #2ecc71; background: rgba(46, 204, 113, 0.2) !important; color: white; }
.option-item.is-wrong { border-color: #e74c3c; background: rgba(231, 76, 60, 0.2) !important; }

.answer-section { margin-top: 20px; min-height: 80px; }
.trivia-feedback { margin-top: 15px; font-weight: bold; font-size: 1.2rem; }
.correct-text { color: #2ecc71; text-shadow: 0 0 10px rgba(46, 204, 113, 0.5); }
.wrong-text { color: #e74c3c; }

.correct-answer { 
  font-size: 1.3rem; 
  color: #2ecc71; 
  font-weight: bold; 
  text-shadow: 0 0 10px rgba(46, 204, 113, 0.4);
  animation: fadeIn 0.5s ease;
}
</style>
