# 5. `gha-runner-scale-set` — Every Value Explained

File: [charts/arc-runner-stack/charts/gha-runner-scale-set/values.yaml](../charts/arc-runner-stack/charts/gha-runner-scale-set/values.yaml)

This chart is **installed once per runner pool**. Each install creates one `AutoscalingRunnerSet` CR.

---

## `githubConfigUrl: ""`  **(required)**

- **What:** Where the runners should attach. Determines the **scope**.

| URL pattern | Scope | Best for |
|-------------|-------|----------|
| `https://github.com/<org>/<repo>` | Repository runners | A single repo |
| `https://github.com/<org>` | Organization runners | Many repos in one org *(your case)* |
| `https://github.com/enterprises/<ent>` | Enterprise runners | Multiple orgs (needs Classic PAT) |

- **This repo uses:** `https://github.com/BkCloudOps` (organization scope).
- **Trailing slash is auto-stripped** by the template.

## `scaleSetLabels: []`

- **What:** Extra labels you can use in workflows: `runs-on: [self-hosted, gpu, linux]`.
- **Validation:** Non-empty strings, each < 256 chars.
- **Example:**
  ```yaml
  scaleSetLabels: ["linux", "x64", "gpu-a100"]
  ```

## `githubConfigSecret`  **(required, one of three forms)**

### Variation A — Inline PAT
```yaml
githubConfigSecret:
  github_token: "ghp_xxx"
```
- Helm creates a Secret containing the PAT. Easiest, but the PAT lives in `values.yaml`.

### Variation B — Inline GitHub App
```yaml
githubConfigSecret:
  github_app_id: "123456"
  github_app_installation_id: "654321"
  github_app_private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
```
- **Recommended over PAT** for orgs (better rate limits, finer permissions, no user lock-in).
- **Required permissions:** `Administration: Read & write`, `Actions: Read`, `Metadata: Read`.

### Variation C — Pre-existing Secret (string)  ← **what this repo uses**
```yaml
githubConfigSecret: github-pat-secret
```
- Refers to an existing Secret in the same namespace. The chart will **not** create one.
- The Secret must have either:
  - `github_token` key (for PAT), or
  - `github_app_id` + `github_app_installation_id` + `github_app_private_key` keys (for App).
- **In this repo,** the Secret is populated by an `ExternalSecret` that pulls `github-pat` from Azure Key Vault. See [07-authentication.md](07-authentication.md).

## `proxy:` (commented)
- HTTP/HTTPS proxy for **controller, listener, AND runner**.
- `credentialSecretRef`: Secret with keys `username` and `password`.
- `noProxy`: list of hostnames to bypass.

## `maxRunners` (commented, no default)

| Value | Behavior |
|-------|----------|
| unset | **Unbounded** — scale up to the number of queued jobs |
| `0` | Combined with `minRunners: 0` → **drain mode**; no new runners |
| `N` | Hard cap |

## `minRunners` (commented, default 0)
- Number of **idle warm runners** always ready.
- Reduces job start latency from ~30s → ~2s.
- **Formula GitHub uses:**
  ```
  desired = minRunners + assignedJobs   (capped by maxRunners)
  ```
- **Cost trade-off:** Warm runners burn Pod resources 24/7.

## `runnerGroup: "default"` (commented)
- GitHub runner groups restrict which repos/workflows can use this scale set.
- Must already exist in GitHub.

## `runnerScaleSetName: ""` (commented)
- Defaults to Helm release name.
- This is what workflows reference: `runs-on: <name>`.
- **Constraint:** ≤ 45 chars, unique within a runner group.

## `githubServerTLS:` (commented)
- For GitHub Enterprise Server with self-signed CA certs.
- Mounts the CA cert into runner Pods and sets `NODE_EXTRA_CA_CERTS`.

## `keyVault:` (commented)
- ARC-native integration with Azure Key Vault (alternative to External Secrets).
- This repo uses External Secrets instead for unified secret management.

