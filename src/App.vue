<template>
  <div class="appFull">    
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
            :class="{ 'selected-chat': chat.selected }"
            style="cursor:pointer; white-space: nowrap; overflow-x: hidden; text-overflow: ellipsis; display: flex; align-items: center; position: relative; min-height: 40px;"
          >
            <span v-if="Number(chat.id) === Number(chatId.value)" style="margin-right: 8px; font-size: 1.5em; color: #ff4d4f; z-index: 10; pointer-events: none;">▶️</span>
            <template v-if="chat.editingName">
              <input v-model="chat.newName" style="flex:1; min-width:0; max-width:140px; font-size:1em; border-radius:6px; border:1px solid #babdc4; padding:2px 8px; margin-right:4px;" @click.stop />
              <button @click.stop="confirmRenameChat(chat)" title="Conferma" style="background:#8ca6db; color:#fff; border:none; border-radius:4px; margin-left:2px; cursor:pointer; padding:2px 8px;">✔</button>
              <button @click.stop="cancelRenameChat(chat)" title="Annulla" style="background:#babdc4; color:#333; border:none; border-radius:4px; margin-left:2px; cursor:pointer; padding:2px 8px;">✕</button>
            </template>
            <template v-else>
              <span style="flex:1; overflow-x: hidden; text-overflow: ellipsis;">{{ chat.name }}</span>
              <button type="button" @click.stop="startRenameChat(chat)" title="Rinomina chat" style="background:transparent; border:none; margin-left:4px; cursor:pointer; padding:0; width:28px; height:28px; display:flex; align-items:center; justify-content:center;">
                <img :src="editIcon" alt="Rinomina" style="width:24px; height:24px;" />
              </button>
              <button type="button" @click.stop="deleteChat(chat.id)" title="Elimina chat" style="background:transparent; border:none; margin-left:4px; cursor:pointer; padding:0; width:28px; height:28px; display:flex; align-items:center; justify-content:center;">
                <img :src="deleteIcon" alt="Elimina" style="width:28px; height:28px;" />
              </button>
            </template>
          </li>
        </ul>

        <button class="add-chat-btn" @click="addNewChat" style="margin-top:10px;width:100%">+ Nuova chat</button>
        <button class="drawer-close" @click="drawerOpen = false">×</button>
      </div>
    </div>
  </div>


  <h1 v-if="!loggedIn" class="login-title">ono assistant</h1>
  <div v-if="!loggedIn" class="login-logo-wrapper">
    <div class="logo-ono login-logo"></div>
  </div>


  <form v-if="!loggedIn && !showResetPassword && !showRegister" @submit.prevent="login" class="login-form">
    <input v-model="form.username" type="text" placeholder="Username" />
    <input v-model="form.password" type="password" placeholder="Password" />
    <button class ="login-btn"type="submit">ACCEDI</button>
    <!-- <button type="button" class="forgot-password-btn" @click="showResetPassword = true" style="background:none;border:none;color:#5b6770;text-decoration:underline;font-size:1em;margin-top:8px;cursor:pointer;">hai dimenticato la password?</button> -->
    <button type="button" class="register-btn" @click="showRegister = true" style="background:none;border:none;color:#5b6770;text-decoration:underline;font-size:1em;margin-top:8px;cursor:pointer; ">CREA UN ACCOUNT</button>
  </form>

  <div v-if="showRegister && !loggedIn" class="register-page">
    <div class="reset-password-page-center">
      <div class="reset-password-inner">
        <h1 class="reset-title">Crea un account</h1>
        <form @submit.prevent="handleRegister" class="reset-form">
          <input v-model="registerUsername" type="text" placeholder="Username" class="login-form-input" />
          <input v-model="registerPassword" type="password" placeholder="Password" class="login-form-input" />
          <button type="submit">Registrati</button>
        </form>
        <p v-if="registerFeedback" :class="{'success': registerFeedbackType==='success', 'error': registerFeedbackType==='error', 'info': registerFeedbackType==='info'}">{{ registerFeedback }}</p>
        <button class="back-to-login" @click="showRegister = false">Torna al login</button>
      </div>
    </div>
  </div>
  <div v-if="showResetPassword && !loggedIn" class="reset-password-page">
    <div class="reset-password-page-center">
      <div class="reset-password-inner">
        <h1 class="reset-title">reset password</h1>
        <form @submit.prevent="handleReset" class="reset-form">
          <input v-model="resetUsername" type="text" placeholder="Username" class="login-form-input" />
          <input v-model="resetNewPassword" type="password" placeholder="Nuova password" class="login-form-input" />
          <button type="submit">Resetta password</button>
        </form>
        <p v-if="resetFeedback" :class="{'success': resetFeedbackType==='success', 'error': resetFeedbackType==='error', 'info': resetFeedbackType==='info'}">{{ resetFeedback }}</p>
        <button class="back-to-login" @click="showResetPassword = false">Torna al login</button>
      </div>
    </div>
  </div>

  <p v-if="errore && !loggedIn" class="errore">{{ errore }}</p>

  <div v-if="loggedIn" class="chat">
    <h2 style="position: relative;">
      <button v-if="!drawerOpen" class="drawer-btn" @click="drawerOpen = true">☰</button>
      assistente ono
      <button class="darkmode-btn" @click="toggleDarkMode" title="Dark mode">
        <img src="/darkmode.png" alt="Dark mode" style="width:40px;height:40px;" />
      </button>
      <button v-if="isAdmin" class="settings-btn" @click="showSettings = true" title="Impostazioni" style="position: absolute; right: 70px; top: 30px; background: transparent; border: none; cursor: pointer; padding: 0; border-radius: 50%; display: flex; align-items: center; justify-content: center;">
        <img src="/settings.png" alt="Settings" style="width:36px;height:36px;" />
      </button>
      <button v-if="loggedIn" class="logout-btn" @click="logout" title="Logout" style="position: absolute; right: 60px; top: 30px; background: transparent; border: none; cursor: pointer; padding: 0; border-radius: 50%; display: flex; align-items: center; justify-content: center;">
        <img src="/logout.svg" alt="Logout" style="width:36px;height:36px;" />
      </button>
      <button v-if="loggedIn" class="info-btn" @click="showInfoModal = true" title="Info" style="position: absolute; right: 20px; top: 30px; background: transparent; border: none; cursor: pointer; padding: 0; border-radius: 50%; display: flex; align-items: center; justify-content: center;">
        <img src="/info.svg" alt="Info" style="width:36px;height:36px;" />
      </button>
  <!-- MODALE INFO README -->
  <div v-if="showInfoModal" class="modal-overlay" @click.self="showInfoModal = false">
    <div class="modal" style="max-width: 100%; max-height: 90vh; overflow-y: auto;">
      <ReadmeViewer />
      <button class="modal-cancel" @click="showInfoModal = false" style="margin-top:18px">Chiudi</button>
    </div>
  </div>


  <!-- MODALE SETTINGS: TABELLA UTENTI E PASSWORD -->
  <div v-if="showSettings" class="modal-overlay" @click.self="showSettings = false">
    <div class="modal settings-modal-darkmode">
      <h3 class="settings-title" style="color: #babdc4;">Gestione utenti</h3>
      <div v-if="usersLoading" class="settings-loading">Caricamento utenti...</div>
      <div v-else>
        <div class="user-table-wrapper">
          <table class="user-table-beautiful settings-table-center">
            <thead>
              <tr>
                <th class="settings-th"><img src="/account.png" class="user-icon-img" alt="Account"/> Username</th>
                <th class="settings-th"><img src="/lock.svg" class="lock-icon-img" alt="Lock"/> 
                <br>
                Password</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="user in users" :key="user.id">
                <td class="settings-td">
                  <div class="user-cell settings-user-cell">
                    <span class="user-name">{{ user.username }}</span>
                  </div>
                </td>
                <td class="settings-td">
                  <div class="password-cell settings-password-cell">
                    <template v-if="user.editingPassword">
                      <input
                        v-model="user.editPassword"
                        :type="user.showPassword ? 'text' : 'password'"
                        class="password-input settings-password-input"
                        :readonly="false"
                        placeholder="Nuova password"
                      />
                      <button @click="togglePassword(user)" class="toggle-password-btn settings-btn">{{ user.showPassword ? 'Nascondi' : 'Mostra' }}</button>
                      <button @click="savePassword(user)" class="toggle-password-btn settings-btn settings-btn-save">Salva</button>
                      <button @click="cancelEditPassword(user)" class="toggle-password-btn settings-btn settings-btn-cancel">Annulla</button>
                    </template>
                    <template v-else>
                      <input
                        :type="user.showPassword ? 'text' : 'password'"
                        :value="user.password"
                        readonly
                        class="password-input settings-password-input"
                      />
                      <button @click="togglePassword(user)" class="toggle-password-btn settings-btn">{{ user.showPassword ? 'Nascondi' : 'Mostra' }}</button>
                      <button @click="startEditPassword(user)" class="toggle-password-btn settings-btn settings-btn-edit">Modifica</button>
                    </template>
                  </div>
                  <div v-if="user.passwordFeedback" :class="{'success': user.passwordFeedbackType==='success', 'error': user.passwordFeedbackType==='error'}" class="settings-feedback">{{ user.passwordFeedback }}</div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <div v-if="users.length === 0" class="settings-no-users">Nessun utente trovato.</div>
      </div>
      <button class="modal-cancel settings-btn-close" @click="showSettings = false">Chiudi</button>
    </div>
  </div>


    </h2>

    <div class="chat-messages" ref="messagesContainer" style="display: flex; flex-direction: column; align-items: stretch;">
      <Message 
        v-for="(msg, i) in messages"
        :key="i"
        :text="msg.text" 
        :sender="msg.sender" 
        :loading="isLoading && msg.sender === 'bot' && !msg.text" 
      />
    </div>
    <div class="logo-ono-bg"></div>
    <div class="chat-footer">
      <form @submit.prevent="() => sendMessage(chatId)" class="chat-form" style="position: relative; width: 100%;">
        <div v-if="showOptions" class="options-multichoice options-above-input" style="position: absolute; left: 50%; top: -70px; transform: translateX(-50%); width: max-content; min-width: 300px; z-index: 20;">
          <div class="options-center-wrapper">
            <button v-for="option in optionsList" :key="option" @click="selectOption(option)" class="option-btn">{{ option }}</button>
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
          title="Messaggio vocale"
          @click="startVoiceMessage"
          :disabled="isLoading"
        >
          <img :src="micIcon" alt="Microfono" style="width: 40px; height: 40px" />
        </button>
        <button @click="stopRecognition" type="button" class="stop-btn" title="Ferma registrazione">
          <img src="/stop.white.png" alt="Stop" style="width: 40px; height: 40px" />
        </button>
        <button v-if="lastUserAudioUrl" @click="playLastUserAudio" type="button" class="play-audio-btn" title="Ascolta l'ultimo audio registrato">
          ▶️ Ascolta
        </button>
        

        <span style="position: relative; display: inline-block; width: 60px; height: 60px;">
          <button
            v-if="!isLoading"
            type="submit"
            class="send-btn"
            title="Invia"
            :disabled="isLoading"
            style="position: absolute; right: -90px; top: 0px;"
          >
            <img :src="sendIcon" alt="Invia" style="width: 40px; height: 40px" />
          </button>
          <button
            v-if="isLoading"
            type="button"
            class="stop-btn"
            @click="stopGeneration"
            :title="'Interrompi generazione'"
            style="position: absolute; left: 0; top: 0;"
          >
            <img :src="stopIcon" alt="Stop" style="width: 40px; height: 40px" />
          </button>
        </span>
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

  <!-- MODALE per nuovo titolo chat -->
  <div v-if="showNewChatModal" class="modal-overlay" @click.self="closeNewChatModal">
    <div class="modal">
      <h3>nuova chat</h3>
      <input
        v-model="newChatTitle"
        type="text"
        placeholder="Titolo della nuova chat"
        maxlength="30"
        @keyup.enter="confirmNewChat"
        autofocus
      />
      <div class="modal-actions">
        <button @click="confirmNewChat" class="modal-confirm">Crea</button>
        <button @click="closeNewChatModal" class="modal-cancel">Annulla</button>
      </div>
    </div>
  </div>

