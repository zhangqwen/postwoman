<template>
  <div class="page">
    <section id="options">
      <input id="tab-one" type="radio" name="options" checked="checked" />
      <label for="tab-one">WebSocket</label>
      <div class="tab">
        <pw-section class="blue" label="请求" ref="request">
          <ul>
            <li>
              <label for="url">URL</label>
              <input
                id="url"
                type="url"
                :class="{ error: !urlValid }"
                v-model="url"
                @keyup.enter="urlValid ? toggleConnection() : null"
              />
            </li>
            <div>
              <li>
                <label for="connect" class="hide-on-small-screen">&nbsp;</label>
                <button
                  :disabled="!urlValid"
                  id="connect"
                  name="connect"
                  @click="toggleConnection"
                >
                  {{ toggleConnectionVerb }}
                  <span>
                    <i class="material-icons" v-if="!connectionState">sync</i>
                    <i class="material-icons" v-if="connectionState"
                      >sync_disabled</i
                    >
                  </span>
                </button>
              </li>
            </div>
          </ul>
        </pw-section>
        <pw-section
          class="purple"
          label="连接"
          id="response"
          ref="response"
        >
          <ul>
            <li>
              <label for="log">日志</label>
              <div id="log" name="log" class="log">
                <span v-if="communication.log">
                  <span
                    v-for="(logEntry, index) in communication.log"
                    :style="{ color: logEntry.color }"
                    :key="index"
                    >@ {{ logEntry.ts }}{{ getSourcePrefix(logEntry.source)
                    }}{{ logEntry.payload }}</span
                  >
                </span>
                <span v-else>(等待连接)</span>
              </div>
            </li>
          </ul>
          <ul>
            <li>
              <label for="message">消息</label>
              <input
                id="message"
                name="message"
                type="text"
                v-model="communication.input"
                :readonly="!connectionState"
                @keyup.enter="connectionState ? sendMessage() : null"
              />
            </li>
            <div>
              <li>
                <label for="send" class="hide-on-small-screen">&nbsp;</label>
                <button
                  id="send"
                  name="send"
                  :disabled="!connectionState"
                  @click="sendMessage"
                >
                  发送
                  <span>
                    <i class="material-icons">send</i>
                  </span>
                </button>
              </li>
            </div>
          </ul>
        </pw-section>
      </div>
      <input id="tab-two" type="radio" name="options" />
      <label for="tab-two">SSE</label>
      <div class="tab">
        <pw-section class="blue" label="请求" ref="request">
          <ul>
            <li>
              <label for="server">服务器</label>
              <input
                id="server"
                type="url"
                :class="{ error: !serverValid }"
                v-model="server"
                @keyup.enter="serverValid ? toggleSSEConnection() : null"
              />
            </li>
            <div>
              <li>
                <label for="start" class="hide-on-small-screen">&nbsp;</label>
                <button
                  :disabled="!serverValid"
                  id="start"
                  name="start"
                  @click="toggleSSEConnection"
                >
                  {{ toggleSSEConnectionVerb }}
                  <span>
                    <i class="material-icons" v-if="!connectionSSEState"
                      >sync</i
                    >
                    <i class="material-icons" v-if="connectionSSEState"
                      >sync_disabled</i
                    >
                  </span>
                </button>
              </li>
            </div>
          </ul>
        </pw-section>
        <pw-section
          class="purple"
          label="链接"
          id="response"
          ref="response"
        >
          <ul>
            <li>
              <label for="log">事件</label>
              <div id="log" name="log" class="log">
                <span v-if="events.log">
                  <span
                    v-for="(logEntry, index) in events.log"
                    :style="{ color: logEntry.color }"
                    :key="index"
                    >@ {{ logEntry.ts }}{{ getSourcePrefix(logEntry.source)
                    }}{{ logEntry.payload }}</span
                  >
                </span>
                <span v-else>(等待连接)</span>
              </div>
              <div id="result"></div>
            </li>
          </ul>
        </pw-section>
      </div>
    </section>
  </div>
</template>
<style lang="scss">
div.log {
  margin: 4px;
  padding: 8px 16px;
  width: calc(100% - 8px);
  border-radius: 8px;
  background-color: var(--bg-dark-color);
  color: var(--fg-color);
  height: 256px;
  overflow: auto;

  &,
  span {
    font-size: 16px;
    font-family: "Roboto Mono", monospace;
    font-weight: 400;
  }

  span {
    display: block;
    white-space: pre-wrap;
    word-wrap: break-word;
    word-break: break-all;
  }
}
</style>

