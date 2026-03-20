# Managing High Traffic Applications with AWS Elastic Load Balancer and Terraform

*Day 5 of the #30DayTerraformChallenge*

---

## Hello

**Topic 1 — Scaling with a Load Balancer**
I completed my production-ready cluster by connecting an Application Load Balancer to the Auto Scaling Group from Day 4. The app can now handle real traffic and survive individual server failures.

**Topic 2 — Terraform State**
I learned what the `terraform.tfstate` file actually is, what happens when it gets out of sync with AWS, and why managing it correctly is one of the most critical habits in infrastructure engineering.

---

## Part 1 — The Elastic Load Balancer Setup

If you followed Day 4, you already have an Auto Scaling Group running multiple EC2 instances. Today I put an **Application Load Balancer (ALB)** in front of them.

Here is what the full picture looks like now:

```
Your Browser
     |
     | HTTP on Port 80
     |
[Application Load Balancer]
     |          |          |
  [Server 1] [Server 2] [Server 3]
  [AZ: 1a]   [AZ: 1b]   [AZ: 1c]
     |
[AWS Region: us-east-1]
```

The Load Balancer is the only thing the internet touches. The individual servers are hidden behind it — users never connect to them directly.

---

### The Terraform Code

Here is my complete configuration. I will explain each block below.

**variables.tf**

```hcl
variable "server_port" {
  description = "The port the server will use for HTTP requests"
  type        = number
  default     = 80
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "server_name" {
  description = "Name prefix for all resources"
  type        = string
  default     = "terraform-web-server-day5"
}

variable "asg_min_size" {
  description = "Minimum number of instances in the ASG"
  type        = number
  default     = 2
}

variable "asg_max_size" {
  description = "Maximum number of instances in the ASG"
  type        = number
  default     = 5
}

variable "asg_desired_capacity" {
  description = "Desired number of instances in the ASG"
  type        = number
  default     = 2
}

variable "server_message" {
  description = "Message displayed on the web page"
  type        = string
  default     = "Hello from Terraform — Day 5!"
}

variable "alb_listener_port" {
  description = "The port the ALB listens on"
  type        = number
  default     = 80
}
```

---

**main.tf**

```hcl
provider "aws" {
  region = var.aws_region
}

# --- DATA SOURCES ---

# Fetch the latest Ubuntu 22.04 AMI automatically
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  owners = ["099720109477"]  # Canonical's official account
}

# Fetch the default VPC
data "aws_vpc" "default" {
  default = true
}

# Fetch subnets in the default VPC
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# --- SECURITY GROUPS ---

# Security group for EC2 instances
# Only allows traffic from the ALB — not directly from the internet
resource "aws_security_group" "instance_sg" {
  name        = "${var.server_name}-instance-sg"
  description = "Allow HTTP from ALB only"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port       = var.server_port
    to_port         = var.server_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]  # ← only from ALB
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security group for the ALB
# Accepts traffic from the internet on port 80
resource "aws_security_group" "alb_sg" {
  name        = "${var.server_name}-alb-sg"
  description = "Allow HTTP from internet"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = var.alb_listener_port
    to_port     = var.alb_listener_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ← internet can reach the ALB
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# --- LAUNCH TEMPLATE ---

# Defines what each EC2 instance looks like when it boots
resource "aws_launch_template" "web" {
  name_prefix   = "${var.server_name}-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.instance_sg.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y apache2
              systemctl start apache2
              systemctl enable apache2
              echo "<h1>${var.server_message}</h1>" > /var/www/html/index.html
              EOF
  )

  tags = {
    Name = var.server_name
  }
}

# --- AUTO SCALING GROUP ---

# Manages the cluster — keeps 2 to 5 instances running at all times
resource "aws_autoscaling_group" "web" {
  desired_capacity    = var.asg_desired_capacity
  min_size            = var.asg_min_size
  max_size            = var.asg_max_size
  vpc_zone_identifier = data.aws_subnets.default.ids

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  # Connect the ASG to the ALB Target Group
  target_group_arns = [aws_lb_target_group.web.arn]
  health_check_type = "ELB"

  tag {
    key                 = "Name"
    value               = "${var.server_name}-instance"
    propagate_at_launch = true
  }
}

# --- APPLICATION LOAD BALANCER ---

# The public-facing entry point for all traffic
resource "aws_lb" "web" {
  name               = "${var.server_name}-alb"
  internal           = false          # public-facing
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = data.aws_subnets.default.ids
}

# --- TARGET GROUP ---

# The list of healthy instances the ALB routes traffic to
resource "aws_lb_target_group" "web" {
  name     = "${var.server_name}-tg"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

# --- LISTENER ---

# Watches port 80 on the ALB and forwards requests to the Target Group
resource "aws_lb_listener" "web" {
  load_balancer_arn = aws_lb.web.arn
  port              = var.alb_listener_port
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# --- OUTPUT ---

# Prints the ALB DNS name after deployment
output "alb_dns_name" {
  value       = aws_lb.web.dns_name
  description = "The DNS name of the load balancer — paste this in your browser"
}
```

