# 15. Recreating the Three-Repo GitHub Self-Hosted Runner Stack

This document describes a common three-repository pattern for running
GitHub Actions self-hosted runners on Kubernetes (AKS, EKS, GKE, or any
cluster) using ARC + Flux + Packer, and explains how to structure and
maintain equivalent repos in **your own personal / org setup** so that
you can reproduce the pattern end-to-end.

The three repos are referred to here by generic names:

- `runner-images` — the image factory
- `runners-platform` — the application / manifests repo
- `runners-gitops` — the cluster / GitOps repo

The goal is to capture the **separation of concerns**, the **directory
conventions**, and the **maintenance workflow** that make the stack work —
not to prescribe specific tool or vendor choices.

> Companion docs in this repo:
> - [02-architecture.md](02-architecture.md) — what ARC is and how it runs
> - [09-this-repo-stack.md](09-this-repo-stack.md) — the umbrella Helm chart
> - [10-install-and-upgrade.md](10-install-and-upgrade.md) — install flow

---

## 15.1 The three-repo model at a glance

| Repo | Layer | Purpose | Consumed by |
|------|-------|---------|-------------|
| `runner-images` | **Image** | Builds the OCI images that runner pods boot from (Packer + GitHub Actions). | Helm/Kustomize values reference `ghcr.io/<org>/runner-images:<tag>`. |
| `runners-platform` | **Application** | Kustomize bases + Helm releases for the ARC controller and per-team runner scale sets. May also hold IaC (Terraform/Pulumi) that provisions cluster-level cloud resources (secret store, managed identity, resource group, state storage). | Pulled by Flux from the GitOps repo via `GitRepository` sources. |
| `runners-gitops` | **Platform / Cluster** | One folder per cluster. Bootstraps Flux, registers tenants, wires `GitRepository`/`Kustomization` CRs that point back at the application repo. | Flux running inside each cluster. |

The flow is always **left-to-right**:

```
runner-images  ──►  runners-platform  ──►  runners-gitops  ──►  Kubernetes cluster
   (CI build)        (helm + kustomize)      (flux bootstrap)       (runtime)
```

A change to a runner image triggers a new tag → the application repo bumps the
image reference → the GitOps repo's Flux loop reconciles it onto every cluster
that subscribes to it.

---

## 15.2 Repo 1 — `runner-images` (the image factory)

### 15.2.1 What it owns

- Packer templates (`.pkr.hcl`) that build runner OS images on top of an
  upstream `summerwind/actions-runner-dind` base.
- Per-OS install scripts, toolset JSON files, and software reports.
- GitHub Actions workflows that turn a push on `dev/<image-name>` into a
  versioned, tagged release pushed to GHCR.

### 15.2.2 Directory layout to mirror

```
runner-images/
├── README.md
├── CONTRIBUTING.md
├── .github/
│   ├── CODEOWNERS
│   ├── ISSUE_TEMPLATE/
│   └── workflows/
│       ├── build-preview-image-template.yml   # reusable build workflow
│       ├── create-release.yml                 # on push to dev/** → preview build
│       ├── release-runner-image-template.yml  # reusable release workflow
│       ├── tag-release.yml                    # promotes preview → release tag
│       └── delete-untagged-versions.yml       # GHCR housekeeping
├── common/                                    # shared scripts / certs
├── helpers/
│   └── software-report-base/                  # PowerShell modules for diff reports
├── images/
│   └── linux/
│       ├── config/ubuntu2204.conf
│       ├── scripts/                           # install-*.sh scripts
│       ├── toolsets/toolset-2204.json
│       ├── post-generation/
│       ├── ubuntu2204-<variant>.pkr.hcl
│       ├── ubuntu-2204-<variant>-Readme.md
│       └── ubuntu-2204-<variant>-software-report.json
└── docs/
    ├── architecture/build-process-architecture.md
    └── guides/ops/{maintenance,support,troubleshooting}/
```

### 15.2.3 Branching & tagging convention

The workflows assume this scheme — keep it if you reuse them as-is:

| Branch / Tag | Purpose |
|--------------|---------|
| `dev/<os>-<variant>` (e.g. `dev/ubuntu-22.04-provisioning-essentials`) | Active development. Every push triggers a Packer build of a **preview** image. |
| `releases/<os>-<variant>/v<major>` | Cut from `dev/**` when a major version ships. Long-lived. |
| Tag `v<semver>-<os>-<variant>` (e.g. `v0.0.9-ubuntu-2204-provisioning-essentials`) | Immutable release. Drives the GHCR tag pushed by `tag-release.yml`. |

The `paulhatch/semantic-version` action computes the next semver per
`namespace` (one namespace per `<os>-<variant>`) so multiple image variants
can evolve independently in the same repo.

