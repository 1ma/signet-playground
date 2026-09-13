# Chapter 10. Jobs, init containers, and startup ordering

## Learning objectives

After completing this chapter you should be able to:

- Explain why Jobs exist and how they differ from Deployments.
- Configure Job completions, parallelism, backoff limits, and active deadlines.
- Use CronJobs to run recurring work on a schedule.
- Use init containers to enforce local prerequisites before the main containers start.
- Understand why Kubernetes does not replicate Docker Compose's `depends_on` and what
  to use instead.
- Design idempotent setup tasks that are safe to retry after partial failure.
- Model the Signet Playground's `wallet-setup` as a Kubernetes Job.

---

## The problem with long-running controllers for finite work

Deployments keep Pods running **indefinitely**. When a container exits, the kubelet
restarts it. This is exactly right for a web server or a database — but what about a
task that should run once, succeed, and stop?

Docker Compose handles this with `restart: no` — the container runs, exits, and Compose
leaves it alone. But the Compose model has no concept of retries, completion tracking,
or failure budgets. If the task fails halfway, you notice manually and rerun it.

Kubernetes models finite work explicitly with **Jobs**.

---

## Jobs

A Job creates one or more Pods and ensures that a specified number of them successfully
terminate. When the required number of completions is reached, the Job is complete.
Unlike a Deployment, it does **not** restart Pods that exit successfully.

### A minimal Job

```yaml
# simple-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
  labels:
    chapter: "10"
spec:
  template:
    metadata:
      labels:
        chapter: "10"
    spec:
      containers:
        - name: hello
          image: busybox:1.37
          command: ["sh", "-c", "echo 'Job completed at' $(date) && sleep 2"]
      restartPolicy: Never
```

```bash
kubectl apply -f simple-job.yaml
kubectl get jobs
kubectl get pods --selector=job-name=hello-job
```

Observations:

- The Pod runs, prints its message, and enters `Completed` status.
- The Job shows `1/1` under `COMPLETIONS`.
- The Pod is **not deleted** — it stays in `Completed` so you can read its logs.
- The Job's `restartPolicy` must be `Never` or `OnFailure` — `Always` is not allowed
  because it would contradict the finite nature of the work.

```bash
kubectl logs job/hello-job
kubectl describe job hello-job
```

### restartPolicy: Never vs OnFailure

The two valid policies produce different behavior when a container fails:

| Policy      | On failure                                                                                  | Pod count               |
|-------------|---------------------------------------------------------------------------------------------|-------------------------|
| `Never`     | The Job controller creates a **new Pod**. The failed Pod stays for inspection.              | Grows with each failure |
| `OnFailure` | The **kubelet** restarts the container in the **same Pod**. The restart counter increments. | Stays at 1              |

`Never` is better for debugging because you can inspect each failed Pod's logs. Use
`OnFailure` when you want cleaner Pod listings and the failure cause is already visible
in the container's output.

### Controlling retries: backoffLimit

By default, a Job retries up to **6 times** before giving up. Each retry uses an
exponential backoff (10s, 20s, 40s, …).

```yaml
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
        - name: flaky
          image: busybox:1.37
          command: ["sh", "-c", "echo 'Attempt...' && exit 1"]
      restartPolicy: Never
```

```bash
kubectl apply -f backoff-job.yaml
kubectl get pods --selector=job-name=backoff-job --watch
```

After 3 failed Pods, the Job stops retrying and its condition becomes `Failed`.

```bash
kubectl describe job backoff-job | grep -A5 Conditions
```

### Active deadline

`activeDeadlineSeconds` sets an absolute time limit for the entire Job. If the Job has
not completed within this period, Kubernetes terminates all its Pods and marks the Job
as failed.

```yaml
spec:
  activeDeadlineSeconds: 30
  backoffLimit: 5
  template:
    spec:
      containers:
        - name: slow
          image: busybox:1.37
          command: ["sh", "-c", "echo 'Working...' && sleep 120"]
      restartPolicy: Never
```

