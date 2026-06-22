# Services

Pods are ephemeral — they come and go with their own changing IPs. A
**Service** gives a stable network identity (a virtual IP + DNS name) in front
of a set of pods, load-balancing traffic across them.

## How Selection Works

A Service finds its target pods using a **label selector**. Our `service.yml`
selects every pod labelled `app: pod-info` — exactly the pods created by
`deployment.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pod-service
  namespace: development
spec:
  type: LoadBalancer
  selector:
    app: pod-info          # matches the Deployment's pod labels
  ports:
    - port: 80             # port the Service exposes
      targetPort: 3000     # port the container listens on
```

- `port` — the port clients hit on the Service.
- `targetPort` — the container port traffic is forwarded to (3000 here, matching
  the pod-info container).

## Service Types

| Type           | Reachable from                | Use case                                 |
|----------------|-------------------------------|------------------------------------------|
| `ClusterIP`    | Inside the cluster only (default) | Internal service-to-service traffic   |
| `NodePort`     | `<NodeIP>:<30000-32767>`      | Quick external access for dev/testing    |
| `LoadBalancer` | External IP (cloud LB)        | Production external exposure (our `pod-service`) |
| `ExternalName` | DNS CNAME to an external host | Alias an off-cluster service             |

`LoadBalancer` builds on `NodePort`, which builds on `ClusterIP`.

## Service DNS

Every Service is reachable in-cluster by name:

```
pod-service                              # same namespace
pod-service.development                  # from another namespace
pod-service.development.svc.cluster.local  # fully qualified
```

## Commands

```bash
# Apply
kubectl apply -f service.yml

# List services (note CLUSTER-IP, EXTERNAL-IP, PORT(S))
kubectl get services -n development        # short: kubectl get svc

# Details: selector, endpoints, ports
kubectl describe service pod-service -n development

# See which pod IPs the Service currently routes to
kubectl get endpoints pod-service -n development
```

If `EXTERNAL-IP` shows `<pending>`, the selector is matching pods correctly but
no external load balancer is provisioned yet — see the minikube notes below.

## Accessing the Service on Minikube

A bare minikube cluster has no cloud load balancer, so `LoadBalancer` services
stay `<pending>`. Two ways to reach it:

```bash
# Option 1: open a tunnel that assigns a real external IP (keep it running)
minikube tunnel
kubectl get svc pod-service -n development   # EXTERNAL-IP now populated

# Option 2: let minikube open the service for you
minikube service pod-service -n development

# Option 3: port-forward (works for any service type)
kubectl port-forward svc/pod-service 8080:80 -n development
# then browse http://localhost:8080
```

## Troubleshooting

- **No endpoints?** The Service `selector` doesn't match any pod labels. Compare
  `kubectl get svc <name> -o yaml` selectors with `kubectl get pods --show-labels`.
- **Connection refused?** `targetPort` must match the container's listening port
  (3000 for pod-info, 5000 for the quote app).
- **Wrong namespace?** Service and pods must be in the same namespace (both are
  in `development` here) for the selector to match.
