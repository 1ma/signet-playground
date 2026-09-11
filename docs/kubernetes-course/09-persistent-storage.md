# Chapter 09. Persistent storage

## Learning objectives

After completing this chapter you should be able to:

- Explain why Pod filesystems are ephemeral and what happens to data when a container
  restarts or a Pod is replaced.
- Distinguish `emptyDir`, `hostPath`, PersistentVolumes, and PersistentVolumeClaims.
- Use Minikube's default StorageClass to provision dynamic persistent storage.
- Describe access modes, reclaim policies, and volume expansion.
- Persist data across Pod deletion, rollout, and rescheduling.
- Classify the Signet Playground volumes by their lifecycle and sharing requirements.

---

## The ephemeral filesystem problem

Every container in a Pod starts with a **writable layer** provided by the container
runtime. This layer disappears when:

- The container crashes and the kubelet restarts it (even within the same Pod).
- The Pod is deleted, rescheduled, or replaced by a rolling update.
- The node is drained or fails.

For a stateless web server this is fine — the container image carries everything the
process needs. But a database, a blockchain node, or anything that writes data it needs
to survive restarts has a problem: the data vanishes with the container.

Docker Compose solves this with named volumes (`node_data:/home/bitcoin/.bitcoin`).
Kubernetes has its own volume system that is more explicit but also more powerful — it
can provision storage dynamically, enforce access modes, survive node failures, and
integrate with cloud-provider disks.

---

## Volume types from simplest to most durable

### emptyDir — scratch space that lives with the Pod

An `emptyDir` volume is created when a Pod is assigned to a node and deleted when the
Pod is removed. It starts empty. Every container in the Pod can mount it at a different
path. It is the simplest volume type and the most limited.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scratch-demo
  labels:
    chapter: "09"
spec:
  containers:
    - name: writer
      image: busybox:1.37
      command: ["sh", "-c", "echo hello > /data/greeting.txt && sleep 3600"]
      volumeMounts:
        - name: scratch
          mountPath: /data
    - name: reader
      image: busybox:1.37
      command: ["sh", "-c", "sleep 5 && cat /data/greeting.txt && sleep 3600"]
      volumeMounts:
        - name: scratch
          mountPath: /data
  volumes:
    - name: scratch
      emptyDir: {}
```

Key properties:

- **Lifetime:** same as the Pod. Container restarts preserve it; Pod deletion destroys it.
- **Backing:** node disk by default. Set `emptyDir: { medium: Memory }` for a RAM-backed
  tmpfs (counts against the container's memory limit — you saw this with Secrets in
  chapter 7).
- **Use cases:** shared scratch space between containers in the same Pod, caches, temporary
  files.
- **Not for:** data that must survive Pod replacement.

### hostPath — mount a directory from the node

A `hostPath` volume mounts a file or directory from the **host node's** filesystem into
the Pod.

```yaml
volumes:
  - name: node-logs
    hostPath:
      path: /var/log
      type: Directory
```

Key properties:

- **Lifetime:** the data lives on the node's disk, independent of the Pod.
- **Portability:** none. If the Pod is rescheduled to a different node, it sees a
  different directory (or the path does not exist). This makes it unsuitable for most
  workloads.
- **Security:** gives the Pod access to the host filesystem, which is a significant
  privilege escalation. Most production clusters restrict or forbid `hostPath` through
  admission policies.
- **Use cases:** node-level agents like log collectors (Fluentd, Promtail) or monitoring
  daemons that genuinely need to read the host. Minikube's default provisioner uses
  `hostPath` under the hood because there is only one node.
- **Not for:** application data. Use PersistentVolumeClaims instead.

### PersistentVolume and PersistentVolumeClaim — durable cluster storage

This is the mechanism you will use for any data that must survive Pod replacement.
Kubernetes splits the concept into two objects:

| Object                          | Who creates it        | What it represents                |
|---------------------------------|-----------------------|-----------------------------------|
| **PersistentVolume (PV)**       | Admin or provisioner  | A piece of storage in the cluster |
| **PersistentVolumeClaim (PVC)** | Application developer | A request for storage by a Pod    |

The separation exists so that the person deploying an application does not need to know
the storage implementation details. They write a PVC ("I need 10 Gi of read-write
storage") and the cluster finds or creates a PV that satisfies the claim.

```text
Pod  ──mounts──▶  PVC  ──binds to──▶  PV  ──backed by──▶  disk / NFS / cloud volume
```

---

## PersistentVolumeClaims in practice

### Creating a PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-data
  labels:
    chapter: "09"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

This asks for 1 Gi of storage that a single node can mount for reading and writing.
You do not specify where the storage comes from — that is the StorageClass's job.

### Mounting a PVC in a Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-demo
  labels:
    chapter: "09"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pvc-demo
  template:
    metadata:
      labels:
        app: pvc-demo
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["sh", "-c", "echo $(date) >> /data/log.txt && sleep 3600"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: demo-data
```

