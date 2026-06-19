# Lab 4 — Kubernetes: Deploy QuickTicket to a Cluster

## Task 1 — Write Manifests & Deploy to k3d

### 1. Output of `kubectl get nodes`

```text
NAME                       STATUS   ROLES           AGE     VERSION
k3d-quickticket-server-0   Ready    control-plane   5m17s   v1.35.5+k3s1
```

---

### 2. Output of `kubectl get pods,svc` showing all running

```text
NAME                            READY   STATUS    RESTARTS   AGE
pod/postgres-76cd478b6b-5l5ct   1/1     Running   0          3m32s
pod/redis-65bb44458c-2tn9m      1/1     Running   0          3m32s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP    28m
service/postgres     ClusterIP   10.43.103.180   <none>        5432/TCP   3m32s
service/redis        ClusterIP   10.43.142.14    <none>        6379/TCP   3m32s
```

---

### 3. Output of `curl localhost:3080/events` via port-forward

```text
{
    "detail": "Events service unavailable"
}

{
    "status": "degraded",
    "checks": {
        "events": "degraded",
        "payments": "ok",
        "circuit_payments": "CLOSED"
    }
}
```

---

### 4. Output of `kubectl get pods -w` during pod deletion showing auto-recovery

```text
pod "gateway-6fc44f68c5-xxnz9" deleted from default namespace

NAME                       READY   STATUS    RESTARTS   AGE
events-6c4df7d6-jnr7x      1/1     Running   0          4m19s
gateway-6fc44f68c5-2fck9   1/1     Running   0          2s
payments-58fb468db-x4cjz   1/1     Running   0          4m19s
postgres-7c7ffc4b-2lh27    1/1     Running   0          4m19s
redis-c46d5dffc-445rc      1/1     Running   0          4m19s
```

---

### 5. Comparison of Kubernetes recovery vs Docker Compose

Kubernetes recreated the deleted pod in approximately **2 seconds**.

Unlike Docker Compose, Kubernetes automatically restored the desired state without manual intervention. With Docker Compose, the container would typically need to be restarted manually.

---

# Task 2 — Probes & Resource Limits

### 1. `kubectl describe pod` output showing probes configured

```text
Liveness:       http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3
Readiness:      http-get http://:8080/health delay=0s timeout=1s period=5s #success=1 #failure=2

Environment:
  EVENTS_URL:          http://events:8081
  PAYMENTS_URL:        http://payments:8082
  GATEWAY_TIMEOUT_MS:  5000
```

---

### 2. Output during Redis deletion showing readiness probe behaviour

```text
redis-c46d5dffc-rp2h8       1/1     Running             0          2s
redis-c46d5dffc-rp2h8       1/1     Terminating         0          17s
redis-c46d5dffc-mm9k8       0/1     Pending             0          0s
redis-c46d5dffc-mm9k8       0/1     ContainerCreating   0          0s
redis-c46d5dffc-rp2h8       0/1     Completed           0          17s
redis-c46d5dffc-mm9k8       1/1     Running             0          1s
```

---

### 3. `kubectl describe node` output showing allocated resources

```text
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)

  Resource           Requests    Limits
  --------           --------    ------
  cpu                450m (2%)   1 (6%)
  memory             460Mi (2%)  1450Mi (9%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)

Events: <none>
```

---

### 4. Liveness vs Readiness

A readiness probe failure removes a pod from Service endpoints but does not restart it.

A liveness probe failure causes Kubernetes to restart the container.

Database connectivity should be checked using a readiness probe because restarting the application does not solve a database outage. The pod should stop receiving traffic until the database becomes available again.

---

# Bonus Task — Helm Chart

### 1. Chart.yaml

```yaml
apiVersion: v2
name: quickticket
description: QuickTicket SRE learning project
version: 0.1.0
```

### 2. values.yaml

```yaml
gateway:
  replicas: 1
  image: quickticket-gateway:v1

events:
  replicas: 1
  image: quickticket-events:v1
  db:
    host: postgres
    port: 5432
    name: quickticket
    user: quickticket
    password: quickticket

payments:
  replicas: 1
  image: quickticket-payments:v1
  failureRate: "0.0"
  latencyMs: "0"
```

---

### 3. Output of `helm list`

```text
NAME         NAMESPACE   REVISION   UPDATED                                  STATUS     CHART               APP VERSION
quickticket  default     1          2026-06-19 20:10:00.732261831 +0300 MSK deployed   quickticket-0.1.0
```

---

### 4. Output of `kubectl get pods` after Helm install

```text
NAME                        READY   STATUS    RESTARTS   AGE
events-78696fcf65-tr6jw     0/1     Running   0          10s
gateway-7cd55d8774-stlkb    0/1     Running   0          10s
payments-d7dc94485-b4hvc    1/1     Running   0          10s
postgres-76cd478b6b-4pm5w   1/1     Running   0          10s
redis-65bb44458c-nf4mr      1/1     Running   0          10s
```
