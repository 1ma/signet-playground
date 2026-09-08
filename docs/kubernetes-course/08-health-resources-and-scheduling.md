# Chapter 08. Health, resources, and scheduling

## Learning objectives

After completing this chapter you should be able to:

- Configure startup, liveness, and readiness probes correctly and explain the
  consequences of getting each one wrong.
- Set CPU and memory requests and limits with an understanding of what the numbers mean.
- Describe the three QoS classes and how they influence eviction order.
- Read scheduling decisions in Events and explain why a Pod is Pending.
- Introduce node selectors, affinity, taints, and tolerations conceptually.
- Tune a workload to roll out reliably under constrained resources.
- Identify appropriate probes, requests, and limits for Signet Playground services.

---

## Probes revisited

Chapter 3 introduced probes briefly. Now that you have Deployments, Services, and
configuration in place, probes become critical — they control whether traffic reaches
your Pods and whether unhealthy containers get restarted.

### The three probe types

| Probe         | Question it answers            | Failure action                     | When it runs                           |
|---------------|--------------------------------|------------------------------------|----------------------------------------|
| **startup**   | Has the app finished starting? | Kills and restarts the container   | Only during startup, before the others |
| **liveness**  | Is the app still alive?        | Kills and restarts the container   | Continuously after startup succeeds    |
| **readiness** | Can the app serve traffic?     | Removes Pod from Service endpoints | Continuously after startup succeeds    |

The relationship:

```text
Pod created
  │
  ▼
startup probe runs (all others disabled)
  │
  ├── succeeds → liveness + readiness probes start
  │
  └── fails (after failureThreshold) → container killed → restart
```

### Probe mechanisms

All three probes support the same four mechanisms:

```yaml
# HTTP GET — most common for web services
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 2

# TCP socket — good when no HTTP endpoint exists
livenessProbe:
  tcpSocket:
    port: 8332
  periodSeconds: 10

# exec command — run a command inside the container
readinessProbe:
  exec:
    command: ["test", "-f", "/var/tmp/.cookie"]
  periodSeconds: 5

# gRPC — for gRPC services (K8s 1.27+)
readinessProbe:
  grpc:
    port: 50051
```

### Probe parameters

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0     # wait before first probe (default 0)
  periodSeconds: 2           # how often to probe (default 10)
  timeoutSeconds: 1          # max time to wait for a response (default 1)
  successThreshold: 1        # consecutive successes to be considered up (default 1)
  failureThreshold: 30       # consecutive failures before action (default 3)
```

The **startup budget** is `failureThreshold × periodSeconds`. With the values above:
30 × 2 = 60 seconds for the application to start. After that, the container is killed.

### Getting probes wrong

Probes are the most common source of unnecessary restarts and outages. The mistakes:

**No startup probe + aggressive liveness probe:**

```yaml
# DANGEROUS for slow-starting apps
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 3
  failureThreshold: 3
```

If the app takes 20 seconds to start, this kills it at second 14 (5 + 3×3). The
container restarts, takes 20 seconds again, gets killed again → CrashLoopBackOff.

Fix: add a startup probe with a generous budget and keep the liveness probe for after
startup:

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 2
# 60 second budget for startup

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
# after startup, kill if unresponsive for 30 seconds
```

**Liveness probe that depends on external services:**

```yaml
# DANGEROUS — kills your Pod when the database is down
livenessProbe:
  httpGet:
    path: /healthz    # this endpoint checks DB connectivity
    port: 8080
```

If `/healthz` queries the database and the database is temporarily down, the liveness
probe fails, Kubernetes kills the container, it restarts, checks the database again, it
is still down, killed again → cascade failure. The database outage took down an
unrelated service.

Liveness probes should check whether **the process itself** is healthy (can it respond
at all?), not whether its dependencies are available. Use readiness probes for dependency
checks — they remove the Pod from the Service without killing it, so the Pod is ready to
serve again as soon as the dependency recovers.

**Readiness probe identical to liveness probe:**

If the readiness check is the same as liveness, a transient failure both removes the Pod
from the Service AND kills it. The readiness removal would have been enough — killing
creates unnecessary restarts and delays recovery.

### Recommended pattern

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 2

livenessProbe:
  httpGet:
    path: /healthz          # lightweight "am I alive" check
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready            # can check dependencies, cache warmup, etc.
    port: 8080
  periodSeconds: 5
  failureThreshold: 3
