# Kubernetes Architecture

A beginner-friendly map of how a Kubernetes cluster is put together. Read this
first — it gives you the big picture that the later docs (pods, deployments,
services) plug into.

## What Is Kubernetes

Kubernetes (often shortened to **k8s** — "k", 8 letters, "s") is a system that
runs and manages containers across a group of machines for you. You tell it the
**desired state** ("run 3 copies of my app"), and it constantly works to make
the real world match that — restarting crashed containers, replacing dead nodes,
and scaling up or down.

This idea is the heart of Kubernetes: a **declarative**, self-healing control
loop. You declare *what* you want; Kubernetes figures out *how* to get there.

## The Big Picture

A **cluster** is a set of machines (**nodes**) working together. It has two
kinds of nodes:

- **Control plane** — the "brain" that makes global decisions (scheduling,
  responding to failures). It manages the cluster.
- **Worker nodes** — the "muscle" that actually run your application containers.

```
                          ┌──────────────────────────────────────┐
                          │            CONTROL PLANE              │
   kubectl ──► API ──────►│  api-server   scheduler               │
   (you)        server    │  controller-manager   etcd (store)    │
                          └──────────────────────────────────────┘
                                        │  (instructions)
                 ┌──────────────────────┼──────────────────────┐
                 ▼                      ▼                      ▼
          ┌────────────┐        ┌────────────┐        ┌────────────┐
          │ WORKER NODE│        │ WORKER NODE│        │ WORKER NODE│
          │ kubelet    │        │ kubelet    │        │ kubelet    │
          │ kube-proxy │        │ kube-proxy │        │ kube-proxy │
          │ [ Pods ]   │        │ [ Pods ]   │        │ [ Pods ]   │
          └────────────┘        └────────────┘        └────────────┘
```

> On **minikube** the control plane and the (single) worker node live inside one
> VM/container, so you don't see this split — but the components are all still
> there.

## Control Plane Components

These run the cluster. You interact with them mostly through the API server.

| Component                  | Job                                                                 |
|----------------------------|---------------------------------------------------------------------|
| **kube-apiserver**         | The front door. Every command and component talks through this REST API. |
| **etcd**                   | The cluster's database — a key-value store holding all cluster state. |
| **kube-scheduler**         | Decides *which node* a new Pod should run on (based on resources, rules). |
| **kube-controller-manager**| Runs control loops that drive actual state toward desired state.    |
| **cloud-controller-manager**| Integrates with a cloud provider (load balancers, nodes, routes). Optional. |

### How they fit together

1. You run `kubectl apply -f deployment.yml`.
2. `kubectl` sends the request to the **api-server**.
3. The api-server validates it and stores the desired state in **etcd**.
4. A **controller** notices "desired = 3 pods, actual = 0" and creates Pods.
5. The **scheduler** assigns each Pod to a suitable worker node.
6. That node's **kubelet** starts the containers.

## Worker Node Components

Every worker node runs these so it can host Pods.

| Component        | Job                                                                       |
|------------------|---------------------------------------------------------------------------|
| **kubelet**      | The node agent. Talks to the api-server and makes sure its Pods' containers are running and healthy. |
| **kube-proxy**   | Handles networking rules so Services can route traffic to the right Pods. |
| **container runtime** | The software that actually runs containers (e.g. containerd, CRI-O). |

## The Objects You'll Actually Work With

Kubernetes exposes higher-level building blocks (covered in the other docs):

| Object         | One-line summary                                              | Doc |
|----------------|--------------------------------------------------------------|-----|
| **Namespace**  | A virtual sub-cluster to group and isolate resources.        | [03](03-namespaces.md) |
| **Pod**        | Smallest deployable unit; wraps one or more containers.      | [04](04-pods.md) |
| **Deployment** | Manages a set of identical Pods: replicas, rollouts, healing.| [05](05-deployments.md) |
| **Service**    | Stable network endpoint + load balancing in front of Pods.   | [06](06-services.md) |

A typical ownership chain looks like:

```
Deployment ──► ReplicaSet ──► Pod ──► Container(s)
   (you define this)   (auto-created)   (your app)
```

You usually only write the Deployment; Kubernetes creates the ReplicaSet and
Pods to satisfy it.

## The Reconciliation Loop (Why Self-Healing Works)

Almost everything in Kubernetes follows the same pattern:

```
        ┌──────────────┐
        │ desired state│  (what you declared in YAML, stored in etcd)
        └──────┬───────┘
               │ compare
        ┌──────▼───────┐
        │ actual state │  (what's really running)
        └──────┬───────┘
               │ if different → take action (create/delete/restart)
               └───────────────► loop forever
```

This is why deleting a Pod from a Deployment just brings a new one back: the
controller sees actual (2) ≠ desired (3) and creates a replacement.

## How a Request Flows Through the Cluster

When a user hits your app:

1. Traffic arrives at a **Service** (a stable IP/DNS name).
2. **kube-proxy** rules forward it to one of the healthy backing **Pods**.
3. The Pod's **container** handles the request and responds.

If a Pod dies, the Service simply stops sending it traffic and routes to the
survivors — no client changes needed.

## Key Takeaways

- A cluster = **control plane** (decides) + **worker nodes** (run your apps).
- Everything goes through the **api-server**; all state lives in **etcd**.
- You declare **desired state**; controllers continuously **reconcile** reality
  to match it — that's what gives you self-healing and scaling.
- You mostly work with high-level objects (Deployments, Services), not raw
  containers.

## Where to Go Next

Continue with [Minikube](01-minikube.md) to spin up a real cluster, then
[kubectl](02-kubectl.md) to start inspecting these components yourself:

```bash
# See the worker nodes in your cluster
kubectl get nodes

# See the control-plane + system components (they run as Pods too)
kubectl get pods -n kube-system

# Inspect a node in detail
kubectl describe node <node-name>
```
