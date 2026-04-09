# A Workflow for Deploying Application Code with Terraform

*Day 20 of the #30DayTerraformChallenge*

---

Today connected two worlds that engineers often treat separately — the way application code gets deployed and the way infrastructure code should get deployed.

Most development teams already trust a seven-step workflow for shipping application changes safely. Today I mapped that exact workflow to Terraform, walked through all seven steps with a real deployment, and set up Terraform Cloud as the platform that makes the infrastructure version of this workflow reliable at team scale.

Here is everything step by step.

---

## The Seven Steps — Application Code vs Infrastructure Code

Before diving in, here is the complete mapping:

| Step | Application Code | Infrastructure Code |
|---|---|---|
| 1. Version control | Git for source code | Git for `.tf` files |
| 2. Run locally | `npm start` / `python app.py` | `terraform plan` |
| 3. Make changes | Edit source files | Edit `.tf` files |
| 4. Review | Code diff in PR | Plan output in PR |
| 5. Automated tests | Unit tests, linting | `terraform validate`, Terratest |
| 6. Merge and release | Merge + tag | Merge + tag |
| 7. Deploy | CI/CD pipeline | `terraform apply` |

The analogy is close — but there are important differences at each step. Let me walk through each one.

---

## Step 1 — Version Control

Your Terraform code lives in Git. Every `.tf` file, every module, every variable definition — all of it version controlled.

**What belongs in Git:**
```
✅ main.tf
✅ variables.tf
✅ outputs.tf
✅ modules/
✅ .terraform.lock.hcl
```

**What does NOT belong in Git:**
```
❌ terraform.tfstate
❌ terraform.tfstate.backup
❌ .terraform/
❌ *.tfvars (if they contain secrets)
```

The critical difference from application code: **the state file is never in Git.** Application code repos do not have a live database of running processes tracked alongside the source. Terraform state is that database — it belongs in a remote backend (S3 or Terraform Cloud), not version control.

**Protecting the main branch:**

In GitHub, set branch protection rules on `main`:
- Require pull request reviews before merging
- Require status checks to pass (CI/CD)
- No direct pushes — everything through PRs

This is the same rule engineering teams apply to application code. Infrastructure deserves the same discipline.

---

## Step 2 — Run Locally

For application code, running locally means starting the app and testing it. For Terraform, running locally means running `terraform plan` — seeing exactly what will change before it changes anything.

The change I made: updating the HTML response in the user data script from `v2` to `v3`.

```bash
# Save the plan to a file — never apply a plan you have not reviewed
terraform plan -out=day20.tfplan
```

Plan output:

```
Terraform will perform the following actions:

  # aws_launch_template.web will be updated in-place
  ~ resource "aws_launch_template" "web" {
      ~ user_data = "IyEvYmluL2Jhc2g..." -> "IyEvYmluL2Jhc2g..."
        # (the base64 encoded user data changed)
    }

  # aws_autoscaling_group.web will be replaced
  - resource "aws_autoscaling_group" "web" {
      # old ASG — will be destroyed after new one is healthy
    }
  + resource "aws_autoscaling_group" "web" {
      # new ASG — create_before_destroy ensures zero downtime
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

The plan shows exactly what will change. The ASG replacement is expected — `create_before_destroy` means the new ASG is created first, traffic shifts, then the old one is destroyed. No surprises.

**Key difference from app code:** Running application code locally shows a working app. Running `terraform plan` shows what *will* change in real cloud infrastructure — without touching anything yet. The plan is a preview, not execution.

---

## Step 3 — Make the Code Change

Create a feature branch before making any changes:

```bash
git checkout -b update-app-version-day20
```

Update the user data in `main.tf`:

```hcl
user_data = base64encode(<<-EOF
            #!/bin/bash
            apt-get update -y
            apt-get install -y apache2
            systemctl start apache2
            systemctl enable apache2
            echo "<h1>Hello from Terraform — v3</h1>" > /var/www/html/index.html
            echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
            EOF
)
```

Commit the change:

```bash
git add .
git commit -m "Update app response to v3 for Day 20"
git push origin update-app-version-day20
```

**Key difference from app code:** Changing a source file changes behaviour in a running application — but only after deployment. Changing a `.tf` file changes the desired state of real cloud infrastructure. A typo in application code might cause a test to fail. A typo in Terraform code might delete a production database. The stakes are higher, which is why review matters more.

---

## Step 4 — Submit for Review

Open a pull request from `update-app-version-day20` to `main`.

The critical habit: **paste the `terraform plan` output as a comment on the PR.**

This is the infrastructure equivalent of a code diff. Your reviewer should not have to run Terraform themselves to understand what the merge will do to production. The plan output tells them:

- Exactly which resources will be created, modified, or destroyed
- What specific attribute values will change
- Whether any dangerous operations (destroy, replace) are planned

Without the plan output in the PR, reviewing infrastructure code is guesswork. With it, the reviewer can make an informed decision.

**PR description template:**

```markdown
## What this changes
Updates the app HTML response from v2 to v3.

