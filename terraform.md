# Terraform — Staff SWE Interview Reference

## Core Concepts Depth Check

### State Management

Terraform state is a source of truth mapping real-world resources to config. At staff level, understand:

- **Remote state:** S3 + DynamoDB (AWS), GCS, Terraform Cloud. Local state is never acceptable in teams.
- **State locking:** DynamoDB TTL-based lock prevents concurrent `apply` from corrupting state. Lock is acquired at plan start, released on completion or error.
- **State isolation:** One state file per environment × per component. Not one giant state for everything.
- **`terraform state` subcommands:** `mv`, `rm`, `show`, `pull`, `push` — know when each is appropriate.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"
    encrypt        = true
  }
}
```

**State file contains:**
- Resource IDs (not configs) — what exists in the provider
- Metadata: serial number, lineage (UUID per `terraform init`)
- Outputs and dependencies

**Lineage:** If two state files have different lineages, Terraform will refuse to use one as a replacement for the other — prevents accidental overwrites.

---

### `terraform plan` internals

1. Refresh: query actual provider state for every tracked resource
2. Build dependency graph (DAG) from `depends_on`, references, resource order
3. Diff actual vs desired
4. Output plan (create/update/destroy/no-op per resource)

`-refresh=false` skips step 1 — faster but risks plan diverging from reality. Use only in CI when you know nothing drifted.

`-target=resource.name` applies only to that resource and its dependencies. Useful for unblocking but leaves state partially applied — always follow up with an untargeted apply.

---

### Resource Lifecycle

```hcl
resource "aws_instance" "app" {
  # ...
  lifecycle {
    create_before_destroy = true   # spin up replacement before destroying old
    prevent_destroy       = true   # block plan if this resource would be destroyed
    ignore_changes        = [tags] # don't track drift on these attributes
    replace_triggered_by  = [aws_launch_template.app.latest_version] # force replace when this changes
  }
}
```

`create_before_destroy` is critical for zero-downtime replacements but requires resources downstream (e.g., ASG) to also set it, otherwise destroy happens before attachment is severed.

---

## Staff-Level Interview Questions

### Q1: How do you structure Terraform for a 200-service platform across 5 AWS accounts and 3 regions?

**Answer framework:**

**Wrong answer:** One repo, one state, `terraform workspace` per env.

**Right answer — separation of concerns:**

```
infra/
  _modules/          # reusable modules, versioned via git tags
    vpc/
    eks-cluster/
    rds/
  accounts/
    prod/
      us-east-1/
        networking/    # state: prod/us-east-1/networking
        eks/           # state: prod/us-east-1/eks
        databases/     # state: prod/us-east-1/databases
      eu-west-1/
        networking/
        eks/
    staging/
      us-east-1/
        networking/
        eks/
```

**Why not workspaces for environments?**
- Workspaces share a backend config — blast radius of a wrong `apply` is too high
- Different environments often need different provider configs (account IDs, regions)
- State files per workspace live in the same bucket path — harder ACL isolation

**Remote state references between components:**
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "prod/us-east-1/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids
  }
}
```

**Module versioning:**
```hcl
module "vpc" {
  source  = "git::ssh://git@github.com/org/infra-modules.git//vpc?ref=v2.3.1"
  # ...
}
```
Pin to git tags, never `main`. Module changes go through PR + version bump.

---

### Q2: A `terraform apply` destroyed a production RDS instance that was supposed to be recreated. What happened and how do you prevent it?

**What happened:**
- A change to an immutable attribute (e.g., `engine_version`, `db_subnet_group_name`, `storage_encrypted`) triggered destroy + create
- `create_before_destroy = true` was not set → Terraform destroys first
- If `create_before_destroy` WAS set but dependent resources (security groups, subnet group) don't have it, the old instance was destroyed before the new one could bind

**Prevention checklist:**
1. `prevent_destroy = true` on stateful resources (RDS, S3, EFS). This causes `plan` to fail if a destroy is proposed.
2. Review plans for `# forces replacement` warnings in CI before merging
3. Use `terraform plan -out=plan.tfplan` in CI; require human approval for any destroy
4. For immutable attribute changes: use `create_before_destroy` + verify dependency chain also has it
5. OPA/Sentinel policy: block plans containing `destroy` on resource types matching `aws_db_instance`, `aws_s3_bucket`, etc.

