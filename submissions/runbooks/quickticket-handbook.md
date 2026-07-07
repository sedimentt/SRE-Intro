# QuickTicket SRE Handbook

## Architecture

```text
                     ┌──────────────────────────────────────┐
                     │          Ingress / Traefik           │
                     │        (k3d LB, port 3080)           │
                     └──────────────┬───────────────────────┘
                                    │
                     ┌──────────────▼───────────────────────┐
                     │         gateway (Rollout)            │
                     │   5 replicas · ClusterIP:8080        │
                     │   /events → events:8081              │
                     │   /reserve → events:8081 + payments  │
                     └──┬───────────────┬───────────────────┘
                        │               │
              ┌─────────▼──┐    ┌───────▼────────┐
              │  events    │    │   payments     │
              │  1 replica │    │   1 replica    │
              │  ClusterIP │    │   ClusterIP    │
              │  :8081     │    │   :8082        │
              └──┬────┬────┘    └────────────────┘
                 │    │
        ┌────────▼┐ ┌─▼────────┐
        │ postgres│ │  redis   │
        │ 1 pod   │ │  1 pod   │
        │ PVC:1Gi │ │  :6379   │
        │ :5432   │ └──────────┘
        └─────────┘
```

- **gateway** — Go HTTP server; reverse-proxies to events/payments; exposes Prometheus metrics (`/metrics`).
- **events** — Python service; manages event inventory, reservations (via Redis holds), and DB queries.
- **payments** — Python service; processes payments with configurable failure/latency injection.
- **postgres** — Primary database; stateful with PVC for data persistence.
- **redis** — In-memory reservation holds (TTL: 300s); ephemeral.

---

## How to Deploy

The deployment uses **GitOps with ArgoCD**:

1. **Push to GitHub** — Merge a PR into `main` (or push directly).
2. **CI builds images** — GitHub Actions builds and pushes container images to `ghcr.io/sedimentt/quickticket-*`.
3. **Auto-update tag** — CI commits the new image tag back to the repo (e.g., `ci: update image tags to <sha>`).
4. **ArgoCD syncs** — ArgoCD polls the repo every 3 minutes, detects the manifest change, and syncs.
5. **Canary rollout** — For `gateway` only (Rollout resource): traffic shifts 20% → 40% → 60% → 80% → 100% with 30–60s pauses. Prometheus AnalysisRun checks error rate at each step; if >5%, the rollout aborts automatically.

**Quick start for a new team member:**

```bash
# 1. Clone the repo
git clone https://github.com/sedimentt/quickticket
cd quickticket

# 2. Check cluster state
kubectl get pods,svc,rollouts

# 3. Check ArgoCD sync status
kubectl get applications -n argocd

# 4. Make a change, commit, push — ArgoCD handles the rest
```

---

## Monitoring

### Prometheus Metrics (exposed at `gateway:8080/metrics`)

| Metric | Type | Labels | What it tells you |
|--------|------|--------|-------------------|
| `gateway_requests_total` | Counter | `path`, `method`, `status`, `pod`, `rs_hash` | Request count by path/status/pod |
| `gateway_request_duration_seconds_bucket` | Histogram | `path`, `le` | Latency distribution per endpoint |
| `gateway_request_duration_seconds_sum` | Sum | `path` | Total latency — divide by count for avg |
| `gateway_request_duration_seconds_count` | Counter | `path` | Request count per endpoint |

### Key PromQL Queries

```promql
# Error rate (5xx / total)
sum(rate(gateway_requests_total{status=~"5.."}[5m]))
/ sum(rate(gateway_requests_total[5m]))

# RPS per pod
sum by (pod) (rate(gateway_requests_total[1m]))

# p99 latency per endpoint
histogram_quantile(0.99, sum by (le, path) (
  rate(gateway_request_duration_seconds_bucket[5m])
))

# 409 rate (inventory exhaustion)
sum(rate(gateway_requests_total{status="409"}[5m]))
```

### Dashboards

- **Grafana** (port-forward: `kubectl -n monitoring port-forward svc/grafana 3000:3000`)
- **Prometheus UI** (`kubectl -n monitoring port-forward deployment/prometheus 9090:9090`)
- **QuickTicket — Golden Signals** dashboard (pre-configured in Grafana): error rate, latency, traffic, saturation.

### Alert Rules (from Lab 6)

| Alert | Condition | Severity |
|-------|-----------|----------|
| HighErrorRate | 5xx rate > 5% for 2 min | Critical |
| SLOBurnRate | 30m error budget burn > 6× | Warning |

---

## Incident Response

### Runbook: High Error Rate (5xx > 5%)

**1. Confirm** — Check gateway health and error rate:
```bash
kubectl get pods -l app=gateway
kubectl logs -l app=gateway --tail=50 --since=5m
```

**2. Check dependencies** — Test each service directly:
```bash
kubectl run smoke --image=curlimages/curl --rm -i --restart=Never -- \
  curl -s http://events:8081/health
kubectl run smoke --image=curlimages/curl --rm -i --restart=Never -- \
  curl -s http://payments:8082/health
```

**3. Check Postgres and Redis:**
```bash
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  pg_isready -U quickticket -d quickticket
kubectl exec -it $(kubectl get pod -l app=redis -o name) -- \
  redis-cli PING
```

**4. Mitigate** — Common fixes:

| Symptom | Cause | Fix |
|---------|-------|-----|
| Payments returns 5xx | `PAYMENT_FAILURE_RATE` > 0 | Set `PAYMENT_FAILURE_RATE=0.0` and redeploy |
| Events returns 5xx | DB/Redis connectivity | Restart events pod |
| Gateway returns 503 | Downstream timeout | Increase `GATEWAY_TIMEOUT_MS` or scale downstream |

**5. Escalation** — If unresolved after 10 minutes, contact the SRE team lead.

---

## Backup/Restore

### Automated Backup (CronJob)

Runs every 5 minutes via `k8s/backup-cronjob.yaml`:

```bash
# List recent backups
kubectl exec deployment/backup-inspector -- ls -la /backups

# Check backup job logs
kubectl logs job/postgres-backup-<timestamp>
```

### Manual Restore

```bash
# 1. Find the backup pod
BACKUP_POD=$(kubectl get pod -l app=backup-inspector -o name | head -1)

# 2. Copy backup to postgres pod
kubectl cp $BACKUP_POD:/backups/quickticket_latest.dump /tmp/restore.dump

# 3. Copy into postgres pod
POD=$(kubectl get pod -l app=postgres -o name | head -1)
kubectl cp /tmp/restore.dump $POD:/tmp/restore.dump

# 4. Drop and restore
kubectl exec $POD -- psql -U quickticket -d quickticket -c \
  'DROP TABLE IF EXISTS orders, events, alembic_version CASCADE'
kubectl exec $POD -- pg_restore -U quickticket -d quickticket \
  --clean --if-exists /tmp/restore.dump
```

### RPO / RTO

| Metric | Value | How to improve |
|--------|-------|----------------|
| RPO | ~5 min (CronJob interval) | Reduce schedule to `*/1 * * * *` |
| RTO (PVC) | ~13 s (pod restart only) | N/A — PVC already persists data |
| RTO (full restore) | ~60 s | Automate with an init container |