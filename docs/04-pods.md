# Pods

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more
containers that share the same network namespace (one IP, shared `localhost`)
and can share storage volumes.

## Key Ideas

- You rarely create bare Pods in production — a controller like a **Deployment**
  creates and manages them for you. Bare Pods don't get rescheduled if they die.
- Every Pod gets its own cluster-internal IP.
- Containers in the same Pod talk to each other over `localhost`.
- A Pod runs on a single node; it is never split across nodes.

## Pod Lifecycle Phases

| Phase       | Meaning                                                |
|-------------|--------------------------------------------------------|
| `Pending`   | Accepted, but not yet running (pulling image, etc.).   |
| `Running`   | Bound to a node, at least one container is running.    |
| `Succeeded` | All containers exited 0 and won't restart.             |
| `Failed`    | All containers terminated, at least one failed.        |
| `Unknown`   | State could not be obtained (node comms issue).        |

Common per-container states you'll see in `describe`: `Waiting`
(`ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`), `Running`,
`Terminated`.

## Inspecting Pods

```bash
# List pods (the deployments below create these)
kubectl get pods -n development
kubectl get pods -o wide -n development        # show node + pod IP

# Full details + recent events (first stop for troubleshooting)
kubectl describe pod <pod-name> -n development

# Logs
kubectl logs <pod-name> -n development
kubectl logs -f <pod-name> -n development      # stream

# Shell into a running container
kubectl exec -it <pod-name> -n development -- sh
```

## Downward API (used in our pod-info deployment)

`deployment.yml` injects Pod metadata into the container as environment
variables using `fieldRef` — this is the **Downward API**:

```yaml
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
```

The app can then read `POD_NAME`, `POD_NAMESPACE`, and `POD_IP` at runtime —
handy for seeing which replica served a request. Verify with:

```bash
kubectl exec <pod-name> -n development -- env | grep POD_
```

## Resource Requests & Limits (from busybox.yml)

```yaml
resources:
  requests:          # guaranteed minimum; used for scheduling
    cpu: 30m         # 30 millicores = 0.03 CPU
    memory: 64Mi
  limits:            # hard ceiling; container is throttled/OOM-killed if exceeded
    cpu: 100m
    memory: 128Mi
```

- **requests** influence which node the scheduler picks.
- **limits** cap usage; exceeding the memory limit triggers an OOM kill.

## Keeping a Container Alive (busybox pattern)

A container exits when its main process ends. `busybox.yml` keeps the container
running so you can exec into it for debugging:

```yaml
command: ["/bin/sh", "-c", "--"]
args: ["while true; do sleep 30; done;"]
```

## Debug Pod Cheat Sheet

```bash
# Throwaway pod for poking at the cluster, deleted on exit
kubectl run tmp --image=busybox -it --rm -- sh

# From inside, test service DNS
wget -qO- http://pod-service.development
```
