# Setup Guide

> **Important**: Make sure you used "Use this template" or manually cloned the repo (see README). Do NOT use GitHub's "Fork" button—forks maintain a link to the original repo and your PRs will accidentally be sent there.

After running `./scripts/setup-repo.sh`, follow these steps to complete your repository setup.

---

## Step 1: Create GitHub Environments

GitHub Environments let you configure different settings for dev, staging, and production deployments.

### How to create environments:

1. Go to your repository on GitHub
2. Click **Settings** (tab at the top)
3. In the left sidebar, click **Environments**
4. Click **New environment**
5. Create these three environments (one at a time):
   - `development`
   - `staging`
   - `production`

### Configure production protection (recommended):

1. After creating the `production` environment, click on it
2. Check **Required reviewers**
3. Add yourself (or your team) as a reviewer
4. Click **Save protection rules**

This ensures someone must approve before deploying to production.

---

## Step 2: Add Secrets

Secrets store sensitive values like passwords and API keys. The pipeline needs different secrets depending on what you're deploying.

### How to add secrets:

1. Go to your repository on GitHub
2. Click **Settings** (tab at the top)
3. In the left sidebar, click **Environments**
4. Click on the environment you want to add secrets to (e.g., `development`)
5. Scroll down to **Environment secrets**
6. Click **Add environment secret**
7. Enter the name and value
8. Click **Add secret**

Repeat for each environment (`development`, `staging`, `production`).

**Note:** Enter secrets as plain text. GitHub encrypts them automatically. The only exception is `KUBECONFIG` which should be base64-encoded (see below) because it's a multi-line file.

---

## Step 3: Which Secrets Do You Need?

You only need to configure secrets for the deployment types you're using. Skip sections that don't apply to you.

### Option A: Kubernetes Deployments

If you're deploying containers to Kubernetes clusters:

| Secret Name | Environment | Description | Format |
|-------------|-------------|-------------|--------|
| `DEV_KUBECONFIG` | development | Kubernetes config for dev cluster | base64-encoded |
| `STAGING_KUBECONFIG` | staging | Kubernetes config for staging cluster | base64-encoded |
| `PROD_KUBECONFIG` | production | Kubernetes config for prod cluster | base64-encoded |

**Getting your kubeconfig:**

```bash
# For a local cluster (minikube, kind, docker-desktop)
cat ~/.kube/config | base64

# For cloud providers, use their CLI:
# AWS EKS
aws eks update-kubeconfig --name my-cluster --region us-east-1
cat ~/.kube/config | base64

# Google GKE
gcloud container clusters get-credentials my-cluster --zone us-central1-a
cat ~/.kube/config | base64

# Azure AKS
az aks get-credentials --resource-group mygroup --name my-cluster
cat ~/.kube/config | base64
```

Copy the base64 output and paste it as the secret value.

### Option B: Server Deployments (SSH/rsync)

If you're deploying to traditional servers via SSH:

| Secret Name | Environment | Description | Format |
|-------------|-------------|-------------|--------|
| `DEV_SSH_PRIVATE_KEY` | development | SSH private key (contents of `~/.ssh/id_ed25519`) | plain text |
| `DEV_SSH_USER` | development | SSH username (e.g., `deploy`) | plain text |
| `STAGING_SSH_PRIVATE_KEY` | staging | SSH private key for staging | plain text |
| `STAGING_SSH_USER` | staging | SSH username | plain text |
| `PROD_SSH_PRIVATE_KEY` | production | SSH private key for production | plain text |
| `PROD_SSH_USER` | production | SSH username | plain text |

**Setting up SSH keys:**

```bash
# Generate a new key pair for deployments (recommended)
ssh-keygen -t ed25519 -C "deploy@github-actions" -f ~/.ssh/deploy_key

# Copy the public key to your server
ssh-copy-id -i ~/.ssh/deploy_key.pub user@your-server.com

# The secret value is the PRIVATE key:
cat ~/.ssh/deploy_key
```

### Option C: Database Migrations (Managed DB like RDS/Cloud SQL)

If you're running Flyway migrations against a managed database:

| Secret Name | Environment | Example |
|-------------|-------------|---------|
| `DEV_DATABASE_URL` | development | `jdbc:mysql://db.dev.example.com:3306/myapp` |
| `DEV_DATABASE_USER` | development | `app_user` |
| `DEV_DATABASE_PASSWORD` | development | (your password) |
| `STAGING_DATABASE_URL` | staging | `jdbc:mysql://db.staging.example.com:3306/myapp` |
| `STAGING_DATABASE_USER` | staging | `app_user` |
| `STAGING_DATABASE_PASSWORD` | staging | (your password) |
| `PROD_DATABASE_URL` | production | `jdbc:mysql://db.example.com:3306/myapp` |
| `PROD_DATABASE_USER` | production | `app_user` |
| `PROD_DATABASE_PASSWORD` | production | (your password) |

All database secrets are plain text.

### Application Secrets

These are used by your applications at runtime:

| Secret Name | Environment | Description |
|-------------|-------------|-------------|
| `DEV_JWT_SECRET` | development | Secret for signing JWTs |
| `STAGING_JWT_SECRET` | staging | Secret for signing JWTs |
| `PROD_JWT_SECRET` | production | Secret for signing JWTs |
| `DEV_MYSQL_PASSWORD` | development | MySQL password (for in-cluster MySQL) |
| `STAGING_MYSQL_PASSWORD` | staging | MySQL password |
| `PROD_MYSQL_PASSWORD` | production | MySQL password |

**Generating secure secrets:**

```bash
# Generate a random 32-character secret
openssl rand -base64 32
```

---

## Step 4: Start Developing

Now you're ready to create your first feature:

```bash
# Switch to the develop branch
git checkout develop

# Create a feature branch
git checkout -b feature/my-service/my-feature

# Make your changes...

# Commit and push
git add .
git commit -m "feat: add my feature"
git push -u origin feature/my-service/my-feature
```

Then go to GitHub and create a Pull Request targeting `develop`.

---

## Quick Reference: Environment Setup Checklist

### Minimum Setup (just to get pipelines running):

- [ ] Create `development` environment
- [ ] Create `staging` environment
- [ ] Create `production` environment with required reviewers
- [ ] Add secrets for your deployment type (see Step 3)

### Full Setup:

- [ ] All environments created
- [ ] Production has required reviewers enabled
- [ ] Kubeconfig secrets (if using Kubernetes)
- [ ] SSH secrets (if using server deployments)
- [ ] Database secrets (if using managed databases)
- [ ] Application secrets (JWT, passwords, etc.)

---

## Troubleshooting

### "Error: Resource not accessible by integration"

Your GitHub token doesn't have permission to create environments. Go to Settings → Actions → General → Workflow permissions and select "Read and write permissions".

### "Error: Context 'production' not found"

The `production` environment doesn't exist. Create it in Settings → Environments.

### "Error: Kubeconfig is invalid"

Make sure you base64-encoded your kubeconfig:
```bash
cat ~/.kube/config | base64 -w 0  # Linux
cat ~/.kube/config | base64       # macOS
```

### Pipeline not triggering

- Check that you're pushing to the correct branch (`develop`, `staging`, or `main`)
- Check Settings → Actions to ensure Actions are enabled
- Check the Actions tab for any failed runs with error details

---

## Need Help?

- Check the [README.md](README.md) for pipeline and branching documentation
- Open an issue if you're stuck
