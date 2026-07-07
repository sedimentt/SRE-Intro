# Lab 10 — SRE Portfolio & Reliability Review

## Task 1 — Load Testing & Reliability Review (6 pts)

### 1. Load-test table across 10/50/100 (and the breaking-point level)

| Users | Ramp | RPS | p50 | p95 | p99 | 5xx error rate | 409 (inventory) |
|------:|-----:|----:|----:|----:|----:|---------------:|----------------:|
| 10    | 2/s  | 7.61 | 7 ms | 10 ms | 12 ms | 0.00% | 0 |
| 50    | 5/s  | 37.03 | 6 ms | 15 ms | 54 ms | 0.00% | 29 |
| 100   | 10/s | 55.11 | 330 ms | 770 ms | 1100 ms | 48.80% | 0 |

**Breaking point:**

| Users | RPS | p99 | 5xx error rate | Reason |
|------:|----:|----:|---------------:|--------|
| 100   | 55.11 | 1100 ms | 48.80% | p99 exceeded 500 ms and 5xx error rate exceeded 0.5% |

At 100 concurrent users, the system reached its capacity ceiling: p99 latency spiked to 1100 ms (threshold: 500 ms) and the 5xx error rate reached 48.80% (threshold: 0.5%). The event 5 inventory (80 tickets) was saturated early in the 50-user test (29 409s), but at 100 users the system itself started failing with 5xx responses.

---

### 2. DORA Metrics

| Metric | Value | Source Data |
|--------|------:|-------------|
| **Deployment Frequency** | ~3.7 deploys/day (67 commits / 18 days) | `git log --oneline main \| wc -l` — 67 commits total |
| **ReplicaSets (rollouts)** | 9 | `kubectl get rs -l app=gateway` — 9 distinct ReplicaSets (one per rollout) |
| **Lead Time (commit → prod)** | ~5–6 min | CI build time (~2–3 min) + ArgoCD poll interval (~3 min) |
| **Change Failure Rate** | 66.67% | 2 Failed AnalysisRuns out of 3 total (60% error rate during canaries) |
| **Recovery Time** | ~seconds (abort) / ~3 min (git revert) | `kubectl argo rollouts abort` promotes stable RS instantly; git revert waits for ArgoCD sync |

**AnalysisRun breakdown:**

```text
2 Failed
1 Successful
```

---

### 3. Top 3 Reliability Risks

1. **Single Postgres instance without replication**
   - The database is a single point of failure. If the Postgres pod dies and the PVC is corrupted, all data is lost until restored from a backup.
   - Fix: Add a read replica or use a managed database. At minimum, verify backup restoration weekly.

2. **Gateway canary rollouts often fail (66.67% failure rate)**
   - Two out of three canary deployments were aborted because the new version had a 45–48% error rate, meaning the analysis template caught bad deploys — but the high failure rate suggests poor CI gating before the rollout starts.
   - Fix: Add integration tests to CI that run against a staging environment before the image tag is promoted to production.

3. **No latency alerting — only error-rate alerts**
   - The current alerting (Lab 6) only watches for 5xx error rate. At 100 users, p99 latency blew past 500 ms before any alert fired because the error rate was still 0%.
   - Fix: Add a Prometheus alert for p99 latency > 500 ms over 5 minutes (e.g., `histogram_quantile(0.99, rate(gateway_request_duration_seconds_bucket[5m])) > 0.5`).

---

### 4. Toil Identification

| Task | Frequency | How to Automate | Time Saved |
|------|-----------|-----------------|-----------:|
| **Re-seeding Postgres after pod restart** (before PVC) | ~10 times across Labs 4–9 | Add init container or postStart hook that runs migrations and seeds; use PVC (already done in Lab 9 Bonus) | ~2 min per restart |
| **Re-creating port-forwards after pod restarts** | ~15+ times across all labs | Use in-cluster curl via `kubectl run` or a persistent loadgen Deployment (already done in Lab 4) | ~30 s per restart |
| **Manually watching canary rollout** (`kubectl argo rollouts get rollout --watch`) | ~5 times in Lab 7 | Rely on the AnalysisTemplate + automatic promote/abort; check rollout status only on failure notification | ~3 min per deploy |

---

### 5. Monitoring Gaps

