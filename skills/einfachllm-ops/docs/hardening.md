# Production hardening & what to watch

Every knob here is opt-in; the chart defaults stay a correct single-replica
install.

## Autoscaling (HPA)

```bash
--set autoscaling.enabled=true \
--set autoscaling.minReplicas=2 --set autoscaling.maxReplicas=5
```

CPU target (default 75% of requests) needs metrics-server. Add a memory target
via `autoscaling.targetMemoryUtilizationPercentage`, or RPS and other custom
metrics via `autoscaling.extraMetrics` (verbatim `autoscaling/v2` entries,
needs a metrics adapter).

When enabled the HPA owns the replica count and `replicaCount` is ignored.

**Rate-limit windows are per-pod**: token buckets burst one minute's allowance
then refill at `limit/60` per second, per replica. The effective RPM/TPM
ceiling therefore scales with the pod count — size limits against
`maxReplicas`, not against one pod. With `rate_limit_redis_url` set, circuit
breaker opens and 429 parks are shared across pods, so a dead provider is
discovered once for the deployment rather than once per replica.

More than one replica also means switching migrations to a Job
(docs/migrations.md).

## PodDisruptionBudget

```bash
--set podDisruptionBudget.enabled=true    # minAvailable: 1 by default
```

Keeps the gateway up through node drains and cluster upgrades. **Only enable
with more than one replica** — at `replicaCount: 1` with `minAvailable: 1`,
drains block forever.

## NetworkPolicy (enforceable sovereignty)

```bash
--set networkPolicy.enabled=true \
--set networkPolicy.egress.providerCidrs[0]=<EU provider CIDR> \
--set networkPolicy.egress.databaseCidr=<DB CIDR>      # external DB only
```

Default-denies ingress and egress for the gateway pods, then allowlists: DNS,
the in-cluster Postgres (automatic when `postgresql.enabled`) or
`egress.databaseCidr` on `egress.databasePort` (5432), and `egress.providerCidrs`
on `egress.providerPort` (443).

This is the control that makes sovereignty *enforceable* rather than
configured: the pod **cannot** reach a non-EU endpoint that isn't listed,
regardless of a configuration bug. Never widen it to make a deploy work.

- Restrict who may call the gateway with `networkPolicy.ingressFrom` (e.g. only
  the ingress controller's namespace). Empty = ingress to the port from
  anywhere.
- Add in-cluster targets (Ollama, vLLM) via `networkPolicy.egress.extraEgress`,
  appended verbatim.
- With `tenancySync.enabled`, the sync Job's pod
  (`app.kubernetes.io/component: tenancy-sync`) needs an `ingressFrom` entry
  too, or it times out reaching the Service.
- Requires a CNI that enforces NetworkPolicy (Calico, Cilium, …). **kind's
  default kindnet does not** — the policy will appear applied and enforce
  nothing.

## Scheduling

`topologySpreadConstraints` (spread replicas across nodes/zones) and
`priorityClassName` pass through verbatim, alongside `nodeSelector`,
`tolerations`, and `affinity`.

## Metrics & alerts

```bash
--set serviceMonitor.enabled=true      # scrapes GET /metrics, interval 15s
--set prometheusRule.enabled=true      # ships the operational alert rules
```

Enable the Prometheus exporter in `settings.yaml`
(`observability.exporters.prometheus.enabled`) first, or the ServiceMonitor
scrapes nothing. For a token-guarded exporter, point
`serviceMonitor.bearerTokenSecret` at the secret holding the token.

`prometheusRule` ships error-rate, latency, sink-health, and auth/429-spike
rules — the same set as `deploy/prometheus/rules/einfachllm-alerts.yml` in the
Compose overlay. A Grafana dashboard is at
`deploy/grafana/einfachllm-dashboard.json`.

## What to watch

| Signal | Where |
| --- | --- |
| Liveness | `GET /health` |
| Upstream reachability | `GET /admin/api/providers/health` — ok / auth_error / error / unreachable + latency |
| Requests, tokens, spend vs budget | `/admin/app` dashboard, or `GET /admin/api/stats/summary` |
| Top keys / models / tools | `GET /admin/api/stats/top/{dimension}` |
| Security events | the `audit_logs` table |

Log signals worth alerting on:

- Usage and audit writers log warnings on flush retries.
- A saturated usage queue logs a dropped-record warning — that is back-pressure,
  not noise.

`audit_logs` records auth failures (including SSO `auth.sso_denied`),
region/budget/rate-limit denials, key mint and revoke,
`key.revoke.group_loss` when an SSO re-login drops a team, and
`upstream.model_mismatch` when a provider answers with a different model than
was requested. **No prompt or response content is ever logged** — don't add it.
