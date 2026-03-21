# Managing Terraform State: Best Practices for DevOps

*Day 6 of the #30DayTerraformChallenge*

---

## Hello 

Today was one of the most important days of the entire challenge.

Not because we deployed anything flashy — but because we tackled something that breaks real teams in production every day: **Terraform state management**.

By the end of today I had moved my state file off my laptop and into a secure, shared, versioned S3 bucket — with DynamoDB locking to prevent two people from destroying each other's work.

Let me walk you through everything.

---

## What Is Terraform State — A Quick Recap

Every time you run `terraform apply`, Terraform writes a file called `terraform.tfstate` to your project folder.

This file is Terraform's memory. It tracks:

- Every resource it created and its real AWS ID
- All the attributes of each resource (instance type, AMI, tags, ARNs)
- The relationships and dependencies between resources
- Output values

Think of it like a receipt. When you run `terraform plan`, Terraform compares your code against this receipt and against real AWS — and tells you what needs to change.

Here is a simplified example of what it looks like inside:

```json
{
  "version": 4,
  "terraform_version": "1.7.5",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "terraform_state",
      "instances": [
        {
          "attributes": {
            "id": "my-terraform-state-bucket",
            "bucket": "my-terraform-state-bucket",
            "region": "us-east-1",
            "arn": "arn:aws:s3:::my-terraform-state-bucket"
          }
        }
      ]
    }
  ]
}
```

---

## The Problem with Local State

When you are working alone, local state is fine. The file lives on your laptop and only you touch it.

But the moment a second person joins your project, everything breaks down:

**Problem 1 — No shared state**
Engineer A applies from their laptop. Engineer B has a different state file on their laptop. Neither knows what the other has deployed. Terraform plans become unreliable.

**Problem 2 — Concurrent runs**
Engineer A and Engineer B both run `terraform apply` at the same time. They both read the same state, make changes, and try to write back simultaneously. The result is corrupted state and potentially duplicated or deleted resources.

**Problem 3 — Secrets in plain text**
The state file can contain sensitive values — database passwords, API keys, private IPs — stored in plain text. Committing it to a Git repository (even a private one) is a serious security risk.

**The fix for all three:** Remote state storage with locking.

---

## The Solution — S3 Backend with DynamoDB Locking

Instead of storing `terraform.tfstate` on your laptop, you store it in an S3 bucket that everyone on the team can access. DynamoDB handles locking so only one person can run `apply` at a time.

Here is what the architecture looks like:

```
Engineer A (terraform apply)
         |
         ├──→ Reads/writes state from S3 bucket
         ├──→ Acquires lock in DynamoDB
         |
Engineer B (terraform apply — at same time)
         |
         └──→ Tries to acquire DynamoDB lock
              Gets blocked until Engineer A finishes
```

---

## Step 1 — Create the S3 Bucket and DynamoDB Table

There is a catch here called the **bootstrap problem**.

You cannot use Terraform to create the S3 bucket that Terraform needs to store its own state — because Terraform needs the bucket to exist before it can store any state at all. It is a chicken-and-egg situation.

The solution: create the S3 bucket and DynamoDB table first, with a separate Terraform configuration that uses local state. Once they exist, configure your main project to use them as the remote backend.

Here is the code to create the backend infrastructure:

```hcl
provider "aws" {
  region = "us-east-1"
}

# S3 bucket to store the state file
resource "aws_s3_bucket" "terraform_state" {
  bucket = "your-unique-terraform-state-bucket"

  # Prevent accidental deletion of this bucket
  # Deleting it means losing all your state history
  lifecycle {
    prevent_destroy = true
  }
}

# Enable versioning so you can recover from accidental state corruption
resource "aws_s3_bucket_versioning" "enabled" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Encrypt the state file at rest
# State files can contain sensitive values — always encrypt them
resource "aws_s3_bucket_server_side_encryption_configuration" "default" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block all public access to the state bucket
resource "aws_s3_bucket_public_access_block" "public_access" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

Run this with:

```bash
terraform init
terraform apply
```

Once both resources exist in AWS, move on to Step 2.

---

## Step 2 — Configure the Remote Backend

Now add a `terraform` block to your main project configuration that points to the S3 bucket and DynamoDB table you just created:

```hcl
terraform {
  backend "s3" {
    bucket         = "your-unique-terraform-state-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    user_lockfile = true
    encrypt        = true
  }
}
```

Let me explain every argument:

| Argument | What It Does |
|---|---|
| `bucket` | The name of the S3 bucket to store state in |
| `key` | The path inside the bucket where the state file is saved |
| `region` | The AWS region where the bucket lives |
| `user_lockfile` |Use for state locking |
| `encrypt` | Encrypts the state file in transit (in addition to the bucket encryption at rest) |

The `key` argument is like a file path inside the bucket. Using a path like `global/s3/terraform.tfstate` keeps things organised when you have multiple projects all using the same bucket.

---

## Step 3 — Migrate Your Local State to S3

Now run `terraform init` again in your main project:

```bash
terraform init
```

Terraform detects the new backend and asks if you want to copy your existing local state to S3:

```
Initializing the backend...

Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend
  to the newly configured "s3" backend. No existing state was found in the
  newly configured "s3" backend. Do you want to copy this state to the new
  backend?

  Enter a value: yes

