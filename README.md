# K8s Monorepo Starter

A Kubernetes monorepo with GitHub Actions CI/CD, supporting multiple services with semantic versioning and environment-based deployments.

## Repository Structure

```
k8s-project-starter/
├── .github/
│   ├── workflows/
│   │   ├── ci-pr.yml              # Build on PR (all branches)
│   │   ├── cd-develop.yml         # Auto-deploy to dev
│   │   ├── cd-staging.yml         # Auto-deploy to staging
│   │   ├── cd-production.yml      # Deploy to prod (manual approval)
│   │   ├── _build-service.yml     # Reusable: build & push images
│   │   └── _deploy-service.yml    # Reusable: kustomize + kubectl
│   └── actions/
│       └── detect-changes/        # Path-based change detection
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
│   ├── mysql/
│   │   ├── migrations/            # Flyway SQL migrations
│   │   └── k8s/
│   ├── redis/
│   └── shared/                    # Namespaces, network policies
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

Only services with changes are built and deployed. Changes are detected by path:

- `apps/dashboard/**` → builds `dashboard`
- `services/api/**` → builds `api`
- `infra/shared/**` → rebuilds ALL services
- `.github/workflows/_*.yml` → rebuilds ALL services

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

Each component can specify its deployment type in `deploy.yaml`:

```yaml
# apps/dashboard/deploy.yaml - PHP app on Apache servers
type: server
method: rsync

targets:
  dev:
    hosts:
      - web1.dev.example.com
    path: /var/www/dashboard
  staging:
    hosts:
      - web1.staging.example.com
      - web2.staging.example.com
    path: /var/www/dashboard
  prod:
    hosts:
      - web1.example.com
      - web2.example.com
      - web3.example.com
    path: /var/www/dashboard
```

```yaml
# services/api/deploy.yaml - Kubernetes deployment
type: kubernetes
```

```yaml
# infra/mysql/deploy.yaml - RDS migrations only
type: migration
engine: flyway
path: migrations
```

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

### Creating Your Project

> **Warning**: Do NOT use GitHub's "Fork" button. Forks maintain a link to this repository, and PRs will accidentally target the original repo instead of your own.

**Option 1: Use as Template (Recommended)**

1. Click the green **"Use this template"** button at the top of this repo
2. Select **"Create a new repository"**
3. Choose your organization/account and name your project
4. Clone your new repo and run the setup script

**Option 2: Manual Clone**

```bash
# Clone the starter
git clone https://github.com/ORIGINAL/k8s-project-starter.git my-project
cd my-project

# Remove the link to the original repo
git remote remove origin

# Create a new repo on GitHub, then:
git remote add origin git@github.com:YOUR_ORG/my-project.git
git push -u origin main
```

### After Creating Your Project

Once you have your own independent copy, run the setup script to create the required branches:

```bash
# Clone your new repo
git clone https://github.com/YOUR_ORG/my-project.git
cd my-project

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

Using Flyway Community Edition for forward migrations.

### Creating Migrations

```bash
# Add migration file
touch infra/mysql/migrations/V002__add_orders_table.sql
```

Naming: `V{NNN}__{description}.sql`
- Zero-padded version number
- Double underscore separator
- Description with underscores

### Execution

Migrations run automatically during deployment:
1. SQL files packaged into ConfigMap
2. Flyway Job executes
3. Application deploys after Job completes

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
