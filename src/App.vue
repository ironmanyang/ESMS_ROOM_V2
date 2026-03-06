<template>
  <div id="app" class="app-shell">
    <div class="background-glow background-glow-left"></div>
    <div class="background-glow background-glow-right"></div>

    <header class="hero-card">
      <div>
        <p class="hero-tag">ESMS Room</p>
        <h1>{{ roomData.title || '接码室控制台' }}</h1>
        <p class="hero-subtitle">
          实时接收短信、筛选号码状态，并快速复制验证码。
        </p>
      </div>
      <div class="hero-status-group">
        <div class="status-chip" :class="wsConnected ? 'is-online' : 'is-offline'">
          <span class="dot"></span>
          {{ wsConnected ? '实时连接中' : '连接已断开' }}
        </div>
        <div class="hero-time">
          最后更新 {{ lastUpdatedLabel }}
        </div>
      </div>
    </header>

    <section class="toolbar-card">
      <div class="toolbar-left">
        <div class="toolbar-field">
          <label for="room-id-input">房间号</label>
          <el-input
            id="room-id-input"
            v-model.trim="roomIdInput"
            placeholder="请输入房间号，例如 abc123"
            clearable
            @keyup.enter.native="connectRoom"
          />
        </div>
        <div class="toolbar-actions">
          <el-button type="primary" @click="connectRoom">
            {{ wsConnected ? '重新连接' : '连接房间' }}
          </el-button>
          <el-button plain @click="backToAllSms">查看最新短信</el-button>
        </div>
      </div>
      <div class="toolbar-right">
        <div class="metric-card">
          <span class="metric-label">在线号码</span>
          <strong>{{ phoneCount }}</strong>
        </div>
        <div class="metric-card">
          <span class="metric-label">等待短信</span>
          <strong>{{ waitingCount }}</strong>
        </div>
        <div class="metric-card">
          <span class="metric-label">短信总数</span>
          <strong>{{ allSms.length }}</strong>
        </div>
      </div>
    </section>

    <main class="content-grid">
      <section class="panel panel-left">
        <div class="panel-header">
          <div>
            <p class="panel-kicker">号码池</p>
            <h2>在线号码</h2>
          </div>
          <el-button type="primary" plain size="small" @click="fetchPhoneSms('all')">
            全部短信
          </el-button>
        </div>

        <div v-if="roomData.phones && roomData.phones.length" class="phone-list">
          <button
            v-for="phone in roomData.phones"
            :key="phone.phone"
            class="phone-item"
            :class="{ active: selectedPhone === phone.phone }"
            @click="fetchPhoneSms(phone.phone)"
          >
            <div class="phone-item-top">
              <div class="phone-meta">
                <span class="country-badge">{{ getCountryFlag(phone.code_country) }}</span>
                <div>
                  <div class="phone-number">{{ phone.phone }}</div>
                  <div class="phone-country">
                    {{ getCountryLabel(phone.code_country) }}
                  </div>
                </div>
              </div>
              <span class="status-badge" :class="getStatusClass(phone.status)">
                {{ getStatusText(phone.status) }}
              </span>
            </div>
            <div class="phone-item-bottom">
              <span>{{ phone.sms_count || 0 }} 条短信</span>
              <span v-if="phone.time_reg">
                {{ getCountdown(phone.time_reg + roomTimeoutSeconds) }}
              </span>
              <span v-else>等待分配</span>
            </div>
          </button>
        </div>
        <div v-else class="empty-state dark">
          <strong>暂无在线号码</strong>
          <p>连接房间后，这里会显示当前可查看的手机号。</p>
        </div>
      </section>

      <section class="panel panel-right">
        <div class="panel-header">
          <div>
            <p class="panel-kicker">短信流</p>
            <h2>{{ activePanelTitle }}</h2>
          </div>
          <div class="panel-header-actions">
            <span class="sms-counter">共 {{ displaySms.length }} 条</span>
            <el-button
              v-if="selectedPhone"
              type="primary"
              plain
              size="small"
              icon="el-icon-back"
              @click="backToAllSms"
            >
              返回最新
            </el-button>
          </div>
        </div>

        <div class="sms-list-container">
          <div v-if="displaySms.length" class="sms-list">
            <article
              v-for="sms in displaySms"
              :key="`${sms.phone || 'all'}-${sms.time_get}-${sms.message}`"
              class="sms-item"
            >
              <div class="sms-item-header">
                <div class="sms-tags">
                  <span v-if="sms.keyword_matched" class="sms-keyword">
                    {{ sms.keyword_matched }}
                  </span>
                  <span class="sms-phone">{{ sms.phone || selectedPhone || '通道短信' }}</span>
                </div>
                <span class="sms-time">{{ formatTime(sms.time_get) }}</span>
              </div>
              <div class="sms-sender-row">
                <span class="sender-label">发送方</span>
                <span class="sender-value">{{ sms.sender || '未知来源' }}</span>
              </div>
              <div class="sms-content" v-html="processSmsContent(sms.message)"></div>
              <div class="sms-actions">
                <el-button type="primary" size="small" @click="copyToClipboard(sms.message)">
                  复制全文
                </el-button>
                <el-button
                  v-if="extractCode(sms.message)"
                  plain
                  size="small"
                  @click="copyToClipboard(extractCode(sms.message))"
                >
                  复制验证码
                </el-button>
              </div>
            </article>
          </div>
          <div v-else class="empty-state">
            <strong>暂无短信</strong>
            <p>当接收到短信后，会自动出现在这里。</p>
          </div>
        </div>

        <div
          v-if="selectedPhone && phonePagination[selectedPhone] && phonePagination[selectedPhone].totalPages > 1"
          class="pagination-wrap"
        >
          <el-pagination
            background
            layout="prev, pager, next"
            :total="phonePagination[selectedPhone].totalItems"
            :page-size="phonePagination[selectedPhone].pageSize"
            :current-page="phonePagination[selectedPhone].currentPage"
            @current-change="handleCurrentChange"
          />
        </div>
      </section>
    </main>

    <footer class="footer-card">
      <div class="footer-item">
        <span>任务状态</span>
        <strong>{{ roomData.task_status === 1 ? '运行中' : '待连接' }}</strong>
      </div>
      <div class="footer-item">
        <span>创建时间</span>
        <strong>{{ formatEndTime(roomData.created_at) || '暂无' }}</strong>
      </div>
      <div class="footer-item">
        <span>结束时间</span>
        <strong>{{ formatEndTime(roomData.end_time) || '暂无' }}</strong>
      </div>
      <div class="footer-item">
        <span>当前房间</span>
        <strong>{{ roomId || '未连接' }}</strong>
      </div>
    </footer>
  </div>
