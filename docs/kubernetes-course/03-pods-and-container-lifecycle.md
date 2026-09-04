# Chapter 03. Pods and container lifecycle

## Learning objectives

After completing this chapter you should be able to:

- Explain what a Pod is and why it is the smallest deployable unit in Kubernetes.
- Create Pods imperatively and declaratively.
- Inspect Pod phases, container states, restart counts, and termination reasons.
- Use `kubectl exec`, `kubectl logs`, `kubectl describe`, and `kubectl port-forward`.
- Distinguish startup, readiness, and liveness probes and explain when each applies.
- Diagnose a CrashLoopBackOff and identify its root cause.

---

## What is a Pod

A Pod is a group of one or more containers that share a network namespace, an IPC
namespace, and optionally storage volumes. Every container in a Pod sees the same
`localhost`, the same IP address, and the same port space.

Most Pods run a single application container. Multi-container Pods are used for
tightly coupled helpers — a sidecar that ships logs, an init container that prepares
data, or a proxy that handles TLS. If two containers can be deployed and scaled
independently, they belong in separate Pods.

```text
┌──────────────── Pod ────────────────┐
│                                     │
│  ┌───────────┐   ┌───────────────┐  │
│  │ container │   │   container   │  │
│  │  (app)    │   │  (sidecar)    │  │
│  └─────┬─────┘   └──────┬────────┘  │
│        │                │           │
│        └──── localhost ─┘           │
│              shared IP              │
│              shared volumes         │
│                                     │
└─────────────────────────────────────┘
```

### Pods are ephemeral

A Pod is not a durable entity. It runs on one node for its entire lifetime — it is
never moved. If the node fails, the Pod is gone. If the container crashes too many
times, the Pod stays on the node but enters a back-off loop. Controllers (Deployments,
ReplicaSets, Jobs) create replacement Pods; they do not repair existing ones.

This is a fundamental difference from Docker Compose, where `restart: unless-stopped`
restarts the same container in place indefinitely. In Kubernetes the unit of recovery
is a new Pod, not a repaired one.

---

## Pod phases

A Pod moves through a sequence of phases visible in `kubectl get pods`:

| Phase       | Meaning                                                                                               |
|-------------|-------------------------------------------------------------------------------------------------------|
| `Pending`   | Accepted by the cluster but not yet running — waiting for scheduling, image pulls, or init containers |
| `Running`   | At least one container is running or starting                                                         |
| `Succeeded` | All containers exited with code 0 (common for Jobs)                                                   |
| `Failed`    | All containers terminated and at least one exited non-zero                                            |
| `Unknown`   | Pod status cannot be determined (usually a node communication failure)                                |

The phase is a high-level summary. For diagnosis you need the per-container state.

---

## Container states

Each container within a Pod has its own state:

| State        | Meaning                                                                                  |
|--------------|------------------------------------------------------------------------------------------|
| `Waiting`    | Not running yet — pulling image, blocked by init container, or in back-off after a crash |
| `Running`    | Executing normally                                                                       |
| `Terminated` | Finished — either completed successfully or crashed                                      |

The `Waiting` state includes a **reason** that tells you why:

| Reason                              | Common cause                                                 |
|-------------------------------------|--------------------------------------------------------------|
| `ContainerCreating`                 | Image is being pulled or volumes are being mounted           |
| `CrashLoopBackOff`                  | Container keeps crashing; kubelet is waiting before retrying |
| `ErrImagePull` / `ImagePullBackOff` | Image not found, wrong tag, or registry auth failure         |
| `CreateContainerConfigError`        | Referenced ConfigMap or Secret does not exist                |

### Restart counts and back-off

When a container exits unexpectedly (non-zero exit code, OOM kill, or failed liveness
probe), the kubelet restarts it with an exponential back-off: 10s, 20s, 40s, ... up to
5 minutes. The restart count in `kubectl get pods` increments each time. A high restart
count means the container has been crashing repeatedly — the back-off is kubelet's way
of avoiding a tight crash loop that wastes CPU.

---

## Creating Pods

### Imperative creation

For quick experiments, create a Pod directly:

```bash
kubectl run nginx-test --image=nginx:1.27 --restart=Never
```

`--restart=Never` creates a bare Pod (not a Deployment). This is useful for one-off
tasks and debugging but should never be used for workloads that need to survive
failures.

### Declarative creation

Write a manifest file:

