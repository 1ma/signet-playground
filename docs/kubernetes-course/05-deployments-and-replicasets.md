# Chapter 05. Deployments and ReplicaSets

## Learning objectives

After completing this chapter you should be able to:

- Explain why bare Pods are insufficient for real workloads.
- Describe the relationship between Deployments, ReplicaSets, and Pods.
- Create a Deployment and understand the objects it produces.
- Scale a Deployment manually.
- Perform a rolling update and observe how Kubernetes transitions between versions.
- Roll back a failed deployment to a previous revision.
- Pause and resume a rollout to batch multiple changes.

---

## Why not bare Pods

In Chapter 03 you created Pods directly. This works for learning, but bare Pods have
critical limitations:

- **No self-healing across nodes.** If the node running the Pod fails, the Pod is gone.
  The kubelet only restarts containers on the same node — it cannot reschedule to another
  node.
- **No scaling.** You would have to create and manage N copies manually.
- **No rollout strategy.** Updating the image means deleting the old Pod and creating a
  new one — with downtime.
- **No rollback.** If the new version is broken, you must manually recreate the previous
  version.

Controllers solve all of these. A Deployment manages ReplicaSets, which manage Pods.
You declare the desired state once and the controllers handle the rest.

---

## The Deployment → ReplicaSet → Pod chain

```text
Deployment
  │
  │  creates and manages
  ▼
ReplicaSet (revision 1)          ReplicaSet (revision 2)
  │                                │
  │  ensures N replicas            │  ensures N replicas
  ▼                                ▼
Pod  Pod  Pod                    Pod  Pod  Pod
```

Each layer has a single responsibility:

| Resource   | Responsibility                                                                                              |
|------------|-------------------------------------------------------------------------------------------------------------|
| Deployment | Manages rollout strategy — creates new ReplicaSets on update, scales old ones down, tracks revision history |
| ReplicaSet | Ensures exactly N Pods matching its template exist at all times                                             |
| Pod        | Runs the containers                                                                                         |

You interact with the Deployment. You almost never create or modify ReplicaSets
directly. You never create Pods directly for a workload that needs to stay running.

---

## Creating a Deployment

### The manifest

```yaml
# examples/05-deployments/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
    chapter: "05"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
          ports:
            - containerPort: 80
```

### Anatomy

| Field                           | Purpose                                                                                      |
|---------------------------------|----------------------------------------------------------------------------------------------|
| `spec.replicas`                 | How many Pods to maintain                                                                    |
| `spec.selector.matchLabels`     | How the Deployment finds its ReplicaSets and Pods                                            |
| `spec.template`                 | The Pod template — every Pod created by this Deployment uses this spec                       |
| `spec.template.metadata.labels` | Must match `spec.selector.matchLabels` — if they do not, the API server rejects the manifest |

The selector is **immutable** after creation. If you need to change it, you must delete
and recreate the Deployment.

### The selector contract

The labels in `spec.selector.matchLabels` must appear in `spec.template.metadata.labels`.
This is how the ReplicaSet finds its Pods. If a Pod has the matching labels but was not
created by this ReplicaSet, the ReplicaSet will adopt it — which can cause unexpected
behavior. Keep selectors specific enough to avoid collisions between Deployments.

### Apply and inspect

```bash
kubectl apply -f examples/05-deployments/nginx-deployment.yaml
```

Wait for rollout:

```bash
kubectl rollout status deployment/web
```

Inspect the chain:

```bash
# The Deployment
kubectl get deployment web

# The ReplicaSet it created
kubectl get replicasets -l app=web

# The Pods the ReplicaSet created
kubectl get pods -l app=web -o wide
```

Note the naming convention:

```text
Deployment:  web
ReplicaSet:  web-5d4f7b8c9a      (Deployment name + template hash)
Pods:        web-5d4f7b8c9a-x7k2  (ReplicaSet name + random suffix)
```

The template hash in the ReplicaSet name is a hash of the Pod template. When you change
the template (e.g., update the image), a new ReplicaSet with a different hash is created.

---

## Scaling

### Manual scaling

```bash
kubectl scale deployment web --replicas=5
kubectl get pods -l app=web -w
```

The ReplicaSet controller detects that actual replicas (3) differ from desired (5) and
creates 2 new Pods. The Deployment itself does not change ReplicaSets — it only changes
the replica count on the existing one.

