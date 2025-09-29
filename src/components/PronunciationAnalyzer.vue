  <!-- src/components/PronunciationAnalyzer.vue (The Final, Absolute, Shameful Fix) -->
  <script setup>
  import { ref, onMounted, computed } from 'vue'; // 引入 computed
  import axios from 'axios';
  import Recorder from 'recorder-core';
  import 'recorder-core/src/engine/wav.js';

  // --- 狀態變數 ---
  const targetSentence = ref('');
  const analysisResult = ref(null);
  const isLoading = ref(false);
  const errorMessage = ref('');
  const statusMessage = ref('');
  const isRecording = ref(false);
  let rec;

  // --- 【【【 核心修改：動態 API URL 】】】 ---
  const apiBaseUrl = ref(''); // 用於雙向綁定輸入框
  const savedApiBaseUrl = ref(''); // 儲存並實際使用的 URL

  // --- 【【【【【 新增：語言相關變數 】】】】】 ---
  const languageInput = ref('en_us'); // 用於綁定輸入框，就像 apiBaseUrl
  const language = ref('en_us');      // 實際在 API 請求中使用的語言，就像 savedApiBaseUrl

  // onMounted 會在元件掛載到頁面後執行
  onMounted(() => {
    // 嘗試從瀏覽器的 localStorage 中讀取已儲存的 URL
    const storedUrl = localStorage.getItem('ngrokApiBaseUrl');
    if (storedUrl) {
      apiBaseUrl.value = storedUrl;
      savedApiBaseUrl.value = storedUrl;
      statusMessage.value = '已載入儲存的 ASR 後端位址。';
    } else {
      statusMessage.value = '請輸入並儲存您的 ASR 後端位址。';
    }

    // --- 【【【【【 新增：讀取已儲存的語言 】】】】】 ---
    const storedLanguage = localStorage.getItem('pronunciationLanguage');
    if (storedLanguage) {
      languageInput.value = storedLanguage;
      language.value = storedLanguage;
      // 如果需要，可以添加一條狀態訊息
      // statusMessage.value += ` 已載入語言設定: ${storedLanguage}`;
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

  // --- 【【【【【 新增：儲存語言的函數 】】】】】 ---
  const saveLanguage = () => {
    const cleanedLanguage = languageInput.value.trim();
    language.value = cleanedLanguage;
    localStorage.setItem('pronunciationLanguage', cleanedLanguage);
    statusMessage.value = `語言已設定為: ${cleanedLanguage}`;
    errorMessage.value = '';
  };

  // --- 核心功能：錄音與分析 ---
  const toggleRecording = ( ) => {
    if (isRecording.value) {
      // 停止錄音
      statusMessage.value = '正在處理錄音...';
      rec.stop(
        (blob, duration) => {
          statusMessage.value = '錄音結束，正在上傳分析...';
          rec.close();
          analyzePronunciation(blob);
        },
        (err) => {
          errorMessage.value = `錄音出錯: ${err}`;
          statusMessage.value = '';
          isRecording.value = false;
          if (rec) {
            rec.close();
          }
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
  // 1. 修正 Unicode 視覺寬度計算
  const getVisualLength = (str) => {
    if (!str) return 0;
    // 這個正則表達式會移除所有 Unicode 組合標記 (如鼻化符 ~)
    // \p{M} 是 Unicode 屬性轉義，匹配所有 "Mark" 類別的字元
    // 'u' 標誌啟用 Unicode 支持
    return str.replace(/\p{M}/gu, '').length;
  };

  // 2. 獲取單個單詞中，每一「列」的最大視覺寬度
  const getColumnWidths = (word) => {
    if (!word || !word.phonemes) return [];
    const widths = [];
    for (let i = 0; i < word.phonemes.length; i++) {
      const p = word.phonemes[i];
      const targetLen = getVisualLength(p.target);
      const userLen = getVisualLength(p.user);
      widths.push(Math.max(targetLen, userLen, 1)); // 最小寬度為 1
    }
    return widths;
  };

  // 3. 根據「列寬度」來填充單個音素
  const padPhoneme = (phonemeStr, columnWidth) => {
    const visualLen = getVisualLength(phonemeStr);
    const padding = ' '.repeat(columnWidth - visualLen);
    return phonemeStr + padding + ' '; // 填充後再加一個分隔空格
  };

  // 4. getPhonemeClass 保持不變
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
      <!-- URL 輸入區域 (保持不變) -->
      <div class="settings-container">
        <!-- 您現有的 Ngrok URL 輸入區域 -->
        <div class="api-url-section">
          <label for="api-url">ASR Service URL:</label>
          <input type="text" id="api-url" v-model="apiBaseUrl" placeholder="例如: https://xxxx.ngrok-free.app">
          <button @click="saveApiUrl">Save</button>
        </div>

        <!-- 【【【【【 新增的語言輸入區域 】】】】】 -->
        <div class="api-url-section"> <!-- 復用完全相同的 class 來保證樣式一致 -->
          <label for="language-input">Language Code:</label>
          <input type="text" id="language-input" v-model="languageInput" placeholder="例如: en_us">
          <!-- 語言輸入是即時綁定的 ，不需要儲存按鈕，但為了視覺統一，可以保留 -->
          <!-- 如果不需要按鈕，可以直接刪除下面的 button 標籤 -->
          <button @click="saveLanguage">Save</button>
        </div>
      </div>
      <!-- 輸入區域 (保持不變 ) -->
      <div class="input-section">
        <label for="sentence">Target Sentence:</label>
        <input type="text" id="sentence" v-model="targetSentence" placeholder="Enter a sentence here">
        <button 
          @click="toggleRecording" 
          :class="{ 'recording': isRecording }" 
          :disabled="isLoading || !savedApiBaseUrl"
          :title="!savedApiBaseUrl ? '請先輸入並儲存後端位址' : ''"
        >
          {{ isRecording ? 'Stop & Analyze' : (isLoading ? 'Analyzing...' : 'Start Recording' ) }}
        </button>
      </div>

      <!-- 狀態顯示區域 (保持不變) -->
      <div v-if="statusMessage" class="status-message">{{ statusMessage }}</div>
      <div v-if="errorMessage" class="status-message error">{{ errorMessage }}</div>

      <!-- 結果顯示區域 (已重構為 Flexbox 佈局) -->
      <div v-if="analysisResult && analysisResult.summary && analysisResult.words" class="report-container">
        <!-- 不再使用 <pre>，改用普通的 div -->
        <div class="report-content">
          <div class="report-title-container">
            <hr class="report-divider main">
            <div class="title-text">Pronunciation Analysis</div>
            <hr class="report-divider main">
          </div>

          <!-- 統一三行的外層 class，讓 y 間距一致 -->
          <div class="analysis-line">
            <span class="line-label">Sentence : </span>
            <span>{{ analysisResult.sentence }}</span>
          </div>

          <div class="analysis-line">
            <span class="line-label">Target&nbsp;&nbsp; :</span>
            <div class="phoneme-wrapper">
              <!-- 對於每個單詞，我們先計算列寬度 -->
              <span class="word-block" v-for="(word, wordIndex) in analysisResult.words" :key="`target-word-${wordIndex}`">
                <template v-if="word.phonemes">
                  [ <span v-for="(p, pIndex) in word.phonemes" :key="`t-phoneme-${pIndex}`">
                      {{ padPhoneme(p.target, getColumnWidths(word)[pIndex]) }}
                    </span>]
                </template>
              </span>
            </div>
          </div>

          <div class="analysis-line">
            <span class="line-label">User&nbsp;&nbsp;&nbsp;&nbsp; :</span>
            <div class="phoneme-wrapper">
              <span class="word-block" v-for="(word, wordIndex) in analysisResult.words" :key="`user-word-${wordIndex}`">
                <template v-if="word.phonemes">
                  [ <span v-for="(p, pIndex) in word.phonemes" :key="`u-phoneme-${pIndex}`" :class="getPhonemeClass(p)">
                      {{ padPhoneme(p.user, getColumnWidths(word)[pIndex]) }}
                    </span>]
                </template>
              </span>
            </div>
          </div>

          <div class="report-title-container">
            <hr class="report-divider sub">
            <div class="title-text">[ Summary ]</div>
            <hr class="report-divider sub">
          </div>

          <!-- 關鍵修改：為 Summary 的每一項都用 div 包裹，以確保換行 -->
          <div class="summary-details">
            <div>- Overall Score:         {{ analysisResult.summary.overallScore.toFixed(1) }}%</div>
            <div>- Total Words:           {{ analysisResult.summary.totalWords }}</div>
            <div>- Correct Words:         {{ analysisResult.summary.correctWords }}</div>
            <div>- Incorrect Words:       {{ analysisResult.summary.totalWords - analysisResult.summary.correctWords }}</div>
            <div>- Phoneme Error Rate:    {{ analysisResult.summary.phonemeErrorRate.toFixed(2) }}% ({{ analysisResult.summary.total_errors }} errors in {{ analysisResult.summary.total_target_phonemes }} target phonemes)</div>
            <div>- Analysis Timestamp:    {{ analysisResult.analysisTimestampUTC }}</div>
          </div>

          <div class="report-title-container">
            <hr class="report-divider main">
          </div>
        </div>
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

  /* 刪除您原有的 .report-pre 樣式 */

  .report-content {
    background-color: #1a1a1a;
    padding: 1.5rem;
    border-radius: 8px;
    border: 1px solid #444;
    font-family: monospace;
    font-size: 1rem;
    line-height: 1.6;
  }

  /* 新增：標題容器樣式 */
  .report-title-container {
    text-align: center;
    margin: 1rem 0;
  }

  .report-divider {
    border: none;
    height: 2px;
    background: linear-gradient(to right, #444 0%, #888 50%, #444 100%);
    margin: 0.5em 0;
    width: 100%;
  }
  .report-divider.main {
    height: 3px;
    background: linear-gradient(to right, #444 0%, #fff 50%, #444 100%);
  }
  .report-divider.sub {
    height: 2px;
    background: linear-gradient(to right, #444 0%, #aaa 50%, #444 100%);
  }

  .analysis-line {
    display: flex;
    align-items: flex-start;
    margin-bottom: 0.5rem;
  }

  .line-label {
    flex-shrink: 0;
    white-space: pre;
  }

  .phoneme-wrapper {
    display: flex;
    flex-wrap: wrap;
  }

  .word-block {
    display: inline-block;
    white-space: pre;
  }

  .phoneme-correct { color: #a5d6a7; }
  .phoneme-substitution { color: #ef9a9a; font-weight: bold; }
  .phoneme-omission { color: #ffcc80; }
  .phoneme-insertion { color: #b39ddb; }
  </style>
