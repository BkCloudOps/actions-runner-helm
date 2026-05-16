# 7. Authentication — How ARC Talks to GitHub

ARC needs credentials to:
1. Register the runner scale set with GitHub.
2. Request **JIT tokens** for each new runner.
3. Receive `Job Available` messages on the Listener.

## 7.1 Three authentication options

| Method | Where it works | Pros | Cons |
|--------|----------------|------|------|
| **PAT (Classic)** | Repo, Org, **Enterprise** | Simplest | User-bound; rotates with user; coarse permissions |
| **GitHub App** | Repo, Org | Fine-grained perms; better rate limits; not tied to a user | More setup |
| **Pre-defined Secret** | Same as above, but secret is provisioned **out of band** | Decouples Helm release from secret rotation; works with ESO / Vault | Operator must populate the secret |

## 7.2 Required permissions

### PAT (Classic)
| Scope | Why |
|-------|-----|
| `repo` | Repo-level runners |
| `admin:org` | Org-level runners |
| `manage_runners:enterprise` | Enterprise-level runners |

### GitHub App
| Permission | Level |
|------------|-------|
| Actions | Read |
| Administration | Read & write |
| Metadata | Read |

## 7.3 What this repo does — ESO + Workload Identity

This repo's `arc-runner-stack` glues three pieces together:

```
Azure Key Vault                Kubernetes (AKS)                    GitHub
┌──────────────────┐         ┌──────────────────────────────┐    ┌────────┐
│ secret:          │         │ ExternalSecret (CRD)         │    │        │
│   github-pat ────┼────────►│   pulls every refreshInterval├───►│ ARC    │
│                  │         │           │                  │    │ Listener│
└──────────────────┘         │           ▼                  │    │ uses it│
        ▲                    │ Secret: github-pat-secret    │    │        │
        │                    │   key: github_token          │    └────────┘
        │ AAD Workload       │           ▲                  │
        │ Identity (OIDC)    │           │ referenced by    │
        │                    │ AutoscalingRunnerSet         │
        │                    │   githubConfigSecret:        │
        │                    │     github-pat-secret        │
        │                    └──────────────────────────────┘
        │
   ServiceAccount (with azure.workload.identity/client-id annotation)
   used by External Secrets Operator
```

### Flow

1. **Azure Key Vault** holds the PAT under secret name `github-pat`.
2. **ClusterSecretStore** (`azure-keyvault-store`) is configured with the Key Vault URL + Tenant ID.
3. **External Secrets Operator** (ESO) runs as a ServiceAccount annotated with `azure.workload.identity/client-id`. AAD trusts that SA via a **Federated Identity Credential**.
4. **ExternalSecret** `github-pat-secret` references the ClusterSecretStore and:
   - Pulls `github-pat` from Key Vault.
   - Materializes a K8s `Secret` named `github-pat-secret` with key `github_token`.
   - Refreshes on `refreshInterval`.
5. **AutoscalingRunnerSet** references `githubConfigSecret: github-pat-secret`.
6. ARC controller reads the Secret and uses the token for all GitHub API calls.

### Why this is better than inline PAT
- PAT never appears in Git, in `values.yaml`, or in Helm release manifests.
- Rotation: change in Key Vault → ESO syncs within `refreshInterval` → next reconcile picks it up.
- Audit: Key Vault logs every read.

## 7.4 Rotating credentials

| Method | How to rotate |
|--------|---------------|
| Inline PAT | `helm upgrade` with new value (downtime: 0 if `updateStrategy=eventual`) |
| GitHub App | New private key → update Secret → `kubectl delete pod -l app.kubernetes.io/component=listener` to force pickup |
| Pre-defined via ESO | Update value in Key Vault → wait `refreshInterval` → delete listener pod |

## 7.5 Common errors

| Symptom | Cause |
|---------|-------|
| Listener CrashLoopBackOff, log: `401 Bad credentials` | PAT expired or wrong scopes |
| Listener log: `403 Resource not accessible by integration` | GitHub App missing Administration: write |
| `Secret github-pat-secret not found` | ExternalSecret hasn't synced yet, or wrong namespace |
| Listener log: `connection refused` to `*.actions.githubusercontent.com` | Egress firewall — open HTTPS to `*.github.com`, `*.githubusercontent.com` |

Continue to [08-scaling-behavior.md](08-scaling-behavior.md).
