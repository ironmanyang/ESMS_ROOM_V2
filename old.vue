<template>
  <div id="app">
    <div class="header">
      <h1>{{ roomData.title || '加载中...' }}</h1>
    </div>
    <div class="container">
      <div class="left-panel">
        <div class="panel-header">
          <h2>在线号码</h2>
          <el-button type="primary" size="small" @click="fetchPhoneSms('all')">
            全部信息
          </el-button>
        </div>
        <div v-if="roomData.phones && roomData.phones.length > 0" class="phone-list">
          <div v-for="phone in roomData.phones" :key="phone.phone" class="phone-item"
            @click="fetchPhoneSms(phone.phone)">
            <div class="phone-info">
              <img :src="`./assets/png/${phone.code_country.toLowerCase()}.png`" alt="" class="flag-image">
              <span class="phone-number">{{ phone.phone }}</span>
            </div>
            <div class="phone-status">
              <span class="status-badge" :class="getStatusClass(phone.status)">
                {{ getStatusText(phone.status) }}
              </span>
              <div v-if="phone.time_reg" class="countdown">
                {{ '倒计时：' + getCountdown(phone.time_reg + roomData.task_timeout * 60) }}
              </div>
            </div>
          </div>
        </div>
        <div v-else class="empty-state">
          暂无在线号码
        </div>
      </div>
      <div class="right-panel">
        <div class="panel-header">
          <h2>{{ selectedPhone ? selectedPhone : '最新短信' }}</h2>
          <el-button v-if="selectedPhone" @click="backToAllSms" type="primary" size="small" icon="el-icon-back">
            返回
          </el-button>
        </div>
        <div class="sms-list-container">
          <div v-if="displaySms.length > 0" class="sms-list">
            <div v-for="sms in displaySms" :key="sms.time_get + sms.message" class="sms-item">
              <div class="sms-header">
                <div class="header-left">
                  <div class="sms-keyword">
                    {{ sms.keyword_matched }}
                  </div>
                  <span class="sms-sender">{{ sms.sender }}</span>
                </div>
                <div class="header-right">
                  <span class="sms-time">{{ formatTime(sms.time_get) }}</span>

                </div>
              </div>
              <div class="sms-content" v-html="processSmsContent(sms.message)">
              </div>
              <div class="sms-btn-box">
                <el-button type="primary" size="small" @click="copyToClipboard(sms.message)">
                  复制全部内容
                </el-button>
              </div>
            </div>
          </div>
          <div v-else class="empty-state">
            暂无短信
          </div>

        </div>
        <!-- 分页组件 -->
        <div v-if="selectedPhone && phonePagination[selectedPhone] && phonePagination[selectedPhone].totalPages > 0"
          class="pagination">
          <el-pagination background layout="prev, pager, next" :total="phonePagination[selectedPhone].totalItems"
            :page-size="phonePagination[selectedPhone].pageSize"
            :current-page="phonePagination[selectedPhone].currentPage" @current-change="handleCurrentChange" />
        </div>
      </div>

    </div>
    <div class="footer">
      <p>接码室状态: {{ roomData.task_status === 1 ? '运行中' : '已停止' }}</p>
      <p>创建时间: {{ formatTime(roomData.created_at) }}</p>
      <p>结束时间: {{ formatEndTime(roomData.end_time) }}</p>
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      roomId: '',
      roomData: {},
      allSms: [],
      selectedPhone: '',
      phoneSms: {},
      ws: null,
      wsConnected: false,
      // 分页相关属性
      pagination: {
        currentPage: 1,
        pageSize: 15,
        totalItems: 0,
        totalPages: 0
      },
      // 为每个手机号存储独立的分页状态
      phonePagination: {}
    };
  },
  computed: {
    displaySms() {
      if (this.selectedPhone) {
        return this.phoneSms[this.selectedPhone] || [];
      }
      return this.allSms.slice(0, 10);
    }
  },
  mounted() {
    this.getRoomIdFromUrl();
    if (this.roomId) {
      this.initWebSocket();
    }
    // 将 copyToClipboard 方法绑定到 window 对象上，以便 onclick 事件访问
    window.copyToClipboard = this.copyToClipboard;
  },
  beforeDestroy() {
    if (this.ws) {
      this.ws.close();
    }
  },
  methods: {
    initWebSocket() {
      const wsUrl = 'wss://ws.8885.me';
      console.log('正在建立WebSocket连接:', wsUrl);
      this.ws = new WebSocket(wsUrl);

      this.ws.onopen = this.handleWebSocketOpen;
      this.ws.onerror = this.handleWebSocketError;
      this.ws.onmessage = this.handleWebSocketMessage;
      this.ws.onclose = this.handleWebSocketClose;
    },
    handleWebSocketOpen() {
      console.log('WebSocket连接已建立');
      this.wsConnected = true;
      this.sendLogin();

    },
    handleWebSocketError(error) {
      console.error('WebSocket错误:', error);
      this.wsConnected = false;
      // 添加重连逻辑
      // this.reconnectWebSocket();
    },
    async handleWebSocketMessage(event) {
      console.log('收到WebSocket消息:', event.data);
      try {
        const data = JSON.parse(event.data);
        this.processWebSocketMessage(data);
        console.log('data', data);

      } catch (error) {
        console.error('解析WebSocket消息失败:', error);
      }
    },
    handleWebSocketClose(event) {
      console.log('WebSocket连接已关闭:', event.code, event.reason);
      this.wsConnected = false;
      // 添加重连逻辑
      this.reconnectWebSocket();

    },
    reconnectWebSocket() {
      console.log('尝试重新连接WebSocket...');
      setTimeout(() => {
        if (!this.wsConnected) {
          this.initWebSocket();
        }
      }, 3000);
    },
    sendLogin() {
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        const loginData = {
          action: 'login',
          key: this.roomId
        };
        this.ws.send(JSON.stringify(loginData));
        console.log('发送登录请求:', loginData);
      }
    },
    processWebSocketMessage(data) {
      console.log('处理WebSocket消息:', data);
      if (data.ping == 'pang') {
        // 心跳包
        this.ws.send(JSON.stringify({ ping: 'pang' }));
        return;
      }
      if (this.selectedPhone) {
        this.$set(this.phoneSms, this.selectedPhone, data.data || []);

        // 更新分页状态
        if (data.total !== undefined) {
          this.phonePagination[this.selectedPhone].totalItems = data.total;
          this.phonePagination[this.selectedPhone].totalPages = Math.ceil(data.total / this.phonePagination[this.selectedPhone].currentPage);
        }

      } else {
        // 初始化数据
        this.roomData = data
        if (data.sms && data.sms.length > 0) {
          this.updateSms(data.sms);
        }
        console.log('初始化数据处理完成:', this.roomData);
      }

      // 这里处理初始化数据
      if (data.type === 'init') {
        console.log('初始化数据类型');
      } else if (data.type === 'sms') {
        // 新短信推送
        if (data.data) {
          this.updateSms([data.data]);
        }
      } else if (data.type === 'phone_status') {
        // 手机号状态更新
        if (data.data && this.roomData.phones) {
          const phoneIndex = this.roomData.phones.findIndex(p => p.phone === data.data.phone);
          if (phoneIndex !== -1) {
            this.roomData.phones[phoneIndex] = { ...this.roomData.phones[phoneIndex], ...data.data };
          }
        }
      }
    },
    getRoomIdFromUrl() {
      const path = window.location.pathname;
      const parts = path.split('/').filter(Boolean);
      this.roomId = parts[0] || '';
    },

    fetchPhoneSms(phone, page = 1, limit = 15) {
      this.selectedPhone = phone;

      // 为当前手机号初始化分页状态
      if (!this.phonePagination[phone]) {
        this.$set(this.phonePagination, phone, {
          currentPage: page,
          pageSize: limit,
          totalItems: 0,
          totalPages: 0
        });
      } else {
        this.phonePagination[phone].currentPage = page;
        this.phonePagination[phone].pageSize = limit;
      }

      let params = {
        "action": "get_sms",
        "key": this.roomId,
        "phone": phone,
        "limit": limit,
        "page": page
      }
      this.ws.send(JSON.stringify(params));

    },
    backToAllSms() {
      this.selectedPhone = '';
    },

    updateSms(newSms) {
      const uniqueSms = [];
      const seen = new Set();

      [...newSms, ...this.allSms].forEach(sms => {
        const key = `${sms.phone}-${sms.time_get}-${sms.message}`;
        if (!seen.has(key)) {
          seen.add(key);
          uniqueSms.push(sms);
        }
      });

      this.allSms = uniqueSms.sort((a, b) => b.time_get - a.time_get);
    },
    getStatusClass(status) {
      const classes = {
        1: 'status-preparing',
        2: 'status-registering',
        3: 'status-waiting'
      };
      return classes[status] || '';
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
      const now = Math.floor(Date.now() / 1000);
      const diff = timestamp - now;

      if (diff <= 0) {
        return '可换卡';
      }

      const minutes = Math.floor(diff / 60);
      const seconds = diff % 60;
      return `${minutes}:${seconds.toString().padStart(2, '0')}`;
    },
    formatTime(timestamp) {
      if (!timestamp) return '';
      const date = new Date(timestamp * 1000);
      return date.toLocaleString('zh-CN');
    },
    formatEndTime(timestamp) {
      if (!timestamp) return '';
      if (timestamp > 2050 * 365 * 24 * 3600) {
        return '永久有效';
      }
      return this.formatTime(timestamp);
    },
    processSmsContent(content) {
      if (!content) return '';
      // 使用正则表达式匹配连续的 4-8 位数字
      const regex = /(\d{4,8})/g;
      return content.replace(regex, '<span class="highlighted-number" onclick="window.copyToClipboard(\$1)">$1</span>');
    },
    copyToClipboard(text) {
      navigator.clipboard.writeText(text)
        .then(() => {
          // 可以添加一个复制成功的提示
          console.log('复制成功:', text);
          this.$message({
            message: '复制成功',
            type: 'success'
          });
        })
        .catch(err => {
          console.error('复制失败:', err);
          this.$message({
            message: '复制失败',
            type: 'error'
          });
        });
    },
    prevPage(phone) {
      const pagination = this.phonePagination[phone];
      if (pagination && pagination.currentPage > 1) {
        this.fetchPhoneSms(phone, pagination.currentPage - 1, pagination.pageSize);
      }
    },
    nextPage(phone) {
      const pagination = this.phonePagination[phone];
      if (pagination && pagination.currentPage < pagination.totalPages) {
        this.fetchPhoneSms(phone, pagination.currentPage + 1, pagination.pageSize);
      }
    },
    handleCurrentChange(page) {
      if (this.selectedPhone) {
        const pagination = this.phonePagination[this.selectedPhone];
        if (pagination) {
          this.fetchPhoneSms(this.selectedPhone, page, pagination.pageSize);
        }
      }
    },

  }
};
</script>

