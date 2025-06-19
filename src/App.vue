<template>
  <!-- Drawer laterale -->
  <div v-if="drawerOpen" class="drawer-overlay" @click.self="drawerOpen = false">
    <div class="drawer">
      <div class="drawer-header">
        chat
        <button class="drawer-close" @click="drawerOpen = false">×</button>
      </div>
    </div>
  </div>

  <h1 v-if="!loggedIn">Login</h1>
  <form v-if="!loggedIn" @submit.prevent="login">
    <input v-model="form.username" type="text" placeholder="Username" />
    <input v-model="form.password" type="password" placeholder="Password" />
    <button type="submit">Accedi</button>
  </form>
  <p v-if="errore && !loggedIn" class="errore">{{ errore }}</p>

  <div v-if="loggedIn" class="chat">
    <h2>
      <button v-if="!drawerOpen" class="drawer-btn" @click="drawerOpen = true">☰</button>
      ASSISTENTE ONO
    </h2>
    <div class="chat-messages" ref="messagesContainer">
      <div v-for="(msg, i) in messages" :key="i" :class="msg.sender">
        <b>{{ msg.sender === "user" ? "Tu" : "Bot" }}:</b>
        <span v-if="msg.sender === 'bot' && isLoading && !msg.text">
          <span class="spinner"></span>
        </span>
        {{ msg.text }}
      </div>
      <div v-if="isLoading" class="bot loading"></div>
    </div>
    <div class="logo-ono-bg"></div>
    <div class="chat-footer">
      <form @submit.prevent="() => sendMessage(chatId)" class="chat-form">
        <!-- Bottone allega e menu -->
        <button type="button" class="allega-btn" title="Allega" @click="toggleAllegaMenu">
          <img :src="allegaImg" alt="Allega" style="width: 28px; height: 28px" />
        </button>
        <div v-if="showAllegaMenu" class="allega-menu-overlay" @click="closeAllegaMenu">
          <div class="allega-menu" @click.stop>
            <button type="button" @click="openFileDialog">File</button>
            <button type="button" @click="openImageDialog">Immagini</button>
          </div>
        </div>
        <input
          v-model="chatInput"
          type="text"
          placeholder="Scrivi un messaggio..."
          autocomplete="off"
          :disabled="isLoading"
        />
        <button
          type="button"
          class="voice-btn"
          :class="{ recording: isRecording }"
          title="Messaggio vocale"
          @click="startVoiceMessage"
          :disabled="isLoading"
        >
          <img :src="micImg" alt="Microfono" style="width: 28px; height: 28px" />
        </button>
        <button type="submit" class="send-btn" title="Invia" :disabled="isLoading">
          <img :src="sendImg" alt="Invia" style="width: 28px; height: 28px" />
        </button>
      </form>
    </div>
  </div>

  <!-- Ellipse 2 -->
  <div class="ellipse-2">
    <div class="logo-ono"></div>
  </div>

  <input ref="fileInput" type="file" style="display: none" @change="handleFile" />
  <input
    ref="imageInput"
    type="file"
    accept="image/*"
    style="display: none"
    @change="handleImage"
  />
</template>

<script setup>
import { reactive, ref, watch, nextTick } from "vue";
import allegaImg from "./assets/allega.png";
import sendImg from "./assets/send.png";
import micImg from "./assets/mic.png";

const form = reactive({
  username: "",
  password: "",
});

const errore = ref("");
const loggedIn = ref(false);
const drawerOpen = ref(false);
const showAllegaMenu = ref(false);
const isLoading = ref(false);

const messages = ref([]);
const messagesContainer = ref(null); // aggiungi questo ref
const chatInput = ref("");
const fileInput = ref(null);
const imageInput = ref(null);
const chatId = ref(1); // deve essere un numero, non stringa

async function login() {
  if (!form.username || !form.password) {
    errore.value = "Compila tutti i campi!";
    return;
  }
  try {
    const response = await fetch("http://localhost:8000/user/login/", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        username: form.username,
        password: form.password
      })
    });
    if (!response.ok) {
      errore.value = "Errore di rete o server.";
      return;
    }
    const data = await response.json();
    if (data.isValid) {
      errore.value = "";
      loggedIn.value = true;
      resetForm();
    } else {
      errore.value = "Credenziali errate!";
    }
  } catch (e) {
    errore.value = "Errore di rete.";
  }
}

