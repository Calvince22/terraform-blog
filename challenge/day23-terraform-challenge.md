# Preparing for the Terraform Associate Exam — Key Resources and Tips

*Day 23 of the #30DayTerraformChallenge*

---

Day 23 was about being honest — auditing exactly where my knowledge is strong, where it is shaky, and where there are genuine gaps. The Terraform Associate exam is more detailed than most people expect, especially on CLI commands and state management edge cases.

Here is my full audit, study plan, and the specific areas most people underestimate.

---

## The Exam Domains

The Terraform Associate exam covers these domains with approximate weightings:

| Domain | Weight | My Rating |
|---|---|---|
| Understand Infrastructure as Code concepts | 16% | 🟢 Green |
| Understand Terraform's purpose | 20% | 🟢 Green |
| Understand Terraform basics | 24% | 🟡 Yellow |
| Use the Terraform CLI | 26% | 🟡 Yellow |
| Interact with Terraform modules | 12% | 🟢 Green |
| Navigate the core Terraform workflow | 8% | 🟢 Green |
| Implement and maintain state | 8% | 🟡 Yellow |
| Read, generate, and modify configuration | 8% | 🟢 Green |
| Understand Terraform Cloud capabilities | 4% | 🟡 Yellow |

The CLI domain at 26% is the highest weighted single domain. It is also the one most people underestimate. Knowing what `terraform state rm` does to real infrastructure (nothing — it only removes from state) versus what `terraform destroy` does is exactly the kind of distinction the exam tests.

---

## The CLI Commands You Must Know Cold

This section is where most people lose points. Not because the commands are hard — because the exam tests edge cases and exact behaviours, not just general knowledge.

Here is every command with what it does and when you would use it:

### `terraform init`
Downloads provider plugins and modules, configures the backend. Run this first in any project, after adding a new provider, or after changing the backend configuration. Key flag: `-backend=false` initialises without configuring a backend — useful in CI for faster validation.

### `terraform validate`
Checks configuration syntax and internal consistency without making any API calls. Does not check that values are valid for the actual provider — only that the HCL structure is correct. Run before `terraform plan` in CI to catch format and schema errors cheaply.

### `terraform fmt`
Formats all `.tf` files to the canonical style — consistent indentation, spacing, and alignment. Run with `-check` flag in CI to fail the build if files are not formatted. Run without flags locally to auto-format.

### `terraform plan`
Compares your configuration and state against real infrastructure and shows what would change. Does not make any changes. Key flag: `-out=planfile` saves the plan for later apply. Always review the plan before every apply.

### `terraform apply`
Creates, updates, or destroys resources to match your configuration. Without a plan file it generates a new plan at apply time. With a plan file (`terraform apply planfile`) it applies exactly what was reviewed — the safer production pattern.

### `terraform destroy`
Destroys all resources managed by the current configuration. Equivalent to running `terraform apply` with every resource marked for deletion. Always run this after each challenge session to avoid unnecessary charges.

### `terraform output`
Reads and displays output values from the state file. Run after apply to see values like ALB DNS names or instance IPs. Use `-raw` flag for a clean value without formatting, useful in scripts.

### `terraform state list`
Lists all resources currently tracked in the state file. Useful for confirming what Terraform manages and for troubleshooting missing or unexpected resources.

### `terraform state show`
Displays all attributes of a specific resource from the state file. Use it to inspect what Terraform knows about a resource — useful when debugging plan output that shows unexpected changes.

### `terraform state mv`
Moves a resource within the state file — either renaming it or moving it between state files. Does not touch real infrastructure. Use it when you rename a resource in your code and want Terraform to track it under the new name instead of destroying and recreating it.

### `terraform state rm`
Removes a resource from the state file without touching the real resource in AWS. After removal, Terraform no longer manages that resource — it will not appear in plans or applies. Use when you want to stop managing a resource with Terraform but keep it running.

### `terraform import`
Adopts an existing real resource into Terraform state management. Write the resource block first, then run import. Run `terraform plan` after import and verify it shows no changes before applying anything.

### `terraform workspace`
Manages workspaces — `new`, `list`, `select`, `show`, `delete`. Each workspace has its own state file. Always run `terraform workspace show` before any apply to confirm which environment you are targeting.

### `terraform providers`
Shows which providers are required by the current configuration and where they are sourced from. Useful for auditing provider dependencies and confirming version constraints are applied.

### `terraform login`
Authenticates to Terraform Cloud by generating an API token and storing it locally. Required before using a `cloud` backend block. Run once per machine.

### `terraform graph`
Outputs the dependency graph of resources in DOT format. Pipe to Graphviz to visualise. Useful for understanding complex dependency chains between resources.

---

## The Critical State Command Distinction

This specific distinction appears on the exam regularly:

| Command | Effect on real infrastructure | Effect on state |
|---|---|---|
| `terraform destroy` | Deletes the resource | Removes from state |
| `terraform state rm` | **No effect** | Removes from state |
| `terraform import` | No effect | Adds to state |
| `terraform taint` (deprecated) | No effect | Marks for recreation on next apply |

**Exam scenario:** "A team member manually deleted an S3 bucket that Terraform manages. What happens when you run `terraform plan`?"

**Answer:** Terraform detects the drift — the bucket exists in state but not in AWS — and shows it as a resource to be created. Terraform does not error. It simply plans to recreate the missing resource.

---

## Non-Cloud Providers

Terraform is not just for cloud infrastructure. The `random` and `local` providers appear in exam questions and are genuinely useful in real configurations.

### The `random` Provider

Generates random values — useful for creating unique resource names without hardcoding:

```hcl
terraform {
  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

# Generate a random suffix for globally unique bucket names
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "example" {
  bucket = "my-app-${random_id.bucket_suffix.hex}"
}

# Generate a random password for a database
resource "random_password" "db_password" {
  length           = 16
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

output "bucket_name" {
  value = aws_s3_bucket.example.id
}

output "db_password" {
  value     = random_password.db_password.result
  sensitive = true
}
```

The `random_id` resource is the standard solution to the S3 bucket naming problem — bucket names must be globally unique and using `random_id` guarantees uniqueness without manual coordination.

### The `local` Provider

Writes files to the local filesystem — useful for generating configuration files during a deployment:

```hcl
resource "local_file" "kubeconfig" {
  content  = module.eks.kubeconfig
  filename = "${path.module}/kubeconfig.yaml"
}
```

### Key exam point about `random` resources

Random values are generated once and stored in state. They do not change on subsequent applies unless you use the `keepers` argument to trigger regeneration:

```hcl
resource "random_id" "server" {
  keepers = {
    ami_id = var.ami_id  # regenerate when AMI changes
  }
  byte_length = 8
}
```

---

## Provider Aliases — Exam Syntax

Provider aliases are tested directly. Know the exact syntax:

```hcl
# Default provider — no alias
provider "aws" {
  region = "us-east-1"
}

# Aliased provider — requires explicit reference
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Uses default provider (us-east-1)
resource "aws_s3_bucket" "primary" {
  bucket = "my-primary-bucket"
}

# Uses aliased provider (us-west-2)
resource "aws_s3_bucket" "replica" {
  provider = aws.west
  bucket   = "my-replica-bucket"
}
```

**Exam point:** If all providers have aliases and a resource does not specify `provider`, Terraform errors — it cannot determine which provider to use. A default provider (no alias) must exist for implicit provider assignment to work.

---

## Five Practice Questions

**Question 1:**
You run `terraform state rm aws_s3_bucket.logs`. What happens to the S3 bucket in AWS?

- A) The bucket is deleted from AWS
- B) The bucket is renamed in AWS
- C) Nothing — the bucket remains in AWS unchanged
- D) The bucket is moved to a different region

**Answer: C** — `terraform state rm` only removes the resource from the state file. It has zero effect on real infrastructure. After this command, Terraform no longer manages the bucket but the bucket still exists in AWS.

---

**Question 2:**
Which file should always be committed to version control and why?

- A) `terraform.tfstate` — so team members share state
- B) `.terraform.lock.hcl` — so team members use the same provider version
- C) `terraform.tfvars` — so variable values are shared
- D) `.terraform/providers` — so provider binaries are shared

**Answer: B** — The lock file records the exact provider version selected and must be committed so all team members and CI systems use identical provider versions. The state file must never be in Git (contains secrets), `.tfvars` often contain secrets, and provider binaries are large and platform-specific.

---

**Question 3:**
A team member manually terminates an EC2 instance that Terraform manages. What does `terraform plan` show?

- A) No changes — Terraform only detects changes made through Terraform
- B) The instance will be created — Terraform detects the drift and plans to recreate it
- C) An error — Terraform cannot continue when state and reality diverge
- D) The instance will be imported — Terraform automatically adopts the replacement

**Answer: B** — Terraform compares state against real AWS on every plan. The instance exists in state but not in AWS — Terraform detects this drift and plans to recreate the resource to match the desired configuration.

---

**Question 4:**
What is the correct way to reference an output from a module called `webserver`?

- A) `output.webserver.alb_dns_name`
- B) `webserver.output.alb_dns_name`
- C) `module.webserver.alb_dns_name`
- D) `var.webserver.alb_dns_name`

**Answer: C** — Module outputs are referenced with `module.<module_name>.<output_name>`. The `module.` prefix is required to distinguish module outputs from resource attributes and variables.

---

**Question 5:**
You need to rename a resource in your Terraform configuration from `aws_instance.old_name` to `aws_instance.new_name` without destroying and recreating the instance. Which command do you use?

- A) `terraform state mv aws_instance.old_name aws_instance.new_name`
- B) `terraform import aws_instance.new_name <instance_id>`
- C) `terraform apply -replace aws_instance.old_name`
- D) Edit the state file directly and change the name

**Answer: A** — `terraform state mv` renames a resource within the state file without touching real infrastructure. After running this command, update your `.tf` file to use `new_name` and run `terraform plan` to confirm no changes are planned. Option B would work but is more complex. Option C forces recreation. Option D is dangerous and should never be done.

---

## My Personal Study Plan

| Topic | Confidence | Study Method | Time |
|---|---|---|---|
| `terraform state` commands | Yellow | Run each against test resource, write notes | 45 min |
| Sentinel policy syntax | Yellow | Read docs, write two policies, test with CLI | 60 min |
| Workspace vs file layout | Green | Write one practice question | 15 min |
| Non-cloud providers | Yellow | Deploy random + local provider locally | 30 min |
| Terraform Cloud variables | Yellow | Configure workspace variables, test sensitive flag | 30 min |
| Provider alias syntax | Green | Write two practice questions | 20 min |
| Backend configuration options | Yellow | Read docs on all backend types | 45 min |
| `terraform import` workflow | Yellow | Import one real resource, verify clean plan | 45 min |
| Official practice questions | Yellow | Complete all, review every wrong answer | 60 min |

---

## What I Learned Today

- **The CLI domain is 26% of the exam** — the highest single domain weight. Do not underestimate it
- **`terraform state rm` has zero effect on real infrastructure** — this distinction is tested repeatedly
- **The lock file must be committed** — the state file must not be committed. These are tested as a pair
- **Non-cloud providers** (`random`, `local`) appear in exam questions — practice them hands-on
- **Writing practice questions** forces deeper understanding than reading — you have to understand material well enough to construct plausible wrong answers
- **Drift detection** — Terraform always checks real AWS on every plan, not just the state file

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*