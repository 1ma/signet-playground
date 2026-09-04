# Chapter 06. Services and cluster networking

## Learning objectives

After completing this chapter you should be able to:

- Explain why Pod IPs are insufficient for reliable communication between workloads.
- Describe what a Service is and how it provides a stable network identity.
- Compare ClusterIP, NodePort, LoadBalancer, ExternalName, and headless Services.
- Distinguish container ports, Service ports, target ports, and node ports.
- Use cluster DNS to connect workloads by name.
- Inspect endpoints and EndpointSlices to diagnose broken connectivity.
- Troubleshoot the most common Service configuration mistakes.

---

## The problem with Pod IPs

Every Pod gets its own cluster-internal IP address. You can see it with:

```bash
kubectl get pods -o wide
```

But Pod IPs are **ephemeral**. When a Pod is replaced (crash, rollout, scaling), the new
Pod gets a different IP. If workload A talks to workload B by IP, it breaks every time B
is replaced.

```text
Before rollout:        After rollout:
Pod B  10.244.0.5      Pod B  10.244.0.9    ← different IP
  ▲                      ▲
  │                      │
Pod A → 10.244.0.5     Pod A → 10.244.0.5   ← still targeting old IP
         (works)                 (broken)
```

Services solve this by providing a **stable virtual IP and DNS name** that automatically
routes to whichever Pods are currently running and ready.

---

## What is a Service

A Service is a Kubernetes resource that defines a stable network endpoint for a set of
Pods. It uses a **label selector** to find its backend Pods, just like a ReplicaSet uses
a selector to find its Pods.

```text
                     Service
                   ┌────────────┐
                   │  web-svc   │
  client ─────►    │ 10.96.0.42 │  ─────► Pod A (10.244.0.5)
                   │  :80       │  ─────► Pod B (10.244.0.6)
                   │            │  ─────► Pod C (10.244.0.7)
                   └────────────┘
                   DNS: web-svc.default.svc.cluster.local
```

The Service:

1. Gets a **virtual IP** (ClusterIP) that never changes for the life of the Service.
2. Gets a **DNS name** that resolves to that virtual IP.
3. Tracks which Pods match its selector and are **ready** (passing their readiness probe).
4. Distributes traffic across the matching Pods.

Clients connect to the Service by DNS name or ClusterIP. They never need to know
individual Pod IPs.

---

## Service types

### ClusterIP (default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: ClusterIP        # default, can be omitted
  selector:
    app: web
  ports:
    - port: 80            # the port the Service listens on
      targetPort: 80      # the port on the Pod to forward to
      protocol: TCP
```

The Service gets a virtual IP from the cluster's Service CIDR (e.g., `10.96.0.0/12`).
This IP is reachable **only from inside the cluster** — other Pods, Jobs, and init
containers can reach it, but nothing outside the cluster can.

This is the right type for internal communication between workloads.

### NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080     # optional — if omitted, a random port in 30000-32767 is assigned
```

A NodePort Service exposes the application on a static port on **every node's IP**. It
also creates a ClusterIP automatically — NodePort is a superset of ClusterIP.

```text
External client ──► <any-node-ip>:30080 ──► Service ──► Pod
```

In Minikube, you reach the node IP with `minikube ip --profile signet-lab` or use
`minikube service --url`.

NodePort is useful for local development and lab environments. In production, a
LoadBalancer or Ingress is preferred.

### LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

In a cloud environment, this provisions an external load balancer (AWS ELB, GCP LB,
etc.) that forwards traffic to the NodePorts. In Minikube, it stays in `Pending` unless
you run `minikube tunnel`.

LoadBalancer is a superset of NodePort, which is a superset of ClusterIP:

```text
LoadBalancer ⊃ NodePort ⊃ ClusterIP
```

### ExternalName

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

No selector, no ClusterIP, no proxying. It creates a **CNAME DNS record** that points
`external-db.default.svc.cluster.local` to `db.example.com`. Useful for referencing
external services through the same DNS-based discovery mechanism.

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None         # this makes it headless
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

Setting `clusterIP: None` disables the virtual IP. DNS queries for the Service name
return the **individual Pod IPs** instead of a single virtual IP. No load balancing
happens at the Service level — the client gets all Pod IPs and decides which to use.

Headless Services are essential for StatefulSets (Chapter 11), where each Pod needs a
stable, individual DNS name. They are also used by clients that implement their own
load balancing or need to discover all endpoints.

```text
Regular Service:                  Headless Service:
  nslookup web-svc                  nslookup web-headless
  → 10.96.0.42 (virtual IP)        → 10.244.0.5
                                    → 10.244.0.6
                                    → 10.244.0.7
```

