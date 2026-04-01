<template>
  <div id="app" class="app-shell">
    <header class="hero-card">
      <div class="refresh-box">
        <div class="refresh-button" role="button" tabindex="0" @click="refreshLatestSms"
          @keydown.enter="refreshLatestSms" @keydown.space.prevent="refreshLatestSms">
          <i class="el-icon-refresh"></i>
        </div>
      </div>
      <div class="name-box">
        <p class="hero-tag">ESMS</p>
        <h1>{{ roomData.title || '接码室控制台' }}</h1>
      </div>
      <div class="hero-status-group">
        <div :class="wsConnected ? 'is-online' : 'is-offline'" class="status-chip">
          <span class="dot"></span>
          {{ wsConnected ? '实时连接中' : '连接已断开' }}
        </div>
        <div class="hero-time-box">
          <div class="hero-time">
            创建时间：{{ formatEndTime(roomData.created_at) || '暂无' }}
          </div>
          <div class="hero-time">
            结束时间：{{ formatEndTime(roomData.end_time) || '暂无' }}
          </div>
          <div class="hero-time">
            最后更新时间：{{ lastUpdatedLabel }}
          </div>
        </div>

      </div>
    </header>

    <main class="content-grid">
      <section class="panel panel-left">
        <div class="panel-header" style="justify-content: space-between;">
          <div>
            <!-- <p class="panel-kicker">号码池</p> -->
            <h2>在线号码</h2>
          </div>
          <el-button plain size="small" type="primary" @click="fetchPhoneSms('all')">
            全部短信
          </el-button>
        </div>
        <div class="phone-search-box">
          <el-input v-model="searchPhone" placeholder="搜索手机号" clearable></el-input>
        </div>

        <div v-if="filteredPhones && filteredPhones.length" class="phone-list">
          <button v-for="phone in filteredPhones" :key="phone.phone" :class="{ active: selectedPhone === phone.phone }"
            class="phone-item" @click="fetchPhoneSms(phone.phone)">
            <div class="phone-item-top">
              <div class="phone-meta">
                <img :src="getCountryFlagImage(phone.code_country)" alt="" class="flag-image">
                <div>
                  <div class="phone-number">
                    <span>{{ phone.phone }}</span>
                    <div class="copy-box" @click.stop="copyToClipboard(phone.phone)">
                      <i class="el-icon-copy-document"></i>
                    </div>
                  </div>
                  <div class="phone-country">
                    {{ getCountryLabel(phone.code_country) }}
                  </div>
                </div>
              </div>
              <div class="phone-status">
                <span :class="getStatusClass(phone.status)" class="status-badge">
                  {{ getStatusText(phone.status) }}
                </span>
                <span class="status-time" v-if="phone.time_reg">
                  {{ getCountdown(phone.time_reg) }}
                </span>
              </div>
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
          <div class="panel-header-actions">
            <el-button v-if="selectedPhone" icon="el-icon-back" plain size="small" type="primary" @click="backToAllSms">
              返回最新
            </el-button>
          </div>
          <div>
            <!-- <p class="panel-kicker">短信流</p> -->
            <div class="panel-box">
              <h2>{{ activePanelTitle }}</h2>
              <div class="copy-box" v-if="!activePanelTitle.includes('短信')" @click="copyToClipboard(activePanelTitle)">
                <i class="el-icon-copy-document"></i>
              </div>
            </div>
          </div>

        </div>

        <div class="sms-list-container">
          <div v-if="displaySms.length" class="sms-list">
            <article v-for="sms in displaySms" :key="`${sms.phone || 'all'}-${sms.time_get}-${sms.message}`"
              class="sms-item">
              <div class="sms-item-header">
                <div class="sms-tags">
                  <span v-if="sms.keyword_matched" class="sms-keyword">
                    {{ sms.keyword_matched }}
                  </span>
                  <div class="sms-phone" v-show="sms.phone" @click="copyToClipboard(sms.phone)">
                    <span>{{ sms.phone }}</span>
                    <i class="el-icon-copy-document"></i>
                  </div>
                </div>
                <div class="sender-value">
                  <span class="" @click="copyToClipboard(sms.sender)">来自：『{{ sms.sender || '未知来源' }}』</span>
                </div>
                <div class="sms-actions-btn" @click="copyToClipboard(sms.message)">
                  复制全文
                </div>
                <span class="sms-time">{{ formatTime(sms.time_get) }}</span>
              </div>
              <div class="sms-content-box">
                <div class="sms-content" v-html="processSmsContent(sms.message)"></div>
              </div>

            </article>
          </div>
          <div v-else class="empty-state">
            <strong>暂无短信</strong>
            <p>当接收到短信后，会自动出现在这里。</p>
          </div>
        </div>

        <div v-if="selectedPhone && phonePagination[selectedPhone] && phonePagination[selectedPhone].totalPages > 0"
          class="pagination-wrap">
          <el-pagination :current-page="phonePagination[selectedPhone].currentPage"
            :page-size="phonePagination[selectedPhone].pageSize" :total="phonePagination[selectedPhone].totalItems"
            background layout="prev, pager, next" @current-change="handleCurrentChange" />
        </div>
      </section>
    </main>

  </div>
