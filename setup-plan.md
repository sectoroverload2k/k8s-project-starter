# K8s Monorepo CI/CD Pipeline Plan

## Summary

A GitHub Actions CI/CD pipeline for a Kubernetes monorepo supporting multiple services, with semantic versioning and environment-based deployments. Supports hybrid architectures with Kubernetes, server (SSH/rsync), and migration-only deployments.

---

## Key Features

- **Dynamic change detection** — Automatically discovers components from `apps/`, `services/`, `infra/` directories
- **Per-environment targets** — Services can deploy to specific environments (e.g., skip dev, only deploy to staging/prod)
- **Multiple build types** — Docker, npm, yarn, Hugo, zip, or none
- **Multiple deploy types** — Kubernetes, server (SSH/rsync), migrations, or static
- **Smart rebuilds** — Skips builds when only k8s config changed, reuses existing images
- **PostgreSQL migrations** — Built-in migration runner using simple SQL files
- **Configurable runners** — Per-target runner selection (GitHub-hosted or self-hosted)
- **Shared package support** — Docker build context can reference repo root for monorepo packages

---

## Branching & Versioning Strategy

| Branch | Version Format | Example | Build Trigger | Deploy Trigger |
|--------|----------------|---------|---------------|----------------|
| PR to `develop` | `X.Y.Z-beta.BUILD` | `1.2.0-beta.123` | On PR open/update | On merge (auto) |
| PR to `staging` | `X.Y.Z-rc.BUILD` | `1.2.0-rc.124` | On PR open/update | On merge (auto) |
| PR to `main` | `X.Y.Z` | `1.2.0` | On PR open/update | On merge (**manual approval**) |

- Each service has its own `VERSION` file (or `package.json` for Node.js)
- Git tags: `service-name/vX.Y.Z-beta.N`, `service-name/vX.Y.Z-rc.N`, `service-name/vX.Y.Z`
- GitHub Releases created on production deploy with lineage tracking

---

## Repository Structure

```
k8s-project-starter/
├── .github/
│   ├── workflows/
│   │   ├── _build-service.yml          # Reusable: build & push images
│   │   ├── _deploy-service.yml         # Reusable: kustomize + kubectl
│   │   ├── ci-pr.yml                   # Triggered on all PRs (build only)
│   │   ├── cd-develop.yml              # Deploy to dev on merge
│   │   ├── cd-staging.yml              # Deploy to staging on merge
│   │   └── cd-production.yml           # Deploy to prod (manual approval)
│   └── actions/
│       └── detect-changes/
│           └── action.yml              # Path-based change detection
│
├── apps/                               # User-facing applications
│   └── <app-name>/
│       ├── VERSION                     # Base version (1.2.0)
│       ├── Dockerfile
│       ├── src/
│       └── k8s/
│           ├── base/
│           └── overlays/
│               ├── dev/
│               ├── staging/
│               └── prod/
│
├── services/                           # Backend microservices
│   └── <service-name>/
│       ├── VERSION
│       ├── Dockerfile
│       ├── src/
│       └── k8s/
│           ├── base/
│           └── overlays/
│
├── infra/                              # Infrastructure components
│   ├── mysql/
│   │   └── k8s/
│   ├── redis/
│   │   └── k8s/
│   └── shared/                         # Shared configs (namespaces, network policies)
│       └── k8s/
│
├── platform/
│   ├── platform.yaml                   # "App of Apps" - platform version manifest
│   └── compatibility.yaml              # Component dependency matrix
│
├── deploy/
│   ├── docker/                         # Shared Dockerfiles if needed
│   │   └── php.Dockerfile.example
│   └── registry/
│       ├── ghcr.env                    # GHCR config (default)
│       ├── dockerhub.env.example
│       └── ecr.env.example
│
└── scripts/
    ├── version-bump.sh                 # Bump VERSION files
    └── validate-platform.sh            # Validate platform compatibility
```

---

## Platform Manifest (App of Apps)

`platform/platform.yaml` - Defines the platform version and component requirements:

```yaml
platform:
  name: "MyPlatform"
  version: "1.5.0"
  description: "Platform release Q1 2024"

components:
  apps:
    dashboard:
      version: ">=1.3.0"
      required: true
    landing:
      version: ">=1.0.0"
      required: false

  services:
    api:
      version: ">=2.0.0, <3.0.0"
      required: true
    auth:
      version: ">=1.1.0"
      required: true

  infra:
    mysql:
      version: "8.0"
      required: true
    redis:
      version: ">=7.0"
      required: false

compatibility:
  api:
    requires:
      auth: ">=1.1.0"
      mysql: ">=8.0"
  dashboard:
    requires:
      api: ">=2.0.0"
```

