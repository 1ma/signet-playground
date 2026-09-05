# Chapter 07. Configuration and Secrets

## Learning objectives

After completing this chapter you should be able to:

- Create ConfigMaps from literals, files, and directories.
- Create Secrets and explain how they differ from ConfigMaps at rest and in transit.
- Inject configuration into Pods as environment variables and as mounted files.
- Describe update propagation behavior for mounted ConfigMaps and explain why environment
  variables do not update automatically.
- Force a rollout when configuration changes using annotation checksums.
- Evaluate when to use environment variables versus mounted files.
- Identify the configuration and secret material in the Signet Playground stack.

---

## Why externalize configuration

In Docker Compose, you inject configuration through `environment:` blocks, bind-mounted
files (`./bitcoin.conf:/home/bitcoin/.bitcoin/bitcoin.conf`), and `.env` files. The
configuration lives outside the image so the same image can serve different environments
without rebuilding.

Kubernetes follows the same principle with two dedicated resource types:

| Resource      | Purpose                                           | Encoded    |
|---------------|---------------------------------------------------|------------|
| **ConfigMap** | Non-sensitive key-value pairs and files           | Plain text |
| **Secret**    | Sensitive data (passwords, tokens, cookies, keys) | Base64     |

Both are API objects that live in a namespace. Both can be injected into Pods as
environment variables or as files mounted into the container filesystem.

---

## ConfigMaps

### Creating ConfigMaps

From literals:

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=CACHE_TTL=300
```

From a file:

```bash
kubectl create configmap bitcoin-conf --from-file=bitcoin.conf
```

The key becomes the filename (`bitcoin.conf`) and the value becomes the file contents.
You can override the key name:

```bash
kubectl create configmap bitcoin-conf --from-file=config.conf=bitcoin.conf
```

From a directory (one key per file):

```bash
kubectl create configmap all-configs --from-file=./configs/
```

### ConfigMap manifest

The declarative equivalent:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    chapter: "07"
data:
  LOG_LEVEL: debug
  CACHE_TTL: "300"         # values are always strings

  bitcoin.conf: |
    chain=signet
    rpccookiefile=/var/tmp/.cookie
```

The `data` field holds string key-value pairs. For files, the key is the filename and the
value is the file content (use the `|` block scalar for multi-line values).

For binary content, use `binaryData` instead of `data` — values are base64-encoded.

### Injecting as environment variables

Individual keys:

```yaml
spec:
  containers:
    - name: app
      image: nginx:1.27
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

All keys at once:

```yaml
spec:
  containers:
    - name: app
      image: nginx:1.27
      envFrom:
        - configMapRef:
            name: app-config
```

With `envFrom`, every key in the ConfigMap becomes an environment variable. Keys that are
not valid environment variable names (contain dots, dashes) are silently skipped.

### Injecting as mounted files

```yaml
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

Each key in the ConfigMap becomes a file under `/etc/app/`. The file `LOG_LEVEL` contains
`debug`, the file `bitcoin.conf` contains the full config.

To mount a single key as a specific file without replacing the entire directory:

```yaml
volumeMounts:
  - name: config-vol
    mountPath: /etc/app/bitcoin.conf
    subPath: bitcoin.conf
    readOnly: true
```

**Warning:** `subPath` mounts do not receive automatic updates when the ConfigMap changes.
Only full-directory mounts are updated automatically.

### Mounting with specific permissions

```yaml
volumes:
  - name: config-vol
    configMap:
      name: app-config
      defaultMode: 0644       # octal file permissions
      items:                  # mount only selected keys
        - key: bitcoin.conf
          path: bitcoin.conf
          mode: 0400          # per-file override
```

---

## Secrets

### How Secrets differ from ConfigMaps

Secrets are structurally almost identical to ConfigMaps, but with important differences:

| Aspect                   | ConfigMap      | Secret                                |
|--------------------------|----------------|---------------------------------------|
| Field for string data    | `data`         | `stringData` (write-only convenience) |
| Field for encoded data   | `binaryData`   | `data` (base64-encoded)               |
| Stored in etcd           | Plain text     | Base64 (NOT encryption)               |
| Mounted file permissions | `0644` default | `0644` default                        |
| Memory-backed tmpfs      | No             | Yes (by default)                      |
| Size limit               | 1 MiB          | 1 MiB                                 |

**Base64 is not encryption.** Anyone who can read the Secret object can decode it
instantly. Kubernetes Secrets are about **access control** (RBAC restricts who can read
them) and **lifecycle separation** (secrets are managed separately from application
manifests), not about cryptographic protection.

