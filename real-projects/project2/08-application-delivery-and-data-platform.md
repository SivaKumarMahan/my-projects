# 08 — Application Delivery and Data Platform

## Application services

| Service | What's in the repo |
|---|---|
| **API** | Dockerfile, docker-compose, `build.sh`, `docker_deploy.yml` (calls the reusable ECS deploy), `check_deployment_status.sh`, env example files, identity-provider debug docs |
| **Frontend** | Dockerfile, docker-compose, ESLint, the same deploy workflow |
| **Ops agent / next-gen agent services** | FastAPI-style services with FusionAuth authentication, tests, Dockerfile, planning notes |

All services deploy through the shared ECR → ECS workflow with rollout verification ([02](02-github-oidc-and-ecs-container-delivery.md)).

### Local-to-cloud parity

- `docker-compose` for local runs with the same image as production.
- `.env.example` files document required variables; real values come from SSM / secrets at deploy time.

## Data platform CI/CD with Databricks Asset Bundles

- Pipelines are defined as **jobs as code** in `databricks.yml`.
- Deployed and run with `databricks bundle deploy -t <target>` and `databricks bundle run <job>`.
- Targets per environment (dev / prod) in the same bundle.

```yaml
# Illustrative databricks.yml
bundle:
  name: viewership_pipeline
targets:
  dev:  { mode: development, workspace: { host: https://<dev-workspace> } }
  prod: { mode: production,  workspace: { host: https://<prod-workspace> } }
resources:
  jobs:
    daily_ingest:
      name: daily-ingest
      tasks:
        - task_key: ingest
          notebook_task: { notebook_path: ./notebooks/ingest.py }
```

- Platform resources (workspaces, networking, identity) stay in OpenTofu; workspace resources (jobs, pipelines) live in bundles — see the ownership-boundary design in [03](03-opentofu-infrastructure-monorepo.md).
- Data pipeline repositories also have the AI PR review wired in. They are mostly data-engineering code, so they are lower priority for DevOps interviews.

## Interview Q&A

**How do you do CI/CD for a data platform?**
Code and job definitions in git, bundles deployed per target from CI, platform infra in IaC, PR review on both, and separate environments with promotion from dev to prod.

**Why split OpenTofu and Asset Bundles?**
Platform and workspace resources change at different speeds and are owned by different teams; one tool per resource avoids two tools fighting over the same object.
