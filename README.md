# Actions Runner Helm Charts

Helm charts for deploying GitHub Actions self-hosted runners on Kubernetes.

## Charts

| Chart | Description |
|-------|-------------|
| [arc-runner-stack](./charts/arc-runner-stack) | Complete ARC stack with ESO & cert-manager |

## Usage

```bash
# Login to GHCR
helm registry login ghcr.io -u YOUR_GITHUB_USERNAME

# Install chart
helm install arc-stack oci://ghcr.io/bkcloudops/charts/arc-runner-stack \
  --namespace arc-systems \
  --create-namespace \
  --values values.yaml
```

## Publishing

Charts are automatically published to GHCR on:
- Push to `main` branch (with changes in `charts/`)
- Git tags matching `v*`
- Manual workflow dispatch

## Architecture

```
actions-runner-helm/
├── charts/
│   └── arc-runner-stack/      # Main umbrella chart
│       ├── Chart.yaml         # Dependencies: ARC, ESO, cert-manager
│       ├── values.yaml        # Default values
│       └── templates/
│           ├── namespaces.yaml
│           ├── cluster-secret-store.yaml
│           ├── external-secrets.yaml
│           ├── service-account.yaml
│           └── runner-deployment.yaml
└── .github/
    └── workflows/
        └── publish-chart.yml  # GHCR publishing
```

## License

MIT
