# A Workflow for Deploying Infrastructure Code with Terraform

*Day 21 of the #30DayTerraformChallenge*

---


Today I ran that workflow for real — against actual infrastructure — and discovered exactly where infrastructure code deployment requires more discipline than application code.

A bad application deployment returns a 500 error. A bad infrastructure deployment can delete a production database. Those are not the same risk level and they do not deserve the same safeguards.

Today I deployed a real infrastructure change through the full seven-step workflow, added four infrastructure-specific safeguards, and explored Sentinel policies as the enforcement layer that makes the whole system trustworthy at scale.

---

## The Infrastructure Change I Deployed

To make this concrete, I added a CloudWatch CPU alarm to the webserver cluster. This is a meaningful, production-relevant change — not just a tag update.

The alarm fires when CPU utilisation exceeds 80% for two consecutive 2-minute periods:

```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.cluster_name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU utilization exceeded 80% for ${var.cluster_name}"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

output "cloudwatch_alarm_arn" {
  value       = aws_cloudwatch_metric_alarm.high_cpu.arn
  description = "ARN of the CPU utilization alarm"
}
```

Now let me walk through every step.

---

## Step 1 — Version Control

Confirmed branch protection rules on `main`:

- Minimum 1 reviewer approval required before merge
- Status checks (CI) must pass before merge is allowed
- Direct pushes to `main` blocked — all changes via pull requests

This is the same protection application code gets. Infrastructure deserves it even more — a direct push to main that triggers an auto-apply could modify production infrastructure with no review.

---

## Step 2 — Run the Code Locally

The infrastructure equivalent of running an application locally is running `terraform plan`. It generates a diff between the current state and the desired configuration — without touching anything.

```bash
terraform workspace select dev
terraform plan -out=day21.tfplan
```

Plan output:

```
Terraform will perform the following actions:

  # aws_cloudwatch_metric_alarm.high_cpu will be created
  + resource "aws_cloudwatch_metric_alarm" "high_cpu" {
      + alarm_name          = "webservers-dev-high-cpu"
      + comparison_operator = "GreaterThanThreshold"
      + evaluation_periods  = 2
      + metric_name         = "CPUUtilization"
      + threshold           = 80
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

One resource created. Zero destroyed. This plan is safe to apply.

**Key review habit — count the destructions:**

Any `destroy` in the plan output deserves extra scrutiny. Before every apply I ask:
- Do I expect this resource to be destroyed?
- What depends on this resource?
- What breaks if the destroy succeeds but the recreation fails?

Zero destructions here — good to proceed.

---

## Step 3 — Make the Code Change on a Feature Branch

```bash
git checkout -b add-cloudwatch-alarms-day21
# Added the CloudWatch alarm resource and output to main.tf
terraform plan -out=day21.tfplan  # reviewed again after changes
git add .
git commit -m "Add CPU alarm for webserver cluster"
git push origin add-cloudwatch-alarms-day21
```

The second `terraform plan` after making changes is important. It confirms the final version of the code produces exactly the plan you intend to apply — not something that drifted during editing.

---

## Step 4 — Submit for Review

Opened a pull request with the full plan output and a completed blast radius assessment:

```markdown
## What this changes
Adds a CloudWatch CPU utilization alarm to the webserver cluster.
Alarm fires when CPU exceeds 80% for two consecutive 2-minute periods.

## Terraform plan output
Plan: 1 to add, 0 to change, 0 to destroy.

+ aws_cloudwatch_metric_alarm.high_cpu
    alarm_name          = "webservers-dev-high-cpu"
    threshold           = 80
    evaluation_periods  = 2

## Resources affected
- Created: 1 (aws_cloudwatch_metric_alarm)
- Modified: 0
- Destroyed: 0

## Blast radius
Low. This change only creates a new CloudWatch alarm. It does not
modify the ASG, ALB, or any networking resources. If the apply
fails partway through, no existing resources are affected. The
worst case is a missing alarm.

## Rollback plan
Run terraform destroy -target=aws_cloudwatch_metric_alarm.high_cpu
to remove the alarm if it causes unexpected behaviour. No other
resources are affected by rolling back this change.
```

The blast radius and rollback plan sections are what separate a good infrastructure PR from a great one. They force the author to think through failure modes before applying — and they give the reviewer the context they need to approve confidently.

---

## Step 5 — Automated Tests

GitHub Actions triggered automatically on PR open:

```yaml
name: Terraform CI

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: "~> 1.5"

      - name: Terraform Init
        run: terraform init

      - name: Format Check
        run: terraform fmt -check -recursive

      - name: Validate
        run: terraform validate

      - name: Plan
        run: terraform plan -no-color
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

