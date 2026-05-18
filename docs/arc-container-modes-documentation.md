# ARC Runner Scale Set Container Modes: Normal, DinD, Kubernetes, and Kubernetes No-Volume

## 1. Purpose of this document

This document explains the container execution modes used with GitHub Actions Runner Controller (ARC) runner scale sets.

It covers:

- What each mode means
- How each mode works
- Where the runner pod image is configured
- Where the child job pod image is configured
- How Docker-based jobs behave
- How `container:` jobs behave
- How Artifactory/private registry usage affects image pulling
- YAML examples for Helm values and GitHub Actions workflows

---

## 2. Core mental model

ARC runs GitHub Actions self-hosted runners inside Kubernetes.

The most important distinction is:

```text
Runner pod image  = configured in ARC Helm values
Job container image = configured in GitHub Actions workflow YAML
```

Example:

```yaml
# values.yaml
spec:
  containers:
    - name: runner
      image: artifactory.company.com/runner/custom-arc-runner:latest
```

This controls the **runner pod image**.

```yaml
# .github/workflows/build.yml
jobs:
  build:
    runs-on: arc-runner-set
    container:
      image: artifactory.company.com/docker-virtual/node:20
```

This controls the **child job pod image** in Kubernetes mode.

---

## 3. Basic ARC job flow

For a normal ephemeral runner setup:

```text
Workflow triggered
   ↓
GitHub creates job
   ↓
ARC listener receives Job Available message
   ↓
ARC scales runner pod
   ↓
Runner pod registers with GitHub
   ↓
Runner pod picks the job
   ↓
Job runs
   ↓
Runner finishes job
   ↓
Runner pod deregisters and is deleted
```

Important:

```text
1 GitHub Actions workflow can have many jobs.
1 job usually maps to 1 ephemeral runner pod.
```

Example:

```yaml
jobs:
  build:
    runs-on: arc-runner-set

  test:
    runs-on: arc-runner-set

  deploy:
    runs-on: arc-runner-set
```

This can create three separate runner pods:

```text
build job  → runner pod 1
test job   → runner pod 2
deploy job → runner pod 3
```

---

## 4. Quick comparison table

| Mode | Runner pod | Child pod? | Docker daemon? | Shared `_work` volume? | Best use case |
|---|---|---:|---:|---:|---|
| Normal runner | Runner container only | No | No | No special shared PVC | Shell commands, Terraform, Azure CLI, kubectl, Helm |
| `dind` | Runner + Docker sidecar | Usually no | Yes, Docker-in-Docker sidecar | Uses pod volume | `docker build`, `docker run`, Docker-based actions |
| `kubernetes` | Runner container | Yes | No Docker daemon required for job container | Yes, PVC | `container:` jobs and service containers as Kubernetes pods |
| `kubernetes-novolume` | Runner container | Yes | No Docker daemon required for job container | No PVC | `container:` jobs without shared PVC complexity |

---

# 5. Normal runner mode

## Type

No special `containerMode`.

## What it is

The job runs directly inside the runner container.

```text
runner pod
 └── runner container
```

## How it works

The runner pod starts, registers to GitHub, picks a job, executes the steps inside the runner container, then gets deleted after the job completes.

## Best for

Use normal runner mode for jobs like:

```text
terraform init / plan / apply
az cli commands
kubectl commands
helm commands
python scripts
PowerShell scripts
basic build commands where tools are already installed in the runner image
```

## Helm values example

```yaml
githubConfigUrl: "https://github.com/your-org"
githubConfigSecret: arc-github-secret
runnerScaleSetName: "arc-normal-runner"
runnerGroup: "Default"

minRunners: 0
maxRunners: 10

template:
  spec:
    containers:
      - name: runner
        image: artifactory.company.com/runner/custom-arc-runner:latest
        command: ["/home/runner/run.sh"]
        imagePullPolicy: Always
    imagePullSecrets:
      - name: artifactory-creds
```

## GitHub Actions workflow example

```yaml
name: Terraform Plan

on:
  workflow_dispatch:

jobs:
  terraform-plan:
    runs-on: arc-normal-runner

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Terraform init
        run: terraform init

      - name: Terraform plan
        run: terraform plan
```

## How this workflow works

```text
1. Workflow is manually triggered.
2. GitHub creates one job: terraform-plan.
3. ARC creates one runner pod using the custom runner image from Helm values.
4. The job runs directly inside the runner container.
5. The runner image must already contain terraform, or the workflow must install it.
6. After the job completes, the runner pod is deleted.
```

## Important point

There is no child pod here.

```text
GitHub job → runner pod → runner container executes steps
```

---

# 6. `type: dind`

## Type

```yaml
containerMode:
  type: "dind"
```

## What it is

`dind` means Docker-in-Docker.

The runner pod contains:

