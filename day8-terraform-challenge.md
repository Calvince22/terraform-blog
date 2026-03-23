# Building Reusable Infrastructure with Terraform Modules

*Day 8 of the #30DayTerraformChallenge*

---

## Hello

For the past week I have been writing the same security group structure, the same load balancer setup, and the same Auto Scaling Group configuration — just with slightly different values each time.

Today I stopped doing that.

**Terraform Modules** are how you package a chunk of infrastructure into a reusable component. Write it once, call it from anywhere, pass in different inputs for different environments. No copy-pasting. No drift between environments. No hunting for which file has the "right" version.

This is how professional infrastructure teams work — and today I built my first real module.

---

## What Is a Module?

A module is just a folder containing Terraform files. That is it.

Every Terraform project you have written so far is technically a module — it is called the **root module**. When you create a subfolder with its own `.tf` files and reference it from your root configuration, that subfolder becomes a **child module**.

The pattern looks like this:

```
Your root config (main.tf)
        |
        └── calls module "webserver-cluster"
                    |
                    ├── creates Security Group
                    ├── creates Launch Template
                    ├── creates Auto Scaling Group
                    ├── creates Load Balancer
                    └── returns alb_dns_name as output
```

The root config does not care how the Load Balancer is built. It just says "give me a webserver cluster with these settings" — and the module handles the details.

---

## The Directory Structure

I organised my project like this:

```
project/
├── modules/
│   └── services/
│       └── webserver-cluster/
│           ├── main.tf        ← all the resources
│           ├── variables.tf   ← inputs the caller provides
│           ├── outputs.tf     ← values exposed back to the caller
│           └── README.md      ← how to use this module
│
└── live/
    ├── dev/
    │   └── services/
    │       └── webserver-cluster/
    │           └── main.tf    ← calls the module with dev settings
    └── production/
        └── services/
            └── webserver-cluster/
                └── main.tf    ← calls the module with production settings
```

The `modules/` folder contains the reusable component. The `live/` folder contains the calling configurations — one per environment. The module code never changes between environments. Only the inputs change.

---

## Building the Module

### variables.tf — What the Caller Must Provide

Every value that might differ between environments goes in `variables.tf`. Nothing is hardcoded inside the module.

```hcl
variable "cluster_name" {
  description = "The name to use for all cluster resources"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type for the cluster"
  type        = string
  default     = "t3.micro"
}

variable "min_size" {
  description = "Minimum number of EC2 instances in the ASG"
  type        = number
}

variable "max_size" {
  description = "Maximum number of EC2 instances in the ASG"
  type        = number
}

variable "server_port" {
  description = "Port the server uses for HTTP requests"
  type        = number
  default     = 80
}

variable "server_message" {
  description = "Message displayed on the web page"
  type        = string
  default     = "Hello from Terraform!"
}
```

Notice that `min_size` and `max_size` have no `default`. This means they are **required** — the caller must provide them. If they do not, Terraform will refuse to run.

`instance_type`, `server_port`, and `server_message` all have defaults — they are **optional**. The caller can override them or leave them as-is.

This is an important design decision: required variables force the caller to make a deliberate choice. Optional variables with sensible defaults make the module easier to use.

---

### main.tf — All the Resources

This is the heart of the module. Every resource is defined here. No hardcoded names — everything uses `var.` references:

```hcl
# Fetch the latest Ubuntu 22.04 AMI automatically
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  owners = ["099720109477"]
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

# Security group for EC2 instances
resource "aws_security_group" "instance_sg" {
  name        = "${var.cluster_name}-instance-sg"
  description = "Allow HTTP traffic to instances"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port       = var.server_port
    to_port         = var.server_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security group for the ALB
resource "aws_security_group" "alb_sg" {
  name        = "${var.cluster_name}-alb-sg"
  description = "Allow HTTP traffic to ALB"
  vpc_id      = data.aws_vpc.default.id

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

# Launch Template
resource "aws_launch_template" "web" {
  name_prefix   = "${var.cluster_name}-"
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
              echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
              EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "${var.cluster_name}-instance"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "web" {
  min_size            = var.min_size
  max_size            = var.max_size
  vpc_zone_identifier = data.aws_subnets.default.ids

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.web.arn]
  health_check_type = "ELB"

  tag {
    key                 = "Name"
    value               = "${var.cluster_name}-asg"
    propagate_at_launch = true
  }
}

# Application Load Balancer
resource "aws_lb" "web" {
  name               = "${var.cluster_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = data.aws_subnets.default.ids
}

# Target Group
resource "aws_lb_target_group" "web" {
  name     = "${var.cluster_name}-tg"
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

# Listener
resource "aws_lb_listener" "web" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}
```