**Recovery:** Restore from automated snapshot. Document RTO/RPO impact. Stateful resources must have automated daily snapshots and tested restore procedures independent of Terraform.

---

### Q3: How do you refactor a monolithic 5,000-resource state file into smaller states without downtime?

**Why it's a problem:**
- Every `plan`/`apply` refreshes all 5,000 resources — slow (10+ minutes)
- A failed apply can leave thousands of resources in an intermediate state
- Team ownership is unclear when one state owns everything

**Approach — `terraform state mv` to new state:**

Step 1: Define target module structure with empty state files
Step 2: For each resource moving to a new state:
```bash
# In the source state
terraform state mv \
  'aws_security_group.app' \
  'module.networking.aws_security_group.app'

# To move to a different state backend:
# 1. terraform state pull > old.tfstate
# 2. terraform state mv in the new workspace
# Actually use: terraform state mv -state-out=new.tfstate ...
```

Step 3: Add `data "terraform_remote_state"` references where cross-state deps exist
Step 4: Import moved resources into new state (or use `terraform state push`)
Step 5: Remove from old state with `terraform state rm` after confirming new state tracks them

**Critical:** Never run `apply` in the old config after `state rm` — it will try to recreate resources. Remove the HCL from old config simultaneously with the state move.

**Terragrunt `run-all` can orchestrate the split** while maintaining dependency ordering.

---

### Q4: How do you handle secrets in Terraform? What are the anti-patterns?

**Anti-patterns:**
- Hardcoding secrets in `.tf` files (committed to git)
- Passing secrets as `-var` flags in CI logs
- Storing secrets in outputs (they appear in state file in plaintext)

**Patterns:**

```hcl
# Pattern 1: Reference existing secret by ARN, never the value
resource "aws_ecs_task_definition" "app" {
  container_definitions = jsonencode([{
    secrets = [{
      name      = "DB_PASSWORD"
      valueFrom = "arn:aws:secretsmanager:us-east-1:123:secret:prod/db/password"
    }]
  }])
}

# Pattern 2: Generate secret, store in Secrets Manager, reference ARN
resource "random_password" "db" {
  length = 32
}
resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
  # This value IS in state — state file must be encrypted at rest
}
```

**State file encryption:** S3 backend with `encrypt = true` + KMS CMK. Restrict KMS key policy to CI role and break-glass admins only.

**Vault provider:**
```hcl
data "vault_generic_secret" "db_creds" {
  path = "secret/prod/database"
}
# Use data.vault_generic_secret.db_creds.data["password"]
# PROBLEM: still ends up in state file
```

**Best practice for credentials that must be in state:** Encrypt state with a KMS key that rotates annually; audit KMS decrypt calls via CloudTrail; restrict who can `terraform state pull`.

---

### Q5: What is Terraform's dependency graph and when does `depends_on` become necessary?

Terraform builds a DAG from **implicit references**: if resource B references `resource.A.id`, Terraform knows A must exist before B.

`depends_on` is needed for **non-obvious ordering** that Terraform can't infer:

```hcl
# IAM policy attachment before EKS pod identity association
resource "aws_eks_pod_identity_association" "app" {
  depends_on = [aws_iam_role_policy_attachment.app]
  # Without this: pod association created before IAM policy propagates → startup failure
}

# Wait for DB to be fully ready before running migration job
resource "kubernetes_job" "db_migrate" {
  depends_on = [helm_release.postgres]
}
```

**`depends_on` on modules:** Forces entire module to complete before the dependent resource is planned — coarse-grained and can slow parallelism. Prefer explicit references where possible.

---

### Q6: How do you test Terraform at scale?

**Layers:**

1. **Static analysis / linting:**
    - `terraform validate` — syntax and internal consistency
    - `tflint` — provider-specific rule enforcement (e.g., invalid instance types)
    - `checkov` / `tfsec` — security policy scanning (open ports, unencrypted resources)

2. **Plan-based tests (no real infra):**
    - OPA/Conftest against plan JSON: `terraform plan -out=plan.tfplan && terraform show -json plan.tfplan | conftest test -`
    - Test policies like "no public S3 buckets", "all resources must have cost-center tag"

