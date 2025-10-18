# Access Matrix

> Source of truth for resource names/IDs: `variables.yml`.  
> No payloads here. Secret values live in **Bitwarden (folder: “variables”)** and **GCP Secret Manager (GSM)**.

## A) Roles → Permissions

| Actor / Principal | Scope | Access | Why |
|---|---|---|---|
| **Harold** (`harold@rakista.com`) | GCP Project | Owner (or Editor + targeted roles) | Break-glass + operations |
| **Service Account — n8n** (`n8n-compute@<project>.iam.gserviceaccount.com`) | GSM | `roles/secretmanager.secretAccessor` | n8n reads `secrets.encryption_key_name` at runtime |
|  | GCS Bucket (`storage.backup_bucket`) | (none required from n8n) | Backups run on VM using VM’s default auth |
| **VM login (IAP SSH)** | VM (`gcp.vm`) | ssh via IAP (firewall tag `iap-ssh`) | Admin access to host |
| **Cloud Monitoring** | Project | read metrics/logs; send alerts | Observability |
| **GitHub repo** | Runbook/docs | Write (you), read (public) | Docs & collaboration |
| **Cloudflare** | Domain (`cloudflare.domain`) | Access policy admin (UI allowlist + webhook bypass) | SSO for UI; uptime/webhooks open |
| **Postgres** | DB in container | Local auth (`POSTGRES_PASSWORD`) | n8n application DB |

**IAM policy checkpoints**
- GSM: only `n8n-compute@…` and **you** should have **Accessor/Admin** as needed.  
- GCS: bucket-level access only to project editors/owners + VM’s runtime identity (via gcloud on the VM).  
- Compute: restrict SSH to IAP IP range (already done) and **no external IP** on the VM.

---

## B) Resource-to-Principal Matrix

| Resource | Principal(s) | Access |
|---|---|---|
| `secrets.encryption_key_name` (GSM) | `n8n-compute@…` (Accessor), `harold@…` (Admin) | Read at runtime; admin for rotations |
| `storage.backup_bucket` (GCS) | Project editors/owners (incl. `harold@…`); VM via `gcloud` | Write backups/read restores |
| VM `gcp.vm` | `harold@…` via IAP SSH | Host ops |
| Cloud Monitoring channels | `harold@…` | Receives alerts |
| Cloudflare apps | `harold@…` | Manage SSO allowlist & bypass |

---

# Secrets SOP

> Names/locations come from `variables.yml`. Payloads live in **Bitwarden** (folder `bitwarden.folder`) and **GSM**.

## 1) Create or update a secret (GSM)
```bash
# Names from variables.yml
SECRET_NAME="$(yq '.secrets.encryption_key_name' variables.yml)"
PROJECT_ID="$(yq '.project.id' variables.yml)"

# Create new secret (first time)
printf '%s' '<SECRET_VALUE_FROM_BITWARDEN>' | gcloud secrets create "$SECRET_NAME" --replication-policy="automatic"   --data-file=- --project "$PROJECT_ID"

# Add a new version (update existing)
printf '%s' '<NEW_SECRET_VALUE_FROM_BITWARDEN>' | gcloud secrets versions add "$SECRET_NAME" --data-file=- --project "$PROJECT_ID"
```

**Grant access (least privilege)**
```bash
SA="serviceAccount:n8n-compute@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud secrets add-iam-policy-binding "$SECRET_NAME"   --member="$SA" --role="roles/secretmanager.secretAccessor" --project "$PROJECT_ID"
```

## 2) Sync to the VM (n8n)
> Keeps `.env` aligned with GSM without printing the value.

```bash
VM="$(yq '.gcp.vm' variables.yml)"; ZONE="$(yq '.gcp.zone' variables.yml)"
SECRET_NAME="$(yq '.secrets.encryption_key_name' variables.yml)"

# Update .env on VM with latest GSM version and restart n8n
gcloud compute ssh "$VM" --zone="$ZONE" --  'KEY="$(gcloud secrets versions access latest --secret '"$SECRET_NAME"' | tr -d "\n")"   && sudo sed -i "s|^N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=${KEY}|" /srv/n8n/.env   && cd /srv/n8n && docker compose up -d'
```

## 3) Verify runtime matches GSM (no secret exposure)
```bash
# Hash of GSM value
gcloud secrets versions access latest --secret "$(yq '.secrets.encryption_key_name' variables.yml)" | sha256sum | cut -c1-12

# Hash of on-disk n8n settings file (VM)
gcloud compute ssh "$(yq '.gcp.vm' variables.yml)" --zone="$(yq '.gcp.zone' variables.yml)" --  "sudo awk -F'\"' '/encryptionKey/ {print \$4}' /srv/docker/volumes/n8n_n8n_data/_data/config | sha256sum | cut -c1-12"
# Hashes must match.
```

## 4) Emergency rotation (compromise suspected)
1. Generate a new strong value (store in Bitwarden).  
2. Add as a **new GSM version** (do not delete old yet).  
3. Sync to VM (`.env` update + `docker compose up -d`).  
4. Verify app starts and health returns 200.  
5. Invalidate old version if required (disable or destroy).  
6. Record rotation in **Runbook change log**.

## 5) Audit reads & IAM (periodic)
```bash
# Who can read secrets (project-wide)
gcloud projects get-iam-policy "$(yq '.project.id' variables.yml)"   --flatten="bindings[].members"   --format="table(bindings.role, bindings.members)"   --filter='bindings.role:("roles/secretmanager.admin","roles/secretmanager.secretAccessor","roles/secretmanager.viewer")'

# Recent secret reads (Logs Explorer filter)
# protoPayload.methodName="google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion"
# protoPayload.resourceName:"secrets/$(yq '.secrets.encryption_key_name' variables.yml)"
```

**Bitwarden hygiene**
- Items to keep up to date (names only in git):  
  `N8N_ENCRYPTION_KEY`, `POSTGRES_PASSWORD`, `CLOUDFLARE_API_TOKEN`, `GITHUB_PAT` (see `bitwarden.items`).  
- Add a short note in each item describing **where it’s used** and the **update procedure**.
