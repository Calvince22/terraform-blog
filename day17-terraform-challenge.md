# The Importance of Manual Testing in Terraform

**Day 17 of 30 — #30DayTerraformChallenge**

---

## Why Manual Testing Comes First

Automated tests are powerful, but they are built on assumptions. Someone has to verify that those assumptions are correct before they get baked into code. That someone is you, and that process is manual testing.

Chapter 9 of *Terraform: Up & Running* makes a case that feels uncomfortable at first: even if you plan to automate everything, you must start by running your infrastructure manually, watching what happens, and documenting the results. Automated tests tell you whether your infrastructure still works the way it used to. Manual tests tell you whether it ever worked correctly in the first place.

Today I built a structured manual testing checklist, ran it against my webserver cluster across dev and production environments, documented every pass and failure, and cleaned up all resources afterward. Here is what I learned.

---

## Building a Structured Test Checklist

A manual test without a checklist is just clicking around. The checklist is what transforms ad hoc exploration into a repeatable process that another engineer can run without asking you questions.

I organised my checklist into five categories, each verifying a different layer of the infrastructure.

### 1. Provisioning Verification

The first layer checks that Terraform itself behaves correctly before a single AWS resource is created:

- `terraform init` completes without errors
- `terraform validate` passes cleanly
- `terraform plan` shows the expected number and type of resources
- `terraform apply` completes without errors

### 2. Resource Correctness

Once resources exist, verify they match the configuration — not just that they exist:

- All expected resources are visible in the AWS Console
- Resource names, tags, and regions match what the variables specify
- Security group rules are exactly as defined — no extra rules, no missing rules

### 3. Functional Verification

This is the layer automated tests most often miss. Does the infrastructure actually do what it is supposed to do?

- ALB DNS name resolves
- `curl http://<alb-dns>` returns the expected response
- All instances in the ASG pass health checks
- Stopping one instance manually triggers ASG replacement

### 4. State Consistency

Terraform's state file is its model of the world. It must match reality:

- `terraform plan` returns **No changes** immediately after a fresh apply
- `terraform show` accurately reflects what exists in AWS

### 5. Regression Check

Make a deliberate small change and verify that only the expected diff appears:

- Add a tag or change a description — `terraform plan` shows only that change
- Apply it — `terraform plan` returns clean afterward

---

## Provisioning vs Functional Verification

The distinction between provisioning verification and functional verification is one of the most valuable concepts in Chapter 9, and it is easy to underestimate.

**Provisioning verification** answers: did Terraform create the resources it was supposed to create? It checks names, tags, security group rules, instance types, and region. You can verify most of this without the infrastructure running at all.

**Functional verification** answers: does the infrastructure actually work? This requires the system to be running and reachable. You cannot verify that the ALB returns the expected response by reading the Terraform state — you have to send an HTTP request and check the output.

Automated tests tend to focus heavily on provisioning verification because it is easy to assert on resource attributes. Functional verification requires real infrastructure, real network calls, and real wait times, which makes it slower and more expensive to automate. That is exactly why manual testing covers it first.

> **Key insight:** `terraform validate` can pass. `terraform plan` can show exactly the resources you expect. `terraform apply` can complete without a single error. And the application can still be broken. Functional verification is the only layer that catches this.

---

## Test Execution Results

I ran the full checklist against my webserver cluster. Here are the results:

| Test | Expected | Actual | Result |
|------|----------|--------|--------|
| `terraform init` | Success | Success | ✅ PASS |
| `terraform validate` | No errors | No errors | ✅ PASS |
| `terraform plan` resource count | 14 resources | 14 resources | ✅ PASS |
| `terraform apply` | No errors | No errors | ✅ PASS |
| Resources in AWS Console | All present | All present | ✅ PASS |
| Security group rules | Exact match | Exact match | ✅ PASS |
| ALB DNS resolves | Resolves | Resolves | ✅ PASS |
| `curl` ALB returns response | `Hello World v2` | `Hello World v2` | ✅ PASS |
| ASG health checks | All healthy | All healthy | ✅ PASS |
| ASG replaces stopped instance | Replacement launched | Replacement launched | ✅ PASS |
| `plan` clean after apply | No changes | 1 change detected | ❌ FAIL |
| Regression: tag change only | 1 change shown | 1 change shown | ✅ PASS |

### The Failure — and Why It Matters

The most valuable result was the FAIL on state consistency. After a clean apply, `terraform plan` detected one resource change: a missing tag on the `aws_security_group.instance` resource. The tag was defined on the ASG launch template but never applied to the security group itself.

```
Test:     terraform plan returns clean after apply
Command:  terraform plan
Expected: "No changes. Your infrastructure matches the configuration."
Actual:   1 resource change detected — missing tag on security group
Result:   FAIL
Fix:      Added missing tag to aws_security_group.instance resource, re-applied
```