```

The key insight: `/healthz` and `/ready` are different endpoints. `/healthz` answers "is
the process running and responsive" (cheap, no external calls). `/ready` answers "can I
serve user traffic right now" (may check database, cache, upstream services).

---

## Resource requests and limits

### CPU and memory

Every container can declare:

- **requests** — the minimum resources the scheduler guarantees.
- **limits** — the maximum resources the container is allowed to consume.

```yaml
resources:
  requests:
    cpu: 100m           # 100 millicores = 0.1 CPU
    memory: 128Mi       # 128 mebibytes
  limits:
    cpu: 500m           # can burst to 0.5 CPU
    memory: 256Mi       # hard cap — exceeding this triggers OOMKill
```

### CPU units

CPU is measured in **millicores** (m):

| Value    | Meaning                        |
|----------|--------------------------------|
| `1`      | 1 full CPU core                |
| `500m`   | Half a CPU core                |
| `100m`   | 10% of a CPU core              |
| `2`      | 2 full CPU cores               |

CPU limits are **throttled**, not killed. If a container tries to use more CPU than its
limit, the kernel CFS scheduler slows it down — the process still runs, just slower.

### Memory units

| Suffix | Meaning                          |
|--------|----------------------------------|
| `Ki`   | Kibibytes (1024 bytes)           |
| `Mi`   | Mebibytes (1024² bytes)          |
| `Gi`   | Gibibytes (1024³ bytes)          |
| `K`    | Kilobytes (1000 bytes)           |
| `M`    | Megabytes (1000² bytes)          |
| `G`    | Gigabytes (1000³ bytes)          |

Always use the binary units (`Mi`, `Gi`) — they match what `free`, `top`, and
`kubectl top` report.

Memory limits are **hard**. If a container tries to allocate more memory than its limit,
the kernel OOM killer terminates the process immediately. The container restarts with
reason `OOMKilled`.

### What the scheduler does with requests

The scheduler uses **requests** (not limits) to decide where to place a Pod:

```text
Node capacity: 4 CPU, 8 Gi memory

Pod A requests: 1 CPU, 2 Gi     ← scheduled, 3 CPU / 6 Gi remaining
Pod B requests: 2 CPU, 3 Gi     ← scheduled, 1 CPU / 3 Gi remaining
Pod C requests: 2 CPU, 1 Gi     ← PENDING — not enough CPU
```

Pod C stays `Pending` even if the node's actual usage is low — the scheduler looks at
the sum of requests, not actual consumption. This prevents overcommitting guarantees.

### Requests without limits

If you set requests but no limits, the container can burst beyond its request up to
whatever the node has available. This is fine for CPU (throttling is graceful) but risky
for memory (an unbounded container can consume all node memory and trigger evictions).

A common practice:

- **CPU**: set requests, skip limits (let it burst — throttling handles the rest).
- **Memory**: set both requests and limits to the same value (prevents OOM surprises).

### What happens without any resource declarations

If you omit both requests and limits, the Pod gets the lowest scheduling priority and is
the first to be evicted under memory pressure. The scheduler places it on any node that
has not explicitly reserved all its capacity. This is the default behavior for all the
Pods we have created so far in the course.

---

## QoS classes

Kubernetes assigns every Pod a **Quality of Service** class based on its resource
declarations. This class determines eviction priority when the node runs out of memory:

| QoS class      | Condition                                                | Eviction priority       |
|----------------|----------------------------------------------------------|-------------------------|
| **Guaranteed** | Every container has requests = limits for CPU and memory | Last (most protected)   |
| **Burstable**  | At least one container has a request or limit set        | Middle                  |
| **BestEffort** | No requests or limits set on any container               | First (least protected) |

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

Under memory pressure, the kubelet evicts BestEffort Pods first, then Burstable (in
order of how much they exceed their requests), and Guaranteed Pods last.

### Practical implication

All the Pods in this course so far have been **BestEffort** — no resource declarations.
In a shared cluster, this means they would be the first to be evicted. For production,
every Pod should have at least requests defined to get Burstable or Guaranteed status.

---

## Scheduling fundamentals

### Why is my Pod Pending?

When a Pod is `Pending`, the scheduler cannot find a suitable node. Common causes:

```bash
kubectl describe pod <pending-pod> | grep -A5 Events
```

| Event message                                                         | Cause                                        | Fix                                      |
|-----------------------------------------------------------------------|----------------------------------------------|------------------------------------------|
| `Insufficient cpu`                                                    | Sum of CPU requests exceeds node capacity    | Reduce requests or add nodes             |
| `Insufficient memory`                                                 | Sum of memory requests exceeds node capacity | Reduce requests or add nodes             |
| `0/1 nodes are available: 1 node(s) had untolerated taint`            | Node has a taint the Pod does not tolerate   | Add a toleration or use a different node |
| `no persistent volumes available`                                     | No PV matches the PVC                        | Create a matching PV (Chapter 9)         |
| `0/1 nodes are available: 1 node(s) didn't match Pod's node affinity` | Node labels do not match                     | Fix node selector or label the node      |

### Node selectors

The simplest way to control where a Pod runs:

```yaml
spec:
  nodeSelector:
    disktype: ssd
    kubernetes.io/arch: amd64
