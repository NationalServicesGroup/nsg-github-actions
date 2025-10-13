# nsg-github-actions

Reusable GitHub Actions and workflows for NSG repositories. This repository contains composite actions and reusable workflows for standardizing CI/CD across NSG projects, particularly for GCP infrastructure and Terraform deployments.

## Features

### Infrastructure (Terraform)
- **Multi-environment Terraform deployments** (stage, hgeprod, etc.)
- **GCP authentication** with service account keys
- **Automated PR plans** with plan output as PR comments
- **Sequential environment deployments** with manual approval gates
- **Environment-specific backend and variable management**

### Python/Django Applications
- **Python environment setup** with uv package manager
- **Code quality checks** with ruff linting
- **Django testing** with pytest, MySQL, and Redis
- **Docker build and push** to AWS ECR and GCP GCR
- **AWS ECS deployments** with automatic task definition updates
- **Static file deployment** to S3 with CloudFront invalidation
- **SSH configuration** for private repository access

## Composite Actions

### Infrastructure Actions

### `gcp-auth`

Authenticates with GCP using a service account key from secrets.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/gcp-auth@main
  with:
    gcp_service_account_key: ${{ secrets.GCP_SERVICE_ACCOUNT_KEY }}
    project_id: ${{ secrets.PROJECT_ID }}
```

### `install-terraform`

Installs a specific version of Terraform.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/install-terraform@main
  with:
    tf_version: "1.12.0"
```

### `terraform-init`

Initializes Terraform with environment-specific backend and variables. Expects a repo structure with:
- `backends/backend.<environment>.tf`
- `variables/<environment>.tfvars`

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/terraform-init@main
  with:
    environment: stage
    working_directory: "."
```

### `terraform-plan-env`

Runs a complete terraform plan for a specific environment (auth + init + plan).

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/terraform-plan-env@main
  with:
    environment: stage
    gcp_service_account_key: ${{ secrets.GCP_SERVICE_ACCOUNT_KEY }}
    project_id: ${{ secrets.PROJECT_ID }}
    working_directory: "."
```

---

### Python/Django Actions

### `python-setup`

Sets up Python environment using the uv package manager. Installs Python, uv, syncs dependencies, and activates the virtual environment.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/python-setup@main
  with:
    python-version: "3.13"
    uv-version: "latest"
    working-directory: "."
    cache-dependencies: "true"
```

### `python-lint`

Runs ruff linting checks on Python code with optional format checking.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/python-lint@main
  with:
    working-directory: "."
    ruff-args: ""
    fail-on-error: "true"
    include-format-check: "true"
```

### `python-test-django`

Runs pytest tests for Django applications with MySQL and Redis service support. Includes database initialization and coverage reporting.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/python-test-django@main
  with:
    working-directory: "."
    database-host: "127.0.0.1"
    database-user: "root"
    database-password: "root"
    database-init-sql: ".bitbucket/01_cicd_pipeline_mysql_initialize.sql"
    pytest-args: "--exitfirst --ds=hge.settings --no-migrations --create-db"
    django-settings: "hge.settings"
    coverage: "true"
```

---

### AWS Actions

### `aws-auth`

Authenticates with AWS using access keys or IAM role (OIDC).

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/aws-auth@main
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: "us-east-1"
```

### `docker-build-ecr`

Builds Docker image with platform detection and pushes to AWS ECR.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/docker-build-ecr@main
  with:
    ecr-registry: ${{ secrets.ECR_REGISTRY }}
    image-name: "hge-django"
    image-tag: ${{ github.sha }}
    context: "."
    dockerfile: "Dockerfile"
    platforms: "linux/amd64"
    push-latest: "true"
    no-cache: "false"
```

### `ecs-deploy`

Updates ECS task definition and deploys to ECS service with optional stability waiting.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/ecs-deploy@main
  with:
    cluster-name: "hge-production"
    service-name: "hge-django"
    task-definition: "hge-django"
    image-uri: ${{ steps.build.outputs.image-uri }}
    aws-region: "us-east-1"
    wait-for-stable: "true"
    force-new-deployment: "true"
```

### `django-collectstatic-s3`

Collects Django static files and uploads them to S3 with optional CloudFront invalidation.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/django-collectstatic-s3@main
  with:
    working-directory: "."
    django-settings: "hge.settings"
    s3-bucket: "my-static-files-bucket"
    s3-prefix: "static/"
    aws-region: "us-east-1"
    cloudfront-distribution-id: "E1234567890ABC"
    delete-removed: "false"
