# Terraform Workflows

`terraform-plan.yml`, `terraform-apply.yml`, and `terraform-destroy.yml` own the entire Terraform pipeline. Caller repositories only define triggers and call these workflows as jobs — checkout, changed-directory detection, matrix creation, AWS OIDC login, and every `terraform` command run inside the reusable workflows.

## terraform-plan.yml

Runs on pull requests (or any trigger the caller chooses). It detects every Terraform root with a changed `.tf`/`.tfvars` file since the base ref (or previous push), builds a matrix from those roots, and for each one runs `fmt -check`, `init`, `validate`, and `plan`. The binary plan is uploaded as an artifact named `tfplan-<root>-<run_id>-<run_attempt>` when `upload-plan` is `true` (default). If nothing changed, the plan job is skipped entirely.

Inputs: `root-path` (optional prefix filter), `aws-region` (required), `terraform-version` (optional; auto-detected from `.terraform-version` or a `required_version` constraint when omitted), `backend-config`, `terraform-args`, `upload-plan`.

Secrets: `AWS_ROLE_ARN` (required — plan also authenticates to AWS so `init`/`validate` can reach remote state and providers), `TF_API_TOKEN` (optional, for Terraform Cloud).

Outputs: `has-changes`, `changed-roots`.

## terraform-apply.yml

Runs the same changed-root detection and matrix, but only applies when the job runs on `push` to `main` (enforced inside the workflow, not just by caller triggers). Each matrix job authenticates to AWS via OIDC, runs `init`, `validate`, and `apply -auto-approve` directly — it does **not** download a plan artifact from a prior run, since plan and apply are typically separate workflow runs and artifacts cannot be shared reliably across them. The job binds to the caller-supplied `environment`, so configure approvals, deployment branches, and environment secrets there.

Inputs: `root-path`, `environment` (required), `aws-region` (required), `terraform-version`, `backend-config`, `terraform-args`.

Secrets: `AWS_ROLE_ARN` (required), `TF_API_TOKEN` (optional).

## terraform-destroy.yml

A separate, explicit workflow intended for `workflow_dispatch` callers. It targets one `terraform-root` (no auto-detection — destroys must be explicit), binds to a protected `environment`, and requires both `action: destroy` and `allow-destroy: true` before running `init` and `destroy -auto-approve`. Missing either guard fails the job immediately with a clear error instead of silently skipping.

Inputs: `terraform-root` (required), `environment` (required), `action` (required), `allow-destroy` (required), `aws-region` (required), `terraform-version`, `backend-config`, `terraform-args`.

Secrets: `AWS_ROLE_ARN` (required), `TF_API_TOKEN` (optional).

## Consumer repositories

Caller workflows own triggers and path filters only. They must not perform checkout, AWS login, Terraform setup, changed-directory detection, matrix creation, or any `terraform` command themselves — all of that logic lives in this repository.

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  plan:
    if: github.event_name == 'pull_request'
    uses: YOUR_ORG/github-common-workflows/.github/workflows/terraform-plan.yml@v1
    with:
      aws-region: us-east-1
    secrets: inherit

  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: YOUR_ORG/github-common-workflows/.github/workflows/terraform-apply.yml@v1
    with:
      environment: prod
      aws-region: us-east-1
    secrets: inherit
```


