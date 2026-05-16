# 1. Overview

## 1.1 What is Actions Runner Controller?

**Actions Runner Controller (ARC)** is the official Kubernetes operator from GitHub that runs your GitHub Actions workflows on self-hosted runners inside a cluster you control. Instead of jobs executing on shared GitHub-hosted virtual machines, they execute in **ephemeral Pods** on your Kubernetes nodes, with full access to your VPC, your private registries, your secrets management, and your compliance boundary.

ARC consists of two cooperating Helm charts published as OCI artifacts under [`oci://ghcr.io/actions/actions-runner-controller-charts`](https://github.com/actions/actions-runner-controller):

| Chart | Cardinality | Role |
|-------|-------------|------|
| `gha-runner-scale-set-controller` | Usually one per cluster (or one per watched namespace) | The Kubernetes operator: a Deployment that watches CRDs and reconciles state. |
| `gha-runner-scale-set` | One per runner pool you want | Declares an `AutoscalingRunnerSet`; the operator turns each declaration into a long-poll listener + a dynamic pool of runner Pods. |

Both charts are pinned to **app version `0.14.1`** (and chart version `0.14.1`) in this repository — see [Chart.yaml](../charts/arc-runner-stack/Chart.yaml).

## 1.2 Why use ARC?

The decision is rarely about cost alone; it is usually about **trust boundaries**:

| Requirement | GitHub-hosted runners | ARC self-hosted |
|-------------|----------------------|-----------------|
| Workflows can reach internal databases / VPC-only services | ✗ | ✓ |
| Custom OS images, GPU, large CPU, FPGA | Limited | ✓ |
| Compliance: code/data never leaves your tenant | ✗ | ✓ |
| Hardened toolchain (Bazel cache, internal mirrors, signed actions) | Hard | ✓ |
| Cost at sustained > 50k minutes/month | Linear pay-per-minute | Approximately flat cluster cost |
| Time to first green build | Minutes (none) | Hours-to-days (cluster, identity, network, secrets) |
| Security responsibility for the runner sandbox | GitHub | **You** |
| Ephemeral, isolated execution per job | Yes (built-in) | Yes (default; see [container modes](06-container-modes.md)) |

The two non-obvious risks to plan for **before** committing to ARC:

1. **You inherit the runner sandbox problem.** A workflow is, by definition, executing third-party code (actions, scripts, container images). The Pod that runs it is therefore an untrusted process inside your cluster. See [12-security-model.md](12-security-model.md) for the namespace and RBAC isolation this repository enforces by default.
2. **GitHub remains the queue.** ARC does not eliminate the dependency on `github.com`; it only changes *where* the work executes. A GitHub outage still stops your CI.

## 1.3 What this repository deploys

This repository contains an **umbrella Helm chart**, [`arc-runner-stack`](../charts/arc-runner-stack), which composes:

- Both upstream ARC sub-charts (vendored under [`charts/arc-runner-stack/charts/`](../charts/arc-runner-stack/charts)).
- Cluster-level glue: namespaces, External Secrets Operator integration, Azure Workload Identity binding.

```
arc-runner-stack/                       # umbrella chart (this repo)
├── templates/
│   ├── namespaces.yaml                 # creates arc-systems, arc-runners, external-secrets
│   ├── cluster-secret-store.yaml       # binds Azure Key Vault to the cluster
│   ├── external-secrets.yaml           # syncs GitHub credential from Key Vault → Secret
│   └── service-account.yaml            # ESO identity, federated to Azure AD
└── charts/                             # pulled from oci://ghcr.io/actions/actions-runner-controller-charts
    ├── gha-runner-scale-set-controller/   # the operator (one per cluster)
    └── gha-runner-scale-set/              # one runner pool per release
```

In production you will typically install the umbrella chart **once** to provision the operator and shared plumbing, then add a `gha-runner-scale-set` release **per runner pool** (e.g. one for Linux x86, one for Linux ARM, one for self-hosted Windows). The umbrella in this repo currently provisions a single pool named `straw-hat-runners-linux` targeting GitHub organization `BkCloudOps`.

## 1.4 The one-paragraph mental model

When you `helm install` the umbrella chart:

1. The **controller** Deployment starts in `arc-systems` and begins watching the four ARC CRDs.
2. The chart also creates an `AutoscalingRunnerSet` CR in `arc-runners` declaring "I want a pool named `straw-hat-runners-linux` attached to `https://github.com/BkCloudOps`, scaling between 0 and N runners."
3. The controller observes that CR, registers the pool with GitHub via the REST API, and creates an `AutoscalingListener` CR. The listener is a small pod that opens an HTTPS **long-poll** to GitHub's Actions service and patches an `EphemeralRunnerSet.spec.replicas` whenever GitHub offers jobs.
4. The controller turns each desired replica into a fresh **JIT-configured runner Pod** that registers itself with GitHub, executes exactly one job, deregisters, and exits.

The next chapters drill into each of these layers. If you want a packet-level walkthrough of step 3, jump to [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md).

## 1.5 Versioning and compatibility

| Component | This repo pins | Notes |
|-----------|----------------|-------|
| Umbrella chart | `1.0.0` | Local SemVer. |
| `gha-runner-scale-set-controller` | `0.14.1` | Chart and image. |
| `gha-runner-scale-set` | `0.14.1` | Must match the controller minor version. |
| Kubernetes API server | ≥ `1.26` | CRD `v1`, server-side apply, projected service-account tokens. |
| `cert-manager` | Not required | Listener uses outbound TLS only. |
| External Secrets Operator | ≥ `0.9` | For the Key Vault → Secret bridge. |

> **Mixing minor versions of the two sub-charts is unsupported.** GitHub publishes both in lockstep. When upgrading, see [10-install-and-upgrade.md](10-install-and-upgrade.md) for the CRD handling caveat.

## 1.6 Reading order

| Audience | Start at |
|----------|----------|
| Engineer who has never seen ARC | [02-architecture.md](02-architecture.md) → [03-end-to-end-flow.md](03-end-to-end-flow.md) |
| SRE installing it for the first time | [09-this-repo-stack.md](09-this-repo-stack.md) → [10-install-and-upgrade.md](10-install-and-upgrade.md) → [07-authentication.md](07-authentication.md) |
| Architect reviewing the design | [02-architecture.md](02-architecture.md) → [13-crd-reference.md](13-crd-reference.md) → [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md) → [12-security-model.md](12-security-model.md) |
| Author of a workflow that uses these runners | [06-container-modes.md](06-container-modes.md) → [08-scaling-behavior.md](08-scaling-behavior.md) |
| Debugging a stuck pool | [11-observability.md](11-observability.md) → [14-listener-protocol-and-jit.md §8](14-listener-protocol-and-jit.md#8-failure-modes-and-recovery) |

Continue to [02-architecture.md](02-architecture.md).
