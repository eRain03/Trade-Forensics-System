<template>
  <div class="retro-terminal">
    <div class="status-bar">
      <div class="status-left">
        <span class="sys-msg">SYSTEM: <span class="blink">ONLINE</span></span>
        <span class="sys-msg">STORAGE: <span :class="storageClass">{{ storageSize }}KB</span></span>
      </div>
      <div class="status-right">
        <span class="sys-time">{{ currentTime }}</span>
      </div>
    </div>

    <div class="input-section">
      <div class="prompt-line">
        <span class="prompt">> ACCESS_CONTROL_PROTOCOL_</span>
        <button v-if="orders.length > 0" @click="clearStorage" class="text-btn danger">[ PURGE_DATA ]</button>
      </div>
      
      <div class="grid-form">
        <div class="form-group">
          <label>API KEY</label>
          <input v-model="config.apiKey" type="text" class="retro-input" placeholder="API Key" />
        </div>
        <div class="form-group">
          <label>SECRET KEY</label>
          <input v-model="config.secretKey" type="password" class="retro-input" placeholder="Secret Key" />
        </div>
        <div class="form-group">
          <label>PASSPHRASE</label>
          <input v-model="config.passphrase" type="password" class="retro-input" placeholder="Passphrase" />
        </div>

        <div class="action-group">
          <button @click="fetchHistory" class="retro-btn" :disabled="loading">
            {{ loading ? 'DOWNLOADING...' : 'INITIATE API SYNC' }}
          </button>

          <div class="upload-wrapper">
            <input type="file" ref="fileInput" @change="handleFileUpload" accept=".csv" class="hidden-input" />
            <button @click="$refs.fileInput.click()" class="retro-btn secondary">
              UPLOAD ARCHIVE (.CSV)
            </button>
          </div>
        </div>
      </div>
      
      <div v-if="sysMsg" class="sys-log" :class="{ error: isError }">
        > {{ sysMsg }}
      </div>
    </div>

    <div class="display-area" v-if="orders.length > 0">
      <div class="area-header">
        <span>=== DATA_MANIFEST: {{ orders.length }} RECORDS ===</span>
        <span class="range-info">RANGE: {{ dateRange }}</span>
      </div>

      <div class="table-container">
        <table class="retro-table">
          <thead>
            <tr>
              <th>TIME</th>
              <th>SOURCE</th>
              <th>SYMBOL</th>
              <th>SIDE</th>
              <th>TYPE</th>
              <th>AVG.PX</th>
              <th>SIZE</th>
              <th>PnL</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="order in sortedOrders" :key="order.ordId">
              <td class="col-time">{{ formatTime(order.uTime) }}</td>
              <td class="col-src">
                <span class="tag" :class="order.source === 'API' ? 'tag-api' : 'tag-csv'">{{ order.source }}</span>
              </td>
              <td class="col-symbol">{{ order.instId }}</td>
              <td :class="getSideClass(order.posSide)">{{ order.posSide.toUpperCase() }}</td>
              <td>{{ (order.ordType || 'UNKNOWN').toUpperCase() }}</td>
              <td class="mono-num">{{ order.avgPx }}</td>
              <td class="mono-num">{{ order.accFillSz }}</td>
              <td :class="getPnlClass(order.pnl)">{{ order.pnl }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, computed, watch } from 'vue';
import CryptoJS from 'crypto-js';
import demoData from './assets/demo-data.json';

const STORAGE_KEY = 'TFS_VAULT_V1';

// --- State ---
const currentTime = ref('');
const loading = ref(false);
const sysMsg = ref('SYSTEM READY. WAITING FOR INPUT.');
const isError = ref(false);
const orders = ref([]); // 核心数据源
const storageSize = ref(0);

const config = reactive({ apiKey: '', secretKey: '', passphrase: '' });

// --- Computed ---
// 按时间倒序排列
const sortedOrders = computed(() => {
  return [...orders.value].sort((a, b) => parseInt(b.uTime) - parseInt(a.uTime));
});

const dateRange = computed(() => {
  if (orders.value.length === 0) return 'N/A';
  const times = orders.value.map(o => parseInt(o.uTime));
  const min = new Date(Math.min(...times)).toLocaleDateString();
  const max = new Date(Math.max(...times)).toLocaleDateString();
  return `${min} -> ${max}`;
});

const storageClass = computed(() => storageSize.value > 500 ? 'text-red' : 'text-green');

// --- 1. Persistence Layer (持久化) ---
const loadVault = () => {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (saved) {
    try {
      // 简单 Base64 解码，防窥探
      const raw = atob(saved);
      orders.value = JSON.parse(raw);
      updateStorageSize();
      sysMsg.value = `RESTORED ${orders.value.length} RECORDS FROM LOCAL VAULT.`;
    } catch (e) {
      console.error("Vault Corrupted", e);
      sysMsg.value = "WARNING: LOCAL VAULT CORRUPTED. RESETTING.";
    }
  }
};

