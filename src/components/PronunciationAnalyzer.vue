<!-- src/components/PronunciationAnalyzer.vue (The Final, Absolute, Shameful Fix) -->
<script setup>
import { ref, onMounted } from 'vue';
import axios from 'axios';
import Recorder from 'recorder-core';
import 'recorder-core/src/engine/wav.js';

// --- 狀態變數 ---
const targetSentence = ref('how was your day');
const language = ref('en_us');
const analysisResult = ref(null);
const isLoading = ref(false);
const errorMessage = ref('');
const statusMessage = ref('');
const isRecording = ref(false);
let rec;

// --- 【【【 核心修改：動態 API URL 】】】 ---
const apiBaseUrl = ref(''); // 用於雙向綁定輸入框
const savedApiBaseUrl = ref(''); // 儲存並實際使用的 URL

// onMounted 會在元件掛載到頁面後執行
onMounted(() => {
  // 嘗試從瀏覽器的 localStorage 中讀取已儲存的 URL
  const storedUrl = localStorage.getItem('ngrokApiBaseUrl');
  if (storedUrl) {
    apiBaseUrl.value = storedUrl;
    savedApiBaseUrl.value = storedUrl;
    statusMessage.value = '已載入儲存的後端位址。';
  } else {
    statusMessage.value = '請輸入並儲存您的後端 Ngrok 位址。';
  }
});

// 儲存 URL 的函數
const saveApiUrl = () => {
  // 移除結尾可能存在的斜線，確保路徑拼接正確
  const cleanedUrl = apiBaseUrl.value.trim().replace(/\/$/, '');
  savedApiBaseUrl.value = cleanedUrl;
  localStorage.setItem('ngrokApiBaseUrl', cleanedUrl);
  statusMessage.value = `後端位址已更新為: ${cleanedUrl}`;
  errorMessage.value = ''; // 清除舊的錯誤訊息
};

// --- 核心功能：錄音與分析 ---
const toggleRecording = ( ) => {
  if (isRecording.value) {
    // 停止錄音
    statusMessage.value = '正在處理錄音...';
    rec.stop(
      (blob, duration) => {
        statusMessage.value = '錄音結束，正在上傳分析...';
        analyzePronunciation(blob);
      },
      (err) => {
        errorMessage.value = `錄音出錯: ${err}`;
        statusMessage.value = '';
        isRecording.value = false;
      }
    );
  } else {
    // 開始錄音
    rec = Recorder({
      type: 'wav',
      sampleRate: 16000,
      bitRate: 16,
    });
    rec.open(
      () => {
        rec.start();
        isRecording.value = true;
        analysisResult.value = null;
        errorMessage.value = '';
        statusMessage.value = '正在錄音...';
      },
      (err, isUserNotAllow) => {
        errorMessage.value = isUserNotAllow ? "您已拒絕麥克風權限。" : `無法開啟錄音: ${err}`;
      }
    );
  }
};

// 呼叫後端 API
const analyzePronunciation = async (wavBlob) => {
  isLoading.value = true;
  errorMessage.value = '';

  const formData = new FormData();
  formData.append('file', wavBlob, 'recording.wav');
  formData.append('target_sentence', targetSentence.value);
  formData.append('language', language.value);

  try {
    const response = await axios.post(`${savedApiBaseUrl.value}/api/v1/recognize`, formData, {
      headers: { 'Content-Type': 'multipart/form-data' }
    });
    analysisResult.value = response.data;
    statusMessage.value = '分析完成！';
  } catch (error) {
    if (error.response) {
      errorMessage.value = `錯誤: ${error.response.status} - ${error.response.data.detail || '伺服器發生錯誤'}`;
    } else if (error.request) {
      errorMessage.value = "請求已發送但未收到回應。請檢查網路連線或後端日誌。";
    } else {
      errorMessage.value = `請求設定出錯: ${error.message}`;
    }
    statusMessage.value = '';
  } finally {
    isLoading.value = false;
    isRecording.value = false;
  }
};

// 【【【【【 罪 魁 禍 首 在 這 裡，現 在 已 經 加 回 來 了 】】】】】
// --- 輔助函數 ---
const getPhonemeClass = (phoneme) => {
  if (!phoneme.isMatch) {
    if (phoneme.user === '-') return 'phoneme-omission';
    if (phoneme.target === '-') return 'phoneme-insertion';
    return 'phoneme-substitution';
  }
  return 'phoneme-correct';
};
</script>

