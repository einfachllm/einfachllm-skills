# Tenancy sync: orgs/teams/users/memberships (#348)

The database owns tenancy (#293) — `tenancy.yaml` only seeds an **empty** DB
once, at first boot. After that, orgs/teams/users/memberships are authored
through the admin API or the SPA, never the file. Requires
`EINFACHLLM_MASTER_KEY`.

`tenancy-sync` gives that authoring surface a declarative, repeatable path —
useful when a Terraform provider isn't in the picture (there isn't one for
einfachllm tenancy) but you still want orgs/teams/users defined in Git and
applied on every deploy.

## Reconcile from a file

```bash
uv run python tools/ectl.py tenancy-sync --file desired.yaml
```

`desired.yaml` is in the exact shape of `config/tenancy.yaml`
(`organizations` / `teams` / `users`, see docs/config.md and
[`docs/configuration.md`](https://github.com/einfachllm/einfachllm/blob/main/docs/configuration.md)
in the gateway repo for the full field reference).

- **Creates** anything in the file that doesn't exist yet.
- **Updates** only the fields that actually differ — an org/team/user already
  at its desired state produces zero API calls. Safe to run on a schedule.
- **Never deletes.** Removing an entry from the file does nothing to the live
  org/team/user; remove it by hand (SPA or the admin API) once you're sure
  nothing still needs it.
- Refuses (fails loudly, exit 1) to move a team or user to a different org, or
  to change a user's `type` — both are immutable identity fields the API has
  no endpoint for; delete and recreate instead.
- Skips (with a warning, does not fail the run) any user whose live `source`
  is `sso` or `scim` — an IdP-provisioned user rejects local edits with 409
  anyway, since the next sync would overwrite them.

## In Kubernetes (Helm + ArgoCD)

The chart can run this on every release: set `tenancySync.enabled: true` and
`tenancySync.desiredState` (same shape as above) in `values.yaml`. A
`post-install,post-upgrade` hook Job (which ArgoCD runs as `PostSync`, after
the gateway is Healthy) calls this exact command against the live admin API —
see [`docs/deploy-kubernetes.md`](https://github.com/einfachllm/einfachllm/blob/main/docs/deploy-kubernetes.md#tenancy-sync-orgsteamsusers-without-terraform-348)
in the gateway repo.

## Propose changes as a diff

Like config edits: read the current state (`ectl.py tenancy-sync` needs a full
desired file, not a patch), show the user what will change, and let them
confirm before running it against a production gateway. Never widen
`allow_non_eu` or a budget without the user explicitly asking.
