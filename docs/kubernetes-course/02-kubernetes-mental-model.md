# Chapter 02. Kubernetes mental model

## Learning objectives

After completing this chapter you should be able to:

- Describe how Docker Compose and Kubernetes differ in their approach to orchestration.
- Name each control plane component and explain its role in one sentence.
- Trace the path of a resource from a YAML manifest to a running container.
- Define desired state, actual state, and reconciliation.
- Explain why Kubernetes is eventually consistent and what that means in practice.
- Use `kubectl` to observe reconciliation happening in real time.

---

## From Docker Compose to Kubernetes

Docker Compose and Kubernetes solve related problems — running multiple containers
together — but their architectures are fundamentally different.

### Docker Compose: imperative process management

Compose reads a `compose.yml` file and translates it into a sequence of Docker API calls.
The `docker compose up` process is the single point of control: it creates networks,
starts containers in dependency order, and streams their logs. If the process dies, no
other agent restarts containers or enforces the declared state. Compose is a client-side
tool that talks to one Docker daemon.

```text
compose.yml ──► docker compose up ──► Docker daemon ──► containers
                  (single process)      (single host)
```

### Kubernetes: declarative reconciliation

Kubernetes stores desired state in a distributed database (etcd) and runs a set of
independent controllers that continuously compare desired state with actual state. If
they diverge — a container crashes, a node goes down, you change a manifest — controllers
act to close the gap. No single process owns the lifecycle. The system is designed to
converge on the declared state without human intervention.

```text
manifest ──► API server ──► etcd (desired state)
                                    │
             ┌──────────────────────┘
             ▼
         controllers ◄──► actual state (nodes, containers, network)
             │
             └──► reconcile: create, update, or delete resources
                  until actual == desired
```

### Key differences

| Aspect                  | Docker Compose                               | Kubernetes                                |
|-------------------------|----------------------------------------------|-------------------------------------------|
| State model             | Imperative (do this now)                     | Declarative (make it look like this)      |
| Failure recovery        | Manual restart or restart policy on one host | Automatic rescheduling across nodes       |
| Scaling                 | `scale:` flag, single host                   | ReplicaSets, multiple nodes               |
| Service discovery       | DNS on a Docker network                      | DNS backed by Services and EndpointSlices |
| Configuration           | Environment, bind mounts                     | ConfigMaps, Secrets, volumes              |
| Single point of failure | The `docker compose` process                 | None — distributed control plane          |
| Dependency ordering     | `depends_on` with conditions                 | No built-in ordering — readiness-based    |

The last row is significant for this course. The Signet Playground stack relies heavily
on `depends_on` conditions. Kubernetes has no equivalent. Chapter 10 addresses how to
replace startup ordering with readiness probes, init containers, and idempotent Jobs.

---

## Control plane components

The control plane is the set of processes that manage cluster state. In a Minikube
single-node cluster they all run on the same machine; in production they typically run on
dedicated nodes.

