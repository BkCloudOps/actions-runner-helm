# 7. Authentication

This chapter is the complete reference for **how ARC authenticates to GitHub**. It covers every supported credential type, every permission each one requires, the lifecycle of derived tokens (ART, installation tokens, JIT registration credentials), and the recommended pattern for storing the source credential outside the cluster.

If you only need the wire-level detail of how a credential becomes a runner registration, see [14-listener-protocol-and-jit.md §2.2](14-listener-protocol-and-jit.md#22-mint-a-github-credential).

---

## 7.1 What needs to authenticate, to what?

There are **three** distinct credentials in play, often conflated. Keep them straight:

| Credential | Held by | Lifetime | Purpose |
|------------|---------|----------|---------|
| **Primary credential** — PAT or GitHub App private key | The Kubernetes `Secret` named in `AutoscalingRunnerSet.spec.githubConfigSecret` | Hours (App JWT) → years (PAT) | Authenticate REST API calls: register scale set, generate JIT configs, deregister runners |
| **Actions Runtime Token (ART)** | Listener process memory | ~1 hour, refreshed automatically | Authenticate against the RunService (long-poll, acquire jobs, generate JIT) |
| **JIT runner registration credential** | Embedded in `encodedJITConfig` env var of each runner Pod | One-time-use, ~1 hour validity window | Register *one specific runner instance* with GitHub |

Only the **primary credential** is something you manage. ARC mints the other two automatically.

---

## 7.2 Choosing a primary credential type

| Method | Works for | Strengths | Weaknesses |
|--------|-----------|-----------|------------|
| **Personal Access Token (Classic)** | Repo, Org, Enterprise | One-line setup; works in every plan | Bound to a human user; rotates when they leave; coarse scopes; rate-limited per user |
| **Fine-grained PAT** | Repo, Org (preview) | Per-repo selectability; expiration enforced | Org support is still limited; admin features incomplete |
| **GitHub App** | Repo, Org | Decoupled from any user; richer rate limits (5,000/hr/installation × 1 per scope); fine-grained permissions; audit-friendly | Two-stage credential (private key → JWT → installation token); more moving parts |
| **GitHub App via ESO + Workload Identity** | Repo, Org | All of GitHub App's strengths, plus the private key never lives in Git or Helm values | Requires External Secrets Operator and a cloud identity provider |

Recommendation: **GitHub App + ESO** for any non-trivial deployment. PAT is acceptable for proofs of concept and lab clusters. Enterprise-scope runners *require* PAT (Classic) today because GitHub Apps cannot yet hold enterprise scopes.

---

## 7.3 Required permissions

### 7.3.1 PAT (Classic) scopes

| Scope | Required when the scale set is attached to a... |
|-------|-------------------------------------------------|
| `repo` | Repository |
| `admin:org` | Organization |
| `manage_runners:enterprise` | Enterprise |
| `read:packages` | Optional — only if the runner image is pulled from a private GHCR. Better practice: use `imagePullSecrets` on the runner Pod. |

A single token may carry multiple scopes; in that case the same secret can serve multiple scale sets at different scopes.

### 7.3.2 Fine-grained PAT permissions

For repository scope: *Repository permissions → Actions: Read & Write*, *Administration: Read & Write*. Organization scope requires the org owner to grant the token.

### 7.3.3 GitHub App permissions

| Permission | Access | Required for |
|------------|--------|--------------|
| **Actions** | Read | Listing workflow jobs, receiving messages |
| **Administration** | Read & Write | Registering scale sets and runners |
| **Metadata** | Read | Implicit; always required |
| **Self-hosted runners** (org-level only) | Read & Write | Managing org-level runner groups |
| **Organization administration** | Read & Write | Only if you want ARC to create runner groups on demand (rare) |

The App must be **installed** on the target organization (or specific repositories within it). The installation ID is the value you place in the Secret under `github_app_installation_id`.

---

## 7.4 Shape of the `githubConfigSecret`

`AutoscalingRunnerSet.spec.githubConfigSecret` may be:

1. **A string** — the name of an existing Kubernetes `Secret` in the same namespace. **Recommended.** Enables ESO, manual rotation, Sealed-Secrets, etc.
2. **An object** — an inline `github_token` literal. The chart materializes it into a Secret. **Avoid in production**; the value ends up in Helm release manifests and possibly Git.

The referenced Secret must contain **exactly one** of these key-sets:

### 7.4.1 PAT mode

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: arc-github-config
  namespace: arc-runners
type: Opaque
stringData:
  github_token: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 7.4.2 GitHub App mode

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: arc-github-config
  namespace: arc-runners
type: Opaque
stringData:
  github_app_id: "123456"
  github_app_installation_id: "78901234"
  github_app_private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEA...
    -----END RSA PRIVATE KEY-----
```

> Mixing the two modes in one Secret is **a validation error**. The listener refuses to start. Watch for this if you reuse a Secret across charts.

### 7.4.3 Schema validation

The listener enforces, at startup:

| Rule | Failure mode |
|------|--------------|
| Exactly one mode's keys present | Pod `CrashLoopBackOff` with log `unexpected secret configuration` |
| `github_token` non-empty | Pod log `empty github_token` |
| `github_app_private_key` parses as RSA PEM | Pod log `failed to parse private key` |
| `github_app_id` and `github_app_installation_id` are integers | Pod log `invalid integer for ...` |

---

## 7.5 The credential pipeline at runtime

Once the listener has the primary credential, it derives short-lived tokens automatically. The full chain for a GitHub App is:

```
github_app_private_key (RSA)
        │
        ▼ (sign JWT: iss=app_id, exp=now+10min)
App JWT  ──────► POST /app/installations/<id>/access_tokens
                                                │
                                                ▼
                                  Installation Access Token   (valid ~1h)
                                                │
                  ┌─────────────────────────────┴───────────────────────────┐
                  ▼                                                         ▼
   POST /actions/runner-registration                                Subsequent REST calls
                  │                                                  (deregister runner, etc.)
                  ▼
   Actions Runtime Token (ART) + RunService URL    (valid ~1h)
                  │
                  ▼
   Authorization: Bearer <ART>   ── used for all RunService calls
                                     ├── POST /sessions             → messageQueueAccessToken (10 min)
                                     ├── POST /generatejitconfig    → encodedJITConfig (per runner)
                                     └── GET  /messages?...         → long-poll
```

For a PAT, the top-left RSA→JWT→Installation-Token leg is skipped; the PAT *is* the credential used to call `/actions/runner-registration`.

**Refresh policy.**

| Token | Refreshed when |
|-------|----------------|
| Installation access token (App) | ~5 minutes before expiry |
| ART | ~10 minutes before expiry |
| `messageQueueAccessToken` | Returned with every poll response; replaced each cycle |
| JIT registration credential | Never refreshed — single-use |

---

## 7.6 The recommended pattern: ESO + Workload Identity

This repository implements the pattern below. It ensures the primary credential never touches Git, Helm values, or a developer laptop.

```
Azure Key Vault                Kubernetes (AKS)                              GitHub
┌──────────────────┐         ┌──────────────────────────────┐              ┌──────────────────┐
│ secret:          │         │ ClusterSecretStore           │              │                  │
│   github-pat ────┼────────►│   azure-keyvault-store       │              │                  │
│                  │         │   auth: WorkloadIdentity      │              │                  │
└──────────────────┘         │           │                  │              │                  │
        ▲                    │           ▼                  │              │  GitHub Actions  │
        │                    │ ExternalSecret               │              │  Service         │
        │ (ESO process       │   pulls every 1h             │              │                  │
        │  presents OIDC     │           │                  │              │                  │
        │  token; AAD trusts │           ▼                  │              │                  │
        │  the K8s SA)       │ Secret: arc-github-config    │              │                  │
        │                    │   key: github_token          │              │                  │
        │                    │           ▲                  │              │                  │
        └────────────────────┤           │ referenced       │ HTTPS         │                  │
                             │ AutoscalingRunnerSet         ├──────────────►                  │
                             │   githubConfigSecret:        │              │                  │
                             │     arc-github-config        │              │                  │
                             └──────────────────────────────┘              └──────────────────┘
```

### 7.6.1 Pieces

1. **Azure Key Vault** holds the PAT (or App private key) at a fixed secret name (e.g. `github-pat`).
2. A **Workload Identity-enabled ServiceAccount** in the `external-secrets` namespace is annotated:

   ```yaml
   annotations:
     azure.workload.identity/client-id: "<UAMI client id>"
   labels:
     azure.workload.identity/use: "true"
   ```

   The User-Assigned Managed Identity (UAMI) has a Federated Identity Credential trusting AKS's OIDC issuer for this SA.

3. **`ClusterSecretStore`** (`azure-keyvault-store`) references the Key Vault URL, Tenant ID, and the SA above.
4. **`ExternalSecret`** in `arc-runners` references the store and pulls `github-pat` into a Kubernetes Secret named `arc-github-config`, refreshing hourly.
5. **`AutoscalingRunnerSet.spec.githubConfigSecret: arc-github-config`** — the chart passes the *name* (string form).

### 7.6.2 Why each piece exists

| Without it... | The risk |
|---------------|----------|
| Without ESO | The PAT must be in Helm values → in Git → in your CI logs → potentially in your container registry. |
| Without Workload Identity | The ESO controller needs a long-lived Azure credential — typically a service principal client secret — which becomes the next thing to rotate. |
| Without the indirection of `ClusterSecretStore` | Every namespace would need its own `SecretStore`, multiplying the federated credential count. |

### 7.6.3 AWS / GCP / on-prem variants

The pattern is identical with the auth substituted:

| Cloud | Vault | Identity binding |
|-------|-------|------------------|
| AWS | AWS Secrets Manager / Parameter Store | IAM Role for ServiceAccount (IRSA) |
| GCP | GCP Secret Manager | GKE Workload Identity Federation |
| On-prem | HashiCorp Vault | Kubernetes auth method on Vault |

In each case the `ClusterSecretStore.spec.provider` changes; the ARC side stays the same.

---

## 7.7 Rotation playbooks

### 7.7.1 PAT — inline (anti-pattern, included for completeness)

```bash
helm upgrade arc-runners ./charts/arc-runner-stack \
  --set runnerScaleSet.githubConfigSecret.github_token="$NEW_PAT"
# Listener picks up new value on next pod restart
kubectl delete pod -n arc-systems -l app.kubernetes.io/component=listener
```

### 7.7.2 PAT — via ESO

```bash
# 1) Rotate in Key Vault
az keyvault secret set --vault-name <kv> --name github-pat --value "$NEW_PAT"

# 2) Wait for ESO refresh window (default refreshInterval: 1h), OR force:
kubectl annotate externalsecret -n arc-runners arc-github-config \
  force-sync=$(date +%s) --overwrite

# 3) Force the listener to pick up the new Secret content
kubectl delete pod -n arc-systems -l app.kubernetes.io/component=listener
```

### 7.7.3 GitHub App private key

```bash
# 1) Generate new key in the GitHub App settings UI; download .pem
# 2) Push to Key Vault
az keyvault secret set --vault-name <kv> --name github-app-key \
  --file ./app.private.pem
# 3) Force sync + listener restart as above
# 4) After confirming new key works, REVOKE the old key in GitHub App settings
```

Always revoke the old credential **after** confirming the new one works. The reverse leaves a window where the listener is locked out.

### 7.7.4 Compromise response

If you believe a primary credential has leaked:

1. **Revoke first, rotate second.** Revoke in GitHub UI immediately. ARC will degrade (no new runners) but jobs in flight finish.
2. Rotate per §7.7.2 or §7.7.3.
3. Audit: `kubectl logs -n arc-systems deploy/arc-gha-rs-controller --since=24h | grep -i 'register\|generatejit'` — look for unfamiliar runner names.
4. On GitHub: *Settings → Audit log* filtered by `action:repo.self_hosted_runner` to inspect external registrations.

---

## 7.8 Common errors

| Symptom | Likely cause | Resolution |
|---------|--------------|-----------|
| Listener `CrashLoopBackOff`, log `401 Bad credentials` | PAT expired, revoked, or wrong | Rotate; check expiration in GitHub UI |
| Listener log `403 Resource not accessible by integration` | GitHub App missing **Administration: Write** | Edit App permissions; the App must accept the change in the org install |
| Listener log `404` on `/runnerscalesets` | Wrong `githubConfigUrl` (e.g. specified a repo URL with admin:org PAT) | Align URL and credential scope |
| Listener log `failed to parse private key` | Newlines stripped from PEM (common when stored as Helm `--set-string`) | Use `--set-file github_app_private_key=app.pem` or store via ESO |
| `Secret arc-github-config not found` | ESO not yet synced; or wrong namespace | `kubectl get externalsecret -n arc-runners` and inspect `.status` |
| Sudden 401 storm after weeks of working | App installation token cache stale or skew | Restart the listener; consider widening NTP tolerance |
| GitHub App works for some scale sets and not others | App not installed on the specific repo when using repo-scope | Install on all targeted repos, or move to org scope |

---

## 7.9 Cross-references

- [13-crd-reference.md §AutoscalingRunnerSet](13-crd-reference.md#kind-autoscalingrunnerset) — every field of `githubConfigSecret`
- [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md) — the wire calls these credentials authenticate
- [12-security-model.md](12-security-model.md) — namespace/RBAC isolation that constrains blast radius if a credential leaks
- [09-this-repo-stack.md](09-this-repo-stack.md) — how the ESO templates in this repo wire everything together

Continue to [08-scaling-behavior.md](08-scaling-behavior.md).
