# 8. Scaling Behavior — The Math

## 8.1 The formula

```
desired_runners = clamp(assigned_jobs + minRunners, 0, maxRunners)
```

Where:
- `assigned_jobs` = number of jobs GitHub has handed to this scale set but **not yet completed**.
- `minRunners` = idle warm pool you always want.
- `maxRunners` = hard ceiling.

## 8.2 Configurations and their behavior

| `minRunners` | `maxRunners` | Behavior |
|--------------|--------------|----------|
| unset / 0 | unset | **Unbounded scale-up**, scales to 0 when idle |
| 0 | N | Scale 0 → N as needed |
| K | unset | Always ≥ K warm; unbounded ceiling |
| K | N | K warm, up to N |
| **0** | **0** | **Drain mode** — accept no new runners (maintenance) |
| K > N | N | ❌ Helm rejects — `minRunners` cannot exceed `maxRunners` (unless `maxRunners` is unset) |

## 8.3 Warm pool trade-offs

| Setting | Job start latency | Idle cost |
|---------|-------------------|-----------|
| `minRunners: 0` | ~30–60s (pod schedule + image pull + register) | 0 |
| `minRunners: 1` | <5s for first concurrent job | 1 pod 24/7 |
| `minRunners: 5` | <5s for first 5 concurrent jobs | 5 pods 24/7 |

## 8.4 Burst scaling

When 50 jobs arrive at once:
1. Listener gets 50 `Job Available` messages.
2. Listener computes `desired = min(50 + minRunners, maxRunners)`.
3. Listener patches `EphemeralRunnerSet.spec.replicas`.
4. Controller creates 50 `EphemeralRunner` CRs (rate-limited by `runnerMaxConcurrentReconciles`).
5. For each CR: request JIT token → create Pod.
6. Pods schedule onto nodes. **If the cluster autoscaler is in play, this triggers node creation** (~3–5 min on cloud).

> **Architect tip** — Pre-warm the cluster (cluster-autoscaler "headroom" pods, or Karpenter `provisioner` with extra capacity) if burst latency is critical.

## 8.5 Scale-down

- A runner Pod exits **immediately** after its job completes (ephemeral).
- Controller deletes the `EphemeralRunner` CR after GitHub confirms deregistration.
- `EphemeralRunnerSet.replicas` is recomputed by the Listener on its next poll.
- No cool-down / hysteresis — scale-down is **immediate**.

## 8.6 Pod failure handling

| Event | Behavior |
|-------|----------|
| Pod fails to start (image pull, scheduling) | Retry up to **5 times** |
| Job assigned but no runner accepts in 24h | GitHub **unassigns** the job, requeues |
| Runner crashes mid-job | Job fails with "runner lost contact" |
| Listener Pod crashes | Controller recreates it; long-poll re-establishes; jobs continue |

## 8.7 Listener metrics for scaling visibility

Enable `metrics:` in the controller chart to expose:

| Metric | Meaning |
|--------|---------|
| `gha_assigned_jobs` | Currently assigned (driving force for scale-up) |
| `gha_running_jobs` | Jobs in progress |
| `gha_desired_runners` | What the listener wants |
| `gha_idle_runners` | Warm pool size |
| `gha_busy_runners` | Active job runners |
| `gha_job_startup_duration_seconds` | Histogram: time from queue → execution start |

Alert on:
- `gha_job_startup_duration_seconds` p95 > 60s → consider increasing `minRunners`.
- `gha_desired_runners >= maxRunners` for > 5 min → consider raising `maxRunners`.

Continue to [09-this-repo-stack.md](09-this-repo-stack.md).
