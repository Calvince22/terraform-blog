# How Conditionals Make Terraform Infrastructure Dynamic and Efficient

*Day 11 of the #30DayTerraformChallenge*

---

## Hello


Conditionals are what allow one Terraform configuration to behave completely differently across environments — without copying code, without maintaining separate files for dev and production, without any duplication at all.

By the end of today my webserver cluster module responds intelligently to a single `environment` variable. Pass in `"dev"` and you get small instances, minimal monitoring, and a lean setup. Pass in `"production"` and you get larger instances, more servers, CloudWatch alarms, and stricter deletion protection.

Same code. Different behaviour. That is the power of conditionals done right.

---

## The Core Pattern — The Ternary Expression

The Terraform conditional uses the ternary operator:

```
condition ? value_if_true : value_if_false
```

It works anywhere Terraform accepts an expression — inside resource arguments, locals, outputs, and data sources.

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

# Simple example
instance_type = var.environment == "production" ? "t3.medium" : "t3.micro"
```

If `environment` is `"production"` → `t3.medium`. Otherwise → `t3.micro`.

Simple. But scattered across multiple resource arguments, this gets messy fast. The right pattern is to centralise all conditional logic in `locals`.

---

## Pattern 1 — Centralise Everything in locals

Instead of putting ternary operators inside every resource, make all the decisions once in a `locals` block and reference those from resources:

```hcl
variable "environment" {
  description = "Deployment environment: dev, staging, or production"
  type        = string
  default     = "dev"
}

locals {
  # One flag that drives everything else
  is_production = var.environment == "production"

  # All conditional decisions in one place
  instance_type     = local.is_production ? "t3.medium" : "t3.micro"
  min_size          = local.is_production ? 3 : 1
  max_size          = local.is_production ? 10 : 3
  enable_monitoring = local.is_production
  deletion_policy   = local.is_production ? "Retain" : "Delete"
}
```

Now your resources are clean and readable:

```hcl
resource "aws_launch_template" "web" {
  instance_type = local.instance_type   # ← reads from locals
}

