# Changelog

All notable changes to this project are documented in this file. Entries are grouped by merge date (newest first) and include the PR branch name.

---

## 2026-02-11

### **Fix Gatekeeper out-of-sync by enabling ServerSideApply**  
**Branch:** `fix/gatekeeper-out-of-sync`  
Adds `ServerSideApply=true` to syncOptions in `infra/gatekeeper.yaml` to resolve persistent out-of-sync issues. Gatekeeper CRDs can exceed Kubernetes' annotation size limits when using client-side apply; server-side apply avoids this by having the API server manage field ownership without storing the full configuration in annotations.

---

## 2026-02-04

### **Replace Mermaid architecture diagram with ASCII art in README**  
**Branch:** `fix/fix-arch-diagram`  
Converts the Mermaid flowchart in `README.md` to an ASCII art diagram using box-drawing characters. The new diagram maintains the same architectural information (Root App → apps/infra directories → Applications → sources) but renders in any text viewer without requiring Mermaid support. Updates flow description to include ESO and Vault applications.

### **Add Gatekeeper policy for required labels with warn enforcement**  
**Branch:** `feature/add-gatekeeper-policies`  
Adds OPA Gatekeeper policy to warn when resources are missing required labels. Creates `infra/policies/` directory with `K8sRequiredLabels` ConstraintTemplate and constraint that checks for `pp.kubernetes.io/name`, and `app.kubernetes.io/version` labels on Deployments, StatefulSets, DaemonSets, and Namespaces. Uses `enforcementAction: warn` to log violations without blocking resources. Includes Argo CD Application (`gatekeeper-policies-app.yaml`) to manage policy deployment.

### **Gatekeeper: add Helm values for webhook, resources, and audit**  
**Branch:** `fix/gatekeeper`  
Configures the Gatekeeper Application with Helm values: `validatingWebhookTimeoutSeconds: 3`, controller manager resource limits/requests (cpu/memory), and audit enabled with 60s interval.

### **Fix External Secrets sync: use ServerSideApply for large CRDs**  
**Branch:** `feature/fix-eso-sync`  
Fixes sync failures for the External Secrets application caused by CRD `metadata.annotations` exceeding Kubernetes’ 262144-byte limit. Adds `ServerSideApply=true` to `infra/eso-app.yaml` syncOptions so Argo CD uses server-side apply and no longer stores the full resource in annotations.

### **Add External Secrets Operator and Vault for secrets management**  
**Branch:** `feature/secrets-management`  
Adds Argo CD Applications for secrets tooling: `infra/eso-app.yaml` deploys External Secrets Operator (chart v1.3.2) into `external-secrets`; `infra/vault-app.yaml` deploys HashiCorp Vault (chart v0.32.0) into `vault`. Both use auto-sync, prune, and self-heal.

### **Add Gatekeeper via Helm**  
**Branch:** `feature/gatekeeper`  
Adds Argo CD Application (`infra/gatekeeper.yaml`) that deploys OPA Gatekeeper from the official Helm chart (v3.21.0) into `gatekeeper-system` with auto-sync, prune, and self-heal.

---

## 2026-02-03

### **Add CHANGELOG.md**  
**Branch:** `feature/add-changlog`  
Adds this changelog at the repo root, populated with entries for all existing PRs (branch name and summary per PR), grouped by date.

### **Update repo URLs after transfer to new GitHub account**  
**Branch:** `feature/update-repo-urls-after-transfer`  
Updates all in-repo references to this repository to the new GitHub location after transferring the repo from one account to another. `bootstrap/root-app.yaml` and `infra/argocd-app.yaml` now point at the new repo URL so Argo CD syncs from the correct source.

### [#20](https://github.com/shields-farm/argocd-apps-config/pull/20) — **Update README.md**  
**Branch:** `clrhdg-patch-1`  
Fixed a single typo in the numbering on README.md.

### [#19](https://github.com/shields-farm/argocd-apps-config/pull/19) — **Add comprehensive README for GitOps setup and bootstrap**  
**Branch:** `feature/add-readme`  
Adds a full README for the argocd-apps-config repo: what it does, architecture overview, Mermaid diagram, directory structure, how to add new apps, and bootstrap (first-time install) with prerequisites and step-by-step instructions.

---

## 2026-02-02

### [#18](https://github.com/shields-farm/argocd-apps-config/pull/18) — **Remove GitHub SSO components and documentation**  
**Branch:** `feature/remove-github-sso`  
Removes all GitHub SSO (Dex) configuration and related docs. Argo CD runs with default auth only. Removed `argocd-cm-github-sso.yaml`, `GITHUB-SSO.md`; updated `kustomization.yaml` and README.

---

## 2026-02-01

### [#17](https://github.com/shields-farm/argocd-apps-config/pull/17) — **Fixed a typo**  
**Branch:** `feature/add-github-sso`  
Typo fix in the GitHub SSO PR branch.

