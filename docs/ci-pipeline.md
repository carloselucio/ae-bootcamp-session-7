# CI/CD Pipeline — Golden Path

This document explains the reusable CI/CD workflow that pavess the road for the todo-service
(and any future service that adopts it), how to onboard a new service, what each required
check validates, and how to configure the AWS OIDC secret.

## What `golden-path-ci.yml` Does

[.github/workflows/golden-path-ci.yml](../.github/workflows/golden-path-ci.yml) is a **reusable
workflow** (`on: workflow_call`) — it contains no triggers of its own. A caller workflow (like
[todo-service-ci.yml](../.github/workflows/todo-service-ci.yml)) invokes it and supplies inputs
and secrets. This means every service team gets the same validated pipeline without
copy-pasting YAML.

### Jobs and why each one exists

| Job               | Runs when                                                      | Why it exists                                                                                                                                                                                                                                                                                    |
| ----------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `lint`            | Always                                                         | Catches style/syntax issues in both `packages/backend` and `packages/frontend` before tests run, keeping feedback fast and cheap.                                                                                                                                                                |
| `test`            | Always                                                         | Runs backend Jest tests with coverage and publishes a coverage summary to the job summary. This is the primary functional safety net — it must pass on every PR and every push.                                                                                                                  |
| `security-scan`   | Only when `run_terraform_plan: true`                           | Runs Checkov against `infra/` and hard-fails on HIGH severity findings. This prevents insecure infrastructure changes (e.g. open security groups, unencrypted storage) from merging.                                                                                                             |
| `terraform-plan`  | Only when `run_terraform_plan: true`                           | Authenticates to AWS via OIDC, initializes the Terraform backend, and runs `terraform plan`, posting a summary and uploading the plan as an artifact. This gives reviewers visibility into exactly what infrastructure will change before it's applied.                                          |
| `docker-build`    | Only on `pull_request` events, after `lint`+`test` pass        | Verifies both Dockerfiles build successfully. This catches Dockerfile regressions early without needing to push images on every PR.                                                                                                                                                              |
| `terraform-apply` | Only when `run_terraform_apply: true` (after `terraform-plan`) | Re-authenticates via OIDC, downloads the previously generated plan artifact, and applies it against the real `infra/stacks/dev` stack. Restricted to pushes on `main` so infrastructure changes only roll out after a PR is merged, never from a PR branch.                                      |
| `build-and-push`  | Only when `build_and_push: true` (after `terraform-apply`)     | Resolves the ECR repository URLs from Terraform outputs, builds and pushes versioned (`sha`) and `latest` tagged images for both containers, then forces a new ECS deployment. Keeping this gated behind `terraform-apply` guarantees the ECR repos already exist before trying to push to them. |

Because every job is parameterized through `inputs`, a caller can run just `lint` + `test` on
every PR (cheap, fast feedback) while only running the AWS-touching jobs
(`terraform-plan`, `terraform-apply`, `build-and-push`) on pushes to `main`.

## How a New Service Team Adopts It

Adopting the golden path is a two-step process:

1. **Make sure your repo layout matches the convention** — an npm workspace with
   `packages/backend` and `packages/frontend`, each with a `Dockerfile`, plus a Terraform
   stack at `infra/stacks/dev` that outputs `backend_ecr_repository_url`,
   `frontend_ecr_repository_url`, and `service_url`.
2. **Add a thin caller workflow** that references the reusable workflow by path. This is the
   _only_ file a new team needs to write:

```yaml
# .github/workflows/todo-service-ci.yml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write # required for OIDC — must be declared here, not just in the reusable workflow

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

That's it — no duplicated job logic, no copy-pasted Terraform or Docker steps. Any
improvements to `golden-path-ci.yml` (e.g. a new security scanner, an updated Terraform
version) automatically apply to every team's caller workflow the next time it runs.

## What Each Required Check Validates — and Why It's Required

| Check            | Validates                                                                                                                                      | Why required                                                                                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lint`           | Code style and static-analysis rules pass for both packages                                                                                    | Keeps the codebase consistent and catches obvious bugs (unused vars, syntax issues) before they reach test/build stages.                                                           |
| `test`           | Backend Jest suite passes and coverage meets the threshold defined in [docs/testing-guidelines.md](testing-guidelines.md) (80% lines+branches) | Functional correctness is the most important signal before merging — this is the primary gate against regressions.                                                                 |
| `security-scan`  | No HIGH severity Checkov findings in `infra/`                                                                                                  | Prevents insecure infrastructure (public ingress, missing encryption, etc.) from being provisioned, satisfying FR-1.5 in [functional-requirements.md](functional-requirements.md). |
| `terraform-plan` | The Terraform configuration initializes and plans cleanly against AWS using OIDC credentials                                                   | Confirms the IaC is syntactically and semantically valid against real AWS APIs (not just `terraform validate`), and gives reviewers a diff of exactly what will change.            |

These four checks (`lint`, `test`, `security-scan`, `terraform-plan`) are the required status
checks referenced in [functional-requirements.md](functional-requirements.md) (FR-2.2) and should
be configured as required checks on the `main` branch protection rule.

## Configuring the OIDC Secret for `terraform-plan`

The `terraform-plan` (and `terraform-apply`, `build-and-push`) jobs authenticate to AWS using
OpenID Connect (OIDC) — no long-lived IAM access keys are stored in GitHub.

### 1. Create an IAM role with a trust policy for GitHub's OIDC provider

The role's trust policy must allow `token.actions.githubusercontent.com` as a federated
principal, scoped to this repository (and optionally to specific branches/environments):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:*"
        }
      }
    }
  ]
}
```

Attach a permissions policy scoped to only what Terraform needs (ECS, ECR, ALB, VPC, IAM,
CloudWatch Logs — matching what `infra/modules/todo-service` provisions).

### 2. Store the role ARN as a repository secret

```bash
gh secret set AWS_ROLE_ARN --body "arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>"
```

This corresponds to FR-1.6 in [functional-requirements.md](functional-requirements.md) —
`AWS_ROLE_ARN` must be configured in repository settings.

### 3. Confirm the caller workflow passes it through and requests the token

Two things must both be true, or the job will fail at runtime with a permissions error:

- `todo-service-ci.yml` declares `id-token: write` in its **top-level** `permissions:` block
  (not just inside `golden-path-ci.yml`) — GitHub only issues the OIDC token to workflows that
  explicitly request it at the caller level.
- The caller passes `secrets: aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}` so the reusable
  workflow receives it as `secrets.aws_role_arn`.

With both in place, `aws-actions/configure-aws-credentials@v4` exchanges the GitHub-issued OIDC
token for short-lived AWS credentials scoped to the assumed role — no static keys ever touch
the repository.
