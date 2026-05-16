# 2. Architecture

This chapter describes ARC's components, the CRDs they reconcile, the planes of trust, and the resources each Helm chart actually deploys. It is the conceptual map; for field-by-field detail on the CRDs see [13-crd-reference.md](13-crd-reference.md), and for the wire-level behavior of the listener see [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md).

## 2.1 The 30-second picture

```
                ┌────────────────────────────────────────────────────────┐
                │              GitHub Actions Service                    │
                │  ┌─────────────────┐    ┌────────────────────────────┐ │
                │  │  api.github.com │    │ *.actions.githubusercontent│ │
                │  │   (REST API)    │    │   .com  (RunService)       │ │
                │  └────────▲────────┘    └──────────▲─────────────────┘ │
                └───────────┼────────────────────────┼───────────────────┘
                            │ REST                   │ long-poll
                            │ (JIT, register,        │ (JobAvailable,
                            │  deregister)           │  AcquireJob)
                            │                        │
                ┌───────────┼────────────────────────┼───────────────────┐
                │           │       Kubernetes       │                   │
                │   namespace: arc-systems           │                   │
                │  ┌────────┴─────────────┐   ┌──────┴──────────┐        │
                │  │ controller-manager   │   │  listener Pod   │        │
                │  │ (Deployment, 1 Pod)  │──►│  (1 per pool)   │        │
                │  └──────────┬───────────┘   └──────┬──────────┘        │
                │             │ reconciles CRDs     │ patches replicas   │
                │             ▼                     ▼                    │
                │   ┌─────────────────────┐  ┌──────────────────────┐    │
                │   │ AutoscalingListener │  │ EphemeralRunnerSet   │    │
                │   │      (CR)           │  │      (CR)            │    │
                │   └─────────────────────┘  └──────────┬───────────┘    │
                │                                       │ owns           │
                │   namespace: arc-runners              ▼                │
                │   ┌────────────────────────┐   ┌──────────────────┐    │
                │   │ AutoscalingRunnerSet   │   │ EphemeralRunner  │    │
                │   │      (CR — yours)      │   │   (CR × N)       │    │
                │   └────────────────────────┘   └──────┬───────────┘    │
                │                                       │ owns           │
                │                                       ▼                │
                │                                ┌─────────────────┐     │
                │                                │ runner Pod × N  │     │
                │                                └─────────────────┘     │
                └────────────────────────────────────────────────────────┘
```

Two services on the GitHub side, two namespaces on the cluster side, four CRDs in between.

## 2.2 Components

### 2.2.1 Controller manager

| Property | Value |
|----------|-------|
| Kind | `Deployment` (typically `replicas: 1`) |
| Image | `ghcr.io/actions/gha-runner-scale-set-controller:0.14.1` |
| Container name | `manager` |
| Namespace | `arc-systems` |
| Replicas | `1` by default; `> 1` is supported via leader-election lease |
| Watches | `AutoscalingRunnerSet`, `AutoscalingListener`, `EphemeralRunnerSet`, `EphemeralRunner` |
| Scope | All namespaces by default; restrictable via `flags.watchSingleNamespace` |

The manager binary embeds four reconcilers, one per CRD. They share a single Kubernetes client and a single controller-runtime cache. On a healthy install:

- `AutoscalingRunnerSetReconciler` materializes per-pool resources: the listener config Secret, ServiceAccount + Role + RoleBinding, the `AutoscalingListener` CR, and an `EphemeralRunnerSet`.
- `AutoscalingListenerReconciler` materializes a listener Pod from the `AutoscalingListener` spec.
- `EphemeralRunnerSetReconciler` matches `.spec.replicas` to `.status.currentReplicas` by creating or deleting `EphemeralRunner` CRs. Deletions are *gentle*: if a runner is executing a job, the reconciler asks GitHub to mark it for removal and waits.
- `EphemeralRunnerReconciler` generates the JIT config, creates the runner Pod, watches for Pod termination, and deletes the runner registration from GitHub.

The reconcilers are **non-blocking** — each enqueues work into a rate-limited workqueue and runs up to `runnerMaxConcurrentReconciles` (default `1`) workers. Increasing this is the primary knob for scale-up throughput; see [04-controller-values.md](04-controller-values.md).

