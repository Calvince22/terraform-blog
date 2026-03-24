# Advanced Terraform Module Usage: Versioning, Gotchas, and Reuse Across Environments

*Day 9 of the #30DayTerraformChallenge*

---

## Hello 

Yesterday I built my first Terraform module and called it from dev and production. It worked great.

But there were things I did not know that could have caused subtle, hard-to-debug problems. Today I learned those things — and fixed them before they bit me in production.

Today covered three topics:

- **Module Gotchas** — three common mistakes that look fine but cause real bugs
- **Module Versioning** — how to pin modules to specific versions using GitHub tags
- **Multi-environment deployment** — running different module versions in dev and production safely

Let me walk through all three.

---

## Part 1 — The Three Module Gotchas

These are mistakes that do not cause immediate errors. They cause *subtle* problems that are hard to trace back to their source. Knowing them now saves hours of debugging later.

---

### Gotcha 1 — File Paths Inside Modules

If your module references a file using a relative path, that path is resolved from wherever you run `terraform` — not from where the module lives.

**The broken version:**

```hcl
# Inside modules/services/webserver-cluster/main.tf
resource "aws_launch_template" "web" {
  user_data = filebase64("./user-data.sh")  # ← WRONG
}
```

When you call this module from `live/dev/services/webserver-cluster/`, Terraform looks for `user-data.sh` in the `live/dev/services/webserver-cluster/` directory — not in the module folder. It will not find it and will error.

**The correct version:**

```hcl
# Inside modules/services/webserver-cluster/main.tf
resource "aws_launch_template" "web" {
  user_data = filebase64("${path.module}/user-data.sh")  # ← CORRECT
}
```

`path.module` always resolves to the directory where the module's `.tf` files live — regardless of where Terraform is being run from. Use it any time you reference a file inside a module.

Similarly if you want the path of the root configuration, use `path.root`. And for the current working directory, use `path.cwd`.

---

### Gotcha 2 — Inline Blocks vs Separate Resources

Some AWS resources support two ways of defining the same thing — as an inline block or as a separate resource. Security group rules are the classic example.

**Inline block approach (inside the security group):**

```hcl
resource "aws_security_group" "instance_sg" {
  name = "my-sg"

  ingress {                    # ← inline block
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Separate resource approach:**

```hcl
resource "aws_security_group" "instance_sg" {
  name = "my-sg"
  # no inline blocks here
}

resource "aws_security_group_rule" "allow_http" {   # ← separate resource
  type              = "ingress"
  security_group_id = aws_security_group.instance_sg.id
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

**The problem:** If you mix both — inline blocks AND separate `aws_security_group_rule` resources — Terraform gets confused and the rules fight each other. You will end up with rules being added and removed on every apply, even when nothing has changed.

**The fix:** Pick one approach and stick to it throughout the module. For modules specifically, **separate resources are better** because they allow the caller to add extra rules without modifying the module itself.

---

### Gotcha 3 — Module Output Dependencies

If your root configuration uses `depends_on` with a module output, Terraform treats the **entire module** as the dependency — not just the specific resource you care about.

**The problem:**

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  depends_on = [module.webserver_cluster]  # ← depends on the whole module
}
```

This tells Terraform: "do not create `aws_instance.app` until every single resource inside `module.webserver_cluster` is done." This can cause unnecessary resource recreation — if anything inside the module changes, Terraform may decide to recreate resources that depend on it even if they are not actually affected.

**The fix:** Expose granular outputs from your module and depend on those instead:

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = module.webserver_cluster.subnet_id  # ← specific output

  # No depends_on needed — the reference creates an implicit dependency
}
```

When you reference a specific module output, Terraform automatically understands the dependency chain without needing an explicit `depends_on`. Only add `depends_on` on a module when there is genuinely no other way to express the dependency.

---

## Part 2 — Module Versioning with GitHub

### Why Versioning Matters

Without version pinning, your module `source` is just a pointer to "whatever is currently in that directory." If someone updates the module, every environment that references it will pull the changes on the next `terraform init`.

In a team environment this means:

- Engineer A updates the module and pushes to GitHub
- Engineer B runs `terraform apply` on production without knowing the module changed
- Production gets a change nobody planned

