# Lab 3 — Monitoring & Observability with Prometheus and Grafana

## Task 1 — Golden Signals Dashboard

### 1. Output of compose ps showing all 7 services

```bash
$ dc ps

NAME               IMAGE                     COMMAND                  SERVICE      CREATED          STATUS                    PORTS
app-events-1       app-events                "uvicorn main:app --…"   events       11 seconds ago   Up 4 seconds              0.0.0.0:8081->8081/tcp, [::]:8081->8081/tcp
app-gateway-1      app-gateway               "uvicorn main:app --…"   gateway      11 seconds ago   Up 4 seconds              0.0.0.0:3080->8080/tcp, [::]:3080->8080/tcp
app-grafana-1      grafana/grafana:13.0.1    "/run.sh"                grafana      33 minutes ago   Up 10 seconds             0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
app-payments-1     app-payments              "uvicorn main:app --…"   payments     11 seconds ago   Up 10 seconds             0.0.0.0:8082->8082/tcp, [::]:8082->8082/tcp
app-postgres-1     postgres:17-alpine        "docker-entrypoint.s…"   postgres     10 days ago      Up 10 seconds (healthy)   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp
app-prometheus-1   prom/prometheus:v3.11.2   "/bin/prometheus --c…"   prometheus   33 minutes ago   Up 10 seconds             0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp
app-redis-1        redis:7-alpine            "docker-entrypoint.s…"   redis        10 days ago      Up 10 seconds (healthy)   0.0.0.0:6379->6379/tcp, [::]:6379->6379/tcp
```

---

### 2. Prometheus targets output (all 3 up)

```bash
$ curl -s http://localhost:9090/api/v1/targets | python3 -c "
import sys, json
for t in json.load(sys.stdin)['data']['activeTargets']:
    print(f\"{t['labels']['job']:12} {t['health']:8} {t['scrapeUrl']}\")
"
```

Output:

```text
events       up       http://events:8081/metrics
gateway      up       http://gateway:8080/metrics
payments     up       http://payments:8082/metrics
```

---

### 3. Custom metrics list

```bash
$ curl -s http://localhost:9090/api/v1/label/__name__/values | python3 -c "
import sys, json
for n in json.load(sys.stdin)['data']:
    if any(x in n for x in ['gateway_', 'events_', 'payments_']):
        print(n)
"
```

Output:

```text
events_db_pool_size
events_orders_created
events_orders_total
events_reservations_active
```

---

### 4. PromQL query output (request rate)

PromQL:

```promql
sum(rate(gateway_requests_total[5m]))
```

Query execution:

```bash
$ curl -s --data-urlencode 'query=sum(rate(gateway_requests_total[5m]))' \
  http://localhost:9090/api/v1/query | python3 -c "
import sys, json
r = json.load(sys.stdin)
print(f\"Request rate: {float(r['data']['result'][0]['value'][1]):.2f} req/s\")"
```

Output:

```text
Request rate: 0.29 req/s
```

---

### 5. PromQL queries used for Latency and Saturation panels

#### Latency

##### p50

```promql
histogram_quantile(
  0.50,
  sum(rate(gateway_request_duration_seconds_bucket[1m])) by (le)
)
```

##### p95

```promql
histogram_quantile(
  0.95,
  sum(rate(gateway_request_duration_seconds_bucket[1m])) by (le)
)
```

##### p99

```promql
histogram_quantile(
  0.99,
  sum(rate(gateway_request_duration_seconds_bucket[1m])) by (le)
)
```

#### Saturation

```promql
events_db_pool_size
```

---

### 6. Dashboard observations: normal traffic vs payments failure

#### Normal

* Error rate 0%.
* p99 latency ~23 ms.
* Availability 100%.
* Burn rate 0.

#### Payments down

* Gateway could not reach the payments service.
* Purchase-flow requests (~10% of traffic) started failing.
* Error rate increased to ~5%.
* Latency remained stable (or changed only slightly), suggesting the gateway failed quickly rather than waiting for long timeouts.
* DB pool utilization remained unchanged because the events service was not affected.
* Availability and burn-rate SLI windows reacted more slowly and started changing about 90 seconds later.