The fix:

```hcl
resource "aws_security_group" "instance" {
  name   = "${var.cluster_name}-instance"
  vpc_id = data.aws_vpc.default.id

  tags = {
    Name      = "${var.cluster_name}-instance-sg"
    ManagedBy = "terraform"
  }
}
```

After adding the tag and re-applying, `terraform plan` returned clean. This failure is now a test case I can automate: assert that every security group resource has a `ManagedBy` tag.

---

## Multi-Environment Comparison

I ran the full checklist against both dev and production. The results were largely consistent, with one notable difference.

The dev environment uses `t2.micro` instances. Production uses `t3.small`. The ASG replacement test passed in both environments, but replacement time differed by approximately 45 seconds — production instances took longer to pass health checks because the `t3.small` runs a slightly heavier application startup process. This did not cause a failure, but it is worth noting for timeout thresholds in future automated tests.

The security group tag issue was present in both environments, which confirmed it was a configuration problem rather than environment-specific drift.

---

## Cleanup Discipline

Chapter 9 is emphatic about this: cleanup is not an afterthought. It is part of the test. Running `terraform destroy` and assuming it succeeded is not enough — you must verify.

My cleanup process after every test run:

```bash
# Review what will be destroyed before running
terraform plan -destroy

# Destroy with explicit approval
terraform destroy

# Verify all EC2 instances are gone
aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=terraform" \
  --query "Reservations[*].Instances[*].InstanceId"

# Verify all load balancers are gone
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[*].LoadBalancerArn"
```

Both commands returned empty arrays after my destroy runs. On the first attempt, one target group was left behind after a partial destroy failure — the destroy had timed out waiting for the ALB to deregister targets. I manually deleted the target group from the Console and ran `terraform destroy` again, which completed cleanly.

### Why Cleanup Is Harder Than It Sounds

The author's point in Chapter 9 is that `terraform destroy` can fail partway through due to dependency ordering issues, AWS API timeouts, or resources that were created outside Terraform and are referenced by managed resources. When this happens, the state file is partially updated — it no longer matches either the original configuration or reality. Terraform will not automatically retry orphaned resources on the next run. You need to find them manually and either import them back into state or delete them manually.

The cost risk is real. A forgotten ALB running idle in `us-east-1` costs roughly $16/month. A forgotten NAT Gateway costs $32/month. A test environment left running over a weekend can generate a surprise bill before you notice. Cleanup verification after every test run is the only way to prevent this.

---

## Lab Takeaways — terraform import

The import lab covered `terraform import`, which solves the brownfield problem: you have existing infrastructure that was created manually or by another tool, and you want to bring it under Terraform management.

**What `terraform import` solves:** it reads an existing resource from AWS and writes its current state into the Terraform state file. From that point on, Terraform knows the resource exists and will include it in plan and apply operations.

**What `terraform import` does not solve:** it does not write HCL configuration for you. After importing, you must manually write the resource block in your `.tf` files to match what was imported. If the HCL does not match the imported state, `terraform plan` will show changes — and applying those changes will modify or destroy the resource you just imported.

The practical workflow is:

1. Import the resource into state
2. Run `terraform plan` to see what Terraform thinks needs to change
3. Update your HCL to eliminate those planned changes
4. Run `terraform plan` again — repeat until **No changes**
5. Only then is the resource fully under Terraform management

The import lab reinforced a broader point: Terraform's state file is a model of intent, not just a record of what exists. The model and reality must be kept in sync, and that synchronisation is your responsibility.

---

## Challenges and Fixes

**Challenge 1 — Missing security group tag causing plan drift**
After a clean apply, `terraform plan` detected a change on `aws_security_group.instance`. Root cause: the `ManagedBy` tag was applied to the launch template and ASG but not to the security group resource itself. Fix: added the `tags` block to the security group resource and re-applied.

**Challenge 2 — Partial destroy leaving orphaned target group**
The first `terraform destroy` run timed out while waiting for the ALB to finish deregistering targets. The target group was left in a `draining` state that Terraform could not delete. Fix: waited for draining to complete in the AWS Console, manually deleted the target group, then ran `terraform destroy` again.

---

## Key Takeaways

- Manual testing is not a step you skip once you have automated tests. It is the baseline that makes automated tests meaningful.
- Provisioning verification and functional verification are different layers. Both are necessary. Automated tests tend to cover provisioning; manual tests are often the only way to verify function.
- Every failure you find manually is a test case you can automate. Document them all — passes and failures both.
- Cleanup is part of the test. Always verify with AWS CLI commands after every destroy.
- `terraform import` solves state import, not configuration generation. You still have to write the HCL.

---
