<template>
  <div class="container">
    <div class="row">
      <div class="col-sm-6 offset-3">

        <div v-if="sessionStarted" id="chat-container" class="card">
          <div class="card-header text-white text-center font-weight-bold subtle-blue-gradient">
            Share the page URL to invite new friends
          </div>
      <!-- упрощаем -->
          <div class="card-body">
            <div class="container chat-body" ref="chatBody">
              <div v-for="(message, index) in messages" :key="index" class="row chat-section">
      <!-- проверка пользователя -->
                <template v-if="username === getUsername(message)">
                  <div class="col-sm-7 offset-3">
                    <span class="card-text speech-bubble speech-bubble-user float-right text-white subtle-blue-gradient">
                      {{ message.message }}
                      <div class="text-right" style="font-size: 0.7em; opacity: 0.8; margin-top: 4px;">
                        {{ formatTime(message.create_date || new Date()) }}
                      </div>
                    </span>
                  </div>
                  <div class="col-sm-2">
                    <img class="rounded-circle" :src="generateAvatar(getUsername(message))"/>
                  </div>
                </template>

      <!-- собеседник -->
                <template v-else>
                  <div class="col-sm-2">
                    <img class="rounded-circle" :src="generateAvatar(getUsername(message))" />
                  </div>
                  <div class="col-sm-7">
                    <span class="card-text speech-bubble speech-bubble-peer">
                      {{ message.message }}
                      <div style="font-size: 0.7em; opacity: 0.5; margin-top: 4px;">
                        {{ formatTime(message.create_date || new Date()) }}
                      </div>
                    </span>
                  </div>
                </template>
              </div>
            </div>
          </div>

          <div class="card-footer text-muted">
            <form @submit.prevent="postMessage">
                <div class="row">
                    <div class="col-sm-10">
                        <input v-model="message" type="text" placeholder="Type a message" />
                            </div>
                            <div class="col-sm-2">
                        <button class="btn btn-primary">Send</button>
                    </div>
                </div>
            </form>
          </div>
        </div>

        <div v-else>
          <h3 class="text-center">Welcome !</h3>
          <br />
          <p class="text-center">
            To start chatting with friends click on the button below, it'll start a new chat session
            and then you can invite your friends over to chat!
          </p>
          <br />
          <button @click="startChatSession" class="btn btn-primary btn-lg btn-block">Start Chatting</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data () {
    return {
      sessionStarted: false,
      messages: [],
      message: ''
    }
  },

  created () {
    this.username = sessionStorage.getItem('username')
    const token = sessionStorage.getItem('authToken')
    if (!token) {
      this.$router.push('/signin')
      return
    }

    if (this.$route.params.uri) {
      this.joinChatSession()
    }

    this.connectToWebSocket()
  },
    updated () {
  // Scroll to bottom of Chat window
      const chatBody = this.$refs.chatBody
      if (chatBody) {
        chatBody.scrollTop = chatBody.scrollHeight
      }
    },

  methods: {
    startChatSession () {
      const token = sessionStorage.getItem('authToken')
      fetch('http://localhost:8000/api/chats/', {
        method: 'POST',
        headers: {
          'Authorization': `Token ${token}`,
          'Content-Type': 'application/json'
        }
      })
      .then(response => {
        if (!response.ok) throw new Error('Ошибка создания сессии')
        return response.json()
      })
      .then(data => {
        alert("Сессия создана! Перенаправляем...")
        this.sessionStarted = true
        this.$router.push(`/chats/${data.uri}/`)
      })
      .catch(error => {
        alert("Не удалось создать чат: " + error.message)
      })
    },

    getUsername(message) {
      if (!message || !message.user) return 'Anonymous';
      // eсли user — это объект {username: "..."}
      if (typeof message.user === 'object') {
        return message.user.username || 'Anonymous';
      }
      // eсли user — это просто строка (из сокетов)
      return message.user;
    },

    postMessage () {
      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');
      const payload = { message: this.message };
      fetch(`http://localhost:8000/api/chats/${uri}/messages/`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Token ${token}`
        },
        body: JSON.stringify(payload)
      })
      .then(() => {
        this.message = '';
      })
      .catch(error => alert("Ошибка отправки: " + error));
    },

     joinChatSession () {
      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');

      fetch(`http://localhost:8000/api/chats/${uri}/`, {
        method: 'PATCH',
        headers: {
          'Authorization': `Token ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ username: this.username })
      })
      .then(res => {
        if (!res.ok) throw new Error('Ошибка входа в сессию');
        return res.json();
      })
      .then(data => {
        console.log("✅ Успешно вошли в сессию:", data);
        this.sessionStarted = true;
        this.fetchChatSessionHistory();
      })
      .catch(err => {
        console.error("❌ Ошибка PATCH:", err);
      });
    },

    fetchChatSessionHistory () {
      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');
      fetch(`http://localhost:8000/api/chats/${uri}/messages/`, {
        headers: { 'Authorization': `Token ${token}` }
      })
      .then(res => res.json())
      .then(data => {
        this.messages = data.messages;
      });
    },

    connectToWebSocket () {
      const websocket = new WebSocket(`ws://localhost:8000/ws/chat/${this.$route.params.uri}/`)
      websocket.onopen = this.onOpen
      websocket.onclose = this.onClose
      websocket.onmessage = this.onMessage
      websocket.onerror = this.onError
    },

    onOpen (event) {
      console.log('Connection opened.', event.data)
    },

    onClose (event) {
      console.log('Connection closed.', event.data)

      // Try and Reconnect after five seconds
      setTimeout(this.connectToWebSocket, 5000)
    },

    onMessage (event) {
      try {
        const data = JSON.parse(event.data);
        // проверка формата
        if (data.message) {
          this.messages.push({
            user: typeof data.user === 'object' ? data.user.username : data.user,
            message: data.message
          });
        }
      } catch (e) {
        console.error("Ошибка получения данных из сокета:", e);
      }
    },

    // onMessage (event) {
    //     const data = JSON.parse(event.data);
    //     console.log("📩 Incoming WebSocket:", data); // <-- debug
    //     this.messages.push({
    //       user: data.user,
    //       message: data.message
    //     });
    // },

    onError (event) {
      alert('An error occured:', event.data)
    },
    generateAvatar(username) {
    const canvas = document.createElement('canvas');
    const size = 40;
    canvas.width = size;
    canvas.height = size;
    const ctx = canvas.getContext('2d');

    // Pick background color from a palette
    const colors = ['#007bff', '#28a745', '#ffc107', '#dc3545', '#6f42c1'];
    const color = colors[username.charCodeAt(0) % colors.length];

    // Draw circular background
    ctx.beginPath();
    ctx.arc(size / 2, size / 2, size / 2, 0, Math.PI * 2);
    ctx.fillStyle = color;
    ctx.fill();

    // Draw initial letter
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 20px Arial';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(username[0].toUpperCase(), size / 2, size / 2);

    // Return image as base64
    return canvas.toDataURL();
  },

    formatTime(dateString) {
    const date = new Date(dateString);
    return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  }


  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1,
h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}

