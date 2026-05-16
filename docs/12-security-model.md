# 12. Security Model

## 12.1 Threat model

ARC runner Pods **execute arbitrary code from your workflows**. Anyone who can push code (or open a PR with `pull_request_target`) can execute commands inside the cluster. The entire security model treats runner Pods as **untrusted workloads**.

## 12.2 Trust boundaries

```
HIGH TRUST                       LOW TRUST (executes user code)
┌────────────────────┐           ┌──────────────────────────────┐
│  arc-systems NS    │           │  arc-runners NS              │
│  ─ Controller       │  ◄──────  │  ─ Ephemeral runner Pods    │
│  ─ Listener         │  (no)     │                              │
│  ─ GitHub creds     │           │  Has: github-pat-secret       │
└────────────────────┘           │       (so jobs can call API) │
                                  └──────────────────────────────┘
```

**Goal:** A compromised runner Pod must **not** be able to:
- Read other tenants' secrets.
- Escalate to controller privileges.
- Pivot to the control plane or other namespaces.

## 12.3 Mandatory controls

### 1. Separate namespaces
- Controller / Listener in `arc-systems`.
- Runners in `arc-runners`.
- This repo enforces this in `templates/namespaces.yaml`.

### 2. NetworkPolicy (you must add)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-runners-to-controlplane
  namespace: arc-runners
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:                                         # Allow only egress to internet (GitHub) and DNS
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8                        # Block intra-cluster
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
```

### 3. PodSecurityStandards
Apply at the namespace level:
```bash
kubectl label ns arc-runners pod-security.kubernetes.io/enforce=baseline
kubectl label ns arc-systems pod-security.kubernetes.io/enforce=restricted
```
> If using `dind`, `arc-runners` cannot be `restricted` (privileged required). Use `baseline` and accept the trade-off, OR switch to `kubernetes-novolume` mode for `restricted` compliance.

### 4. ServiceAccount minimization
- `no_permission_serviceaccount.yaml` is mounted into dind-mode runner Pods. It has **zero** RBAC permissions. Workflows that try `kubectl ...` will get `Forbidden`.
- In Kubernetes mode, the SA gets minimum permissions to create job Pods in **the same namespace only**.

### 5. Image provenance
- Pin runner image by digest, not tag:
  ```yaml
  image: ghcr.io/actions/actions-runner@sha256:<digest>
  ```
- Mirror to a trusted registry; scan with Trivy / Grype in CI.

### 6. Secrets handling
- **Never** put PAT inline in `values.yaml` committed to Git.
- Use ESO + Key Vault (this repo) or Vault.
- Mount Secrets as files (read-only), not env vars, when possible.

## 12.4 Defense-in-depth options

| Control | Effort | Benefit |
|---------|--------|---------|
| Dedicated node pool for runners (taints + nodeSelector) | Low | Blast radius reduction |
| `gVisor` / `Kata` runtime for runner Pods | Medium | Container escape protection |
| OPA / Kyverno policies blocking dangerous workflows | High | Prevent crypto-mining, exfiltration |
| Network egress filter (Calico, Cilium) | Medium | Only allow GitHub, package mirrors |
| Runner Pod read-only root FS | Low | Stops binary persistence |
| Disable `actions/cache` writes | Low | Stops cache poisoning across PRs |
| `pull_request_target` discipline (only run trusted-author code) | Process | Stops malicious PRs |

## 12.5 Audit checklist (production)

- [ ] `arc-systems` and `arc-runners` are separate namespaces.
- [ ] NetworkPolicy restricts runner egress to GitHub + DNS only.
- [ ] PodSecurityStandards labels applied.
- [ ] PAT is sourced from Key Vault via ExternalSecret; **never** stored in Git.
- [ ] GitHub App used instead of PAT where possible.
- [ ] `runnerGroup` restricts which repos can use the scale set.
- [ ] Runner image is pinned by digest and scanned.
- [ ] Resource limits set on runner Pods (prevent DoS).
- [ ] Metrics + alerts in place (`gha_job_startup_duration_seconds`, listener up).
- [ ] Upgrade playbook documented (CRD caveat).
- [ ] Logs shipped to a SIEM with 90-day retention.

---

## End of documentation set

For changes, edit these files in `docs/`. Keep this set in sync with `Chart.yaml` and the vendored charts whenever you upgrade ARC.
