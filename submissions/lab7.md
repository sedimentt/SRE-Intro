# Lab 7 — Progressive Delivery: Canary Deployments

## Task 1 — Manual Canary Deployment

### 1. Output of kubectl argo rollouts version

```text
❯ kubectl argo rollouts version
kubectl-argo-rollouts: v1.9.0+838d4e7
  BuildDate: 2026-03-20T21:08:11Z
  GitCommit: 838d4e792be666ec11bd0c80331e0c5511b5010e
  GitTreeState: clean
  GoVersion: go1.24.13
  Compiler: gc
  Platform: linux/amd64
```

### 2. Output of kubectl argo rollouts get rollout gateway showing Paused at 20% (during canary)

```text
Name:            gateway
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/5
  SetWeight:     20
  ActualWeight:  20
Images:          ghcr.io/sedimentt/quickticket-gateway:8f5e23582570969fee3b4214172f59c14a9c334e (canary, stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                 KIND        STATUS     AGE   INFO
⟳ gateway                            Rollout     ॥ Paused   5m4s
├──# revision:2
│  └──⧉ gateway-8647df4cdc           ReplicaSet  ✔ Healthy  40s   canary
│     └──□ gateway-8647df4cdc-qrmct  Pod         ✔ Running  39s   ready:1/1
└──# revision:1
   └──⧉ gateway-55f78458cf           ReplicaSet  ✔ Healthy  5m4s  stable
      ├──□ gateway-55f78458cf-5hjmb  Pod         ✔ Running  5m3s  ready:1/1
      ├──□ gateway-55f78458cf-crfl9  Pod         ✔ Running  5m3s  ready:1/1
      ├──□ gateway-55f78458cf-qqkrb  Pod         ✔ Running  5m3s  ready:1/1
      └──□ gateway-55f78458cf-vx9jj  Pod         ✔ Running  5m3s  ready:1/1
```

### 3. Output after promote — showing progression to 100%

```text
Name:            gateway
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          5/5
  SetWeight:     100
  ActualWeight:  100
Images:          ghcr.io/sedimentt/quickticket-gateway:8f5e23582570969fee3b4214172f59c14a9c334e (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                 KIND        STATUS        AGE    INFO
⟳ gateway                            Rollout     ✔ Healthy     10m
├──# revision:2
│  └──⧉ gateway-8647df4cdc           ReplicaSet  ✔ Healthy     5m36s  stable
│     ├──□ gateway-8647df4cdc-qrmct  Pod         ✔ Running     5m35s  ready:1/1
│     ├──□ gateway-8647df4cdc-dwj9w  Pod         ✔ Running     54s    ready:1/1
│     ├──□ gateway-8647df4cdc-vlmc5  Pod         ✔ Running     54s    ready:1/1
│     ├──□ gateway-8647df4cdc-dplr7  Pod         ✔ Running     14s    ready:1/1
│     └──□ gateway-8647df4cdc-ptw4s  Pod         ✔ Running     14s    ready:1/1
└──# revision:1
   └──⧉ gateway-55f78458cf           ReplicaSet  • ScaledDown  10m
```

### 4. Output after abort — showing instant rollback

```text
❯ kubectl argo rollouts get rollout gateway
Name:            gateway
Namespace:       default
Status:          ✖ Degraded
Message:         RolloutAborted: Rollout aborted update to revision 3
Strategy:        Canary
  Step:          0/5
  SetWeight:     0
  ActualWeight:  0
Images:          ghcr.io/sedimentt/quickticket-gateway:8f5e23582570969fee3b4214172f59c14a9c334e (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       0
  Ready:         5
  Available:     5

NAME                                 KIND        STATUS        AGE    INFO
⟳ gateway                            Rollout     ✖ Degraded    17m
├──# revision:3
│  └──⧉ gateway-6459858df8           ReplicaSet  • ScaledDown  67s    canary
├──# revision:2
│  └──⧉ gateway-8647df4cdc           ReplicaSet  ✔ Healthy     12m    stable
│     ├──□ gateway-8647df4cdc-qrmct  Pod         ✔ Running     12m    ready:1/1
│     ├──□ gateway-8647df4cdc-dwj9w  Pod         ✔ Running     8m6s   ready:1/1
│     ├──□ gateway-8647df4cdc-dplr7  Pod         ✔ Running     7m26s  ready:1/1
│     ├──□ gateway-8647df4cdc-ptw4s  Pod         ✔ Running     7m26s  ready:1/1
│     └──□ gateway-8647df4cdc-fnzvr  Pod         ✔ Running     10s    ready:1/1
└──# revision:1
   └──⧉ gateway-55f78458cf           ReplicaSet  • ScaledDown  17m
```

### 5. Answer: "How long from abort to all traffic serving the stable version? Compare with git revert rollback from Lab 5."

After `abort`, all traffic returned to the stable version almost immediately (within a few seconds). In comparison, the `git revert` rollback in Lab 5 took approximately 1–2 minutes because it required creating a new commit, pushing it to the repository, waiting for ArgoCD to detect the change, and redeploying the application. Therefore, using `abort` provides a much faster rollback than `git revert`.

## Task 2 — Multi-Step Canary with Observation