resource "aws_autoscaling_group" "web" {
  min_size = local.min_size
  max_size = local.max_size
}
```

**Why this is better than scattering ternaries in resource arguments:**

- All conditional logic is in one place — easy to read and update
- Resources stay clean — they just read values, they do not decide them
- If the condition changes (say you add a `"staging"` tier), you update `locals` once — not every resource
- Easier to test — you can see all environment behaviour at a glance

---

## Pattern 2 — Making Entire Resources Optional

The `count = condition ? 1 : 0` pattern makes a resource optional:

- `count = 1` → resource is created
- `count = 0` → resource is skipped entirely

### Example — Optional CloudWatch Alarm

```hcl
variable "enable_detailed_monitoring" {
  description = "Enable CloudWatch detailed monitoring (incurs additional cost)"
  type        = bool
  default     = false
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_detailed_monitoring ? 1 : 0

  alarm_name          = "${var.cluster_name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU utilization exceeded 80%"
}
```

**`terraform plan` with `enable_detailed_monitoring = true`:**

```
  + resource "aws_cloudwatch_metric_alarm" "high_cpu" {
      + alarm_name = "webservers-dev-high-cpu"
      + threshold  = 80
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**`terraform plan` with `enable_detailed_monitoring = false`:**

```
Plan: 0 to add, 0 to change, 0 to destroy.
```

The resource simply does not appear in the plan when the toggle is false.

### Example — Optional Route53 DNS Record

```hcl
variable "create_dns_record" {
  description = "Whether to create a Route53 DNS record for the ALB"
  type        = bool
  default     = false
}

resource "aws_route53_record" "alb" {
  count = var.create_dns_record ? 1 : 0

  zone_id = data.aws_route53_zone.primary.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_lb.web.dns_name
    zone_id                = aws_lb.web.zone_id
    evaluate_target_health = true
  }
}
```

Dev environments typically do not need a real domain name — this toggle lets you skip the DNS record entirely when not needed.

---

## Pattern 3 — Referencing Conditional Resources Safely

This is where beginners get caught. When a resource uses `count = condition ? 1 : 0`, you **cannot** reference it like a normal resource.

**The broken way:**

```hcl
output "alarm_arn" {
  value = aws_cloudwatch_metric_alarm.high_cpu.arn  # ← ERROR when count = 0
}
```

If `count = 0`, the resource does not exist. Terraform throws an error because there is nothing to get the ARN from.

**The correct way:**

```hcl
output "alarm_arn" {
  value = var.enable_detailed_monitoring ? aws_cloudwatch_metric_alarm.high_cpu[0].arn : null
}
```

When `enable_detailed_monitoring` is `true`, the resource exists at index `[0]` and we return its ARN. When it is `false`, we return `null` — a valid empty value that does not cause errors.

**The same pattern for the DNS record:**

```hcl
output "dns_record_fqdn" {
  value = var.create_dns_record ? aws_route53_record.alb[0].fqdn : null
}
```

**Rule:** Any output that references a conditional resource must use a ternary guard. Without it, the output errors when the resource does not exist.

---

## Pattern 4 — The Environment-Aware Module

Now let me put it all together. Here is my webserver cluster module refactored to be fully environment-aware with a single `environment` variable driving everything:

**variables.tf**

```hcl
variable "cluster_name" {
  description = "Name prefix for all resources"
  type        = string
}

variable "environment" {
  description = "Deployment environment: dev, staging, or production"
  type        = string

  # Input validation — catches invalid values at plan time
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, or production."
  }
}
```

**main.tf — locals block**

```hcl
locals {
  is_production = var.environment == "production"

  instance_type     = local.is_production ? "t3.medium" : "t3.micro"
  min_size          = local.is_production ? 3 : 1
  max_size          = local.is_production ? 10 : 3
  enable_monitoring = local.is_production
  deletion_policy   = local.is_production ? "Retain" : "Delete"
}
```

**main.tf — resources**

```hcl
resource "aws_launch_template" "web" {
  name_prefix   = "${var.cluster_name}-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = local.instance_type

  vpc_security_group_ids = [aws_security_group.instance_sg.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y apache2
              systemctl start apache2
              systemctl enable apache2
              echo "<h1>Hello from ${var.environment}!</h1>" > /var/www/html/index.html
              EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.cluster_name}-instance"
      Environment = var.environment
    }
  }
}

resource "aws_autoscaling_group" "web" {
  min_size            = local.min_size
  max_size            = local.max_size
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

# Optional CloudWatch alarm — only in production
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = local.enable_monitoring ? 1 : 0

  alarm_name          = "${var.cluster_name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU exceeded 80% in ${var.environment}"
}
```

**outputs.tf**

```hcl
output "alb_dns_name" {
  value       = aws_lb.web.dns_name
  description = "The DNS name of the load balancer"
}

# Safe reference to conditional resource
output "alarm_arn" {
  value       = local.enable_monitoring ? aws_cloudwatch_metric_alarm.high_cpu[0].arn : null
  description = "ARN of the CPU alarm — null if monitoring is disabled"
}
```

### Calling the Module — Dev vs Production

**Dev:**
```hcl
module "webserver_cluster" {
  source       = "../../../../modules/services/webserver-cluster"
  cluster_name = "webservers-dev"
  environment  = "dev"
}
```

Dev plan output:
```
+ aws_launch_template.web       instance_type = "t3.micro"
+ aws_autoscaling_group.web     min_size = 1, max_size = 3
  (no CloudWatch alarm)

Plan: 7 to add, 0 to change, 0 to destroy.
```

**Production:**
```hcl
module "webserver_cluster" {
  source       = "../../../../modules/services/webserver-cluster"
  cluster_name = "webservers-production"
  environment  = "production"
}
```

Production plan output:
```
+ aws_launch_template.web          instance_type = "t3.medium"
+ aws_autoscaling_group.web        min_size = 3, max_size = 10
+ aws_cloudwatch_metric_alarm.high_cpu[0]  threshold = 80

Plan: 8 to add, 0 to change, 0 to destroy.
```

Same module. Same code. Production gets one more resource and bigger everything.

---

## Pattern 5 — Input Validation

The `validation` block catches invalid values at `terraform plan` time — before anything is deployed:

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, or production."
  }
}
```

If you pass `environment = "prod"` (a common typo):

```
╷
│ Error: Invalid value for variable
│
│   on main.tf line 3, in module "webserver_cluster":
│    3:   environment = "prod"
│
│ Environment must be one of: dev, staging, or production.
│
│ This was checked by the validation rule at
│ variables.tf line 6, in variable "environment".
╵
```

This is far better than letting a wrong value flow through and cause confusing downstream errors. The validation block gives a clear, human-readable error message immediately.

---

## Pattern 6 — Conditional Data Source Lookups

Conditionals work with data sources too. This pattern lets a module work both for new deployments (greenfield) and existing infrastructure (brownfield):

```hcl
variable "use_existing_vpc" {
  description = "Use an existing VPC instead of creating a new one"
  type        = bool
  default     = false
}

