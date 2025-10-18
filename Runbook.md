# Rakista Ops Runbook (vars‑first edition)

> **Single source of truth:** Do **not** hardcode IDs/URLs here. All live constants are in `variables.yml`.  
> When something changes: update `variables.yml`, then add a one‑liner in the change log below.

## change_summary
- 2025-10-18: Finalized public repo, moved constants to `variables.yml`, completed n8n upgrade to `variables.yml → n8n.version`, enabled GCP uptime + GSM audit alerts, backups to GCS verified, Cloudflare tunnel note (locally managed).  
- 2025-10-18a: Editorial polish — added “Values at a glance”, explicit Cloudflare config path, explicit public‑repo/no‑secrets line, and DR/restore cross‑reference.

---

# How to use this runbook
- Follow the procedures here. Whenever you see a concrete value (project/zone/VM/bucket/domain/IDs), **look it up** in `variables.yml` (e.g., “see `variables.yml → gcp.zone`”).
- Never paste secret **payloads** here. Secret **names** only (payloads live in Bitwarden and/or GCP Secret Manager).

## Values at a glance (keys only; see `variables.yml` for values)
- `project.id`
- `gcp.region`, `gcp.zone`, `gcp.vm`, `gcp.disk`, `gcp.snapshot_policy`
- `storage.backup_bucket`
- `cloudflare.domain`, `cloudflare.tunnel_id`
- `n8n.version`, `n8n.health_url`
- `secrets.encryption_key_name`
- `monitoring.channels.email`, `monitoring.channels.chat`
- `schedules.cron_db_utc`, `schedules.cron_volume_utc`

---

# 0) Network & IAM — ✅

**CLI**
```bash
# Subnet (see vars)
gcloud compute networks subnets create n8n-subnet-asia-east1 \
  --network=wp-vpc \
  --range=10.10.20.0/24 \
  --region=$(yq '.gcp.region' variables.yml)

# Service account + roles (see vars)
gcloud iam service-accounts create n8n-compute --display-name="n8n compute"

gcloud projects add-iam-policy-binding $(yq '.project.id' variables.yml) \
  --member="serviceAccount:n8n-compute@$(yq '.project.id' variables.yml).iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

gcloud projects add-iam-policy-binding $(yq '.project.id' variables.yml) \
  --member="serviceAccount:n8n-compute@$(yq '.project.id' variables.yml).iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# IAP SSH firewall (tag: iap-ssh)
gcloud compute firewall-rules create allow-iap-ssh-n8n \
  --network=wp-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=iap-ssh
```

**GUI**: VPC → Subnets; IAM → Service accounts & role bindings; VPC firewall → new rule (source 35.235.240.0/20).

---

# 1) Storage (GCS) — ✅
- **Bucket**: see `variables.yml → storage.backup_bucket`  
- **Lifecycle**: 30d→Coldline, 180d→Delete  
**GUI**: Cloud Storage → Buckets → Lifecycle rules.

---

# 2) Compute & App (Docker + n8n + Postgres) — ✅

**CLI**
```bash
# Data disk
gcloud compute disks create $(yq '.gcp.disk' variables.yml) \
  --size=50GB --type=pd-ssd --zone=$(yq '.gcp.zone' variables.yml)

# VM (no public IP, IAP tag, SA)
gcloud compute instances create $(yq '.gcp.vm' variables.yml) \
  --zone=$(yq '.gcp.zone' variables.yml) \
  --machine-type=e2-medium \
  --subnet=n8n-subnet-asia-east1 \
  --no-address \
  --service-account=n8n-compute@$(yq '.project.id' variables.yml).iam.gserviceaccount.com \
  --scopes=https://www.googleapis.com/auth/cloud-platform \
  --boot-disk-size=30GB --boot-disk-type=pd-ssd \
  --disk=name=$(yq '.gcp.disk' variables.yml),mode=rw,auto-delete=no \
  --tags=iap-ssh \
  --labels=stack=n8n,env=prod,owner=harold \
  --maintenance-policy=MIGRATE --restart-on-failure
```

**On the VM**
- Format & mount **/dev/sda → `/srv`** via **UUID** in `/etc/fstab`  
- Set Docker `data-root=/srv/docker`; install Docker + Compose  
- App lives in `/srv/n8n`; n8n binds to `127.0.0.1:5678`  
- n8n container version: **see `variables.yml → n8n.version`**

