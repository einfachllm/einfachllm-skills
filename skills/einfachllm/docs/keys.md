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

## Revoke a key

```bash
uv run python tools/ectl.py revoke --id <key_id>
```

Revocation takes effect immediately (the in-process key cache is invalidated).
