# Monitoring & Operations Guide

This guide covers monitoring, alerting, and troubleshooting for GateHouse deployments.

## Health Checks

### API Server Health Endpoint

The API server exposes a health check at `/health`:

```bash
curl https://api.gatehouse.example.com/health
```

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "checks": {
    "database": "ok",
    "temporal": "ok",
    "storage": "ok"
  }
}
```

**Health Check Logic:**
- Returns `200 OK` when all checks pass
- Returns `503 Service Unavailable` when any check fails
- Checks database connectivity, Temporal connection, and S3 access

### ALB Health Check Configuration

```hcl
resource "aws_lb_target_group" "api" {
  # ...

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    timeout             = 5
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    matcher             = "200"
  }
}
```

### ECS Service Health

```hcl
resource "aws_ecs_service" "api" {
  # ...

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }
}
```

## CloudWatch Setup

### Log Groups

Create log groups for each service:

```hcl
resource "aws_cloudwatch_log_group" "api" {
  name              = "/ecs/gatehouse-api"
  retention_in_days = 30

  tags = {
    Environment = "production"
    Service     = "api"
  }
}

resource "aws_cloudwatch_log_group" "worker" {
  name              = "/ecs/gatehouse-worker"
  retention_in_days = 30
}

resource "aws_cloudwatch_log_group" "temporal" {
  name              = "/ecs/gatehouse-temporal"
  retention_in_days = 30
}
```

### Log Configuration in Task Definition

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/gatehouse-api",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "api",
      "awslogs-create-group": "true"
    }
  }
}
```

### Key Metrics

#### ECS Metrics

| Metric | Description | Threshold |
|--------|-------------|-----------|
| `CPUUtilization` | CPU usage percentage | Alert > 80% |
| `MemoryUtilization` | Memory usage percentage | Alert > 85% |
| `RunningTaskCount` | Number of running tasks | Alert < desired |

#### ALB Metrics

| Metric | Description | Threshold |
|--------|-------------|-----------|
| `RequestCount` | Total requests | Baseline monitoring |
| `TargetResponseTime` | Response latency | Alert > 2s (p99) |
| `HTTPCode_Target_5XX_Count` | Server errors | Alert > 10/min |
| `HTTPCode_Target_4XX_Count` | Client errors | Monitor trend |
| `HealthyHostCount` | Healthy targets | Alert < 1 |
| `UnHealthyHostCount` | Unhealthy targets | Alert > 0 |

#### RDS Metrics

| Metric | Description | Threshold |
|--------|-------------|-----------|
| `CPUUtilization` | Database CPU | Alert > 80% |
| `FreeableMemory` | Available memory | Alert < 500MB |
| `DatabaseConnections` | Active connections | Alert > 80% of max |
| `ReadLatency` | Read operation latency | Alert > 20ms |
| `WriteLatency` | Write operation latency | Alert > 50ms |
| `FreeStorageSpace` | Available disk space | Alert < 10GB |

### CloudWatch Dashboard

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "gatehouse-production"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "API Response Time"
          region  = "us-east-1"
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", "TargetGroup", aws_lb_target_group.api.arn_suffix, "LoadBalancer", aws_lb.main.arn_suffix, {stat = "p99"}]
          ]
          period = 60
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "Request Count"
          region  = "us-east-1"
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "TargetGroup", aws_lb_target_group.api.arn_suffix, "LoadBalancer", aws_lb.main.arn_suffix, {stat = "Sum"}]
          ]
          period = 60
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 8
        height = 6
        properties = {
          title   = "ECS CPU Utilization"
          region  = "us-east-1"
          metrics = [
            ["AWS/ECS", "CPUUtilization", "ServiceName", "gatehouse-api", "ClusterName", "gatehouse-prod"],
            ["...", "gatehouse-worker", ".", "."]
          ]
          period = 60
        }
      },
      {
        type   = "metric"
        x      = 8
        y      = 6
        width  = 8
        height = 6
        properties = {
          title   = "ECS Memory Utilization"
          region  = "us-east-1"
          metrics = [
            ["AWS/ECS", "MemoryUtilization", "ServiceName", "gatehouse-api", "ClusterName", "gatehouse-prod"],
            ["...", "gatehouse-worker", ".", "."]
          ]
          period = 60
        }
      },
      {
        type   = "metric"
        x      = 16
        y      = 6
        width  = 8
        height = 6
        properties = {
          title   = "RDS Connections"
          region  = "us-east-1"
          metrics = [
            ["AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", "gatehouse-prod"]
          ]
          period = 60
        }
      }
    ]
  })
}
```

### Recommended Alarms

```hcl
# High API Error Rate
resource "aws_cloudwatch_metric_alarm" "api_5xx_errors" {
  alarm_name          = "gatehouse-api-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "API returning 5XX errors"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    TargetGroup  = aws_lb_target_group.api.arn_suffix
    LoadBalancer = aws_lb.main.arn_suffix
  }
}

