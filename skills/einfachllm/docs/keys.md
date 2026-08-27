# API keys

Keys are stored hashed in Postgres; the plaintext secret (`ein_…`) is shown
**once** at creation and can never be retrieved again. Requires
`EINFACHLLM_MASTER_KEY`.

## Mint a key

```bash
uv run python tools/ectl.py mint \
  --team platform --user alice --name ci-bot \
  [--budget 100] [--allow-non-eu] [--allowed-models mistral-large-eu,llama3-local]
```

- `--team` / `--user` must already exist in `tenancy.yaml` and the user must be a
  member of the team (the gateway enforces this).
- `--allowed-models` (optional) restricts the key; empty = all models.
- `--budget` (optional) EUR spend cap for the key.
- `--allow-non-eu` (optional) lets the key reach non-EU providers — **only takes
  effect if the org also allows non-EU egress** (org is the ceiling).

The command prints the secret once. Relay it to the user a single time; do not
log or repeat it.

## Update a key's limits

```bash
uv run python tools/ectl.py update --id <key_id> \
  [--budget 100|none] [--rpm 60|none] [--tpm 100000|none]
```

Edits the key's budget / requests-per-minute / tokens-per-minute ceilings in
place — no revoke + re-mint, the secret stays valid. A flag you omit leaves
that limit unchanged; the explicit value `none` clears it (the API
distinguishes absent = unchanged from null = clear). Takes effect immediately
(the in-process key cache is invalidated).

## Rotate a key (the compromised-key path)

```bash
uv run python tools/ectl.py rotate --id <key_id> [--grace-seconds 3600]
```

**This is the answer to "a key leaked".** Not revoke + re-mint: rotation
replaces the secret and keeps everything else, so limits, model and MCP
allowlists, IP allowlist, region flag and AI Act classification don't have to
be copied by hand into a new mint.

It also carries over **the accumulated spend** — a rotation must not hand back
a fresh budget. The expiry is the old key's; an absent or already-elapsed one
falls back to the server lifetime policy (`max_key_lifetime_days`).

`--grace-seconds` keeps the old secret alive for that overlap so the new one
can be rolled out first; `0` (the default) revokes it at once. Capped at 7 days
— longer is not a rotation. The window can only shorten an existing expiry,
never extend it.

The new secret prints **once**, like `mint`. Relay it a single time.

For an actual compromise, prefer `--grace-seconds 0` — an overlap keeps the
leaked secret working for exactly as long as you set.

## Revoke a key

```bash
uv run python tools/ectl.py revoke --id <key_id>
```

Revocation takes effect immediately (the in-process key cache is invalidated).

## Delete a key

```bash
uv run python tools/ectl.py delete --id <key_id>
```

Hard-deletes a **revoked** key — the API returns 409 if it isn't revoked yet,
which also means the delete can never race the key cache. Usage and tool-call
history survive (those tables carry no foreign key), and a `deleted_entities`
tombstone records what the id meant, so retained rows stay attributable to a
name and an owner.

Revoke is what stops a key working; delete is only cleanup. Don't reach for it
during an incident.