## Terraform Plan Output
```
Plan: 1 to add, 1 to change, 1 to destroy.
- aws_autoscaling_group.web will be replaced (create_before_destroy)
- aws_launch_template.web user_data updated
```

## Testing
- [ ] Plan reviewed and matches intent
- [ ] CI checks passing
- [ ] No unexpected resource recreation
```

---

## Step 5 — Run Automated Tests

The GitHub Actions workflow triggers automatically on the pull request:

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

The PR is blocked from merging until all checks pass.

**Key difference from app code:** Application unit tests run in milliseconds and cost nothing. Infrastructure tests that deploy real resources take minutes and cost money. This is why the CI for Terraform runs `validate` and `plan` — catching configuration errors before apply — rather than spinning up real infrastructure on every PR.

For deeper testing (confirming the deployed infrastructure actually works), tools like Terratest exist — but these are expensive to run on every PR. They belong in a nightly or pre-release pipeline, not a per-PR check.

---

## Step 6 — Merge and Release

After review and CI pass, merge the pull request to `main`.

Tag the merge commit with a version:

```bash
git checkout main
git pull origin main
git tag -a "v1.3.0" -m "Update app response to v3"
git push origin v1.3.0
```

Confirm the tag exists:

```bash
git tag -l
```

```
v1.0.0
v1.1.0
v1.2.0
v1.3.0
```

**Key difference from app code:** Application releases release software that users download or run. Terraform releases tag the infrastructure configuration that modules and environments can pin to. If your module is consumed by other configurations, this version tag is what they reference with `?ref=v1.3.0`. Consumers stay on stable versions until they deliberately upgrade.

---

## Step 7 — Deploy

Apply the saved plan file from Step 2:

```bash
terraform apply day20.tfplan
```

Using the saved plan file guarantees you are applying exactly what you reviewed. If you run `terraform apply` without a plan file, Terraform generates a new plan at apply time — which might differ from what you reviewed if something changed in AWS between plan and apply.

```
aws_launch_template.web: Modifying...
aws_autoscaling_group.web: Creating...
aws_autoscaling_group.web: Still creating... [30s elapsed]
aws_autoscaling_group.web: Creation complete after 2m15s
aws_autoscaling_group.web (old): Destroying...
aws_autoscaling_group.web (old): Destruction complete after 10s

Apply complete! Resources: 1 added, 1 changed, 1 destroyed.
```

Verify the deployment:

```bash
curl http://webservers-dev-alb-123456789.us-east-1.elb.amazonaws.com
```

```
<h1>Hello from Terraform — v3</h1>
<p>Instance ID: i-0abc123def456789</p>
```

Version 3 is live. Zero downtime — `create_before_destroy` handled the transition cleanly.

---

## Setting Up Terraform Cloud

Moving from an S3 backend to Terraform Cloud gives the workflow a proper platform with built-in plan storage, team access controls, and an audit log.

### The Cloud Backend Configuration

```hcl
terraform {
  cloud {
    organization = "your-org-name"

    workspaces {
      name = "webserver-cluster-dev"
    }
  }
}
```

### Migrating State

```bash
# Authenticate with Terraform Cloud
terraform login

# Migrate state from S3 to Terraform Cloud
terraform init
```

Output:
```
Initializing Terraform Cloud...

Do you wish to proceed?
  As part of migrating to Terraform Cloud, Terraform can optionally
  copy your current workspace state to the configured Terraform Cloud
  workspace.

  Answer "yes" to copy the latest state snapshot to the configured
  Terraform Cloud workspace.

  Enter a value: yes