</template>

<script>
const WS_URL = 'wss://ws.8885.me';

export default {
  name: 'App',
  data() {
    return {
      roomId: '',
      roomIdInput: '',
      roomData: {
        phones: [],
        task_timeout: 0
      },
      allSms: [],
      selectedPhone: '',
      phoneSms: {},
      ws: null,
      wsConnected: false,
      reconnectTimer: null,
      manualClose: false,
      lastUpdatedAt: null,
      phonePagination: {}
    };
  },
  computed: {
    displaySms() {
      if (this.selectedPhone) {
        return this.phoneSms[this.selectedPhone] || [];
      }
      return this.allSms.slice(0, 10);
    },
    phoneCount() {
      return (this.roomData.phones || []).length;
    },
    waitingCount() {
      return (this.roomData.phones || []).filter(phone => Number(phone.status) === 3).length;
    },
    roomTimeoutSeconds() {
      return Number(this.roomData.task_timeout || 0) * 60;
    },
    activePanelTitle() {
      if (this.selectedPhone === 'all') {
        return '全部短信记录';
      }
      return this.selectedPhone || '最新短信';
    },
    lastUpdatedLabel() {
      if (!this.lastUpdatedAt) {
        return '刚刚';
      }
      return this.formatTime(this.lastUpdatedAt, true);
    }
  },
  mounted() {
    this.initRoomId();
    window.copyToClipboard = text => this.copyToClipboard(text);
    if (this.roomId) {
      this.connectRoom();
    }
  },
  beforeDestroy() {
    window.copyToClipboard = () => {};
    this.clearReconnectTimer();
    this.manualClose = true;
    if (this.ws) {
      this.ws.close();
    }
  },
  methods: {
    initRoomId() {
      const path = window.location.pathname.replace(/^\/+|\/+$/g, '');
      const roomIdFromPath = path ? path.split('/')[0] : '';
      const roomIdFromQuery = new URLSearchParams(window.location.search).get('roomId') || '';
      this.roomId = roomIdFromPath || roomIdFromQuery;
      this.roomIdInput = this.roomId;
    },
    connectRoom() {
      const nextRoomId = (this.roomIdInput || '').trim();
      if (!nextRoomId) {
        this.$message.warning('请先输入房间号');
        return;
      }
      this.roomId = nextRoomId;
      this.updateUrl(nextRoomId);
      this.resetRoomState();
      this.initWebSocket(true);
    },
    updateUrl(roomId) {
      const nextUrl = `${window.location.origin}${window.location.pathname}?roomId=${encodeURIComponent(roomId)}`;
      window.history.replaceState({}, '', nextUrl);
    },
    resetRoomState() {
      this.selectedPhone = '';
      this.roomData = {
        phones: [],
        task_timeout: 0
      };
      this.allSms = [];
      this.phoneSms = {};
      this.phonePagination = {};
    },
    initWebSocket(forceReconnect) {
      if (!this.roomId) {
        return;
      }
      this.manualClose = false;
      this.clearReconnectTimer();
      if (this.ws) {
        const previousSocket = this.ws;
        this.manualClose = true;
        previousSocket.onclose = null;
        previousSocket.close();
        this.ws = null;
        this.manualClose = false;
      }
      if (!forceReconnect && this.wsConnected) {
        return;
      }
      this.ws = new WebSocket(WS_URL);
      this.ws.onopen = this.handleWebSocketOpen;
      this.ws.onerror = this.handleWebSocketError;
      this.ws.onmessage = this.handleWebSocketMessage;
      this.ws.onclose = this.handleWebSocketClose;
    },
    handleWebSocketOpen() {
      this.wsConnected = true;
      this.sendLogin();
      this.$message.success('房间连接成功');
    },
    handleWebSocketError() {
      this.wsConnected = false;
    },
    handleWebSocketClose() {
      this.wsConnected = false;
      if (!this.manualClose) {
        this.scheduleReconnect();
      }
    },
    scheduleReconnect() {
      this.clearReconnectTimer();
      this.reconnectTimer = setTimeout(() => {
        if (!this.wsConnected && this.roomId) {
          this.initWebSocket(true);
        }
      }, 3000);
    },
    clearReconnectTimer() {
      if (this.reconnectTimer) {
        clearTimeout(this.reconnectTimer);
        this.reconnectTimer = null;
      }
    },
    sendLogin() {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({
          action: 'login',
          key: this.roomId
        }));
      }
    },
    handleWebSocketMessage(event) {
      try {
        const data = JSON.parse(event.data);
        this.processWebSocketMessage(data);
      } catch (error) {
        console.error('解析 WebSocket 消息失败:', error);
      }
    },
    processWebSocketMessage(data) {
      this.lastUpdatedAt = Math.floor(Date.now() / 1000);

      if (data.ping === 'pang') {
        this.sendSocketPayload({ ping: 'pang' });
        return;
      }

      if (data.type === 'phone_status' && data.data && this.roomData.phones) {
        const phoneIndex = this.roomData.phones.findIndex(item => item.phone === data.data.phone);
        if (phoneIndex !== -1) {
          const nextPhones = [...this.roomData.phones];
          nextPhones.splice(phoneIndex, 1, {
            ...nextPhones[phoneIndex],
            ...data.data
          });
          this.roomData = {
            ...this.roomData,
            phones: nextPhones
          };
        }
        return;
      }

      if (Array.isArray(data.data) && this.selectedPhone) {
        this.$set(this.phoneSms, this.selectedPhone, data.data);
        if (data.total !== undefined && this.phonePagination[this.selectedPhone]) {
          const pageSize = this.phonePagination[this.selectedPhone].pageSize || 15;
          this.$set(this.phonePagination, this.selectedPhone, {
            ...this.phonePagination[this.selectedPhone],
            totalItems: data.total,
            totalPages: Math.ceil(data.total / pageSize)
          });
        }
      }

      if (data.type === 'sms' && data.data) {
        this.updateSms([data.data]);
        return;
      }

      if (Array.isArray(data.sms) || Array.isArray(data.phones)) {
        this.roomData = {
          ...this.roomData,
          ...data,
          phones: Array.isArray(data.phones) ? data.phones : this.roomData.phones
        };
        if (Array.isArray(data.sms) && data.sms.length) {
          this.updateSms(data.sms);
        }
      }
    },
    sendSocketPayload(payload) {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify(payload));
      }
    },
    fetchPhoneSms(phone, page, limit) {
      if (!this.wsConnected) {
        this.$message.warning('请先连接房间');
        return;
      }

      const currentPage = page || 1;
      const pageSize = limit || 15;
      this.selectedPhone = phone;

      this.$set(this.phonePagination, phone, {
        currentPage,
        pageSize,
        totalItems: this.phonePagination[phone] ? this.phonePagination[phone].totalItems : 0,
        totalPages: this.phonePagination[phone] ? this.phonePagination[phone].totalPages : 0
      });

      this.sendSocketPayload({
        action: 'get_sms',
        key: this.roomId,
        phone,
        limit: pageSize,
        page: currentPage
      });
    },
    backToAllSms() {
      this.selectedPhone = '';
    },
    updateSms(newSms) {
      const merged = [];
      const seen = new Set();

      [...newSms, ...this.allSms].forEach(item => {
        const smsKey = `${item.phone || 'all'}-${item.time_get}-${item.message}`;
        if (!seen.has(smsKey)) {
          seen.add(smsKey);
          merged.push(item);
        }
      });

      this.allSms = merged.sort((left, right) => Number(right.time_get) - Number(left.time_get));
    },
    getStatusClass(status) {
      const classes = {
        1: 'status-preparing',
        2: 'status-registering',
        3: 'status-waiting'
      };
      return classes[status] || 'status-unknown';
    },
    getStatusText(status) {
      const texts = {
        1: '准备上卡',
        2: '网络注册',
        3: '等待短信'
      };
      return texts[status] || '未知状态';
    },
    getCountdown(timestamp) {
      const diff = Math.max(Number(timestamp || 0) - Math.floor(Date.now() / 1000), 0);
      if (!diff) {
        return '可换卡';
      }
      const minutes = Math.floor(diff / 60);
      const seconds = diff % 60;
      return `剩余 ${minutes}:${String(seconds).padStart(2, '0')}`;
    },
    getCountryLabel(countryCode) {
      if (!countryCode) {
        return '未知地区';
      }
      return String(countryCode).toUpperCase();
    },
    getCountryFlag(countryCode) {
      const code = this.getCountryLabel(countryCode);
      if (!/^[A-Z]{2}$/.test(code)) {
        return '🌐';
      }
      return code.replace(/./g, char => String.fromCodePoint(127397 + char.charCodeAt(0)));
    },
    formatTime(timestamp, withoutDate) {
      if (!timestamp) {
        return '';
      }
      const date = new Date(Number(timestamp) * 1000);
      const options = withoutDate
        ? { hour: '2-digit', minute: '2-digit', second: '2-digit' }
        : { year: 'numeric', month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', second: '2-digit' };
      return date.toLocaleString('zh-CN', options);
    },
    formatEndTime(timestamp) {
      if (!timestamp) {
        return '';
      }
      if (Number(timestamp) > 2050 * 365 * 24 * 3600) {
        return '永久有效';
      }
      return this.formatTime(timestamp);
    },
    escapeHtml(value) {
      return String(value || '')
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#39;');
    },
    processSmsContent(content) {
      const safeContent = this.escapeHtml(content).replace(/\n/g, '<br>');
      return safeContent.replace(
        /(\d{4,8})/g,
        '<span class="highlighted-number" onclick="window.copyToClipboard(\'$1\')">$1</span>'
      );
    },
    extractCode(content) {
      const match = String(content || '').match(/\b(\d{4,8})\b/);
      return match ? match[1] : '';
    },
    async copyToClipboard(text) {
      try {
        if (!text) {
          return;
        }
        const value = String(text);
        if (navigator.clipboard && navigator.clipboard.writeText) {
          await navigator.clipboard.writeText(value);
        } else {
          const input = document.createElement('textarea');
          input.value = value;
          input.style.position = 'fixed';
          input.style.opacity = '0';
          document.body.appendChild(input);
          input.focus();
          input.select();
          document.execCommand('copy');
          document.body.removeChild(input);
        }
        this.$message.success('复制成功');
      } catch (error) {
        this.$message.error('复制失败');
      }
    },
    handleCurrentChange(page) {
      if (!this.selectedPhone || !this.phonePagination[this.selectedPhone]) {
        return;
      }
      this.fetchPhoneSms(
        this.selectedPhone,
        page,
        this.phonePagination[this.selectedPhone].pageSize
      );
    }
  }
};
</script>

