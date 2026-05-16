# 9. This Repository's Stack — How Everything Fits

## 9.1 Chart hierarchy

```
arc-runner-stack/                                       <- umbrella chart (this repo)
│
├── Chart.yaml                                          declares dependencies
│   dependencies:
│     - gha-runner-scale-set-controller v0.14.1         (vendored under charts/)
│     - gha-runner-scale-set            v0.14.1         (vendored under charts/)
│
├── values.yaml                                         your config — feeds both subcharts
│
├── templates/                                          <- this repo's own resources
│   ├── namespaces.yaml                                 arc-systems, arc-runners, external-secrets
│   ├── cluster-secret-store.yaml                       binds Azure Key Vault via Workload Identity
│   ├── external-secrets.yaml                           syncs github-pat -> github-pat-secret
│   └── service-account.yaml                            for ESO, with workload-identity annotation
│
└── charts/                                             <- vendored ARC charts (downloaded)
    ├── gha-runner-scale-set-controller/                installs the operator
    └── gha-runner-scale-set/                           installs one runner pool
```

## 9.2 Install order — handled by Helm automatically

When you run `helm install arc-runner-stack ./charts/arc-runner-stack`:

1. **CRDs** from `gha-runner-scale-set-controller/crds/` are applied first.
2. **Namespaces** (`arc-systems`, `arc-runners`, `external-secrets`) created by this repo's templates.
3. **External Secrets** resources:
   - ServiceAccount with workload-identity annotation
   - ClusterSecretStore pointing to Azure Key Vault
   - ExternalSecret declaring `github-pat-secret`
4. **Controller** Deployment in `arc-systems`.
5. **AutoscalingRunnerSet** CR in `arc-runners`.
6. Controller reconciles → spawns Listener Pod in `arc-systems`.
7. Listener authenticates to GitHub → registers the scale set → starts long-poll.

> **Critical ordering caveat** — Step 5 requires the Secret from step 3 to exist. If ESO is slow to sync, the Listener will CrashLoop briefly (then recover). For deterministic ordering, install in two phases or use Helm hooks.

## 9.3 Namespace responsibilities

| Namespace | Contents | Trust |
|-----------|----------|-------|
| `external-secrets` | ESO operator (assumed already installed cluster-wide) | High |
| `arc-systems` | Controller, Listener Pods | High (needs GitHub creds) |
| `arc-runners` | EphemeralRunner Pods, `github-pat-secret`, `AutoscalingRunnerSet` | **Untrusted** (runs user code) |

> Always keep `arc-systems` and `arc-runners` separate. NetworkPolicies should prevent runner pods from reaching the controller.

## 9.4 The umbrella `values.yaml`

Top-level keys feed subcharts by their chart name:

```yaml
# Feeds the umbrella chart's own templates
githubConfigUrl: "https://github.com/BkCloudOps"
keyVault:
  name: "kv-prod"
  tenantId: "..."
managedIdentity:
  clientId: "..."
clusterSecretStore:
  enabled: true
  name: azure-keyvault-store
externalSecrets:
  enabled: true
  secrets:
    - name: github-pat-secret
      namespace: arc-runners
      secretKey: github_token
      remoteKey: github-pat

# Feeds the gha-runner-scale-set-controller subchart
gha-runner-scale-set-controller:
  replicaCount: 1
  flags:
    logLevel: info

# Feeds the gha-runner-scale-set subchart
gha-runner-scale-set:
  githubConfigUrl: "https://github.com/BkCloudOps"
  githubConfigSecret: github-pat-secret
  minRunners: 0
  maxRunners: 5
  containerMode:
    type: "dind"
  template:
    spec:
      containers:
        - name: runner
          image: ghcr.io/actions/actions-runner:latest
          command: ["/home/runner/run.sh"]
```

> **Convention** — Subchart values are nested under a key matching the subchart's `name` in `Chart.yaml`.

## 9.5 Why vendor the charts (download into `charts/`)?

| Benefit | Detail |
|---------|--------|
| **Reproducible builds** | The exact chart version is committed; no surprise pulls at install time |
| **Air-gap friendly** | Works without registry access at deploy time |
| **Auditability** | You can read every template that will be applied |
| **GitOps** | Tools like Flux/ArgoCD render templates from disk |

To refresh:
```bash
helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version <new-version> --untar --destination charts/arc-runner-stack/charts/
```

Continue to [10-install-and-upgrade.md](10-install-and-upgrade.md).
