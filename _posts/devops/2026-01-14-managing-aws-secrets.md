---
layout: post
title: 'How to Manage Secrets Securely with SOPS and AWS KMS'
subtitle: 'AWS and Terraform'
date: 2026-01-14
author: 'Theo Cha'
tags:
  - devops
---

Managing secrets like API keys, database passwords, and encryption keys is one of the most critical challenges in software development. This guide walks you through setting up a secure, auditable, and team-friendly secrets management system using SOPS and AWS KMS.

## The Problem with Traditional Secrets Management

Most teams start with one of these approaches:

1. **Environment variables** - Easy to leak, hard to track changes
2. **Shared password managers** - No version control, manual sync
3. **Plain text in private repos** - One breach exposes everything
4. **AWS Secrets Manager only** - No code review for changes, no Git history

Each has significant drawbacks for growing teams.

## A Better Approach: SOPS + AWS KMS + Git

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           The Solution                                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────┐
│        AWS KMS            │
│   (Master Encryption Key) │
└─────────┬─────────────────┘
          │
encrypt/decrypt requests with SOPS
          │
          ▼
  ┌─────────┐    push      ┌──────────────┐   terraform   ┌─────────────────┐
  │ Editor  │ ◄──────────► │ .enc.json    │ ────apply───► │ Secrets Manager │
  │ (plain) │              │ (encrypted)  │               │ (for apps)      │
  └─────────┘              └──────────────┘               └─────────────────┘
       │                          │                              │
       ▼                          ▼                              ▼
  In memory only            Stored in Git                 Apps read from here
  (never on disk)           (safe to commit)
```

### Why This Works

| Benefit            | Description                               |
| ------------------ | ----------------------------------------- |
| Version controlled | Full Git history of all secret changes    |
| Code review        | PRs show which keys changed (not values)  |
| Access control     | IAM controls who can decrypt              |
| Audit trail        | CloudTrail logs every KMS access          |
| Easy revocation    | Remove IAM access = instant revoke        |
| No plain text      | Secrets never written to disk unencrypted |

## Step 1: Create a KMS Key

First, create a dedicated KMS key for secrets encryption:

```bash
aws kms create-key --description "Secrets encryption key for SOPS"
```

Create an alias for easier reference:

```bash
aws kms create-alias \
  --alias-name alias/sops-secrets \
  --target-key-id <key-id-from-above>
```

## Step 2: Set Up IAM Permissions

Create an IAM policy for team members who need access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID"
    }
  ]
}
```

Attach this policy to:

- Individual IAM users who need to edit secrets
- CI/CD roles that deploy secrets

## Step 3: Install SOPS

On macOS:

```bash
brew install sops
```

On Linux:

```bash
curl -LO https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64
chmod +x sops-v3.8.1.linux.amd64
sudo mv sops-v3.8.1.linux.amd64 /usr/local/bin/sops
```

## Step 4: Create Repository Structure

```bash
mkdir secrets-management
cd secrets-management
git init
```

Create the SOPS configuration file `.sops.yaml`:

```yaml
creation_rules:
  - path_regex: secrets/.*\.enc\.json$
    kms: arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID
```

Create `.gitignore`:

```
# Decrypted files (should never exist, but just in case)
*.dec.json
*.dec.yaml

# Terraform
.terraform/
*.tfstate
*.tfstate.backup
```

Create the secrets directory:

```bash
mkdir -p secrets
```

## Step 5: Create Your First Encrypted Secret

```bash
sops secrets/development.enc.json
```

This opens your editor. Add your secrets:

```json
{
  "DATABASE_URL": "postgres://user:pass@localhost:5432/db",
  "API_KEY": "your-secret-api-key",
  "JWT_SECRET": "your-jwt-secret"
}
```

Save and close. SOPS encrypts automatically.

### Using VS Code

```bash
EDITOR="code --wait" sops secrets/development.enc.json
```

## Step 6: Set Up Terraform for Deployment

Create `main.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "secrets/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    sops = {
      source  = "carlpett/sops"
      version = "~> 1.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "sops" {}

# Read encrypted file
data "sops_file" "api_development" {
  source_file = "secrets/development.enc.json"
}

# Create secret in AWS Secrets Manager
resource "aws_secretsmanager_secret" "api_development" {
  name = "api-secrets/development"

  tags = {
    ManagedBy = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "api_development" {
  secret_id     = aws_secretsmanager_secret.api_development.id
  secret_string = data.sops_file.api_development.raw
}
```

