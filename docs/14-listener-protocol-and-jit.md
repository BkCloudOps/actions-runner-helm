# 14. Listener Wire Protocol and JIT Configuration — Deep Dive

This document explains, at the network and protocol level, **exactly** how the ARC listener communicates with the GitHub Actions service, how the long-poll connection is established and maintained, and the complete just-in-time (JIT) runner configuration lifecycle.

If [03-end-to-end-flow.md](03-end-to-end-flow.md) is the *narrative*, this document is the *packet capture*. It is intended for SREs debugging connectivity issues, security reviewers auditing the trust boundary, and engineers extending or replacing the listener.

> Sources: the ARC source tree (`cmd/ghalistener`, `cmd/githubrunnerscalesetlistener`), GitHub's published Actions service API surface, and behavior observed in `kubectl logs` of healthy listeners. Where the public documentation is silent, we describe **observed** behavior and label it accordingly.

---

## Table of contents

1. [Endpoints and trust](#1-endpoints-and-trust)
2. [Listener bootstrap sequence](#2-listener-bootstrap-sequence)
3. [The long-poll mechanism](#3-the-long-poll-mechanism)
4. [Message types the listener handles](#4-message-types-the-listener-handles)
5. [JIT configuration end-to-end](#5-jit-configuration-end-to-end)
6. [Runner-side handshake](#6-runner-side-handshake)
7. [Heartbeats, timeouts and retries](#7-heartbeats-timeouts-and-retries)
8. [Failure modes and recovery](#8-failure-modes-and-recovery)
9. [Egress firewall reference](#9-egress-firewall-reference)
10. [Reproducing the calls with `curl`](#10-reproducing-the-calls-with-curl)

---

## 1. Endpoints and trust

The listener talks to **two distinct services** on GitHub's side. Confusing them is the most common cause of egress-rule failures.

| Service | Purpose | Hostname (github.com) | Hostname (GHES) |
|---------|---------|-----------------------|-----------------|
| **GitHub REST API** | Authentication, runner scale set CRUD, JIT config generation, runner deregistration | `api.github.com` | `<ghes-host>/api/v3` |
| **Actions service (RunService)** | Long-poll for job messages, acquire jobs, report acquisition results | `*.actions.githubusercontent.com` (regional) | `<ghes-host>/_services/pipelines/...` |

Trust model:

- All connections are **outbound from the cluster**, TLS 1.2+ (the listener pins minimum TLS 1.2 in Go's `crypto/tls`).
- No inbound port is opened in the cluster for the listener.
- The Actions service URL is **discovered at runtime** via a REST API call (`GET /actions/runner-registration` or scale-set registration response) — you do **not** hard-code it. This means a listener that cannot reach `api.github.com` cannot even *find* the Actions service.
- The Actions service uses its **own** access tokens, distinct from the PAT/GitHub App credential. We explain how these are obtained in §2.

---

## 2. Listener bootstrap sequence

When the listener pod starts (image: `ghcr.io/actions/gha-runner-scale-set-listener:<version>`), it runs roughly the following sequence. Each numbered step corresponds to a discrete HTTPS request.

### 2.1 Read configuration

The listener reads its configuration from environment variables and a mounted secret. The controller projects the following into the pod:

| Source | Mount / env | Contents |
|--------|-------------|----------|
| Env var | `LISTENER_CONFIG_PATH` | Path to a JSON config file inside the pod. |
| File | `/etc/github/config.json` (typical) | Marshalled `Config` struct: `ConfigureUrl`, `EphemeralRunnerSetName`, `MaxRunners`, `MinRunners`, `RunnerScaleSetId`, `ServerRootCA`, etc. |
| Mount | `/etc/github/auth/` (when PAT) | File `github_token`. |
| Mount | `/etc/github/auth/` (when App) | Files `github_app_id`, `github_app_installation_id`, `github_app_private_key`. |

### 2.2 Mint a GitHub credential

Depending on which auth secret keys are present:

- **PAT**: The token is used verbatim as `Authorization: Bearer <token>` (or `token <token>` — both forms accepted by the REST API).
- **GitHub App**: The listener constructs a short-lived **JWT** signed with the private key (RS256, `iat=now-60s`, `exp=now+600s`, `iss=<app_id>`). It then POSTs to `POST /app/installations/<installation_id>/access_tokens` to exchange the JWT for an **installation access token** (valid for 1 hour). Subsequent REST calls use this installation token. Renewal occurs ~5 minutes before expiry.

### 2.3 Discover the Actions service

```http
POST /actions/runner-registration
Host: api.github.com
Authorization: Bearer <github_token_or_installation_token>
Content-Type: application/json

{ "url": "https://github.com/<owner>/<repo-or-org>", "runner_event": "register" }
```

The 200 response contains:

```json
{
  "url": "https://pipelines.actions.githubusercontent.com/<region>/<id>",
  "token": "<actions_runtime_token>"
}
```

- `url` — base URL for the **Actions RunService**. All long-poll traffic goes here.
- `token` — the **Actions Runtime Token (ART)**. A signed JWT used as `Authorization: Bearer <ART>` for every Actions service call. Typically valid for ~1 hour.

> The listener stores both, and refreshes the ART before expiry by repeating this call.

### 2.4 Register (or look up) the runner scale set

```http
POST <RunService>/runnerscalesets
Authorization: Bearer <ART>
Content-Type: application/json

{
  "name": "straw-hat-runners-linux",
  "runnerGroupId": <id>,
  "labels": [{ "name": "self-hosted", "type": "System" }, { "name": "straw-hat-runners-linux", "type": "User" }],
  "runnerSetting": { "ephemeral": true, "disableUpdate": true }
}
```

Possible outcomes:

| HTTP status | Meaning | Action |
|-------------|---------|--------|
| `201 Created` | New scale set registered | Persist `id` |
| `409 Conflict` + name match | Already registered | `GET /runnerscalesets?name=...` to fetch existing `id` |
| `403 Forbidden` | Token lacks admin scope | Listener crashes; fix PAT/App permissions |

The returned scale-set `id` is the integer also surfaced on `AutoscalingListener.spec.runnerScaleSetId` and used in every subsequent message.

### 2.5 Open the message session

```http
POST <RunService>/runnerscalesets/<id>/sessions
Authorization: Bearer <ART>
Content-Type: application/json

{ "ownerName": "<listener-pod-name>" }
```

Response:

```json
{
  "sessionId": "01HXYZ...",
  "ownerName": "...",
  "runnerScaleSet": { "id": ..., "name": "..." },
  "messageQueueUrl": "https://...",
  "messageQueueAccessToken": "<short-lived JWT>",
  "statistics": { ... }
}
```

The `sessionId` is the listener's "subscription". A scale set can have **only one active session at a time**; if a stale listener still holds one, the new listener gets `409 Conflict` and the controller deletes the stale session via `DELETE /sessions/<id>` before retrying.

`messageQueueAccessToken` is a **second** token, scoped strictly to message-queue reads, valid for ~10 minutes. The listener refreshes it on every long-poll cycle.

> **Key insight**: the session is what makes the listener stateful from GitHub's perspective. Restarting the listener pod tears down the session; recreating it triggers a fresh session and a fresh long-poll loop. Jobs are not lost — they remain assigned to the scale set on GitHub's side and are re-offered to the new session.

### 2.6 Enter the receive loop

The listener now enters its main loop: long-poll → handle messages → patch ERS → repeat. See §3.

---

## 3. The long-poll mechanism

"Long-poll" here is **not** WebSocket, not Server-Sent Events, not gRPC streaming. It is a sequence of regular HTTPS `GET` requests, each of which the server is permitted to hold open for up to ~50 seconds before responding.

### 3.1 The single request

```http
GET <messageQueueUrl>?sessionId=<sid>&lastMessageId=<n>&status=Online
Authorization: Bearer <messageQueueAccessToken>
Accept: application/json; api-version=6.0-preview
User-Agent: actions-runner-controller/<version>
```

Behavior on the server side:

| If during the hold window... | Server responds with |
|------------------------------|----------------------|
| A message is enqueued for this scale set | `200 OK` + JSON message body (immediate) |
| No message arrives | `200 OK` + empty body (after ~50s) **or** `204 No Content` |
| The access token expires | `401 Unauthorized` |
| Another session has taken over | `409 Conflict` |

The listener's HTTP client is configured with a **client-side read timeout of ~100 seconds** — comfortably longer than the server's hold window. If the read times out client-side, the connection is closed and a new request is issued.

### 3.2 Why long-poll and not WebSocket?

| Trade-off | Long-poll | WebSocket |
|-----------|-----------|-----------|
| Egress firewall friendliness | ✓ Plain HTTPS, every proxy supports it | ✗ Many corporate proxies block `Upgrade: websocket` |
| Idempotent retries | ✓ Each cycle is a fresh GET | More complex reconnection |
| Server-side resource cost | Higher (TCP held open) | Lower |
| Latency to deliver a message | ~5 ms (server flushes immediately when one is enqueued) | ~5 ms |

For the small message volume (one per job-event) and the goal of working through arbitrary corporate egress, long-poll is the right choice.

### 3.3 Cursor: `lastMessageId`

Every message returned has a monotonically increasing `messageId` (per-session). The listener includes `lastMessageId=<last successfully processed>` in the next request. This means:

- Messages are **delivered at-least-once** to the listener.
- The listener must be **idempotent** for repeated messages.
- After a crash + restart, the new session begins at `lastMessageId=0` and the server replays undelivered messages (it retains them for a short window — minutes).

### 3.4 The ACK

After processing a message, the listener calls:

```http
DELETE <messageQueueUrl>/<messageId>?sessionId=<sid>
Authorization: Bearer <messageQueueAccessToken>
```

This removes the message from the queue. Failure to ACK means the message will be redelivered on the next poll (and possibly to a fresh listener after a restart).

---

## 4. Message types the listener handles

The listener understands a small set of message types. The exact `messageType` strings are defined by the Actions service; we list the observed values.

| `messageType` | When sent | Listener action |
|---------------|-----------|-----------------|
| `RunnerScaleSetJobMessages` | Batch wrapper. Contains one or more inner job-event messages. | Iterate `body.messages` and dispatch each. |
| `JobAvailable` | A workflow job has been queued and matches this scale set's labels. | Decide whether to acquire it (always yes if `acquiredJobs + busy + idle < maxRunners`). |
| `JobAssigned` | The server confirms a previously-acquired job is bound to this scale set. | Update internal "assigned" counter; bump `EphemeralRunnerSet.spec.replicas` if needed. |
| `JobStarted` | A specific runner has begun executing the job. | Update metrics (`gha_running_jobs`). |
| `JobCompleted` | The job finished (any outcome). | Decrement counters; if no minRunners floor remains, the next reconcile may scale down. |

### 4.1 Acquiring jobs

`JobAvailable` is only an *offer*. The listener accepts by calling:

```http
POST <RunService>/runnerscalesets/<id>/acquirejobs?sessionId=<sid>
Authorization: Bearer <messageQueueAccessToken>
Content-Type: application/json

[<requestId1>, <requestId2>, ...]
```

The body is an array of `requestId` integers (each `JobAvailable` carries one). The server responds with the subset it actually granted to this scale set. Why a subset?

- Other scale sets may compete for the same `runs-on:` labels.
- Server-side rate limits.
- A job may have been canceled in the gap between offer and acquire.

The listener must update its internal `assigned` counter from the **response**, not the request.

### 4.2 Example message body

```json
{
  "messageId": 42,
  "messageType": "RunnerScaleSetJobMessages",
  "body": {
    "messages": [
      {
        "messageType": "JobAvailable",
        "runnerRequestId": 1024,
        "repositoryName": "BkCloudOps/infra",
        "ownerName": "BkCloudOps",
        "jobDisplayName": "build",
        "jobWorkflowRef": "BkCloudOps/infra/.github/workflows/ci.yml@refs/heads/main",
        "eventName": "push",
        "requestLabels": ["self-hosted", "straw-hat-runners-linux"]
      }
    ]
  }
}
```

---

## 5. JIT configuration end-to-end

This section answers the explicit question: *"If JIT, then what are the steps?"*

### 5.1 Why JIT exists

Traditional self-hosted runners are configured with `config.sh --token <REG_TOKEN>`. The registration token is broad: anyone holding it can register *any* runner under the same scope. For ARC the requirements are stricter:

1. Each runner must be **single-use** (cannot be replayed).
2. Each runner is bound to a **specific name** chosen by the controller (so the controller can correlate Pods ↔ runners).
3. Runners must be able to register **without** holding the org/enterprise admin PAT (the Pod is untrusted user-code territory).

JIT (just-in-time) config solves all three:

- One opaque blob.
- Encodes runner name + group + labels + a credential good for exactly one registration.
- Burned on first use.

### 5.2 The JIT generation API

The **controller** (not the listener, not the runner) calls:

```http
POST <RunService>/runnerscalesets/<id>/generatejitconfig
Authorization: Bearer <ART>
Content-Type: application/json

{
  "name": "straw-hat-runners-linux-abcde",
  "runnerScaleSetId": <id>,
  "workFolder": "_work"
}
```

The chosen `name` follows the pattern `<scaleSetName>-<5-char-random>`. Uniqueness is enforced server-side.

Response:

```json
{
  "runner": {
    "id": 9876,
    "name": "straw-hat-runners-linux-abcde",
    "os": "linux",
    "status": "offline",
    "labels": [ ... ]
  },
  "encodedJITConfig": "eyJ..."
}
```

`encodedJITConfig` is a **base64-encoded JSON blob** containing (decoded):

```json
{
  "Url": "https://github.com/<owner>",
  "RunnerName": "straw-hat-runners-linux-abcde",
  "AuthToken": "<one-shot registration credential>",
  "AgentId": 9876,
  "PoolId": 1,
  "GitHubUrl": "...",
  "Labels": "self-hosted,straw-hat-runners-linux",
  "WorkFolder": "_work",
  "AlwaysInteractive": false
}
```

> **Do not log this value at INFO level.** Treat `encodedJITConfig` as a credential.

### 5.3 Injecting into the Pod

The controller writes the encoded blob into the Pod **only** through env (not a Secret), and only at Pod-create time:

```yaml
spec:
  containers:
  - name: runner
    env:
    - name: ACTIONS_RUNNER_INPUT_JITCONFIG
      value: "<encodedJITConfig>"
    - name: GITHUB_ACTIONS_RUNNER_EXTRA_USER_AGENT
      value: "actions-runner-controller/0.14.1"
```

Why env, not Secret?

- The blob is single-use; persisting it in a `Secret` would broaden its exposure and create lifecycle headaches.
- Env vars are scoped to the Pod's process; on exit, they are gone.
- The Pod is owned by an `EphemeralRunner` CR; if the runner crashes before consumption, the controller generates a *fresh* JIT and recreates the Pod.

Trade-off: env vars are visible via `kubectl describe pod` to anyone with read access on the namespace. This is part of why **runners run in a separate namespace** from the controller, and why namespace RBAC must be tight (see [12-security-model.md](12-security-model.md)).

### 5.4 Lifecycle properties of a JIT config

| Property | Value |
|----------|-------|
| Validity window | The `AuthToken` inside is valid for ~1 hour after generation. |
| Number of uses | **Exactly one.** Server-side, the token is burned at first registration. |
| Bound to | One specific runner name + scale set id. |
| Revocable | Yes — delete the runner via `DELETE /actions/runners/<id>` (controller does this when an `EphemeralRunner` is deleted before registration). |

If the Pod fails to start within the validity window, the controller's reconciler detects the stuck `EphemeralRunner`, calls `DELETE /actions/runners/<id>` to clean up the unused registration, and on the next retry generates a **fresh** JIT.

---

## 6. Runner-side handshake

Inside the runner Pod, the container entrypoint is `/home/runner/run.sh` (or `/usr/local/bin/dumb-init` → `run.sh`). The script:

### 6.1 Detect JIT mode

```bash
if [[ -n "$ACTIONS_RUNNER_INPUT_JITCONFIG" ]]; then
    exec ./run.sh --jitconfig "$ACTIONS_RUNNER_INPUT_JITCONFIG"
fi
```

The runner binary base64-decodes the blob, reads `Url`, `RunnerName`, `AuthToken`, etc.

### 6.2 Register

The runner POSTs to GitHub's runner-registration endpoint, presenting `AuthToken`. The server responds with a long-lived **session credential** (different from `AuthToken`) scoped to *this specific runner instance*. The session credential is held only in memory inside the runner process.

### 6.3 Open the runner's own long-poll

The runner now opens a **separate** long-poll to the Actions service, distinct from the listener's. Conceptually similar HTTP pattern, but the messages are job-execution oriented:

| Message | Meaning |
|---------|---------|
| `PipelineAgentJobRequest` | "Here is your job. Execute it." Body contains the resolved workflow steps, secrets (masked), env, and tool cache hints. |
| `JobCancellation` | The user pressed Cancel in the UI. |
| `AgentRefresh` | Server-pushed config update (rare; ARC disables auto-update so this is informational). |

Once a `PipelineAgentJobRequest` arrives, the runner spawns `Runner.Worker` (a child process) which actually executes the steps. The runner streams logs back to the Actions service over yet another HTTPS connection (chunked-encoded, append-only).

### 6.4 Job completion

When `Runner.Worker` exits:

1. The runner POSTs a final job status to the Actions service.
2. Because the runner was started with `--ephemeral` (implied by JIT), it **deregisters itself** by calling `DELETE /actions/runners/<id>` with its session credential.
3. The runner process exits with code 0 (job success or failure are both code 0 from the runner's perspective; only catastrophic failures yield non-zero).
4. The Pod terminates. `EphemeralRunner.status.phase` transitions to `Succeeded` once the controller confirms deregistration via the next reconcile.

---

## 7. Heartbeats, timeouts and retries

### 7.1 Listener side

| Timer | Value | Purpose |
|-------|-------|---------|
| HTTP client read timeout | ~100 s | Bounded by server's ~50 s hold + slack |
| ART refresh interval | ~50 min | Refresh before its ~1 h expiry |
| Message-queue token refresh | Every poll | Token returned alongside each batch |
| Session refresh | On 409 / 401 | Tear down and recreate |
| Backoff on consecutive errors | Exponential, capped at 30 s | Avoids hammering a degraded service |

### 7.2 Runner side

| Timer | Value | Purpose |
|-------|-------|---------|
| Initial registration timeout | 60 s | If exceeded, exit non-zero → Pod restart |
| Job request long-poll | ~50 s server hold | Same pattern as listener |
| Log batch flush interval | 250 ms or 1 KiB | Whichever first |
| Idle ephemeral timeout | None | The runner stays in long-poll indefinitely; the Pod is killed by the controller if the scale set is deleted |

### 7.3 Controller side (driving ERS)

| Event | Behavior |
|-------|----------|
| Listener patches `ERS.spec.replicas` | Reconcile within `~1 s` |
| Pod creation backoff on failure | 5 retries with exponential delay; then `EphemeralRunner.status.phase=Failed` |
| GitHub `429 Too Many Requests` on JIT generation | Honor `Retry-After`; surface as Pod event |

---

## 8. Failure modes and recovery

| Symptom | Where it shows up | Likely cause | Recovery |
|---------|-------------------|--------------|----------|
| Listener `CrashLoopBackOff`, log `401 Bad credentials` | listener logs | PAT expired / wrong scopes / App token failed to mint | Rotate auth secret; delete listener pod |
| Listener log `403 Forbidden` on `runnerscalesets` POST | listener logs | App missing `Administration: write` or PAT missing `admin:org` | Fix permissions |
| Listener log `409 Conflict` on session create, repeating | listener logs | Stale session on server | Wait ~2 min for server-side session timeout, **or** restart listener and let it `DELETE /sessions/<id>` |
| `EphemeralRunner.status.phase=Failed`, `reason=FailedToGetJITConfig` | runner CR | JIT generation returned 4xx (rate limit / org permission lost) | Inspect controller logs; verify PAT |
| Runner pod `Error`, log `Cannot connect to the runner service` | pod logs | Egress blocked to `*.actions.githubusercontent.com` | Open firewall to **regional** Actions hostnames (see §9) |
| Runner registers but immediately deregisters | GitHub UI shows brief flash of runner | JIT config is corrupted or already used | Controller will retry with fresh JIT; if persistent, suspect a stale `EphemeralRunner` CR or clock skew |
| Listener loses messages | none — silently fewer runners | `lastMessageId` cursor reset incorrectly after restart | Updating to a newer listener version usually fixes; verify with `gha_assigned_jobs` metric |

---

## 9. Egress firewall reference

Minimum outbound rules from cluster:

| Destination | Port | Protocol | Who connects | Why |
|-------------|------|----------|--------------|-----|
| `api.github.com` (or GHES `/api/v3`) | 443 | TCP/TLS | Listener, controller | REST API |
| `*.actions.githubusercontent.com` | 443 | TCP/TLS | Listener, runner pod | RunService long-poll, job dispatch, logs |
| `codeload.github.com` | 443 | TCP/TLS | Runner pod | Action source download (`actions/checkout@v4` etc.) |
| `objects.githubusercontent.com` | 443 | TCP/TLS | Runner pod | LFS, releases, artifacts |
| `ghcr.io` | 443 | TCP/TLS | Runner pod (often) | Container actions, image pulls |
| `results-receiver.actions.githubusercontent.com` | 443 | TCP/TLS | Runner pod | Newer log/results service |

For GHES, replace `*.actions.githubusercontent.com` with `<ghes-host>` (the Actions service is co-hosted).

> **Region pinning** — the `*.actions.githubusercontent.com` hostname resolves to a **regional** subdomain assigned at session-creation time (e.g. `pipelinesghubeus2.actions.githubusercontent.com`). Once assigned, a single listener consistently uses the same region. Allowlisting the wildcard is recommended; pinning a single regional subdomain is fragile.

---

## 10. Reproducing the calls with `curl`

For debugging, you can emulate every API the listener uses. All commands assume `export GH_TOKEN=<your PAT>`.

### 10.1 Discover the Actions service

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  https://api.github.com/actions/runner-registration \
  -d '{"url":"https://github.com/BkCloudOps","runner_event":"register"}' | jq .
```

You should receive `url` (the RunService base) and `token` (the ART).

### 10.2 List your runner scale sets

```bash
export ART="<token from previous step>"
export RUN_SVC="<url from previous step>"

curl -sS -H "Authorization: Bearer $ART" \
  -H "Accept: application/json; api-version=6.0-preview" \
  "$RUN_SVC/runnerscalesets" | jq '.value[] | {id,name,runnerSetting}'
```

### 10.3 Generate a JIT config (do **not** do this casually — burns a name)

```bash
SCALE_SET_ID=<id-from-previous-step>
curl -sS -X POST \
  -H "Authorization: Bearer $ART" \
  -H "Accept: application/json; api-version=6.0-preview" \
  -H "Content-Type: application/json" \
  "$RUN_SVC/runnerscalesets/$SCALE_SET_ID/generatejitconfig" \
  -d '{"name":"manual-test-001","runnerScaleSetId":'"$SCALE_SET_ID"',"workFolder":"_work"}' | jq .
```

The response contains `encodedJITConfig`. If you don't actually start a runner with it, clean up:

```bash
RUNNER_ID=<runner.id from response>
curl -sS -X DELETE \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/orgs/BkCloudOps/actions/runners/$RUNNER_ID"
```

### 10.4 Open a session and long-poll once

```bash
# Create session
curl -sS -X POST \
  -H "Authorization: Bearer $ART" \
  -H "Content-Type: application/json" \
  "$RUN_SVC/runnerscalesets/$SCALE_SET_ID/sessions" \
  -d '{"ownerName":"manual-debug"}' | jq .
# capture sessionId and messageQueueAccessToken

MQ_TOKEN="..."
SID="..."
MQ_URL="$RUN_SVC/runnerscalesets/$SCALE_SET_ID/messages"

# Poll (will block up to ~50s)
time curl -sS -H "Authorization: Bearer $MQ_TOKEN" \
  "$MQ_URL?sessionId=$SID&lastMessageId=0&status=Online"
```

Trigger a workflow targeting this scale set's label in another terminal; the poll above should return immediately with a `JobAvailable` message.

Always tear down:

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ART" \
  "$RUN_SVC/runnerscalesets/$SCALE_SET_ID/sessions/$SID"
```

---

## See also

- [03-end-to-end-flow.md](03-end-to-end-flow.md) — the same flow at narrative level
- [07-authentication.md](07-authentication.md) — what PAT / GitHub App scopes are needed
- [11-observability.md](11-observability.md) — which metrics tell you the protocol is healthy
- [13-crd-reference.md](13-crd-reference.md) — `AutoscalingListener.spec.runnerScaleSetId` and friends