# High API Latency
resource "aws_cloudwatch_metric_alarm" "api_latency" {
  alarm_name          = "gatehouse-api-high-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  extended_statistic  = "p99"
  threshold           = 2
  alarm_description   = "API p99 latency exceeds 2 seconds"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    TargetGroup  = aws_lb_target_group.api.arn_suffix
    LoadBalancer = aws_lb.main.arn_suffix
  }
}

# No Healthy Hosts
resource "aws_cloudwatch_metric_alarm" "no_healthy_hosts" {
  alarm_name          = "gatehouse-no-healthy-hosts"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "HealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1
  alarm_description   = "No healthy API hosts"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    TargetGroup  = aws_lb_target_group.api.arn_suffix
    LoadBalancer = aws_lb.main.arn_suffix
  }
}

# High ECS CPU
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "gatehouse-ecs-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "ECS CPU utilization high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ServiceName = "gatehouse-api"
    ClusterName = "gatehouse-prod"
  }
}

# RDS Storage Low
resource "aws_cloudwatch_metric_alarm" "rds_storage_low" {
  alarm_name          = "gatehouse-rds-storage-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 10737418240  # 10 GB in bytes
  alarm_description   = "RDS storage below 10GB"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    DBInstanceIdentifier = "gatehouse-prod"
  }
}
```

### SNS Topic for Alerts

```hcl
resource "aws_sns_topic" "alerts" {
  name = "gatehouse-alerts"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = "ops-team@example.com"
}

# Optional: Slack integration via Lambda
resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notifier.arn
}
```

## Temporal Monitoring

### Temporal UI

Access the Temporal UI for workflow visibility:

```
https://temporal-ui.gatehouse.example.com
```

Or port-forward locally:
```bash
kubectl port-forward svc/temporal-ui 8088:8080
```

### Key Temporal Metrics

| Metric | Description | Threshold |
|--------|-------------|-----------|
| `workflow_task_queue_backlog` | Pending workflow tasks | Alert > 100 |
| `activity_task_queue_backlog` | Pending activity tasks | Alert > 100 |
| `workflow_failed_total` | Failed workflows | Alert > 0 |
| `workflow_timeout_total` | Timed out workflows | Alert > 0 |

### Temporal CloudWatch Integration

Export Temporal metrics to CloudWatch:

```yaml
# temporal-config.yaml
metrics:
  prometheus:
    enabled: true
    listenAddress: "0.0.0.0:9090"