For actual encryption at rest, you need to enable etcd encryption or use an external
secret manager (Vault, cloud KMS). Chapter 14 discusses this further.

### Creating Secrets

From literals:

```bash
kubectl create secret generic db-creds \
  --from-literal=username=mempool \
  --from-literal=password=mempool
```

From a file:

```bash
kubectl create secret generic rpc-cookie --from-file=.cookie=/var/tmp/.cookie
```

### Secret manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
  labels:
    chapter: "07"
type: Opaque
stringData:              # write-only convenience — plain text in the manifest
  username: mempool
  password: mempool
```

When you `kubectl apply` this, Kubernetes stores the values base64-encoded in etcd. If
you `kubectl get secret db-creds -o yaml`, you see the `data` field with base64:

```bash
kubectl get secret db-creds -o yaml
# data:
#   username: bWVtcG9vbA==
#   password: bWVtcG9vbA==
```

Decode:

```bash
echo bWVtcG9vbA== | base64 -d
# mempool
```

### Secret types

| Type                                  | Purpose                              |
|---------------------------------------|--------------------------------------|
| `Opaque`                              | Generic (default)                    |
| `kubernetes.io/basic-auth`            | Username/password                    |
| `kubernetes.io/tls`                   | TLS certificate + key                |
| `kubernetes.io/dockerconfigjson`      | Container registry credentials       |
| `kubernetes.io/service-account-token` | Auto-generated ServiceAccount tokens |

The type is mostly for documentation and tooling — `Opaque` works for everything.

### Injecting Secrets

The syntax is identical to ConfigMaps, with `secretKeyRef` instead of `configMapKeyRef`:

As environment variables:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: password
```

As mounted files:

```yaml
volumes:
  - name: cookie-vol
    secret:
      secretName: rpc-cookie
      defaultMode: 0400     # restrictive permissions for sensitive files
```

Secrets mounted as volumes are backed by **tmpfs** (in-memory filesystem) — they are
never written to the node's disk.

---

## Update behavior

This is one of the most important and confusing aspects of ConfigMaps and Secrets.

### Mounted volumes (full directory)

When you update a ConfigMap or Secret, Kubernetes **automatically** updates the mounted
files. The kubelet polls for changes and replaces the content via a symlink swap — the
mount point contains a symlink chain:

```text
/etc/app/
  bitcoin.conf → ..data/bitcoin.conf
  ..data → ..2024_09_04_12_00_00.123456
  ..2024_09_04_12_00_00.123456/
    bitcoin.conf         ← actual file content
```

When the ConfigMap changes, kubelet creates a new timestamped directory with the new
content and atomically swaps the `..data` symlink. The propagation delay is controlled
by the kubelet's sync period (default: up to 60 seconds + cache TTL).

**However:** the application must notice the change. Most applications read config files
once at startup and do not watch for changes. A file update without a process restart
often has no effect.

### subPath mounts

Files mounted with `subPath` are **never updated**. The file is copied at Pod start
and remains static. If you need automatic updates, do not use `subPath`.

### Environment variables

Environment variables are **never updated** after the container starts. Even if the
ConfigMap or Secret changes, the container keeps the old values until the Pod is
recreated.

### Summary of update behavior

| Injection method              | Auto-updates? | Application sees change?        |
|-------------------------------|---------------|---------------------------------|
| Full volume mount             | Yes           | Only if it re-reads the file    |
| `subPath` mount               | No            | Never                           |
| Environment variable          | No            | Never (until Pod restart)       |

### Forcing a rollout on config change

Since most applications need a restart to pick up new configuration, a common pattern
is to include a checksum of the ConfigMap in the Pod template annotations. When the
ConfigMap changes, the checksum changes, which changes the Pod template, which triggers
a rolling update:

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: "sha256-of-configmap-content"
```

Manually:

```bash
kubectl rollout restart deployment/my-app
```

This is a deliberate design choice — Kubernetes does not automatically restart Pods when
configuration changes because it cannot know whether the application handles live
reloads.

---

## Environment variables versus mounted files

| Criterion                         | Environment variables        | Mounted files              |
|-----------------------------------|------------------------------|----------------------------|
| Simple key-value pairs            | Natural fit                  | Overkill                   |
| Structured config files           | Awkward (one giant string)   | Natural fit                |
| Binary content                    | Not possible                 | Works (`binaryData`)       |
| Auto-update without restart       | No                           | Yes (full mount only)      |
| Visible in `kubectl describe`     | Yes                          | No (need to exec into Pod) |
| Visible in process table (`ps`)   | Possible (`/proc/*/environ`) | No                         |
| Convention in 12-factor apps      | Preferred                    | —                          |

For sensitive values like passwords, environment variables are more exposed — they
appear in `kubectl describe pod`, in `/proc/*/environ` inside the container, and often
end up in log output. Mounting secrets as files with restrictive permissions (0400) is
more secure.

---

## Workshop

### Exercise 1 — ConfigMap from literals and environment injection

```yaml
# examples/07-config/configmap-env.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
  labels:
    chapter: "07"