- **Missing latency metrics.** The gateway exposes request duration but no Prometheus alert queries p99 latency. During chaos experiments (Lab 8), a slow-but-not-yet-failing events service caused degraded用户体验 without triggering any alert.
- **No per-pod request rate dashboard.** During the pod-kill experiment, it was hard to tell whether traffic had been redistributed evenly across the remaining pods. A dashboard showing `sum by (pod) (rate(gateway_requests_total[1m]))` would make this visible immediately.
- **No downstream dependency monitoring.** If the events or payments service becomes slow, the gateway 5xx rate stays low but latency increases. An alert that tracks `gateway_request_duration_seconds` per downstream endpoint (e.g., `/events` vs `/reserve`) would pinpoint which service is causing the slowdown.
- **No database connection pool alert.** When Postgres was under load, the gateway could run out of connections without triggering a 5xx spike until connections were fully exhausted. Monitoring `pg_stat_activity` count would give early warning.

---

### 6. Capacity Plan (2× Traffic)

**Current ceiling:** 55 RPS (at 100 users, p99 = 1100 ms, 48.80% 5xx)

**Per-pod CPU at idle (no load):**

```text
NAME                       CPU(cores)   MEMORY(bytes)
gateway-6c7558887b-5thdv   15m          42Mi
gateway-6c7558887b-99s4k   12m          41Mi
gateway-6c7558887b-fzbpv   16m          38Mi
gateway-6c7558887b-gd2c7   13m          47Mi
gateway-6c7558887b-swqpc   16m          54Mi

events-79c854f79c-7xnm5    21m          42Mi
payments-5d559b95f6-br5k4  12m          44Mi
postgres-7459775f5-tt5l4   8m           47Mi
redis-65bb44458c-j2pq5     7m           13Mi
```

**For 2× traffic (110 RPS target):**

- **gateway**: Scale from 5 → 10 replicas. Current resource requests (50m CPU / 64Mi memory) per pod. At 110 RPS, each pod would handle ~11 RPS (down from ~11 at 55 RPS total — same per-pod load if we double replicas).
- **events**: Keep at 1 replica (21m CPU at idle; add a second replica for HA).
- **payments**: Keep at 1 replica (12m CPU at idle; add a second replica for HA).
- **postgres**: Single pod is borderline at 2× traffic. Add connection pooling (e.g., PgBouncer sidecar) and increase limits to 500m CPU / 512Mi memory.
- **redis**: Single pod is likely OK (7m CPU at idle), but switch to a replicated setup with `--cluster-enabled yes` for HA.

**Resource requests/limits update:**

| Service | Replicas | Requests | Limits | x1 cost | x2 cost |
|---------|:--------:|----------|--------|--------:|--------:|
| gateway | 10 | 50m / 64Mi | 200m / 256Mi | $25/mo | $50/mo |
| events  | 2  | 50m / 64Mi | 200m / 256Mi | $5/mo  | $10/mo |
| payments | 2 | 50m / 64Mi | 200m / 256Mi | $5/mo  | $10/mo |
| postgres | 1 | 100m / 256Mi | 500m / 512Mi | $5/mo  | $5/mo  |
| redis   | 1 | 50m / 64Mi  | 200m / 128Mi | $5/mo  | $5/mo  |
| **Total** |   |            |            | **$45/mo** | **$80/mo** |

**Cost estimate** (at $5/pod/month small-cloud rate): ~$80/month for 2× capacity.

---
## Task 2 — Capacity Plan with Numbers (4 pts)

### 10.7: Per-pod CPU at breaking point (idle)

Since the breaking point was reached at 100 users / 55 RPS, per-pod CPU was sampled at rest. Under full load, gateway CPU would be significantly higher — the bottleneck is likely the gateway itself, as events/payments/Postgres all show <25m CPU at idle.

### 10.8: 2× Traffic Capacity Plan

(See Capacity Plan in section 6 above.)

**Key bottlenecks identified:**
- **Gateway** is the CPU-constrained service. At 55 RPS with 5 replicas, each pod handles ~11 RPS. To handle 110 RPS, scale to 10 replicas.
- **Postgres** needs connection pooling (PgBouncer) to handle 2× query load without exhausting connections.
- **Redis** is low-utilization and can stay single-pod for now, but HA setup recommended.

## Bonus Task — SRE Handbook (Option B)

The complete 2-page SRE handbook is at [`submissions/runbooks/quickticket-handbook.md`](./runbooks/quickticket-handbook.md).

It covers:
- **Architecture** — ASCII diagram with all services and data flow.
- **How to deploy** — GitOps flow (GitHub → CI → ArgoCD → canary rollout).
- **Monitoring** — Prometheus metrics table, key PromQL queries, dashboards, alert rules.
- **Incident response** — Condensed runbook with diagnosis steps and common fixes.
- **Backup/restore** — RPO/RTO table, automated CronJob, manual restore procedure.

---
