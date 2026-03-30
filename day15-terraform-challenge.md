# Deploying Multi-Cloud Infrastructure with Terraform Modules

*Day 15 of the #30DayTerraformChallenge*

---

## Hello

Today was the most advanced day of the challenge so far.

Building on yesterday's provider aliases, today I tackled three bigger challenges:

- **Modules that accept provider configurations from their callers** — the correct pattern for multi-region modules
- **Docker containers managed by Terraform** — using a non-AWS provider
- **A full EKS Kubernetes cluster** — provisioned entirely through Terraform code

By the end of the day I had a containerised nginx application running on Kubernetes — deployed without a single manual console click.

Let me walk through all three.

---

## Part 1 — Modules That Work with Multiple Providers

### The Problem with Providers Inside Modules

Yesterday I used provider aliases to deploy resources to multiple regions. Today I learned a critical rule:

> **Modules must never define their own provider blocks when the provider needs to be aliased.**

Here is why. If a module contains its own `provider "aws"` block, it is locked to that specific configuration. The caller cannot override it. The module becomes inflexible — it always deploys to the same region, same account, same configuration.

The correct pattern is for the module to **receive** provider configurations from the caller — not define its own.

### The configuration_aliases Declaration

To tell Terraform which provider aliases a module expects to receive, you add `configuration_aliases` to the module's `required_providers` block:

```hcl
# modules/multi-region-app/main.tf

terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 5.0"
      configuration_aliases = [aws.primary, aws.replica]
    }
  }
}

# No provider blocks anywhere in this file
# The caller provides them
```

`configuration_aliases = [aws.primary, aws.replica]` tells Terraform:
- This module uses the `aws` provider
- It expects two aliased configurations: one called `aws.primary` and one called `aws.replica`
- The caller is responsible for providing both

### Resources Inside the Module

Resources inside the module reference the expected aliases directly:

```hcl
variable "app_name" {
  description = "Name prefix for all resources"
  type        = string
}

resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "${var.app_name}-primary-2026"

  tags = {
    Name = "${var.app_name}-primary"
  }
}

resource "aws_s3_bucket" "replica" {
  provider = aws.replica
  bucket   = "${var.app_name}-replica-2026"

  tags = {
    Name = "${var.app_name}-replica"
  }
}

output "primary_bucket_arn" {
  value       = aws_s3_bucket.primary.arn
  description = "ARN of the primary bucket"
}

output "replica_bucket_arn" {
  value       = aws_s3_bucket.replica.arn
  description = "ARN of the replica bucket"
}
```

The module does not know what regions `aws.primary` and `aws.replica` point to. That is intentional — the caller decides.

### The Calling Root Configuration

The root configuration defines the actual providers and wires them to the module using the `providers` map:

```hcl
# live/main.tf

provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "replica"
  region = "us-west-2"
}

module "multi_region_app" {
  source   = "../../modules/multi-region-app"
  app_name = "my-terraform-app"

  providers = {
    aws.primary = aws.primary   # wire root's aws.primary → module's aws.primary
    aws.replica = aws.replica   # wire root's aws.replica → module's aws.replica
  }
}

output "primary_bucket_arn" {
  value = module.multi_region_app.primary_bucket_arn
}

output "replica_bucket_arn" {
  value = module.multi_region_app.replica_bucket_arn
}
```

The `providers` map is the wiring between the root configuration and the module. The left side is the alias the module expects. The right side is the provider from the root configuration that satisfies it.

After `terraform apply`:

```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:
primary_bucket_arn = "arn:aws:s3:::my-terraform-app-primary-2026"
replica_bucket_arn = "arn:aws:s3:::my-terraform-app-replica-2026"
```

One bucket appears in `us-east-1`. The other appears in `us-west-2`. Same module, different regions, controlled entirely by the caller.

---

## Part 2 — Docker Containers with Terraform

Before tackling EKS, I practiced with the Docker provider locally. This is a great way to get comfortable with container management in Terraform without incurring AWS costs.

### Setup

Make sure Docker is running on your machine before starting. The Docker provider connects to your local Docker daemon.

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

# Pull the nginx image
resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

# Run the container
resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = "terraform-nginx"

  ports {
    internal = 80
    external = 8080
  }
}