data:
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
  APP_NAME: signet-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-demo
  labels:
    chapter: "07"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: env-demo
  template:
    metadata:
      labels:
        app: env-demo
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["sh", "-c", "env | sort && sleep 3600"]
          envFrom:
            - configMapRef:
                name: app-settings
```

```bash
kubectl apply -f examples/07-config/configmap-env.yaml
kubectl wait --for=condition=Ready pod -l app=env-demo --timeout=60s
kubectl logs deploy/env-demo | grep -E 'LOG_LEVEL|MAX_CONNECTIONS|APP_NAME'
```

Verify the three variables appear in the output.

### Exercise 2 — ConfigMap as mounted file

```yaml
# examples/07-config/configmap-file.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  labels:
    chapter: "07"
data:
  default.conf: |
    server {
        listen 80;
        location / {
            return 200 "ConfigMap version 1\n";
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-demo
  labels:
    chapter: "07"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: file-demo
  template:
    metadata:
      labels:
        app: file-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: config
              mountPath: /etc/nginx/conf.d
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: nginx-config
```

```bash
kubectl apply -f examples/07-config/configmap-file.yaml
kubectl wait --for=condition=Ready pod -l app=file-demo --timeout=60s
kubectl exec deploy/file-demo -- curl -s localhost
# ConfigMap version 1
```

### Exercise 3 — Observe live update (and its limits)

Update the ConfigMap:

```bash
kubectl edit configmap nginx-config
# Change "version 1" to "version 2" in default.conf
```

Wait up to 60 seconds for the file to update:

```bash
kubectl exec deploy/file-demo -- cat /etc/nginx/conf.d/default.conf
# Should show "version 2" after propagation
```

But nginx is still serving the old response:

```bash
kubectl exec deploy/file-demo -- curl -s localhost
# Still "ConfigMap version 1"
```

The file updated, but nginx loaded its config at startup and does not watch for changes.
Force nginx to reload:

```bash
kubectl exec deploy/file-demo -- nginx -s reload
kubectl exec deploy/file-demo -- curl -s localhost
# Now "ConfigMap version 2"
```

This is why most teams use `kubectl rollout restart` — it recreates Pods with the latest
config, which is more reliable than hoping every application watches its config files.

### Exercise 4 — Secret creation and mounting

```yaml
# examples/07-config/secret-demo.yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
  labels:
    chapter: "07"
type: Opaque
stringData:
  username: admin
  password: "s3cret-p4ss!"
  cookie: "__cookie__:abc123def456"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-demo
  labels:
    chapter: "07"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-demo
  template:
    metadata:
      labels:
        app: secret-demo
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["sh", "-c", "ls -la /secrets/ && cat /secrets/cookie && echo && sleep 3600"]
          volumeMounts:
            - name: secrets
              mountPath: /secrets
              readOnly: true
      volumes:
        - name: secrets
          secret:
            secretName: demo-secret
            defaultMode: 0400
```

```bash
kubectl apply -f examples/07-config/secret-demo.yaml
kubectl wait --for=condition=Ready pod -l app=secret-demo --timeout=60s
kubectl logs deploy/secret-demo
```

Verify:

```bash
# Files exist with restrictive permissions
kubectl exec deploy/secret-demo -- ls -la /secrets/

# Content is plain text inside the container (base64 is only in etcd)
kubectl exec deploy/secret-demo -- cat /secrets/cookie

# The mount is tmpfs (in-memory)
kubectl exec deploy/secret-demo -- df -T /secrets/
# Type: tmpfs
```

### Exercise 5 — Environment variable from Secret (and why to avoid it)

```yaml
# examples/07-config/secret-env.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-env-demo
  labels:
    chapter: "07"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-env-demo
  template:
    metadata:
      labels:
        app: secret-env-demo
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["sh", "-c", "echo password is $DB_PASSWORD && sleep 3600"]
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: demo-secret
                  key: password
```

```bash
kubectl apply -f examples/07-config/secret-env.yaml
kubectl wait --for=condition=Ready pod -l app=secret-env-demo --timeout=60s
```

Now observe the exposure:

```bash
# The password appears in plain text in describe output
kubectl describe pod -l app=secret-env-demo | grep DB_PASSWORD

# It also appears in the container logs (because the command echoes it)
kubectl logs deploy/secret-env-demo

# And in the process environment inside the container
kubectl exec deploy/secret-env-demo -- cat /proc/1/environ | tr '\0' '\n' | grep DB_PASSWORD
```

This is why mounted files with restrictive permissions are preferred for sensitive data.

---

## Failure scenarios

### Scenario 1 — Missing ConfigMap reference

```yaml
# examples/07-config/missing-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: missing-cm
  labels:
    chapter: "07"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sleep", "3600"]
      env:
        - name: SETTING
          valueFrom:
            configMapKeyRef:
              name: does-not-exist
              key: value
```

```bash
kubectl apply -f examples/07-config/missing-configmap.yaml
kubectl get pod missing-cm
kubectl describe pod missing-cm | tail -10
```

The Pod stays in `CreateContainerConfigError`. The Events section shows exactly which
ConfigMap and key are missing.

Fix it by creating the missing ConfigMap:

```bash
kubectl create configmap does-not-exist --from-literal=value=hello
# Pod starts automatically
```

To make the reference optional (Pod starts even if the ConfigMap does not exist):

```yaml
configMapKeyRef:
  name: does-not-exist
  key: value
  optional: true           # Pod starts; variable is simply absent
```

### Scenario 2 — envFrom with invalid key names

```yaml
# examples/07-config/invalid-keys.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bad-keys
  labels:
    chapter: "07"
data:
  VALID_KEY: works
  invalid.key: "has a dot"
  also-invalid: "has a dash"
---
apiVersion: v1
kind: Pod
metadata:
  name: bad-keys-pod
  labels:
    chapter: "07"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "env | sort && sleep 3600"]
      envFrom:
        - configMapRef:
            name: bad-keys
```

```bash
kubectl apply -f examples/07-config/invalid-keys.yaml
kubectl wait --for=condition=Ready pod/bad-keys-pod --timeout=60s
kubectl logs bad-keys-pod | grep -E 'VALID|invalid|also'
```

Only `VALID_KEY` appears. The keys with dots and dashes are silently skipped because they
are not valid shell environment variable names. No error, no warning in Pod status — check
Events for a warning mentioning the skipped keys.

### Scenario 3 — subPath mount does not update

```yaml
# examples/07-config/subpath-demo.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: subpath-config
  labels:
    chapter: "07"
data:
  message: "original"
---
apiVersion: v1
kind: Pod
metadata:
  name: subpath-pod
  labels:
    chapter: "07"
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "while true; do cat /config/message; echo; sleep 5; done"]
      volumeMounts:
        - name: config
          mountPath: /config/message
          subPath: message
  volumes:
    - name: config
      configMap:
        name: subpath-config