.btn {
  border-radius: 0 !important;
}

.card-footer input[type="text"] {
  background-color: #ffffff;
  color: #444444;
  padding: 7px;
  font-size: 13px;
  border: 2px solid #cccccc;
  width: 100%;
  height: 38px;
}

.card-header a {
  text-decoration: underline;
}

.card-body {
  background-color: #ddd;
}

.chat-body {
  margin-top: -15px;
  margin-bottom: -5px;
  height: 380px;
  overflow-y: auto;
}

.speech-bubble {
  display: inline-block;
  position: relative;
  border-radius: 0.4em;
  padding: 10px;
  background-color: #fff;
  font-size: 14px;
}

.subtle-blue-gradient {
  background: linear-gradient(45deg,#004bff, #007bff);
}

.speech-bubble-user:after {
  content: "";
  position: absolute;
  right: 4px;
  top: 10px;
  width: 0;
  height: 0;
  border: 20px solid transparent;
  border-left-color: #007bff;
  border-right: 0;
  border-top: 0;
  margin-top: -10px;
  margin-right: -20px;
}

.speech-bubble-peer:after {
  content: "";
  position: absolute;
  left: 3px;
  top: 10px;
  width: 0;
  height: 0;
  border: 20px solid transparent;
  border-right-color: #ffffff;
  border-top: 0;
  border-left: 0;
  margin-top: -10px;
  margin-left: -20px;
}

.chat-section:first-child {
  margin-top: 10px;
}

.chat-section {
  margin-top: 15px;
}

.send-section {
  margin-bottom: -20px;
  padding-bottom: 10px;
}
</style>