```text
runner pod
 ├── runner container
 └── docker:dind sidecar container
```

The runner container uses the Docker daemon from the sidecar.

## How it works

When a workflow runs Docker commands like:

```bash
docker build -t myapp:latest .
docker run myapp:latest
```

The Docker CLI inside the runner container talks to the Docker daemon running in the `docker:dind` sidecar.

Conceptually:

```text
runner container
   ↓ docker CLI
Docker daemon sidecar
   ↓
builds/runs containers
```

## Best for

Use `dind` when workflows need:

```text
docker build
docker run
docker compose
Docker-based actions
local image build and test
```

## Helm values example

```yaml
githubConfigUrl: "https://github.com/your-org"
githubConfigSecret: arc-github-secret
runnerScaleSetName: "arc-dind-runner"
runnerGroup: "Default"

minRunners: 0
maxRunners: 10

containerMode:
  type: "dind"

template:
  spec:
    containers:
      - name: runner
        image: artifactory.company.com/runner/custom-arc-runner-docker:latest
        command: ["/home/runner/run.sh"]
        imagePullPolicy: Always
    imagePullSecrets:
      - name: artifactory-creds
```

## GitHub Actions workflow example

```yaml
name: Docker Build

on:
  push:
    branches:
      - main

jobs:
  docker-build:
    runs-on: arc-dind-runner

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t artifactory.company.com/apps/myapp:${{ github.sha }} .

      - name: Push Docker image
        run: docker push artifactory.company.com/apps/myapp:${{ github.sha }}
```

## How this workflow works

```text
1. Code is pushed to main.
2. GitHub creates one job: docker-build.
3. ARC creates one runner pod.
4. Because containerMode is dind, the runner pod includes:
   - runner container
   - docker:dind sidecar container
5. The checkout step downloads the repo into the runner workspace.
6. docker build runs from the runner container.
7. The Docker command talks to the dind sidecar.
8. The image is built inside the dind Docker daemon.
9. docker push sends the image to Artifactory.
10. After the job completes, the runner pod is deleted.
```

## Important point

The workflow image build target is not the same as the runner image.

```text
Runner image = artifactory.company.com/runner/custom-arc-runner-docker:latest
Built app image = artifactory.company.com/apps/myapp:<sha>
```

## Security note

`dind` usually needs privileged mode because the Docker daemon runs inside the pod.

---

# 7. `type: kubernetes` volume mode

## Type

```yaml
containerMode:
  type: "kubernetes"
```

## What it is

Kubernetes mode uses runner container hooks to create a **new child pod** for container jobs, service containers, or Docker container actions.

Structure:

```text
runner pod
 └── runner container
       ↓ creates
child job pod
 └── job container, for example node:20
```

## Why create a child pod?

Because the workflow may say:

```yaml
container:
  image: node:20
```

This means:

```text
Run this GitHub job inside a node:20 container.
```

Instead of using Docker to run `node:20`, Kubernetes mode asks Kubernetes to create a pod using that image.

So:

```text
dind mode       = use Docker daemon to run containers
kubernetes mode = use Kubernetes pods to run containers
```

## Shared `_work` PVC

In volume mode, the runner pod and child job pod share the GitHub workspace through a PVC.

```text
runner pod <── shared _work PVC ──> child job pod
```

The shared workspace is usually:

```text
/home/runner/_work
```

This is needed because the job pod needs access to checked-out source code and files created during the job.

## Best for

Use `kubernetes` mode when workflows use:

```yaml
container:
  image: some-image
```

or service containers:

```yaml
services:
  postgres:
    image: postgres:16
```

This mode is better when you want Kubernetes-native pod execution instead of privileged Docker-in-Docker.

## Helm values example

```yaml
githubConfigUrl: "https://github.com/your-org"
githubConfigSecret: arc-github-secret
runnerScaleSetName: "arc-k8s-runner"
runnerGroup: "Default"

minRunners: 0
maxRunners: 10

containerMode:
  type: "kubernetes"
  kubernetesModeWorkVolumeClaim:
    accessModes:
      - ReadWriteOnce
    storageClassName: "managed-csi"
    resources:
      requests:
        storage: 5Gi

template:
  spec:
    containers:
      - name: runner
        image: artifactory.company.com/runner/custom-arc-runner-k8s:latest
        command: ["/home/runner/run.sh"]
        imagePullPolicy: Always
    imagePullSecrets:
      - name: artifactory-creds
```

## GitHub Actions workflow example

```yaml
name: Node Test in Container

on:
  pull_request:

jobs:
  node-test:
    runs-on: arc-k8s-runner

    container:
      image: artifactory.company.com/docker-virtual/node:20

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

## How this workflow works

```text
1. Pull request is opened or updated.
2. GitHub creates one job: node-test.
3. ARC creates one runner pod using the runner image from Helm values.
4. The runner pod receives the job.
5. The runner sees this in the workflow:

   container:
     image: artifactory.company.com/docker-virtual/node:20

