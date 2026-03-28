# github-workflows

Reusable GitHub Actions workflows and composite actions.

## Reusable Workflows

### `.NET Docker Deploy`

Build a .NET project, push Docker image to GHCR, deploy via docker compose on a self-hosted runner.

```yaml
name: Build and Deploy
on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  build-deploy:
    uses: wiktorkowalski/github-workflows/.github/workflows/dotnet-docker-deploy.yml@master
    with:
      image-name: wiktorkowalski/myapp
      dockerfile: ./Dockerfile
      context: .
      dotnet-version: "10.0.x"
      dotnet-project: src/MyApp/MyApp.csproj  # empty = skip .NET build
      deploy-compose-path: /home/ubuntu/docker/myapp  # empty = skip deploy
      deploy-runner-labels: '["self-hosted"]'
    secrets: inherit
```

### `Terraform PR`

Runs lint (TFLint), format check, validate, and plan with PR comment on pull requests.

```yaml
name: Terraform PR
on:
  pull_request:
    branches: [master]

jobs:
  terraform:
    uses: wiktorkowalski/github-workflows/.github/workflows/terraform-pr.yml@master
    with:
      working-directory: terraform/
      aws-role-arn: arn:aws:iam::123456789012:role/GitHubActions  # empty = skip AWS
    secrets: inherit
```

Supported secret passthrough: `TF_API_TOKEN`, `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `HCLOUD_TOKEN`.

### `Terraform Apply`

Plan then apply with concurrency protection. Use on master push after PR merge.

```yaml
name: Terraform Apply
on:
  push:
    branches: [master]

jobs:
  terraform:
    uses: wiktorkowalski/github-workflows/.github/workflows/terraform-apply.yml@master
    with:
      working-directory: terraform/
      aws-role-arn: arn:aws:iam::123456789012:role/GitHubActions
    secrets: inherit
```

### `Claude Code Review`

AI code review on PRs and `@claude` mentions. Posts inline comments with fix suggestions.

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    uses: wiktorkowalski/github-workflows/.github/workflows/claude-code-review.yml@master
    with:
      # model: claude-opus-4-6  # default
      # max-turns: 30  # controls cost
      # review-prompt: "Also check for Go-specific issues."  # appended to default
    secrets: inherit
```

Required secrets: `ANTHROPIC_API_KEY`. Optional: `ANTHROPIC_PROXY_URL`.

### `Auto Approve Codeowner PR`

Auto-approves PRs when the author is listed in `CODEOWNERS` and all other checks pass.

```yaml
name: Auto Approve
on:
  check_suite:
    types: [completed]

jobs:
  approve:
    uses: wiktorkowalski/github-workflows/.github/workflows/auto-approve-codeowner.yml@master
```

Requires a `.github/CODEOWNERS` file in the consuming repo.

## Composite Actions

Internal building blocks used by the reusable workflows. Can also be used directly in custom workflows:

| Action | Description |
|--------|-------------|
| `actions/setup-dotnet` | Install .NET SDK + restore NuGet dependencies |
| `actions/docker-build-push` | Buildx + GHCR login + metadata tags + build/push with GHA cache |
| `actions/setup-terraform` | Install Terraform + TFLint |
| `actions/terraform-plan` | `terraform init` + `plan` + upsert PR comment with results |
| `actions/terraform-apply` | `terraform init` + `apply` with optional auto-approve |

Usage:

```yaml
steps:
  - uses: wiktorkowalski/github-workflows/actions/setup-terraform@master
    with:
      terraform-version: "1.12.0"
      install-tflint: "true"
```

## Notes

- All workflows default to `ubuntu-latest` runners (except Claude review which defaults to `self-hosted`)
- Use `secrets: inherit` to pass secrets from the calling workflow
- Runner labels are configurable via `runner-labels` input (JSON array)
