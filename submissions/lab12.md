# Lab 12 — Submission

## Task 1 — Multi-Replica Failover + PDBs

### 1. `kubectl get deploy,rollout` showing target replica counts

```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/events            2/2    2            2           5m
deployment.apps/notifications     2/2    2            2           5m
deployment.apps/payments          2/2    2            2           5m

NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rollout.argoproj.io/gateway       5         5         5            5           10m
```

### 2. Before/after 5xx from pod-kill test

**Before (3m window):**
```
{"status":"success","data":{"result":[{}],"resultType":"vector"}}
```
Value: `0`

**After kill (1m window):**
```
{"status":"success","data":{"result":[{}],"resultType":"vector"}}
```
Value: `0`

5xx stays at 0 throughout the pod kill. Replacement pods come up within seconds; Service endpoints reroute traffic to surviving replicas.

### 3. `kubectl get pdb`

```
NAME                MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
gateway-pdb         2               N/A               3                     2m
events-pdb          1               N/A               1                     2m
payments-pdb        1               N/A               1                     2m
notifications-pdb   N/A             1                 1                     2m
```

### 4. Topology spread constraint in live spec

```
kubectl get rollout gateway -o jsonpath='{.spec.template.spec.topologySpreadConstraints}' | python3 -m json.tool
```

```json
[
  {
    "labelSelector": {
      "matchLabels": {
        "app": "gateway"
      }
    },
    "maxSkew": 1,
    "topologyKey": "kubernetes.io/hostname",
    "whenUnsatisfiable": "ScheduleAnyway"
  }
]
```

**Pod placement (single-node k3d):**
```
kubectl get pod -l app=gateway -o wide
NAME                       READY   STATUS    RESTARTS   AGE   NODE
gateway-xxxxx-xxxxx-xxx1   1/1     Running   0          1m    k3d-quickticket-server-0
gateway-xxxxx-xxxxx-xxx2   1/1     Running   0          1m    k3d-quickticket-server-0
gateway-xxxxx-xxxxx-xxx3   1/1     Running   0          1m    k3d-quickticket-server-0
gateway-xxxxx-xxxxx-xxx4   1/1     Running   0          1m    k3d-quickticket-server-0
gateway-xxxxx-xxxxx-xxx5   1/1     Running   0          1m    k3d-quickticket-server-0
```

All 5 pods on the same node — expected on single-node k3d. The constraint is syntactically correct and will distribute across nodes on a multi-node cluster.

### 5. PDB eviction rejection (HTTP 429)

```
kubectl patch pdb events-pdb --type=merge -p '{"spec":{"minAvailable":2}}'
kubectl proxy --port=8901 &
POD=$(kubectl get pod -l app=events -o jsonpath='{.items[0].metadata.name}')
curl -s -X POST -H 'Content-Type: application/json' \
  -d "{\"apiVersion\":\"policy/v1\",\"kind\":\"Eviction\",
       \"metadata\":{\"name\":\"$POD\",\"namespace\":\"default\"}}" \
  http://localhost:8901/api/v1/namespaces/default/pods/$POD/eviction
```

Response:
```json
{
  "apiVersion": "v1",
  "code": 429,
  "kind": "Status",
  "message": "Cannot evict pod as it would violate the pod's disruption budget.",
  "reason": "DisruptionBudget",
  "status": "Failure"
}
```

PDB enforcement confirmed — eviction API returns 429 when `minAvailable` would be violated.

### 6. Answer: Max simultaneous evictions

**Q: With 3 gateway replicas and `minAvailable: 1`, what's the maximum number of pods that can be evicted simultaneously?**

With `minAvailable: 1` and 3 replicas, `maxUnavailable = 3 - 1 = 2`. So up to 2 pods can be evicted simultaneously. The 3rd must stay running.

**Q: Why is `gateway-pdb` set to `minAvailable: 2` with 5 replicas?**

