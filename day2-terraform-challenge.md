# How I Set Up Terraform and AWS on My Computer

*Posted as part of the #30DayTerraformChallenge*

---

## Hey there!

If you are new to Terraform or cloud infrastructure, this post is for you.

Today was all about getting my computer ready to work with Terraform and AWS. No actual infrastructure was built yet — but by the end of the day, my machine was fully set up and ready to go.

Let me walk you through exactly what I did, in plain English.

---

## First Things First — What Are We Even Setting Up?

Before I dive in, here is a quick breakdown of the tools and why each one matters:

| Tool | What It Is | Why You Need It |
|---|---|---|
| **AWS Account** | Your account on Amazon's cloud platform | Where your infrastructure will live |
| **IAM User** | A safe, limited account inside AWS | So you don't use the all-powerful root account |
| **AWS CLI** | A tool that lets your terminal talk to AWS | Terraform uses this to connect to your AWS account |
| **Terraform** | The main tool we are learning | Lets you build cloud infrastructure using code |
| **VS Code** | A code editor | Where you will write your Terraform files |

Think of it like this: AWS is the building, Terraform is the architect's blueprint tool, and the AWS CLI is the phone line between your computer and the building.

---

## Creating an IAM User for Terraform

Here is something important that I learned today:

> **Never use your root AWS account with Terraform.**

Your root account has access to absolutely everything. If those credentials ever leak — even by accident — someone could delete everything you have built, rack up thousands of dollars in charges, or worse.

So instead, I created a separate IAM user called `terraform-dev`. Think of it as a dedicated work account that only does Terraform things.

### How I Created It

1. Go to **IAM** in the AWS Console
2. Click **Users** → **Create user**
3. Name it something like `terraform-dev`
4. Give it **Programmatic access** — this creates an Access Key and Secret Key
5. Attach the **AdministratorAccess** permission for now
6. Download the credentials CSV file immediately

> ⚠️ Save those credentials somewhere safe like a password manager. You will not be able to see the Secret Key again after closing that screen.

---

## Installing the AWS CLI

The AWS CLI lets your computer communicate with AWS directly from the terminal.

### How I Installed It

**On macOS:**
```bash
brew install awscli
```

**On Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**On Windows:**

Download the installer from the AWS website and run it.

### How I Connected It to My AWS Account

After installing, I ran this command:

```bash
aws configure
```

It asked me four questions:

```
AWS Access Key ID: (paste your key here)
AWS Secret Access Key: (paste your secret here)
Default region name: us-east-1
Default output format: json
```

I used `us-east-1` as my region. It is the most widely supported AWS region and works well for learning.

### How I Checked It Was Working

```bash
aws sts get-caller-identity
```

This command asks AWS "who am I logged in as?" If everything is set up correctly, you will see something like this:

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/terraform-dev"
}
```

See `user/terraform-dev` at the end? That means I am logged in as my IAM user — not root. 

---

## Installing Terraform

This is the main tool. Here is how I installed it.

**On macOS:**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

**On Linux:**
```bash
sudo apt update && sudo apt install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor \
  -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform
```

**On Windows:**
```bash
choco install terraform
```

### Checking It Works

```bash
terraform version
```

You should see something like:

```
Terraform v1.7.5
on linux_amd64
```

If you see a version number — you are good. Terraform is installed. 

---

## Setting Up VS Code

VS Code is the code editor I use to write Terraform files. I installed two extensions that make writing Terraform much easier:

**1. HashiCorp Terraform**
- Gives you colour-coded Terraform syntax
- Auto-completes as you type
- Underlines errors before you even run anything

**2. AWS Toolkit**
- Lets you browse your AWS resources from inside VS Code
- Useful for checking what is actually deployed in your account

To install them:
- Open VS Code
- Press `Ctrl + Shift + X`
- Search for each one by name
- Click Install

---

## Final Check — Does Everything Work Together?

I ran all four of these commands to confirm my full environment was working:

```bash
terraform version
```
```
Terraform v1.7.5
```

```bash
aws --version
```
```
aws-cli/2.15.30 Python/3.11.8
```

```bash
aws sts get-caller-identity
```
```json
{
    "Account": "XXXXXXXXXXXX",
    "Arn": "arn:aws:iam::XXXXXXXXXXXX:user/terraform-dev"
}
```

```bash
aws configure list
```
```
   profile    <not set>
access_key    ****************XXXX    shared-credentials-file
secret_key    ****************XXXX    shared-credentials-file
    region    us-east-1               config-file
```

All four working cleanly = environment is ready. 🎉

---

## Problems I Ran Into

I want to be honest — setup is never perfectly smooth. Here is what went wrong for me and how I fixed it.

**Problem:** After installing the AWS CLI, the command `aws` was not found.

**What happened:** The installation finished but the terminal did not know where to find it.

**Fix:** I added the install location to my PATH and restarted the terminal:
```bash
source ~/.bashrc
```

---

**Problem:** `aws sts get-caller-identity` gave me an authentication error.

**What happened:** I had copied the wrong secret key from the CSV file.

**Fix:** I just ran `aws configure` again and carefully re-entered my credentials.

---

## What I Learned Today

- Never use your AWS root account for day-to-day work — always create an IAM user
- MFA is not optional — it is the bare minimum for account security
- Terraform finds your AWS credentials through the AWS CLI configuration
- The `aws sts get-caller-identity` command is the quickest way to confirm everything is connected correctly

---

iCorp User Group, and EveOps communities.*
