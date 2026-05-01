# ArgoCD App of Apps Demo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a minimal App of Apps demo with a root Application managing two child apps, then observe ArgoCD's repo polling mechanism update a live deployment.

**Architecture:** A single root Application watches the `apps/` directory in this repo. Any ArgoCD `Application` manifest placed in `apps/` is automatically created/updated/deleted. The custom app's raw Kubernetes manifests live in `manifests/custom-app/` in the same repo. Everything is automated — once the root App is bootstrapped with one `kubectl apply`, ArgoCD manages the rest.

**Tech Stack:** ArgoCD, Kubernetes (kind), kubectl, argocd CLI

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `manifests/custom-app/deployment.yaml` | nginx Deployment (1 replica) |
| Create | `manifests/custom-app/service.yaml` | ClusterIP Service on port 80 |
| Create | `apps/custom-app.yaml` | Child Application pointing at `manifests/custom-app/` |
| Modify | `root-app.yaml` | Root Application watching `apps/` directory |

---

### Task 1: Create the custom-app Kubernetes manifests

**Files:**
- Create: `manifests/custom-app/deployment.yaml`
- Create: `manifests/custom-app/service.yaml`

- [ ] **Step 1: Create the manifests directory**

```bash
mkdir -p manifests/custom-app
```

- [ ] **Step 2: Write `manifests/custom-app/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-app
  template:
    metadata:
      labels:
        app: custom-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

- [ ] **Step 3: Write `manifests/custom-app/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: custom-app
  namespace: default
spec:
  selector:
    app: custom-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

- [ ] **Step 4: Validate the YAML is well-formed**

```bash
kubectl apply --dry-run=client -f manifests/custom-app/
```

Expected output:
```
deployment.apps/custom-app created (dry run)
service/custom-app created (dry run)
```

- [ ] **Step 5: Commit**

```bash
git add manifests/custom-app/
git commit -m "feat: add custom-app kubernetes manifests"
```

---

### Task 2: Create the custom-app ArgoCD Application manifest

**Files:**
- Create: `apps/custom-app.yaml`

- [ ] **Step 1: Write `apps/custom-app.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: custom-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/joshua-c-webb/argocd-app-of-apps.git
    targetRevision: HEAD
    path: manifests/custom-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- [ ] **Step 2: Validate the YAML is well-formed**

```bash
kubectl apply --dry-run=client -f apps/custom-app.yaml
```

Expected output:
```
application.argoproj.io/custom-app created (dry run)
```

- [ ] **Step 3: Commit**

```bash
git add apps/custom-app.yaml
git commit -m "feat: add custom-app ArgoCD Application manifest"
```

---

### Task 3: Write the root Application manifest

**Files:**
- Modify: `root-app.yaml`

The root App must deploy to `namespace: argocd` (not `default`) because `Application` objects live in the argocd namespace.

- [ ] **Step 1: Write `root-app.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/joshua-c-webb/argocd-app-of-apps.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- [ ] **Step 2: Validate the YAML is well-formed**

```bash
kubectl apply --dry-run=client -f root-app.yaml
```

Expected output:
```
application.argoproj.io/root-app created (dry run)
```

- [ ] **Step 3: Commit and push everything to GitHub**

```bash
git add root-app.yaml
git commit -m "feat: add root ArgoCD Application"
git push origin main
```

GitHub must have these commits before bootstrapping, because ArgoCD will immediately try to pull from the repo.

---

### Task 4: Bootstrap the root App in ArgoCD

This is the only manual step. After this, ArgoCD manages everything.

- [ ] **Step 1: Apply the root App to the cluster**

```bash
kubectl apply -f root-app.yaml
```

Expected output:
```
application.argoproj.io/root-app created
```

- [ ] **Step 2: Watch ArgoCD create the child Applications**

```bash
kubectl get applications -n argocd -w
```

Within ~30 seconds you should see `custom-app` and `helm-guestbook` appear with `Syncing` then `Synced` status. Press Ctrl+C when done.

- [ ] **Step 3: Verify all three apps are healthy**

```bash
kubectl get applications -n argocd
```

Expected output (all three apps `Synced` and `Healthy`):
```
NAME             SYNC STATUS   HEALTH STATUS
custom-app       Synced        Healthy
helm-guestbook   Synced        Healthy
root-app         Synced        Healthy
```

- [ ] **Step 4: Verify the custom-app pod is running**

```bash
kubectl get pods -n default -l app=custom-app
```

Expected output:
```
NAME                          READY   STATUS    RESTARTS   AGE
custom-app-<hash>             1/1     Running   0          <age>
```

---

### Task 5: Trigger the polling demo

This demonstrates ArgoCD detecting a git change and reconciling the cluster state.

- [ ] **Step 1: Edit the replica count in `manifests/custom-app/deployment.yaml`**

Change `replicas: 1` to `replicas: 2`:

```yaml
spec:
  replicas: 2
```

- [ ] **Step 2: Commit and push**

```bash
git add manifests/custom-app/deployment.yaml
git commit -m "demo: scale custom-app to 2 replicas"
git push origin main
```

- [ ] **Step 3: Watch ArgoCD detect the drift**

Option A — wait for the automatic poll (default 3 minutes), then run:
```bash
kubectl get applications -n argocd custom-app -w
```
Watch `SYNC STATUS` flip from `Synced` → `OutOfSync` → `Synced`.

Option B — trigger an immediate sync without waiting for the poll:
```bash
argocd app sync custom-app
```

- [ ] **Step 4: Confirm two pods are running**

```bash
kubectl get pods -n default -l app=custom-app
```

Expected output:
```
NAME                          READY   STATUS    RESTARTS   AGE
custom-app-<hash>             1/1     Running   0          <age>
custom-app-<hash>             1/1     Running   0          <age>
```

The cluster now matches the desired state in git — the polling loop is complete.
