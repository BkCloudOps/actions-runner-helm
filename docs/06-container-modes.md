# 6. Container Modes — `dind` vs `kubernetes` vs `kubernetes-novolume`

When a workflow uses `container:`, `services:`, or a Docker action, the runner needs a way to start containers. ARC offers three strategies.

## 6.1 At a glance

| Aspect | `dind` | `kubernetes` | `kubernetes-novolume` |
|--------|--------|--------------|-----------------------|
| How job container runs | Docker daemon in **sibling sidecar** | Sibling **K8s Pod** | Sibling K8s Pod |
| Privileged required | ✅ yes | ❌ no | ❌ no (but runs as root) |
| Shared storage between runner & job | emptyDir (same Pod) | **RWX PersistentVolume** | None (uses lifecycle hooks) |
| Setup complexity | Low | Medium (needs storage class) | Low |
| Security | Worst | Best | Good |
| Compatible with all workflows | ✅ | ⚠️ requires `container:` block | ⚠️ same |
| ARC version required | All | All | 0.10.0+ |

## 6.2 Mode `dind` — Docker-in-Docker

```yaml
containerMode:
  type: "dind"
```

### What ARC injects into `template.spec`
- An `init-dind-externals` initContainer that copies the runner externals to a shared volume.
- A `dind` sidecar (image: `docker:dind`) running `dockerd` as **privileged**.
- The `runner` container with `DOCKER_HOST=unix:///var/run/docker.sock`.
- Shared volumes: `work`, `dind-sock`, `dind-externals`.

On Kubernetes ≥ 1.29, `dind` is a **native sidecar** (uses `initContainers` with `restartPolicy: Always`). On older versions it's a regular container.

### When to use
- Quick start / dev clusters.
- Workflows that use plain `docker build`, `docker run`.

### Security caveats
- `dind` requires `securityContext.privileged: true` → equivalent to root on the node.
- **Mitigations:**
  - Pod-Security-Standards: at most `baseline` namespace (`restricted` will reject).
  - Dedicated node pool for runners.
  - NetworkPolicy isolating the runner namespace.
  - Consider `docker:dind-rootless` (still privileged but unprivileged user inside).

## 6.3 Mode `kubernetes`

```yaml
containerMode:
  type: "kubernetes"
  kubernetesModeWorkVolumeClaim:
    accessModes: ["ReadWriteOnce"]
    storageClassName: "managed-premium"     # cluster-specific
    resources:
      requests:
        storage: 5Gi
```

### What ARC injects
- The `runner` container with env:
  - `ACTIONS_RUNNER_CONTAINER_HOOKS=/home/runner/k8s/index.js`
  - `ACTIONS_RUNNER_POD_NAME=<self>` (downward API)
  - `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=true`
- An `ephemeral` PVC (volume `work`) using your `kubernetesModeWorkVolumeClaim`.
- A ServiceAccount with permissions to create Pods, Secrets in the runner namespace.

### How it works
The runner image bundles `runner-container-hooks`. When a step needs a container, the hook calls the **Kubernetes API** to create a sibling Pod in the same namespace, mounting the same PVC so files persist between runner and job.

### Storage requirement
- Needs a **dynamic provisioner** (StorageClass) that supports `ReadWriteOnce`.
- For test clusters: [OpenEBS dynamic-localpv](https://github.com/openebs/dynamic-localpv-provisioner).
- For AKS: `managed-csi`. For EKS: `gp3`. For GKE: `standard-rwo`.

### When to use
- Production clusters where privileged containers are forbidden.
- Multi-tenant environments.
- Workflows that **always** use `container:` blocks.

### Caveat
Workflows without `container:` will **fail** with:
```
Jobs without a job container are forbidden on this runner...
```
Override with env `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=false` (read security note below first).

## 6.4 Mode `kubernetes-novolume`

```yaml
containerMode:
  type: "kubernetes-novolume"
```

Like `kubernetes` mode but no PVC. Job filesystem is transferred between Pods via **container lifecycle hooks** (tar/untar). Ideal when:
- The cluster has no RWX storage class.
- You want local-disk performance.

**Constraint:** Runner container must run as **root** (UID 0) because the lifecycle hooks need to chown files.

## 6.5 Decision tree

```
Are you OK with privileged containers?
├── Yes → Need fastest setup? ───────────► dind
│
└── No  → Do you have shared (RWX) storage?
         ├── Yes → kubernetes (with PVC)
         └── No  → kubernetes-novolume
```

## 6.6 Customizing beyond the modes

You can leave `containerMode` **unset** and write the full `template.spec` yourself. This is what the official docs call "customized container mode" — useful for:
- Rootless dind
- Sysbox runtime
- Kata containers
- Mixing modes per workflow

See ARC docs section "Customizing container modes" for `dind-rootless` example.

Continue to [07-authentication.md](07-authentication.md).
