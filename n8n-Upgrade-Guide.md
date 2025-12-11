# n8n Upgrade Guide (Docker on GCE VM)

Production n8n at **https://n8n.rakista.com** on **GCE VM `n8n-vm-01`**.

---

## 🔎 Quick Upgrade Checklist (TL;DR)

1. **Decide target version** (e.g. `1.122.5`) from n8n release notes.
2. **Create VM Image** of `n8n-vm-01`
   - Name: `n8n-vm-01-pre-upgrade-YYYYMMDD`
3. **Update repo**
   - `variables.yml → n8n.version = "<new_version>"`
   - Add entry in `Runbook.md` change log.
4. **SSH to VM**
   - `gcloud compute ssh n8n-vm-01 --zone=<zone>`
5. **Run upgrade helper**
   - `cd /srv/n8n`
   - `./update-n8n.sh` → enter `<new_version>`
6. **Verify**
   - `docker exec -it n8n-n8n-1 n8n --version`
   - `curl 127.0.0.1:5678/rest/settings | grep -i version`
   - `curl -i https://n8n.rakista.com/webhook/healthz | head -n 5`
   - Incognito UI → About n8n → run 1 simple Prod workflow.
7. **If broken → rollback**
   - Either restore VM from **GCE Image**, or
   - Switch Docker image tag back to previous version and `docker compose up -d`.

---

## 0. Context

- Host: **GCE VM `n8n-vm-01`** (same project as WordPress infra).
- Stack: **Docker Compose** running:
  - `postgres` (Postgres 15)
  - `n8n` (`n8nio/n8n:<version>`)
- Compose + script live in: `/srv/n8n`
- Data volumes:
  - DB: `n8n_pgdata`
  - App data: `n8n_n8n_data` mounted at `/srv/docker/volumes/n8n_n8n_data/_data`
- Backups: `/srv/backup` (DB dumps and app-data tarballs).

This guide standardizes upgrades so they are **repeatable**, **backed up**, and **documented**.

---

## 1. Create GCE VM Image (pre-upgrade backup)

### 1.1 Console (GUI)

1. Go to **Google Cloud Console → Compute Engine → VM instances**.
2. Select **`n8n-vm-01`**.
3. Click **More actions → Create image**.
4. Configure:
   - **Name:** `n8n-vm-01-pre-upgrade-YYYYMMDD`
   - **Source disk:** boot disk of `n8n-vm-01`
   - Keep defaults unless you need special storage options.
5. Click **Create** and wait until the image is ready.

Use this image to recreate the VM if the upgrade badly breaks the system.

### 1.2 CLI (gcloud)

From Cloud Shell or a machine with gcloud configured:

    gcloud compute images create n8n-vm-01-pre-upgrade-$(date +%Y%m%d) \
      --source-disk=n8n-vm-01 \
      --source-disk-zone=asia-southeast1-b

> Adjust zone if `n8n-vm-01` is in a different zone.

---

## 2. Update the Repo (`variables.yml` + Runbook)

In the **`rakista-ops`** repo:

1. Open `variables.yml`.
2. Find:

       n8n:
         version: "OLD_VERSION"

3. Change to:

       n8n:
         version: "NEW_VERSION"

4. In `Runbook.md` → Change Log, add, for example:

       - 2025-12-05 – Upgraded n8n (n8n.rakista.com) from 1.115.3 → 1.122.5 using update-n8n.sh.

5. Commit:

       git add variables.yml Runbook.md n8n-Upgrade-Guide.md
       git commit -m "docs: document n8n upgrade and bump n8n.version to NEW_VERSION"
       git push

---

## 3. SSH to `n8n-vm-01`

From Cloud Shell or local:

    gcloud compute ssh n8n-vm-01 --zone=asia-southeast1-b

Once logged in:

    cd /srv/n8n
    ls
    # Expect: docker-compose.yml, update-n8n.sh, etc.

If `update-n8n.sh` does not exist yet, create it using the script in the Appendix.

---

## 4. Run the Upgrade Helper Script

`update-n8n.sh` handles:

- DB backup (`pg_dump` → `/srv/backup/n8n-<DATE>.sql.gz`)
- App data backup (`tar` → `/srv/backup/n8n-data-<DATE>.tgz`)
- Updating the `docker-compose.yml` image tag
- Pulling the new n8n image
- Restarting `n8n` + showing the running version

### 4.1 Execute

On the VM:

    cd /srv/n8n
    ./update-n8n.sh

When prompted:

    Enter new n8n version (e.g. 1.123.0):

Type e.g.:

    1.122.5

The script will:

1. Back up DB and data into `/srv/backup`.
2. Edit `docker-compose.yml` `image: n8nio/n8n:<tag>`.
3. Run `docker compose pull n8n`.
4. Run `docker compose up -d`.
5. Print the version from inside the container.

