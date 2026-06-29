# Lab 8 — Chaos Engineering: Break Things on Purpose

## Task 1 — Three Chaos Experiments

## Experiment 1 – Pod Kill Under Load

### 1. Hypothesis

**Hypothesis:** If I delete one gateway pod while traffic is flowing, the application will continue serving requests with little or no disruption because Kubernetes will automatically create a replacement pod and redistribute traffic to the remaining healthy replicas.

---

### 2. Commands Executed

```bash
VICTIM=$(kubectl get pods -l app=gateway -o name | head -1)
kubectl delete "$VICTIM"
```

```bash
kubectl get pods -l app=gateway -w
```

```bash
kubectl exec -n monitoring deployment/prometheus -- wget -qO- \
'http://localhost:9090/api/v1/query?query=sum(increase(gateway_requests_total%7Bstatus%3D~%225..%22%7D%5B3m%5D))'
```

```bash
kubectl exec -n monitoring deployment/prometheus -- wget -qO- \
'http://localhost:9090/api/v1/query?query=sum+by+(pod)+(rate(gateway_requests_total%5B1m%5D))'
```

---

### 3. What I Observed
**Timestamp:** 21:17:34 — the gateway pod was deleted.
The following pod was deleted:

```text
pod "gateway-668ddfb4d9-hh9pr" deleted from default namespace
```

Kubernetes immediately started creating a replacement pod. During recovery, the pod status changed from `0/1 Running` to `1/1 Running` within a few seconds.

```text
NAME                       READY   STATUS    RESTARTS   AGE
gateway-668ddfb4d9-j6qp6   1/1     Running   0          7h41m
gateway-668ddfb4d9-lrtxb   1/1     Running   0          7h42m
gateway-668ddfb4d9-sgs62   1/1     Running   0          7h45m
gateway-668ddfb4d9-zx84b   0/1     Running   0          5s
gateway-668ddfb4d9-zxjcw   1/1     Running   0          7h44m
gateway-668ddfb4d9-zx84b   1/1     Running   0          8s
```

After recovery, all five gateway replicas were running again.

The Prometheus query for HTTP 5xx responses returned:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [1782757429.509, "1845.5109801304448"]
      }
    ]
  }
}
```

The request rate was distributed across all gateway pods after the replacement became ready.

Immediately after recovery:

```text
gateway-668ddfb4d9-j6qp6   2.98 RPS
gateway-668ddfb4d9-lrtxb   2.89 RPS
gateway-668ddfb4d9-sgs62   3.35 RPS
gateway-668ddfb4d9-zxjcw   2.69 RPS
gateway-668ddfb4d9-zx84b   3.82 RPS
```

A few minutes later, the load remained balanced:

```text
gateway-668ddfb4d9-j6qp6   3.27 RPS
gateway-668ddfb4d9-lrtxb   3.05 RPS
gateway-668ddfb4d9-sgs62   3.42 RPS
gateway-668ddfb4d9-zxjcw   3.29 RPS
gateway-668ddfb4d9-zx84b   2.71 RPS
```

These observations show that Kubernetes automatically recreated the failed pod and the Service redistributed traffic among all available replicas.

---

### 4. Comparison: Hypothesis vs Reality

The experiment confirmed the hypothesis. Deleting one gateway pod did not cause the deployment to fail. Kubernetes automatically created a replacement pod, which became ready within a few seconds. During the recovery process, the remaining gateway pods continued serving requests, and after the new pod became ready, traffic was distributed across all five replicas again.

One unexpected observation was the high value returned by the Prometheus query for HTTP 5xx responses. Because the query measures the increase over the previous three minutes, these errors cannot be attributed solely to deleting the gateway pod and may include failures that occurred before the experiment.

---

### 5. To Improve Resilience Against This Failure, I Would...

To improve resilience against this failure, I would configure a PodDisruptionBudget and ensure readiness probes prevent traffic from being routed to new pods until they are fully initialized.

## Experiment 2 – Payment Latency Injection

### 1. Hypothesis

**Hypothesis:** If the payment service takes 2 seconds to process each request, the `/pay` endpoint latency will increase, but the gateway will continue serving requests without significant failures because the payment latency is still below the gateway timeout of 5 seconds.

---

### 2. Commands Executed

```bash
kubectl set env deployment/payments PAYMENT_LATENCY_MS=2000
```

```bash
kubectl rollout status deployment/payments --timeout=30s
```

```bash
kubectl exec -n monitoring deployment/prometheus -- wget -qO- \
'http://localhost:9090/api/v1/query?query=sum(rate(gateway_requests_total%7Bstatus%3D~%225..%22%7D%5B1m%5D))/sum(rate(gateway_requests_total%5B1m%5D))'
```

```bash
kubectl exec -n monitoring deployment/prometheus -- wget -qO- \
'http://localhost:9090/api/v1/query?query=histogram_quantile(0.99,+sum+by+(le,path)+(rate(gateway_request_duration_seconds_bucket%5B1m%5D)))'
```

After the experiment, the original configuration was restored:

```bash
kubectl set env deployment/payments PAYMENT_LATENCY_MS=0
kubectl rollout status deployment/payments --timeout=30s
```

---

### 3. What I Observed

The payment deployment was successfully updated using a rolling update.

```text
deployment.apps/payments env updated

Waiting for deployment "payments" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "payments" rollout to finish: 1 old replicas are pending termination...
deployment "payments" successfully rolled out
```

The Prometheus query for the HTTP 5xx error ratio returned:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [1782758146.844, "0.6623376224715459"]
      }
    ]
  }
}
```

