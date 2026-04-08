# State Isolation: Workspaces vs File Layouts — When to Use Each

*Day 7 of the #30DayTerraformChallenge*

---

## Hello

Yesterday I moved Terraform state to a remote S3 backend. Today I tackled the next problem: **what happens when you need dev, staging, and production environments that never interfere with each other?**

Terraform gives you two approaches to solve this:

- **Workspaces** — multiple state files within the same configuration directory
- **File Layouts** — separate directories per environment, each with its own backend

I implemented both today. By the end I had a clear opinion on when each one is appropriate — and which one I would trust in production.

---

## The Problem We Are Solving

Imagine you deploy your web server to dev, test it, and it works. Then you run `terraform apply` again — but this time you are accidentally pointing at production. Your production server gets replaced.

This is not a hypothetical. It happens. State isolation is how you prevent it.

The goal is simple: **dev, staging, and production should have completely separate state files so changes in one environment can never touch another.**

---

## Approach 1 — Terraform Workspaces

### What Are Workspaces?

A workspace is like a named slot for a state file inside the same backend and the same configuration directory.

By default every Terraform project has one workspace called `default`. You can create more:

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new production
```

Each workspace gets its own state file in S3, stored at a different path automatically. The code is shared — only the state is separate.

### Setting Up Workspaces

```bash
# Create the three environment workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new production

# See all workspaces
terraform workspace list
```

Output:

```
  default
* dev
  production
  staging