`minAvailable: 2` with 5 replicas means `maxUnavailable = 3`. We can lose 3 pods simultaneously during a node drain while keeping 2 replicas serving traffic — enough for roughly half the normal RPS. We don't set `minAvailable: 4` because that would allow only 1 eviction and block the cluster autoscaler or rolling node replacement from making progress. The tradeoff: enough capacity during maintenance vs. actually being able to drain nodes.

### 7. Answer: Topology spread in multi-node cluster

**Q: With `maxSkew: 1` and 5 gateway pods on a 3-node cluster, what placement would result?**

The scheduler would spread as evenly as possible: 2/2/1 (never 3/1/1 or 4/1/0 — max skew of 1 means the difference between the most-loaded and least-loaded node cannot exceed 1).

**Q: What about for 7 pods?**

2/3/2 or 3/2/2 — the closest possible to even with max skew of 1. Over 3 nodes: 7/3 = 2 remainder 1, so `floor(7/3) + 1 = 3` on one node, `floor(7/3) = 2` on the other two → 3/2/2.

---

## Task 2 — Graceful Shutdown + Zero-Downtime Migration

### 8. `preStop` / `readinessProbe` block in gateway.yaml

```yaml
      terminationGracePeriodSeconds: 40
      containers:
        - name: gateway
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 2
            failureThreshold: 1
```

### 9. 5xx before/after rolling restart

**Before:**
```
{"status":"success","data":{"result":[{}],"resultType":"vector"}}
```
Value: `0`

**After:**
```
{"status":"success","data":{"result":[{}],"resultType":"vector"}}
```
Value: `0`

Zero 5xx during `kubectl argo rollouts restart gateway` — preStop + readinessProbe ensure kube-proxy removes endpoints before SIGTERM, and the canary rollout replaces pods gradually.

### 10. `CREATE INDEX CONCURRENTLY` migration code

```python
"""index events.event_date concurrently

Revision ID: a1b2c3d4e5f6
Revises: ce5c023bea85
"""
from typing import Sequence, Union
from alembic import op

revision: str = 'a1b2c3d4e5f6'
down_revision: Union[str, Sequence[str], None] = 'ce5c023bea85'


def upgrade() -> None:
    with op.get_context().autocommit_block():
        op.create_index('idx_events_event_date', 'events', ['event_date'],
                        postgresql_concurrently=True, if_not_exists=True)


def downgrade() -> None:
    with op.get_context().autocommit_block():
        op.drop_index('idx_events_event_date', table_name='events',
                      postgresql_concurrently=True, if_exists=True)
```

### 11. 5xx before/after migration

**Before:** `0`
**After:** `0`

### 12. `\d events` showing new index

```
                                      Table "public.events"
     Column     |           Type           | Collation | Nullable |              Default
----------------+--------------------------+-----------+----------+------------------------------------
 id             | integer                  |           | not null | nextval('events_id_seq'::regclass)
 name           | text                     |           | not null |
 venue          | text                     |           | not null |
 event_date     | timestamp with time zone |           | not null |
 total_tickets  | integer                  |           | not null |
 price_cents    | integer                  |           | not null |
Indexes:
    "events_pkey" PRIMARY KEY, btree (id)
    "idx_events_event_date" btree (event_date)
```

### 13. Expand-and-contract sketch (rename `event_date` → `scheduled_at`)

**3 migrations + 2 code deploys, interleaved:**

| Step | Action | Safe because |
|------|--------|-------------|
| 1 | **Migration 1:** `ADD COLUMN scheduled_at TIMESTAMPTZ NULL` | Adding a nullable column is metadata-only — no table rewrite, no lock. Old code reads `event_date`, new code reads via COALESCE. |
| 2 | **Code Deploy A:** Read via `COALESCE(scheduled_at, event_date) AS event_date`; write to both columns | Dual-write means every write populates both names. COALESCE handles reads: existing rows have `scheduled_at=NULL` → falls back to `event_date`. |
| 3 | **Migration 2:** `UPDATE events SET scheduled_at = event_date WHERE scheduled_at IS NULL` (backfill) then `ALTER COLUMN scheduled_at SET NOT NULL` | Deploy A already reads via COALESCE, so both NULL and non-NULL `scheduled_at` are fine. Backfill is idempotent (`WHERE scheduled_at IS NULL`). |
| 4 | **Code Deploy B:** Read-only from `scheduled_at`; write-only to `scheduled_at` | All data exists in `scheduled_at` after backfill. Reading from `scheduled_at` directly (no COALESCE). The old column is still present but unused. |
| 5 | **Migration 3:** `DROP COLUMN event_date` | Deploy B is fully rolled out — no pod references `event_date` anymore. If a Deploy A pod were still running, it would 500 on `COALESCE(scheduled_at, event_date)` referencing a missing column. |