output "container_name" {
  value       = docker_container.nginx.name
  description = "Name of the running container"
}
```

### Running It

```bash
terraform init
terraform apply
```

```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:
container_name = "terraform-nginx"
```

Confirm it is running:

```bash
docker ps
```

```
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                  NAMES
a1b2c3d4e5f6   nginx:latest   "/docker-entrypoint.…"   5 seconds ago   Up 4 seconds   0.0.0.0:8080->80/tcp   terraform-nginx
```

Visit `http://localhost:8080` in your browser and you see the nginx welcome page — a container managed entirely by Terraform.

Destroy when done:

```bash
terraform destroy
```

This is an excellent pattern for local development — you can define your application's container configuration in Terraform and keep it consistent with production.

---

## Part 3 — EKS Cluster with Terraform

This is the most complex deployment of the challenge. An EKS cluster involves multiple AWS services working together — the control plane, worker nodes, VPC networking, IAM roles, and security groups.

The good news: the official `terraform-aws-modules/eks` module handles all the complexity. You declare what you want, the module figures out how to build it.

> ⚠️ **Cost warning:** An EKS cluster costs approximately $0.10/hour for the control plane plus EC2 costs for worker nodes. Destroy the cluster as soon as you have confirmed it works. Leaving it running for 24 hours costs roughly $5–8.

### Step 1 — Create the VPC

EKS needs a VPC with public and private subnets. Use the official VPC module:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraform-eks-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    "kubernetes.io/cluster/terraform-challenge-cluster" = "shared"
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/terraform-challenge-cluster" = "shared"
    "kubernetes.io/role/internal-elb"                   = "1"
  }

  public_subnet_tags = {
    "kubernetes.io/cluster/terraform-challenge-cluster" = "shared"
    "kubernetes.io/role/elb"                            = "1"
  }
}
```

### Step 2 — Deploy the EKS Cluster

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "terraform-challenge-cluster"
  cluster_version = "1.29"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    default = {
      min_size       = 1
      max_size       = 3
      desired_size   = 2
      instance_types = ["t3.small"]
    }
  }

  tags = {
    Environment = "dev"
    Challenge   = "30DayTerraform"
  }
}
```

### Step 3 — Configure the Kubernetes Provider

After the EKS cluster is provisioned, configure the Kubernetes provider to deploy workloads onto it:

```hcl
provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}
```

The `exec` block is how the Kubernetes provider authenticates. Instead of using a static token, it runs `aws eks get-token` to get a short-lived authentication token every time it needs to talk to the cluster. This is the recommended approach because it uses your existing AWS credentials — no separate Kubernetes credentials to manage.

### Step 4 — Deploy a Workload onto the Cluster

```hcl
resource "kubernetes_deployment" "nginx" {
  metadata {
    name = "nginx-deployment"
    labels = {
      app = "nginx"
    }
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "nginx"
      }
    }

    template {
      metadata {
        labels = {
          app = "nginx"
        }
      }

      spec {
        container {
          image = "nginx:latest"
          name  = "nginx"

          port {
            container_port = 80
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "nginx" {
  metadata {
    name = "nginx-service"
  }

  spec {
    selector = {
      app = "nginx"
    }

    port {
      port        = 80
      target_port = 80
    }

    type = "LoadBalancer"
  }
}

output "cluster_name" {
  value       = module.eks.cluster_name
  description = "EKS cluster name"
}

output "cluster_endpoint" {
  value       = module.eks.cluster_endpoint
  description = "EKS cluster API endpoint"
}
```

### Running It

```bash
terraform init
terraform apply
```

The apply takes 10–15 minutes. Most of that time is EKS provisioning the control plane. After it completes:

```
Apply complete! Resources: 52 added, 0 changed, 0 destroyed.

Outputs:
cluster_name     = "terraform-challenge-cluster"
cluster_endpoint = "https://ABCDEF1234567890.gr7.us-east-1.eks.amazonaws.com"
```

Update your kubeconfig and confirm the pods are running:

```bash
aws eks update-kubeconfig --name terraform-challenge-cluster --region us-east-1

kubectl get pods
```

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5d59d67564-7xk9p   1/1     Running   0          2m
nginx-deployment-5d59d67564-m8h4q   1/1     Running   0          2m
```

Two nginx pods running on Kubernetes — deployed entirely through Terraform.

### Destroy Immediately After

```bash
terraform destroy
```

This takes another 10–15 minutes. Do not skip this step — an EKS cluster left running overnight accumulates meaningful charges.

---

## Problems I Ran Into

### ❌ Problem 1: Module Missing `configuration_aliases`

When I first called the multi-region module and passed the `providers` map, I got:

```
Error: Provider configuration not present

