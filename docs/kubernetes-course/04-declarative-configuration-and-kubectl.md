# Chapter 04. Declarative configuration and kubectl

## Learning objectives

After completing this chapter you should be able to:

- Read any Kubernetes YAML manifest and identify its `apiVersion`, `kind`, `metadata`,
  `spec`, and `status` sections.
- Explain the difference between `spec` (desired state) and `status` (actual state).
- Use `kubectl apply`, `kubectl diff`, `kubectl get`, `kubectl explain`, `kubectl wait`,
  and `kubectl delete` confidently.
- Distinguish names, UIDs, labels, annotations, and selectors.
- Identify immutable fields and understand what happens when you try to change them.
- Understand field ownership and the `last-applied-configuration` annotation.

---

## The structure of a Kubernetes object

Every Kubernetes resource follows the same top-level structure:

```yaml
apiVersion: v1            # API group and version
kind: Pod                 # resource type
metadata:                 # identity and organization
  name: nginx-lab
  namespace: default
  uid: 8a3f2...           # assigned by the API server
  resourceVersion: "4521" # assigned by etcd
  labels:
    app: nginx-lab
  annotations:
    description: "lab exercise"
spec:                     # desired state — what you want
  containers:
    - name: nginx
      image: nginx:1.27
status:                   # actual state — what exists now
  phase: Running
  podIP: 10.244.0.5
  conditions:
    - type: Ready
      status: "True"
```

### apiVersion

Identifies which API group and version handles this resource. The format is
`group/version` or just `version` for the core group:

| apiVersion             | Resources                                                         |
|------------------------|-------------------------------------------------------------------|
| `v1`                   | Pod, Service, ConfigMap, Secret, Namespace, PersistentVolumeClaim |
| `apps/v1`              | Deployment, ReplicaSet, StatefulSet, DaemonSet                    |
| `batch/v1`             | Job, CronJob                                                      |
| `networking.k8s.io/v1` | Ingress, NetworkPolicy                                            |

If you do not know the `apiVersion` for a resource, `kubectl explain` will tell you
(covered below).

### kind

The resource type. Always PascalCase: `Pod`, `Deployment`, `ConfigMap`. Together with
`apiVersion`, it tells the API server which controller and schema to use.

### metadata

Identity and organizational data:

| Field             | Purpose                                    | Who sets it  |
|-------------------|--------------------------------------------|--------------|
| `name`            | Unique identifier within the namespace     | You          |
| `namespace`       | Scope (defaults to `default`)              | You          |
| `uid`             | Globally unique identifier                 | API server   |
| `resourceVersion` | Optimistic concurrency token               | etcd         |
| `labels`          | Key-value pairs for selection and grouping | You          |
| `annotations`     | Key-value pairs for arbitrary metadata     | You or tools |
| `ownerReferences` | Link to the controller that created this   | Controllers  |

### spec vs. status

This is the declarative model in one sentence:

- **spec** — what you declared. The desired state. You write it.
- **status** — what actually exists. The actual state. Controllers and the kubelet
  write it.

You never write `status` in a manifest. The API server ignores it on `apply`. The
reconciliation loop exists to make `status` converge toward `spec`.

When you read a resource with `kubectl get -o yaml`, both sections are present. Learning
to read `status` is how you diagnose problems — it tells you what Kubernetes thinks is
happening, not what you asked for.

---

## Labels, annotations, and selectors

### Labels

Labels are key-value pairs attached to any resource. They are the primary mechanism for
selecting and grouping objects:

```yaml
metadata:
  labels:
    app: mempool
    component: api
    tier: backend
    chapter: "06"
```

Rules:

- Keys and values are strings, max 63 characters.
- Keys can have an optional prefix (`app.kubernetes.io/name`). Prefixed keys are
  conventional for shared tooling; unprefixed keys are fine for your own use.
- Labels are indexed — Kubernetes can filter on them efficiently.

### Selectors

Selectors query resources by label. They are how controllers find the objects they
manage:

```bash
# Equality-based
kubectl get pods -l app=mempool
kubectl get pods -l app=mempool,tier=backend

# Set-based
kubectl get pods -l 'app in (mempool, faucet)'
kubectl get pods -l 'tier notin (frontend)'
kubectl get pods -l 'chapter'          # has the label, any value
kubectl get pods -l '!chapter'         # does not have the label
```

Selectors are not just a `kubectl` convenience. Services use them to find backend Pods.
Deployments use them to find their ReplicaSets. A wrong selector means a broken
connection between controller and resource — one of the most common Kubernetes mistakes.

### Annotations

Annotations are key-value pairs for metadata that is **not** used for selection:

```yaml
metadata:
  annotations:
    description: "Mempool API backend"
    config-hash: "a3f8c9..."
    kubectl.kubernetes.io/last-applied-configuration: "{...}"
```

Unlike labels, annotations can hold long strings (up to 256 KB total per object), JSON
blobs, URLs, or tool-specific data. They are not indexed and cannot be used in selectors.

**When to use which:**

| Need                                                 | Use                           |
|------------------------------------------------------|-------------------------------|
| Select or group resources                            | Label                         |
| Attach metadata for tools or humans                  | Annotation                    |
| Link parent to child (Deployment → ReplicaSet → Pod) | `ownerReferences` (automatic) |

---

## Names, UIDs, and immutable fields

### Names

A name is unique within a namespace and resource type. You can have a Pod called `nginx`
and a Service called `nginx` in the same namespace — they are different resource types.
But you cannot have two Pods called `nginx` in the same namespace.

Names must be DNS-compatible: lowercase, alphanumeric, hyphens allowed, max 253
characters.

### UIDs

Every object gets a UID assigned by the API server. UIDs are globally unique across the
entire cluster and across time — if you delete a Pod and create one with the same name,
it gets a different UID. Controllers use UIDs, not names, to track ownership.

### Immutable fields

Some fields cannot be changed after creation:

| Field                     | Why immutable                                        |
|---------------------------|------------------------------------------------------|
| `metadata.name`           | It is the object's identity                          |
| `metadata.namespace`      | Moving an object between namespaces is not supported |
| `spec.containers[*].name` | Container identity within the Pod                    |
| `Pod.spec.nodeName`       | A Pod is bound to a node for life                    |
| `Job.spec.selector`       | Changing it would orphan existing Pods               |

If you try to change an immutable field with `kubectl apply`, the API server rejects the
request. The fix is to delete and recreate the resource. Controllers like Deployments
handle this transparently — they create new Pods instead of modifying existing ones.

---

## Essential kubectl commands

### kubectl apply

```bash
kubectl apply -f manifest.yaml        # single file
kubectl apply -f directory/            # all YAML files in a directory
kubectl apply -f manifest.yaml --dry-run=client  # parse and validate without sending
kubectl apply -f manifest.yaml --dry-run=server  # API server validates without persisting
```

`apply` is the primary declarative command. It creates the resource if it does not exist,
or patches it if it does. It stores the full manifest in the
`kubectl.kubernetes.io/last-applied-configuration` annotation so it can compute diffs on
the next apply.

### kubectl diff

```bash
kubectl diff -f manifest.yaml
```

Shows what `apply` would change, without changing anything. The output is a unified diff
comparing the live object with the manifest. Run this before every `apply` when you are
unsure of the effect.

### kubectl get

```bash
kubectl get pods                          # list Pods in the current namespace
kubectl get pods -A                       # all namespaces
kubectl get pods -o wide                  # extra columns
kubectl get pods -o yaml                  # full API object
kubectl get pods -o json                  # JSON format
kubectl get pods -o jsonpath='{.items[*].metadata.name}'  # extract specific fields
kubectl get pods -l app=nginx             # filter by label
kubectl get pods --sort-by=.metadata.creationTimestamp     # sort
kubectl get pods -w                       # watch for changes
```

`-o yaml` is the most important form for learning. It shows you the complete object —
spec, status, metadata, conditions — exactly as the API server sees it.

### kubectl explain

```bash
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.livenessProbe
kubectl explain deployment.spec.strategy --recursive
```