The flow:

1. You apply the PVC. Kubernetes looks for a matching PV or asks the StorageClass to
   create one.
2. The PV is **bound** to the PVC — no other PVC can use it.
3. The Pod mounts the PVC by name. The kubelet attaches the underlying storage to the
   container at `/data`.
4. If you delete the Pod or roll out a new version, the PVC and PV remain. The new Pod
   picks up the same data.

### Inspecting PVC and PV state

```bash
kubectl get pvc demo-data
```

```text
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
demo-data   Bound    pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890   1Gi        RWO            standard       30s
```

Key fields:

- **STATUS:** `Pending` (waiting for a PV), `Bound` (matched to a PV), `Lost` (the
  bound PV was deleted).
- **VOLUME:** the name of the PV this claim is bound to.
- **STORAGECLASS:** the provisioner that created the PV (or `""` for manual provisioning).

```bash
kubectl get pv
```

```text
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS
pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890   1Gi        RWO            Delete           Bound    default/demo-data    standard
```

Notice that PVs are **cluster-scoped** — they do not belong to a namespace. PVCs are
namespaced. The PV's `CLAIM` field shows which namespace/PVC it is bound to.

---

## Access modes

Access modes describe how nodes (not Pods) can mount the volume:

| Mode               | Short | Meaning                                         |
|--------------------|-------|-------------------------------------------------|
| `ReadWriteOnce`    | RWO   | One node can mount it read-write                |
| `ReadOnlyMany`     | ROX   | Many nodes can mount it read-only               |
| `ReadWriteMany`    | RWX   | Many nodes can mount it read-write              |
| `ReadWriteOncePod` | RWOP  | Exactly one Pod across the cluster (K8s 1.27+)  |

Important: **RWO does not mean one Pod** — it means one node. If two Pods are on the
same node and both mount the same RWO PVC, both can write to it. Use RWOP if you need
strict single-writer guarantees.

On Minikube with a single node, RWO effectively covers all cases. On multi-node
clusters, the access mode becomes a real constraint — an RWO volume cannot be mounted
on two nodes simultaneously, which matters during rolling updates if the old and new
Pod land on different nodes.

---

## StorageClasses and dynamic provisioning

In the PVC above, you did not create a PV manually. The cluster's **StorageClass**
handled it automatically.

### What is a StorageClass?

A StorageClass defines a "class" of storage — the provisioner, parameters (disk type,
IOPS, replication), and reclaim policy. When a PVC references a StorageClass (or uses
the default one), the provisioner creates a PV automatically.

```bash
kubectl get storageclass
```

```text
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false
```

Minikube's `standard` StorageClass uses the `minikube-hostpath` provisioner. Under the
hood it creates directories on the Minikube node's filesystem. This is fine for learning
but is obviously not suitable for production — the data lives on one node and cannot
survive node loss.

### Cloud StorageClasses

In a production cluster, StorageClasses map to real storage systems:

| Provider   | Provisioner                     | Creates                |
|------------|---------------------------------|------------------------|
| GKE        | `pd.csi.storage.gke.io`         | GCE Persistent Disk    |
| EKS        | `ebs.csi.aws.com`               | AWS EBS volume         |
| AKS        | `disk.csi.azure.com`            | Azure Managed Disk     |
| Bare metal | varies (Longhorn, Rook-Ceph...) | Software-defined block |

The beauty of the PVC abstraction is that your manifests do not change. The same PVC
YAML that works on Minikube works on GKE — only the StorageClass is different, and the
default one is usually sensible for general workloads.

### Requesting a specific StorageClass

```yaml
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

If you omit `storageClassName`, the cluster uses its default StorageClass. You can set
it explicitly when you need a specific tier (SSD vs HDD, high IOPS, replicated storage).

---

## Reclaim policies

When a PVC is deleted, what happens to the PV and its data?

| Policy   | What happens                                                       |
|----------|--------------------------------------------------------------------|
| `Delete` | The PV and its underlying storage are deleted. Data is lost.       |
| `Retain` | The PV becomes `Released` but the data is preserved. An admin must |
|          | manually clean up or rebind it.                                    |

Minikube's default is `Delete` — convenient for a lab, dangerous for production data.
Production StorageClasses for databases and other critical data typically use `Retain`.

You can change the reclaim policy of an existing PV:

```bash
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### The lifecycle in practice

