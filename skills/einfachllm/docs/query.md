# Query (health, models, usage)

## Health

```bash
uv run python tools/ectl.py health
# -> {"status": "ok", "version": "..."}
```

## Configured models and aliases

```bash
uv run python tools/ectl.py models [--dir config]
```

Lists what the gateway can route to (from `models.yaml`). To see what a specific
key may call, request `GET /v1/models` with that key's bearer token.

## Usage / spend / budgets

Two sources, both read-only:

- **Dashboard** — `/admin/app` shows totals (requests, tokens, spend) and
  spend-vs-budget per org and team; `/admin/app/usage` and
  `/admin/app/requests` break it down, `/admin/app/providers` has provider
  reachability ("Check health"). Point a human here.
- **JSON API** — `GET /admin/api/stats/*`, gated on the `analytics:read`
  permission (the master key has it; a scoped operator token needs it
  granted). Use this when you need machine-readable numbers.

`ectl.py` has no wrapper for the stats endpoints yet, so these are the one
exception to "always use ectl": a plain authenticated `GET` is fine because
nothing is written and no secret is returned. Send the credential as an
`Authorization` header read from the environment — never interpolate it into a
command line that gets logged or echoed.

| Endpoint | Returns |
| --- | --- |
| `/admin/api/stats/summary` | Totals for the window (`previous=true` for the preceding period) |
| `/admin/api/stats/timeseries` | Bucketed series for charts |
| `/admin/api/stats/performance` | Latency / throughput per target |
| `/admin/api/stats/top/{dimension}` | Leaderboard; `limit` 1–100, default 10 |

`{dimension}` is one of `models`, `tools`, `providers`, `keys`, `users`,
`agents`, `sessions` — anything else is a 404.

All four take `window` = `1h` / `24h` / `7d` / `30d` / `all` (absent = all-time;
an unknown value is a 400), or an explicit `since` / `until` range which
overrides `window`. `timeseries` additionally takes `bucket` = `minute` /
`hour` / `day`, rejected with a 400 when it is too fine for the span.

Results are scoped to what the caller may see: a principal limited to certain
teams gets only those teams' rows, so numbers can legitimately differ between
callers. Never scrape the dashboard HTML — use these endpoints or the UI.
