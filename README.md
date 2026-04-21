# Deployment Repository

This repository contains the Helm charts and overlays for all applications managed by ArgoCD.

## Architecture

Each Kubernetes cluster runs its own ArgoCD instance. The deployment repo is shared across all clusters.

```
┌─────────────────────────────────────────────────────────────┐
│                    Deployment Repository                     │
│                         (main branch)                        │
├─────────────────────────────────────────────────────────────┤
│  apps/                                                      │
│  ├── simple-app/                                            │
│  │   ├── base/           <- Helm chart                       │
│  │   └── overlays/                                          │
│  │       ├── dev/        <- Dev cluster values              │
│  │       └── prod/       <- Prod cluster values             │
│  ├── dev/               <- Dev ApplicationSet              │
│  └── prod/              <- Prod ApplicationSet              │
└─────────────────────────────────────────────────────────────┘
          │                                   │
          ▼                                   ▼
   Dev ArgoCD                       Prod ArgoCD (future)
   ┌─────────────┐                 ┌─────────────┐
   │ AppSet: dev │                 │ AppSet: prod│
   └─────────────┘                 └─────────────┘
```

## Structure

```
deployment-repo/
├── apps/
│   └── simple-app/
│       ├── base/
│       │   ├── Chart.yaml
│       │   ├── values.yaml
│       │   └── templates/
│       │       ├── deployment.yaml
│       │       ├── service.yaml
│       │       └── _helpers.tpl
│       └── overlays/
│           ├── dev/values.yaml
│           └── prod/values.yaml
├── 01-dev-appsets.yaml       # ApplicationSet for dev cluster
├── 02-prod-appsets.yaml      # ApplicationSet for prod cluster
└── README.md
```

## Adding a New Application

1. Create the directory structure under `apps/`:
   ```bash
   mkdir -p apps/my-app/base/templates
   mkdir -p apps/my-app/overlays/dev
   mkdir -p apps/my-app/overlays/prod
   ```

2. Copy base files from `simple-app/base/` to `apps/my-app/base/`

3. Update `Chart.yaml` with the app name

4. Create overlay values:
   - `apps/my-app/overlays/dev/values.yaml`
   - `apps/my-app/overlays/prod/values.yaml`

5. Commit and push - the ApplicationSet will automatically create the ArgoCD Application

## Setup Steps

### 1. Dev Cluster (current)

```bash
kubectl apply -f 01-dev-appsets.yaml -n argocd
```

### 2. Prod Cluster (future)

```bash
kubectl apply -f 02-prod-appsets.yaml -n argocd
```

## Image Updater

The `argocd-image-updater` monitors container registries and automatically updates the image tag when a new image is pushed.

Configuration is in `base/values.yaml`:
```yaml
podAnnotations:
  argocd-image-updater.argoproj.io/image-list: image=ghcr.io/yourorg/simple-app
  argocd-image-updater.argoproj.io/write-back-method: git
```

## CI/CD Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ dev branch   │────▶│   CI Build   │────▶│  GHCR Push   │
└──────────────┘     └──────────────┘     └──────────────┘
                                                  │
                                                  ▼
                                         ┌──────────────┐
                                         │ ImageUpdater │
                                         └──────────────┘
                                                  │
                                                  ▼
                                         ┌──────────────┐
                                         │ ArgoCD Sync  │
                                         └──────────────┘
```

1. **Dev**: Push to `dev` branch → CI builds → pushes to registry → Image Updater updates tag → ArgoCD syncs
2. **Prod**: Release (tag or manual) → manually update prod values or use semver pattern

## Prod vs Dev Differences

| Setting | Dev | Prod |
|---------|-----|------|
| Auto-sync | Enabled | Manual only |
| Auto-heal | Enabled | Disabled |
| Replica count | 1 | 2+ |
| Resources | Smaller | Larger |
| Ingress | Disabled | Enabled |

See `overlays/dev/values.yaml` and `overlays/prod/values.yaml` for details.