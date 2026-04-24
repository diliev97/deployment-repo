# Deployment Repository

This repository contains the Helm charts and overlays for all applications managed by ArgoCD.

## Architecture

Each environment in the cluster is presented as a new namespace with the environment name (e.g., `dev`, `prod`). Each environment has its own ApplicationSet which controls how the applications in ArgoCD will be created.

Currently the ApplicationSet is manually applied in the cluster with `kubectl`, however in the future this may be moved to the Terraform configuration that deploys ArgoCD so that Terraform can be used when applying a new ApplicationSet in the cluster.

```
┌─────────────────────────────────────────────────────────────┐
│                    Deployment Repository                     │
│                         (main branch)                        │
├─────────────────────────────────────────────────────────────┤
│  apps/                                                      │
│  ├── simple-app/                                            │
│  │   ├── base/           <- Helm chart                       │
│  │   └── overlays/                                          │
│  │       ├── dev/        <- Dev environment values          │
│  │       └── prod/       <- Prod environment values         │
│  ├── integration-service/                                   │
│  └── simple-app-2/                                          │
├─────────────────────────────────────────────────────────────┤
│  01-dev-appsets.yaml    <- ApplicationSet for dev           │
│  02-prod-appsets.yaml   <- ApplicationSet for prod           │
└─────────────────────────────────────────────────────────────┘
```

## Structure

```
deployment-repo/
├── apps/
│   └── <app-name>/
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
├── 01-dev-appsets.yaml       # ApplicationSet for dev environment
├── 02-prod-appsets.yaml      # ApplicationSet for prod environment
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

### 1. Apply ApplicationSet to Cluster

```bash
kubectl apply -f 01-dev-appsets.yaml -n argocd
kubectl apply -f 02-prod-appsets.yaml -n argocd
```

## CI/CD Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Source Repository                         │
│                   (e.g., client-web)                         │
│                         │                                    │
│                         ▼                                    │
│                  ┌──────────────┐                           │
│                  │  Build Job   │                           │
│                  └──────────────┘                           │
│                         │                                    │
│                         ▼                                    │
│            ┌──────────────────────────┐                     │
│            │ Azure Container Registry │                     │
│            │     (new image pushed)   │                     │
│            └──────────────────────────┘                     │
│                         │                                    │
│                         ▼                                    │
│            ┌──────────────────────────┐                     │
│            │  ArgoCD Image Updater    │                     │
│            │  (tracks registry via    │                     │
│            │   ApplicationSet        │                     │
│            │   annotations)          │                     │
│            └──────────────────────────┘                     │
│                         │                                    │
│                         ▼                                    │
│            ┌──────────────────────────┐                     │
│            │  Deployment Repository   │                     │
│            │  (updates tag in         │                     │
│            │   values.yaml, commits)  │                     │
│            └──────────────────────────┘                     │
│                         │                                    │
│                         ▼                                    │
│            ┌──────────────────────────┐                     │
│            │   ApplicationSet         │                     │
│            │  (auto-sync enabled,      │                     │
│            │   detects commit)        │                     │
│            └──────────────────────────┘                     │
│                         │                                    │
│                         ▼                                    │
│            ┌──────────────────────────┐                     │
│            │    ArgoCD Application    │                     │
│            │  (updates deployment)    │                     │
│            └──────────────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

### Flow Description

1. **Source Repository**: A build job (e.g., in the source repo) creates a Docker image and pushes it to Azure Container Registry

2. **ArgoCD Image Updater**: Monitors the container registry via annotations configured in the ApplicationSet. When it detects a new tag compared to the one currently deployed, it:
   - Connects to this deployment repository
   - Finds the `values.yaml` file for the specific app and environment
   - Updates the image tag
   - Commits and pushes the change

3. **ApplicationSet**: The dev ApplicationSet has auto-sync configured. When it detects the commit, it automatically updates the corresponding ArgoCD Application, which then syncs the new image to the cluster

## Image Updater Configuration

The Image Updater is configured via annotations in the ApplicationSet, which are inherited by every Application created by it:

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: "image=ghcr.io/diliev97/<app-name>"
  argocd-image-updater.argoproj.io/write-back-method: git
  argocd-image-updater.argoproj.io/helm.write-back-target: "../overlays/dev/values.yaml"
  argocd-image-updater.argoproj.io/git-branch: master
  argocd-image-updater.argoproj.io/image-update-strategy: latest
```

- **image-list**: Specifies which image to track
- **write-back-method**: Uses git to write back changes to this repository
- **helm.write-back-target**: Path to the values.yaml file to update
- **git-branch**: Branch to commit changes to
- **image-update-strategy**: Strategy for determining which tag to use (`latest`, `semver`, etc.)

## Prod vs Dev Differences

| Setting | Dev | Prod |
|---------|-----|------|
| Auto-sync | Enabled | Manual only |
| Auto-heal | Enabled | Disabled |
| Replica count | 1 | 2+ |
| Resources | Smaller | Larger |
| Ingress | Disabled | Enabled |

See `overlays/dev/values.yaml` and `overlays/prod/values.yaml` for details.