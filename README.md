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

## Technologies used

- [n8n](https://n8n.io) — workflow orchestration (self-hosted)
- [Ollama](https://ollama.com) — local LLM inference for the first-principles paper analysis
- [Telegram Bot API](https://core.telegram.org/bots/api) — notification and interactive yes/no delivery channel
- arXiv public listing pages (`arxiv.org/list/<category>/new`) — no API key required

## Project structure

```
.
├── arxiv-research-intelligence-workflow.json   # Daily monitor + interactive analysis workflow
├── .env.example                                 # Placeholder environment variables (reference only)
├── .gitignore
└── README.md
```

> Note: n8n workflows configure most values (URLs, model names, credentials) inside the node parameters and n8n's credential store, not via `.env` files. The `.env.example` in this repo is a reference for the values you'll need to have on hand when configuring nodes and credentials manually in the n8n editor — it is not consumed automatically by n8n.

## Setup requirements

- A running n8n instance (self-hosted).
- [Ollama](https://ollama.com) running and reachable from your n8n instance, with an analysis/chat model pulled (the original used `qwen3:4b` — any Ollama chat-capable model works, adjust for your hardware).
- A Telegram bot and your personal Telegram chat/user ID. To create one: open a chat with [@BotFather](https://t.me/BotFather) on Telegram and send `/newbot`. Follow the prompts to choose a name and a unique username for your bot. BotFather will reply with a Telegram API Key — copy this, you'll need it for setup below. Then message [@userinfobot](https://t.me/userinfobot) to get your own numeric chat/user ID, which the workflow uses to know where to send you messages.

## Credentials required

None of the credentials below are included in this repository. You must create your own in n8n and select them on the relevant nodes after import.

| Credential type | Used by | What you need |
|---|---|---|
| Telegram API | `arxiv-research-intelligence-workflow.json` (all Telegram nodes) | A Telegram API Key from [@BotFather](https://t.me/BotFather), added as an n8n "Telegram API" credential |
| Ollama API | LLM analysis nodes | The base URL of your running Ollama instance, added as an n8n "Ollama" credential |

## Configuration required after import

1. **Import the workflow JSON file** into n8n (see below).
2. **Assign your own credentials** on every node that references `telegramApi` or `ollamaApi` — the credential `id` fields have been stripped, so n8n will show these as unset. Select or create your own credential of the matching type on each node.
3. **Replace the Telegram chat ID placeholder.** The nodes `GR-QC — Telegram`, `Q-FIN — Telegram`, `GR-QC — Send Brief`, and `Q-FIN — Send Brief` have `chatId` set to `YOUR_TELEGRAM_CHAT_ID`. Replace this with your own numeric Telegram user/chat ID (message [@userinfobot](https://t.me/userinfobot) to find yours).
4. **Replace the Telegram API Key placeholder.** In the `Get Telegram Updates` node, the URL contains a placeholder in place of your Telegram API Key — n8n's Telegram credential type does not cover raw HTTP Request nodes, so this node authenticates via the key embedded directly in the URL. Replace the placeholder with your own Telegram API Key (keep the `bot` prefix in the URL), or better, store it in an n8n credential/variable and reference it via an expression instead of hardcoding it.
5. **Replace the Ollama URL placeholder.** The nodes `GR-QC — Ollama Full-Paper Analysis` and `Q-FIN — Ollama Full-Paper Analysis` call Ollama directly via HTTP Request (`http://YOUR_OLLAMA_URL/api/chat`) rather than through the Ollama credential type. Replace `YOUR_OLLAMA_URL` with your Ollama instance's host and port (for a local Docker-based n8n install talking to Ollama running on the host machine, this is typically `host.docker.internal:11434`; for a native install, `localhost:11434`).
6. **Review the keyword lists and categories** in the `Build Daily Query` and `Parse + Keyword Gate` code nodes for each domain branch, and adjust the arXiv category codes and keywords to match your research interests.
7. **Review the schedule trigger times** (`Daily 08:30 IST` and the polling `Schedule Trigger`) and adjust to your timezone/preferences.
8. The workflow is imported with `active: false`. Activate it from the n8n editor once credentials and configuration are verified.

## How to import the n8n workflow

1. Open your n8n instance.
2. Go to **Workflows → Import from File** (or use the "..." menu → **Import from File** on the workflow list), and select `arxiv-research-intelligence-workflow.json`.
3. Follow the **Configuration required after import** steps above before activating it.

## How to run the automation

- Once activated, the `Daily 08:30 IST`-equivalent trigger runs automatically each day; the second `Schedule Trigger` polls Telegram for your `yes`/`no` replies on its own interval. No manual execution is needed once configured.
- The workflow can also be run manually from the n8n editor ("Execute workflow") for testing.

## Security considerations

- **Never commit real credentials.** This repository's workflow JSON file contains no API keys, bot tokens, or credential IDs — all such values were replaced with placeholders (`YOUR_TELEGRAM_BOT_TOKEN`, `YOUR_TELEGRAM_CHAT_ID`, `YOUR_OLLAMA_URL`) or stripped entirely (credential `id` fields). You must supply your own via n8n's credential store or by editing the placeholder values directly, and should avoid pasting secrets into files that get committed to version control.
- **Telegram API Key in a URL.** Because the `Get Telegram Updates` node uses a raw HTTP Request rather than the Telegram credential type, your Telegram API Key lives in the node's URL field once configured. Treat your n8n workflow exports and instance backups as sensitive once you've filled in a real key — do not re-export and share this workflow publicly after configuring it with real values.
- **Local-only LLM inference.** This workflow is designed around a self-hosted Ollama instance, so paper content is not sent to a third-party LLM API by default. If you adapt this to use a cloud LLM provider, review what data you are sending externally.
- **This workflow was exported with `active: false`.** Review all node configuration and credentials before activating it against your own Telegram bot and Ollama instance.