<style lang="scss">
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: Arial, sans-serif;
  background-color: #f5f5f5;
  color: #333;
}

#app {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;

  .header {
    background-color: #2c3e50;
    color: white;
    padding: 20px;
    border-radius: 0 0 8px 8px;
    margin-bottom: 20px;
    text-align: center;

    h1 {
      font-size: 24px;
      font-weight: bold;
      line-height: 1;
    }
  }

  .container {
    display: flex;
    height: calc(100vh - 84px - 65px);
    gap: 20px;
    margin-bottom: 20px;

    .left-panel {
      flex: 0 0 300px;
      background-color: #1a1a2e;
      border-radius: 0 8px 8px 0;
      padding: 20px;
      box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
      display: flex;
      flex-direction: column;
      overflow: hidden;
      height: 100%;

      .panel-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        flex-shrink: 0;
        margin-bottom: 15px;
        border-bottom: 2px solid #3498db;
        padding-bottom: 10px;

        h2 {
          font-size: 18px;
          color: white;
        }
      }

      .phone-list {
        display: flex;
        flex-direction: column;
        gap: 10px;
        overflow-y: auto;
        overflow-x: hidden;

        .phone-item {
          background-color: #16213e;
          border-radius: 6px;
          padding: 15px;
          cursor: pointer;
          transition: all 0.3s ease;
          border-left: 4px solid #3498db;

          &:hover {
            background-color: #0f3460;
            transform: translateX(5px);
          }

          .phone-info {
            display: flex;
            align-items: center;
            gap: 10px;
            margin-bottom: 10px;

            .flag-image {
              width: 20px;
              height: 20px;
              border-radius: 50%;
              vertical-align: middle;
            }

            .phone-number {
              font-weight: bold;
              flex: 1;
              color: white;
            }

            .sms-count {
              background-color: #3498db;
              color: white;
              padding: 2px 8px;
              border-radius: 10px;
              font-size: 12px;
            }
          }

          .phone-status {
            display: flex;
            justify-content: space-between;
            align-items: center;

            .status-badge {
              padding: 2px 8px;
              border-radius: 10px;
              font-size: 12px;
              font-weight: bold;

              &.status-preparing {
                background-color: #f39c12;
                color: white;
              }

              &.status-registering {
                background-color: #9b59b6;
                color: white;
              }

              &.status-waiting {
                background-color: #27ae60;
                color: white;
              }
            }

            .countdown {
              font-size: 12px;
              color: #95a5a6;
            }
          }
        }
      }

      .empty-state {
        text-align: center;
        color: #95a5a6;
        padding: 40px 20px;
        background-color: #16213e;
        border-radius: 6px;
      }
    }

    .right-panel {
      flex: 1;
      background-color: white;
      border-radius: 8px 0 0 8px;
      padding: 20px;
      box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
      display: flex;
      flex-direction: column;
      overflow: hidden;

      .panel-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 15px;
        border-bottom: 2px solid #3498db;
        padding-bottom: 10px;

        h2 {
          font-size: 18px;
          color: #333;
          margin: 0;
        }


      }

      .sms-list-container {
        flex: 1;
        overflow-y: auto;
        margin-bottom: 20px;

        .sms-list {
          display: flex;
          flex-direction: column;
          gap: 15px;

          .sms-item {
            background-color: #f9f9f9;
            border-radius: 6px;
            padding: 15px;
            border-left: 4px solid #3498db;

            .sms-header {
              display: flex;
              justify-content: space-between;
              margin-bottom: 10px;
              font-size: 14px;
              color: #666;

              .header-left {
                flex: 1;
                display: flex;
                justify-content: flex-start;
                flex-direction: row;
                align-items: center;

                .sms-keyword {
                  padding: 3px 10px;
                  box-sizing: border-box;
                  border-radius: 5px;
                  border: 1px solid #3498db;
                  font-size: 12px;
                  color: #3498db;
                  font-weight: bold;
                }

                .sms-sender {
                  margin-left: 10px;
                  font-weight: bold;
                }
              }

              .header-right {
                flex-shrink: 0;
              }

            }

            .sms-content {
              margin-bottom: 10px;
              line-height: 1.5;
            }



            .highlighted-number {
              color: #3498db;
              font-size: 26px;
              font-weight: bold;
              cursor: pointer;
              text-decoration: underline;
              text-decoration-style: dotted;
              text-decoration-color: #3498db;
              padding: 0 2px;
              border-radius: 2px;
              transition: background-color 0.2s ease;

              &:hover {
                background-color: rgba(52, 152, 219, 0.1);
              }
            }
          }
        }

        .empty-state {
          text-align: center;
          color: #999;
          padding: 40px 20px;
          background-color: #f9f9f9;
          border-radius: 6px;
        }
      }



    }

  }

  .footer {
    background-color: #0f3460;
    padding: 15px;
    border-radius: 8px 8px 0 0;
    font-size: 15px;
    line-height: 1;
    color: #fff;
    display: flex;
    justify-content: space-around;
    flex-wrap: wrap;
    gap: 10px;
  }
}
</style>