3. **Integration tests (real infra, ephemeral):**
   ```go
   // Terratest
   func TestEKSCluster(t *testing.T) {
       opts := &terraform.Options{TerraformDir: "../modules/eks"}
       defer terraform.Destroy(t, opts)
       terraform.InitAndApply(t, opts)
       
       clusterName := terraform.Output(t, opts, "cluster_name")
       assert.NotEmpty(t, clusterName)
       // Additional API calls to verify cluster is functional
   }
   ```
    - Run in isolated AWS account with budget alerts
    - Parallelize with `t.Parallel()` + unique resource names via `random_id`

4. **Drift detection:** Scheduled CI job running `terraform plan -detailed-exitcode` — exit code 2 means drift detected, triggers alert.

---

### Q7: Blue/green deployment of an EKS cluster with Terraform — how?

**Challenge:** You can't rename a cluster or change certain immutable fields without destroy/recreate. Users need zero downtime.

**Approach:**

```hcl
# Use a variable to switch between blue and green
variable "active_color" {
  default = "blue"
}

module "eks_blue" {
  source       = "../modules/eks"
  cluster_name = "prod-blue"
  count        = var.active_color == "blue" ? 1 : 0
}

module "eks_green" {
  source       = "../modules/eks"
  cluster_name = "prod-green"
  count        = var.active_color == "green" ? 1 : 0
}

# Route 53 / load balancer weighted routing controlled separately
```

**Cutover process:**
1. Apply with `active_color = "green"` → creates green cluster alongside blue
2. Deploy workloads to green; validate
3. Shift traffic 10% → 50% → 100% via weighted routing (outside Terraform or managed in a separate TF state to avoid coupling)
4. Apply with `active_color = "blue"` removed → blue destroyed after traffic fully cut

**Alternative:** Use Terragrunt's `--include` to manage blue/green as separate environment directories sharing the same module.

---

## Production Case Studies

### Case Study 1: State Lock Deadlock During Incident

**Scenario:** A `terraform apply` was killed mid-run during a production incident (Ctrl+C after deploy started). The DynamoDB lock was held and never released. The on-call engineer tried to apply a critical hotfix and got `Error: state is locked`.

**Investigation:**
- `terraform force-unlock <lock-id>` requires the lock ID from the error message or DynamoDB table
- Check DynamoDB: `aws dynamodb get-item --table-name tf-state-lock --key '{"LockID": {"S": "..."}}'`
- Verify the previous apply process is truly dead (check CI job status, no running containers)

**Resolution:**
```bash
# Confirm no active process holds the lock
terraform force-unlock -force LOCK_ID
```

**Prevention:**
- CI pipelines must trap SIGTERM and run `terraform force-unlock` on cancellation
- Set DynamoDB lock TTL (custom Lambda cleanup job) — Terraform itself doesn't expire stale locks
- Distinguish between locks from interrupted applies (safe to force-unlock) and concurrent applies (never force-unlock)

---

### Case Study 2: Provider Upgrade Broke 200 Resources

**Scenario:** Team bumped AWS provider from `4.x` to `5.x` across a monorepo. 200 resources showed plan diffs because `aws_security_group` removed inline `ingress`/`egress` blocks in favor of separate `aws_security_group_rule` resources.

**What happened without careful planning:**
- Plans showed destroy + recreate for security groups
- Brief period with no security group rules → inbound traffic blocked
- 6 separate teams trying to apply simultaneously caused state conflicts

**Structured migration:**
1. **Inventory:** `grep -r "aws_security_group" . --include="*.tf" -l` → 47 files
2. **Test in non-prod:** Apply provider upgrade in staging, observe diffs
3. **Migration script:** Write a script to convert inline rules to `aws_security_group_rule` HCL
4. **Lifecycle trick:**
   ```hcl
   resource "aws_security_group" "app" {
     lifecycle { ignore_changes = [ingress, egress] }
     # Existing inline rules kept temporarily during migration
   }
   ```
5. **Phased rollout:** One team per week, with review of plan output before each apply
6. **Lock provider version in root module:** `required_providers { aws = { version = "~> 5.0" } }`

---

### Case Study 3: Accidental `terraform destroy` in Production

**Scenario:** Engineer ran `terraform destroy` intending to destroy a staging environment. The shell had wrong AWS_PROFILE set — it hit production.