Every resource name uses `var.cluster_name` as a prefix — `"${var.cluster_name}-alb"`, `"${var.cluster_name}-tg"`, and so on. This means when you call the module twice with different `cluster_name` values, all the resources get unique names and do not conflict with each other.

---

### outputs.tf — What the Module Exposes Back

```hcl
output "alb_dns_name" {
  value       = aws_lb.web.dns_name
  description = "The DNS name of the load balancer — paste this in your browser"
}

output "asg_name" {
  value       = aws_autoscaling_group.web.name
  description = "The name of the Auto Scaling Group"
}

output "alb_security_group_id" {
  value       = aws_security_group.alb_sg.id
  description = "The ID of the ALB security group"
}
```

Outputs are what the caller gets back from the module. Think of them like return values from a function.

I exposed three things:
- **`alb_dns_name`** — the URL the caller needs to access the cluster
- **`asg_name`** — useful if the caller wants to attach scaling policies
- **`alb_security_group_id`** — useful if another module needs to allow traffic from this ALB

I did not expose things like the Launch Template ID or individual instance IDs — those are internal implementation details the caller does not need.

---

## Calling the Module — Dev and Production

Now in the `live/` directory, I create a simple `main.tf` for each environment that calls the module with different inputs:

**live/dev/services/webserver-cluster/main.tf**

```hcl
provider "aws" {
  region = "us-east-1"
}

module "webserver_cluster" {
  source = "../../../../modules/services/webserver-cluster"

  cluster_name   = "webservers-dev"
  instance_type  = "t3.micro"
  min_size       = 2
  max_size       = 4
  server_message = "Hello from Dev!"
}

output "alb_dns_name" {
  value       = module.webserver_cluster.alb_dns_name
  description = "The DNS name of the dev load balancer"
}
```

**live/production/services/webserver-cluster/main.tf**

```hcl
provider "aws" {
  region = "us-east-1"
}

module "webserver_cluster" {
  source = "../../../../modules/services/webserver-cluster"

  cluster_name   = "webservers-production"
  instance_type  = "t3.medium"
  min_size       = 4
  max_size       = 10
  server_message = "Hello from Production!"
}

output "alb_dns_name" {
  value       = module.webserver_cluster.alb_dns_name
  description = "The DNS name of the production load balancer"
}
```

Same module. Different inputs. Zero duplicated resource code.

Dev gets `t3.micro` instances with a minimum of 2. Production gets `t3.medium` instances with a minimum of 4. If I need to change how the Load Balancer works, I change it once in the module and both environments pick up the change on the next apply.

---

## Deploying the Module

```bash
cd live/dev/services/webserver-cluster
terraform init
terraform apply
```

`terraform init` here does something new — it downloads the module source. You will see:

```
Initializing modules...
- webserver_cluster in ../../../../modules/services/webserver-cluster

Initializing the backend...
Initializing provider plugins...

Terraform has been successfully initialized!
```

After apply:

```
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:
alb_dns_name = "webservers-dev-alb-123456789.us-east-1.elb.amazonaws.com"
```

Paste the DNS name into your browser and you will see:

```
Hello from Dev!
Instance ID: i-0abc123def456789
```

---

## Clean Up

```bash
terraform destroy
```

> Always destroy after confirming the deployment works. The ALB and ASG instances cost money while running.

---

## What I Decided to Keep Inside the Module vs Expose as Variables

This was the most interesting design challenge of the day.

**Exposed as variables (caller decides):**
- `cluster_name` — must be unique per environment, so the caller controls it
- `instance_type` — different environments need different sizes
- `min_size` and `max_size` — production needs more capacity than dev
- `server_message` — useful for showing which environment you are hitting
- `server_port` — kept configurable in case the default port needs to change