```

The Pod only runs on nodes that have **all** the specified labels. If no node matches,
the Pod stays Pending.

```bash
# Label a node
kubectl label node minikube disktype=ssd

# See node labels
kubectl get nodes --show-labels
```

### Affinity and anti-affinity (conceptual)

Node affinity is a more expressive version of `nodeSelector`:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd", "nvme"]
```

Supports operators like `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`. Also has
a `preferredDuringSchedulingIgnoredDuringExecution` variant that is a soft preference
rather than a hard requirement.

**Pod anti-affinity** spreads replicas across nodes:

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
```

This tells the scheduler: "prefer not to schedule this Pod on a node that already runs
a Pod with `app: web`". Useful for high availability — if a node goes down, not all
replicas go with it.

In Minikube with a single node, anti-affinity has no effect — there is nowhere else to
schedule. It becomes relevant in multi-node clusters.

### Taints and tolerations (conceptual)

Taints are the inverse of node selectors — instead of a Pod choosing a node, a node
**repels** Pods unless they explicitly tolerate the taint:

```bash
# Taint a node
kubectl taint nodes minikube dedicated=gpu:NoSchedule
```

Only Pods with a matching toleration can be scheduled on this node:

```yaml
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
```

Common use cases:

- Dedicated nodes for specific workloads (GPU, high-memory).
- The control plane node is tainted by default — that is why user Pods do not run on it.
- `NoExecute` taints also evict already-running Pods that do not tolerate them.

---

## Workshop

### Exercise 1 — Observe QoS classes

Create three Pods with different resource configurations:

```yaml
# examples/08-health/qos-classes.yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
  labels:
    chapter: "08"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 64Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
  labels:
    chapter: "08"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 200m
          memory: 128Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
  labels:
    chapter: "08"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f examples/08-health/qos-classes.yaml
kubectl get pods -l chapter=08 -o custom-columns=\
NAME:.metadata.name,\
QOS:.status.qosClass,\
STATUS:.status.phase
```

You should see `Guaranteed`, `Burstable`, and `BestEffort` in the QOS column.

### Exercise 2 — Probes in action

Deploy a web server with all three probes:

```yaml
# examples/08-health/probes-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probes-demo
  labels:
    chapter: "08"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probes-demo
  template:
    metadata:
      labels:
        app: probes-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 10
            periodSeconds: 2
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 64Mi
```

```bash
kubectl apply -f examples/08-health/probes-demo.yaml
kubectl wait --for=condition=Ready pod -l app=probes-demo --timeout=60s
```

Watch the probe events:

```bash
kubectl describe pod -l app=probes-demo | grep -A20 Events
```

Now break the liveness probe by removing the default page:

```bash
kubectl exec deploy/probes-demo -- rm /usr/share/nginx/html/index.html
```

Watch what happens:

```bash
kubectl get pods -l app=probes-demo -w
```

The liveness probe fails (404), the container is killed, and on restart nginx recreates
the default page. The Pod self-heals.

### Exercise 3 — Readiness probe removing traffic

```yaml
# examples/08-health/readiness-gate.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
  labels:
    chapter: "08"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            periodSeconds: 3
            failureThreshold: 1
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-svc
  labels:
    chapter: "08"
spec:
  selector:
    app: readiness-demo
  ports:
    - port: 80
