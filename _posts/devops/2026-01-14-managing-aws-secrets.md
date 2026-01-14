---
layout: post
title: 'Managing AWS Secrets with Terraform and Git-Crypt'
subtitle: 'AWS and Terraform'
date: 2024-03-20
author: 'Theo Cha'
tags:
  - devops
---

Managing secrets in a team environment is challenging. You need version control, audit trails, and security - all while keeping things simple enough for developers to use daily.

This post explains how we set up a secure secrets management system using **Terraform**, **AWS Secrets Manager**, and **git-crypt**.

## The Problem

| Challenge                              | Impact                                |
| -------------------------------------- | ------------------------------------- |
| Secrets edited manually in AWS console | No history of who changed what        |
| No review process                      | Mistakes go straight to production    |
| No backup                              | Accidentally deleted secrets are gone |
| Scattered across services              | Hard to audit or rotate               |

## The Solution

A dedicated private repository that:

- Stores secrets as JSON files
- Encrypts them automatically with git-crypt
- Uses Terraform to sync to AWS Secrets Manager
- Requires PR reviews for all changes

```
Local JSON files → git-crypt encrypts → GitHub (encrypted) → Terraform → AWS Secrets Manager
```

## Architecture Overview

```
aws-secrets/
├── backend.hcl          # S3 backend configuration
├── terraform.tfvars     # Variables (encrypted by git-crypt)
├── main.tf              # AWS provider
├── variables.tf         # Variable definitions
├── secrets.tf           # Dynamic secret creation
├── outputs.tf           # Secret ARNs
└── secrets/             # All secret JSON files (encrypted)
    ├── app-name/
    │   ├── development.json
    │   └── production.json
    └── another-app/
        └── api-keys.json
```

## Setting Up Git-Crypt

Git-crypt transparently encrypts files when pushing and decrypts when pulling. Only team members with the key can read the secrets.

### Installation

```bash
# macOS
brew install git-crypt

# Ubuntu
sudo apt-get install git-crypt
```

### Initialize in Your Repo

```bash
cd aws-secrets
git-crypt init

# Export the key - store this securely (1Password, etc.)
git-crypt export-key ./git-crypt-key
```

### Configure Which Files to Encrypt

Create `.gitattributes`:

```
*.tfvars filter=git-crypt diff=git-crypt
secrets/**/*.json filter=git-crypt diff=git-crypt
```

Now any `.tfvars` files and JSON files in `secrets/` are automatically encrypted.

## Terraform Configuration

### Dynamic Secret Discovery

The key insight: use `fileset()` to automatically discover all JSON files. No need to manually define each secret.

**secrets.tf**:

```hcl
locals {
  secret_files = fileset("${path.module}/secrets", "**/*.json")

  secrets = {
    for file in local.secret_files :
    trimsuffix(file, ".json") => {
      content = file("${path.module}/secrets/${file}")
    }
  }
}

resource "aws_secretsmanager_secret" "this" {
  for_each = local.secrets

  name        = each.key
  description = "Managed by Terraform"

  tags = {
    ManagedBy   = "terraform"
    Environment = var.environment
    Owner       = var.owner
  }
}

resource "aws_secretsmanager_secret_version" "this" {
  for_each = local.secrets

  secret_id     = aws_secretsmanager_secret.this[each.key].id
  secret_string = each.value.content
}
```

**How it works:**

- `secrets/myapp/production.json` becomes AWS secret `myapp/production`
- Add a new JSON file, run `terraform apply` - done
- No code changes needed for new secrets

### Backend Configuration

Store Terraform state in S3 with encryption and locking:

**backend.hcl**:

```hcl
bucket         = "your-terraform-state-bucket"
key            = "secrets/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-lock"
encrypt        = true
```

Initialize with:

```bash
terraform init -backend-config=backend.hcl
```

## Daily Workflow

### Adding a New Secret

1. Create a JSON file in `secrets/`:

```bash
mkdir -p secrets/my-new-app
```

Create `secrets/my-new-app/production.json`:

```json
{
  "DATABASE_URL": "postgres://...",
  "API_KEY": "sk_live_..."
}
```

2. Commit and create PR:

```bash
git checkout -b add-my-new-app-secrets
git add .
git commit -m "Add my-new-app production secrets"
git push origin add-my-new-app-secrets
```

3. After PR approval, apply:

```bash
terraform plan
terraform apply
```

### Updating an Existing Secret

1. Edit the JSON file
2. Commit, push, create PR
3. After approval: `terraform apply`

### Importing Existing AWS Secrets

If secrets already exist in AWS:

```bash
# Import the secret
terraform import 'aws_secretsmanager_secret.this["app/production"]' 'app/production'

# Get version ID
vid=$(aws secretsmanager get-secret-value --secret-id app/production --query 'VersionId' --output text)

# Import the version
terraform import 'aws_secretsmanager_secret_version.this["app/production"]' "app/production|$vid"
```

Then sync the local file from AWS:

```bash
aws secretsmanager get-secret-value \
  --secret-id app/production \
  --query 'SecretString' \
  --output text | jq . > secrets/app/production.json
```

## Team Onboarding

When a new team member joins:

1. Add them to the private GitHub repository
2. Share the `git-crypt-key` file securely (via 1Password or similar)
3. They clone and unlock:

```bash
git clone git@github.com:your-org/aws-secrets.git
cd aws-secrets

# Files appear as binary gibberish until unlocked
git-crypt unlock /path/to/git-crypt-key

# Now files are readable
```

After unlocking once, all future `git pull` operations auto-decrypt.

## Accessing Secrets in Applications

### AWS CLI

```bash
aws secretsmanager get-secret-value \
  --secret-id myapp/production \
  --query 'SecretString' \
  --output text | jq .
```

### Node.js

```javascript
const { SecretsManager } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManager({ region: 'us-east-1' });

async function getSecret(secretId) {
  const response = await client.getSecretValue({ SecretId: secretId });
  return JSON.parse(response.SecretString);
}

const secrets = await getSecret('myapp/production');
console.log(secrets.DATABASE_URL);
```

### Kubernetes (External Secrets Operator)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: myapp-secrets
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: myapp/production
        property: DATABASE_URL
```

## Security Considerations

### What's Protected

- **At rest in GitHub**: Files encrypted by git-crypt
- **In transit**: HTTPS for GitHub, TLS for AWS
- **In S3 state**: `encrypt = true` in backend config
- **In AWS**: Secrets Manager handles encryption

### Best Practices

1. **Rotate the git-crypt key** when team members leave
2. **Use separate AWS accounts** for dev/prod if possible
3. **Enable CloudTrail** for AWS Secrets Manager audit logs
4. **Review PRs carefully** - the diff shows actual secret values to reviewers
5. **Never commit the git-crypt key** - it's in `.gitignore`

### State File Contains Secrets

The Terraform state file contains secret values in plain text. Ensure:

- S3 bucket has encryption enabled
- Bucket access is restricted
- Consider using a production S3 bucket for state (stricter IAM)

---

_Have questions or improvements? Feel free to reach out._