</div>
</template>

<script setup>
import ReadmeViewer from './components/ReadmeViewer.vue';
import Message from './components/Message.vue'
import { reactive, ref, watch, nextTick, onMounted, computed } from "vue";
import sendImg from "./assets/send.png";
import sendWhiteImg from "./assets/send.white.png";
import micImg from "./assets/mic.png";
import micWhiteImg from "./assets/mic.white.png";
import stopImg from "/stop.png";
import stopWhiteImg from "/stop.white.png";
import deleteImg from "./assets/delete.png";
import deleteWhiteImg from "./assets/delete.white.png";
import { marked } from 'marked' // oppure markdown-it
const showInfoModal = ref(false);
// La funzione showInfo non serve più, ora si usa showInfoModal = true
import editImg from "./assets/edit.svg";
import editWhiteImg from "./assets/edit.white.svg";
const editIcon = computed(() => isDark.value ? editImg : editWhiteImg);

const form = reactive({
  username: "",
  password: "",
});

const errore = ref("");
const loggedIn = ref(false);
const drawerOpen = ref(false);
const isLoading = ref(false);
const showReadmeModal = ref(false);
const messages = ref([]);
const messagesContainer = ref(null); // aggiungi questo ref
const chatInput = ref("");
const fileInput = ref(null);
const imageInput = ref(null);
const chatId = ref(1); // deve essere un numero, non stringa
const userId = ref(null);
const userChats = ref([]);
const isAdmin = ref(false);