<style>
:root {
  color-scheme: dark;
}

* {
  box-sizing: border-box;
}

html,
body {
  margin: 0;
  min-height: 100%;
  font-family: "Microsoft YaHei", "PingFang SC", Arial, sans-serif;
  background:
    radial-gradient(circle at top left, rgba(78, 132, 255, 0.25), transparent 26%),
    radial-gradient(circle at top right, rgba(0, 219, 198, 0.16), transparent 18%),
    linear-gradient(160deg, #07111f 0%, #0a1630 42%, #050912 100%);
  color: #e8f0ff;
}

body {
  min-height: 100vh;
}

button,
input,
textarea {
  font: inherit;
}

#app {
  min-height: 100vh;
}

.app-shell {
  position: relative;
  max-width: 100%;
  padding: 28px;
  overflow: hidden;
}

.background-glow {
  position: absolute;
  width: 380px;
  height: 380px;
  border-radius: 50%;
  filter: blur(80px);
  opacity: 0.28;
  pointer-events: none;
}

.background-glow-left {
  top: -120px;
  left: -120px;
  background: #2e6bff;
}

.background-glow-right {
  right: -120px;
  bottom: 10%;
  background: #18d5bf;
}

.hero-card,
.toolbar-card,
.panel,
.footer-card {
  position: relative;
  z-index: 1;
  backdrop-filter: blur(20px);
  background: rgba(8, 20, 39, 0.76);
  border: 1px solid rgba(118, 160, 255, 0.18);
  box-shadow: 0 24px 60px rgba(0, 0, 0, 0.28);
}

