# Putting It All Together: Application and Infrastructure Workflows with Terraform

*Day 22 of the #30DayTerraformChallenge*

---


More importantly, today I combined everything from the last three weeks into one integrated pipeline — version control, CI, automated tests, Sentinel policies, cost gates, and immutable plan promotion across environments — running as one coherent system.

This is the post where everything comes together.

---

## The Key Insight — Immutable Artifact Promotion

The most important architectural insight from Chapter 10 is this:

> **The same artifact that was tested in staging is the exact artifact that gets deployed to production. It is never regenerated.**

In application code, the artifact is a Docker image or a compiled binary. You build it once in CI, tag it with a version, and promote that exact image through dev → staging → production.

In Terraform, the artifact is a saved plan file. You generate it once in CI after review, and you promote that exact plan file through environments. The production apply uses the same plan that was reviewed — not a fresh plan that might differ.

This is the pattern that makes infrastructure deployments trustworthy at scale.

```
PR opened
    ↓
CI runs validate + tests
    ↓
CI generates plan file (the artifact)
    ↓
Plan reviewed and approved
    ↓
Plan promoted to staging → applied
    ↓
Same plan promoted to production → applied
    ↓
No plan is ever regenerated after review
```

---

## The Integrated CI Pipeline

Here is the complete GitHub Actions workflow that runs on every pull request:

```yaml
name: Infrastructure CI

on:
  pull_request:
    branches: [main]

jobs:
  # Job 1 — Validate first, before touching AWS
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~> 1.5"

      - name: Format check
        run: terraform fmt -check -recursive

      - name: Init (no backend — fast, no AWS needed)
        run: terraform init -backend=false

      - name: Validate
        run: terraform validate

      - name: Unit tests
        run: terraform test

  # Job 2 — Plan only runs after validate passes
  plan:
    runs-on: ubuntu-latest
    needs: validate
    env:
      AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Init
        run: terraform init

      - name: Plan
        run: terraform plan -out=ci.tfplan

      # Save the plan as an artifact so it can be applied later
      # without being regenerated
      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: terraform-plan
          path: ci.tfplan
```

### Why Two Separate Jobs

The `validate` job runs without AWS credentials — it only needs Terraform itself. This makes it fast (under 30 seconds) and cheap. It catches format errors, syntax errors, and test failures without making any API calls.

The `plan` job only runs after `validate` passes (`needs: validate`). It uses real AWS credentials to generate an accurate plan against actual infrastructure state. If `validate` fails, `plan` never runs — no wasted API calls or AWS costs.

### The Plan Artifact

The plan file uploaded by `actions/upload-artifact` is the immutable artifact. It is stored by GitHub Actions and can be downloaded and applied in a subsequent workflow without regenerating. This is what makes the promotion pattern possible — the reviewed plan is preserved.

---

## Sentinel Policies

### Policy 1 — Approved Instance Types

Prevents engineers from deploying expensive instance types that have not been approved:

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

**What it blocks:** Any plan that creates or modifies an EC2 instance or launch template with an instance type not in the approved list. A `t3.2xlarge` accidentally deployed in dev would cost ~$250/month — this policy makes that impossible.

**Why it matters:** Engineers make mistakes under deadline pressure. Policy enforcement removes the human error factor from cost control entirely. The policy runs automatically — no human needs to remember to check.

---

### Policy 2 — Required ManagedBy Tag

Ensures every resource deployed through Terraform is tagged for auditability:

```python
# sentinel/require-terraform-tag.sentinel
import "tfplan/v2" as tfplan

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.change.after.tags["ManagedBy"] is "terraform"
  }
}
```

**What it blocks:** Any plan that creates a resource without a `ManagedBy = "terraform"` tag. Without this, resources accumulate in AWS with no indication of how they were created — making audits, cost attribution, and cleanup significantly harder.

**Why it matters:** When you have hundreds of resources across multiple environments, the `ManagedBy` tag is how you know which ones are safe to touch manually and which ones Terraform will overwrite on the next apply.

---

### Policy 3 — Cost Estimation Gate

Blocks applies that would increase monthly costs by more than $50:

```python
# sentinel/cost-check.sentinel
import "tfrun"

maximum_monthly_increase = 50.0

main = rule {
  tfrun.cost_estimate.delta_monthly_cost < maximum_monthly_increase
}
```

**What it blocks:** Any apply that adds more than $50/month to the infrastructure bill. This catches accidental over-provisioning — creating five load balancers instead of one, or spinning up ten EC2 instances when two were intended.

**Terraform Cloud cost estimation output:**
```
Cost Estimation
  Resources: 9
  Monthly cost estimate: $42.18
  Previous monthly estimate: $39.45
  Delta: +$2.73/month ✅ (under $50 threshold)
```

For the CloudWatch alarm deployment from Day 21:
```
Delta: +$0.10/month ✅
```

For a hypothetical large EC2 instance:
```
Delta: +$156.20/month ❌ — blocked by cost policy
```

---

## The Complete Side-by-Side Comparison

