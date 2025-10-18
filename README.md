# Rakista Ops Docs

This repo contains the **single source of truth** for Rakista infra docs.  
Constants live in `variables.yml`. The detailed operational runbook is `Runbook.md`.

> ❗ No secrets in git. Secret payloads live in **Bitwarden** (folder: `variables`) and **GCP Secret Manager (GSM)`**.

## Documents

- **Runbook** — end‑to‑end setup & operations: [`Runbook.md`](Runbook.md)
- **Variables** — live constants referenced by all docs: [`variables.yml`](variables.yml)

### Focused guides
- **Architecture Overview**: [`docs/Architecture-20251018.md`](docs/Architecture-20251018.md)
- **On-call Quick Sheet**: [`docs/On-call-Quick-Sheet-20251018.md`](docs/On-call-Quick-Sheet-20251018.md)
- **Backup & Restore / DR Guide**: [`docs/Backup-Restore-DR-Guide-20251018.md`](docs/Backup-Restore-DR-Guide-20251018.md)
- **Access Matrix + Secrets SOP**: [`docs/Access-Matrix-and-Secrets-SOP-20251018.md`](docs/Access-Matrix-and-Secrets-SOP-20251018.md)
- **n8n Dev Conventions**: [`docs/n8n-Dev-Conventions-20251018.md`](docs/n8n-Dev-Conventions-20251018.md)
- **Integration & Webhook Catalog**: [`docs/Integration-and-Webhook-Catalog-20251018.md`](docs/Integration-and-Webhook-Catalog-20251018.md)

## Repo layout

```
/                Runbook.md, variables.yml, README.md
/docs            Architecture & playbooks (Markdown only)
/workflows       (optional) Exported n8n workflow JSONs (no credentials)
```

## Working model

- Treat **`variables.yml`** as the only place for live constants (project IDs, domains, VM names).  
- All docs should reference variables by path, e.g., `variables.yml → gcp.zone`.
- When a constant changes (IP, domain, bucket), update it **once** in `variables.yml`, then open a PR that touches any docs where wording depends on it.

## PRs & branch rules

- Commit directly to `main` or use feature branches.  
- For larger changes, open a PR with the provided **PR template**.  
- Keep commit messages short and descriptive (e.g., `docs: add DR verification step`).

## Contact

- Owner: **harold@rakista.com**  
- Alerts: see `variables.yml → monitoring.channels.*`