```text
┌─────────────────────────────────────────────────────────┐
│                     Control Plane                       │
│                                                         │
│  ┌───────────────┐  ┌────────┐  ┌────────────────────┐  │
│  │  API server   │  │  etcd  │  │ controller manager │  │
│  │  (kube-       │  │        │  │ (kube-controller-  │  │
│  │   apiserver)  │◄─┤        │  │  manager)          │  │
│  └──────┬────────┘  └────────┘  └────────────────────┘  │
│         │                                               │
│  ┌──────┴───────┐                                       │
│  │  scheduler   │                                       │
│  │  (kube-      │                                       │
│  │   scheduler) │                                       │
│  └──────────────┘                                       │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                        Node                             │
│                                                         │
│  ┌───────────┐  ┌────────────┐  ┌───────────────────┐   │
│  │  kubelet  │  │ kube-proxy │  │ container runtime │   │
│  └───────────┘  └────────────┘  └───────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### kube-apiserver

The API server is the front door to the cluster. Every interaction — `kubectl` commands,
controller decisions, kubelet reports — goes through the API server as REST calls. It
validates requests, persists them to etcd, and notifies watchers of changes. It does not
run containers or make scheduling decisions.

### etcd

A distributed key-value store that holds all cluster state: every Pod, Service,
ConfigMap, Secret, and internal record. The API server is the only component that talks
to etcd directly. Losing etcd means losing the cluster's memory of what should exist.

### kube-scheduler

Watches for newly created Pods that have no node assignment. For each one, it evaluates
resource requests, constraints (affinity, taints, topology), and available capacity, then
writes a binding decision back to the API server. The scheduler does not start
containers — it only decides _where_ they should run.

### kube-controller-manager

Runs a collection of controllers, each responsible for reconciling one type of resource.
Examples:

| Controller     | Watches           | Reconciles by                               |
|----------------|-------------------|---------------------------------------------|
| ReplicaSet     | ReplicaSets, Pods | Creating or deleting Pods to match replicas |
| Deployment     | Deployments       | Creating or updating ReplicaSets            |
| Job            | Jobs, Pods        | Creating Pods until completions are met     |
| Node           | Node heartbeats   | Marking nodes NotReady after timeout        |
| ServiceAccount | Namespaces        | Creating default ServiceAccount per ns      |

Each controller runs an independent loop: watch → compare → act. They do not coordinate
with each other except through the API server.

### kubelet

An agent that runs on every node. It receives Pod specifications from the API server,
ensures the declared containers are running via the container runtime, reports status
back, and executes probes. If a container crashes the kubelet restarts it according to
the Pod's restart policy.

### kube-proxy

Maintains network rules on each node so that Service virtual IPs route to the correct
Pod endpoints. It watches Service and EndpointSlice objects and updates iptables or IPVS
rules accordingly.

### container runtime

The low-level component that actually pulls images and runs containers. Minikube
typically uses containerd or Docker. Kubernetes talks to any runtime that implements the
Container Runtime Interface (CRI). You rarely interact with the runtime directly —
kubelet abstracts it.

---

## Desired state, actual state, and reconciliation

This is the single most important concept in Kubernetes.

### The reconciliation loop

Every controller follows the same pattern:

```text
1. Observe   — read the desired state from the API server
2. Compare   — diff desired vs. actual
3. Act       — create, update, or delete resources to close the gap
4. Report    — write status back to the API server
5. Repeat
```

This runs continuously. There is no "deploy" command that executes once and exits. The
system is always converging.

### Example: what happens when you create a Deployment

Suppose you apply a Deployment manifest requesting 3 replicas of an nginx Pod:

```text
 You                API server           etcd
  │                    │                   │
  ├─ apply Deployment ─►                   │
  │                    ├─ store Deployment ─►
  │                    │                   │
  │              Deployment controller     │
  │                    │                   │
  │          "0 ReplicaSets exist,         │
  │           spec says 3 replicas"        │
  │                    │                   │
  │                    ├─ create RS ───────►
  │                    │                   │
  │              ReplicaSet controller     │
  │                    │                   │
  │          "0 Pods exist,                │
  │           RS says 3 replicas"          │
  │                    │                   │
  │                    ├─ create 3 Pods ───►
  │                    │                   │
  │              Scheduler                 │
  │                    │                   │
  │          "3 Pods have no node"         │
  │                    │                   │
  │                    ├─ bind Pods ───────►
  │                    │       to node     │
  │              kubelet (on the node)     │
  │                    │                   │
  │          "3 Pods bound to me,          │
  │           containers not running"      │
  │                    │                   │
  │                    ├─ start containers │
  │                    ├─ report Running ──►
