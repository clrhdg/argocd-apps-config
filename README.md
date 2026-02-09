# argocd-apps-config

GitOps repository that defines **which applications and infrastructure** Argo CD deploys. Application manifests (Kubernetes YAML, Helm, Kustomize) live in other repos; this repo holds the Argo CD `Application` definitions and bootstrap.

---

## What it does

- **GitOps with Argo CD** — All deployments are driven from Git. You add, remove, or change apps by editing YAML in this repo and pushing; Argo CD syncs the cluster to match.
- **App-of-apps** — A single **root Application** (`bootstrap/root-app.yaml`) syncs the `apps/` and `infra/` directories. Every Argo CD `Application` in those dirs is created/updated by the root, so you don’t run `argocd app create` for each app.
- **Clear split** — This repo = *what to deploy* (Application CRs). Other repos = *what gets deployed* (manifests, charts). Infra that lives in this repo (e.g. Argo CD + LB) is under `infra/resources/` and deployed by the `argocd-lb` Application.

---

## Architecture overview

| Layer | What happens |
|-------|----------------|
| **1. Bootstrap** | You create the root Application once (from `bootstrap/root-app.yaml`). It points at this repo and syncs two paths: `apps` and `infra` (with `infra/resources/*` excluded). |
| **2. Application definitions** | The root app syncs **Application** manifests from `apps/` and `infra/`. Those are plain Argo CD Application CRs; the root doesn’t deploy app workloads from those dirs. |
| **3. Deployments** | Each Application syncs from its own `source` (another Git repo or `infra/resources` in this repo). That’s where the actual Kubernetes resources come from. |

So: **Root Application → syncs Application CRs from this repo → each Application syncs its source (this repo or external).**

---

## Architecture diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BOOTSTRAP PHASE                                 │
│                                                                              │
│                            ┌──────────────┐                                 │
│                            │   Root App   │                                 │
│                            │ (root-apps)  │                                 │
│                            └──────┬───────┘                                 │
│                                   │                                          │
└───────────────────────────────────┼──────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
        ┌──────────────────┐          ┌──────────────────────┐
        │  apps/ directory │          │  infra/ directory    │
        └──────────────────┘          └──────────────────────┘
                    │                               │
        ┌───────────┴───────────┐       ┌──────────┴──────────┬──────────────┐
        │                       │       │                     │              │
        ▼                       ▼       ▼                     ▼              ▼
┌──────────────┐      ┌──────────────┐ ┌──────────────┐ ┌─────────┐ ┌──────────┐
│  guestbook   │      │  blue-green  │ │  argocd-lb   │ │   ESO   │ │  Vault   │
│ Application  │      │ Application  │ │ Application  │ │   App   │ │   App    │
└──────┬───────┘      └──────┬───────┘ └──────┬───────┘ └────┬────┘ └────┬─────┘
       │                     │                │               │           │
       │                     │                │               │           │
       ▼                     ▼                ▼               ▼           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION SOURCES                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────┐  ┌───────────┐ │
│  │ argocd-example │  │ argocd-example │  │infra/resources│  │  External │ │
│  │  apps/repo     │  │  apps/repo     │  │ (this repo)   │  │Helm Charts│ │
│  │  /guestbook    │  │  /blue-green   │  │               │  │           │ │
│  └────────────────┘  └────────────────┘  └───────────────┘  └───────────┘ │
│                                                                              │
│  Deploys actual workloads (Deployments, Services, etc.)                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Flow:**
1. Root Application syncs `apps/` and `infra/` directories from this repo
2. Creates child Applications (guestbook, blue-green, argocd-lb, ESO, Vault, etc.)
3. Each Application syncs from its configured source:
   - `guestbook` & `blue-green` → external argocd-example-apps repo
   - `argocd-lb` → `infra/resources/` in this repo (Argo CD install + LB service)
   - `ESO` & `Vault` → external Helm charts

---

## Directory structure

