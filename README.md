# argocd-apps-config

GitOps repository that defines which applications and infrastructure run on your Kubernetes cluster, managed by Argo CD.

---

## Overview

This repo is the **single source of truth** for a cluster’s workloads and for Argo CD itself. It provides:

- **Bootstrap** – A root Application (app-of-apps) that registers all other Applications with Argo CD.
- **Infrastructure** – Argo CD’s own install (with Kustomize patches and a LoadBalancer service) and the external-mdns controller.
- **Applications** – Example apps (guestbook, blue-green) and any other apps you add as Application manifests.

When the root Application is applied, Argo CD syncs from this repo and creates/updates the Application resources. Those Applications then sync their sources (other Git repos or this repo’s `infra/resources`) so the cluster state matches the repo.

---

## Architecture

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    Git (this repo)                      │
                    │  bootstrap/   infra/   apps/   infra/resources/           │
                    └─────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                    ┌─────────────────────────────────────────────────────────┐
                    │              Argo CD (already installed)                 │
                    │         Apply bootstrap/root-app.yaml once               │
                    └─────────────────────────────────────────────────────────┘
                                              │
                    ┌─────────────────────────┴─────────────────────────┐
                    ▼                                                   ▼
        ┌───────────────────────┐                         ┌───────────────────────┐
        │   Root Application    │                         │   Root Application    │
        │   sources: apps/      │                         │   sources: infra/     │
        │   + infra/ (excl.     │                         │   (excl. resources)   │
        │     resources)        │                         │                       │
        └───────────┬───────────┘                         └───────────┬───────────┘
                    │                                                 │
        ┌───────────┴───────────┐                         ┌──────────┴──────────┐
        ▼                       ▼                         ▼                     ▼
  ┌───────────┐         ┌─────────────┐         ┌─────────────┐       ┌─────────────────┐
  │ guestbook │         │ blue-green   │         │ argocd-lb    │       │ external-mdns   │
  │ Application│         │ Application │         │ Application │       │ Application     │
  └─────┬─────┘         └──────┬──────┘         └──────┬──────┘       └────────┬────────┘
        │                      │                       │                       │
        ▼                      ▼                       ▼                       ▼
  argocd-example-apps   argocd-example-apps    infra/resources/         external-mdns
  (guestbook path)      (blue-green path)      (Argo CD install +        (blake/external-mdns
                                                patches + LB service)      manifests/rbac)
```

**High-level flow**

1. You install Argo CD once (e.g. from `infra/resources` with `kubectl apply -k`).
2. You apply `bootstrap/root-app.yaml` so Argo CD has the root Application.
3. The root Application syncs `apps/` and `infra/` (excluding `infra/resources/`), creating Applications for guestbook, blue-green, argocd-lb, and external-mdns.
4. The **argocd-lb** Application syncs `infra/resources/`, which installs/updates Argo CD (Kustomize base + patches) and the Argo CD LoadBalancer service.
5. Other Applications sync their respective sources; the cluster converges to the state defined in Git.

---

## Prerequisites

- **Kubernetes cluster** – e.g. [MicroK8s](https://microk8s.io/) (see below).
- **kubectl** configured to talk to the cluster.
- **Argo CD** installed at least once so the root Application can be applied (see “How to replicate this setup”).

### MicroK8s (example)

```bash
# Install MicroK8s (Ubuntu / compatible Linux)
sudo snap install microk8s --classic
sudo usermod -a -G microk8s $USER
# Log out and back in (or newgrp microk8s)

# Enable add-ons
microk8s enable dns storage ingress

# Use kubectl via microk8s
microk8s kubectl get nodes
# Or alias: alias kubectl='microk8s kubectl'
```

### Argo CD installation (one-time)

Argo CD must be running before you apply the root Application. Two options:

**Option A – Install from this repo (recommended for replication)**  
This uses the same Kustomize overlay (Argo CD + patches + LB) that the argocd-lb Application will later manage:

```bash
kubectl create namespace argocd
kubectl apply -k infra/resources/
# Wait for Argo CD to be ready, then get initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Option B – Install upstream, then adopt**  
Install vanilla Argo CD, then apply `bootstrap/root-app.yaml`. The argocd-lb Application will sync `infra/resources/` and take over management (patches, LB service, etc.).