---

## Ports terminology

This is one of the most confusing areas for newcomers. Four different "port" concepts
coexist:

```text
External client
    │
    │  nodePort (30080)
    ▼
  Node
    │
    │  port (80) ── the Service's port
    ▼
  Service (ClusterIP)
    │
    │  targetPort (8080) ── the container's port
    ▼
  Pod
    │
    │  containerPort (8080) ── documentation only
    ▼
  Container process listening on 8080
```

| Name            | Where it lives                       | Who connects to it                             |
|-----------------|--------------------------------------|------------------------------------------------|
| `containerPort` | Pod spec                             | Informational — does not open anything         |
| `targetPort`    | Service spec                         | The Service forwards traffic here (to the Pod) |
| `port`          | Service spec                         | Other Pods connect to the Service on this port |
| `nodePort`      | Service spec (NodePort/LoadBalancer) | External clients connect here                  |

A common pattern: the Service `port` is 80 (conventional HTTP) but the `targetPort` is
8080 (what the application actually listens on). The Service translates between them.

If `targetPort` is omitted, it defaults to the value of `port`.

### Named ports

You can use names instead of numbers for `targetPort`:

```yaml
# In the Pod spec:
ports:
  - name: http
    containerPort: 8080

# In the Service spec:
ports:
  - port: 80
    targetPort: http    # resolves to whatever "http" is in the Pod spec
```

This decouples the Service from the specific port number. If the application changes
from 8080 to 9090, you only update the Pod spec — the Service still targets `http`.

---

## Cluster DNS

Every Service gets a DNS entry automatically. The format is:

```text
<service-name>.<namespace>.svc.cluster.local
```

For a Service called `web-svc` in the `default` namespace:

```text
web-svc.default.svc.cluster.local    # fully qualified
web-svc.default                      # without the cluster suffix
web-svc                              # within the same namespace
```

Pods in the same namespace can use just the Service name (`web-svc`). Pods in a
different namespace must include the namespace (`web-svc.default`).

### How DNS resolution works

CoreDNS runs as a Deployment in `kube-system` and watches the API server for Service
changes. When a Pod does a DNS lookup, the request goes to CoreDNS (configured via
`/etc/resolv.conf` inside every Pod), which resolves Service names to ClusterIPs.

```bash
# From inside any Pod:
nslookup web-svc
# Returns the ClusterIP

nslookup web-headless
# Returns individual Pod IPs (for headless Services)
```

### SRV records

Services also get SRV records for named ports:

```text
_http._tcp.web-svc.default.svc.cluster.local
```

SRV records include both the port number and the target hostname, which some service
discovery clients use for automatic port detection.

---

## Endpoints and EndpointSlices

When a Service has a selector, Kubernetes automatically creates **EndpointSlice** objects
that list the IPs and ports of all matching Pods that are ready.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web-svc
kubectl describe endpointslice -l kubernetes.io/service-name=web-svc
```

Each entry in an EndpointSlice has:

- The Pod IP
- The target port
- A `ready` condition (only Pods passing readiness probes are included)
- A reference to the Pod (name, UID)

When a Pod fails its readiness probe, it is removed from the EndpointSlice. When it
passes again, it is added back. The Service never sends traffic to an unready Pod.

### Why this matters for debugging

If traffic is not reaching your Pods, check the EndpointSlice:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web-svc -o yaml
```

Common problems:

| Symptom                                 | Cause                                                            |
|-----------------------------------------|------------------------------------------------------------------|
| EndpointSlice has 0 entries             | No Pods match the selector, or all Pods are unready              |
| EndpointSlice does not exist            | The Service selector is wrong or no Pods exist with those labels |
| Pod IP is in the list but traffic fails | Wrong `targetPort` — the Pod is not listening on that port       |

---

## Workshop

### Exercise 1 — Create a backend and expose it

Create a simple backend Deployment and Service:

```yaml
# examples/06-services/backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    chapter: "06"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: httpd
          image: httpd:2.4
          ports:
            - name: http
              containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  labels:
    chapter: "06"
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: http
```

```bash
kubectl apply -f examples/06-services/backend.yaml
kubectl get service backend-svc
kubectl get endpointslices -l kubernetes.io/service-name=backend-svc
```

Verify the EndpointSlice has 3 entries (one per Pod).

### Exercise 2 — Connect a frontend by DNS

Create a frontend that fetches from the backend using the Service DNS name:

```yaml
# examples/06-services/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    chapter: "06"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: curl
          image: curlimages/curl:8.11.0
          command: ["sleep", "3600"]
```

