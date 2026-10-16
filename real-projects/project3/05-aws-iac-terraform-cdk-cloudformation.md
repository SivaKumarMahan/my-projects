# 05 — AWS IaC: Terraform, CDK and CloudFormation

The same IaC toolkit has an AWS side, used to stand up VPC-based environments: Terraform run inside Docker from Jenkins, an AWS CDK (Python) stack with input validation, and a CloudFormation template library.

## Terraform run in Docker from a parameterised Jenkins job

```
Jenkins (choose WORKSPACE, ACTION, MODULE, ROLE_ARN)
   │  AWS credentials from Jenkins credentials store
   ▼
docker compose build ──► image FROM hashicorp/terraform
                          terraform init (S3 backend, assume role)
                          terraform workspace select <env> || new <env>
                          terraform plan|apply|destroy -target=<module>
```

### Jenkins parameters

| Parameter | Choices |
|---|---|
| `WORKSPACE` | `dev`, `stage`, `prod` |
| `ACTION` | `plan`, `apply`, `destroy` |
| `MODULE` | `module.vpc`, `module.ec2`, `module.alb`, `module.s3`, `module.dns`, `module.iam`, `module.cloudfront`, `module.cloudwatch`, VPC peering modules, `module.acm_certificates` |
| `ROLE_ARN` | IAM role Terraform assumes in the target account |

```groovy
withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: '<aws-creds>',
                  accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
  sh "docker compose -f ./aws/terraform/environments/docker-compose.yaml build \
        --build-arg workspace=${params.WORKSPACE} \
        --build-arg action=${params.ACTION} \
        --build-arg module=${params.MODULE} \
        --build-arg ROLE_ARN=${params.ROLE_ARN}"
}
```

### Terraform design

- **Workspaces per environment** with maps keyed by `terraform.workspace` (`env`, `vpc_cidr` → dev `10.91.0.0/16`, stage `10.92.0.0/16`, shared `10.93.0.0/16`).
- **S3 remote state**; each workspace stores state under `env:/<workspace>/terraform.tfstate`.
- **Optional assume-role** via a `dynamic` block — empty `ROLE_ARN` means no assume role:

```hcl
provider "aws" {
  region = var.region
  dynamic "assume_role" {
    for_each = var.ROLE_ARN != "" ? { role_arn = var.ROLE_ARN } : {}
    content { role_arn = assume_role.value }
  }
}
```

- **Shared modules from a separate Git repo** (`vpc`, `vpcpeering`, `ec2`, `s3`, `dns`, `acm`, `cloudfront`, `iam`, `alb`, `cloudwatch`).
- **Cross-workspace data** with `terraform_remote_state`: a shared VPC reads the dev and stage state to peer with them (VPC IDs, CIDRs, route tables).

```hcl
data "terraform_remote_state" "dev" {
  backend = "s3"
  config  = { bucket = "<state-bucket>", key = "env:/dev/terraform.tfstate", region = "us-east-1" }
}

module "vpcpeering_shared_dev" {
  source              = "git::https://github.com/<org>/terraform-modules.git//aws/services/vpcpeering?ref=<tag>"
  vpc_id_this         = local.vpc_id_this
  peer_vpc_id         = data.terraform_remote_state.dev.outputs.vpc_id
  peer_cidr_block     = data.terraform_remote_state.dev.outputs.vpc_cidr
  route_table_id_peer = data.terraform_remote_state.dev.outputs.aws_route_table_id
}
```

- **Workload stack:** VPC with self-managed NAT instance, EC2, ALB with launch template + Auto Scaling group + health checks + access logs to S3, CloudWatch scale-out/scale-in policies and dashboard, Route 53, ACM with SANs per workspace, CloudFront for S3 static content, IAM role + instance profile for EC2 → S3.
- **`compact()` with workspace conditions** to add the right SANs / aliases per environment.

### What I'd change (and how to explain it)

