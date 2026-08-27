---
name: einfachllm
description: >-
  Manage a running einfachllm gateway — an EU-sovereign, OpenAI-compatible LLM
  gateway. Use this skill to mint, update, rotate and revoke API keys, respond
  to a leaked key, mint scoped operator tokens instead of sharing the master
  key, inspect usage, spend and budgets, view and edit the YAML routing config
  (providers/models/settings) and reload it, reconcile orgs/teams/users
  declaratively, query, verify and export the audit log, and check health.
  Routing config lives in YAML (GitOps); API keys live hashed in Postgres;
  tenancy lives in Postgres, reconcilable from a file via tenancy-sync. For
  deploying, migrating or upgrading the gateway, use the einfachllm-ops skill.
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

1. **Never print or log secrets.** `mint`, `rotate` and `operator-token-mint`
   each show one exactly once by design (`ein_…` / `ein_op_…`); relay it to the
   user a single time and never repeat it.
2. **Never reveal the master key.** It comes only from `EINFACHLLM_MASTER_KEY`
   in the environment; never echo it, put it in a file, or pass it on a command
   line that gets logged.
3. **Never print resolved provider credentials.** Use `config-show` (it reads raw
   YAML and keeps `${ENV}` placeholders / redacts secrets) — never `cat` a config
   that has interpolated secrets.
4. **Use `ectl.py`, not raw HTTP**, for every action it covers — without
   exception for anything that writes. Only two read-only areas lack a wrapper:
   the stats and usage endpoints (docs/query.md). The tenancy operations in
   docs/tenancy.md ("What sync cannot do") also lack one but *do* write —
   confirm with the user before every one of them.
5. **Prefer a scoped operator token over the master key** for anything
   recurring (docs/operator-tokens.md). Never propose working around the
   server's refusal to put `keys:write` / `tenancy:write` on one.
6. **Security decisions come from the key, never client input.** When editing
   config, never weaken `allow_non_eu` / region tags or budgets without the user
   explicitly asking.
7. **Edit config as a reviewable change**, then run `config-validate` before the
   user applies/reloads it. Routing config is YAML — propose a diff, don't mutate
   silently.

## How to execute

1. Read `docs/setup.md` for environment variables and prerequisites.
2. Match the user's request to the Actions table below.
3. Read the referenced doc, then run the `ectl.py` action.
4. If the request doesn't match any action, say so and show this table.

## Actions

### Health & config

| Intent | Doc | Command |
| --- | --- | --- |
| Check the gateway is up | docs/query.md | `ectl.py health` |
| List configured models/aliases | docs/query.md | `ectl.py models` |
| View the (sanitized) config | docs/config.md | `ectl.py config-show` |
| Validate config before reload | docs/config.md | `ectl.py config-validate` |
| Add/edit a provider or model | docs/config.md | edit YAML, then `config-validate` |
| Read the gateway's live config | docs/config.md | `ectl.py config-get [NAME]` |
| Write one config file remotely | docs/config.md | `ectl.py config-set NAME (--file P \| --stdin)` |
| Edit a config file in `$EDITOR` | docs/config.md | `ectl.py config-edit NAME` |
| **Apply saved config to the running gateway** | docs/config.md | `ectl.py reload` |

### API keys

| Intent | Doc | Command |
| --- | --- | --- |
| Mint an API key | docs/keys.md | `ectl.py mint --team T --user U --name N` |
| Update a key's limits | docs/keys.md | `ectl.py update --id ID [--budget 100\|none] [--rpm 60\|none] [--tpm 100000\|none]` |
| **A key leaked — rotate it** | docs/keys.md | `ectl.py rotate --id ID [--grace-seconds 0]` |
| Revoke an API key | docs/keys.md | `ectl.py revoke --id ID` |
| Delete a revoked key | docs/keys.md | `ectl.py delete --id ID` |

### Tenancy & admin credentials

| Intent | Doc | Command |
| --- | --- | --- |
| Reconcile orgs/teams/users from a file | docs/tenancy.md | `ectl.py tenancy-sync --file desired.yaml` |
| ... from a config dir with `teams/` | docs/tenancy.md | `ectl.py tenancy-sync --dir config` |
| Delete an org/team/user, edit memberships | docs/tenancy.md | admin API or the SPA — `tenancy-sync` never deletes |
| Mint a scoped admin token | docs/operator-tokens.md | `ectl.py operator-token-mint --name N --permissions 'a,b'` |
| List scoped admin tokens | docs/operator-tokens.md | `ectl.py operator-token-list` |
| Revoke a scoped admin token | docs/operator-tokens.md | `ectl.py operator-token-revoke --id ID` |

### Usage & audit

| Intent | Doc | Command |
| --- | --- | --- |
| View usage / spend / budgets | docs/query.md | `GET /admin/api/usage/*` and `/admin/api/stats/*`, or `/admin/app` |
| Query the audit log | docs/audit.md | `ectl.py audit [--actor A] [--action A] [--since T]` |
| Verify audit integrity | docs/audit.md | `ectl.py audit-verify` (exit 1 on a break) |
| Export the audit log | docs/audit.md | `ectl.py audit-export [--format jsonl\|csv] [--out FILE]` |
| Deploy, migrate, upgrade, roll back | — | see the `einfachllm-ops` skill |

## Notes / limits

- Adding providers/models means editing YAML (`providers.yaml`, `models.yaml`)
  and reloading — there is no DB-backed provider store by design.
- Usage/spend has a JSON API but no `ectl` wrapper (docs/query.md); the
  dashboard at `/admin/app` is the human-facing source.
- `tenancy-sync` creates and updates, **never deletes**, and refuses to move a
  team or user between orgs or change a user's `type`. Those go through the
  admin API or the SPA.
- Disabling or enabling a user (`POST /admin/users/{id}/disable|enable`) has no
  `ectl` action yet — offboarding runs through the SPA or the admin API.
- This skill manages a **running** gateway. Deployment, schema migrations,
  backup/restore, and upgrades belong to the `einfachllm-ops` skill.