### [#15](https://github.com/shields-farm/argocd-apps-config/pull/15) — **Initial GitHub SSO Setup**  
**Branch:** `feature/add-github-sso`  
Enables "Login via GitHub" in the Argo CD UI using Dex and a GitHub OAuth App, restricted to members of a configured GitHub organization. Adds `argocd-cm-github-sso.yaml`, `GITHUB-SSO.md`; updates `kustomization.yaml` and README.

---

## 2026-01-31

### [#14](https://github.com/shields-farm/argocd-apps-config/pull/14) — **Add mDNS hostname annotation for ArgoCD LoadBalancer**  
**Branch:** `feature/add-mdns-annotation`  
Adds `external-dns.alpha.kubernetes.io/hostname: argocd.local` to the ArgoCD LoadBalancer service so external-mdns advertises `argocd.local`.

### [#13](https://github.com/shields-farm/argocd-apps-config/pull/13) — **fixed typo**  
**Branch:** `feature/fix-argocd-sync`  
Fixes kustomization resource filename: `argocd-lb-app.yaml` → `argocd-lb-service.yaml`.

### [#12](https://github.com/shields-farm/argocd-apps-config/pull/12) — **Switch to Kustomize for ArgoCD installation**  
**Branch:** `feature/add-kustomize`  
Replaces Helm-based ArgoCD deployment with Kustomize; uses ArgoCD v3.2.6 upstream manifests and includes the LoadBalancer service in the kustomization.

### [#11](https://github.com/shields-farm/argocd-apps-config/pull/11) — **Fix LoadBalancer targetPort to match ArgoCD server**  
**Branch:** `feature/fix-app-of-apps`  
Updates LoadBalancer service targetPort from 443/80 to 8080 to match the ArgoCD server container port.

### [#10](https://github.com/shields-farm/argocd-apps-config/pull/10) — **Remove Ingress in favor of LoadBalancer with mDNS**  
**Branch:** `feature/remove-ingress`  
Removes nginx Ingress for ArgoCD; access is via LoadBalancer + mDNS at `argocd.local`. Deleted `infra/resources/argocd-ingress.yaml`.

### [#9](https://github.com/shields-farm/argocd-apps-config/pull/9) — **Add ArgoCD self-management via Helm chart**  
**Branch:** `feature/argocd-manage-itself`  
Adds ArgoCD Application to manage ArgoCD via official Helm chart (v9.3.5), with resource exclusions and ignore rules; server remains ClusterIP with external access via `argocd-server-lb`.

### [#8](https://github.com/shields-farm/argocd-apps-config/pull/8) — **Fix SharedResourceWarning by restructuring App of Apps pattern**  
**Branch:** `feature/fix-scan-root`  
Fixes SharedResourceWarning by separating Application definitions from raw resources. Root app now uses explicit `sources` for `apps/` and `infra/` with exclude for `resources/*`; moved LB/Ingress to `infra/resources/`; removed redundant `argocd-infra-app.yaml`.

### [#7](https://github.com/shields-farm/argocd-apps-config/pull/7) — **Make root-apps watch entire repo + add ArgoCD LoadBalancer service**  
**Branch:** `feature/fix-scan-root`  
Root Application watches entire repo (`path: .`, `directory.recurse: true`). Adds `argocd-server-lb` LoadBalancer service and `argocd-lb-app.yaml` for MetalLB and mDNS at `argocd.local`.

### [#6](https://github.com/shields-farm/argocd-apps-config/pull/6) — **Adding Load Balancer to point to argocd.local using mdns**  
**Branch:** `feature/load-balancer`  
Adds LoadBalancer service for ArgoCD (MetalLB), updates Ingress backend to use it, adds `argocd-lb-app.yaml` for GitOps management and mDNS at `argocd.local`.

---

## 2026-01-25

### [#5](https://github.com/shields-farm/argocd-apps-config/pull/5) — **add external-mdns**  
**Branch:** `feature/add-external-mdns`  
Adds external-mdns so ArgoCD can get its own IP/hostname.

### [#4](https://github.com/shields-farm/argocd-apps-config/pull/4) — **setting up ingress**  
**Branch:** `feature/add-ingress`  
Sets up ingress to support external-mdns and a static address for ArgoCD.

### [#3](https://github.com/shields-farm/argocd-apps-config/pull/3) — **renaming bootstrap/root.yaml to bootstrap/root-app.yaml**  
**Branch:** `feature/fix-root-app`  
Renames `bootstrap/root.yaml` to `bootstrap/root-app.yaml` for consistency.

### [#2](https://github.com/shields-farm/argocd-apps-config/pull/2) — **moved apps to app directory and renamed blue-green**  
**Branch:** `feature/fix-directory`  
Moves application YAMLs from repo root into `apps/` and renames `nginx-app.yaml` to `blue-green-app.yaml`.

### [#1](https://github.com/shields-farm/argocd-apps-config/pull/1) — **Feature/initial setup**  
**Branch:** `feature/initial-setup`  
Initial setup: Argo CD and two example apps (guestbook, blue-green).