`explain` is a built-in reference that describes every field of every resource type.
It is faster than searching documentation and always matches your cluster's API version.
The `--recursive` flag shows the full field tree without descriptions.

### kubectl wait

```bash
kubectl wait --for=condition=Ready pod/nginx-lab --timeout=60s
kubectl wait --for=condition=Available deployment/hello --timeout=120s
kubectl wait --for=delete pod/nginx-lab --timeout=30s
```

`wait` blocks until a condition is met or the timeout expires. Use it in scripts instead
of `sleep` — it reacts to actual state changes, not arbitrary delays.

### kubectl delete

```bash
kubectl delete pod nginx-lab                  # by name
kubectl delete -f manifest.yaml               # by manifest
kubectl delete pods -l chapter=03             # by label selector
kubectl delete pod nginx-lab --grace-period=0 --force  # immediate (use with caution)
```

Deletion is not instant. By default, Kubernetes sends SIGTERM to the containers, waits
up to 30 seconds (the grace period) for them to shut down, then sends SIGKILL. The Pod
shows `Terminating` during this window.

`--grace-period=0 --force` skips the graceful shutdown. Use it only for stuck Pods in
a lab, never in production.

---

## Field ownership and apply mechanics

### last-applied-configuration

When you run `kubectl apply`, it stores the full manifest as a JSON annotation on the
object:

```yaml
annotations:
  kubectl.kubernetes.io/last-applied-configuration: |
    {"apiVersion":"v1","kind":"Pod","metadata":{"name":"nginx-lab",...},...}
```

On the next `apply`, kubectl computes a **three-way merge**:

1. The new manifest (what you are applying now).
2. The last-applied-configuration (what you applied last time).
3. The live object (what currently exists in the cluster).

This three-way merge allows `apply` to distinguish between "the user removed this
field intentionally" and "this field was added by something else and should be kept."

### Practical consequence

If you create a resource with `kubectl create` or `kubectl run` (no last-applied
annotation) and later switch to `kubectl apply`, you will see a warning:

```text
Warning: resource pods/nginx-lab is missing the
kubectl.kubernetes.io/last-applied-configuration annotation
```

This is harmless — `apply` adds the annotation on its first run. But it means the first
`apply` cannot detect intentional field removals, since there is no "last time" to
compare against.

**Rule:** pick either imperative (`create`, `run`, `edit`) or declarative (`apply`) and
stick with it for each resource. Mixing them causes confusion about which fields are
managed and which are not.

---

## Workshop

### Exercise 1 — Read and understand a live object

Create a simple Pod and examine its full API representation:

```yaml
# examples/04-declarative/inspect-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: inspect-me
  labels:
    chapter: "04"
    exercise: inspect
  annotations:
    purpose: "learning to read API objects"
spec:
  containers:
    - name: httpd
      image: httpd:2.4
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f examples/04-declarative/inspect-pod.yaml
kubectl get pod inspect-me -o yaml
```

In the output, identify:

1. The `uid` and `resourceVersion` in metadata (you did not set these).
2. The `last-applied-configuration` annotation (kubectl added this).
3. The `status` section with phase, conditions, container states, and Pod IP.
4. The `ownerReferences` field — it should be absent because no controller owns this Pod.

### Exercise 2 — Use kubectl explain

Without looking at documentation, answer these questions using only `kubectl explain`:

```bash
# What fields does a Pod's spec accept?
kubectl explain pod.spec | head -30

# What types of probes exist?
kubectl explain pod.spec.containers.livenessProbe

# What is the default restart policy?
kubectl explain pod.spec.restartPolicy

# What fields does a Service spec accept?
kubectl explain service.spec
```

Now explore a resource type you have not used yet:

```bash
kubectl explain deployment.spec.strategy
kubectl explain deployment.spec.strategy.rollingUpdate
```

This is the skill: when you encounter a new field in someone else's YAML, use `explain`
before guessing what it does.

### Exercise 3 — Evolve a resource declaratively

Start with a minimal manifest:

```yaml
# examples/04-declarative/evolving-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: evolving
  labels:
    chapter: "04"
    version: "1"
spec:
  containers:
    - name: nginx
      image: nginx:1.26
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f examples/04-declarative/evolving-pod.yaml
```