| Component | Application Code | Infrastructure Code |
|---|---|---|
| Source of truth | Git repository | Git repository |
| Local run | `npm start` / `python app.py` | `terraform plan` |
| Artifact | Docker image / binary | Saved `.tfplan` file |
| Versioning | Semantic version tag | Semantic version tag |
| Automated tests | Unit + integration tests | `terraform test` + Terratest |
| Policy enforcement | Linting / SAST | Sentinel policies |
| Cost gate | N/A | Cost estimation policy |
| Promotion | Image promoted across envs | Plan promoted across envs |
| Deployment | CI/CD pipeline | `terraform apply <plan>` |
| Rollback | Redeploy previous image | `terraform apply <previous plan>` |
| State tracking | N/A | `terraform.tfstate` in S3 |
| Partial failure | App not running | Infrastructure in broken partial state |

The two workflows are now converging. The main structural difference is the state file — application code does not have an equivalent concern, and that difference drives several of the infrastructure-specific safeguards we added in Day 21.

---

## Journey Reflection — 22 Days Honest

### What I Built

Looking back at 22 days, the list of what was actually deployed is longer than I expected:

- Single EC2 web server with security groups and user data
- Auto Scaling Group with Launch Template and Application Load Balancer
- Multi-environment deployments (dev, staging, production) using file layouts and workspaces
- S3 remote backend with versioning, encryption, and state locking
- Reusable webserver cluster module with versioning on GitHub
- Multi-region S3 bucket replication using provider aliases
- EKS cluster with managed node groups and nginx on Kubernetes
- Docker containers managed by Terraform locally
- CloudWatch alarms, IAM roles, Target Groups, Listeners
- Full CI/CD pipeline with Terraform Cloud and Sentinel policies

Most engineers take 6–12 months to build this breadth of experience through normal project work. The challenge compressed it into three weeks.

---

### What Changed in How I Think

Before this challenge I thought about infrastructure as something you set up once and then manage. Click through the console, get the server running, document it somewhere.

After this challenge I think about infrastructure as code that has a lifecycle — it gets written, reviewed, tested, versioned, and deployed exactly like application code. The mental model shift is from "infrastructure as a one-time task" to "infrastructure as a continuously evolving, version-controlled artefact."

The specific moment this clicked was Day 5 — when I manually changed a tag in the AWS Console and watched `terraform plan` detect it and plan to revert it. That is when I understood that Terraform is not just a provisioning tool — it is an ownership model. If Terraform manages a resource, Terraform owns it. Nobody else touches it.

---

### What Was Harder Than Expected

State management was significantly harder than the documentation made it sound.

Not the mechanics — the S3 backend, locking, versioning. Those are straightforward once you follow the steps. What was harder was the conceptual model: understanding that the state file is not just a record of what exists, but the source of truth that Terraform uses to calculate every plan.

The two experiments on Day 5 made it concrete — editing the state file did not trigger a plan change because Terraform checks real AWS, not just the state file. But changing something in the AWS Console did trigger a change because the state file no longer matched AWS reality.

Understanding the three-way relationship between code, state, and real infrastructure is not obvious from documentation alone. You have to break it to understand it.

---

### What I Would Do Differently

If I started again from Day 1, I would set up the S3 remote backend and `.gitignore` before writing a single resource block.

On Day 1 I created a local state file, accumulated several days of deployments, and then had to migrate state on Day 6. That migration worked but it added unnecessary complexity. Setting up the backend on Day 1 means every deployment from the start goes through the correct workflow — nothing to migrate later.

I would also write a `variables.tf` with proper types and descriptions from the very first configuration. Starting with hardcoded values and refactoring to variables later is a habit that scales badly. Starting with proper variables and defaults from the beginning costs nothing and builds the right muscle memory.

---

### What Comes Next

The Terraform Associate certification exam is the immediate next step — the challenge was designed to cover the exam syllabus and the knowledge from 22 days is a strong foundation.

After the exam, the first real project is applying this to my team's infrastructure. The four-phase adoption plan from Day 19 is the roadmap — Phase 1 starting with one new resource, zero migration risk, building confidence before touching anything critical.

The longer-term goal is module development. Taking the reusable patterns built during this challenge — the webserver cluster, the remote state setup, the multi-environment file layout — and packaging them as properly documented, versioned modules that the whole team can use.

---

## What I Learned Today

- **Immutable artifact promotion** is the key pattern — generate the plan once, review it once, apply the same plan across environments
- **Two-job CI** — validate fast without credentials, plan after validation passes — is the right structure for efficiency and cost
- **Sentinel policies enforce what humans forget to check** — cost limits, required tags, approved instance types
- **Cost estimation gates** make the financial consequence of every infrastructure change visible before it is applied
- **The two workflows are converging** — application and infrastructure deployments are becoming the same process with the same discipline

---

## The Single Most Important Insight from Chapter 10

The most important insight is that infrastructure code should be treated exactly like application code — with the same version control discipline, the same review process, the same automated testing, and the same promotion workflow.

Most teams that struggle with Terraform are not struggling with the Terraform syntax. They are struggling because they apply infrastructure code discipline that is two generations behind their application code discipline. They commit to main directly. They apply without review. They store state locally. They have no automated tests.

The gap between "Terraform works on my laptop" and "Terraform is safe to run in production" is entirely filled by workflow and discipline — not by learning more HCL.

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*