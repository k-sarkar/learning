# OpenTofu 1.10.6 Configuration Examples for AWS

This document provides working examples for the AWS resources in your project: ElastiCache, Security Groups, IAM roles, and Secrets Manager.

---

## Project Structure

```
terraform-project/
├── main.tf                 # Main configuration
├── variables.tf            # Input variables
├── outputs.tf              # Output values
├── backend.tf              # Backend configuration
├── elasticache.tf          # ElastiCache resources
├── security_groups.tf      # Security group resources
├── iam.tf                  # IAM role resources
├── secrets_manager.tf      # Secrets Manager resources
├── terraform.tfvars        # Terraform variables (optional)
└── .gitignore              # Git ignore file
```

---

## 1. Backend Configuration (backend.tf)

### S3 Backend (Recommended for Production)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-prod"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    
    # Optional: Use native S3 locking (OpenTofu 1.10+)
    # use_lockfile = true
  }
}
```

### Local Backend (Development Only)

```hcl
# backend.tf - For local development
terraform {
  backend "local" {
    path = "./terraform.tfstate"
  }
}
```

### Azure Backend

```hcl
# backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "my-terraform-rg"
    storage_account_name = "mytfstatestg"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### PostgreSQL Backend

```hcl
# backend.tf - OpenTofu 1.10+ supports multi-project setup
terraform {
  backend "pg" {
    conn_str   = "postgresql://tfuser:password@postgres.example.com/terraform"
    table_name = "terraform_state_prod"
    index_name = "terraform_locks_prod"
  }
}
```

---

## 2. Provider Configuration (main.tf)

```hcl
# main.tf - Provider and required versions
terraform {
  required_version = ">= 1.6"  # OpenTofu 1.10.6+ or Terraform 1.6+
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"  # AWS provider 5.x
    }
  }

  # Optional: Enable state encryption (OpenTofu 1.7+)
  encryption {
    method "aes_gcm" "aws_kms" {
      keys = key_provider.aws_kms.main
    }

    key_provider "aws_kms" "main" {
      kms_key_id = aws_kms_key.tfstate.arn
    }

    state {
      method = method.aes_gcm.aws_kms
      fallback {
        method = method.unencrypted.migrate
      }
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "OpenTofu"
      CreatedAt   = timestamp()
    }
  }
}

# Optional: KMS key for state encryption
resource "aws_kms_key" "tfstate" {
  description             = "KMS key for OpenTofu state encryption"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Name = "opentofu-state-key"
  }
}

resource "aws_kms_alias" "tfstate" {
  name          = "alias/opentofu-state"
  target_key_id = aws_kms_key.tfstate.key_id
}
```

---

## 3. Variables Configuration (variables.tf)

```hcl
# variables.tf

variable "aws_region" {
  type        = string
  description = "AWS region for resources"
  default     = "us-east-1"
}

variable "project_name" {
  type        = string
  description = "Project name for tagging and naming"
  validation {
    condition     = length(var.project_name) > 0 && length(var.project_name) <= 32
    error_message = "Project name must be between 1 and 32 characters."
  }
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, prod)"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# ElastiCache Variables
variable "elasticache_engine" {
  type        = string
  description = "ElastiCache engine type"
  default     = "redis"
  validation {
    condition     = contains(["redis", "memcached"], var.elasticache_engine)
    error_message = "Engine must be redis or memcached."
  }
}

variable "elasticache_node_type" {
  type        = string
  description = "ElastiCache node type"
  default     = "cache.t3.micro"
}

variable "elasticache_engine_version" {
  type        = string
  description = "ElastiCache engine version"
  default     = "7.0"
}

variable "elasticache_num_cache_nodes" {
  type        = number
  description = "Number of cache nodes"
  default     = 1
  validation {
    condition     = var.elasticache_num_cache_nodes >= 1 && var.elasticache_num_cache_nodes <= 100
    error_message = "Number of nodes must be between 1 and 100."
  }
}

# Security Group Variables
variable "allowed_security_groups" {
  type        = list(string)
  description = "Security group IDs allowed to access ElastiCache"
  default     = []
}

variable "allowed_cidr_blocks" {
  type        = list(string)
  description = "CIDR blocks allowed to access ElastiCache"
  default     = []
}

# Secrets Manager Variables
variable "enable_secret_rotation" {
  type        = bool
  description = "Enable automatic secret rotation"
  default     = false
}

variable "secret_rotation_days" {
  type        = number
  description = "Number of days for secret rotation"
  default     = 30
  validation {
    condition     = var.secret_rotation_days >= 1 && var.secret_rotation_days <= 365
    error_message = "Rotation days must be between 1 and 365."
  }
}
```