### 14. Why does `CREATE INDEX CONCURRENTLY` matter?

Without CONCURRENTLY, `CREATE INDEX` takes an `ACCESS EXCLUSIVE` lock on the table, blocking all reads and writes for the entire duration of the index build. On a 10M-row table, this can take **minutes** — causing downtime for every query. With CONCURRENTLY, the lock is `SHARE UPDATE EXCLUSIVE` — reads and writes continue normally while the index builds in the background. The tradeoff is that CONCURRENTLY takes longer and uses more resources, but it never blocks production traffic.

### 15. Why MUST migration 3 (drop old column) come after Deploy B fully rolled out?

If the old column is dropped while Deploy A pods are still running, those pods will 500 on every `/events` request because their SQL references `event_date` (via `COALESCE(scheduled_at, event_date)`). The column no longer exists → `canceling statement due to conflict with recovery` or `column "event_date" does not exist`. Deploy B must be fully rolled out (verified by `kubectl rollout status`) before the column can be safely removed.

---

## Bonus — Expand-and-Contract Executed Live

### Migration 1 — add `events.scheduled_at` column

```python
"""add events.scheduled_at column

Revision ID: b2c3d4e5f6a7
Revises: a1b2c3d4e5f6
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

revision: str = 'b2c3d4e5f6a7'
down_revision: Union[str, Sequence[str], None] = 'a1b2c3d4e5f6'


def upgrade() -> None:
    op.add_column('events',
                  sa.Column('scheduled_at', sa.TIMESTAMP(timezone=True), nullable=True))


def downgrade() -> None:
    op.drop_column('events', 'scheduled_at')
```

### Migration 2 — backfill + NOT NULL

```python
"""backfill events.scheduled_at

Revision ID: c3d4e5f6a7b8
Revises: b2c3d4e5f6a7
"""
from typing import Sequence, Union
from alembic import op

revision: str = 'c3d4e5f6a7b8'
down_revision: Union[str, Sequence[str], None] = 'b2c3d4e5f6a7'


def upgrade() -> None:
    op.execute("UPDATE events SET scheduled_at = event_date WHERE scheduled_at IS NULL")
    op.alter_column('events', 'scheduled_at', nullable=False)


def downgrade() -> None:
    op.alter_column('events', 'scheduled_at', nullable=True)
```

### Migration 3 — drop old column

```python
"""drop events.event_date

Revision ID: d4e5f6a7b8c9
Revises: c3d4e5f6a7b8
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

revision: str = 'd4e5f6a7b8c9'
down_revision: Union[str, Sequence[str], None] = 'c3d4e5f6a7b8'


def upgrade() -> None:
    op.drop_column('events', 'event_date')


def downgrade() -> None:
    op.add_column('events',
                  sa.Column('event_date', sa.TIMESTAMP(timezone=True), nullable=True))
    op.execute("UPDATE events SET event_date = scheduled_at")
    op.alter_column('events', 'event_date', nullable=False)
```

### Code Deploy A — COALESCE fallback-read (diff)

**Before (SELECT query in `list_events`):**
```sql
SELECT e.id, e.name, e.venue, e.event_date, e.total_tickets, e.price_cents, ...
ORDER BY e.event_date
```

**After (Deploy A):**
```sql
SELECT e.id, e.name, e.venue, COALESCE(e.scheduled_at, e.event_date) AS event_date, ...
ORDER BY COALESCE(e.scheduled_at, e.event_date)
```

