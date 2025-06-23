<template>
  <div class="appFull">
  <!-- Drawer laterale -->
  <div v-if="drawerOpen" class="drawer-overlay" @click.self="drawerOpen = false">
    <div class="drawer">
      <div class="drawer-header" style="overflow-x: hidden;">
         <br><div class="drawer-h2" >Le tue chat</div>
        <div style="height:1px; background:#333; margin-bottom:0px; width:100%;"></div>
        <ul v-if="userChats.length" style="margin-top:0; padding-left:12px; overflow-x: hidden;">
          <li
            v-for="chat in userChats"
            :key="chat.id"
            @click="selectChat(chat.id)"
            :class="{ 'selected-chat': chat.id === chatId }"
            style="cursor:pointer; white-space: nowrap; overflow-x: hidden; text-overflow: ellipsis;"
          >
            {{ chat.name }}
          </li>
        </ul>
        <button class="add-chat-btn" @click="addNewChat" style="margin-top:10px;width:100%">+ Nuova chat</button>
        <button class="drawer-close" @click="drawerOpen = false">×</button>
      </div>
    </div>
  </div>

  <h1 v-if="!loggedIn" class="login-title">Login</h1>
  <form v-if="!loggedIn" @submit.prevent="login" class="login-form">
    <input v-model="form.username" type="text" placeholder="Username" />
    <input v-model="form.password" type="password" placeholder="Password" />
    <button type="submit">Accedi</button>
  </form>
  <p v-if="errore && !loggedIn" class="errore">{{ errore }}</p>

  <div v-if="loggedIn" class="chat">
    <h2>
      <button v-if="!drawerOpen" class="drawer-btn" @click="drawerOpen = true">☰</button>
      Assistente ONO
      <button class="darkmode-btn" @click="toggleDarkMode" title="Dark mode">
        <img src="/darkmode.png" alt="Dark mode" style="width:40px;height:40px;" />
      </button>
    </h2>
    <div class="chat-messages" ref="messagesContainer">
      <div v-for="(msg, i) in messages" :key="i" :class="msg.sender">
        <b>{{ msg.sender === "user" ? "Tu" : "Bot" }}:</b>
        <span v-if="msg.sender === 'bot' && isLoading && !msg.text">
          <span class="spinner"></span>
        </span>
        {{ msg.text }}
      </div>
    </div>
    <div class="logo-ono-bg"></div>
    <div class="chat-footer">
        <form @submit.prevent="() => sendMessage(chatId)" class="chat-form">
          <!-- Bottone allega e menu -->
          <button type="button" class="allega-btn" title="Allega" @click="toggleAllegaMenu">
            <img :src="allegaIcon" alt="Allega" style="width: 28px; height: 28px" />
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
            color = "#fff6fc"
          />
          <button
            type="button"
            class="voice-btn"
            :class="{ recording: isRecording }"
            title="Messaggio vocale"
            @click="startVoiceMessage"
            :disabled="isLoading"
          >
            <img :src="micIcon" alt="Microfono" style="width: 28px; height: 28px" />
          </button>
          <button type="submit" class="send-btn" title="Invia" :disabled="isLoading">
            <img :src="sendIcon" alt="Invia" style="width: 28px; height: 28px" />
          </button>
        </form>
      </div>
  </div>

 
  <input ref="fileInput" type="file" style="display: none" @change="handleFile" />
  <input
    ref="imageInput"
    type="file"
    accept="image/*"
    style="display: none"
    @change="handleImage"
  />
</div>
</template>

<script setup>
import { reactive, ref, watch, nextTick, onMounted, computed } from "vue";
import allegaImg from "./assets/allega.png";
import allegaWhiteImg from "./assets/allega.white.png";
import sendImg from "./assets/send.png";
import sendWhiteImg from "./assets/send.white.png";
import micImg from "./assets/mic.png";
import micWhiteImg from "./assets/mic.white.png";

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
const userId = ref(null);
const userChats = ref([]);

const isDark = ref(false);
function toggleDarkMode() {
  isDark.value = !isDark.value;
  if (isDark.value) {
    document.body.classList.add('darkmode');
  } else {
    document.body.classList.remove('darkmode');
  }
}