```yaml
# examples/03-pods/simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-lab
  labels:
    app: nginx-lab
    chapter: "03"
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Apply it:

```bash
kubectl apply -f examples/03-pods/simple-pod.yaml
```

The declarative form is what you will use throughout the course. It is version-
controllable, diffable, and repeatable.

### Anatomy of the manifest

| Field             | Purpose                                                                         |
|-------------------|---------------------------------------------------------------------------------|
| `apiVersion: v1`  | Pods are in the core API group                                                  |
| `kind: Pod`       | The resource type                                                               |
| `metadata.name`   | Unique name within the namespace                                                |
| `metadata.labels` | Key-value pairs for selection and organization                                  |
| `spec.containers` | List of containers to run in this Pod                                           |
| `containerPort`   | Documentation — the container listens here; does not publish it outside the Pod |

`containerPort` is informational. It does not open the port to the cluster or the host.
Networking is handled by Services (Chapter 06). Including it makes intent clear and
enables tools like `kubectl port-forward` to suggest a default port.

---

## Inspecting Pods

### kubectl get

```bash
kubectl get pods
kubectl get pods -o wide          # adds IP, node, nominated node
kubectl get pod nginx-lab -o yaml  # full API object including status
```

### kubectl describe

```bash
kubectl describe pod nginx-lab
```

`describe` combines the spec, status, conditions, events, and container details into
a human-readable report. The **Events** section at the bottom is often the fastest way
to find out what went wrong.

Key fields to check:

| Section    | What to look for                                            |
|------------|-------------------------------------------------------------|
| Status     | Pod phase                                                   |
| Conditions | `PodScheduled`, `Initialized`, `ContainersReady`, `Ready`   |
| Containers | State, restart count, last termination reason and exit code |
| Events     | Scheduling, image pull, start, probe failure, OOM kill      |

### kubectl logs

```bash
kubectl logs nginx-lab                  # current container output
kubectl logs nginx-lab --previous       # output from the last crashed instance
kubectl logs nginx-lab -f               # follow (stream)
kubectl logs nginx-lab --tail=50        # last 50 lines
kubectl logs nginx-lab -c sidecar       # specific container in a multi-container Pod
```

`--previous` is essential for crash diagnosis. When a container restarts, its current
logs are empty — the crash output is in the previous instance.

### kubectl exec

```bash
kubectl exec nginx-lab -- ls /etc/nginx/
kubectl exec -it nginx-lab -- /bin/sh
```

`-it` allocates a TTY for interactive sessions. The `--` separates `kubectl` flags from
the command passed to the container.

Use `exec` for live debugging, not for modifying application state. Changes inside a
container are lost when the Pod is replaced.

### kubectl port-forward

```bash
kubectl port-forward pod/nginx-lab 8080:80
```

This tunnels `localhost:8080` on your machine to port 80 inside the Pod. It is a
debugging tool, not a production exposure mechanism. Services and Ingress (Chapters 06
and 13) handle real traffic routing.

Open a second terminal and test:

```bash
curl -s http://localhost:8080 | head -5
```

Stop the port-forward with `Ctrl+C`.

---

## Probes

Probes are periodic checks that the kubelet performs on a container. They determine
whether a container is alive, ready to serve traffic, or has finished starting up.

### Liveness probe

Answers: **is the process still working?**

If a liveness probe fails, the kubelet kills the container and restarts it according to
the Pod's restart policy. Use a liveness probe when a process can get stuck (deadlock,
infinite loop) but not exit on its own.

### Readiness probe

Answers: **can this container accept traffic right now?**

A container that fails its readiness probe is removed from Service endpoints. It is not
killed or restarted — it is just temporarily excluded from load balancing. Use a
readiness probe for warmup periods, dependency checks, or graceful drain.

### Startup probe

Answers: **has the container finished its initial startup?**

While a startup probe is defined and has not yet succeeded, liveness and readiness
probes are suspended. This prevents slow-starting containers from being killed by an
impatient liveness probe before they are ready.

### Probe mechanisms

| Type        | How it works                                         |
|-------------|------------------------------------------------------|
| `httpGet`   | HTTP GET to a path and port; 2xx/3xx = success       |
| `tcpSocket` | TCP connection to a port; connection = success       |
| `exec`      | Run a command inside the container; exit 0 = success |
| `grpc`      | gRPC health check (Kubernetes >= 1.27)               |

### Probe parameters

| Parameter             | Default | Meaning                                                    |
|-----------------------|---------|------------------------------------------------------------|
| `initialDelaySeconds` | 0       | Wait before first probe                                    |
| `periodSeconds`       | 10      | Interval between probes                                    |
| `timeoutSeconds`      | 1       | Probe must respond within this time                        |
| `failureThreshold`    | 3       | Consecutive failures before the probe takes effect         |
| `successThreshold`    | 1       | Consecutive successes to mark healthy (readiness only > 1) |

A common mistake is setting a liveness probe with tight thresholds on a slow-starting
application. The container gets killed before it finishes starting. A startup probe
solves this without loosening the liveness probe once the application is running.

### Example with probes

```yaml
# examples/03-pods/probed-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-probed
  labels:
    app: httpd-probed
    chapter: "03"
