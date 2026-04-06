<template>
  <div id="app">
    <!-- 頂部標題 -->
    <header>
      <h1>🛡️ 香港防騙AI助手</h1>
      <p class="subtitle">輸入可疑訊息，我會幫你分析是否詐騙</p>
    </header>

    <!-- 主要聊天區域 -->
    <main>
      <div class="chat-container" ref="chatContainer">
        <!-- 顯示所有訊息 -->
        <div v-for="(msg, index) in messages" :key="index" 
             :class="['message', msg.type === 'user' ? 'user-message' : 'bot-message']">
          <div class="avatar">
            {{ msg.type === 'user' ? '👤' : '🤖' }}
          </div>
          <div class="content">
            <div class="text">{{ msg.content }}</div>
            
            <!-- 如果有分析結果，就顯示 -->
            <div v-if="msg.analysis" class="analysis-result" :class="msg.analysis.riskLevel">
              <div class="risk-badge">
                {{ msg.analysis.riskLevel }} 風險
              </div>
              <div class="scam-type" v-if="msg.analysis.scamType">
                🕵️ 類型：{{ msg.analysis.scamType }}
              </div>
              <div class="advice-list">
                <div v-for="(advice, i) in msg.analysis.advice" :key="i" class="advice-item">
                  {{ advice }}
                </div>
              </div>
              
              <!-- 報案草稿按鈕 -->
              <div class="report-section" v-if="msg.analysis">
                <button @click="generateReport(msg)" class="report-btn" :disabled="reportLoading">
                  📄 生成報案草稿
                </button>
                <div v-if="msg.report" class="report-content">
                  <pre>{{ msg.report }}</pre>
                  <button @click="copyReport(msg.report)" class="copy-btn">
                    📋 一鍵複製
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <!-- 圖片上傳預覽區域（獨立顯示） -->
        <div v-if="uploadPreview" class="upload-preview-message">
          <div class="message bot-message">
            <div class="avatar">📷</div>
            <div class="content">
              <div class="text">上傳嘅圖片：</div>
              <div class="upload-preview">
                <img :src="uploadPreview" alt="Preview">
                <button @click="clearUpload" class="remove-image">✕</button>
              </div>
              <div v-if="ocrResult" class="ocr-result">
                <div class="ocr-text">識別到文字：{{ ocrResult }}</div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 輸入區域 -->
      <div class="input-area">
        <textarea
          v-model="userInput"
          placeholder="請輸入可疑訊息..."
          @keyup.enter="sendMessage"
          rows="2"
        ></textarea>
        
        <!-- 圖片上傳區 -->
        <div class="upload-section">
          <div class="upload-area" @click="triggerFileInput" @dragover.prevent @drop="handleDrop">
            <input 
              type="file" 
              ref="fileInput" 
              @change="handleFileUpload" 
              accept="image/*" 
              style="display: none"
            >
            <span class="upload-icon">📷</span>
            <span>{{ uploadLoading ? '分析中...' : '上傳截圖' }}</span>
          </div>
        </div>
        
        <div class="actions">
          <button @click="sendMessage" :disabled="!userInput.trim() || loading">
            {{ loading ? '分析中...' : '發送分析' }}
          </button>
        </div>
      </div>
    </main>

    <!-- 底部小提示 -->
    <footer>
      <p>📞 防騙易諮詢熱線：18222 | 警方防騙視伏器：fallback.mirror.cyberdefender.gov.hk</p>
    </footer>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'App',
  data() {
    return {
      userInput: '',
      loading: false,
      reportLoading: false,
      uploadLoading: false,
      uploadPreview: null,
      uploadFile: null,
      ocrResult: '',
      messages: [
        {
          type: 'bot',
          content: '你好！我是防騙AI助手。你可以貼上任何可疑的 WhatsApp、SMS 或 Telegram 訊息，或者上傳截圖，我會幫你分析是否詐騙。'
        }
      ]
    }
  },
  methods: {
    // 圖片上傳相關
    triggerFileInput() {
      this.$refs.fileInput.click();
    },

    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        this.processUpload(file);
      }
    },

    handleDrop(event) {
      event.preventDefault();
      const file = event.dataTransfer.files[0];
      if (file) {
        this.processUpload(file);
      }
    },

    processUpload(file) {
      // 檢查檔案類型
      if (!file.type.startsWith('image/')) {
        alert('請上傳圖片檔案');
        return;
      }
      
      // 檢查檔案大小（限制 5MB）
      if (file.size > 5 * 1024 * 1024) {
        alert('圖片不能超過 5MB');
        return;
      }
      
      this.uploadFile = file;
      
      // 生成預覽
      const reader = new FileReader();
      reader.onload = (e) => {
        this.uploadPreview = e.target.result;
      };
      reader.readAsDataURL(file);
      
      // 自動開始 OCR 分析
      this.analyzeImage(file);
    },

    clearUpload() {
      this.uploadPreview = null;
      this.uploadFile = null;
      this.ocrResult = '';
      this.$refs.fileInput.value = '';
    },

    async analyzeImage(file) {
      this.uploadLoading = true;
      
      const formData = new FormData();
      formData.append('image', file);
      
      try {
        // 1. 先 call OCR API 提取文字
        const ocrResponse = await axios.post('http://localhost:8080/api/ocr/extract', formData, {
          headers: { 'Content-Type': 'multipart/form-data' }
        });
        
        if (ocrResponse.data.success) {
          const extractedText = ocrResponse.data.extractedText;
          this.ocrResult = extractedText;
          
          // 2. 將提取嘅文字放入輸入框
          this.userInput = extractedText;
          
          // 3. 自動彈出提示
          this.$nextTick(() => {
            alert('✅ 圖片文字識別成功！已自動填入輸入框，按「發送分析」即可。');
          });
        } else {
          alert('❌ 圖片文字識別失敗：' + ocrResponse.data.message);
        }
        
      } catch (error) {
        console.error('OCR Error:', error);
        alert('❌ OCR 處理失敗，請稍後再試。');
      } finally {
        this.uploadLoading = false;
      }
    },

    // 發送訊息分析
    async sendMessage() {
      if (!this.userInput.trim() || this.loading) return;
      
      // 1. 加入用戶的訊息
      this.messages.push({
        type: 'user',
        content: this.userInput
      });
      
      this.loading = true;
      
      try {
        // 2. 叫後端 API 分析
        const response = await axios.post('http://localhost:8080/api/analyze', {
          message: this.userInput
        });
        
        // 3. 加入 AI 的回覆
        this.messages.push({
          type: 'bot',
          content: '分析完成：',
          analysis: response.data
        });
        
        // 4. 清空 OCR 預覽（如果係來自圖片）
        this.clearUpload();
        
      } catch (error) {
        console.error('API 錯誤:', error);
        
        // 如果後端未準備好，用返本地分析
        const analysis = this.fallbackAnalyze(this.userInput);
        this.messages.push({
          type: 'bot',
          content: '分析完成（離線模式）：',
          analysis: analysis
        });
      }
      
      this.loading = false;
      this.userInput = '';
      
      // 5. 滾動到最新訊息
      this.$nextTick(() => {
        const container = this.$refs.chatContainer;
        container.scrollTop = container.scrollHeight;
      });
    },

    // 生成報案草稿
    async generateReport(msg) {
      this.reportLoading = true;
      
      try {
        const response = await axios.post('http://localhost:8080/api/generate-report', {
          message: msg.content,
          riskLevel: msg.analysis.riskLevel,
          scamType: msg.analysis.scamType
        });
        
        msg.report = response.data.report;
        
      } catch (error) {
        console.error('Report generation error:', error);
        alert('生成報案草稿失敗，請稍後再試。');
      } finally {
        this.reportLoading = false;
      }
    },

    // 複製報案草稿
    copyReport(report) {
      navigator.clipboard.writeText(report).then(() => {
        alert('✅ 報案草稿已複製到剪貼簿！');
      }).catch(err => {
        console.error('Copy failed:', err);
        alert('❌ 複製失敗，請手動選取複製。');
      });
    },

    // 後備分析（當後端未準備好時用）
    fallbackAnalyze(message) {
      // 詐騙關鍵字
      const scamPatterns = {
        high: ['凍結', '通緝', '洗黑錢', '公安', '法院', '入境處', '拘捕', '犯罪', '調查', '保密', '安全賬戶'],
        medium: ['點擊連結', '驗證', '退款', '重複扣款', '解除設定', '更新資料', '帳戶異常', '積分到期', '中獎', '禮物'],
        low: ['優惠', '折扣', '免費', '促銷', '限時']
      };
      
      let riskLevel = 'LOW';
      let foundKeywords = [];
      let scamType = '';
      
      for (let keyword of scamPatterns.high) {
        if (message.includes(keyword)) {
          riskLevel = 'HIGH';
          foundKeywords.push(keyword);
          if (keyword.includes('公安') || keyword.includes('法院') || keyword.includes('入境處')) {
            scamType = '假冒官員';
          } else if (keyword.includes('凍結') || keyword.includes('洗黑錢')) {
            scamType = '虛假指控';
          }
          break;
        }
      }
      
      if (riskLevel === 'LOW') {
        for (let keyword of scamPatterns.medium) {
          if (message.includes(keyword)) {
            riskLevel = 'MEDIUM';
            foundKeywords.push(keyword);
            if (keyword.includes('點擊連結')) {
              scamType = '釣魚連結';
            } else if (keyword.includes('退款') || keyword.includes('重複扣款')) {
              scamType = '假冒客服';
            } else if (keyword.includes('中獎')) {
              scamType = '虚假中獎';
            }
            break;
          }
        }
      }
      
      let advice = [];
      if (riskLevel === 'HIGH') {
        advice = [
          '❌ 這是高風險詐騙！',
          '❌ 不要點擊任何連結',
          '❌ 不要提供個人資料',
          '✅ 打18222向防騙易諮詢'
        ];
      } else if (riskLevel === 'MEDIUM') {
        advice = [
          '⚠️ 此訊息有可疑特徵',
          '❌ 不要點擊連結',
          '✅ 有疑問可致電18222'
        ];
      } else {
        advice = [
          '✅ 沒有發現明顯詐騙特徵',
          '💡 但仍需保持警惕'
        ];
      }
      
      if (foundKeywords.length > 0) {
        advice.unshift(`🔍 發現可疑關鍵字：${foundKeywords.join('、')}`);
      }
      
      return {
        riskLevel,
        scamType: scamType || (riskLevel !== 'LOW' ? '疑似詐騙' : ''),
        advice,
        confidence: 0.85
      };
    }
  }
}
</script>