# Only looks up the existing VPC if use_existing_vpc is true
data "aws_vpc" "existing" {
  count = var.use_existing_vpc ? 1 : 0
  tags = {
    Name = "existing-vpc"
  }
}

# Only creates a new VPC if use_existing_vpc is false
resource "aws_vpc" "new" {
  count      = var.use_existing_vpc ? 0 : 1
  cidr_block = "10.0.0.0/16"
}

# Whichever one exists, use its ID
locals {
  vpc_id = var.use_existing_vpc ? data.aws_vpc.existing[0].id : aws_vpc.new[0].id
}
```

**Greenfield** (`use_existing_vpc = false`) — creates a brand new VPC.

**Brownfield** (`use_existing_vpc = true`) — looks up the existing VPC by tag and uses it. No new VPC is created.

This is extremely useful when deploying into accounts that already have networking infrastructure you do not want to touch.

---

## Problems I Ran Into

### ❌ Problem 1: Index Out of Range on Conditional Resource

```
Error: Invalid index

  The given key does not identify an element in this collection value.
  aws_cloudwatch_metric_alarm.high_cpu has no element [0].
```

**What happened:** I referenced `aws_cloudwatch_metric_alarm.high_cpu[0].arn` in an output without wrapping it in a ternary guard. When `count = 0`, index `[0]` does not exist.

**Fix:** Always wrap conditional resource references in a ternary:

```hcl
output "alarm_arn" {
  value = local.enable_monitoring ? aws_cloudwatch_metric_alarm.high_cpu[0].arn : null
}
```

---

### ❌ Problem 2: Validation Block Rejecting a Valid Value

I wrote this validation:

```hcl
validation {
  condition     = var.environment == "dev" || "staging" || "production"
  error_message = "Must be dev, staging, or production."
}
```

And it rejected `"dev"` — a value that should be valid.

**What happened:** The expression `"staging"` and `"production"` are non-empty strings which Terraform evaluates as truthy — but the logic was wrong. `"staging"` alone is not a boolean comparison, so the condition was not working as intended.

**Fix:** Use `contains()` which is the correct function for this check:

```hcl
validation {
  condition     = contains(["dev", "staging", "production"], var.environment)
  error_message = "Environment must be one of: dev, staging, or production."
}
```

---

### ❌ Problem 3: Plan-Time Error on Conditional That References Unknown Value

```
Error: Invalid operand

The given value is not known until apply. Terraform cannot evaluate
the condition at plan time.
```

**What happened:** I used a conditional that referenced a value only known after apply — like a resource ID that does not exist yet:

```hcl
# Wrong — resource ID not known at plan time
count = aws_lb.web.id != "" ? 1 : 0
```

**Fix:** Terraform evaluates conditionals at plan time. The condition must use values that are known before apply — like input variables or locals, not resource attributes:

```hcl
# Correct — variable is known at plan time
count = var.enable_monitoring ? 1 : 0
```

---

## What I Learned Today

- **`locals` is the right home for conditional logic** — keep resources clean by moving all decisions into a `locals` block
- **`local.is_production`** as a single derived flag keeps all environment logic DRY
- **`count = condition ? 1 : 0`** is the standard pattern for optional resources
- **Always use a ternary guard** when referencing a conditional resource in an output
- **`validation` blocks** catch bad input at plan time — before anything is deployed
- **Conditionals must use plan-time values** — resource attributes known only after apply cannot be used in conditions
- **The brownfield/greenfield pattern** using paired `count` on a data source and a resource is elegant and widely used in real modules

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
