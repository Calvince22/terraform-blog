# Automating Terraform Testing: From Unit Tests to End-to-End Validation

**Day 18 of 30 — #30DayTerraformChallenge**

---

## The Problem with Manual Testing at Scale

Yesterday I built a structured manual testing checklist and ran it against my webserver cluster. It worked — but it took the better part of an afternoon, required me to be present for every step, and would need to be repeated every single time something changed.

That does not scale.

The moment infrastructure grows beyond what one person can test in an afternoon, you need automated tests that run on every change, catch regressions before they reach production, and give the entire team confidence to move fast. Today I completed Chapter 9 and implemented all three layers of Terraform automated testing: unit tests, integration tests, and end-to-end tests — then wired them together into a CI/CD pipeline.

---

## The Three Layers of Terraform Testing

Before writing any code, it helps to understand what each layer is actually for and why all three are necessary.

| Test Type | Tool | Deploys Real Infra | Time | Cost | What It Catches |
|-----------|------|-------------------|------|------|-----------------|
| Unit | `terraform test` | No | Seconds | Free | Config logic errors, variable wiring, naming conventions |
| Integration | Terratest | Yes | 5–15 min | Low (~$0.10–0.50) | Real resource behaviour, outputs, health checks |
| End-to-End | Terratest | Yes (full stack) | 15–30 min | Medium (~$1–5) | Cross-module wiring, full application path |

Unit tests are fast and free but only verify your configuration logic. End-to-end tests are thorough but slow and expensive. The right strategy uses all three — run unit tests on every pull request, integration tests on every merge to main, and end-to-end tests on a schedule or before major releases.

---

## Unit Tests with terraform test

Terraform 1.6+ ships with a native testing framework using `.tftest.hcl` files. These tests run against `terraform plan` only — no real AWS resources are created, no costs incurred, and results come back in seconds.

### The Test File

```hcl
# modules/services/webserver-cluster/webserver_cluster_test.tftest.hcl

variables {
  cluster_name  = "test-cluster"
  instance_type = "t2.micro"
  min_size      = 1
  max_size      = 2
  environment   = "dev"
}

run "validate_cluster_name" {
  command = plan

  assert {
    condition     = aws_autoscaling_group.example.name_prefix == "test-cluster-"
    error_message = "ASG name prefix must match the cluster_name variable"
  }
}

run "validate_instance_type" {
  command = plan

  assert {
    condition     = aws_launch_configuration.example.instance_type == "t2.micro"
    error_message = "Instance type must match the instance_type variable"
  }
}

run "validate_security_group_port" {
  command = plan

  assert {
    condition     = aws_security_group.instance.ingress[0].from_port == 8080
    error_message = "Security group must allow traffic on port 8080"
  }
}
```

### What Each Block Tests and Why

**`validate_cluster_name`** — verifies that the `cluster_name` variable is correctly wired into the ASG `name_prefix`. This catches a common mistake: the variable is defined and accepted, but never actually used in the resource. Without this test, a broken variable reference would silently result in a hardcoded name in production.

**`validate_instance_type`** — verifies that the `instance_type` variable flows through to the launch configuration. This matters because launch configurations are easy to misconfigure — the variable might be passed to the wrong argument, or the wrong resource altogether.

**`validate_security_group_port`** — verifies that the security group allows traffic on port 8080. This is a regression guard: if someone changes the server port or the security group resource without updating both, this test fails immediately. It catches the kind of subtle mismatch that only shows up as a broken application at runtime.

### Running Unit Tests

```bash
terraform init
terraform test
```

### Output

```
webserver_cluster_test.tftest.hcl... in progress
  run "validate_cluster_name"... pass
  run "validate_instance_type"... pass
  run "validate_security_group_port"... pass
webserver_cluster_test.tftest.hcl... tearing down
webserver_cluster_test.tftest.hcl... pass

Success! 3 passed, 0 failed.
```

All three tests passed in under 8 seconds with no AWS resources created.

---

## Integration Tests with Terratest

Integration tests deploy real infrastructure, run assertions against it, and then destroy it. They verify that the resources Terraform creates actually behave the way the configuration intends — something a plan-only test cannot tell you.

### The Test