## `containerMode:` (commented)

```yaml
containerMode:
  type: "dind"   # or "kubernetes" or "kubernetes-novolume"
```

- **Most important architectural decision.** Determines how `container:` and `services:` blocks in workflows run.
- Full deep-dive in [06-container-modes.md](06-container-modes.md).

| Mode | Runs Docker via | Trade-off |
|------|-----------------|-----------|
| `dind` | Docker-in-Docker sidecar (privileged) | Easy, but privileged → security risk |
| `kubernetes` | Sibling K8s Pods (RWX PVCs) | Secure, but needs shared storage |
| `kubernetes-novolume` | Sibling Pods + lifecycle hooks | Secure + no shared storage; runner must run as root |

## `listenerTemplate:` (commented)
- Customize the Listener Pod (resources, securityContext, sidecars).
- **Critical:** Container name **must remain `listener`**; renaming makes it a sidecar.
- **Example — set listener limits:**
  ```yaml
  listenerTemplate:
    spec:
      containers:
        - name: listener
          resources:
            limits: { cpu: "1", memory: 1Gi }
  ```

## `listenerMetrics:` (commented, big block)
- Defines which Prometheus metrics and labels the Listener emits.
- Without it, defaults are applied. Override to drop high-cardinality labels (e.g., `job_workflow_ref`) to control TSDB cost.
- Histogram bucket boundaries are also customizable — useful if your jobs are very short or very long.

## `template:`  **(the runner Pod spec)**

- This is a raw `PodSpec`. ARC merges it with whatever `containerMode` auto-injects.
- **Minimum required:**
  ```yaml
  template:
    spec:
      containers:
        - name: runner                                    # MUST be "runner"
          image: ghcr.io/actions/actions-runner:latest
          command: ["/home/runner/run.sh"]
  ```
- **The container MUST be named `runner`** or ARC won't inject env vars/volumes.

### Common customizations

**Add resource limits:**
```yaml
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        resources:
          requests: { cpu: "1", memory: 2Gi }
          limits:   { cpu: "4", memory: 8Gi }
```

**Pin to GPU nodes:**
```yaml
template:
  spec:
    nodeSelector:
      accelerator: nvidia-a100
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
    containers:
      - name: runner
        image: my-registry/gpu-runner:latest
        command: ["/home/runner/run.sh"]
        resources:
          limits:
            nvidia.com/gpu: "1"
```

**Add a custom CA cert:**
```yaml
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        volumeMounts:
          - name: ca
            mountPath: /usr/local/share/ca-certificates/extra.crt
            subPath: ca.crt
    volumes:
      - name: ca
        configMap: { name: corporate-ca }
```

## `controllerServiceAccount:` (commented)
- Tells this chart **which controller SA** owns it (needed for RoleBinding).
- **Only required** if `flags.watchSingleNamespace` is used on the controller, or if the chart can't auto-discover the controller.

## `namespaceOverride: ""`
- Override the target namespace.

## `annotations` / `labels`
- Applied to **all** resources created by this chart.

## `resourceMeta:` (commented, large block)
- Fine-grained per-resource labels/annotations (e.g., only stamp the `githubConfigSecret`).
- Useful for: tagging the Secret for External Secrets cleanup, applying ArgoCD ignore annotations to specific resources, etc.

---

## Production preset for this repo

```yaml
githubConfigUrl: "https://github.com/BkCloudOps"
githubConfigSecret: github-pat-secret      # synced via ExternalSecret from Key Vault
runnerScaleSetName: "straw-hat-runners-linux"
runnerGroup: "straw-hat-runners-linux"
minRunners: 1
maxRunners: 10
scaleSetLabels: ["linux", "x64"]
containerMode:
  type: "dind"
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        resources:
          requests: { cpu: "1", memory: 2Gi }
          limits:   { cpu: "4", memory: 8Gi }
```

Continue to [06-container-modes.md](06-container-modes.md).