**Step 1 — Preview a change:**

Edit the manifest: change the image to `nginx:1.27` and the version label to `"2"`.
Before applying, preview:

```bash
kubectl diff -f examples/04-declarative/evolving-pod.yaml
```

The diff shows exactly what will change. Apply it:

```bash
kubectl apply -f examples/04-declarative/evolving-pod.yaml
```

Verify:

```bash
kubectl get pod evolving -o jsonpath='{.spec.containers[0].image}'
# nginx:1.27
```

**Step 2 — Add a label:**

Add `tier: frontend` to the labels. Diff and apply:

```bash
kubectl diff -f examples/04-declarative/evolving-pod.yaml
kubectl apply -f examples/04-declarative/evolving-pod.yaml
```

**Step 3 — Remove a label:**

Remove the `tier` label from the manifest. Diff and apply:

```bash
kubectl diff -f examples/04-declarative/evolving-pod.yaml
kubectl apply -f examples/04-declarative/evolving-pod.yaml
```

Verify the label is gone:

```bash
kubectl get pod evolving --show-labels
```

This works because of the three-way merge: `apply` sees that `tier` was in
`last-applied-configuration` but is no longer in the manifest, so it removes it.

**Step 4 — Hit an immutable field:**

Try changing the container name from `nginx` to `web`:

```bash
kubectl diff -f examples/04-declarative/evolving-pod.yaml
kubectl apply -f examples/04-declarative/evolving-pod.yaml
```

The API server rejects the change — container names are immutable on a Pod. The only way
to change it is to delete and recreate:

```bash
kubectl delete pod evolving
kubectl apply -f examples/04-declarative/evolving-pod.yaml
```

This is why Deployments exist: they handle delete-and-recreate transparently when the
Pod template changes.

### Exercise 4 — Labels and selectors

Create several Pods with different labels:

```yaml
# examples/04-declarative/labeled-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-frontend
  labels:
    chapter: "04"
    app: myapp
    tier: frontend
spec:
  containers:
    - name: nginx
      image: nginx:1.27
---
apiVersion: v1
kind: Pod
metadata:
  name: app-backend
  labels:
    chapter: "04"
    app: myapp
    tier: backend
spec:
  containers:
    - name: httpd
      image: httpd:2.4
---
apiVersion: v1
kind: Pod
metadata:
  name: other-app
  labels:
    chapter: "04"
    app: other
    tier: backend
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

```bash
kubectl apply -f examples/04-declarative/labeled-pods.yaml
```

Practice selectors:

```bash
# All Pods in this chapter
kubectl get pods -l chapter=04

# Only the backend tier
kubectl get pods -l tier=backend

# Only myapp backends
kubectl get pods -l app=myapp,tier=backend

# Everything except frontend
kubectl get pods -l 'tier notin (frontend)'

# Pods that have the "tier" label (any value)
kubectl get pods -l 'tier'

# Custom columns with labels
kubectl get pods -l chapter=04 -L app,tier
```

### Exercise 5 — kubectl wait in a script

Write a small script that creates a Pod and waits for it to be ready before making a
request:

```bash
kubectl apply -f examples/04-declarative/inspect-pod.yaml

kubectl wait --for=condition=Ready pod/inspect-me --timeout=60s

kubectl port-forward pod/inspect-me 8080:80 &
PF_PID=$!
sleep 1
curl -s http://localhost:8080 | head -3
kill $PF_PID 2>/dev/null
wait $PF_PID 2>/dev/null
```

Compare this with using `sleep 10` instead of `kubectl wait`. The wait-based version is
both faster (does not wait unnecessarily) and more reliable (does not fail if startup
takes longer than expected).

### Exercise 6 — Dry-run modes

Compare client-side and server-side dry-run:

```yaml
# examples/04-declarative/bad-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  labels:
    chapter: "04"
spec:
  containers:
    - name: app
      image: nginx:1.27
      ports:
        - containerPort: 99999
