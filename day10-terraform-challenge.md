# Mastering Loops and Conditionals in Terraform

*Day 10 of the #30DayTerraformChallenge*

---

## Hello

Until today every resource in my Terraform code was declared individually. One resource block, one resource in AWS. If I needed three IAM users I wrote three blocks. If I needed five S3 buckets I wrote five blocks.

That works until it does not. Imagine needing 20 IAM users, or a security group rule per environment, or optional resources that only exist in production.

Today I learned the four tools that fix this completely:

- **`count`** — create N copies of a resource
- **`for_each`** — create resources from a map or set, keyed by value
- **`for` expressions** — transform collections inline
- **Conditionals** — make resources optional with a toggle

These are what make Terraform feel like a real programming language. Let me walk through each one.

---

## Tool 1 — `count`

### The Basics

`count` is the simplest loop. Add it to any resource and Terraform creates that many copies:

```hcl
resource "aws_iam_user" "example" {
  count = 3
  name  = "user-${count.index}"
}
```

`count.index` gives you the position — 0, 1, 2. This creates three users named `user-0`, `user-1`, and `user-2`.

You can also drive `count` from a variable:

```hcl
variable "user_names" {
  type    = list(string)
  default = ["alice", "bob", "charlie"]
}

resource "aws_iam_user" "example" {
  count = length(var.user_names)
  name  = var.user_names[count.index]
}
```

This creates one IAM user per name in the list.

---

### The Problem with `count` and Lists

Here is the trap that catches engineers off guard.

Say you have three users: `["alice", "bob", "charlie"]`.

Terraform creates them like this internally:

```
aws_iam_user.example[0] → alice
aws_iam_user.example[1] → bob
aws_iam_user.example[2] → charlie
```

Now you remove `alice` from the list: `["bob", "charlie"]`.

Terraform renumbers everything from the beginning:

```
aws_iam_user.example[0] → bob    (was alice)
aws_iam_user.example[1] → charlie (was bob)
```

Terraform sees that index 0 changed from `alice` to `bob`, index 1 changed from `bob` to `charlie`, and index 2 no longer exists. So it **destroys and recreates** `bob` and `charlie` — even though you only wanted to remove `alice`.

This is destructive behaviour. In production it means resources you did not intend to touch get deleted and recreated.

**This is exactly why `for_each` exists.**

---

## Tool 2 — `for_each`

`for_each` keys each resource on a value from a set or map — not on a numeric index. Remove one item and only that resource is affected. Everything else stays untouched.

### Using a Set

```hcl
variable "user_names" {
  type    = set(string)
  default = ["alice", "bob", "charlie"]
}

resource "aws_iam_user" "example" {
  for_each = var.user_names
  name     = each.value
}
```

Terraform creates:
```
aws_iam_user.example["alice"]
aws_iam_user.example["bob"]
aws_iam_user.example["charlie"]
```

Now remove `alice`. Terraform only destroys `aws_iam_user.example["alice"]`. Bob and Charlie are not touched at all — because their keys have not changed.

### Using a Map

With a map you can carry additional configuration per item:

```hcl
variable "users" {
  type = map(object({
    department = string
    admin      = bool
  }))
  default = {
    alice = { department = "engineering", admin = true  }
    bob   = { department = "marketing",   admin = false }
  }
}

resource "aws_iam_user" "example" {
  for_each = var.users
  name     = each.key

  tags = {
    Department = each.value.department
  }
}
```

- `each.key` — the map key (`"alice"`, `"bob"`)
- `each.value` — the object for that key (`{ department = "engineering", admin = true }`)
- `each.value.department` — a specific field from that object

This is significantly more powerful than `count` — each resource carries its own configuration.

---

### count vs for_each — When to Use Which

