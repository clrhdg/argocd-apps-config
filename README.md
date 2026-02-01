# argocd-apps-config

## GitHub SSO for Argo CD

To use GitHub SSO for the Argo CD UI, see **[infra/resources/GITHUB-SSO.md](infra/resources/GITHUB-SSO.md)** for:

1. Creating a GitHub OAuth App and callback URL
2. Configuring the `argocd-cm` patch (client ID, org, URL)
3. Storing the client secret in `argocd-secret`
4. Optional RBAC by GitHub org/team