```

Seven components collaborate through the API server. None of them knows the full
picture — each watches only what it owns and acts on the difference.

### Eventual consistency

Kubernetes does not guarantee that the cluster matches the desired state at any given
instant. It guarantees that controllers will keep trying to make it match. Between a
change and convergence there is a window where desired and actual state differ. This is
normal.

Consequences you will encounter:

- A `kubectl apply` returns immediately, before containers are running.
- `kubectl get pods` may show `Pending` or `ContainerCreating` for several seconds.
- Deleting a Pod does not instantly remove it — a graceful termination period runs first.
- A node failure is detected by heartbeat timeout (default 40 seconds), not instantly.

Understanding eventual consistency prevents a common mistake: assuming that a successful
`kubectl apply` means the workload is ready. Use `kubectl wait` or probe-based readiness
to confirm actual state.

### Resource versions

Every object in etcd carries a `resourceVersion` — an opaque string that changes on
every update. Controllers use it to detect conflicts (optimistic concurrency) and to
watch for changes efficiently (the API server streams only events newer than the
requested resource version). You do not manage resource versions directly, but you will
see them in `kubectl get -o yaml` output.

---

## Workshop

### Exercise 1 — Observe control plane Pods

List the control plane components running as Pods:

```bash
kubectl get pods -n kube-system -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
NODE:.spec.nodeName,\
RESTARTS:.status.containerStatuses[0].restartCount
```

Identify each component from the earlier table. Note that they all run on the single
Minikube node.

Check the logs of the API server (the last 20 lines are enough):

```bash
kubectl logs -n kube-system kube-apiserver-signet-lab --tail=20
```

You will see request logs — every `kubectl` command you run appears here as an API call.

### Exercise 2 — Watch reconciliation in real time

Open two terminal windows (or use `tmux`/`screen`).

**Terminal 1** — watch Pods continuously:

```bash
kubectl get pods -w
```

**Terminal 2** — create a Deployment:

```bash
kubectl create deployment hello --image=nginx:1.27 --replicas=3
```

In Terminal 1 you will see events arriving in order:

1. Three Pods appear with status `Pending`.
2. They transition to `ContainerCreating` (kubelet pulling the image).
3. They reach `Running`.

This is reconciliation in action: the Deployment controller created a ReplicaSet, the
ReplicaSet controller created Pods, the scheduler assigned them, and the kubelet started
containers.

### Exercise 3 — Trace the object chain

Inspect the ownership chain that reconciliation created:

```bash
# The Deployment
kubectl get deployment hello -o wide

# The ReplicaSet it created (note the name includes the Deployment name)
kubectl get replicasets -l app=hello

