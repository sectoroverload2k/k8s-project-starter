# K8s Monorepo Starter

A Kubernetes monorepo with GitHub Actions CI/CD, supporting multiple services with semantic versioning and environment-based deployments.

## Repository Structure

```
k8s-project-starter/
├── .github/
│   ├── workflows/
│   │   ├── ci-pr.yml               # Build on PR (all branches)
│   │   ├── cd-develop.yml          # Auto-deploy to dev
│   │   ├── cd-staging.yml          # Auto-deploy to staging
│   │   ├── cd-production.yml       # Deploy to prod (manual approval)
│   │   ├── _build-service.yml      # Reusable: build services
│   │   ├── _deploy-service.yml     # Reusable: Kubernetes deploy
│   │   ├── _deploy-server.yml      # Reusable: SSH/rsync deploy
│   │   ├── _deploy-migrations.yml  # Reusable: Flyway migrations
│   │   └── _deploy-pg-migrations.yml # Reusable: PostgreSQL migrations
│   └── actions/
│       └── detect-changes/         # Dynamic change detection
│
├── apps/                          # User-facing applications
│   └── <app-name>/
│       ├── VERSION
│       ├── Dockerfile
│       ├── src/
│       └── k8s/
│           ├── base/
│           └── overlays/{dev,staging,prod}/
│
├── services/                      # Backend microservices
│   └── <service-name>/
│       ├── VERSION
│       ├── Dockerfile
│       ├── src/
│       └── k8s/
│           ├── base/
│           └── overlays/{dev,staging,prod}/
│
├── infra/                         # Infrastructure components
│   ├── <component>/               # e.g., email, redis
│   │   ├── deploy.yaml            # Deployment configuration
│   │   └── k8s/                   # Kubernetes manifests
│   └── shared/                    # Shared resources, network policies
│
├── contracts/                     # API & event contracts
│   ├── api/                       # OpenAPI specifications
│   └── events/                    # Event schemas (JSON Schema)
│
├── platform/
│   ├── platform.yaml              # App-of-apps manifest
│   └── compatibility.yaml         # Component dependency matrix
│
├── deploy/
│   └── registry/                  # Container registry configs
│
└── scripts/
    ├── setup-repo.sh              # Setup after forking
    ├── version-bump.sh            # Bump VERSION files
    └── validate-platform.sh       # Validate compatibility
```

## Branching Strategy

### Branch Purposes

| Branch | Purpose | Deploys To |
|--------|---------|------------|
| `main` | Production-ready code | Production (manual approval) |
| `staging` | Pre-production testing | Staging (auto) |
| `develop` | Integration branch | Development (auto) |
| `feature/*` | New features | — (PR only) |
| `fix/*` | Bug fixes | — (PR only) |
| `hotfix/*` | Production hotfixes | — (PR only) |

### Branch Flow

```
feature/api/add-users ──┐
                        ├──► develop ──► staging ──► main
feature/ui/new-dashboard┘        │          │         │
                                 ▼          ▼         ▼
                               [dev]    [staging]  [prod]
                               (auto)    (auto)   (manual)
```

### Branch Naming Convention

All branches must follow this pattern:

```
<type>/<service>/<description>
```

**Types:**
- `feature/` — New functionality
- `fix/` — Bug fixes
- `hotfix/` — Urgent production fixes
- `refactor/` — Code improvements (no behavior change)
- `contract/` — API contract changes (OpenAPI specs, event schemas)
- `docs/` — Documentation only
- `test/` — Test additions or fixes

**Examples:**
```bash
feature/api/add-user-authentication
feature/dashboard/implement-dark-mode
fix/auth/token-expiration-bug
hotfix/api/null-pointer-crash
refactor/api/cleanup-database-queries
contract/api/users-endpoint              # API contract before implementation
docs/readme/update-setup-instructions
```