.hero-card {
  display: flex;
  justify-content: space-between;
  gap: 24px;
  padding: 28px 32px;
  border-radius: 26px;
  margin-bottom: 20px;
}

.hero-tag,
.panel-kicker {
  margin: 0 0 10px;
  letter-spacing: 0.24em;
  font-size: 12px;
  text-transform: uppercase;
  color: #70b8ff;
}

.hero-card h1,
.panel-header h2 {
  margin: 0;
}

.hero-card h1 {
  font-size: 34px;
  font-weight: 700;
}

.hero-subtitle {
  max-width: 620px;
  margin: 12px 0 0;
  color: #92a9d6;
  line-height: 1.7;
}

.hero-status-group {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  justify-content: space-between;
  min-width: 180px;
}

.status-chip {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  border-radius: 999px;
  padding: 10px 16px;
  font-size: 13px;
  font-weight: 600;
}

.status-chip .dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.status-chip.is-online {
  color: #7ef0c7;
  background: rgba(33, 191, 115, 0.14);
  border: 1px solid rgba(33, 191, 115, 0.32);
}

.status-chip.is-online .dot {
  background: #35e0a1;
  box-shadow: 0 0 12px #35e0a1;
}

.status-chip.is-offline {
  color: #ffb3c1;
  background: rgba(255, 74, 110, 0.12);
  border: 1px solid rgba(255, 74, 110, 0.28);
}