All checks passed. PR was eligible for merge.

**Key difference from app code:** Application unit tests confirm code logic is correct. `terraform validate` confirms the configuration syntax and schema is valid — but it does not confirm the configuration will apply successfully. The only way to know that is to run `terraform apply` — which deploys real resources. Deep integration tests (using Terratest) are expensive and belong in nightly pipelines, not per-PR checks.

---

## Step 6 — Merge and Release

After review approval and green CI, merged the PR to main. Tagged the merge commit:

```bash
git checkout main
git pull origin main
git tag -a "v1.4.0" -m "Add CPU alarm for webserver cluster"
git push origin v1.4.0
```

```bash
git tag -l
```

```
v1.0.0
v1.1.0
v1.2.0
v1.3.0
v1.4.0
```

---

## Step 7 — Deploy

Applied the saved plan file from Step 2:

```bash
terraform apply day21.tfplan
```

```
aws_cloudwatch_metric_alarm.high_cpu: Creating...
aws_cloudwatch_metric_alarm.high_cpu: Creation complete after 1s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:
cloudwatch_alarm_arn = "arn:aws:cloudwatch:us-east-1:123456789012:alarm:webservers-dev-high-cpu"
```

Confirmed the alarm exists in CloudWatch. Then ran plan immediately after:

```bash
terraform plan
```

```
No changes. Your infrastructure matches the configuration.
```

A clean plan after apply confirms state is accurate and no drift was introduced. This post-apply plan check is a habit worth building — it takes 10 seconds and gives immediate confidence that the deployment was clean.

---

## The Four Infrastructure-Specific Safeguards

These have no equivalent in application code deployment. Every team running Terraform in production should implement all four.

---

### Safeguard 1 — Plan File Pinning

Always apply from a saved plan file. Never run `terraform apply` without one for production changes.

```bash
# Correct — apply exactly what was reviewed
terraform plan -out=reviewed.tfplan
terraform apply reviewed.tfplan

# Risky — new plan generated at apply time may differ from reviewed plan
terraform apply
```

The gap between `terraform plan` and `terraform apply` can be minutes or hours. During that time, another engineer might have applied a change, AWS might have modified a resource, or state might have drifted. A fresh plan at apply time could show different actions than what was reviewed. Applying from a saved plan file eliminates this risk — you apply exactly what was approved, nothing more.

---

### Safeguard 2 — Approval Gates for Destructive Changes

If `terraform plan` shows any resource destructions, require a second explicit approval before applying — separate from the PR review.

In Terraform Cloud, configure mandatory apply approval:
- Navigate to Workspace Settings → General
- Enable "Require approval for applies"

This creates a two-step process for any apply that would destroy resources:
1. PR review approves the code change
2. Apply approval confirms the destructive action is intentional

One approval gate catches coding mistakes. The second catches the cases where the code is correct but the timing is wrong — applying a database deletion during a traffic spike, for example.

---

### Safeguard 3 — State Backup Verification

Before any significant apply, confirm S3 bucket versioning is enabled and you know how to restore a previous state version:

```bash
# List available state versions
aws s3api list-object-versions \
  --bucket your-terraform-state-bucket \
  --prefix production/terraform.tfstate \
  --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified}'
```

Output:
```json
[
  {
    "VersionId": "abc123",
    "LastModified": "2026-03-20T10:15:30.000Z"
  },
  {
    "VersionId": "def456",
    "LastModified": "2026-03-19T14:22:10.000Z"
  }
]
```

To restore a previous state version if an apply corrupts state:

```bash
aws s3api copy-object \
  --bucket your-terraform-state-bucket \
  --copy-source your-terraform-state-bucket/production/terraform.tfstate?versionId=def456 \
  --key production/terraform.tfstate
```

Knowing how to do this before you need it is the difference between a 5-minute recovery and a 2-hour incident.

---

### Safeguard 4 — Blast Radius Documentation

Every PR that touches shared infrastructure must document what depends on it and what breaks if the apply fails midway:

```markdown
## Blast radius
This change modifies the ALB security group. The following resources
depend on this security group:
- All EC2 instances in the webserver ASG
- The ALB listener health checks

If the apply fails after modifying the security group but before
the instances are updated, traffic may be blocked. Monitor the
ALB target group health during apply.
```

