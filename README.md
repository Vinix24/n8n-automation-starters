# n8n Automation Starters

13 n8n-workflows die ik draaide vóór ik overstapte naar een eigen Python-orchestrator. Dit is een werkende basis om te forken, niet een production-grade systeem zoals het bij mij draaide. Ik heb ze gesaneerd (geen chat-ID's, geen eigen Supabase-project, geen eigen LinkedIn-URL meer hardcoded) en gecureerd: de generiek-bruikbare workflows zitten erin, twee hyper-specifieke zijn eruit gelaten (zie onderaan).

*A working starting point for n8n-based LinkedIn/Telegram/Supabase automation — the workflows I ran before I migrated to my own Python orchestrator. Sanitized of personal identifiers; two overly-coupled workflows were excluded rather than shipped half-working (see "Wat ik heb weggelaten" below).*

## Wat je hier krijgt

Alle workflows draaien op drie bouwstenen: **Telegram** (commando's + notificaties), **Supabase/Postgres** (opslag, via de REST API — geen directe DB-credentials nodig) en **Apify** (LinkedIn-scraping via publieke marketplace-actors). Een paar workflows roepen daarnaast een eigen Python-module aan die niet in dit pakket zit — dat staat er expliciet bij.

| Workflow | Wat het doet |
|---|---|
| `01-morning-brief.json` | Elke ochtend om 07:00 een Telegram-samenvatting: content gepland voor vandaag, hot leads, openstaande comment-suggesties, ideeën-inbox. |
| `13-callback-handler.json` | Vangt Telegram inline-button clicks op (ICP-label kiezen voor een gevolgd profiel, een topic permanent opslaan als keyword) en schrijft het resultaat terug naar Supabase. |
| `13-healthcheck-channels.json` | Elke 15 minuten een check op Supabase, een optionele lokale wrapper en n8n zelf; bij een storing volgt direct een Telegram-alert. |
| `14-track-relay.json` | `/track <linkedin-url>` — scraped een post + comments via Apify, herkent je eigen reacties (via `OWN_LINKEDIN_URL`) en slaat alles gestructureerd op in Supabase. |
| `15-follow-relay.json` | `/follow <linkedin-profiel>` — scraped het profiel via Apify, zet het op je watchlist en toont een inline-keyboard om meteen een ICP-label te kiezen. |
| `15-self-scout.json` | Wekelijkse Apify-scrape van je eigen LinkedIn-posts en comments, voor je eigen content-analytics (impressions, reacties, engagement). |
| `16-watchlist-daily-scrape.json` | Dagelijkse cron die je watchlist van gevolgde profielen scraped. **Bevat een TODO-placeholder command-node** — zie "Python-afhankelijkheid" hieronder. |
| `17-author-memory-cron.json` | Elk uur een aggregatie van je eigen reactiegeschiedenis per profiel (hoe vaak gereageerd, gemiddelde engagement). Puur Supabase-gedreven, geen externe dependency. |
| `18-dig-relay.json` | `/dig <topic>` — on-demand Apify-scout op een topic, met een maandbudget-guard (€50 cap) vóór de scrape start, en toont de top-5 resultaten in Telegram. |
| `21-question-clustering-weekly.json` | Wekelijkse cron die vragen uit comments clustert tot content-ideeën. **Bevat een TODO-placeholder command-node.** |
| `22-task-lease-reaper.json` | Elke 5 minuten verlopen taak-leases vrijgeven via een Postgres RPC — een generiek patroon voor elke taak-queue met lease/lock-semantiek. |
| `chain-2a-classifier-writer-pipeline.json` | Referentie-architectuur voor een AI-comment-pipeline: budget-preflight → ingest → auteur-cooldown-check → classificeren → concept-reactie schrijven → status bijwerken, met een aparte foutafhandelings-tak. **Vijf TODO-placeholder command-nodes**, zie hieronder — dit toont de orkestratie-vorm, geen drop-in werkend script. |
| `dashboard-write-relay.json` | Generieke, API-key-beveiligde webhook die insert/update/delete naar Supabase relayt en optioneel een Apify-actor kan starten. Bruikbaar als schrijf-laag tussen een dashboard/frontend en je database, zonder databasecredentials in de frontend. |

## Python-afhankelijkheid (belangrijk, lees dit voor je begint)

Drie workflows zijn **templates**: de `n8n-nodes-base.executeCommand`-nodes bevatten geen werkende Python-aanroep meer, maar een `# TODO: ...`-placeholder die het verwachte stdout-contract (welke JSON-keys de downstream nodes verwachten) documenteert:

- `16-watchlist-daily-scrape.json` → TODO-placeholder voor je eigen watchlist-scraper (verwacht: `{ scraped: <n> }`)
- `21-question-clustering-weekly.json` → TODO-placeholder voor je eigen content-pipeline (verwacht: `comments_fetched`, `questions_substantive`, `cluster_count`, `ideas_inserted`, `correlation_id`)
- `chain-2a-classifier-writer-pipeline.json` → vijf TODO-placeholders (cost pre-flight, ingester, cooldown-check, classifier, writer) — vrijwel elke stap

Deze workflows tonen de **orkestratie-vorm** (cron/webhook → guard-rails → job → notificatie), niet een kant-en-klaar werkend script. Vervang elke `# TODO: ...`-placeholder door je eigen script, API-call, of een simpele HTTP-request naar een eigen microservice — zolang de vervanging op stdout dezelfde JSON-keys print die in de placeholder-comment staan, blijft de rest van de workflow (budget-checks, foutafhandeling, Telegram-notificaties, Supabase-writes) ongewijzigd bruikbaar.

De overige 10 workflows zijn zelfstandig: Telegram + Supabase + Apify, geen verborgen dependency.

## Setup

1. **n8n draaien** — lokaal (`docker run -it --rm -p 5678:5678 n8nio/n8n`) of self-hosted.
2. **Credentials importeren** — `credentials.json.example` laat zien welke n8n-credentials je nodig hebt (MC API Key, Supabase Service Role, Apify Token) plus een aparte **Telegram**-credential (niet in het JSON-bestand, want n8n's Telegram-credential-type vraagt om een los bot-token via de UI). Maak een bot via [@BotFather](https://t.me/BotFather) en voeg die credential toe als "VNX Telegram Bot" (of hernoem de credential-referenties in de workflows naar je eigen naam).
3. **Env vars zetten** — kopieer `.env.example`, vul in, en zet ze in n8n (`N8N_ENV_FILE`, docker `--env-file`, of je hosting-platform se env-vars).
4. **Workflows importeren** — importeer elk bestand uit `workflows/` los in n8n (Import from File). Koppel per workflow de juiste credential aan de Telegram- en HTTP Request-nodes.
5. **Tabellen/schema** — de workflows verwachten tabellen zoals `captured_posts`, `comment_history`, `scraped_comments`, `creator_watchlist`, `monitoring_keywords`, `apify_usage`, `system.events` (of `events`/`system_events`, afhankelijk van de workflow — niet 100% consistent, een erfenis van hoe het organisch gegroeid is). **Pas tabelnamen en kolommen gerust aan je eigen schema aan** — dit is illustratief, geen vaste contractlaag.
6. **Testen** — start met `13-healthcheck-channels.json` en `01-morning-brief.json`: die hebben de minste bewegende delen en bevestigen dat je Supabase- en Telegram-credentials werken.

## Waarom ik ben overgestapt op Python

Deze n8n-workflows deden hun werk, maar een 100+ node Telegram-router in een visuele canvas wordt op een gegeven moment onbeheersbaar: geen versiebeheer op node-niveau, geen echte tests, en een n8n-restart die een workflow met een subtiele node-fout stil kapot kan maken (dat overkwam mij eind april 2026 — tien uur zonder bot). Ik heb de kern overgezet naar een eigen, open-source governance-first orchestrator: [VNX Orchestration](https://github.com/Vinix24/vnx-orchestration). Geen vendor-lock-in op een visuele workflow-tool, wel volledige NDJSON-receipts per stap en testbare Python-modules. Dit repo is de n8n-laag die eraan voorafging — een prima startpunt als je liever met een visuele tool begint dan met code.

## Wat ik heb weggelaten

Twee van de 15 bron-workflows zitten niet in dit pakket:

- **`03-telegram-router.json`** (110 nodes) — de centrale command-router die mijn hele bot-commandoset afhandelt (`/linkedin`, `/keyword`, `/status`, taalherkenning, pause/resume, etc.). Te specifiek gekoppeld aan mijn eigen command-surface om als generieke starter te dienen, en bevatte bovendien een hardcoded Supabase project-referentie diep in een paar Code-nodes — een teken dat 'm los saneren meer risico gaf dan waarde. Wil je een Telegram-commandorouter bouwen: kijk naar het webhook+switch-patroon in `13-callback-handler.json` en `18-dig-relay.json` en bouw je eigen command-set op die basis.
- **`16-leads-from-own-post-comments.json`** — detecteert leads onder je eigen LinkedIn-posts, maar roept een lokale Python-webhook aan (`http://localhost:8765/leads/process`) die niet in dit pakket zit. Zonder die classifier is de workflow een dead end. Combineer in plaats daarvan `14-track-relay.json` (scrape + eigen-comment-detectie) met je eigen classificatielogica.

## Wat ik heb gesaneerd

- Hardcoded Tailscale-IP (`100.86.73.22`) in `13-healthcheck-channels.json` → env var `LOCAL_CLAUDE_WRAPPER_URL` met placeholder-fallback.
- Hardcoded eigen LinkedIn-profiel-URL in `14-track-relay.json` (2 plekken) en `15-self-scout.json` (1 plek) → env var `OWN_LINKEDIN_URL`, geen fallback naar mijn eigen profiel meer. De interne veldnamen `vincent_comments`/`vincent_comment_found` in `14-track-relay.json` zijn hernoemd naar `own_comments`/`own_comment_found`.
- Verder is er getest op hardcoded chat-ID's, Supabase-URL's/project-refs, tokens, webhook-secrets en e-mailadressen in alle 13 opgenomen workflows — die gebruiken allemaal `$env.*`-referenties, geen literals.
- Interne projectnaam-referenties (n8n `instanceId`, credential-beschrijvingen, een Telegram-berichttekst, een `source`-filterwaarde en een persoonlijke naam in een code-comment) zijn verwijderd of hernoemd naar generieke termen.

## Licentie

MIT — zie `LICENSE`. Doe ermee wat je wilt.