**Rules:**
- Use lowercase with hyphens (kebab-case)
- Keep descriptions concise but meaningful
- Service name should match directory name (`api`, `dashboard`, `auth`, etc.)
- For cross-service changes, use `platform` as service: `feature/platform/upgrade-node-version`

## Versioning Strategy

### Version Format

Each service maintains its own version in a `VERSION` file (or `package.json`).

| Environment | Version Format | Example | Git Tag |
|-------------|----------------|---------|---------|
| Development | `X.Y.Z-beta.BUILD` | `1.2.0-beta.45` | `api/v1.2.0-beta.45` |
| Staging | `X.Y.Z-rc.BUILD` | `1.2.0-rc.46` | `api/v1.2.0-rc.46` |
| Production | `X.Y.Z` | `1.2.0` | `api/v1.2.0` |

- `X` (major) — Breaking changes
- `Y` (minor) — New features, backwards compatible
- `Z` (patch) — Bug fixes
- `BUILD` — GitHub Actions run number (auto-incremented)

### Version Bumping

Use the version bump script before creating a PR:

```bash
# Bump patch version (1.0.0 → 1.0.1)
./scripts/version-bump.sh services/api patch

# Bump minor version (1.0.1 → 1.1.0)
./scripts/version-bump.sh services/api minor

# Bump major version (1.1.0 → 2.0.0)
./scripts/version-bump.sh services/api major
```

### When to Bump Versions

| Change Type | Version Bump |
|-------------|--------------|
| Bug fix | Patch |
| New feature (backwards compatible) | Minor |
| Breaking API change | Major |
| Dependency updates (minor) | Patch |
| Dependency updates (major) | Minor or Major |

## CI/CD Pipeline

### Pull Request Flow

1. Open PR targeting `develop`, `staging`, or `main`
2. CI detects which services changed
3. Changed services are built with environment-appropriate version suffix
4. Images pushed to container registry
5. Git tags created

### Deployment Flow

```
PR Merged to develop
        │
        ▼
┌─────────────────────┐
│  cd-develop.yml     │
│  • Build beta.N     │
│  • Deploy to dev    │
│  • Auto (no gate)   │
└─────────────────────┘
        │
        ▼ (PR to staging)
┌─────────────────────┐
│  cd-staging.yml     │
│  • Build rc.N       │
│  • Deploy to staging│
│  • Auto (no gate)   │
└─────────────────────┘
        │
        ▼ (PR to main)
┌─────────────────────┐
│  cd-production.yml  │
│  • Build X.Y.Z      │
│  • Manual approval  │◄── Required reviewers
│  • Deploy to prod   │
│  • Create Release   │
└─────────────────────┘
```

### Change Detection

Only services with changes are built and deployed. Change detection is **automatic** — no hardcoded paths required. The pipeline:

1. Uses `git diff` to find changed files
2. Extracts unique component directories (`apps/<name>`, `services/<name>`, `infra/<name>`)
3. Reads each component's `deploy.yaml` for deployment configuration

**Shared changes trigger ALL services:**
- `infra/shared/**` → rebuilds ALL services
- `.github/workflows/_*.yml` → rebuilds ALL services
- `platform/platform.yaml` → rebuilds ALL services

### Smart Rebuilds

The pipeline distinguishes between code and config changes:

| Change Type | Build | Deploy |
|-------------|-------|--------|
| Code changed (`src/`, `Dockerfile`, etc.) | Full rebuild | New image |
| Config only (`k8s/` directory) | Skipped | Reuse existing image |

This enables fast config-only deployments (HPA tuning, resource limits, replica counts) without rebuilding images.

## Parallel Development

This repo supports multiple developers (or bots) working on different services simultaneously.

### How It Works

```
Developer A                    Developer B
    │                              │
    ├─ branch: feature/api/xyz     ├─ branch: feature/dashboard/abc
    ├─ edits services/api/*        ├─ edits apps/dashboard/*
    └─ PR → develop                └─ PR → develop
            │                              │
            ▼                              ▼
    ┌───────────────────────────────────────────┐
    │  CI builds only affected service per PR   │
    │  No conflicts, independent pipelines      │
    └───────────────────────────────────────────┘
```

