# Chapter 01. Local lab setup

## Learning objectives

After completing this chapter you should be able to:

- Confirm that Docker Engine, Minikube, and `kubectl` are installed and functional.
- Start a Minikube cluster with the Docker driver and explicit resource limits.
- Explain the relationship between Docker containers on the host and the Minikube node.
- Switch between Kubernetes contexts and understand what a context represents.
- Enable specific Minikube add-ons required by later chapters.
- Destroy and recreate a lab cluster from scratch in under two minutes.

---

## Prerequisites

This course assumes a Debian 13 workstation with:

- **Docker Engine - Community Edition** (not Docker Desktop). The daemon must be running
  and your user must belong to the `docker` group.
- **Minikube** (>= 1.34).
- **kubectl** matching the Kubernetes minor version Minikube will run. Minikube bundles a
  compatible `kubectl`, but a standalone install gives you tab completion and avoids
  `minikube kubectl --` indirection.

Verify the three tools before proceeding:

```bash
docker version --format '{{.Server.Version}}'
minikube version --short
kubectl version --client -o yaml | grep gitVersion
```

All three commands must succeed without `sudo`. If `docker version` fails with a
permission error, add your user to the `docker` group and start a new login session:

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

---

## Concepts

### Why Minikube with the Docker driver

Minikube creates a single-node Kubernetes cluster inside a local environment. The
**Docker driver** runs the Kubernetes node as a Docker container rather than a full
virtual machine. This makes startup fast and avoids hypervisor dependencies, but it means
the Minikube "node" shares the host kernel and Docker daemon.

```text
+---------------------------------------------+
|  Host (Debian 13)                           |
|                                             |
|  Docker Engine                              |
|  ├── minikube container (the "node")        |
|  │   ├── kubelet                            |
|  │   ├── kube-apiserver                     |
|  │   ├── etcd                               |
|  │   ├── kube-scheduler                     |
|  │   ├── kube-controller-manager            |
|  │   ├── kube-proxy                         |
|  │   └── workload containers (nested)       |
|  └── other host containers (unrelated)      |
+---------------------------------------------+
```

Workload containers run inside the Minikube container through nested container
runtimes. They are **not** siblings of the Minikube container in `docker ps` on the
host — you will only see them with `minikube ssh` or through `kubectl`.

### Profiles

A Minikube **profile** is an isolated cluster with its own name, Kubernetes version,
resource limits, and add-ons. The default profile is called `minikube`. This course uses
a dedicated profile called `signet-lab` so it never collides with other local clusters.

### Contexts

`kubectl` determines which cluster to talk to using a **context** stored in
`~/.kube/config`. Each context combines a cluster endpoint, a user credential, and an
optional default namespace. When Minikube starts a profile it creates (or updates) a
matching context automatically.

```text
context "signet-lab"
  ├── cluster: signet-lab  (https://127.0.0.1:<port>)
  ├── user:    signet-lab  (client certificate)
  └── namespace: default
```

You switch contexts with `kubectl config use-context <name>`. The active context is what
every `kubectl` command targets — running a command against the wrong context is one of
the most common operational mistakes.

### Namespaces (first look)

A namespace is a scope boundary inside a cluster. Kubernetes starts with four namespaces:

| Namespace         | Purpose                                         |
|-------------------|-------------------------------------------------|
| `default`         | Where resources land when no namespace is given |
| `kube-system`     | Control plane and cluster-critical components   |
| `kube-public`     | Conventionally readable by everyone             |
| `kube-node-lease` | Node heartbeat leases                           |

For now, treat namespaces as a way to keep your lab resources separate from system
internals. Chapter 12 covers them in depth.

---

## Creating the lab cluster

Start a fresh cluster with explicit resource limits:

```bash
minikube start \
  --profile signet-lab \
  --driver docker \
  --cpus 4 \
  --memory 4096 \
  --disk-size 40g \
  --kubernetes-version stable
```

Minikube will pull the base image, create the container, bootstrap the control plane, and
register the context. The first start takes longer because images must be downloaded;
subsequent starts reuse them.

### Why explicit resource limits matter

