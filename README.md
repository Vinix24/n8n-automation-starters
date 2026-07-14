# n8n Automation Starters

Thirteen working n8n workflows for LinkedIn / Telegram / Supabase automation. These are the workflows I ran to power a personal intelligence and outreach cockpit before I migrated the core to my own Python orchestrator. They are a working starting point to fork and adapt, not a production-grade system.

Everything here is sanitized: no chat IDs, no personal Supabase project, no hardcoded LinkedIn URL, no tokens. Every secret is a `$env.*` reference. Two overly-coupled workflows were left out rather than shipped half-working (see [What I left out](#what-i-left-out)).

License: MIT. Do whatever you want with it.

## What these workflows do

They all sit on three building blocks:

- **Telegram** for commands and notifications (a bot you create via [@BotFather](https://t.me/BotFather)).
- **Supabase / Postgres** for storage, over the REST API, so no direct database credentials live in n8n.
- **Apify** for LinkedIn scraping, through public marketplace actors.

A few workflows also call a Python module of your own that is not shipped here. That is called out explicitly per workflow, and in full in [WORKFLOWS.md](WORKFLOWS.md).

| Workflow | Trigger | What it does |
|---|---|---|
| `01-morning-brief.json` | Cron 07:00 | A daily Telegram digest: content planned for today, hot leads, open comment suggestions, your idea inbox. |
| `13-callback-handler.json` | Webhook | Catches Telegram inline-button taps (pick an ICP label for a followed profile, save a topic as a keyword) and writes the result back to Supabase. |
| `13-healthcheck-channels.json` | Cron 15 min | Pings Supabase, an optional local wrapper, and n8n itself; sends a Telegram alert the moment something is down. |
| `14-track-relay.json` | Webhook (`/track`) | `/track <linkedin-url>` scrapes a post plus its comments via Apify, detects your own replies (via `OWN_LINKEDIN_URL`), and stores everything structured in Supabase. |
| `15-follow-relay.json` | Webhook (`/follow`) | `/follow <linkedin-profile>` scrapes the profile via Apify, adds it to your watchlist, and shows an inline keyboard to pick an ICP label right away. |
| `15-self-scout.json` | Cron weekly | A weekly Apify scrape of your own posts and comments, for your own content analytics (impressions, reactions, engagement). |
| `16-watchlist-daily-scrape.json` | Cron daily | Scrapes your watchlist of followed profiles. **Contains a TODO-placeholder command node**, see [Python dependency](#python-dependency-read-this-first). |
| `17-author-memory-cron.json` | Cron hourly | Aggregates your own reply history per profile (how often you replied, average engagement). Pure Supabase, no external dependency. |
| `18-dig-relay.json` | Webhook (`/dig`) | `/dig <topic>` runs an on-demand Apify scout on a topic, with a monthly-budget guard before the scrape starts, and shows the top five results in Telegram. |
| `21-question-clustering-weekly.json` | Cron Sunday 12:00 | Clusters questions found in comments into content ideas. **Contains a TODO-placeholder command node.** |
| `22-task-lease-reaper.json` | Cron 5 min | Releases expired task leases via a Postgres RPC. A generic pattern for any task queue with lease/lock semantics. |
| `chain-2a-classifier-writer-pipeline.json` | Webhook | Reference architecture for an AI comment pipeline: budget preflight, ingest, author-cooldown check, classify, write a draft reply, update status, with a separate error branch. **Five TODO-placeholder command nodes.** |
| `dashboard-write-relay.json` | Webhook | A generic, API-key-protected webhook that relays insert/update/delete to Supabase and can optionally start an Apify actor. A write layer between a dashboard/frontend and your database, with no database credentials in the frontend. |

Full per-workflow documentation, including every table each one touches and how to adapt it, is in [WORKFLOWS.md](WORKFLOWS.md).

## Python dependency (read this first)

Three workflows are **templates**. Their `n8n-nodes-base.executeCommand` nodes no longer contain a working Python call, only a `# TODO: ...` placeholder that documents the expected stdout contract (which JSON keys the downstream nodes read):

- `16-watchlist-daily-scrape.json` expects `{ "scraped": <n> }`
- `21-question-clustering-weekly.json` expects `comments_fetched`, `questions_substantive`, `cluster_count`, `ideas_inserted`, `correlation_id`
- `chain-2a-classifier-writer-pipeline.json` has five placeholders (cost preflight, ingester, cooldown check, classifier, writer), so almost every step

These workflows show you the **orchestration shape** (cron/webhook, guard rails, job, notification), not a drop-in working script. Replace each `# TODO: ...` placeholder with your own script, API call, or a simple HTTP request to a microservice of your own. As long as your replacement prints the same JSON keys on stdout that the placeholder comment lists, the rest of the workflow (budget checks, error handling, Telegram notifications, Supabase writes) keeps working unchanged.

The other ten workflows are self-contained: Telegram plus Supabase plus Apify, no hidden dependency.

## Setup

1. **Run n8n**, locally (`docker run -it --rm -p 5678:5678 n8nio/n8n`) or self-hosted.
2. **Create credentials.** `credentials.json.example` shows the n8n credentials you need (MC API Key, Supabase Service Role, Apify Token) plus a separate **Telegram** credential. The Telegram credential is not in the JSON file, because n8n's Telegram credential type takes a bot token through the UI. Create a bot via [@BotFather](https://t.me/BotFather) and add the credential as `VNX Telegram Bot`, or rename the credential references in the workflows to your own name.
3. **Set env vars.** Copy `.env.example`, fill it in, and load it into n8n (`N8N_ENV_FILE`, docker `--env-file`, or your host's env-var settings).
4. **Import workflows.** Import each file from `workflows/` individually (Import from File). Attach the right credential to the Telegram and HTTP Request nodes per workflow.
5. **Create tables.** The workflows expect tables such as `captured_posts`, `comment_history`, `scraped_comments`, `creator_watchlist`, `monitoring_keywords`, `apify_usage`, and an events table (`system.events`, `events`, or `system_events` depending on the workflow, an artifact of how this grew organically). Adjust table and column names to your own schema. This is illustrative, not a fixed contract. WORKFLOWS.md lists the tables per workflow.
6. **Test.** Start with `13-healthcheck-channels.json` and `01-morning-brief.json`. They have the fewest moving parts and confirm your Supabase and Telegram credentials work.

## Why I moved to Python

These n8n workflows did their job, but a 100-plus-node Telegram router on a visual canvas eventually becomes unmanageable: no node-level version control, no real tests, and an n8n restart that can silently break a workflow with a subtle node error. That happened to me at the end of April 2026, ten hours with the bot down. I moved the core to my own open-source, governance-first orchestrator: [VNX Orchestration](https://github.com/Vinix24/vnx-orchestration). No vendor lock-in on a visual workflow tool, full NDJSON receipts per step, and testable Python modules. This repo is the n8n layer that came before it, a good starting point if you would rather begin with a visual tool than with code.

## What I left out

Two of the fifteen source workflows are not in this package:

- **`03-telegram-router.json`** (110 nodes), the central command router that handled my whole bot command set (`/linkedin`, `/keyword`, `/status`, language detection, pause/resume, and so on). It was too tightly coupled to my own command surface to serve as a generic starter, and it carried a hardcoded Supabase project reference deep inside a few Code nodes, a sign that sanitizing it in isolation was more risk than value. If you want to build a Telegram command router, look at the webhook-plus-switch pattern in `13-callback-handler.json` and `18-dig-relay.json` and build your own command set on that basis.
- **`16-leads-from-own-post-comments.json`**, which detects leads under your own LinkedIn posts but calls a local Python webhook (`http://localhost:8765/leads/process`) that is not in this package. Without that classifier the workflow is a dead end. Combine `14-track-relay.json` (scrape plus own-comment detection) with your own classification logic instead.

## What I sanitized

- A hardcoded Tailscale IP in `13-healthcheck-channels.json` became the env var `LOCAL_CLAUDE_WRAPPER_URL` with a placeholder fallback.
- A hardcoded personal LinkedIn profile URL in `14-track-relay.json` (two places) and `15-self-scout.json` (one place) became the env var `OWN_LINKEDIN_URL`, with no fallback to my own profile. The internal field names `vincent_comments` / `vincent_comment_found` in `14-track-relay.json` were renamed to `own_comments` / `own_comment_found`.
- All thirteen included workflows were checked for hardcoded chat IDs, Supabase URLs and project refs, tokens, webhook secrets, and email addresses. They all use `$env.*` references, no literals.
- Internal project-name references (n8n `instanceId`, credential descriptions, one Telegram message string, one `source` filter value, and a personal name in a code comment) were removed or renamed to generic terms.

## License

MIT, see [LICENSE](LICENSE). Do whatever you want with it.
