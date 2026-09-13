# Chapter 11. StatefulSets and stable identity

## Learning objectives

After completing this chapter you should be able to:

- Explain what problems StatefulSets solve that Deployments cannot.
- Describe the guarantees StatefulSets provide: stable Pod names, ordered creation and
  deletion, and stable persistent storage.
- Create a headless Service and explain why StatefulSets require one.
- Use volumeClaimTemplates to give each Pod its own PVC automatically.
- Choose between `OrderedReady` and `Parallel` Pod management policies.
- Evaluate whether a given workload genuinely needs a StatefulSet or can use a simpler
  controller.
- Assess StatefulSet suitability for each Signet Playground stateful service.

---

## The problem with Deployments for stateful workloads

Deployments treat Pods as **interchangeable**. When a Pod is replaced, the new Pod gets:

- A new random name (`nginx-7f8c9b-xk4rz` → `nginx-7f8c9b-p2m7q`).
- A new IP address.
- No relationship to the storage the previous Pod used.

This is perfect for stateless web servers — any replica can handle any request. But some
workloads need **identity**:

| Requirement                                           | Why Deployments fail                                    |
|-------------------------------------------------------|---------------------------------------------------------|
| Stable network identity (DNS)                         | Pod names are random; DNS records change on replacement |
| Stable storage per replica                            | No built-in mapping between replica and PVC             |
| Ordered startup and shutdown                          | All replicas start in parallel by default               |
| Predictable scaling (add replica 3, not a random one) | Replica names are random                                |

Databases, distributed key-value stores, consensus systems (etcd, ZooKeeper), and
message brokers (Kafka) all need some combination of these. StatefulSets provide them.

---

## What a StatefulSet guarantees

A StatefulSet manages Pods with **three stable properties** that Deployments lack:

### 1. Stable, predictable Pod names

Pods are named `<statefulset-name>-<ordinal>`, starting from 0:

```text
Deployment Pods:          StatefulSet Pods:
nginx-7f8c9b-xk4rz       web-0
nginx-7f8c9b-p2m7q       web-1
nginx-7f8c9b-a9d3f       web-2
```

If `web-1` is deleted, its replacement is also called `web-1` — same name, same ordinal.

### 2. Stable, persistent storage per Pod

Each Pod gets its own PVC through `volumeClaimTemplates`. The PVC is named
`<template-name>-<statefulset-name>-<ordinal>`:

```text
PVC: data-web-0    →  bound to PV-A    →  mounted by Pod web-0
PVC: data-web-1    →  bound to PV-B    →  mounted by Pod web-1
PVC: data-web-2    →  bound to PV-C    →  mounted by Pod web-2
```

When `web-1` is rescheduled, it remounts `data-web-1` — the same PVC, the same data.
A Deployment has no such mapping: you would have to manage PVCs manually and somehow
bind each replica to its own PVC.

### 3. Ordered, graceful lifecycle

With the default `podManagementPolicy: OrderedReady`:

- **Creation:** `web-0` must be Running and Ready before `web-1` starts. `web-1` must be
  Running and Ready before `web-2` starts.
- **Deletion:** `web-2` is terminated before `web-1`. `web-1` is terminated before
  `web-0`.
- **Updates:** Pods are updated in reverse ordinal order (highest first), one at a time.

This ordering matters when a replica must register with the cluster before the next one
joins — primary/replica databases, consensus protocols, or anything where the first
replica bootstraps the system.

---

## Headless Services

A StatefulSet requires a **headless Service** — a Service with `clusterIP: None`. Unlike
a regular Service, a headless Service does not allocate a virtual IP. Instead, DNS
returns the individual Pod IPs directly.

```text
Regular Service (ClusterIP):
  nslookup my-svc → 10.96.0.15 (single VIP)

Headless Service:
  nslookup my-svc → 10.244.0.5, 10.244.0.6, 10.244.0.7 (one per Pod)
```

The headless Service also creates individual DNS records for each Pod:

```text
web-0.my-svc.default.svc.cluster.local → 10.244.0.5
web-1.my-svc.default.svc.cluster.local → 10.244.0.6
web-2.my-svc.default.svc.cluster.local → 10.244.0.7
```

