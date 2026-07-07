# Lab 9 — Stateful Services & DB Reliability

## Task 1 — Migrations & Backup/Restore

### 1. `alembic history` output showing the two revisions (baseline + email)

```bash
$ alembic history

09495fc813b4 -> ce5c023bea85 (head), add email column to events
<base> -> 09495fc813b4, baseline - pre-existing schema
```

---

### 2. `\d events` output showing the new `email` column

```bash
$ kubectl exec -i $(kubectl get pod -l app=postgres -o name) -- \
  psql -U quickticket -d quickticket -c '\d events'

                                        Table "public.events"
    Column     |           Type           | Collation | Nullable |              Default
---------------+--------------------------+-----------+----------+------------------------------------
 id            | integer                  |           | not null | nextval('events_id_seq'::regclass)
 name          | text                     |           | not null |
 venue         | text                     |           | not null |
 event_date    | timestamp with time zone |           | not null |
 total_tickets | integer                  |           | not null |
 price_cents   | integer                  |           | not null |
 email         | character varying(255)   |           |          |

Indexes:
    "events_pkey" PRIMARY KEY, btree (id)
```

---

### 3. `time alembic upgrade head` output (elapsed time — expect <1s for nullable add)

```bash
$ time alembic upgrade head

INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 09495fc813b4 -> ce5c023bea85, add email column to events

alembic upgrade head  0,24s user 0,03s system 90% cpu 0,301 total
```

---

### 4. Prometheus `5xx last 1min` before and after migration

**Before migration**

```text
5xx last 1min: 0
```

**After migration**

```text
5xx last 1min: 0
```

---

### 5. `ls -lh /tmp/quickticket.dump` + `pg_restore --list` output showing backup is valid

**Backup file**

```bash
$ ls -lh /tmp/quickticket.dump

-rw-rw-r-- 1 eoskadi eoskadi 7.2K Jul 7 02:24 /tmp/quickticket.dump
```

```bash
$ file /tmp/quickticket.dump

/tmp/quickticket.dump: PostgreSQL custom database dump - v1.16-0
```

**Backup contents**

```bash
$ kubectl exec $POD -- pg_restore --list /tmp/backup.dump | head -25

;
; Archive created at 2026-07-06 23:24:42 UTC
;     dbname: quickticket
;     TOC Entries: 18
;     Compression: gzip
;     Dump Version: 1.16-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 17.10
;     Dumped by pg_dump version: 17.10
;
; Selected TOC Entries:
;
220; 1259 16411 TABLE public alembic_version quickticket
218; 1259 16389 TABLE public events quickticket
217; 1259 16388 SEQUENCE public events_id_seq quickticket
3481; 0 0 SEQUENCE OWNED BY public events_id_seq quickticket
219; 1259 16397 TABLE public orders quickticket
3316; 2604 16392 DEFAULT public events id quickticket
3474; 0 16411 TABLE DATA public alembic_version quickticket
3472; 0 16389 TABLE DATA public events quickticket
3473; 0 16397 TABLE DATA public orders quickticket
3482; 0 0 SEQUENCE SET public events_id_seq quickticket
```

---

### 6. Row counts before disaster / after DROP / after restore for `events` and `orders`

**Before disaster**

```bash
$ kubectl exec $POD -- psql -U quickticket -d quickticket \
  -c 'SELECT count(*) FROM events; SELECT count(*) FROM orders'

 count
-------
    10
(1 row)

 count
-------
    50
(1 row)
```

**After DROP**

```bash
$ kubectl exec $POD -- psql -U quickticket -d quickticket \
  -c 'SELECT count(*) FROM events; SELECT count(*) FROM orders'

 count
-------
    10
(1 row)

ERROR: relation "orders" does not exist
LINE 1: SELECT count(*) FROM events; SELECT count(*) FROM orders
```

```bash
$ kubectl run smoke --image=curlimages/curl:latest --rm -i --restart=Never --quiet \
  --command -- curl -s -o /dev/null -w "/events=%{http_code}\n" http://gateway:8080/events

/events=502
```

