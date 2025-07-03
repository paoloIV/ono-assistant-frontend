<template>
  <div class="reset-password-page">
    <h1 class="reset-title">Reset Password</h1>
    <div class="reset-form-wrapper">
      <form @submit.prevent="handleReset" class="reset-form">
        <input v-model="username" type="text" placeholder="Username" />
        <input v-model="newPassword" type="password" placeholder="Nuova password" />
        <button type="submit">Resetta password</button>
      </form>
      <p v-if="feedback" :class="{'success': feedbackType==='success', 'error': feedbackType==='error', 'info': feedbackType==='info'}">{{ feedback }}</p>
      <router-link to="/" class="back-to-login">Torna al login</router-link>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router';

const username = ref("");
const newPassword = ref("");
const feedback = ref("");
const feedbackType = ref("");
const router = useRouter();
const ip = ref("");

async function fetchServerIp() {
  try {
    const res = await fetch("http://"+window.location.hostname+":8000/models/ip");
    const data = await res.json();
    ip.value = data.ip;
  } catch {
    feedback.value = "Errore di rete.";
    feedbackType.value = "error";
  }
}
fetchServerIp();

async function handleReset() {
  feedback.value = "";
  if (!username.value || !newPassword.value) {
    feedback.value = "Compila tutti i campi.";
    feedbackType.value = "error";
    return;
  }
  if (newPassword.value.length < 4) {
    feedback.value = "La password deve essere di almeno 4 caratteri.";
    feedbackType.value = "error";
    return;
  }
  // Recupera user id e verifica admin
  let userIdTmp = null;
  let isAdminTmp = false;
  try {
    const res = await fetch(`http://${ip.value}:8000/user/by_username/${encodeURIComponent(username.value)}`);
    if (res.ok) {
      const data = await res.json();
      userIdTmp = data.user_id;
      isAdminTmp = !!data.is_admin;
    }
  } catch {}
  if (!userIdTmp) {
    feedback.value = "Utente non trovato.";
    feedbackType.value = "error";
    return;
  }
  if (!isAdminTmp) {
    feedback.value = "Solo gli amministratori possono resettare la password. Contatta un admin.";
    feedbackType.value = "info";
    return;
  }
  // Procedi con reset
  try {
    const res = await fetch(`http://${ip.value}:8000/user/update_password`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userIdTmp, new_password: newPassword.value })
    });
    const data = await res.json();
    if (data.success) {
      feedback.value = "Password aggiornata con successo!";
      feedbackType.value = "success";
      setTimeout(() => router.push('/'), 1500);
    } else {
      feedback.value = "Errore durante l'aggiornamento della password.";
      feedbackType.value = "error";
    }
  } catch {
    feedback.value = "Errore di rete.";
    feedbackType.value = "error";
  }
}
</script>

<style scoped>
.reset-password-page {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(255,255,255,0.95);
}
.reset-title {
  font-size: 2.2rem;
  color: #5b6770;
  margin-bottom: 24px;
  font-family: "Nokia Expanded", sans-serif;
}
.reset-form-wrapper {
  background: #fff;
  border-radius: 18px;
  box-shadow: 0 4px 24px rgba(44, 62, 80, 0.12);
  padding: 32px 24px;
  min-width: 320px;
  max-width: 95vw;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.reset-form {
  display: flex;
  flex-direction: column;
  gap: 18px;
  width: 100%;
}
.reset-form input {
  font-size: 1.1rem;
  padding: 12px 18px;
  border-radius: 8px;
  border: 1px solid #babdc4;
  width: 100%;
  margin-bottom: 8px;
}
.reset-form button[type="submit"] {
  font-size: 1.1rem;
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
.success {
  color: #2e7d32;
  font-weight: bold;
  margin-top: 12px;
}
.error {
  color: #ff4d4f;
  font-weight: bold;
  margin-top: 12px;
}
.info {
  color: #5b6770;
  font-weight: bold;
  margin-top: 12px;
}
.back-to-login {
  display: block;
  margin-top: 18px;
  color: #5b6770;
  text-decoration: underline;
  font-size: 1em;
  cursor: pointer;
}
</style>