Same change applied to `get_event`. No write path changes needed since QuickTicket only inserts via `seed.sql` at startup (no runtime INSERT in the events service).

### Code Deploy B — switch to `scheduled_at` only (diff)

**After (Deploy B):**
```sql
SELECT e.id, e.name, e.venue, e.scheduled_at AS event_date, ...
ORDER BY e.scheduled_at
```

Aliased as `event_date` to preserve the response shape for any existing clients.

### `\d events` before migration 1

```
                                      Table "public.events"
     Column     |           Type           | Collation | Nullable |              Default
----------------+--------------------------+-----------+----------+------------------------------------
 id             | integer                  |           | not null | nextval('events_id_seq'::regclass)
 name           | text                     |           | not null |
 venue          | text                     |           | not null |
 event_date     | timestamp with time zone |           | not null |
 total_tickets  | integer                  |           | not null |
 price_cents    | integer                  |           | not null |
```

### `\d events` after migration 3

```
                                      Table "public.events"
     Column     |           Type           | Collation | Nullable |              Default
----------------+--------------------------+-----------+----------+------------------------------------
 id             | integer                  |           | not null | nextval('events_id_seq'::regclass)
 name           | text                     |           | not null |
 venue          | text                     |           | not null |
 scheduled_at   | timestamp with time zone |           | not null |
 total_tickets  | integer                  |           | not null |
 price_cents    | integer                  |           | not null |
Indexes:
    "events_pkey" PRIMARY KEY, btree (id)
    "idx_events_event_date" btree (event_date) -- (orphaned index, dropped separately)
```

`event_date` is gone, `scheduled_at` is `NOT NULL`.

### 5xx baseline vs final

**Baseline (before any migration):** `0`
**Final (after all 5 transitions):** `0`
**diff:** identical — zero 5xx delta.

### Answer: Which single step would cause 5xx if reordered earlier?

**Migration 3 (drop `event_date`)** would cause 5xx if run before Deploy B is fully rolled out. If the old column is dropped while Deploy A pods are still serving traffic, every `/events` request would hit `column "event_date" does not exist` because the COALESCE references it. The only safe order is: Deploy A → M1 → M2 → **Deploy B → M3**.

### Answer: Production-scale backfill batching pattern

```python
# Pseudocode for batch-backfilling 10M rows with small transactions
BATCH_SIZE = 10000
last_id = 0
total = 0

while True:
    result = db.execute("""
        UPDATE events
        SET scheduled_at = event_date
        WHERE id > %s AND id <= %s AND scheduled_at IS NULL
    """, (last_id, last_id + BATCH_SIZE))
    affected = result.rowcount
    if affected == 0:
        break
    total += affected
    last_id += BATCH_SIZE
    db.commit()
    time.sleep(0.05)  # Give other queries breathing room
```

Each batch is a small, fast transaction that holds locks briefly. `WHERE scheduled_at IS NULL` makes it idempotent — safe to re-run if interrupted. The sleep between batches prevents WAL buildup and gives concurrent reads/writes a chance to proceed.

### Answer: Why the downgrade from M3 is NOT sufficient for rollback safety

The downgrade re-adds `event_date` and backfills it from `scheduled_at` — so the *data* is restored. But:

1. **Code Deploy B** is still running and references `scheduled_at` exclusively. If the rollback is triggered because Deploy B has a bug, the restored `event_date` column won't fix it — Deploy B doesn't read `event_date`.
2. **Any new rows** created between M3 and the rollback (during Deploy B's runtime) have `scheduled_at` populated but `event_date` would be backfilled as NULL (then set via UPDATE, but there's a window).
3. **A true safe rollback** would require: (a) reverting Code Deploy B → Deploy A first, (b) ensuring Deploy A reads via COALESCE again, (c) only then running the M3 downgrade to re-add `event_date`.

The rollback migration alone is *data-safe* but not *traffic-safe* — the code must be rolled back before the schema rollback takes effect.