```

```bash
kubectl apply -f examples/07-config/subpath-demo.yaml
kubectl wait --for=condition=Ready pod/subpath-pod --timeout=60s
kubectl logs subpath-pod --tail=1
# original
```

Update the ConfigMap:

```bash
kubectl patch configmap subpath-config --type merge -p '{"data":{"message":"updated"}}'
```

Wait and check again:

```bash
sleep 90
kubectl logs subpath-pod --tail=1
# Still "original" — subPath never updates
```

The only fix is to delete and recreate the Pod, or switch to a full directory mount.

---

## Signet Playground preview

The Signet Playground stack has three distinct categories of configuration:

### 1. Configuration files (ConfigMaps)

| Compose pattern                                      | Kubernetes resource                 | Injection method   |
|------------------------------------------------------|-------------------------------------|--------------------|
| `./bitcoin.conf:/home/bitcoin/.bitcoin/bitcoin.conf` | ConfigMap `bitcoin-conf`            | Volume mount       |
| `./fulcrum.conf:/etc/fulcrum.conf`                   | ConfigMap `fulcrum-conf`            | Volume mount       |
| `node.command` flags (`-chain=signet`, `-txindex=1`) | Part of the Pod spec (args/command) | Direct in manifest |

The `bitcoin.conf` file only carries client-side settings (`chain=signet`,
`rpccookiefile`). The server flags live in `compose.yml`'s `node.command` block and will
become `args:` in the Kubernetes Pod spec — they stay in the manifest, not in a ConfigMap.

`fulcrum.conf` sets the RPC endpoint, cookie path, TCP listener, and admin port. In
Kubernetes, the hostname changes from `node` to `node.default.svc.cluster.local` (or
just `node` within the same namespace), so the file content may not need to change at all.

### 2. Environment variables (ConfigMaps)

| Service        | Variables                                                                 | Sensitive? |
|----------------|---------------------------------------------------------------------------|------------|
| `wallet-setup` | `WALLET_NAME`                                                             | No         |
| `miner`        | `CLI_CMD_ARGS`, `MAX_INTERVAL`, `MINING_XPUB`                             | No         |
| `frigate`      | `CORE_SERVER`, `CORE_ZMQ_SEQUENCE_ENDPOINT`, `SERVER_TCP`, ...            | No         |
| `mempool-web`  | `BACKEND_HTTP_HOST`, `FRONTEND_HTTP_PORT`, `ROOT_NETWORK`, ...            | No         |
| `mempool-api`  | `MEMPOOL_NETWORK`, `CORE_RPC_HOST`, `ELECTRUM_HOST`, `DATABASE_HOST`, ... | Mixed      |
| `faucet`       | `FAUCET_REDIS_ENDPOINT`, `FAUCET_BITCOIN_RPC_ENDPOINT`, ...               | No         |

Most of these are connection strings and feature flags — they belong in ConfigMaps.

### 3. Secrets

| Material                  | Compose mechanism              | Kubernetes resource    | Notes                                        |
|---------------------------|--------------------------------|------------------------|----------------------------------------------|
| RPC cookie (`.cookie`)    | `cookie_dir` named volume      | Secret `rpc-cookie`    | Generated by `node`, shared via volume mount |
| Wallet descriptors (xprv) | `environment:` in wallet-setup | Secret `wallet-keys`   | Private keys — must not be in a ConfigMap    |
| MariaDB password          | `environment:` in mariadb      | Secret `mariadb-creds` | Currently `mempool` — still a credential     |

The RPC cookie is the most interesting case. In Docker Compose, `node` writes it to a
named volume (`cookie_dir`) and every service that needs RPC access mounts the same
volume. In Kubernetes, the cookie is generated at runtime by the Bitcoin node — it cannot
be a static Secret created before deployment.

Chapter 10 (init containers and Jobs) will address this by having dependent services wait
for the cookie to be available, similar to how `depends_on: service_healthy` works in
Compose. The exact mechanism (init container that waits, shared volume, or a Job that
extracts and creates a Secret) is a design decision for the capstone.

### Lifecycle summary

```text
Static (known before deploy)        Dynamic (generated at runtime)
─────────────────────────────       ──────────────────────────────
bitcoin.conf        → ConfigMap     .cookie          → shared volume or init container
fulcrum.conf        → ConfigMap
env vars            → ConfigMap
wallet xprv keys    → Secret
MariaDB password    → Secret
```

The key insight: not everything that looks like configuration can be pre-declared as a
ConfigMap or Secret. The RPC cookie is generated by `bitcoind` on startup and must be
shared through a runtime mechanism — this is a recurring pattern in systems that
generate their own authentication material.

---

## Cleanup

```bash
kubectl delete deployment,service,pod,configmap,secret -l chapter=07
```

Verify:

```bash
kubectl get all,configmap,secret
```

The default ConfigMap and ServiceAccount Secrets are expected — they are not part of this
chapter.

---

## Knowledge check

1. What is the structural difference between a ConfigMap and a Secret? Why does base64
   encoding not make Secrets secure?
2. You inject a ConfigMap as environment variables with `envFrom`. One key is named
   `app.setting`. The Pod starts but the variable is missing. Why?
3. You update a ConfigMap that is mounted as a volume. The file inside the Pod changes
   after about a minute, but the application still serves the old configuration. What
   happened?
4. You mount a ConfigMap using `subPath`. You update the ConfigMap. The file inside the
   Pod does not change no matter how long you wait. Why?
5. Why are mounted files preferred over environment variables for sensitive data like
   passwords?
6. A Pod references a Secret that does not exist. What status does the Pod show, and
   how do you fix it?
7. In the Signet Playground, the RPC cookie file is generated by `bitcoind` at startup.
   Why can it not be a pre-created Kubernetes Secret?
8. You have a Deployment with 3 replicas using `envFrom` to load a ConfigMap. You change
   a value in the ConfigMap. What do the running Pods see, and what must you do to apply
   the change?

---

## Summary

ConfigMaps and Secrets externalize configuration from container images. ConfigMaps hold
non-sensitive data; Secrets hold credentials and keys with base64 encoding (not
encryption) and RBAC-controlled access. Both can be injected as environment variables or
mounted files. Mounted files auto-update (except `subPath`), but environment variables
require a Pod restart. For the Signet Playground, static configuration maps cleanly to
ConfigMaps and Secrets, but runtime-generated material like the RPC cookie requires a
different mechanism explored in later chapters.

Chapter 08 covers health checks, resource requests and limits, and scheduling — making
workloads observable and predictable for the cluster scheduler.