Version pinning solves this. You pin each environment to a specific version. When a new version is ready and tested in dev, you deliberately upgrade production.

---

### Step 1 — Push the Module to GitHub

```bash
cd modules/services/webserver-cluster

git init
git add .
git commit -m "Initial release of webserver-cluster module"
git tag -a "v0.0.1" -m "First stable release"
git remote add origin https://github.com/your-username/terraform-aws-webserver-cluster
git push origin main --tags
```

Confirm both commits and tags pushed:

```bash
git tag -l
```

Output:
```
v0.0.1
```

---

### Step 2 — Make a Change and Tag v0.0.2

I added a new variable `enable_instance_id` that controls whether the instance ID is shown on the web page — a small but meaningful change:

```hcl
# Added to variables.tf
variable "enable_instance_id" {
  description = "Whether to display the instance ID on the web page"
  type        = bool
  default     = true
}
```

Then updated the `user_data` script in `main.tf` to conditionally show it:

```hcl
user_data = base64encode(<<-EOF
            #!/bin/bash
            apt-get update -y
            apt-get install -y apache2
            systemctl start apache2
            systemctl enable apache2
            echo "<h1>${var.server_message}</h1>" > /var/www/html/index.html
            %{ if var.enable_instance_id }
            echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
            %{ endif }
            EOF
)
```

Commit and tag the new version:

```bash
git add .
git commit -m "Add enable_instance_id variable"
git tag -a "v0.0.2" -m "Add optional instance ID display"
git push origin main --tags
```

```bash
git tag -l
```

Output:
```
v0.0.1
v0.0.2
```

---

## Part 3 — Different Versions Across Environments

Now the key pattern. Dev uses the latest version for testing. Production stays pinned to the last stable version until the new one is validated.

**live/dev/services/webserver-cluster/main.tf**

```hcl
provider "aws" {
  region = "us-east-1"
}

# Dev uses v0.0.2 — testing the new enable_instance_id feature
module "webserver_cluster" {
  source = "github.com/your-username/terraform-aws-webserver-cluster?ref=v0.0.2"

  cluster_name       = "webservers-dev"
  instance_type      = "t3.micro"
  min_size           = 2
  max_size           = 4
  server_message     = "Hello from Dev!"
  enable_instance_id = true   # ← new variable available in v0.0.2
}

output "alb_dns_name" {
  value = module.webserver_cluster.alb_dns_name
}
```

**live/production/services/webserver-cluster/main.tf**

```hcl
provider "aws" {
  region = "us-east-1"
}

# Production stays on v0.0.1 — stable and tested
module "webserver_cluster" {
  source = "github.com/your-username/terraform-aws-webserver-cluster?ref=v0.0.1"

  cluster_name   = "webservers-production"
  instance_type  = "t3.medium"
  min_size       = 4
  max_size       = 10
  server_message = "Hello from Production!"
  # enable_instance_id not available in v0.0.1 — not included here
}

output "alb_dns_name" {
  value = module.webserver_cluster.alb_dns_name
}
```

When you run `terraform init` in each directory, Terraform downloads the specific version for that environment:

```bash
cd live/dev/services/webserver-cluster
terraform init
```

```
Initializing modules...
Downloading github.com/your-username/terraform-aws-webserver-cluster?ref=v0.0.2
for webserver_cluster...
- webserver_cluster in .terraform/modules/webserver_cluster

Terraform has been successfully initialized!
```

Production downloads v0.0.1. Dev downloads v0.0.2. They are completely independent.

---

## Module Source Types — Quick Reference

Terraform supports several ways to reference a module source:

| Source Type | Example | When to Use |
|---|---|---|
| Local path | `"../../../../modules/services/webserver-cluster"` | Module lives in the same repo |
| GitHub (no version) | `"github.com/user/repo"` | Quick test — not for production |
| GitHub (pinned) | `"github.com/user/repo?ref=v0.0.1"` | Recommended for shared modules |
| Terraform Registry | `"hashicorp/consul/aws"` | Official or published modules |
| S3 bucket | `"s3::https://s3.amazonaws.com/bucket/module.zip"` | Private module storage |

Always use a versioned source for anything shared across a team. An unpinned source is a ticking time bomb.

---

## The Module README

Every shared module needs a README. Here is what mine looks like:

````markdown
# terraform-aws-webserver-cluster

A reusable Terraform module that deploys a load-balanced, auto-scaling
web server cluster on AWS using an Application Load Balancer and EC2
Auto Scaling Group.

## Usage

```hcl
module "webserver_cluster" {
  source = "github.com/your-username/terraform-aws-webserver-cluster?ref=v0.0.2"

  cluster_name  = "my-cluster"
  min_size      = 2
  max_size      = 5
}
```

## Inputs

| Name | Description | Type | Default | Required |
|---|---|---|---|---|
| `cluster_name` | Name prefix for all resources | string | — | yes |
| `instance_type` | EC2 instance type | string | `t3.micro` | no |
| `min_size` | Minimum instances in ASG | number | — | yes |
| `max_size` | Maximum instances in ASG | number | — | yes |
| `server_port` | Port the server listens on | number | `80` | no |
| `server_message` | Message on the web page | string | `Hello from Terraform!` | no |
| `enable_instance_id` | Show instance ID on page | bool | `true` | no |

## Outputs

| Name | Description |
|---|---|
| `alb_dns_name` | DNS name of the load balancer |
| `asg_name` | Name of the Auto Scaling Group |
| `alb_security_group_id` | ID of the ALB security group |

## Known Limitations

- Uses the default VPC. For production, provide a custom VPC and subnets.
- `cluster_name` must be unique per AWS account and region to avoid
  resource naming conflicts.
````

---

## Problems I Ran Into

### ❌ Problem 1: `terraform init` Not Re-downloading After Version Change

I updated the `source` in my calling config from `v0.0.1` to `v0.0.2` and ran `terraform apply`. It still used the old version.

**What happened:** Terraform caches downloaded modules in `.terraform/modules/`. Changing the version in the source does not automatically clear the cache.

**Fix:** Run `terraform init` again after changing a module version. Terraform will detect the version change and download the new one:

```bash
terraform init -upgrade
```

The `-upgrade` flag forces Terraform to re-check all module sources and download updated versions even if a cached version exists.

---

### ❌ Problem 2: GitHub Authentication Error

When I ran `terraform init` with a GitHub source, I got:

```
Error: Failed to download module

Could not download module "webserver_cluster" source code from
"https://github.com/your-username/terraform-aws-webserver-cluster":
error downloading
'https://github.com/your-username/terraform-aws-webserver-cluster':
/usr/bin/git exited with 128: fatal: could not read Username for
'https://github.com': terminal prompts disabled
```

**What happened:** Terraform uses Git to download modules from GitHub. It could not authenticate because I was using HTTPS and had not configured Git credentials.

**Fix:** Two options — switch to SSH or configure a GitHub token.

I used SSH since my key was already set up:

```hcl
# Use SSH format instead of HTTPS
source = "git@github.com:your-username/terraform-aws-webserver-cluster.git?ref=v0.0.2"
```

Alternatively, set up a GitHub personal access token and configure Git to use it.

---

### ❌ Problem 3: Tag Not Pushed to GitHub

After tagging locally and updating the source to `?ref=v0.0.2`, `terraform init` failed with:

```
Error: Failed to download module

Could not checkout 'v0.0.2' in /path/to/module
```

**What happened:** I created the tag locally but forgot to push it to GitHub. The tag existed on my machine but not in the remote repository, so Terraform could not find it.

**Fix:**

```bash
git push origin --tags
```

After pushing the tags, `terraform init` found `v0.0.2` and downloaded it successfully.

> 💡 Always remember: `git push` pushes commits. Tags need a separate `git push origin --tags` or `git push origin v0.0.2` to reach the remote.

---

## What I Learned Today

- **`path.module`** always resolves to the module's own directory — use it for any file references inside a module
- **Never mix inline blocks and separate resources** for the same configuration — pick one and stick to it
- **`depends_on` on a whole module** forces full module recreation — use specific output references instead
- **Version pinning** protects production from unplanned changes — always use `?ref=vX.Y.Z` for shared modules
- **`terraform init -upgrade`** is needed when you change a module version — the regular `init` uses the cache
- **Tags must be explicitly pushed** to GitHub — they do not go up with a regular `git push`
- **A README is not optional** for a shared module — an undocumented module is an unusable module

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
