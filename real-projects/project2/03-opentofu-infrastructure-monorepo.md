# 03 — OpenTofu Infrastructure Monorepo

One OpenTofu monorepo manages AWS (12+ accounts such as dev, stage, prod, QA, ingest and office), plus Azure, GCP, Cloudflare and the customer identity platform.

## Conventions

| Rule | Why |
|---|---|
| **One directory = one state, no workspaces** | Each environment root has its own backend and state. A plan can only touch what that directory owns, which keeps blast radius small and makes the target obvious from the path. Workspaces hide which environment you're in and share code paths too easily. |
| **No `.tfvars` files** | Defaults are baked into `variables.tf` per root, so a root is self-describing and there's no "which tfvars did you pass?" ambiguity. |
| **`global-variables` module is outputs-only** | A shared config dictionary (CIDRs, account IDs, tags, names) consumed by every root — one place to change shared values. |
| **Every environment root manages its own S3 state bucket** | Versioning, noncurrent-version expiry and access logging on the bucket. |
| **Documented bootstrap** | First apply with no backend (creates the bucket) → add the backend block → `tofu init -migrate-state`. |
| **All four US regions declared as provider aliases** | Multi-region resources without per-root provider boilerplate. |
| **IAM account alias per account** | Readable sign-in URLs and logs. |
| **`tofu init --upgrade`; lock file gitignored** | Team choice to always take latest allowed provider versions. |

### Layout (illustrative)

```
infrastructure/
├── modules/
│   ├── global-variables/        # outputs only
│   └── <module>/
├── environments-aws/
│   ├── dev01/   (backend.tf, main.tf, variables.tf, outputs.tf)
│   ├── stage01/
│   └── prod01/
├── environments-azure/
├── cloudflare/
├── identity/
├── audit/
└── docs/
```

### Bootstrap a new environment root

```bash
# 1. backend.tf commented out → creates the state bucket with local state
tofu init && tofu apply
# 2. enable the S3 backend block, then move local state into it
tofu init -migrate-state
```

## Automation

- **Pre-commit:** `tofu fmt` and `terraform-docs` (module READMEs generated from code).
- **Per-environment workflows** (one per root) call a reusable `opentofu.yml`.
- **`opentofu-pr-*`** — runs `tofu plan` and posts the plan as a PR comment.
- **`opentofu-apply-*`** — runs `tofu apply` after merge to `main`.
- Variants for **AWS (OIDC)**, **Cloudflare** and the **identity platform** providers.
- **Teams notifications** on plan/apply results.

```
PR opened ──► tofu fmt -check / validate ──► tofu plan ──► plan posted on PR ──► review + approve
merge to main ──► tofu apply (same root) ──► Teams notification
```

## Design document worth knowing: platform vs workspace ownership

A proposal to split Databricks management:

- **OpenTofu owns the platform:** workspaces, networking, storage, identity, Unity Catalog metastore-level setup.
- **Databricks Asset Bundles own workspace resources:** jobs, pipelines, notebooks — deployed by data teams with `databricks bundle deploy`.

The point is an explicit **ownership boundary** so the same resource is never managed by two tools, and data teams can ship without touching platform state. It's a good example of design-doc and trade-off thinking.

## Other design docs and runbooks on the platform

- Infrastructure executive summary for a new environment.
- Account-to-account production migration notes.
- Office/DevOps compute consolidation plan.
- Firewall rules documentation.
- Monitoring rollout runbook.
- Lambda → Fargate migration guide (container-based clone jobs with ECR CI/CD).

## Interview Q&A

**Why not workspaces?**
Workspaces share one backend config and code path; it's easy to apply to the wrong one and hard to see which environment a change targets. Separate directories make the target explicit and allow different provider versions or backends per environment.

**How do you limit blast radius?**
Small roots (one state per directory), plan review on every PR, and apply only from `main` with an OIDC role scoped to that branch.

**Lock file gitignored — trade-off?**
Simpler upgrades, but builds aren't fully reproducible. Committing `.terraform.lock.hcl` gives reproducibility at the cost of explicit upgrade PRs; be ready to argue both.

**OpenTofu vs Terraform?**
OpenTofu is the open-source (MPL) fork of Terraform, CLI-compatible with the same HCL, providers and state format.
