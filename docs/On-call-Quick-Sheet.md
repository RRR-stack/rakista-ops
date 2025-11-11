# On-call Quick Sheet

> Single source of truth for constants: see `variables.yml`.
> No secrets in docs. Secret payloads are in Bitwarden (folder **variables**) and GSM.

## 0) When the pager fires
- **Common alerts**
  - **n8n health down**: Uptime failure for `GET /webhook/healthz` (see `n8n.health_url`)
  - **Secret read**: Access to `secrets.encryption_key_name`
  - **Infra**: NAT port issues, 5xx spikes (Rakista LB), etc.
- **Where alerts go**: `monitoring.channels.email`, `monitoring.channels.chat` (ID: `monitoring.chat_channel_id`)

## 1) First 60 seconds (triage)
1. Open **GCP Monitoring → Alerting → Incidents**; read the incident’s **Timeline**.
2. Check **Uptime Check result** (it logs the HTTP status/body).
3. Confirm **status page** in your head: is main site up? (We only monitor n8n here.)
4. **Acknowledge** incident to stop noisy duplicates.

## 2) Fast health checks (CLI)
> VM name/zone from `gcp.vm`, `gcp.zone`.

```bash
# Reachability (public path)
curl -i "$(yq '.n8n.health_url' variables.yml)" | sed -n '1,3p'

# Container status
gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --   "docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' &&    docker compose -f /srv/n8n/docker-compose.yml ps"

# n8n quick logs (last 80 lines)
gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --   "docker compose -f /srv/n8n/docker-compose.yml logs --tail=80 n8n"

# Cloudflared tunnel status
gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --   "systemctl status cloudflared --no-pager; sudo sed -n '1,80p' /etc/cloudflared/config.yml"
```

## 3) Known quick fixes (safe)
- **Container wedged** (app up locally, but not answering):
  ```bash
  # Restart n8n container
  gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --     "cd /srv/n8n && docker compose up -d --force-recreate n8n"
  ```
- **Tunnel looks down** (cloudflared not active):
  ```bash
  gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --     "sudo systemctl restart cloudflared && systemctl --no-pager status cloudflared"
  ```
- **Health workflow missing** (404 at /webhook/healthz):
  Re-activate the “Healthcheck – Public” workflow in n8n UI; confirm `200 ok`.

## 4) If n8n is down but VM is fine
1. **Check DB**:
   ```bash
   gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --      "docker exec -it n8n-postgres-1 pg_isready -U n8n -d n8n"
   ```
2. **Config mismatches** (encryption key, etc.):
   ```bash
   # Compare GSM key to on-disk hash (no secret printed)
   gcloud secrets versions access latest --secret="$(yq '.secrets.encryption_key_name' variables.yml)" > /tmp/gsm_key
   gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --      "sudo awk -F'\"' '/encryptionKey/ {print \$4}' /srv/docker/volumes/n8n_n8n_data/_data/config | sha256sum | cut -c1-12"
   sha256sum /tmp/gsm_key | cut -c1-12
   # Hashes should match
   ```
   ## 4b) PHP-FPM & Redis quick checks (web)

   - Workers/memory:
     `ps --no-headers -o rss -C php-fpm8.3 | awk '{s+=$1;n++} END{printf("php8.3 total=%.1fMB avg=%.1fMB workers=%d\n", s/1024,(n?s/n/1024:0), n)}'`
   - FPM config test:
     `sudo php-fpm8.3 -t`  → expect “test is successful”
   - Redis health & cap:
     `systemctl status redis-server --no-pager | sed -n '1,5p'`
     `redis-cli info memory | egrep 'used_memory_human|maxmemory_human'`
     `redis-cli info stats  | egrep 'keyspace_hits|keyspace_misses'`
   - If needed, disable object cache quickly:
     `sudo -u www-data wp --path=/var/www/rakista.com/htdocs redis disable`

## 5) Rollback/Restore pointers (quick)
- **Snapshots**: disk snapshots via `gcp.snapshot_policy` (Compute → Disks → Snapshots).
- **Backups** (GCS): DB dumps under `gs://storage.backup_bucket/db/`, volume tars under `.../app/`.
- **Full procedure**: see **Backup & Restore / DR Guide** (includes restore verification).

## 6) Exit criteria
- Health URL returns **HTTP 200** (2 consecutive probes).
- No critical errors in `docker compose logs n8n`.
- Incident **Resolved** in Monitoring.

## 7) Who to contact / escalation
- **Owner**: harold@rakista.com
- **Channels**: `monitoring.channels.chat`, `monitoring.channels.email`
- **Vendors**: Cloudflare (tunnel), Google Cloud (Monitoring/Compute/Storage)
