# Architecture Overview

```mermaid
flowchart LR
  subgraph Internet
    U[Users & external services]
    GCM[Google Cloud Monitoring\nUptime Check]
  end

  subgraph Cloudflare
    CF[Cloudflare Access (SSO)]
    DN[DNS for n8n.rakista.com]
  end

  subgraph GCP[rakista-417013]
    subgraph VPC
      VM[n8n VM: see variables.yml → gcp.vm]
      TUN[cloudflared tunnel\n/etc/cloudflared/config.yml]
    end

    subgraph Docker on VM
      N8N[n8n (container)\nversion → n8n.version]
      PG[(Postgres 15)]
      VOL[(/srv/docker/volumes\nn8n_data)]
    end

    GCS[(GCS backup bucket\n→ storage.backup_bucket)]
    SM[(Secret Manager\n→ secrets.encryption_key_name)]
    SNAP[Disk snapshot policy\n→ gcp.snapshot_policy]
  end

  %% Edges
  U -->|HTTPS| DN
  DN --> CF
  CF -->|SSO-protected UI| TUN
  GCM -->|HTTPS GET /webhook/healthz| TUN
  TUN -->|http://localhost:5678| N8N
  N8N --- PG
  N8N --- VOL
  VM --> SNAP
  VM -->|cron: pg_dump + tar| GCS
  N8N -->|reads secret at startup| SM
```

## Five Key Points

1) **Entry & Auth**  
   - Traffic to `cloudflare.domain` (see `variables.yml`) lands on **Cloudflare Access**.  
   - UI is **Allow (SSO)**; **`/webhook*` is Bypass** so external triggers & uptime checks work.

2) **Tunnel & App**  
   - **cloudflared** runs on the VM (locally managed at `/etc/cloudflared/config.yml`) and forwards to **n8n** on `127.0.0.1:5678`.  
   - n8n runs in Docker (`n8n.version`), Postgres 15 as sidecar.

3) **Secrets & Config**  
   - **N8N_ENCRYPTION_KEY** is stored in **GCP Secret Manager** (`secrets.encryption_key_name`), with payload sources in **Bitwarden** (`bitwarden.*`).  
   - Runtime config is pulled into `.env`; values in docs reference **`variables.yml`** (single source of truth).

4) **Backups & Snapshots**  
   - Nightly **DB dump** + **n8n data tar** pushed to **GCS** (`storage.backup_bucket`).  
   - Persistent disk uses **snapshot policy** (`gcp.snapshot_policy`) for fast rollbacks.

5) **Monitoring & Health**  
   - Tiny n8n workflow serves `GET /webhook/healthz` returning `200 ok`.  
   - **GCP Uptime Check** probes every **`monitoring.uptime.frequency`** with **`monitoring.uptime.timeout`**, alerting after **`monitoring.uptime.failure_window`** to channels in `monitoring.channels.*` (plus `monitoring.chat_channel_id` reference).
