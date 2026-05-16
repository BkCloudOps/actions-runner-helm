# 1. Overview

## 1.1 What is ARC?

**Actions Runner Controller (ARC)** is the official GitHub Kubernetes **operator** that runs your GitHub Actions jobs on **your own** Kubernetes cluster instead of GitHub-hosted runners.

> **Analogy** — Think of GitHub-hosted runners as Uber: convenient but you don't control the car. ARC is like running your own taxi fleet: you own the cars, choose the model, and decide when to add or retire them.

## 1.2 Why use it?

| Need | GitHub-hosted | ARC self-hosted |
|------|---------------|-----------------|
| Access to private network (DBs, VPCs) | ❌ | ✅ |
| Custom OS / GPU / large CPU | Limited | ✅ |
| Cost at high volume | Pay per minute | Fixed cluster cost |
| Compliance (data stays in your tenant) | ❌ | ✅ |
| Setup effort | Zero | Non-trivial |
| Security responsibility | GitHub | **You** |

## 1.3 What this repository deploys

```
arc-runner-stack/                       <- the umbrella Helm chart in this repo
├── (own templates)
│   ├── namespaces                      arc-systems, arc-runners, external-secrets
│   ├── ClusterSecretStore              binds Azure Key Vault to the cluster via Workload Identity
│   ├── ExternalSecret                  syncs the GitHub PAT from Key Vault -> k8s Secret
│   └── ServiceAccount                  for the External Secrets operator to assume the Managed Identity
└── charts/                             <- pulled from oci://ghcr.io/actions/actions-runner-controller-charts
    ├── gha-runner-scale-set-controller   the OPERATOR (one per cluster, usually)
    └── gha-runner-scale-set              one INSTALLATION per runner pool you want
```

### The two ARC sub-charts at a glance

| Chart | Lifecycle | What it deploys | Think of it as |
|-------|-----------|-----------------|----------------|
| `gha-runner-scale-set-controller` | Install once per cluster | A Deployment (the **controller manager** Pod), CRDs, ClusterRoles | The **brain** |
| `gha-runner-scale-set` | Install N times (once per "pool") | An `AutoscalingRunnerSet` CR + Secret + Roles. The controller reacts to this CR and spawns a **Listener Pod** | The **request** to create a pool |

## 1.4 Mental model in one sentence

> *The **controller** watches for `AutoscalingRunnerSet` resources you create; for each one it spawns a **Listener** that long-polls GitHub; when GitHub has a job, the Listener tells the controller to spawn short-lived **EphemeralRunner** Pods that run the job and disappear.*

Continue to [02-architecture.md](02-architecture.md).