spec:
  containers:
    - name: httpd
      image: httpd:2.4
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
        failureThreshold: 2
```

The startup probe allows up to 20 seconds (10 failures × 2s) for initial startup.
Once it succeeds, the liveness and readiness probes take over with their own cadences.

---

## Workshop

### Exercise 1 — Create and inspect a Pod

Create the simple Pod manifest:

```bash
mkdir -p examples/03-pods
```

```yaml
# examples/03-pods/simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-lab
  labels:
    app: nginx-lab
    chapter: "03"
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f examples/03-pods/simple-pod.yaml
kubectl get pod nginx-lab -o wide
kubectl describe pod nginx-lab
```

Find the following in the `describe` output:

1. Which node is the Pod running on?
2. What IP address was assigned?
3. What image was pulled and from which registry?
4. Is the Pod `Ready`? What conditions are `True`?

### Exercise 2 — Interact with the running container

Read the default nginx page from inside the container:

```bash
kubectl exec nginx-lab -- curl -s http://localhost:80 | head -3
```

> If `curl` is not available inside the nginx image, use `wget -qO-` instead.

Open an interactive shell:

```bash
kubectl exec -it nginx-lab -- /bin/bash
```

Inside the container:

```bash
hostname            # should print "nginx-lab"
cat /etc/resolv.conf  # shows cluster DNS configuration
ip addr              # shows the Pod's network interface and IP
exit
```

The hostname matches the Pod name. The DNS configuration points to CoreDNS
(`kube-dns.kube-system.svc.cluster.local`). These are set automatically by the kubelet.

### Exercise 3 — Logs and port-forwarding

Check the nginx access log:

```bash
kubectl logs nginx-lab
```

It may be empty if no requests have been made. Generate some traffic:

```bash
kubectl port-forward pod/nginx-lab 8080:80 &
PF_PID=$!
curl -s http://localhost:8080 > /dev/null
curl -s http://localhost:8080 > /dev/null
kill $PF_PID 2>/dev/null
wait $PF_PID 2>/dev/null
```

Now check logs again:

```bash
kubectl logs nginx-lab --tail=5
```

You should see the two GET requests.

### Exercise 4 — Diagnose a CrashLoopBackOff

Create a Pod that crashes intentionally:

```yaml
# examples/03-pods/crashing-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: crasher
  labels:
    chapter: "03"
spec:
  containers:
    - name: crasher
      image: busybox:1.37
      command: ["sh", "-c", "echo 'starting...'; sleep 2; exit 1"]
```

```bash
kubectl apply -f examples/03-pods/crashing-pod.yaml
```

Watch the Pod cycle through states:

```bash
kubectl get pod crasher -w
```

After a few cycles you will see `CrashLoopBackOff`. The back-off delay increases with
each restart. Diagnose:

```bash
# Check the last termination reason and exit code
kubectl get pod crasher -o jsonpath='{.status.containerStatuses[0].lastState.terminated}' | jq .

# Read the output from the crashed instance
kubectl logs crasher --previous
```

The exit code is `1` and the logs show `starting...` — the container ran but exited
with an error. In a real scenario the logs would point to the root cause (missing
config, unreachable dependency, application bug).

### Exercise 5 — Probes in action

Create the probed Pod:

```yaml
# examples/03-pods/probed-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-probed
  labels:
    chapter: "03"
spec:
  containers:
    - name: httpd
      image: httpd:2.4
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
        failureThreshold: 2
```

```bash
kubectl apply -f examples/03-pods/probed-pod.yaml
kubectl describe pod httpd-probed
```

Observe the Events section. The startup probe succeeds first, then the liveness and
readiness probes begin.

Now break the liveness probe by removing the default page:

```bash
kubectl exec httpd-probed -- rm /usr/local/apache2/htdocs/index.html
```

Watch the Pod:

```bash
kubectl get pod httpd-probed -w
```

After three consecutive liveness probe failures (about 30 seconds) the kubelet kills and
restarts the container. The restart count increments. After the restart, the default
`index.html` is back (the container starts fresh) and the probes succeed again.

Check the events:

```bash
kubectl describe pod httpd-probed | grep -A20 Events
```

You should see `Unhealthy` events for the liveness probe followed by a `Killing` event.

### Exercise 6 — Multi-container Pod

Create a Pod with an application container and a sidecar that tails the access log:

```yaml
# examples/03-pods/sidecar-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-logger
  labels:
    chapter: "03"
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      volumeMounts:
        - name: logs
          mountPath: /var/log/nginx
    - name: log-reader
      image: busybox:1.37
      command: ["sh", "-c", "tail -f /shared-logs/access.log"]
      volumeMounts:
        - name: logs
          mountPath: /shared-logs
  volumes:
    - name: logs
      emptyDir: {}