### 15.2.4 Maintenance loop

1. Open a PR into `dev/<image>` with script/toolset changes.
2. Merge → `create-release.yml` builds a preview image, runs the software
   report diff, and posts results.
3. When ready, run `tag-release.yml` to mint `v<x.y.z>-<image>` and push the
   release tag to GHCR.
4. Bump the image reference in repo 2 (`github-self-hosted-runners-aks`).

### 15.2.5 What to change for your setup

- Replace `summerwind/actions-runner-dind:ubuntu-22.04` only if you need a
  different base; otherwise keep it — ARC expects the `actions-runner`
  binary in the image.
- Replace registry references (`ghcr.io/<your-org>/runner-images`) in both
  the workflows and downstream consumers.
- Strip any vendor-specific Packer variables (license keys, internal CA
  bundles, proprietary tool installers) you do not need.

---

## 15.3 Repo 2 — `runners-platform` (the application repo)

### 15.3.1 What it owns

- **Infrastructure as Code** (Terraform / Pulumi / Bicep) for cloud resources
  every cluster needs: a resource group, a secret store, a workload-identity
  / managed-identity for the in-cluster secret operator, and remote state
  storage.
- **Kustomize bases** for runner scale sets (`gitops/base/`).
- **Cluster-scoped overlays** (`gitops/<cluster>/`) that select a base, patch
  the image reference, set the GitHub org URL, and wire the ARC controller's
  HelmRelease.
- Optional CI pipeline definitions (Jenkinsfile, GitHub Actions, etc.) that
  run the IaC on a schedule.

### 15.3.2 Directory layout to mirror

```
runners-platform/
├── README.md
├── ci/                                         # (optional) IaC pipeline definitions
│   ├── iac-nonprod-deploy.pipeline
│   └── iac-prod-deploy.pipeline
├── iac/                                        # Terraform / Pulumi / Bicep
│   ├── non-prod/
│   │   ├── main.tf            # RG, workload identity, secret store, state storage
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── state/             # backend config per env
│   └── prod/...
└── gitops/
    ├── base/                                   # reusable, cluster-agnostic
    │   ├── runner-resources/                   # default Linux runner pool
    │   │   ├── kustomization.yaml
    │   │   ├── runner-resource-deployment.yaml
    │   │   └── runner-resource-autoscaler.yaml
    │   ├── runner-resources-large/             # large-pool variant
    │   ├── runner-resources-win/               # Windows runner pool
    │   ├── runners-team-a/                     # team overlay of runner-resources
    │   ├── runners-team-a-large/
    │   ├── runners-team-a-win/
    │   ├── runners-team-b/                     # one per team / GitHub org
    │   └── secrets/                            # ExternalSecret templates
    ├── nonprod-region1/                        # one folder per cluster
    │   ├── runners-controller/
    │   │   ├── controller-resources/
    │   │   │   ├── actions-runner-controller-<tenant>.yaml   # HelmRelease + ExternalSecret
    │   │   │   ├── ingress/
    │   │   │   └── sources.yaml                # HelmRepository / GitRepository
    │   │   └── overlays/
    │   │       └── kustomizations.yaml         # Flux Kustomization for the controller
    │   ├── runners-<tenant>/                   # per-tenant runner scale sets
    │   │   └── overlays/...
    │   └── ...
    ├── prod-region1/
    ├── prod-region2/
    ├── prod-region3/
    ├── prod-region4/
    └── win-nonprod-region1/                    # Windows clusters get their own root
```

### 15.3.3 The base/overlay pattern (key to maintainability)

Bases are intentionally generic: they declare a `RunnerDeployment` and
`HorizontalRunnerAutoscaler` with placeholder names. A team overlay
(`runners-team-a/kustomization.yaml`) does three things:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
nameSuffix: -team-a                           # 1. differentiates resources
resources:
  - ../runner-resources
patches:                                      # 2. retargets the HRA
  - patch: |-
      - op: replace
        path: /spec/scaleTargetRef/name
        value: provisioning-essentials-runner-team-a
    target:
      kind: HorizontalRunnerAutoscaler
      name: provisioning-essentials-runner-autoscaler
  - patch: |-                                 # 3. pins the image + labels
      - op: add
        path: /spec/template/spec/image
        value: ghcr.io/<your-org>/runner-images:ubuntu-22.04-provisioning-essentials
      - op: add
        path: /spec/template/spec/labels/-
        value: ubuntu-22.04
      - op: add
        path: /spec/template/spec/labels/-
        value: team-a
    target:
      kind: RunnerDeployment
      name: provisioning-essentials-runner
