# Architecture Overview

> Constants come from `variables.yml` (do not hardcode). Health URL: see `n8n.health_url`.

```mermaid
flowchart LR
  %% --- Internet ---
  subgraph internet[Internet]
    U[Users and external services]
    GCM[Google Cloud Monitoring\nUptime Check]
  end

  %% --- Cloudflare ---
  subgraph cf[Cloudflare]
    CF[Cloudflare Access SSO]
    DN[DNS n8n.rakista.com]
  end

  %% --- GCP project ---
  subgraph gcp[Project: rakista-417013]
    subgraph vpc[VPC]
      VM[n8n VM\nsee variables.yml -> gcp.vm]
      TUN[cloudflared tunnel\n/etc/cloudflared/config.yml]
    end

    subgraph docker[Docker on VM]
      N8N[n8n container\nversion -> n8n.version]
      PG[(Postgres 15)]
      VOL[Data volume\n/srv/docker/volumes/n8n_data]
    end

    GCS[(GCS backup bucket\n-> storage.backup_bucket)]
    SM[(Secret Manager\n-> secrets.encryption_key_name)]
    SNAP[Disk snapshot policy\n-> gcp.snapshot_policy]
  end

  %% --- Edges ---
  U -->|HTTPS| DN
  DN --> CF
  CF -->|SSO protected UI| TUN
  GCM -->|HTTPS GET /webhook/healthz| TUN
  TUN -->|http://localhost:5678| N8N
  N8N --- PG
  N8N --- VOL
  VM --> SNAP
  VM -->|cron: pg_dump + tar| GCS
  N8N -->|reads key at startup| SM
```

## Five Key Points

1. **Ingress** — Public DNS (`cloudflare.domain`) points to Cloudflare; UI is **SSO**; `/webhook*` is **Bypass** for probes/integrations.  
2. **n8n on Docker** — Single container on VM `gcp.vm`, listening on **localhost:5678**; Cloudflared exposes it.  
3. **Secrets** — `secrets.encryption_key_name` in GSM; value never in git. n8n reads it at startup via env.  
4. **Data & backups** — Postgres 15 + n8n data volume under `/srv/docker/volumes/n8n_data`; backups to `storage.backup_bucket` (cron `pg_dump` and tar).  
5. **Resilience** — Disk snapshot policy `gcp.snapshot_policy` on `gcp.disk_name`; uptime check hits `n8n.health_url` every minute.