**Kept internal to the module (caller does not need to know):**
- The AMI ID — fetched dynamically inside the module, the caller never needs to think about it
- Security group rules — the module owns its own networking logic
- Health check settings — sensible defaults that work for most use cases
- The exact S3 key or listener protocol — implementation details

The rule I followed: if a value changes between environments or between teams, expose it as a variable. If it is the same everywhere and changing it would require understanding the module internals, keep it hidden.

---

## Problems I Ran Into

### ❌ Problem 1: Wrong Relative Path in `source`

When I first called the module, I got this error:

```
Error: Module not installed

  on main.tf line 3, in module "webserver_cluster":
   3:   source = "../../../../modules/services/webserver-cluster"

This module is not yet installed. Run "terraform init" to install
all modules required by this configuration.
```

**What happened:** Two things — first I had the wrong number of `../` in the path, and second I had not run `terraform init` yet after adding the module source.

**Fix:** I counted the directory levels carefully:

```
live/dev/services/webserver-cluster/main.tf
  ↑    ↑    ↑         ↑
  1    2    3         4  levels up to reach project root
```

So the correct path is `../../../../modules/services/webserver-cluster`. Then I ran `terraform init` and it downloaded the module correctly.

> 💡 Always run `terraform init` after adding or changing a `source` path. Terraform needs to register the module before it can use it.

---

### ❌ Problem 2: Duplicate Resource Names When Calling the Module Twice

When I tried deploying both dev and production from the same directory (as a test), I got:

```
Error: creating Security Group: InvalidGroup.Duplicate:
The security group 'webservers-dev-alb-sg' already exists
```

**What happened:** I accidentally used the same `cluster_name` for both calls. Since every resource in the module is named using `var.cluster_name`, both calls tried to create security groups with the same name.

**Fix:** Made sure each module call has a unique `cluster_name`:

```hcl
# First call
module "dev_cluster" {
  cluster_name = "webservers-dev"      # unique
  ...
}

# Second call
module "prod_cluster" {
  cluster_name = "webservers-production"  # unique
  ...
}
```

This is exactly why `cluster_name` is a required variable with no default — forcing the caller to choose a unique name prevents this conflict.

---

### ❌ Problem 3: Missing Required Variable

When I first called the module without providing `min_size`, Terraform refused to run:

```
Error: No value for required variable

  on main.tf line 3, in module "webserver_cluster":
   3: module "webserver_cluster" {

The root module variable "min_size" is not set, and has no default value.
Use a -var or -var-file command line argument to provide a value for
this variable.
```

**What happened:** `min_size` and `max_size` have no default values — they are required. I forgot to include them in the module call.

**Fix:** Added the missing variables to the module block:

```hcl
module "webserver_cluster" {
  source        = "../../../../modules/services/webserver-cluster"
  cluster_name  = "webservers-dev"
  instance_type = "t3.micro"
  min_size      = 2   # ← was missing
  max_size      = 4   # ← was missing
}
```

> 💡 Required variables with no defaults are a deliberate design choice. They force the caller to make a conscious decision instead of accidentally using a wrong default.

---

## What I Learned Today

- **A module is just a folder** — the abstraction is in how you structure and call it, not in any special syntax
- **Root module vs child module** — your calling configuration is the root module, the folder you reference is the child module
- **`terraform init` downloads modules** — any time you add or change a `source`, you must re-run init
- **Required variables force deliberate choices** — no default means the caller must provide a value
- **`cluster_name` as a prefix** prevents resource name conflicts when calling the same module multiple times
- **Outputs are the module's public interface** — only expose what callers actually need, keep implementation details internal
- **Module outputs in state** are stored under `module.<name>.<output_name>` — you can inspect them with `terraform state show`

---

## Root Module vs Child Module — Quick Summary

| | Root Module | Child Module |
|---|---|---|
| What it is | Your main calling configuration | The reusable folder being called |
| Where it lives | `live/dev/main.tf` | `modules/services/webserver-cluster/` |
| How it runs | `terraform apply` from its directory | Called via `module` block in root |
| Has its own state | Yes | No — shares the root state file |
| Can call other modules | Yes | Yes |

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*
---
