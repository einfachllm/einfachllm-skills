# Deploy

## Docker Compose (local / single host)

`docker-compose.yml` brings up Postgres 16 + the gateway:

```bash
EINFACHLLM_MASTER_KEY=secret OPENROUTER_API_KEY=sk-or-... docker compose up --build
```

- Gateway on `:8000`; Postgres data in the `pgdata` volume.
- The repo `./config` is mounted **writable** at `/app/config`, and the gateway
  runs as the host user (`user: "${UID:-1000}:${GID:-1000}"`) so the SPA config
  editor and `ectl config-set` can save. For a strict GitOps posture append
  `:ro` to the mount — the editor then degrades to view-only.
- Both env vars are optional for a smoke test (defaults keep it bootable). Set
  a real `EINFACHLLM_MASTER_KEY` before minting keys.
- Smoke: `curl localhost:8000/health`; admin UI at `localhost:8000/admin/app`;
  Swagger at `/docs`.

The image (`docker/Dockerfile`) is a two-stage, non-root build
(`uv sync --frozen --no-dev`) with a stdlib healthcheck. Its entrypoint runs
`alembic upgrade head` and then starts the gateway; set `RUN_MIGRATIONS=false`
to skip that when an external step owns migrations.

## Kubernetes (Helm)

The chart is at `deploy/helm/einfachllm/`. It deploys the gateway `Deployment`
+ `Service`, renders the config YAMLs into a `ConfigMap` mounted at
`EINFACHLLM_CONFIG_DIR`, holds secrets in a single `Secret` (`envFrom`), and
runs `alembic upgrade head` as an init container before the gateway starts.

Postgres is an **optional** bitnami subchart (`postgresql.enabled`, default
`true`) meant for local clusters. Production should disable it.

Prerequisites: `kubectl`, `helm` >= 3, and a cluster (kind / minikube / k3s).

### 1. Build and load the image

No public registry yet — build locally and load into the cluster (the chart
defaults to `einfachllm:dev`, `pullPolicy: IfNotPresent`):

```bash
docker build -f docker/Dockerfile -t einfachllm:dev .

kind load docker-image einfachllm:dev      # kind
minikube image load einfachllm:dev         # minikube
```

### 2. Fetch the Postgres subchart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency build deploy/helm/einfachllm
```

Required **even when `postgresql.enabled=false`** — Helm resolves declared
dependencies before install.

### 3. Install

```bash
helm install einfachllm deploy/helm/einfachllm \
  --set secrets.masterKey=secret \
  --set secrets.providerKeys.MISTRAL_API_KEY=unused
```

Every `${VAR}` referenced in `config.providers.yaml` needs an entry under
`secrets.providerKeys` — the config loader fail-fasts at startup on an unset
one. The default `providers.yaml` references `${MISTRAL_API_KEY}`, which is why
it appears above even for a smoke test.

### 4. Smoke test

```bash
kubectl rollout status deploy/einfachllm
kubectl port-forward svc/einfachllm 8000:8000 &

curl -fsS localhost:8000/health        # -> {"status":"ok",...}

export EINFACHLLM_BASE_URL=http://localhost:8000
export EINFACHLLM_MASTER_KEY=secret
uv run python tools/ectl.py mint --team platform --user alice --name smoke
```

A successful `mint` (an `ein_…` key) proves the whole path: image → migrations
→ Postgres → master key → config. Treat the returned secret as a secret.

## Production: external Postgres

```bash
helm install einfachllm deploy/helm/einfachllm \
  --set postgresql.enabled=false \
  --set externalDatabase.url='postgresql+asyncpg://user:pass@db.internal:5432/einfachllm' \
  --set secrets.masterKey="$EINFACHLLM_MASTER_KEY" \
  --set secrets.providerKeys.MISTRAL_API_KEY="$MISTRAL_API_KEY"
```

Prefer a pre-created Secret carrying `EINFACHLLM_MASTER_KEY`, `DATABASE_URL`,
and every provider `${VAR}`, referenced with
`--set secrets.existingSecret=my-secret` — the chart then creates no Secret of
its own and no credential passes through a shell history or a values file.

Read `externalDatabase.url` as the **async** DSN (`postgresql+asyncpg://`).
`pg_dump` / `pg_restore` want the same DSN without the `+asyncpg` marker.

## Config

The chart's `config.*` values are a verbatim copy of the repo-root
`config/*.yaml`. Keep them in sync, or override per environment with a values
file:

```bash
helm install einfachllm deploy/helm/einfachllm --values prod-values.yaml
```

A `helm upgrade` that changes config or secrets rolls the pods automatically
(checksum annotations). To apply a config change *without* a rollout, wait for
the ConfigMap to propagate to the pod filesystem (kubelet sync, up to ~1 min;
the swap is atomic, so no reader sees a half-written file) and then trigger
`POST /admin/reload`. That hits only the one pod the Service routes to — call
it per pod for a fleet-wide reload.

`DATABASE_URL` and the shared HTTP client's default timeouts still require a
pod restart; routing, budgets, and limits apply on reload.