# The Pods the ReplicaSet created
kubectl get pods -l app=hello -o custom-columns=\
NAME:.metadata.name,\
OWNER:.metadata.ownerReferences[0].name,\
STATUS:.status.phase
```

Each Pod's `ownerReferences` points to the ReplicaSet. The ReplicaSet's
`ownerReferences` points to the Deployment. This chain is how Kubernetes knows which
controller is responsible for which resource.

Verify:

```bash
kubectl get replicaset -l app=hello -o jsonpath='{.items[0].metadata.ownerReferences[0].name}'
```

The output should be `hello` — the Deployment name.

### Exercise 4 — Break something and watch recovery

Delete one of the three Pods directly:

```bash
POD=$(kubectl get pods -l app=hello -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod "$POD"
```

In Terminal 1 (still running `kubectl get pods -w`) observe:

1. The deleted Pod enters `Terminating`.
2. A new Pod appears almost immediately with status `Pending`, then `Running`.

The ReplicaSet controller detected that actual replicas (2) diverged from desired
replicas (3) and created a replacement. You did not ask for this — the reconciliation
loop handled it automatically.

Check the ReplicaSet events:

```bash
kubectl describe replicaset -l app=hello | grep -A5 Events
```

You should see a `SuccessfulCreate` event for the replacement Pod.

### Exercise 5 — Observe resource versions

```bash
kubectl get deployment hello -o jsonpath='{.metadata.resourceVersion}'
```

Note the value. Now scale the Deployment:

```bash
kubectl scale deployment hello --replicas=2
```

Check the resource version again:

```bash
kubectl get deployment hello -o jsonpath='{.metadata.resourceVersion}'
```

It changed. Every mutation increments the resource version. Controllers use this to
detect concurrent modifications without locking.

### Exercise 6 — Compare with Docker Compose

If the Signet Playground stack is available on the host, consider these architectural
differences:

| Signet Playground (Compose)                    | Kubernetes equivalent                     |
|------------------------------------------------|-------------------------------------------|
| `depends_on: node: condition: service_healthy` | Readiness probe + init container or retry |
| `restart: unless-stopped`                      | Pod `restartPolicy: Always` (default)     |
| `wallet-setup` exits after success             | A Job with `restartPolicy: Never`         |
| Named volume `node_data`                       | PersistentVolumeClaim                     |
| `ports: "127.0.0.1:8080:8080"`                 | Service (ClusterIP) + Ingress or NodePort |
| Shared `cookie_dir` volume                     | A volume mounted into multiple Pods       |

This table is a preview. Each row becomes a full exercise in later chapters.

---

## Failure scenarios

### Scenario 1 — API server unreachable

The API server is the single gateway to the cluster. Simulate losing access by
pointing `kubectl` at a wrong endpoint:

```bash
# Save the current server URL
REAL_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# Point at a non-existent address
kubectl config set-cluster signet-lab --server=https://127.0.0.1:1

# Try any command
kubectl get nodes
```

Every command fails because the API server is unreachable. However, containers already
running on the node **continue to run** — the kubelet keeps them alive independently.
The cluster is not "down" in the sense that workloads stop; it is unmanageable.

Restore:

```bash
kubectl config set-cluster signet-lab --server="$REAL_SERVER"
kubectl get nodes   # works again
```

**Lesson:** the API server is the control plane, not the data plane. Losing it prevents
management but does not kill running Pods.

### Scenario 2 — Killing a controller's input

Scale the Deployment to 5, then immediately delete the ReplicaSet:

```bash
kubectl scale deployment hello --replicas=5
kubectl delete replicaset -l app=hello
```

Watch what happens (in Terminal 1 with `kubectl get pods -w`):

1. The existing Pods enter `Terminating` (the ReplicaSet that owned them is gone).
2. The Deployment controller notices its ReplicaSet is missing and creates a new one.
3. The new ReplicaSet creates 5 fresh Pods.

The system converges back to the desired state. The Deployment controller does not
panic — it simply runs its reconciliation loop and re-creates what is missing.

**Lesson:** controllers are resilient to partial state loss. Deleting intermediate objects
triggers re-creation, not permanent damage — as long as the top-level desired state
(the Deployment) still exists.

---

## Cleanup

Remove the resources created during the workshop:

```bash
kubectl delete deployment hello
```

Verify nothing is left:

```bash
kubectl get all
```

The `kubernetes` Service in the default namespace is a system resource and should remain.

---

## Knowledge check

1. You run `kubectl apply -f deployment.yaml` and the command succeeds. Does that mean
   your containers are running?
2. What is the only component that reads from and writes to etcd?
3. A Pod is running on a node. The API server goes down. What happens to the Pod?
4. Name the four steps of a reconciliation loop.
5. You delete a Pod managed by a ReplicaSet. What happens and why?
6. The Signet Playground stack uses `depends_on` conditions to start services in order.
   Why does Kubernetes not offer an equivalent mechanism?
7. What does `resourceVersion` track, and which components use it?

---

## Summary

Kubernetes is a declarative system built on reconciliation loops. You describe the
desired state; controllers continuously work to make actual state match it. The control
plane (API server, etcd, scheduler, controller manager) stores and coordinates state,
while the kubelet on each node runs the workloads. Understanding this loop — observe,
compare, act, report — is the foundation every later chapter builds on.

Chapter 03 focuses on the Pod: the smallest deployable unit and the object the kubelet
actually runs.