This becomes the source of truth for ArgoCD "app of apps" when you migrate.

---

## Workflow Files

### 1. `ci-pr.yml` - PR Builds (All Branches)

```yaml
on:
  pull_request:
    branches: [develop, staging, main]

jobs:
  detect-changes:
    # Uses dorny/paths-filter to detect changed services

  determine-version:
    # Based on target branch:
    # - develop → beta.RUN_NUMBER
    # - staging → rc.RUN_NUMBER
    # - main → (no suffix, production version)

  build:
    # Matrix build for changed services only
    # Push images to GHCR with version tags
    # Create git tags
```

### 2. `cd-develop.yml` - Deploy to Dev

```yaml
on:
  push:
    branches: [develop]

jobs:
  # Detect changes, rebuild, deploy
  # Uses Kustomize overlays for dev
  # Auto-deploys on merge
```

### 3. `cd-staging.yml` - Deploy to Staging

```yaml
on:
  push:
    branches: [staging]

jobs:
  # Detect changes, rebuild with rc tag, deploy
  # Uses Kustomize overlays for staging
  # Auto-deploys on merge
```

### 4. `cd-production.yml` - Deploy to Production

```yaml
on:
  push:
    branches: [main]

jobs:
  build:
    # Build production images

  deploy:
    needs: build
    environment: production  # <-- Manual approval required
    steps:
      # Deploy with Kustomize
      # Create GitHub Release with lineage
```

### 5. `_build-service.yml` - Reusable Build Workflow

Inputs:
- `service`: Service name
- `service-path`: Path to service (apps/dashboard, services/api, etc.)
- `build-type`: docker, npm, yarn, hugo, zip, or none
- `build-output`: Output directory (for npm, yarn, hugo builds)
- `build-context`: Docker context path (default: service-path, use "." for shared packages)
- `version-suffix`: beta.N, rc.N, or empty
- `push`: Whether to push to registry (Docker only)
- `skip-build`: Skip build for deploy-only changes

Outputs:
- `version`: Full version string
- `image`: Full image name with tag (Docker only)
- `artifact-name`: Build artifact name (non-Docker builds)
- `keep-existing-image`: True if deploy should use currently running image

Steps:
1. Read VERSION file
2. Calculate full version
3. Build based on build-type (Docker, npm, yarn, Hugo, zip)
4. Push to registry (Docker) or upload artifact (others)
5. Create git tag

### 6. `_deploy-service.yml` - Reusable Deploy Workflow

Inputs:
- `service`: Service name
- `service-path`: Path to service
- `environment`: dev, staging, prod
- `image-tag`: Image tag to deploy (optional if keep-existing-image)
- `keep-existing-image`: Preserve current image for config-only changes
- `run-migrations`: Run MySQL/Flyway migrations
- `run-pg-migrations`: Run PostgreSQL migrations
- `runner`: Runner to use (string or JSON array for labels)

Steps:
1. Setup kubectl and Kustomize
2. Determine image tag (new version or fetch from running deployment)
3. Run database migrations (MySQL/Flyway and/or PostgreSQL)
4. Substitute secrets in manifests
5. Update image in overlay: `kustomize edit set image`
6. Build manifests: `kustomize build`
7. Apply to cluster: `kubectl apply`
8. Wait for rollout: `kubectl rollout status`

### 7. `_deploy-pg-migrations.yml` - PostgreSQL Migrations

Standalone workflow for PostgreSQL migrations using simple SQL files (not Flyway).

Inputs:
- `service`: Service name
- `service-path`: Path to service
- `environment`: Target environment
- `migrations-path`: Path to migrations (defaults to {service-path}/migrations)

Outputs:
- `applied_count`: Number of migrations applied
- `status`: Migration status (success, skipped, failed)

Features:
- Uses `schema_migrations` tracking table
- Runs migrations in order (sorted by filename)
- Each migration runs in a transaction
- Creates Kubernetes Job with postgres:16-alpine image

---

## Kustomize Structure Per Service

```
apps/dashboard/k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml      # namespace: dev, replicas: 1
    │   └── patch-env.yaml
    ├── staging/
    │   ├── kustomization.yaml      # namespace: staging, replicas: 2
    │   └── patch-env.yaml
    └── prod/
        ├── kustomization.yaml      # namespace: prod, replicas: 3
        ├── patch-env.yaml
        └── hpa.yaml                # Production-only HPA
```