<template>
  <div class="analyzer-container">
    <!-- 【【【 新增的 Ngrok URL 輸入區域 】】】 -->
    <div class="api-url-section">
      <label for="api-url">Backend Ngrok URL:</label>
      <input type="text" id="api-url" v-model="apiBaseUrl" placeholder="例如: https://xxxx.ngrok-free.app">
      <button @click="saveApiUrl">儲存</button>
    </div>
    <!-- 輸入區域 -->
    <div class="input-section">
      <label for="sentence">Target Sentence:</label>
      <input type="text" id="sentence" v-model="targetSentence">
      <button 
        @click="toggleRecording" 
        :class="{ 'recording': isRecording }" 
        :disabled="isLoading || !savedApiBaseUrl"
        :title="!savedApiBaseUrl ? '請先輸入並儲存後端位址' : ''"
      >
        {{ isRecording ? 'Stop & Analyze' : (isLoading ? 'Analyzing...' : 'Start Recording' ) }}
      </button>
    </div>

    <!-- 狀態顯示區域 -->
    <div v-if="statusMessage" class="status-message">{{ statusMessage }}</div>
    <div v-if="errorMessage" class="status-message error">{{ errorMessage }}</div>

    <!-- 結果顯示區域 (使用我們最終修正的、包含所有細節的版本) -->
    <div v-if="analysisResult && analysisResult.summary && analysisResult.words" class="report-container">
      <pre class="report-pre">
======================================================================
                        Pronunciation Analysis
======================================================================

Sentence: {{ analysisResult.sentence }}

Target  : <span v-for="(word, index) in analysisResult.words" :key="`target-${index}`">[ <span v-for="(p, pIndex) in word.phonemes" :key="`t-${pIndex}`">{{ p.target }} </span>] </span>
User    : <span v-for="(word, index) in analysisResult.words" :key="`user-${index}`">[ <span v-for="(p, pIndex) in word.phonemes" :key="`u-${pIndex}`" :class="getPhonemeClass(p)">{{ p.user }} </span>] </span>

----------------------------------------------------------------------
[ Summary ]
----------------------------------------------------------------------
- Overall Score:         {{ analysisResult.summary.overallScore.toFixed(1) }}%
- Total Words:           {{ analysisResult.summary.totalWords }}
- Correct Words:         {{ analysisResult.summary.correctWords }}
- Incorrect Words:       {{ analysisResult.summary.totalWords - analysisResult.summary.correctWords }}
- Phoneme Error Rate:    {{ analysisResult.summary.phonemeErrorRate.toFixed(2) }}% ({{ analysisResult.summary.total_errors }} errors in {{ analysisResult.summary.total_target_phonemes }} target phonemes)
- Analysis Timestamp:    {{ analysisResult.analysisTimestampUTC }}

======================================================================
      </pre>
    </div>
  </div>
</template>

<style scoped>
/* 新增 API URL 區域的樣式 */
.api-url-section {
  display: flex;
  gap: 1rem;
  align-items: center;
  margin-bottom: 1.5rem;
  background-color: #2f343d;
  padding: 1rem;
  border-radius: 8px;
  border: 1px solid #444;
}
.api-url-section label {
  font-weight: bold;
  white-space: nowrap;
}
.api-url-section input {
  flex-grow: 1;
  padding: 0.5rem;
  background-color: #333;
  border: 1px solid #444;
  color: #e1e1e1;
  border-radius: 4px;
}
.api-url-section button {
  padding: 0.5rem 1rem;
  background-color: #5c6bc0; /* Indigo color */
  color: #fff;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.analyzer-container {
  background-color: #20232a;
  border: 1px solid #444;
  padding: 2rem;
  border-radius: 8px;
  color: #e1e1e1;
}

.input-section {
  display: flex;
  gap: 1rem;
  align-items: center;
  margin-bottom: 2rem;
}

.input-section label {
  font-weight: bold;
}

.input-section input {
  flex-grow: 1;
  padding: 0.5rem;
  background-color: #333;
  border: 1px solid #444;
  color: #e1e1e1;
  border-radius: 4px;
}

.input-section button {
  padding: 0.5rem 1rem;
  background-color: #61dafb;
  color: #000;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s;
}

.input-section button:disabled {
  background-color: #555;
  cursor: not-allowed;
}

.input-section button.recording {
  background-color: #ff6b6b;
  color: #fff;
}

.status-message {
  text-align: center;
  margin: 1rem 0;
  font-size: 1.1rem;
}
.status-message.error {
  color: #ff6b6b;
  font-weight: bold;
}

.report-container {
  margin-top: 2rem;
}

.report-pre {
  background-color: #1a1a1a;
  padding: 1.5rem;
  border-radius: 8px;
  white-space: pre-wrap;
  word-wrap: break-word;
  font-size: 1rem;
  line-height: 1.6;
  border: 1px solid #444;
}

.phoneme-correct { color: #a5d6a7; }
.phoneme-substitution { color: #ef9a9a; font-weight: bold; }
.phoneme-omission { color: #ffcc80; }
.phoneme-insertion { color: #b39ddb; }
</style>
