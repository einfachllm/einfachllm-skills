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
- **JSON API** — two families, split by permission. `/admin/api/stats/*`
  (`analytics:read`) is traffic and performance; `/admin/api/usage/*`
  (`costs:read`) is money and request logs. The master key holds both; a scoped
  operator token needs them granted (docs/operator-tokens.md).

`ectl.py` has no wrapper for the stats endpoints yet, so these are the one
exception to "always use ectl": a plain authenticated `GET` is fine because
nothing is written and no secret is returned. Send the credential as an
`Authorization` header read from the environment — never interpolate it into a
command line that gets logged or echoed.

### Traffic and performance — `analytics:read`

| Endpoint | Returns |
| --- | --- |
| `/admin/api/stats/summary` | Totals for the window (`previous=true` for the preceding period) |
| `/admin/api/stats/timeseries` | Bucketed series for charts |
| `/admin/api/stats/performance` | Latency / throughput per target |
| `/admin/api/stats/top/{dimension}` | Leaderboard; `limit` 1–100, default 10 |

`{dimension}` is one of `models`, `tools`, `providers`, `keys`, `users`,
`agents`, `sessions` — anything else is a 404.

### Spend and request logs — `costs:read`

| Endpoint | Returns |
| --- | --- |
| `/admin/api/usage/summary` | Windowed totals + applicable budgets + this-month linear forecast |
| `/admin/api/usage/timeseries` | Bucketed spend series |
| `/admin/api/usage/models` | Spend per model |
| `/admin/api/usage/sessions` | Spend per session |
| `/admin/api/usage/export` | Raw usage rows as JSONL/CSV for chargeback or FinOps |
| `/admin/api/usage/logs` | Per-request log; filter by `request_id`, `model`, `provider_id`, `status` |
| `/admin/api/usage/logs/{request_id}` | One request in detail |

Use `usage/summary` — not `stats/summary` — when the question is about money:
it is the one that returns the budgets and the forecast.

### Shared parameters

Both families take `window` = `1h` / `24h` / `7d` / `30d` / `all` (absent =
all-time; an unknown value is a 400), or an explicit `since` / `until` range
which overrides `window`. The `timeseries` endpoints additionally take
`bucket` = `minute` / `hour` / `day`, rejected with a 400 when it is too fine
for the span.

Results are scoped to what the caller may see: a principal limited to certain
teams gets only those teams' rows, so numbers can legitimately differ between
callers. Never scrape the dashboard HTML — use these endpoints or the UI.
