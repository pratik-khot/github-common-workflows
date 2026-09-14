# Docker Workflows

`docker-build.yml` builds a caller-provided Docker context without publishing by default. It accepts context, Dockerfile, image name, tags, platforms, and build arguments.

`docker-publish.yml` pushes to Amazon ECR using `AWS_ROLE_ARN` and GitHub OIDC. Supply an ECR registry hostname, image name, tags, and platforms. Its `image-digest` output is the immutable reference to pass to GitOps updates.

The caller must grant `contents: read` and `id-token: write` permissions and configure an AWS role trust policy for the caller repository.