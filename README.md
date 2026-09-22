# ArXiv Research Analysis Agent




An automated n8n workflow that monitors [arXiv](https://arxiv.org) for new papers in chosen categories, filters them by keyword, runs a local-LLM "first-principles" analysis on the full paper text, and delivers the results to Telegram — with an interactive yes/no confirmation step before committing to a full analysis.

## What this solves

Keeping up with new arXiv submissions in a niche research area is manual and time-consuming: checking listings daily, judging relevance from abstracts alone, and reading full PDFs to decide if a paper matters. This project automates that loop:

1. Every morning, it checks arXiv's "new submissions" listing for one or more categories (by default General Relativity & Quantum Cosmology `gr-qc` and Quantitative Finance `q-fin`).
2. It filters new papers against a keyword list you define.
3. For each match, it sends you a Telegram message with the title, abstract, and a `yes`/`no` prompt.
4. If you reply `yes <arxiv_id>`, it downloads the full PDF, extracts the text, and asks a local Ollama model to walk through the paper's derivation/methodology from first principles, producing a Markdown report delivered back to you on Telegram.
5. If you reply `no <arxiv_id>`, it marks the paper as skipped and does nothing further.

## Architecture

`arxiv-research-intelligence-workflow.json` — Daily paper monitor + interactive analysis

Two parallel branches (one per arXiv category, e.g. `gr-qc` and `q-fin`), each driven by a daily schedule trigger:

```
Schedule Trigger (daily, e.g. 08:30)
   +---> Build Daily Query          (constructs the arXiv category/date query)
          +---> Query arXiv           (fetches the public "new submissions" listing page)
                +---> Parse + Keyword Gate   (parses the HTML listing, filters by keyword list)
                      +---> Store Pending + Brief  ---> Send Brief (Telegram)   [awaiting yes/no]
```

A second, independently scheduled branch (polling every 1 minute) handles the interactive reply:

```
Schedule Trigger (polling interval)
   +---> Get Telegram Offset (reads offset from ChromaDB)
          +---> Get Telegram Updates (long-poll getUpdates with offset tracking)
                +---> Process Callbacks   (parses "yes <id>" / "no <id>" replies against pending papers)
                      +---> Filter - Toasts ---> Send Reply Ack   (acknowledgement messages)
                      +---> Lookup Pending Paper (gr-qc)  ---> Download Full PDF
                      |     +---> Extract Full PDF Text
                      |           +---> Build First-Principles Prompt
                      |                 +---> Ollama Full-Paper Analysis (local LLM)
                      |                       +---> Create Markdown Report
                      |                             +---> Telegram (sendDocument)
                      +---> Lookup Pending Paper (q-fin)  ---> [identical analysis chain]
```

### State management

Paper state (pending/in-progress/declined) and the Telegram polling offset are persisted in a [ChromaDB](https://www.trychroma.com/) collection, used purely as a metadata key-value store (no vector search involved). This avoids reliability issues with n8n's built-in static data across separate trigger executions.

The ChromaDB collection (`arxiv_pending_papers`) stores:
- **Per-paper records** keyed as `<domain>|<arxiv_id>` with metadata: `domain`, `arxiv_id`, `status`, and `meta_json` (full paper metadata as a JSON string).
- **One offset record** keyed as `system|telegram_offset` tracking the Telegram `getUpdates` offset.

## Technologies used

- [n8n](https://n8n.io) — workflow orchestration (self-hosted)
- [Ollama](https://ollama.com) — local LLM inference for the first-principles paper analysis
- [ChromaDB](https://www.trychroma.com/) — lightweight state persistence for paper status and Telegram offset tracking
- [Telegram Bot API](https://core.telegram.org/bots/api) — notification and interactive yes/no delivery channel
- arXiv public listing pages (`arxiv.org/list/<category>/new`) — no API key required

## Project structure

```
.
├── arxiv-research-intelligence-workflow.json   # n8n workflow (sanitized for public sharing)
├── .env.example                                 # Placeholder environment variables (reference only)
├── .gitignore
└── README.md
```

> Note: n8n workflows configure most values (URLs, model names, credentials) inside the node parameters and n8n's credential store, not via `.env` files. The `.env.example` in this repo is a reference for the values you'll need to have on hand when configuring nodes and credentials manually in the n8n editor — it is not consumed automatically by n8n.

## Setup requirements

- A running **n8n** instance (self-hosted).
- **[Ollama](https://ollama.com)** running and reachable from your n8n instance, with a chat model pulled (the workflow uses `qwen3:4b` — any Ollama chat-capable model works, adjust for your hardware).
- **[ChromaDB](https://www.trychroma.com/)** running and reachable from your n8n instance. The simplest setup is a Docker container:
  ```bash
  docker run -d --name chroma -p 8000:8000 chromadb/chroma
  ```
  The workflow will automatically create the `arxiv_pending_papers` collection on first use via the Chroma REST API.
- A **Telegram bot** and your personal Telegram chat/user ID. To create one: open a chat with [@BotFather](https://t.me/BotFather) on Telegram and send `/newbot`. Follow the prompts to choose a name and a unique username for your bot. BotFather will reply with a Telegram API Key — copy this, you'll need it for setup below. Then message [@userinfobot](https://t.me/userinfobot) to get your own numeric chat/user ID, which the workflow uses to know where to send you messages.

## Credentials required

None of the credentials below are included in this repository. You must create your own in n8n and select them on the relevant nodes after import.

| Credential type | Used by | What you need |
|---|---|---|
| Telegram API | All Telegram nodes (Send Brief, Send Reply Ack, sendDocument) | A Telegram API Key from [@BotFather](https://t.me/BotFather), added as an n8n "Telegram API" credential |

## Configuration required after import

1. **Import the workflow JSON file** into n8n (see below).
2. **Assign your own Telegram credentials** on every Telegram node — the credential `id` fields have been stripped, so n8n will show these as unset. Select or create your own Telegram API credential on each node.
3. **Replace the Telegram chat ID placeholder.** The nodes `GR-QC — Send Brief`, `Q-FIN — Send Brief`, `GR-QC — Telegram`, and `Q-FIN — Telegram` have `chatId` set to `YOUR_TELEGRAM_CHAT_ID`. Replace this with your own numeric Telegram user/chat ID.
4. **Replace the Telegram Bot Token placeholder.** In the `Get Telegram Updates` node, the URL contains `YOUR_TELEGRAM_BOT_TOKEN` — replace it with your actual bot token (keep the `bot` prefix in the URL format: `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates`).
5. **Replace the Ollama URL placeholder.** The nodes `GR-QC — Ollama Full-Paper Analysis` and `Q-FIN — Ollama Full-Paper Analysis` call Ollama via HTTP at `http://YOUR_OLLAMA_HOST/api/chat`. Replace `YOUR_OLLAMA_HOST` with your Ollama instance's host and port (typically `host.docker.internal:11434` for Docker-based n8n, or `localhost:11434` for a native install).
6. **Replace the ChromaDB URL placeholder.** All Code nodes that interact with ChromaDB reference `YOUR_CHROMADB_HOST`. Replace this with your ChromaDB instance's host and port (typically `host.docker.internal:8000` for Docker-based n8n, or `localhost:8000` for a native install).
7. **Review the keyword lists and categories** in the `Build Daily Query` and `Parse + Keyword Gate` code nodes for each domain branch, and adjust the arXiv category codes and keywords to match your research interests.
8. **Review the schedule trigger times** (`Daily 08:30 IST` and the 1-minute polling `Schedule Trigger`) and adjust to your timezone/preferences.
9. The workflow is imported with `active: false`. Activate it from the n8n editor once credentials and configuration are verified.

## How to import the n8n workflow

1. Open your n8n instance.
2. Go to **Workflows > Import from File** (or use the "..." menu > **Import from File** on the workflow list), and select `arxiv-research-intelligence-workflow.json`.
3. Follow the **Configuration required after import** steps above before activating it.

## How to run the automation

- Once activated, the `Daily 08:30 IST`-equivalent trigger runs automatically each day; the second `Schedule Trigger` polls Telegram for your `yes`/`no` replies every minute. No manual execution is needed once configured.
- The workflow can also be run manually from the n8n editor ("Execute workflow") for testing — each manual run will re-check arXiv listings, store any keyword-matched papers as pending, and send the brief to Telegram.

## Security considerations

- **Never commit real credentials.** This repository's workflow JSON file contains no API keys, bot tokens, or credential IDs — all such values were replaced with placeholders (`YOUR_TELEGRAM_BOT_TOKEN`, `YOUR_TELEGRAM_CHAT_ID`, `YOUR_OLLAMA_HOST`, `YOUR_CHROMADB_HOST`) or stripped entirely. You must supply your own via n8n's credential store or by editing the placeholder values directly.
- **Telegram API Key in a URL.** Because the `Get Telegram Updates` node uses a raw HTTP Request rather than the Telegram credential type, your Telegram API Key lives in the node's URL field once configured. Treat your n8n workflow exports and instance backups as sensitive once you've filled in a real key — do not re-export and share this workflow publicly after configuring it with real values.
- **Local-only LLM inference.** This workflow is designed around a self-hosted Ollama instance, so paper content is not sent to a third-party LLM API by default. If you adapt this to use a cloud LLM provider, review what data you are sending externally.
- **ChromaDB access.** The workflow connects to ChromaDB without authentication by default. If your ChromaDB instance is network-accessible, consider adding authentication or restricting access to localhost.
- **This workflow was exported with `active: false`.** Review all node configuration and credentials before activating it against your own Telegram bot and Ollama instance.
