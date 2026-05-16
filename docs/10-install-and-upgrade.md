# 10. Install, Upgrade, Uninstall

This chapter is the operational runbook: install from scratch, upgrade across minor versions (with the CRD caveat), rollback, drain, and uninstall. Trigger semantics on the GitHub side are documented in [14-listener-protocol-and-jit.md §2](14-listener-protocol-and-jit.md#2-listener-bootstrap-sequence) — useful when an install hangs at the listener-registration step.

## 10.1 Prerequisites

| Requirement | How to check |
|-------------|--------------|
| Kubernetes ≥ 1.27 | `kubectl version` |
| Helm ≥ 3.12 | `helm version` |
| External Secrets Operator installed | `kubectl get crd externalsecrets.external-secrets.io` |
| cert-manager (only if using webhook-based features) | `kubectl get crd certificates.cert-manager.io` |
| Azure Workload Identity enabled on the cluster (AKS) | `az aks show ... --query oidcIssuerProfile` |
| `github-pat` secret exists in Key Vault | `az keyvault secret show --vault-name <kv> --name github-pat` |

## 10.2 First install

```bash
# 1. Set required values
cat > my-values.yaml <<EOF
keyVault:
  name: "kv-bkcloudops-prod"
  tenantId: "<aad-tenant-id>"
managedIdentity:
  clientId: "<workload-identity-client-id>"
gha-runner-scale-set:
  githubConfigUrl: "https://github.com/BkCloudOps"
EOF

# 2. Install
helm install arc-stack ./charts/arc-runner-stack \
  --namespace arc-systems --create-namespace \
  -f my-values.yaml

# 3. Verify
kubectl get pods -n arc-systems
# Expect:
#   <release>-gha-rs-controller-...-...   Running
#   <release>-listener                    Running

kubectl get autoscalingrunnersets -A
kubectl get externalsecrets -n arc-runners
```

## 10.3 Triggering a test job

In any repo under your org, create `.github/workflows/test.yml`:
```yaml
on: workflow_dispatch
jobs:
  test:
    runs-on: straw-hat-runners-linux
    steps:
      - run: echo "hello from ARC"
```
Then trigger via GitHub UI. Within ~30s you should see:
```bash
kubectl get pods -n arc-runners --watch
# straw-hat-...-runner-xxx   0/2   ContainerCreating
# straw-hat-...-runner-xxx   2/2   Running
# straw-hat-...-runner-xxx   0/2   Completed
```

## 10.4 Upgrades — the CRD caveat

> Helm does **not** upgrade CRDs that already exist. This is by design (see Helm docs on CRDs).

### Safe upgrade (no CRD changes)
```bash
helm upgrade arc-stack ./charts/arc-runner-stack -n arc-systems -f my-values.yaml
```

### Full upgrade (with CRD changes — major version bump)
1. **Drain jobs:** Set `minRunners: 0, maxRunners: 0` and `helm upgrade`. Wait for `kubectl get pods -n arc-runners` to be empty.
2. **Uninstall runner scale set:** `helm uninstall arc-stack -n arc-systems` (this removes the AutoscalingRunnerSet first; controller cleans up).
3. **Wait** for all `arc-runners` resources to vanish.
4. **Delete old CRDs** if upgrading across a breaking change:
   ```bash
   kubectl delete crd autoscalingrunnersets.actions.github.com \
                      autoscalinglisteners.actions.github.com \
                      ephemeralrunners.actions.github.com \
                      ephemeralrunnersets.actions.github.com
   ```
5. **Re-pull** the new chart version:
   ```bash
   helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
     --version <new> --untar --destination charts/arc-runner-stack/charts/
   helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
     --version <new> --untar --destination charts/arc-runner-stack/charts/
   ```
6. Update `Chart.yaml` `dependencies[*].version` and `appVersion`.
7. **Reinstall** with `helm install`.

### Zero-downtime upgrades — High Availability
Deploy ARC in **two clusters** (different regions). Each runs a scale set with the **same `runnerScaleSetName`** but a **different `runnerGroup`**. GitHub distributes jobs arbitrarily; if one cluster is upgrading, the other absorbs traffic.

## 10.5 Rollback

```bash
helm history arc-stack -n arc-systems
helm rollback arc-stack <revision> -n arc-systems
```
**Limitations:** CRD schemas are NOT rolled back. If the upgrade added new required CRD fields, rollback can leave them dangling.

## 10.6 Uninstall

```bash
# Drain first
helm upgrade arc-stack ./charts/arc-runner-stack -n arc-systems \
  --set gha-runner-scale-set.minRunners=0 \
  --set gha-runner-scale-set.maxRunners=0
# Wait...
kubectl get pods -n arc-runners

# Uninstall
helm uninstall arc-stack -n arc-systems

# Manual cleanup (Helm leaves CRDs)
kubectl delete crd autoscalingrunnersets.actions.github.com \
                   autoscalinglisteners.actions.github.com \
                   ephemeralrunners.actions.github.com \
                   ephemeralrunnersets.actions.github.com
kubectl delete ns arc-systems arc-runners
```

Continue to [11-observability.md](11-observability.md).
