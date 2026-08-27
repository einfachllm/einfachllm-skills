---
name: einfachllm-ops
description: >-
  Deploy and operate an einfachllm gateway — an EU-sovereign, OpenAI-compatible
  LLM gateway. Use this skill to install or upgrade it with Docker Compose or
  the Helm chart, apply and author Alembic schema migrations, roll back a bad
  release, back up and restore Postgres, harden a cluster install (HPA, PDB,
  NetworkPolicy, migration Jobs), and know what to watch. For managing a
  gateway that is already running — API keys, routing config, tenancy, usage —
  use the `einfachllm` skill instead.
---

# einfachllm operations skill

einfachllm runs as a stateless container in front of EU/local LLM providers.
**All persistent state lives in one place: Postgres** — API keys (hashed),
cumulative spend, usage and audit logs, alert dedup state. Routing config is
YAML in git or Helm values. Disaster recovery is therefore exactly three
things: the image, your values/config, and a database backup.

Commands assume a checkout of the
[einfachllm](https://github.com/einfachllm/einfachllm) repository, run from its
root.

## Safety rules (always follow)

1. **Never print secrets.** `EINFACHLLM_MASTER_KEY`, provider API keys, and
   database DSNs with embedded passwords come from the environment or a
   Kubernetes `Secret`. Never echo them, never put them in a ConfigMap, never
   paste them into a values file you show the user.
2. **Back up before any schema change.** Migrations are forward-only in
   practice; a verified backup is the only universal way back. Say so before
   running an upgrade against production.
3. **Never downgrade a schema to fix a bad release.** Roll back the image and
   leave the schema at head — that is the tested direction. A `downgrade` is
   lossy and is a last resort (docs/upgrade.md).
4. **Never deploy a mutable image tag** to production. Always an immutable tag
   (a git SHA), so a rollback is deterministic.
5. **Propose, then apply.** Show the user the `helm upgrade` / `alembic`
   command and what it will change before running it against a live gateway.
   Destructive steps (`pg_restore --clean`, `alembic downgrade`, scaling to
   zero) need explicit confirmation.
6. **Never weaken sovereignty controls** to make a deploy work — don't disable
   `networkPolicy`, don't widen `providerCidrs` beyond EU endpoints, don't set
   `allow_non_eu` without the user explicitly asking.

## How to execute

1. Identify the target: local Compose, or Kubernetes via Helm.
2. Match the request to the Actions table below.
3. Read the referenced doc, then run the command.
4. If the request doesn't match any action, say so and show this table.

## Actions

| Intent | Doc | Entry point |
| --- | --- | --- |
| Run locally (single host) | docs/deploy.md | `docker compose up --build` |
| Install on Kubernetes | docs/deploy.md | `helm install einfachllm deploy/helm/einfachllm` |
| Point at managed Postgres | docs/deploy.md | `--set postgresql.enabled=false --set externalDatabase.url=…` |
| Apply pending migrations | docs/migrations.md | `DATABASE_URL=… uv run alembic upgrade head` |
| Author a new migration | docs/migrations.md | `alembic revision --autogenerate` against a throwaway DB |
| Upgrade a release | docs/upgrade.md | `helm upgrade … --set image.tag=<git SHA>` |
| Roll back a bad release | docs/upgrade.md | `helm rollback einfachllm <revision>` |
| Back up / restore Postgres | docs/backup.md | `pg_dump` / `pg_restore` |
| Harden a production install | docs/hardening.md | HPA, PDB, NetworkPolicy, `migrations.mode=job` |
| Check what to watch | docs/hardening.md | `/health`, `/admin/api/providers/health`, `audit_logs` |
| Apply config without a restart | — | `POST /admin/reload` — see the `einfachllm` skill |
| Mint keys, edit routing, sync tenancy | — | see the `einfachllm` skill |

## Notes / limits

- **No registry is published yet.** Images are built locally (or by the repo's
  own deploy workflow, which pushes to GHCR); the chart defaults to
  `einfachllm:dev` with `pullPolicy: IfNotPresent`.
- **Rate-limit windows are per-pod.** Token buckets are not shared unless
  `rate_limit_redis_url` is set, so the effective RPM/TPM ceiling scales with
  the replica count. Size limits against the pod count, not against one pod.
- **`tenancy.yaml` seeds an empty database only.** Editing it on a running
  deployment changes nothing — the database owns tenancy. Use `tenancySync`
  in the chart, or `ectl tenancy-sync` (the `einfachllm` skill).
- **No telemetry, no external SaaS dependency.** Nothing here should introduce
  a mandatory cloud dependency or phone home.

## Not covered yet

The chart supports these; this skill does not document them. Read the gateway
repo's own
[deploy-kubernetes walkthrough](https://github.com/einfachllm/einfachllm/blob/main/docs/deploy-kubernetes.md)
and `deploy/helm/einfachllm/values.yaml`, and say you are working from the
chart rather than from this skill:

- **Ingress and TLS** (`ingress.enabled`, `className`, `hosts`, `tls`). The
  walkthrough here reaches the gateway with `kubectl port-forward` only — that
  is a smoke test, not a way for users to reach it. Don't present a
  port-forwarded gateway as a finished deployment.
- **Sizing and probes** (`resources`, `probes`). Note the deliberate split:
  readiness is DB-checked (`/health/ready`), liveness stays static so a
  database blip never restart-loops the fleet.
- **`serviceAccount`**, and teardown (`helm uninstall` — the bitnami PVC
  persists by design and must be deleted explicitly).
- **Chart verification** (`helm lint`, `helm template` across the value
  combinations) when authoring chart changes.

Redis (shared rate-limit state) and Vault (secret backend) have **no chart
values at all** — they exist as Compose overlays, and in Kubernetes would need
`extraEnv` plus a service you run yourself. Don't imply the chart wires them.