```

```bash
kubectl apply -f examples/08-health/readiness-gate.yaml
kubectl wait --for=condition=Ready pod -l app=readiness-demo --timeout=60s
```

All Pods fail readiness (404 on `/ready`) — check endpoints:

```bash
kubectl get endpoints readiness-svc
# No endpoints — all Pods are unready
```

Fix one Pod by creating the readiness path:

```bash
POD=$(kubectl get pods -l app=readiness-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- sh -c 'echo ok > /usr/share/nginx/html/ready'
```

```bash
kubectl get endpoints readiness-svc
# One endpoint appears
```

The Service only routes traffic to the ready Pod. The other two are still running but
receive no traffic.

### Exercise 4 — Resource limits and OOMKill

```yaml
# examples/08-health/oom-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
  labels:
    chapter: "08"
spec:
  containers:
    - name: stress
      image: polinux/stress:1.0.4
      command: ["stress", "--vm", "1", "--vm-bytes", "256M", "--vm-hang", "60"]
      resources:
        requests:
          memory: 64Mi
        limits:
          memory: 128Mi
```

```bash
kubectl apply -f examples/08-health/oom-demo.yaml
kubectl get pod oom-demo -w
```

The container tries to allocate 256 MiB but the limit is 128 MiB. It gets OOMKilled:

```bash
kubectl describe pod oom-demo | grep -A3 "Last State"
# Reason: OOMKilled
```

### Exercise 5 — Scheduling failure

```yaml
# examples/08-health/unschedulable.yaml
apiVersion: v1
kind: Pod
metadata:
  name: too-big
  labels:
    chapter: "08"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: "16"
          memory: 64Gi
```

```bash
kubectl apply -f examples/08-health/unschedulable.yaml
kubectl get pod too-big
# STATUS: Pending

kubectl describe pod too-big | grep -A5 Events
# Insufficient cpu / Insufficient memory
```

The scheduler cannot find a node with 16 CPUs and 64 Gi of memory. In Minikube with 4
CPUs and limited memory, this Pod will never be scheduled.

```bash
kubectl delete pod too-big
```

### Exercise 6 — Tuning requests with metrics-server

Ensure metrics-server is running:

```bash
minikube addons enable metrics-server --profile signet-lab
```

Check actual resource consumption:

```bash
kubectl top nodes
kubectl top pods -l chapter=08
```

Compare the actual usage with the requests you set. This is how you tune resource
declarations — set requests based on observed usage with headroom, not guesses.

```bash
# Example: see how much CPU and memory nginx actually uses
kubectl top pod -l app=probes-demo
# CPU: ~1m    Memory: ~5Mi
# The requests (50m CPU, 32Mi memory) are generous for idle nginx
```

---

## Failure scenarios

### Scenario 1 — Liveness probe kills slow startup

```yaml
# examples/08-health/slow-start.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-start
  labels:
    chapter: "08"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slow-start
  template:
    metadata:
      labels:
        app: slow-start
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command:
            - sh
            - -c
            - "sleep 30 && echo ready && httpd -f -p 8080 -h /tmp"
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 3
            failureThreshold: 3
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
```

```bash
kubectl apply -f examples/08-health/slow-start.yaml
kubectl get pods -l app=slow-start -w
```

The app takes 30 seconds to start. The liveness probe starts at second 5 and kills the
container at second 14 (5 + 3×3). You see repeated restarts.

Fix by adding a startup probe:

```bash
kubectl patch deployment slow-start --type=json -p='[
  {"op": "add", "path": "/spec/template/spec/containers/0/startupProbe", "value": {
    "httpGet": {"path": "/", "port": 8080},
    "failureThreshold": 30,
    "periodSeconds": 2
  }}
]'
```

Watch the Pod start successfully this time.

### Scenario 2 — OOMKill during rolling update

```yaml
# examples/08-health/tight-limits.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tight-limits
  labels:
    chapter: "08"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tight-limits
  template:
    metadata:
      labels:
        app: tight-limits
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: 50m
              memory: 4Mi         # too tight for nginx
            limits:
              memory: 4Mi
```

```bash
kubectl apply -f examples/08-health/tight-limits.yaml
kubectl get pods -l app=tight-limits -w
```

nginx needs ~5-10 MiB to start. With a 4 MiB limit, containers OOMKill repeatedly. A
rolling update with this configuration never completes — new Pods keep crashing while old
Pods stay alive (respecting `maxUnavailable`).

```bash
kubectl describe pod -l app=tight-limits | grep OOMKilled
```

Fix by increasing the limit:

```bash
kubectl patch deployment tight-limits --type=json -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "64Mi"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "32Mi"}
]'
```

### Scenario 3 — Node selector with no matching node

```yaml
# examples/08-health/wrong-selector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: wrong-selector
  labels:
    chapter: "08"
