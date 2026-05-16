# ARC Custom Resource Definitions — Complete Reference

> API group: `actions.github.com`
> API version: `v1alpha1`
> Scope: Namespaced (all four kinds)
> Source of truth: [`charts/arc-runner-stack/charts/gha-runner-scale-set-controller/crds/`](../charts/arc-runner-stack/charts/gha-runner-scale-set-controller/crds)

This document is the field-by-field reference for every Custom Resource Definition (CRD) installed by the `gha-runner-scale-set-controller` Helm chart. It is written in the style of the official Kubernetes API reference: every field, every nuance, every default, every validation rule, every operational caveat.

If you only need the *concept* level, read [02-architecture.md](02-architecture.md) first; this file is the *reference* you come back to.

---

## Table of contents

1. [Reading this document](#reading-this-document)
2. [The four CRDs at a glance](#the-four-crds-at-a-glance)
3. [Cross-cutting types](#cross-cutting-types)
   - [`ResourceMeta`](#resourcemeta)
   - [`ProxyConfig`](#proxyconfig)
   - [`GitHubServerTLSConfig`](#githubservertlsconfig)
   - [`VaultConfig`](#vaultconfig)
   - [`MetricsConfig`](#metricsconfig)
4. [`AutoscalingRunnerSet`](#kind-autoscalingrunnerset)
5. [`AutoscalingListener`](#kind-autoscalinglistener)
6. [`EphemeralRunnerSet`](#kind-ephemeralrunnerset)
7. [`EphemeralRunner`](#kind-ephemeralrunner)
8. [Ownership graph and finalizers](#ownership-graph-and-finalizers)
9. [Phase state machines](#phase-state-machines)
10. [Validation rules summary](#validation-rules-summary)
11. [Common operational queries](#common-operational-queries)

---

## Reading this document

Each field entry follows this shape:

> **`fieldName`** *`<type>`* — required/optional — defaulting/immutability notes.
> Description, semantics, gotchas.

- **Required** means the API server rejects the object if the field is absent.
- **Immutable** means the field is fixed once the object is created. The controller reads it but no admission policy enforces immutability today — changing it produces undefined behavior. Treat as immutable.
- **Managed by controller** means humans should not set the field; the controller writes it.

`PodTemplateSpec`, `PodSpec`, `Affinity`, `Toleration`, `Container`, `Volume`, etc., are the **standard Kubernetes types** as defined in `core/v1`. ARC embeds them verbatim. The fields are too numerous to repeat here; consult [`kubectl explain pod.spec`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.29/#podspec-v1-core) or the Kubernetes API reference. Where ARC interprets a particular Pod field specially, this document calls it out.

---

## The four CRDs at a glance

| Kind | Plural | Short name | Status subresource | Typical creator | Typical namespace |
|------|--------|------------|--------------------|-----------------|-------------------|
| `AutoscalingRunnerSet` | `autoscalingrunnersets` | — | yes | Helm (user) | `arc-runners` |
| `AutoscalingListener` | `autoscalinglisteners` | — | yes (empty) | Controller | `arc-systems` |
| `EphemeralRunnerSet` | `ephemeralrunnersets` | — | yes | Controller (owned by ARS) | `arc-runners` |
| `EphemeralRunner` | `ephemeralrunners` | — | yes | Controller (owned by ERS) | `arc-runners` |

All four CRDs are **namespaced**, served at `actions.github.com/v1alpha1`, and use the standard `/status` subresource (so `.status` updates do not bump `.metadata.generation`).

The CRD manifests on disk:

- [actions.github.com_autoscalingrunnersets.yaml](../charts/arc-runner-stack/charts/gha-runner-scale-set-controller/crds/actions.github.com_autoscalingrunnersets.yaml)
- [actions.github.com_autoscalinglisteners.yaml](../charts/arc-runner-stack/charts/gha-runner-scale-set-controller/crds/actions.github.com_autoscalinglisteners.yaml)
- [actions.github.com_ephemeralrunnersets.yaml](../charts/arc-runner-stack/charts/gha-runner-scale-set-controller/crds/actions.github.com_ephemeralrunnersets.yaml)
- [actions.github.com_ephemeralrunners.yaml](../charts/arc-runner-stack/charts/gha-runner-scale-set-controller/crds/actions.github.com_ephemeralrunners.yaml)

---

## Cross-cutting types

These nested types appear inside multiple CRDs.

### `ResourceMeta`

Metadata common to internal resources that the controller creates from a parent CR. Used as the type of `*Metadata` fields (e.g. `ephemeralRunnerMetadata`, `listenerRoleMetadata`).

| Field | Type | Description |
|-------|------|-------------|
| `annotations` | `map[string]string` | Annotations to set on the resource the controller materializes. |
| `labels` | `map[string]string` | Labels to set on the resource the controller materializes. |

Notes:
- These do **not** propagate to grandchildren. For example, `ephemeralRunnerSetMetadata` annotates the `EphemeralRunnerSet` but **not** its child `EphemeralRunner` pods.
- ARC adds its own labels regardless (e.g. `actions.github.com/scale-set-name`, `actions.github.com/scale-set-namespace`). User labels are merged on top but cannot override reserved keys.

### `ProxyConfig`

Outbound HTTP(S) proxy used by ARC components when talking to GitHub.

| Field | Type | Description |
|-------|------|-------------|
| `http` | `ProxyServerConfig` | Proxy for `http://` URLs. |
| `https` | `ProxyServerConfig` | Proxy for `https://` URLs. |
| `noProxy` | `[]string` | List of hostnames/CIDRs that bypass the proxy. |

`ProxyServerConfig`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | `string` | yes | Proxy URL, e.g. `http://proxy.corp:3128`. |
| `credentialSecretRef` | `string` | no | Name of a `Secret` in the same namespace with keys `username` and `password` for proxy basic auth. |

Used by both the **listener** (outbound to `api.github.com`) and the **runner pods** (outbound to GitHub for actions/registry).

### `GitHubServerTLSConfig`

Pinned TLS bundle for GitHub Enterprise Server with a private CA.

| Field | Type | Description |
|-------|------|-------------|
| `certificateFrom` | `TLSCertificateSource` | Source of the CA bundle. |

`TLSCertificateSource.configMapKeyRef`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | yes | `ConfigMap` name in the same namespace. |
| `key` | `string` | yes | Key inside the `ConfigMap` whose value is the PEM bundle. |
| `optional` | `bool` | no | If `true`, missing ConfigMap is tolerated. |

### `VaultConfig`

Pluggable secrets backend. Currently only Azure Key Vault is implemented.

| Field | Type | Description |
|-------|------|-------------|
| `type` | `string` | Vault type. Today: `azure_key_vault`. Empty string means "do not use a vault". |
| `azureKeyVault` | `AzureKeyVaultConfig` | Settings when `type=azure_key_vault`. |
| `proxy` | `ProxyConfig` | Optional proxy for reaching the vault. |

`AzureKeyVaultConfig` (all four sub-fields **required** when `type=azure_key_vault`):

| Field | Type | Description |
|-------|------|-------------|
| `url` | `string` | Vault URL, e.g. `https://my-kv.vault.azure.net/`. |
| `tenantId` | `string` | Entra ID tenant. |
| `clientId` | `string` | Workload Identity / managed identity client ID. |
| `certificatePath` | `string` | Path inside the listener pod where the client certificate is mounted. |

> If you are using **External Secrets Operator + Workload Identity** (the pattern in this repo, see [07-authentication.md](07-authentication.md)), you typically do **not** set `vaultConfig`. ESO syncs the secret into Kubernetes and the listener reads it as a normal `Secret`. `vaultConfig` is for ARC's own direct vault integration which bypasses ESO.

### `MetricsConfig`

Configuration of Prometheus metrics emitted by the listener.

| Field | Type | Description |
|-------|------|-------------|
| `counters` | `map[string]CounterMetric` | Counter metric definitions (name → config). |
| `gauges` | `map[string]GaugeMetric` | Gauge metric definitions. |
| `histograms` | `map[string]HistogramMetric` | Histogram metric definitions (with bucket boundaries). |

Each sub-type carries `labels []string` (label dimensions) and, for histograms, `buckets []float64`.

The controller chart ships a default `MetricsConfig` for the listener; you only override it to add custom labels or disable metrics. See [11-observability.md](11-observability.md).

---

## Kind: `AutoscalingRunnerSet`

> The **public API** of ARC. This is the only CR you create by hand (via Helm). It declares "I want a pool of runners attached to this GitHub URL with these scaling bounds and this pod template."

| Property | Value |
|----------|-------|
| Group/Version | `actions.github.com/v1alpha1` |
| Kind | `AutoscalingRunnerSet` |
| Plural | `autoscalingrunnersets` |
| Scope | Namespaced |
| Created by | Helm (`gha-runner-scale-set` chart) |
| Owns | exactly one `AutoscalingListener` (cluster-wide, in `arc-systems`) and zero-or-one **active** `EphemeralRunnerSet` (in the same namespace as the ARS) |
| Status subresource | yes |

### `.spec`

`AutoscalingRunnerSetSpec` defines the desired state.

#### Required fields

> **`githubConfigUrl`** *`string`* — **required** — immutable.
> The GitHub URL the runners attach to. Three forms accepted by GitHub:
> - Repository: `https://github.com/<owner>/<repo>`
> - Organization: `https://github.com/<org>`
> - Enterprise: `https://github.com/enterprises/<enterprise>`
> For GitHub Enterprise Server, replace the host accordingly. The listener uses this URL to call `api.github.com/...` (or `<ghes>/api/v3/...`) and register the runner scale set. Changing this after install requires deleting and recreating the runner scale set; in practice, uninstall+install the Helm release.

> **`githubConfigSecret`** *`string` or `GitHubServerConfigSecret`* — **required**.
> Either the **name** of a Kubernetes `Secret` in the same namespace (string form) **or** an inline reference. The Secret must contain **exactly one** authentication mode:
> - `github_token` (PAT or fine-grained token), **or**
> - `github_app_id`, `github_app_installation_id`, `github_app_private_key` (GitHub App).
>
> When the secret name (string form) is used, ARC reads keys from that Secret directly. The secret may be created by External Secrets Operator (see [07-authentication.md](07-authentication.md)).

#### Scaling bounds

> **`minRunners`** *`int`* — optional — default `0`.
> Floor for the autoscaler. Runners are kept warm at this count even when no jobs are queued. `0` means scale-to-zero is allowed. Negative values are rejected by the controller.

> **`maxRunners`** *`int`* — optional — default `unbounded` (the listener uses `MaxInt32`).
> Ceiling for the autoscaler. The listener will not request more than this many ephemeral runners from GitHub, even if the job queue is longer. `0` is treated as "no maximum"; do **not** set `maxRunners: 0` if you intend to disable the pool — use Helm uninstall instead. `maxRunners < minRunners` is a configuration error; the controller will surface this via events.

> **`runnerGroup`** *`string`* — optional — default `"default"`.
> GitHub runner group the scale set is assigned to. Must already exist on GitHub. Enterprise/org admins manage groups in the GitHub UI.

> **`runnerScaleSetName`** *`string`* — optional — default: the value of `.metadata.name`.
> Name shown in the GitHub UI under *Settings → Actions → Runner groups*. Must be unique within the GitHub configuration scope (repo/org/enterprise). Renaming this field after registration causes the listener to register a **new** scale set; the old one becomes orphaned on GitHub and must be cleaned up manually.

> **`runnerScaleSetLabels`** *`[]string`* — optional.
> Custom GitHub Actions labels applied to runners in this set, in addition to the implicit `self-hosted` and the scale set name. These are the labels users target with `runs-on:` in workflows.

#### Pod template

> **`template`** *`PodTemplateSpec`* — **required**.
> The full Kubernetes `PodTemplateSpec` used for every runner Pod. The `template.spec.containers[0]` must be named `runner`. ARC injects environment variables (`ACTIONS_RUNNER_INPUT_*`), mounts the runner work volume, and may add init containers depending on `containerMode` (see [06-container-modes.md](06-container-modes.md)).
>
> Notable Pod-level behaviors:
> - `restartPolicy` is forced to `Never` (ephemeral, single-job).
> - `serviceAccountName` defaults to a controller-managed SA when `containerMode=kubernetes`. Override only if you understand the RBAC required (see `kube_mode_*` templates).
> - `automountServiceAccountToken` may be forced based on container mode.
> - Any `containers[name=runner].image` change takes effect for **new** runner pods only.

#### Listener customization

> **`listenerTemplate`** *`PodTemplateSpec`* — optional.
> Override the Pod template of the listener pod. Use to add tolerations, node selectors, sidecars, or resource requests for the listener. The listener container must remain named `listener`; the controller injects required env vars and the config secret mount regardless of overrides.

> **`listenerMetrics`** *`MetricsConfig`* — optional.
> Override metric definitions. See [`MetricsConfig`](#metricsconfig) and [11-observability.md](11-observability.md).

#### Metadata propagation (advanced)

These fields let you decorate the resources the controller materializes:

| Field | Type | Applied to |
|-------|------|-----------|
| `listenerMetadata` | *(implicit via listenerTemplate.metadata)* | Listener Pod |
| `listenerConfigSecretMetadata` | `ResourceMeta` | The auto-generated listener config `Secret` |
| `listenerRoleMetadata` | `ResourceMeta` | The listener `Role` |
| `listenerRoleBindingMetadata` | `ResourceMeta` | The listener `RoleBinding` |
| `listenerServiceAccountMetadata` | `ResourceMeta` | The listener `ServiceAccount` |
| `ephemeralRunnerSetMetadata` | `ResourceMeta` | The owned `EphemeralRunnerSet` |
| `ephemeralRunnerMetadata` | `ResourceMeta` | Each `EphemeralRunner` |
| `ephemeralRunnerConfigSecretMetadata` | `ResourceMeta` | Per-runner config `Secret` (when ARC creates one) |

Use these for cost-allocation labels, network policy selectors, or audit annotations.

#### Networking & secrets

> **`proxy`** *`ProxyConfig`* — optional. Outbound proxy used by both listener and runners.

> **`githubServerTLS`** *`GitHubServerTLSConfig`* — optional. CA bundle for GHES with private PKI.

> **`vaultConfig`** *`VaultConfig`* — optional. Direct vault integration; see [`VaultConfig`](#vaultconfig).

### `.status`

`AutoscalingRunnerSetStatus`. All fields managed by the controller.

| Field | Type | Description |
|-------|------|-------------|
| `currentRunners` | `int` | Total count of `EphemeralRunner` resources currently owned by the active `EphemeralRunnerSet`. |
| `pendingEphemeralRunners` | `int` | Count in `Pending` phase (pod scheduled, not yet `Running`). |
| `runningEphemeralRunners` | `int` | Count in `Running` phase. |
| `failedEphemeralRunners` | `int` | Count in `Failed` phase. Non-zero is a signal to investigate. |
| `phase` | `string` | Free-form phase string. Reserved; usually empty when healthy. |

> The status is **eventually consistent**. Expect a few seconds of lag during scale events.

### Example

```yaml
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata:
  name: straw-hat-runners-linux
  namespace: arc-runners
spec:
  githubConfigUrl: https://github.com/BkCloudOps
  githubConfigSecret: arc-github-config  # ESO-synced secret
  runnerGroup: default
  runnerScaleSetName: straw-hat-runners-linux
  minRunners: 1
  maxRunners: 20
  template:
    spec:
      containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
```

### Lifecycle

- **Create**: Controller (a) registers the scale set with GitHub (gets a `runnerScaleSetId`), (b) creates a config `Secret`, `ServiceAccount`, `Role`, `RoleBinding` for the listener, (c) creates an `AutoscalingListener` CR in `arc-systems`, (d) creates an `EphemeralRunnerSet` with `replicas=minRunners` in the same namespace.
- **Update of `.spec`**: Most fields are reconciled in-place. Changing `template` rolls new pods on next scale event (existing runners finish their job first). Changing `githubConfigUrl` or `runnerScaleSetName` re-registers with GitHub (avoid in production).
- **Delete**: Finalizer `autoscalingrunnerset.actions.github.com/finalizer` blocks deletion until (a) the listener is drained, (b) all `EphemeralRunners` are terminated, (c) GitHub-side scale set is deleted via API. If GitHub is unreachable, the finalizer will stall — see [11-observability.md](11-observability.md).

---

## Kind: `AutoscalingListener`

> **Internal CRD.** Never create one by hand. It is the controller's way of saying "this scale set needs a listener pod running."

| Property | Value |
|----------|-------|
| Group/Version | `actions.github.com/v1alpha1` |
| Kind | `AutoscalingListener` |
| Created by | Controller (from an `AutoscalingRunnerSet`) |
| Lives in | `arc-systems` (the controller's namespace) |
| Owns | exactly one listener `Pod` |
| Status subresource | yes (currently empty schema) |

### `.spec`

All fields are **required** unless marked optional. The controller fills them in from the parent `AutoscalingRunnerSet`.

| Field | Type | Description |
|-------|------|-------------|
| `autoscalingRunnerSetName` | `string` | Name of the parent ARS. |
| `autoscalingRunnerSetNamespace` | `string` | Namespace of the parent ARS (e.g. `arc-runners`). The listener watches `EphemeralRunnerSets` in this namespace. |
| `ephemeralRunnerSetName` | `string` | Name of the `EphemeralRunnerSet` the listener will patch (`.spec.replicas`) to drive scaling. |
| `runnerScaleSetId` | `int` | The integer ID GitHub assigned when the scale set was registered. The listener uses this in API calls. |
| `githubConfigUrl` | `string` | Same semantics as on the ARS. |
| `githubConfigSecret` | `string` | Name of the auth secret (in `arc-systems` — the controller copies it). |
| `minRunners` | `int` | Mirrored from ARS. |
| `maxRunners` | `int` | Mirrored from ARS. |
| `image` | `string` | Container image for the listener pod. Defaults from the controller chart's `values.yaml`. |
| `imagePullSecrets` | `[]LocalObjectReference` | Pull secrets for `image`. |
| `template` | `PodTemplateSpec` | Pod template for the listener pod. Comes from the ARS's `listenerTemplate` if set; otherwise minimal default. |
| `proxy` | `ProxyConfig` | Mirrored from ARS. |
| `githubServerTLS` | `GitHubServerTLSConfig` | Mirrored from ARS. |
| `vaultConfig` | `VaultConfig` | Mirrored from ARS. |
| `metrics` | `MetricsConfig` | Mirrored from ARS. |
| `configSecretMetadata` | `ResourceMeta` | Annotations/labels on the listener's config secret. |
| `serviceAccountMetadata` | `ResourceMeta` | On the listener's SA. |
| `roleMetadata` | `ResourceMeta` | On the listener's Role. |
| `roleBindingMetadata` | `ResourceMeta` | On the listener's RoleBinding. |

### `.status`

`AutoscalingListenerStatus` — schema currently has no defined fields (`type: object` only). Listener health is observed via the **Pod**'s status and via metrics, not via this CR.

### Behavior

The listener pod runs a long-poll loop against the GitHub Actions service:
1. Acquires a session for `runnerScaleSetId`.
2. Calls `AcquireJob` / receives `JobAvailable` messages.
3. Computes desired replicas = `min(maxRunners, max(minRunners, pendingJobs))`.
4. `PATCH`es the `EphemeralRunnerSet.spec.replicas` accordingly, bumping `patchID` each time.
5. Reports metrics on `/metrics` (port from `listenerMetrics`).

When the listener Pod is deleted, the controller recreates it (the `AutoscalingListener` CR is the source of truth). If the `AutoscalingListener` CR itself is deleted, the controller recreates it from the parent ARS.

---

## Kind: `EphemeralRunnerSet`

> **Internal CRD.** Analogous to a Kubernetes `ReplicaSet`, but for ARC runners.

| Property | Value |
|----------|-------|
| Group/Version | `actions.github.com/v1alpha1` |
| Kind | `EphemeralRunnerSet` |
| Created by | Controller (from an `AutoscalingRunnerSet`) |
| Lives in | same namespace as the parent ARS |
| Owns | N `EphemeralRunner` resources |
| Driven by | the listener (which patches `.spec.replicas` and `.spec.patchID`) |
| Status subresource | yes |

### `.spec`

> **`replicas`** *`int`* — optional — default `0`.
> Desired number of `EphemeralRunner` children. Patched by the **listener**, not by humans. Manually overriding works but the listener will overwrite on its next reconcile loop.

> **`patchID`** *`int`* — **required**.
> Monotonically increasing counter set by the listener with each patch. The controller uses this to detect the freshest desired state and to make scale decisions idempotent. Manual edits without bumping `patchID` may be ignored.

> **`ephemeralRunnerSpec`** *`EphemeralRunnerSpec`* — **required**.
> Embedded spec template used to materialize each child `EphemeralRunner`. Identical schema to [`EphemeralRunner.spec`](#kind-ephemeralrunner) — see that section.

> **`ephemeralRunnerMetadata`** *`ResourceMeta`* — optional.
> Annotations/labels applied to each child `EphemeralRunner`.

### `.status`

| Field | Type | Description |
|-------|------|-------------|
| `currentReplicas` | `int` (**required**) | Total `EphemeralRunner` children currently owned. |
| `pendingEphemeralRunners` | `int` | Children in `Pending` phase. |
| `runningEphemeralRunners` | `int` | Children in `Running` phase. |
| `failedEphemeralRunners` | `int` | Children in `Failed` phase. |
| `phase` | `string` | Phase of the set itself. See [phase state machines](#phase-state-machines). |

### Lifecycle nuances

- Scale-**up**: controller creates new `EphemeralRunner` CRs to reach `replicas`. Each one in turn creates a Pod.
- Scale-**down** is **graceful**, not arbitrary: the controller does **not** delete a running runner that is currently executing a job. It instead delegates to GitHub: it calls the API to mark idle runners for removal, then deletes their CRs. This is why the actual pod count can lag `.spec.replicas` by minutes during a downscale.
- One `EphemeralRunner` runs **exactly one job** and then exits. The controller observes the Pod exit and replaces it with a fresh one if `replicas` still requires it.

---

## Kind: `EphemeralRunner`

> **Internal CRD.** One `EphemeralRunner` ↔ one runner `Pod` ↔ one CI job.

| Property | Value |
|----------|-------|
| Group/Version | `actions.github.com/v1alpha1` |
| Kind | `EphemeralRunner` |
| Created by | Controller (from an `EphemeralRunnerSet`) |
| Owns | exactly one `Pod`, optionally a per-runner config `Secret` |
| Status subresource | yes |

### `.spec`

Required fields:
- `githubConfigUrl` *`string`*
- `githubConfigSecret` *`string`* — name of the auth secret
- `runnerScaleSetId` *`int`* — the GitHub-assigned ID; identifies which scale set this runner belongs to

Pod template:
- `spec` *`PodSpec`* — **required**, embedded directly (not a `PodTemplateSpec`). The `containers[name=runner]` must exist; the controller injects:
  - env: `ACTIONS_RUNNER_INPUT_JITCONFIG`, `RUNNER_WORKSPACE`, `GITHUB_ACTIONS_RUNNER_EXTRA_USER_AGENT`, etc.
  - volumes: `work` (emptyDir or PVC, depending on container mode), `runner-config` (projected from `githubConfigSecret`).
- `metadata` *`ResourceMeta`-shaped* — annotations/labels for the Pod.

Optional:
- `proxy` *`ProxyConfig`*
- `proxySecretRef` *`string`* — name of a Secret holding proxy credentials (alternative to inlining via `proxy.http.credentialSecretRef`).
- `githubServerTLS` *`GitHubServerTLSConfig`*
- `vaultConfig` *`VaultConfig`*
- `ephemeralRunnerConfigSecretMetadata` *`ResourceMeta`* — annotations/labels for the per-runner Secret the controller may create.

### `.status`

| Field | Type | Description |
|-------|------|-------------|
| `runnerId` | `int` | GitHub's internal runner ID (set after registration). |
| `runnerName` | `string` | Name shown in the GitHub UI. |
| `phase` | `string` | One of: `Pending`, `Running`, `Succeeded`, `Failed`. Underlying type is Kubernetes `PodPhase` but with restricted semantics — see below. |
| `ready` | `bool` | `true` only when the runner has registered with GitHub and reported `online`. **This is not the same as Pod `Ready`.** |
| `reason` | `string` | Short machine-readable reason for the current phase. |
| `message` | `string` | Human-readable detail. |
| `jobId` | `int64` | GitHub job ID, set once a job is dispatched. |
| `jobRequestId` | `int64` | Job request ID. |
| `jobDisplayName` | `string` | Pretty job name (matches workflow YAML). |
| `jobRepositoryName` | `string` | `owner/repo` the job belongs to. |
| `jobWorkflowRef` | `string` | Ref like `owner/repo/.github/workflows/ci.yml@refs/heads/main`. |
| `workflowRunId` | `int64` | GitHub workflow run ID; lets you correlate a Pod with a GitHub run URL. |
| `failures` | `map[string]time.Time` | Map of failure-reason → timestamp. Used for retry/backoff bookkeeping. |

#### Phase semantics (important)

The controller deliberately **narrows** the meaning of `PodPhase`:

| Phase | When set |
|-------|----------|
| `Pending` | Pod created, container not yet ready / not yet registered with GitHub. |
| `Running` | Pod running, runner registered, possibly executing a job. |
| `Succeeded` | Set **only after** the controller has confirmed the runner executed its job and was removed from GitHub. A Pod that exits successfully but isn't deregistered does **not** reach `Succeeded`. |
| `Failed` | Set **only after** multiple registration/start retries have failed. Indicates manual inspection is required (often: bad token, bad image, network egress blocked). |

> Treat `Failed` as actionable: it is **not** "the job failed." Job failures are reported on GitHub; ARC's `Failed` means the runner itself never became usable.

---

## Ownership graph and finalizers

```
AutoscalingRunnerSet (arc-runners)
        │ owns (via OwnerReference, blockOwnerDeletion=true)
        ├──► AutoscalingListener (arc-systems)
        │           │ owns
        │           └──► Listener Pod
        │
        └──► EphemeralRunnerSet (arc-runners)
                    │ owns
                    └──► EphemeralRunner × N (arc-runners)
                                │ owns
                                └──► Runner Pod
```

Finalizers in play:

| Resource | Finalizer | Removed when |
|----------|-----------|--------------|
| `AutoscalingRunnerSet` | `autoscalingrunnerset.actions.github.com/finalizer` | Listener drained, ERS gone, GitHub scale set deleted |
| `EphemeralRunnerSet` | `ephemeralrunnerset.actions.github.com/finalizer` | All child runners removed cleanly |
| `EphemeralRunner` | `ephemeralrunner.actions.github.com/finalizer` | Runner deregistered from GitHub and Pod terminated |

If GitHub is unreachable during deletion, finalizers will not clear. The escape hatch is `kubectl patch ... -p '{"metadata":{"finalizers":[]}}' --type=merge` — but this leaves orphan registrations on GitHub that you must clean up via the API.

---

## Phase state machines

### `EphemeralRunner.status.phase`

```
        +-----------+
   ---> | (created) |
        +-----+-----+
              |
              v
        +-----------+   register OK    +-----------+
        |  Pending  | ---------------> |  Running  |
        +-----+-----+                  +-----+-----+
              |                               |
              | retries exhausted             | job done + deregistered
              v                               v
        +-----------+                   +-----------+
        |  Failed   |                   | Succeeded |
        +-----------+                   +-----------+
```

### `EphemeralRunnerSet.status.phase`

Free-form; typical values include empty string (steady), and transient strings during scale events. Treat the numeric `*EphemeralRunners` counts as the authoritative health signal.

---

## Validation rules summary

These are enforced by the CRD's OpenAPI schema (returned as 4xx by the API server) or by the controller at reconcile time:

| Rule | Enforced where |
|------|----------------|
| `AutoscalingRunnerSet.spec.githubConfigUrl` non-empty | OpenAPI (required) |
| `AutoscalingRunnerSet.spec.githubConfigSecret` non-empty | OpenAPI (required) |
| `AutoscalingRunnerSet.spec.template` non-empty (must contain a `runner` container) | OpenAPI (required) + controller |
| `AutoscalingRunnerSet.spec.minRunners ≥ 0` | controller (event on violation) |
| `AutoscalingRunnerSet.spec.maxRunners ≥ minRunners` (when both set and `maxRunners > 0`) | controller |
| `AutoscalingListener.spec.runnerScaleSetId` non-empty | OpenAPI (required) |
| `EphemeralRunnerSet.spec.patchID` non-empty | OpenAPI (required) |
| `EphemeralRunner.spec.githubConfigUrl/Secret/runnerScaleSetId` non-empty | OpenAPI (required) |
| `vaultConfig.azureKeyVault.*` all four fields set when `vaultConfig.type=azure_key_vault` | OpenAPI |

ARC does **not** ship a `ValidatingAdmissionWebhook`; everything beyond OpenAPI is enforced at reconcile time and surfaced via Events on the offending object.

---

## Common operational queries

```bash
# What exists in this cluster?
kubectl get autoscalingrunnerset,autoscalinglistener,ephemeralrunnerset,ephemeralrunner -A

# Why is my runner pool stuck?
kubectl describe autoscalingrunnerset -n arc-runners straw-hat-runners-linux
kubectl logs -n arc-systems deploy/arc-gha-rs-controller

# What jobs are individual runners executing right now?
kubectl get ephemeralrunner -n arc-runners \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,JOB:.status.jobDisplayName,REPO:.status.jobRepositoryName

# Inspect the listener's view of scale
kubectl get ephemeralrunnerset -n arc-runners \
  -o custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,CURRENT:.status.currentReplicas,RUNNING:.status.runningEphemeralRunners

# Field documentation directly from the cluster
kubectl explain autoscalingrunnerset.spec --recursive | less
kubectl explain ephemeralrunner.status --recursive | less

# Force-clear a stuck finalizer (last resort — leaves GitHub-side orphans)
kubectl patch autoscalingrunnerset -n arc-runners <name> \
  --type=merge -p '{"metadata":{"finalizers":[]}}'
```

---

## See also

- [02-architecture.md](02-architecture.md) — components and control vs data plane
- [03-end-to-end-flow.md](03-end-to-end-flow.md) — step-by-step trace from `git push` to job completion
- [07-authentication.md](07-authentication.md) — what goes into `githubConfigSecret`
- [08-scaling-behavior.md](08-scaling-behavior.md) — how `minRunners`/`maxRunners` interact with the listener's algorithm
- [11-observability.md](11-observability.md) — metrics, logs, and troubleshooting
