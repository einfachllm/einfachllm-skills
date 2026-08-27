# Operator tokens (scoped admin credentials)

The master key is all-powerful and never expires. An **operator token** is a
scoped, expiring machine credential for the admin API — what automation should
carry instead. It is a different thing from a gateway API key (`ein_…`, used to
call `/v1/*`): an operator token (`ein_op_…`) authenticates against `/admin/*`.

Minting, listing and revoking them **requires the master key** — an operator
token can never mint a peer or widen its own authority.

## Mint

```bash
uv run python tools/ectl.py operator-token-mint \
  --name ci-dashboard \
  --permissions 'analytics:read,costs:read' \
  [--expires-in-days 90]
```

`--expires-in-days` is 1–365, default 90. The token prints **once**; store it
immediately, it cannot be retrieved again.

Grant the narrowest set that does the job. Available permissions:

| Permission | Covers |
| --- | --- |
| `providers:read` | provider catalog + health |
| `keys:read` | list API keys |
| `keys:write` | mint / revoke API keys — **refused on a token** |
| `tenancy:read` | orgs / teams / users |
| `tenancy:write` | author tenancy — **refused on a token** |
| `users:write` | disable/enable users, team membership |
| `policies:read` | rate limits / budgets / named policies |
| `analytics:read` | requests / tokens / latency stats |
| `costs:read` | spend / budget burn-down |
| `audit:read` | audit log |
| `playground:use` | the chat playground |
| `catalog:read` | end-user model catalog |
| `config:write` | edit config YAML via the editor or `ectl` |
| `reload:write` | `POST /admin/reload` + setup checks |

## Why two permissions are refused

The server rejects `keys:write` and `tenancy:write` on an operator token, and
this is not an arbitrary restriction — each one issues a credential a human can
log in with, so a token holding it could trade it for an interactive admin
session and escalate past its own scope:

- `keys:write` → mint an API key bound to an admin user, then
  `POST /admin/login {"api_key": ...}` as that admin.
- `tenancy:write` → create a user with role `admin` (or reset an existing
  admin's password), then log in as them.

Credential administration stays on the break-glass identity. **Automation that
genuinely must rotate keys or author tenancy keeps using the master key** — do
not try to work around this by splitting permissions differently.

## List

```bash
uv run python tools/ectl.py operator-token-list
```

Returns id, name, prefix, permissions, expiry, and revocation state for every
token — never the secret.

## Revoke

```bash
uv run python tools/ectl.py operator-token-revoke --id <token_id>
```

Idempotent: revoking an already-revoked token is a no-op that returns its
current state. Both mint and revoke are recorded in the audit log
(`operator_token.mint` / `operator_token.revoke`).

## When to suggest one

If the user is setting up anything recurring against the admin API — a metrics
scraper, a CI smoke test, a dashboard, a compliance export job — propose an
operator token scoped to exactly what it reads, rather than handing that job
the master key. Say what permissions you would grant and why.
