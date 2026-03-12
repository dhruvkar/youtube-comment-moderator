---
name: youtube-comment-moderator
description: >
  AI-powered YouTube comment moderation. Fetches comments, classifies them
  (spam, question, praise, hate, neutral, constructive), drafts replies,
  and deletes spam. Works with any YouTube channel via OAuth.
  Use when moderating YouTube comments, setting up auto-replies, analyzing
  comment sentiment, managing spam, or building a comment moderation pipeline.
---

# YouTube Comment Moderator

Classify, reply to, and delete YouTube comments automatically using Gemini Flash + YouTube Data API v3.

## Requirements

Environment variables in `.env`:
- `YOUTUBE_API_KEY` — YouTube Data API v3 key (free, 10K units/day)
- `GEMINI_API_KEY` — Gemini Flash for classification (~$0.50-2/day per 500 comments)

For write operations (reply/delete), also set:
- `YT_MOD_CLIENT_ID` — Google OAuth client ID
- `YT_MOD_CLIENT_SECRET` — Google OAuth client secret

See [references/oauth-setup.md](references/oauth-setup.md) for step-by-step OAuth setup (10 min, free).

Optional env overrides:
- `YT_MOD_DB` — SQLite DB path (default: `data/youtube-moderator/moderator.db`)
- `YT_MOD_OAUTH` — OAuth token path (default: `skills/youtube-moderator/oauth.json`)
- `YT_MOD_CONFIG` — Config path (default: `skills/youtube-moderator/config.json`)

## Quick Start

```bash
# 1. Interactive setup (channel, mode, voice, FAQ, OAuth)
python3 scripts/setup.py

# 2. Dry run on a video
python3 scripts/moderate.py --video-id VIDEO_ID --dry-run

# 3. Moderate channel (uses config.json defaults)
python3 scripts/moderate.py

# 4. Check stats
python3 scripts/moderate.py --stats

# 5. Review approval queue
python3 scripts/moderate.py --queue

# 6. Approve pending items
python3 scripts/moderate.py --approve
```

## Scripts

| Script | Purpose |
|--------|---------|
| `setup.py` | Interactive wizard: channel, API keys, OAuth, voice style, FAQ |
| `moderate.py` | Main pipeline: fetch → classify → act. Also: `--stats`, `--queue`, `--approve` |
| `fetch_comments.py` | Standalone fetcher (API key only, no OAuth) |
| `classify_comments.py` | Standalone classifier (reads JSON, writes JSON) |
| `db.py` | SQLite persistence layer (imported by moderate.py) |

## Setup Flow

1. User creates Google Cloud project + enables YouTube Data API ([detailed guide](references/oauth-setup.md))
2. User sets `YOUTUBE_API_KEY`, `YT_MOD_CLIENT_ID`, `YT_MOD_CLIENT_SECRET` in `.env`
3. Run `python3 scripts/setup.py` — interactive wizard handles OAuth, channel config, voice/FAQ
4. Run `python3 scripts/moderate.py --dry-run` to verify
5. Switch to live: `python3 scripts/moderate.py` or schedule via cron

### Agent-Driven Setup (VPS / Telegram / Non-Interactive)

When the user can't run the interactive wizard (e.g. OpenClaw on a VPS, chatting via Telegram):

```bash
# 1. Generate OAuth URL — send this to the user to open in their browser
python3 scripts/setup.py --auth-url

# 2. User opens URL, clicks Allow, browser redirects to localhost (won't load).
#    User copies the FULL URL from their browser address bar and pastes it back in chat.

# 3. Exchange the code for tokens
python3 scripts/setup.py --exchange-code "http://127.0.0.1:8976/callback?code=4/0A..."

# 4. Create config
python3 scripts/setup.py --create-config --channel-id UC... --channel-name "Name" --mode approval --voice "friendly, casual"
```

No sensitive info needs to be pasted in chat. The callback URL contains a one-time auth code
(not a token), and it expires within minutes.

## Modes

- **monitor** — classify + report only (no OAuth needed)
- **approval** — drafts replies and queues deletions for human review
- **auto** — auto-replies to questions, auto-deletes spam/hate

## Classification Categories

| Category | Action | Description |
|----------|--------|-------------|
| spam | delete | Promotional links, scam offers, bot text, self-promo |
| question | reply | Genuine question about video/creator/topic |
| praise | thank | Positive feedback, compliments |
| hate | delete | Hateful, abusive, harassing content |
| neutral | skip | Generic reactions, timestamps, "lol" |
| constructive | flag | Thoughtful criticism, corrections, suggestions |

## Architecture

- **State:** SQLite (one DB per installation, all channels)
- **Read pipeline:** YouTube Data API v3 (API key, no OAuth)
- **Classification:** Gemini 2.0 Flash (batches of 50, ~$0.001/comment)
- **Write pipeline:** YouTube Data API v3 (OAuth per channel)
- **Deduplication:** comment_id is primary key; already-classified comments are skipped

## Channel Config

Stored in `config.json` (created by setup.py):

```json
{
  "channel_id": "UC...",
  "channel_name": "My Channel",
  "mode": "approval",
  "auto_delete_spam": true,
  "auto_reply_questions": false,
  "voice_style": "friendly, casual, helpful",
  "faq": [{"q": "What camera?", "a": "Sony A7IV"}]
}
```

## Cron Usage

Run every 15-30 min for near-realtime moderation:

```bash
python3 scripts/moderate.py --max-videos 3 --max-comments 100
```

For approval mode, the agent checks `--queue` and presents items for review.
