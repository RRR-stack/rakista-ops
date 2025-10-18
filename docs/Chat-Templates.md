# Chat Templates

Use these as the **first message** in a new ChatGPT chat. Paste the **Universal Header** first, then the scenario-specific prompt.

---

## Universal Header (paste first)
```
PROJECT CONTEXT (read-only)
- Name: Rakista (infra + content automation)
- Source of truth: GitHub repo with Runbook + docs + variables.yml
  • Runbook: <repo>/blob/main/Runbook.md
  • Variables: <repo>/blob/main/variables.yml
  • Docs index: <repo>/blob/main/README.md
- Infra: GCP (n8n on Docker + Postgres), Cloudflare Access (UI) and Bypass (/webhook*)
- Health URL: see variables.yml → n8n.health_url
- Secrets: NEVER in chat. All secrets live in Bitwarden (folder “variables”) and GSM.
- Editing rule: Provide a DRAFT first, await “approved.” Do not autosave/apply changes.
- Output style: numbered steps with GUI clicks + CLI + brief “why this step.”
- Risk handling: call out destructive commands and offer safer, no-downtime alternatives.

ASSISTANT BEHAVIOR
- Follow our n8n Dev Conventions and Integration Catalog
- Use constants from variables.yml (say “see variables.yml → …”) — do NOT invent values.
- If unclear, assume the safest default and continue; do not block on questions.
```

---

## 1) AI agent: daily email triage → Telegram summary, delete spam, reply to partnerships
```
PROJECT CONTEXT
- Stack: n8n on Docker + Postgres; secrets in Bitwarden & GSM; constants in variables.yml
- Comms: Telegram bot for notifications; Gmail for inbox operations

TASK
Design an n8n workflow that runs daily to:
1) Read Gmail with query: is:unread newer_than:1d
2) Classify (spam / partnership / other) with lightweight rules or LLM
3) Delete spam, auto-reply to “partnership” using a canned template (safe, polite), leave others unread
4) Send me a Telegram summary (totals + top subjects + actions taken)

CONSTRAINTS
- No secrets in chat (use n8n Credentials, e.g., svc-google-gmail-rw, svc-telegram-bot)
- Timeouts/retries on 429/5xx; add “dry_run=true” toggle
- Log actions to a Google Sheet row

DELIVERABLES
- Node list & settings (in order)
- Classifier rules or small LLM prompt
- Reply template (short, editable)
- Test plan and rollback (disable deletes in dry run)
```

---

## 2) Strategy prompt: 2026 Red Horse Beer × Rakista partnership proposal
```
ROLE
You are a brand strategist and music culture marketer.

TASK
Draft a proposal outline and key ideas for a 2026 partnership between Red Horse Beer and Rakista:
- Objectives, audience, messaging pillars
- Flagship activation (live/online), content series, influencer plan, sampling mechanics, safety/compliance
- 12-month comms calendar (high-level), media mix, KPIs, success metrics
- Rough budget buckets and risk mitigations

CONSTRAINTS
- Keep it Philippines-context, rock/alt audience
- Emphasize measurable outcomes and responsible consumption
- Output: 2-page brief + bullet slides outline (headlines + talking points)
```

---

## 3) 1-month content plan using Airtable (review/approval + client notifications)
```
PROJECT CONTEXT
- Planning DB: Airtable base with tables: Content, Channels, Approvals
- Notifications: WeChat or Email to client on status changes

TASK
Design an n8n-driven workflow + Airtable schema to:
1) Generate a 1-month multi-channel content plan (FB/IG/Reels/TikTok/site) with themes & hooks
2) Write rows to Airtable (title, copy draft, asset checklist, due date, owner)
3) Manage review/approval in Airtable (statuses: draft → review → approved)
4) Notify client via WeChat/Email on state changes
5) Export approved items as a scheduling feed for other automations

DELIVERABLES
- Airtable tables/fields (exact names)
- n8n node sequence for create/update + webhook from Airtable automations
- Example 1-month calendar grid (CSV or Markdown)
- Notification templates and test steps
```

---

## 4) AI agent: schedule approved content submitted via Telegram chat
```
TASK
Create an n8n workflow where I send content via Telegram (text + image/link), the bot stores it, routes for approval, then schedules on approved channels.

REQUIREMENTS
- Telegram intake → normalize to fields (title, body, media_url, target_channel, desired_time)
- Approval path: auto-approve me; optional client approval via Airtable/Email
- Scheduling: produce “ready” items with exact timestamps, respecting rate limits per platform
- Confirmation back to Telegram with a preview and calendar link

DELIVERABLES
- Node list & configs (Telegram → validate → store → approve → schedule)
- Data schema for “content item”
- Error handling & retries (failed posts requeue)
- Dry-run toggle and test checklist
```

---

