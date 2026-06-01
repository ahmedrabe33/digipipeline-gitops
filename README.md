# AuraWeb — Kubernetes & GitOps Repository

> Kubernetes manifests, Kustomize overlays, and ArgoCD Applications for the AuraWeb platform.  
> The application source code and CI pipeline live at [Khairy-808/auraweb-app](https://github.com/Khairy-808/auraweb-app).

---

## How it works

This repo is the **single source of truth** for what runs in the cluster.  
ArgoCD watches this repo. When anything changes here, ArgoCD syncs the cluster automatically.  
**Nobody runs `kubectl apply` manually.** Everything goes through Git.

```
Developer pushes code → auraweb-app
        │
        ▼
Jenkins builds Docker images
        │
        ▼
Jenkins updates image tags in THIS repo
(kustomize edit set image frontend=khairy808/frontend:abc1234)
        │
        ▼
ArgoCD detects the commit
        │
        ▼
ArgoCD syncs the EKS cluster
        │
        ▼
New version is live ✓
```

---

## Repository Structure

```
auraweb-k8s/
├── k8s/
│   ├── base/                      # Shared manifests for all environments
│   │   ├── frontend-deployment.yaml
│   │   ├── admin-deployment.yaml
│   │   ├── gateway-deployment.yaml
│   │   ├── catalog-deployment.yaml
│   │   ├── inventory-deployment.yaml
│   │   ├── shopping-deployment.yaml
│   │   ├── order-payment-deployment.yaml
│   │   ├── fulfillment-deployment.yaml
│   │   ├── user-auth-deployment.yaml
│   │   ├── platform-deployment.yaml
│   │   ├── postgres-statefulset.yaml
│   │   ├── redis-deployment.yaml
│   │   ├── rabbitmq-deployment.yaml
│   │   └── kustomization.yaml
│   │
│   └── overlays/
│       ├── development/           # Dev/staging environment
│       │   ├── kustomization.yaml # Adds image tags, LOG_LEVEL=debug
│       │   └── ingress.yaml
│       └── production/            # Production environment
│           └── kustomization.yaml # Adds image tags, 2 replicas per service
│
└── argocd/
    └── applications.yaml          # ArgoCD Application CRDs (apply once)
```

---

## Environments

| Environment | Namespace | Branch watched | Replicas |
|---|---|---|---|
| Staging / Dev | `auraweb-dev` | `main` → `overlays/development` | 1 |
| Production | `auraweb-prod` | `main` → `overlays/production` | 2 |

---

## How Kustomize overlays work

The `base/` folder holds the core manifests shared by every environment.  
Each overlay **extends** the base and only changes what's different:

**Development overlay adds:**
- Image tags (updated by Jenkins on every build)
- `LOG_LEVEL=debug` environment variable
- Ingress for dev domain

**Production overlay adds:**
- Image tags (same images, promoted from the same build)
- `LOG_LEVEL=info`
- 2 replicas per deployment (for high availability)

---

## One-time Setup

**Prerequisites:** An EKS cluster with ArgoCD installed.

```bash
# 1. Install ArgoCD (if not already installed)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Register both Applications with ArgoCD
kubectl apply -f argocd/applications.yaml

# Done — ArgoCD takes over from here.
# It will sync both staging and production automatically on every commit.
```

---

## ArgoCD Sync Behaviour

| Setting | Value | What it means |
|---|---|---|
| `automated` | enabled | ArgoCD syncs on every new commit, no manual trigger needed |
| `prune` | true | Resources deleted from Git are deleted from the cluster |
| `selfHeal` | true | Manual `kubectl` changes are reverted to match Git |
| `ignoreDifferences` | replicas | HPA can scale pods freely without ArgoCD reverting it |

---

## Making a Change

**To update a manifest** (e.g. change a resource limit):
1. Edit the file in `k8s/base/` or the relevant overlay
2. Commit and push to `main`
3. ArgoCD detects the commit and applies the change — nothing else needed

**Image tags are updated automatically** by Jenkins in `auraweb-app` — you never edit them by hand.

---

## Related

- **Application source code & CI:** [Khairy-808/auraweb-app](https://github.com/Khairy-808/auraweb-app)  
  React frontends, Node.js microservices, Dockerfiles, and the Jenkins pipeline that feeds image tags into this repo.