```text
PVC created  ──▶  PV provisioned (Bound)  ──▶  Pod mounts PV
                                                     │
PVC deleted  ──▶  Reclaim policy applies:            │
                  Delete → PV and disk removed       │
                  Retain → PV stays as Released      ▼
                           (manual intervention)   Pod gone,
                                                   PVC gone
```

---

## Volume expansion

Some StorageClasses support resizing volumes after creation. Check the
`ALLOWVOLUMEEXPANSION` column:

```bash
kubectl get storageclass
```

If `true`, you can edit the PVC to request more storage:

```bash
kubectl patch pvc demo-data -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
```

The underlying volume is resized. If the volume is mounted, the filesystem resize
happens on the next Pod restart (or live, depending on the provisioner and CSI driver).

Minikube's default StorageClass does **not** support expansion (`false`). Cloud
providers generally do — GKE and EKS both support online expansion for their default
disk types.

Volumes can only grow, never shrink. If you request a smaller size, the API server
rejects the change.

---

## Hands-on: data survives Pod deletion

### Exercise 1: emptyDir lifetime

Create a Pod that writes to an emptyDir, then verify the data disappears when the Pod
is deleted.

```yaml
# ex01-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
  labels:
    chapter: "09"
spec:
  containers:
    - name: writer
      image: busybox:1.37
      command: ["sh", "-c", "date > /scratch/timestamp.txt && sleep 3600"]
      volumeMounts:
        - name: scratch
          mountPath: /scratch
  volumes:
    - name: scratch
      emptyDir: {}
```

```bash
kubectl apply -f ex01-emptydir.yaml
kubectl exec emptydir-demo -- cat /scratch/timestamp.txt
```

Now delete the Pod and recreate it:

```bash
kubectl delete pod emptydir-demo
kubectl apply -f ex01-emptydir.yaml
kubectl exec emptydir-demo -- cat /scratch/timestamp.txt
```

The timestamp is different — the data from the first Pod is gone. This is the ephemeral
nature of emptyDir.

### Exercise 2: PVC survives Pod deletion

Create a PVC and a Pod that writes to it, then delete the Pod and verify the data is
still there.

```yaml
# ex02-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: survive-demo
  labels:
    chapter: "09"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 64Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-writer
  labels:
    chapter: "09"
spec:
  containers:
    - name: writer
      image: busybox:1.37
      command: ["sh", "-c", "date >> /data/log.txt && cat /data/log.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: survive-demo
```

```bash
kubectl apply -f ex02-pvc.yaml
kubectl logs pvc-writer
```

You should see one timestamp. Now delete the Pod and recreate it:

```bash
kubectl delete pod pvc-writer
kubectl apply -f ex02-pvc.yaml
kubectl logs pvc-writer
```

You should see **two** timestamps — the old one from the deleted Pod plus the new one.
The PVC kept the data alive.

### Exercise 3: PVC with a Deployment

Deployments create new Pods on every rollout. Verify that data persists across rollouts.

```yaml
# ex03-deploy-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: deploy-data
  labels:
    chapter: "09"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 64Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: persistent-app
  labels:
    chapter: "09"
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: persistent-app
  template:
    metadata:
      labels:
        app: persistent-app
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["sh", "-c", "date >> /data/log.txt && cat /data/log.txt && sleep 3600"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: deploy-data
```

```bash
kubectl apply -f ex03-deploy-pvc.yaml
kubectl logs -l app=persistent-app
```

Trigger a rollout (this creates a new Pod):

```bash
kubectl rollout restart deployment persistent-app
kubectl rollout status deployment persistent-app
kubectl logs -l app=persistent-app
```

Two timestamps again — different Pods, same data. The PVC outlives the Pod.

**Why `strategy: Recreate`?** With RWO volumes on a single-node cluster, `RollingUpdate`
works because both old and new Pods are on the same node. But on a multi-node cluster,
the old Pod might be on node A and the new one scheduled on node B — RWO prevents two
nodes from mounting the volume simultaneously, so the new Pod gets stuck. `Recreate`
avoids this by terminating the old Pod before creating the new one. For stateful
single-replica workloads with RWO volumes, `Recreate` is the safe default.

