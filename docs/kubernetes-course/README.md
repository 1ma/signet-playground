# Kubernetes from First Principles

A hands-on Kubernetes course built around Minikube and the Signet Playground stack.

The course starts with small, disposable workloads so that each Kubernetes primitive can
be understood in isolation. It then migrates this repository from Docker Compose to
Kubernetes incrementally. Helm and other higher-level tools are introduced only after the
underlying manifests and controller behavior are familiar.

## Course goals

By the end of the course, you should be able to:

- Explain how Kubernetes reconciles desired and actual state.
- Use `kubectl` to inspect, troubleshoot, and modify a cluster confidently.
- Choose suitable workload controllers, networking primitives, and storage mechanisms.
- Translate a multi-service Docker Compose application into Kubernetes resources.
- Operate stateful and one-shot workloads with explicit startup dependencies.
- Package an application with Helm without losing sight of the generated Kubernetes
  resources.
- Diagnose common scheduling, networking, configuration, storage, and lifecycle failures.

## Lab environment

The primary environment is:

- Debian 13
- native Docker Engine - Community Edition
- Minikube using the Docker driver
- `kubectl`
- This repository as the capstone application

Unless a chapter says otherwise, every exercise should work on a single-node local
Minikube cluster. Cloud-provider-specific features are deliberately deferred.

## Learning approach

Each chapter should contain:

1. Learning objectives.
2. A concise conceptual explanation.
3. Commands and manifests written from scratch.
4. Inspection tasks that explain what Kubernetes created.
5. A practical workshop.
6. Failure scenarios to diagnose.
7. Cleanup instructions.
8. A short knowledge check.

The workshops should prefer observable experiments over memorizing YAML. Generated
resources may be used for exploration, but the important manifests should be understood
and maintained explicitly.

## Course roadmap

