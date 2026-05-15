# ARC Runner Stack Configuration Chart

Configuration chart for GitHub Actions Runner infrastructure on Kubernetes.

> **Note**: This chart provides configuration resources only (ClusterSecretStore, ExternalSecrets, namespaces).
> The actual ARC, ESO, and cert-manager are deployed via GitOps in the [actions-runner-gitops](https://github.com/BkCloudOps/actions-runner-gitops) repo.

## What This Chart Creates

- **ClusterSecretStore**: Azure Key Vault connection using Workload Identity
- **ExternalSecret**: Syncs `github-pat` from Key Vault to cluster
- **ServiceAccount**: With Workload Identity annotations
- **Namespaces**: `arc-systems`, `arc-runners`, `external-secrets`

## Architecture

We use **GitHub-supported ARC** (from `ghcr.io/actions/...`), NOT the community version:

| Component | Source | Managed By |
|-----------|--------|------------|
| gha-runner-scale-set-controller | `ghcr.io/actions/actions-runner-controller-charts` | GitOps |
| gha-runner-scale-set | `ghcr.io/actions/actions-runner-controller-charts` | GitOps |
| external-secrets | `charts.external-secrets.io` | GitOps |
| cert-manager | `charts.jetstack.io` | GitOps |
| **This chart** | `ghcr.io/bkcloudops/charts` | GitOps |

## Usage

Deploy via GitOps HelmRelease:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: arc-config
  namespace: flux-system
spec:
  chart:
    spec:
      chart: arc-runner-stack
      sourceRef:
        kind: OCIRepository
        name: arc-runner-stack
  values:
    keyVault:
      name: "your-keyvault"
      tenantId: "your-tenant-id"
    managedIdentity:
      clientId: "your-mi-client-id"
```

## Values Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `githubConfigUrl` | GitHub org URL | `https://github.com/BkCloudOps` |
| `runnerGroup` | Runner group name | `straw-hat-runners-linux` |
| `keyVault.name` | Azure Key Vault name | `""` |
| `keyVault.tenantId` | Azure AD Tenant ID | `""` |
| `managedIdentity.clientId` | Managed Identity client ID | `""` |
| `clusterSecretStore.enabled` | Enable ClusterSecretStore | `true` |
| `externalSecrets.enabled` | Enable ExternalSecrets | `true` |

## Runner Group

**Name**: `straw-hat-runners-linux`

Use in workflows:
```yaml
jobs:
  build:
    runs-on: straw-hat-runners-linux
```

## License

MIT