### 2.2.2 Listener Pod

| Property | Value |
|----------|-------|
| Kind | `Pod` (created by the controller, not by a higher-level controller) |
| Image | `ghcr.io/actions/gha-runner-scale-set-listener:0.14.1` |
| Container name | `listener` |
| Namespace | `arc-systems` |
| Cardinality | **Exactly one per `AutoscalingRunnerSet`** |
| Network | Outbound HTTPS only (no inbound port) |
| State | Stateless; session lives on GitHub's side and is recreated on restart |

The listener is a thin Go program that:

1. Authenticates to GitHub (PAT or App; see [07-authentication.md](07-authentication.md)).
2. Discovers the regional RunService URL and acquires an Actions Runtime Token.
3. Registers (or looks up) the runner scale set, obtains its integer `runnerScaleSetId`.
4. Opens a **session** and enters a long-poll loop.
5. On each `JobAvailable` / `JobAssigned` / `JobCompleted` message, recomputes the desired runner count and patches `EphemeralRunnerSet.spec.replicas` accordingly.

That is the **entirety** of the listener's responsibility. It does not create Pods, mint JIT configs, or call the Kubernetes node API. Those are the controller's job. Every protocol detail of the listener is documented in [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md).

### 2.2.3 Ephemeral runner Pod

| Property | Value |
|----------|-------|
| Kind | `Pod` (created from the `EphemeralRunner` CR by the controller) |
| Image | User-provided (default: `ghcr.io/actions/actions-runner:latest`) |
| Container name | `runner` (additional sidecars permitted in `dind` and `kubernetes` modes) |
| Namespace | `arc-runners` (by convention; configurable) |
| Lifecycle | One job, then exit. `restartPolicy: Never`. |
| Identity | JIT-configured at Pod-create time via env var; deregisters on exit |

The runner Pod has no awareness that ARC exists. It is just the standard `actions/runner` binary configured with a JIT blob. See [06-container-modes.md](06-container-modes.md) for how `dind` and `kubernetes` modes change the Pod spec.

## 2.3 The four CRDs

All four belong to the API group `actions.github.com/v1alpha1` and are namespaced. Detailed field reference in [13-crd-reference.md](13-crd-reference.md).

| Kind | Created by | Lives in | Children |
|------|------------|----------|----------|
| `AutoscalingRunnerSet` | You (Helm) | `arc-runners` | 1 × `AutoscalingListener` + 1 × `EphemeralRunnerSet` |
| `AutoscalingListener` | Controller | `arc-systems` | 1 × listener Pod |
| `EphemeralRunnerSet` | Controller | `arc-runners` | N × `EphemeralRunner` |
| `EphemeralRunner` | Controller | `arc-runners` | 1 × runner Pod, optionally 1 × per-runner Secret |

This is deliberately a **mirror of Kubernetes' own** workload hierarchy:

```
Deployment   →  ReplicaSet     →  Pod
AutoscalingRunnerSet →  EphemeralRunnerSet →  EphemeralRunner → Pod
```

The asymmetry is `AutoscalingListener`, which has no native equivalent — it is the bridge between the cluster and GitHub.

## 2.4 Resources deployed by each chart

### 2.4.1 `gha-runner-scale-set-controller` (installed once)

| Template | Kind | Suffix appended to release name | Purpose |
|----------|------|---------------------------------|---------|
| `deployment.yaml` | `Deployment` | `-gha-rs-controller` | Controller-manager pod |
| `serviceaccount.yaml` | `ServiceAccount` | `-gha-rs-controller` | Identity for the controller |
| `manager_cluster_role.yaml` | `ClusterRole` | `-gha-rs-controller` | Cluster-wide reconcile permissions |
| `manager_cluster_role_binding.yaml` | `ClusterRoleBinding` | `-gha-rs-controller` | Binds the ClusterRole |
| `manager_single_namespace_*` | `Role` / `RoleBinding` | `-gha-rs-controller-single-namespace` | Used **instead of** the ClusterRole when `flags.watchSingleNamespace` is set |
| `manager_listener_role.yaml` | `Role` | `-gha-rs-controller-listener` | Per-runner-namespace permissions the controller delegates to listener pods |
| `leader_election_*` | `Role` / `RoleBinding` | (auto) | Coordination lease when `replicaCount > 1` |
| `crds/` (4 files) | `CustomResourceDefinition` | n/a | The four CRDs |