<style>
/* 全部 style 同之前一樣，只係加少少 upload 相關嘅樣式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Microsoft JhengHei', 'PingFang TC', sans-serif;
  background-color: #f0f2f5;
}

#app {
  max-width: 800px;
  margin: 0 auto;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: white;
  box-shadow: 0 0 20px rgba(0,0,0,0.1);
}

header {
  background: linear-gradient(135deg, #1a73e8, #0d47a1);
  color: white;
  padding: 15px 20px;
  text-align: center;
}

header h1 {
  font-size: 22px;
  margin-bottom: 5px;
}

.subtitle {
  font-size: 13px;
  opacity: 0.9;
}

main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background-color: #f8f9fa;
}

.chat-container {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.message {
  display: flex;
  gap: 10px;
  max-width: 85%;
}

.user-message {
  align-self: flex-end;
  flex-direction: row-reverse;
}

.avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  flex-shrink: 0;
}

.user-message .avatar {
  background: linear-gradient(135deg, #f093fb, #f5576c);
}

.content {
  flex: 1;
}

.text {
  background-color: white;
  padding: 10px 15px;
  border-radius: 18px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  word-break: break-word;
  line-height: 1.4;
  font-size: 14px;
}

.user-message .text {
  background: linear-gradient(135deg, #1a73e8, #0d47a1);
  color: white;
}

.analysis-result {
  margin-top: 10px;
  padding: 12px;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  font-size: 13px;
}

.analysis-result.HIGH {
  background-color: #ffebee;
  border-left: 4px solid #c62828;
}

.analysis-result.MEDIUM {
  background-color: #fff3e0;
  border-left: 4px solid #ef6c00;
}

.analysis-result.LOW {
  background-color: #e8f5e8;
  border-left: 4px solid #2e7d32;
}

.risk-badge {
  display: inline-block;
  padding: 4px 10px;
  border-radius: 15px;
  font-weight: bold;
  font-size: 12px;
  margin-bottom: 8px;
}

.HIGH .risk-badge {
  background-color: #c62828;
  color: white;
}

.MEDIUM .risk-badge {
  background-color: #ef6c00;
  color: white;
}

.LOW .risk-badge {
  background-color: #2e7d32;
  color: white;
}

.scam-type {
  font-weight: bold;
  margin-bottom: 8px;
  padding-bottom: 5px;
  border-bottom: 1px dashed #ccc;
  font-size: 13px;
}

.advice-list {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.advice-item {
  padding: 3px 0;
  font-size: 13px;
  line-height: 1.4;
}

.input-area {
  padding: 15px;
  background-color: white;
  border-top: 1px solid #e0e0e0;
}

textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  font-size: 14px;
  font-family: inherit;
  resize: none;
}

textarea:focus {
  outline: none;
  border-color: #1a73e8;
}

.upload-section {
  margin: 10px 0;
}

.upload-area {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 8px;
  border: 1px dashed #ccc;
  border-radius: 8px;
  cursor: pointer;
  font-size: 13px;
  color: #666;
}

.upload-area:hover {
  border-color: #1a73e8;
  background-color: #f8f9fa;
}

.upload-icon {
  font-size: 18px;
}

.upload-preview-message {
  margin-bottom: 10px;
}

.upload-preview {
  position: relative;
  max-width: 200px;
  margin-top: 8px;
}

.upload-preview img {
  width: 100%;
  max-height: 150px;
  object-fit: contain;
  border-radius: 5px;
  border: 1px solid #ddd;
}

.remove-image {
  position: absolute;
  top: -8px;
  right: -8px;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #ff4444;
  color: white;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
}

.ocr-result {
  margin-top: 8px;
  padding: 8px;
  background-color: #f0f0f0;
  border-radius: 5px;
  font-size: 13px;
}

.ocr-text {
  word-break: break-word;
  max-height: 100px;
  overflow-y: auto;
}

.actions {
  display: flex;
  justify-content: flex-end;
}

button {
  padding: 8px 20px;
  background: linear-gradient(135deg, #1a73e8, #0d47a1);
  color: white;
  border: none;
  border-radius: 20px;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.report-section {
  margin-top: 12px;
  border-top: 1px dashed #ccc;
  padding-top: 10px;
}

.report-btn {
  background: #28a745;
  padding: 5px 12px;
  font-size: 12px;
  margin-bottom: 8px;
}

.report-btn:hover {
  background: #218838;
}

.report-content {
  background: #f8f9fa;
  border-radius: 8px;
  padding: 10px;
  position: relative;
}

.report-content pre {
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: inherit;
  font-size: 12px;
  margin: 0;
  padding: 8px;
  background: white;
  border-radius: 5px;
  max-height: 200px;
  overflow-y: auto;
}

.copy-btn {
  position: absolute;
  top: 15px;
  right: 15px;
  background: #007bff;
  padding: 3px 8px;
  font-size: 11px;
}

footer {
  padding: 8px;
  text-align: center;
  font-size: 11px;
  color: #666;
  background-color: #f8f9fa;
  border-top: 1px solid #e0e0e0;
}
</style>