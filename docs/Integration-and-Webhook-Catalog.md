# Integration & Webhook Catalog

> Source of constants: `variables.yml`.  
> UI is behind Cloudflare Access; **`/webhook*` is Bypass** for external triggers and uptime.  
> Health endpoint: `n8n.health_url`.

## A) Conventions

- **Naming**: Workflow = `Area – Verb – Object` (e.g., `Social – Post – Facebook Reel`).  
- **Public webhooks** live under `https://<cloudflare.domain>/webhook/...` (Production URL).  
- Add **shared secret header** for public webhooks: `X-Auth-Token: <non-secret toggle or signed token>`.  
- **Credentials**: name format `svc-<provider>-<scope>` (stored in n8n Credentials; secrets live in GSM/Bitwarden).  
- **Owners**: set `owner:<email>` tag on each workflow.

## B) Global security & headers

- Cloudflare Access: UI **Allow list**, `/webhook*` **Bypass**.  
- For each public webhook workflow:
  - First node validates `X-Auth-Token` (see Dev Conventions §2).  
  - Reject missing/invalid tokens with **HTTP 401** (Code node + Respond to Webhook).

## C) Active Integrations (Production)

| Integration | Purpose | Trigger | Endpoint / Path | Auth | Rate Limit | Owner | Status |
|---|---|---|---|---|---|---|---|
| **Healthcheck – Public** (ID in n8n) | External uptime probe | Webhook (GET) | `/webhook/healthz` | None | n/a | harold@rakista.com | **Active** |
| **Social – Post – Facebook Reel** | Publish reels from queue | Cron (every 15m) | n/a | `svc-meta-graph-publish` | Provider limits | harold@… | Planned |
| **Social – Post – Instagram** | Publish posts/stories | Cron | n/a | `svc-meta-graph-publish` | Provider limits | harold@… | Planned |
| **Content – Pull – RSS** | Ingest RSS to queue | Cron | n/a | None | n/a | harold@… | Planned |
| **Content – Webhook – WordPress** | Receive WP publish events | Webhook (POST) | `/webhook/wp/published` | `X-Auth-Token` | n/a | harold@… | Planned |
| **Analytics – Push – Sheet** | Append run metrics | Node call | n/a | `svc-google-sheets-rw` | Google API | harold@… | **Active** (if you turned it on) |

> Keep this table short. For detailed node-by-node info, store each workflow JSON under `workflows/…` in git (per Dev Conventions §8).

## D) Test matrix (per webhook)

| Check | Command / Action | Expect |
|---|---|---|
| Reachability (prod) | `curl -i "https://<cloudflare.domain>/webhook/healthz"` | `HTTP/2 200`, body “ok” |
| Auth required | `curl -i -X POST "https://<cloudflare.domain>/webhook/wp/published"` | `401 Unauthorized` |
| Auth ok | `curl -i -H "X-Auth-Token: <token>" -d '{}' "https://<cloudflare.domain>/webhook/wp/published"` | `2xx` and trace ID |
| Rate-limit behavior | Trigger 10x in 10s (if applicable) | Some 429s or queued behavior |

> Replace `<cloudflare.domain>` with `variables.yml → cloudflare.domain`. Tokens and API keys must not be hardcoded—use Credentials/Bitwarden/GSM.

## E) Adding a new Integration (checklist)

1. **Design**: define trigger (Cron/Webhook), input schema, outputs, and external API limits.  
2. **Credentials**: create `svc-<provider>-<scope>` in n8n; secret payload sourced from Bitwarden/GSM.  
3. **Workflow**: build in `Sandbox/**`, add **Notes** node (purpose, inputs/outputs, failure modes).  
4. **Security**: for webhooks, add `X-Auth-Token` validation Code node.  
5. **Observability**: add final metrics Set node; optional Google Sheet log line.  
6. **Promotion**: duplicate to `Prod/**`, enable, export JSON → commit under `workflows/`.  
7. **Catalog**: add one table row here (Purpose, Trigger, Path, Auth, Owner).  
8. **Runbook touchpoints**: if constants/URLs change, update `variables.yml` (single source of truth).

## F) Sample webhook skeleton (n8n)

- **Nodes**: Webhook (Production URL), Code (auth check), Set (constants), your logic…, Respond to Webhook.
- **Code node (auth)**:
  ```js
  const token = $json.headers?.['x-auth-token'];
  if (token !== $env.SECRET_TOKEN) throw new Error('Unauthorized');
  return $input.all();
  ```
- **Respond to Webhook**: `200 OK`, JSON `{ "ok": true, "workflow": $workflow.name, "execId": $execution.id }`.

## G) Known external limits (track here)

| Provider | Endpoint | Limit | Mitigation |
|---|---|---|---|
| Meta Graph | Publish | Per-app/user rate | Set **Retry on 429** and **Split In Batches** |
| Google Sheets | Append | ~100 req/min | Buffer to batches of 50 |
