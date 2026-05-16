# 11. Observability — Logs, Metrics, Troubleshooting

This chapter is the operator's handbook: what to look at, in what order, and what each signal means. It complements the protocol-level failure modes catalogued in [14-listener-protocol-and-jit.md §8](14-listener-protocol-and-jit.md#8-failure-modes-and-recovery) and the scaling alerts proposed in [08-scaling-behavior.md §8.7](08-scaling-behavior.md#87-observability-for-scaling).

## 11.1 Logs

### Controller manager
```bash
kubectl logs -n arc-systems -l app.kubernetes.io/component=controller-manager -f
```
Look for: `Reconciling AutoscalingRunnerSet`, `Creating EphemeralRunner`, `Failed to get JIT token`.

### Listener
```bash
kubectl logs -n arc-systems -l app.kubernetes.io/component=runner-scale-set-listener -f
```
Look for: `Job available`, `Acquired N jobs`, `Patched ephemeralrunnerset`.

### Runner pod (during a job)
```bash
kubectl logs -n arc-runners <runner-pod> -c runner -f
```
Watch the job execute in real time.

## 11.2 Metrics (Prometheus)

Enable in controller `values.yaml`:
```yaml
metrics:
  controllerManagerAddr: ":8080"
  listenerAddr: ":8080"
  listenerEndpoint: "/metrics"
```

Then add Prometheus scrape (PodMonitor):
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: arc
  namespace: arc-systems
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: gha-runner-scale-set-controller
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
```

### Controller metrics
| Metric | Type | Meaning |
|--------|------|---------|
| `gha_controller_pending_ephemeral_runners` | gauge | Runners waiting to start |
| `gha_controller_running_ephemeral_runners` | gauge | Active runners |
| `gha_controller_failed_ephemeral_runners` | gauge | Runners that errored |
| `gha_controller_running_listeners` | gauge | Active Listener Pods |

### Listener metrics (per scale set)
| Metric | Type | Use |
|--------|------|-----|
| `gha_assigned_jobs` | gauge | Demand signal |
| `gha_running_jobs` | gauge | In-progress count |
| `gha_registered_runners` | gauge | Total registered |
| `gha_busy_runners` | gauge | Executing a job |
| `gha_idle_runners` | gauge | Warm pool |
| `gha_min_runners` / `gha_max_runners` | gauge | Configured bounds |
| `gha_desired_runners` | gauge | Scale target |
| `gha_started_jobs_total` | counter | Throughput |
| `gha_completed_jobs_total` | counter | Throughput (with `job_result` label) |
| `gha_job_startup_duration_seconds` | histogram | **Latency SLO** |
| `gha_job_execution_duration_seconds` | histogram | Workload duration |

> Counters reset on Listener restart.

### Suggested alerts
```yaml
- alert: ARCJobStartupSlow
  expr: histogram_quantile(0.95, rate(gha_job_startup_duration_seconds_bucket[5m])) > 60
  for: 10m
  annotations:
    summary: "p95 job startup > 60s — consider raising minRunners"

- alert: ARCAtMaxCapacity
  expr: gha_desired_runners >= gha_max_runners
  for: 5m
  annotations:
    summary: "Scale set at maxRunners — jobs are queued"

- alert: ARCListenerDown
  expr: gha_controller_running_listeners == 0
  for: 2m
  annotations:
    summary: "No ARC listeners running — no jobs will be dispatched"
```

## 11.3 Troubleshooting matrix

| Symptom | Where to look | Likely cause |
|---------|---------------|--------------|
| Workflow stays "queued" forever | Listener logs | Wrong `runs-on` label; listener can't authenticate |
| Listener `CrashLoopBackOff` | Listener logs | Secret missing / bad credentials |
| Pods stuck `Pending` | `kubectl describe pod` | Node capacity; missing PVC; PodSecurity policy |
| `dind` Pod crashes immediately | dind container logs | Privileged not allowed (PodSecurityStandards) |
| Jobs without `container:` fail in K8s mode | Runner logs | Set `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=false` (review security first) |
| Runner registers then disappears | GitHub side | JIT token expired; cluster too slow to start the Pod |
| ExternalSecret status `SecretSyncedError` | ESO logs, AAD logs | Workload Identity not bound; missing Key Vault `get` permission |
| Massive runner thrash on Helm upgrade | controller logs | `updateStrategy: immediate` — consider `eventual` |

## 11.4 Useful kubectl one-liners

```bash
# What scale sets exist?
kubectl get autoscalingrunnersets -A -o wide

# How many runners are currently desired vs running?
kubectl get ephemeralrunnersets -A

# Inspect a specific ephemeral runner
kubectl get ephemeralrunner -n arc-runners <name> -o yaml

# Check Listener status
kubectl get autoscalinglisteners -A

# Restart the listener (forces reauth)
kubectl delete pod -n arc-systems -l app.kubernetes.io/component=runner-scale-set-listener
```

Continue to [12-security-model.md](12-security-model.md).
