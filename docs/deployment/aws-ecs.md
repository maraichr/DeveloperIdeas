# AWS ECS Fargate Deployment Guide

This guide covers deploying GateHouse to AWS using ECS Fargate with Terraform.

## Table of Contents

- [Infrastructure Overview](#infrastructure-overview)
- [Prerequisites](#prerequisites)
- [Terraform Deployment](#terraform-deployment)
- [Service-by-Service Setup](#service-by-service-setup)
- [Database Setup](#database-setup)
- [Storage Configuration](#storage-configuration)
- [Security Configuration](#security-configuration)
- [Load Balancer Setup](#load-balancer-setup)
- [DNS and SSL](#dns-and-ssl)

## Infrastructure Overview

### VPC Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              VPC (10.0.0.0/16)                              │
│                                                                             │
│  ┌─────────────────────────────┐    ┌─────────────────────────────┐        │
│  │    Availability Zone A      │    │    Availability Zone B      │        │
│  │                             │    │                             │        │
│  │  ┌───────────────────────┐  │    │  ┌───────────────────────┐  │        │
│  │  │   Public Subnet A     │  │    │  │   Public Subnet B     │  │        │
│  │  │    10.0.1.0/24       │  │    │  │    10.0.2.0/24       │  │        │
│  │  │  ┌─────┐  ┌─────┐    │  │    │  │  ┌─────┐             │  │        │
│  │  │  │ NAT │  │ ALB │    │  │    │  │  │ ALB │             │  │        │
│  │  │  └──┬──┘  └─────┘    │  │    │  │  └─────┘             │  │        │
│  │  └─────┼────────────────┘  │    │  └───────────────────────┘  │        │
│  │        │                   │    │                             │        │
│  │  ┌─────▼─────────────────┐ │    │  ┌───────────────────────┐  │        │
│  │  │  Private Subnet A    │  │    │  │  Private Subnet B     │  │        │
│  │  │    10.0.10.0/24      │  │    │  │    10.0.20.0/24      │  │        │
│  │  │  ┌─────┐  ┌────────┐ │  │    │  │  ┌─────┐  ┌────────┐ │  │        │
│  │  │  │ ECS │  │  RDS   │ │  │    │  │  │ ECS │  │  RDS   │ │  │        │
│  │  │  │Tasks│  │Standby │ │  │    │  │  │Tasks│  │Primary │ │  │        │
│  │  │  └─────┘  └────────┘ │  │    │  │  └─────┘  └────────┘ │  │        │
│  │  └──────────────────────┘  │    │  └───────────────────────┘  │        │
│  └─────────────────────────────┘    └─────────────────────────────┘        │
│                                                                             │
│  Internet Gateway ◄────────────────► Internet                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Network Configuration

| Subnet Type | CIDR Range | Purpose |
|-------------|------------|---------|
| Public A | 10.0.1.0/24 | ALB, NAT Gateway |
| Public B | 10.0.2.0/24 | ALB (multi-AZ) |
| Private A | 10.0.10.0/24 | ECS tasks, RDS |
| Private B | 10.0.20.0/24 | ECS tasks, RDS (multi-AZ) |

## Prerequisites

1. **AWS CLI configured** with appropriate credentials
2. **Terraform >= 1.5** installed
3. **Docker** for building container images
4. **Route 53 hosted zone** for your domain
5. **ACM certificate** for HTTPS (or let Terraform create one)

```bash
# Verify AWS access
aws sts get-caller-identity

# Check Terraform version
terraform version
```

## Terraform Deployment

### Initialize Terraform

```bash
cd terraform

# Initialize with backend configuration
terraform init

# Review planned changes
terraform plan -var-file=environments/prod.tfvars

# Apply infrastructure
terraform apply -var-file=environments/prod.tfvars
```

### Required Variables

Create `environments/prod.tfvars`:

```hcl
# Core settings
environment    = "prod"
aws_region     = "us-east-1"
project_name   = "gatehouse"

# Networking
vpc_cidr           = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]

# Domain
domain_name     = "gatehouse.example.com"
route53_zone_id = "Z1234567890"

# Database
db_instance_class = "db.t3.medium"
db_multi_az       = true
db_storage_gb     = 100

# ECS
api_cpu           = 512
api_memory        = 1024
api_desired_count = 2

worker_cpu           = 512
worker_memory        = 1024
worker_desired_count = 2

# Container images (from ECR)
api_image    = "123456789.dkr.ecr.us-east-1.amazonaws.com/gatehouse-api:latest"
worker_image = "123456789.dkr.ecr.us-east-1.amazonaws.com/gatehouse-worker:latest"
```

## Service-by-Service Setup

### 1. API Server

The API server is the main HTTP endpoint for GateHouse.

**Task Definition:**

```json
{
  "family": "gatehouse-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/gatehouse-api-task-role",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/gatehouse-api:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "GATEHOUSE_PORT", "value": "8080"},
        {"name": "GATEHOUSE_LOG_LEVEL", "value": "info"},
        {"name": "GATEHOUSE_LOG_FORMAT", "value": "json"},
        {"name": "TEMPORAL_ADDRESS", "value": "temporal.gatehouse.local:7233"},
        {"name": "TEMPORAL_NAMESPACE", "value": "default"},
        {"name": "SPICEDB_ENDPOINT", "value": "spicedb.gatehouse.local:50051"},
        {"name": "GATEHOUSE_STORAGE_USE_SSL", "value": "true"},
        {"name": "AUTO_MIGRATE", "value": "false"}
      ],
      "secrets": [
        {
          "name": "GATEHOUSE_DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/database-url"
        },
        {
          "name": "GATEHOUSE_JWT_SECRET",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/jwt-secret"
        },
        {
          "name": "GATEHOUSE_SECRETS_KEY",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/secrets-key"
        },
        {
          "name": "SPICEDB_PRESHARED_KEY",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/spicedb-key"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/gatehouse-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "api"
        }
      }
    }
  ]
}
```

**Service Configuration:**

```hcl
resource "aws_ecs_service" "api" {
  name            = "gatehouse-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.api.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"
    container_port   = 8080
  }

  service_registries {
    registry_arn = aws_service_discovery_service.api.arn
  }
}
```

### 2. Worker

The worker processes Temporal workflows and handles LLM interactions.

**Task Definition:**

```json
{
  "family": "gatehouse-worker",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/gatehouse-worker-task-role",
  "containerDefinitions": [
    {
      "name": "worker",
      "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/gatehouse-worker:latest",
      "essential": true,
      "command": ["./gatehouse", "worker"],
      "environment": [
        {"name": "GATEHOUSE_LOG_LEVEL", "value": "info"},
        {"name": "GATEHOUSE_LOG_FORMAT", "value": "json"},
        {"name": "TEMPORAL_ADDRESS", "value": "temporal.gatehouse.local:7233"},
        {"name": "TEMPORAL_NAMESPACE", "value": "default"},
        {"name": "GATEHOUSE_STORAGE_USE_SSL", "value": "true"}
      ],
      "secrets": [
        {
          "name": "GATEHOUSE_DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/database-url"
        },
        {
          "name": "GATEHOUSE_SECRETS_KEY",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/secrets-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/gatehouse-worker",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "worker"
        }
      }
    }
  ]
}
```

### 3. Temporal Server

Option A: **Self-hosted on ECS** (lower cost)

```json
{
  "family": "temporal",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "temporal",
      "image": "temporalio/auto-setup:latest",
      "essential": true,
      "portMappings": [
        {"containerPort": 7233, "protocol": "tcp"}
      ],
      "environment": [
        {"name": "DB", "value": "postgres12"},
        {"name": "DB_PORT", "value": "5432"},
        {"name": "POSTGRES_SEEDS", "value": "gatehouse-db.xxxxx.us-east-1.rds.amazonaws.com"}
      ],
      "secrets": [
        {
          "name": "POSTGRES_USER",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/db-username"
        },
        {
          "name": "POSTGRES_PWD",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/db-password"
        }
      ]
    }
  ]
}
```

Option B: **Temporal Cloud** (managed service)

```hcl
# Configure worker to connect to Temporal Cloud
environment = [
  {"name": "TEMPORAL_ADDRESS", "value": "your-namespace.tmprl.cloud:7233"},
  {"name": "TEMPORAL_NAMESPACE", "value": "your-namespace"},
  {"name": "TEMPORAL_TLS_ENABLED", "value": "true"}
]

secrets = [
  {
    "name": "TEMPORAL_TLS_CERT",
    "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:temporal/tls-cert"
  },
  {
    "name": "TEMPORAL_TLS_KEY",
    "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:temporal/tls-key"
  }
]
```

### 4. SpiceDB (Optional)

SpiceDB provides relationship-based access control:

```json
{
  "family": "spicedb",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "spicedb",
      "image": "authzed/spicedb:latest",
      "essential": true,
      "command": [
        "serve",
        "--grpc-preshared-key=${SPICEDB_PRESHARED_KEY}",
        "--datastore-engine=postgres",
        "--http-enabled=true"
      ],
      "portMappings": [
        {"containerPort": 50051, "protocol": "tcp"},
        {"containerPort": 8443, "protocol": "tcp"}
      ],
      "secrets": [
        {
          "name": "SPICEDB_PRESHARED_KEY",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:gatehouse/spicedb-key"
        }
      ],
      "environment": [
        {"name": "SPICEDB_DATASTORE_CONN_URI", "value": ""}
      ]
    }
  ]
}
```

### 5. TSX Builder

Internal service for bundling React components:

```json
{
  "family": "tsx-builder",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "tsx-builder",
      "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/tsx-builder:latest",
      "essential": true,
      "portMappings": [
        {"containerPort": 3000, "protocol": "tcp"}
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -q --spider http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

### 6. Frontend (S3 + CloudFront)

Build and deploy the frontend to S3:

```bash
# Build frontend
cd frontend
pnpm install
pnpm build

# Deploy to S3
aws s3 sync dist/ s3://gatehouse-frontend-prod/ --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/*"
```

**CloudFront Configuration:**

```hcl
resource "aws_cloudfront_distribution" "frontend" {
  origin {
    domain_name = aws_s3_bucket.frontend.bucket_regional_domain_name
    origin_id   = "S3-frontend"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.frontend.cloudfront_access_identity_path
    }
  }

  enabled             = true
  default_root_object = "index.html"
  aliases             = ["gatehouse.example.com"]

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-frontend"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  # SPA routing - return index.html for 404s
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
}
```

## Database Setup

### RDS PostgreSQL Configuration

```hcl
resource "aws_db_instance" "main" {
  identifier = "gatehouse-prod"

  engine         = "postgres"
  engine_version = "17"
  instance_class = "db.t3.medium"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "gatehouse"
  username = "gatehouse"
  password = random_password.db.result

  multi_az               = true
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  parameter_group_name = aws_db_parameter_group.main.name

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "gatehouse-prod-final"

  performance_insights_enabled = true

  tags = {
    Name        = "gatehouse-prod"
    Environment = "production"
  }
}

resource "aws_db_parameter_group" "main" {
  family = "postgres17"
  name   = "gatehouse-prod"

  # Enable pgvector
  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }

  # Connection pooling optimization
  parameter {
    name  = "max_connections"
    value = "200"
  }
}
```

### Enable pgvector Extension

After RDS is created, connect and enable pgvector:

```bash
# Connect to RDS
psql "postgresql://gatehouse:PASSWORD@gatehouse-prod.xxx.us-east-1.rds.amazonaws.com:5432/gatehouse"

# Enable extensions
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

# Verify
SELECT * FROM pg_extension WHERE extname = 'vector';
```

### Run Migrations

Run migrations using a one-off ECS task:

```bash
# Create task definition for migrations
aws ecs run-task \
  --cluster gatehouse-prod \
  --task-definition gatehouse-migrate \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=DISABLED}"
```

Or via Terraform:

```hcl
resource "null_resource" "migrate" {
  triggers = {
    migration_hash = filesha256("${path.module}/../internal/db/migrations")
  }

  provisioner "local-exec" {
    command = <<-EOT
      aws ecs run-task \
        --cluster ${aws_ecs_cluster.main.name} \
        --task-definition ${aws_ecs_task_definition.migrate.family} \
        --launch-type FARGATE \
        --network-configuration '${jsonencode({
          awsvpcConfiguration = {
            subnets        = aws_subnet.private[*].id
            securityGroups = [aws_security_group.api.id]
            assignPublicIp = "DISABLED"
          }
        })}'
    EOT
  }
}
```

## Storage Configuration

### S3 Buckets

```hcl
# Main storage bucket (artifacts, templates, etc.)
resource "aws_s3_bucket" "main" {
  bucket = "gatehouse-storage-prod"

  tags = {
    Name        = "gatehouse-storage"
    Environment = "production"
  }
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# CORS configuration for browser uploads
resource "aws_s3_bucket_cors_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["https://gatehouse.example.com"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}

# Frontend bucket (static assets)
resource "aws_s3_bucket" "frontend" {
  bucket = "gatehouse-frontend-prod"
}

resource "aws_s3_bucket_website_configuration" "frontend" {
  bucket = aws_s3_bucket.frontend.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "index.html"
  }
}
```

### IAM Policy for S3 Access

```hcl
resource "aws_iam_policy" "s3_access" {
  name = "gatehouse-s3-access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.main.arn,
          "${aws_s3_bucket.main.arn}/*"
        ]
      }
    ]
  })
}
```

## Security Configuration

### Security Groups

```hcl
# ALB Security Group
resource "aws_security_group" "alb" {
  name        = "gatehouse-alb"
  description = "Security group for ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

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

# API Security Group
resource "aws_security_group" "api" {
  name        = "gatehouse-api"
  description = "Security group for API tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# RDS Security Group
resource "aws_security_group" "rds" {
  name        = "gatehouse-rds"
  description = "Security group for RDS"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [
      aws_security_group.api.id,
      aws_security_group.worker.id,
      aws_security_group.temporal.id
    ]
  }
}
```

### Secrets Manager

```hcl
resource "aws_secretsmanager_secret" "database_url" {
  name = "gatehouse/database-url"
}

resource "aws_secretsmanager_secret_version" "database_url" {
  secret_id = aws_secretsmanager_secret.database_url.id
  secret_string = "postgresql://${aws_db_instance.main.username}:${random_password.db.result}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}?sslmode=require"
}

resource "aws_secretsmanager_secret" "jwt_secret" {
  name = "gatehouse/jwt-secret"
}

resource "aws_secretsmanager_secret_version" "jwt_secret" {
  secret_id     = aws_secretsmanager_secret.jwt_secret.id
  secret_string = random_password.jwt.result
}

resource "random_password" "jwt" {
  length  = 64
  special = false
}

resource "aws_secretsmanager_secret" "secrets_key" {
  name = "gatehouse/secrets-key"
}

resource "aws_secretsmanager_secret_version" "secrets_key" {
  secret_id     = aws_secretsmanager_secret.secrets_key.id
  secret_string = base64encode(random_bytes.secrets_key.base64)
}

resource "random_bytes" "secrets_key" {
  length = 32
}
```

## Load Balancer Setup

```hcl
resource "aws_lb" "main" {
  name               = "gatehouse-prod"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_lb_target_group" "api" {
  name        = "gatehouse-api"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 3
  }
}
```

## DNS and SSL

```hcl
# ACM Certificate
resource "aws_acm_certificate" "main" {
  domain_name               = "gatehouse.example.com"
  subject_alternative_names = ["api.gatehouse.example.com"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

# DNS validation
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = var.route53_zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

# Wait for validation
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# API DNS record
resource "aws_route53_record" "api" {
  zone_id = var.route53_zone_id
  name    = "api.gatehouse.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# Frontend DNS record (CloudFront)
resource "aws_route53_record" "frontend" {
  zone_id = var.route53_zone_id
  name    = "gatehouse.example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.frontend.domain_name
    zone_id                = aws_cloudfront_distribution.frontend.hosted_zone_id
    evaluate_target_health = false
  }
}
```

## Service Discovery

Enable internal service discovery for ECS services:

```hcl
resource "aws_service_discovery_private_dns_namespace" "main" {
  name        = "gatehouse.local"
  description = "Private DNS namespace for GateHouse services"
  vpc         = aws_vpc.main.id
}

resource "aws_service_discovery_service" "api" {
  name = "api"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }
  }

  health_check_custom_config {
    failure_threshold = 1
  }
}

resource "aws_service_discovery_service" "temporal" {
  name = "temporal"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }
  }
}
```

## Next Steps

1. **Configure LLM Providers** - See [OpenRouter Setup](./openrouter.md)
2. **Set up Monitoring** - See [Monitoring & Operations](./monitoring.md)
3. **Review Security** - See [Security Hardening](./security.md)
