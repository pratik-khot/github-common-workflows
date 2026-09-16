# Terraform Workflows

`terraform-plan.yml` and `terraform-apply.yml` run from the required `terraform-root` directory. It must contain the complete Terraform configuration for one application and environment, such as `myapp/nonprod`.

Plan accepts an optional `TF_API_TOKEN` secret for Terraform Cloud and newline-delimited `backend-config` values. Apply accepts `AWS_ROLE_ARN` and `TF_API_TOKEN`; it assumes the AWS role through GitHub OIDC.

The apply workflow binds its job to the supplied `environment`. Configure approvals, deployment branches, and environment secrets in every caller repository. `destroy` only runs when `action: destroy` and `allow-destroy: true` are both supplied.

Caller workflows own triggers and path filters. Use paths such as `myapp/dev/**`, not a common `iac/**` path.