```go
// test/webserver_cluster_test.go
package test

import (
  "fmt"
  "testing"
  "time"

  "github.com/gruntwork-io/terratest/modules/http-helper"
  "github.com/gruntwork-io/terratest/modules/random"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/stretchr/testify/assert"
)

func TestWebserverClusterIntegration(t *testing.T) {
  t.Parallel()

  uniqueID    := random.UniqueId()
  clusterName := fmt.Sprintf("test-cluster-%s", uniqueID)

  terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "../modules/services/webserver-cluster",
    Vars: map[string]interface{}{
      "cluster_name":  clusterName,
      "instance_type": "t2.micro",
      "min_size":      1,
      "max_size":      2,
      "environment":   "dev",
    },
  })

  // Always destroy at the end, even if assertions fail
  defer terraform.Destroy(t, terraformOptions)

  terraform.InitAndApply(t, terraformOptions)

  albDnsName := terraform.Output(t, terraformOptions, "alb_dns_name")
  url        := fmt.Sprintf("http://%s", albDnsName)

  // Retry for up to 5 minutes — ALB takes time to register instances
  http_helper.HttpGetWithRetryWithCustomValidation(
    t,
    url,
    nil,
    30,
    10*time.Second,
    func(status int, body string) bool {
      return status == 200 && len(body) > 0
    },
  )

  assert.NotEmpty(t, albDnsName, "ALB DNS name should not be empty")
}
```

### Why `defer terraform.Destroy` Is Critical

`defer terraform.Destroy(t, terraformOptions)` registers the destroy call to run when the test function exits — regardless of whether it exits cleanly, panics, or fails an assertion midway through. Without this, a failed assertion would leave real AWS resources running and billing your account.

The `defer` keyword in Go guarantees execution even in failure paths. This is the single most important pattern in Terratest. Without it, you would need to manually clean up after every failed test run, which is error-prone and expensive.

`terraform.WithDefaultRetryableErrors` wraps the options with automatic retry logic for transient AWS API errors — throttling, eventual consistency issues, and intermittent network failures — so the test does not fail on infrastructure noise.

### Installing Dependencies and Running

```bash
cd test
go mod init test
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/gruntwork-io/terratest/modules/http-helper
go test -v -timeout 30m ./...
```

### Output

```
=== RUN   TestWebserverClusterIntegration
=== PAUSE TestWebserverClusterIntegration
=== CONT  TestWebserverClusterIntegration

TestWebserverClusterIntegration 2026-04-07T10:12:34Z terraform [command=apply]
...
Apply complete! Resources: 14 added, 0 changed, 0 destroyed.

Outputs:
alb_dns_name = "test-cluster-abc123.us-east-1.elb.amazonaws.com"

TestWebserverClusterIntegration 2026-04-07T10:16:22Z http-helper [GET http://test-cluster-abc123.us-east-1.elb.amazonaws.com - attempt 4/30 - status 200]
--- PASS: TestWebserverClusterIntegration (7m14s)
PASS
ok      test    434.112s
```

The test took just over 7 minutes — most of that waiting for the ALB to register instances and pass health checks.

---

## End-to-End Tests

End-to-end tests deploy the complete stack and verify that modules work together. They catch things integration tests miss: a VPC ID that does not get passed correctly between modules, a subnet CIDR conflict, or a security group that blocks traffic between tiers.

```go
func TestFullStackEndToEnd(t *testing.T) {
  t.Parallel()

  uniqueID := random.UniqueId()

  // Deploy VPC first
  vpcOptions := &terraform.Options{
    TerraformDir: "../modules/networking/vpc",
    Vars: map[string]interface{}{
      "vpc_name": fmt.Sprintf("test-vpc-%s", uniqueID),
    },
  }
  defer terraform.Destroy(t, vpcOptions)
  terraform.InitAndApply(t, vpcOptions)

  vpcID     := terraform.Output(t, vpcOptions, "vpc_id")
  subnetIDs := terraform.OutputList(t, vpcOptions, "private_subnet_ids")

  // Deploy app using VPC outputs
  appOptions := &terraform.Options{
    TerraformDir: "../modules/services/webserver-cluster",
    Vars: map[string]interface{}{
      "cluster_name": fmt.Sprintf("test-app-%s", uniqueID),
      "vpc_id":       vpcID,
      "subnet_ids":   subnetIDs,
      "environment":  "dev",
    },
  }
  defer terraform.Destroy(t, appOptions)
  terraform.InitAndApply(t, appOptions)

  albDnsName := terraform.Output(t, appOptions, "alb_dns_name")
  http_helper.HttpGetWithRetry(
    t,
    fmt.Sprintf("http://%s", albDnsName),
    nil,
    200,
    "Hello",
    30,
    10*time.Second,
  )
}
```

Note the destroy order: because `defer` runs in LIFO order (last-in, first-out), the app stack is destroyed before the VPC — which is the correct dependency order. If the VPC were destroyed first while EC2 instances still existed inside it, the destroy would fail.

---

## The CI/CD Pipeline

The pipeline runs unit tests on every pull request and integration tests only on merges to main. This keeps PR feedback fast (seconds) while still catching real infrastructure regressions on every commit that lands.