```

Repeat per team. To onboard a new GitHub org/tenant, copy a `runners-<team>/`
folder, change `nameSuffix`, label, and (if needed) the image tag.

### 15.3.4 Controller HelmRelease pattern

Each cluster runs **one ARC controller per GitHub org it serves** (so the
`scope.watchNamespace` matches a single tenant namespace). The HelmRelease
references a `HelmRepository` declared in `sources.yaml`, an `ExternalSecret`
syncs the PAT or GitHub App credentials from your chosen secret backend, and
an optional `ServiceMonitor` is enabled for the metrics server.

Keep one file per controller, e.g.
`actions-runner-controller-<orgname>.yaml`, so PR diffs stay focused.

### 15.3.5 IaC conventions

- One root module per environment (`iac/non-prod`, `iac/prod`).
- Module sources live in a shared `templates/` repo and are versioned (`/v1`).
- The workload-identity / federated-credential subject is hard-wired to the
  namespace + service-account used by your secret operator in the cluster
  (e.g. `system:serviceaccount:<ns>:secretstore-sa`).
- Remote state lives in the object store created by the same plan (bootstrap
  once with local state, then migrate).

### 15.3.6 Maintenance loop

1. Image bump from repo 1 → PR changing the `image:` patch in the relevant
   overlay(s). Flux picks it up on next reconcile.
2. New tenant onboarding → add a `runners-<team>/` base, a per-cluster
   overlay, and a controller HelmRelease.
3. Cluster onboarding → copy `gitops/<existing-cluster>/` to
   `gitops/<new-cluster>/`, then register it from repo 3.

---

## 15.4 Repo 3 — `runners-gitops` (the platform / GitOps repo)

### 15.4.1 What it owns

- The Flux bootstrap manifests for every cluster.
- The `GitRepository` and `Kustomization` CRs that point Flux at the
  application repo (repo 2) and at shared platform repos.
- Tenant registration via a `tenant-provision` Helm chart that creates the
  namespace, RBAC, ESO `SecretStore`, and a per-tenant Flux `GitRepository`.
- Locals (cluster ingress, service mesh) that are cluster-specific but not
  tenant-specific.

### 15.4.2 Directory layout to mirror

```
runners-gitops/
├── README.md
└── clusters/
    ├── nonprod-region1/                        # one folder per cluster
    │   ├── k8s-catalog/
    │   │   ├── flux-system/                    # `flux bootstrap` output
    │   │   │   ├── gotk-components.yaml
    │   │   │   ├── gotk-sync.yaml              # points at this cluster path
    │   │   │   └── kustomization.yaml
    │   │   ├── gitops-infra/                   # cert-manager, ESO, scanners, etc.
    │   │   ├── overlays/                       # one file per tenant
    │   │   │   ├── secret-operator.yaml
    │   │   │   ├── tenant-actions-runners-system.yaml
    │   │   │   ├── tenant-<other>.yaml
    │   │   │   └── tenant-<other>.yaml
    │   │   └── templates/                      # cluster-local helpers
    │   ├── locals/
    │   │   ├── cluster-ingress/
    │   │   └── service-mesh/
    │   └── tenants/
    │       └── actions-runners-system/
    │           ├── tenant-provisioning/
    │           │   └── actions-runners-system.yaml    # HelmRelease → tenant-provision chart
    │           └── tenant-config/
    │               ├── auth-allow-all.yaml
    │               ├── gateway.yaml
    │               └── ingress-gateway-certificate.yaml
    ├── nonprod-region1-win/                    # Windows cluster sibling
    ├── prod-region1/
    ├── prod-region2/
    ├── prod-region2-win/
    ├── prod-region3/
    └── prod-region4/
```

### 15.4.3 How Flux is wired

`flux-system/gotk-sync.yaml` declares the source and the cluster's root path:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata: { name: flux-system, namespace: flux-system }
spec:
  interval: 1m0s
  ref: { branch: nonprod }
  url: ssh://git@github.com/<you>/runners-gitops
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: { name: flux-system, namespace: flux-system }
spec:
  interval: 10m0s
  path: ./clusters/nonprod-region1/k8s-catalog
  prune: true
  sourceRef: { kind: GitRepository, name: flux-system }
```

From there, `k8s-catalog/overlays/tenant-actions-runners-system.yaml`
creates a Flux `Kustomization` for the tenant-provisioning HelmRelease, and
that HelmRelease in turn registers a tenant-scoped `GitRepository` pointing
at **repo 2** (the application repo):

```yaml
flux:
  enabled: true
  url: ssh://git@github.com/<you>/runners-platform.git
  path: ./gitops/nonprod-region1/runners-controller/overlays
  ref: { branch: main }
```

That overlay (`kustomizations.yaml`) finally creates the Flux `Kustomization`
that applies the controller manifests from
`gitops/nonprod-region1/runners-controller/controller-resources` into the
`actions-runners-system` namespace.