## 5) AI agent: watch artists’ social accounts → post curated updates to WordPress
```
TASK
Build an n8n pipeline that:
1) Monitors selected artists’ official accounts (RSS/API/scrape where allowed)
2) Dedupes and filters to “noteworthy updates”
3) Creates a WordPress post (title, excerpt, canonical link, category: “Artist Updates”)
4) Sends a daily digest to Telegram

CONSTRAINTS
- Respect platform ToS; prefer official APIs or RSS
- Add rate limiting + backoff; log each action to a Sheet
- Provide a “human-in-the-loop” pause/approve option

DELIVERABLES
- Source list & fetch method per source
- Node sequence (fetch → dedupe → enrich → WP create → notify)
- WP post template (fields, tags, categories)
- Test data and rollback steps (unpublish/delete safely)
```

---

## 6) AI agent: PR intake (URL/file) → post on WordPress → share to social
```
TASK
Design an n8n flow to accept a PR via Webhook or file upload (URL, PDF, DOCX), extract the essentials, draft a WP news post, and share to social.

REQUIREMENTS
- Ingest: webhook with link or file; parse title/summary/quotes/attribution
- Draft WP post with redaction check; require manual approve or quick rules
- On publish: generate social snippets for FB/IG/Twitter, queue via scheduling workflow
- Send confirmation & links to Telegram

DELIVERABLES
- Node graph (ingest → parse → draft → approve → publish → social)
- Parsing method (LLM prompt or rules); quality checks
- Templates: WP body + social copy variants
- Test plan with 2 sample PRs
```

---

## 7) AI agent: events/gigs → publish to WordPress → weekly “featured gigs” social posts
```
TASK
Create an n8n workflow that collects event submissions (form/Telegram/Google Sheet), validates fields, publishes to WordPress Events, and compiles a weekly featured list for social.

SCHEMA
- Required: artist, date/time (PH), venue, city, ticket/link, poster_url
- Validation: date in future, location not empty, poster optional

DELIVERABLES
- Intake options (webhook/Sheet/Telegram) and normalization node
- WP post mapping (custom post type or category “Gigs”)
- Weekly aggregator (cron) → pick top N by recency/artist priority → generate carousel copy
- Approval checkpoints and safety checks
```

---

## 8) Marketing/branding strategies
```
ROLE
You are a brand strategist with experience in music, youth culture, and beverage partnerships.

TASK
Propose 3 distinct branding strategies for Rakista over the next 12 months:
- Positioning, audience segments, value props
- Content pillars and flagship programs
- Channel mix (owned/earned/paid), community activation, partnerships
- KPIs and learning agenda per quarter

CONSTRAINTS
- Philippines context; budget-aware (small, medium, stretch)
- Output: strategy one-pager per route + 90-day action plan
```


---

## 9) “Prompt Composer” — generate tailored prompts for any scenario
```
ROLE
You are a prompt designer. Your job is to turn my requirements into a set of high-quality prompts I can use with ChatGPT (and other LLM tools).

CONTEXT (read-only)
- Project: Rakista (automation + content)
- Source of truth: Runbook + variables.yml (constants only; never invent values)
- Secrets live in Bitwarden/GSM (never request or include secrets)
- Tools I often use: n8n, WordPress, Airtable, Google (Gmail/Calendar/Sheets), Telegram, Cloudflare

MY REQUIREMENTS
<paste bullet points describing the task, goal, audience, constraints, tools, deadlines, tone, and any examples>

OUTPUT FORMAT
1) Quick Summary (3 bullets): what the prompts will help me achieve.
2) Prompt Set (6–10 items), each with:
   - Name: short title (e.g., “Plan → 30-day IG Calendar”)
   - When to use: one line
   - Prompt: a complete copy-paste block
   - Knobs (optional): variables I can tweak (e.g., timeframe, tone, target)
   Types to include:
   - Planning (strategy/brief)
   - Execution (step-by-step with GUI + CLI when relevant)
   - Critique/Refine (review my draft; point out risks)
   - Data-aware (references to variables.yml paths instead of raw values)
   - Safety/Privacy (call out secrets & destructive ops)
   - Dev-specific (n8n/WordPress/Airtable)
3) Follow-ups (5 one-liners I can ask next)
4) Self-check (what you assumed + what to confirm)

RULES
- Prefer actionable prompts that produce numbered steps + brief “why”
- When constants are needed, say: “see variables.yml → path”
- No secrets, tokens, or PII in outputs
- Default to safe choices; flag destructive operations
- For dev prompts, include test/rollback steps
```

### Micro version
```
Turn these requirements into 8 strong prompts (plan, execute, refine, data-aware, safe). 
Use variables.yml references for constants (e.g., “→ gcp.project_id”). 
No secrets. Include test + rollback in dev prompts.

Requirements:
<your bullets here>
```

### Example item format
```
Name: Execute → Build n8n Gmail triage
When to use: I’m ready to implement and test the workflow.
Prompt:
Design step-by-step instructions to build an n8n workflow that triages unread Gmail daily and sends a Telegram summary. 
Use credentials by name only (svc-google-gmail-rw, svc-telegram-bot). 
Reference constants by path (variables.yml → n8n.health_url). 
Include: node order, key settings, timeouts/retries, a dry_run toggle, and a verification checklist.
Knobs: schedule (“daily 07:30 PH”), label/query, dry_run (true/false).
```
