<!-- src/App.vue (The Final, Absolutely Correct Version) -->
<script setup>
import PronunciationAnalyzer from './components/PronunciationAnalyzer.vue';
</script>

<template>
  <div id="app-container">
    <header>
      <h1>Pronunciation Analysis Tool</h1>
    </header>
    <main>
      <PronunciationAnalyzer />
    </main>
  </div>
</template>

<!-- 【【【【【 終 極 修 正 在 這 裡 】】】】】 -->
<style scoped>
/* :root 和 body 應該在全域的 main.css 中，但為了快速修復，我們先這樣處理 */
:root {
  --dark-bg: #282c34;
  --light-text: #e1e1e1; /* 提高文字亮度 */
  --green: #61dafb;
  --red: #ff6b6b;
  --border-color: #444;
}

/* 移除 body 樣式，因為它會造成全域污染，且在 scoped 中無效 */

#app-container {
  max-width: 900px;
  margin: auto;
  /* 為整個容器設定文字顏色，這樣所有子元件都會繼承 */
  color: var(--light-text); 
}

header h1 {
  text-align: center;
  color: var(--green);
  border-bottom: 1px solid var(--border-color);
  padding-bottom: 1rem;
}
</style>