---

## How to replicate this setup

1. **Clone the repo**
   ```bash
   git clone https://github.com/mclaireh/argocd-apps-config.git
   cd argocd-apps-config
   ```

2. **Ensure cluster and kubectl**
   - Use MicroK8s (or another cluster) and enable add-ons as needed.
   - Confirm `kubectl get nodes` works.

3. **Install Argo CD once**
   ```bash
   kubectl create namespace argocd
   kubectl apply -k infra/resources/
   ```
   Wait for pods in `argocd` to be ready. Optionally patch the server to be reachable (e.g. LoadBalancer or port-forward).

4. **Apply the root Application**
   ```bash
   kubectl apply -f bootstrap/root-app.yaml
   ```
   Argo CD will sync the root Application, which will create the guestbook, blue-green, argocd-lb, and external-mdns Applications. Those apps will then sync their sources.

5. **Optional: GitHub SSO**
   To log in to the Argo CD UI with GitHub, follow **[infra/resources/GITHUB-SSO.md](infra/resources/GITHUB-SSO.md)** (OAuth App, patch `argocd-cm`, store client secret, restart Dex).

6. **Optional: Point the repo at your fork**
   If you fork this repo, replace `mclaireh/argocd-apps-config` with your org/repo in:
   - `bootstrap/root-app.yaml` (sources)
   - `infra/argocd-app.yaml` (source.repoURL)
   - Any other Application that references this repo.

---

## Design decisions

### Why Argo CD?

- **GitOps** – Desired state lives in Git; changes are auditable and revertible.
- **Declarative** – Applications and sources are declared as YAML; no one-off `kubectl apply` scripts.
- **Drift correction** – Argo CD continuously reconciles cluster state with Git (e.g. automated sync with selfHeal).
- **Single place** – One repo (or a small set) defines what runs on the cluster, making it easy to replicate or recover.

### Why this repo structure?

| Choice | Reason |
|--------|--------|
| **`bootstrap/`** | Holds the root Application only. Applied once (or by an external installer); keeps bootstrap separate from day-2 config. |
| **`apps/`** | Application manifests for workloads (guestbook, blue-green, etc.). Adding a new app = add a YAML file and let the root app pick it up. |
| **`infra/`** | Application manifests for cluster infrastructure (Argo CD itself, external-mdns). Keeps “platform” apps in one place. |
| **`infra/resources/`** | Raw manifests and Kustomize for Argo CD (install.yaml + patches + LB service). The **argocd-lb** Application syncs this path; the root app does **not** sync it (see below). |
| **Root app excludes `infra/resources`** | The root Application syncs `infra/` but excludes `infra/resources/`. So it creates the *Application* “argocd-lb” that points at `infra/resources`, but it does not apply the Argo CD install itself. That avoids the root app trying to deploy Argo CD from inside Argo CD in a circular way. The actual Argo CD install is done once (manually or by CI), then the argocd-lb Application takes over updates. |

### Why Kustomize for Argo CD?

- **Patches over upstream** – The repo uses the official Argo CD install manifest as a base and patches it (e.g. `argocd-cm` for GitHub SSO, `argocd-lb-service` for external access) without forking the full install.
- **Version pinning** – The base URL pins a specific Argo CD version (e.g. `v3.2.6`), so upgrades are explicit and controlled by Git.

---

## Repository layout

```
argocd-apps-config/
├── README.md                 # This file
├── bootstrap/
│   └── root-app.yaml         # Root Application (app-of-apps)
├── apps/
│   ├── guestbook-app.yaml
│   └── blue-green-app.yaml
├── infra/
│   ├── argocd-app.yaml       # Application that syncs infra/resources
│   ├── external-mdns-app.yaml
│   └── resources/            # Argo CD install + patches (synced by argocd-lb)
│       ├── kustomization.yaml
│       ├── install base (remote)
│       ├── argocd-cm-github-sso.yaml
│       ├── argocd-lb-service.yaml
│       └── GITHUB-SSO.md
```

---

## See also

- [Argo CD docs](https://argo-cd.readthedocs.io/)
- [GitHub SSO for Argo CD](infra/resources/GITHUB-SSO.md) (this repo)