If the script errors, read the message (most likely disk space or network), fix, and re-run.

---

## 5. Post-upgrade Verification

### 5.1 Check container version

    docker exec -it n8n-n8n-1 n8n --version
    # Expect: NEW_VERSION (e.g. 1.122.5)

If the container name differs, find it:

    docker ps | grep n8n

### 5.2 Check REST settings API

    curl -sS http://127.0.0.1:5678/rest/settings | grep -i version

Look for:

    "versionCli":"NEW_VERSION"

### 5.3 Health endpoint

    curl -i https://n8n.rakista.com/webhook/healthz | head -n 5

Expect `HTTP/2 200` or `HTTP/1.1 200`.

### 5.4 UI & workflow test

1. Open a **new incognito/private window**.
2. Visit `https://n8n.rakista.com`.
3. Log in (via Cloudflare Access).
4. Open **About n8n** → verify version is `NEW_VERSION`.
5. Pick a **simple Prod workflow** (ping / test flow).
6. Run **Test** and confirm:
   - No missing node types.
   - Credentials work.
   - Webhooks / HTTP nodes behave normally.

If you still see an old “You’re on 1.115.3…” banner but About shows `NEW_VERSION`, it’s usually stale cache. Dismiss it or wait for n8n’s version check to refresh.

---

## 6. Rollback Options

### 6.1 Rollback via GCE VM Image (full VM restore)

Use if the whole VM is broken or misconfigured.

1. In GCP Console → **Compute Engine → Images**.
2. Find `n8n-vm-01-pre-upgrade-YYYYMMDD`.
3. Click **Create VM instance** from that image.
4. Recreate VM config (zone, machine type, network, static IP, etc.).
5. Point DNS / Cloudflare back to the restored VM.
6. Decommission the broken VM once the restored one is verified.

### 6.2 Rollback via Docker Image (only n8n version)

Use if VM is fine but new n8n version is buggy.

1. Edit `/srv/n8n/docker-compose.yml`:

       n8n:
         image: n8nio/n8n:OLD_VERSION

2. Apply:

       cd /srv/n8n
       docker compose pull n8n
       docker compose up -d

3. Re-run the verification steps (Section 5).

### 6.3 Full data restore from backups

Use if DB or workflow data is corrupted and you have good backups in `/srv/backup` (or GCS).

High level:

1. Stop containers (`docker compose down`).
2. Restore DB from `/srv/backup/n8n-<DATE>.sql.gz` into Postgres.
3. Restore app data from `/srv/backup/n8n-data-<DATE>.tgz` into `/srv/docker/volumes/n8n_n8n_data/_data`.
4. `docker compose up -d`.
5. Verify using Section 5.

(For exact restore commands, see **Backup-Restore-DR-Guide.md**.)

---

## Appendix – `update-n8n.sh` Script

This script should live at `/srv/n8n/update-n8n.sh` on `n8n-vm-01`.

    #!/usr/bin/env bash
    set -euo pipefail

    echo "=== n8n upgrade helper ==="
    read -rp "Enter new n8n version (e.g. 1.123.0): " NEW_VERSION

    if [[ -z "${NEW_VERSION}" ]]; then
      echo "No version entered, aborting."
      exit 1
    fi

    cd /srv/n8n

    DATE="$(date -u +%F-%H%MZ)"
    BACKUP_DIR="/srv/backup"
    DBPASS="$(docker exec n8n-postgres-1 printenv POSTGRES_PASSWORD)"

    echo "==> Backing up DB and data to ${BACKUP_DIR} (${DATE})"
    mkdir -p "${BACKUP_DIR}"

    # DB backup
    docker exec -e PGPASSWORD="${DBPASS}" -i n8n-postgres-1 \
      pg_dump -U n8n -d n8n | gzip > "${BACKUP_DIR}/n8n-${DATE}.sql.gz"

    # App data backup
    sudo tar -czf "${BACKUP_DIR}/n8n-data-${DATE}.tgz" \
      -C /srv/docker/volumes/n8n_n8n_data/_data .

    echo "==> Updating docker-compose.yml image tag to n8nio/n8n:${NEW_VERSION}"
    sudo sed -i "s|\(image: n8nio/n8n:\).*|\1${NEW_VERSION}|" docker-compose.yml

    echo "==> Pulling image and restarting containers"
    docker compose pull n8n
    docker compose up -d

    echo "==> Checking running version"
    docker exec -it n8n-n8n-1 n8n --version || true

    echo "=== Done. Remember to test a simple workflow in the UI. ==="

Make it executable:

    cd /srv/n8n
    chmod +x update-n8n.sh

---

**End of n8n-Upgrade-Guide.md**
