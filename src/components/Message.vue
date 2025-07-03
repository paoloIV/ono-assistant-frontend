<template>
  <div :class="[props.sender, 'message']">
    <span v-if="props.loading && props.sender === 'bot' && !props.text" class="spinner"></span>
    <span v-else v-html="renderedText"></span>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { marked } from 'marked'

const props = defineProps({
  text: { type: String, default: '' },
  sender: { type: String, default: 'bot' },
  loading: { type: Boolean, default: false }
})

const renderedText = computed(() => props.text ? marked.parse(props.text) : '')
</script>

<style scoped>
.message {
  margin: 2px 0;
  font-size: 35px;
  display: inline-block;
  max-width: 70%;
  word-break: break-word;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.08);
  font-family: 'Aptos', Arial, sans-serif;
  line-height: 1.1;
  padding-top: 4px;
  padding-bottom: 4px;
}
.user.message {
  background: #8ca6db;
  color: #fff;
  border-radius: 20px 20px 4px 20px;
  padding: 4px 24px;
  text-align: right;
}
.bot.message {
  align-self: flex-start;
  background: #babdc4;
  color: #2f3336;
  border-radius: 20px 20px 20px 4px;
  padding: 4px 24px;
  text-align: left;
  animation: fadeInUp 0.3s;
  z-index: 1;
}
.spinner {
  display: inline-block;
  width: 40px;
  height: 40px;
  border: 5px solid #8ca6db;
  border-top: 5px solid #fff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin-right: 10px;
  vertical-align: middle;
}
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}
</style>
