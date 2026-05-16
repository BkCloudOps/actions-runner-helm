# 2. Architecture

## 2.1 The big picture

```
                         ┌─────────────────────────────────────────────┐
                         │              GitHub Actions Service          │
                         │   (job queue, runner registry, JIT tokens)   │
                         └───────────────▲────────────▲────────────────┘
                       HTTPS long-poll   │            │  HTTPS (register/heartbeat)
                                         │            │
┌────────────────────── Kubernetes Cluster ──────────────────────────────────┐
│                                                                            │
│   namespace: arc-systems                  namespace: arc-runners           │
│   ┌──────────────────────────────┐        ┌──────────────────────────┐    │
│   │  controller-manager (Pod)    │        │  Listener Pod             │    │
│   │  ─ watches CRDs              │  spawns│  ─ long-polls GitHub      │    │
│   │  ─ reconciles state          ├───────►│  ─ requests scale-up      │    │
│   └──────────┬───────────────────┘        └──────────┬───────────────┘    │
│              │ creates / patches                      │ patches replicas   │
│              ▼                                        ▼                    │
│   ┌──────────────────────────────┐        ┌──────────────────────────┐    │
│   │ AutoscalingRunnerSet (CR)    │◄───────│ EphemeralRunnerSet (CR)  │    │
│   └──────────────────────────────┘        └──────────┬───────────────┘    │
│                                                      │ owns                │
│                                                      ▼                    │
│                                          ┌──────────────────────────┐    │
│                                          │ Ephemeral Runner Pods    │    │
│                                          │ (runner + optional dind) │    │
│                                          └──────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────────┘
```

## 2.2 Components

### 2.2.1 Controller manager (the operator)
- **One Deployment per cluster** (unless `flags.watchSingleNamespace` is set, then one per watched namespace).
- Image: `ghcr.io/actions/gha-runner-scale-set-controller:<version>`.
- Reconciles four CRDs (next section).
- When `replicaCount > 1`, only one replica holds the leader-election lease and does reconciliation; others stand by.

### 2.2.2 Listener Pod
- **One per `AutoscalingRunnerSet`** — i.e., one per runner pool.
- Spawned by the controller (not by you).
- A small Go program that opens an **HTTPS long-poll connection** to the GitHub Actions service. No inbound ports are required — all traffic is outbound from your cluster.
- When GitHub sends a "job available" message, the Listener computes the desired number of runners and patches the `EphemeralRunnerSet`.

### 2.2.3 Ephemeral runner Pod
- The actual workload pod that contains the GitHub Actions runner binary.
- **Ephemeral** = lives for exactly **one job**, then exits. No state leaks between jobs.
- May include a `dind` sidecar (for Docker-in-Docker mode) or extra init containers.

## 2.3 Custom Resource Definitions (CRDs)

All under API group `actions.github.com/v1alpha1`. These are installed by the **controller** chart under `crds/`.

| CRD | Created by | Purpose |
|-----|-----------|---------|
| `AutoscalingRunnerSet` | **You** (via the `gha-runner-scale-set` Helm install) | High-level "I want a runner pool" declaration. Holds config: GitHub URL, min/max, pod template, container mode. |
| `AutoscalingListener` | Controller | Spec for the Listener Pod. One per `AutoscalingRunnerSet`. |
| `EphemeralRunnerSet` | Controller (owned by `AutoscalingRunnerSet`) | Like a ReplicaSet but for ephemeral runners. Holds desired replica count. |
| `EphemeralRunner` | Controller (owned by `EphemeralRunnerSet`) | One CR per runner Pod. Tracks JIT token, registration state, completion. |

> **Architect note** — CRDs are intentionally *not* upgradable by Helm (Helm's CRD lifecycle is limited). Upgrades require uninstall → wait → reapply CRDs → reinstall. See [10-install-and-upgrade.md](10-install-and-upgrade.md).

## 2.4 Resources deployed (reference table)

### By `gha-runner-scale-set-controller` (the operator chart)

| Template | Kind | Suffix appended to release name | Purpose |
|----------|------|---------------------------------|---------|
| `deployment.yaml` | Deployment | `-gha-rs-controller` | The controller-manager Pod |
| `serviceaccount.yaml` | ServiceAccount | `-gha-rs-controller` | Identity for the controller |
| `manager_cluster_role.yaml` | ClusterRole | `-gha-rs-controller` | Cluster-wide reconcile permissions (when watching all namespaces) |
| `manager_cluster_role_binding.yaml` | ClusterRoleBinding | `-gha-rs-controller` | Binds the ClusterRole |
| `manager_single_namespace_*` | Role / RoleBinding | `-gha-rs-controller-single-namespace` | Used instead of ClusterRole when `flags.watchSingleNamespace` is set |
| `manager_listener_role.yaml` | Role | `-gha-rs-controller-listener` | Permissions the controller needs to manage listener Pods |
| `leader_election_*` | Role / RoleBinding | (auto) | Lease for HA when `replicaCount > 1` |

### By `gha-runner-scale-set` (one per runner pool)

| Template | Kind | Purpose |
|----------|------|---------|
| `autoscalingrunnerset.yaml` | `AutoscalingRunnerSet` (CR) | Your pool declaration |
| `githubsecret.yaml` | Secret | PAT or GitHub App credentials (only if `githubConfigSecret` is an object, not a string) |
| `manager_role.yaml` / `_binding` | Role / RoleBinding | Lets the controller reconcile in this namespace |
| `no_permission_serviceaccount.yaml` | ServiceAccount | Mounted into runner Pods that **must not** talk to the K8s API (dind mode default) |
| `kube_mode_*` | Role / RoleBinding / SA | Permissions for runners to create job Pods (Kubernetes mode only) |

## 2.5 Control plane vs data plane

| Plane | What runs there | Trust level |
|-------|-----------------|-------------|
| **Control plane** | controller-manager, Listener | High — needs GitHub auth, patches CRs |
| **Data plane** | Ephemeral runner Pods | **Untrusted** — they execute arbitrary user code from workflows |

> **Security imperative** — Always run control plane and data plane in **separate namespaces** (this repo uses `arc-systems` for the controller and `arc-runners` for the runners). NetworkPolicies should isolate runners from the control plane.

Continue to [03-end-to-end-flow.md](03-end-to-end-flow.md).