This is how stable network identity works. A client that needs to talk to a specific
replica uses `web-0.my-svc` — that name always resolves to whichever IP Pod `web-0`
currently has, even after rescheduling.

### Why not a regular Service?

A regular Service load-balances across all Pods. That is fine for stateless workloads
but wrong when a client needs to reach a **specific** replica. A primary database and a
read replica are not interchangeable — the write query must go to the primary.

You can still create a **second**, regular (ClusterIP) Service that points to the same
Pods if you also want load-balanced access for read traffic. The headless Service is
for identity; the regular Service is for generic access.

---

## Anatomy of a StatefulSet

```yaml
# statefulset-demo.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
  labels:
    chapter: "11"
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
      name: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  labels:
    chapter: "11"
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        chapter: "11"
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 64Mi
```

Key elements:

| Field                  | Purpose                                                          |
|------------------------|------------------------------------------------------------------|
| `serviceName`          | Names the headless Service that governs Pod DNS. Required.       |
| `replicas`             | Number of ordered Pods (0, 1, 2, …).                             |
| `volumeClaimTemplates` | One PVC template per volume. Each Pod gets its own PVC instance. |
| `podManagementPolicy`  | `OrderedReady` (default) or `Parallel`.                          |
| `updateStrategy`       | `RollingUpdate` (default) or `OnDelete`.                         |

Note that `serviceName` must match an existing headless Service — the StatefulSet does
not create it. You must define both resources.

---

## Hands-on: StatefulSet lifecycle

### Exercise 1: create and observe

```bash
kubectl apply -f statefulset-demo.yaml
kubectl get statefulsets
kubectl get pods -l app=web --watch
```

**Observe the ordered creation:**

```text
web-0   0/1   Pending       0     0s
web-0   0/1   ContainerCreating   0     1s
web-0   1/1   Running       0     3s
web-1   0/1   Pending       0     0s      ← starts only after web-0 is Ready
web-1   0/1   ContainerCreating   0     1s
web-1   1/1   Running       0     3s
web-2   0/1   Pending       0     0s      ← starts only after web-1 is Ready
web-2   1/1   Running       0     3s
```

Compare this to a Deployment where all 3 Pods start simultaneously.

### Exercise 2: stable DNS identity

```bash
kubectl run dns-test --image=busybox:1.37 --restart=Never --rm -it -- sh -c '
  echo "=== Headless Service ==="
  nslookup web-headless.default.svc.cluster.local

  echo ""
  echo "=== Individual Pod DNS ==="
  nslookup web-0.web-headless.default.svc.cluster.local
  nslookup web-1.web-headless.default.svc.cluster.local
  nslookup web-2.web-headless.default.svc.cluster.local
'
```

Each Pod has its own DNS A record. These names survive Pod deletion and rescheduling.

### Exercise 3: stable storage

Write unique data into each Pod, then delete a Pod and verify the data survives:

```bash
# Write identity into each Pod's PVC
for i in 0 1 2; do
  kubectl exec web-$i -- sh -c "echo 'I am web-$i' > /usr/share/nginx/html/index.html"
done

# Verify
for i in 0 1 2; do
  kubectl exec web-$i -- cat /usr/share/nginx/html/index.html
done
```

Now delete `web-1` and watch it come back with its data:

```bash
kubectl delete pod web-1
kubectl get pods -l app=web --watch
```

Once `web-1` is Running again:

```bash
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
```

It still says "I am web-1". The PVC `data-web-1` was not deleted — only the Pod was.
The replacement Pod with the same ordinal remounted the same PVC.

### Exercise 4: inspect the PVCs

```bash
kubectl get pvc -l app=web
```

```text
NAME         STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-web-0   Bound    pv-xxx     64Mi       RWO            standard       5m
data-web-1   Bound    pv-yyy     64Mi       RWO            standard       5m
data-web-2   Bound    pv-zzz     64Mi       RWO            standard       5m
```