---

## Required GitHub Configuration

### Secrets

| Secret | Description |
|--------|-------------|
| `DEV_KUBECONFIG` | Base64 kubeconfig for dev cluster |
| `STAGING_KUBECONFIG` | Base64 kubeconfig for staging cluster |
| `PROD_KUBECONFIG` | Base64 kubeconfig for prod cluster |

### Environments

| Environment | Protection Rules |
|-------------|-----------------|
| `development` | None (auto-deploy) |
| `staging` | None (auto-deploy) |
| `production` | **Required reviewers**, optional wait timer |

---

## Change Detection

Change detection is **fully automatic** — no hardcoded paths required. The detect-changes action:

1. Uses `git diff` to find changed files
2. Extracts unique component directories (`apps/<name>`, `services/<name>`, `infra/<name>`)
3. Reads each component's `deploy.yaml` for deployment configuration
4. Builds separate matrices for Kubernetes, server, and migration deployments

### Shared Changes Trigger All

Changes to these paths trigger a rebuild of ALL components:
- `infra/shared/**`
- `.github/workflows/_*.yml`
- `platform/platform.yaml`

### Smart Build Detection

The pipeline distinguishes between:
- **Code changes** — Changes outside `k8s/` directory trigger full rebuild
- **Config-only changes** — Changes only in `k8s/` directory skip build, reuse existing image

This enables fast config-only deployments (HPA tuning, resource limits, etc.) without rebuilding images.

---

## Files to Create

### Workflows
1. `.github/workflows/ci-pr.yml` - PR builds for all branches
2. `.github/workflows/cd-develop.yml` - Auto-deploy to dev
3. `.github/workflows/cd-staging.yml` - Auto-deploy to staging
4. `.github/workflows/cd-production.yml` - Deploy to prod (manual approval)
5. `.github/workflows/_build-service.yml` - Reusable build workflow
6. `.github/workflows/_deploy-service.yml` - Reusable Kubernetes deploy workflow
7. `.github/workflows/_deploy-server.yml` - Reusable SSH/rsync deploy workflow
8. `.github/workflows/_deploy-migrations.yml` - Reusable Flyway migrations workflow
9. `.github/workflows/_deploy-pg-migrations.yml` - Reusable PostgreSQL migrations workflow
10. `.github/actions/detect-changes/action.yml` - Change detection composite action

### Platform Config
8. `platform/platform.yaml` - App of apps manifest
9. `platform/compatibility.yaml` - Component dependency matrix

### Scripts
10. `scripts/version-bump.sh` - Bump VERSION files
11. `scripts/validate-platform.sh` - Validate compatibility

### Registry Config
12. `deploy/registry/ghcr.env` - GHCR config (default)
13. `deploy/registry/dockerhub.env.example` - Docker Hub example
14. `deploy/registry/ecr.env.example` - AWS ECR example

### Example Structure (Placeholders)
15. `apps/README.md` - Explains apps folder
16. `services/README.md` - Explains services folder
17. `infra/README.md` - Explains infra folder
18. `infra/shared/k8s/base/namespace.yaml` - Namespace definitions

### Flyway Migrations
19. `infra/mysql/migrations/V001__initial_schema.sql.example` - Example migration
20. `infra/mysql/k8s/base/migration-job.yaml` - Flyway Job template
21. `infra/mysql/k8s/base/kustomization.yaml` - Kustomize for migrations

---

## Verification Plan

1. Create example app with VERSION file and Dockerfile
2. Create Kustomize base + overlays
3. Open PR to develop → verify beta build triggers
4. Merge PR → verify auto-deploy to dev
5. Create PR from develop to staging → verify RC build
6. Merge → verify staging deploy
7. Create PR from staging to main → verify production build
8. Merge → verify approval gate works, then deploy
9. Check GitHub Release is created with lineage

---

## Decisions Finalized

- **Repo structure**: By type (apps/services/infra)
- **Sample services**: Minimal structure with README placeholders
- **Database migrations**: Flyway Community Edition (Kubernetes Job, raw SQL files)
- **Secrets management**: Template files populated from GitHub Secrets during deploy

---

## Secrets Pattern

Secret templates in Kustomize overlays (committed to git):

