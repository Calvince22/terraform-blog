# How to Convince Your Team to Adopt Infrastructure as Code

*Day 19 of the #30DayTerraformChallenge*

---

Today was different from every other day in this challenge.

No new Terraform syntax. No new AWS resources. No errors to debug.

Today was about people, process, and strategy — the hardest part of Infrastructure as Code that nobody talks about in tutorials.

Writing Terraform code is the easy part. **Getting a team to actually adopt it** — changing habits, earning trust, migrating existing infrastructure, convincing leadership to invest time — that is where most IaC efforts fail.

Today I built a real adoption plan. Here is everything I learned.

---

## Why IaC Adoption Fails

Before building a plan, it helps to understand why adoption attempts fail. The author of *Terraform: Up and Running* identifies several common failure modes — and they ring true from what I have seen and experienced.

**Failure Mode 1 — Trying to migrate everything at once**

The team decides to move all existing infrastructure to Terraform. It is a huge project. It takes months. Nothing ships during that time. People lose confidence. The project gets cancelled halfway through, leaving infrastructure in a worse state than before — half in Terraform, half not.

**Failure Mode 2 — Underestimating the learning curve**

Terraform looks simple at first. Variables, resources, outputs — how hard can it be? Then you hit remote state, module dependencies, provider versioning, and state drift. The learning curve is real. Teams that do not give engineers dedicated time to learn end up with fragile, poorly structured configurations written under deadline pressure.

**Failure Mode 3 — No buy-in before starting**

An engineer discovers Terraform, gets excited, and starts converting infrastructure without telling anyone. When something breaks, there is no support. When they leave the team, nobody else understands the configurations. Buy-in — from teammates and from management — is not optional.

**Failure Mode 4 — Treating IaC as purely a technical change**

IaC is a workflow change, not just a tooling change. It changes how engineers propose infrastructure changes, how those changes are reviewed, how they get deployed. If the workflow does not change alongside the tooling, you end up with Terraform code that nobody trusts enough to use — and engineers keep making changes manually in the console.

---

## The Right Starting Point — An Honest Assessment

Before building any adoption plan, you need an honest picture of where your team is right now.

Here are the questions I used to assess my own situation:

**How is infrastructure currently provisioned?**
A mix of manual console clicks and some bash scripts. No consistent pattern. Different engineers provision things differently. There is drift between what is documented and what actually exists in AWS.

**How many people are involved in infrastructure changes?**
Two to three engineers handle most infrastructure work. No formal review process — changes go directly to production. The person who makes the change is often the only one who knows it happened.

**How often do infrastructure changes cause incidents?**
More often than it should. Manual changes are the most common cause — someone changes a security group rule or modifies an instance type and forgets to update the documentation. The next person to touch it does not know what changed or when.

**Is there drift between documented and actual infrastructure?**
Significant drift. The documentation is months out of date in places. Nobody has time to keep it current. Terraform would replace the documentation with code — code that is always up to date because it is the infrastructure.

**Are secrets managed properly?**
Not consistently. Some credentials are hardcoded in scripts. Some are shared via Slack. A move to Terraform gives a natural forcing function to fix this — Secrets Manager integration, `sensitive = true` variables, proper `.gitignore` setup.

**What is the team's familiarity with version control for infrastructure?**
Application code goes through Git and pull requests. Infrastructure changes do not. This is the biggest culture gap to close.

---

## The Four-Phase Adoption Plan

The key insight from the book is this: **do not try to migrate existing infrastructure first.** Start with something new. Build a success story. Build confidence. Then migrate incrementally.

Here is the plan I would present to my team.

---

### Phase 1 — Start with Something New (Weeks 1–2)

**What:** Pick one new piece of infrastructure that needs to be built anyway — not a migration, not a rewrite. A new S3 bucket for log storage, a new IAM role for a service, a new security group. Build it entirely with Terraform.

**Why this works:** There is zero migration risk. If something goes wrong, you have not touched anything that was already working. The team gets a real Terraform workflow — `init`, `plan`, `apply`, state in S3 — without the pressure of production impact.

**What gets done:**

```hcl
# Example Phase 1 resource — a new S3 bucket for application logs
resource "aws_s3_bucket" "app_logs" {
  bucket = "my-team-app-logs-2026"

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    Team        = "platform"
  }
}

resource "aws_s3_bucket_versioning" "app_logs" {
  bucket = aws_s3_bucket.app_logs.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "app_logs" {
  bucket = aws_s3_bucket.app_logs.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

**Success criteria:**
- Terraform configuration is in a Git repository
- Remote state is configured in S3
- At least two engineers have reviewed the code via pull request
- At least two engineers can run `terraform plan` and explain what they see
- The resource exists in AWS and matches the Terraform configuration

**Who:** One experienced engineer leads, one other engineer reviews.

**Timeline:** Two weeks maximum.

---

### Phase 2 — Import Existing Infrastructure (Weeks 3–6)

**What:** Begin bringing critical existing resources under Terraform management using `terraform import`. Prioritise resources that change frequently or have caused incidents.

**Why now:** The team has run Terraform. They understand the workflow. They trust it enough to import something real.

**The import process:**

Step 1 — Write the resource block in Terraform to match what already exists:

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Security group for web servers"
  vpc_id      = "vpc-0c7fbd6b8ddf09055"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Step 2 — Import the existing resource:

```bash
terraform import aws_security_group.web_sg sg-0abc123def456789
```

Output:
```
aws_security_group.web_sg: Importing from ID "sg-0abc123def456789"...
aws_security_group.web_sg: Import prepared!
aws_security_group.web_sg: Refreshing state... [id=sg-0abc123def456789]