This is a safety net for Jobs that might hang forever — a network call that never
returns, a lock that is never released. It takes precedence over `backoffLimit`.

### Multiple completions and parallelism

A Job can require multiple successful completions and run them in parallel:

```yaml
spec:
  completions: 5
  parallelism: 2
```

This runs 2 Pods at a time until 5 total have succeeded. Useful for batch processing
where each Pod handles a chunk of work. For initialization tasks like `wallet-setup`,
you want `completions: 1` (the default).

### TTL cleanup

By default, completed Job Pods accumulate. The `ttlSecondsAfterFinished` field
automatically deletes the Job (and its Pods) after the specified duration:

```yaml
spec:
  ttlSecondsAfterFinished: 300
```

This deletes the Job 5 minutes after it finishes (whether it succeeded or failed). Set
it long enough to inspect logs if needed, short enough to avoid clutter.

---

## CronJobs

A CronJob creates a Job on a recurring schedule, using standard cron syntax:

```yaml
# cron-demo.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: periodic-report
  labels:
    chapter: "10"
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            chapter: "10"
        spec:
          containers:
            - name: reporter
              image: busybox:1.37
              command: ["sh", "-c", "echo 'Report generated at' $(date)"]
          restartPolicy: Never
```

```bash
kubectl apply -f cron-demo.yaml
kubectl get cronjobs
```

Key fields:

| Field                        | Purpose                                                          | Default    |
|------------------------------|------------------------------------------------------------------|------------|
| `schedule`                   | Cron expression (minute hour day month weekday)                  | (required) |
| `concurrencyPolicy`          | `Allow`, `Forbid` (skip if previous still running), or `Replace` | `Allow`    |
| `startingDeadlineSeconds`    | How late a Job can start if it misses its window                 | unlimited  |
| `successfulJobsHistoryLimit` | Number of completed Jobs to keep                                 | 3          |
| `failedJobsHistoryLimit`     | Number of failed Jobs to keep                                    | 1          |

CronJobs are not relevant for `wallet-setup` (which runs once) but they matter for
operational tasks — rotating logs, generating backups, cleaning expired sessions.

---

## Init containers