6. The runner uses Kubernetes container hooks.
7. The hook creates a child job pod in the same namespace.
8. The child pod image is node:20 from Artifactory.
9. Kubernetes pulls the node:20 image from Artifactory.
10. The shared PVC is mounted between runner pod and child pod.
11. actions/checkout places code in the shared workspace.
12. npm ci and npm test run inside the node:20 child job container.
13. Child pod is deleted after the job.
14. Runner pod is deleted after the job.
```

## Important point about image pulling

If the workflow says:

```yaml
container:
  image: node:20
```

Kubernetes interprets that as:

```text
docker.io/library/node:20
```

In a restricted enterprise network, that may fail because the cluster cannot pull from Docker Hub.

So in your setup, use Artifactory:

```yaml
container:
  image: artifactory.company.com/docker-virtual/node:20
```

## Service container example

```yaml
name: Java Integration Test

on:
  pull_request:

jobs:
  integration-test:
    runs-on: arc-k8s-runner

    container:
      image: artifactory.company.com/docker-virtual/maven:3.9-eclipse-temurin-17

    services:
      postgres:
        image: artifactory.company.com/docker-virtual/postgres:16
        env:
          POSTGRES_PASSWORD: password

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run integration tests
        run: mvn test
```

## How this service-container workflow works

```text
1. ARC creates one runner pod.
2. The runner sees a job container: maven.
3. The runner also sees a service container: postgres.
4. Kubernetes container hooks create pod resources for the job/service container setup.
5. Maven runs the job commands.
6. PostgreSQL runs as a service container reachable by the job.
7. Images are pulled from Artifactory.
8. The shared PVC provides the workspace between the runner and job pod.
9. After the job completes, child pod resources are cleaned up.
10. The runner pod is deleted.
```

---

# 8. `type: kubernetes-novolume`

## Type

```yaml
containerMode:
  type: "kubernetes-novolume"
```

## What it is

This is also Kubernetes mode, but without the shared `_work` PVC.

Structure:

```text
runner pod
 └── runner container
       ↓ creates
child job pod
 └── job container
```

But there is no shared PVC:

```text
runner pod      child job pod
       no shared _work PVC
```

## How it works

Instead of relying on a shared persistent volume, no-volume mode uses container lifecycle hook behavior to restore/export job filesystem data between pods.

Conceptually:

```text
kubernetes volume mode:
runner pod <── shared PVC ──> child job pod

kubernetes no-volume mode:
runner pod ── lifecycle hook filesystem handling ── child job pod
```

## What is `ACTIONS_RUNNER_IMAGE`?

In no-volume/container-hook setups, `ACTIONS_RUNNER_IMAGE` is related to the runner image used by the hook/bootstrap process.

Simple meaning:

```text
ACTIONS_RUNNER_IMAGE tells the hook what runner image should be used/referenceable during no-volume job execution.
```

This is separate from the workflow job container image.

Example:

```text
ACTIONS_RUNNER_IMAGE = artifactory.company.com/runner/custom-arc-runner-k8s-novolume:latest

workflow container image = artifactory.company.com/docker-virtual/node:20
```

They are not the same thing.

## Best for

Use `kubernetes-novolume` when:

```text
You want Kubernetes child pod execution
You do not want to manage RWX/shared PVC complexity
Your cluster has limited shared storage support
You want to avoid storage class / access mode / PVC mount issues
```

## Important root note

No-volume mode may require the container to run as root for lifecycle hook operations, depending on the ARC/version behavior and configuration.

## Helm values example

```yaml
githubConfigUrl: "https://github.com/your-org"
githubConfigSecret: arc-github-secret
runnerScaleSetName: "arc-k8s-novolume-runner"
runnerGroup: "Default"

minRunners: 0
maxRunners: 10

containerMode:
  type: "kubernetes-novolume"

template:
  spec:
    containers:
      - name: runner
        image: artifactory.company.com/runner/custom-arc-runner-k8s-novolume:latest
        command: ["/home/runner/run.sh"]
        imagePullPolicy: Always
        env:
          - name: ACTIONS_RUNNER_IMAGE
            value: "artifactory.company.com/runner/custom-arc-runner-k8s-novolume:latest"
    imagePullSecrets:
      - name: artifactory-creds
```

## GitHub Actions workflow example

```yaml
name: Node Test No Volume

on:
  pull_request:

jobs:
  node-test:
    runs-on: arc-k8s-novolume-runner

    container:
      image: artifactory.company.com/docker-virtual/node:20

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

## How this workflow works

