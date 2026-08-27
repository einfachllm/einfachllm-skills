---
name: einfachllm
description: >-
  Manage an einfachllm gateway — an EU-sovereign, OpenAI-compatible LLM gateway.
  Use this skill to mint/revoke API keys, inspect usage and budgets, view and
  edit the YAML routing config (providers/models/tenancy/settings), reconcile
  orgs/teams/users declaratively, and check health. Routing config lives in
  YAML (GitOps); API keys live hashed in Postgres; tenancy lives in Postgres,
  reconcilable from a file via tenancy-sync.
---

# einfachllm management skill

einfachllm fronts EU/local LLM providers behind an OpenAI-compatible API, with
per-key model allowlists, non-EU egress enforcement (key **and** org ceiling),
usage metering, and key/team/org budgets.

All actions go through `ectl.py` — never call the admin API with raw curl or
hand-built HTTP. `ectl.py` ships with the gateway, not with this skill: run it
from the root of an [einfachllm](https://github.com/einfachllm/einfachllm)
checkout.

```bash
uv run python tools/ectl.py <action> [...]
```

Inside the gateway container the same script is at `/app/ectl.py`. See
`docs/setup.md` before the first command.

## Security rules (always follow)

1. **Never print or log API-key secrets** (`ein_…`). `mint` shows the secret
   exactly once by design; relay it to the user once and do not repeat it.
2. **Never reveal the master key.** It comes only from `EINFACHLLM_MASTER_KEY`
   in the environment; never echo it, put it in a file, or pass it on a command
   line that gets logged.
3. **Never print resolved provider credentials.** Use `config-show` (it reads raw
   YAML and keeps `${ENV}` placeholders / redacts secrets) — never `cat` a config
   that has interpolated secrets.
4. **Use `ectl.py`, not raw HTTP**, for every action it covers — always for
   anything that writes. The read-only stats endpoints have no `ectl` wrapper
   yet and are the sole exception (docs/query.md).
5. **Security decisions come from the key, never client input.** When editing
   config, never weaken `allow_non_eu` / region tags or budgets without the user
   explicitly asking.
6. **Edit config as a reviewable change**, then run `config-validate` before the
   user applies/reloads it. Routing config is YAML — propose a diff, don't mutate
   silently.

## How to execute

1. Read `docs/setup.md` for environment variables and prerequisites.
2. Match the user's request to the Actions table below.
3. Read the referenced doc, then run the `ectl.py` action.
4. If the request doesn't match any action, say so and show this table.

## Actions

| Intent | Doc | Command |
| --- | --- | --- |
| Check the gateway is up | docs/query.md | `ectl.py health` |
| List configured models/aliases | docs/query.md | `ectl.py models` |
| View the (sanitized) config | docs/config.md | `ectl.py config-show` |
| Validate config before reload | docs/config.md | `ectl.py config-validate` |
| Add/edit a provider or model | docs/config.md | edit YAML, then `config-validate` |
| Mint an API key | docs/keys.md | `ectl.py mint --team T --user U --name N` |
| Update a key's limits | docs/keys.md | `ectl.py update --id ID [--budget 100\|none] [--rpm 60\|none] [--tpm 100000\|none]` |
| Revoke an API key | docs/keys.md | `ectl.py revoke --id ID` |
| Reconcile orgs/teams/users from a file | docs/tenancy.md | `ectl.py tenancy-sync --file desired.yaml` |
| View usage / spend / budgets | docs/query.md | `GET /admin/api/stats/*`, or the dashboard at `/admin/app` |
| Deploy, migrate, upgrade, roll back | — | see the `einfachllm-ops` skill |

## Notes / limits

- Adding providers/models means editing YAML (`providers.yaml`, `models.yaml`)
  and reloading — there is no DB-backed provider store by design.
- Usage/spend has a JSON API (`/admin/api/stats/*`, `analytics:read`) but no
  `ectl` wrapper; the dashboard at `/admin/app` is the human-facing source.
- This skill manages a **running** gateway. Deployment, schema migrations,
  backup/restore, and upgrades belong to the `einfachllm-ops` skill.
