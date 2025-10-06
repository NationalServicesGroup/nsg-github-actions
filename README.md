# nsg-github-actions

Reusable GitHub Actions and workflows for NSG repositories. This repository contains composite actions and reusable workflows for standardizing CI/CD across NSG projects, particularly for GCP infrastructure and Terraform deployments.

## Features

- **Multi-environment Terraform deployments** (stage, hgeprod, etc.)
- **GCP authentication** with service account keys
- **Automated PR plans** with plan output as PR comments
- **Sequential environment deployments** with manual approval gates
- **Environment-specific backend and variable management**

## Composite Actions

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

Set these secrets in your repository or organization:

- `GCP_SERVICE_ACCOUNT_KEY` - Base64 encoded GCP service account key
- `PROJECT_ID` - GCP project ID

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
