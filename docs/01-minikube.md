# Minikube

Minikube runs a single-node Kubernetes cluster locally inside a VM or
container — ideal for learning and development.

## Why Minikube

- Spin up a real Kubernetes cluster on your laptop in minutes.
- Safe sandbox to practice without touching a cloud cluster.
- Supports add-ons (dashboard, ingress, metrics-server).

## Install

- macOS: `brew install minikube`
- Windows: `winget install Kubernetes.minikube` (or `choco install minikube`)
- Linux: download the binary from the
  [official docs](https://minikube.sigs.k8s.io/docs/start/).

## Lifecycle Commands

```bash
# Start a cluster (uses the default driver: docker, hyperv, etc.)
minikube start

# Start with a specific driver and resources
minikube start --driver=docker --cpus=2 --memory=4096

# Check cluster status
minikube status

# Stop the cluster (keeps state)
minikube stop

# Delete the cluster entirely
minikube delete

# Pause / unpause without losing state
minikube pause
minikube unpause
```

## Useful Commands

```bash
# Open the Kubernetes dashboard in a browser
minikube dashboard

# List enabled/available add-ons
minikube addons list

# Enable an add-on (e.g. ingress, metrics-server)
minikube addons enable ingress
minikube addons enable metrics-server

# Get the cluster IP
minikube ip

# SSH into the minikube node
minikube ssh

# Open a Service in the browser (creates a tunnel if needed)
minikube service pod-service -n development

# Tunnel so LoadBalancer services get an external IP
minikube tunnel
```

> The `pod-service` is the LoadBalancer Service defined in `service.yml`. On
> minikube a `LoadBalancer` stays `<pending>` until `minikube tunnel` is
> running, or you can reach it with `minikube service <name>`.

## Working with Local Images

By default, minikube has its own Docker daemon. To build images that pods can
use without pushing to a registry:

```bash
# Point your shell's docker CLI at minikube's daemon
eval $(minikube docker-env)               # macOS/Linux
minikube docker-env | Invoke-Expression   # PowerShell

# Now `docker build` lands inside minikube
docker build -t myapp:local .
```

Then reference `image: myapp:local` with `imagePullPolicy: Never` in your pod
spec.

## Gotchas

- `minikube start` selects a driver automatically; pin it with `--driver` for
  reproducibility.
- The `docker-env` trick only affects the current shell session.
- `minikube tunnel` must keep running in a terminal to serve LoadBalancer IPs.