const isDark = ref(false);
function toggleDarkMode() {
  isDark.value = !isDark.value;
  if (isDark.value) {
    document.body.classList.add('darkmode');
  } else {
    document.body.classList.remove('darkmode');
  }
}

const micIcon = computed(() => isDark.value ? micImg : micWhiteImg);
const sendIcon = computed(() => isDark.value ? sendImg : sendWhiteImg);
const stopIcon = computed(() => isDark.value ? stopImg : stopImg);
const deleteIcon = computed(() => isDark.value ? deleteImg : deleteWhiteImg);
window.stopRequested = false;

function stopGeneration() {
  window.stopRequested = true;
  isLoading.value = false;
  // Rimuovi l'ultimo messaggio bot vuoto (spinner)
  const last = messages.value[messages.value.length - 1];
  if (last && last.sender === 'bot' && !last.text) {
    messages.value.pop();
  }
}
const ip = ref("");
const stopRequested = ref(false);

async function fetchServerIp() {
  const res = await fetch("http://"+window.location.hostname+":8000/models/ip");
  const data = await res.json();
  ip.value = data.ip;
}
onMounted(async () => {
  await fetchServerIp();
});
async function login() {
  if (!ip.value) {
    errore.value = "Connessione al server in corso, riprova tra qualche secondo.";
    return;
  }
  console.log("Login chiamato");
  if (!form.username || !form.password) {
    errore.value = "Compila tutti i campi!";
    return;
  }
  try {
    const response = await fetch("http://" + ip.value + ":8000/user/login/", {
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
      // Recupera se admin
      try {
        const adminRes = await fetch(`http://${ip.value}:8000/user/${data.user_id}/is_admin`);
        if (adminRes.ok) {
          const adminData = await adminRes.json();
          isAdmin.value = !!adminData.is_admin;
        } else {
          isAdmin.value = false;
        }
      } catch (e) {
        isAdmin.value = false;
      }
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

// isAdmin è ora disponibile globalmente per la UI
async function fetchUserChats() {
  if (!userId.value) return;
  try {
    const res = await fetch(`http://${ip.value}:8000/user/chats?user_id=${userId.value}`);
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

  const response = await fetch("http://" + ip.value + ":8000/message/reply/", {
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
  window.stopRequested = false;

  do {
    if (window.stopRequested) {
      isLoading.value = false;
      break;
    }
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
  window.stopRequested = false;

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
      // UNIFICA HOST
      const chatRes = await fetch("http://" + ip.value + ":8000/chat/create", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(chatPayload)
      });
      if (chatRes.ok) {
        const chatData = await chatRes.json();
        if (chatData.chat_id) {
          chatId.value = chatData.chat_id;
          await fetchUserChats();
          // Correggi il payload per update_chat: serve id, user_id, name, at
          await fetch(`http://${ip.value}:8000/chat/update_chat`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              id: chatId.value,
              user_id: userId.value,
              name: chatTitle,
              at: now
            })
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
    const res = await fetch(`http://${ip.value}:8000/chat/${chat_id}/messages`);
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
  // Deseleziona tutte le chat
  userChats.value.forEach(c => c.selected = false);
  // Seleziona solo quella cliccata
  const chat = userChats.value.find(c => Number(c.id) === Number(id));
  if (chat) chat.selected = true;
  chatId.value = Number(id);
  drawerOpen.value = false;
  loadChatMessages(chatId.value);
}

const showNewChatModal = ref(false);
const newChatTitle = ref("");

function addNewChat() {
  if (!userId.value) return;
  newChatTitle.value = "";
  showNewChatModal.value = true;
}

function closeNewChatModal() {
  showNewChatModal.value = false;
  newChatTitle.value = "";
}

function confirmNewChat() {
  let titolo = newChatTitle.value.trim();
  if (!titolo) return;
  const now = Math.floor(Date.now() / 1000);
  const chatPayload = {
    user_id: userId.value,
    name: titolo.length > 30 ? titolo.slice(0, 30) + '…' : titolo,
    at: now,
    messages: []
  };
  // UNIFICA HOST
  fetch("http://" + ip.value + ":8000/chat/create", {
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
        closeNewChatModal();
      }
    })
    .catch(e => {
      console.error("Errore creazione nuova chat:", e);
      closeNewChatModal();
    });
}

// Elimina una chat
async function deleteChat(chatIdToDelete) {
  
  try {
    const res = await fetch(`http://${ip.value}:8000/chat/delete/${chatIdToDelete}`,
      { method: 'DELETE' });
    if (res.ok) {
      await fetchUserChats();
      if (chatId.value === chatIdToDelete) {
        chatId.value = null;
        messages.value = [];
      }
    } else {
      const err = await res.text();
      alert('Errore eliminazione chat: ' + err);
    }
  } catch (e) {
    alert('Errore di rete nell\'eliminazione chat');
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


onMounted(async () => {
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

// AGGIUNGI questa funzione per evitare l'errore resetForm is not defined
function resetForm() {
  form.username = "";
  form.password = "";
}


// Gestione reset password integrata nella pagina
const showResetPassword = ref(false);
const resetUsername = ref("");
const resetNewPassword = ref("");
const resetFeedback = ref("");
const resetFeedbackType = ref("");

async function handleReset() {
  resetFeedback.value = "";
  if (!resetUsername.value || !resetNewPassword.value) {
    resetFeedback.value = "Compila tutti i campi.";
    resetFeedbackType.value = "error";
    return;
  }
  if (resetNewPassword.value.length < 4) {
    resetFeedback.value = "La password deve essere di almeno 4 caratteri.";
    resetFeedbackType.value = "error";
    return;
  }
  // Recupera user id e verifica admin
  let userIdTmp = null;
  let isAdminTmp = false;
  try {
    const res = await fetch(`http://${ip.value}:8000/user/by_username/${encodeURIComponent(resetUsername.value)}`);
    if (res.ok) {
      const data = await res.json();
      userIdTmp = data.user_id;
      isAdminTmp = !!data.is_admin;
    }
  } catch {}
  if (!userIdTmp) {
    resetFeedback.value = "Utente non trovato.";
    resetFeedbackType.value = "error";
    return;
  }
  if (!isAdminTmp) {
    resetFeedback.value = "Solo gli amministratori possono resettare la password. Contatta un admin.";
    resetFeedbackType.value = "info";
    return;
  }
  // Procedi con reset
  try {
    const res = await fetch(`http://${ip.value}:8000/user/update_password`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userIdTmp, new_password: resetNewPassword.value })
    });
    const data = await res.json();
    if (data.success) {
      resetFeedback.value = "Password aggiornata con successo!";
      resetFeedbackType.value = "success";
      setTimeout(() => {
        showResetPassword.value = false;
        resetUsername.value = "";
        resetNewPassword.value = "";
        resetFeedback.value = "";
        resetFeedbackType.value = "";
      }, 1500);
    } else {
      resetFeedback.value = "Errore durante l'aggiornamento della password.";
      resetFeedbackType.value = "error";
    }
  } catch {
    resetFeedback.value = "Errore di rete.";
    resetFeedbackType.value = "error";
  }
}
// --- MULTICHOICE BUTTONS STATE ---
const showOptions = ref(false);
const optionsList = ref(["Sì", "No", "Dimmi di più"]);

function selectOption(option) {
  // Simula invio come messaggio utente
  showOptions.value = false;
  chatInput.value = option;
  sendMessage(chatId.value);
}

// Mostra i pulsanti solo se l'ultimo messaggio è del bot
watch(
  () => messages.value.length,
  () => {
    if (
      messages.value.length > 0 &&
      messages.value[messages.value.length - 1].sender === "bot"
    ) {
      showOptions.value = true;
    } else {
      showOptions.value = false;
    }
  }
);
// Stato per mostrare la modale settings
const showSettings = ref(false);
/* Bottone impostazioni (settings) */
// --- MODALE GESTIONE UTENTI (ADMIN) ---
const users = ref([]);
const usersLoading = ref(false);

async function fetchUsers() {
  usersLoading.value = true;
  try {
    const res = await fetch(`http://${ip.value}:8000/user/all`);
    const data = await res.json();
    users.value = (data || []).map(u => ({
      ...u,
      showPassword: false
    }));
  } catch (e) {
    users.value = [];
  } finally {
    usersLoading.value = false;
  }
}

function togglePassword(user) {
  user.showPassword = !user.showPassword;
}

watch(showSettings, (val) => {
  if (val) fetchUsers();
});


function startEditPassword(user) {
  user.editingPassword = true;
  user.editPassword = user.password;
  user.passwordFeedback = '';
  user.passwordFeedbackType = '';
}

function cancelEditPassword(user) {
  user.editingPassword = false;
  user.editPassword = undefined;
  user.passwordFeedback = '';
  user.passwordFeedbackType = '';
}

async function savePassword(user) {
  if (!user.editPassword || user.editPassword.length < 4) {
    user.passwordFeedback = 'La password deve essere di almeno 4 caratteri.';
    user.passwordFeedbackType = 'error';
    return;
  }
  user.passwordFeedback = '';
  user.passwordFeedbackType = '';
  try {
    const res = await fetch(`http://${ip.value}:8000/user/update_password`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: user.id, new_password: user.editPassword })
    });
    const data = await res.json();
    if (data.success) {
      user.password = user.editPassword;
      user.editingPassword = false;
      user.editPassword = undefined;
      user.passwordFeedback = 'Password aggiornata!';
      user.passwordFeedbackType = 'success';
      setTimeout(() => {
        user.passwordFeedback = '';
        user.passwordFeedbackType = '';
      }, 1500);
    } else {
      user.passwordFeedback = 'Errore durante il salvataggio.';
      user.passwordFeedbackType = 'error';
    }
  } catch {
    user.passwordFeedback = 'Errore di rete.';
    user.passwordFeedbackType = 'error';
  }
}
const showRegister = ref(false);
const registerUsername = ref("");
const registerPassword = ref("");
const registerFeedback = ref("");
const registerFeedbackType = ref("");

async function handleRegister() {
  registerFeedback.value = "";
  if (!registerUsername.value || !registerPassword.value) {
    registerFeedback.value = "Compila tutti i campi.";
    registerFeedbackType.value = "error";
    return;
  }
  if (registerPassword.value.length < 4) {
    registerFeedback.value = "La password deve essere di almeno 4 caratteri.";
    registerFeedbackType.value = "error";
    return;
  }
  try {
    const res = await fetch(`http://${ip.value}:8000/user/create`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username: registerUsername.value, password: registerPassword.value, is_admin: false })
    });
    const data = await res.json();
    if (res.ok && data.userID) {
      registerFeedback.value = "Registrazione avvenuta! Ora puoi accedere.";
      registerFeedbackType.value = "success";
      setTimeout(() => {
        showRegister.value = false;
        registerUsername.value = "";
        registerPassword.value = "";
        registerFeedback.value = "";
        registerFeedbackType.value = "";
      }, 1500);
    } else {
      registerFeedback.value = data.detail || "Errore durante la registrazione.";
      registerFeedbackType.value = "error";
    }
  } catch {
    registerFeedback.value = "Errore di rete.";
    registerFeedbackType.value = "error";
  }
}
// Gestione rinomina chat
function startRenameChat(chat) {
  chat.editingName = true;
  chat.newName = chat.name;
}
function cancelRenameChat(chat) {
  chat.editingName = false;
  chat.newName = '';
}
async function confirmRenameChat(chat) {
  const newName = (chat.newName || '').trim();
  if (!newName || newName === chat.name) {
    chat.editingName = false;
    return;
  }
  try {
    const now = Math.floor(Date.now() / 1000);
    const payload = {
      id: chat.id,
      user_id: userId.value,
      name: newName.length > 30 ? newName.slice(0, 30) + '…' : newName,
      at: now
    };
    const res = await fetch(`http://${ip.value}:8000/chat/update_chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
    if (res.ok) {
      chat.name = payload.name;
    }
  } catch {}
  chat.editingName = false;
}
function logout() {
  userId.value = null;
  chatId.value = 1;
  messages.value = [];
  userChats.value = [];
  isAdmin.value = false;
  errore.value = "";
  loggedIn.value = false;
  resetForm();
}
function showInfo() {
  window.open('/README.md', '_blank');
}
// --- AUDIO RECORDING ---

let mediaRecorder = null;
const lastUserAudioUrl = ref(null);
const audioChunks = [];

function startVoiceMessage() {
  try {
    if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
      alert('getUserMedia non disponibile.');
      return;
    }
    if (typeof window.MediaRecorder === 'undefined') {
      alert('MediaRecorder non disponibile.');
      return;
    }
    navigator.mediaDevices.getUserMedia({ audio: true }).then(stream => {
      try {
        mediaRecorder = new window.MediaRecorder(stream);
      } catch (err) {
        alert('MediaRecorder non supporta il tipo di stream audio.');
        return;
      }
      audioChunks.length = 0;
      mediaRecorder.ondataavailable = e => {
        if (e.data.size > 0) audioChunks.push(e.data);
      };
      mediaRecorder.onstop = () => {
        const blob = new Blob(audioChunks, { type: 'audio/webm' });
        lastUserAudioUrl.value = URL.createObjectURL(blob);
      };
      mediaRecorder.start();
    }).catch((err) => {
      alert('Permesso microfono negato o non disponibile.');
    });
  } catch (e) {
    alert('Errore JS: ' + e);
  }
}

function stopRecognition() {
  if (mediaRecorder && mediaRecorder.state !== 'inactive') {
    mediaRecorder.stop();
  }
}

function playLastUserAudio() {
  if (lastUserAudioUrl.value) {
    const audio = new Audio(lastUserAudioUrl.value);
    audio.play();
  }
}

</script>

<style scoped>
@import url("https://fonts.googleapis.com/css?family=Inter:400,700&display=swap");
@import "./assets/font/stylesheet.css";

html, body, #app {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  overflow: hidden !important;
  box-sizing: border-box;
  touch-action: none; /* Previene scroll con el dito */
}

.appFull {
  position: fixed;
  top: 0;
  left: 0;
  inset: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  /* font-family spostato sotto */
  z-index: 1;
  background: transparent;
  touch-action: none; /* Previene scroll táctil */
  align-items: center;
  justify-content: center;

}

/* Applica Aptos a tutto tranne header e login */
body, .appFull, .chat, .chat-messages, .chat-footer, .chat-form, .user, .bot, .drawer, .drawer-header ul, .drawer-close, .add-chat-btn, .errore, .logo-ono, .logo-ono-bg, .selected-chat, .send-btn, .voice-btn, .stop-btn, .modal, .modal-actions, .modal-confirm, .modal-cancel, pre, code, table, th, td {
  font-family: "Aptos", sans-serif !important;
}

/* Escludi header e login */
.chat h2,
.login-title,
.login-form {
  font-family: "Nokia Expanded", sans-serif !important;
}

/* Chat box principale */
.chat {
  position: relative;
  width: 100%;
  height: 100%; /* aumenta la percentuale per occupare più spazio nella container */
  background: rgba(66, 64, 64, 0.85);
  box-sizing: border-box;
  display: flex;
  flex: 1;
  flex-grow: 1;
  flex-direction: column;
  box-shadow: 0 4px 24px rgba(44, 62, 80, 0.12);
  backdrop-filter: blur(4px);
  border-radius: 0; /* rimosso ogni arrotondamento */
}

/* Header chat */
.chat h2 {
  position: relative;
  width: 100%;
  height:100px;
  background: #5b6770;
  color: #fff;
  font-size: 1.8em;
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
  right: 80px; /* spostato più a sinistra */
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
  margin-bottom: 0;
  padding: 0 0 0 0;
  background: transparent;
  border-radius: 0; /* rimosso ogni arrotondamento */
  display: flex;
  flex-direction: column;
  gap: 0;
  scroll-behavior: smooth;
  width: 100%;
  position: relative;
  z-index: 1;
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
  margin-right: 10px;
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
  margin-left: 10px;
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
  padding: 0; /* padding-bottom a 0 */
  margin-bottom: 0px;
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
  padding-bottom: 0; /* aggiunto per sicurezza */
}

.chat-form input {
  position: relative;
  flex-grow: 1;
  right: 5%;
  flex: 1;
  border-radius: 40px;
  border: none;
  padding: 10px 18px; 
  font-size: 30px;
  background: rgba(107, 107, 107, 0);
  margin-bottom: 0;
  outline: none;
  transition: box-shadow 0.2s, border 0.2s;
  margin-left: 5%;
  color: #f2f6fc;
  width: 100%;

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
  background: url("/Materiali_ONO_Lean_logistics_V1-01.png") no-repeat center/contain;
  border-radius: 20px;
  z-index: 3;
  pointer-events: none;
}

.login-logo-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
}

.login-logo {
  position: static;
  width: 100px;
  height: 75px;}

/* Sfondo logo ONO */
.logo-ono-bg {
  position: absolute;
  left: 50%;
  bottom: 38%;
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
  max-height: 100%;
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
.send-btn {
  background: transparent !important;
  box-shadow: none !important;
  color: #ffffff00;
  border: none;
  cursor: pointer;
  margin-right: 100px !important; /* sposta più a sinistra */
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
  cursor: pointer;
  margin-right: 10px; /* sposta anche il microfono più a sinistra */
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

.stop-btn {
  background: transparent !important;
  box-shadow: none !important;
  border: none;
  padding: 0 8px 0 10px;
  margin: 0;
  width: 60px;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.spinner {
  display: inline-block;
  width: 40px;
  height: 40px;
  border: 3px solid #8ca6db;
  border-top: 5px solid #fff;
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
  background-color: #5b6770 !important;
  font-weight: bold !important;
  box-shadow: 0 0 8px #419ffc55 !important;
  z-index: 2 !important;
  border-radius: 10px !important;
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
  font-size: 1.5em;
  margin-bottom: 0px;
  color: #ffffff;
  width: 100%;
  font-family: "Aptos", sans-serif;

}

/* Login styles */
.login-title {
  font-size: 2.3rem;
  font-weight: bold;
  color: #5b6770;
  text-align: center;
  margin-top: 60px;
  margin-bottom: 24px;
  letter-spacing: 2px;
  font-family: "Nokia Expanded", sans-serif;
}

.login-form {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 18px;
  max-width: 400px;
  margin: 0 auto 32px auto;
  padding: 32px 24px;
  background: rgba(255, 255, 255, 0);
  border-radius: 18px;
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
body.darkmode .chat-form input {
  color: #000000 !important;
}


body,
body.darkmode,
.chat,
.chat.darkmode,
.chat h2,
.user,
.bot,
.drawer-header,
.drawer,
.chat-form,
.chat-form input,
.drawer-btn,
.drawer-close,
.send-btn,
.voice-btn {
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

/* Nuovi stili per la modale nuova chat */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(44, 62, 80, 0.25);
  z-index: 2000;
  display: flex;
  align-items: center;
  justify-content: center;
}
.modal {
  background: #ffffff00;
  border-radius: 18px;
  box-shadow: 0 8px 32px rgba(44, 62, 80, 0.18);
  padding: 48px 40px 32px 40px;
  min-width: 340px;
  max-width: 95vw;
  min-height: 220px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 18px;
  position: relative;
}
.modal h3 {
  font-size: 2.2em;
  font-family: "Aptos", sans-serif;
  color: #5b6770;
  margin-bottom: 12px;
  font-weight: bold;
  letter-spacing: 1px;
}
.modal input[type="text"] {
  font-size: 1.3em;
  padding: 14px 20px;
  border-radius: 10px;
  border: 1.5px solid #babdc4;
  width: 100%;
  max-width: 320px;
  margin-bottom: 8px;
  background: #f7f8fa;
  color: #333;
  outline: none;
  transition: border 0.2s;
}
.modal input[type="text"]:focus {
  border: 1.5px solid #8ca6db;
}
.modal-actions {
  display: flex;
  gap: 18px;
  width: 100%;
  justify-content: center;
  margin-top: 10px;
}
.modal-confirm {
  background: linear-gradient(135deg, #8ca6db 60%, #5b6770 100%);
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 12px 32px;
  font-size: 1.1em;
  font-weight: bold;
  cursor: pointer;
  transition: background 0.2s;
}
.modal-cancel {
  background: #babdc4;
  color: #2f3336;
  border: none;
  border-radius: 8px;
  padding: 12px 32px;
  font-size: 1.1em;
  font-weight: bold;
  cursor: pointer;
  transition: background 0.2s;
}
.modal-confirm:hover {
  background: linear-gradient(135deg, #5b6770 60%, #8ca6db 100%);
}
.modal-cancel:hover {
  background: #8ca6db;
  color: #fff;
}
@media (max-width: 600px) {
  .modal {
    min-width: 90vw;
    padding: 32px 8vw 24px 8vw;
  }
  .modal h3 {
    font-size: 1.3em;
  }
}
button:focus, .add-chat-btn:focus, .drawer-close:focus, .drawer-btn:focus, .darkmode-btn:focus, .voice-btn:focus, .send-btn:focus {
  outline: none !important;
  box-shadow: none !important;
}
.login-form-input {
  font-size: 1.3rem;
  padding: 12px 18px;
  border-radius: 8px;
  border: 1px solid #babdc4;
  width: 100%;
  max-width: 320px;
  margin-bottom: 8px;
  box-sizing: border-box;
  display: block;
  align-content: center;
}

.center-reset-form {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  min-height: 320px;
}

/* Centra la pagina di reset password */
.reset-password-page-center {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100vw;
}
.reset-password-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  max-width: 400px;
  width: 100%;
}
.reset-title {
  font-size: 2.5rem;
  font-weight: bold;
  color: #5b6770;
  text-align: center;
  margin-bottom: 18px;
  letter-spacing: 2px;
  font-family: "Aptos", sans-serif;
}
.reset-form {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 18px;
  width: 100%;
}
.reset-form button[type="submit"] {
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
.reset-form button[type="submit"]:hover {
  background: linear-gradient(135deg, #5b6770 60%, #8ca6db 100%);
}
.back-to-login {
  margin-top: 12px;
  background: none;
  border: none;
  color: #5b6770;
  text-decoration: underline;
  font-size: 1em;
  cursor: pointer;
}
.success {
  color: #2e7d32;
  font-weight: bold;
  margin-top: 10px;
  text-align: center;
}
.error {
  color: #ff4d4f;
  font-weight: bold;
  margin-top: 10px;
  text-align: center;
}
.info {
  color: #5b6770;
  font-weight: bold;
  margin-top: 10px;
  text-align: center;
}
.options-multichoice {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 0;
  padding: 0;
  background: none;
  box-shadow: none;
}
.options-center-wrapper {
  display: flex;
  gap: 18px;
  justify-content: center;
  align-items: center;
  width: 100%;
  margin: 0;
}
.options-above-input {
  margin-bottom: 0 !important;
  margin-top: 0 !important;
  padding-bottom: 0 !important;
  position: absolute !important;
  left: 50% !important;
  top: -70px !important;
  transform: translateX(-50%) !important;
  width: max-content !important;
  min-width: 300px;
  z-index: 20;
}
.option-btn {
  font-size: 1.1rem;
  padding: 12px 28px;
  border-radius: 8px;
  background: linear-gradient(135deg, #8ca6db 60%, #5b6770 100%);
  color: #000000;
  border: none;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}

.options-btn:hover {
  background: linear-gradient(135deg, #5b6770 60%, #8ca6db 100%);
}

/* Pulsanti sopra la chat input */
.options-above-input {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 0 !important;
  margin-top: 0 !important;
  padding-bottom: 0 !important;
  position: relative;
  z-index: 10;
  
}
.settings-btn {
  margin-right: 205px;
  background: transparent;
  border: none;
  cursor: pointer;
  padding: 0;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}
.settings-btn:hover {
  background: #5b6770;
  color: #ffffff;
}
.user-table {
  width: 100%;
  margin-top: 20px;
  color: #000000;
  background-color: #5b6770;
  
}
.user-name{
  font-size: 50px;


}
.user-cell{
  margin-bottom: 15px;
}

.password-cell {
  display: flex;
  align-items: center;
  gap: 8px;
}
.password-input {
  font-size: 1.1rem !important;
  padding: 6px 12px;
  border-radius: 6px;
  border: 1px solid #babdc4;
  width: auto;
  min-width: 300px;
  margin-bottom: 0;
  background: #f7f8fa;
  color: #333;
  outline: none;
  transition: border 0.2s;
  box-sizing: border-box;
  flex: 1 1 ;
}
.toggle-password-btn{
  width: auto;
  height: 50px;
  font-size: medium;
  margin-right: 0px;
  
}
.logout-btn {
  background: #5b6770;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 10px 20px;
  cursor: pointer;
  font-size: 1em;
  transition: background 0.2s;
  margin-right: 150px;
}
.info-btn {
  background: #8ca6db;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 10px 20px;
  cursor: pointer;
  font-size: 1em;
  transition: background 0.2s;
  margin-right: 125px;
}
/* Assicurati che questi siano nel tuo <style scoped> in App.vue */
.modal-overlay {
  background: rgba(255, 255, 255, 0.721);  
  z-index: 99999 !important; /* ALTISSIMO */
  position: fixed !important; /* FISSO SULLA FINESTRA */
  top: 0 !important;
  left: 0 !important;
  width: 100vw !important; /* TUTTA LA LARGHEZZA */
  height: 100vh !important; /* TUTTA L'ALTEZZA */
}

.readme-modal {
  border: 5px solid blue !important; /* BORDO BLU SPESSO */
  background-color: white !important; /* SFONDO BIANCO */
  opacity: 1 !important; /* COMPLETAMENTE OPACO */
  min-width: 300px !important; /* MINIMO */
  min-height: 200px !important; /* MINIMO */
}
.settings-modal-darkmode {
  background: var(--settings-bg, #23272f);
  color: var(--settings-fg, #fff);
  border-radius: 18px;
  box-shadow: 0 4px 32px #0006;
  padding: 32px 32px 24px 32px;
  min-width: 70%;
  max-width: 520px;
}

.settings-table-center {
  margin: 0 auto;
  width: 100%;
  max-width: 420px;
  background: transparent;
}
.settings-th, .settings-td {
  background: transparent !important;
  border: none !important;
  font-size: 1.1em;
  color: var(--settings-fg, #fff);
}
.settings-user-cell, .settings-password-cell {
  display: flex;
  align-items: right;
  justify-content: right;
  padding-right: 120px;
}
.settings-password-input {
  width: 120px;
  text-align: center;
  border-radius: 8px;
  border: 1px solid #babdc4;
  padding: 4px 8px;
  font-size: 1em;
  background: #23272f;
  color: #fff;
}
.settings-btn {
  margin-left: 4px;
  border-radius: 6px;
  border: none;
  padding: 4px 10px;
  font-size: 1em;
  background: #3a4250;
  color: #fff;
  cursor: pointer;
  transition: background 0.2s;
}
.settings-btn-save {
  background: #8ca6db;
}
.settings-btn-cancel {
  background: #babdc4;
  color: #333;
}
.settings-btn-edit {
  background: #5b6770;
}
.settings-btn-close {
  display: block;
  background: #ff4d4f;
  color: #fff;
  font-weight: bold;
  font-size: 1.1em;
  border-radius: 8px;
  padding: 8px 24px;
}
.settings-feedback {
  font-size: 0.95em;
  margin-top: 2px;
}
.settings-no-users {
  text-align: center;
  color: #888;
  margin-top: 18px;
}
.settings-loading {
  text-align: center;
  color: #8ca6db;
  margin-bottom: 18px;
}
.user-icon-img, .lock-icon-img {
  vertical-align: middle;
  width: 28px;
  height: 28px;
  margin-right: 8px;
  padding-left: 75px;
}
.settings-th {
  font-weight: bold;
  color: #5b6770;
  font-size: 1.2em;
  text-align: left;
  padding-left: 70px;
  padding-right: 40px;
}
.settings-td {
  padding: 8px 0;
  text-align: right;
  color: #fff;
}
.login-btn{
  font-family: "Aptos", sans-serif;
}
.register-btn {
  font-family: "Aptos", sans-serif;
  background: linear-gradient(135deg, #8ca6db 60%, #5b6770 100%);
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 10px 20px;
  cursor: pointer;
  font-size: 1em;
  transition: background 0.2s;
}
</style>