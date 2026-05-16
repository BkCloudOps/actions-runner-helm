# 8. Scaling Behavior

This chapter explains the autoscaling algorithm precisely — the formula, the inputs, the timing, the failure modes — and gives you the knobs to tune it for cold start, burst, and steady-state regimes.

For the wire-level mechanism (how `JobAvailable` flows from GitHub to a `replicas` patch), see [14-listener-protocol-and-jit.md](14-listener-protocol-and-jit.md). For where the values are set, see [13-crd-reference.md §AutoscalingRunnerSet](13-crd-reference.md#kind-autoscalingrunnerset).

---

## 8.1 The scaling algorithm

The **listener** owns scale decisions; it patches `EphemeralRunnerSet.spec.replicas`. On every poll cycle it recomputes:

```
desired = clamp(assignedJobs, minRunners, maxRunners)

where:
  assignedJobs  = jobs the listener has acquired and not yet observed completed
  minRunners    = AutoscalingRunnerSet.spec.minRunners   (default 0)
  maxRunners    = AutoscalingRunnerSet.spec.maxRunners   (default MaxInt32 = "unbounded")
  clamp(x,a,b)  = max(a, min(b, x))
```

`assignedJobs` is incremented on a successful `acquirejobs` response and decremented on `JobCompleted`. The listener does **not** poll the Kubernetes side for actual runner counts when computing `desired`; it trusts its in-memory ledger. The reconciler on the other side is responsible for converging actual Pods toward `desired`.

> **No HPA, no metrics-server, no cooldown.** ARC autoscaling is event-driven, not metrics-driven. There is no `cooldownPeriod`, no `stabilizationWindow`, no `targetUtilization`. Scale-up is immediate on `JobAvailable`; scale-down is immediate on `JobCompleted`, modulo the graceful drain rule (§8.5).

---

## 8.2 Configuration matrix

Behavior changes meaningfully with these two knobs. All combinations are valid except the noted error case.

| `minRunners` | `maxRunners` | Effective behavior | Use case |
|--------------|--------------|--------------------|----------|
| unset (`0`) | unset (`∞`) | Pure on-demand, unbounded ceiling | Lab clusters, no budget guardrail |
| `0` | `N` | Scale 0 → N as needed; cold start every time | Burst-friendly, cost-conscious |
| `K` | unset | Always at least `K` warm; unbounded ceiling | Small CI flow with low latency requirement |
| `K` | `N` (`K < N`) | `K` warm idle, scales up to `N` busy | **Recommended default.** |
| `K` | `N` (`K == N`) | Fixed-size pool. Behaves like a `StatefulSet` of runners but still ephemeral | Predictable cost; latency-critical |
| `0` | `0` | **Drain mode.** Listener accepts no new jobs (acquires nothing). Existing runners finish. | Planned maintenance |
| `K > N` (and `N > 0`) | — | **Invalid.** Controller logs an event `Warning ValidationFailed minRunners must be <= maxRunners`. The CR remains, but the listener will not start. | Misconfiguration |

> `maxRunners: 0` is a special drain-mode signal **only when** `minRunners` is also `0`. Setting only `maxRunners: 0` with `minRunners > 0` is the invalid `K > N` case.

---

## 8.3 Cold start vs warm pool — the latency trade-off

Time-to-execute (TTE) is the wall-clock from `git push` to the first workflow step starting. Dominant factors:

| Phase | Cold (`minRunners=0`) | Warm slot available (`minRunners ≥ 1`) |
|-------|----------------------|-----------------------------------------|
| GitHub queue → `JobAvailable` to listener | ~500 ms | ~500 ms |
| Listener acquire + patch | ~200 ms | ~200 ms |
| Reconciler creates Pod | ~200 ms | (skipped — runner already idle) |
| Pod scheduling (node already exists) | 1–3 s | (skipped) |
| Pod scheduling (cluster-autoscaler must provision node) | **120–300 s** | (skipped) |
| Image pull (cached on node) | 1–3 s | (skipped) |
| Image pull (not cached) | **5–60 s** | (skipped) |
| Runner registration with GitHub | 1–3 s | (skipped) |
| Job dispatch to runner | < 1 s | < 1 s |
| **Total (typical, cached, node ready)** | **8–15 s** | **2–4 s** |
| **Total (node provisioning)** | **2–6 min** | (n/a) |

Practical guidance:

- For developer-experience workflows (pre-commit, small unit tests), **set `minRunners` ≥ peak-instantaneous-developer-count / 2** to keep TTE under 5 s.
- For nightly-batch or release pipelines, `minRunners: 0` is fine; the few extra seconds are invisible.
- For TTE-critical pipelines run inside CI/CD-driven business processes (e.g. on-merge production deploys), pair `minRunners ≥ 1` with **runtime-cached image pulls** (image preloaded onto the runner node).

---

## 8.4 Burst scaling

Suppose 50 jobs arrive within a 5-second window (e.g. `git push` of a chain of commits, or a matrix workflow with 50 entries).

1. Listener receives a `RunnerScaleSetJobMessages` batch with up to ~25 inner messages per poll cycle.
2. Listener `acquirejobs` for each batch; updates `assignedJobs`.
3. Listener patches `ERS.spec.replicas` once per poll cycle with the new total — **not** once per job. This is critical: it means the controller sees a single `replicas: 50` patch, not 50 separate patches.
4. `EphemeralRunnerSetReconciler` enqueues 50 `EphemeralRunner` creations. Concurrency is bounded by `controller.runnerMaxConcurrentReconciles` (default `1`).
5. Each `EphemeralRunner` reconcile performs: `generatejitconfig` REST call + Pod create. Wall-clock per reconcile: ~200–500 ms.

With the default `runnerMaxConcurrentReconciles=1`, 50 runners take ~25 s of *just-the-control-plane* work before scheduling begins. Raising to `5` cuts that to ~5 s.

| Knob | Default | Effect of raising | Where to set |
|------|---------|-------------------|---------------|
| `controller.runnerMaxConcurrentReconciles` | `1` | Faster burst creation; more concurrent GitHub API calls (subject to rate limits) | Controller chart values |
| Cluster autoscaler `--scale-up-unneeded-time` | varies | Faster node provisioning at the cost of cluster cost stability | Cluster-autoscaler args |
| Karpenter `provisioner.spec.consolidation` | enabled | Disable for hot pipelines so nodes don't churn | Karpenter NodePool |

### 8.4.1 Pre-warming nodes

If TTE for the 50th job matters, *the node side dominates*, not ARC. Two patterns:

| Pattern | Mechanism |
|---------|-----------|
| **Cluster-autoscaler headroom** | Pin a low-priority placeholder Deployment requesting a node's worth of resources, with `priorityClass` lower than the runner Pods. When real runners arrive, they evict the placeholder, which then re-queues and forces another node. |
| **Karpenter pre-emption** | Mark a NodePool with `disruption.consolidationPolicy: WhenEmpty` and set a baseline replica count via a "warming" Deployment of pause Pods. |

Either way, your **`AutoscalingRunnerSet.spec.template.spec.priorityClassName`** must be higher than the headroom pods. Without this, scaling is no faster than vanilla cluster-autoscaler.

---

## 8.5 Scale-down (graceful)

The reconciler's scale-down logic is **not** a simple delete loop:

```
if   currentReplicas > desired   then
    for each runner ordered by (idle first, then oldest):
        if runner is idle:
            call GitHub: mark for removal
            wait for GitHub: confirmation
            delete EphemeralRunner CR  → Pod cascade-deletes
        else:
            skip (let it finish its job)
    repeat on next reconcile
```

Consequences:

- **A scale-down does not cancel an in-flight job.** Even if `replicas` drops to 0 mid-job, the busy runner stays until completion.
- The actual Pod count therefore *lags* `.spec.replicas` during downscales — sometimes by minutes.
- There is no eviction grace period to tune — graceful is the only mode.

If you need *forceful* termination (e.g. to evacuate a node for upgrade), delete the runner Pod directly via `kubectl`. The job will fail with "runner lost contact"; GitHub will report it as failed; the workflow will retry per its own policy.

---

## 8.6 Failure modes during scaling

| Event | Behavior | Visible in |
|-------|----------|-----------|
| `generatejitconfig` returns `429 Too Many Requests` | Reconciler honors `Retry-After`, retries with backoff; event on `EphemeralRunner` | `kubectl describe ephemeralrunner` |
| Pod stuck `Pending` (insufficient nodes) | After cluster-autoscaler timeout, Pod stays Pending; ARC does not retry create — it waits for scheduler | Pod events |
| Pod `ImagePullBackOff` | Reconciler does **not** delete-and-recreate; you must fix the image or pull secret | Pod events |
| Pod starts but runner fails to register within 60 s | Pod exits non-zero → `EphemeralRunner.status.phase=Failed` after 5 retry creations | `kubectl get ephemeralrunner` |
| Listener loses GitHub connection | Session re-established on next poll; `assignedJobs` re-derived from `lastMessageId` replay; no jobs lost | Listener logs |
| Listener Pod deleted | Controller recreates it; new session; queued jobs are re-offered | Listener uptime metric |
| Controller Pod deleted | Leader election re-elects; reconciles resume; no scale events lost (queue persisted in API) | Controller uptime |

> **Crucial**: `EphemeralRunner.status.phase = Failed` is *not* "the job failed". It means the runner could not even start. A failed workflow job leaves the runner in `Succeeded`.

---

## 8.7 Observability for scaling

Metrics exposed by the listener on its `:8080/metrics` endpoint (when `listenerMetrics` is enabled — default in 0.14.x):

| Metric | Type | Meaning | Alert when |
|--------|------|---------|------------|
| `gha_assigned_jobs` | Gauge | Jobs currently acquired and not completed | Trending up unbounded → maxRunners too low |
| `gha_running_jobs` | Gauge | Jobs in execution | — |
| `gha_desired_runners` | Gauge | Listener's `desired` value | `== maxRunners` sustained for > 5 min |
| `gha_idle_runners` | Gauge | Registered runners with no job | Higher than `minRunners` → scale-down lag |
| `gha_busy_runners` | Gauge | Registered runners executing a job | — |
| `gha_registered_runners` | Gauge | All runners known to GitHub | Should equal `idle + busy` |
| `gha_job_startup_duration_seconds` | Histogram | TTE | p95 > 60 s → raise `minRunners` |
| `gha_job_execution_duration_seconds` | Histogram | Per-job wall-clock | Sudden tail-latency growth → noisy neighbors |
| `gha_acquired_jobs_total` | Counter | Lifetime acquisitions | — |
| `gha_completed_jobs_total` | Counter | Lifetime completions | — |
| `gha_message_queue_errors_total` | Counter | Long-poll errors | Rate > 0.1/min → network/auth issue |

Sample PromQL alerts:

```promql
# Saturation
max by (autoscaling_runner_set) (gha_desired_runners)
  == on(autoscaling_runner_set) max by (autoscaling_runner_set) (gha_max_runners)
  for 5m

# Slow cold start
histogram_quantile(0.95,
  rate(gha_job_startup_duration_seconds_bucket[5m])) > 60

# Stalled listener
rate(gha_acquired_jobs_total[10m]) == 0
  and rate(gha_assigned_jobs[10m]) > 0
```

See [11-observability.md](11-observability.md) for log-based diagnosis.

---

## 8.8 Worked examples

### Example A — Steady CI for a small team

```yaml
minRunners: 2          # one warm slot per pair of developers
maxRunners: 10
```

Outcome: most jobs start in < 5 s; bursts of up to 10 succeed; ≥ 11 concurrent jobs queue briefly.

### Example B — Nightly batch

```yaml
minRunners: 0
maxRunners: 50
```

Outcome: zero cost when idle; nightly burst scales to up to 50 in ~3 min (node-provisioning dominated). Acceptable for a 4-hour pipeline.

### Example C — Hard cost ceiling

```yaml
minRunners: 0
maxRunners: 5
```

Outcome: no more than 5 concurrent jobs regardless of queue depth. Jobs 6+ wait in GitHub's queue (status: *queued*). Acceptable for a workflow that runs at most a few times/hour.

### Example D — Maintenance drain

```yaml
minRunners: 0
maxRunners: 0
```

Apply via `helm upgrade --set runnerScaleSet.maxRunners=0 --set runnerScaleSet.minRunners=0`. Listener stops acquiring; in-flight jobs finish. Reverse the change after maintenance.

### Example E — Two-pool sharding

Two `AutoscalingRunnerSet` releases pointing at the same `githubConfigUrl` but with different `runnerScaleSetLabels` and different `template.spec.tolerations`:

| Pool | Labels | Node selector | Workload |
|------|--------|---------------|----------|
| `linux-small` | `[self-hosted, small]` | `node.kubernetes.io/instance-type: t3.medium` | Lint, unit tests |
| `linux-gpu` | `[self-hosted, gpu]` | `nvidia.com/gpu: present` | Model training |

Workflows opt-in via `runs-on:`. Each pool scales independently.

---

## 8.9 Cross-references

- [14-listener-protocol-and-jit.md §4.1](14-listener-protocol-and-jit.md#41-acquiring-jobs) — what `acquirejobs` actually does on the wire
- [13-crd-reference.md](13-crd-reference.md) — every field of `AutoscalingRunnerSet`
- [11-observability.md](11-observability.md) — how to consume the metrics above
- [06-container-modes.md](06-container-modes.md) — Pod-template choices that affect cold-start

Continue to [09-this-repo-stack.md](09-this-repo-stack.md).
