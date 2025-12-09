<template>
  <div class="retro-terminal">
    <div class="status-bar">
      <span class="sys-msg">SYSTEM: <span class="blink">ONLINE</span></span>
      <span class="sys-msg">MODE: <span class="highlight">{{ isDemo ? 'DEMO_SIMULATION' : 'LIVE_ACCESS' }}</span></span>
      <span class="sys-time">{{ currentTime }}</span>
    </div>

    <div class="input-section">
      <div class="prompt-line">
        <span class="prompt">> AUTHENTICATION_PROTOCOL_</span>
        <div class="tip">(Leave empty for DEMO MODE)</div>
      </div>
      
      <div class="grid-form">
        <div class="form-group">
          <label>API KEY</label>
          <input v-model="config.apiKey" type="text" class="retro-input" placeholder="Enter Key..." />
        </div>
        <div class="form-group">
          <label>SECRET KEY</label>
          <input v-model="config.secretKey" type="password" class="retro-input" placeholder="Enter Secret..." />
        </div>
        <div class="form-group">
          <label>PASSPHRASE</label>
          <input v-model="config.passphrase" type="password" class="retro-input" placeholder="Enter Passphrase..." />
        </div>
        <button @click="fetchHistory" class="retro-btn" :disabled="loading">
          {{ loading ? 'PROCESSING...' : 'EXECUTE RETRIEVAL' }}
        </button>
      </div>
      
      <div v-if="sysMsg" class="sys-log" :class="{ error: isError }">
        > {{ sysMsg }}
      </div>
    </div>

    <div class="display-area" v-if="orders.length > 0">
      <div class="area-header">=== TRADE LOGS // SECURITY LEVEL: DECLASSIFIED ===</div>
      <table class="retro-table">
        <thead>
          <tr>
            <th>TIME</th>
            <th>SYMBOL</th>
            <th>SIDE</th>
            <th>TYPE</th>
            <th>AVG.PX</th>
            <th>SIZE</th>
            <th>PnL</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="order in orders" :key="order.ordId">
            <td class="col-time">{{ formatTime(order.uTime) }}</td>
            <td class="col-symbol">{{ order.instId }}</td>
            <td :class="getSideClass(order.posSide)">{{ order.posSide.toUpperCase() }}</td>
            <td>{{ order.ordType.toUpperCase() }}</td>
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
import demoData from './assets/demo-data.json'; // 导入本地演示数据

const currentTime = ref('');
const loading = ref(false);
const sysMsg = ref('');
const isError = ref(false);
const isDemo = ref(false);
const orders = ref([]);

const config = reactive({ apiKey: '', secretKey: '', passphrase: '' });

// --- 签名算法 ---
const signRequest = (timestamp, method, requestPath, body = '') => {
  const message = timestamp + method + requestPath + body;
  const hmac = CryptoJS.HmacSHA256(message, config.secretKey);
  return CryptoJS.enc.Base64.stringify(hmac);
};

const fetchHistory = async () => {
  loading.value = true;
  sysMsg.value = '';
  isError.value = false;
  orders.value = [];

  // --- 演示模式判断 ---
  if (!config.apiKey) {
    setTimeout(() => {
      orders.value = demoData;
      isDemo.value = true;
      sysMsg.value = 'ACCESS GRANTED via SIMULATION NODE. DISPLAYING CACHED DATA.';
      loading.value = false;
    }, 1200); // 假装加载了一会儿
    return;
  }

  // --- 真实模式 ---
  isDemo.value = false;
  try {
    const method = 'GET';
    // 拉取过去3个月的永续合约订单
    const path = '/api/v5/trade/orders-history-archive?instType=SWAP'; 
    const timestamp = new Date().toISOString();
    const signature = signRequest(timestamp, method, path);

    // 发送到 /api/okx (会被 vite 或 vercel 转发)
    const proxyUrl = `/api/okx${path.replace('/api', '')}`; 

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
      throw new Error(`SERVER_REJECT: ${data.msg} (CODE: ${data.code})`);
    }

    orders.value = data.data;
    sysMsg.value = `SUCCESS. ${data.data.length} RECORDS RETRIEVED.`;

  } catch (err) {
    console.error(err);
    isError.value = true;
    sysMsg.value = `CRITICAL FAILURE: ${err.message}`;
  } finally {
    loading.value = false;
  }
};

const formatTime = (ts) => new Date(parseInt(ts)).toLocaleString('en-GB', { hour12: false });
const getSideClass = (side) => (side === 'long' ? 'text-green' : side === 'short' ? 'text-red' : '');
const getPnlClass = (pnl) => parseFloat(pnl) > 0 ? 'text-green bold' : parseFloat(pnl) < 0 ? 'text-red bold' : 'text-gray';

onMounted(() => {
  setInterval(() => {
    currentTime.value = new Date().toISOString().replace('T', ' ').slice(0, 19);
  }, 1000);
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=VT323&display=swap');

.retro-terminal {
  background-color: #050505;
  color: #33ff33;
  font-family: 'VT323', monospace;
  font-size: 18px;
  min-height: 100vh;
  padding: 20px;
  box-sizing: border-box;
  text-transform: uppercase;
}

.status-bar { border-bottom: 1px dashed #33ff33; padding-bottom: 10px; margin-bottom: 30px; display: flex; justify-content: space-between; }
.blink { animation: blinker 1s linear infinite; }
@keyframes blinker { 50% { opacity: 0; } }
.highlight { background: #33ff33; color: #000; padding: 0 4px; font-weight: bold; }
.tip { font-size: 14px; color: #555; margin-left: 10px; text-transform: none; display: inline-block;}

.grid-form { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 15px; max-width: 800px; }
.form-group { display: flex; flex-direction: column; gap: 4px; }
.form-group label { font-size: 14px; opacity: 0.7; }

.retro-input { background: #111; border: 1px solid #335533; color: #33ff33; padding: 8px; font-family: inherit; font-size: 18px; outline: none; }
.retro-input:focus { border-color: #33ff33; background: #000; }

.retro-btn { background: #000; color: #33ff33; border: 2px solid #33ff33; padding: 10px 20px; font-family: inherit; font-size: 20px; cursor: pointer; margin-top: 20px; }
.retro-btn:hover:not(:disabled) { background: #33ff33; color: #000; }
.retro-btn:disabled { border-color: #555; color: #555; cursor: not-allowed; }

.sys-log { margin-top: 15px; padding: 10px; border-left: 3px solid #33ff33; background: #0a1a0a; }
.sys-log.error { border-color: #ff3333; color: #ff3333; }

.display-area { border-top: 2px solid #33ff33; margin-top: 40px; padding-top: 20px; }
.area-header { margin-bottom: 15px; opacity: 0.8; letter-spacing: 1px; }

.retro-table { width: 100%; border-collapse: collapse; text-align: left; font-size: 16px; }
.retro-table th { border-bottom: 1px solid #33ff33; padding: 8px; color: #ccffcc; }
.retro-table td { padding: 8px; border-bottom: 1px solid #1a331a; }
.retro-table tr:hover { background: rgba(51, 255, 51, 0.05); }

.text-green { color: #33ff33; }
.text-red { color: #ff3333; }
.text-gray { color: #777; }
.bold { font-weight: bold; }
</style>