spec:
  nodeSelector:
    gpu: "true"
  containers:
    - name: app
      image: busybox:1.37
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
```

```bash
kubectl apply -f examples/08-health/wrong-selector.yaml
kubectl describe pod wrong-selector | grep -A5 Events
# didn't match Pod's node affinity/selector
```

Fix by labeling the node:

```bash
kubectl label node minikube gpu=true
# Pod gets scheduled immediately
```

Clean up:

```bash
kubectl label node minikube gpu-
```

---

## Signet Playground preview

### Probes

| Service       | Startup probe                                                     | Liveness probe             | Readiness probe                                     |
|---------------|-------------------------------------------------------------------|----------------------------|-----------------------------------------------------|
| `node`        | `exec: test -f /var/tmp/.cookie` (already in Compose healthcheck) | TCP 38332 (RPC port)       | `exec: bitcoin-cli -getinfo` (RPC responds)         |
| `fulcrum`     | `exec: FulcrumAdmin -p 8000 getinfo` (already in Compose)         | TCP 60601                  | `exec: FulcrumAdmin -p 8000 getinfo`                |
| `mariadb`     | `exec: mariadb-admin ping` (already in Compose)                   | `exec: mariadb-admin ping` | `exec: mariadb-admin ping`                          |
| `valkey`      | `exec: valkey-cli ping`                                           | TCP 6379                   | `exec: valkey-cli ping`                             |
| `mempool-api` | HTTP GET / on API port                                            | HTTP GET /                 | HTTP GET / (depends on upstream, use separate path) |
| `mempool-web` | HTTP GET / on 8080                                                | TCP 8080                   | HTTP GET / on 8080                                  |
| `faucet`      | HTTP GET / on 8080                                                | TCP 8080                   | HTTP GET / on 8080                                  |

The existing Compose `healthcheck` directives map naturally to Kubernetes startup probes.
In Kubernetes, we split the concern: the startup check ensures the service is up, the
liveness check detects process-level hangs, and the readiness check confirms the service
can handle traffic.

Notice that `node`'s readiness probe uses `bitcoin-cli -getinfo` rather than just
checking the cookie file. The cookie exists almost immediately, but the node is not truly
ready to serve RPC until it has completed startup — the readiness probe should reflect
functional availability, not just file existence.

### Resource estimates

| Service       | CPU request | Memory request | Memory limit | Notes                                 |
|---------------|-------------|----------------|--------------|---------------------------------------|
| `node`        | 500m        | 512Mi          | 1Gi          | CPU-intensive during IBD and mining   |
| `fulcrum`     | 200m        | 256Mi          | 512Mi        | Indexing is memory-hungry initially   |
| `mariadb`     | 100m        | 128Mi          | 256Mi        | Light usage with a small signet chain |
| `valkey`      | 50m         | 32Mi           | 64Mi         | Tiny dataset for faucet rate limiting |
| `mempool-api` | 100m        | 128Mi          | 256Mi        | Varies with chain size                |
| `mempool-web` | 50m         | 32Mi           | 64Mi         | Static file server                    |
| `faucet`      | 50m         | 32Mi           | 64Mi         | Low traffic                           |
| `miner`       | 100m        | 64Mi           | 128Mi        | Periodic bursts every MAX_INTERVAL    |

These are starting estimates. The correct approach is to deploy with generous limits,
observe actual usage with `kubectl top`, and adjust. The Minikube lab has constrained
resources (4 CPUs, limited memory), so the full stack may need tuning to fit.

---

## Cleanup

```bash
kubectl delete deployment,service,pod -l chapter=08
```

If you labeled the Minikube node during the exercises:

```bash
kubectl label node minikube disktype- gpu-
```

Verify:

```bash
kubectl get all
```

---

## Knowledge check

1. What is the difference between a liveness probe and a readiness probe in terms of
   what happens when each one fails?
2. Why should a liveness probe never check external dependencies like a database?
3. A Deployment has `replicas: 3`. You set CPU requests to `2` per Pod. Your Minikube
   node has 4 CPUs. How many Pods can be scheduled, and what happens to the rest?
4. What is the difference between how Kubernetes enforces CPU limits versus memory limits?
5. You deploy a Pod with `resources: {}` (no requests or limits). What QoS class does it
   get, and why is this risky in a shared cluster?
6. A Pod is stuck in `Pending`. The Events show `Insufficient memory`. The node has 4 Gi
   free according to `kubectl top node`. Why might the Pod still not fit?
7. What is the purpose of a startup probe, and what happens if you rely on
   `initialDelaySeconds` instead?
8. In the Signet Playground, why should the Bitcoin node's readiness probe use
   `bitcoin-cli -getinfo` instead of `test -f /var/tmp/.cookie`?

---

## Summary

Probes make workloads self-healing: startup probes protect slow starts, liveness probes
detect hung processes, and readiness probes gate traffic. Resource requests and limits
make workloads predictable: the scheduler uses requests to place Pods, the kernel enforces
limits (throttling for CPU, OOMKill for memory). QoS classes determine eviction priority
under pressure. Node selectors, affinity, and taints control Pod placement beyond simple
resource math.

Chapter 09 introduces persistent storage — how to keep data across Pod replacement using
volumes, PersistentVolumeClaims, and StorageClasses.
