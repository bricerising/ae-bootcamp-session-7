# CI/CD Pipeline — Golden Path Reusable Workflow

## Overview

The golden-path CI pipeline is a reusable GitHub Actions workflow that encodes all required checks for services adopting the platform golden path. It validates code quality, security, infrastructure, and container builds in a single, composable template.

## Workflow Architecture

```
todo-service-ci.yml (caller)
  └── golden-path-ci.yml (reusable)
        ├── lint          — ESLint on backend + frontend
        ├── test          — Jest with 80% coverage gate
        ├── security-scan — Checkov against infra/ (HIGH = fail)
        ├── terraform-plan — validate + plan with OIDC credentials
        ├── docker-build  — validate Dockerfiles on PRs
        ├── terraform-apply — apply on merge to main (OIDC)
        └── build-and-push — ECR push + ECS deploy on merge to main
```

## Jobs

### lint

Runs ESLint on both `packages/backend` and `packages/frontend`. Catches style issues and common bugs before tests run.

### test

Runs Jest with coverage reporting. Fails the pipeline if line or branch coverage drops below 80%. Posts a coverage summary to the PR via `$GITHUB_STEP_SUMMARY`.

### security-scan

Runs Checkov against the `infra/` directory. Hard-fails on HIGH severity findings. Only runs when `run_terraform_plan` is true (i.e., when infrastructure changes are in scope).

### terraform-plan

Authenticates to AWS via OIDC, initialises the dev stack with the S3 backend, and runs `terraform plan`. Uploads the plan as an artifact for the apply job. On PRs (when apply is not enabled), injects mock VPC variables to allow validation without existing infrastructure.

### docker-build

Builds both Docker images (backend and frontend) without pushing. Validates that Dockerfiles are correct on every PR. Runs after lint and test pass.

### terraform-apply

Applies the Terraform plan from the previous job. Only runs on push to `main` (controlled by `run_terraform_apply` input). Uses OIDC for keyless AWS authentication.

### build-and-push

Builds Docker images, pushes to ECR, and triggers an ECS service redeployment. Only runs on push to `main` after terraform-apply completes successfully.

## How to Adopt

A new service team needs only a caller workflow:

```yaml
name: My Service CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  ci:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

## Configuring Secrets

The `terraform-plan` and `terraform-apply` jobs require an AWS IAM role ARN for OIDC federation:

1. Create an IAM role with a trust policy that allows your GitHub org/repo to assume it via OIDC
2. Add the role ARN as a repository secret named `AWS_ROLE_ARN`
3. The caller workflow passes it to the reusable workflow via `secrets: aws_role_arn`

The caller must declare `id-token: write` at the top-level permissions block for OIDC to work.