**Immediate response:**
1. Stop any destroy in progress: Ctrl+C (Terraform doesn't have a pause — this leaves partial state)
2. Check state: `terraform state list` to see what's already destroyed vs still tracked
3. For destroyed compute resources: restore from snapshots (RDS, EBS)
4. For destroyed networking (VPC, subnets): recreate via `terraform apply` with the same config — re-creates resources but with new IDs
5. For resources that changed ID: update all downstream references

**Root cause fix:**
- Require MFA for production AWS profile: `aws sts get-caller-identity` as first CI step
- Add `prevent_destroy = true` to all stateful resources
- Require two-person approval for `terraform destroy` via CI (never run locally against prod)
- Mandatory AWS_PROFILE prompt: shell alias that prints account name before running any `terraform` command

---

### Case Study 4: Terragrunt at Scale — Parallelizing 80 Modules

**Scenario:** A full `terragrunt run-all apply` across 80 modules took 45 minutes due to sequential execution. Every morning deploy blocked the team.

**Analysis:**
- `terragrunt run-all` respects `dependency` blocks and only runs modules in dependency order
- Without explicit dependencies, it runs sequentially by default
- Many modules (e.g., 15 independent microservice ECS service modules) had no cross-dependencies but weren't running in parallel

**Fix:**
```hcl
# terragrunt.hcl for each microservice
dependency "cluster" {
  config_path = "../../ecs-cluster"
}
dependency "vpc" {
  config_path = "../../networking"
}
# No dependency on sibling microservices → runs in parallel

inputs = {
  cluster_arn = dependency.cluster.outputs.cluster_arn
  vpc_id      = dependency.vpc.outputs.vpc_id
}
```

```bash
# Run with explicit parallelism
terragrunt run-all apply --terragrunt-parallelism 10
```

**Result:** 45 min → 12 min. The bottleneck became the longest sequential dependency chain (networking → ECS cluster → services), not the total count of modules.

**Additional optimization:** `--terragrunt-fetch-dependency-output-from-state` reads outputs directly from state without running `terraform output` per dependency (cuts ~20 s per dependency lookup at scale).

---

### Case Study 5: Drift Detection Revealing Shadow Infrastructure

**Scenario:** Quarterly compliance audit found 34 EC2 instances running in prod that were not in any Terraform state. No one knew who created them.

**Detection:**
```bash
# Scheduled CI job
terraform plan -detailed-exitcode -refresh-only
# Exit code 2 = drift detected
# Parse plan JSON for resource additions not in state
```

**Root cause:** Engineers had been creating EC2 instances via AWS Console for "quick tests" and never cleaning up. No guardrails prevented Console access.

**Remediation options per untracked resource:**
1. **Import into state:** If it should exist and matches a config
   ```bash
   terraform import aws_instance.legacy_worker i-0abc123def456789
   ```
2. **Tag and schedule deletion:** If it's genuinely temporary
3. **Write HCL and import:** If it should be managed going forward

**Prevention:**
- SCP (Service Control Policy): Block `ec2:RunInstances` for non-Terraform IAM roles in prod account
- AWS Config rule alerting on untagged resources (all Terraform-managed resources must have `managed-by = terraform` tag)
- Terraform Cloud or Atlantis: Centralized apply — no local `terraform apply` against prod

---

### Case Study 6: Zero-Downtime RDS Major Version Upgrade

**Scenario:** PostgreSQL 13 → 16 upgrade required. AWS requires destroy + recreate for major versions on RDS (not in-place). 2 TB database, 99.9% SLO.

**Approach using Terraform:**

```hcl
# Phase 1: Add replica pointing at new version (Blue/Green in RDS)
resource "aws_db_instance" "primary_v16" {
  identifier        = "prod-postgres-v16"
  engine_version    = "16.2"
  # replicate_source_db = existing instance (for initial sync)
  # Can't be done purely in Terraform for cross-version replication
  # Use AWS RDS Blue/Green deployment API
}
```

**Actual process:**
1. Enable RDS Blue/Green Deployments via AWS console or CLI (Terraform `aws_rds_cluster_blue_green_deployment` resource — GA since provider v5.22)
2. Terraform manages switchover timing:
   ```hcl
   resource "aws_rds_blue_green_deployment" "pg_upgrade" {
     source          = aws_db_instance.primary.arn
     target_engine_version = "16.2"
   }
   ```
3. AWS handles replication and switchover (< 1 min failover)
4. After switchover: update Terraform state to reference new instance, remove old

**Lesson:** Some zero-downtime operations require coordination between Terraform and cloud-native features (Blue/Green, read replicas). Terraform manages state tracking; the actual migration uses provider APIs.

---

### Case Study 7: Multi-Account Terraform with Assume Role

**Scenario:** Central CI account needs to manage resources across 12 AWS accounts without storing per-account credentials.

```hcl
# Root provider: CI account credentials from environment
provider "aws" {
  region = "us-east-1"
}

# Per-account providers using assume_role
provider "aws" {
  alias  = "prod_networking"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformExecutionRole"
  }
}

provider "aws" {
  alias  = "staging_networking"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::STAGING_ACCOUNT_ID:role/TerraformExecutionRole"
  }
}

module "prod_vpc" {
  source    = "../modules/vpc"
  providers = { aws = aws.prod_networking }
}
```

**`TerraformExecutionRole` trust policy** (in each target account):
```json
{
  "Principal": {
    "AWS": "arn:aws:iam::CI_ACCOUNT_ID:role/CIExecutionRole"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "terraform-ci-unique-id"
    }
  }
}
```

**State isolation:** Each account × region × component has its own S3 key under the central state bucket, with bucket policy restricting access per-account role.

---

## Common Gotchas at Staff Level

| Scenario | Gotcha | Fix |
|----------|--------|-----|
| `count` vs `for_each` | Removing middle element from `count` list re-indexes, causing destroy+recreate of all following resources | Use `for_each` with stable keys for any list that may change |
| `terraform refresh` on large state | Hits API rate limits, slow, can cause plan inconsistencies | Use `-refresh=false` in CI with scheduled drift detection jobs |
| Module outputs as sensitive | Sensitive outputs propagate — all downstream resources referencing them will show `(sensitive value)` in plan | Annotate `sensitive = true` explicitly; document what's sensitive |
| `lifecycle.ignore_changes` and drift | Terraform ignores real drift on ignored attributes — you can't detect manual changes to ignored fields | Use `ignore_changes` sparingly; document why |
| Provider aliasing and default provider | Forgetting to pass `providers` to a module means it uses the default provider — wrong account | Always explicit `providers` block in module calls for multi-account setups |
| `data` source caching | Data sources are evaluated at plan time; if a resource they reference was just created in the same apply, the data source may return stale values | Use direct resource references (`resource.name.attribute`) not data sources for same-run resources |
| State file size | 5,000+ resources → multi-MB JSON state → slow S3 operations, long plan times | Split states; use `-target` only as tactical measure, not ongoing practice |

---

## Atlantis / CI Workflow at Scale

```
PR opened
  → atlantis plan (automatic)
  → Post plan diff as PR comment
  → Policy checks run (OPA/Conftest against plan JSON)
  → Security scan (checkov)
  → Required reviewers approve
  → PR author comments "atlantis apply"
  → Atlantis applies, posts output to PR
  → Merge blocked until apply succeeds
```

**Parallelism with per-repo locking:** Atlantis locks a workspace during plan/apply. Multiple PRs touching the same workspace queue. This prevents concurrent applies but can cause bottleneck — mitigate by splitting states so PRs rarely touch the same workspace.

---

## Terraform Cloud vs Self-Hosted (Atlantis) — Staff Decision Framework

| Factor | Terraform Cloud | Atlantis (self-hosted) |
|--------|----------------|----------------------|
| State management | Built-in, encrypted | You manage S3 + DynamoDB |
| Sentinel policies | Built-in (Business tier) | OPA/Conftest via CI scripts |
| Run history / audit | Built-in UI | CI logs (ephemeral unless stored) |
| Cost at scale | $$$  at 50+ users (Plus tier) | Infra cost only |
| Air-gapped env | Not possible | Possible |
| Custom run environments | Limited | Full control (Docker image) |
| VCS integration | GitHub/GitLab/Bitbucket native | GitHub/GitLab/Bitbucket webhooks |

**Recommendation for staff interviews:** "We chose Atlantis because we need full control over the execution environment for compliance reasons and to run in our private VPC. For a greenfield team without compliance constraints, Terraform Cloud reduces operational overhead significantly."
