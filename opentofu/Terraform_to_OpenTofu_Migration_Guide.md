# Migration Guide: Terraform 1.5.7 to OpenTofu 1.10.6

## Overview
This guide provides a step-by-step process for migrating your AWS Infrastructure as Code project from Terraform 1.5.7 to OpenTofu 1.10.6. Your project creates Elastic Cache, Security Groups, IAM roles, and AWS Secrets Manager secrets—all of which are fully compatible with OpenTofu.

OpenTofu aims to maintain compatibility with Terraform configurations, and most Terraform code will work without modification.

---

## Understanding the Versions

### Terraform 1.5.7
- Last open-source release under Mozilla Public License 2.0 (MPL 2.0)
- Released before HashiCorp's license change (August 10, 2023)
- Stable, widely-used foundation

### OpenTofu 1.10.6
- OpenTofu was forked from Terraform 1.5.7 and is a drop-in replacement for Terraform 1.5.x with the same HCL, same providers, and same state format
- Maintained by the Linux Foundation under MPL 2.0
- Community-driven with independent roadmap
- Enhanced with new features while maintaining backward compatibility

---

## Key Differences Between Terraform 1.5.7 and OpenTofu 1.10.6

### 1. **License & Governance**
| Aspect | Terraform 1.5.7 | OpenTofu 1.10.6 |
|--------|-----------------|-----------------|
| License | MPL 2.0 | MPL 2.0 |
| Governance | HashiCorp | Linux Foundation |
| Future Commercial Use | Restricted (1.6+) | Open Community |

### 2. **State Format Compatibility**
- ✅ **Fully Compatible**: State files from Terraform 1.5.7 work directly with OpenTofu 1.10.6
- No state migration needed
- Remote backends (S3, Azure Blob, GCS) continue functioning without reconfiguration

### 3. **New Features in OpenTofu 1.10.6** (Not in Terraform 1.5.7)

#### OCI Registry Support
- Distribute providers and modules through container registries, perfect for air-gapped environments
- Use `oci://` scheme for module sources

#### Native S3 State Locking
- Simplified state management without requiring DynamoDB
- S3 backends can now use native locking features

#### Module Deprecation Support
- Module authors can now mark variables and outputs as deprecated which will raise a warning to users
- Helps gracefully evolve module APIs

#### Enhanced Planning
- New -target-file and -exclude-file options for better resource management
- Better control over resource targeting

#### Global Provider Cache Lock
- Safe concurrent operations in CI/CD environments
- Prevents race conditions in parallel CI/CD pipelines

#### State Encryption (1.7+)
- Built-in client-side encryption for state files
- Client-side encryption ensures secrets never leave your machine unencrypted, valuable for compliance (SOC 2, HIPAA, PCI-DSS)
- Not available in Terraform 1.5.7 OSS

### 4. **Provider & Module Compatibility**
- Terraform Registry modules work with OpenTofu
- AWS Provider 4.x and 5.x fully compatible
- No provider changes needed for your Elastic Cache, Security Group, IAM, or Secrets Manager resources

### 5. **Command Compatibility**
All commands remain identical—just replace `terraform` with `tofu`:
```bash
tofu init       # Initialize working directory
tofu plan       # Create an execution plan
tofu apply      # Apply changes
tofu destroy    # Destroy resources
```

---

## Step-by-Step Migration Process

### **Phase 1: Pre-Migration (Planning & Backup)**

#### Step 1: Verify Current Terraform State
Ensure all infrastructure is in sync before migration:

```bash
terraform plan
# Expected output: No changes. Your infrastructure matches the configuration.
```

If there are pending changes, apply them first:
```bash
terraform apply
```

**Why**: Ensures clean state for migration and reduces confusion.

---

#### Step 2: Backup Your State File

**For Local State:**
```bash
# Create a backup copy
cp terraform.tfstate terraform.tfstate.backup
cp terraform.tfstate.backup.md5 terraform.tfstate.backup.md5.backup
```