</template>

<script>
const WS_URL = window.APP_CONFIG?.WS_URL

export default {
  name: 'App',
  data() {
    return {
      roomId: '',
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
      phonePagination: {},
      searchPhone: '',
      pagination: {
        currentPage: 1,
        pageSize: 15,
        totalItems: 0,
        totalPages: 0
      },
    };
  },
  computed: {
    filteredPhones() {
      if (!this.searchPhone) {
        return this.roomData.phones || [];
      }
      return this.roomData.phones.filter(phone =>
        String(phone.phone).includes(this.searchPhone)
      );
    },
    displaySms() {
      if (this.selectedPhone) {
        return this.phoneSms[this.selectedPhone] || [];
      }
      return this.allSms.slice(0, 10);
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
        return '暂无';
      }
      return this.formatTime(this.lastUpdatedAt, true);
    }
  },
  mounted() {
    this.initRoomId();

    window.copyToClipboard = text => this.copyToClipboard(text);
    if (this.roomId) {
      this.resetRoomState();
      this.initWebSocket(true);
    }
  },
  beforeDestroy() {
    window.copyToClipboard = () => {
    };
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
      // console.log('收到消息:', data.sms);

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
    refreshLatestSms() {
      this.backToAllSms();
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
      let now = new Date().getTime();
      let diff = this.roomData.task_timeout - ((now - timestamp * 1000) / 1000);
      console.log(diff);
      // diff转为时分秒
      const hours = Math.floor(diff / 3600);
      const minutes = Math.floor((diff % 3600) / 60);
      const seconds = Math.floor(diff % 60);
      console.log(hours, minutes, seconds);
      if (hours > 0) {
        return `超时时间：${hours}时${minutes}分${seconds}秒`;
      } else if (minutes > 0) {
        return `超时时间：${minutes}分${seconds}秒`;
      }
    },
    getCountryLabel(countryCode) {
      if (!countryCode) {
        return '未知地区';
      }
      return String(countryCode).toUpperCase();
    },
    getCountryFlagImage(countryCode) {
      const code = String(countryCode || '').toLowerCase();
      if (!code) {
        return '';
      }
      return require(`@/assets/png/${code}.png`);
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
      if (Number(timestamp) >= 4070880000) {
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

<style lang="scss">
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
  background: #0B0E14;
  color: #E2E8F0;
}

body {
  min-height: 100vh;
}

button,
input,
textarea {
  font: inherit;
}

::-webkit-scrollbar {
  width: 10px;
  background-color: #f1f1f1;
  border-radius: 5px;
}

::-webkit-scrollbar-thumb {
  background-color: #777;
  border-radius: 5px;
}

::-webkit-scrollbar-thumb:hover {
  background-color: #555;
}

// #app {
//   min-height: 100vh;
// }

.app-shell {
  position: relative;
  width: 100%;
  height: 100vh;
  box-sizing: border-box;
  padding: 15px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  flex-wrap: nowrap;
  justify-content: flex-start;
  gap: 15px;


  .hero-card,
  .panel {
    position: relative;
    z-index: 1;
    backdrop-filter: blur(20px);
    background: rgba(28, 34, 45, 0.9);
    border: 1px solid rgba(255, 255, 255, 0.1);
    box-shadow: 0 24px 60px rgba(0, 0, 0, 0.20);
  }

  .hero-card {
    display: flex;
    justify-content: flex-start;
    gap: 15px;
    box-sizing: border-box;
    padding: 15px;
    border-radius: 20px;
    flex-shrink: 0;
    align-items: center;

    .name-box {
      flex: 1;
    }

    h1 {
      margin: 0;
      font-size: 34px;
      font-weight: 700;
    }

    .hero-tag {
      margin: 0 0 10px;
      letter-spacing: 0.24em;
      font-size: 12px;
      text-transform: uppercase;
      color: #38BDF8;
    }


    .refresh-box {
      display: flex;
      justify-content: flex-end;

      .refresh-button {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        width: 60px;
        height: 60px;
        box-sizing: border-box;
        padding: 0 20px;
        border-radius: 14px;
        color: #E2E8F0;
        background: rgba(45, 212, 191, 0.15);
        border: 1px solid rgba(45, 212, 191, 0.3);
        cursor: pointer;
        user-select: none;
        transition: background 0.2s ease, border-color 0.2s ease, transform 0.2s ease;
        font-size: 30px;

        &:hover {
          background: rgba(45, 212, 191, 0.25);
          border-color: rgba(45, 212, 191, 0.5);
          transform: translateY(-1px);
        }

        &:focus {
          outline: none;
          box-shadow: 0 0 0 3px rgba(45, 212, 191, 0.2);
        }
      }
    }

    .hero-status-group {
      display: flex;
      flex-direction: column;
      align-items: flex-end;
      justify-content: space-between;
      min-width: 180px;

      .status-chip {
        display: inline-flex;
        align-items: center;
        gap: 8px;
        border-radius: 999px;
        box-sizing: border-box;
        padding: 10px 16px;
        font-size: 13px;
        font-weight: 600;

        .dot {
          width: 8px;
          height: 8px;
          border-radius: 50%;
        }

        &.is-online {
          color: #2DD4BF;
          background: rgba(45, 212, 191, 0.15);
          border: 1px solid rgba(45, 212, 191, 0.3);

          .dot {
            background: #2DD4BF;
            box-shadow: 0 0 12px #2DD4BF;
          }
        }

        &.is-offline {
          color: #F87171;
          background: rgba(248, 113, 113, 0.15);
          border: 1px solid rgba(248, 113, 113, 0.3);

          .dot {
            background: #F87171;
          }
        }
      }

      .hero-time-box {
        display: flex;
        margin-top: 10px;

        .hero-time {
          color: #94A3B8;
          font-size: 13px;
          padding: 0 20px;
          border-right: 1px dashed rgba(148, 163, 184, 0.3);

          &:last-child {
            border-right: none;
          }
        }
      }

    }
  }

  .content-grid {
    display: grid;
    flex: 1;
    grid-template-columns: 360px minmax(0, 1fr);
    gap: 15px;
    min-height: calc(100vh - 150px);
  }

  .panel {
    border-radius: 20px;
    box-sizing: border-box;
    padding: 15px;

    &.panel-left {
      background: rgba(21, 25, 33, 0.9);
    }

    &.panel-left,
    &.panel-right {
      display: flex;
      flex-direction: column;
      min-height: 0;
    }

    .panel-header {
      display: flex;
      justify-content: flex-start;
      align-items: flex-start;
      gap: 20px;
      margin-bottom: 18px;

      h2 {
        margin: 0;
        font-size: 24px;
      }

      .panel-kicker {
        margin: 0 0 10px;
        letter-spacing: 0.24em;
        font-size: 12px;
        text-transform: uppercase;
        color: #38BDF8;
      }

      .panel-box {
        display: flex;
        gap: 8px;
        flex-direction: row;
        justify-content: flex-start;
        align-items: center;
      }

      .panel-header-actions {
        display: flex;
        align-items: center;
        gap: 12px;
      }
    }
  }

  .phone-list,
  .sms-list-container {
    min-height: 0;
  }

  .phone-search-box {
    margin-bottom: 15px;

    .el-input {
      .el-input__inner {
        background: rgba(28, 34, 45, 0.9);
        border: 1px solid rgba(255, 255, 255, 0.1);
        border-radius: 14px;
        color: #E2E8F0;
        font-size: 14px;
        height: 40px;
        line-height: 40px;
        padding: 0 15px;
        transition: border-color 0.2s ease, box-shadow 0.2s ease;

        &::placeholder {
          color: #94A3B8;
        }

        &:focus {
          border-color: rgba(45, 212, 191, 0.5);
          box-shadow: 0 0 0 3px rgba(45, 212, 191, 0.1);
          outline: none;
        }
      }

      .el-input__icon {
        color: #94A3B8;
        font-size: 16px;

        &.el-icon-circle-close {
          color: #94A3B8;

          &:hover {
            color: #E2E8F0;
          }
        }
      }
    }
  }

  .phone-list {
    padding: 4px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    overflow-y: auto;
    box-sizing: border-box;

    .phone-item {
      border: 1px solid rgba(255, 255, 255, 0.1);
      background: rgba(28, 34, 45, 0.9);
      color: #E2E8F0;
      border-radius: 15px;
      box-sizing: border-box;
      padding: 10px;
      cursor: pointer;
      transition: transform 0.2s ease, border-color 0.2s ease, box-shadow 0.2s ease;

      &:hover,
      &.active {
        transform: translateY(-2px);
        border-color: rgba(45, 212, 191, 0.5);
        box-shadow: 0 16px 34px rgba(0, 0, 0, 0.4);
      }

      .phone-item-top {
        display: flex;
        align-items: center;
        justify-content: space-between;
      }

      .phone-meta {
        display: flex;
        align-items: center;
        gap: 12px;


        .flag-image {
          width: 42px;
          height: 42px;
          border-radius: 14px;
          object-fit: cover;
          background: rgba(255, 255, 255, 0.08);
          border: 1px solid rgba(255, 255, 255, 0.12);
          flex-shrink: 0;
        }

        .phone-number {
          font-size: 17px;
          font-weight: 700;
          margin-bottom: 4px;
          display: flex;
          gap: 8px;
          flex-direction: row;
          justify-content: flex-start;
          align-items: center;
        }

        .phone-country {
          color: #94A3B8;
          font-size: 13px;
          text-align: left;
        }
      }

      .phone-status {
        display: flex;
        flex-direction: column;
        align-items: flex-end;
        gap: 8px;


        .status-badge {
          box-sizing: border-box;
          padding: 7px 12px;
          border-radius: 999px;
          font-size: 12px;
          font-weight: 700;

          &.status-preparing {
            background: rgba(251, 191, 36, 0.15);
            color: #FBBF24;
          }

          &.status-registering {
            background: rgba(167, 139, 250, 0.15);
            color: #A78BFA;
          }

          &.status-waiting {
            background: rgba(45, 212, 191, 0.15);
            color: #2DD4BF;
          }

          &.status-unknown {
            background: rgba(255, 255, 255, 0.1);
            color: #94A3B8;
          }
        }

        .status-time {
          line-height: 1;
          color: #89a2cf;
          font-size: 13px;
        }
      }
    }
  }

  .sms-list-container {
    flex: 1;
    overflow-y: auto;
    box-sizing: border-box;
    padding-right: 6px;

    .sms-list {
      display: flex;
      flex-direction: column;
      gap: 14px;

      .sms-item {
        border-radius: 20px;
        box-sizing: border-box;
        padding: 10px;
        background: #1C222D;
        color: #E2E8F0;
        border: 1px solid rgba(255, 255, 255, 0.1);
        box-shadow: 0 12px 26px rgba(0, 0, 0, 0.3);

        .sms-item-header {
          display: flex;
          align-items: center;
          justify-content: flex-start;
          gap: 20px;

          .sms-tags {
            display: flex;
            align-items: center;
            gap: 10px;
            flex-wrap: wrap;
            flex-shrink: 0;
            min-width: 200px;

            .sms-keyword,
            .sms-phone {
              display: inline-flex;
              align-items: center;
              height: 20px;
              box-sizing: border-box;
              padding: 0 10px;
              border-radius: 10px;
              font-size: 12px;
              font-weight: 700;
              cursor: pointer;
            }

            .sms-keyword {
              color: #38BDF8;
              background: rgba(56, 189, 248, 0.15);
            }

            .sms-phone {
              color: #94A3B8;
              background: rgba(148, 163, 184, 0.15);
              gap: 4px;
            }


          }

          .sender-value {
            font-size: 14px;
            font-weight: 600;
            color: #CBD5E1;
            flex: 1;
            cursor: pointer;
          }

          .sms-time {
            color: #64748B;
            font-size: 13px;
          }

          .sms-actions-btn {
            min-width: 80px;
            height: 30px;
            line-height: 26px;
            border-radius: 15px;
            font-size: 12px;
            font-weight: 500;
            text-align: center;
            border: 1px solid #0b3631;
            cursor: pointer;
            color: #6ae3f8;
            background: rgba(56, 189, 248, 0.15);
            // background: linear-gradient(135deg, #0b3631, #0d2b38);
          }
        }

        .sms-content {
          margin-top: 10px;
          line-height: 1.75;
          color: #E2E8F0;
          font-size: 15px;

          .highlighted-number {
            display: inline-block;
            margin: 0 4px;
            box-sizing: border-box;
            padding: 2px 8px;
            border-radius: 10px;
            font-size: 16px;
            font-weight: 700;
            color: #38BDF8;
            background: rgba(56, 189, 248, 0.15);
            cursor: pointer;
          }
        }

      }
    }
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
    box-sizing: border-box;
    padding: 20px;
    background: rgba(255, 255, 255, 0.03);
    color: #94A3B8;

    &.dark {
      background: rgba(255, 255, 255, 0.02);
    }

    strong {
      color: #E2E8F0;
      margin-bottom: 10px;
      font-size: 18px;
    }

    p {
      margin: 0;
      max-width: 320px;
      line-height: 1.7;
    }
  }

  .pagination-wrap {
    margin-top: 20px;
    display: flex;
    justify-content: flex-end;
  }

}

.el-button {
  border-radius: 14px;
}

.el-button--primary {
  background: linear-gradient(135deg, #2DD4BF, #38BDF8);
  border-color: transparent;

  &.is-plain {
    color: #E2E8F0;
    background: rgba(45, 212, 191, 0.15);
    border-color: rgba(45, 212, 191, 0.3);
  }
}

.el-pagination.is-background {

  .el-pager li,
  .btn-next,
  .btn-prev {
    background: rgba(255, 255, 255, 0.08);
    color: #E2E8F0;
  }

  .el-pager li:not(.disabled).active {
    background: linear-gradient(135deg, #2DD4BF, #38BDF8);
  }
}
</style>
