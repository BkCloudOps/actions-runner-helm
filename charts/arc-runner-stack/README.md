# ARC Runner Stack

Complete GitHub Actions Runner infrastructure for Kubernetes with:

- **Actions Runner Controller (ARC)** - Self-hosted GitHub runners
- **External Secrets Operator (ESO)** - Sync secrets from Azure Key Vault
- **cert-manager** - Certificate management

## Quick Start

```bash
# Add OCI registry
helm registry login ghcr.io -u USERNAME

# Install
helm install arc-stack oci://ghcr.io/bkcloudops/actions-runner-helm/arc-runner-stack \
  --namespace arc-systems \
  --create-namespace \
  --set global.githubConfigUrl="https://github.com/YOUR_ORG" \
  --set global.runnerGroup="straw-hat-runners-linux" \
  --set global.keyVault.name="your-keyvault" \
  --set global.keyVault.tenantId="your-tenant-id" \
  --set global.managedIdentity.clientId="your-mi-client-id"
```

## Prerequisites

1. **AKS Cluster** with:
   - Workload Identity enabled
   - OIDC Issuer enabled

2. **Azure Key Vault** with secrets:
   - `github-pat` - GitHub PAT with `repo`, `admin:org`, `workflow` scopes

3. **Managed Identity** with:
   - Key Vault Secrets User role on the Key Vault
   - Federated credentials for Kubernetes workload identity

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AKS Cluster                               │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │   cert-manager   │  │   External       │  │ ARC Controller│  │
│  │   (namespace)    │  │   Secrets (ESO)  │  │ (arc-systems) │  │
│  └──────────────────┘  └────────┬─────────┘  └───────┬───────┘  │
│                                 │                     │          │
│                    ┌────────────┴─────────────────────┘          │
│                    ▼                                             │
│         ┌──────────────────────┐                                 │
│         │  ClusterSecretStore  │──────┐                          │
│         │  (Azure Key Vault)   │      │                          │
│         └──────────────────────┘      │                          │
│                                       ▼                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                     arc-runners (namespace)                 │ │
│  │  ┌─────────────────┐  ┌─────────────────────────────────┐  │ │
│  │  │ ExternalSecret  │  │     RunnerDeployment            │  │ │
│  │  │ (github-pat)    │──│     straw-hat-runners-linux     │  │ │
│  │  └─────────────────┘  │     (min: 1, max: 10)           │  │ │
│  │                       └─────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
                    ┌───────────────────────┐
                    │   Azure Key Vault     │
                    │   - github-pat        │
                    │   - acr-password      │
                    └───────────────────────┘
```

## Values Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.githubConfigUrl` | GitHub org/repo URL | `https://github.com/BkCloudOps` |
| `global.runnerGroup` | Runner group name | `straw-hat-runners-linux` |
| `global.keyVault.name` | Azure Key Vault name | `""` |
| `global.keyVault.tenantId` | Azure tenant ID | `""` |
| `global.managedIdentity.clientId` | Managed Identity client ID | `""` |
| `arc-runners.minRunners` | Minimum runner count | `1` |
| `arc-runners.maxRunners` | Maximum runner count | `10` |
| `external-secrets.enabled` | Enable ESO | `true` |
| `cert-manager.enabled` | Enable cert-manager | `true` |

## Using Self-Hosted Runners

After deployment, use in your workflows:

```yaml
jobs:
  build:
    runs-on: straw-hat-runners-linux
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on self-hosted AKS runner!"
```

## License

MIT
