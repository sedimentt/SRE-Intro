# Lab 5 — CI/CD and GitOps with ArgoCD

## Task 1 — GitOps Deployment with ArgoCD

### 1. Link to GitHub Actions run (green check)

https://github.com/sedimentt/SRE-Intro/actions/runs/28018588892

---

### 2. Output of `gh api user/packages?package_type=container` showing pushed images

```text
quickticket-gateway
quickticket-events
quickticket-payments
```

---

### 3. Output of `argocd app get quickticket` showing Synced + Healthy

```text
Name:               argocd/quickticket
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8443/applications/quickticket
Source:
- Repo:             https://github.com/sedimentt/SRE-Intro.git
  Target:
  Path:             k8s

Sync Policy:        Automated
Sync Status:        Synced to (61480eb)
Health Status:      Healthy
```

---

### 4. Output proving a Git change was synced

```text
$ kubectl get deployment gateway -o jsonpath='{.metadata.labels.version}'

v2
```

---

### 5. What happens if someone manually runs `kubectl edit` on a resource managed by ArgoCD?

Manual changes made with `kubectl edit` are considered configuration drift because they are not reflected in Git.

ArgoCD detects the difference between the cluster and the repository and marks the application as `OutOfSync`.

With auto-sync enabled, ArgoCD automatically reverts the changes and restores the Git version. Without auto-sync, a manual synchronization is required.

---

# Task 2 — Rollback via GitOps

### 1. `argocd app get` showing Degraded after bad deploy

```text
Health Status: Degraded
```

---

### 2. `kubectl get pods` showing ImagePullBackOff

```text
NAME                        READY   STATUS             RESTARTS        AGE
events-78696fcf65-89gbk     1/1     Running            4 (4h16m ago)   16h
events-bd5dbf567-p9897      0/1     CrashLoopBackOff   9 (41s ago)     16m
gateway-746987b6cf-54n7z    0/1     ErrImagePull       0               11s
gateway-7b64648c9-pbrcm     1/1     Running            0               16m
payments-7dd48fbf96-h4rmd   1/1     Running            0               16m
postgres-76cd478b6b-z4jnv   1/1     Running            0               16h
redis-65bb44458c-vc4f8      1/1     Running            0               16h
```

---

### 3. `git log --oneline -3` showing deploy and revert commits

```text
0c10af3 Revert "feat: deploy new gateway version"
79a6233 feat: deploy new gateway version
b7ebb06 feat: add version label to gateway again
```

---

### 4. `argocd app get` showing Healthy after revert

```text
Name:               argocd/quickticket
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default

Sync Status:        Synced to (61480eb)
Health Status:      Healthy
```

---

### 5. How long from `git revert` + push to pods being healthy again?

Recovery took approximately **1–2 minutes**.

After pushing the revert commit, ArgoCD detected the change, synchronized the application, and Kubernetes rolled out healthy pods automatically.

---

# Bonus Task — Automated Image Tag Update

### 1. Updated workflow file showing auto-tag update

```yaml
jobs:
  build:
    if: "!startsWith(github.event.head_commit.message, 'ci:')"

    permissions:
      contents: write
      packages: write

    ...

    - name: Update image tags in manifests
      run: |
        SHA=${{ github.sha }}

        sed -i "s|image: ghcr.io/.*/quickticket-gateway:.*|image: ghcr.io/sedimentt/quickticket-gateway:${SHA}|" k8s/gateway.yaml

        sed -i "s|image: ghcr.io/.*/quickticket-events:.*|image: ghcr.io/sedimentt/quickticket-events:${SHA}|" k8s/events.yaml

        sed -i "s|image: ghcr.io/.*/quickticket-payments:.*|image: ghcr.io/sedimentt/quickticket-payments:${SHA}|" k8s/payments.yaml

    - name: Commit and push manifest update
      run: |
        git config user.name "github-actions"
        git config user.email "github-actions@github.com"

        git add k8s/

        git diff --cached --quiet || git commit -m "ci: update image tags to ${{ github.sha }}"

        git push
```

---

### 2. Git log showing code commit → CI tag-update commit

```text
640f5d1 ci: update image tags to 1f4d89d3c2765c68b9fe4897845dbb3baa2451b3
1f4d89d trigger auto tag update
61480eb fix: use events image for events deployment
0c10af3 Revert "feat: deploy new gateway version"
79a6233 feat: deploy new gateway version
```

---

### 3. ArgoCD syncing the auto-updated tag without manual intervention

```text
Name:               argocd/quickticket
Project:            default
Server:             https://kubernetes.default.svc

Sync Status:        Synced to (1f4d89d)
Health Status:      Healthy
```
