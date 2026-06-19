# Kubernetes

Personal learning repository for Kubernetes — notes, practice manifests, and
hands-on experiments while studying core concepts.

## Contents

This repo is a sandbox for exploring Kubernetes. Over time it will collect:

- **Notes** — concepts, commands, and gotchas captured while learning.
- **Manifests** — practice YAML for Pods, Deployments, Services, ConfigMaps,
  Secrets, Ingress, and more.
- **Experiments** — small scenarios used to test behavior and understanding.

## Prerequisites

- [`kubectl`](https://kubernetes.io/docs/tasks/tools/) — the Kubernetes CLI.
- A local cluster such as [kind](https://kind.sigs.k8s.io/),
  [minikube](https://minikube.sigs.k8s.io/), or Docker Desktop's built-in
  Kubernetes.

## Usage

Apply a manifest to your current cluster context:

```bash
kubectl apply -f path/to/manifest.yaml
```

Inspect what is running:

```bash
kubectl get pods
kubectl get all
```

Remove resources created from a manifest:

```bash
kubectl delete -f path/to/manifest.yaml
```

> Tip: check your active context first with `kubectl config current-context`
> to avoid applying practice manifests to the wrong cluster.

## Notes

This is a personal learning space, so manifests and notes may be incomplete or
experimental by design.