---

## 4. ElastiCache Configuration (elasticache.tf)

### Redis Cluster

```hcl
# elasticache.tf

# ElastiCache Parameter Group for Redis
resource "aws_elasticache_parameter_group" "redis" {
  name        = "${var.project_name}-redis-params-${var.environment}"
  family      = "redis7"
  description = "Redis parameter group for ${var.project_name}"

  # Common Redis parameters
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  parameter {
    name  = "timeout"
    value = "300"
  }

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "${var.project_name}-redis-params"
  }
}

# ElastiCache Redis Cluster
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "${var.project_name}-redis-${var.environment}"
  engine               = var.elasticache_engine
  node_type            = var.elasticache_node_type
  num_cache_nodes      = var.elasticache_num_cache_nodes
  parameter_group_name = aws_elasticache_parameter_group.redis.name
  engine_version       = var.elasticache_engine_version
  port                 = 6379
  
  # Security
  security_group_ids = [aws_security_group.elasticache.id]
  
  # Automatic backups for production
  snapshot_retention_limit = var.environment == "prod" ? 30 : 5
  snapshot_window          = var.environment == "prod" ? "03:00-05:00" : null
  maintenance_window       = var.environment == "prod" ? "sun:05:00-sun:07:00" : null
  
  # Encryption at rest (available in supported node types)
  at_rest_encryption_enabled = true
  auth_token_enabled         = var.environment == "prod" ? true : false
  
  # Encryption in transit
  transit_encryption_enabled = true
  transit_encryption_mode    = "preferred"
  
  # Logging
  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.elasticache_engine.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    enabled          = true
  }

  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.elasticache_slowlog.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    enabled          = var.environment == "prod"
  }

  # Monitoring
  enable_automatic_failover = var.environment == "prod" ? true : false
  
  # Notification topic for alerts
  notification_topic_arn = var.environment == "prod" ? aws_sns_topic.elasticache_alerts.arn : null

  depends_on = [aws_security_group.elasticache]

  tags = {
    Name = "${var.project_name}-redis"
  }

  lifecycle {
    ignore_changes = [engine_version]  # Prevent forced replacement on minor updates
  }
}

# CloudWatch Log Groups
resource "aws_cloudwatch_log_group" "elasticache_engine" {
  name              = "/aws/elasticache/${var.project_name}-engine-${var.environment}"
  retention_in_days = var.environment == "prod" ? 30 : 7

  tags = {
    Name = "${var.project_name}-elasticache-engine-logs"
  }
}

resource "aws_cloudwatch_log_group" "elasticache_slowlog" {
  name              = "/aws/elasticache/${var.project_name}-slowlog-${var.environment}"
  retention_in_days = var.environment == "prod" ? 30 : 7

  tags = {
    Name = "${var.project_name}-elasticache-slowlog"
  }
}

# SNS Topic for Alerts (Production only)
resource "aws_sns_topic" "elasticache_alerts" {
  count = var.environment == "prod" ? 1 : 0
  name  = "${var.project_name}-elasticache-alerts"

  tags = {
    Name = "${var.project_name}-elasticache-alerts"
  }
}

# CloudWatch Alarm for CPU
resource "aws_cloudwatch_metric_alarm" "elasticache_cpu" {
  alarm_name          = "${var.project_name}-elasticache-cpu-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  alarm_description   = "Alert when ElastiCache CPU exceeds 75%"
  alarm_actions       = var.environment == "prod" ? [aws_sns_topic.elasticache_alerts[0].arn] : []

  dimensions = {
    CacheClusterId = aws_elasticache_cluster.redis.id
  }

  tags = {
    Name = "${var.project_name}-elasticache-cpu-alarm"
  }
}
```

