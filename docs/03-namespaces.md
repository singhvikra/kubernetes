# Namespaces

Namespaces partition a single cluster into virtual sub-clusters. They scope
resource names, enable per-team isolation, and allow quotas/limits per group.

## Concepts

- Most resources (pods, deployments, services, configmaps, secrets) are
  **namespaced** — their names only need to be unique within a namespace.
- Some resources are **cluster-scoped** (nodes, namespaces, persistent volumes,
  cluster roles) and live outside any namespace.
- Default namespaces in every cluster:
  - `default` — where resources land if you don't specify one.
  - `kube-system` — Kubernetes control-plane components.
  - `kube-public` — readable by all users, rarely used directly.
  - `kube-node-lease` — node heartbeat objects.

## Our Example

`namespace.yml` creates a `development` namespace with an `environment: dev`
label:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
```

The `pod-info`, `quote-service`, and `pod-service` resources all live in this
namespace; `busybox-deployment` lives in `default`.

## Commands

```bash
# List namespaces
kubectl get namespaces            # short: kubectl get ns

# Create from our manifest
kubectl apply -f namespace.yml

# Or create imperatively
kubectl create namespace development

# Run a command against a specific namespace
kubectl get pods -n development

# Across all namespaces
kubectl get pods -A

# Make development the default for your current context
kubectl config set-context --current --namespace=development

# Delete a namespace (deletes everything inside it!)
kubectl delete namespace development
```

## Cross-Namespace DNS

Services get a DNS name of the form:

```
<service>.<namespace>.svc.cluster.local
```

- Same namespace: reach a service by just `<service>` (e.g. `pod-service`).
- Different namespace: use `<service>.<namespace>`
  (e.g. `pod-service.development`).

## When to Use Them

- Separate environments on one cluster: `dev`, `staging`, `qa`.
- Isolate teams or applications.
- Apply `ResourceQuota` and `LimitRange` per group to cap CPU/memory.

> Deleting a namespace cascades to all the objects inside it — double-check
> before running `kubectl delete namespace`.