| Issue | Better approach |
|---|---|
| AWS keys passed as **Docker build args** and baked into image layers via `ENV` | Pass credentials at **run** time (`docker run -e` or an instance role / OIDC); use `docker run`, not `docker build`, to execute Terraform |
| Terraform executed in a `RUN` step during image build | Build a tooling image once; run `terraform` in a container step so output, exit codes and plans are visible |
| `-target` for every run | Fine for a module menu, but it skips dependency checks — use separate root modules/states instead |
| `apply`/`destroy` with `-auto-approve` straight from a parameter | Save the plan (`-out`), add a Jenkins `input` approval, then apply that plan |
| `hashicorp/terraform:latest`, provider `>=` constraints | Pin Terraform and provider versions; commit the lock file |
| Module source with a token in the URL | Use SSH deploy keys or a Git credential helper and pin `?ref=<tag>` |
| Security group allowing all traffic from `0.0.0.0/0` | Only 80/443 from anywhere; admin ports from the VPN/office range |

## AWS CDK (Python) with config validation

```
config.yaml ─► validations.py (account, region, CIDR checks) ─► app.py ─► CdkStack (VPC, subnets, SGs, EC2, DB)
                                                                 │
Jenkins ─► docker compose build ─► python:3.10-alpine + node + pipenv ─► cdk synth ─► cdk bootstrap/deploy
```

```yaml
# config.yaml
CDK_DEFAULT_ACCOUNT: "<12-digit-account>"
CDK_DEFAULT_REGION: "us-east-1"
VPC_NAME: "cdk-VPC"
VPC_CIDR: "10.0.0.0/16"     # must be /16
MAX_AZS: 3
SUBNET_NAME_1: "Web-subnet"
SUBNET_CIDR_1: 24           # must be /24
SUBNET_NAME_2: "DB_subnet"
SUBNET_CIDR_2: 24
SG_1: "Web-SG"
SG_2: "DB-SG"
INSTANCE_NAME: "webserver"
INSTANCE_TYPE: "t2.micro"
KEY_NAME: "<existing-key-pair>"
```

`validations.py` checks the account is 12 digits, the region matches an AWS region pattern, the VPC CIDR is a `/16` and both subnets are `/24`, and stops before synthesis if anything is wrong. The Docker image runs as a **non-root `cdk` user**.

```python
app = cdk.App()
CdkStack(app, "CdkStack", env=cdk.Environment(account=CDK_DEFAULT_ACCOUNT, region=CDK_DEFAULT_REGION))
app.synth()
```

**Improvements:** the account check only computed a comparison without asserting it (use `if not (...): sys.exit(1)`), validation failures for CIDRs only printed a message instead of exiting, and the regex accepted invalid octets — `ipaddress.ip_network()` is a cleaner validator. Exit with a non-zero code so Jenkins marks the build failed.

## CloudFormation template library

| Template | Resources |
|---|---|
| VPC | VPC, IGW, 3 public + 3 private subnets, self-hosted NAT instance, SSM-managed instance role, route tables |
| EC2 | Instance with EBS volume, SSM role + instance profile, security group |
| ALB | Security group, ALB, target group, HTTP + HTTPS listeners (ACM cert) |
| RDS | DB subnet group, **Secrets Manager** secret for credentials, RDS instance, security group |
| WAF | WAFv2 Web ACL + association with the ALB |
| Data Lifecycle Manager | IAM role + DLM policy for scheduled EBS snapshots / AMIs |
| ECS on EC2 | ECS cluster, ALB + listener rule + target group, CloudWatch log group, task definition, launch configuration + ASG for container instances, ECS service + role |
| Organizations | New member **account** in an organizational unit |

The account-creation template is deployed from Jenkins with the AWS pipeline plugin:

```groovy
withAWS(credentials: env.credentials, region: env.region) {
  cfnUpdate(stack: "${env.environment}-${env.product}-ou-account",
            params: [environment: env.environment, product: env.product, email: env.email],
            file: 'aws/cloudformation/services/Organization/ou.yaml',
            timeoutInMinutes: 30)
}
```

Folder-level properties (`withFolderProperties`) supply environment, product, email and credentials per Jenkins folder, so one Jenkinsfile serves many accounts.

## Interview Q&A

**Workspaces or directories for environments?**
This toolkit used workspaces with maps keyed by `terraform.workspace`. It's compact, but easy to apply to the wrong workspace; separate directories/states make the target explicit (see project2).

**How do two Terraform states share data?**
`terraform_remote_state` reads outputs from another state (here VPC IDs and route tables for peering). Alternatives are SSM parameters or data sources that look resources up by tag.

**Why validate CDK config before synth?**
Fail fast with a clear message instead of a half-deployed CloudFormation stack.

**Why Secrets Manager in the RDS template?**
The DB password is generated and stored by AWS and referenced dynamically, so it never appears in the template or parameters.
