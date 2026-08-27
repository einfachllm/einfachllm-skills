# Backup & restore

All persistent state lives in **one place: Postgres** — API keys (hashed),
cumulative spend, usage and audit logs, alert dedup state. Routing config is
YAML (in git or Helm values) and pods are stateless. So disaster recovery is
exactly: the container image + your values/config + a database backup.

## External / managed Postgres (production)

Use the platform's native backups (PITR / WAL archiving where available) — the
gateway needs nothing special. A plain logical backup also works:

```bash
pg_dump --format=custom --file=einfachllm.dump "$POSTGRES_DSN"
pg_restore --clean --if-exists --dbname="$POSTGRES_DSN" einfachllm.dump
```

`pg_dump` / `pg_restore` take a regular `postgresql://` DSN — drop the
`+asyncpg` marker from the application URL.

## In-cluster Postgres (bitnami subchart)

The bitnami PVC is the only stateful volume, and **a PVC is not a backup**.
Snapshot the data out of the cluster on a schedule:

```bash
# Ad-hoc logical dump (release `einfachllm`, default chart credentials):
kubectl exec einfachllm-postgresql-0 -- env PGPASSWORD=einfachllm \
  pg_dump --format=custom -U einfachllm einfachllm > einfachllm.dump
```

Schedule that as a Kubernetes `CronJob` writing to off-cluster storage, or run
[wal-g](https://github.com/wal-g/wal-g) / pgBackRest for continuous WAL
archiving if you need point-in-time recovery.

Store dumps **encrypted and EU-resident**: they contain usage metadata and key
hashes. They never contain prompt or response content — the gateway does not
log it.

## Restore into a fresh install

```bash
helm install einfachllm deploy/helm/einfachllm --set migrations.enabled=false ...
kubectl exec -i einfachllm-postgresql-0 -- env PGPASSWORD=einfachllm \
  pg_restore --clean --if-exists -U einfachllm -d einfachllm < einfachllm.dump
helm upgrade einfachllm deploy/helm/einfachllm ...   # re-enable migrations, roll pods
```

Restoring a dump taken on an older app version is fine: the next rollout's
migration step brings the schema to head.

`pg_restore --clean` drops objects before recreating them. Confirm with the
user before pointing it at anything that isn't a fresh database.

## After any restore — tell the user this

Restoring rewinds security-relevant state. Three consequences, every time:

- **Spend is rewound.** The `spend` columns are canonical for budget
  enforcement, so spend accrued after the backup was taken is forgotten —
  budgets re-open by that amount.
- **Keys minted after the backup no longer exist.** Their secrets stop working;
  re-mint and redistribute them.
- **Keys revoked after the backup are live again.** This is the dangerous one:
  a key someone revoked for a reason is valid until you revoke it again.
  **Re-revoke them first**, before the gateway serves traffic.

Reconstruct the revocation list from `audit_logs` in the *pre-restore* database
if you still have it, or from whatever ticket trail recorded the revocations.
