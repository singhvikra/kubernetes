# kubectl

`kubectl` is the command-line client for talking to a Kubernetes cluster's API
server. Every action — create, inspect, update, delete — goes through it.

## Anatomy of a Command

```
kubectl <verb> <resource-type> <name> [flags]
#       apply   deployment       web    -n development
```

Common verbs: `get`, `describe`, `create`, `apply`, `delete`, `edit`, `logs`,
`exec`, `scale`, `rollout`.

## Context & Configuration

`kubectl` reads `~/.kube/config` to know which cluster, user, and namespace to
target.

```bash
# Show the active context (which cluster you're pointing at)
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context (minikube sets this for you on start)
kubectl config use-context minikube

# Set a default namespace for the current context
kubectl config set-context --current --namespace=development

# Cluster info & health
kubectl cluster-info
kubectl get nodes
```

> Always confirm your context before applying manifests so you don't change
> the wrong cluster.

## Inspecting Resources

```bash
# List resources
kubectl get pods -n development
kubectl get pods -o wide                 # more columns (node, IP)
kubectl get all -n development           # common resources in the namespace
kubectl get pods -A                      # across every namespace

# Detailed human-readable info incl. events
kubectl describe pod <name> -n development

# Raw YAML/JSON of a live object
kubectl get pod <name> -n development -o yaml

# Filter by label (matches the app labels in our manifests)
kubectl get pods -l app=pod-info -n development
kubectl get pods -l app=quote-service -n development
```

## Creating & Updating

```bash
# Declarative (preferred): apply a manifest, reconcile to desired state
kubectl apply -f namespace.yml
kubectl apply -f deployment.yml
kubectl apply -f service.yml

# Imperative quick-create
kubectl create deployment web --image=nginx
kubectl run tmp --image=busybox -it --rm -- sh   # throwaway debug pod

# Edit a live object in your default editor
kubectl edit deployment pod-info-depoloyment -n development

# Delete
kubectl delete -f deployment.yml
kubectl delete pod <name> -n development
```

## Debugging

```bash
# Container logs
kubectl logs <pod> -n development
kubectl logs -f <pod> -n development      # follow (stream)
kubectl logs --previous <pod> -n development  # logs from a crashed container

# Run a command inside a container (busybox is handy for this)
kubectl exec -it <busybox-pod> -- sh
kubectl exec <pod> -n development -- env

# Forward a local port to a pod/service
kubectl port-forward svc/pod-service 8080:80 -n development

# Live events (great for troubleshooting)
kubectl get events --sort-by=.lastTimestamp -n development
```

## Handy Flags & Tips

- `-n <namespace>` / `-A` — target one namespace / all namespaces.
- `-o wide|yaml|json|name` — output format.
- `--dry-run=client -o yaml` — generate a manifest without creating anything:
  ```bash
  kubectl create deployment web --image=nginx --dry-run=client -o yaml > deployment.yaml
  ```
- `kubectl explain <resource>` — built-in schema docs, e.g.
  `kubectl explain pod.spec.containers`.
- `kubectl api-resources` — list every resource type and its short name.
