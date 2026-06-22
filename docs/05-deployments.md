# Deployments

A **Deployment** manages a replicated set of identical Pods. It provides
declarative updates: you describe the desired state (image, replica count) and
the Deployment continuously reconciles reality to match it.

## How It Fits Together

```
Deployment ──manages──▶ ReplicaSet ──manages──▶ Pods
```

- The **Deployment** records desired state and rollout strategy.
- It creates a **ReplicaSet** that ensures N pod replicas exist.
- Each rollout (e.g. image change) creates a new ReplicaSet and shifts pods
  over, enabling rolling updates and rollbacks.

## Anatomy (from deployment.yml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-info-depoloyment
  namespace: development
spec:
  replicas: 3                 # desired number of pods
  selector:
    matchLabels:
      app: pod-info           # which pods this Deployment owns
  template:                   # the Pod template stamped out for each replica
    metadata:
      labels:
        app: pod-info         # MUST match spec.selector.matchLabels
    spec:
      containers:
        - name: pod-info-container
          image: kimschles/pod-info-app:latest
          ports:
            - containerPort: 3000
```

> **Critical rule:** `spec.selector.matchLabels` must match
> `spec.template.metadata.labels`, or the API server rejects the Deployment.

Our repo has three Deployments:

| Manifest         | Name                    | Namespace     | Replicas | Image                     |
|------------------|-------------------------|---------------|----------|---------------------------|
| `deployment.yml` | `pod-info-depoloyment`  | `development` | 3        | `kimschles/pod-info-app`  |
| `quote.yml`      | `quote-deployment`      | `development` | 2        | `datawire/quote:0.5.0`    |
| `busybox.yml`    | `busybox-deployment`    | `default`     | 1        | `busybox:latest`          |

## Common Commands

```bash
# Apply / update
kubectl apply -f deployment.yml

# List deployments and their replica status
kubectl get deployments -n development
kubectl get deploy,rs,pods -n development   # see the whole chain

# Detailed view + events
kubectl describe deployment pod-info-depoloyment -n development
```

## Scaling

```bash
# Imperative scale
kubectl scale deployment pod-info-depoloyment --replicas=5 -n development

# Declarative: edit replicas in deployment.yml, then re-apply
kubectl apply -f deployment.yml
```

## Rollouts & Updates

```bash
# Change the image (triggers a rolling update)
kubectl set image deployment/pod-info-depoloyment \
  pod-info-container=kimschles/pod-info-app:v2 -n development

# Watch the rollout progress
kubectl rollout status deployment/pod-info-depoloyment -n development

# View rollout history
kubectl rollout history deployment/pod-info-depoloyment -n development

# Roll back to the previous revision
kubectl rollout undo deployment/pod-info-depoloyment -n development

# Restart all pods (e.g. to pick up a new :latest image or config)
kubectl rollout restart deployment/pod-info-depoloyment -n development
```

## Self-Healing Demo

Delete a pod and watch the Deployment recreate it to maintain the replica count:

```bash
kubectl delete pod <one-pod-name> -n development
kubectl get pods -n development -w        # a replacement appears immediately
```

## Tips

- Avoid `:latest` in real environments — pin image tags so rollouts/rollbacks
  are deterministic. (`pod-info-app:latest` and `busybox:latest` are fine for
  learning.)
- `kubectl delete deployment <name>` removes the Deployment, its ReplicaSet,
  and all its pods.