```text
1. Pull request triggers the workflow.
2. GitHub creates one job: node-test.
3. ARC creates one runner pod using the no-volume runner image from Helm values.
4. The runner sees the workflow has:

   container:
     image: artifactory.company.com/docker-virtual/node:20

5. The runner uses Kubernetes container hooks.
6. The hook creates a child pod using the node:20 image.
7. The child pod image is pulled from Artifactory.
8. There is no shared _work PVC.
9. Lifecycle hook behavior handles the job filesystem movement/restore/export.
10. npm ci and npm test run inside the child job pod.
11. Child pod is deleted.
12. Runner pod is deleted.
```

## Important point

`kubernetes-novolume` is not the same as normal runner mode.

```text
Normal runner:
job runs inside runner container, no child pod

kubernetes-novolume:
runner creates child pod, but without shared PVC
```

---

# 9. Where images are configured

## Runner pod image

Configured in Helm values:

```yaml
template:
  spec:
    containers:
      - name: runner
        image: artifactory.company.com/runner/custom-runner:latest
```

This image should contain:

```text
GitHub runner binary
runner startup script
required tools such as az, kubectl, helm, terraform
Docker CLI if using dind
runner container hooks if using kubernetes/kubernetes-novolume mode
company CA certificates if required
```

## Child pod image

Configured in GitHub Actions workflow YAML:

```yaml
container:
  image: artifactory.company.com/docker-virtual/node:20
```

This image should contain the job runtime/tooling:

```text
node/npm
maven/java
python/pip
postgres/mysql service image
any other runtime required by the job
```

## Service container image

Also configured in GitHub Actions workflow YAML:

```yaml
services:
  postgres:
    image: artifactory.company.com/docker-virtual/postgres:16
```

---

# 10. Artifactory/private registry setup

If your AKS cluster has restricted internet access, avoid public image names like:

```yaml
container:
  image: node:20
```

Because Kubernetes will try:

```text
docker.io/library/node:20
```

Use your Artifactory path instead:

```yaml
container:
  image: artifactory.company.com/docker-virtual/node:20
```

Same rule applies for service containers:

```yaml
services:
  postgres:
    image: artifactory.company.com/docker-virtual/postgres:16
```

And runner images:

```yaml
template:
  spec:
    containers:
      - name: runner
        image: artifactory.company.com/runner/custom-arc-runner:latest
```

## ImagePullSecret example

```bash
kubectl create secret docker-registry artifactory-creds \
  --docker-server=artifactory.company.com \
  --docker-username=<username> \
  --docker-password=<password> \
  --namespace=arc-runners
```

Use it in Helm values:

```yaml
template:
  spec:
    imagePullSecrets:
      - name: artifactory-creds
```

---

# 11. Which mode should you choose?

## Use normal runner when

```text
You only run shell commands.
You use Terraform, Azure CLI, kubectl, Helm, PowerShell, Python.
You do not need workflow-level `container:`.
You do not need `docker build`.
```

## Use `dind` when

```text
You need docker build.
You need docker run.
You need Docker Compose.
You use Docker-based actions.
You accept privileged pod requirements.
```

## Use `kubernetes` when

```text
Your workflows use container: image.
Your workflows use service containers.
You want Kubernetes-native child pods.
You can provide a proper shared PVC/storage class.
```

## Use `kubernetes-novolume` when

```text
Your workflows use container: image.
You want Kubernetes-native child pods.
You want to avoid shared PVC complexity.
Your runner image supports the required no-volume hook behavior.
```

---

# 12. Final simple summary

```text
Normal runner
= Job runs directly inside runner container.

DinD
= Job runs in runner pod, Docker commands use docker:dind sidecar.

Kubernetes
= Runner creates child pod for job container, with shared _work PVC.

Kubernetes-novolume
= Runner creates child pod for job container, without shared _work PVC.
```

Most important separation:

```text
Helm values control the runner pod image.
GitHub Actions workflow YAML controls the child job pod image.
```

Example:

```yaml
# Helm values
runner image: artifactory.company.com/runner/custom-arc-runner:latest
```

```yaml
# GitHub Actions workflow
container image: artifactory.company.com/docker-virtual/node:20
```

Meaning:

```text
Runner pod uses custom-arc-runner.
Child job pod uses node:20.
Both should come from Artifactory if internet egress is restricted.
```

---

# 13. Official references

- GitHub Docs - Actions Runner Controller overview: https://docs.github.com/en/actions/concepts/runners/actions-runner-controller
- GitHub Docs - Deploying runner scale sets with ARC: https://docs.github.com/en/actions/how-tos/manage-runners/use-actions-runner-controller/deploy-runner-scale-sets
- GitHub Docs - Docker-in-Docker and Kubernetes mode for containers: https://docs.github.com/en/actions/how-tos/manage-runners/use-actions-runner-controller/deploy-runner-scale-sets#using-docker-in-docker-or-kubernetes-mode-for-containers
- GitHub Actions Runner Controller repository: https://github.com/actions/actions-runner-controller