Scale down:

```bash
kubectl scale deployment web --replicas=2
```

Three Pods enter `Terminating`. The ReplicaSet controller selects which Pods to remove
(youngest first by default).

### Declarative scaling

Instead of `kubectl scale`, update the `replicas` field in the manifest and apply:

```yaml
spec:
  replicas: 4
```

```bash
kubectl apply -f examples/05-deployments/nginx-deployment.yaml
```

This is the preferred method — the manifest stays in sync with the cluster state.
`kubectl scale` is convenient for quick experiments but creates drift between the
manifest and the cluster.

---

## Rolling updates

A rolling update replaces Pods gradually so that the application stays available
throughout the transition.

### How it works

When you change the Pod template (typically the image), the Deployment controller:

1. Creates a **new ReplicaSet** with the updated template.
2. Scales the new ReplicaSet **up** incrementally.
3. Scales the old ReplicaSet **down** incrementally.
4. Continues until the new ReplicaSet has all replicas and the old has zero.

```text
Time ──────────────────────────────────────►

Old RS:  ███████████  ████████  █████  ██  ·
New RS:  ·            ███       ██████ █████████
         ─────────────────────────────────────
         replicas always >= desired during transition
```

### Strategy parameters

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

| Parameter        | Meaning                                                                                                                                           |
|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| `maxSurge`       | How many extra Pods above the desired count are allowed during the update. `1` means at most `replicas + 1` Pods exist at any time.               |
| `maxUnavailable` | How many Pods below the desired count are allowed during the update. `0` means all desired replicas must be available at all times — no downtime. |

The defaults are `maxSurge: 25%` and `maxUnavailable: 25%`. For critical workloads,
`maxSurge: 1, maxUnavailable: 0` is the safest — it always keeps full capacity and adds
one new Pod at a time.

The other strategy type is `Recreate`: kill all old Pods, then create all new ones. This
causes downtime but is necessary for workloads that cannot tolerate two versions running
simultaneously (e.g., a database migration that changes the schema).

### Performing a rolling update

Update the image in the manifest:

```yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.27    # was 1.26
```

```bash
kubectl apply -f examples/05-deployments/nginx-deployment.yaml
kubectl rollout status deployment/web
```

Or imperatively (useful for quick experiments, not for production):

```bash
kubectl set image deployment/web nginx=nginx:1.27
```

### Observing the rollout

```bash
# Watch Pods being replaced
kubectl get pods -l app=web -w

# See both ReplicaSets during the transition
kubectl get replicasets -l app=web

# Check rollout history
kubectl rollout history deployment/web
```

After the rollout completes, the old ReplicaSet still exists with 0 replicas. Kubernetes
keeps it for rollback purposes.

```bash
kubectl get replicasets -l app=web
```

You should see two ReplicaSets: the old one with `DESIRED: 0` and the new one with
`DESIRED: 3` (or whatever your replica count is).

---

## Rollback

If a deployment goes wrong, you can roll back to the previous ReplicaSet.

### Checking history

```bash
kubectl rollout history deployment/web
```

This shows revision numbers. To see what changed in a specific revision:

```bash
kubectl rollout history deployment/web --revision=1
```

### Rolling back

```bash
# Roll back to the previous revision
kubectl rollout undo deployment/web

# Roll back to a specific revision
kubectl rollout undo deployment/web --to-revision=1
```

A rollback is internally just another rolling update — the Deployment controller scales
the old ReplicaSet back up and the current one down. It creates a new revision number in
the history.

### Revision history limit

By default, Kubernetes keeps the last 10 ReplicaSets for rollback
(`spec.revisionHistoryLimit`). Old ReplicaSets beyond this limit are garbage collected.
In a busy deployment with frequent updates, you may want to increase this.

---

## Pause and resume

If you need to make multiple changes to a Deployment without triggering a rollout for
each one, you can pause it:

```bash
kubectl rollout pause deployment/web
```

Now make several changes:

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl set resources deployment/web -c nginx --limits=memory=128Mi
```

No rollout happens yet. Resume to apply all changes as a single rollout:

```bash
kubectl rollout resume deployment/web
kubectl rollout status deployment/web
```

This avoids unnecessary intermediate ReplicaSets and Pod churn.

---

## Workshop

### Exercise 1 — Create and inspect a Deployment

```yaml
# examples/05-deployments/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
    chapter: "05"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f examples/05-deployments/nginx-deployment.yaml