### Exercise 4: inspect the PV

```bash
kubectl get pvc -l chapter=09
kubectl get pv
kubectl describe pv <pv-name-from-above>
```

Look at:

- `Source.Type` — on Minikube this will be `hostPath`.
- `Source.Path` — the actual directory on the Minikube node's filesystem.
- `Status` — `Bound`.
- `Claim` — which namespace/PVC it is bound to.
- `Reclaim Policy` — `Delete` (Minikube default).

You can even verify the data on disk by running a shell inside the Minikube node:

```bash
minikube ssh -p signet-lab -- ls <source-path-from-describe>
```

### Exercise 5: reclaim policy in action

Delete the PVC and see what happens to the PV:

```bash
kubectl delete pvc deploy-data
kubectl get pv
```

With the `Delete` reclaim policy, the PV disappears along with its data. Now imagine
this was a production database — that is why critical workloads use `Retain`.

### Exercise 6: manual PV with Retain

Create a PV manually with `Retain` policy, bind it to a PVC, write data, delete the
PVC, and verify the PV survives.

```yaml
# ex06-retain.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
  labels:
    chapter: "09"
spec:
  capacity:
    storage: 64Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/manual-pv-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-claim
  labels:
    chapter: "09"
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 64Mi
```

```bash
kubectl apply -f ex06-retain.yaml
kubectl get pv manual-pv
```

The PV should be `Bound`. Now delete the PVC:

```bash
kubectl delete pvc manual-claim
kubectl get pv manual-pv
```

The PV is now `Released` — the data is still there, but no PVC can bind to it
automatically. An admin must either delete the PV, clean its `claimRef` to make it
available again, or reclaim the data manually.

---

## Failure scenarios

### Scenario 1: PVC stuck in Pending

```yaml
# fail01-no-class.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stuck-claim
  labels:
    chapter: "09"
spec:
  storageClassName: premium-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f fail01-no-class.yaml
kubectl get pvc stuck-claim
kubectl describe pvc stuck-claim
```

**Diagnose it:** the PVC stays `Pending` because there is no StorageClass named
`premium-ssd` in Minikube. The Events tell you exactly why. Fix: change the
`storageClassName` to `standard` (or remove it to use the default).

### Scenario 2: Pod cannot start because PVC is bound to another node

This scenario is hard to reproduce on single-node Minikube, but understanding it is
important. On a multi-node cluster:

1. Pod A on node-1 mounts a RWO PVC. The PV is attached to node-1.
2. Pod A is deleted. Pod B is scheduled on node-2 and tries to mount the same PVC.
3. The volume must be detached from node-1 and attached to node-2.

If the detach fails (node-1 is unreachable, or the cloud provider API is slow), Pod B
gets stuck with an event like:

```text
Multi-Attach error for volume "pvc-xxx": Volume is already exclusively attached to node-1
```

**Resolution:** wait for the detach to complete, or force-detach the volume using cloud
provider tools. Using `strategy: Recreate` reduces the window where both Pods exist
but does not eliminate the detach delay.

### Scenario 3: data loss from Delete reclaim policy

1. Deploy a database-like Pod with a PVC using the default `Delete` reclaim policy.
2. Accidentally delete the PVC (`kubectl delete pvc <name>`).
3. The PV and all data are destroyed immediately.

**Lesson:** for any data you cannot regenerate, either set the reclaim policy to
`Retain` or take regular backups. In production, admission policies can enforce
`Retain` for certain StorageClasses.

---

## Signet Playground storage preview

The Signet Playground uses several named volumes in Docker Compose. Here is how each
one maps to Kubernetes storage:

| Compose volume     | Content                        | Survives restart? | Sharing                    | K8s mapping        |
|--------------------|--------------------------------|-------------------|----------------------------|--------------------|
| `node_data`        | Bitcoin blockchain data        | Must              | Only the node Pod          | PVC (RWO)          |
| `fulcrum_data`     | Fulcrum index                  | Must              | Only the Fulcrum Pod       | PVC (RWO)          |
| `mariadb_data`     | MariaDB database files         | Must              | Only the MariaDB Pod       | PVC (RWO)          |
| `frigate_data`     | Frigate index                  | Must              | Only the Frigate Pod       | PVC (RWO)          |
| `valkey_data`      | Valkey RDB/AOF persistence     | Must              | Only the Valkey Pod        | PVC (RWO)          |
| `valkey_socket`    | Valkey Unix socket             | No                | mempool-api + Valkey       | emptyDir           |
| `cookie_dir`       | RPC authentication cookie      | No                | node + miner + fulcrum + … | emptyDir or Secret |