const saveVault = () => {
  const raw = JSON.stringify(orders.value);
  const encoded = btoa(raw); // 简单 Base64 编码
  localStorage.setItem(STORAGE_KEY, encoded);
  updateStorageSize();
};

const clearStorage = () => {
  if(confirm('CONFIRM: PURGE ALL LOCAL DATA? THIS CANNOT BE UNDONE.')) {
    localStorage.removeItem(STORAGE_KEY);
    orders.value = [];
    sysMsg.value = "MEMORY WIPED CLEAN.";
    updateStorageSize();
  }
};

const updateStorageSize = () => {
  const saved = localStorage.getItem(STORAGE_KEY);
  storageSize.value = saved ? (saved.length / 1024).toFixed(1) : 0;
};

// 监听数据变化自动保存
watch(orders, () => saveVault(), { deep: true });

// --- 2. Merge Engine (去重合并) ---
const mergeOrders = (newBatch) => {
  const existingMap = new Map(orders.value.map(o => [o.ordId, o]));
  let addedCount = 0;

  newBatch.forEach(item => {
    if (!existingMap.has(item.ordId)) {
      orders.value.push(item);
      addedCount++;
    } else {
      // 可选：如果已存在，更新数据（比如 API 数据比 CSV 更详细）
      // 这里我们选择保留现有数据，或基于来源优先级覆盖
    }
  });

  sysMsg.value = `MERGE COMPLETE. +${addedCount} NEW RECORDS ADDED.`;
};

// --- 3. API Logic ---
const signRequest = (timestamp, method, requestPath, body = '') => {
  const message = timestamp + method + requestPath + body;
  const hmac = CryptoJS.HmacSHA256(message, config.secretKey);
  return CryptoJS.enc.Base64.stringify(hmac);
};

const fetchHistory = async () => {
  loading.value = true;
  sysMsg.value = 'ESTABLISHING SECURE CONNECTION...';
  isError.value = false;

  if (!config.apiKey) {
    // Demo Mode
    setTimeout(() => {
      // 标记来源为 DEMO
      const taggedDemo = demoData.map(o => ({...o, source: 'DEMO'}));
      mergeOrders(taggedDemo);
      loading.value = false;
    }, 1000);
    return;
  }

  try {
    const method = 'GET';
    const path = '/api/v5/trade/orders-history-archive?instType=SWAP';
    const timestamp = new Date().toISOString();
    const signature = signRequest(timestamp, method, path);
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
    if (data.code !== '0') throw new Error(`${data.msg} (Code: ${data.code})`);

    // 标记来源为 API
    const taggedData = data.data.map(o => ({...o, source: 'API'}));
    mergeOrders(taggedData);

  } catch (err) {
    isError.value = true;
    sysMsg.value = `CONNECTION FAILURE: ${err.message}`;
  } finally {
    loading.value = false;
  }
};

