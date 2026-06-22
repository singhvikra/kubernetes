# Kubernetes — Learning Notes

Personal learning repository for Kubernetes. It collects hands-on notes,
practice manifests, and a walkthrough built from concepts I've worked through:
**minikube, kubectl, namespaces, pods, deployments, and services**.

## Learning Docs

Read these in order — each builds on the previous one:

0. [Architecture](docs/00-architecture.md) — the big picture: control plane, nodes, and how it all fits together.
1. [Minikube](docs/01-minikube.md) — run a local cluster and manage its lifecycle.
2. [kubectl](docs/02-kubectl.md) — the CLI: contexts, inspecting, applying, debugging.
3. [Namespaces](docs/03-namespaces.md) — partition a cluster into virtual sub-clusters.
4. [Pods](docs/04-pods.md) — the smallest deployable unit; lifecycle & the Downward API.
5. [Deployments](docs/05-deployments.md) — replicas, rollouts, scaling, self-healing.
6. [Services](docs/06-services.md) — stable networking and load balancing for pods.
7. [kubectl Command Flow](docs/07-command-flow.md) — trace a command through every component it touches.

## Practice Manifests

| File             | Kind        | Namespace     | What it demonstrates                                  |
|------------------|-------------|---------------|-------------------------------------------------------|
| `namespace.yml`  | Namespace   | —             | Creating the `development` namespace with a label.    |
| `deployment.yml` | Deployment  | `development` | 3 replicas + Downward API env vars (`POD_NAME`, etc). |
| `service.yml`    | Service     | `development` | `LoadBalancer` fronting the `pod-info` pods.          |
| `quote.yml`      | Deployment  | `development` | A second app (`datawire/quote`) with 2 replicas.      |
| `busybox.yml`    | Deployment  | `default`     | Resource requests/limits + keep-alive debug pod.      |
| `example.yml`    | —           | —             | Plain YAML syntax practice (not a Kubernetes object). |

## Prerequisites

- [`kubectl`](https://kubernetes.io/docs/tasks/tools/) — the Kubernetes CLI.
- A local cluster such as [minikube](https://minikube.sigs.k8s.io/),
  [kind](https://kind.sigs.k8s.io/), or Docker Desktop's built-in Kubernetes.

## Quick Start

```bash
# 1. Start a local cluster
minikube start

# 2. Create the namespace first (other resources depend on it)
kubectl apply -f namespace.yml

# 3. Deploy the app and expose it
kubectl apply -f deployment.yml
kubectl apply -f service.yml

# 4. (Optional) deploy the extra examples
kubectl apply -f quote.yml
kubectl apply -f busybox.yml

# 5. Check what's running
kubectl get all -n development

# 6. Reach the LoadBalancer service on minikube
minikube service pod-service -n development
#   or: kubectl port-forward svc/pod-service 8080:80 -n development

# 7. Tear it all down
kubectl delete -f service.yml -f deployment.yml -f quote.yml
kubectl delete -f busybox.yml
kubectl delete -f namespace.yml          # removes everything left in it
```

> Confirm your context first with `kubectl config current-context` to avoid
> applying practice manifests to the wrong cluster.

## Notes

This is a personal learning space, so manifests and notes may be incomplete or
experimental by design.