const allegaIcon = computed(() => isDark.value ? allegaImg : allegaWhiteImg);
const micIcon = computed(() => isDark.value ? micImg : micWhiteImg);
const sendIcon = computed(() => isDark.value ? sendImg : sendWhiteImg);

async function login() {
  console.log("Login chiamato");
  if (!form.username || !form.password) {
    errore.value = "Compila tutti i campi!";
    return;
  }
  try {
    const response = await fetch("http://192.168.1.42:8000/user/login/", {
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
      userId.value = data.user_id;
      // RIMOSSA la creazione chat qui!
      await fetchUserChats();
      // Solo ora imposta loggedIn e resetta il form
      loggedIn.value = true;
      resetForm();
    } else {
      errore.value = "Credenziali errate!";
    }
  } catch (e) {
    errore.value = "Errore di rete.";
    console.error(e);
  }
}

async function fetchUserChats() {
  if (!userId.value) return;
  try {
    const res = await fetch(`http://192.168.1.42:8000/user/chats?user_id=${userId.value}`);
    if (res.ok) {
      const data = await res.json();
      userChats.value = data.chats || [];
    }
  } catch (e) {
    console.error("Errore nel recupero delle chat utente:", e);
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

  const response = await fetch("http://192.168.1.42:8000/message/reply/", {
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

  // Crea una nuova chat solo se chatId non corrisponde a una chat esistente
  const chatEsistente = userChats.value.some(c => c.id === chatId.value);
  let firstMsg = chatInput.value.trim();
  if (!chatEsistente) {
    const now = Math.floor(Date.now() / 1000);
    const chatTitle = firstMsg.length > 10 ? firstMsg.slice(0, 10) + '…' : firstMsg;
    const chatPayload = {
      user_id: userId.value,
      name: chatTitle || `Chat di ${form.username}`,
      at: now,
      messages: []
    };
    try {
      const chatRes = await fetch("http://192.168.1.42:8000/chat/create", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(chatPayload)
      });
      if (chatRes.ok) {
        const chatData = await chatRes.json();
        if (chatData.chat_id) {
          chatId.value = chatData.chat_id;
          await fetchUserChats();
          // Aggiorna il titolo dopo la creazione
          await fetch(`http://192.168.1.42:8000/chat/update_chat`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ chat_id: chatId.value, name: chatTitle })
          });
        }
      } else {
        const errorText = await chatRes.text();
        console.error("Errore nella creazione chat:", errorText);
        isLoading.value = false;
        return;
      }
    } catch (e) {
      console.error("Errore di rete nella creazione chat:", e);
      isLoading.value = false;
      return;
    }
  }

  const userMsg = chatInput.value;
  messages.value.push({ sender: "user", text: userMsg });
  chatInput.value = "";
  isLoading.value = true;
  messages.value.push({ sender: "bot", text: "" });

  await askBackend(userMsg, chatId.value, (partial) => {
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

async function loadChatMessages(chat_id) {
  try {
    const res = await fetch(`http://192.168.1.42:8000/chat/${chat_id}/messages`);
    if (res.ok) {
      const data = await res.json();
      messages.value = (data.messages || []).map(msg => ({
        sender: msg.role === 'user' ? 'user' : 'bot',
        text: msg.content
      }));
    }
  } catch (e) {
    console.error("Errore nel caricamento messaggi chat:", e);
  }
}

// Sostituisci il click handler nel drawer:
// <li v-for="chat in userChats" :key="chat.id" @click="selectChat(chat.id)" ...>
function selectChat(id) {
  chatId.value = id;
  drawerOpen.value = false;
  loadChatMessages(id);
}

function addNewChat() {
  if (!userId.value) return;
  let titolo = prompt("Titolo della nuova chat:");
  if (!titolo) return;
  titolo = titolo.trim();
  if (!titolo) return;
  const now = Math.floor(Date.now() / 1000);
  const chatPayload = {
    user_id: userId.value,
    name: titolo.length > 30 ? titolo.slice(0, 30) + '…' : titolo,
    at: now,
    messages: []
  };
  fetch("http://192.168.1.42:8000/chat/create", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(chatPayload)
  })
    .then(res => res.json())
    .then(async data => {
      if (data.chat_id) {
        chatId.value = data.chat_id;
        await fetchUserChats();
        messages.value = [];
        drawerOpen.value = false;
      }
    })
    .catch(e => console.error("Errore creazione nuova chat:", e));
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

onMounted(() => {
  // Scrolla sempre in fondo quando l'input riceve focus
  const input = document.querySelector('.chat-form input');
  if (input) {
    input.addEventListener('focus', () => {
      setTimeout(() => {
        const messages = document.querySelector('.chat-messages');
        if (messages) {
          messages.scrollTo({ top: messages.scrollHeight, behavior: 'smooth' });
        }
      }, 350);
    });
  }
});
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css?family=Inter:400,700&display=swap");

html, body, #app {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  overflow: hidden !important;
  box-sizing: border-box;
  touch-action: none; /* Previene scroll con el dedo */
}

.appFull {
  position: fixed;
  top: 0;
  left: 0;
  inset: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  font-family: "Inter", sans-serif;
  z-index: 1;
  background: transparent;
  touch-action: none; /* Previene scroll táctil */
}
/* Chat box principale */
.chat {
  position: relative;
  width: 100%;
  height: 100%; /* aumenta la percentuale per occupare più spazio nella container */
  background: rgba(66, 64, 64, 0.85);
  /*border-radius: 22px;*/
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
  width: 100%;
  height:100px;
  background: #5b6770;
  color: #fff;
  font-size: 2.3em;
  font-family: "Nokia Expanded", sans-serif;
  font-weight: 700;
  margin: 0;
  display: flex;
  align-items: center;
  justify-content: flex-start; /* allinea tutto a sinistra */
  padding-left: 18px; /* aggiungi padding a sinistra */
  letter-spacing: 2px;
  box-shadow: 0 2px 12px rgba(44, 62, 80, 0.1);
  text-shadow: 0 2px 8px rgba(44, 62, 80, 0.1);
  gap: 12px;
}


.darkmode-btn {
  position: absolute;
  right: 40px;
  top: 30px;
  background: transparent;
  border: none;
  cursor: pointer;
  padding: 0;
  border-radius: 50%;
  transition: background 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
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
  width: 100%;
}

.user {
  align-self: flex-end;
  background: linear-gradient(135deg, #8ca6db 60%, #5b6770 100%);
  color: #fff;
  border-radius: 20px 20px 4px 20px;
  padding: 14px 24px;
  margin: 2px 0;
  font-size: 35px;
  display: inline-block;
  max-width: 70%;
  word-break: break-word;
  text-align: right;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.08);
  margin-right: 40px;
}

.bot {
  align-self: flex-start;
  background: #babdc4;
  color: #2f3336;
  border-radius: 20px 20px 20px 4px;
  padding: 14px 24px;
  margin: 2px 0;
  font-size: 35px;
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
  padding: 5px 0 px 0;
  margin-bottom: 0px; /* <--- aggiungi o aumenta questo valore */
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
  width: 96%;
  height: 100px;
  background: rgba(107, 107, 107, 0.85);
  box-shadow: 0 2px 12px rgba(44, 62, 80, 0.08);
  margin-bottom: 0;
}

.chat-form input {
  position: relative;
  flex-grow: 1;
  right: 5%;
  flex: 1;
  border-radius: 40px;
  border: none;
  padding: 10px 18px; 
  font-size: 18px;
  background: rgba(107, 107, 107, 0);
  margin-bottom: 0;
  outline: none;
  transition: box-shadow 0.2s, border 0.2s;
  box-shadow: 0 1px 4px rgba(44, 62, 80, 0.06);
  margin-left: 35px;
  color: #f2f6fc;
  width: 100%;

}

/* Placeholder più chiaro per l'input chat */
.chat-form input::placeholder {
  color: #a1a1a1;
  font-size: 1.1em;
  font-weight: 500;
}

.errore {
  color: #ff4d4f;
  margin-top: 1rem;
  font-size: 20px;
  text-align: center;
  font-weight: 600;
  letter-spacing: 1px;
}

/* Materiali_ONO_Lean_logistics_V1-01 1 */
.logo-ono {
  width: 60px;
  height: 50px;
  position: relative;
  left: 50%;
  bottom: 50%; 
  background: url("/Materiali_ONO_Lean_logistics_V1-01.png") no-repeat center/contain;
  border-radius: 20px;
  z-index: 3;
  pointer-events: none;
}

/* Sfondo logo ONO */
.logo-ono-bg {
  position: absolute;
  left: 50%;
  bottom: 1200px; /* più vicino al fondo */
  width: 250px;
  height: 200px;
  transform: translateX(-50%);
  background: url("/Materiali_ONO_Lean_logistics_V1-01.png") no-repeat center/contain;
  opacity: 0.3;
  z-index: 0;
  pointer-events: none;
  margin: 0;
}

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
  background: rgba(98, 97, 97, 0.85);
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
  letter-spacing: 2px;
  background-color:  rgba(98, 97, 97, 0);
  /* Rendi la sezione header flessibile verticalmente */
  display: flex;
  flex-direction: column;
  height: 100%;
}

/* Rendi la lista delle chat scrollabile se supera l'altezza */
.drawer-header ul {
  list-style: none;
  padding-left: 0;
  margin: 0;
  color: #fff;
  max-height: 45vh;
  overflow-y: auto;
  /* Aggiungi un po' di padding a destra per la scrollbar */
  padding-right: 8px;
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
  background: transparent !important;
  box-shadow: none !important;
  color: #ffffff00;
  border: none;
  font-size: 1.5em;
  cursor: pointer;
  margin-left: 20px;
  padding: 0 8px 0 10px;
  display: flex;
  align-items: center;
  transition: color 0.2s;
  width: 60px;
  height: 60px;
  
}


.send-btn {
  background: transparent !important;
  box-shadow: none !important;
  color: #ffffff00;
  border: none;
  font-size: 1.5em;
  cursor: pointer;
  margin-right: 20px;
  padding: 0 8px 0 10px;
  display: flex;
  align-items: center;
  transition: color 0.2s;
  width: 60px;
  height: 60px;
}


.voice-btn {
  background: transparent !important;
  box-shadow: none !important;
  color: #ffffff00;
  border: none;
  font-size: 1.5em;
  cursor: pointer;
  margin-right: 20px;
  padding: 0 8px 0 10px;
  display: flex;
  align-items: center;
  transition: color 0.2s;
  width: 60px;
  height: 60px;
}


.voice-btn.recording {
  background: #419ffc;
}

/* Stili per il menu Allega */
.allega-menu {
  position: absolute;
  bottom: 60px;
  left: 24px;
  box-shadow: 0 4px 16px rgba(44, 62, 80, 0.13);
  padding: 12px 0;
  display: flex;
  flex-direction: column;
  min-width: 120px;
  z-index: 200;
  color: #f2f6fc;
}
.allega-menu button {
  border: none;
  color: #5b6770;
  font-size: 1em;
  padding: 10px 24px;
  text-align: left;
  cursor: pointer;
  transition: background 0.2s;
  background-color: transparent;
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
  background-color: transparent;
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

.selected-chat {
  background: var(--accent);
  color: #5b6770;
  border-radius: 8px;
  z-index: 200;
  background-color: #8ca6db !important;
}
.add-chat-btn {
  background: #5b6770;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 10px;
  cursor: pointer;
  font-size: 1em;
  transition: background 0.2s;
  font-weight: bold;
}
.drawer-h2{
  font-weight: bold;
  font-size: 1.4em;
  margin-bottom: 8px;
  color: #ffffff;
  width: 100%;
}

/* Login styles */
.login-title {
  font-size: 3rem;
  font-weight: bold;
  color: #5b6770;
  text-align: center;
  margin-top: 60px;
  margin-bottom: 24px;
  letter-spacing: 2px;
}

.login-form {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 18px;
  max-width: 400px;
  margin: 0 auto 32px auto;
  padding: 32px 24px;
  background: rgba(255,255,255,0.95);
  border-radius: 18px;
  box-shadow: 0 4px 24px rgba(44, 62, 80, 0.12);
}

.login-form input {
  font-size: 1.3rem;
  padding: 12px 18px;
  border-radius: 8px;
  border: 1px solid #babdc4;
  width: 100%;
  max-width: 320px;
  margin-bottom: 8px;
}

.login-form button[type="submit"] {
  font-size: 1.2rem;
  padding: 12px 32px;
  border-radius: 8px;
  background: linear-gradient(135deg, #8ca6db 60%, #5b6770 100%);
  color: #fff;
  border: none;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}

.login-form button[type="submit"]:hover {
  background: linear-gradient(135deg, #5b6770 60%, #8ca6db 100%);
}

/* Dark mode styles */

body.darkmode .chat {
  background: rgba(255, 255, 255, 0.85) !important;
}
body.darkmode .chat h2 {
  color: #fff !important;
  background: #5b6770 !important;
}
body.darkmode .user {
  color: #fff !important;
  background: linear-gradient(135deg, #8ca6db 60%, #5b6770 100%) !important;
}
body.darkmode .bot {
  color: #2f3336 !important;
  background: #f2f6fc !important;
}
body.darkmode .errore {
  color: #ff4d4f !important;
}
body.darkmode .drawer-header {
  color: #5b6770 !important;
}
body.darkmode .allega-menu button {
  color: #5b6770 !important;
}
body.darkmode .chat-footer {
  background: transparent !important;
}
body.darkmode .chat-form {
  background: rgba(255, 255, 255, 0.85) !important;
}

body.darkmode .drawer-btn {
  background: #5b6770 !important;
}
body.darkmode .drawer-close,
body.darkmode .allega-btn,
body.darkmode .send-btn,
body.darkmode .voice-btn {
  background: transparent !important;
  box-shadow: none !important;
}

body.darkmode .voice-btn.recording {
  background: #419ffc !important;
}
body.darkmode .drawer-overlay {
  background: rgba(44, 62, 80, 0.18) !important;
}
body.darkmode .drawer {
  background: #fff !important;
}
body.darkmode .allega-menu {
  background: #fff !important;
}


body.darkmode .spinner {
  border: 3px solid #8ca6db !important;
  border-top: 3px solid #fff !important;
  
}
body.darkmode .drawer-header ul {
  list-style: none;
  padding-left: 0;
  margin: 0;
  color: #5b6770 !important;
}
body.darkmode .drawer-header li {
  color: #5b6770 !important;

}
body.darkmode .drawer-h2{
  color :  #5b6770 !important;

}
body.darkmode .logo-ono-bg {
  opacity: 0.15 !important;
}

body,
body.darkmode,
.chat,
.chat.darkmode,
.chat h2,
.user,
.bot,
.drawer-header,
.allega-menu,
.drawer,
.chat-form,
.chat-form input,
.drawer-btn,
.drawer-close,
.allega-btn,
.send-btn,
.voice-btn,
.selected-chat {
  transition: background 0.4s, color 0.4s, border 0.4s, box-shadow 0.4s;
}

@media (max-width: 600px) {
  .chat {
    height: 100vh;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }
  .chat-messages {
    flex: 1 1 auto;
    overflow-y: auto;
    padding-bottom: 16px;
    min-height: 0;
    max-height: none;
    box-sizing: border-box;
  }
  .chat-footer {
    /* Nessun fixed/sticky! */
    width: 100vw;
    background: rgba(107, 107, 107, 0.85);
    z-index: 1001;
    margin-bottom: 0;
    padding-bottom: env(safe-area-inset-bottom, 0);
  }
  .chat-form {
    width: 100vw;
    height: 70px;
    border-radius: 0;
    margin-bottom: 0;
    box-shadow: 0 -2px 12px rgba(44, 62, 80, 0.08);
    background: rgba(107, 107, 107, 0.85);
    z-index: 1002;
  }
}

button:focus, .add-chat-btn:focus, .drawer-close:focus, .drawer-btn:focus, .darkmode-btn:focus, .allega-btn:focus, .voice-btn:focus, .send-btn:focus {
  outline: none !important;
  box-shadow: none !important;
}
</style>