Init containers run **before** any regular container in a Pod starts. They execute
sequentially — each must complete successfully before the next one begins. If an init
container fails, the kubelet retries it (subject to the Pod's `restartPolicy`), and
no regular container starts until all init containers have succeeded.

### Why they exist

A regular container is designed to run indefinitely (or until the Job's work is done).
An init container is designed to **prepare the environment** and then exit. Common uses:

- Wait for a dependency to become available (a database, an API, a DNS record).
- Generate or fetch configuration that the main container needs.
- Set up filesystem permissions or populate a shared volume.
- Run schema migrations before starting the application.

### Structure

Init containers appear in the Pod spec alongside but separate from regular containers:

```yaml
# init-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
  labels:
    chapter: "10"
spec:
  initContainers:
    - name: wait-for-service
      image: busybox:1.37
      command: ["sh", "-c", "until nslookup mydb.default.svc.cluster.local; do echo 'Waiting for mydb...'; sleep 2; done"]
    - name: prepare-config
      image: busybox:1.37
      command: ["sh", "-c", "echo 'db_host=mydb' > /config/app.conf"]
      volumeMounts:
        - name: config-vol
          mountPath: /config
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "cat /config/app.conf && sleep 3600"]
      volumeMounts:
        - name: config-vol
          mountPath: /config
  volumes:
    - name: config-vol
      emptyDir: {}
```

Observations:

- `wait-for-service` loops until DNS resolution succeeds. Only then does
  `prepare-config` run.
- `prepare-config` writes a file into the shared `emptyDir`. Only then does `app` start.
- Init containers can use different images than the main containers — a common pattern is
  a small utility image for the init work and the full application image for the main
  container.
- Init containers share the Pod's volumes, which is how they pass data to the main
  containers.

### What init containers cannot do

Init containers are **per-Pod** — they enforce ordering within a single Pod. They cannot
enforce ordering between Pods. This is the fundamental difference from Docker Compose's
`depends_on`.

```text
Docker Compose                         Kubernetes
┌────────────────────────┐             ┌─────────────────────────────────┐
│ depends_on:            │             │ No cross-Pod ordering.          │
│   service_a:           │             │                                 │
│     condition:         │             │ Use readiness probes + retries  │
│       service_healthy  │             │ so that consumers tolerate a    │
│                        │             │ dependency that is not ready    │
│ → Compose waits for A  │             │ yet, rather than requiring a    │
│   before starting B.   │             │ specific startup order.         │
└────────────────────────┘             └─────────────────────────────────┘
```

---

## Startup ordering in Kubernetes

### Why there is no `depends_on`

Docker Compose runs on a single machine and controls the startup order of all
containers. Kubernetes runs on a **distributed** cluster where:

- Pods are scheduled independently — there is no orchestrator that serializes Pod
  creation across the cluster.
- A dependency might be on a different node, in a different namespace, or managed by a
  different team.
- A dependency might be temporarily unavailable due to a rollout, a crash, or a network
  partition. Requiring strict ordering would make the entire system fragile.

Kubernetes replaces startup ordering with three mechanisms:

### 1. Readiness probes (the primary mechanism)

A Service only sends traffic to Pods that pass their readiness probe. A consumer that
connects through a Service will not reach a backend until the backend is actually ready.
You already configured these in chapter 8.

### 2. Init containers (local prerequisites)

If a Pod needs to wait for something before starting — "my database must be reachable"
— an init container can poll until the condition is true. This blocks the individual
Pod's startup, not the entire cluster.

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.37
    command:
      - sh
      - -c
      - |
        until nc -z postgres-svc 5432; do
          echo "Waiting for postgres..."
          sleep 3
        done
```

### 3. Application-level retries (the resilient pattern)

The most Kubernetes-native approach is for the application itself to handle a missing
dependency gracefully — retry with backoff, use circuit breakers, or degrade instead of
crashing. This is not always under your control (you do not write the application), but
when it is, it produces the most resilient system.

### Combining the three

For the Signet Playground capstone:

| Compose pattern                                         | Kubernetes replacement                                                                             |
|---------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| `node` starts first, others depend on `service_healthy` | Node Pod has readiness probe. Other Pods use init containers to wait for the node Service endpoint |
| `wallet-setup` runs after `node` is healthy             | Kubernetes Job with init container that waits for node readiness                                   |
| `miner` starts after `wallet-setup` completes           | Miner Deployment with init container that checks wallet existence                                  |
| `fulcrum` depends on `node` healthy + `miner` started   | Fulcrum Pod with init container that waits for node RPC                                            |

The key insight: each Pod is responsible for its own prerequisites. There is no global
ordering — just local checks.

---

## Designing idempotent Jobs

A Job might run more than once — the first Pod might fail after partial completion and
Kubernetes retries it. If the Job's work is not idempotent, the retry can cause
duplicates, conflicts, or corruption.

### What idempotent means

An operation is idempotent if running it once produces the same result as running it
multiple times. Examples:

| Operation                                      | Idempotent? | Why                                         |
|------------------------------------------------|-------------|---------------------------------------------|
| `CREATE TABLE IF NOT EXISTS users (...)`       | Yes         | The second run does nothing                 |
| `CREATE TABLE users (...)`                     | No          | The second run fails with "already exists"  |
| `INSERT INTO users (id, name) VALUES (1, 'a')` | No          | The second run fails or creates a duplicate |
| `INSERT ... ON CONFLICT DO NOTHING`            | Yes         | The second run skips silently               |
| `createwallet wallet_name=X blank=true`        | No          | The second run fails with "already exists"  |
| Check if wallet exists, create only if missing | Yes         | The second run skips                        |

### Making wallet-setup idempotent

The current Docker Compose `wallet-setup` is **not idempotent** — if you run it twice,
`createwallet` fails because the wallet already exists. This is fine in Compose because
`restart: no` means it only runs once. But a Kubernetes Job might retry, so the script
must handle the "already done" case.

Strategy:

```text
1. Try to create the wallet.
2. If it already exists, that is not an error — continue.
3. Import descriptors (importdescriptors is idempotent — re-importing
   the same descriptor with the same range is a no-op).
4. Verify the wallet has the expected descriptors.
5. Exit 0.
```

---

## Hands-on: Jobs and init containers

### Exercise 1: Job lifecycle

```yaml
# job-lifecycle.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-lifecycle
  labels:
    chapter: "10"
spec:
  backoffLimit: 4
  template:
    metadata:
      labels:
        chapter: "10"
    spec:
      containers:
        - name: worker
          image: busybox:1.37
          command:
            - sh
            - -c
            - |
              ATTEMPT=$(cat /tmp/attempt 2>/dev/null || echo 0)
              echo "Attempt: $ATTEMPT"
              if [ "$ATTEMPT" -lt 2 ]; then
                echo "Simulating failure..."
                exit 1
              fi
              echo "Success on attempt $ATTEMPT"
      restartPolicy: Never
```

```bash
kubectl apply -f job-lifecycle.yaml
kubectl get pods --selector=job-name=job-lifecycle --watch
```

**Questions to answer:**

1. How many Pods does the Job create before it succeeds (or fails)?
2. What status do the failed Pods show?
3. What does `kubectl describe job job-lifecycle` show under `Pods Statuses`?

**Key insight:** each Pod in a `restartPolicy: Never` Job gets its own isolated
filesystem. The `ATTEMPT` counter in `/tmp/attempt` resets every time because each retry
is a new Pod. This Job will exhaust its `backoffLimit` and fail — a deliberate design to
illustrate the point. True retry state must live outside the Pod (in a database, a
ConfigMap, or the condition being checked).

### Exercise 2: Job with OnFailure

```yaml
# job-onfailure.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-onfailure
  labels:
    chapter: "10"
spec:
  backoffLimit: 4
  template:
    metadata:
      labels:
        chapter: "10"
    spec:
      containers:
        - name: worker
          image: busybox:1.37
          command:
            - sh
            - -c
            - |
              echo "Container restart count (from Pod status, not visible here)"
              echo "Simulating failure..."
              exit 1
      restartPolicy: OnFailure
```

```bash
kubectl apply -f job-onfailure.yaml
kubectl get pods --selector=job-name=job-onfailure --watch
```

**Compare with Exercise 1:**

- How many Pods are created?
- Where does the restart counter show up?
- When does the Job finally give up?

```bash
kubectl describe pod -l job-name=job-onfailure | grep -A2 "Restart Count"
kubectl describe job job-onfailure | grep -A5 Conditions
```

### Exercise 3: init container ordering

```yaml
# init-ordering.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-ordering
  labels:
    chapter: "10"
spec:
  initContainers:
    - name: step-1
      image: busybox:1.37
      command: ["sh", "-c", "echo 'Step 1: preparing...' && sleep 5 && echo 'done' > /shared/step1"]
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: step-2
      image: busybox:1.37
      command: ["sh", "-c", "cat /shared/step1 && echo 'Step 2: configuring...' && sleep 3 && echo 'ready' > /shared/step2"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "echo 'App started. Step 2 status:' && cat /shared/step2 && sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

```bash
kubectl apply -f init-ordering.yaml
kubectl get pod init-ordering --watch
```

**Observe the Pod status transitions:**

```text
NAME             READY   STATUS            RESTARTS   AGE
init-ordering    0/1     Init:0/2          0          2s
init-ordering    0/1     Init:1/2          0          7s
init-ordering    0/1     PodInitializing   0          10s
init-ordering    1/1     Running           0          11s
```

```bash
kubectl logs init-ordering -c step-1
kubectl logs init-ordering -c step-2
kubectl logs init-ordering -c app
```

Each init container's logs are accessible individually. The app container started only
after both init containers succeeded, and it found the data they wrote.

### Exercise 4: init container waiting for a Service

This exercise demonstrates the most common init container pattern — waiting for a
dependency.

```bash
# Start the Pod first — it will block in the init phase
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: waiter-pod
  labels:
    chapter: "10"
spec:
  initContainers:
    - name: wait-for-backend
      image: busybox:1.37
      command:
        - sh
        - -c
        - |
          echo "Waiting for backend service..."
          until nslookup backend-svc.default.svc.cluster.local; do
            echo "backend-svc not found yet, retrying in 3s..."
            sleep 3
          done
          echo "backend-svc is available!"
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "echo 'App started — backend is reachable' && sleep 3600"]
EOF
```

```bash
kubectl get pod waiter-pod --watch
```

The Pod stays in `Init:0/1`. The init container is looping, waiting for DNS resolution.

Now create the Service and a backing Pod:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  labels:
    chapter: "10"
spec:
  selector:
    app: backend
  ports:
    - port: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  labels:
    app: backend
    chapter: "10"
spec:
  containers:
    - name: server
      image: busybox:1.37
      command: ["sh", "-c", "echo 'Backend running' && sleep 3600"]
EOF
```

Within a few seconds, the init container's DNS lookup succeeds and the app container
starts.

```bash
kubectl logs waiter-pod -c wait-for-backend
kubectl logs waiter-pod -c app
```

### Exercise 5: an idempotent database init Job

This exercise models the `wallet-setup` pattern — a one-time initialization task that
must be safe to retry.

```yaml
# db-init-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-init
  labels:
    chapter: "10"
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        chapter: "10"
    spec:
      initContainers:
        - name: wait-for-db
          image: busybox:1.37
          command:
            - sh
            - -c
            - |
              echo "Waiting for database to accept connections..."
              until nc -z db-svc 5432; do
                sleep 2
              done
              echo "Database is ready."
      containers:
        - name: migrate
          image: busybox:1.37
          command:
            - sh
            - -c
            - |
              echo "=== Database initialization ==="
              echo "Step 1: CREATE TABLE IF NOT EXISTS users"
              echo "Step 2: INSERT INTO config ON CONFLICT DO NOTHING"
              echo "Step 3: Verify schema version"
              echo "=== Initialization complete ==="
          env:
            - name: DB_HOST
              value: db-svc
            - name: DB_PORT
              value: "5432"
      restartPolicy: Never
```

**Design observations (discuss, don't run — there is no actual database):**

1. The init container waits for the database Service to be reachable.
2. The main container uses `IF NOT EXISTS` and `ON CONFLICT` — both idempotent.
3. If the main container fails mid-migration, the Job retries and the idempotent
   statements skip what was already done.
4. `backoffLimit: 3` prevents infinite retries.
5. The Job does not care whether the database was started before or after it — the init
   container simply waits.

### Exercise 6: Job that depends on another Job

Kubernetes has no built-in "Job B waits for Job A" primitive. The most common workaround
is an init container that checks for the result of the first Job:

```yaml
# dependent-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: second-job
  labels:
    chapter: "10"
spec:
  template:
    metadata:
      labels:
        chapter: "10"
    spec:
      initContainers:
        - name: wait-for-first
          image: bitnami/kubectl:1.31
          command:
            - sh
            - -c
            - |
              echo "Waiting for first-job to complete..."
              until kubectl get job first-job -o jsonpath='{.status.succeeded}' | grep -q 1; do
                echo "first-job not yet complete, waiting..."
                sleep 5
              done
              echo "first-job completed successfully."
      containers:
        - name: worker
          image: busybox:1.37
          command: ["sh", "-c", "echo 'Second job running — first job is done.'"]
      restartPolicy: Never
```

**Important caveat:** this init container uses `kubectl`, which means the Pod's
ServiceAccount must have permission to `get` Jobs. In chapter 14 (Security foundations)
you will learn how to grant this with RBAC. For now, the pattern is what matters — the
init container is a polling loop, just like the DNS check in exercise 4.

---

## Failure scenarios

### Scenario 1: Job exceeds backoffLimit

```yaml
# fail01-exhausted.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: doomed-job
  labels:
    chapter: "10"
spec:
  backoffLimit: 2
  template:
    metadata:
      labels:
        chapter: "10"
    spec:
      containers:
        - name: fail
          image: busybox:1.37
          command: ["sh", "-c", "echo 'Failing on purpose' && exit 1"]
      restartPolicy: Never
```

```bash
kubectl apply -f fail01-exhausted.yaml
kubectl get pods --selector=job-name=doomed-job --watch
```

After 3 Pods (the original + 2 retries), the Job stops:

```bash
kubectl describe job doomed-job | grep -A5 Conditions
```

The condition reads `type: Failed` with reason `BackoffLimitExceeded`. The Pods remain
for inspection.

**Recovery:** fix the underlying cause, delete the failed Job, and apply a corrected
version. Jobs are immutable — you cannot edit `spec.template` on an existing Job.

### Scenario 2: init container stuck forever

```yaml
# fail02-stuck-init.yaml
apiVersion: v1
kind: Pod
metadata:
  name: stuck-init
  labels:
    chapter: "10"
spec:
  initContainers:
    - name: wait-forever
      image: busybox:1.37
      command: ["sh", "-c", "echo 'Waiting for service that will never appear...' && sleep infinity"]
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "echo 'This will never run'"]
```

```bash
kubectl apply -f fail02-stuck-init.yaml
kubectl get pod stuck-init --watch
```

The Pod stays in `Init:0/1` indefinitely. The app container never starts.

**Diagnose it:**

```bash
kubectl describe pod stuck-init
kubectl logs stuck-init -c wait-forever
```

The events show the init container started but never completed. This is why init
containers that wait for external dependencies should have their own timeout:

```bash
# Better pattern: timeout after 60 seconds
command:
  - sh
  - -c
  - |
    TIMEOUT=60
    ELAPSED=0
    until nslookup myservice.default.svc.cluster.local; do
      ELAPSED=$((ELAPSED + 3))
      if [ "$ELAPSED" -ge "$TIMEOUT" ]; then
        echo "ERROR: timed out waiting for myservice"
        exit 1
      fi
      sleep 3
    done
```

If the init container is inside a Job, the failure triggers the Job's retry logic. If
it is inside a plain Pod, the kubelet restarts the init container (with `restartPolicy:
Always` or `OnFailure`). Either way, the failure is visible rather than silently hanging.

### Scenario 3: non-idempotent Job retry causes duplicate work

Consider a Job that inserts a row without checking for duplicates:

```text
1. Job Pod 1 starts, inserts row with id=42, then crashes before exiting cleanly.
2. Kubernetes sees no successful completion — it creates Pod 2.
3. Pod 2 inserts row with id=42 again → duplicate or primary key violation.
```

**Diagnose it:** the second Pod fails with a database constraint error that looks
unrelated to the original crash. The fix is to make the operation idempotent:
`INSERT ... ON CONFLICT DO NOTHING` or check-then-act.

---

## Signet Playground: wallet-setup as a Job

The current Docker Compose `wallet-setup` service:

1. Runs after `node` is healthy (`depends_on` with `condition: service_healthy`).
2. Creates a blank descriptor wallet named `BBO`.
3. Imports three Taproot descriptors (receiving, change, mining).
4. Exits with `restart: no`.
5. `miner` depends on `wallet-setup` with `condition: service_completed_successfully`.

### Translation to Kubernetes

```text
┌─────────────────────────────────────────────────────┐
│ Job: wallet-setup                                   │
│                                                     │
│  initContainer: wait-for-node                       │
│  ┌───────────────────────────────────────────────┐  │
│  │ Poll node Service RPC until it responds       │  │
│  └───────────────────────────────────────────────┘  │
│                     │                               │
│                     ▼                               │
│  container: setup                                   │
│  ┌───────────────────────────────────────────────┐  │
│  │ 1. createwallet (skip if already exists)      │  │
│  │ 2. importdescriptors (idempotent)             │  │
│  │ 3. verify descriptors are correct             │  │
│  │ 4. exit 0                                     │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Deployment: miner                                   │
│                                                     │
│  initContainer: wait-for-wallet                     │
│  ┌───────────────────────────────────────────────┐  │
│  │ Poll node RPC: does wallet BBO exist and      │  │
│  │ have the expected descriptors?                │  │
│  └───────────────────────────────────────────────┘  │
│                     │                               │
│                     ▼                               │
│  container: miner                                   │
│  ┌───────────────────────────────────────────────┐  │
│  │ Run the mining loop                           │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Design decisions

**The init container replaces `depends_on`.** Instead of Compose telling the miner
"wallet-setup completed successfully", the miner's init container checks the actual
condition — "does wallet BBO exist with the right descriptors?" This is more resilient
because:

- It survives cluster restarts (the wallet already exists, so the check passes
  immediately).
- It does not require knowing which Job created the wallet.
- It works even if someone created the wallet manually.

**The Job is idempotent.** If `createwallet` fails because the wallet already exists,
the script continues. `importdescriptors` is already idempotent for the same
descriptor/range combination.

**The miner does not depend on the Job — it depends on the state.** This is the
fundamental Kubernetes pattern: check for the condition, not for the process that
creates it. In Docker Compose, the dependency graph is about process ordering. In
Kubernetes, the dependency graph is about state readiness.

### Cookie access

The wallet-setup Job needs the RPC cookie to talk to the node, just like every other
RPC consumer. The cookie challenge from chapter 9 applies here too — the Job Pod must
mount the same cookie that the node generates. The capstone chapter will resolve this;
for now, assume the cookie is available through some shared mechanism (emptyDir in the
same Pod, a Secret, or a shared PVC).

---

## Cleanup

```bash
kubectl delete all -l chapter=10
```

As always, `delete all` covers Pods, Services, Deployments, ReplicaSets, and Jobs. It
does not cover CronJobs:

```bash
kubectl delete cronjob -l chapter=10
```

---

## Knowledge check

1. What `restartPolicy` values are valid for a Job Pod? Why is `Always` not allowed?
2. Explain the difference between `backoffLimit` and `activeDeadlineSeconds`. When would
   you use both?
3. A Job with `restartPolicy: Never` and `backoffLimit: 3` fails. How many Pods will
   exist after the Job gives up? What status will each Pod show?
4. An init container in a Deployment Pod loops forever waiting for a Service that does
   not exist. What happens to the Deployment's rollout? How do you diagnose it?
5. Why is there no `depends_on` in Kubernetes? What mechanisms replace it?
6. The Signet Playground `wallet-setup` runs `createwallet` and then `importdescriptors`.
   If the Pod crashes between these two steps and the Job retries, what happens? How
   would you make the sequence safe?
7. A CronJob's `concurrencyPolicy` is set to `Allow`. The Job it spawns takes 10 minutes,
   but the schedule is `*/5 * * * *`. What happens?
8. The miner Deployment's init container checks "does wallet BBO exist?" rather than
   "did the wallet-setup Job succeed?". Why is this better?

---

## Summary

Jobs model finite work — run to completion and stop. Unlike Deployments, they do not
restart successful Pods. `backoffLimit` and `activeDeadlineSeconds` control how many
retries and how long the Job runs. CronJobs create Jobs on a schedule.

Init containers run before regular containers within the same Pod, enforcing sequential
prerequisites. They replace Docker Compose's `depends_on` at the Pod level but cannot
order Pods against each other — cross-Pod ordering uses readiness probes, Service DNS,
and application-level retries.

Idempotent design is critical for Jobs that may retry: use `IF NOT EXISTS`, `ON CONFLICT
DO NOTHING`, and check-then-act patterns so that partial progress does not break a
retry.

Chapter 11 introduces StatefulSets — controllers that give Pods stable names, stable
storage, and ordered deployment, bridging the gap between Deployments and the needs of
stateful applications like databases.
