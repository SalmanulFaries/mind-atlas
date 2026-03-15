# Mind Atlas — Self-Hosted Knowledge Capture System

A personal automation pipeline that captures, enriches, and organises everything you consume online — articles, YouTube videos, LinkedIn posts, Instagram reels, Twitter/X threads, and plain text — into a structured Notion database. Built entirely on self-hosted, open-source, and free-tier infrastructure.

---

## What It Does

Share any URL or text to a Telegram bot. Within seconds, the pipeline:

1. Detects the source platform (YouTube, Instagram, LinkedIn, Twitter/X, Article, Text)
2. Fetches full metadata using the YouTube Data API, Microlink, and Jina AI Reader
3. Passes the content to Groq's Llama 3.3 70B to generate a clean summary, topic tags, and content type
4. Creates a structured entry in a Notion database with title, URL, source, summary, tags, status, and save date
5. Sends a confirmation back via Telegram

Every Friday at 6:00 PM IST, a second workflow queries the full database and delivers a weekly intelligence report — items by status, new additions, overdue content, top tags, source breakdown, and completion rate — to Telegram and Notion.

---

## Architecture

```
Telegram Bot (capture endpoint)
        │
        ▼
n8n (self-hosted on AWS EC2)
        │
        ├── YouTube URL ──► YouTube Data API v3 ──► Groq ──► Notion
        │
        ├── Other URL ───► Microlink (metadata) ──► Jina AI (full text) ──► Groq ──► Notion
        │
        └── Plain Text ──► Groq ──────────────────────────────────────────► Notion
                                                                              │
                                                                              ▼
                                                                    Telegram confirmation

Weekly Report Workflow:
Schedule Trigger (Friday 6PM IST) ──► Notion query ──► Metrics Calculation ──► Notion Reports ──► Telegram
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Infrastructure | AWS EC2 + Docker |
| Automation engine | n8n (self-hosted) |
| Capture endpoint | Telegram Bot |
| Video metadata | YouTube Data API v3 |
| URL metadata | Microlink.io |
| Full text extraction | Jina AI Reader |
| AI enrichment | Groq — Llama 3.3 70B |
| Knowledge storage | Notion |
| Security | Nginx + Certbot (SSL) |

---

## Workflows

This repository contains two n8n workflow files:

| File | Description |
|---|---|
| `Knowledge_Capture_Pipeline_Public.json` | Main capture pipeline — handles all incoming Telegram messages |
| `Mind_Atlas_Weekly_Report_Public.json` | Scheduled weekly report — fires every Friday at 6 PM IST |

---

## Prerequisites

Before importing the workflows, set up the following:

- **n8n** — self-hosted instance with a public HTTPS URL (required for Telegram webhooks)
- **Telegram Bot** — create via [@BotFather](https://t.me/BotFather), note the bot token and your chat ID
- **Notion** — create two databases (Mind Atlas and Mind Atlas Reports) and an integration with access to both
- **Groq** — free API key from [console.groq.com](https://console.groq.com)
- **YouTube Data API v3** — free API key from [Google Cloud Console](https://console.cloud.google.com)

---

## Notion Database Schema

### Mind Atlas (Knowledge Capture)

| Property | Type |
|---|---|
| Title | Title |
| URL | URL |
| Source | Select |
| Tags | Multi-select |
| Summary | Text |
| Saved On | Date |
| Status | Select |
| Notes | Text |

**Source options:** YouTube, Instagram, LinkedIn, Twitter/X, Article, Text, WhatsApp

**Status options:** Unread, In Progress, Done

---

### Mind Atlas Reports (Weekly Report)

| Property | Type |
|---|---|
| Title | Title |
| Week Of | Date |
| Total Items | Number |
| New This Week | Number |
| Unread | Number |
| In Progress | Number |
| Done | Number |
| Completion Rate | Number |
| Overdue Items | Number |
| Top Tags | Text |
| Overdue Details | Text |
| New Items Details | Text |
| YouTube Count | Number |
| Instagram Count | Number |
| LinkedIn Count | Number |
| X Count | Number |
| Article Count | Number |
| Text Count | Number |

---

## Setup Instructions

### Step 1 — Import the workflows into n8n

1. Open your n8n instance
2. Click **Add workflow** → **Import from file**
3. Import `Knowledge_Capture_Pipeline_Public.json`
4. Repeat for `Mind_Atlas_Weekly_Report_Public.json`

### Step 2 — Configure credentials

In n8n, go to **Credentials** and create the following:

- **Telegram API** — paste your bot token
- **Notion API** — paste your internal integration secret

### Step 3 — Replace placeholders

Search for and replace all placeholders in both workflows with your actual values:

| Placeholder | Replace With |
|---|---|
| `[YOUR_TELEGRAM_BOT_TOKEN]` | Your Telegram bot token |
| `[YOUR_TELEGRAM_CHAT_ID]` | Your Telegram chat ID |
| `[YOUR_NOTION_DATABASE_ID]` | Mind Atlas database ID |
| `[YOUR_NOTION_REPORT_DATABASE_ID]` | Mind Atlas Reports database ID |
| `[YOUR_NOTION_CREDENTIAL_ID]` | n8n Notion credential ID |
| `[YOUR_TELEGRAM_CREDENTIAL_ID]` | n8n Telegram credential ID |
| `[YOUR_GROQ_API_KEY]` | Groq API key |
| `[YOUR_YOUTUBE_API_KEY]` | YouTube Data API v3 key |
| `[YOUR_WEBHOOK_ID]` | Auto-generated by n8n after import |
| `[YOUR_N8N_INSTANCE_ID]` | Your n8n instance ID |
| `[YOUR_WORKFLOW_ID]` | Auto-generated by n8n after import |

### Step 4 — Connect Notion databases

1. Open each database in Notion
2. Click `...` → **Connections** → select your n8n integration
3. Confirm access

### Step 5 — Activate both workflows

Toggle both workflows from **Inactive** to **Active** in n8n.

### Step 6 — Test

Send a YouTube URL to your Telegram bot. Within a few seconds you should receive a confirmation message and see a new row in your Mind Atlas database.

---

## How the URL Classification Works

The pipeline uses a character-count heuristic to distinguish URL-only messages from text messages that contain embedded links:

- If the message text after removing the URL is **fewer than 600 characters** → treated as a URL capture → routed to YouTube or microlink branch
- If the remaining text is **600 characters or more** → treated as plain text → routed to the text branch with Groq generating a title and summary from the full message

Adjust the threshold in the **Code** node (URL extractor) to suit your usage pattern.

---

## AI Enrichment Details

All three branches use **Groq's Llama 3.3 70B** (free tier) for content enrichment.

**YouTube and microlink branches** generate:
- `summary` — 2–8 sentence description of the content
- `tags` — 2–4 lowercase topic tags
- `content_type` — one of: tutorial, news, tool, concept, case-study, opinion

**Text branch** generates:
- `title` — 5–7 word descriptive title
- `summary` — 3–8 sentence comprehensive summary
- `tags` — 2–4 lowercase topic tags
- `content_type` — one of: note, article, tutorial, news, concept, opinion

For Article, LinkedIn, and Twitter/X URLs, **Jina AI Reader** (`r.jina.ai`) fetches the full page content before passing it to Groq — producing significantly richer summaries than OpenGraph metadata alone.

---

## Weekly Report Details

The report workflow fires every **Friday at 12:30 UTC (6:00 PM IST)**. To change the schedule, edit the Schedule Trigger node.

Metrics calculated:

- **Items by status** — cumulative count of Unread, In Progress, Done
- **New this week** — items saved in the last 7 days
- **Overdue items** — Unread items saved more than 14 days ago
- **Completion rate** — percentage of all items marked Done
- **Top tags** — five most frequent tags across the full database with counts
- **Sources this week** — breakdown by platform for the last 7 days

---

## Known Limitations

- **Instagram URLs** — share links from the Instagram app sometimes send app deep links rather than clean web URLs, which may produce incomplete metadata
- **Paywalled articles** — Jina AI Reader cannot extract content behind paywalls; only the title is captured in these cases
- **Twitter/X metadata** — OpenGraph data from Twitter/X is inconsistent post-2023 API changes; Jina Reader is used as a fallback for better extraction
- **Groq rate limits** — free tier allows 30 requests per minute; at high capture volumes you may hit this limit temporarily

---

## Infrastructure Cost

Running this on AWS EC2 t3.micro (Mumbai region):

- **First 12 months** — free tier eligible ($0)
- **After free tier** — ~$8–10/month on-demand, ~$3–4/month with a 1-year Reserved Instance

All APIs (Groq, YouTube Data API, Jina AI, Microlink) are used within free tier limits at typical personal usage volumes.

---

## Built By

**Salmanul Faries**

[LinkedIn](https://www.linkedin.com/in/salmanul-faries/) · [GitHub](https://github.com/SalmanulFaries/)
