# Argo CD GitHub SSO Setup

Argo CD uses the **bundled Dex** IdP for GitHub OAuth. Follow these steps to enable "Login via GitHub" in the UI.

## 1. Create a GitHub OAuth App

1. Open **GitHub** → **Settings** → **Developer settings** → [OAuth Apps](https://github.com/settings/developers) → **New OAuth App**.
2. Set:
   - **Application name**: e.g. `Argo CD`
   - **Homepage URL**: your Argo CD URL (e.g. `https://argocd.local`)
   - **Authorization callback URL**:  
     `https://<your-argocd-url>/api/dex/callback`  
     Example: `https://argocd.local/api/dex/callback`
3. Register the app, then create a **Client Secret** and copy the **Client ID** and **Client Secret**.

## 2. Configure the patch

Edit `argocd-cm-github-sso.yaml` in this directory:

- Replace `GITHUB_CLIENT_ID` with your OAuth app **Client ID**.
- Replace `GITHUB_ORG` with your **GitHub organization** name. Only members of that org can log in.
- Ensure `url` matches how you access Argo CD (e.g. `https://argocd.local`). This must match the host used in the OAuth callback URL.

Optional: to restrict to a specific team within the org, add a `teams` list under the org:

```yaml
orgs:
- name: GITHUB_ORG
  teams:
  - my-team
```

## 3. Store the client secret in the cluster

Argo CD reads the Dex GitHub client secret from the `argocd-secret` Secret. Add it **after** the first Argo CD sync (so the secret already exists):

```bash
kubectl patch secret argocd-secret -n argocd --type merge \
  -p '{"stringData": {"dex.github.clientSecret": "<paste-your-client-secret>"}}'
```

Or create the key manually:

```bash
kubectl edit secret argocd-secret -n argocd
# Add under data: (value must be base64):
#   dex.github.clientSecret: <base64-encoded-client-secret>
```

To base64-encode the secret: `echo -n 'your-client-secret' | base64`

## 4. Apply and restart Dex

- Sync the Argo CD app that deploys `infra/resources` so the updated `argocd-cm` is applied.
- Restart the Dex server so it reloads config:

```bash
kubectl rollout restart deployment argo-cd-argocd-dex-server -n argocd
```

## 5. (Optional) RBAC by GitHub org/team

If you use an org (and optionally teams), you can map GitHub groups to Argo CD roles in `argocd-rbac-cm`. Dex exposes org/team membership as OIDC groups like `github:your-org` and `github:your-org:your-team`. Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:admin, github:your-org:admins, *, */*, allow
    g, your-user@example.com, role:admin
```

Then add a patch for `argocd-rbac-cm` in `kustomization.yaml` (or apply the ConfigMap separately).

## Summary

| Item | Where |
|------|--------|
| Client ID | `argocd-cm-github-sso.yaml` → `dex.config` → `clientID` |
| Client secret | Kubernetes Secret `argocd-secret`, key `dex.github.clientSecret` |
| Argo CD URL | `argocd-cm-github-sso.yaml` → `url` (and GitHub callback URL) |
| Who can log in | Set `GITHUB_ORG` in `argocd-cm-github-sso.yaml`; only members of that org can log in. |

After this, the Argo CD login page shows **Login via GitHub**; only members of your GitHub organization can sign in.