Terraform Cloud has been successfully initialized!
```

After migration, the state file is visible in the Terraform Cloud UI under your workspace — versioned, with a full history of every apply.

### Secure Variable Management

Move credentials and sensitive variables out of your local environment and into Terraform Cloud workspace variables:

**Environment Variables (marked Sensitive):**
```
AWS_ACCESS_KEY_ID      = AKIAIOSFODNN7EXAMPLE     ← sensitive
AWS_SECRET_ACCESS_KEY  = wJalrXUtnFEMI/EXAMPLE    ← sensitive
```

**Terraform Variables:**
```
cluster_name    = "webservers-dev"
instance_type   = "t3.micro"
environment     = "dev"
min_size        = 2
max_size        = 4
server_message  = "Hello from Terraform — v3"
```

Once configured, runs triggered from Terraform Cloud use these variables automatically. No credentials on any developer's machine. No secrets in CI logs. No `.tfvars` files to manage or accidentally commit.

**Why sensitive variables must never appear in `.tf` files or CI logs:**
- `.tf` files are committed to Git — secrets in Git are permanent even after deletion
- CI logs are often stored, shared, and accessible to more people than production infrastructure
- Terraform Cloud stores sensitive variables encrypted and never displays their values after initial entry

---

## The Private Registry

The Terraform Cloud private registry lets your team publish and consume internal modules the same way they use public Registry modules — with versioning, documentation, and a consistent source URL.

### Publishing a Module

Repository naming convention: `terraform-<provider>-<name>`

```
terraform-aws-webserver-cluster
```

```bash
git tag v1.0.0
git push origin v1.0.0
```

In Terraform Cloud: Registry → Publish → Module → connect the GitHub repository.

### Using the Published Module

```hcl
module "webserver_cluster" {
  source  = "app.terraform.io/your-org/webserver-cluster/aws"
  version = "1.0.0"

  cluster_name  = "prod-cluster"
  instance_type = "t3.medium"
  min_size      = 3
  max_size      = 10
  environment   = "production"
}
```

**Advantages over a GitHub URL:**
- Version browsing and documentation in the Terraform Cloud UI
- Access control — only users in your organisation can see internal modules
- The source URL format matches public Registry modules — engineers use one consistent pattern
- Automatic README rendering from the module's `README.md`
- Input/output documentation generated automatically from `variables.tf` and `outputs.tf`

---

## Problems I Ran Into

### ❌ Problem 1: Terraform Login Token Expired

When I ran `terraform login`, it opened a browser window for authentication. I closed it too quickly and the token was never generated:

```
Error: No token provided for app.terraform.io

Run "terraform login" to obtain a new token.
```

**Fix:** Ran `terraform login` again, waited for the browser window to fully load, clicked "Create API token", copied the token, and pasted it into the terminal prompt.

---

### ❌ Problem 2: State Migration Failed — S3 Bucket Permission Error

When running `terraform init` to migrate state to Terraform Cloud:

```
Error: Error acquiring the state lock

Error message: 2 errors occurred:
  * failed to retrieve lock info: AccessDenied: Access Denied
```

**What happened:** The S3 backend had strict IAM policies restricting access. Terraform needed to read the existing state to migrate it but the IAM user running `terraform init` did not have `s3:GetObject` permission on the state file path.

**Fix:** Temporarily added `s3:GetObject` and `s3:ListBucket` permissions to the IAM user, ran `terraform init` to complete the migration, then removed the temporary permissions. State was successfully migrated to Terraform Cloud.

---

### ❌ Problem 3: Workspace Variable Not Picked Up

I added `cluster_name` as a Terraform variable in the Terraform Cloud workspace but it was still prompting me for the value during a remote run.

**What happened:** I had set it as an Environment variable instead of a Terraform variable. Environment variables are available to the shell — they work for `AWS_ACCESS_KEY_ID` but not for Terraform input variables. Terraform variables must be set in the Terraform Variables section, not Environment Variables.

**Fix:** Deleted the variable from the Environment Variables section and re-added it in the Terraform Variables section. The next run picked it up correctly with no prompt.

---

## What I Learned Today

- **The seven-step workflow exists for application code for good reasons** — every step catches a different category of mistake. Skipping any step in the infrastructure workflow creates the same categories of risk
- **`terraform plan -out=planfile`** saves a plan that can be applied exactly as reviewed — use it for any production change
- **Plan output in PRs** is non-negotiable — reviewers cannot evaluate infrastructure changes without seeing what will happen
- **Terraform Cloud workspace variables** remove credentials from local machines and CI logs — the right place for secrets in an automated workflow
- **The private registry** gives internal modules the same discoverability and versioning as public ones
- **State migration to Terraform Cloud** requires read access to the existing S3 state — plan for this before attempting the migration

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*