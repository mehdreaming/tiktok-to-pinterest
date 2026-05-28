# 🎬 TikTok → Pinterest Automation

> Find viral TikToks, download them watermark-free in HD, generate AI-powered Pinterest copy, and queue everything in Google Sheets for review — all on autopilot.

Built with [n8n](https://n8n.io), runs on free tiers of every service used.

![Status](https://img.shields.io/badge/status-production-brightgreen)
![n8n](https://img.shields.io/badge/n8n-1.x-EA4B71)
![License](https://img.shields.io/badge/license-MIT-blue)

---

## ✨ Features

- 🔍 **Keyword-based discovery** — scrapes TikTok for any niche (anime edits, recipes, fitness, etc.)
- 🎯 **Smart filtering** — only keeps videos with 100K+ views, 100+ shares, 7-60s duration
- 📥 **HD watermark-free downloads** — bypasses TikTok CDN restrictions via tikwm.com
- ☁️ **Auto-upload to Google Drive** — organized in a dedicated folder
- 🤖 **AI-generated Pinterest copy** — titles and descriptions via Groq (Llama 3.3 70B)
- 📊 **Google Sheets review queue** — every pin logged with status tracking
- 🔁 **Deduplication** — never pulls the same video twice across runs
- ⏰ **Dual triggers** — on-demand chat OR scheduled daily auto-runs

---

## 🏗️ Architecture
```text
┌─────────────────────┐     ┌──────────────────────┐
│  Chat Trigger       │     │  Schedule Trigger    │
│  (on-demand)        │     │  (daily 9 AM)        │
└──────────┬──────────┘     └──────────┬───────────┘
           │                           │
           ▼                           ▼
      Edit Fields              Set Schedule Keyword
      (chat input)             (random from list)
           │                           │
           └─────────────┬─────────────┘
                         ▼
              Apify TikTok Scraper
               (50 results / keyword)
                         │
                         ▼
                 Normalize (Code JS)
                         │
                         ▼
        Filter (views/shares/duration)
                         │
                         ▼
          Get Existing URLs (Sheets)
                         │
                         ▼
        Dedup Code (filter known URLs)
                         │
                         ▼
                   Limit (top 3)
                         │
                         ▼
        Get Direct URL (tikwm.com API)
                         │
                         ▼
       HTTP Request1 (HD MP4 download)
                         │
                         ▼
               Google Drive Upload
                         │
                         ▼
    Code in JavaScript1 (merge meta + prompt)
                         │
                         ▼
         AI Agent (Groq llama-3.3-70b)
                         │
                         ▼
              Google Sheets Append
                         │
                         ▼
          Code in JavaScript2 (parse JSON)
```


---

## 🛠️ Tech Stack

| Service | Purpose | Free Tier |
|---------|---------|-----------|
| [n8n](https://n8n.io) | Workflow orchestrator | ✅ Cloud trial / self-host |
| [Apify](https://apify.com) | TikTok scraper (`clockworks/tiktok-scraper`) | ~$5/mo credits |
| [tikwm.com](https://tikwm.com) | TikTok HD video bypass | ✅ Free public API |
| [Google Drive](https://drive.google.com) | Video storage | ✅ 15 GB |
| [Google Sheets](https://sheets.google.com) | Review queue | ✅ Free |
| [Groq](https://groq.com) | AI inference (Llama 3.3 70B) | ✅ Generous free tier |

**Total monthly cost: $0** (within free tier limits).

---

## 🚀 Setup

### 1. Prerequisites

- An n8n instance (self-hosted via Docker, or [n8n Cloud](https://n8n.io))
- Accounts on: Apify, Google Cloud (Drive + Sheets APIs enabled), Groq
- A Google Sheet named `Pinterest Queue` with columns:
  `timestamp | keyword | tiktok_url | views | shares | duration | drive_file_id | drive_url | pin_title | pin_description | status`

### 2. Credentials in n8n

Create these under **Credentials → New**:

| Credential | Type | Notes |
|-----------|------|-------|
| `Apify` | HTTP Header Auth | Header: `Authorization`, Value: `Bearer YOUR_APIFY_TOKEN` |
| `Google Drive OAuth2` | Google Drive OAuth2 | Use your GCP OAuth client |
| `Google Sheets OAuth2` | Google Sheets OAuth2 | Same OAuth client, separate credential |
| `Groq` | HTTP Header Auth or Groq node | Header: `Authorization`, Value: `Bearer YOUR_GROQ_KEY` |

### 3. Import the workflow

1. Download `TikTok → Pinterest.json` from this repo.
2. In n8n: **Workflows → Import from File** → select `TikTok → Pinterest.json`.
3. Open each node with a 🔑 icon and assign your credentials.
4. Update the Google Sheets nodes to point to YOUR `Pinterest Queue` sheet.
5. Update the Drive Upload node's parent folder to your `TikTok Pins` folder.

### 4. Configure scheduled keywords

Open the **Set Schedule Keyword** node and edit the keyword list to your niche:
['naruto edit', 'demon slayer edit', 'one piece edit', 'jujutsu kaisen edit'][Math.floor(Math.random() * 4)] 

### 5. Test both triggers

- **Chat:** open the Chat panel, type a keyword (e.g. `naruto edit`), press Enter. Wait ~90s.
- **Schedule:** click the Schedule Trigger node → **Execute step**. Wait ~90s.

Both should produce 3 new rows in your sheet + 3 MP4s in Drive.

### 6. Activate

Toggle the workflow **Active** (top-right) for daily auto-runs at 9 AM.

---

## ⚙️ Configuration

### Filter thresholds (Filter node)

Adjust to fit your niche's volume:
views     ≥  100000

shares    ≥  100

duration  ≥  7

duration  ≤  60

### Schedule time

Default: 9 AM Asia/Calcutta. Change in the Schedule Trigger node.

### AI model

Default: `llama-3.3-70b-versatile` via Groq. Swap in any OpenRouter / OpenAI model by editing the AI Agent → Chat Model sub-node.

---

## 🐛 Common Issues & Fixes

| Problem | Cause | Fix |
|---------|-------|-----|
| `403 Forbidden` on download | TikTok CDN blocks bare requests | Use tikwm.com bypass (already configured) |
| `Edit Fields hasn't been executed` error | Code node referenced wrong trigger path | Wrapped in try/catch with fallback to `Set Schedule Keyword` |
| `Data limit reached` (AI) | Free model token cap hit | Switch model or add a `Wait` node between items |
| Low quality video (~3 MB) | Using `data.play` instead of HD | URL: ` $json.data.hdplay \|\| $json.data.play ` |
| Apify timeout | Sync endpoint limited to ~50 results | Use `run-sync-get-dataset-items` (already configured) |
| Drive upload "File not found" | Binary property mismatch | Ensure HTTP node Response Format = `File`, Property = `data` |
| Same videos repeating across runs | No dedup | Get Existing URLs + Dedup Code (already configured) |

---

## 📁 Repo Structure
.

├── [README.md](http://README.md)              # This file

├── TikTok → Pinterest.json          # Exportable n8n workflow

├── docs/

│   ├── [setup-guide.md](http://setup-guide.md)     # Full setup walkthrough

│   └── screenshots/       # Canvas + node screenshots

└── .env.example           # Template for env vars

---

## 🗺️ Roadmap

- [ ] Auto-post approved rows to Pinterest via Pinterest API
- [ ] AI cover image generation for higher CTR pins
- [ ] Vertical thumbnail generation with ffmpeg
- [ ] Telegram / Slack notification on new rows
- [ ] Multi-niche parallel runs (one workflow → multiple sheets)
- [ ] Web dashboard for approve / skip / edit

---

## 🤝 Contributing

PRs welcome! Especially for:
- Adding new content sources (YouTube Shorts, Instagram Reels)
- Output to additional platforms (Twitter, Threads, Tumblr)
- Smarter dedup (semantic similarity, not just URL match)

---

## 📜 License

MIT — see [LICENSE](LICENSE).

Use at your own risk. Respect TikTok creators by crediting them, and follow Pinterest's content guidelines when posting.

---

## 🙏 Credits

- [n8n](https://n8n.io) — the orchestrator that makes this possible
- [Apify](https://apify.com) — reliable TikTok scraping infrastructure
- [tikwm.com](https://tikwm.com) — free TikTok HD video API
- [Groq](https://groq.com) — blazing-fast LLM inference

Built with ☕ and a lot of trial & error.
