# Mastering Zero-Downtime Deployments with Terraform

*Day 12 of the #30DayTerraformChallenge*

---

## Hello

Today was one of the most practical days of the entire challenge.

Deploying infrastructure updates without taking your application offline is one of the hardest problems in operations. Most tutorials skip it entirely. Today I tackled it head on — and by the end I had live proof that a Terraform deployment can update a running application with zero service interruption.

Here is exactly how it works, why the default behaviour causes downtime, and two strategies to fix it.

---

## The Problem — Why Default Terraform Causes Downtime

When you update something that cannot be modified in-place — like a Launch Template — Terraform's default behaviour is:

1. **Destroy** the old resource
2. **Create** the new resource

For an Auto Scaling Group, this plays out like this:

```
Step 1: Old ASG destroyed → all instances terminated → APP IS DOWN ❌
Step 2: New ASG created → new instances boot up → app comes back ✅
```

The gap between Step 1 and Step 2 is your downtime window. It can be anywhere from 30 seconds to several minutes — long enough to cause real user impact in production.

The fix is a single lifecycle rule that **reverses this order**.

---

## The Fix — `create_before_destroy`

The `lifecycle` block controls how Terraform manages resource replacement. Adding `create_before_destroy = true` tells Terraform:

> "Before you destroy the old version, create the new one first."

```
Step 1: New ASG created → new instances boot → health checks pass ✅
Step 2: Old ASG destroyed → traffic has already shifted to new instances ✅
```

No downtime window. The new version is serving traffic before the old one is gone.

---

## Implementing `create_before_destroy`

### Launch Configuration with Lifecycle

```hcl
resource "aws_launch_configuration" "web" {
  image_id        = data.aws_ami.ubuntu.id
  instance_type   = var.instance_type
  security_groups = [aws_security_group.instance_sg.id]

  user_data = templatefile("${path.module}/user-data.sh", {
    server_port = var.server_port
    server_text = var.server_text
  })

  lifecycle {
    create_before_destroy = true  # ← the key line
  }
}
```

### The ASG Naming Problem

Here is a subtle but critical issue: when `create_before_destroy = true`, the new ASG must exist **alongside** the old one before the old is destroyed.

AWS does not allow two ASGs with the same name to exist at the same time.

If your ASG has a fixed name like `"webservers-dev"`, the apply will fail:

```
Error: creating Auto Scaling Group: AlreadyExists:
AutoScalingGroup by this name already exists
```

**The fix — use `name_prefix` instead of `name`:**

```hcl
resource "aws_autoscaling_group" "web" {
  name_prefix         = "${var.cluster_name}-"  # ← AWS generates a unique name
  launch_configuration = aws_launch_configuration.web.name
  vpc_zone_identifier  = data.aws_subnets.default.ids

  target_group_arns = [aws_lb_target_group.web.arn]
  health_check_type = "ELB"

  min_size = var.min_size
  max_size = var.max_size

  lifecycle {
    create_before_destroy = true  # ← must be on ASG too
  }

  tag {
    key                 = "Name"
    value               = var.cluster_name
    propagate_at_launch = true
  }
}
```

With `name_prefix`, AWS appends a unique suffix automatically — for example `webservers-dev-20260320094106`. The new ASG gets a different suffix from the old one, so they can coexist during the transition.

**Important:** The `lifecycle` block must be on **both** the Launch Configuration and the ASG. If it is only on one, the dependency chain breaks and you still get downtime.

### user-data.sh

```bash
#!/bin/bash
apt-get update -y
apt-get install -y apache2
systemctl start apache2
systemctl enable apache2
echo "<h1>${server_text}</h1>" > /var/www/html/index.html
echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
```

### variables.tf

```hcl
variable "server_text" {
  description = "The text displayed on the web page — change this to trigger a rolling update"
  type        = string
  default     = "Hello World v1"
}
```

---

## Proving Zero-Downtime Works

### Step 1 — Deploy Version 1

```bash
terraform apply
```

Confirm the ALB is serving traffic:

```bash
curl http://<your-alb-dns-name>
# <h1>Hello World v1</h1>
```

### Step 2 — Start a Traffic Loop

In a second terminal, run a loop that hits the ALB every 2 seconds:

```bash
while true; do
  curl -s http://<your-alb-dns-name>
  sleep 2
done
```

Leave this running. Watch it carefully during the next step.

### Step 3 — Deploy Version 2

Change `server_text` to `"Hello World v2"` in your variables and run:

```bash
terraform apply
```

### Step 4 — What You Should See

In the traffic loop terminal, responses continue uninterrupted throughout the entire apply. At some point they switch from v1 to v2:

```
<h1>Hello World v1</h1>
<h1>Hello World v1</h1>
<h1>Hello World v1</h1>
<h1>Hello World v1</h1>
<h1>Hello World v2</h1>   ← version switched here
<h1>Hello World v2</h1>
<h1>Hello World v2</h1>
```

No errors. No timeouts. No connection refused. Just a clean transition from v1 to v2 while the application was live.

This is what zero-downtime deployment looks like.

---