Without `--cpus` and `--memory`, Minikube picks defaults that vary across versions and
drivers. Explicit values make the lab reproducible and prevent the cluster from
over-committing the host. The Signet Playground capstone (Chapter 18) needs at least
2 CPUs and 4 GiB of memory for the full stack; setting these now avoids mid-course
surprises.

### Verify the cluster

```bash
minikube status --profile signet-lab
```

Expected output (the exact container runtime version varies):

```text
signet-lab
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Confirm `kubectl` reaches the cluster:

```bash
kubectl cluster-info
```

The output should show the control plane URL and CoreDNS. If it points to a different
cluster, check the active context:

```bash
kubectl config current-context
```

It should print `signet-lab`. If it does not:

```bash
kubectl config use-context signet-lab
```

---

## Inspecting what Minikube created

### The node

A Kubernetes node is a machine (physical or virtual) that runs workloads. In Minikube
with the Docker driver, that "machine" is a Docker container:

```bash
docker ps --filter "name=signet-lab" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Inspect the node from the Kubernetes side:

```bash
kubectl get nodes -o wide
```

There is exactly one node. Its `INTERNAL-IP` is the IP address of the Minikube container
on the Docker bridge network, **not** the host's IP.

### System Pods

The control plane components run as Pods in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system
```

You should see at least:

- `coredns-*` — cluster DNS
- `etcd-signet-lab` — key-value store for cluster state
- `kube-apiserver-signet-lab` — the API server
- `kube-controller-manager-signet-lab` — reconciliation controllers
- `kube-proxy-*` — network rules on each node
- `kube-scheduler-signet-lab` — decides where Pods run
- `storage-provisioner` — Minikube's dynamic storage provisioner

Every one of these should be `Running` with `READY 1/1`. If any Pod is restarting or
pending, the cluster is not healthy — diagnose before continuing.

### Understanding the context entry

Examine the full context configuration:

```bash
kubectl config view --minify
```

`--minify` shows only the active context. Note the three linked objects (cluster, user,
context) and the server URL. This is the information `kubectl` uses to authenticate and
route every request.

---

## Enabling add-ons

Minikube ships with optional add-ons. Enable only what later chapters need:

```bash
minikube addons enable metrics-server --profile signet-lab
minikube addons enable ingress --profile signet-lab
```

- **metrics-server** provides `kubectl top` and resource-aware scheduling (Chapter 08).
- **ingress** installs an NGINX ingress controller for HTTP routing (Chapter 13).

List the state of all add-ons to confirm:

```bash
minikube addons list --profile signet-lab
```

Do not enable add-ons speculatively. Each one runs additional Pods that consume memory.
If a later chapter needs another add-on it will say so explicitly.

---

## Workshop

### Exercise 1 — Cluster lifecycle

Practice the full lifecycle so that destroying and recreating the cluster feels routine:

```bash
# Stop the cluster (preserves state)
minikube stop --profile signet-lab

# Verify the Docker container is stopped
docker ps -a --filter "name=signet-lab" --format "{{.Names}}: {{.Status}}"

# Start it again
minikube start --profile signet-lab

# Verify the node is Ready
kubectl get nodes

# Delete the cluster entirely (destroys all data)
minikube delete --profile signet-lab

# Recreate from scratch
minikube start \
  --profile signet-lab \
  --driver docker \
  --cpus 2 \
  --memory 4096 \
  --disk-size 20g \
  --kubernetes-version stable
```

After the final start, re-enable the add-ons:

```bash
minikube addons enable metrics-server --profile signet-lab
minikube addons enable ingress --profile signet-lab
```

This should become a muscle-memory sequence. A broken lab is never worth debugging longer
than a fresh start takes.

### Exercise 2 — Explore the node from inside

Open a shell inside the Minikube node:

```bash
minikube ssh --profile signet-lab
```

Once inside, list the running containers at the runtime level:

```bash
docker ps --format "table {{.Names}}\t{{.Image}}" 2>/dev/null \
  || crictl ps -o table