### Memcached Cluster (Alternative)

```hcl
# elasticache.tf - Memcached alternative

resource "aws_elasticache_cluster" "memcached" {
  count = var.elasticache_engine == "memcached" ? 1 : 0
  
  cluster_id           = "${var.project_name}-memcached-${var.environment}"
  engine               = "memcached"
  node_type            = var.elasticache_node_type
  num_cache_nodes      = var.elasticache_num_cache_nodes
  engine_version       = "1.6.17"
  port                 = 11211
  
  security_group_ids = [aws_security_group.elasticache.id]

  tags = {
    Name = "${var.project_name}-memcached"
  }
}
```

---

## 5. Security Groups Configuration (security_groups.tf)

```hcl
# security_groups.tf

# Security Group for ElastiCache
resource "aws_security_group" "elasticache" {
  name        = "${var.project_name}-elasticache-${var.environment}"
  description = "Security group for ElastiCache cluster"
  
  # Allow inbound from specified security groups
  dynamic "ingress" {
    for_each = var.allowed_security_groups
    content {
      from_port       = 6379
      to_port         = 6379
      protocol        = "tcp"
      security_groups = [ingress.value]
      description     = "Redis from security group ${ingress.value}"
    }
  }

  # Allow inbound from CIDR blocks
  dynamic "ingress" {
    for_each = var.allowed_cidr_blocks
    content {
      from_port   = 6379
      to_port     = 6379
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
      description = "Redis from CIDR ${ingress.value}"
    }
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }

  tags = {
    Name = "${var.project_name}-elasticache-sg"
  }

  lifecycle {
    create_before_destroy = true
  }

  depends_on = [aws_elasticache_cluster.redis]
}

# Security Group for Application Servers (example)
resource "aws_security_group" "application" {
  name        = "${var.project_name}-app-${var.environment}"
  description = "Security group for application servers"

  # Allow HTTP
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP traffic"
  }

  # Allow HTTPS
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS traffic"
  }

  # Allow SSH (restricted to specific IPs)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
    description = "SSH access"
  }

  # Allow access to ElastiCache
  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.elasticache.id]
    description     = "Redis access"
  }

  # Allow all outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }

  tags = {
    Name = "${var.project_name}-app-sg"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Security Group for Databases (example)
resource "aws_security_group" "database" {
  name        = "${var.project_name}-db-${var.environment}"
  description = "Security group for RDS/databases"

  # Allow from application servers
  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.application.id]
    description     = "MySQL access from app servers"
  }

  # Allow all outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }

  tags = {
    Name = "${var.project_name}-db-sg"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## 6. IAM Configuration (iam.tf)

```hcl
# iam.tf

# IAM Role for ElastiCache
resource "aws_iam_role" "elasticache_role" {
  name        = "${var.project_name}-elasticache-role-${var.environment}"
  description = "IAM role for ElastiCache operations"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = [
            "elasticache.amazonaws.com",
            "ec2.amazonaws.com"
          ]
        }
      }
    ]
  })

  tags = {
    Name = "${var.project_name}-elasticache-role"
  }
}