**After restore**

```bash
$ kubectl exec $POD -- psql -U quickticket -d quickticket \
  -c 'SELECT count(*) FROM events; SELECT count(*) FROM orders'

 count
-------
    10
(1 row)

 count
-------
    50
(1 row)
```

```bash
$ kubectl run smoke --image=curlimages/curl:latest --rm -i --restart=Never --quiet \
  --command -- curl -s -o /dev/null -w "/events=%{http_code}\n" http://gateway:8080/events

/events=200
```

---

### 7. What's the RPO of your current setup (single `pg_dump`)? How would you improve it?

The current RPO is the time since the last `pg_dump` backup. Any data created after the backup will be lost. I would improve it by running automated backups with a Kubernetes CronJob. Using a PersistentVolumeClaim (PVC) and WAL archiving would reduce the risk of data loss even further.

## Task 2 — Disaster Recovery Under Load

### 1. Timestamps for the disaster recovery process

```text
Disaster at      02:38:01
New pod ready    02:38:18
Restored         02:38:47
App fully up     02:39:02
```

---

### 2. Actual RTO

```text
RTO = 61 seconds
```

---

### 3. Orders count before disaster vs after restore (RPO gap)

**Before disaster**

```bash
$ kubectl exec $POD -- psql -U quickticket -d quickticket \
  -c 'SELECT count(*) FROM orders'

 count 
-------
    50
(1 row)
```

**After restore**

```bash
$ kubectl exec $NEW_POD -- psql -U quickticket -d quickticket \
  -c 'SELECT count(*) FROM orders'

 count 
-------
    50
(1 row)
```

**RPO gap**

```text
Orders before disaster: 50
Orders after restore: 50
Lost orders (RPO gap): 0
```

---

### 4. Prometheus error-rate around the incident

```bash
$ kubectl exec -n monitoring deployment/prometheus -- wget -qO- \
'http://localhost:9090/api/v1/query?query=sum(rate(gateway_requests_total%7Bstatus%3D~%225..%22%7D%5B30s%5D))'

{"status":"success","data":{"resultType":"vector","result":[{"metric":{},"value":[1783381397.329,"0"]}]}}%     
```

---

### 5. Why was the new Postgres pod empty? How would you eliminate this failure mode?

The new Postgres pod was empty because the deployment used ephemeral container storage without a PersistentVolumeClaim (PVC). When the pod was deleted, all database files were lost. This failure can be eliminated by using a PVC so the database survives pod restarts and recreations.

---

## Bonus Task — Persistent Storage + Automated Backup CronJob

### B.1: Diff of `k8s/postgres.yaml` (PVC added)

```diff
diff --git a/k8s/postgres.yaml b/k8s/postgres.yaml
index 9bb51e8..7177c6f 100644
--- a/k8s/postgres.yaml
+++ b/k8s/postgres.yaml
@@ -1,3 +1,15 @@
+---
+apiVersion: v1
+kind: PersistentVolumeClaim
+metadata:
+  name: postgres-data
+spec:
+  accessModes: [ReadWriteOnce]
+  resources:
+    requests:
+      storage: 1Gi
+
+---
 apiVersion: apps/v1
 kind: Deployment
 metadata:
@@ -24,6 +36,11 @@ spec:
               value: "quickticket"
             - name: POSTGRES_PASSWORD
               value: "quickticket"
+            - name: PGDATA
+              value: /var/lib/postgresql/data/pgdata
+          volumeMounts:
+            - name: data
+              mountPath: /var/lib/postgresql/data
           resources:
             requests:
               cpu: 50m
@@ -31,6 +48,10 @@ spec:
             limits:
               cpu: 200m
               memory: 256Mi
+      volumes:
+        - name: data
+          persistentVolumeClaim:
+            claimName: postgres-data
 
 ---
 apiVersion: v1
@@ -43,5 +64,4 @@ spec:
   ports:
     - port: 5432
       targetPort: 5432
-  type: ClusterIP
-
+  type: ClusterIP
\ No newline at end of file
```