```bash
kubectl apply -f examples/06-services/frontend.yaml
kubectl wait --for=condition=Ready pod -l app=frontend --timeout=60s
```

From the frontend Pod, reach the backend by DNS:

```bash
# By Service name only (same namespace)
kubectl exec deploy/frontend -- curl -s http://backend-svc

# Fully qualified
kubectl exec deploy/frontend -- curl -s http://backend-svc.default.svc.cluster.local

# Check DNS resolution
kubectl exec deploy/frontend -- nslookup backend-svc
```

All three work. The first form is the most common — within the same namespace, the
short name is sufficient.

### Exercise 3 — Observe load balancing

Make multiple requests and observe traffic being distributed:

```bash
for i in $(seq 1 10); do
  kubectl exec deploy/frontend -- curl -s http://backend-svc | head -1
done
```

The responses come from different backend Pods. To see which Pod served each request,
check the backend logs:

```bash
kubectl logs -l app=backend --prefix --tail=5
```

The `--prefix` flag prepends the Pod name to each log line, showing the distribution.

### Exercise 4 — NodePort exposure

Expose the backend outside the cluster:

```yaml
# examples/06-services/nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
  labels:
    chapter: "06"
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: http
      nodePort: 30080
```

```bash
kubectl apply -f examples/06-services/nodeport.yaml
```

Access from the host:

```bash
minikube service backend-nodeport --profile signet-lab --url
# Use the returned URL:
curl -s http://<minikube-ip>:30080
```

### Exercise 5 — Headless Service and DNS

Create a headless Service for the same backend:

```yaml
# examples/06-services/headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
  labels:
    chapter: "06"
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: http
```

```bash
kubectl apply -f examples/06-services/headless.yaml
```

Compare DNS responses:

```bash
# Regular Service — returns one ClusterIP
kubectl exec deploy/frontend -- nslookup backend-svc

# Headless Service — returns individual Pod IPs
kubectl exec deploy/frontend -- nslookup backend-headless
```

The headless response should list 3 IPs (one per backend Pod). Verify these match the
Pod IPs:

```bash
kubectl get pods -l app=backend -o custom-columns=NAME:.metadata.name,IP:.status.podIP
```

### Exercise 6 — Port mapping

Create a Deployment where the container listens on a non-standard port, and use the
Service to map it to port 80:

```yaml
# examples/06-services/port-mapping.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-port
  labels:
    chapter: "06"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom-port
  template:
    metadata:
      labels:
        app: custom-port
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          # nginx listens on 80 by default, but imagine it was 8080
          ports:
            - name: http
              containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: custom-port-svc
  labels:
    chapter: "06"
spec:
  selector:
    app: custom-port
  ports:
    - port: 8080          # clients connect to the Service on 8080
      targetPort: http    # forwards to the Pod's "http" port (80)
```

```bash
kubectl apply -f examples/06-services/port-mapping.yaml
kubectl exec deploy/frontend -- curl -s http://custom-port-svc:8080
```

The client connects to port 8080, but the traffic arrives at the container on port 80.
The Service handles the translation.

---

## Failure scenarios

### Scenario 1 — Selector mismatch

```yaml
# examples/06-services/broken-selector.yaml
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
  labels:
    chapter: "06"
spec:
  selector:
    app: frontend-typo     # does not match any Pod
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f examples/06-services/broken-selector.yaml
kubectl get endpointslices -l kubernetes.io/service-name=broken-svc
```

The EndpointSlice exists but has **0 endpoints**. Traffic to `broken-svc` goes nowhere.

Diagnose:

```bash
# Check what the Service is selecting
kubectl get svc broken-svc -o jsonpath='{.spec.selector}'
# {"app":"frontend-typo"}

# Check what Pods have that label
kubectl get pods -l app=frontend-typo
# No resources found

# Find the right label
kubectl get pods --show-labels | grep frontend
# app=frontend
```

Fix the selector to `app: frontend` and apply again.

**Lesson:** a Service with no endpoints does not error — it silently drops all traffic.
Always verify endpoints when connectivity fails.

### Scenario 2 — Wrong targetPort

```yaml
# examples/06-services/wrong-port.yaml
apiVersion: v1
kind: Service
metadata:
  name: wrong-port-svc
  labels:
    chapter: "06"
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 9999     # backend does not listen on 9999
```

```bash
kubectl apply -f examples/06-services/wrong-port.yaml
kubectl get endpointslices -l kubernetes.io/service-name=wrong-port-svc
```

The EndpointSlice **has entries** (the Pods match the selector), but requests fail:

```bash
kubectl exec deploy/frontend -- curl -s --connect-timeout 3 http://wrong-port-svc
# Connection refused or timeout
```