// --- 4. CSV Parser Logic ---
const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  sysMsg.value = `PARSING ARCHIVE: ${file.name}...`;
  const reader = new FileReader();

  reader.onload = (e) => {
    const text = e.target.result;
    const lines = text.split('\n');

    // OKX CSV 通常有前几行元数据，我们需要找到包含 "Order ID" 或 "ordId" 的表头行
    let headerIndex = -1;
    let headers = [];

    for(let i=0; i<lines.length; i++) {
      if(lines[i].includes('Order ID') || lines[i].includes('ordId') || lines[i].includes('订单ID')) {
        headerIndex = i;
        headers = lines[i].split(',').map(h => h.trim().replace(/"/g, ''));
        break;
      }
    }

    if (headerIndex === -1) {
      isError.value = true;
      sysMsg.value = 'PARSE ERROR: INVALID CSV FORMAT (HEADER NOT FOUND)';
      return;
    }

    const parsedBatch = [];

    // 从表头下一行开始解析
    for(let i = headerIndex + 1; i < lines.length; i++) {
      const line = lines[i].trim();
      if(!line) continue;

      // 处理 CSV 引号内逗号的情况 (简单版 regex)
      // 如果你的 CSV 比较复杂，建议用 papaparse，这里为了无依赖手写了一个简单的
      const values = line.split(/,(?=(?:(?:[^"]*"){2})*[^"]*$)/).map(v => v.trim().replace(/"/g, ''));

      if(values.length !== headers.length) continue;

      const row = {};
      headers.forEach((h, index) => row[h] = values[index]);

      // --- 关键：Mapping ---
      // 将 CSV 的列名映射到我们内部统一的 key (instId, uTime, pnl 等)
      // 这里需要适配 OKX 导出的列名 (英文版 export 为例)
      // 如果是中文表头，你需要添加中文映射
      try {
        const orderObj = {
          ordId: row['Order ID'] || row['订单ID'],
          instId: row['Instrument'] || row['合约'],
          posSide: (row['Side'] || row['方向'] || '').toLowerCase(), // buy/sell or long/short
          ordType: row['Order Type'] || row['订单类型'],
          avgPx: row['Avg. Fill Price'] || row['成交均价'],
          accFillSz: row['Filled'] || row['成交数量'],
          pnl: row['P&L'] || row['收益'] || '0',
          uTime: new Date(row['Time'] || row['创建时间']).getTime().toString(), // 转为时间戳
          source: 'CSV'
        };

        // 只有解析成功的才加进去
        if(orderObj.ordId && orderObj.instId) {
          parsedBatch.push(orderObj);
        }
      } catch (err) {
        console.warn('Skipped row', i, err);
      }
    }

    mergeOrders(parsedBatch);
  };

  reader.readAsText(file);
  // 重置 input 以允许再次上传同名文件
  event.target.value = '';
};

// --- Helpers ---
const formatTime = (ts) => new Date(parseInt(ts)).toLocaleString('en-GB', { hour12: false });
const getSideClass = (side) => {
  const s = side.toLowerCase();
  if (s.includes('buy') || s.includes('long')) return 'text-green';
  if (s.includes('sell') || s.includes('short')) return 'text-red';
  return '';
};
const getPnlClass = (pnl) => {
  const val = parseFloat(pnl);
  return val > 0 ? 'text-green bold' : val < 0 ? 'text-red bold' : 'text-gray';
};

onMounted(() => {
  setInterval(() => {
    currentTime.value = new Date().toISOString().replace('T', ' ').slice(0, 19);
  }, 1000);
  loadVault(); // 启动时加载
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

.status-bar { border-bottom: 2px solid #33ff33; padding-bottom: 5px; margin-bottom: 30px; display: flex; justify-content: space-between; align-items: flex-end;}
.blink { animation: blinker 1s steps(2, start) infinite; }
@keyframes blinker { 50% { opacity: 0; } }

.input-section { margin-bottom: 40px; }
.prompt-line { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; color: #aaa; border-bottom: 1px dashed #333; padding-bottom: 5px;}
.text-btn.danger { background: none; border: none; color: #ff3333; cursor: pointer; font-family: inherit; font-size: inherit; }
.text-btn.danger:hover { background: #ff3333; color: #000; }

.grid-form { display: grid; grid-template-columns: 1fr 1fr 1fr auto; gap: 15px; align-items: end; }
.form-group { display: flex; flex-direction: column; gap: 4px; }
.form-group label { font-size: 14px; opacity: 0.7; }
.retro-input { background: #000; border: 1px solid #335533; color: #33ff33; padding: 8px; font-family: inherit; font-size: 18px; outline: none; width: 100%; box-sizing: border-box;}
.retro-input:focus { border-color: #33ff33; box-shadow: 0 0 5px #33ff33; }

.action-group { display: flex; gap: 10px; }
.retro-btn { background: #33ff33; color: #000; border: 1px solid #33ff33; padding: 8px 15px; font-family: inherit; font-size: 18px; cursor: pointer; font-weight: bold; height: 42px; white-space: nowrap;}
.retro-btn:hover:not(:disabled) { background: #ccffcc; }
.retro-btn:disabled { background: #1a331a; color: #555; border-color: #1a331a; cursor: not-allowed; }
.retro-btn.secondary { background: transparent; color: #33ff33; border: 1px solid #33ff33; }
.retro-btn.secondary:hover { background: rgba(51, 255, 51, 0.2); }

.hidden-input { display: none; }

.sys-log { margin-top: 15px; padding: 10px; border-left: 4px solid #33ff33; background: #001100; font-family: 'Courier New', Courier, monospace; letter-spacing: -0.5px;}
.sys-log.error { border-color: #ff3333; color: #ff3333; background: #110000; }

.area-header { display: flex; justify-content: space-between; margin-bottom: 10px; border-bottom: 1px dashed #33ff33; padding-bottom: 5px; opacity: 0.8;}

.table-container { overflow-x: auto; }
.retro-table { width: 100%; border-collapse: collapse; text-align: left; font-size: 16px; }
.retro-table th { border-bottom: 2px solid #33ff33; padding: 8px; color: #ccffcc; white-space: nowrap; }
.retro-table td { padding: 8px; border-bottom: 1px solid #1a331a; white-space: nowrap;}
.retro-table tr:hover { background: rgba(51, 255, 51, 0.1); }

.tag { font-size: 12px; padding: 1px 4px; border: 1px solid #555; }
.tag-api { border-color: #33ff33; color: #33ff33; }
.tag-csv { border-color: #ffff33; color: #ffff33; }

.text-green { color: #33ff33; }
.text-red { color: #ff3333; }
.text-gray { color: #777; }
.mono-num { font-family: 'Consolas', monospace; }
</style>