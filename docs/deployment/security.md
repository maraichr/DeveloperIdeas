# Security Hardening Guide

This guide covers security best practices for production GateHouse deployments.

## Overview

GateHouse handles sensitive data including:
- User credentials and API keys
- LLM provider API keys
- Database connections
- Generated code and specifications

This guide ensures proper protection of these assets.

## IAM Policies

### Principle of Least Privilege

Each service should have minimal required permissions.

### ECS Task Execution Role

Role used by ECS to pull images and write logs:

```hcl
resource "aws_iam_role" "ecs_task_execution" {
  name = "gatehouse-ecs-task-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Secrets Manager access
resource "aws_iam_policy" "secrets_read" {
  name = "gatehouse-secrets-read"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          "arn:aws:secretsmanager:${var.region}:${var.account_id}:secret:gatehouse/*"
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "secrets_read" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = aws_iam_policy.secrets_read.arn
}
```

### API Server Task Role

Role assumed by the API container:

```hcl
resource "aws_iam_role" "api_task" {
  name = "gatehouse-api-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# S3 access - specific bucket only
resource "aws_iam_policy" "api_s3" {
  name = "gatehouse-api-s3"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = [
          "${aws_s3_bucket.main.arn}/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.main.arn
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "api_s3" {
  role       = aws_iam_role.api_task.name
  policy_arn = aws_iam_policy.api_s3.arn
}
```

### Worker Task Role

```hcl
resource "aws_iam_role" "worker_task" {
  name = "gatehouse-worker-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# Worker needs Bedrock access (if using)
resource "aws_iam_policy" "worker_bedrock" {
  name = "gatehouse-worker-bedrock"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "bedrock:InvokeModel",
          "bedrock:InvokeModelWithResponseStream"
        ]
        Resource = [
          "arn:aws:bedrock:*::foundation-model/anthropic.*",
          "arn:aws:bedrock:*::foundation-model/amazon.*"
        ]
      }
    ]
  })
}
```

## Network Isolation

### VPC Security

```hcl
# VPC Flow Logs for audit
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/gatehouse-flow-logs"
  retention_in_days = 90
}
```

### Security Groups

#### ALB Security Group

```hcl
resource "aws_security_group" "alb" {
  name        = "gatehouse-alb"
  description = "ALB - HTTPS only"
  vpc_id      = aws_vpc.main.id

  # Allow HTTPS from anywhere
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Redirect HTTP to HTTPS (handled by ALB)
  ingress {
    description = "HTTP redirect"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "To ECS"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    security_groups = [aws_security_group.api.id]
  }

  tags = {
    Name = "gatehouse-alb"
  }
}
```

#### ECS Security Groups

```hcl
resource "aws_security_group" "api" {
  name        = "gatehouse-api"
  description = "API server - ALB traffic only"
  vpc_id      = aws_vpc.main.id

  # Only from ALB
  ingress {
    description     = "From ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # Outbound to RDS
  egress {
    description     = "To RDS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
  }

  # Outbound to Temporal
  egress {
    description     = "To Temporal"
    from_port       = 7233
    to_port         = 7233
    protocol        = "tcp"
    security_groups = [aws_security_group.temporal.id]
  }

  # Outbound HTTPS (for S3, Secrets Manager via VPC endpoints)
  egress {
    description = "HTTPS outbound"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Or restrict to VPC endpoints
  }

  tags = {
    Name = "gatehouse-api"
  }
}

resource "aws_security_group" "worker" {
  name        = "gatehouse-worker"
  description = "Worker - internal only"
  vpc_id      = aws_vpc.main.id

  # No inbound rules - workers don't accept connections

  # Outbound to RDS
  egress {
    description     = "To RDS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
  }

  # Outbound to Temporal
  egress {
    description     = "To Temporal"
    from_port       = 7233
    to_port         = 7233
    protocol        = "tcp"
    security_groups = [aws_security_group.temporal.id]
  }

  # Outbound HTTPS (for LLM providers)
  egress {
    description = "HTTPS outbound"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "gatehouse-worker"
  }
}

resource "aws_security_group" "rds" {
  name        = "gatehouse-rds"
  description = "RDS - ECS traffic only"
  vpc_id      = aws_vpc.main.id

  # Only from ECS services
  ingress {
    description = "From API"
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    security_groups = [
      aws_security_group.api.id,
      aws_security_group.worker.id,
      aws_security_group.temporal.id
    ]
  }

  tags = {
    Name = "gatehouse-rds"
  }
}
```

### VPC Endpoints

Use VPC endpoints to keep traffic private:

```hcl
# S3 Gateway Endpoint
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.region}.s3"

  route_table_ids = aws_route_table.private[*].id

  tags = {
    Name = "gatehouse-s3-endpoint"
  }
}

# Secrets Manager Interface Endpoint
resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.secretsmanager"
  vpc_endpoint_type = "Interface"

  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true

  tags = {
    Name = "gatehouse-secretsmanager-endpoint"
  }
}

# ECR Endpoints
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type = "Interface"

  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type = "Interface"

  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true
}

# CloudWatch Logs Endpoint
resource "aws_vpc_endpoint" "logs" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.logs"
  vpc_endpoint_type = "Interface"

  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true
}
```

## Encryption

### Encryption at Rest

#### RDS Encryption

```hcl
resource "aws_db_instance" "main" {
  # ...
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
}

resource "aws_kms_key" "rds" {
  description             = "KMS key for RDS encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = {
    Name = "gatehouse-rds"
  }
}
```

#### S3 Encryption

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.s3.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

