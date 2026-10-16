# 02 — GitHub OIDC to AWS and ECR → ECS Container Delivery

## GitHub OIDC to AWS (no static keys)

Pipelines never store AWS access keys. GitHub issues a short-lived OIDC token per job; AWS STS exchanges it for temporary role credentials.

```
GitHub Actions job ──(OIDC token: repo, ref, workflow)──► AWS STS AssumeRoleWithWebIdentity
                                                             │  trust policy checks sub / aud
                                                             ▼
                                                   Temporary credentials (1 h)
```

- The IAM role's **trust policy is scoped to repo and branch** (e.g. only `refs/heads/main` may assume the deploy or cost-report role).
- The workflow needs `permissions: id-token: write`.

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:ref:refs/heads/main" }
  }
}
```

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@<sha>
    with:
      role-to-assume: arn:aws:iam::<account>:role/<deploy-role>
      aws-region: us-east-1
```

This is the AWS equivalent of Azure workload identity federation with managed identities (see project1).

## Reusable ECR build/push and ECS deploy workflows

**Workflows:** `aws-ecr-build-push.yml`, `aws-ecr-ecs-deploy.yml`, `aws-ecr-ecs-qa-release-deploy.yml`, `aws-ecr-ecs-on-demand-deploy.yml`
**Scripts:** `aws-ecr-ecs-deploy.sh`, `check-deployment-status.sh`

### Flow

```
Push / release
   │
   ▼
Build image ──► push to ECR (shared registry account)
   │
   ▼
Assume deploy role (OIDC, deploy account)
   │
   ▼
Register new task definition with the new image
   │
   ▼
Force ECS service rollout
   │
   ▼
Verify: PRIMARY deployment COMPLETED on the NEW task definition
   │
   ▼
Poll health endpoint until it reports the deployed version
   │
   ▼
Record image tag in SSM Parameter Store ──► Teams notification
```

### Features

- **OIDC role assumption** — no keys.
- **Cross-account ECR** — one shared registry account, separate deploy accounts per environment.
- **Task definition registered per image**, so every deploy is a new, traceable revision.
- **Forced rollout** of the ECS service.
- **Rollout verification** — the job only passes when the PRIMARY deployment is `COMPLETED` *and* is running the new task definition (not just "service stable").
- **Health polling** — the health endpoint must report the new version.
- **Version record** — the deployed image tag is written to SSM Parameter Store, so anyone (or any job) can see what's live.
- **Naming conventions** derive the ECR repository and ECS service names, so callers pass very few inputs.
- **Variants** — standard deploy, QA release deploy, on-demand deploy.

### Illustrative verification

```bash
# Wait until the PRIMARY deployment uses the new task definition and has completed
for i in $(seq 1 60); do
  read -r td state < <(aws ecs describe-services --cluster "$CLUSTER" --services "$SERVICE" \
    --query "services[0].deployments[?status=='PRIMARY'] | [0].[taskDefinition, rolloutState]" --output text)
  [[ "$td" == "$NEW_TD_ARN" && "$state" == "COMPLETED" ]] && break
  [[ "$state" == "FAILED" ]] && { echo "Rollout failed"; exit 1; }
  sleep 10
done

# Confirm the app reports the new version
curl -fsS "$HEALTH_URL" | jq -e --arg v "$IMAGE_TAG" '.version == $v'

aws ssm put-parameter --name "/deploy/$SERVICE/image-tag" --value "$IMAGE_TAG" --type String --overwrite
```

### Callers

- API service, frontend and an operations agent service (`docker_deploy.yml`).
- Seven Lambda-to-container "clone" workflows that **build only when their paths change** (`on.push.paths`).

## ECS rolling deploy vs AKS rolling update

| | ECS | AKS / Kubernetes |
|---|---|---|
| Unit of change | New task definition revision | New ReplicaSet from a changed pod template |
| Rollout control | `minimumHealthyPercent` / `maximumPercent`, deployment circuit breaker | `maxSurge` / `maxUnavailable`, readiness probes |
| Health | Target group health checks, container health checks | Readiness / liveness probes |
| Rollback | Circuit breaker auto-rollback, or redeploy previous revision | `kubectl rollout undo`, `helm rollback` |
| Verification here | PRIMARY `COMPLETED` on new TD + version endpoint | `kubectl rollout status` + smoke tests |

## Interview Q&A

**Why check the task definition, not just "service stable"?**
A service can be stable on the *old* task definition if the new one failed and rolled back. Checking the PRIMARY deployment's task definition proves the new version is the one serving.

**Why record the version in SSM?**
It gives a single source of truth for "what's deployed where", used by other jobs, dashboards and humans.

**Why a shared ECR account?**
Images are built once and promoted across environments; deploy accounts get pull-only access, which limits blast radius.
