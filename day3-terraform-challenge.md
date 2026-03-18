# Deploying Your First Server with Terraform: A Beginner's Guide

*Day 3 of the #30DayTerraformChallenge*

---

## Hello again!

Today was the day everything became real.

No more just reading about Terraform. No more setup. Today I actually used Terraform to deploy a live server on AWS — and I watched it come to life right in my terminal.

In this post I will walk you through exactly what I did, explain every line of code in plain English, and share the mistakes I made along the way.

Let's go.

---

## What Are We Building Today?

Here is a simple picture of what we are deploying:

```
Your Browser
     |
     | (HTTP on Port 80)
     |
[Security Group] ← controls what traffic is allowed in
     |
[EC2 Instance]   ← your actual server running on AWS
     |
  [AWS Cloud - us-east-1]
```

By the end of this, you will have a real web server running on AWS that you can visit in your browser.

---

## Two Things You Must Understand First

Before we write any code, there are two Terraform building blocks you need to know. Everything in Terraform is built on these two things.

### 1. The Provider Block

A **provider** tells Terraform which cloud platform you are using.

Think of it like telling a delivery company which country you want to ship to. Without knowing the destination, they cannot do anything.

```hcl
provider "aws" {
  region = "us-east-1"
}
```

This tells Terraform: "I am working with AWS, and I want everything in the us-east-1 region."

### 2. The Resource Block

A **resource** is the actual thing you want to create — a server, a database, a security group, anything.

Think of it like placing an order. You tell Terraform exactly what you want, and it goes and creates it.

```hcl
resource "aws_instance" "my_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

This tells Terraform: "Create an EC2 server using this machine image, and make it a t2.micro (which is free tier eligible)."

---

## The Full Terraform Code

Here is everything I wrote in my `main.tf` file. I will explain each section right after.

```hcl
# Tell Terraform we are using AWS in us-east-1
provider "aws" {
  region = "us-east-1"
}

# Create a security group to allow web traffic
resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Allow HTTP traffic from the internet"

  # Allow anyone to access port 80 (regular web traffic)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow the server to reach the internet
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Create the actual web server
resource "aws_instance" "web_server" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  # This script runs automatically when the server starts
  user_data = <<-EOF
              #!/bin/bash
              echo "<h1>Hello from Terraform!</h1>" > index.html
              nohup python3 -m http.server 80 &
              EOF

  tags = {
    Name = "MyFirstTerraformServer"
  }
}

# Show us the server's public IP when done
output "public_ip" {
  value       = aws_instance.web_server.public_ip
  description = "The public IP of the web server"
}
```

---

## Let Me Explain Each Part

### The Provider Block

```hcl
provider "aws" {
  region = "us-east-1"
}
```

This is always the starting point. It tells Terraform you are working with AWS and sets the region where everything will be created.

---

### The Security Group

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Allow HTTP traffic from the internet"

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
```

A security group is like a firewall for your server. It controls who can connect to it and what kind of traffic is allowed.

- **ingress** = traffic coming IN to the server
- **egress** = traffic going OUT from the server
- **port 80** = the standard port for websites
- **0.0.0.0/0** = allow connections from anywhere on the internet

---

### The EC2 Instance (Your Server)

```hcl
resource "aws_instance" "web_server" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              echo "<h1>Hello from Terraform!</h1>" > index.html
              nohup python3 -m http.server 80 &
              EOF

  tags = {
    Name = "MyFirstTerraformServer"
  }
}
```

Breaking this down:

- **ami** — This is the operating system image. Think of it like choosing which version of Windows or Linux to install. This one is Amazon Linux 2.
- **instance_type** — The size of the server. `t3.micro` is free tier eligible — it will not cost you anything.
- **vpc_security_group_ids** — This attaches the security group we created above to this server.
- **user_data** — This is a script that runs automatically the moment the server starts up. It creates a simple HTML page and starts a web server on port 80.
- **tags** — Just a label so you can find this server easily in the AWS Console.

---

### The Output

```hcl
output "public_ip" {
  value = aws_instance.web_server.public_ip
}
```

After Terraform finishes creating everything, this prints the public IP address of your server directly in the terminal. Without this, you would have to go into the AWS Console to find it.

---

## Running the Terraform Commands

Now for the exciting part. There are three commands you run every time you deploy with Terraform.

### Command 1: `terraform init`

```bash
terraform init
```

This is always the first command. It downloads the AWS provider plugin that Terraform needs to talk to AWS.

```
Initializing the backend...
Initializing provider plugins...
- Installing hashicorp/aws v5.0.0...

Terraform has been successfully initialized!
```

