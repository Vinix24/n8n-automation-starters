# Workflow reference

Per-workflow documentation: what it does, what it needs, which tables and services it touches, and what you can do with it once you adapt it. For the high-level overview and setup, see [README.md](README.md).

Table names below are the ones the workflows actually reference. Some are schema-qualified (`system.events`, `content.drafts`, `crm.contacts`). Treat all of them as illustrative. Rename them to your own schema. The workflow logic does not care what the tables are called, only that the columns the nodes read exist.

The Apify actors referenced are public marketplace actors:

- [`johnvc/linkedin-posts-api`](https://apify.com/johnvc/linkedin-posts-api)
- [`johnvc/linkedin-post-search-api`](https://apify.com/johnvc/linkedin-post-search-api)
- [`johnvc/linkedin-profile-api`](https://apify.com/johnvc/linkedin-profile-api)

Swap them for whichever actor you prefer. Each workflow reads a small, predictable slice of the actor output, documented per workflow below.

---

## `01-morning-brief.json` — Morning Brief

**Trigger:** cron, daily at 07:00.

**What it does:** builds one Telegram message that summarizes your day. It reads four sources in parallel, merges them, formats a single digest, and sends it to your chat: content drafts scheduled for today, hot CRM contacts, open comment suggestions, and your idea inbox.

**Tables:** `content.drafts`, `crm.contacts`, `comment_queue`, `intelligence.content_ideas`, `system.executions`.

**What you can do with it:** this is the cleanest template for any scheduled digest. Point the four read nodes at your own tables (open orders, support tickets, calendar, whatever), keep the merge-plus-format-plus-send tail, and you have a daily standup in Telegram. It has no external dependency beyond Supabase and Telegram, so it is a good first workflow to get running.

---

## `13-callback-handler.json` — Callback Handler

**Trigger:** webhook (Telegram sets this as its callback URL).

**What it does:** handles taps on Telegram inline buttons. A `switch` node routes on the callback payload: choosing an ICP label for a followed profile patches `creator_watchlist`; saving a topic as a keyword patches `monitoring_keywords`. It then answers the callback so the button stops spinning and edits the message to confirm.

**Tables:** `creator_watchlist`, `monitoring_keywords`.

**What you can do with it:** this is the pattern for any interactive Telegram bot. Inline buttons plus a webhook plus a `switch` gives you a menu-driven interface without a frontend. Add cases to the switch for your own button actions.

---

## `13-healthcheck-channels.json` — Healthcheck Channels

**Trigger:** cron, every 15 minutes.

**What it does:** probes three channels: a Supabase read (list one row from `monitoring_keywords`), an optional local wrapper service (`LOCAL_CLAUDE_WRAPPER_URL`), and an events write to confirm Supabase accepts writes. Each probe normalizes to `{ ok, detail }`. If any channel is down it sends a Telegram alert; if all are healthy it stays silent.

**Tables:** `monitoring_keywords` (read), `events` (write probe).

**What you can do with it:** a zero-infrastructure uptime monitor. Add or remove probe branches for your own services (an API, a queue, a third-party endpoint). The normalize-then-merge-then-alert-on-failure shape stays the same. The local-wrapper probe is optional; leave `LOCAL_CLAUDE_WRAPPER_URL` empty and it fails gracefully.

---

## `14-track-relay.json` — Track Relay (`/track`)

**Trigger:** webhook, `/track <linkedin-url>`.

**What it does:** scrapes a LinkedIn post and its visible comments through the Apify posts actor, then walks the comments to detect your own replies by matching against `OWN_LINKEDIN_URL`. It stores the post in `captured_posts`, the comments in `scraped_comments`, and your own comments in `comment_history`, then confirms in Telegram. It responds to the webhook immediately and does the scrape asynchronously so Telegram does not time out.

**Dependencies:** Apify actor `johnvc/linkedin-posts-api` (`postUrls`).

**Tables:** `captured_posts`, `scraped_comments`, `comment_history`, `system.events`.

**What you can do with it:** capture any post plus its comment thread into structured storage on a single command. The own-comment detection (match author URL against an env var) is a small, reusable trick for any "which of these did I write" problem.

---

## `15-follow-relay.json` — Follow Relay (`/follow`)

**Trigger:** webhook, `/follow <linkedin-profile>`.

**What it does:** scrapes a profile through the Apify profile actor, adds it to `creator_watchlist`, and replies in Telegram with an inline keyboard so you can label the profile's ICP fit immediately. The label tap is then handled by `13-callback-handler.json`.

**Dependencies:** Apify actor `johnvc/linkedin-profile-api` (`profileUrls`).

**Tables:** `creator_watchlist`, `system.events`.

**What you can do with it:** a one-command "add to watchlist and classify" flow. Pairs with the callback handler to give you capture-then-label in two taps. Repoint it at any list you curate manually.

---

## `15-self-scout.json` — Self-Scout Weekly

**Trigger:** cron, weekly.

**What it does:** scrapes your own recent posts and comments through Apify and writes analytics (impressions, reactions, engagement) to `own_post_analytics`, plus your own comments to `comment_history`. It records the Apify run cost in `apify_usage`.

**Dependencies:** Apify actor `johnvc/linkedin-posts-api` (`profileUrls`). Uses `OWN_LINKEDIN_URL`.

**Tables:** `own_post_analytics`, `comment_history`, `apify_usage`.

**What you can do with it:** a weekly self-analytics pull, so you have your own engagement numbers in your own database instead of only inside LinkedIn. The `apify_usage` write is a simple pattern for tracking scrape spend over time.

---

## `16-watchlist-daily-scrape.json` — Watchlist Daily Scrape

**Trigger:** cron, daily.

**What it does:** runs a daily scrape of the profiles on your watchlist. **This is a template.** The `executeCommand` node is a `# TODO` placeholder, not a working call. It documents the expected stdout contract: your scraper should print `{ "scraped": <n> }`. The workflow then reports that count to Telegram.

**Dependencies:** your own scraper script (not shipped). Expected stdout: `{ "scraped": <n> }`.

**What you can do with it:** wrap any batch job (a scraper, an importer, a sync) in a cron-plus-notify shell. Replace the placeholder with your script or an HTTP call to a service of your own; as long as it prints `scraped`, the notification tail works unchanged.

---

## `17-author-memory-cron.json` — Author Memory Hourly Aggregate

**Trigger:** cron, hourly.

**What it does:** aggregates your reply history per profile: how often you have replied to each author and your average engagement with them. Reads from `comment_history`, writes the rollup to `author_interactions`. Pure Supabase, no external service.

**Tables:** `comment_history` (read), `author_interactions` (write), `events`.

**What you can do with it:** a recurring rollup pattern. Any "aggregate raw rows into a per-entity summary on a schedule" job fits this shape. No Apify, no Python, so it is safe to run often.

---

## `18-dig-relay.json` — Dig Relay (`/dig`)

**Trigger:** webhook, `/dig <topic>`.

**What it does:** an on-demand topic scout with a spend guard. Before scraping, it calls the `apify_mtd_spend` RPC to get month-to-date Apify spend and checks it against a monthly cap; if you are over budget it refuses and tells you in Telegram. Otherwise it runs the scrape, records the cost in `apify_usage`, and returns the top five results.

**Dependencies:** Apify actor `johnvc/linkedin-post-search-api` (`keywords`). Postgres RPC `apify_mtd_spend` (returns month-to-date spend).

**Tables:** `apify_usage`, `monitoring_keywords`, `system.events`. RPC: `apify_mtd_spend`.

**What you can do with it:** the budget-guard-before-expensive-call pattern is the reusable part. Any workflow that spends money per run (a paid API, an LLM call, a scrape) should gate on a spend check first. Lift the preflight and put it in front of your own costly node.

---

## `21-question-clustering-weekly.json` — Question Clustering Weekly

**Trigger:** cron, Sunday 12:00 UTC.

**What it does:** clusters questions found in comments into content ideas. **This is a template.** The `executeCommand` node is a `# TODO` placeholder. It documents the expected stdout contract: `comments_fetched`, `questions_substantive`, `cluster_count`, `ideas_inserted`, `correlation_id`. The workflow reports those numbers and branches on whether anything was inserted.

**Dependencies:** your own clustering pipeline (not shipped). Expected stdout keys listed above.

**Tables:** `events`.

**What you can do with it:** a scheduled "run an analysis job, report its summary numbers" shell. Any weekly batch that produces a handful of metrics fits. Replace the placeholder with your pipeline; keep the report-and-branch tail.

---

## `22-task-lease-reaper.json` — Task Lease Reaper

**Trigger:** cron, every 5 minutes.

**What it does:** releases expired task leases by calling the `release_expired_task_leases` Postgres RPC. If the RPC reports rows released it logs an event; otherwise it stays quiet.

**Dependencies:** Postgres RPC `release_expired_task_leases`.

**Tables:** `tasks`, `events`. RPC: `release_expired_task_leases`.

**What you can do with it:** if you build any task queue with lease/lock semantics (worker claims a task, must renew or it expires), this is the reaper that frees crashed-worker tasks back to the queue. Generic to any lease-based queue, not LinkedIn-specific at all.

---

## `chain-2a-classifier-writer-pipeline.json` — Classifier to Writer Pipeline

**Trigger:** webhook (paste a post URL).

**What it does:** a reference architecture for an AI comment pipeline, end to end. Auth check (401 on bad key), cost preflight, ingest the post, author-cooldown check (skip if you engaged this author too recently), classify the post, write a draft reply, update status, and a separate error branch that responds 500 and notifies. **This is the most template-heavy workflow: five `executeCommand` nodes are `# TODO` placeholders** (cost preflight, ingester, cooldown check, classifier, writer).

**Dependencies:** five of your own scripts (not shipped), one per placeholder. It shows the orchestration, not the intelligence.

**Tables:** `captured_posts`, `system_events`.

**What you can do with it:** use it as a blueprint for any multi-stage AI pipeline with guard rails: authenticate, check budget, do work in stages, handle cooldowns, and always have an error branch that responds and alerts. Fill in the five placeholders with your own logic and the orchestration is done for you.

---

## `dashboard-write-relay.json` — Dashboard Write Relay

**Trigger:** webhook, API-key protected.

**What it does:** a generic write proxy. It checks an API key (401 if wrong), routes on an `action` field (`insert` / `update` / `delete`, 400 on unknown), performs the corresponding Supabase write, and can optionally kick off an Apify actor. Responds with the result.

**Dependencies:** none fixed; you point it at your own tables and, optionally, an Apify actor.

**What you can do with it:** put this between a frontend/dashboard and your database so the frontend never holds database credentials. The frontend sends an authenticated request with an action and a payload; this relay does the privileged write server-side. A clean pattern for any "let the browser write to the database, safely" need.