| Phase | Chapter                                                                                          | Main concepts                                              | Practical outcome                                 |
|-------|--------------------------------------------------------------------------------------------------|------------------------------------------------------------|---------------------------------------------------|
| 0     | [01. Local lab setup](#01-local-lab-setup)                                                       | Minikube, Docker driver, `kubectl`, cluster context        | A reproducible local cluster                      |
| 0     | [02. Kubernetes mental model](#02-kubernetes-mental-model)                                       | Control plane, nodes, API objects, reconciliation          | Inspect the cluster as a distributed system       |
| 1     | [03. Pods and container lifecycle](#03-pods-and-container-lifecycle)                             | Pods, container states, logs, exec, probes                 | Run and debug a single workload                   |
| 1     | [04. Declarative configuration and kubectl](#04-declarative-configuration-and-kubectl)           | YAML, metadata, spec/status, apply, diff, delete           | Manage resources declaratively                    |
| 1     | [05. Deployments and ReplicaSets](#05-deployments-and-replicasets)                               | Controllers, replicas, rollout, rollback                   | Operate a stateless application                   |
| 2     | [06. Services and cluster networking](#06-services-and-cluster-networking)                       | Service discovery, DNS, ClusterIP, NodePort, port-forward  | Connect workloads reliably                        |
| 2     | [07. Configuration and secrets](#07-configuration-and-secrets)                                   | ConfigMaps, Secrets, environment variables, mounted files  | Externalize application configuration             |
| 2     | [08. Health, resources, and scheduling](#08-health-resources-and-scheduling)                     | Probes, requests, limits, QoS, scheduling                  | Make workloads observable and schedulable         |
| 3     | [09. Persistent storage](#09-persistent-storage)                                                 | Volumes, PVs, PVCs, StorageClasses, access modes           | Persist data across Pod replacement               |
| 3     | [10. Jobs, init containers, and startup ordering](#10-jobs-init-containers-and-startup-ordering) | Jobs, init containers, idempotency, readiness              | Model setup tasks and dependencies                |
| 3     | [11. StatefulSets and stable identity](#11-statefulsets-and-stable-identity)                     | StatefulSets, headless Services, stable storage and DNS    | Operate a stateful replicated workload            |
| 4     | [12. Namespaces, labels, and policy boundaries](#12-namespaces-labels-and-policy-boundaries)     | Labels, selectors, annotations, namespaces, quotas         | Organize and query a growing application          |
| 4     | [13. Ingress and local traffic exposure](#13-ingress-and-local-traffic-exposure)                 | Ingress, ingress controller, hostnames, TLS overview       | Expose HTTP services through one entry point      |
| 4     | [14. Security foundations](#14-security-foundations)                                             | ServiceAccounts, RBAC, security contexts, secret handling  | Apply least privilege to the lab                  |
| 5     | [15. Troubleshooting and observability](#15-troubleshooting-and-observability)                   | Events, logs, metrics, debug containers, failure isolation | Diagnose deliberately broken workloads            |
| 5     | [16. Kustomize and environment overlays](#16-kustomize-and-environment-overlays)                 | Bases, overlays, patches, generated configuration          | Maintain local environment variants               |
| 5     | [17. Helm after the fundamentals](#17-helm-after-the-fundamentals)                               | Charts, templates, values, releases, hooks                 | Package the application without hiding Kubernetes |
| 6     | [18. Capstone: Signet Playground on Kubernetes](#18-capstone-signet-playground-on-kubernetes)    | Architecture migration, state, dependencies, operations    | Run the complete repository on Minikube           |
| 6     | [19. Production gaps and next steps](#19-production-gaps-and-next-steps)                         | HA, managed Kubernetes, autoscaling, monitoring, GitOps    | Evaluate what must change beyond the lab          |

## Detailed chapter index

### 01. Local lab setup

**Proposed file:** `01-local-lab-setup.md`

- Verify Docker Desktop, Minikube, and `kubectl`.
- Start Minikube with the Docker driver and explicit CPU, memory, and disk settings.
- Understand profiles, contexts, namespaces, and the relationship between Docker Desktop
  and the Minikube node.
- Enable only the add-ons needed by later chapters.
- Workshop: create, inspect, stop, start, and delete a disposable lab cluster.
- Troubleshooting drill: fix an incorrect context and an under-provisioned cluster.

### 02. Kubernetes mental model

**Proposed file:** `02-kubernetes-mental-model.md`

- Compare Docker Compose's orchestration model with Kubernetes.
- Introduce the API server, etcd, scheduler, controller manager, kubelet, and container
  runtime.
- Explain desired state, reconciliation loops, resource versions, and eventual
  consistency.
- Workshop: observe cluster components and follow an object from manifest to running
  container.

### 03. Pods and container lifecycle

**Proposed file:** `03-pods-and-container-lifecycle.md`

- Create Pods imperatively and declaratively.
- Inspect Pod phases, container states, restart counts, logs, and termination reasons.
- Use `exec`, `logs`, `describe`, and port forwarding.
- Introduce startup, readiness, and liveness probes without configuring all of them yet.
- Workshop: run a tiny HTTP server and diagnose a crash loop.

### 04. Declarative configuration and kubectl

**Proposed file:** `04-declarative-configuration-and-kubectl.md`

- Read Kubernetes YAML: `apiVersion`, `kind`, `metadata`, `spec`, and `status`.
- Use `apply`, `diff`, `get`, `explain`, `wait`, and safe deletion.
- Understand names, UIDs, labels, annotations, field ownership, and immutable fields.
- Workshop: evolve a resource through several declarative changes and inspect the API.

### 05. Deployments and ReplicaSets

**Proposed file:** `05-deployments-and-replicasets.md`

- Understand why naked Pods are rarely an application deployment strategy.
- Explore Deployments, ReplicaSets, selectors, templates, and rollout strategies.
- Scale, update, pause, resume, and roll back an application.
- Workshop: perform a rolling update and recover from a broken image.

### 06. Services and cluster networking

**Proposed file:** `06-services-and-cluster-networking.md`

- Explain Pod IPs, Service virtual IPs, endpoints, EndpointSlices, and cluster DNS.
- Compare ClusterIP, NodePort, LoadBalancer, ExternalName, and headless Services.
- Distinguish container ports, Service ports, target ports, and host exposure.
- Workshop: connect frontend and backend Deployments by DNS.
- Troubleshooting drill: repair incorrect selectors and port mappings.

### 07. Configuration and secrets

**Proposed file:** `07-configuration-and-secrets.md`

- Create ConfigMaps and Secrets from literals and files.
- Inject configuration as environment variables and mounted volumes.
- Explain update behavior, checksums, restarts, and why Secrets are not encrypted merely
  because they are Kubernetes objects.
- Workshop: move an application's configuration out of its Pod template.
- Signet preview: model `bitcoin.conf`, `fulcrum.conf`, and the RPC cookie's different
  lifecycle requirements.

### 08. Health, resources, and scheduling

**Proposed file:** `08-health-resources-and-scheduling.md`

- Configure startup, readiness, and liveness probes correctly.
- Set CPU and memory requests and limits.
- Inspect scheduling decisions, QoS classes, evictions, and out-of-memory failures.
- Introduce node selectors, affinity, anti-affinity, and taints conceptually.
- Workshop: tune a workload until it rolls out reliably under constrained resources.

### 09. Persistent storage

**Proposed file:** `09-persistent-storage.md`

- Distinguish ephemeral volumes, persistent volumes, claims, and storage classes.
- Explore Minikube's default dynamic provisioner.
- Understand access modes, reclaim policies, binding, and volume expansion.
- Workshop: persist database data across Pod deletion and application rollout.
- Signet preview: classify `node_data`, `fulcrum_data`, `mariadb_data`, and the shared
  cookie volume.

### 10. Jobs, init containers, and startup ordering

**Proposed file:** `10-jobs-init-containers-and-startup-ordering.md`

- Model finite work with Jobs and recurring work with CronJobs.
- Use init containers for local prerequisites, not as a universal dependency mechanism.
- Replace Compose startup ordering with readiness, retries, and idempotent setup.
- Understand Job retries, completion, cleanup, and failure policies.
- Workshop: initialize a database exactly once and make the operation safe to retry.
- Signet preview: redesign `wallet-setup` as an idempotent Kubernetes Job.

### 11. StatefulSets and stable identity

**Proposed file:** `11-statefulsets-and-stable-identity.md`

- Compare Deployments and StatefulSets.
- Use stable Pod names, ordered lifecycle, headless Services, and volume claim templates.
- Discuss when a singleton stateful application needs a StatefulSet and when it does not.
- Workshop: deploy a small stateful system and observe stable DNS and storage.
- Signet preview: evaluate StatefulSets for the Bitcoin node, Fulcrum, Valkey, and
  MariaDB rather than applying one controller type indiscriminately.

### 12. Namespaces, labels, and policy boundaries

**Proposed file:** `12-namespaces-labels-and-policy-boundaries.md`

- Design consistent labels and selectors.
- Use namespaces for scope and administration rather than as hard security boundaries.
- Apply resource quotas and limit ranges.
- Query and operate groups of resources safely.
- Workshop: organize the capstone resources and perform label-driven operations.

### 13. Ingress and local traffic exposure

**Proposed file:** `13-ingress-and-local-traffic-exposure.md`

- Install and inspect Minikube's ingress controller.
- Route multiple HTTP services by host and path.
- Compare Ingress with NodePort, LoadBalancer, and `kubectl port-forward`.
- Introduce certificates and TLS termination without making certificate automation a
  prerequisite.
- Workshop: expose the mempool frontend and faucet through local hostnames.

### 14. Security foundations

**Proposed file:** `14-security-foundations.md`

- Understand ServiceAccounts, Roles, ClusterRoles, and bindings.
- Configure security contexts, filesystem permissions, capabilities, and non-root users.
- Discuss image pinning, admission controls, secret management, and network policies.
- Workshop: remove unnecessary API access and privileges from a workload.
- Signet exercise: protect wallet descriptors and RPC authentication material.

### 15. Troubleshooting and observability

**Proposed file:** `15-troubleshooting-and-observability.md`

- Build a repeatable investigation flow from symptoms to events, status, logs, DNS,
  endpoints, processes, and storage.
- Use ephemeral debug containers and temporary diagnostic Pods.
- Introduce resource metrics and distinguish metrics, logs, and traces.
- Workshop: solve a set of broken manifests covering image pulls, probes, selectors,
  DNS, permissions, scheduling, and PVCs.

### 16. Kustomize and environment overlays

**Proposed file:** `16-kustomize-and-environment-overlays.md`

- Introduce Kustomize only after maintaining plain manifests.
- Create a reusable base and Minikube-specific overlay.
- Apply patches, name prefixes, labels, ConfigMap generators, and Secret generators.
- Workshop: maintain development and constrained-lab variants without copying manifests.

### 17. Helm after the fundamentals

**Proposed file:** `17-helm-after-the-fundamentals.md`

- Explain charts, templates, values, releases, dependencies, and lifecycle commands.
- Render templates locally and inspect the resulting Kubernetes resources.
- Identify useful abstraction boundaries and avoid turning every field into a value.
- Compare Helm hooks with regular Jobs and controller-driven lifecycle management.
- Workshop: package a previously working set of plain manifests as a chart.

### 18. Capstone: Signet Playground on Kubernetes

**Proposed file:** `18-capstone-signet-playground.md`

The capstone should migrate the repository in checkpoints so that each stage remains
observable and recoverable:

1. Document the Compose dependency graph, ports, configuration, data, and trust
   boundaries.
2. Deploy the Bitcoin node with configuration, persistent storage, probes, and an
   internal RPC Service.
3. Implement `wallet-setup` as a finite, retry-safe Job.
4. Deploy the miner and validate block production.
5. Add Fulcrum and Frigate, including shared RPC authentication and ZMQ connectivity.
6. Add MariaDB and Valkey with persistent storage and health checks.
7. Add the mempool backend, frontend, and faucet.
8. Expose the HTTP applications through Ingress and Electrum ports through an
   appropriate local mechanism.
9. Add resource requests, security contexts, labels, and operational documentation.
10. Convert the validated manifests into a Kustomize layout and then a Helm chart.
11. Run recovery drills: Pod loss, Job rerun, PVC retention, bad configuration, failed
    probe, and full cluster recreation.

The completed system should preserve the important behavior of the Compose stack:

- The node becomes ready before RPC clients depend on it.
- Wallet initialization completes successfully before mining starts.
- The miner produces blocks using the matching descriptor.
- Stateful data survives Pod replacement.
- Internal services communicate through Kubernetes DNS rather than `localhost`.
- The RPC cookie is shared only with workloads that require it.
- Host exposure remains intentional and minimal.

### 19. Production gaps and next steps

**Proposed file:** `19-production-gaps-and-next-steps.md`

- Compare Minikube with managed and multi-node Kubernetes.
- Discuss high availability, backups, disruption budgets, autoscaling, monitoring,
  centralized logging, certificate management, external secret stores, and GitOps.
- Identify which capstone choices are appropriate only for a local learning cluster.
- Final exercise: write a production-readiness assessment rather than pretending the
  local chart is production-ready.

## Suggested pacing

| Stage                            | Chapters | Suggested effort |
|----------------------------------|----------|------------------|
| Foundations                      | 01-05    | 8-12 hours       |
| Networking and configuration     | 06-08    | 7-10 hours       |
| Stateful workloads and lifecycle | 09-11    | 8-12 hours       |
| Operations and security          | 12-15    | 10-14 hours      |
| Packaging                        | 16-17    | 5-8 hours        |
| Capstone and assessment          | 18-19    | 12-20 hours      |

The pacing is intentionally flexible. A chapter is complete when its workshop can be
repeated without blindly copying commands and its failure scenarios can be explained.

## Capstone design decision

Signet Playground is a strong capstone because it is more realistic than a conventional
frontend/backend demo:

- It contains stateless, stateful, and finite workloads.
- It relies on explicit health and startup relationships.
- It combines TCP, HTTP, RPC, ZMQ, and Unix-socket communication.
- It needs persistent and shared storage with different semantics.
- It contains sensitive wallet and authentication material.
- It exposes only selected services to the host.

It should not be the first workload in the course. Using it incrementally after the
relevant primitives have been learned keeps Bitcoin-specific complexity from obscuring
Kubernetes fundamentals.

## Repository layout as the course grows

```text
docs/kubernetes-course/
├── README.md
├── 01-local-lab-setup.md
├── 02-kubernetes-mental-model.md
├── ...
├── 18-capstone-signet-playground.md
├── 19-production-gaps-and-next-steps.md
├── examples/
│   ├── 03-pods/
│   ├── 05-deployments/
│   └── ...
└── solutions/
    ├── 03-pods/
    ├── 05-deployments/
    └── ...
```

Exercise prompts and starter manifests should live under `examples/`. Complete reference
solutions should be kept separately under `solutions/` so workshops can be attempted
without seeing the final manifests accidentally.