### 1. Your multi-step canary strategy YAML
```text
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: gateway
  labels:
    version: "v2"
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 60s }
        - setWeight: 40
        - pause: { duration: 60s }
        - setWeight: 60
        - pause: { duration: 60s }
        - setWeight: 80
        - pause: { duration: 30s }
        - setWeight: 100
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      imagePullSecrets:
        - name: ghcr-secret
      containers:
        - name: gateway
          image: ghcr.io/sedimentt/quickticket-gateway:8f5e23582570969fee3b4214172f59c14a9c334e
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health
              port: 8080
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
            - name: APP_VERSION
              value: "v3-bad"
            - name: EVENTS_URL
              value: "http://events:8081"
            - name: PAYMENTS_URL
              value: "http://payments:8082"
            - name: GATEWAY_TIMEOUT_MS
              value: "5000"
---
apiVersion: v1
kind: Service
metadata:
  name: gateway
spec:
  selector:
    app: gateway
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP

```
### 2. Output of kubectl argo rollouts get rollout gateway --watch showing at least 3 steps
step 1
```text
Name:            gateway
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          2/9
  SetWeight:     20
  ActualWeight:  20
Images:          ghcr.io/sedimentt/quickticket-gateway:8f5e23582570969fee3b4214172f59c14a9c334e (canary, stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                 KIND        STATUS        AGE    INFO
⟳ gateway                            Rollout     ॥ Paused      32m
├──# revision:4
│  └──⧉ gateway-6db75f47bf           ReplicaSet  ✔ Healthy     1m8s   canary
│     └──□ gateway-6db75f47bf-hpm56  Pod         ✔ Running     1m6s   ready:1/1
├──# revision:3
│  └──⧉ gateway-6459858df8           ReplicaSet  • ScaledDown  16m
├──# revision:2
│  └──⧉ gateway-8647df4cdc           ReplicaSet  ✔ Healthy     28m    stable
│     ├──□ gateway-8647df4cdc-qrmct  Pod         ✔ Running     28m    ready:1/1
│     ├──□ gateway-8647df4cdc-dplr7  Pod         ✔ Running     22m    ready:1/1
│     ├──□ gateway-8647df4cdc-ptw4s  Pod         ✔ Running     22m    ready:1/1
│     └──□ gateway-8647df4cdc-7kf9n  Pod         ✔ Running     22m    ready:1/1
└──# revision:1
   └──⧉ gateway-55f78458cf           ReplicaSet  • ScaledDown  32m
```
step 3 
```text
Name:            gateway
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          3/9
  SetWeight:     40
  ActualWeight:  40
Images:          ghcr.io/sedimentt/quickticket-gateway:8f5e23582570969fee3b4214172f59c14a9c334e (canary, stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       2
  Ready:         5
  Available:     5

NAME                                 KIND        STATUS        AGE    INFO
⟳ gateway                            Rollout     ॥ Paused      33m
├──# revision:4
│  └──⧉ gateway-6db75f47bf           ReplicaSet  ✔ Healthy     2m10s  canary
│     ├──□ gateway-6db75f47bf-hpm56  Pod         ✔ Running     2m8s   ready:1/1
│     └──□ gateway-6db75f47bf-wjbbw  Pod         ✔ Running     56s    ready:1/1
├──# revision:3
│  └──⧉ gateway-6459858df8           ReplicaSet  • ScaledDown  17m
├──# revision:2
│  └──⧉ gateway-8647df4cdc           ReplicaSet  ✔ Healthy     29m    stable
│     ├──□ gateway-8647df4cdc-qrmct  Pod         ✔ Running     29m    ready:1/1
│     ├──□ gateway-8647df4cdc-dplr7  Pod         ✔ Running     23m    ready:1/1
│     └──□ gateway-8647df4cdc-ptw4s  Pod         ✔ Running     23m    ready:1/1
└──# revision:1
   └──⧉ gateway-55f78458cf           ReplicaSet  • ScaledDown  33m
```
step 5
```text
Name:            gateway
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          5/9
  SetWeight:     60
  ActualWeight:  60
Images:          ghcr.io/sedimentt/quickticket-gateway:8f5e23582570969fee3b4214172f59c14a9c334e (canary, stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       3
  Ready:         5
  Available:     5

NAME                                 KIND        STATUS        AGE    INFO
⟳ gateway                            Rollout     ॥ Paused      34m
├──# revision:4
│  └──⧉ gateway-6db75f47bf           ReplicaSet  ✔ Healthy     2m34s  canary
│     ├──□ gateway-6db75f47bf-hpm56  Pod         ✔ Running     2m32s  ready:1/1
│     ├──□ gateway-6db75f47bf-wjbbw  Pod         ✔ Running     80s    ready:1/1
│     └──□ gateway-6db75f47bf-2kkfr  Pod         ✔ Running     10s    ready:1/1
├──# revision:3
│  └──⧉ gateway-6459858df8           ReplicaSet  • ScaledDown  18m
├──# revision:2
│  └──⧉ gateway-8647df4cdc           ReplicaSet  ✔ Healthy     29m    stable
│     ├──□ gateway-8647df4cdc-qrmct  Pod         ✔ Running     29m    ready:1/1
│     └──□ gateway-8647df4cdc-dplr7  Pod         ✔ Running     24m    ready:1/1
└──# revision:1
   └──⧉ gateway-55f78458cf           ReplicaSet  • ScaledDown  34m
```
### 3. Dashboard observation during the rollout
During the rollout, the canary deployment progressed through multiple stages (20%, 40%, and 60%). At each stage, the number of updated replicas increased gradually while the stable replicas were reduced. The rollout paused at each configured step, allowing verification before continuing. Throughout the deployment, all five replicas remained available, and no service interruption was observed.
### 4 .Answer: "At what canary percentage would you want an automated abort? Why?"
I would configure an automatic abort at 40%. By that point, the canary receives enough production traffic to detect most issues while 60% of the traffic is still served by the stable version. This provides a good balance between early problem detection and minimizing the impact of a faulty release.

