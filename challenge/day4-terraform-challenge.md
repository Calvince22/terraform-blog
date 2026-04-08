# Deploying a Highly Available Web App on AWS Using Terraform

*Day 4 of the #30DayTerraformChallenge*

---

Yesterday I deployed one server. It worked great.

But here is the problem with one server — if it crashes, your entire website goes down. If traffic suddenly spikes, it gets overwhelmed. One server is fine for learning, but it is not how real production systems work.

Today I fixed that. I went from one hardcoded server to a **flexible, clustered, load-balanced setup** that can handle real traffic and survive individual server failures.

Here is everything I did, in plain English — with the actual code I used.

---

Today has two parts:

**Part 1 — Make the server configurable**
Stop hardcoding values like instance type and port number. Use variables instead so the code is flexible and reusable.

**Part 2 — Deploy a cluster**
Instead of one server, deploy multiple servers behind a load balancer that spreads traffic evenly across all of them.

Here is what the final setup looks like:

```
Your Browser
     |
     | (HTTP on Port 80)
     |
[Application Load Balancer]    ← spreads traffic across all servers
     |          |          |
  [Server 1] [Server 2] [Server 3]  ← Auto Scaling Group manages these
  [us-east-1a] [us-east-1b] [us-east-1c]  ← spread across zones
```

If Server 1 crashes, the Load Balancer automatically sends all traffic to Server 2 and 3. If traffic doubles, the Auto Scaling Group spins up more servers. This is called **High Availability**.

---

## Part 1 — Making the Server Configurable with Variables

### The Problem with Hardcoded Values

On Day 3, my code looked like this:

```hcl
instance_type = "t2.micro"
region        = "us-east-1"
```

Those values are hardcoded — baked directly into the code. If I want to change the region or instance size, I have to hunt through every file and update it manually. In a team of engineers, this becomes a nightmare fast.

This is the problem that **input variables** solve.

### The DRY Principle

DRY stands for **Don't Repeat Yourself**.

Define a value once in one place. Reference it everywhere. If you need to change it, you change it once and it updates everywhere automatically.

In Terraform, `variables.tf` is where DRY lives.

### My variables.tf

Here is every variable I defined for today's deployment:

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
  description = "Name tag for the web server"
  type        = string
  default     = "terraform-web-server-day4"
}

variable "asg_min_size" {
  description = "Minimum number of instances in the Auto Scaling Group"
  type        = number
  default     = 2
}

variable "asg_max_size" {
  description = "Maximum number of instances in the Auto Scaling Group"
  type        = number
  default     = 5
}

variable "asg_desired_capacity" {
  description = "Desired number of instances in the Auto Scaling Group"
  type        = number
  default     = 2
}

variable "server_message" {
  description = "Message served by the web server"
  type        = string
  default     = "Hello from Terraform!"
}

variable "alb_listener_port" {
  description = "The port the ALB listens on"
  type        = number
  default     = 80
}
```

Here is what each variable controls:

| Variable | Type | What It Controls |
|---|---|---|
| `server_port` | number | The port each EC2 instance serves traffic on |
| `instance_type` | string | The size of each EC2 server |
| `aws_region` | string | Which AWS region everything is deployed in |
| `server_name` | string | A name prefix used across all resources |
| `asg_min_size` | number | Fewest servers allowed in the cluster |
| `asg_max_size` | number | Most servers allowed in the cluster |
| `asg_desired_capacity` | number | How many servers to start with |
| `server_message` | string | The text shown on the web page |
| `alb_listener_port` | number | The port the Load Balancer listens on |

Notice that `server_name` is used as a prefix for naming every resource — security groups, load balancer, target group. This keeps everything consistently named and easy to find in the AWS Console.

---

## Part 2 — Deploying the Cluster

### New Concepts Before the Code

**Auto Scaling Group (ASG)**
Automatically manages a group of servers. You set minimum and maximum sizes, and it handles the rest — launching servers when traffic rises, terminating them when it drops, and replacing any that crash.

**Application Load Balancer (ALB)**
Sits in front of all your servers and directs incoming traffic. Spreads requests evenly so no single server gets overwhelmed.

**Data Sources**
Let Terraform *read* existing information from AWS rather than creating something new. Today I used three:

- Fetch the current AWS region automatically
- Fetch available Availability Zones dynamically
- Fetch the latest Ubuntu 22.04 AMI — so I never have to hardcode an AMI ID

---

### The Full main.tf — With Explanations

```hcl
# Region comes from a variable — not hardcoded
provider "aws" {
  region = var.aws_region
}