The Prometheus query for p99 latency per endpoint returned:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "path": "/health"
        },
        "value": [1782758153.318, "0.09200014544969734"]
      },
      {
        "metric": {
          "path": "/events"
        },
        "value": [1782758153.318, "0.024819986622735338"]
      },
      {
        "metric": {
          "path": "/events/{id}/reserve"
        },
        "value": [1782758153.318, "NaN"]
      }
    ]
  }
}
```

After the experiment, the payment service configuration was restored successfully.

```text
deployment.apps/payments env updated

Waiting for deployment "payments" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "payments" rollout to finish: 1 old replicas are pending termination...
deployment "payments" successfully rolled out
```

---

### 4. Comparison: Hypothesis vs Reality

The experiment partially confirmed the hypothesis. The payment service accepted the new configuration and completed a successful rolling update. The system remained operational after the latency injection. The observed p99 latency for `/events` and `/health` remained low, indicating that read operations were not noticeably affected. However, the expected `/pay` latency metric was not present in the Prometheus output, so it was not possible to directly verify the latency increase for payment requests. The Prometheus query also reported a non-zero HTTP 5xx error ratio (0.6623), indicating that some requests failed during the observation period.

---

### 5. To Improve Resilience Against This Failure, I Would...

To improve resilience against slow payment processing, I would implement a circuit breaker with appropriate request timeouts and fallback handling to prevent slow downstream services from affecting the overall system.


## Experiment 3 – Redis Failure

### 1. Hypothesis

**Hypothesis:** If Redis becomes unavailable, users should still be able to list events because event data is retrieved directly from the database. However, users should not be able to reserve tickets because the reservation workflow depends on Redis for concurrency control.

---

### 2. Commands Executed

```bash
kubectl scale deployment/redis --replicas=0

kubectl get pods -l app=redis -w

kubectl run chaos-probe \
  --image=curlimages/curl:latest \
  --rm -i \
  --restart=Never \
  --quiet \
  --command -- \
  sh -c '
    echo "GET /events:";
    curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" \
      http://gateway:8080/events;

    echo "POST /reserve:";
    curl -s -X POST \
      -w "%{http_code} %{time_total}s\n" \
      -H "Content-Type: application/json" \
      -d "{\"quantity\":1}" \
      http://gateway:8080/events/1/reserve;

    echo "GET /health:";
    curl -s http://gateway:8080/health
  '

kubectl scale deployment/redis --replicas=1

kubectl wait \
  --for=condition=Available \
  deployment/redis \
  --timeout=60s
```

---

### 3. What I Observed

The Redis deployment was successfully scaled down to zero replicas, leaving no Redis pods running.

```text
deployment.apps/redis scaled
```

The chaos probe produced the following results:

```text
GET /events:
200 0.021339s

POST /reserve:
500 0.011139s

GET /health:
{"status":"degraded","checks":{"events":"down","payments":"ok","circuit_payments":"CLOSED"}}
```

* `GET /events` returned **200 OK**, demonstrating that event listing continued to function without Redis.
* `POST /events/1/reserve` returned **500 Internal Server Error**, confirming that ticket reservations depend on Redis.
* The `/health` endpoint reported a **degraded** system state, with the `events` service marked as **down** while the `payments` service remained healthy.

After scaling Redis back to one replica, the deployment became available again, confirming successful recovery.

```text
deployment.apps/redis scaled
deployment.apps/redis condition met
```

---

### 4. Comparison: Hypothesis vs. Reality

The experiment confirmed the hypothesis. Users were still able to list events after Redis was disabled, while ticket reservations failed because the reservation workflow relies on Redis for concurrency control.

One unexpected observation was that the `/health` endpoint reported the **events** service as **down** instead of identifying Redis as unavailable. This suggests that the health status of the events service is directly tied to Redis availability, which appears to be an implementation detail rather than an indication that the service itself has failed.

---

### 5. To Improve Resilience Against This Failure, I Would...

To improve resilience against this type of failure, I would:

* Deploy Redis in a highly available configuration (e.g., **Redis Sentinel** or **Redis Cluster**) to provide automatic failover.
* Implement graceful degradation so that Redis-dependent features fail gracefully while read-only operations remain available.
* Add robust error handling, retries where appropriate, and sensible timeouts for Redis operations to prevent the application from becoming unresponsive when Redis is unavailable.


## Task 2 – Combined Failure Scenario

### Scenario Design

I designed a combined failure scenario by injecting multiple faults simultaneously:

* Payment service configured with a 30% failure rate and 500 ms artificial latency.
* Events service database connection pool limited to 3 connections.
* Load increased by scaling the `mixedload` deployment to 3 replicas.

The objective was to identify which component would become the bottleneck under combined load and degraded dependencies.

---

### Observations

The experiment was executed for approximately 3–5 minutes while repeatedly collecting Prometheus metrics.

The gateway error rate gradually increased during the experiment:

* 0.6549
* 0.6611
* 0.6710
* 0.6881
* 0.7001
* 0.7067
* 0.7054
* 0.7042

The p99 latency remained relatively stable throughout the experiment:

| Endpoint               | Approximate p99 latency |
| ---------------------- | ----------------------: |
| `/events`              |           0.028–0.048 s |
| `/events/{id}/reserve` |          ~0.048–0.050 s |
| `/health`              |           0.060–0.087 s |

No dramatic latency spikes were observed, although the gateway maintained a consistently high error ratio throughout the test.

---

### Weakest Link

The weakest component appeared to be the payment service. Injecting payment failures together with additional latency caused the gateway error ratio to remain above 65% and gradually increase to approximately 70% under higher load.

The endpoint with the highest latency amplification was `/events/{id}/reserve`, while the read endpoint `/events` remained relatively stable.

---

### Resilience Improvement

To improve resilience, I would introduce a circuit breaker for the payment service, increase the database connection pool only after load testing, and add autoscaling based on request latency and error rate. These changes would help isolate downstream failures and prevent them from affecting the entire application.
