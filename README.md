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

### `.NET CI`

PR checks for .NET: restore → format check (`dotnet format --verify-no-changes`) → build with warnings-as-errors → test with TRX report + Cobertura coverage summary. Point `dotnet-project` at a solution (or leave empty for the repo-root solution) so tests are included.

Notes:

- **`treat-warnings-as-errors` (default `true`) is the most likely first-run failure** for an existing repo — it promotes latent compiler/analyzer warnings to errors. Fix them, or pass `treat-warnings-as-errors: false` to adopt incrementally.
- Coverage requires each test project to reference the `coverlet.collector` NuGet package (otherwise the collector silently produces nothing). The coverage-summary step is a Docker action, so it only runs on Linux runners with Docker; it's non-fatal and skips cleanly elsewhere.
- `dotnet-project` empty resolves a root `.sln`/`.slnx` (SDK 9+) or single root project. Multi-project repos with no root solution must pass an explicit path. `.slnx` is not supported on SDK 8.x.
- If a repo already runs `dotnet test` via `.NET Docker Deploy`, this workflow duplicates the test run — use it to add the format/warnings/coverage gate on PRs, and consider whether you still need tests in the deploy path.

```yaml
name: CI
on:
  pull_request:
    branches: [master]

jobs:
  dotnet-ci:
    uses: wiktorkowalski/github-workflows/.github/workflows/dotnet-ci.yml@master
    with:
      dotnet-project: MyApp.sln  # empty = solution in repo root
      # dotnet-version: "10.0.x"           # default
      # treat-warnings-as-errors: true     # default
      # run-format: true                   # default
      # collect-coverage: true             # default
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
      # build-artifact-name: lambda-zips  # artifact from a caller build job, downloaded before init in the plan job; empty = skip
      # build-artifact-path: dist/  # where to download it, relative to repo root (default ".")
      # run-plan: false  # skip the plan job (e.g. backend not bootstrapped yet); lint/format/validate still run
    secrets: inherit
```

Supported secret passthrough: `TF_API_TOKEN`, `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `HCLOUD_TOKEN`.

If the Terraform config references build outputs (e.g. Lambda zips + `source_code_hash`), build them in a caller job and hand them over as an artifact — the workflow stays language-agnostic:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: wiktorkowalski/github-workflows/actions/setup-dotnet@master
        with:
          dotnet-version: "10.0.x"
      - run: ./build.sh  # produce dist/*.zip
      - uses: actions/upload-artifact@v7
        with:
          name: lambda-zips
          path: dist/

  terraform:
    needs: build
    uses: wiktorkowalski/github-workflows/.github/workflows/terraform-pr.yml@master
    with:
      working-directory: terraform/
      build-artifact-name: lambda-zips
      build-artifact-path: dist/
    secrets: inherit
```

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
      # build-artifact-name: lambda-zips  # artifact from a caller build job, downloaded before init in both plan and apply jobs; empty = skip
      # build-artifact-path: dist/  # where to download it, relative to repo root (default ".")
    secrets: inherit
```

Build-artifact handoff works the same as in Terraform PR (see above) — build once in a caller job, and plan and apply consume the exact same zips (matters when the build is nondeterministic).

### `Claude Code Review`

AI code review on every PR push. Posts inline comments with fix suggestions, then gates: the job fails unless Claude submitted a formal review approving the **current** head commit (stale approvals and missing reviews are both red). Draft PRs are skipped.

```yaml
name: Claude Code
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  review:
    uses: wiktorkowalski/github-workflows/.github/workflows/claude-code-review.yml@master
    with:
      # model: claude-opus-4-8  # default
      # max-turns: 60  # default; controls cost
      # timeout-minutes: 30  # default; frees a stuck runner slot
      # review-prompt: "Also check for Go-specific issues."  # appended to default
    secrets: inherit
```

Required secret: `CLAUDE_CODE_OAUTH_TOKEN` (generate via `claude setup-token`; recommend setting as an org-level secret so all consuming repos inherit it).

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

- All workflows default to `ubuntu-latest` runners (except Claude review which defaults to `["self-hosted", "homelab"]`)
- Use `secrets: inherit` to pass secrets from the calling workflow
- Runner labels are configurable via `runner-labels` input (JSON array)