```
argocd-apps-config/
├── README.md
├── bootstrap/
│   └── root-app.yaml       # Root Application; syncs apps/ and infra/
├── apps/
│   ├── guestbook-app.yaml  # Application → argocd-example-apps/guestbook
│   └── blue-green-app.yaml # Application → argocd-example-apps/blue-green
└── infra/
    ├── argocd-app.yaml     # Application → infra/resources (Argo CD + LB)
    ├── external-mdns-app.yaml
    └── resources/
        ├── kustomization.yaml
        └── argocd-lb-service.yaml
```

| Directory | Purpose |
|-----------|---------|
| `bootstrap/` | Single entry point. Apply `root-app.yaml` once; it syncs `apps/` and `infra/` from this repo. |
| `apps/` | One YAML file per **application** Application (guestbook, blue-green, etc.). Each references an external repo/path. |
| `infra/` | Application manifests for **infrastructure** (argocd-lb, external-mdns). `argocd-app.yaml` deploys from `infra/resources/`. |
| `infra/resources/` | Kustomize: upstream Argo CD install + `argocd-lb-service.yaml`. Deployed by the `argocd-lb` Application. |

---

## How to add new apps

1. **Create an Application manifest** in `apps/` (or `infra/` for infra). Example:

   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/org/repo.git
       targetRevision: HEAD
       path: path/in/repo
     destination:
       server: https://kubernetes.default.svc
       namespace: my-app-ns
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

2. **Commit and push** to the branch the root app tracks. The root app will sync and create the new Application; Argo CD will then sync that Application from its `source`.

3. **No change to root** — The root Application already syncs everything under `apps/` and `infra/` (except `infra/resources/`), so new files are picked up automatically.

To deploy from **this repo** (like argocd-lb): set `source.repoURL` to this repo and `source.path` to the directory (e.g. `infra/resources`).

---

## Bootstrap (first-time install)

Use this section when standing up a new cluster or using this repo for the first time.

### Prerequisites

- **Kubernetes cluster** with `kubectl` configured and cluster-admin (or sufficient RBAC) to create namespaces and resources in `argocd` and application namespaces.
- **This repo reachable** from the cluster (e.g. public GitHub, or a private repo with credentials configured in Argo CD).
- **Argo CD installed** in the cluster before you create the root Application (see below).

### Step 1: Install Argo CD

The root Application and all child Applications are managed by Argo CD, so Argo CD must be installed first.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

After you create the root Application (Step 3), the `argocd-lb` Application will sync `infra/resources/` in this repo, which uses Kustomize to reference the same upstream install plus `argocd-lb-service.yaml`. For future Argo CD upgrades, update the manifest URL in `infra/resources/kustomization.yaml` and sync the `argocd-lb` app.

### Step 2: Create the root Application

Create the root Application so Argo CD starts syncing `apps/` and `infra/` from this repo.

```bash
kubectl apply -f bootstrap/root-app.yaml
```

The root Application will appear in the Argo CD UI and begin syncing. It has **multiple sources**: `apps` and `infra` (with `infra/resources/*` excluded), both from this repo.

### Step 3: Verify

1. **Root app:** In the UI or CLI, confirm the root Application (`root-apps`) is **Synced** and **Healthy**. If it’s not, check repo URL, branch, and credentials.
2. **Child apps:** After the root syncs, you should see Applications for `guestbook`, `blue-green`, `argocd-lb`, and `external-mdns`. Each will sync from its own source; some may need repo access if they use private repos.
3. **Optional:** Enable auto-sync on the root if it’s not already (the repo’s `root-app.yaml` has `syncPolicy.automated` with `prune` and `selfHeal`).

### Step 4: What happens next

- The root Application keeps syncing `apps/` and `infra/` on its refresh interval. Any new or changed Application YAML in those directories will be applied.
- Each child Application syncs its `source` (external repo or `infra/resources`). To add apps, add new Application manifests under `apps/` or `infra/` and push; no change to the root Application is required.
