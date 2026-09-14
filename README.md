# GitHub Common Workflows

Reusable GitHub Actions workflows for Terraform, Docker, Helm, GitOps, security scans, and releases.

## Terraform layout

Each Terraform environment is a self-contained root:

```text
myapp/
  dev/
    backend.tf
    main.tf
    variables.tf
    terraform.tfvars
  nonprod/
    backend.tf
    main.tf
    variables.tf
    terraform.tfvars
  prod/
    backend.tf
    main.tf
    variables.tf
    terraform.tfvars
```

Pass the environment directory to workflows as `terraform-root`, for example `myapp/dev`.

## Consume a workflow

Use a released major version in callers. Pin production-critical workflows to an immutable release tag after the first release.

```yaml
jobs:
  plan:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/terraform-plan.yml@v1
    with:
      terraform-root: myapp/dev
      terraform-version: 1.14.3
    secrets: inherit
```

Terraform apply requires a caller environment and AWS OIDC permissions:

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  apply:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/terraform-apply.yml@v1
    with:
      terraform-root: myapp/prod
      environment: prod
      aws-region: us-east-1
    secrets: inherit
```

Configure GitHub Environment protection rules in the caller repository. The `environment` input is the approval boundary before Terraform applies or destroys infrastructure.

## Reusable workflows

Use workflows under `.github/workflows/` from a caller job with `uses`. Caller repositories own triggers, path filters, and environment rules.

| Workflow | What it does | Key inputs | Required permissions and secrets |
| --- | --- | --- | --- |
| `terraform-plan.yml` | Runs `fmt`, `init`, `validate`, and `plan` from one app/environment root. Optionally uploads `tfplan`. | `terraform-root`, `terraform-version`, `backend-config`, `terraform-args` | `contents: read`; optional `TF_API_TOKEN` for Terraform Cloud. |
| `terraform-apply.yml` | Assumes AWS role through OIDC, initializes and validates Terraform, then applies or explicitly allowed destroys. | `terraform-root`, `environment`, `aws-region`, `action`, `allow-destroy` | `contents: read`, `id-token: write`; `AWS_ROLE_ARN`; optional `TF_API_TOKEN`. |
| `docker-build.yml` | Checks out the caller, configures Buildx/QEMU, and builds a Docker image. It does not push unless `push: true`. | `context`, `file`, `image-name`, `tags`, `platforms` | `contents: read`. |
| `docker-publish.yml` | Assumes AWS role through OIDC, authenticates to ECR, builds multi-platform images, pushes tags, and returns an image digest. | `ecr-registry`, `image-name`, `tags`, `aws-region`, `platforms` | `contents: read`, `id-token: write`; `AWS_ROLE_ARN`. |
| `helm-package.yml` | Builds dependencies, lints, packages a Helm chart, uploads its archive, and can push it to OCI. | `chart-path`, `oci-registry`, `push` | `contents: read`; caller must make OCI credentials available when pushing. |
| `gitops-update.yml` | Clones a GitOps repository, updates one Kustomize image or Helm values path, validates Kustomize, and creates a PR by default. | `gitops-repository`, `target-directory`, `update-type`, `image-reference` | `contents: read`; `GITOPS_TOKEN` with access to the target GitOps repository. |
| `security-scan.yml` | Runs selected Checkov Terraform, Gitleaks secret, and Trivy image scans; uploads Trivy SARIF. | `terraform-root`, `image-reference`, `scan-terraform`, `scan-secrets`, `scan-image` | `contents: read`, `security-events: write`. |
| `release.yml` | Runs Release Please and returns whether a release was created plus its tag. | `release-type`, `config-file`, `manifest-file` | `contents: write`, `pull-requests: write`; optional `RELEASE_TOKEN`. |

### Docker build

```yaml
jobs:
  build:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/docker-build.yml@v1
    with:
      context: src/catalog
      file: src/catalog/Dockerfile
      image-name: catalog
      tags: pr-${{ github.event.pull_request.number }}
      platforms: linux/amd64
```

### Docker publish to ECR

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  publish:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/docker-publish.yml@v1
    with:
      context: src/catalog
      image-name: catalog
      tags: v1.2.3,latest
      ecr-registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
      aws-region: us-east-1
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

### Helm package

```yaml
jobs:
  package:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/helm-package.yml@v1
    with:
      chart-path: charts/catalog
      push: false
```

### GitOps update

```yaml
jobs:
  update-gitops:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/gitops-update.yml@v1
    with:
      gitops-repository: YOUR_ORG/gitops-repo
      target-directory: apps/catalog/overlays/dev
      update-type: kustomize-image
      image-name: catalog
      image-reference: 123456789012.dkr.ecr.us-east-1.amazonaws.com/catalog@sha256:REPLACE_ME
    secrets:
      GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
```

For a Helm value update, set `update-type: helm-value` and add `helm-values-path`, such as `.image.tag`.

### Security scan

```yaml
permissions:
  contents: read
  security-events: write

jobs:
  security:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/security-scan.yml@v1
    with:
      terraform-root: myapp/dev
      scan-image: true
      image-reference: 123456789012.dkr.ecr.us-east-1.amazonaws.com/catalog:v1.2.3
```

### Release Please

```yaml
permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    uses: YOUR_ORG/github-common-workflows/.github/workflows/release.yml@v1
    with:
      release-type: simple
```

## Composite actions

Composite actions are lower-level building blocks for a workflow that needs only tool installation or AWS authentication.

| Action | What it does | Required inputs |
| --- | --- | --- |
| `setup-terraform` | Enables the Terraform provider-plugin cache and installs the requested Terraform version; can configure Terraform Cloud credentials. | `terraform-version`; optional `terraform-cloud-token`. |
| `setup-kubectl` | Installs the requested kubectl version. | Optional `kubectl-version`. |
| `setup-helm` | Installs the requested Helm version. | Optional `helm-version`. |
| `aws-login` | Exchanges the GitHub OIDC token for AWS role credentials. | `role-to-assume`, `aws-region`; optional `role-session-name`. |

Use a composite action only in a normal step, not as a job-level reusable workflow:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4
  - uses: YOUR_ORG/github-common-workflows/.github/actions/aws-login@v1
    with:
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      aws-region: us-east-1
```

See [Terraform](docs/terraform.md), [Docker](docs/docker.md), and [GitOps](docs/gitops.md) for implementation details and authorization requirements.