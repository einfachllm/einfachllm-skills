# Setup

`ectl.py` ships with the gateway, not with this skill. It lives at
[`tools/ectl.py`](https://github.com/einfachllm/einfachllm/blob/main/tools/ectl.py)
in the [einfachllm](https://github.com/einfachllm/einfachllm) repository, and
every command in this skill is written for a checkout of that repo:

```bash
uv run python tools/ectl.py <action>
```

Run it from the repository root with `uv` so the project is importable. If the
user has no checkout, say so before proposing commands — do not substitute raw
`curl` against the admin API.

## Where else ectl lives

| Context | Invocation |
| --- | --- |
| Gateway checkout (normal case) | `uv run python tools/ectl.py <action>` |
| Inside the gateway container | `python /app/ectl.py <action>` |
| Helm `tenancySync` hook Job | runs `/app/ectl.py tenancy-sync` itself — nothing to invoke by hand |

The container copy is stdlib-only plus the project, which is already installed
in the image, so every action works there too.

## Actions that need the project importable

Most actions are plain HTTP against the admin API and need nothing but Python.
Two import `einfachllm.config.*` for schema validation:

- `config-validate`
- `tenancy-sync`

Both fail with a clear message (`needs the project installed: run via uv run`)
outside a checkout or the image. `models` and `config-show` read the YAML
themselves and need PyYAML. The remaining actions — `health`,
`config-get/set/edit`, `reload`, `mint`, `update`, `revoke` — are stdlib-only
HTTP. Use the invocations above regardless; the split matters only when
diagnosing an import error.

## Environment variables

| Variable | Purpose | Default |
| --- | --- | --- |
| `EINFACHLLM_BASE_URL` | Gateway base URL | `http://localhost:8000` |
| `EINFACHLLM_MASTER_KEY` | Required for `mint` / `rotate` / `revoke` / config / tenancy actions | — |
| `EINFACHLLM_OPERATOR_TOKEN` | Scoped admin token; the `audit` commands prefer it over the master key | — |

Neither credential is ever printed, and both are read only from the
environment. Set them in your shell session, not on a command line that gets
logged:

```bash
export EINFACHLLM_MASTER_KEY=...      # same value the gateway runs with
export EINFACHLLM_BASE_URL=http://localhost:8000
```

For a recurring job that only reads the audit log, set
`EINFACHLLM_OPERATOR_TOKEN` to a token holding `audit:read` and leave the
master key unset entirely — `audit`, `audit-verify` and `audit-export` use it
when it is present (docs/operator-tokens.md). Everything else is master-only,
either because the server requires it (operator-token management) or because
the permission it needs cannot be held by a token (`keys:write`,
`tenancy:write`).

## Prerequisites

- The gateway running (locally: `docker compose up --build`, or
  `uv run uvicorn einfachllm.main:app`).
- `config-validate` and `models` / `config-show` read the YAML config directory
  (`--dir`, default `config`). These need the config files, not a running
  gateway.