| Situation | Use |
|---|---|
| Creating N identical resources | `count` |
| Creating resources from a list that might change | `for_each` |
| Making a resource optional (0 or 1) | `count = var.enable ? 1 : 0` |
| Creating resources with different configs | `for_each` with a map |
| The resource has no meaningful unique key | `count` |

The rule of thumb: if the resources have unique identities — names, IDs, environments — use `for_each`. If they are truly identical copies, `count` is fine.

---

## Tool 3 — `for` Expressions

`for` expressions do not create resources. They **transform collections** — turn a list into a map, filter items, change values. Think of them as a way to reshape data.

### In an Output — List of Uppercase Names

```hcl
variable "user_names" {
  type    = set(string)
  default = ["alice", "bob", "charlie"]
}

output "upper_names" {
  value = [for name in var.user_names : upper(name)]
}
```

Output:
```
upper_names = ["ALICE", "BOB", "CHARLIE"]
```

### In an Output — Map of Name to ARN

This is the most useful pattern. After creating IAM users with `for_each`, produce a clean map of every username and its ARN:

```hcl
output "user_arns" {
  value = {
    for name, user in aws_iam_user.example : name => user.arn
  }
}
```

Output:
```
user_arns = {
  "alice" = "arn:aws:iam::123456789012:user/alice"
  "bob"   = "arn:aws:iam::123456789012:user/bob"
}
```

This is useful when another part of your infrastructure needs to know the ARN of a specific user — you can look it up by name from this map.

### Filtering with `if`

You can filter a collection inside a `for` expression:

```hcl
output "admin_users" {
  value = [
    for name, user in var.users : name
    if user.admin == true
  ]
}
```

Output:
```
admin_users = ["alice"]
```

---

## Tool 4 — Conditionals

Terraform's conditional uses the ternary operator: `condition ? value_if_true : value_if_false`

### Making Resources Optional with `count`

The most common pattern — toggle a resource on or off with a boolean variable:

```hcl
variable "enable_autoscaling" {
  description = "Enable autoscaling policy for the cluster"
  type        = bool
  default     = true
}

resource "aws_autoscaling_policy" "scale_out" {
  count = var.enable_autoscaling ? 1 : 0

  name                   = "${var.cluster_name}-scale-out"
  autoscaling_group_name = aws_autoscaling_group.web.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 1
  cooldown               = 300
}
```

- `var.enable_autoscaling = true` → `count = 1` → resource is created
- `var.enable_autoscaling = false` → `count = 0` → resource is skipped entirely

When you run `terraform plan` with `enable_autoscaling = false`:

```
  # aws_autoscaling_policy.scale_out will not be created
  (count = 0)

Plan: 0 to add, 0 to change, 0 to destroy.
```

---

### Environment-Based Instance Sizing with `locals`

Instead of scattering ternary operators through your resource arguments, centralise conditional logic in `locals`:

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

locals {
  instance_type    = var.environment == "production" ? "t3.medium" : "t3.micro"
  min_size         = var.environment == "production" ? 4 : 2
  max_size         = var.environment == "production" ? 10 : 4
  enable_autoscaling = var.environment == "production" ? true : false
}

resource "aws_launch_template" "web" {
  name_prefix   = "${var.cluster_name}-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = local.instance_type  # ← reads from locals
}

resource "aws_autoscaling_group" "web" {
  min_size = local.min_size
  max_size = local.max_size
}
```

All the conditional logic lives in one place. The resources just read from `local.*`. If the logic changes, you update the `locals` block — not every resource that uses it.

---

## Refactoring the Webserver Cluster

I went back to my webserver cluster from earlier in the challenge and applied everything above.

**Before — repeated security group rules:**

```hcl
# Before — two separate ingress rules written out individually
resource "aws_security_group_rule" "allow_http" {
  type              = "ingress"
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.alb_sg.id
}