resource "aws_kms_key" "s3" {
  description             = "KMS key for S3 encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```

#### Secrets Manager Encryption

```hcl
resource "aws_secretsmanager_secret" "main" {
  name       = "gatehouse/database-url"
  kms_key_id = aws_kms_key.secrets.arn
}

resource "aws_kms_key" "secrets" {
  description             = "KMS key for Secrets Manager"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```

### Encryption in Transit

#### ALB HTTPS

```hcl
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
```

#### RDS SSL

```bash
# Force SSL connections
GATEHOUSE_DATABASE_URL=postgresql://user:pass@host:5432/db?sslmode=verify-full
```

```hcl
resource "aws_db_parameter_group" "main" {
  family = "postgres17"

  parameter {
    name  = "rds.force_ssl"
    value = "1"
  }
}
```

## Secret Management

### Secrets Rotation

#### RDS Password Rotation

```hcl
resource "aws_secretsmanager_secret_rotation" "rds" {
  secret_id           = aws_secretsmanager_secret.rds_password.id
  rotation_lambda_arn = aws_lambda_function.rotate_rds.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

#### API Key Rotation

For application secrets (JWT, encryption keys):

1. **Add new secret version:**
   ```bash
   aws secretsmanager put-secret-value \
     --secret-id gatehouse/jwt-secret \
     --secret-string "new-secret-value" \
     --version-stage AWSPENDING
   ```

2. **Deploy new version to ECS:**
   ```bash
   aws ecs update-service \
     --cluster gatehouse-prod \
     --service gatehouse-api \
     --force-new-deployment
   ```

3. **Promote new secret:**
   ```bash
   aws secretsmanager update-secret-version-stage \
     --secret-id gatehouse/jwt-secret \
     --version-stage AWSCURRENT \
     --move-to-version-id $NEW_VERSION_ID
   ```

### What Should NEVER Be in Environment Variables

- Database passwords (use Secrets Manager)
- API keys (use Secrets Manager or database)
- Encryption keys (use Secrets Manager)
- TLS private keys (use Secrets Manager or ACM)

### What's Acceptable in Environment Variables

- Service URLs (non-sensitive)
- Feature flags
- Log levels
- Region configuration

## Container Security

### Image Scanning

```hcl
resource "aws_ecr_repository" "api" {
  name = "gatehouse-api"

  image_scanning_configuration {
    scan_on_push = true
  }

  image_tag_mutability = "IMMUTABLE"
}
```

### Read-Only Root Filesystem

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "readonlyRootFilesystem": true,
      "mountPoints": [
        {
          "sourceVolume": "tmp",
          "containerPath": "/tmp"
        }
      ]
    }
  ],
  "volumes": [
    {
      "name": "tmp"
    }
  ]
}
```

### Non-Root User

In Dockerfile:
```dockerfile
FROM golang:1.23-alpine AS builder
# Build...

FROM alpine:3.19
RUN addgroup -g 1000 gatehouse && \
    adduser -u 1000 -G gatehouse -s /bin/sh -D gatehouse

USER gatehouse
COPY --from=builder /app/gatehouse /app/gatehouse
ENTRYPOINT ["/app/gatehouse"]
```

### Security Context

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "user": "1000:1000",
      "privileged": false,
      "linuxParameters": {
        "capabilities": {
          "drop": ["ALL"]
        }
      }
    }
  ]
}
```

## Audit Logging

### CloudTrail

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "gatehouse-audit"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["${aws_s3_bucket.main.arn}/"]
    }
  }
}
```

### Application Audit Logs

GateHouse logs security-relevant events:

```go
// Logged events
- User login/logout
- Failed authentication attempts
- API key creation/deletion
- Permission changes
- Sensitive data access
```

Configure log retention:

```hcl
resource "aws_cloudwatch_log_group" "api" {
  name              = "/ecs/gatehouse-api"
  retention_in_days = 365  # 1 year for audit
}
```

### Log Analysis

CloudWatch Insights query for security events:

```sql
fields @timestamp, @message
| filter @message like /auth|permission|access|denied|forbidden/
| sort @timestamp desc
| limit 1000
```

## Web Application Security

### HTTP Security Headers

GateHouse API sets security headers:

```go
// Security headers middleware
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        next.ServeHTTP(w, r)
    })
}
```

### CORS Configuration

```go
// Restrict CORS to known origins
cors := handlers.CORS(
    handlers.AllowedOrigins([]string{"https://gatehouse.example.com"}),
    handlers.AllowedMethods([]string{"GET", "POST", "PUT", "DELETE", "OPTIONS"}),
    handlers.AllowedHeaders([]string{"Authorization", "Content-Type"}),
    handlers.AllowCredentials(),
)
```

### Rate Limiting

```hcl
# AWS WAF rate limiting
resource "aws_wafv2_web_acl" "main" {
  name  = "gatehouse-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "rate-limit"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "gatehouse-waf"
    sampled_requests_enabled   = true
  }
}

resource "aws_wafv2_web_acl_association" "main" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

## Security Checklist

### Pre-Deployment

- [ ] All secrets in Secrets Manager
- [ ] IAM roles follow least privilege
- [ ] Security groups restrict traffic
- [ ] RDS encryption enabled
- [ ] S3 bucket encryption enabled
- [ ] SSL/TLS certificates valid
- [ ] Container images scanned
- [ ] VPC endpoints configured

### Post-Deployment

- [ ] CloudTrail enabled
- [ ] VPC Flow Logs enabled
- [ ] CloudWatch alarms for security events
- [ ] WAF rules configured
- [ ] Penetration testing scheduled
- [ ] Incident response plan documented

### Ongoing

- [ ] Rotate secrets every 30 days
- [ ] Review IAM policies quarterly
- [ ] Update container images monthly
- [ ] Audit access logs weekly
- [ ] Security patches applied promptly