**For Remote State (S3 backend):**
```bash
# Enable S3 versioning if not already enabled
aws s3api put-bucket-versioning \
  --bucket your-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Create a tagged backup copy
aws s3 cp s3://your-terraform-state-bucket/prod/terraform.tfstate \
  s3://your-terraform-state-bucket/backups/terraform.tfstate.tf1.5.7.backup

# Test restore (in non-prod):
aws s3 cp s3://your-terraform-state-bucket/backups/terraform.tfstate.tf1.5.7.backup \
  s3://your-terraform-state-bucket/terraform.tfstate
```

**For Remote State (Azure Blob):**
```bash
# Create a snapshot backup
az storage blob snapshot \
  --container-name tfstate \
  --name terraform.tfstate \
  --account-name yourstorageaccount
```

**For Remote State (Google Cloud Storage):**
```bash
# Create a versioned backup
gsutil cp gs://your-tf-state-bucket/terraform.tfstate \
  gs://your-tf-state-bucket/backups/terraform.tfstate.tf1.5.7.backup
```

**Why**: Enables quick rollback if migration fails.

---

#### Step 3: Backup Your Code
```bash
# Version control backup
git tag -a terraform-1.5.7-baseline -m "Baseline before OpenTofu migration"
git push origin terraform-1.5.7-baseline

# Filesystem backup
tar -czf terraform-project-backup.tar.gz .
cp terraform-project-backup.tar.gz /secure/location/
```

---

### **Phase 2: Installation & Setup**

#### Step 4: Install OpenTofu 1.10.6

**Option A: Binary Download (Linux/macOS/Windows)**
```bash
# Visit GitHub releases
https://github.com/opentofu/opentofu/releases/tag/v1.10.6

# For Linux (x86_64)
wget https://github.com/opentofu/opentofu/releases/download/v1.10.6/tofu_1.10.6_linux_amd64.zip
unzip tofu_1.10.6_linux_amd64.zip
sudo mv tofu /usr/local/bin/
chmod +x /usr/local/bin/tofu

# Verify installation
tofu version
# Expected: OpenTofu v1.10.6
```

**Option B: Package Manager (macOS)**
```bash
brew install opentofu
```

**Option C: Using tofuenv (Recommended - Manage Multiple Versions)**
```bash
# Install tofuenv
git clone --depth=1 https://github.com/tofuproject/tofuenv.git ~/.tofuenv
export PATH="$HOME/.tofuenv/bin:$PATH"

# Install OpenTofu 1.10.6
tofuenv install 1.10.6
tofuenv use 1.10.6

# Verify
tofu version
```

**Option D: Docker**
```bash
docker pull ghcr.io/opentofu/opentofu:1.10.6
docker run ghcr.io/opentofu/opentofu:1.10.6 version
```

**Why**: Ensures you have the exact version for reproducibility.

---

#### Step 5: Verify Installation
```bash
tofu version
# Should output: OpenTofu v1.10.6 on <platform>
```

---

### **Phase 3: Configuration Changes**

#### Step 6: Review and Update Backend Configuration

**If using S3 backend:**

Check your current `backend.tf` or `main.tf`:

```hcl
# Before (Terraform 1.5.7 compatible)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**Changes needed for OpenTofu 1.10.6:**

1. **Remove deprecated options:**
   - Remove `skip_s3_checksum` if present (not needed in OpenTofu)
   - Remove `endpoints.sso` if used (use AWS credentials instead)

2. **Optional: Add native S3 locking** (New in OpenTofu 1.10)
   ```hcl
   terraform {
     backend "s3" {
       bucket         = "my-terraform-state"
       key            = "prod/terraform.tfstate"
       region         = "us-east-1"
       encrypt        = true
       # Optional: Use native S3 locking instead of DynamoDB
       use_lockfile   = true  # New in OpenTofu 1.10
     }
   }
   ```

**If using Azure backend:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "my-rg"
    storage_account_name = "mystateaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
    # No changes required for basic compatibility
  }
}
```

