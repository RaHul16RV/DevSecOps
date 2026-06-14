# HashiCorp Vault + GitHub OIDC + AWS Secrets Engine + Terraform

## Architecture

```text
GitHub Actions
      ↓
OIDC Token
      ↓
Vault (EC2)
      ↓
Vault Policy
      ↓
AWS Secrets Engine
      ↓
Temporary AWS Credentials
      ↓
Terraform Apply
      ↓
AWS Infrastructure
```

## Goal

- No AWS keys stored in GitHub
- GitHub authenticates using OIDC
- Vault validates GitHub identity
- Vault generates temporary AWS credentials
- Terraform uses temporary credentials

---

# 1. What is HashiCorp Vault?

HashiCorp Vault is a Secret Management Tool used to securely store and manage:

- AWS Credentials
- Database Passwords
- API Keys
- Tokens
- Certificates

Instead of storing secrets in:

- GitHub Secrets
- Jenkins Credentials
- Environment Variables
- Source Code

We store them in Vault.

---

# 2. Why Use Vault?

## Without Vault

```text
GitHub Actions
      ↓
AWS Access Keys
      ↓
Terraform
      ↓
AWS
```

Problems:

- Long-lived credentials
- Manual rotation
- Credential leakage risk

## With Vault

```text
GitHub Actions
      ↓
OIDC
      ↓
Vault
      ↓
Temporary AWS Credentials
      ↓
Terraform
      ↓
AWS
```

Benefits:

- No hardcoded secrets
- Automatic credential rotation
- Temporary credentials
- Better security

---

# 3. What is OIDC?

OIDC (OpenID Connect) is an authentication protocol.

GitHub Actions generates a temporary OIDC token.

```text
GitHub Actions
      ↓
OIDC Token
      ↓
Vault
```

Vault verifies:

- Who issued the token
- Repository name
- Branch
- Workflow identity

---

# 4. Vault Production Setup on EC2

## Launch EC2

### AMI
Ubuntu 22.04

### Instance Type
t2.micro

### Security Group

- SSH (22) → Your IP
- Vault (8200) → GitHub IPs (Demo: 0.0.0.0/0)

### SSH

```bash
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP
```

---

## Install Vault

```bash
sudo apt update
sudo apt install -y unzip wget

wget https://releases.hashicorp.com/vault/1.15.5/vault_1.15.5_linux_amd64.zip

unzip vault_1.15.5_linux_amd64.zip

sudo mv vault /usr/local/bin/

vault version
```

---

## Run Vault

```bash
vault server -dev \
-dev-root-token-id="root" \
-dev-listen-address="0.0.0.0:8200"
```

---

## Configure Vault

```bash
export VAULT_ADDR='http://127.0.0.1:8200'

vault login root
```

---

## Enable AWS Secrets Engine

```bash
vault secrets enable aws
```

### Configure AWS Root Credentials

```bash
vault write aws/config/root \
    access_key="AKIA..." \
    secret_key="SECRET..." \
    region="us-east-1"
```

---

## Create AWS Role

```bash
vault write aws/roles/terraform-role \
    credential_type=iam_user \
    policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
EOF
```

---

## Enable JWT Authentication

```bash
vault auth enable jwt
```

---

## Configure GitHub OIDC

```bash
vault write auth/jwt/config \
    oidc_discovery_url="https://token.actions.githubusercontent.com" \
    bound_issuer="https://token.actions.githubusercontent.com"
```

---

## Create Terraform Policy

```bash
vault policy write terraform-policy - <<EOF
path "aws/creds/terraform-role" {
  capabilities = ["read"]
}
EOF
```

---

## Bind GitHub Repository To Policy

```bash
vault write auth/jwt/role/gh-actions-role - <<EOF
{
  "role_type": "jwt",
  "bound_audiences": ["https://github.com/iam-veeramalla"],
  "user_claim": "sub",
  "bound_claims_type": "glob",
  "bound_claims": {
    "sub": "repo:iam-veeramalla/DevSecOps-Zero-to-Hero:*"
  },
  "token_policies": ["terraform-policy"],
  "token_ttl": "1h"
}
EOF
```

---

# Terraform Code

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "test_bucket" {
  bucket = "vault-demo-bucket-${random_id.bucket_suffix.hex}"
}
```

---

# GitHub Actions Workflow

```yaml
name: Terraform Deployment

on: [push]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform

    steps:
      - uses: actions/checkout@v4

      - name: Fetch Keys from Vault
        uses: hashicorp/vault-action@v3
        with:
          url: http://<YOUR_EC2_IP>:8200
          role: gh-actions-role
          method: jwt
          secrets: |
            aws/creds/terraform-role access_key | AWS_ACCESS_KEY_ID ;
            aws/creds/terraform-role secret_key | AWS_SECRET_ACCESS_KEY

      - uses: hashicorp/setup-terraform@v3

      - run: terraform init
      - run: terraform plan
      - run: terraform apply -auto-approve
```

---

# Complete Flow

```text
GitHub Actions
      ↓
Generate OIDC Token
      ↓
Vault JWT/OIDC Authentication
      ↓
Vault verifies GitHub Repository
      ↓
Vault JWT Role
      ↓
Terraform Policy
      ↓
AWS Secrets Engine
      ↓
Temporary AWS Credentials
      ↓
Terraform Apply
      ↓
AWS S3 Bucket Created
```

---

# Interview Answer

"We use HashiCorp Vault to avoid storing AWS credentials in GitHub. GitHub Actions authenticates using OIDC/JWT. Vault validates the GitHub token using the GitHub OIDC discovery URL. The repository is mapped to a Vault JWT role which has a policy attached. The policy grants read access to AWS credentials generated by the AWS Secrets Engine. Vault generates temporary AWS credentials with limited permissions and Terraform uses them to deploy infrastructure. This approach eliminates long-lived secrets and follows the principle of least privilege."