> Think of this like installing the dependencies for a project before running it.

---

### Command 2: `terraform plan`

```bash
terraform plan
```

This is Terraform's preview step. It shows you exactly what it is going to create, change, or delete — **before** it actually does anything.

```
Terraform will perform the following actions:

  # aws_instance.web_server will be created
  + resource "aws_instance" "web_server" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t2.micro"
      ...
    }

  # aws_security_group.web_sg will be created
  + resource "aws_security_group" "web_sg" {
      ...
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

Green `+` means something will be created. Always read the plan before applying. It is your safety net.

---

### Command 3: `terraform apply`

```bash
terraform apply
```

This is where Terraform actually goes and builds everything. It will show you the plan one more time and ask you to confirm by typing `yes`.

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_security_group.web_sg: Creating...
aws_security_group.web_sg: Creation complete after 2s
aws_instance.web_server: Creating...
aws_instance.web_server: Still creating... [10s elapsed]
aws_instance.web_server: Creation complete after 32s

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:
public_ip = "34.235.114.181"
```

See that IP address at the bottom? That is your live server.

---

## Confirming the Server Works

After apply finished, I copied the IP address from the output and pasted it into my browser:

```
http://34.235.114.181/
```

And there it was:

```
Hello from Terraform!
```

A real web page, running on a real server, that I built with code. That feeling never gets old.

---

## Cleaning Up — Always Do This!

When you are done, destroy everything so you do not get charged:

```bash
terraform destroy
```

Type `yes` to confirm. Terraform will delete everything it created.

```
Destroy complete! Resources: 2 destroyed.
```

> Get into this habit from Day 1. Unused cloud resources cost real money.

---

## Problems I Ran Into

This is the part most tutorials skip. Here are the real errors I hit and exactly how I fixed them.

---

### ❌ Problem 1: Free Tier Error

Right when I ran `terraform apply`, I got this:

```
Error: The specified instance type is not eligible for Free Tier
```

**What happened:** I had used `t2.micro` as my instance type, assuming it was always free tier eligible. But in some AWS regions or newer accounts, `t2.micro` is no longer listed under free tier — `t3.micro` is the replacement.

**Fix:** I opened my `main.tf` and changed one line:

```hcl
# Before
instance_type = "t2.micro"

# After
instance_type = "t3.micro"
```

Then I ran `terraform apply` again and it worked. A one word change — but it took me a while to figure out why the error was happening in the first place.

> 💡 **Tip for others:** If you hit this error, check the [AWS Free Tier page](https://aws.amazon.com/free/) to confirm which instance types are eligible in your region. It varies.

---

### ❌ Problem 2: Provider Plugin Timeout

When I first ran `terraform init`, it got stuck and eventually showed this:

```
Failed to load plugin schemas
```

**What happened:** The Terraform provider download timed out midway through. This happens when your internet connection drops or slows down while `terraform init` is trying to pull down the AWS provider plugin from the HashiCorp registry.

**Fix:** Two things sorted it out:

1. I re-ran `terraform init` — Terraform picks up where it left off
2. I made sure I had a stable internet connection before running it again

```bash
terraform init
```

```
Initializing provider plugins...
- Installing hashicorp/aws v5.0.0...
- Installed hashicorp/aws v5.0.0

Terraform has been successfully initialized!
```

It completed cleanly the second time.

> 💡 **Tip for others:** If `terraform init` fails with a network-related error, just re-run it. You do not need to delete anything or start over. Also avoid running it on a weak or shared WiFi connection — the plugin download is a few hundred megabytes.

---

## What I Learned Today

- The **provider block** tells Terraform which cloud platform to use
- The **resource block** describes the actual infrastructure you want to create
- `terraform plan` is your best friend — always review it before applying
- The **user_data** script is how you configure a server automatically when it boots
- Security groups act like a firewall — without opening port 80, no one can reach your web server
- Always run `terraform destroy` when you are done experimenting

---

## My Architecture Diagram

Here is a simple view of what I deployed today:

```
Internet
   |
   | HTTP (Port 80)
   |
[Security Group: web-server-sg]
   |
   |--- Allows port 80 inbound from 0.0.0.0/0
   |--- Allows all outbound traffic
   |
[EC2 Instance: t2.micro]
   |--- Amazon Linux 2
   |--- Runs a Python HTTP server on port 80
   |--- Serves a simple HTML page
   |
[AWS Region: us-east-1]
```
---
*Part of the #30DayTerraformChallenge with AWS AI/ML UserGroup Kenya, Meru HashiCorp User Group, and EveOps.*