The naming convention `<template-name>-<statefulset-name>-<ordinal>` makes it easy to
identify which PVC belongs to which Pod.

### Exercise 5: scale down and PVC retention

```bash
kubectl scale statefulset web --replicas=2
kubectl get pods -l app=web
kubectl get pvc -l app=web
```

`web-2` is deleted, but `data-web-2` **persists**. This is by design — StatefulSets
never delete PVCs automatically. If you scale back up to 3:

```bash
kubectl scale statefulset web --replicas=3
kubectl exec web-2 -- cat /usr/share/nginx/html/index.html
```

The data is still there. The new `web-2` Pod remounts the existing `data-web-2` PVC.

**This is critical for databases:** scaling down does not destroy data. The PVCs must be
deleted manually if the data is truly no longer needed:

```bash
# Only do this if you actually want to destroy the data
# kubectl delete pvc data-web-2
```

### Exercise 6: ordered deletion

```bash
kubectl delete statefulset web
kubectl get pods -l app=web --watch
```

```text
web-2   1/1   Terminating   0     10m    ← highest ordinal first
web-1   1/1   Terminating   0     10m
web-0   1/1   Terminating   0     10m
```

Check that PVCs survive even StatefulSet deletion:

```bash
kubectl get pvc -l app=web
```

All three PVCs remain. Recreating the StatefulSet (with the same name and
volumeClaimTemplates) would rebind the Pods to their original PVCs.

---

## Pod management policies

### OrderedReady (default)

Pods are created in order (0, 1, 2) and each must be Running and Ready before the next
starts. Deletion is in reverse order. This is required when:

- The first replica bootstraps the cluster (primary election, schema creation).
- Later replicas register with earlier ones (consensus, replication setup).
- Shutdown order matters (drain replicas before the primary).

### Parallel

All Pods are created and deleted simultaneously, like a Deployment. Use this when the
Pods do not depend on each other's readiness but you still need stable names and storage:

```yaml
spec:
  podManagementPolicy: Parallel
```

This is common for workloads that need identity (stable storage per instance) but not
ordering — a pool of independent workers that each process their own queue, or a set of
cache nodes that do not replicate between themselves.

---

## Update strategies

### RollingUpdate (default)

Pods are updated in **reverse ordinal order** (highest first), one at a time. Each Pod
must become Ready before the next is updated.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
```

The `partition` field controls a **canary rollout**: only Pods with ordinal ≥ partition
are updated. Setting `partition: 2` updates only `web-2` while leaving `web-0` and
`web-1` on the old version. Useful for testing an update on a single replica before
rolling it out.

```bash
# Update the image and observe the rollout
kubectl set image statefulset/web nginx=nginx:1.27-alpine
kubectl rollout status statefulset/web
```

### OnDelete

Pods are updated only when manually deleted. This gives you full control over when each
replica transitions:

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

Useful for databases where you want to failover to a replica before updating the primary
— delete the primary Pod manually after the replica is confirmed healthy on the new
version.

---

## StatefulSet vs Deployment: decision guide

Not every stateful application needs a StatefulSet. Use this decision tree:

```text
Does the workload need persistent data?
├── No  → Deployment
└── Yes
    ├── Is there only one replica (singleton)?
    │   ├── Does it need a stable DNS name other Pods address by name?
    │   │   ├── Yes → StatefulSet (1 replica) or Deployment + dedicated Service
    │   │   └── No  → Deployment + PVC
    │   └── (either works — Deployment + PVC is simpler for singletons)
    └── Are there multiple replicas?
        ├── Do replicas need distinct identity (name, storage, DNS)?
        │   ├── Yes → StatefulSet
        │   └── No  → Deployment + shared PVC (if storage allows RWX)
        └── Does startup/shutdown order matter?
            ├── Yes → StatefulSet with OrderedReady
            └── No  → StatefulSet with Parallel (or Deployment if no identity needed)