---

### 7. Answer: Which golden signal showed the failure first? How long after killing payments?

**Error Rate** detected the incident first. After the payments service was stopped, errors became visible roughly **30 seconds later**. The delay was caused by Prometheus collection intervals and the use of a rate-based query window.

**Latency** and **Saturation** remained largely unchanged, indicating that requests failed quickly rather than timing out and that database resources were not under pressure. Availability and SLO metrics reacted later because they were based on a longer observation window.


## Task 2 — Define SLOs & Recording Rules

### 1. SLI/SLO definitions with error budget math

#### Availability

- **SLI:** percentage of gateway requests returning a non-5xx response.
- **SLO:** 99.5% availability over a rolling 7-day window.

#### Latency

- **SLI:** percentage of gateway requests completed within 500 ms.
- **SLO:** 95% of requests must complete within 500 ms.

#### Error Budget

Expected traffic:

```text
1000 requests/day × 7 days = 7000 requests/week
```

Error budget:

```text
100% − 99.5% = 0.5%
```

Allowed failures:

```text
7000 × 0.005 = 35 failed requests/week
```

---

### 2. Rules loaded output

```text
gateway:sli_availability:ratio_rate5m         = ok
gateway:sli_latency_500ms:ratio_rate5m        = ok
gateway:error_budget_burn_rate:ratio_rate5m   = ok
```

---

### 3. SLO gauge observation during failure

The Grafana panel **"SLO – Availability (7d target 99.5%)"** displays the metric:

```promql
gateway:sli_availability:ratio_rate5m * 100
```

During the payments outage:

- Availability decreased from **100.000%** to **98.409%**.
- The **99.5% SLO** threshold was breached.
- Burn rate increased from **0** to **3.18** and peaked at approximately **5**.

This indicates that the error budget was being consumed 3–5 times faster than allowed by the defined SLO.

Because the metric uses a 5-minute rolling window, recovery was gradual after the payments service was restored.

---

## Bonus Task — Correlate Failure Across Metrics & Logs

### 1. Failure injection setup

Traffic generation:

```bash
./loadgen/run.sh 5 120 &
```

After 30 seconds, the payments service was restarted with:

```bash
PAYMENT_FAILURE_RATE=0.5
PAYMENT_LATENCY_MS=1000
```

Health endpoint:

```text
{"status":"healthy","failure_rate":0.5,"latency_ms":1000}
```

---

### 2. Failure timeline

```text
12:14:05  Fault injection applied
12:14:13  First "Injecting 1000ms latency" log entry
12:14:32  First injected payment failure (HTTP 500)
12:16:09  Grafana shows p99 latency spike (~2086 ms)
12:16:13  Burn rate increases to ~6
12:16:36  Payments service restored
```

---

### 3. Log excerpts

#### payments

```text
{"level":"INFO","service":"payments","msg":"Injecting 1000ms latency for a9033244-..."}
{"level":"WARNING","service":"payments","msg":"Payment failed (injected) for b1e533r6-..."}
INFO: 172.19.0.8:56240 - "POST /charge HTTP/1.1" 500 Internal Server Error
```

#### gateway

```text
{"level":"ERROR","service":"gateway","msg":"payment error: [Errno -2] Name or service not known"}
INFO: "POST /reserve/.../pay HTTP/1.1" 502 Bad Gateway
```

---

### 4. Metrics correlation

During the injected failure:

- p99 latency increased to approximately **2086 ms**.
- Availability dropped to **96.987%**.
- Burn rate reached **6.03**.

The metric changes occurred at the same time as the injected latency and HTTP 500 errors observed in the payments logs.

---

### 5. Root cause

The payments service was intentionally configured to:

- add **1000 ms latency** to requests;
- return **HTTP 500** for approximately **50%** of payment operations.

As a result, Grafana showed a significant increase in latency together with a reduction in availability and an increase in burn rate. The payments logs confirmed the injected latency and failures, allowing the dashboard metrics to be directly correlated with the underlying cause of the degradation.
