# 3. End-to-End Flow — From `git push` to Job Completion

This is the most important document. It traces what happens, step-by-step, from the moment a developer triggers a workflow until the runner Pod is destroyed.

## 3.1 Pre-conditions (assumed already in place)

1. Helm release `arc` of `gha-runner-scale-set-controller` is installed in `arc-systems`. The controller-manager Pod is `Running`.
2. Helm release of `gha-runner-scale-set` is installed in `arc-runners` with:
   - `githubConfigUrl: https://github.com/BkCloudOps`
   - `githubConfigSecret: github-pat-secret` (synced from Azure Key Vault by External Secrets)
   - `runnerScaleSetName: "straw-hat-runners-linux"`
   - `minRunners: 0`, `maxRunners: 5`
3. The `AutoscalingRunnerSet` CR exists. The controller has reconciled it and spawned a **Listener Pod** named `<release>-listener` in `arc-systems`.
4. The Listener has authenticated to GitHub and opened a long-poll HTTPS connection.
5. **No runner Pods exist** (because `minRunners=0` and no jobs are queued).

## 3.2 The flow

### ⓵ Developer pushes code
```yaml
# .github/workflows/build.yml
jobs:
  build:
    runs-on: straw-hat-runners-linux     # <-- matches runnerScaleSetName
    steps:
      - uses: actions/checkout@v4
      - run: make test
```
`git push` → GitHub receives webhook → workflow is parsed → job `build` is **queued**.

### ⓶ GitHub matches the job to a runner scale set
GitHub looks at `runs-on: straw-hat-runners-linux` and finds a registered scale set with that name. The job sits in the queue *for that scale set*.

### ⓷ GitHub sends "Job Available" to the Listener
The Listener's long-poll connection unblocks with a `Job Available` message. (No webhooks, no inbound firewall holes — the cluster initiated the connection.)

### ⓸ Listener decides whether to scale up
The Listener computes:
```
desired = min(maxRunners, assignedJobs + minRunners)
```
With `minRunners=0`, `maxRunners=5`, and 1 newly-assigned job → `desired = 1`. It ACKs the message back to GitHub.

### ⓹ Listener patches the EphemeralRunnerSet
Using a Kubernetes ServiceAccount + Role, the Listener does:
```bash
kubectl patch ephemeralrunnerset <name> --type=merge -p '{"spec":{"replicas":1}}'
```

### ⓺ Controller creates an EphemeralRunner CR
The `EphemeralRunnerSet` controller (inside controller-manager) sees `replicas=1` but only 0 `EphemeralRunner` CRs exist. It creates one.

### ⓻ Controller requests a JIT token from GitHub
For that `EphemeralRunner`, the controller calls GitHub's REST API:
```
POST /actions/runners/generate-jitconfig
```
returning a **Just-in-Time (JIT) configuration token**. This token is:
- Single-use
- Bound to one runner name
- Short-lived

> **Why JIT?** Traditional self-hosted runners use a long-lived registration token; if leaked, an attacker can register a malicious runner. JIT tokens cannot be replayed.

### ⓼ Controller creates the runner Pod
The Pod spec is assembled from:
- `template.spec` from your `values.yaml`
- Auto-injected by `containerMode` (e.g., the `dind` sidecar)
- Auto-injected env vars: `ACTIONS_RUNNER_INPUT_JITCONFIG`, `RUNNER_NAME`, etc.
- The `github-pat-secret` mount (so the runner image can reach the GitHub API if needed)

If the Pod fails to start, the controller retries up to **5 times**. If after **24 hours** no runner accepts the job, GitHub un-assigns it.

### ⓽ Runner Pod registers with GitHub
Inside the Pod, `/home/runner/run.sh` starts. It:
1. Reads the JIT token from env.
2. Calls GitHub to register: `POST /actions/runners/...`
3. Opens its **own** HTTPS long-poll to GitHub waiting for the job payload.

### ⓾ GitHub dispatches the job to the runner
GitHub sends the job payload (steps, env, secrets) over the runner's long-poll connection.

### ⑪ Runner executes the job
- For each step, the runner:
  - Downloads the action (e.g., `actions/checkout@v4`)
  - Executes it
  - Streams logs back to GitHub in real time
- If a step uses `container:` or `services:`:
  - In **dind mode**: the `dockerd` sidecar starts the container.
  - In **kubernetes mode**: `runner-container-hooks` calls the K8s API to create a sibling Pod in the same namespace.

### ⑫ Job completes
The runner reports the final status (success/failure) and **exits the process**. Because the runner was started with `--ephemeral`, it deregisters itself from GitHub.

### ⑬ Pod terminates
The runner container exits with code 0 → the Pod enters `Succeeded` state.

### ⑭ Controller deletes the EphemeralRunner CR
The `EphemeralRunner` controller polls GitHub: "Can I delete this runner?" GitHub confirms deregistration → the CR is deleted → owner-reference cascade deletes the Pod.

### ⑮ EphemeralRunnerSet scales back down
Now `len(EphemeralRunner CRs) = 0`. If `minRunners = 0` and no other jobs are queued, the replica count goes to 0. The cluster is back to idle.

## 3.3 Sequence diagram (compressed)

```
Dev      GitHub-Actions       Listener        Controller       EphemeralRunner       Runner Pod
 │           │                   │                │                  │                    │
 │ push ──►  │                   │                │                  │                    │
 │           │ ◄── long-poll ───►│                │                  │                    │
 │           │   "JobAvailable"  │                │                  │                    │
 │           │                   │── patch RS ──►│                  │                    │
 │           │                   │                │── create CR ──►│                    │
 │           │ ◄── /jitconfig ──────────────────  │                  │                    │
 │           │                   │                │── create Pod ────────────────────► (start)
 │           │ ◄────────── register (JIT) ─────────────────────────────────────────── │
 │           │ ────────── job payload ──────────────────────────────────────────────► │
 │           │ ◄────────── logs/status ─────────────────────────────────────────────── │
 │           │                   │                │ ◄── completed ──┤                    │
 │           │ ◄── deregister ────────────────────│                  │                    │
 │           │                   │                │── delete CR ──►│                    │
 │           │                   │                │                                     (exit)
```

Continue to [04-controller-values.md](04-controller-values.md).
