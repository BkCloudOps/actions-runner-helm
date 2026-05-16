# ARC Runner Stack — Documentation Index

This documentation set explains, end-to-end, how **GitHub Actions Runner Controller (ARC)** is deployed and used by this repository. It is written for two audiences at once:

- **Novices** — short narrative explanations, plain-language definitions, real-world analogies.
- **Architects / SREs** — CRD model, reconciliation loops, security boundaries, scaling math, every value explained.

| # | Document | What it covers |
|---|----------|----------------|
| 1 | [01-overview.md](01-overview.md) | What ARC is, why we use it, and what this repo deploys |
| 2 | [02-architecture.md](02-architecture.md) | Components, CRDs, control-plane vs data-plane |
| 3 | [03-end-to-end-flow.md](03-end-to-end-flow.md) | Step-by-step: from `git push` to job completion |
| 4 | [04-controller-values.md](04-controller-values.md) | Every value in `gha-runner-scale-set-controller/values.yaml` |
| 5 | [05-runner-scale-set-values.md](05-runner-scale-set-values.md) | Every value in `gha-runner-scale-set/values.yaml` |
| 6 | [06-container-modes.md](06-container-modes.md) | `dind` vs `kubernetes` vs `kubernetes-novolume` deep-dive |
| 7 | [07-authentication.md](07-authentication.md) | PAT, GitHub App, pre-defined secret, Azure Key Vault + Workload Identity |
| 8 | [08-scaling-behavior.md](08-scaling-behavior.md) | The math: `minRunners`, `maxRunners`, JIT tokens, ephemeral runners |
| 9 | [09-this-repo-stack.md](09-this-repo-stack.md) | How `arc-runner-stack` glues ESO + ARC together |
| 10 | [10-install-and-upgrade.md](10-install-and-upgrade.md) | Install order, upgrade caveats (CRDs), rollback |
| 11 | [11-observability.md](11-observability.md) | Metrics, logs, troubleshooting matrix |
| 12 | [12-security-model.md](12-security-model.md) | Namespaces, RBAC, privileged containers, secrets-at-rest |
| 13 | [13-crd-reference.md](13-crd-reference.md) | Field-by-field reference for all four ARC CRDs (Kubernetes-API-doc style) |
| 14 | [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md) | Wire-level: how the listener long-polls GitHub and how JIT runner configs are minted, injected, and consumed |

> **TL;DR** — A push to GitHub triggers a workflow. GitHub holds the job in a queue. A **Listener Pod** in your cluster is long-polling GitHub. When GitHub says "job available", the Listener tells the **Controller** to create an **Ephemeral Runner Pod**. That pod registers with GitHub using a Just-in-Time token, runs the job, then self-destructs.
