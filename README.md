## ono assistant project

ONO Assistant is an AI-powered chat assistant platform that provides a seamless conversational experience for users and teams. It features a modern Vue.js frontend and a Python FastAPI backend, enabling real-time, multi-user chat with advanced AI capabilities. The system supports:

- User authentication and secure login
- Persistent chat history and chat management (create, rename, delete chats)
- Real-time messaging with AI-powered responses
- Markdown rendering and code formatting in messages
- Customizable chat interface with dark mode
- Multi-platform support (desktop and mobile responsive)
- Integration with local or remote AI models (OpenAI, Ollama, etc.)
- Configurable AI settings and system prompts
- RESTful API endpoints for all chat and user operations
- Database storage for users, chats, and messages


## PROJECT STRUCTURE

- ono-assistant-fronted/ # Frontend (Vue.js + Vite)
- ono-assistant/ # Backend (FastAPI + Python)

---

## HOW TO START THE PROJECT

# fronted

- npm install
- npm run dev --- --host

# backend

- pip install -r requirements.txt
- uvicorn main:app --reload --host

### 1. frontend: `ono-assistant-fronted`

- Purpose:
  - Provides the user interface for chat, login, and chat management.
  - Handles user authentication, chat display, and message input.
- Technologies:
  - Vue.js 3 (Composition API)
  - Vite (build tool)
  - CSS3 (custom styles)
- Communication:
  - Communicates with the backend via HTTP (REST API calls to `/user/login/`, `/user/chats`, `/chat/{chat_id}`, etc.)
  - Sends JSON payloads for login and chat messages
  - Receives JSON responses for authentication, chat history, and AI responses

### 2. backend: `ono-assistant`

- Purpose:
  - Handles all business logic, user authentication, chat management, and AI message generation.
  - Exposes RESTful API endpoints for the frontend.
- Technologies:

  - Python 3
  - FastAPI (web framework)
  - SQLite (or other DB, via `models/`)
  - Custom AI logic (in `ai.py`)

- Communication:
  - Receives HTTP requests from the frontend
  - Returns JSON responses
  - Handles CORS preflight requests
  - May call external AI APIs or local AI models (see `ai.py`)

### 3. database layer

- Purpose:
  - database access for users, chats, messages, and system data.
- Technologies:
  - Python (DB access via custom code in `models/`)
- Communication:
  - Used internally by the backend

### 4. ai service (ai.py)

- Purpose:
  - Generates AI responses for chat messages.
  - Can be configured to use local or remote AI models.
- Technologies:
  - Python
  - Custom logic (can integrate with OpenAI, Ollama, etc.)
- Communication:
  - Called by backend endpoints when a bot response is needed

## WORKFLOW OVERVIEW
1. Browser (Vue.js)
  - L’utente interagisce con l’interfaccia: ad esempio, invia una domanda tramite un modulo.
2. HTTP Request
  - Il frontend invia una richiesta HTTP (tipicamente POST o GET) al backend FastAPI.
3. Database (Lettura/Scrittura)
  - Il backend può leggere dati di contesto dalla base dati (es. storico conversazioni, prompt personalizzati).
  - Potrebbe anche salvare la richiesta in una tabella di log o “richieste in attesa”.
4. FastAPI Backend
  - Riceve la richiesta.
  - Prepara il prompt per Ollama (modello LLM locale) basandosi sui dati raccolti.
5. Ollama
  - Riceve il prompt e genera una risposta usando un modello LLM.
  - Restituisce la risposta testuale al backend.
6. Database (Scrittura risposta)
  - Il backend salva la risposta generata da Ollama nel database .
7. FastAPI Response
  - Il backend invia la risposta (in formato JSON) al frontend.
8. Browser (UI aggiornata)
  - Vue.js riceve la risposta e aggiorna l’interfaccia utente, mostrando la risposta generata.