<script>
export default {
  components: {
    "pw-section": () => import("../components/section")
  },
  data() {
    return {
      connectionState: false,
      url: "wss://echo.websocket.org",
      socket: null,
      communication: {
        log: null,
        input: ""
      },
      connectionSSEState: false,
      server: "https://express-eventsource.herokuapp.com/events",
      sse: null,
      events: {
        log: null,
        input: ""
      }
    };
  },
  computed: {
    toggleConnectionVerb() {
      return !this.connectionState ? "连接" : "断开连接";
    },
    urlValid() {
      const pattern = new RegExp(
        "^(wss?:\\/\\/)?" +
          "((([a-z\\d]([a-z\\d-]*[a-z\\d])*)\\.)+[a-z]{2,}|" +
          "((\\d{1,3}\\.){3}\\d{1,3}))" +
          "(\\:\\d+)?(\\/[-a-z\\d%_.~+@]*)*" +
          "(\\?[;&a-z\\d%_.~+=-]*)?" +
          "(\\#[-a-z\\d_]*)?$",
        "i"
      );
      return pattern.test(this.url);
    },
    toggleSSEConnectionVerb() {
      return !this.connectionSSEState ? "开始" : "停止";
    },
    serverValid() {
      const pattern = new RegExp(
        "^(http(s)?:\\/\\/)?" +
          "((([a-z\\d]([a-z\\d-]*[a-z\\d])*)\\.)+[a-z]{2,}|" +
          "((\\d{1,3}\\.){3}\\d{1,3}))" +
          "(\\:\\d+)?(\\/[-a-z\\d%_.~+@]*)*" +
          "(\\?[:\\;&a-z\\d%_.~+=-]*)?" +
          "(\\#[-a-z\\d_]*)?$",
        "i"
      );
      return pattern.test(this.server);
    }
  },
  methods: {
    toggleConnection() {
      // If it is connecting:
      if (!this.connectionState) return this.connect();
      // Otherwise, it's disconnecting.
      else return this.disconnect();
    },
    connect() {
      this.communication.log = [
        {
          payload: `正在连接到 ${this.url}...`,
          source: "info",
          color: "var(--ac-color)"
        }
      ];
      try {
        this.socket = new WebSocket(this.url);
        this.socket.onopen = event => {
          this.connectionState = true;
          this.communication.log = [
            {
              payload: `连接到 ${this.url}.`,
              source: "info",
              color: "var(--ac-color)",
              ts: new Date().toLocaleTimeString()
            }
          ];
          this.$toast.success("连接完成", {
            icon: "sync"
          });
        };
        this.socket.onerror = event => {
          this.handleError();
        };
        this.socket.onclose = event => {
          this.connectionState = false;
          this.communication.log.push({
            payload: `断开链接 ${this.url}.`,
            source: "info",
            color: "#ff5555",
            ts: new Date().toLocaleTimeString()
          });
          this.$toast.error("断开连接", {
            icon: "sync_disabled"
          });
        };
        this.socket.onmessage = event => {
          this.communication.log.push({
            payload: event.data,
            source: "server",
            ts: new Date().toLocaleTimeString()
          });
        };
      } catch (ex) {
        this.handleError(ex);
        this.$toast.error("发生了一些错误!", {
          icon: "error"
        });
      }
    },
    disconnect() {
      this.socket.close();
    },
    handleError(error) {
      this.disconnect();
      this.connectionState = false;
      this.communication.log.push({
        payload: `发生了一个错误.`,
        source: "info",
        color: "#ff5555",
        ts: new Date().toLocaleTimeString()
      });
      if (error !== null)
        this.communication.log.push({
          payload: error,
          source: "info",
          color: "#ff5555",
          ts: new Date().toLocaleTimeString()
        });
    },
    sendMessage() {
      const message = this.communication.input;
      this.socket.send(message);
      this.communication.log.push({
        payload: message,
        source: "client",
        ts: new Date().toLocaleTimeString()
      });
      this.communication.input = "";
    },
    collapse({ target }) {
      const el = target.parentNode.className;
      document.getElementsByClassName(el)[0].classList.toggle("hidden");
    },
    getSourcePrefix(source) {
      const sourceEmojis = {
        // Source used for info messages.
        info: "\tℹ️ [INFO]:\t",
        // Source used for client to server messages.
        client: "\t👽 [SENT]:\t",
        // Source used for server to client messages.
        server: "\t📥 [RECEIVED]:\t"
      };
      if (Object.keys(sourceEmojis).includes(source))
        return sourceEmojis[source];
      return "";
    },
    toggleSSEConnection() {
      // If it is connecting:
      if (!this.connectionSSEState) return this.start();
      // Otherwise, it's disconnecting.
      else return this.stop();
    },
    start() {
      this.events.log = [
        {
          payload: `正在连接到 ${this.server}...`,
          source: "info",
          color: "var(--ac-color)"
        }
      ];
      if (typeof EventSource !== "undefined") {
        try {
          this.sse = new EventSource(this.server);
          this.sse.onopen = event => {
            this.connectionSSEState = true;
            this.events.log = [
              {
                payload: `连接到 ${this.server}.`,
                source: "info",
                color: "var(--ac-color)",
                ts: new Date().toLocaleTimeString()
              }
            ];
            this.$toast.success("连接成功", {
              icon: "sync"
            });
          };
          this.sse.onerror = event => {
            this.handleSSEError();
          };
          this.sse.onclose = event => {
            this.connectionSSEState = false;
            this.events.log.push({
              payload: `断开连接 ${this.server}.`,
              source: "info",
              color: "#ff5555",
              ts: new Date().toLocaleTimeString()
            });
            this.$toast.error("断开了连接", {
              icon: "sync_disabled"
            });
          };
          this.sse.onmessage = event => {
            this.events.log.push({
              payload: event.data,
              source: "server",
              ts: new Date().toLocaleTimeString()
            });
          };
        } catch (ex) {
          this.handleSSEError(ex);
          this.$toast.error("发生了一些错误!", {
            icon: "error"
          });
        }
      } else {
        this.events.log = [
          {
            payload: `这个浏览器似乎没有服务器发送事件的支持.`,
            source: "info",
            color: "#ff5555",
            ts: new Date().toLocaleTimeString()
          }
        ];
      }
    },
    handleSSEError(error) {
      this.stop();
      this.connectionSSEState = false;
      this.events.log.push({
        payload: `发生了一个错误.`,
        source: "info",
        color: "#ff5555",
        ts: new Date().toLocaleTimeString()
      });
      if (error !== null)
        this.events.log.push({
          payload: error,
          source: "info",
          color: "#ff5555",
          ts: new Date().toLocaleTimeString()
        });
    },
    stop() {
      this.sse.onclose();
      this.sse.close();
    }
  },
  updated: function() {
    this.$nextTick(function() {
      const divLog = document.getElementById("log");
      divLog.scrollBy(0, divLog.scrollHeight + 100);
    });
  }
};
</script>
