# 3. End-to-End Flow — From `git push` to Job Completion

This chapter traces the **complete** life of a single workflow run, from the moment a developer pushes code to the moment the runner Pod is garbage-collected. Each step names the actor, the action, and the resulting cluster or GitHub state change. Where the step involves an actual HTTP request, the [protocol document](14-listener-protocol-and-jit.md) is cross-referenced.

> **Convention.** Numbered steps below are *causal*, not strictly wall-clock-ordered. Several near-simultaneous events are grouped for clarity. A timing budget for each step is given in §3.4.

## 3.1 Pre-conditions

The flow below assumes the cluster is already in the steady state described in [02-architecture.md §2.6](02-architecture.md#26-reconcile-order-at-install):

| Resource | State |
|----------|-------|
| `Deployment/arc-gha-rs-controller` in `arc-systems` | `Available`, 1/1 Ready |
| `AutoscalingRunnerSet/straw-hat-runners-linux` in `arc-runners` | `Created`; `.status.currentRunners=0` |
| `AutoscalingListener/...` in `arc-systems` | Pod `Running`; long-poll session established |
| `EphemeralRunnerSet/...` in `arc-runners` | `.spec.replicas=0`, `.status.currentReplicas=0` |
| `Secret/arc-github-config` in `arc-runners` (ESO-synced) | Contains `github_token` |
| Runner pool visible on GitHub | *Settings → Actions → Runner groups → default → "straw-hat-runners-linux"* shows **0 online, 0 idle, 0 busy** |

## 3.2 The flow

### Step 1 — Developer triggers a workflow

```yaml
# .github/workflows/build.yml
name: build
on: [push]
jobs:
  build:
    runs-on: [self-hosted, straw-hat-runners-linux]
    steps:
      - uses: actions/checkout@v4
      - run: make test
```

`git push origin main` causes GitHub to evaluate the workflow file, instantiate a *workflow run*, and enqueue one *job* (`build`) with `labels = [self-hosted, straw-hat-runners-linux]`.

**State change:** GitHub's internal job queue now has one entry. From the cluster's perspective, nothing has happened yet.

### Step 2 — GitHub matches the job to a scale set

GitHub's job dispatcher iterates registered runner scale sets across the org/enterprise and finds one with matching labels. The match has two outputs:

- The job is *offered* to that scale set's active session.
- Until acquired, the job remains `queued` in the GitHub UI.

> Multiple scale sets can match a single job. The first one to call `acquirejobs` wins. This is how a slow listener can lose work to a faster one — see [14-listener-protocol-and-jit.md §4.1](14-listener-protocol-and-jit.md#41-acquiring-jobs).

### Step 3 — Listener receives a `JobAvailable` message

The listener's open long-poll connection (held on `messageQueueUrl?sessionId=…&lastMessageId=N`) returns `200 OK` with a `RunnerScaleSetJobMessages` envelope containing one `JobAvailable` inner message. Wire detail: [14-listener-protocol-and-jit.md §4](14-listener-protocol-and-jit.md#4-message-types-the-listener-handles).

The listener immediately:

1. ACKs the message (`DELETE` against the queue with the message id).
2. Calls `POST .../acquirejobs` with the offered `requestId`.
3. Updates its in-memory `assigned` counter from the server's response (which lists the *granted* request IDs; the server may have already given the job to someone else).

### Step 4 — Listener computes desired replicas

```
desired = clamp(assignedJobs + minRunners,  /* floor not relevant here */
                minRunners,
                maxRunners or +∞)
```

With `assignedJobs=1`, `minRunners=0`, `maxRunners=20` → `desired=1`. For the scale-down direction the formula is the same, applied after `JobCompleted` decrements `assignedJobs`. The full algorithm including burst behavior is in [08-scaling-behavior.md](08-scaling-behavior.md).

### Step 5 — Listener patches the `EphemeralRunnerSet`

Equivalent to:

```bash
kubectl patch ephemeralrunnerset/straw-hat-runners-linux -n arc-runners \
  --type=merge -p '{"spec":{"replicas":1,"patchID":12345}}'
```

`patchID` is monotonically increasing per listener session. The controller uses it to discard out-of-order patches.

**RBAC.** The listener's ServiceAccount has a `Role` in `arc-runners` permitting `get/list/watch/patch` on `ephemeralrunnersets`. It cannot create Pods, Secrets, or any other resource — this is by design (see [12-security-model.md](12-security-model.md)).

### Step 6 — Controller scales the `EphemeralRunnerSet`

`EphemeralRunnerSetReconciler` is triggered by the watch event. It observes `replicas=1` but `currentReplicas=0` and creates one `EphemeralRunner` CR with `generateName: straw-hat-runners-linux-`. The owner reference points to the `EphemeralRunnerSet` so garbage collection is automatic.

### Step 7 — Controller generates the JIT configuration

The controller calls:

```http
POST <RunService>/runnerscalesets/<id>/generatejitconfig
Authorization: Bearer <ART>

{ "name": "straw-hat-runners-linux-abcde", "runnerScaleSetId": <id>, "workFolder": "_work" }
```

The response contains `encodedJITConfig` (a base64-encoded blob holding a single-use registration credential bound to the chosen runner name) and `runner.id` (the integer GitHub-side runner ID). The controller writes these into the `EphemeralRunner` CR:

- `.spec` is unchanged (it was created from the ERS template).
- `.status.runnerId` and `.status.runnerName` are updated.

The JIT blob itself is **not** persisted to a Secret; it is staged in memory and injected directly into the Pod spec in the next step. Full lifecycle in [14-listener-protocol-and-jit.md §5](14-listener-protocol-and-jit.md#5-jit-configuration-end-to-end).

### Step 8 — Controller creates the runner Pod

The Pod spec is assembled from three layers, applied in order:

1. `AutoscalingRunnerSet.spec.template` — the user-supplied PodTemplateSpec.
2. Container-mode mutations (see [06-container-modes.md](06-container-modes.md)): adds `dind` sidecar, or `kubernetes`-mode init container, or neither.
3. ARC-managed env vars and volumes:
   - `ACTIONS_RUNNER_INPUT_JITCONFIG = <encodedJITConfig>`
   - `RUNNER_WORKSPACE`, `GITHUB_ACTIONS_RUNNER_EXTRA_USER_AGENT`
   - `restartPolicy: Never`
   - A projected token mount for the runner's ServiceAccount (in `kubernetes` mode)

Once submitted to the API server, the Pod follows the normal Kubernetes scheduling path. The controller's job here is complete until the Pod's phase changes.

### Step 9 — Runner Pod registers with GitHub

The Pod starts. `entrypoint` (`/usr/local/bin/dumb-init` → `/home/runner/run.sh`) detects `ACTIONS_RUNNER_INPUT_JITCONFIG` is set, decodes it, and the `Runner.Listener` binary calls GitHub's runner-registration endpoint, presenting the embedded one-shot token. GitHub responds with a long-lived **session credential** scoped to this exact runner instance.

**State changes:**
- GitHub UI: the runner appears under *Runner groups* with status *idle*.
- Cluster: `EphemeralRunner.status.phase` transitions `Pending → Running` and `.status.ready=true` once the runner reports `Online`.

### Step 10 — Runner opens its own long-poll

The runner now establishes its own HTTPS long-poll to the RunService — **distinct** from the listener's session. This connection receives the actual job payload (workflow steps, masked secrets, env, tool-cache hints) once GitHub dispatches the job to it.

> The listener and the runners are independent subscribers. The listener never sees the job payload; it only sees lifecycle messages.

### Step 11 — GitHub dispatches the job to the runner

GitHub sends a `PipelineAgentJobRequest` over the runner's long-poll. The runner forks `Runner.Worker`, which:

1. Parses the request, evaluates expressions, resolves secret references.
2. Downloads each referenced action (e.g. `actions/checkout@v4`) from `codeload.github.com`.
3. Executes each step in sequence, streaming logs back to the Actions service over a separate chunked-encoded HTTPS upload.
4. Applies the workflow's `container:` / `services:` directives via the configured container mode:
   - **`dind`** — `dockerd` sidecar receives `docker run` calls on a shared socket volume.
   - **`kubernetes`** — `runner-container-hooks` calls the Kubernetes API to create *sibling Pods* in `arc-runners`, mounting shared volumes for inputs/outputs.
   - **`kubernetes-novolume`** — same but without the shared volume; data passed by other means.

### Step 12 — Job completes

`Runner.Worker` exits. The runner posts a final status (`succeeded` / `failed` / `cancelled`) to the Actions service. Because the runner was started with the `--ephemeral` semantics that JIT implies, it then:

1. Calls `DELETE /actions/runners/<runner.id>` with its session credential, deregistering itself from GitHub.
2. Exits the `Runner.Listener` process with code 0.

### Step 13 — Pod reaches terminal phase

`restartPolicy: Never` means the Pod's phase becomes `Succeeded` once the runner container exits cleanly (any other code yields `Failed`). The Pod is not deleted automatically by Kubernetes.

### Step 14 — Controller reaps the `EphemeralRunner`

`EphemeralRunnerReconciler` is triggered by the Pod's phase change. It:

1. Optionally calls GitHub to verify deregistration succeeded.
2. Sets `EphemeralRunner.status.phase = Succeeded`.
3. Removes the `ephemeralrunner.actions.github.com/finalizer`.
4. Deletes the `EphemeralRunner` CR. Owner-reference cascade deletes the Pod.

### Step 15 — Scale-down settles

`EphemeralRunnerSetReconciler` observes `currentReplicas (1) > replicas (still 1 at this point on listener's side, eventually 0)`. Meanwhile the listener has received `JobCompleted` and patched `.spec.replicas=max(minRunners, assignedJobs)=0`. The reconciler converges; the cluster is back to the idle state described in §3.1.

## 3.3 Sequence diagram (compact)

```
Dev      GitHub-REST     GitHub-RunService   Listener      Controller      EphemeralRunnerSet    EphemeralRunner    Runner Pod
 │           │                 │                  │            │                  │                    │                │
 │ push ───► │                 │                  │            │                  │                    │                │
 │           │  ──── queue job ────►             │            │                  │                    │                │
 │           │                 │ JobAvailable ──► │            │                  │                    │                │
 │           │                 │ ◄── acquirejobs ─│            │                  │                    │                │
 │           │                 │                  │ patch RS ─►│                  │                    │                │
 │           │                 │                  │            │ reconcile ──────►│                    │                │
 │           │                 │                  │            │                  │ create ───────────►│                │
 │           │ ◄── /generatejitconfig ──────────────────────── │                  │                    │                │
 │           │                 │                  │            │ create Pod ───────────────────────────────────────────►│
 │           │ ◄────────── register (JIT) ──────────────────────────────────────────────────────────────────────────────│
 │           │                 │ ◄── runner long-poll ──────────────────────────────────────────────────────────────────│
 │           │                 │ ── PipelineAgentJobRequest ───────────────────────────────────────────────────────────►│
 │           │                 │ ◄── logs/results (streaming) ──────────────────────────────────────────────────────────│
 │           │ ◄────── DELETE /actions/runners/<id> ───────────────────────────────────────────────────────────────────│
 │           │                 │ JobCompleted ──► │            │                  │                    │                │
 │           │                 │                  │ patch RS ─►│                  │                    │                │
 │           │                 │                  │ replicas=0 │ reconcile ──────►│ ◄─── delete CR ────│                │
 │           │                 │                  │            │                  │                    │ ◄── delete Pod ─│ (gone)
```

## 3.4 Timing budget (typical, healthy cluster)

| Phase | Typical wall-clock | Dominated by |
|-------|--------------------|--------------|
| Step 1 → Step 3 (push to `JobAvailable`) | < 1 s | GitHub-side queueing |
| Step 3 → Step 5 (acquire + patch ERS) | 100–500 ms | Listener processing |
| Step 5 → Step 8 (reconcile + Pod create) | 100–500 ms | Controller reconcile |
| Step 8 → Step 9 (Pod scheduled + image pulled + registered) | **3–60 s** (cold), < 5 s (warm pool) | **Image pull + node availability** |
| Step 11 → Step 12 (job execution) | seconds to hours | Your workflow |
| Step 12 → Step 15 (cleanup) | 1–5 s | Two API calls + finalizer |

The big variable is step 8 → 9. With `minRunners ≥ 1` (warm pool), the first job per concurrent slot inherits an already-registered runner and starts in seconds. With `minRunners = 0`, every cold start pays the full Pod-creation + image-pull + runner-registration cost. See [08-scaling-behavior.md](08-scaling-behavior.md) for tuning.

## 3.5 Cluster state at each phase

| Phase | `currentRunners` | `pendingEphemeralRunners` | `runningEphemeralRunners` | Pods in `arc-runners` |
|-------|------------------|---------------------------|----------------------------|------------------------|
| Idle (before push) | 0 | 0 | 0 | 0 |
| Listener acquired job, Pod creating | 1 | 1 | 0 | 1 (Pending) |
| Pod registered, awaiting job dispatch | 1 | 0 | 1 | 1 (Running, idle runner) |
| Job executing | 1 | 0 | 1 | 1 (Running, busy) |
| Job done, Pod terminating | 1 | 0 | 0 | 1 (Succeeded, NotReady) |
| Reconciled away | 0 | 0 | 0 | 0 |

Useful watch command for observing this transition live:

```bash
watch -n1 'kubectl get autoscalingrunnerset,ephemeralrunnerset,ephemeralrunner,pod -n arc-runners'
```

Continue to [04-controller-values.md](04-controller-values.md).
