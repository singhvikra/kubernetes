# kubectl Command Flow

A step-by-step trace of what actually happens when you run a `kubectl` command.
This builds on [Architecture](00-architecture.md) — read that first for the
component overview. Here we follow a single command through every component it
touches.

## The 30-Second Version

```
  YOU                                   CONTROL PLANE                         WORKER NODE
   │                                                                              │
   │ kubectl apply -f deployment.yml                                             │
   ▼                                                                              │
┌────────┐  1. HTTPS   ┌────────────┐  2. validate   ┌──────┐                    │
│kubectl │────────────►│ api-server │──── + store ──►│ etcd │                    │
└────────┘   (REST)    └─────┬──────┘                └──────┘                    │
                             │ 3. "new Deployment" event (watch)                 │
              ┌──────────────┼───────────────────────────┐                       │
              ▼              ▼                            ▼                       │
      ┌──────────────┐  ┌───────────┐             ┌────────────┐                 │
      │ controller-  │  │ scheduler │             │  api-server│◄──── watch ─────┤
      │ manager      │  │           │             │            │   ┌─────────┐   │
      └──────┬───────┘  └─────┬─────┘             └────────────┘   │ kubelet │───┤
   4. creates Pods    5. assigns Pod                  ▲            └────┬────┘   │
      (via api-server)   to a node                    │          7. start        │
                         (via api-server)             │             containers   │
                                                6. kubelet sees its Pod          ▼
                                                                          ┌────────────┐
                                                                          │  Pod runs  │
                                                                          └────────────┘
```

## Step-by-Step Walkthrough

Take a concrete command:

```bash
kubectl apply -f deployment.yml
```

### 1. kubectl builds and sends a request

- `kubectl` reads your local **kubeconfig** (`~/.kube/config`) to find the
  cluster URL, the right **context**, and your credentials.
- It converts your YAML into a JSON request and sends it over **HTTPS** to the
  **kube-apiserver**. Nothing happens locally beyond this — `kubectl` is just a
  REST client.

```bash
# See which cluster/credentials kubectl will use
kubectl config current-context
kubectl config view --minify
```

### 2. The api-server authenticates, authorizes, admits

The api-server is the **only** component that talks to etcd, so every request
runs a gauntlet first:

| Stage             | Question it answers                                  |
|-------------------|------------------------------------------------------|
| **Authentication**| Who are you? (certs, tokens, etc.)                   |
| **Authorization** | Are you allowed to do this? (RBAC rules)             |
| **Admission**     | Should this be modified or rejected? (policies, defaults) |
| **Validation**    | Is the object well-formed and schema-valid?          |

If any stage fails, you get an error back immediately (e.g. `Forbidden`,
`error validating data`).

### 3. The desired state is persisted to etcd

- The api-server writes the validated Deployment object to **etcd**.
- At this point the object *exists* but nothing is running yet — only the
  **desired state** has been recorded.
- The api-server returns success to `kubectl` (`deployment.apps/... created`).

> Components don't poll etcd directly. They **watch** the api-server, which
> streams change events to anyone interested.

### 4. Controllers react (reconciliation begins)

- The **deployment controller** (inside kube-controller-manager) is watching for
  Deployment changes. It sees the new object and creates a **ReplicaSet**.
- The **replicaset controller** then sees "desired = 3 Pods, actual = 0" and
  creates 3 **Pod** objects — again by calling the api-server, which stores them
  in etcd.
- These new Pods have **no node assigned** yet (`nodeName` is empty).

### 5. The scheduler places the Pods

- The **kube-scheduler** watches for Pods with no node.
- For each one it picks the best node based on resource **requests**, taints,
  affinity rules, etc.
- It records the decision by updating the Pod's `nodeName` through the
  api-server (which writes it to etcd).

### 6. The kubelet on the chosen node notices

- Each node's **kubelet** watches the api-server for Pods assigned to *its* node.
- When it sees a Pod bound to itself, it pulls the container image and instructs
  the **container runtime** (containerd/CRI-O) to start the containers.

### 7. The Pod runs and reports status

- The container runtime starts the containers; the **kubelet** monitors them and
  reports status (`Running`, `Ready`, restarts) back to the api-server → etcd.
- **kube-proxy** programs the node's networking so any **Service** in front of
  these Pods can route traffic to them.
- Now `kubectl get pods` reflects the live state — because it's reading that same
  status back out of the api-server.

## Read Commands Take a Shorter Path

A command like `kubectl get pods` doesn't involve the scheduler, controllers, or
kubelet at all:

```
kubectl ──HTTPS──► api-server ──read──► etcd ──► response ──► kubectl
```

It's just authenticated, authorized, and answered from etcd. That's why reads
are fast and why the api-server is the single source of truth for every query.

## Who Talks to Whom (Quick Reference)

| From                | To           | Purpose                                  |
|---------------------|--------------|------------------------------------------|
| kubectl             | api-server   | Submit/read objects (the only entry point)|
| api-server          | etcd         | Persist & read all cluster state         |
| controller-manager  | api-server   | Watch desired state, create child objects|
| scheduler           | api-server   | Watch unscheduled Pods, assign nodes     |
| kubelet             | api-server   | Watch its Pods, report node/Pod status   |
| kube-proxy          | api-server   | Watch Services/Endpoints, program routing|

Notice the pattern: **everything goes through the api-server**. No component
talks to etcd except the api-server, and components coordinate by *watching*,
not by calling each other directly.

## Tracing It Yourself

```bash
# Watch the chain unfold in real time (run in a second terminal first)
kubectl get pods -n development --watch

# Then apply and observe Pending ──► ContainerCreating ──► Running
kubectl apply -f deployment.yml

# See the events that the components emitted along the way
kubectl describe deployment pod-info -n development
kubectl get events -n development --sort-by=.lastTimestamp

# See verbose api-server requests kubectl actually makes
kubectl get pods -n development -v=6
```

## Key Takeaways

- `kubectl` is a thin REST client; the **api-server** does the real work.
- The flow is always **declare → store → reconcile**: write desired state to
  etcd, then controllers/scheduler/kubelet drive reality to match it.
- Components are **decoupled** — they watch the api-server and act independently,
  which is what makes the system resilient and extensible.

## Where to Go Next

- [Architecture](00-architecture.md) — the component map this flow runs on.
- [kubectl](02-kubectl.md) — the day-to-day commands you'll send.
- [Deployments](05-deployments.md) — the object this walkthrough created.