**Compose (excerpt)**
```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}   # from Bitwarden, not in git
      POSTGRES_DB: n8n
    volumes:
      - pgdata:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:${N8N_VERSION}  # export from vars when deploying
    depends_on: [postgres]
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}  # value from GSM/Bitwarden
      N8N_HOST: $(yq '.cloudflare.domain' variables.yml)
      N8N_PROTOCOL: https
      N8N_PORT: 5678
      WEBHOOK_URL: https://$(yq '.cloudflare.domain' variables.yml)/
      N8N_DIAGNOSTICS_ENABLED: "false"
      N8N_METRICS: "true"
      N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS: "true"
      N8N_RUNNERS_ENABLED: "true"
      N8N_GIT_NODE_DISABLE_BARE_REPOS: "true"
      N8N_BLOCK_ENV_ACCESS_IN_NODE: "true"
    volumes:
      - n8n_data:/home/node/.n8n
    ports:
      - "127.0.0.1:5678:5678"

volumes:
  pgdata:
  n8n_data:
```

---

## 2.4) Snapshots — ✅
- **Policy**: see `variables.yml → gcp.snapshot_policy` (PH ~03:00, 14d retention)  
- **Attached to**: `variables.yml → gcp.disk`

**CLI**
```bash
gcloud compute resource-policies create snapshot-schedule $(yq '.gcp.snapshot_policy' variables.yml) \
  --region=$(yq '.gcp.region' variables.yml) \
  --daily-schedule --start-time=19:00 \
  --max-retention-days=14 --on-source-disk-delete=apply-retention-policy \
  --snapshot-labels=stack=n8n

gcloud compute disks add-resource-policies $(yq '.gcp.disk' variables.yml) \
  --zone=$(yq '.gcp.zone' variables.yml) \
  --resource-policies=$(yq '.gcp.snapshot_policy' variables.yml)
```

---

# 3) Backups to GCS — ✅ (Option A)

**Destinations**  
- DB dump → `gs://(variables.yml → storage.backup_bucket)/db/`  
- n8n volume → `gs://(variables.yml → storage.backup_bucket)/app/`  
- Staging dir on VM: `/srv/backup`

**Manual run (VM)**
```bash
# DB
DBPASS=$(docker exec n8n-postgres-1 printenv POSTGRES_PASSWORD)
DATE=$(date -u +%F-%H%MZ)
mkdir -p /srv/backup
docker exec -e PGPASSWORD="$DBPASS" -i n8n-postgres-1 \
  pg_dump -U n8n -d n8n | gzip > "/srv/backup/n8n-$DATE.sql.gz"

# n8n volume
sudo tar -czf /srv/backup/n8n-data-$DATE.tgz \
  -C /srv/docker/volumes/n8n_n8n_data/_data .

# Upload
gsutil cp /srv/backup/n8n-$DATE.sql.gz gs://$(yq '.storage.backup_bucket' variables.yml)/db/
gsutil cp /srv/backup/n8n-data-$DATE.tgz gs://$(yq '.storage.backup_bucket' variables.yml)/app/
```

**Cron (UTC)**  
- DB dump: `variables.yml → schedules.cron_db_utc`  
- Volume: `variables.yml → schedules.cron_volume_utc`

> **Restore note:** The step‑by‑step restore lives in the **Backup & Restore / DR Guide**. Keep this runbook focused on backup creation; see that guide for restore & verification.

---

## 3.2) Secret Manager (N8N_ENCRYPTION_KEY) — ✅

- Secret name: `variables.yml → secrets.encryption_key_name`  
- Payload stored in **Bitwarden (folder: variables)** and **GSM**; not in git.

**Sync GSM → .env (VM)**
```bash
KEY="$(gcloud secrets versions access latest --secret=$(yq '.secrets.encryption_key_name' variables.yml) | tr -d '\n')"
sudo sed -i "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=${KEY}|" /srv/n8n/.env
docker compose -f /srv/n8n/docker-compose.yml up -d
```

**Verify VM key matches GSM (no secret printed)**
```bash
# Cloud Shell
gcloud secrets versions access latest --secret=$(yq '.secrets.encryption_key_name' variables.yml) > /tmp/gsm_key
sha256sum /tmp/gsm_key | cut -c1-12

# VM (reads on-disk n8n settings)
sudo awk -F'"' '/encryptionKey/ {print $4}' /srv/docker/volumes/n8n_n8n_data/_data/config \
  | sha256sum | cut -c1-12
# Hashes must match
```

---

# 4) Ingress (Cloudflare Tunnel + Access) — ✅

- **Tunnel** → `variables.yml → cloudflare.domain` → `http://localhost:5678`  
- **Zero Trust Applications**:
  - App **UI** (domain = `variables.yml → cloudflare.domain`): **Allow** (SSO wall)  
  - App **Webhooks** (path `/webhook*`): **Bypass** (so uptime checks & external triggers can reach)