```yaml
# .github/workflows/terraform-test.yml
name: Terraform Tests

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"

      - name: Terraform Init
        run: terraform init
        working-directory: modules/services/webserver-cluster

      - name: Run Unit Tests
        run: terraform test
        working-directory: modules/services/webserver-cluster

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    needs: unit-tests

    env:
      AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_DEFAULT_REGION:    us-east-1

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v4
        with:
          go-version: "1.21"

      - name: Run Integration Tests
        run: go test -v -timeout 30m ./...
        working-directory: test
```

### Why the Pipeline Is Structured This Way

**`needs: unit-tests`** — integration tests only run if unit tests pass first. There is no point deploying real infrastructure to test configuration that is already known to be broken. This also means a unit test failure gives feedback in seconds and stops the pipeline before any AWS costs are incurred.

**`if: github.event_name == 'push'`** — integration tests only run on merges to main, not on pull requests. Running Terratest on every PR would mean every developer's draft PR triggers a 10-minute AWS deployment. That is slow, expensive, and fills the AWS account with test resources. Unit tests give PR authors fast feedback; integration tests confirm the merged result is solid.

**AWS credentials as secrets** — `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are stored in GitHub Actions secrets and injected as environment variables. They are never written to the workflow file or committed to the repository.

---

## Integration vs End-to-End: The Key Difference

An **integration test** deploys one module in isolation and verifies it works on its own. It tests the webserver cluster as a unit: does it deploy, does the ALB respond, do the health checks pass?

An **end-to-end test** deploys multiple modules together and verifies they work as a system. It tests whether the VPC module's outputs are compatible with the webserver module's inputs, whether the security groups allow traffic across tiers, and whether the full application path from browser to backend functions correctly.

The author recommends running unit tests on every PR because they are free and take seconds — the cost of running them is effectively zero. End-to-end tests are recommended less frequently (on merges to main, nightly, or before major releases) because they deploy a complete stack, take 20-30 minutes, and generate real AWS costs. Running them on every PR would slow down development significantly and create a constant stream of partial infrastructure in your AWS account.

The right cadence is: unit tests on every commit, integration tests on every merge, end-to-end tests on a schedule or before releases.

---

## Challenges and Fixes

**Challenge 1 — Go module path conflict**

Running `go mod init test` caused an import conflict because `test` is a reserved package name in Go. Fix: renamed the module to `go mod init github.com/yourusername/terraform-tests` and updated all import references.

**Challenge 2 — Terratest timeout on ALB health check**

The first integration test run timed out waiting for the ALB to return a 200 response. The retry was set to 10 attempts at 10-second intervals (100 seconds total), but the ALB needed closer to 4 minutes to register instances and pass health checks. Fix: increased retries to 30 (5 minutes total), which matched the behaviour observed during manual testing yesterday.

**Challenge 3 — GitHub Actions IAM permission failure**

The integration test job failed with `AccessDenied` when trying to create the Auto Scaling Group. The IAM user attached to the GitHub Actions secret had EC2 and ELB permissions but was missing `autoscaling:CreateAutoScalingGroup`. Fix: updated the IAM policy to include the full set of autoscaling permissions required by the module.

```json
{
  "Effect": "Allow",
  "Action": [
    "autoscaling:CreateAutoScalingGroup",
    "autoscaling:UpdateAutoScalingGroup",
    "autoscaling:DeleteAutoScalingGroup",
    "autoscaling:DescribeAutoScalingGroups",
    "autoscaling:CreateLaunchConfiguration",
    "autoscaling:DeleteLaunchConfiguration",
    "autoscaling:DescribeLaunchConfigurations"
  ],
  "Resource": "*"
}
```

**Challenge 4 — End-to-end test destroy order**

The first E2E test run failed during cleanup because the VPC was destroyed while EC2 instances still existed inside it. The `defer` calls were in the wrong order — VPC defer was registered before the app defer, so it ran last (LIFO). Fix: registered app defer before VPC defer, so the app destroys first and the VPC destroys cleanly afterward.

---

## Key Takeaways

- Unit tests with `terraform test` are free, instant, and should run on every commit. They test configuration logic, not behaviour.
- Integration tests with Terratest deploy real infrastructure and verify real behaviour. They are the only way to know the ALB actually responds, the ASG actually replaces instances, and the outputs are actually correct.
- End-to-end tests verify that modules work together as a system. They catch cross-module wiring issues that no single-module test can find.
- `defer terraform.Destroy` is not optional. It is the guarantee that prevents runaway AWS costs when tests fail.
- CI/CD makes the testing strategy sustainable: fast feedback on PRs, real validation on merges.

---

`#30DayTerraformChallenge #TerraformChallenge #Terraform #Testing #DevOps #CICD #AWSUserGroupKenya #EveOps`