```

Use CloudWatch agent to scrape Prometheus endpoint:

```json
{
  "logs": {
    "metrics_collected": {
      "prometheus": {
        "prometheus_config_path": "/etc/prometheus.yml",
        "emf_processor": {
          "metric_namespace": "GateHouse/Temporal"
        }
      }
    }
  }
}
```

### Failed Workflow Alerts

```hcl
resource "aws_cloudwatch_metric_alarm" "workflow_failures" {
  alarm_name          = "gatehouse-workflow-failures"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "workflow_failed_total"
  namespace           = "GateHouse/Temporal"
  period              = 300
  statistic           = "Sum"
  threshold           = 5
  alarm_description   = "Multiple workflow failures detected"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

## Troubleshooting Guide

### API Server Issues

#### API Won't Start

**Symptoms:** Container restarts, health check failures

**Check:**
1. View container logs:
   ```bash
   aws logs tail /ecs/gatehouse-api --follow
   ```

2. Common causes:
   - Missing environment variables
   - Database unreachable
   - Invalid JWT secret

**Fix:**
```bash
# Verify secrets
aws secretsmanager get-secret-value --secret-id gatehouse/database-url

# Test database connectivity from bastion
psql "$DATABASE_URL" -c "SELECT 1"
```

#### High Latency

**Symptoms:** Slow API responses, p99 > 2s

**Check:**
1. Database query performance:
   ```sql
   SELECT query, calls, mean_time
   FROM pg_stat_statements
   ORDER BY mean_time DESC
   LIMIT 10;
   ```

2. Connection pool exhaustion:
   ```sql
   SELECT count(*) FROM pg_stat_activity;
   ```

3. ECS resource utilization

**Fix:**
- Increase connection pool size
- Add database indexes
- Scale ECS tasks
- Increase task CPU/memory

#### 502 Bad Gateway

**Symptoms:** ALB returns 502

**Check:**
1. Target group health:
   ```bash
   aws elbv2 describe-target-health \
     --target-group-arn $TARGET_GROUP_ARN
   ```

2. Container status:
   ```bash
   aws ecs describe-tasks --cluster gatehouse-prod --tasks $TASK_ARN
   ```

**Fix:**
- Check security groups allow ALB -> ECS traffic
- Verify health check path returns 200
- Increase health check timeout

### Worker Issues

#### Worker Crashes

**Symptoms:** Worker restarts frequently, workflows stuck

**Check:**
1. Worker logs:
   ```bash
   aws logs tail /ecs/gatehouse-worker --follow
   ```

2. Memory usage (OOM kills):
   ```bash
   aws ecs describe-tasks --cluster gatehouse-prod --tasks $TASK_ARN
   ```

**Fix:**
- Increase worker memory allocation
- Check for memory leaks in activities
- Review workflow payload sizes

#### Workflows Not Processing

**Symptoms:** Workflows stuck in "Running" state

**Check:**
1. Worker registered with Temporal:
   ```bash
   # In Temporal UI, check Workers tab
   ```

2. Activity queue backlog
3. Worker connectivity to Temporal

**Fix:**
- Restart worker service
- Check Temporal connectivity
- Verify task queue names match

### Database Issues

#### Connection Exhaustion

**Symptoms:** "too many connections" errors

**Check:**
```sql
SELECT count(*) as total,
       state,
       application_name
FROM pg_stat_activity
GROUP BY state, application_name;
```

**Fix:**
- Increase `max_connections` in parameter group
- Reduce connection pool sizes
- Use connection pooler (PgBouncer)

#### Slow Queries

**Symptoms:** High database latency

**Check:**
```sql
-- Enable query logging
ALTER SYSTEM SET log_min_duration_statement = 1000;
SELECT pg_reload_conf();

-- Find slow queries
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;
```

**Fix:**
- Add missing indexes
- Optimize queries
- Increase RDS instance size

### LLM Provider Issues

#### API Key Errors

**Symptoms:** 401 Unauthorized from OpenRouter

**Check:**
1. Verify API key in Secrets Manager
2. Test key directly:
   ```bash
   curl https://openrouter.ai/api/v1/models \
     -H "Authorization: Bearer $API_KEY"
   ```

**Fix:**
- Rotate API key in OpenRouter dashboard
- Update secret in AWS Secrets Manager
- Restart worker to pick up new key

#### Rate Limiting

**Symptoms:** 429 responses, workflows failing

**Check:**
1. OpenRouter dashboard for usage
2. Worker logs for rate limit errors

**Fix:**
- Implement backoff/retry
- Reduce concurrent requests
- Upgrade OpenRouter tier

### Storage Issues

#### S3 Access Denied

**Symptoms:** File upload/download failures

**Check:**
1. Task role permissions:
   ```bash
   aws iam get-role-policy \
     --role-name gatehouse-api-task-role \
     --policy-name s3-access
   ```

2. Bucket policy
3. VPC endpoint configuration

**Fix:**
- Add missing IAM permissions
- Check VPC endpoint routes
- Verify bucket CORS configuration

## Debug Logging

Enable debug logging temporarily:

```bash
# Update task definition with
GATEHOUSE_LOG_LEVEL=debug

# Or use ECS exec to modify running container
aws ecs execute-command \
  --cluster gatehouse-prod \
  --task $TASK_ARN \
  --container api \
  --interactive \
  --command "/bin/sh"

# Inside container
export GATEHOUSE_LOG_LEVEL=debug
```

## Log Analysis

### CloudWatch Insights Queries

**Error Summary:**
```sql
fields @timestamp, @message
| filter @message like /error|ERROR|Error/
| stats count() by bin(1h)
```

**Slow Requests:**
```sql
fields @timestamp, @message
| filter @message like /latency_ms/
| parse @message /latency_ms":(?<latency>\d+)/
| filter latency > 1000
| sort @timestamp desc
| limit 100
```

**Failed Workflows:**
```sql
fields @timestamp, @message
| filter @message like /workflow.*failed/
| sort @timestamp desc
| limit 50
```

## Runbooks

### Scaling Up

1. **Increase ECS Tasks:**
   ```bash
   aws ecs update-service \
     --cluster gatehouse-prod \
     --service gatehouse-api \
     --desired-count 4
   ```

2. **Scale RDS:**
   ```bash
   aws rds modify-db-instance \
     --db-instance-identifier gatehouse-prod \
     --db-instance-class db.t3.large \
     --apply-immediately
   ```

### Emergency Rollback

1. **Rollback ECS Service:**
   ```bash
   aws ecs update-service \
     --cluster gatehouse-prod \
     --service gatehouse-api \
     --task-definition gatehouse-api:PREVIOUS_REVISION
   ```

2. **Rollback Database:**
   ```bash
   # Restore from snapshot
   aws rds restore-db-instance-from-db-snapshot \
     --db-instance-identifier gatehouse-prod-restored \
     --db-snapshot-identifier gatehouse-prod-backup
   ```

### Incident Response

1. **Acknowledge** - Update status page
2. **Triage** - Check dashboards, identify affected services
3. **Mitigate** - Scale up, rollback, or disable problematic features
4. **Communicate** - Update stakeholders
5. **Resolve** - Apply permanent fix
6. **Post-mortem** - Document and improve