To work with aws_s3_bucket.replica its original provider configuration
at provider["registry.terraform.io/hashicorp/aws"].replica is required,
but it has been removed. This occurs when a provider configuration is
removed while objects created by that provider still exist in the state.
```

**What happened:** I had not added `configuration_aliases` to the module's `required_providers` block. Terraform did not know the module expected to receive aliased providers.

**Fix:** Added `configuration_aliases` to the module's `terraform` block:

```hcl
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 5.0"
      configuration_aliases = [aws.primary, aws.replica]
    }
  }
}
```

After adding this, `terraform init` re-initialised the module correctly and the apply succeeded.

---

### ❌ Problem 2: Docker Provider Not Found

When I ran `terraform init` for the Docker configuration:

```
Error: Failed to query available provider packages

Could not retrieve the list of available versions for provider
kreuzwerker/docker: could not connect to the Terraform Registry.
```

**What happened:** I had a network issue connecting to the Terraform Registry. The Docker provider is published by `kreuzwerker` (not HashiCorp) so it requires an internet connection to download.

**Fix:** Checked my internet connection, then re-ran:

```bash
terraform init
```

It worked on the second attempt. If you are on a restricted network, you may need to configure a provider mirror.

---

### ❌ Problem 3: EKS Nodes Not Joining the Cluster

After the EKS cluster provisioned, `kubectl get nodes` showed no nodes:

```
No resources found.
```

**What happened:** The worker node IAM role was missing the `AmazonEKSWorkerNodePolicy` and `AmazonEKS_CNI_Policy` managed policies. Without these, the worker nodes could not authenticate with the control plane and join the cluster.

**Fix:** The `terraform-aws-modules/eks` module version `~> 20.0` handles this automatically when you use `eks_managed_node_groups`. The issue was I had initially tried to define node groups manually. Switching to the managed node group configuration inside the EKS module resolved it — all required IAM policies were attached automatically.

---

### ❌ Problem 4: Kubernetes Provider Timing Issue

After the EKS cluster was created, Terraform tried to create the Kubernetes deployment immediately — but the cluster was not fully ready yet:

```
Error: Post "https://ABCDEF.eks.amazonaws.com/apis/apps/v1/namespaces/default/deployments":
dial tcp: lookup ABCDEF.eks.amazonaws.com: no such host
```

**What happened:** The EKS cluster endpoint was provisioned but DNS had not propagated yet when Terraform tried to use the Kubernetes provider.

**Fix:** Added a `depends_on` to the Kubernetes deployment:

```hcl
resource "kubernetes_deployment" "nginx" {
  depends_on = [module.eks]
  # ...
}
```

This tells Terraform to wait for the entire EKS module to complete before creating Kubernetes resources. After adding this, the apply worked cleanly.

---

## What I Learned Today

- **Modules must not define their own aliased providers** — they should receive provider configurations from the caller via the `providers` map
- **`configuration_aliases`** declares which provider aliases a module expects — Terraform requires this to wire providers correctly
- **The `providers` map** in a module call wires root providers to module-expected aliases
- **The Docker provider** works like any other — declare it in `required_providers`, configure it, and manage containers as resources
- **EKS takes 10–15 minutes to provision** — plan your time accordingly and always destroy immediately after
- **The Kubernetes `exec` block** uses `aws eks get-token` for authentication — no static credentials needed
- **`depends_on` on Kubernetes resources** is needed to ensure the cluster is fully ready before workloads are deployed
- **EKS managed node groups** handle all required IAM policies automatically — manual node group configuration is more complex and error-prone

---

## EKS Cost Awareness

An EKS cluster creates and charges for:

| Resource | Approximate Cost |
|---|---|
| EKS Control Plane | $0.10/hour |
| EC2 Worker Nodes (2x t3.small) | ~$0.042/hour each |
| NAT Gateway | $0.045/hour + data transfer |
| Load Balancer | $0.022/hour |

**Total approximate cost: ~$0.25/hour or ~$6/day**

Leaving a cluster running overnight after this exercise costs real money. Always run `terraform destroy` as soon as you have confirmed your deployment works. The destroy takes 10–15 minutes but saves you from unexpected charges.

---

*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*

---
