<template>
  <div class="retro-terminal">
    <div class="status-bar">
      <span class="sys-msg">SYSTEM: <span class="blink">ONLINE</span></span>
      <span class="sys-msg">MODE: <span class="highlight">FORENSICS</span></span>
      <span class="sys-time">{{ currentTime }}</span>
    </div>

    <div class="input-section">
      <div class="prompt-line">
        <span class="prompt">> ENTER API CONFIGURATION_</span>
      </div>
      
      <div class="grid-form">
        <div class="form-group">
          <label>API KEY</label>
          <input v-model="config.apiKey" type="text" class="retro-input" placeholder="Paste Key Here..." />
        </div>
        <div class="form-group">
          <label>SECRET KEY</label>
          <input v-model="config.secretKey" type="password" class="retro-input" placeholder="Paste Secret Here..." />
        </div>
        <div class="form-group">
          <label>PASSPHRASE</label>
          <input v-model="config.passphrase" type="password" class="retro-input" placeholder="Your Passphrase" />
        </div>
        <button @click="fetchHistory" class="retro-btn" :disabled="loading">
          {{ loading ? 'DOWNLOADING...' : 'INITIATE DATA RETRIEVAL' }}
        </button>
      </div>
      
      <div v-if="errorMsg" class="error-log">
        [ERROR] {{ errorMsg }}
      </div>
    </div>

    <div class="display-area" v-if="orders.length > 0">
      <div class="area-header">=== DECLASSIFIED ORDER HISTORY (LAST 3 MONTHS) ===</div>
      <table class="retro-table">
        <thead>
          <tr>
            <th>TIME</th>
            <th>SYMBOL</th>
            <th>SIDE</th>
            <th>TYPE</th>
            <th>PRICE</th>
            <th>AVG.PRICE</th>
            <th>FILLED</th>
            <th>PnL</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="order in orders" :key="order.ordId">
            <td class="col-time">{{ formatTime(order.uTime) }}</td>
            <td class="col-symbol">{{ order.instId }}</td>
            <td :class="getSideClass(order.posSide)">{{ order.posSide.toUpperCase() }}</td>
            <td>{{ order.ordType.toUpperCase() }}</td>
            <td class="mono-num">{{ order.px || 'MKT' }}</td>
            <td class="mono-num">{{ order.avgPx }}</td>
            <td class="mono-num">{{ order.accFillSz }}</td>
            <td :class="getPnlClass(order.pnl)">{{ order.pnl }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';
import CryptoJS from 'crypto-js';

// --- 状态 ---
const currentTime = ref('');
const loading = ref(false);
const errorMsg = ref('');
const orders = ref([]);

// 这里的默认值仅供测试，请填写新的 Key
const config = reactive({
  apiKey: '', 
  secretKey: '',
  passphrase: '' 
});

// --- 核心工具：OKX 签名算法 (V5) ---
const signRequest = (timestamp, method, requestPath, body = '') => {
  const message = timestamp + method + requestPath + body;
  const hmac = CryptoJS.HmacSHA256(message, config.secretKey);
  return CryptoJS.enc.Base64.stringify(hmac);
};

// --- 核心逻辑：拉取历史订单 ---
const fetchHistory = async () => {
  if (!config.apiKey || !config.secretKey || !config.passphrase) {
    errorMsg.value = 'MISSING CREDENTIALS. ACCESS DENIED.';
    return;
  }

  loading.value = true;
  errorMsg.value = '';
  
  try {
    // 1. 准备请求参数
    // OKX API: 获取最近3个月的历史订单 (archive)
    // InstType: SWAP (永续合约), 可以改为 SPOT (现货)
    const method = 'GET';
    const path = '/api/v5/trade/orders-history-archive?instType=SWAP'; 
    // 注意：如果是最近7天，用 orders-history。如果是更久，用 orders-history-archive
    
    // 2. 生成签名头
    const timestamp = new Date().toISOString();
    // 注意：这里的 path 必须和实际发给 OKX 的 path 一致，不包含域名前缀
    // 但在发 fetch 时，我们要发给本地的代理 '/api/okx'
    const signature = signRequest(timestamp, method, path);

    // 3. 发送请求 (通过 Vite Proxy)
    const proxyUrl = `/api/okx${path.replace('/api', '')}`; 
    // 解释：上面 config 配置的是 rewrite，这里我们需要拼接出 '/api/okx/v5/trade/...'

    const response = await fetch(proxyUrl, {
      method: method,
      headers: {
        'OK-ACCESS-KEY': config.apiKey,
        'OK-ACCESS-SIGN': signature,
        'OK-ACCESS-TIMESTAMP': timestamp,
        'OK-ACCESS-PASSPHRASE': config.passphrase,
        'Content-Type': 'application/json'
      }
    });

    const data = await response.json();

    if (data.code !== '0') {
      throw new Error(`OKX ERROR: ${data.msg} (Code: ${data.code})`);
    }

    orders.value = data.data;

  } catch (err) {
    console.error(err);
    errorMsg.value = err.message || 'NETWORK CONNECTION FAILED';
  } finally {
    loading.value = false;
  }
};

// --- 辅助函数 ---
const formatTime = (ts) => {
  return new Date(parseInt(ts)).toLocaleString('zh-CN', { hour12: false });
};
const getSideClass = (side) => {
  if (side === 'long') return 'text-green';
  if (side === 'short') return 'text-red';
  return '';
};
const getPnlClass = (pnl) => {
  const val = parseFloat(pnl);
  if (val > 0) return 'text-green bold';
  if (val < 0) return 'text-red bold';
  return 'text-gray';
};

// 时钟
onMounted(() => {
  setInterval(() => {
    currentTime.value = new Date().toISOString().replace('T', ' ').slice(0, 19);
  }, 1000);
});
</script>

<style scoped>
/* --- 复古终端 CSS 风格 --- */
@import url('https://fonts.googleapis.com/css2?family=VT323&display=swap');

.retro-terminal {
  background-color: #0d0d0d; /* 深黑背景 */
  color: #33ff33; /* 经典的荧光绿 */
  font-family: 'VT323', monospace; /* 像素字体 */
  font-size: 18px;
  min-height: 100vh;
  padding: 20px;
  border: 4px double #333;
  box-shadow: inset 0 0 20px rgba(0, 255, 0, 0.1);
  text-transform: uppercase;
}

/* 顶部栏 */
.status-bar {
  border-bottom: 2px dashed #33ff33;
  padding-bottom: 10px;
  margin-bottom: 30px;
  display: flex;
  justify-content: space-between;
}
.blink { animation: blinker 1s linear infinite; }
@keyframes blinker { 50% { opacity: 0; } }
.highlight { background: #33ff33; color: #000; padding: 0 5px; font-weight: bold; }

/* 输入区 */
.prompt-line { margin-bottom: 15px; color: #aaa; }
.grid-form {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
  max-width: 800px;
  margin-bottom: 30px;
}
.form-group { display: flex; flex-direction: column; gap: 5px; }
.form-group label { font-size: 14px; color: #33ff33; opacity: 0.8; }

/* 复古输入框 */
.retro-input {
  background: #000;
  border: 1px solid #33ff33;
  color: #33ff33;
  padding: 8px;
  font-family: 'VT323', monospace;
  font-size: 18px;
  outline: none;
  box-shadow: 2px 2px 0 #1a4d1a;
}
.retro-input:focus { background: #111; box-shadow: 2px 2px 0 #33ff33; }

/* 复古按钮 */
.retro-btn {
  background: #000;
  color: #33ff33;
  border: 2px solid #33ff33;
  padding: 10px 20px;
  font-family: 'VT323', monospace;
  font-size: 20px;
  cursor: pointer;
  margin-top: 24px;
  transition: all 0.1s;
}
.retro-btn:hover:not(:disabled) {
  background: #33ff33;
  color: #000;
}
.retro-btn:active { transform: translate(2px, 2px); }
.retro-btn:disabled { border-color: #555; color: #555; cursor: not-allowed; }

.error-log { color: #ff3333; border: 1px solid #ff3333; padding: 10px; margin-top: 10px; }

/* 表格区域 */
.display-area { border-top: 2px solid #33ff33; padding-top: 20px; }
.area-header { margin-bottom: 15px; letter-spacing: 2px; opacity: 0.8; }

.retro-table { width: 100%; border-collapse: collapse; text-align: left; }
.retro-table th {
  border-bottom: 1px solid #33ff33;
  padding: 10px 5px;
  color: #aaffaa;
}
.retro-table td {
  padding: 8px 5px;
  border-bottom: 1px dashed #1a4d1a;
  color: #ccffcc;
}
.retro-table tr:hover { background: rgba(51, 255, 51, 0.1); }

/* 辅助色 */
.text-green { color: #33ff33; }
.text-red { color: #ff3333; }
.text-gray { color: #777; }
.bold { font-weight: bold; }
.mono-num { letter-spacing: 1px; }
</style>