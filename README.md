# Ollama Chat Interface

**A multi-model AI chat app built on Ollama Cloud and Chainlit, with document understanding, voice input, MCP tool calling, and persistent chat history.**

## Overview

Ollama Chat Interface is a ChatGPT-style web app that lets you talk to several large open-weight models hosted on **Ollama Cloud** from one interface. Pick a model, attach documents or images, speak instead of typing, and connect external tools through the **Model Context Protocol (MCP)**. The model decides when to call them.

Conversations are stored in PostgreSQL, so you can sign in with GitHub, leave, and pick up any past chat where you left off.

## Features

- **8 cloud models in one UI.** Switch between Gemma 4, GPT-OSS, DeepSeek-V3.2, Qwen 3.5, Kimi-K2.5, GLM-5.1, MiniMax-M2.7, and Mistral-Large-3 using chat profiles.
- **Streaming responses.** Tokens appear as they are generated.
- **Document and image understanding.** Upload PDF, DOCX, TXT, JPG, JPEG, or PNG files. Text is extracted and summarized, and images are analyzed by a vision model.
- **Voice input.** Record a message in the browser. ElevenLabs Speech-to-Text transcribes it and sends it as a normal chat message.
- **MCP tool calling.** Connect any MCP server (stdio, SSE, or streamable HTTP). Its tools are discovered automatically, and the model can call them mid-conversation.
- **Quick commands.** `Search`, `Chat`, and `Summary` shortcuts in the message bar.
- **Authentication.** Sign in with GitHub OAuth or Auth0.
- **Persistent history.** Threads, messages, and feedback are saved to PostgreSQL, uploaded files go to Azure Blob Storage, and any chat can be resumed or shared.
- **Auto-titled threads.** Conversations are renamed automatically based on their content.

---

## Architecture

```text
                    ┌──────────────────────────────┐
   Browser  ───────►│      Chainlit UI (main.py)   │
 (text / voice /    └──────────────┬───────────────┘
  files / MCP)                     │
        ┌──────────────┬───────────┼─────────────┬────────────────┐
        ▼              ▼           ▼             ▼                ▼
  Speech-to-Text   Document     LLM client     MCP tools      Data layer
  (ElevenLabs)     processor   (Ollama Cloud)  (Tavily,       (SQLAlchemy)
                   PDF/DOCX/   chat + tool     GitHub, any       │
                   TXT/images  calling         MCP server)   ┌───┴────────┐
                       │           ▲                         ▼            ▼
                       └── vision ─┘                    PostgreSQL   Azure Blob
                           model                        (threads)    (files)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI and app framework | [Chainlit](https://docs.chainlit.io) |
| LLM inference | [Ollama Cloud](https://docs.ollama.com/cloud) (`ollama` Python client) |
| Tool integration | Model Context Protocol (MCP) |
| Speech-to-text | ElevenLabs (`scribe_v1`) |
| Document parsing | PyMuPDF, python-docx, pytesseract, python-magic |
| Persistence | SQLAlchemy and asyncpg on PostgreSQL |
| File storage | Azure Blob Storage |
| Auth | GitHub OAuth / Auth0 |
| Package manager | [uv](https://docs.astral.sh/uv/) |

---

## Getting Started

### Prerequisites

- **Python 3.13+**
- **[uv](https://docs.astral.sh/uv/getting-started/installation/)**
- An **[Ollama Cloud](https://ollama.com/cloud)** account and API key
- An **[ElevenLabs](https://elevenlabs.io/)** API key (for voice input)
- A **PostgreSQL** database (local, or a hosted one such as Supabase)
- A **GitHub OAuth app** or an **Auth0** application
- An **Azure Storage account** (used for uploaded-file storage)

### 1. Clone and install

```bash
git clone <your-repo-url> ollama-chat-interface
cd ollama-chat-interface
uv sync
```

### 2. Configure environment variables

```bash
# macOS / Linux
cp .env.example .env

# Windows
copy .env.example .env
```

Then fill in `.env` (see [Configuration](#configuration)). Generate the auth secret with:

```bash
uv run chainlit create-secret
```

### 3. Create the database tables

Run the SQL schema from [`LEARN.md`](LEARN.md#database-setup) against your PostgreSQL database. It creates the `users`, `threads`, `steps`, `elements`, and `feedbacks` tables.

### 4. Run the app

```bash
uv run chainlit run main.py -w
```

Open **http://localhost:8000**, sign in, pick a model, and start chatting.

---

## Configuration

All settings are read from `.env`.

| Variable | Required | Description |
|---|:---:|---|
| `OLLAMA_SECRET_KEY` | ✅ | Ollama Cloud API key |
| `OLLAMA_BASE_URL` | ✅ | Ollama API endpoint (default `https://ollama.com`) |
| `DEFAULT_MODEL` | ✅ | Fallback model when a profile is not recognized (for example `deepseek-v3.2:cloud`) |
| `VISION_MODEL` | ✅ | Vision model used for image analysis (for example `qwen3-vl:235b-cloud`) |
| `ELEVENLABS_API_KEY` | ✅ | ElevenLabs key for speech-to-text |
| `CHAINLIT_AUTH_SECRET` | ✅ | JWT secret for sessions (`chainlit create-secret`) |
| `OAUTH_GITHUB_CLIENT_ID` / `OAUTH_GITHUB_CLIENT_SECRET` | ⚠️ | GitHub OAuth credentials (configure GitHub **or** Auth0) |
| `OAUTH_AUTH0_CLIENT_ID` / `OAUTH_AUTH0_CLIENT_SECRET` / `OAUTH_AUTH0_DOMAIN` | ⚠️ | Auth0 credentials (configure GitHub **or** Auth0) |
| `LOCAL_DATABASE` | ✅ | SQLAlchemy URL, for example `postgresql+asyncpg://user:pass@localhost:5432/db` |
| `CONTAINER_NAME` / `STORAGE_ACCOUNT_NAME` / `STORAGE_KEY` | ✅ | Azure Blob Storage settings for uploaded files |
| `CHAINLIT_URL` | Deploy only | Public URL of your deployment, used for OAuth callbacks |

