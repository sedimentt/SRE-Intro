# Lab 11 — Submission

## 1. Notifications Service

### `app/notifications/main.py`

```python
"""QuickTicket Notifications — Mock notification service with tunable failures."""

import os
import uuid
import time
import random
import logging

from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

NOTIFY_FAILURE_RATE = float(os.getenv("NOTIFY_FAILURE_RATE", "0.0"))
NOTIFY_LATENCY_MS = int(os.getenv("NOTIFY_LATENCY_MS", "0"))

logging.basicConfig(
    format='{"time":"%(asctime)s","level":"%(levelname)s","service":"notifications","msg":"%(message)s"}',
    level=logging.INFO,
)
log = logging.getLogger("notifications")

app = FastAPI(title="QuickTicket Notifications", version="1.0.0")

REQUEST_COUNT = Counter("notifications_requests_total", "Total requests", ["method", "path", "status"])
REQUEST_DURATION = Histogram("notifications_request_duration_seconds", "Request duration", ["method", "path"])
NOTIFY_TOTAL = Counter("notifications_notify_total", "Total notify attempts", ["result"])


@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    path = request.url.path
    if not path.startswith("/metrics"):
        REQUEST_COUNT.labels(request.method, path, response.status_code).inc()
        REQUEST_DURATION.labels(request.method, path).observe(duration)
    return response


@app.get("/health")
def health():
    return {"status": "healthy", "failure_rate": NOTIFY_FAILURE_RATE, "latency_ms": NOTIFY_LATENCY_MS}


@app.get("/metrics")
def metrics():
    from starlette.responses import Response
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)


@app.post("/notify")
def notify(body: dict = None):
    event = (body or {}).get("event", "unknown")
    order_id = (body or {}).get("order_id", "unknown")

    if NOTIFY_LATENCY_MS > 0:
        delay = NOTIFY_LATENCY_MS / 1000
        log.info(f"Injecting {NOTIFY_LATENCY_MS}ms latency for {order_id}")
        time.sleep(delay)

    if random.random() < NOTIFY_FAILURE_RATE:
        NOTIFY_TOTAL.labels("failed").inc()
        log.warning(f"Notification failed (injected) for {order_id}")
        raise HTTPException(500, "Notification processing failed")

    notification_ref = f"NOTIFY-{uuid.uuid4().hex[:8].upper()}"
    NOTIFY_TOTAL.labels("success").inc()
    log.info(f"Notification sent: {notification_ref} for {order_id} event={event}")
    return {"status": "sent", "notification_ref": notification_ref}
```

### `app/notifications/requirements.txt`

```
fastapi==0.136.0
uvicorn==0.44.0
prometheus-client==0.25.0
```

### `app/notifications/Dockerfile`

```dockerfile
FROM python:3.13-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .

EXPOSE 8083
RUN addgroup --system app && adduser --system --ingroup app app
USER app
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8083"]
```

---

## 2. `k8s/notifications.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notifications
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notifications
  template:
    metadata:
      labels:
        app: notifications
    spec:
      containers:
        - name: notifications
          image: quickticket-notifications:v1
          imagePullPolicy: Never
          ports:
            - containerPort: 8083
          livenessProbe:
            httpGet:
              path: /health
              port: 8083
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 8083
            periodSeconds: 5
            failureThreshold: 2
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
          env:
            - name: NOTIFY_FAILURE_RATE
              value: "0.0"
            - name: NOTIFY_LATENCY_MS
              value: "0"

---
apiVersion: v1
kind: Service
metadata:
  name: notifications
spec:
  selector:
    app: notifications
  ports:
    - port: 8083
      targetPort: 8083
  type: ClusterIP
```

---

## 3. `call_with_retry()` Implementation

```python
async def call_with_retry(func, target: str, max_retries: int = RETRY_MAX):
    attempt = 0
    base_delay = RETRY_BASE_DELAY_MS / 1000
    while True:
        try:
            result = await func()
            if attempt > 0:
                RETRY_TOTAL.labels(target, "succeeded_after_retry").inc()
            return result
        except Exception as e:
            is_retryable = (
                isinstance(e, (httpx.TimeoutException, httpx.ConnectError))
                or (isinstance(e, httpx.HTTPStatusError)
                    and (e.response.status_code >= 500
                         or e.response.status_code in (408, 429)))
            )
            if not is_retryable:
                RETRY_TOTAL.labels(target, "non_retryable").inc()
                raise
            if attempt >= max_retries:
                RETRY_TOTAL.labels(target, "exhausted").inc()
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, base_delay)
            RETRY_TOTAL.labels(target, "retried").inc()
            await asyncio.sleep(delay)
            attempt += 1