```

The key question is: **does each replica need its own identity?** If not, a Deployment
with a PVC is simpler and sufficient.

### The singleton trap

A common mistake is deploying every database as a StatefulSet "because it is stateful."
A single-replica PostgreSQL does not need stable ordinal names — there is only one Pod.
It does not need ordered startup — there is only one Pod. It needs persistent storage,
which a Deployment with a PVC already provides.

A StatefulSet for a singleton adds complexity (headless Service, volumeClaimTemplates)
without benefit. Use it only when you plan to scale to multiple replicas, need the
specific DNS naming, or follow a Helm chart convention that standardizes on
StatefulSets for all stateful workloads.

---

## Failure scenarios

### Scenario 1: StatefulSet stuck — Pod stays Pending

```yaml
# fail01-no-storage.yaml
apiVersion: v1
kind: Service
metadata:
  name: stuck-headless
  labels:
    chapter: "11"
spec:
  clusterIP: None
  selector:
    app: stuck-sts
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stuck-sts
  labels:
    chapter: "11"
spec:
  serviceName: stuck-headless
  replicas: 2
  selector:
    matchLabels:
      app: stuck-sts
  template:
    metadata:
      labels:
        app: stuck-sts
        chapter: "11"
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["sh", "-c", "sleep 3600"]
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: nonexistent-class
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f fail01-no-storage.yaml
kubectl get pods -l app=stuck-sts --watch
kubectl get pvc -l app=stuck-sts
```

**Diagnose it:** `stuck-sts-0` stays Pending because its PVC cannot be provisioned —
the StorageClass `nonexistent-class` does not exist. Because of `OrderedReady`,
`stuck-sts-1` never starts. The entire StatefulSet is blocked by the first replica.

```bash
kubectl describe pvc data-stuck-sts-0
```

The Events explain the provisioning failure.

### Scenario 2: Pod identity confusion after manual PVC deletion

```bash
# After running the main demo:
# 1. Scale to 0
kubectl scale statefulset web --replicas=0

# 2. Delete web-1's PVC
kubectl delete pvc data-web-1

# 3. Scale back to 3
kubectl scale statefulset web --replicas=3

# 4. Check web-1's data
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
```

**Diagnose it:** `web-1` now has a **new, empty PVC** — the previous data is gone. The
Pod name is the same, the PVC name is the same, but the underlying PV is different.
This is why PVC deletion in a StatefulSet must be deliberate — there is no "undo."

### Scenario 3: split-brain on forced Pod deletion

In a multi-node cluster, if a node becomes unreachable and you force-delete a
StatefulSet Pod:

```bash
kubectl delete pod web-0 --force --grace-period=0
```

Kubernetes creates a new `web-0` on a healthy node, but the old `web-0` might still be
running on the unreachable node. Now there are **two Pods** with the same identity,
potentially both writing to storage. This is a split-brain condition.

**Prevention:** never force-delete StatefulSet Pods unless you are certain the old Pod is
truly gone (node confirmed down, power fenced). The StatefulSet controller intentionally
refuses to create a replacement while the old Pod's status is unknown — this is a safety
feature, not a bug.

---

## Signet Playground: StatefulSet evaluation

The Signet Playground has five services that store persistent data. For each one,
evaluate whether a StatefulSet adds value over a simpler Deployment + PVC:

| Service   | Replicas | Needs stable DNS?           | Needs ordered startup? | Recommendation       | Reason                                                                    |
|-----------|----------|-----------------------------|------------------------|----------------------|---------------------------------------------------------------------------|
| `node`    | 1        | No — accessed via a Service | No — singleton         | **Deployment + PVC** | Single replica, no identity needed. A Service provides a stable endpoint. |
| `fulcrum` | 1        | No — accessed via a Service | No — singleton         | **Deployment + PVC** | Same reasoning. Fulcrum is a singleton indexer.                           |
| `mariadb` | 1        | No — accessed via a Service | No — singleton         | **Deployment + PVC** | Single replica. If scaled to primary/replica later, reconsider.           |
| `valkey`  | 1        | No — accessed via a Service | No — singleton         | **Deployment + PVC** | Single Valkey instance. No replication, no Sentinel.                      |
| `frigate` | 1        | No — accessed via a Service | No — singleton         | **Deployment + PVC** | Single replica indexer.                                                   |

### The verdict

None of the Signet Playground services currently benefit from a StatefulSet. They are
all **singletons** accessed through a regular Service. A Deployment with a PVC is
simpler, better understood, and sufficient.

This is a common outcome in small-to-medium applications: StatefulSets solve real
problems, but those problems appear when you have **multiple replicas of the same
stateful workload** — a 3-node etcd cluster, a MariaDB primary with read replicas, a
Kafka broker set. A singleton database does not need ordinal naming or ordered startup.

### When would StatefulSets become relevant?

If the Signet Playground evolved to require:

- **MariaDB replication** (primary + replica): a StatefulSet gives each instance a stable
  name (`mariadb-0` as primary, `mariadb-1` as replica), ordered startup (primary
  initializes first), and per-replica storage.
- **Valkey Sentinel or Cluster mode**: each Valkey node needs a stable identity for
  cluster discovery. The Sentinel configuration references nodes by name.
- **Multiple Bitcoin nodes** (one mining, others syncing): each needs its own blockchain
  data directory and a predictable name for peer configuration.

Until then, Deployments with PVCs are the right choice. The capstone chapter (18) will
use Deployments for all Signet Playground stateful services.

### Deployment + PVC pattern for singletons

For reference, this is the pattern the capstone will use for each singleton stateful
service:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-data
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 256Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
spec:
  replicas: 1
  strategy:
    type: Recreate        # ← critical for RWO PVCs (chapter 9 lesson)
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
        - name: mariadb
          image: mariadb:12.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mariadb-data
```

