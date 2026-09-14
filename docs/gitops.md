# GitOps Updates

`gitops-update.yml` checks out the selected GitOps repository using `GITOPS_TOKEN`, updates exactly the supplied target directory, validates Kustomize changes, and opens a pull request by default.

For Kustomize, set `update-type: kustomize-image`, `image-name`, and `image-reference`. For Helm values, set `update-type: helm-value`, `helm-values-path`, and `image-reference`.

Use a GitHub App installation token or fine-grained PAT authorized only for the target GitOps repository. The caller repository's `GITHUB_TOKEN` normally cannot write to a different repository. Set `create-pull-request: false` only when direct pushes are explicitly permitted by repository policy.