The Pods are reachable, but nothing is listening on port 9999. The EndpointSlice does not
validate that a Pod actually listens on the target port — it only checks that the Pod
exists and is ready.

**Lesson:** EndpointSlice entries do not guarantee connectivity. If endpoints exist but
requests fail, check whether the Pod is actually listening on the `targetPort`.

### Scenario 3 — Cross-namespace Service discovery

```bash
# Create a namespace and a Service inside it
kubectl create namespace other-ns
kubectl create deployment -n other-ns isolated --image=httpd:2.4
kubectl expose deployment -n other-ns isolated --port=80
```

Try to reach it from the default namespace by short name:

```bash
kubectl exec deploy/frontend -- curl -s --connect-timeout 3 http://isolated
# Fails — "isolated" resolves only within other-ns
```

Use the fully qualified name:

```bash
kubectl exec deploy/frontend -- curl -s http://isolated.other-ns.svc.cluster.local
# Or shorter:
kubectl exec deploy/frontend -- curl -s http://isolated.other-ns
```

**Lesson:** short Service names only resolve within the same namespace. Cross-namespace
communication requires at least `<service>.<namespace>`.

Clean up:

```bash
kubectl delete namespace other-ns
```

---

## Signet Playground preview

The Signet Playground stack uses Docker Compose service names for inter-container
communication (`node`, `fulcrum`, `mariadb`, `valkey`). In Kubernetes, these become
Services:

| Compose service name | Kubernetes Service                  | Type                                                                                                 | Why                                    |
|----------------------|-------------------------------------|------------------------------------------------------------------------------------------------------|----------------------------------------|
| `node`               | `node` (ClusterIP)                  | Internal RPC access for wallet-setup, miner, frigate, fulcrum, mempool, faucet                       | All RPC clients connect to `node:8332` |
| `fulcrum`            | `fulcrum` (ClusterIP + NodePort)    | ClusterIP for internal clients (frigate, mempool); NodePort for external wallets (Sparrow, Electrum) | TCP 50001 needs host exposure          |
| `mariadb`            | `mariadb` (ClusterIP)               | Database for mempool backend                                                                         | Internal only                          |
| `valkey`             | `valkey` (ClusterIP)                | Cache for faucet                                                                                     | Internal only                          |
| `mempool-web`        | `mempool-web` (NodePort or Ingress) | Browser-facing frontend                                                                              | Needs host exposure                    |
| `faucet`             | `faucet` (NodePort or Ingress)      | Browser-facing faucet                                                                                | Needs host exposure                    |

The internal Services use ClusterIP — no reason to expose RPC, database, or cache ports
outside the cluster. Fulcrum is a dual case: internal clients (frigate, mempool) reach it
via ClusterIP, but external wallets like Sparrow need a NodePort to connect over the
Electrum protocol (TCP 50001). The browser-facing applications need external exposure,
which Chapter 13 (Ingress) will handle properly.

The DNS names in Kubernetes (`node.default.svc.cluster.local`) replace Docker's internal
DNS (`node` on the Compose network). Within the same namespace, the short form (`node`)
works identically — application configuration does not need to change.

---

## Cleanup

```bash
kubectl delete deployment,service,pod -l chapter=06
```

Verify:

```bash
kubectl get all
```

---

## Knowledge check

1. Why can you not rely on Pod IPs for communication between workloads?
2. A Service has 3 backend Pods. One fails its readiness probe. How many endpoints does
   the Service have, and what happens to traffic for that Pod?
3. What is the difference between a Service's `port` and its `targetPort`?
4. You create a ClusterIP Service. Can you reach it from your laptop? What Service type
   would you use instead?
5. What does a headless Service (`clusterIP: None`) return when you do a DNS lookup?
   When is this useful?
6. You run `kubectl exec` into a Pod and `curl http://my-svc` works. You try
   `curl http://my-svc` from a different namespace and it fails. Why?
7. Traffic to a Service times out. The EndpointSlice has 3 entries. What is the most
   likely cause?
8. In the Signet Playground, why should the `mariadb` Service be ClusterIP and not
   NodePort?

---

## Summary

Services give Pods a stable network identity — a virtual IP and a DNS name that survives
Pod replacement. ClusterIP handles internal traffic, NodePort and LoadBalancer expose
services externally, and headless Services let clients discover individual Pods.
EndpointSlices track which Pods are ready to receive traffic. Cluster DNS lets workloads
find each other by name, with short names within a namespace and qualified names across
namespaces.

Chapter 07 introduces ConfigMaps and Secrets — how to inject configuration into Pods
without baking it into the container image.
