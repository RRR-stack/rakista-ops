# Backup & Restore / DR Guide

> Constants come from `variables.yml`. Secret payloads live in **Bitwarden (folder: “variables”)** and **GCP Secret Manager (GSM)**.  
> Target VM runs Docker Compose at `/srv/n8n`.

## What we back up (daily)
- **Postgres DB** (logical dump via `pg_dump`) → `gs://<storage.backup_bucket>/db/`
- **n8n application data** (`/home/node/.n8n`) → tarball → `gs://<storage.backup_bucket>/app/`
- **Persistent disk snapshots** → policy `gcp.snapshot_policy` (fast infra rollback)

---

## A) Run backups on demand (VM)
```bash
# Load constants locally (Cloud Shell or your admin host)
BUCKET="$(yq '.storage.backup_bucket' variables.yml)"

# SSH to VM
gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)"

# On the VM:
DBPASS="$(docker exec n8n-postgres-1 printenv POSTGRES_PASSWORD)"
DATE="$(date -u +%F-%H%MZ)"
mkdir -p /srv/backup

# 1) DB dump
docker exec -e PGPASSWORD="$DBPASS" -i n8n-postgres-1   pg_dump -U n8n -d n8n | gzip > "/srv/backup/n8n-$DATE.sql.gz"

# 2) n8n app data tar
sudo tar -czf "/srv/backup/n8n-data-$DATE.tgz"   -C /srv/docker/volumes/n8n_n8n_data/_data .

# 3) Upload to GCS
gsutil cp "/srv/backup/n8n-$DATE.sql.gz" "gs://$BUCKET/db/"
gsutil cp "/srv/backup/n8n-data-$DATE.tgz" "gs://$BUCKET/app/"
```

**Verify in GCS**
```bash
gsutil ls -l "gs://$BUCKET/db/"  | tail
gsutil ls -l "gs://$BUCKET/app/" | tail
```

---

## B) Routine restore test (non-destructive)
Goal: prove we can read both artifacts and that the DB dump is valid.

```bash
BUCKET="$(yq '.storage.backup_bucket' variables.yml)"

# 1) Pull the most recent artifacts locally (Cloud Shell)
gsutil ls -l "gs://$BUCKET/db/"  | sort -k2,2 | tail -n 1
gsutil ls -l "gs://$BUCKET/app/" | sort -k2,2 | tail -n 1

# Substitute the exact object names from the lines above:
gsutil cp "gs://$BUCKET/db/<LATEST_DB>.sql.gz"  /tmp/db.sql.gz
gsutil cp "gs://$BUCKET/app/<LATEST_APP>.tgz"   /tmp/app.tgz

# 2) Validate DB dump structure (no import yet)
gzip -cd /tmp/db.sql.gz | head
gzip -t  /tmp/db.sql.gz && echo "DB dump gzip OK"

# 3) Validate app tar readability
tar -tzf /tmp/app.tgz | head
```

Optionally, stand up a throwaway **local Postgres** (Docker on your workstation) and `pg_restore`/`psql` the dump to confirm tables create cleanly.

---

## C) Full restore to the same VM (in-place)
> Use only when you intend to roll back the live system. Stops containers briefly.

1) **Choose backup set (Cloud Shell):**
```bash
BUCKET="$(yq '.storage.backup_bucket' variables.yml)"
# Pick timestamps for DB and APP you want to restore:
gsutil ls -l "gs://$BUCKET/db/"  | tail -n 10
gsutil ls -l "gs://$BUCKET/app/" | tail -n 10
```

2) **SSH to VM & stop app:**
```bash
gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --   "cd /srv/n8n && docker compose down"
```

3) **Restore DB:**
```bash
# Get DB password
DBPASS="$(docker exec n8n-postgres-1 printenv POSTGRES_PASSWORD)"

# Drop and recreate schema (minimal downtime approach: drop public; recreate)
docker exec -i n8n-postgres-1 psql -U n8n -d n8n <<'SQL'
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO n8n;
GRANT ALL ON SCHEMA public TO public;
SQL

# Import from chosen dump
gsutil cp "gs://$(yq '.storage.backup_bucket' variables.yml)/db/<CHOSEN_DB>.sql.gz" /tmp/db.sql.gz
gzip -cd /tmp/db.sql.gz | docker exec -i n8n-postgres-1 psql -U n8n -d n8n
```

4) **Restore n8n app data:**
```bash
# Backup current app dir just in case
sudo mkdir -p /srv/backup/prev && sudo tar -czf "/srv/backup/prev/n8n-data-pre-restore-$(date -u +%F-%H%MZ).tgz"   -C /srv/docker/volumes/n8n_n8n_data/_data .

# Pull and expand chosen tar
gsutil cp "gs://$(yq '.storage.backup_bucket' variables.yml)/app/<CHOSEN_APP>.tgz" /tmp/app.tgz
sudo tar -xzf /tmp/app.tgz -C /srv/docker/volumes/n8n_n8n_data/_data
sudo chown -R 1000:1000 /srv/docker/volumes/n8n_n8n_data/_data
```

5) **Start app & verify:**
```bash
cd /srv/n8n && docker compose up -d
docker compose ps
docker exec -it n8n-n8n-1 n8n --version
curl -i "$(yq '.n8n.health_url' variables.yml)" | sed -n '1,3p'
```

---

## D) DR via disk snapshot (fast infra rollback)
> Use when the whole application state (VM disk) must be rolled back quickly.

1) **Identify a good snapshot** (Compute → Disks → Snapshots, or CLI).

2) **Create a new disk from snapshot** (safe) **or** **restore original disk** (destructive):
```bash
# (Example) create new disk from snapshot and attach to a replacement VM
gcloud compute disks create "n8n-data-restore-$(date +%Y%m%d-%H%M)"   --source-snapshot="<SNAPSHOT_NAME>"   --zone="$(yq '.gcp.zone' variables.yml)" --type=pd-ssd
```

3) **Attach/mount** to a clean VM, ensure `/srv` mounts correctly, then redeploy Compose.  
4) **Point tunnel/DNS** if using a replacement VM.

---

## E) Post-restore verification (must pass)
- Health URL returns **HTTP 200** (twice):  
  `curl -i "$(yq '.n8n.health_url' variables.yml)" | sed -n '1,3p'`
- n8n UI loads via Cloudflare Access; can log in.  
- A known test workflow runs end-to-end.  
- If a restore was due to corruption, run a **fresh backup** immediately after success.

---

## F) Notes & gotchas
- **Encryption key**: Restores keep `/home/node/.n8n/config`. If key mismatches env (`N8N_ENCRYPTION_KEY`), n8n won’t start. Sync from GSM:
  ```bash
  KEY="$(gcloud secrets versions access latest --secret="$(yq '.secrets.encryption_key_name' variables.yml)" | tr -d '\n')"
  sudo sed -i "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=${KEY}|" /srv/n8n/.env
  (cd /srv/n8n && docker compose up -d)
  ```
- **Permissions**: n8n fixes loose perms but we prefer explicit:  
  `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`
- **Backups are only useful if tested**: schedule a quarterly restore test (lab VM).