```

---

## 4. Test #1 — Fire-and-forget under notify failure

### Result

```
result: ok=30 fail=0
```

### `/pay` p99 latency during notify-failure injection

```
{"status":"success","data":{"result":[{"metric":{"le":"+Inf","path":"/reserve/{id}/pay"},"value":[...,"30"]},...]}}
```

p99 < 100ms — proves fire-and-forget is genuinely non-blocking.

---

## 5. Test #2 — Retries under transient payment failure

### Result

```
result: ok=29 fail=1
```

### `gateway_retry_total` from Prometheus

```
{"status":"success","data":{"result":[
  {"metric":{"target":"payments","result":"retried"},"value":[...,"9"]},
  {"metric":{"target":"payments","result":"succeeded_after_retry"},"value":[...,"8"]}
]}}
```

Both `result="retried"` and `result="succeeded_after_retry"` are non-zero — retries actually fired.

---

## 6. Real notify failure rate from `/metrics`

```
notifications_notify_total{result="success"} 21
notifications_notify_total{result="failed"} 9
```

Failure rate ≈ 30%, matching the injected `NOTIFY_FAILURE_RATE=0.3`.

---

## 7. Why should notifications be non-blocking (fire-and-forget)?

Notifications are a **non-critical, best-effort** side effect of the checkout flow. If the notification service is slow or down, it must not delay the user's HTTP response or cause the checkout to fail. Making it fire-and-forget (via `asyncio.create_task`) ensures the user gets their 200 response immediately while the notification attempt runs in the background. Failures are logged and swallowed, not propagated to the user.

---

## 8. Design Prompt: Why `cb.call(retry(...))` and not `retry(lambda: cb.call(...))`?

The correct composition is `cb.call(retry(_charge))` because:

- **`cb.call(retry(_charge))`**: The circuit breaker sees the *final outcome* after all retries are exhausted. If the first 2 attempts fail but the 3rd succeeds, the CB counts it as a success (no open). Only when all retries are exhausted does the CB see a failure.
- **`retry(lambda: cb.call(_charge))`**: If the circuit is OPEN, `cb.call()` raises `CircuitOpenError` immediately — then the retry loop catches that and retries, defeating the entire purpose of the fast-fail. The circuit breaker would never actually protect the downstream because retries would keep hammering it.

In short: **retries should happen inside the CB's view**, so the CB only counts unrecoverable failures, and the CB's fast-fall bypasses retries entirely.

---

## 9. Circuit Breaker Implementation

```python
class CircuitBreaker:
    OPEN = "OPEN"
    CLOSED = "CLOSED"
    HALF_OPEN = "HALF_OPEN"

    def __init__(self, threshold: int, cooldown_s: float, name: str = "cb"):
        self.threshold = threshold
        self.cooldown = cooldown_s
        self.name = name
        self.failures = 0
        self.state = self.CLOSED
        self.opened_at = 0.0

    def _transition(self, new_state: str):
        if self.state != new_state:
            log.warning(f"circuit[{self.name}] {self.state} -> {new_state}")
            CB_STATE_TRANSITIONS.labels(new_state).inc()
        self.state = new_state

    async def call(self, func):
        if self.state == self.OPEN:
            if time.time() - self.opened_at >= self.cooldown:
                self._transition(self.HALF_OPEN)
            else:
                raise CircuitOpenError(f"circuit[{self.name}] OPEN")
        try:
            result = await func()
            self.failures = 0
            self._transition(self.CLOSED)
            return result
        except Exception as e:
            self.failures += 1
            self.opened_at = time.time()
            if self.state == self.HALF_OPEN or self.failures >= self.threshold:
                self._transition(self.OPEN)
            raise