# IAM Policy for ElastiCache CloudWatch Logs
resource "aws_iam_role_policy" "elasticache_cloudwatch_logs" {
  name = "${var.project_name}-elasticache-cloudwatch-policy"
  role = aws_iam_role.elasticache_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:log-group:/aws/elasticache/*"
      }
    ]
  })
}

# IAM Policy for KMS (state encryption)
resource "aws_iam_role_policy" "elasticache_kms" {
  count = var.environment == "prod" ? 1 : 0
  name  = "${var.project_name}-elasticache-kms-policy"
  role  = aws_iam_role.elasticache_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey",
          "kms:CreateGrant"
        ]
        Resource = [
          aws_kms_key.tfstate.arn
        ]
      }
    ]
  })
}

# IAM Role for Application Servers
resource "aws_iam_role" "application_role" {
  name        = "${var.project_name}-app-role-${var.environment}"
  description = "IAM role for application servers"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = [
            "ec2.amazonaws.com",
            "ecs-tasks.amazonaws.com"
          ]
        }
      }
    ]
  })

  tags = {
    Name = "${var.project_name}-app-role"
  }
}

# IAM Policy for application to access ElastiCache
resource "aws_iam_role_policy" "application_elasticache" {
  name = "${var.project_name}-app-elasticache-policy"
  role = aws_iam_role.application_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "elasticache:DescribeCacheClusters",
          "elasticache:DescribeReplicationGroups"
        ]
        Resource = "*"
      }
    ]
  })
}

# IAM Policy for Secrets Manager access
resource "aws_iam_role_policy" "application_secrets" {
  name = "${var.project_name}-app-secrets-policy"
  role = aws_iam_role.application_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = aws_secretsmanager_secret.app_secrets[*].arn
      }
    ]
  })
}

# Instance Profile for EC2 instances
resource "aws_iam_instance_profile" "application" {
  name = "${var.project_name}-app-profile-${var.environment}"
  role = aws_iam_role.application_role.name
}
```

---

## 7. Secrets Manager Configuration (secrets_manager.tf)

```hcl
# secrets_manager.tf

# Database Credentials Secret
resource "aws_secretsmanager_secret" "db_credentials" {
  name                    = "${var.project_name}/db/credentials-${var.environment}"
  description             = "Database credentials for ${var.project_name}"
  recovery_window_in_days = var.environment == "prod" ? 30 : 7
  kms_key_id              = aws_kms_key.secrets.id

  tags = {
    Name = "${var.project_name}-db-credentials"
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = "dbadmin"
    password = random_password.db_password.result
    engine   = "mysql"
    host     = "example.rds.amazonaws.com"
    port     = 3306
    dbname   = "myapp_${var.environment}"
  })
}

# ElastiCache Authentication Secret
resource "aws_secretsmanager_secret" "elasticache_auth" {
  count                   = var.environment == "prod" ? 1 : 0
  name                    = "${var.project_name}/elasticache/auth-${var.environment}"
  description             = "ElastiCache authentication token"
  recovery_window_in_days = 30
  kms_key_id              = aws_kms_key.secrets.id

  tags = {
    Name = "${var.project_name}-elasticache-auth"
  }
}

resource "aws_secretsmanager_secret_version" "elasticache_auth" {
  count      = var.environment == "prod" ? 1 : 0
  secret_id  = aws_secretsmanager_secret.elasticache_auth[0].id
  secret_string = jsonencode({
    auth_token = random_password.elasticache_token[0].result
    endpoint   = aws_elasticache_cluster.redis.cache_nodes[0].address
    port       = 6379
  })
}

# API Keys Secret
resource "aws_secretsmanager_secret" "api_keys" {
  name                    = "${var.project_name}/api/keys-${var.environment}"
  description             = "External API keys for ${var.project_name}"
  recovery_window_in_days = var.environment == "prod" ? 30 : 7
  kms_key_id              = aws_kms_key.secrets.id

  tags = {
    Name = "${var.project_name}-api-keys"
  }
}

resource "aws_secretsmanager_secret_version" "api_keys" {
  secret_id = aws_secretsmanager_secret.api_keys.id
  secret_string = jsonencode({
    stripe_api_key = var.stripe_api_key
    sendgrid_key   = var.sendgrid_key
    # Add other API keys as needed
  })

  sensitive = true  # Prevent terraform output from showing secret
}

# Application Configuration Secret
resource "aws_secretsmanager_secret" "app_config" {
  name                    = "${var.project_name}/app/config-${var.environment}"
  description             = "Application configuration secrets"
  recovery_window_in_days = var.environment == "prod" ? 30 : 7
  kms_key_id              = aws_kms_key.secrets.id

  tags = {
    Name = "${var.project_name}-app-config"
  }
}

resource "aws_secretsmanager_secret_version" "app_config" {
  secret_id = aws_secretsmanager_secret.app_config.id
  secret_string = jsonencode({
    jwt_secret        = random_password.jwt_secret.result
    encryption_key    = random_password.encryption_key.result
    session_secret    = random_password.session_secret.result
    environment       = var.environment
  })

  sensitive = true
}

# Secret for CI/CD Deployments
resource "aws_secretsmanager_secret" "deployment_credentials" {
  name                    = "${var.project_name}/deployment/credentials-${var.environment}"
  description             = "Deployment credentials for CI/CD"
  recovery_window_in_days = var.environment == "prod" ? 30 : 7
  kms_key_id              = aws_kms_key.secrets.id

  tags = {
    Name = "${var.project_name}-deployment-creds"
  }
}

resource "aws_secretsmanager_secret_version" "deployment_credentials" {
  secret_id = aws_secretsmanager_secret.deployment_credentials.id
  secret_string = jsonencode({
    github_token      = var.github_token
    docker_registry   = var.docker_registry
    docker_username   = var.docker_username
    docker_password   = var.docker_password
  })

  sensitive = true
}

# KMS Key for Secrets encryption
resource "aws_kms_key" "secrets" {
  description             = "KMS key for Secrets Manager encryption"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow Secrets Manager"
        Effect = "Allow"
        Principal = {
          Service = "secretsmanager.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey",
          "kms:CreateGrant"
        ]
        Resource = "*"
      },
      {
        Sid    = "Allow IAM Role Access"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.application_role.arn
        }
        Action = [
          "kms:Decrypt",
          "kms:GetPublicKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name = "secrets-encryption-key"
  }
}

resource "aws_kms_alias" "secrets" {
  name          = "alias/${var.project_name}-secrets"
  target_key_id = aws_kms_key.secrets.key_id
}

# Secret rotation configuration (optional)
resource "aws_secretsmanager_secret_rotation" "db_credentials" {
  count               = var.enable_secret_rotation ? 1 : 0
  secret_id           = aws_secretsmanager_secret.db_credentials.id
  rotation_rules {
    automatically_after_days = var.secret_rotation_days
  }
}

# Data source for AWS account ID
data "aws_caller_identity" "current" {}

# Random passwords
resource "random_password" "db_password" {
  length            = 32
  special           = true
  override_special  = "!#$%&*()-_=+[]{}<>:?"
}

resource "random_password" "elasticache_token" {
  count             = var.environment == "prod" ? 1 : 0
  length            = 32
  special           = true
  override_special  = "!#$%&*()-_=+[]{}<>:?"
}

resource "random_password" "jwt_secret" {
  length            = 64
  special           = true
  override_special  = "!#$%&*()-_=+[]{}<>:?"
}

resource "random_password" "encryption_key" {
  length            = 64
  special           = true
  override_special  = "!#$%&*()-_=+[]{}<>:?"
}

resource "random_password" "session_secret" {
  length            = 32
  special           = true
  override_special  = "!#$%&*()-_=+[]{}<>:?"
}
```

---

## 8. Outputs (outputs.tf)

```hcl
# outputs.tf

# ElastiCache outputs
output "elasticache_cluster_id" {
  description = "ElastiCache cluster ID"
  value       = aws_elasticache_cluster.redis.id
}

output "elasticache_endpoint" {
  description = "ElastiCache cluster endpoint"
  value       = aws_elasticache_cluster.redis.cache_nodes[0].address
  sensitive   = false  # Endpoints are not sensitive
}

output "elasticache_port" {
  description = "ElastiCache port"
  value       = aws_elasticache_cluster.redis.port
}

output "elasticache_engine_version" {
  description = "ElastiCache engine version"
  value       = aws_elasticache_cluster.redis.engine_version
}

# Security Group outputs
output "elasticache_security_group_id" {
  description = "Security group ID for ElastiCache"
  value       = aws_security_group.elasticache.id
}

output "application_security_group_id" {
  description = "Security group ID for application servers"
  value       = aws_security_group.application.id
}

output "database_security_group_id" {
  description = "Security group ID for databases"
  value       = aws_security_group.database.id
}

# IAM Role outputs
output "application_role_arn" {
  description = "ARN of application IAM role"
  value       = aws_iam_role.application_role.arn
}

output "application_instance_profile" {
  description = "IAM instance profile for EC2 instances"
  value       = aws_iam_instance_profile.application.name
}

output "elasticache_role_arn" {
  description = "ARN of ElastiCache IAM role"
  value       = aws_iam_role.elasticache_role.arn
}

# Secrets Manager outputs
output "db_credentials_secret_arn" {
  description = "ARN of database credentials secret"
  value       = aws_secretsmanager_secret.db_credentials.arn
  sensitive   = true
}

output "elasticache_auth_secret_arn" {
  description = "ARN of ElastiCache auth secret"
  value       = try(aws_secretsmanager_secret.elasticache_auth[0].arn, "N/A")
  sensitive   = true
}

output "api_keys_secret_arn" {
  description = "ARN of API keys secret"
  value       = aws_secretsmanager_secret.api_keys.arn
  sensitive   = true
}

output "app_config_secret_arn" {
  description = "ARN of application configuration secret"
  value       = aws_secretsmanager_secret.app_config.arn
  sensitive   = true
}

# KMS Key outputs
output "secrets_kms_key_id" {
  description = "KMS key ID for Secrets Manager"
  value       = aws_kms_key.secrets.key_id
}

output "tfstate_kms_key_id" {
  description = "KMS key ID for state file encryption"
  value       = aws_kms_key.tfstate.key_id
}
```

---

## 9. Example terraform.tfvars

```hcl
# terraform.tfvars
# Copy this file and modify values for your environment

aws_region = "us-east-1"

project_name = "myapp"
environment  = "prod"

# ElastiCache configuration
elasticache_engine         = "redis"
elasticache_node_type      = "cache.t3.micro"
elasticache_engine_version = "7.0"
elasticache_num_cache_nodes = 1

# Security
allowed_security_groups = ["sg-12345678"]
allowed_cidr_blocks     = ["10.0.0.0/8"]

# Secrets Manager
enable_secret_rotation = true
secret_rotation_days   = 30

# External API keys (store in AWS Secrets Manager, not in code)
stripe_api_key = "sk_live_..."  # Use AWS Secrets Manager instead
sendgrid_key   = "SG...."        # Use AWS Secrets Manager instead
github_token   = "ghp_..."       # Use AWS Secrets Manager instead
```

---

## 10. .gitignore

```gitignore
# .gitignore

# Local OpenTofu files
.terraform/
.terraform.lock.hcl
terraform.tfstate
terraform.tfstate.backup
*.tfstate
*.tfstate.backup

# Local .terraform directories
**/.terraform/*

# .tfvars files (they may contain secrets)
*.tfvars
*.tfvars.json
!terraform.tfvars.example

# Ignore override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Include override files you do wish to add to version control
# !example_override.tf

# Include tfplan files to ignore the plan output of command: terraform plan -out=tfplan
*tfplan*

# Ignore CLI configuration files
.terraformrc
terraform.rc

# Ignore plan files
*.tfplan

# Ignore crash dumps
crash.dump

# Ignore lock files
.terraform.lock.hcl

# IDE
.idea/
*.swp
*.swo
*~
.vscode/
*.iml

# OS
.DS_Store
Thumbs.db

# Logs
*.log
```

---

## Migration Checklist for Configuration Files

- [ ] Updated `backend.tf` (removed deprecated options)
- [ ] Updated `main.tf` with `required_version = ">= 1.6"`
- [ ] All variables defined in `variables.tf`
- [ ] All outputs defined in `outputs.tf`
- [ ] ElastiCache resources configured in `elasticache.tf`
- [ ] Security groups configured in `security_groups.tf`
- [ ] IAM roles configured in `iam.tf`
- [ ] Secrets Manager resources configured in `secrets_manager.tf`
- [ ] `.gitignore` updated to exclude sensitive files
- [ ] `terraform.tfvars` created and customized (not committed to git)
- [ ] All files formatted with `tofu fmt -recursive`
- [ ] Configuration validated with `tofu validate`
- [ ] Example `.terraform.tfvars` created for documentation
- [ ] Backup of original Terraform 1.5.7 configuration created

---

This configuration set provides a production-ready, secure setup for managing AWS infrastructure with OpenTofu 1.10.6!
