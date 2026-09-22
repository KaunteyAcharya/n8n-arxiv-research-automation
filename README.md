# ArXiv Research Analysis Agent

Automated n8n workflows that monitor [arXiv](https://arxiv.org) for new papers in chosen categories, filter them by keyword, run a local-LLM "first-principles" analysis on the full paper text, and deliver the results to Telegram — with an interactive yes/no confirmation step before committing to a full analysis. A companion workflow provides a Retrieval-Augmented Generation (RAG) document Q&A agent for chatting with your own uploaded documents.

## What this solves

Keeping up with new arXiv submissions in a niche research area is manual and time-consuming: checking listings daily, judging relevance from abstracts alone, and reading full PDFs to decide if a paper matters. This project automates that loop:

1. Every morning, it checks arXiv's "new submissions" listing for one or more categories (by default General Relativity & Quantum Cosmology `gr-qc` and Quantitative Finance `q-fin`).
2. It filters new papers against a keyword list you define.
3. For each match, it sends you a Telegram message with the title, abstract, and a `yes`/`no` prompt.
4. If you reply `yes <arxiv_id>`, it downloads the full PDF, extracts the text, and asks a local Ollama model to walk through the paper's derivation/methodology from first principles, producing a Markdown report delivered back to you on Telegram.
5. If you reply `no <arxiv_id>`, it marks the paper as skipped and does nothing further.

A second, independent workflow (`RAG Doc QnA agent`) lets you upload your own documents (PDF/TXT/DOCX/MD) through a simple web form, embeds and stores them in a local ChromaDB vector store, and exposes a chat interface backed by a local Ollama model that answers questions strictly from the uploaded document content.

## Architecture

### 1. `arxiv-research-intelligence-workflow.json` — Daily paper monitor + interactive analysis

Two parallel branches (one per arXiv category, e.g. `gr-qc` and `q-fin`), each driven by a daily schedule trigger:

```
Schedule Trigger (daily, e.g. 08:30)
   └─▶ Build Daily Query          (constructs the arXiv category/date query)
        └─▶ Query arXiv           (fetches the public "new submissions" listing page)
             └─▶ Parse + Keyword Gate   (parses the HTML listing, filters by keyword list)
                  ├─▶ Store Pending + Brief  ─▶ Send Brief (Telegram)   [awaiting yes/no]
                  └─▶ Download Full PDF
                       └─▶ Extract Full PDF Text
                            └─▶ Build First-Principles Prompt
                                 └─▶ Ollama Full-Paper Analysis (local LLM, POST to Ollama)
                                      └─▶ Create Markdown Report
                                           └─▶ Telegram (sendDocument)
```

A second, independently scheduled branch handles the interactive reply:

```
Schedule Trigger (polling interval)
   └─▶ Get Telegram Updates (long-poll getUpdates, offset-tracked)
        └─▶ Process Callbacks   (parses "yes <id>" / "no <id>" replies against pending papers)
             ├─▶ Filter — Toasts ─▶ Send Reply Ack   (acknowledgement messages)
             ├─▶ Lookup Pending Paper (gr-qc)  ─▶ [feeds into the Download Full PDF branch above]
             └─▶ Lookup Pending Paper (q-fin)  ─▶ [feeds into the Download Full PDF branch above]
```

State (which papers are pending/in-progress/declined, and the Telegram update offset) is kept in n8n's workflow static data — no external database is required for this workflow.

### 2. `rag-doc-qna-workflow.json` — Document upload + RAG chat

Two independent triggers in one workflow:

**Ingestion pipeline:**
```
Form Trigger (file upload: PDF/TXT/DOCX/MD)
   └─▶ Chroma Vector Store (insert)
        ▲ fed by: Default Data Loader (PDF loader) ◀── Recursive Character Text Splitter (chunk 400 / overlap 100)
        ▲ fed by: Embeddings Ollama (nomic-embed-text)
```

**Chat pipeline:**
```
Chat Trigger (chat message received)
   └─▶ Question and Answer Chain (Retrieval QA, strict "answer only from context" system prompt)
        ▲ fed by: Ollama Chat Model (llama3.2:1b)
        ▲ fed by: Vector Store Retriever (topK 15) ◀── Chroma Vector Store ◀── Embeddings Ollama
```

## Technologies used

- [n8n](https://n8n.io) — workflow orchestration (self-hosted)
- [Ollama](https://ollama.com) — local LLM inference (analysis model + embeddings + chat model)
- [ChromaDB](https://www.trychroma.com) — self-hosted vector store for the RAG workflow
- [Telegram Bot API](https://core.telegram.org/bots/api) — notification and interactive yes/no delivery channel
- arXiv public listing pages (`arxiv.org/list/<category>/new`) — no API key required

## Project structure

```
.
├── arxiv-research-intelligence-workflow.json   # Daily monitor + interactive analysis workflow
├── rag-doc-qna-workflow.json                   # Document upload + RAG chat workflow
├── .env.example                                 # Placeholder environment variables (reference only)
├── .gitignore
└── README.md
```

> Note: n8n workflows configure most values (URLs, model names, credentials) inside the node parameters and n8n's credential store, not via `.env` files. The `.env.example` in this repo is a reference for the values you'll need to have on hand when configuring nodes and credentials manually in the n8n editor — it is not consumed automatically by n8n.

## Setup requirements

- A running n8n instance (self-hosted; tested against a recent n8n version with the `@n8n/n8n-nodes-langchain` community/built-in nodes available for the RAG workflow).
- [Ollama](https://ollama.com) running and reachable from your n8n instance, with the following models pulled:
  - An analysis/chat model (the original used `qwen3:4b` for paper analysis and `llama3.2:1b` for RAG chat — any Ollama chat-capable model works, adjust for your hardware).
  - An embedding model (`nomic-embed-text`).
- A self-hosted [ChromaDB](https://www.trychroma.com) instance (only required for the RAG workflow).
- A Telegram bot (only required for the arXiv monitor workflow) and your personal Telegram chat/user ID. To create one: open a chat with [@BotFather](https://t.me/BotFather) on Telegram and send `/newbot`. Follow the prompts to choose a name and a unique username for your bot. BotFather will reply with a bot token (looks like `123456789:ABC-your-token`) — copy this, you'll need it for setup below. Then message [@userinfobot](https://t.me/userinfobot) to get your own numeric chat/user ID, which the workflow uses to know where to send you messages.

## Credentials required

None of the credentials below are included in this repository. You must create your own in n8n and select them on the relevant nodes after import.

| Credential type | Used by | What you need |
|---|---|---|
| Telegram API | `arxiv-research-intelligence-workflow.json` (all Telegram nodes) | A bot token from [@BotFather](https://t.me/BotFather), added as an n8n "Telegram API" credential |
| Ollama API | Both workflows (LLM/embedding nodes) | The base URL of your running Ollama instance, added as an n8n "Ollama" credential |
| ChromaDB (self-hosted) | `rag-doc-qna-workflow.json` (vector store nodes) | The URL of your running ChromaDB instance, added as an n8n "ChromaDB Self-Hosted" credential |

## Configuration required after import

1. **Import both workflow JSON files** into n8n (see below).
2. **Assign your own credentials** on every node that references `telegramApi`, `ollamaApi`, or `chromaSelfHostedApi` — the credential `id` fields have been stripped, so n8n will show these as unset. Select or create your own credential of the matching type on each node.
3. **Replace the Telegram chat ID placeholder.** In `arxiv-research-intelligence-workflow.json`, the nodes `GR-QC — Telegram`, `Q-FIN — Telegram`, `GR-QC — Send Brief`, and `Q-FIN — Send Brief` have `chatId` set to `YOUR_TELEGRAM_CHAT_ID`. Replace this with your own numeric Telegram user/chat ID (message [@userinfobot](https://t.me/userinfobot) to find yours).
4. **Replace the Telegram bot token placeholder.** In the `Get Telegram Updates` node, the URL contains `botYOUR_TELEGRAM_BOT_TOKEN` — n8n's Telegram credential type does not cover raw HTTP Request nodes, so this node authenticates via the token embedded directly in the URL. Replace `YOUR_TELEGRAM_BOT_TOKEN` with your bot's actual token (keep the `bot` prefix, e.g. `.../bot123456:ABC-your-token/getUpdates`), or better, store it in an n8n credential/variable and reference it via an expression instead of hardcoding it.
5. **Replace the Ollama URL placeholder.** The nodes `GR-QC — Ollama Full-Paper Analysis` and `Q-FIN — Ollama Full-Paper Analysis` call Ollama directly via HTTP Request (`http://YOUR_OLLAMA_URL/api/chat`) rather than through the Ollama credential type. Replace `YOUR_OLLAMA_URL` with your Ollama instance's host and port (for a local Docker-based n8n install talking to Ollama running on the host machine, this is typically `host.docker.internal:11434`; for a native install, `localhost:11434`).
6. **Review the keyword lists and categories** in the `Build Daily Query` and `Parse + Keyword Gate` code nodes for each domain branch, and adjust the arXiv category codes and keywords to match your research interests.
7. **Review the schedule trigger times** (`Daily 08:30 IST` and the polling `Schedule Trigger`) and adjust to your timezone/preferences.
8. **Set the ChromaDB collection name** (`rag_documents` by default, in the Chroma Vector Store nodes) if you want a different collection.
9. Both workflows are imported with `active: false`. Activate them from the n8n editor once credentials and configuration are verified.

## How to import the n8n workflows

1. Open your n8n instance.
2. Go to **Workflows → Import from File** (or use the "..." menu → **Import from File** on the workflow list), and select `arxiv-research-intelligence-workflow.json`. Repeat for `rag-doc-qna-workflow.json`.
3. Follow the **Configuration required after import** steps above before activating either workflow.

## How to run the automation

- **ArXiv monitor:** once activated, the `Daily 08:30 IST`-equivalent trigger runs automatically each day; the second `Schedule Trigger` polls Telegram for your `yes`/`no` replies on its own interval. No manual execution is needed once configured.
- **RAG Doc QnA agent:** open the form trigger's public URL to upload documents into the vector store; open the chat trigger's chat URL (or embed it) to ask questions against everything you've uploaded.
- Both workflows can also be run manually from the n8n editor ("Execute workflow") for testing.

## Security considerations

- **Never commit real credentials.** This repository's workflow JSON files contain no API keys, bot tokens, or credential IDs — all such values were replaced with placeholders (`YOUR_TELEGRAM_BOT_TOKEN`, `YOUR_TELEGRAM_CHAT_ID`, `YOUR_OLLAMA_URL`) or stripped entirely (credential `id` fields). You must supply your own via n8n's credential store or by editing the placeholder values directly, and should avoid pasting secrets into files that get committed to version control.
- **Telegram bot token in a URL.** Because the `Get Telegram Updates` node uses a raw HTTP Request rather than the Telegram credential type, the bot token lives in the node's URL field. Treat your n8n workflow exports and instance backups as sensitive once you've filled in a real token — do not re-export and share this workflow publicly after configuring it with real values.
- **The RAG chat trigger's `public` flag has been set to `false`** in this export. Enabling it makes the chat endpoint reachable without n8n authentication; only enable it if you understand the exposure and have appropriate network/access controls in front of it.
- **Local-only LLM inference.** Both workflows are designed around a self-hosted Ollama instance, so paper/document content is not sent to a third-party LLM API by default. If you adapt this to use a cloud LLM provider, review what data you are sending externally.
- **This workflow was exported with `active: false`.** Review all node configuration and credentials before activating it against your own Telegram bot and Ollama/ChromaDB instances.