### Avoiding Conflicts

| Scenario | Recommendation |
|----------|----------------|
| Same service, different files | Usually fine, git merges automatically |
| Same service, same file | Coordinate before starting |
| Different services | No coordination needed |
| Shared files (platform.yaml, etc.) | Assign ownership or coordinate |

### Dependency Coordination

If work has dependencies (e.g., frontend needs new API):

1. **Define contract first** — Agree on API shape before coding
2. **API merges first** — Backend changes deploy before frontend
3. **Use feature flags** — Frontend can merge with flag off until API is ready

### Contract-First Development

For parallel development with dependencies, use the `contracts/` directory:

```
┌────────────────────────────────────────────────────────────────┐
│  Step 1: Backend creates OpenAPI spec                         │
│          PR: contract/api/users-endpoint                      │
│          Merges to develop                                    │
└────────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
┌────────────────────────┐          ┌────────────────────────┐
│  Backend Dev           │          │  Frontend Dev          │
│  Implements endpoints  │          │  Generates API client  │
│  from contract         │          │  from contract         │
│                        │          │  Mocks responses       │
│  PR: feature/api/      │          │  PR: feature/          │
│      users-impl        │          │      dashboard/users   │
└────────────────────────┘          └────────────────────────┘
            │                                   │
            └─────────────────┬─────────────────┘
                              ▼
┌────────────────────────────────────────────────────────────────┐
│  Both merge independently - compatible because same contract   │
└────────────────────────────────────────────────────────────────┘
```

See [`contracts/README.md`](contracts/README.md) for details on:
- Creating OpenAPI specifications
- Generating typed API clients
- Mocking API responses during development
- Event schema definitions

## Deployment Types

This pipeline supports hybrid architectures with multiple deployment types:

| Type | Use Case | Method |
|------|----------|--------|
| `kubernetes` | Containerized services | Kustomize + kubectl |
| `server` | Traditional servers (Apache/PHP, etc.) | SSH + rsync |
| `migration` | Managed databases (RDS, Cloud SQL) | Flyway only |
| `static` | Static sites | S3 + CloudFront (future) |

### Configuring Deployment Type

Each component specifies its deployment configuration in `deploy.yaml`:

```yaml
# services/api/deploy.yaml - Kubernetes deployment
build:
  type: docker

targets:
  dev:
    type: kubernetes
    runner: self-hosted        # Use self-hosted runner (optional)
    namespace: develop         # Target namespace (must match kustomization.yaml)
  staging:
    type: kubernetes
    runner: self-hosted
    namespace: staging
  prod:
    type: kubernetes
    runner: self-hosted
    namespace: prod
```

```yaml
# apps/dashboard/deploy.yaml - PHP app on Apache servers
build:
  type: none  # Or: npm, yarn, hugo, zip

targets:
  dev:
    type: server
    method: rsync
    hosts:
      - web1.dev.example.com
    path: /var/www/dashboard
    owner: www-data:www-data
    keep_releases: 5
    shared_dirs:
      - storage
      - uploads
    post_deploy:
      - php artisan cache:clear
  staging:
    type: server
    hosts:
      - web1.staging.example.com
      - web2.staging.example.com
    path: /var/www/dashboard
  prod:
    type: server
    hosts:
      - web1.example.com
      - web2.example.com
    path: /var/www/dashboard
```

```yaml
# infra/database/deploy.yaml - Managed database migrations only
build:
  type: none

targets:
  dev:
    type: migration
  staging:
    type: migration
  prod:
    type: migration
```

### Per-Environment Targeting

Services don't have to deploy to all environments. If a target is not defined, the service is skipped:

```yaml
# services/staging-only/deploy.yaml - Only deploy to staging (for testing)
build:
  type: docker

targets:
  staging:
    type: kubernetes
    namespace: staging
```

```yaml
# apps/internal-tools/deploy.yaml - Skip development, deploy to staging+prod
build:
  type: npm

targets:
  staging:
    type: server
    hosts:
      - internal.staging.example.com
  prod:
    type: server
    hosts:
      - internal.example.com
```

### Build Types

| Build Type | Description | Output |
|------------|-------------|--------|
| `docker` | Build Docker image and push to registry | Container image |
| `npm` | Run `npm ci && npm run build` | `dist/` directory |
| `yarn` | Run `yarn install && yarn build` | `dist/` directory |
| `hugo` | Build Hugo static site | `public/` directory |
| `zip` | Create zip archive of source | `.zip` file |
| `none` | No build step needed | Source files |

### Build Context for Shared Packages

For monorepos with shared packages, set `build.context` to reference the repo root:

```yaml
# services/api/deploy.yaml
build:
  type: docker
  context: .  # Build context is repo root, Dockerfile still in services/api/
```

This allows your Dockerfile to copy shared directories:

```dockerfile
# services/api/Dockerfile
FROM node:20-alpine
WORKDIR /app

# Copy shared packages first
COPY packages/shared ./packages/shared

# Copy service code
COPY services/api ./services/api

WORKDIR /app/services/api
RUN npm ci && npm run build
```

### Namespace Configuration

For Kubernetes deployments, the namespace is extracted from your `kustomization.yaml`:

```yaml
# services/api/k8s/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: develop  # This namespace is used for deployment

resources:
  - ../../base
```

The pipeline reads the namespace from `kustomization.yaml` rather than assuming it matches the environment name. This allows you to use custom namespaces per environment.

### Hybrid Architecture Example

```
┌─────────────────────────────────────────────────────────────────┐
│                        Your Infrastructure                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Frontend   │    │   Backend    │    │   Database   │       │
│  │  (Apache/PHP)│    │ (Kubernetes) │    │   (AWS RDS)  │       │
│  │              │    │              │    │              │       │
│  │  deploy.yaml │    │  deploy.yaml │    │  deploy.yaml │       │
│  │  type:server │    │  type:k8s    │    │  type:migr.  │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│     SSH/rsync          kubectl apply      Flyway migrate         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

See [`deploy/deploy.schema.yaml`](deploy/deploy.schema.yaml) for full configuration options.

## Getting Started

### After Forking

When you fork this repository, GitHub gives you the option to fork only the `main` branch. If you chose that option (or want to ensure all required branches exist), run the setup script:

```bash
# Clone your fork
git clone https://github.com/YOUR_USERNAME/k8s-project-starter.git
cd k8s-project-starter

# Run the setup script to create required branches
./scripts/setup-repo.sh
```

This creates the `develop` and `staging` branches required for the CI/CD pipeline:

```
main (production)
  └── staging (release candidates)
        └── develop (development)
```

**Options:**

```bash
# Preview what will be done (no changes)
./scripts/setup-repo.sh --dry-run

# Create branches locally without pushing
./scripts/setup-repo.sh --local-only

# Also configure GitHub environments (requires gh CLI)
./scripts/setup-repo.sh --setup-envs
```

After running the script, follow the **[Setup Guide](SETUP_GUIDE.md)** to configure GitHub Environments and Secrets.

### Prerequisites

- Docker
- kubectl configured for your clusters (if using Kubernetes)
- SSH access to servers (if using server deployments)
- GitHub repository with Actions enabled

### Required GitHub Configuration

**Secrets (Kubernetes deployments):**
```
DEV_KUBECONFIG          # Base64 kubeconfig for dev cluster
STAGING_KUBECONFIG      # Base64 kubeconfig for staging cluster
PROD_KUBECONFIG         # Base64 kubeconfig for prod cluster

