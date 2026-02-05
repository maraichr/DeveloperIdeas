# OpenRouter Setup Guide

This guide covers configuring OpenRouter as the LLM provider for GateHouse.

## Overview

OpenRouter provides access to multiple AI models through a unified API, including:
- Anthropic (Claude)
- OpenAI (GPT-4)
- Google (Gemini)
- Meta (Llama)
- Mistral
- And many more

GateHouse supports OpenRouter as the recommended LLM provider for production deployments.

## Getting Started

### 1. Create an OpenRouter Account

1. Go to [openrouter.ai](https://openrouter.ai)
2. Sign up or log in
3. Navigate to **Keys** in the dashboard

### 2. Generate an API Key

1. Click **Create Key**
2. Give the key a descriptive name (e.g., "GateHouse Production")
3. Set usage limits if desired
4. Copy the API key (starts with `sk-or-`)

### 3. Add Credits

OpenRouter uses a pay-as-you-go model:
1. Go to **Credits** in the dashboard
2. Add funds via credit card or crypto
3. Monitor usage in the **Activity** tab

## Configuration Methods

### Method 1: Database Configuration (Recommended)

Configure providers through the GateHouse UI for runtime management:

1. Log into GateHouse
2. Navigate to **Settings > AI Providers**
3. Click **Add Provider**
4. Select **OpenRouter**
5. Enter your API key and configure settings

**Benefits:**
- No restart required for changes
- API keys encrypted in database
- Supports multiple provider configurations
- Audit trail for changes

### Method 2: Environment Variables (Fallback)

For CLI usage or when database configuration isn't available:

```bash
# Required
CODEGEN_PROVIDER=openrouter
CODEGEN_API_KEY=sk-or-v1-your-api-key-here

# Optional
CODEGEN_URL=https://openrouter.ai/api/v1
CODEGEN_MODEL=anthropic/claude-sonnet-4
```

### Method 3: API Endpoint

Update provider configuration via API:

```bash
curl -X POST https://api.gatehouse.example.com/api/v1/settings/providers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "openrouter",
    "type": "openrouter",
    "config": {
      "api_key": "sk-or-v1-xxx",
      "default_model": "anthropic/claude-sonnet-4"
    }
  }'
```

## Model Selection

### Model Identifier Format

OpenRouter uses the format `provider/model-name`:

```
anthropic/claude-sonnet-4
openai/gpt-4o
google/gemini-2.0-flash-001
meta-llama/llama-3.3-70b-instruct
```

### Recommended Models by Use Case

| Use Case | Recommended Model | Notes |
|----------|-------------------|-------|
| General coding | `anthropic/claude-sonnet-4` | Best balance of quality and cost |
| Complex reasoning | `anthropic/claude-opus-4` | Highest quality, higher cost |
| Fast responses | `anthropic/claude-3-5-haiku-latest` | Quick, low cost |
| Budget-friendly | `meta-llama/llama-3.3-70b-instruct` | Good quality, lower cost |
| Long context | `google/gemini-2.0-flash-001` | 1M token context |

### Model Pricing

Prices vary by model. Check [openrouter.ai/models](https://openrouter.ai/models) for current pricing.

**Example Costs (per 1M tokens):**

| Model | Input | Output |
|-------|-------|--------|
| claude-sonnet-4 | $3.00 | $15.00 |
| claude-opus-4 | $15.00 | $75.00 |
| gpt-4o | $2.50 | $10.00 |
| llama-3.3-70b | $0.40 | $0.40 |

## Production Configuration

### Database Setup

Store provider configuration in the `system_settings` table:

```sql
INSERT INTO system_settings (key, value, description)
VALUES (
  'llm_providers',
  '{
    "providers": [
      {
        "name": "openrouter",
        "type": "openrouter",
        "is_default": true,
        "config": {
          "base_url": "https://openrouter.ai/api/v1",
          "default_model": "anthropic/claude-sonnet-4"
        }
      }
    ]
  }',
  'LLM provider configurations'
);
```

> **Note:** API keys are stored separately and encrypted using `GATEHOUSE_SECRETS_KEY`.

### Worker Configuration

The worker reads provider configurations from the database on startup:

```go
// Worker loads providers from database
providers, err := db.GetSystemSetting(ctx, "llm_providers")
```

No environment variables needed for the worker in production.

### Secrets Management

Store the OpenRouter API key in AWS Secrets Manager:

```bash
aws secretsmanager create-secret \
  --name gatehouse/openrouter-api-key \
  --secret-string "sk-or-v1-your-api-key"
```

Reference in ECS task definition:

```json
{
  "secrets": [
    {
      "name": "CODEGEN_API_KEY",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:gatehouse/openrouter-api-key"
    }
  ]
}
```

## Rate Limiting

### OpenRouter Limits

OpenRouter has rate limits based on your account tier:

| Tier | Requests/min | Tokens/min |
|------|--------------|------------|
| Free | 10 | 100,000 |
| Paid | 500 | 5,000,000 |
| Enterprise | Custom | Custom |

### GateHouse Rate Limiting

Configure rate limiting in GateHouse to stay within limits:

```yaml
# In provider configuration
rate_limit:
  requests_per_minute: 400  # Below OpenRouter limit
  tokens_per_minute: 4000000
  concurrent_requests: 50
```

### Handling Rate Limits

GateHouse automatically handles rate limit responses (429):

1. Exponential backoff with jitter
2. Retry up to 3 times
3. Queue requests when approaching limits

## Fallback Configuration

Configure fallback providers for resilience:

```json
{
  "providers": [
    {
      "name": "openrouter-primary",
      "type": "openrouter",
      "is_default": true,
      "priority": 1,
      "config": {
        "default_model": "anthropic/claude-sonnet-4"
      }
    },
    {
      "name": "openrouter-fallback",
      "type": "openrouter",
      "priority": 2,
      "config": {
        "default_model": "openai/gpt-4o"
      }
    }
  ]
}
```

GateHouse automatically falls back to lower-priority providers on failure.

## Cost Monitoring

### OpenRouter Dashboard

Monitor costs in the OpenRouter dashboard:
- Real-time usage graphs
- Per-model breakdown
- Daily/monthly summaries

### Set Usage Limits

Configure limits in OpenRouter:
1. Go to **Keys**
2. Edit your API key
3. Set **Monthly Limit** (e.g., $500)

### CloudWatch Metrics

Track LLM costs with CloudWatch custom metrics:

```go
// Worker emits metrics
cloudwatch.PutMetricData(&cloudwatch.PutMetricDataInput{
    Namespace: "GateHouse/LLM",
    MetricData: []*cloudwatch.MetricDatum{
        {
            MetricName: "TokensUsed",
            Value:      aws.Float64(float64(tokensUsed)),
            Dimensions: []*cloudwatch.Dimension{
                {Name: aws.String("Provider"), Value: aws.String("openrouter")},
                {Name: aws.String("Model"), Value: aws.String(model)},
            },
        },
    },
})
```

### Cost Alerts

Set up CloudWatch alarms for cost thresholds:

```hcl
resource "aws_cloudwatch_metric_alarm" "llm_cost_high" {
  alarm_name          = "gatehouse-llm-cost-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "EstimatedCost"
  namespace           = "GateHouse/LLM"
  period              = 3600  # 1 hour
  statistic           = "Sum"
  threshold           = 50    # $50/hour
  alarm_description   = "LLM costs exceeding threshold"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

## Request/Response Logging

### Enable Debug Logging

For troubleshooting, enable debug logging:

```bash
GATEHOUSE_LOG_LEVEL=debug
```

This logs:
- Request payloads (prompts truncated)
- Response metadata
- Token counts
- Latency

### Structured Logging

LLM requests are logged with structured fields:

```json
{
  "level": "info",
  "msg": "LLM request completed",
  "provider": "openrouter",
  "model": "anthropic/claude-sonnet-4",
  "input_tokens": 1500,
  "output_tokens": 500,
  "latency_ms": 2500,
  "workflow_id": "wf-123",
  "trace_id": "abc-123"
}
```

### Audit Trail

All LLM requests are recorded in the database for audit:

```sql
SELECT
  created_at,
  provider,
  model,
  input_tokens,
  output_tokens,
  cost_usd
FROM llm_requests
WHERE created_at > NOW() - INTERVAL '1 day'
ORDER BY created_at DESC;
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 401 Unauthorized | Invalid API key | Verify key in OpenRouter dashboard |
| 429 Rate Limited | Too many requests | Implement rate limiting |
| 500 Model Error | Model unavailable | Use fallback model |
| Timeout | Slow response | Increase timeout, use faster model |

### Verify Configuration

Test OpenRouter connectivity:

```bash
curl https://openrouter.ai/api/v1/models \
  -H "Authorization: Bearer $CODEGEN_API_KEY"
```

### Check Worker Logs

```bash
# ECS logs
aws logs tail /ecs/gatehouse-worker --follow

# Look for provider initialization
grep "LLM provider" /var/log/gatehouse/worker.log
```

### Test Chat Endpoint

```bash
curl -X POST https://api.gatehouse.example.com/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hello, test message",
    "model": "anthropic/claude-sonnet-4"
  }'
```

## Best Practices

1. **Use Database Configuration** - Avoid environment variables for API keys in production
2. **Set Usage Limits** - Prevent runaway costs with OpenRouter limits
3. **Configure Fallbacks** - Ensure availability with backup providers
4. **Monitor Costs** - Set up alerts before costs get out of control
5. **Use Appropriate Models** - Match model capability to task requirements
6. **Enable Logging** - Track usage for debugging and optimization
7. **Rotate API Keys** - Regularly rotate keys for security
