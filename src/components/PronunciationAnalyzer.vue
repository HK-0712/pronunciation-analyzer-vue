<!-- src/components/PronunciationAnalyzer.vue (The Final, Absolute, Shameful Fix) -->
<script setup>
import { ref } from 'vue';
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

// --- API 設定 ---
const API_BASE_URL = 'https://b40c635fc726.ngrok-free.app'; // 請確保這是您最新的 Ngrok URL

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
    const response = await axios.post(`${API_BASE_URL}/api/v1/recognize`, formData, {
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
    <!-- 輸入區域 -->
    <div class="input-section">
      <label for="sentence">Target Sentence:</label>
      <input type="text" id="sentence" v-model="targetSentence">
      <button @click="toggleRecording" :class="{ 'recording': isRecording }" :disabled="isLoading">
        {{ isRecording ? 'Stop & Analyze' : (isLoading ? 'Analyzing...' : 'Start Recording') }}
      </button>
    </div>

    <!-- 狀態顯示區域 -->
    <div v-if="statusMessage" class="status-message">{{ statusMessage }}</div>
    <div v-if="errorMessage" class="status-message error">{{ errorMessage }}</div>

    <!-- 結果顯示區域 (使用防禦性渲染) -->
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
- Phoneme Error Rate:    {{ analysisResult.summary.phonemeErrorRate.toFixed(2) }}%
- Analysis Timestamp:    {{ new Date(analysisResult.analysisTimestampUTC).toLocaleString() }}

======================================================================
      </pre>
    </div>
  </div>
</template>

<style scoped>
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