```yaml
# apps/api/k8s/overlays/dev/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
stringData:
  MYSQL_PASSWORD: "${MYSQL_PASSWORD}"
  JWT_SECRET: "${JWT_SECRET}"
```

During deployment, `envsubst` replaces variables from GitHub secrets:

```yaml
# In _deploy-service.yml
- name: Substitute secrets
  run: |
    envsubst < k8s/overlays/${{ inputs.environment }}/secret.yaml > /tmp/secret.yaml
  env:
    MYSQL_PASSWORD: ${{ secrets.MYSQL_PASSWORD }}
    JWT_SECRET: ${{ secrets.JWT_SECRET }}
```

---

## Deploy Configuration (deploy.yaml)

Each component can have a `deploy.yaml` file to configure build and deployment settings.

### Full Schema

```yaml
# Build configuration
build:
  type: docker           # docker, npm, yarn, hugo, zip, none
  command: ''            # Custom build command (optional)
  output: ''             # Output directory (defaults by type: npm→dist, hugo→public)
  context: ''            # Docker context path (default: service path, use "." for repo root)

# Database configuration
database:
  type: postgresql       # postgresql, mysql
  migrations: true       # Enable automatic migrations

# Per-environment targets
targets:
  dev:
    type: kubernetes     # kubernetes, server, migration, static
    runner: ubuntu-latest  # Runner to use (string or JSON array)
    namespace: develop   # Kubernetes namespace (read from kustomization.yaml)
  staging:
    type: kubernetes
    runner: self-hosted
  prod:
    type: kubernetes
    runner: [self-hosted, linux, production]  # Runner with labels
```

### Per-Environment Targeting

Services don't have to deploy to all environments. If a target is not defined, the service is skipped for that environment:

```yaml
# Only deploy to staging and prod (skip dev)
targets:
  staging:
    type: kubernetes
  prod:
    type: kubernetes
```

### Build Context for Shared Packages

For monorepos with shared packages, set `build.context` to the repo root:

```yaml
# services/api/deploy.yaml
build:
  type: docker
  context: .  # Build from repo root, Dockerfile still in services/api/
```

This allows the Dockerfile to copy from shared directories:
```dockerfile
COPY packages/shared ./packages/shared
COPY services/api ./services/api
```

---

## Database Migrations

### PostgreSQL Migrations (Native)

For PostgreSQL, the pipeline includes a native migration runner (no Flyway required):

- SQL files in `{service}/migrations/` directory
- Uses `schema_migrations` tracking table
- Runs as Kubernetes Job with `postgres:16-alpine`

Enable via `deploy.yaml`:
```yaml
database:
  type: postgresql
  migrations: true
```

Migration naming: `{timestamp}_{description}.sql` or `V{NNN}__{description}.sql`

### MySQL Migrations with Flyway

**Tool**: Flyway Community Edition (free, forward migrations)

### Migration File Structure

```
infra/mysql/migrations/
├── V001__create_users_table.sql
├── V002__create_orders_table.sql
├── V003__add_user_email_index.sql
└── U001__create_users_table.sql    # Optional undo (run manually if needed)
```

**Naming convention**: `V{version}__{description}.sql`
- Version: Zero-padded number (001, 002, ..., 999) for clean directory listings
- Double underscore between version and description
- Description uses underscores for spaces

Note: Flyway sorts numerically regardless, but zero-padding keeps `ls` output in order.

### Flyway Kubernetes Job

```yaml
# infra/mysql/k8s/base/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: flyway-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync  # Ready for ArgoCD
spec:
  template:
    spec:
      containers:
        - name: flyway
          image: flyway/flyway:10
          args: ["migrate"]
          env:
            - name: FLYWAY_URL
              value: "jdbc:mysql://${MYSQL_HOST}:3306/${MYSQL_DATABASE}"
            - name: FLYWAY_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: username
            - name: FLYWAY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: password
          volumeMounts:
            - name: migrations
              mountPath: /flyway/sql
      volumes:
        - name: migrations
          configMap:
            name: flyway-migrations
      restartPolicy: Never
  backoffLimit: 3
```

### CI/CD Integration

In `_deploy-service.yml`:
1. Package SQL files into ConfigMap
2. Apply migration Job
3. Wait for Job completion: `kubectl wait --for=condition=complete job/flyway-migrate`
4. Deploy application
5. Clean up Job

### Rollback (Manual)

For the rare rollback, run undo scripts directly:
```bash
kubectl exec -it mysql-pod -- mysql -u root -p database < U003__add_user_email_index.sql
```