# Reads which region we are in
data "aws_region" "current" {}

# Fetches available Availability Zones in the region
# Filtered to specific zones that have full service support
data "aws_availability_zones" "supported" {
  state = "available"

  filter {
    name   = "zone-name"
    values = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
  }
}

# Fetches subnets that sit in our supported availability zones
data "aws_subnets" "filtered" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }

  filter {
    name   = "availability-zone"
    values = data.aws_availability_zones.supported.names
  }
}

# Fetches the latest Ubuntu 22.04 AMI automatically
# No hardcoded AMI ID — this always picks the most recent one
data "aws_ami" "ubuntu_22_04" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  owners = ["099720109477"]  # Canonical's official AWS account ID
}

# Security group for EC2 instances in the ASG
resource "aws_security_group" "instance_sg" {
  name        = "${var.server_name}-instance-sg"
  description = "Allow HTTP traffic to instances"

  ingress {
    from_port   = var.server_port
    to_port     = var.server_port
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

# Separate security group for the ALB
resource "aws_security_group" "alb_sg" {
  name        = "${var.server_name}-alb-sg"
  description = "Allow HTTP traffic to ALB"

  ingress {
    from_port   = var.alb_listener_port
    to_port     = var.alb_listener_port
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

# Launch Template — the blueprint for every server in the cluster
resource "aws_launch_template" "web_server" {
  name_prefix   = "${var.server_name}-"
  image_id      = data.aws_ami.ubuntu_22_04.id  # ← fetched dynamically
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.instance_sg.id]

  # This runs on every server when it first boots
  # Installs Apache and writes the HTML page
  user_data = base64encode(<<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y apache2
              systemctl restart apache2
              systemctl enable apache2
              echo "<h1>${var.server_message}</h1>" > /var/www/html/index.html
              EOF
  )

  tags = {
    Name = var.server_name
  }
}

# Auto Scaling Group — manages the cluster
resource "aws_autoscaling_group" "web_asg" {
  desired_capacity    = var.asg_desired_capacity  # start with 2
  min_size            = var.asg_min_size           # never go below 2
  max_size            = var.asg_max_size           # never exceed 5
  vpc_zone_identifier = data.aws_subnets.filtered.ids  # spread across zones

  launch_template {
    id      = aws_launch_template.web_server.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.web_tg.arn]
  health_check_type = "ELB"  # ALB decides if instances are healthy

  tag {
    key                 = "Name"
    value               = "${var.server_name}-asg"
    propagate_at_launch = true
  }
}

# Application Load Balancer
resource "aws_lb" "web_alb" {
  name               = "${var.server_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = data.aws_subnets.default.ids
}

# Target Group — the list of servers the ALB sends traffic to
resource "aws_lb_target_group" "web_tg" {
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

# Listener — watches port 80 on the ALB and forwards requests
resource "aws_lb_listener" "web_listener" {
  load_balancer_arn = aws_lb.web_alb.arn
  port              = var.alb_listener_port
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web_tg.arn
  }
}

# Fetch the default VPC
data "aws_vpc" "default" {
  default = true
}

# Fetch the default subnets
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Print the ALB DNS name when done
output "alb_dns_name" {
  value       = aws_lb.web_alb.dns_name
  description = "The DNS name of the load balancer"
}
```

---

### How All the Pieces Connect

Here is a plain English walkthrough of how everything works together:

1. **Data sources** fetch the Ubuntu AMI, available zones, and subnets automatically — nothing hardcoded
2. **Launch Template** defines the blueprint — Ubuntu 22.04, Apache web server, HTML page using `var.server_message`
3. **Auto Scaling Group** launches 2 servers to start, spread across availability zones, and scales between 2 and 5 based on demand
4. **Target Group** keeps a list of healthy servers the ALB can send traffic to, confirmed via health checks
5. **Listener** watches port 80 on the ALB and forwards every request to the Target Group
6. **Load Balancer** receives all user traffic and spreads it evenly across healthy instances

---

### Running the Deployment

```bash
terraform init
terraform plan
terraform apply
```

After `apply` completes:

```
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:
alb_dns_name = "terraform-web-server-day4-alb-1234567890.us-east-1.elb.amazonaws.com"
```

Paste the DNS name into your browser and you will see:

```
Hello from Terraform!
```

That message comes directly from `var.server_message` in your variables file. To change it — update one line, run `terraform apply`, done.

---

### Clean Up When Done

```bash
terraform destroy
```

> ⚠️ The ALB, ASG instances, and related resources cost money while running. Always destroy at the end of each session.

---

## Problems I Ran Into

These are real errors with the exact messages I saw — so if you see the same thing, you know exactly what to do.

---

### ❌ Problem 1: Duplicate Security Groups

```
Error: creating Security Group (terraform-instance-sg): operation error EC2:
CreateSecurityGroup, https response error StatusCode: 400,
RequestID: 70bc822f-024a-43de-9484-cc7382bf2d53,
api error InvalidGroup.Duplicate: The security group 'terraform-instance-sg'
already exists for VPC 'vpc-0c7fbd6b8ddf09055'

Error: creating Security Group (terraform-alb-sg): operation error EC2:
CreateSecurityGroup, https response error StatusCode: 400,
RequestID: 39767468-ace4-497a-854e-e1626c48bad9,
api error InvalidGroup.Duplicate: The security group 'terraform-alb-sg'
already exists for VPC 'vpc-0c7fbd6b8ddf09055'
```

**What happened:** A previous `terraform apply` was interrupted before finishing. The security groups were created in AWS but Terraform never recorded them in its state file. When I ran `apply` again, Terraform tried to create them from scratch — but they already existed in AWS.

This mismatch between what is in AWS and what Terraform knows about is called **state drift**. It is one of the most common beginner errors.

**Fix — Option 1: Delete manually in the AWS Console**

1. Go to **EC2** → **Security Groups**
2. Find `terraform-instance-sg` and `terraform-alb-sg`
3. Delete them both
4. Run `terraform apply` again

**Fix — Option 2: Import into Terraform state**

```bash
terraform import aws_security_group.instance_sg sg-xxxxxxxxxxxxxxxxx
terraform import aws_security_group.alb_sg sg-xxxxxxxxxxxxxxxxx
```

Replace `sg-xxxxxxxxxxxxxxxxx` with the actual IDs from the AWS Console. This tells Terraform to adopt the existing resources instead of trying to create them.

I used Option 1 since this was a learning environment. In a real team, Option 2 is safer because nothing gets deleted.

> 💡 **Root cause:** I forgot `terraform destroy` at the end of a previous session. That one habit prevents this entire class of error.

---

### ❌ Problem 2: Duplicate Target Group

```
Error: ELBv2 Target Group (terraform-web-tg) already exists

  with aws_lb_target_group.web_tg,
  on main.tf line 105, in resource "aws_lb_target_group" "web_tg":
 105: resource "aws_lb_target_group" "web_tg" {
```

**What happened:** Same root cause — a Target Group from a previous interrupted run was still sitting in AWS.

**Fix:**

1. Go to **EC2** → **Target Groups**
2. Find `terraform-web-tg` and delete it
3. Run `terraform apply` again

After cleaning up all three orphaned resources, `terraform apply` completed cleanly.

---

### ❌ Lesson: How to Prevent This Entirely

**Always destroy at the end of every session:**
```bash
terraform destroy
```

**Check what Terraform is currently tracking:**
```bash
terraform state list
```

If you see resources in AWS that are missing from this list, that is state drift — and duplicate errors are coming on your next `apply`.

**Use `name_prefix` for resources you recreate often:**
```hcl
# Fixed name = duplicate error if it already exists
name = "terraform-instance-sg"

# name_prefix = unique suffix generated each time, no conflicts
name_prefix = "terraform-instance-"
```

---

## What I Learned Today

- **Variables make infrastructure reusable** — change a value once and it updates everywhere
- **The DRY principle** prevents inconsistency, especially in teams
- **Data sources are powerful** — fetching the latest Ubuntu AMI dynamically means I never manually update an AMI ID again
- **Two security groups are better than one** — separating ALB and instance security groups gives precise traffic control
- **`health_check_type = "ELB"`** means the ASG trusts the Load Balancer's view of instance health — not just whether the EC2 instance is powered on
- **State drift causes duplicate errors** — `terraform destroy` after every session prevents it entirely

---

## Single Server vs Cluster — What Changed?

| | Day 3 (Single Server) | Day 4 (Cluster) |
|---|---|---|
| Number of servers | 1 | 2–5 (auto-managed) |
| If a server crashes | Website goes down | Other servers keep running |
| Traffic handling | One server handles everything | Load Balancer shares the load |
| Scaling | Manual | Automatic |
| OS image | Hardcoded AMI ID | Fetched dynamically |
| Web server | Python HTTP server | Apache (production-grade) |
| Good for | Learning | Real production workloads |

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