- **Important note:** This tunnel is **locally managed**. Config lives at **`/etc/cloudflared/config.yml`** on the VM (the Zero Trust dashboard shows “locally configured,” and cannot edit the tunnel).

**Verify**
```bash
# UI should redirect to Access login
curl -I https://$(yq '.cloudflare.domain' variables.yml) | sed -n '1p'

# Webhook endpoint (will 404 until workflow exists)
curl -I https://$(yq '.cloudflare.domain' variables.yml)/webhook/test | sed -n '1p'
```

---

# 5) Monitoring — ✅

## 5.1 n8n “health” workflow
- Create **Webhook** node: **GET** `/healthz`, **Production URL** only.  
- Add **Respond to Webhook** node: status **200**, header `content-type: text/plain`, body `ok`.  
- **Activate** the workflow.

**Check**
```bash
curl -i "$(yq '.n8n.health_url' variables.yml)" | sed -n '1,3p'
# Expect: HTTP/2 200 and body 'ok'
```

## 5.2 GCP Uptime Check (HTTPS)
- Hostname: `variables.yml → cloudflare.domain`  
- Path: `/webhook/healthz`  
- Frequency **1 minute**, Timeout **10s**, Global regions  
- Alert policy: **fire after 3m of failures**, notifications to channels in `variables.yml → monitoring.channels.*`.

---

# 6) Hardening / Ops polish — ✅

- **Diagnostics disabled** (`N8N_DIAGNOSTICS_ENABLED=false`)  
- **Metrics enabled** (`N8N_METRICS=true`)  
- **Enforce settings file perms** (`N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`)  
- **Task runners** (`N8N_RUNNERS_ENABLED=true`)  
- **Disable bare Git repos** (`N8N_GIT_NODE_DISABLE_BARE_REPOS=true`)  
- **Block env in Code nodes** (`N8N_BLOCK_ENV_ACCESS_IN_NODE=true`) — appropriate for our use cases  
- **GSM Data Access audit logs** (Admin read + Data read) enabled  
- **GSM alert**: Logs-based alert on `AccessSecretVersion` for secret name = `variables.yml → secrets.encryption_key_name`  
- **Container & VM updates**: apt + docker package updates applied; restart verified  
- **n8n upgraded** to `variables.yml → n8n.version` (migrations completed; health OK)

**Pending (optional improvements)**
- Monthly **machine image** of VM (we already snapshot the data disk). If added, note schedule here.

---

# 7) Runbook housekeeping — ✅

- All live constants are in **`variables.yml`**:
  - Project/region/zone/VM/disk/snapshot policy
  - Backup bucket
  - Domain, tunnel ID (identifier only)
  - n8n version and health URL
  - Secret **names** (not values), Bitwarden folder/items
  - Monitoring channels & schedules
- **Change management**:
  1. Update `variables.yml`.  
  2. Add one line in **`change_summary`** above.  
  3. Commit via PR (use the PR template checklist).  

---

# GitHub (docs repo)

- Repo name: **rakista-ops** (**public by design**). **No secrets in git;** secret payloads live in **Bitwarden (folder: variables)** and **GSM**.  
- Branch model: `main` with PRs, linear history (Squash & merge)  
- Files:
  - `variables.yml` → single source of truth (no secrets)  
  - `Runbook.md` → procedures (references vars)  
  - `.github/PULL_REQUEST_TEMPLATE.md` → self-check before merge  
- Security:
  - **Secret scanning** enabled; **Push protection** enabled  
  - **Dependabot alerts/updates** enabled  

---

## Appendix: quick checks

**n8n version & health**
```bash
docker exec -it n8n-n8n-1 n8n --version
curl -i "$(yq '.n8n.health_url' variables.yml)" | sed -n '1,3p'
```

**Cloudflared**
```bash
systemctl status cloudflared
sudo cat /etc/cloudflared/config.yml
```

**Backups in GCS**
```bash
gsutil ls -l gs://$(yq '.storage.backup_bucket' variables.yml)/db/ | tail
gsutil ls -l gs://$(yq '.storage.backup_bucket' variables.yml)/app/ | tail
```

**GSM audit reads (Logs Explorer filter)**
```
logName="projects/$(vars.project.id)/logs/cloudaudit.googleapis.com%2Fdata_access"
protoPayload.serviceName="secretmanager.googleapis.com"
protoPayload.methodName="google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion"
protoPayload.resourceName: "secrets/$(variables.yml → secrets.encryption_key_name)"
```
