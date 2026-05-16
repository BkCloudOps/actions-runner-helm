# 4. `gha-runner-scale-set-controller` — Every Value Explained

File: [charts/arc-runner-stack/charts/gha-runner-scale-set-controller/values.yaml](../charts/arc-runner-stack/charts/gha-runner-scale-set-controller/values.yaml)

> **Convention** — Each section shows: default → what it does → when to change → example.

---

## `labels: {}`
- **Default:** empty.
- **What:** Extra labels stamped on **every** resource the chart creates (Deployment, SA, Roles).
- **When:** You want to identify ARC resources for cost-allocation, monitoring, or NetworkPolicy selectors.
- **Example:**
  ```yaml
  labels:
    team: platform
    cost-center: "1234"
  ```

## `replicaCount: 1`
- **What:** Number of controller-manager Pods.
- **When:** `> 1` for high availability. Leader-election ensures only one is active at a time; others are warm standbys.
- **Architect note:** Setting to 2+ does **not** parallelize reconciliation — it only reduces recovery time if the active pod dies. Combine with PodDisruptionBudget.
- **Example:** `replicaCount: 2`

## `image:`
```yaml
image:
  repository: "ghcr.io/actions/gha-runner-scale-set-controller"
  pullPolicy: IfNotPresent
  tag: ""           # defaults to Chart.appVersion
```
- **When to change `repository`:** Air-gapped clusters that mirror images to a private registry.
- **When to change `tag`:** Pinning to a canary build (`canary-<sha>`) for testing.

## `imagePullSecrets: []`
- **What:** List of Secret names that hold registry credentials.
- **When:** Pulling from a private mirror.
- **Example:** `imagePullSecrets: [{ name: my-registry-cred }]`

## `nameOverride` / `fullnameOverride`
- Standard Helm hooks to override the generated resource names. Use `fullnameOverride` if you hit the **63-character** Kubernetes name limit.

## `env: []`
- **What:** Extra env vars injected into the controller-manager container.
- **When:** Custom logging endpoints, feature flags, or HTTP_PROXY for the controller itself.
- **Example:**
  ```yaml
  env:
    - name: HTTP_PROXY
      value: "http://proxy.corp:3128"
    - name: GITHUB_API_HOST
      valueFrom:
        secretKeyRef:
          name: ghe-config
          key: host
  ```

## `serviceAccount:`
```yaml
serviceAccount:
  create: true
  annotations: {}
  name: ""
```
- **When `annotations` is critical:** Workload Identity (GKE/EKS/AKS). Example for AKS:
  ```yaml
  serviceAccount:
    annotations:
      azure.workload.identity/client-id: "<managed-identity-client-id>"
  ```
- **Why you can't use `default` SA:** The controller needs CRD reconcile permissions; `default` has none.

## `podAnnotations`, `podLabels`
- Applied only to the controller Pod (not other resources).
- Common use: Prometheus scrape annotations.
  ```yaml
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
  ```

## `podSecurityContext` / `securityContext`
- **Pod-level vs container-level** security context.
- **Recommended hardening:**
  ```yaml
  podSecurityContext:
    runAsNonRoot: true
    fsGroup: 2000
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop: ["ALL"]
  ```

## `resources: {}`
- **Recommended baseline (production):**
  ```yaml
  resources:
    requests: { cpu: 100m, memory: 256Mi }
    limits:   { cpu: 500m, memory: 512Mi }
  ```
- The controller is mostly idle; CPU only spikes during reconcile bursts.

## `nodeSelector`, `tolerations`, `affinity`, `topologySpreadConstraints`
- Standard Kubernetes scheduling controls. Use to keep the controller on a dedicated **system node pool**, separate from runner Pods.
- **Example — pin to system pool:**
  ```yaml
  nodeSelector:
    workload-type: system
  tolerations:
    - key: dedicated
      value: system
      effect: NoSchedule
  ```

## `volumes` / `volumeMounts`
- Mount extra volumes (e.g., a CA bundle ConfigMap) into the controller container.

## `priorityClassName: ""`
- Use `system-cluster-critical` so the controller is one of the last things evicted under node pressure.

## `metrics:` (commented out by default)
```yaml
metrics:
  controllerManagerAddr: ":8080"      # Controller's own metrics
  listenerAddr: ":8080"               # Listener Pod metrics port
  listenerEndpoint: "/metrics"        # Listener Pod metrics path
```
- **Important:** If this block is **missing or commented**, all three flags are passed empty → metrics are **disabled**. Uncomment to enable.
- **What you get:** Prometheus metrics like `gha_assigned_jobs`, `gha_running_jobs`, `gha_job_startup_duration_seconds`. Full list in [11-observability.md](11-observability.md).

## `flags:`

### `flags.logLevel: "debug"`
- Values: `debug | info | warn | error`. Default `debug` is **noisy** — drop to `info` in production.

### `flags.logFormat: "text"`
- `text | json`. Use `json` if you ship logs to Elasticsearch / Loki / Datadog.

### `flags.watchSingleNamespace: ""` (commented)
- **Empty (default):** Controller watches **all** namespaces → needs ClusterRole.
- **Set to a namespace:** Controller is restricted → uses Role only → smaller blast radius. Required in multi-tenant clusters.
- **Trade-off:** You need one controller install per watched namespace.

### `flags.runnerMaxConcurrentReconciles: 2`
- How many `EphemeralRunner` CRs the controller will reconcile in parallel.
- Increase (e.g., 10) for large pools (100+ runners) — but watch API-server load and GitHub rate limits.

### `flags.updateStrategy: "immediate"`
| Value | Behavior |
|-------|----------|
| `immediate` *(default)* | On Helm upgrade, recreate listener + runners now. May briefly **over-provision** runners. |
| `eventual` | Wait for in-flight jobs to finish before recreating. Slower upgrade, no over-provisioning. |
- **Pick `eventual`** if jobs are long-running and resource-expensive.

### `flags.excludeLabelPropagationPrefixes` (commented)
- ARC normally copies labels from `AutoscalingRunnerSet` → child resources (Pods, Secrets). GitOps tools (ArgoCD, Flux) add their own labels that should NOT propagate, otherwise you get reconciliation loops.
- **Example:** `excludeLabelPropagationPrefixes: ["argocd.argoproj.io/instance"]`

## `namespaceOverride: ""`
- Override `.Release.Namespace` for resources. Rarely needed.

## `k8sClientRateLimiterQPS / Burst` (commented at bottom)
- Tune the controller's K8s client when reconciling huge numbers of resources. Defaults are fine for <100 runners.

---

## Production preset (copy-paste)

```yaml
replicaCount: 2
flags:
  logLevel: info
  logFormat: json
  updateStrategy: eventual
metrics:
  controllerManagerAddr: ":8080"
  listenerAddr: ":8080"
  listenerEndpoint: "/metrics"
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
priorityClassName: system-cluster-critical
nodeSelector:
  workload-type: system
```

Continue to [05-runner-scale-set-values.md](05-runner-scale-set-values.md).
