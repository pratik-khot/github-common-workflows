# Changed Terraform Dirs

Detect changed top-level folders and expose them as a JSON array for a GitHub Actions matrix.

## Usage

Run the action after checking out the full repository history:

```yaml
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      dirs: ${{ steps.changed.outputs.dirs }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - id: changed
        uses: YOUR_ORG/github-common-workflows/.github/actions/changed-terraform-dirs@v1

  terraform:
    needs: detect-changes
    if: needs.detect-changes.outputs.dirs != '["none"]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        working_dir: ${{ fromJSON(needs.detect-changes.outputs.dirs) }}
    steps:
      - uses: actions/checkout@v4
      - name: Run Terraform commands
        working-directory: ${{ matrix.working_dir }}
        run: |
          terraform fmt -check -recursive
          terraform init -backend=false -no-color
          terraform validate -no-color
```

The action outputs a JSON array such as `["modules"]` for changed paths under `modules/`. When no changed path contains a folder separator, it outputs `["none"]`. For pull requests it compares the base branch with `HEAD`; for pushes it compares `github.event.before` with `github.sha`.

The action only detects folders. The caller is responsible for installing Terraform, selecting the Terraform command, configuring credentials, and applying any path filters.