## Going Further — Blue/Green Deployment

`create_before_destroy` handles rolling updates. But there is a more powerful strategy called **blue/green deployment**.

### What Is Blue/Green?

Instead of updating instances in place, you maintain two complete, separate environments:

- **Blue** — the current live environment serving production traffic
- **Green** — the new version, fully deployed and tested but not yet live

When you are ready to switch, you shift all traffic from blue to green at the Load Balancer level. The switch is instantaneous — a single API call. If anything goes wrong, you switch back to blue just as quickly.

```
Before switch:           After switch:
User → ALB → Blue ✅    User → ALB → Green ✅
              Green 🔵                 Blue 🔵 (idle, ready for rollback)
```

### The Terraform Configuration

```hcl
variable "active_environment" {
  description = "Which environment is currently active: blue or green"
  type        = string
  default     = "blue"

  validation {
    condition     = contains(["blue", "green"], var.active_environment)
    error_message = "active_environment must be blue or green."
  }
}

# Blue target group — running v1
resource "aws_lb_target_group" "blue" {
  name     = "${var.cluster_name}-blue-tg"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  health_check {
    path                = "/"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

# Green target group — running v2
resource "aws_lb_target_group" "green" {
  name     = "${var.cluster_name}-green-tg"
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  health_check {
    path                = "/"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

# Listener rule — controls which target group gets traffic
resource "aws_lb_listener_rule" "blue_green" {
  listener_arn = aws_lb_listener.web.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = var.active_environment == "blue" ? aws_lb_target_group.blue.arn : aws_lb_target_group.green.arn
  }

  condition {
    path_pattern {
      values = ["/*"]
    }
  }
}
```

### Switching from Blue to Green

To shift all traffic to the green environment:

1. Change `active_environment = "green"` in your variables
2. Run `terraform apply`

The listener rule update is a single AWS API call. `terraform apply` for just this change completes in under 10 seconds — with no visible interruption to the traffic loop.

```
<h1>Hello World v1</h1>   ← blue serving traffic
<h1>Hello World v1</h1>
<h1>Hello World v2</h1>   ← green now serving traffic
<h1>Hello World v2</h1>
```

To roll back instantly — just change back to `active_environment = "blue"` and apply again.

---

## Limitations of `create_before_destroy`

The book is honest about what this approach does not solve:

**1. Health check delay**
New instances must pass health checks before the old ASG is destroyed. If your application takes a long time to start up, the deployment takes longer — and if health checks never pass, the apply hangs.

**2. Database migrations**
If your v2 code requires a database schema change, you cannot run the migration and switch traffic atomically. You need to handle backwards compatibility separately.

**3. ASG replacement is still slow**
`create_before_destroy` avoids downtime but the full deployment — boot instances, install software, pass health checks — still takes several minutes. It is not instant like a blue/green switch.

**How blue/green addresses these:**
- Traffic switch is atomic and instant — no waiting for instances to boot
- You can fully test green before switching — health checks, smoke tests, whatever you need
- Rollback is one variable change and one apply

**Blue/green tradeoffs:**
- Costs more — you are running double the infrastructure during a deployment
- More complex to set up and manage
- Database compatibility still requires careful handling

---

## Problems I Ran Into

### ❌ Problem 1: ASG Name Conflict

```
Error: creating Auto Scaling Group (webservers-dev):
AlreadyExists: AutoScalingGroup by this name already exists
- {"autoScalingGroupName":"webservers-dev"}
```

**What happened:** I had `name = var.cluster_name` on the ASG instead of `name_prefix`. When `create_before_destroy` tried to create the new ASG alongside the old one, AWS rejected it because the name was already taken.

**Fix:** Changed from `name` to `name_prefix`:

```hcl
# Before
name = var.cluster_name

# After
name_prefix = "${var.cluster_name}-"
```

---

### ❌ Problem 2: lifecycle Block Only on Launch Configuration

I added `create_before_destroy` to the Launch Configuration but forgot to add it to the ASG. The apply still caused downtime.

**What happened:** Terraform creates a dependency chain — the ASG depends on the Launch Configuration. When the Launch Configuration is replaced, Terraform needs to replace the ASG too. But if the ASG does not have `create_before_destroy`, it still uses the default destroy-then-create order.

**Fix:** Add `lifecycle { create_before_destroy = true }` to **both** resources:

```hcl
resource "aws_launch_configuration" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```
---

## What I Learned Today

- **Default Terraform destroys first** — which causes a downtime window for resources that cannot be updated in-place
- **`create_before_destroy`** reverses the order — new resource is created and passing health checks before the old one is destroyed
- **`name_prefix` is required** on the ASG when using `create_before_destroy` — two resources with the same name cannot exist simultaneously in AWS
- **Both the Launch Configuration and ASG need `lifecycle` blocks** — the dependency chain means both must participate
- **Blue/green is atomic** — traffic shifts in a single API call, rollback is instant
- **`create_before_destroy` does not solve everything** — database migrations and slow startup times still require careful handling
- **Health check tuning matters** — slow-starting applications need more generous health check settings to avoid failed deployments

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