Successfully configured the backend "s3"!
```

Type `yes`. Terraform copies your entire local state file to the S3 bucket.

**Confirm it worked:**

Go to the AWS Console → S3 → your bucket. You should see your state file at the path you specified in `key`. It will be versioned and encrypted.

From this point on, every `terraform apply` reads and writes state from S3 — not your local machine.

---

## Step 4 — Testing State Locking

This was the most satisfying experiment of the day.

I opened two terminal windows pointing to the same Terraform project.

**Terminal 1:**
```bash
terraform apply
```

While that was running, in **Terminal 2:**
```bash
terraform plan
```

Terminal 2 immediately showed this:

```
╷
│ Error: Error acquiring the state lock
│
│ Error message: ConditionalCheckFailedException: The conditional request failed
│
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time. Please resolve the issue above and try
│ again. For most commands, you can disable locking with the "-lock=false"
│ flag, but this is not recommended.
│
│ Lock Info:
│   ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890
│   Path:      your-bucket/global/s3/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       user@machine
│   Version:   1.7.5
│   Created:   2026-03-20 10:15:30 UTC
│   Info:
╵
```

This is exactly what you want to see. The lock prevented the second terminal from reading or writing state while the first apply was in progress.

**Why this matters in a team:** Without locking, two engineers running apply simultaneously can each read the same state, make conflicting changes, and both try to write back. The result is corrupted state that is very hard to recover from. DynamoDB locking makes this physically impossible.

---

## Inspecting the State File

After deploying, I ran these commands to inspect what Terraform is tracking:

```bash
terraform state list
```

Output:
```
aws_dynamodb_table.terraform_locks
aws_s3_bucket.terraform_state
aws_s3_bucket_public_access_block.public_access
aws_s3_bucket_server_side_encryption_configuration.default
aws_s3_bucket_versioning.enabled
```

Then I inspected one resource in detail:

```bash
terraform state show aws_s3_bucket.terraform_state
```

Output:
```
# aws_s3_bucket.terraform_state:
resource "aws_s3_bucket" "terraform_state" {
    arn                         = "arn:aws:s3:::your-unique-terraform-state-bucket"
    bucket                      = "your-unique-terraform-state-bucket"
    bucket_domain_name          = "your-unique-terraform-state-bucket.s3.amazonaws.com"
    id                          = "your-unique-terraform-state-bucket"
    region                      = "us-east-1"
    tags                        = {}
}
```

Every attribute AWS knows about this bucket is recorded in the state. This is how Terraform knows what to change on the next apply — and it is also why the state file can contain sensitive information if you deploy databases or secrets.

---

## What Happens to State After terraform destroy?

After running `terraform destroy`, the state file is not deleted — it is emptied. It remains in S3 as an empty resources list.

This is intentional. The empty state file is Terraform saying "I know about this project, and I know it has nothing deployed." If the file were deleted, Terraform would not know whether the infrastructure was destroyed or simply untracked.

With S3 versioning enabled, you can also see every previous version of the state file — so if you accidentally destroy something, you can recover the previous state and understand what existed before.

---

## Why Never Commit the State File to Git

Add these lines to your `.gitignore` right now:

```
# .gitignore
terraform.tfstate
terraform.tfstate.backup
.terraform/
.terraform.lock.hcl
```

Reasons:

- The state file contains resource IDs, IP addresses, and can contain passwords and API keys in plain text
- Git is not designed for files that change on every apply — it creates noisy, unhelpful diffs
- Merging conflicting state files from two branches is extremely difficult and error-prone
- A public repository exposing your state file gives attackers a detailed map of your infrastructure

Remote state in S3 solves all of these.

---

## Problems I Ran Into

### ❌ Problem 1: Bucket Name Already Taken

```
Error: creating S3 Bucket: BucketAlreadyExists
```

**What happened:** S3 bucket names are globally unique across all AWS accounts. The name I chose was already taken by someone else.

**Fix:** I added a unique suffix to make the name less likely to conflict:

```hcl
bucket = "terraform-state-yourname-2026"
```

Use your name, project name, or a random string to make it unique.

---

### ❌ Problem 1: dynamodb_table Is Deprecated — Use use_lockfile
When I ran terraform init after configuring the S3 backend, I got this warning:
╷
│ Warning: Deprecated Parameter
│
│   on backend.tf line 6, in terraform:
│    6:     dynamodb_table = "terraform-state-locks"
│
│ The parameter "dynamodb_table" is deprecated.
│ Use parameter "use_lockfile" instead.
╵
What happened: Newer versions of the Terraform AWS S3 backend have replaced dynamodb_table with a simpler use_lockfile parameter. Terraform was warning me that the argument I used is on its way out.
What I tried first: I updated the backend block to use use_lockfile = true as the warning suggested:
hclterraform {
  backend "s3" {
    bucket       = "your-unique-terraform-state-bucket"
    key          = "global/s3/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
It worked perfectly. Terraform now supports native state locking using S3 lockfiles via the use_lockfile parameter. 
This reduces dependency on DynamoDB for locking while still preventing concurrent operations.
---
### ❌ Problem 3: Backend Initialisation Error After Bucket Name Typo

```
Error: Failed to get existing workspaces: S3 bucket does not exist
```

**What happened:** I had a typo in the bucket name inside the `backend "s3"` block — it did not match the actual bucket name.

**Fix:** Double-checked the exact bucket name in the AWS Console, corrected the spelling in `main.tf`, and ran `terraform init` again.

---

## Key Takeaways

- **Local state breaks the moment two people work on the same project** — concurrent runs, lost state, and exposed secrets
- **The bootstrap problem** means you cannot use Terraform to create its own remote backend — you must create the S3 bucket and DynamoDB table first with a separate config
- **S3 versioning** lets you recover from accidental state corruption or deletion
- **Encryption** protects sensitive values stored in the state file
- **State locking** with DynamoDB or user_Lockfile prevents two engineers from running apply at the same time and corrupting state
- **Never commit state to Git** — use remote backends and add `.gitignore` entries from day one

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