.status-chip.is-offline .dot {
  background: #ff607e;
}

.hero-time {
  color: #9ab0db;
  font-size: 13px;
}

.toolbar-card {
  display: flex;
  justify-content: space-between;
  gap: 20px;
  padding: 20px 24px;
  border-radius: 22px;
  margin-bottom: 20px;
}

.toolbar-left {
  flex: 1;
  display: flex;
  align-items: end;
  gap: 16px;
}

.toolbar-field {
  flex: 1;
  max-width: 420px;
}

.toolbar-field label {
  display: block;
  margin-bottom: 8px;
  font-size: 13px;
  color: #94a7cc;
}

.toolbar-actions {
  display: flex;
  gap: 12px;
}

.toolbar-right {
  display: grid;
  grid-template-columns: repeat(3, minmax(110px, 1fr));
  gap: 12px;
}

.metric-card {
  min-width: 110px;
  padding: 14px 16px;
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(130, 158, 224, 0.15);
}

.metric-label {
  display: block;
  color: #8fa3cb;
  font-size: 12px;
  margin-bottom: 6px;
}

.metric-card strong {
  font-size: 28px;
  font-weight: 700;
  color: #fff;
}

.content-grid {
  display: grid;
  grid-template-columns: 360px minmax(0, 1fr);
  gap: 20px;
  min-height: calc(100vh - 310px);
}