```

## 10. Rate Limiter Implementation

```python
class RateLimiter:
    def __init__(self, rps: int):
        self.rps = rps
        self.window_s = 1.0
        self.hits: dict[str, deque] = defaultdict(deque)

    def allow(self, key: str) -> bool:
        now = time.time()
        q = self.hits[key]
        cutoff = now - self.window_s
        while q and q[0] < cutoff:
            q.popleft()
        if len(q) >= self.rps:
            return False
        q.append(now)
        return True
```

---

## 11. CB Test under 100% payment failure

### 500s/503s breakdown

```
500s=12 503s=45
```

12 retry-exhausted (500), 45 fast-fail circuit-open (503). After the initial failures trip open, subsequent requests fast-fail with 503.

### After recovery (cooldown + `PAYMENT_FAILURE_RATE=0.0`)

```
[1] 200
[2] 200
[3] 200
...
```

All 200s after cooldown — circuit closed.

---

## 12. Rate Limiter burst test

### 200/429 split

```
200=47 429=53
```

### `Retry-After: 1` header

```
HTTP/1.1 429 Too Many Requests
retry-after: 1
```

### Prometheus counters

```
gateway_rate_limit_rejections_total{path="/events/{id}"} 53
gateway_circuit_breaker_transitions_total{to="OPEN"} 5
```

---

## Bonus — Bulkhead Isolation

### `Bulkhead.call` implementation

```python
class BulkheadFullError(Exception):
    pass

class Bulkhead:
    def __init__(self, name: str, max_concurrent: int, acquire_timeout_s: float):
        self.name = name
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.acquire_timeout_s = acquire_timeout_s

    async def call(self, func):
        try:
            await asyncio.wait_for(self.semaphore.acquire(), timeout=self.acquire_timeout_s)
        except asyncio.TimeoutError:
            BULKHEAD_REJECTIONS.labels(self.name).inc()
            raise BulkheadFullError(f"bulkhead[{self.name}] full")
        BULKHEAD_IN_FLIGHT.labels(self.name).inc()
        try:
            return await func()
        finally:
            BULKHEAD_IN_FLIGHT.labels(self.name).dec()
            self.semaphore.release()
```

### Wrapping line in `pay_reservation`

```python
pay_resp = await payments_bulkhead.call(
    lambda: payments_cb.call(lambda: call_with_retry(_charge, target="payments"))
)
```

### Concurrent test (with bulkhead)

```
EVENTS: ok=30 slow=0
```

Bulkhead protected `/events` from slow `/pay` calls.

### Concurrent test (without bulkhead)

```
EVENTS: ok=0 slow=30
```

Without bulkhead, the event loop is saturated by slow `/pay` calls.

### Prometheus metrics

```
gateway_bulkhead_rejections_total{target="payments"} 20
max_over_time(gateway_bulkhead_in_flight{target="payments"}[2m]) = 10
```

Rejections non-zero (slots filled), in_flight capped at MAX=10 — cap binds.

### Why does the bulkhead wrap the circuit breaker (outside), not the other way around?

Bulkhead must be **outside** CB because:
- **Bulkhead gates ENTRY** to the entire payments call path. Retries happen **inside** the bulkhead's semaphore slot, so 3 retries count as 1 occupant — keeping the bound meaningful.
- **CB inside the bulkhead**: when the circuit is OPEN, the CB fast-fails instantly (no real downstream call). If the CB were outside the bulkhead, the fast-fail would release the slot immediately — but that's actually fine for fast-fails. The real issue is the reverse: if CB wraps bulkhead, a slow payment call that's inside the bulkhead would hold its slot for the entire duration, and retries would each try to acquire their own slot, exceeding the intended concurrency limit.
- **Correct composition (bulkhead → CB → retry → call)**: one slot per user request, regardless of retries. The bulkhead limits concurrent *logical operations*, not concurrent HTTP calls.

### Bulkhead vs Rate Limiter — what do they protect against?

- **Rate limiter**: protects the **gateway itself** from excessive traffic (cluster-wide ceiling per endpoint). It rejects requests early, before any downstream call, based on a per-pod sliding window. The goal is to prevent a single client or burst from overwhelming the system.
- **Bulkhead**: protects **individual downstream dependencies** from each other. It ensures that one slow dependency (e.g., payments) cannot starve the shared event loop and degrade other dependencies (e.g., events). The goal is fault isolation — one "compartment" leaking doesn't sink the whole ship.

In short: rate limiter = "don't let too much in"; bulkhead = "don't let one slow thing clog everything".