Blast radius thinking is a discipline. It forces the author to understand the dependency graph before applying — and surfaces risks the reviewer would not otherwise see.

---

## Sentinel Policies

Sentinel is Terraform Cloud's policy-as-code framework. It runs after `terraform plan` but before `terraform apply` is permitted. Violations block the apply.

### Policy — Restrict Instance Types

This policy prevents engineers from accidentally deploying expensive instance types in non-production environments:

```python
# sentinel/require-approved-instance-types.sentinel
import "tfplan/v2" as tfplan

allowed_instance_types = [
  "t3.micro",
  "t3.small",
  "t3.medium",
  "t2.micro",
  "t2.small"
]

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type is not "aws_instance" and
    rc.type is not "aws_launch_template" or
    rc.change.after.instance_type in allowed_instance_types
  }
}
```

**What it enforces:** Any plan that includes an `aws_instance` or `aws_launch_template` with an instance type not in the allowed list is blocked. Trying to deploy a `t3.2xlarge` in dev returns:

```
Sentinel Result: FAIL

Policy Name: require-approved-instance-types
Description: Instance type not in allowed list

  Rule "main" FAIL
```

**How it differs from `terraform validate`:**

`terraform validate` checks that your configuration is syntactically correct and uses valid resource arguments. It does not know or care what values those arguments have.

Sentinel enforces business rules on top of valid configurations. A `t3.2xlarge` is perfectly valid HCL — `terraform validate` accepts it. Sentinel rejects it because it violates your organisation's cost policy. The two tools operate at different levels.

---

## Infrastructure vs Application Workflow — Key Differences

**Difference 1 — State files have no application code equivalent**

Application code does not have a separate file tracking every running process. Terraform state tracks every resource ever created. Corrupting or losing the state file is an infrastructure incident that can take hours to recover from. Remote state with versioning and locking has no equivalent concern in application deployments.

**Difference 2 — Failures mid-apply can leave infrastructure in a broken partial state**

A failed application deployment usually means the new version is not running. A failed Terraform apply partway through might mean half the resources exist and half do not — the security group is updated but the instances are still using the old rules, or the database is deleted but the application is still pointing at it. Blast radius documentation exists because partial failure is a distinct failure mode that application deployments do not have.

**Difference 3 — Destructive changes are permanent and sometimes irreversible**

A bad application deploy that crashes the app can be rolled back in seconds by redeploying the previous version. A Terraform apply that deletes a database and all its data cannot be rolled back — the data is gone. This is why destructive changes require additional approval gates that application deployments do not.

---

## Problems I Ran Into

### ❌ Problem 1: Saved Plan File Became Invalid

I saved a plan file with `terraform plan -out=day21.tfplan`, got distracted, and applied it three hours later:

```
Error: Saved plan is stale

The given plan file can no longer be applied because the state was
changed by another operation after the plan was created.
```

**What happened:** Another `terraform apply` had run in the meantime (an automated CI apply from a different PR merge). The state changed and the saved plan was no longer valid.

**Fix:** Re-ran `terraform plan -out=day21.tfplan` to generate a fresh plan, reviewed it again, and applied immediately. Lesson: apply saved plans promptly — do not let them sit overnight.

---

### ❌ Problem 2: Sentinel Policy Syntax Error

My first Sentinel policy failed to parse:

```
Error: Failed to parse policy

  1 errors occurred:
  * sentinel/require-approved-instance-types.sentinel:8:3:
    expected expression but found "and"
```

**What happened:** Sentinel uses its own policy language — not Python, not HCL. Boolean operators in Sentinel are `and`, `or`, `not` but the operator precedence and placement rules differ from what I expected.

**Fix:** Rewrote the condition more explicitly:

```python
# Clearer boolean logic
rc.type is not "aws_instance" or
rc.change.after.instance_type in allowed_instance_types
```

Tested the policy using the Sentinel CLI (`sentinel apply`) before uploading to Terraform Cloud.

---

## What I Learned Today

- **Always apply from a saved plan file** — a plan file is a contract between what was reviewed and what gets applied
- **Blast radius documentation** forces thinking through failure modes before they happen
- **Post-apply plan check** — always run `terraform plan` after apply to confirm clean state
- **Sentinel enforces business rules** that `terraform validate` cannot — valid HCL does not mean correct policy
- **Destructive changes need two approval gates** — one for the PR review, one for the explicit apply approval
- **State versioning is your recovery mechanism** — know how to restore before you need to
- **Saved plan files expire** when state changes — apply them promptly or regenerate

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*