.panel {
  border-radius: 26px;
  padding: 24px;
}

.panel-left,
.panel-right {
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 16px;
  margin-bottom: 18px;
}

.panel-header h2 {
  font-size: 24px;
}

.panel-header-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.sms-counter {
  color: #8ea6d8;
  font-size: 13px;
}

.phone-list,
.sms-list-container {
  min-height: 0;
}

.phone-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  overflow-y: auto;
  padding-right: 4px;
}

.phone-item {
  border: 1px solid rgba(126, 157, 222, 0.16);
  background: linear-gradient(180deg, rgba(14, 31, 58, 0.96), rgba(11, 24, 46, 0.96));
  color: #fff;
  border-radius: 18px;
  padding: 16px;
  cursor: pointer;
  transition: transform 0.2s ease, border-color 0.2s ease, box-shadow 0.2s ease;
}

.phone-item:hover,
.phone-item.active {
  transform: translateY(-2px);
  border-color: rgba(82, 147, 255, 0.65);
  box-shadow: 0 16px 34px rgba(12, 28, 55, 0.5);
}

.phone-item-top,
.phone-item-bottom,
.phone-meta,
.sms-item-header,
.sms-tags,
.sms-sender-row,
.sms-actions,
.footer-card {
  display: flex;
  align-items: center;
}

.phone-item-top,
.phone-item-bottom,
.sms-item-header,
.sms-sender-row,
.footer-card {
  justify-content: space-between;
}

.phone-meta {
  gap: 12px;
}

.country-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 42px;
  height: 42px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.08);
  font-size: 20px;
}

.phone-number {
  font-size: 17px;
  font-weight: 700;
  margin-bottom: 4px;
}

.phone-country,
.phone-item-bottom {
  color: #89a2cf;
  font-size: 13px;
}

.phone-item-bottom {
  margin-top: 14px;
}

.status-badge {
  padding: 7px 12px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 700;
}

.status-preparing {
  background: rgba(255, 184, 0, 0.16);
  color: #ffd56a;
}

.status-registering {
  background: rgba(138, 99, 255, 0.18);
  color: #c9b7ff;
}

.status-waiting {
  background: rgba(39, 203, 140, 0.16);
  color: #8ef0c0;
}

.status-unknown {
  background: rgba(255, 255, 255, 0.1);
  color: #dce7ff;
}

.sms-list-container {
  flex: 1;
  overflow-y: auto;
  padding-right: 6px;
}

.sms-list {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.sms-item {
  border-radius: 20px;
  padding: 20px;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.96), rgba(241, 246, 255, 0.96));
  color: #14243f;
  border: 1px solid rgba(72, 127, 255, 0.12);
  box-shadow: 0 12px 26px rgba(7, 24, 55, 0.08);
}

