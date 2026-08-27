# Config (providers, models, tenancy, settings, pricing)

Routing config lives in YAML and is loaded into memory (GitOps) — there is no
DB-backed provider store. To change providers/models/teams, edit the YAML and
**reload** the gateway — no restart needed (see "Reload" below).

## View (sanitized)

```bash
uv run python tools/ectl.py config-show [--dir config]
```

Reads the raw YAML (keeps `${ENV}` placeholders, redacts any literal secret).
Never `cat` a config whose secrets have been interpolated.

## Validate before reload

```bash
uv run python tools/ectl.py config-validate [--dir config]
```

Runs the full Pydantic validation (`einfachllm.config.loader.load_config`):
unique ids, cross-file references, region/tenancy rules, env interpolation. Fix
any reported error before the user reloads the gateway.

## Remote get/set (no filesystem access needed)

When the gateway runs elsewhere (Docker/K8s) and you can't touch its `config/`
directly, drive the raw-config endpoints instead:

```bash
uv run python tools/ectl.py config-get                  # list files + writable?
uv run python tools/ectl.py config-get models.yaml      # print one (raw) to stdout
uv run python tools/ectl.py config-set models.yaml --file new.yaml
uv run python tools/ectl.py config-set models.yaml --stdin < new.yaml
uv run python tools/ectl.py config-edit models.yaml     # $EDITOR round-trip
```

Files travel **raw** (`${ENV_VAR}` references are never resolved — secrets never
pass through the wire or your terminal). The server validates the **whole**
config set before writing; an invalid file is rejected with the validation
message and disk stays untouched. A save is not applied until `ectl reload`.
If the gateway's config dir is read-only (`:ro` mount / GitOps), `config-set`
fails with the server's 409 — commit via Git instead.

## Editing safely (agent workflow)

1. Propose the YAML change as a **diff** for the user to review.
2. Keep secrets as `${ENV_VAR}` references — never inline a real key.
3. Preserve sovereignty: do not weaken a provider's `region` tag, an org/key
   `allow_non_eu`, or budgets unless the user explicitly asks.
4. Validate: locally `config-validate` on the edited directory, or remotely by
   `config-set` (the server rejects invalid config before writing).
5. Apply with `ectl reload` (no restart) — see below.

## Reload (no restart)

```bash
uv run python tools/ectl.py reload
```

Re-reads the YAML and atomically hot-swaps it on the running gateway (`POST
/admin/reload`, master key). The new config is validated first; on error the
running config stays live and the command reports the validation error. Edit
the files, then reload — don't reload mid-edit.

**Applies immediately:** providers, models, aliases, pricing, tenancy
(orgs/teams/users are upserted), team RPM/TPM, provider concurrency caps +
breaker thresholds, budget limits, and per-request settings.

**Still needs a restart:** `DATABASE_URL` / DB pool and the HTTP client
timeouts (not config-derived).

**Never applied by reload:** tenancy *deletions* — removing an org/team/user
from YAML leaves the DB row (and its keys) intact, by design; deletion is an
explicit admin action.

In Kubernetes, `POST /admin/reload` hits only the one pod the Service routes to.
For a fleet-wide reload, trigger every pod (after the ConfigMap has propagated to
the pod filesystem).

## Adding an EU provider

The admin UI (`/admin/app/providers`) lists an EU provider catalog with
copy-paste YAML snippets (Mistral, Scaleway, OVHcloud, IONOS, Aleph Alpha,
Ollama, vLLM). Paste a snippet under `providers:` in `providers.yaml`, add a
model entry in `models.yaml`, then `config-validate`.

## Pricing & currency (`models.yaml` + `settings.yaml`)

Per-model prices are vendored on each `models.yaml` entry (never fetched at
runtime — sovereignty rule). Each entry has optional `input_price`/`output_price`
in **per-1M-token** units. An entry with no prices is unpriced (cost 0); a
missing side charges only the priced side.

```yaml
models:
  - name: mistral-large-eu
    provider: mistral
    upstream_model: mistral-large-latest
    input_price: 2.0
    output_price: 6.0
```

Prices belong to the *entry*, not the upstream model id — so the same model
served both locally (free) and by a cloud provider (paid) can carry two
different prices.

`settings.currency` is a single deployment-wide label (no conversion): all
prices, stored costs, and budgets are in it, and the admin UI shows its symbol.
To switch currency, re-author the prices and change `currency`. It defaults to
`EUR`.

## Per-model rate limits (`settings.yaml`)

Global ceilings for one model across **all** keys and teams — use them to protect
an expensive or scarce model that per-key/per-team limits can't bound in
aggregate. Keyed by the public model/alias name; both fields are optional.

```yaml
model_rate_limits:
  mistral-large-eu:
    total: 600         # all requests/min (counted on entry; over → 429)
    successful: 500    # only 2xx responses/min (over → 429 on the next entry)
```

Enforced at router entry (after auth + allowlist, before the upstream call); an
exceed returns HTTP 429 and audits `action="rate_limit.model"`. Counters are
per-pod fixed 60s windows (same bounded-stale trade-off as the key/team limiter:
with N replicas the effective ceiling is up to N×). Changes apply on
`ectl reload`.