---

### Running It

```bash
terraform init
terraform plan
terraform apply
```

After apply:

```
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:
alb_dns_name = "terraform-web-server-day5-alb-123456789.us-east-1.elb.amazonaws.com"
```

Paste the DNS name into your browser and you will see:

```
Hello from Terraform — Day 5!
```

### Testing High Availability

To prove the setup actually works, I stopped one of the EC2 instances manually in the AWS Console. Within about 30 seconds:

- The ALB stopped sending traffic to the stopped instance
- The remaining instance kept serving the page without interruption
- The Auto Scaling Group noticed the instance was gone and launched a replacement automatically

That is high availability working exactly as intended.

---

### Clean Up

```bash
terraform destroy
```

> Always destroy after each session. These resources cost money while running.

---

## Part 2 — Understanding Terraform State

This was the most important concept I learned today — and it is one that separates beginners from engineers who can be trusted with production systems.

### What Is the State File?

Every time you run `terraform apply`, Terraform writes a file called `terraform.tfstate` to your project folder.

This file is Terraform's memory. It records every resource it created — the IDs, the configurations, the relationships between resources, everything.

Here is a small example of what it looks like inside:

```json
{
  "version": 4,
  "terraform_version": "1.7.5",
  "resources": [
    {
      "type": "aws_instance",
      "name": "web_server",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",
            "instance_type": "t3.micro",
            "ami": "ami-0c55b159cbfafe1f0",
            "tags": {
              "Name": "terraform-web-server-day5"
            }
          }
        }
      ]
    }
  ]
}
```

Think of it like this: your `.tf` files describe what you *want*. The state file records what actually *exists*. Every time you run `terraform plan`, Terraform compares these two things and tells you what needs to change.

---

### Experiment 1 — What Happens When You Edit the State File Manually

I opened `terraform.tfstate` in VS Code and changed a value inside it — without touching any of my actual Terraform code.

Then I ran:

```bash
terraform plan
```

This is the output I got:

```
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration
and found no differences, so no changes are needed.
```

This was not what I expected. I thought Terraform would detect the manual edit and flag it. But it did not — and the reason why is actually one of the most important things I learned today.

**Why Terraform showed no changes:**

Terraform does not just read the state file in isolation. When you run `terraform plan`, it does three things:

1. Reads your `.tf` configuration files — what you *want*
2. Reads the state file — what it *thinks* exists
3. **Makes live API calls to AWS** — what *actually* exists right now

The comparison that drives the plan output is between your **code** and **real AWS** — not between your code and the state file alone.

When I edited the state file manually, the value in AWS had not changed. So when Terraform called the AWS API to check the real state of that resource, it matched the code exactly. Result: no changes needed.

This revealed something important — the state file is not the ultimate source of truth. **AWS itself is the source of truth.** The state file is Terraform's cached record of what it last knew about AWS. If you edit the state file but not the real resource in AWS, Terraform will still see what is actually in AWS and plan accordingly.

```
State file (edited) ──┐
                       ├──→ Terraform compares CODE vs REAL AWS → No diff found
Real AWS (unchanged) ──┘
```

**The real danger of editing the state file:**

The risk is not that Terraform immediately breaks — it is that the state file becomes a lie. Over time, as more changes happen, the state file diverges further from reality. Eventually Terraform starts making incorrect decisions — trying to create resources that already exist, or failing to track resources it should be managing.

This is why the rule exists: **Terraform is the only thing that should ever write to the state file.** If you genuinely need to modify state, use the proper commands:

```bash
terraform state list           # see what Terraform is tracking
terraform state show <resource> # inspect a specific resource
terraform state rm <resource>   # remove a resource from state
terraform import <resource> <id> # add an existing resource to state
```

These commands update state safely and keep the record accurate.

---

### Experiment 2 — What Happens When You Change Something in AWS Directly

This one was the most eye-opening experiment of the day.

I went into the AWS Console and manually changed the `Name` tag on my Auto Scaling Group from `terraform-web-server-asg` to something different — without touching a single line of Terraform code.

Then I ran:

```bash
terraform plan
```

Terraform detected the drift immediately and produced this output:

```
Terraform will perform the following actions:

  # aws_autoscaling_group.web_asg will be updated in-place
  ~ resource "aws_autoscaling_group" "web_asg" {
        id   = "terraform-20260320094106952000000003"
        name = "terraform-20260320094106952000000003"
        # (32 unchanged attributes hidden)

      - tag {
          - key                 = "Name" -> null
          - propagate_at_launch = true -> null
          - value               = "terraform-web-server-asg" -> null
        }
      + tag {
          + key                 = "Name"
          + propagate_at_launch = true
          + value               = "terraform-web-server-day4-asg"
        }

        # (4 unchanged blocks hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Let me break down exactly what Terraform is showing here:

**The red `-` lines** show what Terraform currently sees in AWS — the tag that was manually changed. Terraform is saying "I see this tag exists in AWS right now."

**The green `+` lines** show what the Terraform code says the tag should be — the original value defined in `main.tf`.

**The plan result** is `1 to change` — meaning Terraform wants to update the ASG tag back to what the code defines. It is not going to delete and recreate the ASG — just update the tag in place.

This is called **state drift** — when the real state of your infrastructure in AWS is different from what your Terraform code and state file say it should be.

The critical takeaway: **Terraform always wins.** The moment someone runs `terraform apply`, it will restore the tag back to `terraform-web-server-day4-asg` and overwrite the manual change. This is why you should never manually change infrastructure that Terraform manages — your changes will be silently undone on the next deploy.

In a real team, this causes real confusion: an engineer manually fixes something in the AWS Console, another engineer runs `terraform apply` an hour later, and the manual fix disappears with no warning.

---

### Why You Should Never Commit the State File to Git

This is a rule most people learn the hard way. The state file contains sensitive information — resource IDs, IP addresses, and sometimes even passwords or secrets depending on what you have deployed.

If you commit `terraform.tfstate` to a public GitHub repository, you are leaking infrastructure details that attackers can use.

The right approach is to add it to `.gitignore`:

```
# .gitignore
terraform.tfstate
terraform.tfstate.backup
.terraform/
```

And store it remotely instead — in an S3 bucket, Terraform Cloud, or another backend. We will cover remote state properly in the coming days.

---

### What Is State Locking?

Imagine two engineers on your team both run `terraform apply` at the same time. They are both reading the same state file, making changes, and trying to write back to it simultaneously.

The result? Corrupted state. Resources get created twice. Configurations conflict. Things break in ways that are very hard to debug.

**State locking** solves this. When one person runs `terraform apply`, the state file gets locked — no one else can run apply until the first one finishes.

In AWS, locking is typically handled by a **DynamoDB table** paired with your S3 remote backend. We will set this up properly on Day 8.

---

## Terraform Block Types — Quick Reference

Here is a summary of every block type covered so far:

| Block | What It Does | When to Use | Example |
|---|---|---|---|
| `provider` | Tells Terraform which cloud to use | Once per cloud platform | `provider "aws" { region = "us-east-1" }` |
| `resource` | Creates a piece of infrastructure | Every resource you want to build | `resource "aws_instance" "web" { ... }` |
| `variable` | Defines an input value | To avoid hardcoding values | `variable "instance_type" { default = "t3.micro" }` |
| `output` | Prints a value after apply | To surface IPs, DNS names, IDs | `output "alb_dns" { value = aws_lb.web.dns_name }` |
| `data` | Reads existing AWS information | To reference things Terraform did not create | `data "aws_ami" "ubuntu" { ... }` |
| `terraform` | Configures Terraform itself | To set backend, required providers, version | `terraform { required_version = ">= 1.0" }` |
| `locals` | Defines reusable values within a config | To avoid repeating the same expression | `locals { name_prefix = "my-app-${var.env}" }` |

---

## Problems I Ran Into

### ❌ Problem 1: Instances Showing as Unhealthy in Target Group

After deployment the ALB was returning `503 Service Unavailable`.

**What happened:** The instance security group was allowing traffic from `0.0.0.0/0` instead of specifically from the ALB security group. The health checks were timing out because Apache had not finished installing by the time the ALB ran its first check.

**Fix:** Two things:

First, I tightened the instance security group to only allow traffic from the ALB security group:

```hcl
ingress {
  from_port       = var.server_port
  to_port         = var.server_port
  protocol        = "tcp"
  security_groups = [aws_security_group.alb_sg.id]  # ← only from ALB
}
```

Second, I gave Apache more time to start by increasing the health check interval and raising the `unhealthy_threshold`:

```hcl
health_check {
  interval            = 15
  timeout             = 3
  healthy_threshold   = 2
  unhealthy_threshold = 2
}
```

After about 2 minutes, instances showed as healthy and the ALB started routing traffic correctly.

---

### ❌ Problem 2: State File Out of Sync After Failed Apply

Midway through a `terraform apply`, my internet dropped. When it came back, running `apply` again gave errors about resources already existing.

**What happened:** Some resources were created in AWS before the connection dropped. The state file was partially written — some resources were recorded, others were not.

**Fix:**

```bash
terraform refresh
```

This command re-reads the real state of AWS and updates the state file to match. After running it, `terraform plan` showed a clean diff and `apply` completed successfully.

> 💡 `terraform refresh` is your recovery tool when you suspect the state file and AWS have drifted apart.

---

## What I Learned Today

- The **state file** is Terraform's memory — it records everything that was created and is the source of truth for every `plan` and `apply`
- **State drift** happens when someone changes infrastructure outside of Terraform — Terraform will detect it and try to correct it on the next `apply`
- **Never edit the state file by hand** — Terraform is the only thing that should write to it
- **Never commit the state file to Git** — it contains sensitive infrastructure details
- **State locking** prevents two people from running `apply` at the same time and corrupting the state
- The **instance security group** should only allow traffic from the ALB security group — not from the open internet

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
