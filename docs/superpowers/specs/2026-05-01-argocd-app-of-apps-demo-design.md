# ArgoCD App of Apps Demo Design

**Date:** 2026-05-01

## Goal

Build a minimal demo to understand the ArgoCD App of Apps pattern and observe the repo polling/reconciliation mechanism in action against a local kind cluster.

## Repo Layout

```
argocd-app-of-apps/
├── root-app.yaml              # Bootstrapped once via kubectl; watches apps/
├── apps/
│   ├── helm-guestbook.yaml    # Existing child Application
│   └── custom-app.yaml        # New child Application pointing at manifests/custom-app/
└── manifests/
    └── custom-app/
        ├── deployment.yaml    # 1-replica nginx Deployment
        └── service.yaml       # ClusterIP Service
```

## Components

### root-app.yaml
- Kind: `Application` (argoproj.io/v1alpha1)
- Bootstrapped once with `kubectl apply -f root-app.yaml`
- `source.repoURL: https://github.com/joshua-c-webb/argocd-app-of-apps.git`
- `source.path: apps` — watches the `apps/` directory in this repo
- `destination.namespace: argocd` — child Application objects are created in the argocd namespace
- `syncPolicy.automated` with `prune: true` and `selfHeal: true` — adding/removing files in `apps/` automatically creates/removes child apps

### apps/helm-guestbook.yaml
- Existing child Application (already present in repo and already deployed standalone in the cluster)
- When the root App syncs, it will reconcile this Application in place — no conflict, ArgoCD updates the existing object
- Deploys the ArgoCD example helm-guestbook app to `namespace: default`

### apps/custom-app.yaml
- New child Application
- `source.path: manifests/custom-app` — points at the manifests directory in this repo
- Deploys to `namespace: default`

### manifests/custom-app/
- `deployment.yaml`: 1-replica nginx Deployment named `custom-app`
- `service.yaml`: ClusterIP Service exposing port 80

## Polling Demo Flow

1. Bootstrap: `kubectl apply -f root-app.yaml` (done once)
2. ArgoCD detects `apps/` directory, creates both child Applications automatically
3. Both apps sync and deploy to the kind cluster
4. To trigger the polling demo: push a change to `manifests/custom-app/deployment.yaml` (e.g., change replica count)
5. ArgoCD repo-server polls the repo (default 3-minute interval) and marks the app `OutOfSync`
6. Automated sync kicks in, rolls out the change — visible in the ArgoCD UI

## Constraints

- Target cluster: local kind cluster
- ArgoCD already installed and running
- `helm-guestbook` Application already exists in the cluster — the root App will adopt it
- Repo is public at `https://github.com/joshua-c-webb/argocd-app-of-apps.git` — no credentials needed