Note the `strategy: Recreate` — as you learned in chapter 9, a RWO PVC cannot be
mounted by two Pods simultaneously. `RollingUpdate` would create the new Pod before
deleting the old one, causing the new Pod to hang waiting for the volume. `Recreate`
deletes first, then creates.

---

## Cleanup

```bash
kubectl delete statefulset -l chapter=11
kubectl delete service -l chapter=11
kubectl delete pod -l chapter=11
kubectl delete pvc -l chapter=11
```

Note that deleting a StatefulSet does not delete its PVCs. You must delete them
explicitly. In these exercises the PVCs use Minikube's default `Delete` reclaim policy,
so the PVs and their data are removed when the PVCs are deleted.

---

## Knowledge check

1. What three stable properties does a StatefulSet guarantee that a Deployment does not?
2. Why does a StatefulSet require a headless Service? What happens if you use a regular
   ClusterIP Service instead?
3. A StatefulSet has `replicas: 3` and `podManagementPolicy: OrderedReady`. Pod `web-1`
   fails its readiness probe. What happens to `web-2`?
4. You scale a StatefulSet from 3 replicas to 1. What happens to the PVCs for `web-1`
   and `web-2`? What happens if you scale back to 3?
5. Explain the difference between `podManagementPolicy: OrderedReady` and `Parallel`.
   When would you choose Parallel?
6. A Helm chart deploys a single-replica PostgreSQL as a StatefulSet. Is this the
   simplest correct choice? What alternative would work?
7. Why should you never force-delete a StatefulSet Pod without confirming the original
   Pod is gone?
8. For the Signet Playground, all stateful services are singletons. What change in
   requirements would make a StatefulSet the better choice for MariaDB?

---

## Summary

StatefulSets give Pods stable names, stable per-replica storage through
volumeClaimTemplates, and ordered lifecycle management. They require a headless Service
to provide individual DNS records for each Pod. PVCs created by a StatefulSet survive
scaling and deletion — they must be removed manually.

Not every stateful workload needs a StatefulSet. Singleton databases and services are
simpler as Deployments with explicit PVCs and `strategy: Recreate`. StatefulSets become
valuable when multiple replicas need distinct identity — stable names for replication
configuration, per-replica storage that follows the Pod, and ordered startup for
primary/replica initialization.

For the Signet Playground, all stateful services are singletons and fit the
Deployment + PVC pattern. The capstone chapter will use this simpler approach.

Chapter 12 introduces namespaces, labels, and policy boundaries — how to organize,
query, and constrain resources as an application grows.
