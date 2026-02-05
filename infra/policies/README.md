# Gatekeeper Policies

This directory contains OPA Gatekeeper policies for the cluster.

## Required Labels Policy

The `require-common-labels` constraint warns (does not block) when resources are missing required labels.

### Current Configuration

**Enforcement**: `warn` - violations are logged but resources are not rejected

**Applies to**:
- Namespaces
- Deployments, StatefulSets, DaemonSets

**Excluded Namespaces**:
- kube-system, kube-public, kube-node-lease
- gatekeeper-system, argocd

**Required Labels**:
- `owner` - any alphanumeric string (team or person responsible)
- `environment` - must be one of: `dev`, `staging`, `prod`
- `app` - alphanumeric string (application name)

### Example Compliant Resource

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    owner: platform-team
    environment: prod
    app: my-app
spec:
  # ... deployment spec
```

### Customization

**To change required labels**: Edit `require-labels-constraint.yaml` under `parameters.labels`

**To change enforcement to blocking**: Change `enforcementAction: warn` to `enforcementAction: deny`

**To add more resource types**: Add them to `spec.match.kinds` in the constraint

**To exclude more namespaces**: Add them to `spec.match.excludedNamespaces`

### Viewing Violations

Check audit results:
```bash
kubectl get k8srequiredlabels require-common-labels -o yaml
```

View all violations:
```bash
kubectl get k8srequiredlabels require-common-labels -o jsonpath='{.status.violations}' | jq
```

### Deployment

This policy is deployed via the `gatekeeper-policies` Argo CD Application in `infra/gatekeeper-policies-app.yaml`.

Make sure to update the `repoURL` in that file to point to your repository before deploying.