Import successful!
```

Step 3 — Run `terraform plan` and verify no changes are needed:

```bash
terraform plan
```

```
No changes. Your infrastructure matches the configuration.
```

A clean plan after import means your Terraform code accurately describes the existing resource. If the plan shows changes, update the code until the plan is clean before applying anything.

**Success criteria:**
- At least 5 existing resources imported into Terraform
- All imported resources show clean `terraform plan` output
- No manual changes made to these resources in the AWS Console after import

**Timeline:** Three to four weeks. Do not rush this — import errors cause real disruption.

---

### Phase 3 — Establish Team Practices (Weeks 7–10)

**What:** Now that multiple engineers are writing Terraform, establish the practices that prevent chaos at scale.

**The rules:**

```
✅ All infrastructure changes go through pull requests
✅ terraform plan output is pasted into every PR description
✅ terraform validate and terraform fmt run in CI before merge
✅ Module versioning — shared modules are tagged and pinned
✅ State locking enforced — use_lockfile = true
✅ No manual console changes to Terraform-managed resources — ever
✅ Secrets via AWS Secrets Manager — never hardcoded
✅ .gitignore includes *.tfstate, .terraform/, *.tfvars
```

The most important rule is the last one — no manual console changes. Every time someone makes a manual change to Terraform-managed infrastructure, it creates state drift. The next `terraform plan` shows unexpected changes. Trust in the plan output erodes. Engineers stop reading it carefully. This is how production incidents happen.

**CI pipeline configuration:**

```yaml
name: Terraform Validate

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

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -no-color
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

**Success criteria:**
- CI pipeline running on every pull request
- No infrastructure changes merged without a passing plan
- Module registry established with at least one versioned module
- All team members have run `terraform import` at least once

**Timeline:** Four weeks.

---

### Phase 4 — Automate Deployments (Week 11+)

**What:** Connect Terraform to the CI/CD pipeline so that merges to `main` automatically trigger `terraform apply`. Infrastructure changes go through the same review and deployment process as application code.

**Why last:** Automated apply is powerful — and risky if the team is not ready. You need high confidence in your Terraform code, your plan review process, and your state management before automating apply.

```yaml
name: Terraform Apply

on:
  push:
    branches: [main]

jobs:
  apply:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        run: terraform init

      - name: Terraform Apply
        run: terraform apply -auto-approve
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

**Success criteria:**
- Every merge to `main` triggers an automated apply
- Engineers no longer run `terraform apply` locally for production
- Deployment history is visible in the CI/CD tool
- Rollback process is documented and tested

**Timeline:** Ongoing.

---

## The Business Case

When presenting this to leadership, frame everything around outcomes they care about — not technology:

| Business Problem | IaC Solution | Measurable Outcome |
|---|---|---|
| Infrastructure incidents from manual errors | Code review catches mistakes before apply | Target: 50% reduction in infra-related incidents |
| Hours spent on repetitive environment setup | Reusable modules provision in minutes | New environment setup from 2 days → 30 minutes |
| No audit trail for infrastructure changes | Every change is a Git commit with author and timestamp | Full audit trail — meets compliance requirements |
| Dev environment differs from production | Identical Terraform configs for all environments | Fewer "works on my machine" incidents |
| Slow engineer onboarding to infrastructure | Documented, version-controlled configurations | New engineer productive on infra in days not weeks |
| Credentials shared or hardcoded | Secrets Manager integration enforced | Zero hardcoded credentials in codebase |

Estimate real numbers for your organisation where you can. Even rough estimates are more persuasive than no numbers at all.

---

## Terraform Cloud — What It Adds Beyond S3

The Terraform Cloud lab today demonstrated several things that a plain S3 backend does not provide:

**Remote execution** — `terraform apply` runs in Terraform Cloud, not on an engineer's laptop. No "it worked on my machine" deployment issues.

**Policy as code (Sentinel)** — enforce rules like "no EC2 instances larger than t3.large in dev" or "all S3 buckets must have encryption." Violations block the apply before any resources are created.

**Cost estimation** — shows estimated cost impact for every plan. Engineers see "this change will add $45/month" before applying.

**Team management** — role-based access control, audit logs, workspace management across multiple teams and environments.

**VCS integration** — every pull request automatically runs `terraform plan` and posts the result as a comment.

For small teams getting started, S3 is sufficient. For larger organisations managing multiple teams and environments, Terraform Cloud removes significant operational overhead.

---

## What I Learned Today

- **The technical part of Terraform is not the hard part** — adoption, workflow change, and team culture are harder
- **Start with something new** — the worst way to begin is a big-bang migration of existing infrastructure
- **`terraform import` is how you adopt existing infrastructure** — write the resource block first, import second, verify clean plan third
- **No manual console changes is the most important rule** — every exception erodes trust in the plan output
- **The business case must speak leadership's language** — outcomes and numbers, not technology
- **Terraform Cloud adds policy enforcement and remote execution** — valuable for teams that have outgrown S3
- **Phase your adoption** — four phases over three months is realistic. Six months is fine. One week is not.

---

## The Hardest Part

The hardest part of IaC adoption is not technical. It is convincing people to stop doing something they already know how to do — clicking through the AWS Console — and replace it with something new that requires learning and discipline.

The console feels faster in the short term. It is not. Every manual change is undocumented, unreviewed, and one mistake away from an incident.

The path forward is building enough early wins — with zero disruption — that engineers start to trust the new workflow. Once they trust it, they defend it. Once the team defends it, adoption becomes self-sustaining.

That is the real goal of Phase 1.

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*