### Observations

**Most volumes are straightforward PVCs.** `node_data`, `fulcrum_data`, `mariadb_data`,
`frigate_data`, and `valkey_data` are each owned by a single service, need to survive
Pod replacement, and fit the standard PVC (RWO) pattern. Each gets its own PVC.

**`valkey_socket` is an emptyDir.** The Unix socket is a runtime artifact — it does not
need to survive restarts and is only useful while the process is running. Valkey and
mempool-api are in the same Pod (they share the socket via localhost semantics), so an
emptyDir mounted in both containers is the natural choice.

**`cookie_dir` is the most interesting case.** The RPC cookie is generated by the
Bitcoin node at startup. Multiple services need to read it (miner, fulcrum, frigate,
mempool-api, faucet). In Docker Compose this is a shared named volume. In Kubernetes
the options are:

1. **emptyDir inside the node Pod** — works if all consumers are in the same Pod, but
   they are not.
2. **Init container that copies the cookie into a Secret** — the node Pod generates the
   cookie, an init container or sidecar reads it and creates/updates a Kubernetes
   Secret, and other Pods mount the Secret. This is more Kubernetes-native but adds
   complexity.
3. **Shared PVC (RWX)** — mount the same volume in multiple Pods. Requires a
   StorageClass that supports ReadWriteMany (NFS, CephFS, etc.). Minikube's default
   provisioner does not support RWX, but it is possible with additional setup.

The capstone chapter (18) will decide the approach. For now, note that the cookie's
lifecycle (regenerated on each node startup, shared read-only by many consumers) does
not fit neatly into any single Kubernetes primitive. This is the kind of real-world
problem that makes the migration interesting.

### Storage estimates

| PVC            | Initial size | Notes                                             |
|----------------|--------------|---------------------------------------------------|
| `node-data`    | 1 Gi         | Signet blockchain is small; mainnet would be 600+ |
| `fulcrum-data` | 512 Mi       | Index size depends on chain size                  |
| `mariadb-data` | 256 Mi       | Mempool schema is modest for signet               |
| `frigate-data` | 256 Mi       | Index size depends on chain size                  |
| `valkey-data`  | 64 Mi        | Small — faucet rate limiting and mempool cache    |

These are starting estimates for a private signet with low traffic. They can be tuned
after observing actual usage — remember `kubectl exec` into the Pod and check `df -h`
or `du -sh /data` to see real consumption.

---

## Cleanup

```bash
kubectl delete all,pvc,pv -l chapter=09
```

Note: PVs with `Retain` policy survive PVC deletion. If the manual PV from exercise 6
still exists:

```bash
kubectl delete pv manual-pv
```

---

## Knowledge check

1. What happens to data in an emptyDir volume when the Pod is deleted? What about when
   a single container in the Pod crashes and restarts?
2. Explain the difference between a PersistentVolume and a PersistentVolumeClaim.
   Who creates each one, and why are they separate objects?
3. A PVC is stuck in `Pending`. What are the most likely causes?
4. What is the difference between `ReadWriteOnce` and `ReadWriteOncePod`? When does
   the distinction matter?
5. Your production database runs on a PVC with the default `Delete` reclaim policy.
   A teammate accidentally runs `kubectl delete pvc db-data`. What happens, and how
   would you have prevented it?
6. Why does the Signet Playground's `valkey_socket` volume map to emptyDir rather than
   a PVC?
7. A Deployment with `strategy: RollingUpdate` and a RWO PVC works on Minikube but gets
   stuck on a two-node cluster. Explain why and how to fix it.
8. In the Signet Playground, why is the `cookie_dir` volume harder to model in
   Kubernetes than the other volumes?

---

## Summary

Pods are ephemeral — their filesystem dies with them. emptyDir provides scratch space
that lives with the Pod. PersistentVolumeClaims request durable storage that survives
Pod replacement, backed by PersistentVolumes provisioned either manually or dynamically
through StorageClasses. Access modes control how many nodes can mount a volume
simultaneously. Reclaim policies determine whether data is preserved or destroyed when
a claim is deleted. For the Signet Playground, most volumes are straightforward RWO
PVCs, but the shared RPC cookie presents a design challenge that connects storage,
configuration, and inter-Pod communication.

Chapter 10 introduces Jobs, init containers, and startup ordering — how to model
one-time setup tasks and dependencies between workloads.