async function askBackend(prompt, chat_id, onToken) {
  // Debug: stampa il payload prima di inviare
  const payload = {
    role: "user",
    content: prompt,
    chat_id: Number(chat_id)
  };
  console.log("Payload inviato a /message/reply/:", payload);

  const response = await fetch("http://localhost:8000/message/reply/", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload)
  });

  if (!response.ok) {
    const text = await response.text();
    console.error("Errore HTTP:", response.status, text);
    throw new Error("Errore HTTP " + response.status + ": " + text);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let fullText = "";
  let buffer = "";
  let done = false;

  do {
    const { value, done: streamDone } = await reader.read();
    done = streamDone;
    if (done && !value) break;
    const chunk = decoder.decode(value || new Uint8Array(), { stream: true });
    buffer += chunk;
    let lines = buffer.split("\n");
    buffer = lines.pop(); // l'ultima riga potrebbe essere incompleta
    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        const data = JSON.parse(line);
        if (data.message && data.message.content) {
          fullText += data.message.content;
          if (typeof onToken === "function") {
            onToken(fullText);
          }
        }
      } catch (e) {
        console.error("Errore parsing JSON line:", line, e);
      }
    }
  } while (!done);

  return fullText;
}
async function sendMessage(chat_id) {
  if (isLoading.value) return;
  if (!chatInput.value.trim()) return;
  const userMsg = chatInput.value;
  messages.value.push({ sender: "user", text: userMsg });
  chatInput.value = "";
  isLoading.value = true;
  messages.value.push({ sender: "bot", text: "" });

  // Corretto: passa chat_id come secondo parametro, callback come terzo
  await askBackend(userMsg, chat_id, (partial) => {
    messages.value[messages.value.length - 1].text = partial;
    messages.value = [...messages.value]; 
  });

  isLoading.value = false;
}

const isRecording = ref(false);
let recognitionInstance = null;

function startVoiceMessage() {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    alert("Il riconoscimento vocale non è supportato su questo browser.");
    return;
  }

  // Se già sta registrando, fermati
  if (isRecording.value && recognitionInstance) {
    recognitionInstance.stop();
    isRecording.value = false;
    recognitionInstance = null;
    return;
  }

  // Altrimenti, inizia a registrare
  const recognition = new SpeechRecognition();
  recognition.lang = "it-IT";
  recognition.interimResults = false;
  recognition.maxAlternatives = 1;

  recognition.onstart = () => {
    isRecording.value = true;
  };
  recognition.onresult = (event) => {
    const transcript = event.results[0][0].transcript;
    chatInput.value = transcript;
  };
  recognition.onerror = () => {
    isRecording.value = false;
    recognitionInstance = null;
  };
  recognition.onend = () => {
    isRecording.value = false;
    recognitionInstance = null;
  };

  recognitionInstance = recognition;
  recognition.start();
}

function toggleAllegaMenu() {
  showAllegaMenu.value = !showAllegaMenu.value;
}

function closeAllegaMenu() {
  showAllegaMenu.value = false;
}

function openFileDialog() {
  closeAllegaMenu();
  fileInput.value && fileInput.value.click();
}

function openImageDialog() {
  closeAllegaMenu();
  imageInput.value && imageInput.value.click();
}

function handleFile(event) {
  const file = event.target.files[0];
  if (file) {
    // Qui puoi gestire il file selezionato
    console.log("File selezionato:", file);
  }
}

function handleImage(event) {
  const file = event.target.files[0];
  if (file) {
    // Qui puoi gestire l'immagine selezionata
    console.log("Immagine selezionata:", file);
  }
}

// Scroll automatico ogni volta che cambia messages
watch(
  () => messages.value.length,
  async () => {
    await nextTick();
    if (messagesContainer.value) {
      messagesContainer.value.scrollTo({
        top: messagesContainer.value.scrollHeight,
        behavior: "smooth",
      });
    }
  }
);
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css?family=Inter:400,700&display=swap");

/* Chat box principale */
.chat {
  position: relative;
  width: 600px;
  height: 800px; /* aumenta la percentuale per occupare più spazio nella container */
  margin: 48px auto 0 auto;
  background: rgba(255, 255, 255, 0.85);
  border-radius: 22px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  box-shadow: 0 4px 24px rgba(44, 62, 80, 0.12);
  border: none;
  /* Effetto vetro */
  backdrop-filter: blur(4px);
}