resource "aws_security_group_rule" "allow_https" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.alb_sg.id
}
```

**After — `for_each` over a map of rules:**

```hcl
variable "alb_ingress_rules" {
  type = map(object({
    from_port = number
    to_port   = number
    protocol  = string
  }))
  default = {
    http  = { from_port = 80,  to_port = 80,  protocol = "tcp" }
    https = { from_port = 443, to_port = 443, protocol = "tcp" }
  }
}

resource "aws_security_group_rule" "alb_ingress" {
  for_each = var.alb_ingress_rules

  type              = "ingress"
  from_port         = each.value.from_port
  to_port           = each.value.to_port
  protocol          = each.value.protocol
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.alb_sg.id
}
```

Adding a new rule now means adding one line to the `alb_ingress_rules` variable — not writing a new resource block.

---

## Problems I Ran Into

### ❌ Problem 1: `count` and `for_each` Cannot Be Used Together

I tried to use both `count` and `for_each` on the same resource:

```hcl
resource "aws_iam_user" "example" {
  count    = 2
  for_each = var.user_names  # ← ERROR
  name     = each.value
}
```

**Error:**

```
Error: Invalid combination of "count" and "for_each"

  on main.tf line 2, in resource "aws_iam_user" "example":

The "count" and "for_each" meta-arguments are mutually exclusive;
only one may be set.
```

**Fix:** Pick one. If you need unique keys, use `for_each`. If you just need a fixed number of identical resources, use `count`. They cannot be combined on the same resource.

---

### ❌ Problem 2: `each.key` vs `each.value` Confusion

When using `for_each` with a set, I kept using `each.key` instead of `each.value` and got empty names:

```hcl
# Wrong
resource "aws_iam_user" "example" {
  for_each = var.user_names  # set of strings
  name     = each.key        # ← for sets, each.key == each.value, but this is confusing
}
```

**What happened:** For a **set**, `each.key` and `each.value` are actually the same thing — both return the element value. But for a **map**, `each.key` is the key and `each.value` is the value object. Using `each.key` on a set works but is misleading.

**Fix:** Use `each.value` for sets and `each.key` / `each.value` correctly for maps:

```hcl
# For sets — use each.value
resource "aws_iam_user" "example" {
  for_each = var.user_names   # set
  name     = each.value       # ← the string value
}

# For maps — use each.key and each.value
resource "aws_iam_user" "example" {
  for_each = var.users        # map
  name     = each.key         # ← the map key
  tags = {
    Department = each.value.department  # ← field from the value object
  }
}
```

---
### ❌ Problem 3: Referencing `for_each` Resources in Outputs

After creating users with `for_each`, I tried to reference one specific user's ARN like this:

```hcl
output "alice_arn" {
  value = aws_iam_user.example.arn  # ← ERROR
}
```

**Error:**

```
Error: Missing resource instance key

Because aws_iam_user.example has "for_each" set, its attributes must
be accessed on specific instances. For example, to correlate with indices
of a referring resource, use:
    aws_iam_user.example["alice"]
```

**Fix:** Reference a specific instance using its key, or use a `for` expression to get all of them:

```hcl
# Reference one specific user
output "alice_arn" {
  value = aws_iam_user.example["alice"].arn
}

# Or output all user ARNs as a map
output "all_user_arns" {
  value = {
    for name, user in aws_iam_user.example : name => user.arn
  }
}
```

---

## What I Learned Today

- **`count`** is simple but fragile with lists — removing an item from the middle destroys everything after it
- **`for_each`** keys resources on unique values — only the removed item is affected, everything else stays
- **`for` expressions** reshape data — they do not create resources, they transform collections for outputs and locals
- **`count = var.enable ? 1 : 0`** is the standard pattern for optional resources
- **`locals`** is the right place for conditional logic — keeps resource blocks clean and readable
- **`count` and `for_each` are mutually exclusive** — you cannot use both on the same resource
- **Always use `toset()`** when passing a list variable to `for_each`
- **`for_each` resources need a key** when referenced — `resource["key"].attribute`
---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*
---