## Step 7: Deploy

```bash
terraform init
terraform plan
terraform apply
```

## The Developer Workflow

### Updating a Secret

```bash
# 1. Create a branch
git checkout -b update-api-key

# 2. Edit the secret
EDITOR="code --wait" sops secrets/development.enc.json

# 3. Commit and push
git add secrets/
git commit -m "Rotate API key for payment provider"
git push -u origin update-api-key

# 4. Create PR for review

# 5. After merge, apply
git checkout main
git pull
terraform apply
```

### What Reviewers See

In the PR diff, reviewers see:

```diff
  "sops": {
-   "lastmodified": "2024-01-15T10:00:00Z",
+   "lastmodified": "2024-01-16T14:30:00Z",
    ...
  },
- "API_KEY": "ENC[AES256_GCM,data:abc123...,type:str]",
+ "API_KEY": "ENC[AES256_GCM,data:xyz789...,type:str]",
```

They can see:

- Which keys changed
- When it changed
- Who changed it

They cannot see:

- Actual secret values

## Security Best Practices

### 1. Separate Keys by Environment

Use different KMS keys for different environments:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/development/.*\.enc\.json$
    kms: arn:aws:kms:us-east-1:ACCOUNT:key/DEV_KEY_ID
  - path_regex: secrets/production/.*\.enc\.json$
    kms: arn:aws:kms:us-east-1:ACCOUNT:key/PROD_KEY_ID
```

### 2. Limit Production Access

Only give production KMS access to:

- Senior engineers
- CI/CD deployment roles

### 3. Enable KMS Key Rotation

```bash
aws kms enable-key-rotation --key-id KEY_ID
```

### 4. Monitor with CloudTrail

All KMS operations are logged. Set up alerts for:

- Failed decryption attempts
- Unusual access patterns
- Access from new IP addresses

## Revoking Access

When someone leaves the team:

```bash
# Remove their IAM policy attachment
aws iam detach-user-policy \
  --user-name departed-user \
  --policy-arn arn:aws:iam::ACCOUNT:policy/sops-kms-access
```

Immediately, they can no longer decrypt secrets. No need to:

- Rotate the KMS key
- Re-encrypt all secrets
- Change any passwords

## Comparison: Before and After

| Aspect         | Before (Plain Text)  | After (SOPS + KMS)   |
| -------------- | -------------------- | -------------------- |
| Git storage    | Unsafe               | Safe (encrypted)     |
| Code review    | Shows actual secrets | Shows only key names |
| Access control | All or nothing       | Fine-grained IAM     |
| Audit          | None                 | Full CloudTrail logs |
| Offboarding    | Rotate everything    | Remove IAM access    |
| Compliance     | Fails audits         | Passes audits        |

## Cost

| Component       | Monthly Cost                |
| --------------- | --------------------------- |
| KMS key         | ~$1/month per key           |
| KMS requests    | ~$0.03 per 10,000 requests  |
| Secrets Manager | ~$0.40 per secret per month |

For most teams: **Under $10/month total.**

## Conclusion

SOPS + AWS KMS gives you:

1. **Git-native workflow** - Version control, PRs, and code review
2. **Strong encryption** - AWS KMS with audit trails
3. **Simple access control** - IAM policies
4. **Easy revocation** - Remove IAM access instantly
5. **Compliance ready** - Audit logs and access controls

The initial setup takes about an hour. The long-term benefits—security, auditability, and team workflow—are well worth it.

## Quick Reference

| Task               | Command                                           |
| ------------------ | ------------------------------------------------- |
| Create/edit secret | `EDITOR="code --wait" sops secrets/file.enc.json` |
| View secret        | `sops -d secrets/file.enc.json`                   |
| Deploy to AWS      | `terraform apply`                                 |
| Rotate KMS key     | `aws kms enable-key-rotation --key-id KEY_ID`     |

---

_This guide covers the essential setup. For production deployments, consider adding CI/CD automation, multi-region replication, and automated secret rotation._