```

```bash
# Client-side: validates YAML syntax and basic schema
kubectl apply -f examples/04-declarative/bad-pod.yaml --dry-run=client
```

Client-side dry-run may accept this (it does minimal validation). Now try server-side:

```bash
# Server-side: the API server validates without persisting
kubectl apply -f examples/04-declarative/bad-pod.yaml --dry-run=server
```

The API server rejects port 99999 (valid range is 1-65535). Server-side dry-run catches
errors that client-side misses because it runs the full admission and validation chain.

**Rule:** use `--dry-run=server` when you want real validation. Use `--dry-run=client`
when you want to test manifest generation without cluster access (CI pipelines, offline
environments).

---

## Failure scenarios

### Scenario 1 — Conflicting imperative and declarative management

```bash
# Create a Pod imperatively
kubectl run conflict-test --image=nginx:1.27

# Add a label imperatively
kubectl label pod conflict-test team=platform

# Now try to manage it declaratively
cat <<'EOF' > /tmp/conflict-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: conflict-test
  labels:
    chapter: "04"
spec:
  containers:
    - name: conflict-test
      image: nginx:1.27
EOF

kubectl apply -f /tmp/conflict-test.yaml
```

The warning appears: missing `last-applied-configuration`. More importantly, the `team`
label you added imperatively is preserved — `apply` does not remove it because it was
never in a previous `last-applied-configuration`.

Check:

```bash
kubectl get pod conflict-test --show-labels
```

The `team=platform` label is still there even though it is not in the manifest.

```bash
# Apply a second time — now last-applied exists without "team"
# but team was never in last-applied, so it survives
kubectl apply -f /tmp/conflict-test.yaml
kubectl get pod conflict-test --show-labels
```

**Lesson:** mixing imperative and declarative management creates invisible state. Fields
added imperatively are invisible to `apply`'s three-way merge and persist indefinitely.

Clean up:

```bash
kubectl delete pod conflict-test
rm /tmp/conflict-test.yaml
```

### Scenario 2 — Applying to the wrong namespace

```bash
kubectl apply -f examples/04-declarative/inspect-pod.yaml -n kube-system
```

If a Pod named `inspect-me` already exists in `default`, this creates a **second** Pod
with the same name in `kube-system`. They are independent objects — namespaces are
separate scopes.

```bash
kubectl get pod inspect-me -n kube-system
kubectl get pod inspect-me -n default
```

Two different Pods, two different UIDs, same name.

**Lesson:** always verify your target namespace. In production environments, many teams
set a default namespace in their context to avoid accidents:

```bash
kubectl config set-context --current --namespace=my-app
```

Clean up:

```bash
kubectl delete pod inspect-me -n kube-system
```

---

## Cleanup

```bash
kubectl delete pod -l chapter=04
```

Verify:

```bash
kubectl get pods
```

---

## Knowledge check

1. What is the difference between `spec` and `status` in a Kubernetes object? Who writes
   each one?
2. You apply a manifest, then add a label with `kubectl label`. On the next `apply` (with
   the same manifest, without the label), is the label removed? Why or why not?
3. What does `kubectl diff` show, and why should you run it before `apply`?
4. Name two differences between labels and annotations.
5. You try to change a Pod's container name with `kubectl apply` and it fails. How do you
   make the change?
6. What is the difference between `--dry-run=client` and `--dry-run=server`?
7. Why is `kubectl explain` more reliable than searching the web for field documentation?
8. You have two Pods named `nginx` — one in namespace `default`, one in namespace
   `staging`. Are they the same object?

---

## Summary

Kubernetes resources follow a consistent structure: `apiVersion`, `kind`, `metadata`,
`spec`, and `status`. You declare the desired state in `spec` and observe the actual
state in `status`. The declarative workflow — `diff`, `apply`, `get`, `wait` — is
idempotent and version-controllable. Labels enable selection and grouping; annotations
carry metadata for tools and humans. Understanding the three-way merge, immutable fields,
and field ownership prevents a category of subtle operational mistakes.

Chapter 05 introduces Deployments and ReplicaSets — the controllers that make it
practical to manage Pods at scale.