---

### B.2: Re-run timestamps showing the new RTO with PVC (pod-restart-only, no `pg_restore` needed)

```text
Disaster at      02:49:42
New pod ready    02:49:42
App fully up     02:49:55
```

```text
RTO = 13 seconds
```

**Data survival after pod restart (PVC):**

```bash
$ kubectl exec $NEW_POD -- psql -U quickticket -d quickticket -c '\dt'

           List of relations
 Schema |  Name  | Type  |    Owner
--------+--------+-------+-------------
 public | events | table | quickticket
 public | orders | table | quickticket
(2 rows)

$ kubectl exec $NEW_POD -- psql -U quickticket -d quickticket \
  -c 'SELECT count(*) FROM events; SELECT count(*) FROM orders'

 count
-------
     5
(1 row)

 count
-------
    25
(1 row)
```

No `pg_restore` was needed — the data persisted on the PVC. RTO dropped from 61s (Task 2) to 13s (just pod restart + app reconnect).

---

### B.3: `k8s/backup-cronjob.yaml` contents

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: pg-dump
              image: postgres:17-alpine
              env:
                - name: PGHOST
                  value: postgres
                - name: PGUSER
                  value: quickticket
                - name: PGPASSWORD
                  value: quickticket
                - name: PGDATABASE
                  value: quickticket
              command:
                - sh
                - -c
                - |
                  TIMESTAMP=$(date -u +%Y%m%dT%H%M%S)
                  FILENAME="/backups/quickticket_${TIMESTAMP}.dump"
                  pg_dump -Fc -f "$FILENAME"
                  echo "Backup written: $FILENAME"
                  ls -1t /backups/quickticket_*.dump | tail -n +6 | xargs -r rm -v
              volumeMounts:
                - name: backups
                  mountPath: /backups
          volumes:
            - name: backups
              persistentVolumeClaim:
                claimName: postgres-backups
```

---

### B.4: Logs from `manual-7` showing the rotation kicked in

```text
=== manual-1 ===
Backup written: /backups/quickticket_20260706T235035.dump

=== manual-2 ===
Backup written: /backups/quickticket_20260706T235039.dump

=== manual-3 ===
Backup written: /backups/quickticket_20260706T235043.dump

=== manual-4 ===
Backup written: /backups/quickticket_20260706T235047.dump

=== manual-5 ===
Backup written: /backups/quickticket_20260706T235050.dump

=== manual-6 ===
Backup written: /backups/quickticket_20260706T235053.dump
removed '/backups/quickticket_20260706T235035.dump'

=== manual-7 ===
Backup written: /backups/quickticket_20260706T235056.dump
removed '/backups/quickticket_20260706T235039.dump'
```

Jobs 1–5 created backups without deletion. Job 6 removed the oldest (1st file). Job 7 removed the next oldest (2nd file) — keeping exactly the 5 newest.

---

### B.5: Output of `ls -la /backups` showing exactly 5 files after 7 runs

```bash
$ kubectl exec deployment/backup-inspector -- ls -la /backups

total 48
drwxrwxrwx    2 root     root          4096 Jul  6 23:50 .
drwxr-xr-x    1 root     root          4096 Jul  6 23:49 ..
-rw-r--r--    1 root     root          5448 Jul  6 23:50 quickticket_20260706T235043.dump
-rw-r--r--    1 root     root          5448 Jul  6 23:50 quickticket_20260706T235047.dump
-rw-r--r--    1 root     root          5448 Jul  6 23:50 quickticket_20260706T235050.dump
-rw-r--r--    1 root     root          5448 Jul  6 23:50 quickticket_20260706T235053.dump
-rw-r--r--    1 root     root          5448 Jul  6 23:50 quickticket_20260706T235056.dump
```

7 runs → 5 newest files retained. Retention works correctly.