**If using GCS backend:**
```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod"
    # No changes required for basic compatibility
  }
}
```

**If using Postgres backend:**
```hcl
terraform {
  backend "pg" {
    conn_str = "postgresql://user:password@localhost/tfstate"
    # In OpenTofu 1.10+, use table_name and index_name for multi-project support
    table_name = "terraform_prod_state"
    index_name = "terraform_prod_locks"
  }
}
```

**Why**: Removes deprecated options and enables new features.

---

#### Step 7: Update Terraform Block (Optional but Recommended)

**Current (Terraform 1.5.7):**
```hcl
terraform {
  required_version = ">= 1.5"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**Updated for OpenTofu 1.10.6:**
```hcl
terraform {
  required_version = ">= 1.6"  # Or ">= 1.10" to use 1.10+ features
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**Why**: Ensures clarity about version compatibility and enables OpenTofu-specific features.

---

#### Step 8: Configure AWS Credentials

**Ensure your AWS credentials are available (same as with Terraform):**

```bash
# Option 1: AWS credentials file
cat ~/.aws/credentials
# [default]
# aws_access_key_id = AKIA...
# aws_secret_access_key = ...

# Option 2: Environment variables
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"

# Option 3: IAM roles (EC2, ECS, Lambda, etc.)
# Automatically available in AWS-hosted environments

# Verify credentials work
aws sts get-caller-identity
# Should return your AWS account info
```

**Why**: OpenTofu uses the same credential chain as Terraform and AWS CLI.

---

### **Phase 4: Migration Execution**

#### Step 9: Initialize OpenTofu
```bash
cd /path/to/your/terraform/project

tofu init
# Output should show:
# - Downloading AWS provider
# - Configuring the backend
# - Initializing the working directory
```

**If using remote backend:**
```bash
tofu init -backend=true
# This confirms backend migration
```

**What happens:**
- Downloads provider plugins
- Initializes backend
- Creates `.terraform` directory (same as Terraform)

**Why**: Prepares OpenTofu to manage your infrastructure.

---

#### Step 10: Verify Plan (Dry Run)
```bash
tofu plan -out=tfplan
# Expected output: 
# No changes. Your infrastructure matches the configuration.
```

**If unexpected changes appear:**
```bash
# Detailed analysis
tofu plan -json | jq . > plan-details.json

# Review line by line
tofu plan

# Common causes:
# - Provider version differences (verify AWS provider version)
# - State drift in AWS (run terraform refresh first if using Terraform 1.5.7)
# - Attribute behavior changes (rare, check provider release notes)

# If issues found, rollback and troubleshoot
tofu destroy -force  # Or use terraform if rollback needed
```

---

#### Step 11: Review Detailed Plan Output

For your specific resources, check the plan carefully:

```bash
# View plan with details
tofu plan

# Look for your AWS resources:
# - aws_elasticache_cluster
# - aws_security_group
# - aws_iam_role
# - aws_secretsmanager_secret
```

**Expected output pattern:**
```
No changes. Your infrastructure matches the configuration.

Infrastructure is up to date with the configuration.
```

**Why**: Confirms OpenTofu correctly interprets your configuration.

---

#### Step 12: Apply Changes (Update State Format)
```bash
tofu apply tfplan
# Or without pre-created plan:
tofu apply

# Expected output:
# No changes. Your infrastructure matches the configuration.
# 
# Outputs:
# ... (your outputs)
```

Even though there are no infrastructure changes, this step updates the state file to OpenTofu format.

**Why**: Ensures state file is optimized for OpenTofu.

---

### **Phase 5: Validation & Testing**

#### Step 13: Test with a Small Change
Make a minor, non-critical change to validate OpenTofu can apply changes:

**Example 1: Add a tag to ElastiCache cluster**
```hcl
resource "aws_elasticache_cluster" "example" {
  cluster_id           = "my-cache"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  engine_version       = "7.0"
  
  tags = {
    Name           = "MyCache"
    ManagedBy      = "OpenTofu"  # Add this tag
    LastMigration  = "2025-04-15"  # Add this tag
  }
}
```

**Plan the change:**
```bash
tofu plan
# Output should show:
# resource "aws_elasticache_cluster" "example" will be updated in-place
```

**Apply the change:**
```bash
tofu apply
```

**Verify in AWS Console:**
```bash
aws elasticache describe-cache-clusters \
  --cache-cluster-id my-cache \
  --query 'CacheClusters[0].Tags'
# Should show the new tags
```

**Why**: Validates that OpenTofu can successfully modify AWS resources.

---

#### Step 14: Verify All Resources

For your specific resources, run verification commands:

**ElastiCache Cluster:**
```bash
tofu state show aws_elasticache_cluster.example

# Or via AWS CLI:
aws elasticache describe-cache-clusters --cache-cluster-id my-cache
```

**Security Groups:**
```bash
tofu state show aws_security_group.example

# Or via AWS CLI:
aws ec2 describe-security-groups --group-ids sg-xxxxx
```

**IAM Roles:**
```bash
tofu state show aws_iam_role.example

# Or via AWS CLI:
aws iam get-role --role-name my-role
```

**Secrets Manager Secrets:**
```bash
tofu state show aws_secretsmanager_secret.example

# Or via AWS CLI:
aws secretsmanager describe-secret --secret-id my-secret
```

---

#### Step 15: Test in Non-Production First (Recommended)

**If you have dev/staging environments:**

```bash
# Migrate dev environment first
cd terraform/dev
tofu init
tofu plan
tofu apply  # One small test change

# After successful validation, migrate staging
cd ../staging
tofu init
tofu plan
tofu apply

# Finally, migrate production
cd ../prod
tofu init
tofu plan
tofu apply
```

**Why**: Reduces risk by validating in lower environments first.

---

### **Phase 6: Update CI/CD Pipeline**

#### Step 16: Update CI/CD Configuration

**GitHub Actions Example:**

Before (Terraform 1.5.7):
```yaml
name: Terraform Plan and Apply

on: [push]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.5.7
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Plan
        run: terraform plan
      
      - name: Terraform Apply
        run: terraform apply -auto-approve
```

After (OpenTofu 1.10.6):
```yaml
name: OpenTofu Plan and Apply

on: [push]

jobs:
  opentofu:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: 1.10.6
      
      - name: OpenTofu Init
        run: tofu init
      
      - name: OpenTofu Plan
        run: tofu plan
      
      - name: OpenTofu Apply
        run: tofu apply -auto-approve
```

**GitLab CI Example:**

Before:
```yaml
stages:
  - plan
  - apply

variables:
  TF_VERSION: "1.5.7"

plan:
  image: hashicorp/terraform:${TF_VERSION}
  script:
    - terraform init
    - terraform plan
```

After:
```yaml
stages:
  - plan
  - apply

variables:
  TOFU_VERSION: "1.10.6"

plan:
  image: ghcr.io/opentofu/opentofu:${TOFU_VERSION}
  script:
    - tofu init
    - tofu plan
```

**Jenkins Example:**

Before:
```groovy
pipeline {
  agent any
  
  environment {
    TF_VERSION = '1.5.7'
  }
  
  stages {
    stage('Init') {
      steps {
        sh 'terraform init'
      }
    }
    stage('Plan') {
      steps {
        sh 'terraform plan'
      }
    }
  }
}
```

After:
```groovy
pipeline {
  agent any
  
  environment {
    TOFU_VERSION = '1.10.6'
  }
  
  stages {
    stage('Init') {
      steps {
        sh 'tofu init'
      }
    }
    stage('Plan') {
      steps {
        sh 'tofu plan'
      }
    }
  }
}
```

**Why**: Ensures CI/CD pipeline uses OpenTofu instead of Terraform.

---

#### Step 17: Update Build Scripts
```bash
# Old script (update-infrastructure.sh)
#!/bin/bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# New script (update-infrastructure.sh)
#!/bin/bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

Or make it dynamic:
```bash
#!/bin/bash
IaC_TOOL=${IaC_TOOL:-tofu}  # Default to OpenTofu, fall back to terraform

$IaC_TOOL init
$IaC_TOOL plan -out=tfplan
$IaC_TOOL apply tfplan
```

---

### **Phase 7: Post-Migration**

#### Step 18: Document Migration in Version Control

```bash
# Create a migration summary
cat > MIGRATION_LOG.md << 'EOF'
# Terraform → OpenTofu Migration Log

**Date**: 2025-04-15
**From**: Terraform 1.5.7
**To**: OpenTofu 1.10.6
**Environment**: Production

## Changes Made
- Replaced terraform binary with tofu
- Updated CI/CD pipelines (GitHub Actions, GitLab CI)
- Updated backend configuration (removed skip_s3_checksum)
- Added test tags to resources for validation

## Resources Migrated
- aws_elasticache_cluster: 1 instance
- aws_security_group: 3 groups
- aws_iam_role: 2 roles
- aws_secretsmanager_secret: 5 secrets

## Validation
- ✅ tofu init successful
- ✅ tofu plan showed no changes
- ✅ tofu apply successful
- ✅ Test changes applied successfully
- ✅ AWS resources verified in console

## Rollback Plan
If issues arise:
1. tofu destroy (optional)
2. Restore state: aws s3 cp s3://.../backups/terraform.tfstate.tf1.5.7.backup s3://.../terraform.tfstate
3. Switch back to terraform binary
4. terraform apply -refresh-only

## New Features Enabled (Optional)
- OCI Registry support available for module distribution
- Native S3 state locking (instead of DynamoDB)
- State encryption support (via encryption{} block)
- Deprecation warnings for module inputs/outputs
EOF

git add MIGRATION_LOG.md
git commit -m "docs: Log Terraform to OpenTofu 1.10.6 migration"
git push
```

**Why**: Provides documentation for future reference and team knowledge.

---

#### Step 19: Update Documentation
```bash
# Update README.md
cat >> README.md << 'EOF'

## Infrastructure as Code

This project uses **OpenTofu 1.10.6** to manage AWS infrastructure.

### Prerequisites
- OpenTofu 1.10.6 or later ([Installation Guide](https://opentofu.org/docs/intro/install/))
- AWS credentials configured (via `~/.aws/credentials` or environment variables)
- Terraform/OpenTofu state backend access (S3 bucket permissions)

### Quick Start
```bash
# Initialize
tofu init

# Review changes
tofu plan

# Apply changes
tofu apply
```

### Migrating from Terraform
If you're using Terraform, simply:
1. Install OpenTofu: `brew install opentofu` (or see installation guide)
2. Run `tofu init` in your project directory
3. Verify with `tofu plan`
EOF

git add README.md
git commit -m "docs: Add OpenTofu 1.10.6 setup instructions"
git push
```

---

#### Step 20: Implement Optional OpenTofu 1.10+ Features

**Option A: Enable State Encryption** (Not available in Terraform 1.5.7)

```hcl
# Add to main.tf or create encryption.tf
terraform {
  encryption {
    # Define encryption method
    method "aes_gcm" "aws_kms" {
      keys = key_provider.aws_kms.main
    }

    # Define key provider
    key_provider "aws_kms" "main" {
      kms_key_id = aws_kms_key.tfstate.arn
    }

    # Apply encryption to state
    state {
      method = method.aes_gcm.aws_kms
      
      # Optional fallback for decryption if key unavailable
      fallback {
        method = method.unencrypted.migrate
      }
    }

    # Optional: Encrypt plan files
    plan {
      method = method.aes_gcm.aws_kms
    }
  }
}

# KMS key for state encryption
resource "aws_kms_key" "tfstate" {
  description             = "KMS key for OpenTofu state encryption"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Name = "opentofu-state-encryption"
  }
}

resource "aws_kms_alias" "tfstate" {
  name          = "alias/opentofu-state"
  target_key_id = aws_kms_key.tfstate.key_id
}
```

**Apply encryption:**
```bash
tofu plan
tofu apply
# State file will now be encrypted with AWS KMS
```

---

**Option B: Use OCI Registries for Modules** (Not available in Terraform 1.5.7)

If you maintain private modules:

```hcl
module "my_module" {
  source = "oci://ghcr.io/myorg/terraform-modules/my-module"
  # Or: source = "oci://registry.example.com/myorg/terraform-modules/my-module"

  # Your module variables
}
```

---

**Option C: Mark Module Variables as Deprecated**

In your module's `variables.tf`:

```hcl
variable "old_parameter" {
  type        = string
  description = "This parameter is deprecated"
  deprecated  = "Use 'new_parameter' instead. This will be removed in v2.0.0"
}

variable "new_parameter" {
  type        = string
  description = "The new recommended parameter"
}
```

When someone uses `old_parameter`, they'll see:
```
Warning: Variable marked as deprecated by the module author

This parameter is deprecated. Use 'new_parameter' instead. 
This will be removed in v2.0.0.
```

---

---

## Troubleshooting Guide

### Issue 1: "Failed to query available provider packages"
**Cause**: Provider not available in OpenTofu registry
**Solution**:
```bash
# Verify provider is available
tofu init -upgrade

# Check provider version constraints
grep -A 5 "required_providers" *.tf

# Use Terraform registry as fallback
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

---

### Issue 2: "Module not found" error
**Cause**: Module not in OpenTofu registry
**Solution**:
```bash
# For modules using source paths (relative/local):
# No action needed - works the same

# For modules in Git:
source = "git::https://github.com/myorg/mymodule.git?ref=v1.0.0"

# For modules in Terraform Registry:
source = "myorg/mymodule/aws"
version = "~> 1.0"
```

---

### Issue 3: Unexpected plan changes
**Cause**: State version mismatch or Terraform 1.6+ state incompatibility
**Solution**:
```bash
# Compare Terraform and OpenTofu plans
terraform plan -json > tf.plan.json
tofu plan -json > tofu.plan.json

# Diff the resource changes
diff tf.plan.json tofu.plan.json

# If using Terraform 1.6+ state, review provider versions
terraform version
tofu version
aws --version
```

---

### Issue 4: AWS credential issues
**Cause**: Missing or expired AWS credentials
**Solution**:
```bash
# Verify credentials are loaded
aws sts get-caller-identity

# Set explicitly if needed
export AWS_PROFILE=my-profile
export AWS_REGION=us-east-1

# Test connection
tofu plan
```

---

### Issue 5: State lock timeout
**Cause**: Another process holding state lock
**Solution**:
```bash
# Check for locks
aws dynamodb scan --table-name terraform-locks \
  --region us-east-1

# Force unlock (use with caution!)
tofu force-unlock LOCK_ID

# Or remove DynamoDB entry manually
aws dynamodb delete-item \
  --table-name terraform-locks \
  --key '{"LockID":{"S":"my-bucket-key"}}'
```

---

## Rollback Procedure

If critical issues arise during migration:

### Immediate Rollback (Preserve OpenTofu attempt)
```bash
# Stop any running operations
# Ctrl+C to interrupt

# Switch back to Terraform
terraform init

# Verify state
terraform plan

# If state is corrupted, restore from backup
aws s3 cp s3://your-bucket/backups/terraform.tfstate.tf1.5.7.backup \
  s3://your-bucket/terraform.tfstate

# Reinitialize with Terraform
terraform init

# Verify
terraform plan
```

### Full Rollback to Terraform 1.5.7
```bash
# Remove OpenTofu
brew uninstall opentofu  # macOS
apt-get remove opentofu   # Ubuntu/Debian

# Install Terraform 1.5.7
brew install terraform@1.5  # macOS
# Or download binary from https://releases.hashicorp.com/terraform/1.5.7/

# Restore code from backup
git checkout terraform-1.5.7-baseline

# Reinitialize
terraform init
terraform plan
terraform apply  # if needed
```

---

## Comparing Terraform 1.5.7 vs OpenTofu 1.10.6

| Feature | Terraform 1.5.7 | OpenTofu 1.10.6 | Notes |
|---------|-----------------|-----------------|-------|
| **State Format** | ✅ | ✅ | 100% compatible |
| **HCL Syntax** | ✅ | ✅ | Same language |
| **AWS Provider** | ✅ | ✅ | Works with 4.x, 5.x |
| **Modules** | ✅ | ✅ | Compatible |
| **State Encryption** | ❌ | ✅ | New in OpenTofu 1.7+ |
| **OCI Registries** | ❌ | ✅ | New in OpenTofu 1.10+ |
| **S3 Native Locking** | ❌ | ✅ | New in OpenTofu 1.10+ |
| **Module Deprecation** | ❌ | ✅ | New in OpenTofu 1.10+ |
| **Ephemeral Resources** | ❌ | ✅ | New in OpenTofu 1.11+ |
| **License** | MPL 2.0 | MPL 2.0 | Same |
| **Community Support** | HashiCorp | Linux Foundation | Open governance |
| **Cost** | Free | Free | No licensing fees |

---

## Post-Migration Validation Checklist

- [ ] OpenTofu 1.10.6 installed and verified (`tofu version`)
- [ ] State file backed up (local and remote)
- [ ] Code committed to git with `terraform-1.5.7-baseline` tag
- [ ] Backend configuration updated (removed deprecated options)
- [ ] `tofu init` successful
- [ ] `tofu plan` shows no changes
- [ ] `tofu apply` successful
- [ ] Test change applied and verified in AWS
- [ ] All resources verified (ElastiCache, Security Groups, IAM, Secrets Manager)
- [ ] CI/CD pipeline updated to use `tofu` instead of `terraform`
- [ ] Documentation updated (README, runbooks, wiki)
- [ ] Team notified and trained on OpenTofu
- [ ] Monitoring updated (if any terraform-specific checks)
- [ ] Rollback procedure tested (recommended)

---

## Resources & Documentation

- **Official OpenTofu Docs**: https://opentofu.org/docs/
- **Migration Guide**: https://opentofu.org/docs/intro/migration/
- **OpenTofu GitHub**: https://github.com/opentofu/opentofu
- **Community Slack**: Join via https://opentofu.org/community/
- **AWS Provider Documentation**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- **ElastiCache Resources**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elasticache_cluster
- **Security Group Resources**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group
- **IAM Resources**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role
- **Secrets Manager Resources**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret

---

## Quick Reference Commands

```bash
# Installation
brew install opentofu              # macOS
apt-get install opentofu           # Ubuntu/Debian
choco install opentofu             # Windows

# Or use tofuenv for version management
git clone --depth=1 https://github.com/tofuproject/tofuenv.git ~/.tofuenv
export PATH="$HOME/.tofuenv/bin:$PATH"
tofuenv install 1.10.6
tofuenv use 1.10.6

# Migration workflow
tofu version                        # Verify installation
tofu init                           # Initialize (same as terraform init)
tofu plan -out=tfplan              # Plan changes
tofu show tfplan                    # Review plan
tofu apply tfplan                   # Apply changes
tofu state list                     # List resources
tofu state show aws_elasticache_cluster.example  # Show specific resource
tofu destroy                        # Destroy infrastructure
tofu output                         # Show outputs

# Advanced
tofu validate                       # Validate configuration
tofu fmt -recursive                 # Format all HCL files
tofu console                        # Interactive REPL
tofu graph                          # Generate resource graph
tofu console
```

---

**Migration completed successfully!** You're now using OpenTofu 1.10.6 with all its modern features while maintaining full compatibility with your existing Terraform configuration.