> **Never commit your `.env` file.** It is already excluded by `.gitignore`.

---

## Usage

### Chat with a model
Choose a model from the profile picker at the top, then type a message. You can switch models at any time by starting a new chat with a different profile.

### Attach files
Drag files into the message box or click the attachment button. Supported types are **PDF, DOCX, TXT, JPG, JPEG, and PNG**. Use the **Summary** command to get a combined summary of every attached file, or ask a question and the file contents are added to your prompt as context.

### Talk instead of type
Click the microphone, speak, and stop recording. Your speech is transcribed and answered like a typed message.

### Connect MCP tools
Click the **MCP** (plug) icon and add a server, for example the **Tavily** server for web search or the **GitHub** server for repository actions. The app lists the server's tools automatically, and the model calls them whenever they help answer your question. Allowed stdio commands (`uv`, `uvx`, `npx`) are configured in `.chainlit/config.toml`.

---

## Project Structure

```text
ollama-chat-interface/
├── main.py                         # Chainlit entry point and event handlers
├── src/
│   ├── llm/
│   │   ├── call_model.py           # Ollama streaming chat, tool-call loop, model mapping
│   │   └── speech_to_text.py       # Audio buffering and ElevenLabs transcription
│   ├── document/
│   │   ├── document_processor.py   # PDF/DOCX/TXT extraction and image analysis
│   │   └── processor_config.py     # File types, size limits, models
│   ├── ui/
│   │   ├── chat_profiles.py        # Available models (chat profiles)
│   │   ├── chat_starters.py        # Suggested starter prompts
│   │   ├── chat_resume.py          # Rebuilds history when resuming a thread
│   │   └── commands.py             # Search / Chat / Summary commands
│   ├── database/
│   │   └── persistent_data_layer.py  # SQLAlchemy data layer and Azure Blob client
│   ├── log/logger.py               # Logger setup
│   └── utils/config.py             # Environment variable loading
├── .chainlit/config.toml           # Chainlit features (uploads, audio, MCP, UI)
├── public/                         # Logos, model icons, theme, demo GIFs
├── notebook/Ollama_Cloud.ipynb     # Ollama Cloud API exploration notebook
├── LEARN.md                        # In-depth guide: internals, schema, deployment
├── pyproject.toml
└── requirements.txt
```

---

## Customization

| To… | Edit |
|---|---|
| Add or remove a model | `src/ui/chat_profiles.py`, the mapping in `src/llm/call_model.py`, and an icon in `public/model/` |
| Change supported file types or size limits | `src/document/processor_config.py` |
| Change upload, audio, or MCP settings | `.chainlit/config.toml` |
| Change the welcome screen | `chainlit.md` |
| Change the theme | `public/theme.json` |

---

## Deployment

The app deploys to [Railway](https://railway.com/) straight from GitHub:

1. Create a new Railway project from your GitHub repository.
2. Add all variables from your `.env` in the service settings.
3. Set the start command to `chainlit run main.py --host 0.0.0.0 --port $PORT`.
4. Generate a public domain, set `CHAINLIT_URL` to it, and update your GitHub OAuth callback URL to `<CHAINLIT_URL>/auth/oauth/github/callback`.

See [`LEARN.md`](LEARN.md#deploying-your-project-from-github) for a step-by-step guide with screenshots.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| App fails on startup | A required variable in `.env` is missing or empty |
| "Model not found" errors | The model name exists on Ollama Cloud and your API key is valid |
| Login redirect fails | The OAuth callback URL matches `CHAINLIT_URL` exactly |
| Chats are not saved | `LOCAL_DATABASE` is reachable and the schema has been created |
| File upload errors | Azure storage credentials, plus the file type and size limits |
| MCP tool errors | The MCP server is running and its command is in `allowed_executables` |

---

## Documentation

For a deeper walkthrough of how the app works internally, the full database schema, and the complete deployment guide, see **[LEARN.md](LEARN.md)**.