kubectl rollout status deployment/web
```

Trace the full object chain:

```bash
# Deployment
kubectl get deployment web -o wide

# ReplicaSet — note the template hash in the name
kubectl get rs -l app=web -o custom-columns=\
NAME:.metadata.name,\
DESIRED:.spec.replicas,\
CURRENT:.status.replicas,\
READY:.status.readyReplicas,\
IMAGE:.spec.template.spec.containers[0].image

# Pods — note the ReplicaSet name prefix
kubectl get pods -l app=web -o custom-columns=\
NAME:.metadata.name,\
OWNER:.metadata.ownerReferences[0].name,\
STATUS:.status.phase,\
IP:.status.podIP
```

Verify the ownership chain:

```bash
RS_NAME=$(kubectl get rs -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl get rs "$RS_NAME" -o jsonpath='{.metadata.ownerReferences[0].name}'
# Should print: web
```

### Exercise 2 — Scale and observe

Open a watch in a separate terminal:

```bash
kubectl get pods -l app=web -w
```

Scale up:

```bash
kubectl scale deployment web --replicas=5
```

Observe new Pods appearing. Scale back down:

```bash
kubectl scale deployment web --replicas=3
```

Observe Pods being terminated. Check which Pods were removed — they should be the
youngest ones.

### Exercise 3 — Rolling update

Update the image from 1.26 to 1.27. Edit the manifest and apply:

```bash
kubectl diff -f examples/05-deployments/nginx-deployment.yaml
kubectl apply -f examples/05-deployments/nginx-deployment.yaml
kubectl rollout status deployment/web
```

While the rollout is in progress (or immediately after), inspect both ReplicaSets:

```bash
kubectl get rs -l app=web
```

You should see the old ReplicaSet scaled to 0 and the new one at full capacity.

Verify the Pods are running the new image:

```bash
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

### Exercise 4 — Recover from a broken image

Deploy an image that does not exist:

```bash
kubectl set image deployment/web nginx=nginx:99.99.99
```

Watch the rollout stall:

```bash
kubectl rollout status deployment/web --timeout=30s
```

The timeout will expire. Investigate:

```bash
kubectl get pods -l app=web
kubectl describe pod -l app=web | grep -A3 "State:"
```

Some Pods will show `ImagePullBackOff`. Critically, the old Pods are **still running** —
the rolling update strategy kept them alive because the new Pods never became ready.

Roll back:

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

All Pods should return to the previous working image.

Check the revision history:

```bash
kubectl rollout history deployment/web
```

The rollback created a new revision, not a deletion of the failed one.

### Exercise 5 — Pause and batch changes

```bash
kubectl rollout pause deployment/web
kubectl set image deployment/web nginx=nginx:1.27-alpine
kubectl set resources deployment/web -c nginx --requests=cpu=50m,memory=64Mi
```

Verify no rollout started:

```bash
kubectl get rs -l app=web
kubectl rollout status deployment/web
```

Resume:

```bash
kubectl rollout resume deployment/web
kubectl rollout status deployment/web
```

Both changes apply in a single rollout — only one new ReplicaSet is created.

### Exercise 6 — maxSurge and maxUnavailable

Modify the strategy to see different rollout behaviors:

```yaml
# examples/05-deployments/slow-rollout.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-web
  labels:
    app: slow-web
    chapter: "05"
spec:
  replicas: 5
  selector:
    matchLabels:
      app: slow-web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: slow-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 2
```

```bash
kubectl apply -f examples/05-deployments/slow-rollout.yaml
kubectl rollout status deployment/slow-web
```

Now update the image and watch:

```bash
kubectl set image deployment/slow-web nginx=nginx:1.27
kubectl get pods -l app=slow-web -w
```

With `maxSurge: 1` and `maxUnavailable: 0`, you will see at most 6 Pods at any time
(5 + 1 surge). New Pods are created one at a time, and old Pods are only removed after
the new one passes its readiness probe. This is the safest but slowest rollout pattern.

Compare with a faster strategy:

```bash
kubectl rollout undo deployment/slow-web
kubectl patch deployment slow-web -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":2,"maxUnavailable":1}}}}'
kubectl set image deployment/slow-web nginx=nginx:1.27
kubectl get pods -l app=slow-web -w
```

Now up to 7 Pods can exist (5 + 2 surge) and 1 Pod can be unavailable during the
transition. The rollout completes faster but with a brief capacity reduction.

---

## Failure scenarios

### Scenario 1 — Mismatched selector and template labels

```yaml
# examples/05-deployments/bad-selector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-selector
  labels:
    chapter: "05"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: backend      # does not match selector
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

```bash
kubectl apply -f examples/05-deployments/bad-selector.yaml
```

The API server rejects the manifest immediately:

```text
Invalid value: ... `selector` does not match template `labels`
```

**Lesson:** this validation prevents a Deployment from creating Pods it cannot find. The
match between `selector.matchLabels` and `template.metadata.labels` is enforced at
admission time, not at runtime.

### Scenario 2 — Selector collision between Deployments

Create two Deployments with overlapping selectors:

```yaml
# examples/05-deployments/collision-a.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collision-a
  labels:
    chapter: "05"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shared-label
  template:
    metadata:
      labels:
        app: shared-label
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
---
# examples/05-deployments/collision-b.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collision-b
  labels:
    chapter: "05"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shared-label
  template:
    metadata:
      labels:
        app: shared-label
    spec:
      containers:
        - name: httpd
          image: httpd:2.4
```

```bash
kubectl apply -f examples/05-deployments/collision-a.yaml
kubectl apply -f examples/05-deployments/collision-b.yaml
```

Watch the chaos:

```bash
kubectl get pods -l app=shared-label -w
```

Both Deployments (and their ReplicaSets) try to manage all Pods with `app: shared-label`.
Pods are continuously created and deleted as each ReplicaSet fights the other for
ownership. The Pod count oscillates and never stabilizes.

**Lesson:** selectors must be unique per Deployment. Use specific labels
(`app: collision-a`, `app: collision-b`) to avoid this. In a real cluster, this bug is
subtle — it looks like instability rather than a configuration error.

Clean up:

```bash
kubectl delete deployment collision-a collision-b
```

### Scenario 3 — Stuck rollout with deadline

```bash
kubectl patch deployment web -p '{"spec":{"progressDeadlineSeconds":30}}'
kubectl set image deployment/web nginx=nginx:99.99.99
```

After 30 seconds:

```bash
kubectl rollout status deployment/web
```

The Deployment reports `ProgressDeadlineExceeded`. This does **not** automatically roll
back — the broken rollout stays in place, with old Pods still running and new Pods stuck
in `ImagePullBackOff`. You must decide: fix forward or roll back.

```bash
kubectl rollout undo deployment/web
```

**Lesson:** `progressDeadlineSeconds` is a detection mechanism, not an automatic
recovery. It sets the Deployment condition to `Progressing: False`, which monitoring
tools can alert on.

---

## Cleanup

```bash
kubectl delete deployment -l chapter=05
```

Verify:

```bash
kubectl get all -l chapter=05
```

---

## Knowledge check

1. Why is a bare Pod insufficient for a production workload?
2. A Deployment has 3 replicas. You change the image. How many ReplicaSets exist during
   the rollout? How many after it completes?
3. What is the difference between `maxSurge` and `maxUnavailable`? What settings give
   zero-downtime rollouts?
4. You run `kubectl rollout undo`. Does this delete the failed ReplicaSet?
5. Two Deployments have the same `selector.matchLabels`. What happens and why?
6. After a rollout, the old ReplicaSet has 0 replicas but still exists. Why does
   Kubernetes keep it?
7. What happens if you change `spec.selector.matchLabels` on an existing Deployment?
8. You need to update both the image and the resource limits simultaneously. How do you
   avoid two separate rollouts?

---

## Summary

Deployments are the standard controller for stateless workloads. They manage ReplicaSets,
which ensure the desired number of Pods are always running. Rolling updates replace Pods
gradually with zero downtime; rollbacks reactivate a previous ReplicaSet. The selector
links everything together — a wrong or colliding selector breaks the ownership chain.

Chapter 06 introduces Services — the mechanism that gives a stable network identity to
the ephemeral Pods that Deployments manage.