```

---

### GCP Actions

### `docker-build-gcr`

Builds Docker image and pushes to Google Container Registry.

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/docker-build-gcr@main
  with:
    gcp-project-id: ${{ secrets.PROJECT_ID }}
    image-name: "hge-django"
    image-tag: ${{ github.sha }}
    registry: "gcr.io"
    context: "."
    dockerfile: "Dockerfile"
    push-latest: "true"
```

---

### Utility Actions

### `setup-ssh`

Configures SSH keys for accessing private repositories (GitHub, Bitbucket, etc.).

**Usage:**
```yaml
- uses: NationalServicesGroup/nsg-github-actions/setup-ssh@main
  with:
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
    ssh-known-hosts: "github.com,bitbucket.org"
    ssh-key-name: "id_ed25519"
```

---

## Reusable Workflows

### Terraform PR Plan

Runs terraform plan for all specified environments when a PR is opened/updated.

**Example usage in your repo:**

Create `.github/workflows/pr-plan.yaml`:

```yaml
name: Terraform PR Plan

on:
  pull_request:
    branches: [main, master]

jobs:
  plan:
    uses: NationalServicesGroup/nsg-github-actions/.github/workflows/terraform-pr-plan.yaml@main
    with:
      environments: '["stage", "hgeprod"]'
      tf_version: "1.12.0"
    secrets:
      gcp_service_account_key: ${{ secrets.GCP_SERVICE_ACCOUNT_KEY }}
      project_id: ${{ secrets.PROJECT_ID }}
```

### Terraform Deploy

Deploys to a single environment with optional manual approval.

**Example usage in your repo:**

Create `.github/workflows/deploy-stage.yaml`:

```yaml
name: Deploy to Stage

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: NationalServicesGroup/nsg-github-actions/.github/workflows/terraform-deploy.yaml@main
    with:
      environment: stage
      tf_version: "1.12.0"
      require_approval: true
    secrets:
      gcp_service_account_key: ${{ secrets.GCP_SERVICE_ACCOUNT_KEY }}
      project_id: ${{ secrets.PROJECT_ID }}
```

### Terraform Deploy All

Deploys to multiple environments sequentially, each with approval gates.

**Example usage in your repo:**

Create `.github/workflows/deploy-all.yaml`:

```yaml
name: Deploy All Environments

on:
  workflow_dispatch:
  push:
    tags:
      - 'v*.*.*'

jobs:
  deploy-all:
    uses: NationalServicesGroup/nsg-github-actions/.github/workflows/terraform-deploy-all.yaml@main
    with:
      environments: '["stage", "hgeprod"]'
      tf_version: "1.12.0"
    secrets:
      gcp_service_account_key: ${{ secrets.GCP_SERVICE_ACCOUNT_KEY }}
      project_id: ${{ secrets.PROJECT_ID }}
```

## Repository Structure Requirements

For repos using these actions, the expected structure is:

```
your-repo/
├── backends/
│   ├── backend.stage.tf
│   └── backend.hgeprod.tf
├── variables/
│   ├── stage.tfvars
│   └── hgeprod.tfvars
├── main.tf
├── variables.tf
└── .github/
    └── workflows/
        ├── pr-plan.yaml
        ├── deploy-stage.yaml
        └── deploy-all.yaml
```

## Secrets Required

### For Terraform/GCP Projects
- `GCP_SERVICE_ACCOUNT_KEY` - Base64 encoded GCP service account key
- `PROJECT_ID` - GCP project ID

### For Python/Django Projects
- `SSH_PRIVATE_KEY` - SSH private key for accessing private repositories (optional, base64 encoded)

### For AWS Projects
- `AWS_ACCESS_KEY_ID` - AWS access key ID
- `AWS_SECRET_ACCESS_KEY` - AWS secret access key
- `ECR_REGISTRY` - ECR registry URL (e.g., 123456789.dkr.ecr.us-east-1.amazonaws.com)

### For Combined Projects
Use all relevant secrets from the categories above

## Environment Protection Rules

To enable manual approvals for deployments, configure environment protection rules in your repository settings:

1. Go to Settings → Environments
2. Add environments: `stage`, `hgeprod`, etc.
3. Configure protection rules:
   - Required reviewers
   - Wait timer
   - Deployment branches

## Contributing

Contributions are welcome! Please ensure any new actions or workflows follow the existing patterns and include documentation.

## License

Internal use for National Services Group.
