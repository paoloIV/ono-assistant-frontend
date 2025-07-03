<template>
    <div class="readme-content" v-html="renderedReadme"></div>
  </template>
  
  <script>
  import { marked } from 'marked'; 
  export default {
    name: 'ReadmeViewer',
    data() {
      return {
        renderedReadme: '', // Qui verrà memorizzato l'HTML renderizzato
      };
    },
    async created() {
      try {
        const readmeMarkdown = await import('../../README.md?raw'); 
        this.renderedReadme = marked(readmeMarkdown.default || readmeMarkdown);
      } catch (error) {
        console.error("Errore nel caricamento o nella renderizzazione del README.md:", error);
        this.renderedReadme = '<p>Impossibile caricare il README.md. Assicurati che esista e che la configurazione del bundler sia corretta.</p>';
      }
    },
  };
  </script>
  
  <style scoped>
  
  .readme-content h1,
  .readme-content h2,
  .readme-content h3 {
    border-bottom: 1px solid #eee;
    padding-bottom: 0.3em;
    margin-top: 1em;
    color: #5b6770; /* Colore per i titoli */
    font-family: "Aptos", sans-serif; /* Font per i titoli */
  }
  
  .readme-content pre {
    background-color: #f6f8fa;
    border-radius: 3px;
    padding: 16px;
    overflow: auto;
    font-size: 0.9em;
  }
  
  .readme-content code {
    font-family: monospace;
    background-color: rgba(255, 255, 255, 0.989);
    padding: 0.2em 0.4em;
    border-radius: 3px;
  }
  
 
 .readme-content {
    color: #454e55;
    text-align: left;
    font-family: "Aptos", sans-serif; /* Font per il contenuto */
  }
  
  body.darkmode .readme-content h1,
  body.darkmode .readme-content h2,
  body.darkmode .readme-content h3 {
    color: #8ca6db; /* Colore per i titoli in modalità scura */
    border-bottom-color: #444;
  }
  
  .readme-content pre {
    background-color: #333;
    color: #f2f6fc;
  }
  
  body.darkmode .readme-content code {
    background-color: rgba(255,255,255,.1);
    color: #f2f6fc;
  }
  </style>