# n8n Dev Conventions

> Constants live in `variables.yml`. No secrets in workflows; payloads are in **Bitwarden** (folder `variables`) and **GSM**.  
> n8n version: `n8n.version`. Task runners are enabled; health is `/webhook/healthz`.

## 1) Structure & naming
- **Folders**: `Prod/**`, `Sandbox/**`, `Shared/**` (helpers).  
- **Workflow name**: `Area – Verb – Object` (e.g., `Social – Post – Facebook Reel`).  
- **IDs & tags**: set **owner tag** (`owner:harold`) and **system tag** (`env:prod|stage`) per workflow.

## 2) Triggers & endpoints
- Prefer **Schedule** or **Cron** for periodic jobs; use **Webhook** for external triggers.  
- Webhooks live under Cloudflare, path format: `/webhook/<service>/<action>`.  
- **Public-only** webhooks: use the Bypass app; add a shared secret header (e.g., `X-Auth-Token`) and verify early:
  ```js
  // Code node at the top
  const token = $json.headers?.['x-auth-token'];
  if (token !== $env.SECRET_TOKEN) { // set in .env or GSM → env var
    throw new Error('Unauthorized');
  }
  return $input.all();
  ```

## 3) Secrets & configuration
- **Never** paste secret values. Use **Credentials** or env via GSM:
  - API keys → n8n **Credentials** (scoped, named `svc-<provider>-<scope>`).
  - Env access is **blocked by default** (`N8N_BLOCK_ENV_ACCESS_IN_NODE=true`).  
    If you must read an env var, justify in the workflow notes and keep it **non-secret** (e.g., feature toggles).
- Reference constants from `variables.yml` in docs/runbook; do **not** hardcode static URLs or IDs in nodes—place them in one **Set** node at the top named `constants`.

## 4) Error handling, retries, timeouts
- **Per node**: set **Max attempts = 3**, **Wait between = exponential** where supported.  
- **HTTP Request**: set **timeout (10–30s)** and **Retry on 429/5xx**.  
- Add an **Error Trigger** workflow (`On Error: Global`) that:
  - Captures: workflow name, execution ID, input snippet, error message/stack.  
  - Notifies: Chat/email channel (use the same channels as Monitoring).  
  - Writes a compact log row to a Google Sheet or GCS JSONL.

## 5) Data contracts
- At top of each workflow, a **Set → constants** and a **Function → validateInput**:
  ```js
  // validateInput: throw if key fields missing
  const r = $json;
  const missing = ['id','source'].filter(k => r[k] == null);
  if (missing.length) throw new Error('Bad input: '+missing.join(','));
  return $input.all();
  ```
- Use **Item Lists** / **Split In Batches** to control fan-out; set **Concurrency** on nodes hitting external APIs.

## 6) Observability
- Add a final **Set → metrics** node: `status`, `duration_ms`, `items_processed`, `rate_limited`.  
- Send a compact metric to a webhook or Google Sheet (one line per run).  
- Long jobs: break into **producer (queue)** and **consumer (runner)** patterns.

## 7) Testing & promotion
- Build in `Sandbox/**`, duplicate to `Prod/**` for go-live.  
- For webhooks, keep a **-test** URL variant (n8n “Test URL”) during dev; switch to **Production URL** before enabling.  
- Keep **example payloads** in the workflow notes (redact PII/secrets).  
- After changes: verify health endpoint still returns **200** and that the **On-call Quick Sheet** checks pass.

## 8) Versioning & export
- After a change in `Prod/**`: **Export** the workflow JSON and commit to git under `workflows/` with filename:
  `YYYYMMDD-<folder>-<name>-<workflowId>.json` and a short CHANGELOG line in the PR.
- Don’t store credentials in git; only workflow JSON.

## 9) Performance & limits
- Respect provider rate limits; centralize rate limiting in one **Limiter** sub-workflow if needed.  
- Avoid massive in-memory arrays—paginate with **Split In Batches**.  
- For CPU-heavy steps, prefer **Task Runners** or split into multiple workflows chained by webhooks/queues.

## 10) Notes block (required)
Each workflow must include a **Notes** node with:
- Purpose (1–2 lines) and **owner**.  
- Inputs (schema) & outputs.  
- Credential names used.  
- Known failure modes & retry policy.

---

**Checklist before enabling in Prod**
- [ ] Owner tag and notes present  
- [ ] Inputs validated  
- [ ] Timeouts & retries set  
- [ ] Secrets via Credentials (no plaintext)  
- [ ] Health unchanged (200)  
- [ ] Exported JSON committed