### 15.4.4 Branch strategy

| Branch | Cluster targets |
|--------|----------------|
| `nonprod` | every `*nonprod*` cluster's `GitRepository.ref.branch` |
| `main` (or `master`) | production clusters |

This lets you merge to `nonprod` first, verify with Flux, then promote with a
fast-forward merge to `main`.

### 15.4.5 Maintenance loop

1. New cluster: bootstrap Flux (`flux bootstrap github …`) into
   `clusters/<new>/k8s-catalog/flux-system/`, then copy a sibling cluster's
   `tenants/` and `overlays/`.
2. New tenant on an existing cluster: add
   `clusters/<cluster>/tenants/<tenant>/` and a matching overlay file.
3. Promote nonprod → prod: merge branch `nonprod` → `main` and let Flux on
   the prod clusters reconcile.

### 15.4.6 What to change for your setup

- Replace the `tenant-provision` Helm chart with your own (or drop it and
  inline namespace + RBAC manifests directly under `tenants/`).
- Plug in whatever secret backend your cluster uses (ExternalSecrets +
  cloud KMS, SealedSecrets, HashiCorp Vault, etc.).
- Replace any vendor-specific group / role IDs in RBAC with your own
  identity-provider group IDs.

---

## 15.5 End-to-end: how a change propagates

```
Developer commits to repo 1 (runner-images / dev/ubuntu-22.04-provisioning-essentials)
   │
   ▼  create-release.yml builds & pushes preview image to GHCR
   │  tag-release.yml mints v0.0.10-ubuntu-2204-provisioning-essentials
   │
Developer opens PR in repo 2 (runners-platform)
   bumping the `image:` patch in gitops/<cluster>/runners-<team>/overlays
   │
   ▼  Merge to `main`
   │
Flux running in cluster (configured from repo 3, runners-gitops)
   │  reconciles GitRepository → Kustomization → ARC scale set
   ▼
New runner pods spawn from the new image; old pods drain after current jobs.
```

A change in **repo 3** alone (e.g. registering a new tenant) does not need
repos 1 or 2. A change in **repo 2** alone (e.g. tuning autoscaler) does not
need repo 3 — Flux already watches that path. A change in **repo 1** alone
does not roll out until repo 2 is bumped (this is the intentional gate that
prevents auto-promoting broken images).

---

## 15.6 Minimum viable personal setup

If you want the same shape but cheaper / simpler, here is the smallest split
that still preserves the model:

| Yours | Equivalent of | What to keep | What to drop |
|-------|--------------|--------------|-------------|
| `my-runner-images` | repo 1 | Packer template, `create-release.yml`, GHCR push | Software-report diff, multiple OS variants, vendor licenses |
| `my-arc-stack` | repo 2 + this repo | `gitops/base` + per-cluster overlays; the umbrella Helm chart in [`charts/arc-runner-stack`](../charts/arc-runner-stack) | IaC + CI pipelines (use cloud CLI or local Helm install) |
| `my-gitops` | repo 3 | `flux bootstrap` output + a single `Kustomization` pointing at `my-arc-stack` | tenant-provision chart, multi-cluster folders, service-mesh locals |

For a single-cluster homelab you can collapse repo 2 + repo 3 entirely and
use the umbrella chart in this repository (see
[10-install-and-upgrade.md](10-install-and-upgrade.md)).

---

## 15.7 Conventions worth keeping (a checklist)

- **One folder per cluster** at the top of the GitOps repo. Never share files
  between clusters — use Kustomize bases to share content.
- **One folder per tenant / GitHub org** under each cluster. One ARC
  controller per tenant namespace; do not multiplex.
- **Image tag = immutable**. Never overwrite a tag in GHCR. Bump via PR.
- **Secrets via an ExternalSecrets-style operator only.** No PATs in Git.
- **Flux `path:` is always a folder, not a file.** Makes adding manifests a
  no-op diff.
- **Branches map to environments** (`nonprod`/`main`) consistently across
  all three repos.
- **CODEOWNERS** on every repo, scoped narrowly to the team that owns each
  subdirectory. Especially important on repo 2 where tenants share a repo.
- **Reusable workflows** (`workflow_call`) for image build/release in repo 1
  so each image variant only declares a thin trigger.

---

## 15.8 Where to look next

- For ARC internals and the umbrella chart: [02-architecture.md](02-architecture.md), [09-this-repo-stack.md](09-this-repo-stack.md)
- For authentication (PAT vs GitHub App): [07-authentication.md](07-authentication.md)
- For the security model that the namespace split enforces: [12-security-model.md](12-security-model.md)
- For how the listener registers and pulls jobs: [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md)
