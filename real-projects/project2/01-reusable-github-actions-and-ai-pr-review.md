# 01 — Reusable GitHub Actions Library and AI-Assisted PR Review

A central **CI/CD workflows repository** holds reusable workflows, composite actions and scripts. Around 30 application, data and infrastructure repositories call it through small caller workflows, so one change in the central repo updates every pipeline.

## Central library layout

```
cicd-workflows/
├── .github/workflows/        # reusable workflows (workflow_call)
│   ├── claude-code-review.yml
│   ├── aws-ecr-build-push.yml
│   ├── aws-ecr-ecs-deploy.yml
│   ├── aws-ecr-ecs-qa-release-deploy.yml
│   ├── aws-ecr-ecs-on-demand-deploy.yml
│   └── test-release-image-tag-resolver.yml
├── .github/actions/          # composite actions
│   ├── claude-code-review-prompt/
│   └── hardcoded-env-check/
├── scripts/                  # Bash used by the actions
├── prompts/claude-code-review/
│   ├── default.md
│   ├── roles/…               # role checklists
│   └── repositories/<repo>.txt
├── tests/                    # shell script tests with mocks
└── docs/
```

## 1. AI code review on every pull request

### How it works

1. Each repo has a tiny **caller workflow** that triggers on `pull_request` (draft PRs are skipped) and calls the central reusable workflow.
2. The central workflow checks out the repo and **resolves a prompt**: `repositories/<repo>.txt` if it exists, otherwise `default.md`.
3. The prompt is composed from **role checklists** that match what the repo contains:

| Role checklist | Focus |
|---|---|
| `aws.md` | IAM scope, public resources, encryption |
| `opentofu.md` | State, providers, blast radius, drift |
| `containers.md` | Base images, root user, secrets in layers |
| `github.md` | Workflow permissions, untrusted input, action pinning |
| `shell.md` | Quoting, `set -euo pipefail`, injection |
| `python.md` | Error handling, dependency risk |
| `sensitive-data.md` | PII, credentials, logging of secrets |
| `privileged-automation.md` | Scripts with admin rights, destructive actions |
| `llm-agents.md` | Prompt injection, tool permissions |
| `databricks.md` | Workspace permissions, cluster/warehouse config |

4. Claude reviews the diff and **comments on the PR** with findings.

### Security and cost controls

- API key passed as a **repository/organization secret**, never in the workflow file.
- **Least-privilege token:** `contents: read`, `pull-requests: write`.
- Third-party actions **pinned to commit SHAs** (e.g. `actions/checkout`).
- **Draft PRs skipped** to control cost.

### Illustrative caller workflow

```yaml
name: AI code review
on:
  pull_request:
    types: [opened, synchronize, ready_for_review]

jobs:
  review:
    if: github.event.pull_request.draft == false
    permissions:
      contents: read
      pull-requests: write
    uses: <org>/cicd-workflows/.github/workflows/claude-code-review.yml@main
    secrets: inherit
```

### Rollout

About 29 repositories, each with its own prompt file. Infrastructure repos get IaC-specific scrutiny; data repos get PII scrutiny.

### Talking points

- **Centralising** means one change updates every repo — governance without copy-paste.
- **Per-repo prompts** keep the review relevant and reduce noise.
- **Limits:** an LLM reviewer supplements human review and SAST; it does not replace them. Findings are advisory; branch protection still requires human approval.

## 2. Hard-coded environment variable gate

- **Where:** `.github/actions/hardcoded-env-check` and `scripts/check-hardcoded-env.sh`.
- Scans **only the added lines** of the PR diff for hard-coded environment assignments.
- Ignores placeholders such as `${{ … }}` and `$VAR`.
- **Angle:** a shift-left policy check in Bash, diff-scoped so it doesn't flag legacy code — comparable to a Checkov or quality-gate check.

```bash
# Illustrative core idea
git diff --unified=0 "origin/${BASE_REF}...HEAD" \
  | grep -E '^\+[^+]' \
  | grep -E '^[+][[:space:]]*[A-Z_]+=' \
  | grep -vE '\$\{\{|\$[A-Z_]+' \
  && { echo "Hard-coded env assignment found"; exit 1; } || exit 0
```

## 3. Ticket-driven workflow with Claude 🟡

- A `workflow_dispatch` workflow takes a **ticket key** as input.
- It generates and validates a data-copy artifact with Claude.
- Only the issue-tracker REST secrets are exposed to the job, with a **15-minute timeout** and minimal permissions.
- **Angle:** ChatOps / ticket-driven automation with tightly scoped secrets.

## 4. Testing the shell scripts

`tests/test-release-image-tag-resolver.sh` tests the image-tag resolver with mocked commands, run by its own workflow. Shell scripts in the platform get CI tests like any other code.

## Interview Q&A

**Why reusable workflows instead of copy-pasted YAML?**
One source of truth, consistent security settings, and a single place to fix bugs or bump action versions.

**Reusable workflow vs composite action?**
A reusable workflow is a whole job (or jobs) with its own runner and permissions; a composite action is a bundle of steps inside a job. Use workflows for pipelines, composite actions for shared step logic.

**How do you stop a reusable workflow from being a supply-chain risk?**
Pin third-party actions to SHAs, restrict `permissions:`, limit which repos can call it, and protect the central repo with required reviews.
