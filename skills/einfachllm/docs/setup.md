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
| `EINFACHLLM_MASTER_KEY` | Required for `mint` / `revoke` / other admin actions | — |

The master key is read only from the environment and is never printed. Set it in
your shell session, not on the command line that gets logged:

```bash
export EINFACHLLM_MASTER_KEY=...      # same value the gateway runs with
export EINFACHLLM_BASE_URL=http://localhost:8000
```

## Prerequisites

- The gateway running (locally: `docker compose up --build`, or
  `uv run uvicorn einfachllm.main:app`).
- `config-validate` and `models` / `config-show` read the YAML config directory
  (`--dir`, default `config`). These need the config files, not a running
  gateway.