/* Header chat */
.chat h2 {
  position: relative;
  width: 99, 5%;
  height: 70px;
  background: #5b6770;
  color: #fff;
  font-size: 2.3em;
  font-family: "Nokia Expanded", sans-serif;
  font-weight: 700;
  margin: 0;
  border-radius: 8px 8px 0 0;
  border-style: solid;
  border-width: 2px 2px 2px 2px;
  border-color: #fff; /* <-- bordo superiore bianco visibile */
  display: flex;
  align-items: center;
  justify-content: flex-start; /* allinea tutto a sinistra */
  padding-left: 18px; /* aggiungi padding a sinistra */
  letter-spacing: 2px;
  box-shadow: 0 2px 12px rgba(44, 62, 80, 0.1);
  text-shadow: 0 2px 8px rgba(44, 62, 80, 0.1);
  gap: 12px;
}

/* Messaggi */
.chat-messages {
  flex: 1;
  overflow-y: auto;
  margin-bottom: 0; /* <--- Cambia da 24px a 0 */
  padding: 28px 18px 0 18px;
  background: transparent;
  border-radius: 0 0 22px 22px;
  display: flex;
  flex-direction: column;
  gap: 14px;
  scroll-behavior: smooth;
}

.user {
  align-self: flex-end;
  background: linear-gradient(135deg, #8ca6db 60%, #5b6770 100%);
  color: #fff;
  border-radius: 20px 20px 4px 20px;
  padding: 14px 24px;
  margin: 2px 0;
  font-size: 18px;
  display: inline-block;
  max-width: 70%;
  word-break: break-word;
  text-align: right;
  box-shadow: 0 2px 12px rgba(44, 62, 80, 0.1);
  animation: fadeInUp 0.3s;
  z-index: 1; /* Assicurati che i messaggi dell'utente siano sopra */
}

.bot {
  align-self: flex-start;
  background: #f2f6fc;
  color: #2f3336;
  border-radius: 20px 20px 20px 4px;
  padding: 14px 24px;
  margin: 2px 0;
  font-size: 18px;
  display: inline-block;
  max-width: 70%;
  word-break: break-word;
  text-align: left;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.08);
  animation: fadeInUp 0.3s;
  z-index: 1; /* Assicurati che i messaggi del bot siano sopra */
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Footer grigio largo */
.chat-footer {
  width: 100%;
  background: transparent;
  border-radius: 0 0 0 0;
  padding: 5px 0 5px 0;
  margin-bottom: 5px; /* <--- aggiungi o aumenta questo valore */
  display: flex;
  justify-content: center;
  z-index: 2;
}

/* Form invio messaggio */
.chat-form {
  display: flex;
  gap: 14px;
  border-radius: 40px;
  padding: 0 0px;
  align-items: center;
  width: 95%;
  background: rgba(255, 255, 255, 0.85);
  box-shadow: 0 2px 12px rgba(44, 62, 80, 0.08);
  border: 1px solid #ffffff;
  margin-bottom: 0;
}

.chat-form input {
  position: relative;
  flex-grow: 1;
  right: 5%;
  flex: 1;
  border-radius: 40px;
  border: none;
  padding: 10px 18px; /* meno padding verticale */
  font-size: 18px;
  background: #ffffff;
  margin-bottom: 0;
  outline: none;
  transition: box-shadow 0.2s, border 0.2s;
  box-shadow: 0 1px 4px rgba(44, 62, 80, 0.06);
}

.chat-form input:focus {
  box-shadow: 0 1px 5px #ffffff;
  border: 1px solid #ffffff;
}

.errore {
  color: #ff4d4f;
  margin-top: 1rem;
  font-size: 20px;
  text-align: center;
  font-weight: 600;
  letter-spacing: 1px;
}

/* Ellipse 2 */
.ellipse-2 {
  display: none;
}

/* Materiali_ONO_Lean_logistics_V1-01 1 */
.logo-ono {
  width: 60px;
  height: 50px;
  background: url("/Materiali_ONO_Lean_logistics_V1-01.png") no-repeat center/contain;
  border-radius: 20px;
  z-index: 3;
  pointer-events: none;
}

/* Sfondo logo ONO */
.logo-ono-bg {
  position: absolute;
  left: 50%;
  bottom: 250px; /* più vicino al fondo */
  width: 220px;
  height: 180px;
  transform: translateX(-50%);
  background: url("/Materiali_ONO_Lean_logistics_V1-01.png") no-repeat center/contain;
  opacity: 0.13;
  z-index: 0;
  pointer-events: none;
  margin: 0;
}

/* Stili per il drawer */
.drawer-btn {
  position: static;
  margin-right: 12px; /* meno spazio tra bottone e testo */
  font-size: 1.3em;
  width: 32px;
  height: 32px;
  background: #5b6770;
  color: #fff;
  border-radius: 8px;
  border: none;
  cursor: pointer;
  transition: background 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

/*.drawer-btn:hover {
  background: #8ca6db;
}*/

.drawer-close {
  position: absolute;
  top: -7px;
  right: 16px;
  background: transparent;
  color: #5b6770;
  border: none;
  font-size: 1.5em;
  cursor: pointer;
  transition: color 0.2s;
}
.drawer-close:hover {
  color: #8ca6db;
}

.drawer-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(44, 62, 80, 0.18);
  z-index: 99;
}

.drawer {
  position: fixed;
  top: 0;
  left: 0;
  width: 50vw;
  max-width: 400px;
  height: 100vh;
  background: #fff;
  box-shadow: 2px 0 16px rgba(44, 62, 80, 0.18);
  z-index: 100;
  display: flex;
  flex-direction: column;
  animation: slideInDrawer 0.3s;
}

.drawer-header {
  font-family: "Aptos", "Inter", sans-serif;
  font-size: 2em;
  font-weight: bold;
  padding: 32px 24px 16px 24px;
  color: #5b6770;
  border-bottom: 1px solid #e0e0e0;
  letter-spacing: 2px;
}

@keyframes slideInDrawer {
  from {
    transform: translateX(-100%);
  }
  to {
    transform: translateX(0);
  }
}

.allega-btn {
  background: none;
  border: none;
  font-size: 1.5em;
  color: #5b6770;
  cursor: pointer;
  margin-right: 8px;
  padding: 0 8px 0 10px;
  display: flex;
  align-items: center;
  transition: color 0.2s;
}
.allega-btn:hover {
  color: #8ca6db;
}

.send-btn {
  background: #ffffff;
  color: #fff;
  border: none;
  border-radius: 50%;
  width: 0px;
  height: 38px;
  font-size: 1.3em;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-left: 8px;
  cursor: pointer;
  transition: background 0.2s;
}
.send-btn:hover {
  background: #ffffff;
}

.voice-btn {
  position: absolute;
  bottom: 13px;
  right: 65px;
  background: #ffffff;
  color: #ffffff;
  border: none;
  border-radius: 20%;
  width: 18px;
  height: 38px;
  font-size: 1.3em;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background 0.2s;
}
.voice-btn:hover {
  background: #ffffff;
}

.voice-btn.recording {
  background: #419ffc;
}

/* Stili per il menu Allega */
.allega-menu {
  position: absolute;
  bottom: 60px;
  left: 24px;
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(44, 62, 80, 0.13);
  padding: 12px 0;
  display: flex;
  flex-direction: column;
  min-width: 120px;
  z-index: 200;
  border: 1px solid #e0e0e0;
}
.allega-menu button {
  background: none;
  border: none;
  color: #5b6770;
  font-size: 1em;
  padding: 10px 24px;
  text-align: left;
  cursor: pointer;
  transition: background 0.2s;
}
.allega-menu button:hover {
  background: #f2f6fc;
}

/* Nuovi stili per il menu Allega (overlay) */
.allega-menu-overlay {
  position: fixed;
  inset: 0;
  z-index: 199;
  background: transparent;
}

.allega-menu-close {
  position: absolute;
  top: 16px;
  right: 16px;
  background: transparent;
  color: #5b6770;
  border: none;
  font-size: 1.5em;
  cursor: pointer;
  transition: color 0.2s;
}
.allega-menu-close:hover {
  color: #8ca6db;
}

.spinner {
  display: inline-block;
  width: 22px;
  height: 22px;
  border: 3px solid #8ca6db;
  border-top: 3px solid #fff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin-right: 10px;
  vertical-align: middle;
}
@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
.bot.loading {
  opacity: 0.7;
  font-style: italic;
}
</style>