DEV_MYSQL_PASSWORD      # MySQL password for dev (in-cluster)
STAGING_MYSQL_PASSWORD  # MySQL password for staging
PROD_MYSQL_PASSWORD     # MySQL password for prod

DEV_JWT_SECRET          # JWT secret for dev
STAGING_JWT_SECRET      # JWT secret for staging
PROD_JWT_SECRET         # JWT secret for prod
```

**Secrets (Server deployments - SSH/rsync):**
```
DEV_SSH_PRIVATE_KEY     # SSH private key for dev servers
DEV_SSH_USER            # SSH username (default: deploy)
STAGING_SSH_PRIVATE_KEY
STAGING_SSH_USER
PROD_SSH_PRIVATE_KEY
PROD_SSH_USER
```

**Secrets (Managed database - RDS/Cloud SQL):**
```
DEV_DATABASE_URL        # JDBC URL: jdbc:mysql://host:3306/db
DEV_DATABASE_USER       # Database username
DEV_DATABASE_PASSWORD   # Database password
STAGING_DATABASE_URL
STAGING_DATABASE_USER
STAGING_DATABASE_PASSWORD
PROD_DATABASE_URL
PROD_DATABASE_USER
PROD_DATABASE_PASSWORD
```

**Environments:**
- `development` — No protection rules
- `staging` — No protection rules
- `production` — Required reviewers enabled

### Creating a New Service

1. Create directory structure:
   ```bash
   mkdir -p services/my-service/{src,k8s/{base,overlays/{dev,staging,prod}}}
   echo "1.0.0" > services/my-service/VERSION
   ```

2. Add Dockerfile and source code

3. Create Kustomize manifests (see existing services for examples)

4. Register in change detection:
   ```yaml
   # .github/actions/detect-changes/action.yml
   my-service:
     - 'services/my-service/**'
   ```

5. Open PR to `develop`

## Database Migrations

The pipeline supports two migration approaches depending on your database.

### PostgreSQL Migrations (Native Runner)

For PostgreSQL, enable migrations in your `deploy.yaml`:

```yaml
# services/my-service/deploy.yaml
database:
  type: postgresql
  migrations: true

targets:
  dev:
    type: kubernetes
```

Create migration files:

```bash
mkdir -p services/my-service/migrations
touch services/my-service/migrations/001_create_users.sql
touch services/my-service/migrations/002_add_email_index.sql
```

Migrations run automatically before deployment:
- Uses `schema_migrations` tracking table
- Runs as Kubernetes Job with `postgres:16-alpine`
- Each migration runs in a transaction
- Expects credentials in `{service}-secrets` Secret (`DB_PASSWORD` key)
- Database service naming convention: `{service}-postgres`

### MySQL/Flyway Migrations

For MySQL or managed databases (RDS, Cloud SQL), use Flyway:

```bash
mkdir -p services/my-service/migrations
touch services/my-service/migrations/V001__initial_schema.sql
```

Naming: `V{NNN}__{description}.sql`
- Zero-padded version number
- Double underscore separator
- Description with underscores

Set `type: migration` in your `deploy.yaml` and configure database secrets per environment.

### Execution

Migrations run automatically via the `_deploy-migrations.yml` workflow:
1. Flyway runs in a Docker container
2. Connects directly to your managed database via JDBC
3. Applies pending migrations before application deployment

## Scripts

```bash
# Setup repository after forking (creates develop/staging branches)
./scripts/setup-repo.sh

# Bump service version
./scripts/version-bump.sh <service-path> <major|minor|patch>

# Validate platform compatibility
./scripts/validate-platform.sh [--strict]
```

## Container Registries

Default: GitHub Container Registry (GHCR)

Alternative configs available in `deploy/registry/`:
- `ghcr.env` — GitHub Container Registry (default)
- `dockerhub.env.example` — Docker Hub
- `ecr.env.example` — AWS ECR

## License

This project is licensed under the [MIT License](LICENSE).