### 2.4.2 `gha-runner-scale-set` (installed per pool)

| Template | Kind | Purpose |
|----------|------|---------|
| `autoscalingrunnerset.yaml` | `AutoscalingRunnerSet` (CR) | The pool declaration |
| `githubsecret.yaml` | `Secret` | Created **only** if `githubConfigSecret` is provided inline as an object; skipped when you pass a string (the recommended path with ESO) |
| `manager_role.yaml` / `_binding` | `Role` / `RoleBinding` | Authorizes the controller in the runner namespace |
| `no_permission_serviceaccount.yaml` | `ServiceAccount` | Mounted into runner Pods that must **not** reach the K8s API (default for `dind`) |
| `kube_mode_*` | `Role` / `RoleBinding` / `ServiceAccount` | Permissions for runners to create job Pods (only when `containerMode.type=kubernetes`) |

### 2.4.3 Umbrella `arc-runner-stack` (this repo)

| Template | Kind | Purpose |
|----------|------|---------|
| `namespaces.yaml` | `Namespace` × 3 | `arc-systems`, `arc-runners`, `external-secrets` (with workload-identity labels) |
| `cluster-secret-store.yaml` | `ClusterSecretStore` | Bind to Azure Key Vault via federated identity |
| `external-secrets.yaml` | `ExternalSecret` | Sync GitHub credential from KV into a Kubernetes `Secret` |
| `service-account.yaml` | `ServiceAccount` | The identity ESO assumes; annotated with `azure.workload.identity/client-id` |

## 2.5 Control plane vs data plane

| Plane | Components | Trust posture |
|-------|------------|---------------|
| **Control plane** | controller-manager, listener pods, ESO, the four ARC Secrets (auth tokens) | High trust: must hold GitHub credentials and patch CRs |
| **Data plane** | runner Pods executing workflow steps | **Untrusted**: every job is third-party code |

The repository encodes this split as **two namespaces** (`arc-systems` for control, `arc-runners` for data). Best-practice additions (some default, some left for the operator):

- A `NetworkPolicy` denying egress from `arc-runners` to the cluster's internal services except where needed (DNS, GitHub, container registry).
- A `NetworkPolicy` denying ingress to `arc-systems` from `arc-runners`.
- `PodSecurity` admission set to `restricted` on `arc-runners` for `kubernetes`-mode pools and `baseline` for `dind`-mode pools (DinD requires privileged).
- A dedicated node pool / taint for runner Pods so a compromised runner cannot reach the control-plane node's kubelet.

See [12-security-model.md](12-security-model.md) for the audit checklist.

## 2.6 Reconcile order at install

A common confusion is "which resource appears first when I install?" The order is:

1. CRDs registered (controller chart `crds/`).
2. Controller Deployment becomes Ready.
3. Helm post-install hooks finish; control returns to the user.
4. **You** (or the umbrella chart) install the `gha-runner-scale-set` chart.
5. `AutoscalingRunnerSet` CR appears.
6. Controller reconciles it. In one pass it creates: listener config Secret, listener ServiceAccount/Role/RoleBinding, `AutoscalingListener` CR (in `arc-systems`), and `EphemeralRunnerSet` CR (in `arc-runners`, replicas = `minRunners`).
7. The `AutoscalingListener` reconciler materializes the listener Pod.
8. Once the listener establishes a session with GitHub, the runner scale set appears in the GitHub UI under *Runner groups*.

If step 1 is missing (CRDs not yet installed), Helm install of the runner-scale-set chart fails with `no matches for kind "AutoscalingRunnerSet"`. This is why the umbrella chart's [Chart.yaml](../charts/arc-runner-stack/Chart.yaml) lists the controller chart as a dependency rendered *before* the scale-set chart.

Continue to [03-end-to-end-flow.md](03-end-to-end-flow.md).
