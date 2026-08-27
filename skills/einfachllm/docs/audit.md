# Audit log

Every security-relevant action lands in `audit_logs`: auth failures (including
SSO `auth.sso_denied`), region / budget / rate-limit denials, key mint, revoke
and rotate, operator-token mint and revoke, `key.revoke.group_loss` when an SSO
re-login drops a team, and `upstream.model_mismatch` when a provider answers
with a different model than was requested.

**No prompt or response content is ever recorded.** Don't propose changes that
would add it.

Reads require `audit:read`. A team-scoped grant sees only that team's rows.

## Query

```bash
uv run python tools/ectl.py audit \
  [--actor alice] [--action key.revoke] \
  [--since 2026-08-01T00:00:00Z] [--until 2026-08-27T00:00:00Z] \
  [--limit 50] [--offset 0]
```

Newest first. `--actor` and `--action` are case-insensitive **substring**
filters, not exact matches — `--action key.` catches every key event.
`--limit` is 1–200 (server default 50); page with `--offset`. The response
carries `total` for the filtered set, so you can tell whether more rows exist.

## Verify the hash chain

```bash
uv run python tools/ectl.py audit-verify
```

Rows are hash-chained. This walks the chain and reports the first break, and
**exits 1 when the chain is broken** — so it can gate a compliance job.

It detects a modified row (recomputed hash differs), a deleted or inserted row
(the `prev_hash` linkage breaks), and a row stripped of its hashes after the
chain started. The response also carries `head`, the current chain head, for
operators who anchor it externally.

What it **cannot** detect: truncation at the tail, and the legitimate retention
purge at the head. Say that plainly rather than reporting a passing verify as
proof nothing was removed.

`ok: true` on an empty or unchained table is not a false pass — nothing claimed
integrity, so nothing failed it. `pre_chain` counts rows predating the chain.

Whole-table by construction, so it needs an org-wide `audit:read`; a
team-scoped viewer cannot see every link.

## Export

```bash
uv run python tools/ectl.py audit-export \
  [--since T] [--until T] [--format jsonl|csv] [--out audit-export.jsonl]
```

Streams the log as a compliance archive — JSONL (default) with an integrity
trailer, or CSV carrying `prev_hash` and `row_hash` per row. Without `--out` it
goes to stdout, so it can be piped.

The export is itself audited (`audit.export`, with the range and format as the
target). Treat the file as sensitive: it holds actor identities and usage
metadata, and belongs in EU-resident, encrypted storage like any other
compliance artifact.