```

The `*` shows the currently active workspace.

```bash
# Switch to a specific workspace
terraform workspace select dev
```

### Making Configuration Respond to the Workspace

The real power of workspaces is using `terraform.workspace` inside your code to change behaviour per environment.

First, define your instance types as a map in `variables.tf`:

```hcl
variable "instance_type" {
  description = "EC2 instance type per environment"
  type        = map(string)
  default = {
    dev        = "t3.micro"
    staging    = "t3.small"
    production = "t3.medium"
  }
}
```

Then use `terraform.workspace` as the map key in your Launch Template so each environment gets the right instance size:

```hcl
resource "aws_launch_template" "web_server" {
  name_prefix   = "${var.server_name}-"
  image_id      = data.aws_ami.ubuntu_22_04.id
  instance_type = var.instance_type[terraform.workspace]  # ← picks size based on workspace

  vpc_security_group_ids = [aws_security_group.instance_sg.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y apache2
              systemctl restart apache2
              systemctl enable apache2
              echo "<h1>${var.server_message}</h1>" > /var/www/html/index.html
              echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
              EOF
  )

  # Tags go inside a tag_specifications block for launch templates
  tag_specifications {
    resource_type = "instance"

    tags = {
      Name        = "web-${terraform.workspace}"
      Environment = terraform.workspace
    }
  }
}
```

Two things worth noting about this code:

**`instance_type = var.instance_type[terraform.workspace]`** — this looks up the map using the current workspace name as the key. In `dev` it returns `t3.micro`, in `production` it returns `t3.medium`. Same code, different behaviour per environment.

**`tag_specifications` block** — this is how tags work inside a Launch Template. Unlike `aws_instance` where you put `tags = {}` directly on the resource, a Launch Template uses a nested `tag_specifications` block with a `resource_type` to specify what is being tagged (the EC2 instance in this case). Putting `tags = {}` directly at the top level of `aws_launch_template` would tag the template itself — not the instances it launches. This is a common mistake that looks correct but produces unexpected results.

### The Backend Configuration for Workspaces

With `use_lockfile = true` the S3 backend handles locking natively — no DynamoDB table needed:

```hcl
terraform {
  backend "s3" {
    bucket       = "your-terraform-state-bucket"
    key          = "workspaces/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

Terraform automatically creates separate state paths in S3 for each workspace:

```
your-terraform-state-bucket/
├── workspaces/
│   ├── terraform.tfstate          ← default workspace
│   ├── env:/dev/terraform.tfstate
│   ├── env:/staging/terraform.tfstate
│   └── env:/production/terraform.tfstate
```

Each environment has its own state file. A change in `dev` does not touch `staging` or `production`.

### Deploying to Each Workspace

```bash
# Deploy to dev
terraform workspace select dev
terraform apply

# Deploy to staging
terraform workspace select staging
terraform apply

# Deploy to production
terraform workspace select production
terraform apply
```

---

## Approach 2 — File Layout Isolation

### What Is File Layout Isolation?

Instead of one directory with multiple workspaces, you create a completely separate directory for each environment. Each directory has its own `main.tf`, `variables.tf`, `outputs.tf`, and `backend.tf`.

```
environments/
├── dev/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── backend.tf
├── staging/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── backend.tf
└── production/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── backend.tf
```

### The Backend Configuration Per Environment

Each environment has its own `backend.tf` pointing to a unique `key` path in S3:

**environments/dev/backend.tf**

```hcl
terraform {
  backend "s3" {
    bucket       = "your-terraform-state-bucket"
    key          = "environments/dev/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

**environments/staging/backend.tf**

```hcl
terraform {
  backend "s3" {
    bucket       = "your-terraform-state-bucket"
    key          = "environments/staging/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

**environments/production/backend.tf**

```hcl
terraform {
  backend "s3" {
    bucket       = "your-terraform-state-bucket"
    key          = "environments/production/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

The only difference between them is the `key` path. This is what keeps the state files completely separate in S3:

```
your-terraform-state-bucket/
├── environments/
│   ├── dev/
│   │   └── terraform.tfstate
│   ├── staging/
│   │   └── terraform.tfstate
│   └── production/
│       └── terraform.tfstate
```

### Deploying with File Layouts

You `cd` into each directory and run Terraform independently:

```bash
# Deploy dev
cd environments/dev
terraform init
terraform apply

# Deploy production — completely separate, cannot affect dev
cd ../production
terraform init
terraform apply
```

Because these are separate directories, you cannot accidentally apply production code while working in dev. The directory you are in makes it obvious which environment you are touching.

---

## Approach 3 — Remote State Data Source

Once you have separate state files per environment, you sometimes need one environment to read outputs from another. For example, your application layer needs to know the VPC ID created by your networking layer.

The `terraform_remote_state` data source solves this:

```hcl
# In your application layer — reads outputs from the networking state file
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "your-terraform-state-bucket"
    key    = "environments/dev/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the output from the networking state file
resource "aws_instance" "web" {
  ami       = "ami-0c55b159cbfafe1f0"
  subnet_id = data.terraform_remote_state.network.outputs.subnet_id

  tags = {
    Name = "web-server"
  }
}
```

This lets separate Terraform configurations share information without being in the same directory or the same state file.

**Important limitation:** The remote state data source only exposes values that were explicitly defined as `output` blocks in the source configuration. If the value you need was not outputted, you cannot access it this way.

---

## Workspaces vs File Layouts — Honest Comparison

Here is the honest side-by-side:

| | Workspaces | File Layouts |
|---|---|---|
| Code isolation | ❌ Same code for all environments | ✅ Each environment can have different code |
| State isolation | ✅ Separate state per workspace | ✅ Separate state per directory |
| Risk of wrong environment | ⚠️ High — easy to forget which workspace you are in | ✅ Low — directory makes it obvious |
| Setup effort | ✅ Low — one directory, a few commands | ⚠️ Higher — duplicate files across directories |
| Scales across large teams | ⚠️ Risky — shared code means shared risk | ✅ Better — each team owns their directory |
| Good for production | ⚠️ Not recommended | ✅ Recommended |

### When to Use Workspaces

Workspaces are fine when:

- You are working alone or in a very small team
- The environments are genuinely identical except for a few variable values
- You are spinning up temporary environments for testing (e.g. feature branch environments)

### When to Use File Layouts

File layouts are better when:

- You have a real team working on shared infrastructure
- Production needs different configuration from dev — not just different variable values
- You want the strongest possible guarantee that a deploy to dev cannot touch production

### My Recommendation

**Use file layouts for anything that matters.**

The reason is simple: with workspaces, there is nothing stopping you from running `terraform apply` in the wrong workspace. With file layouts, you have to physically `cd` into the production directory. That extra friction is a feature, not a bug.

The only time I would reach for workspaces is for short-lived test environments that mirror dev exactly — where the goal is speed, not safety.

---

## State Locking Across Environments

With `use_lockfile = true`, the S3 backend creates a `.tflock` file in the same bucket path as the state file when an operation is in progress.

Because each workspace and each file layout environment has its own state file at a different path, they each have their own lock file too. **There is no risk of two environments locking each other** — a lock on `environments/dev/terraform.tfstate` has zero effect on `environments/production/terraform.tfstate`.

I tested this by running `terraform apply` in dev and `terraform apply` in production at the same time. Both ran without blocking each other — because the lock files are at completely different paths.

---

## Problems I Ran Into

### ❌ Problem 1: Applied to the Wrong Workspace

I switched to the `staging` workspace, made a change, and ran `terraform apply`. It worked — but then I realised I had forgotten to switch back to `dev` first and had just modified staging infrastructure by mistake.

**What happened:** There is no confirmation prompt telling you which workspace you are in before apply runs.

**Fix:** I added this habit — always run `terraform workspace show` before any apply:

```bash
terraform workspace show
# staging

terraform workspace select dev
terraform workspace show
# dev

terraform apply
```

I also added the current workspace to my shell prompt so it is always visible. This is the biggest practical risk with workspaces and it is worth building the habit early.

---

### ❌ Problem 2: `terraform init` Required for Every New Environment Directory

When I set up the file layout and `cd` into each environment directory for the first time, I had to run `terraform init` separately in each one.

```bash
cd environments/dev && terraform init
cd environments/staging && terraform init
cd environments/production && terraform init
```

**What happened:** Each directory is a completely separate Terraform project. The `.terraform` folder that `init` creates is local to each directory and not shared.

**Fix:** This is expected behaviour — not really a bug. I just had to remember to always `terraform init` when working in a new directory for the first time. If you skip it you will get:

```
Error: Backend initialization required, please run "terraform init"
```

---

###  Problem 4: VPC Limit Exceeded When Deploying Multiple Environments
When I deployed to multiple workspaces back to back — dev, staging, and production — I hit this error on one of the applies:
```bash
Error: creating VPC: VpcLimitExceeded: The maximum number of VPCs
has been reached for this account in this region.
```
Fix — Option 1: Request a VPC limit increase
If you genuinely need more than 5 VPCs — for example running many environments simultaneously — you can request a limit increase from AWS:

Go to Service Quotas in the AWS Console
Search for VPCs per Region
Click Request quota increase and submit
---

## What I Learned Today

- **Workspaces share code** — all environments run the same `.tf` files, only state is separate
- **File layouts isolate everything** — separate directories mean separate code, separate state, separate risk
- **`terraform.workspace`** lets you make configuration conditional on the environment — useful for instance sizing and naming
- **`use_lockfile = true`** handles locking directly in S3 — no DynamoDB table needed
- **Locks are per state file** — separate environments never block each other
- **File layouts are safer for production** — the directory you are in makes it physically impossible to accidentally touch the wrong environment
- **`terraform_remote_state`** lets configurations share outputs across separate state files

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
