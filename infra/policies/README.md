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