```

> Depending on the Minikube version the container runtime may be Docker or containerd.
> The command above tries both.

Compare this output with `kubectl get pods -A`. Every Pod you see in `kubectl` has one
or more containers visible at the runtime level, plus an infrastructure `pause` container
that holds the network namespace.

Exit the SSH session with `exit`.

### Exercise 3 — Multiple profiles

Create a second, throwaway profile to observe how contexts work:

```bash
minikube start --profile throwaway --driver docker --cpus 1 --memory 2048
```

List all contexts:

```bash
kubectl config get-contexts
```

The `*` marks the active context. Switch between them:

```bash
kubectl config use-context signet-lab
kubectl get nodes   # shows the signet-lab node

kubectl config use-context throwaway
kubectl get nodes   # shows the throwaway node
```

Delete the throwaway profile:

```bash
minikube delete --profile throwaway
```

Verify the context was removed:

```bash
kubectl config get-contexts
```

If the deleted profile's context is still listed (some Minikube versions leave orphan
entries), clean it manually:

```bash
kubectl config delete-context throwaway 2>/dev/null
kubectl config delete-cluster throwaway 2>/dev/null
kubectl config delete-user throwaway 2>/dev/null
```

Set the active context back to the lab:

```bash
kubectl config use-context signet-lab
```

---

## Failure scenarios

### Scenario 1 — Wrong context

A colleague asks you to check something on their cluster. You switch contexts, forget to
switch back, and your next `kubectl apply` targets the wrong cluster.

Simulate this:

```bash
# Pretend you switched to a non-existent context
kubectl config set-context fake-prod --cluster=fake --user=fake
kubectl config use-context fake-prod
kubectl get nodes
```

`kubectl get nodes` fails because the cluster does not exist. Diagnose:

```bash
kubectl config current-context          # shows "fake-prod" — wrong cluster
kubectl config get-contexts             # find the right context
kubectl config use-context signet-lab   # switch back
kubectl get nodes                       # works again
```

Clean up the fake context:

```bash
kubectl config delete-context fake-prod
```

**Lesson:** always check `kubectl config current-context` before a destructive operation
in a multi-cluster environment. Many teams set shell prompts or terminal status lines to
display the active context permanently.

### Scenario 2 — Under-provisioned cluster

Start a cluster with deliberately insufficient resources:

```bash
minikube start --profile starved --driver docker --cpus 1 --memory 1024
```

Wait for it to settle, then check Pod health:

```bash
kubectl --context starved get pods -n kube-system
```

Some system Pods may be stuck in `Pending`, restarting, or `OOMKilled`. The
`metrics-server` add-on, if enabled, is especially likely to fail under 1 GiB.

Check node conditions:

```bash
kubectl --context starved describe node starved | grep -A5 Conditions
```

Look for `MemoryPressure` or `DiskPressure` conditions set to `True`.

Delete the broken cluster:

```bash
minikube delete --profile starved
```

**Lesson:** resource limits are not cosmetic. An under-provisioned cluster exhibits
real scheduling and stability failures identical to production resource exhaustion.

---

## Cleanup

If you want a clean slate before continuing to Chapter 02, delete and recreate:

```bash
minikube delete --profile signet-lab
minikube start \
  --profile signet-lab \
  --driver docker \
  --cpus 4 \
  --memory 4096 \
  --disk-size 40g \
  --kubernetes-version stable
minikube addons enable metrics-server --profile signet-lab
minikube addons enable ingress --profile signet-lab
```

Otherwise, simply ensure the cluster is running and the context is correct:

```bash
minikube status --profile signet-lab
kubectl config current-context   # must print "signet-lab"
```

---

## Knowledge check

1. What is the difference between `minikube stop` and `minikube delete`?
2. When using the Docker driver, where does the Kubernetes node run? How would you find
   its container with `docker ps`?
3. What three objects make up a `kubectl` context?
4. You run `kubectl get pods` and see nothing, but you know system Pods exist. What is
   the most likely reason?
5. Why does this course set `--cpus` and `--memory` explicitly instead of accepting
   Minikube defaults?
6. After deleting a Minikube profile, the corresponding `kubectl` context sometimes
   remains in `~/.kube/config`. How do you remove it?

---

## Summary

You now have a reproducible, disposable Kubernetes lab built on Minikube and Docker.
The cluster runs in a single Docker container, has explicit resource limits, and exposes
a `kubectl` context called `signet-lab`. You can destroy and recreate it in under two
minutes.

Chapter 02 uses this cluster to explore how Kubernetes manages state internally.