```

```bash
kubectl apply -f examples/03-pods/sidecar-pod.yaml
```

Generate traffic and read the sidecar's output:

```bash
kubectl port-forward pod/nginx-with-logger 8080:80 &
PF_PID=$!
sleep 1
curl -s http://localhost:8080 > /dev/null
kill $PF_PID 2>/dev/null
wait $PF_PID 2>/dev/null

kubectl logs nginx-with-logger -c log-reader --tail=5
```

The sidecar reads from the shared `emptyDir` volume. Both containers see the same
files because they are in the same Pod. This pattern — sharing data through a volume
rather than through the network — is a core reason multi-container Pods exist.

---

## Failure scenarios

### Scenario 1 — Image pull failure

```yaml
# examples/03-pods/bad-image.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-image
  labels:
    chapter: "03"
spec:
  containers:
    - name: app
      image: nginx:99.99.99
```

```bash
kubectl apply -f examples/03-pods/bad-image.yaml
kubectl get pod bad-image -w
```

The Pod stays in `Pending` or shows `ErrImagePull` / `ImagePullBackOff`. Diagnose:

```bash
kubectl describe pod bad-image | grep -A5 Events
```

The event message includes the registry error. Fix: correct the image tag and
`kubectl apply` again.

**Lesson:** Kubernetes does not validate that an image exists when you submit the
manifest. The error surfaces only when the kubelet tries to pull it on the assigned node.

### Scenario 2 — Missing ConfigMap reference

```yaml
# examples/03-pods/missing-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: missing-config
  labels:
    chapter: "03"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sleep", "3600"]
      env:
        - name: APP_SETTING
          valueFrom:
            configMapKeyRef:
              name: nonexistent-config
              key: setting
```

```bash
kubectl apply -f examples/03-pods/missing-config.yaml
kubectl get pod missing-config
```

The container shows `CreateContainerConfigError`. It will not start until the ConfigMap
exists:

```bash
kubectl describe pod missing-config | grep -B2 -A5 Warning
```

**Lesson:** environment variable references to ConfigMaps and Secrets are resolved at
container creation time, not at `apply` time. A typo in the reference name blocks the
container silently.

### Scenario 3 — OOMKilled

```yaml
# examples/03-pods/oom-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
  labels:
    chapter: "03"
spec:
  containers:
    - name: eater
      image: busybox:1.37
      command: ["sh", "-c", "dd if=/dev/zero of=/dev/null bs=1M & head -c 200m /dev/zero | tail -c +1 > /dev/null; sleep 3600"]
      resources:
        limits:
          memory: "64Mi"
```

```bash
kubectl apply -f examples/03-pods/oom-pod.yaml
kubectl get pod oom-test -w
```

The container is killed by the kernel's OOM killer because it exceeds its 64 MiB memory
limit. Diagnose:

```bash
kubectl get pod oom-test -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

The reason is `OOMKilled`. The exit code is 137 (128 + SIGKILL signal 9).

**Lesson:** memory limits are enforced by the kernel via cgroups, not by Kubernetes
itself. Exceeding the limit results in an immediate kill — there is no warning or
graceful shutdown.

---

## Cleanup

Remove all resources created in this chapter:

```bash
kubectl delete pod -l chapter=03
kubectl delete pod nginx-test 2>/dev/null
```

Verify:

```bash
kubectl get pods
```

Only the default `kubernetes` Service should remain (visible with `kubectl get all`).

---

## Knowledge check

1. What is the difference between a Pod phase and a container state?
2. You see a Pod with `STATUS: Running` but `READY: 0/1`. What does this mean and which
   probe is likely failing?
3. A container has restarted 12 times. How do you read the logs from the previous crash?
4. Why does `containerPort` in a Pod spec not make the port reachable from outside the
   Pod?
5. Explain the difference between a liveness probe and a readiness probe. What happens
   when each one fails?
6. A Pod is stuck in `Pending`. Name three possible causes.
7. In a multi-container Pod, both containers mount the same `emptyDir` volume. Container
   A writes a file. Can container B read it? What happens to the file when the Pod is
   deleted?

---

## Summary

Pods are ephemeral, single-node units that run one or more containers sharing a network
and optional storage. You inspect them with `get`, `describe`, `logs`, `exec`, and
`port-forward`. Probes let the kubelet detect startup completion, liveness failures, and
readiness changes. Understanding container states, restart back-off, and termination
reasons is the foundation for diagnosing every workload problem in later chapters.

Chapter 04 introduces declarative configuration with `kubectl apply` and the structure
of Kubernetes YAML in detail.