.sms-tags {
  gap: 10px;
  flex-wrap: wrap;
}

.sms-keyword,
.sms-phone {
  display: inline-flex;
  align-items: center;
  height: 28px;
  padding: 0 10px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 700;
}

.sms-keyword {
  color: #206dff;
  background: rgba(32, 109, 255, 0.1);
}

.sms-phone {
  color: #495d86;
  background: rgba(73, 93, 134, 0.08);
}

.sms-time,
.sender-label {
  color: #6d7ea4;
  font-size: 13px;
}

.sms-sender-row {
  margin: 14px 0 12px;
}

.sender-value {
  font-size: 14px;
  font-weight: 600;
  color: #2d3f61;
}

.sms-content {
  line-height: 1.75;
  color: #223656;
  font-size: 15px;
}

.highlighted-number {
  display: inline-block;
  margin: 0 4px;
  padding: 2px 8px;
  border-radius: 10px;
  font-size: 24px;
  font-weight: 700;
  color: #0b5cff;
  background: rgba(11, 92, 255, 0.1);
  cursor: pointer;
}

.sms-actions {
  gap: 10px;
  margin-top: 16px;
}

.empty-state {
  flex: 1;
  min-height: 260px;
  border-radius: 20px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  padding: 24px;
  background: rgba(255, 255, 255, 0.05);
  color: #9eb1d6;
}

.empty-state.dark {
  background: rgba(255, 255, 255, 0.04);
}

.empty-state strong {
  color: #fff;
  margin-bottom: 10px;
  font-size: 18px;
}

.empty-state p {
  margin: 0;
  max-width: 320px;
  line-height: 1.7;
}

.pagination-wrap {
  margin-top: 20px;
  display: flex;
  justify-content: flex-end;
}

.footer-card {
  justify-content: space-between;
  gap: 16px;
  margin-top: 20px;
  padding: 18px 24px;
  border-radius: 22px;
  flex-wrap: wrap;
}

.footer-item span {
  display: block;
  color: #8da3cc;
  font-size: 12px;
  margin-bottom: 6px;
}

.footer-item strong {
  font-size: 15px;
  color: #fff;
}

.el-input__inner {
  height: 44px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(128, 154, 216, 0.18);
  color: #fff;
}

.el-input__inner::placeholder {
  color: #7288b2;
}

.el-button {
  border-radius: 14px;
}

.el-button--primary {
  background: linear-gradient(135deg, #2d7dff, #15d1c3);
  border-color: transparent;
}

.el-button--primary.is-plain {
  color: #cfe2ff;
  background: rgba(53, 122, 255, 0.08);
  border-color: rgba(83, 146, 255, 0.28);
}

.el-pagination.is-background .el-pager li,
.el-pagination.is-background .btn-next,
.el-pagination.is-background .btn-prev {
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
}

.el-pagination.is-background .el-pager li:not(.disabled).active {
  background: linear-gradient(135deg, #2d7dff, #15d1c3);
}

@media (max-width: 1200px) {
  .toolbar-card,
  .hero-card,
  .footer-card {
    flex-direction: column;
    align-items: stretch;
  }

  .toolbar-left,
  .toolbar-right {
    width: 100%;
  }

  .hero-status-group {
    align-items: flex-start;
  }
}

@media (max-width: 960px) {
  .app-shell {
    padding: 18px;
  }

  .content-grid {
    grid-template-columns: 1fr;
  }

  .panel-header,
  .phone-item-top,
  .phone-item-bottom,
  .sms-item-header,
  .sms-sender-row {
    flex-direction: column;
    align-items: flex-start;
  }

  .panel-header-actions,
  .toolbar-left,
  .toolbar-actions,
  .toolbar-right,
  .sms-actions {
    width: 100%;
  }

  .toolbar-left,
  .toolbar-actions {
    flex-direction: column;
    align-items: stretch;
  }

  .toolbar-right {
    grid-template-columns: 1fr;
  }

  .sms-actions .el-button {
    width: 100%;
  }
}
</style>
