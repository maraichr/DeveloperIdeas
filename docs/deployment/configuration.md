# Configuration Reference

This document provides a complete reference for all GateHouse configuration options.

## Configuration Methods

GateHouse configuration can be provided through:

1. **Environment Variables** - Primary method for production
2. **`.env` File** - Automatically loaded if present (development)
3. **Secrets Manager** - Recommended for sensitive values in AWS
4. **Database (system_settings)** - Runtime-configurable settings (LLM providers)

## Required Environment Variables

These variables must be set for GateHouse to start:

| Variable | Description | Example |
|----------|-------------|---------|
| `GATEHOUSE_DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/gatehouse?sslmode=require` |
| `GATEHOUSE_JWT_SECRET` | JWT signing key (min 32 chars) | `your-secure-random-string-minimum-32-characters` |

## Complete Environment Variable Reference

### Database

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_DATABASE_URL` | Yes | - | PostgreSQL connection URL |
| `POSTGRES_HOST` | No | - | PostgreSQL host (for health checks) |
| `POSTGRES_PORT` | No | `5432` | PostgreSQL port |
| `POSTGRES_USER` | No | - | PostgreSQL username |

**Connection String Format:**
```
postgresql://[user]:[password]@[host]:[port]/[database]?sslmode=[mode]
```

**SSL Modes:**
- `disable` - No SSL (development only)
- `require` - SSL required, no verification
- `verify-ca` - SSL required, verify CA
- `verify-full` - SSL required, verify CA and hostname (recommended)

**Example:**
```bash
GATEHOUSE_DATABASE_URL=postgresql://gatehouse:secret@db.example.com:5432/gatehouse?sslmode=verify-full
```

### Server

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_PORT` | No | `8080` | HTTP server port |
| `GATEHOUSE_LOG_LEVEL` | No | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `GATEHOUSE_LOG_FORMAT` | No | `json` | Log format: `json`, `text` |

### Authentication

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_JWT_SECRET` | Yes | - | JWT signing key |
| `GATEHOUSE_JWT_EXPIRY_HOURS` | No | `24` | JWT token expiry in hours |

**Generating a Secure JWT Secret:**
```bash
openssl rand -base64 48
```

### Secrets Encryption

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_SECRETS_KEY` | No | - | Base64-encoded 32-byte AES-256 key |

Used to encrypt sensitive data stored in the database (API keys, credentials).

**Generating a Secrets Key:**
```bash
openssl rand -base64 32
```

### Temporal (Workflow Orchestration)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `TEMPORAL_ADDRESS` | No | `localhost:7233` | Temporal server address |
| `TEMPORAL_NAMESPACE` | No | `default` | Temporal namespace |

**For Temporal Cloud:**
```bash
TEMPORAL_ADDRESS=your-namespace.tmprl.cloud:7233
TEMPORAL_NAMESPACE=your-namespace
TEMPORAL_TLS_ENABLED=true
TEMPORAL_TLS_CERT=/path/to/cert.pem
TEMPORAL_TLS_KEY=/path/to/key.pem
```

### Storage (S3/MinIO)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_STORAGE_ENDPOINT` | No | `localhost:9000` | S3 endpoint (without protocol) |
| `GATEHOUSE_STORAGE_PUBLIC_ENDPOINT` | No | Same as endpoint | Public endpoint for browser access |
| `GATEHOUSE_STORAGE_ACCESS_KEY` | No | - | S3 access key |
| `GATEHOUSE_STORAGE_SECRET_KEY` | No | - | S3 secret key |
| `GATEHOUSE_STORAGE_BUCKET` | No | `gatehouse` | Primary bucket name |
| `GATEHOUSE_STORAGE_USE_SSL` | No | `false` | Use HTTPS for S3 |

**AWS S3 Configuration:**
```bash
GATEHOUSE_STORAGE_ENDPOINT=s3.us-east-1.amazonaws.com
GATEHOUSE_STORAGE_USE_SSL=true
# Access keys from IAM role (no env vars needed) or:
GATEHOUSE_STORAGE_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE
GATEHOUSE_STORAGE_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**MinIO (Development):**
```bash
GATEHOUSE_STORAGE_ENDPOINT=minio:9000
GATEHOUSE_STORAGE_ACCESS_KEY=gatehouse
GATEHOUSE_STORAGE_SECRET_KEY=gatehouse123
GATEHOUSE_STORAGE_USE_SSL=false
```

### SpiceDB (Authorization)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SPICEDB_ENDPOINT` | No | `localhost:50051` | SpiceDB gRPC endpoint |
| `SPICEDB_PRESHARED_KEY` | No | - | SpiceDB authentication key |
| `SPICEDB_INSECURE` | No | `true` | Disable TLS (development only) |

### LLM Providers (CLI/Legacy)

> **Note:** In production, LLM providers are configured via the UI at **Settings > AI Providers**. The worker reads configurations from the database. These environment variables are only for CLI usage.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `CODEGEN_PROVIDER` | No | `openrouter` | Provider: `openrouter`, `bedrock`, `openai`, `lmstudio` |
| `CODEGEN_URL` | No | Provider default | API endpoint URL |
| `CODEGEN_MODEL` | No | `anthropic/claude-3.5-sonnet` | Model identifier |
| `CODEGEN_API_KEY` | No | - | Provider API key |
| `CODEGEN_BEDROCK_REGION` | No | `us-east-1` | AWS Bedrock region |
| `OPENAI_API_KEY` | No | - | OpenAI API key |
| `ANTHROPIC_API_KEY` | No | - | Anthropic API key |

### Embeddings

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `EMBEDDING_URL` | No | `https://api.openai.com/v1` | Embeddings API endpoint |
| `EMBEDDING_API_KEY` | No | - | Embeddings API key |
| `EMBEDDING_MODEL` | No | `text-embedding-3-small` | Model for embeddings |
| `GATEHOUSE_EMBEDDING_DIMENSIONS` | No | `1536` | Vector dimensions |

### Sandbox (Code Execution)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_SANDBOX_ENABLED` | No | `false` | Enable sandbox execution |
| `GATEHOUSE_SANDBOX_IMAGE` | No | `gatehouse-sandbox:latest` | Sandbox Docker image |
| `GATEHOUSE_SANDBOX_NETWORK` | No | `gatehouse_default` | Docker network for sandbox |
| `GATEHOUSE_SANDBOX_BUCKET` | No | `sandbox` | S3 bucket for sandbox outputs |
| `GATEHOUSE_SANDBOX_DEFAULT_MEMORY_MB` | No | `512` | Default memory limit |
| `GATEHOUSE_SANDBOX_MAX_MEMORY_MB` | No | `2048` | Maximum memory limit |
| `GATEHOUSE_SANDBOX_DEFAULT_TIMEOUT_SEC` | No | `300` | Default execution timeout |
| `GATEHOUSE_SANDBOX_MAX_TIMEOUT_SEC` | No | `600` | Maximum execution timeout |

> **Warning:** Sandbox execution requires Docker socket access. Not supported on Fargate without modifications. See [Special Considerations](#sandbox-execution-on-fargate).

### TSX Builder

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_TSX_BUILD_SERVICE_URL` | No | `http://tsx-builder:3000` | TSX builder service URL |

### Skills

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_SKILLS_DIRECTORIES` | No | `./skills/system,./skills/custom` | Comma-separated skill directories |

### Icons (D2 Diagrams)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_ICON_PACK_PATH` | No | `./icons` | Icon pack directory |
| `GATEHOUSE_ICON_CACHE_PATH` | No | `./cache/icon-catalog.json` | Icon catalog cache |
| `GATEHOUSE_ICON_SIZE` | No | `48` | Default icon size |

### Git Repositories

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GATEHOUSE_GIT_REPOS_PATH` | No | `./data/repos` | Directory for cloned repositories |

### Development

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AUTO_MIGRATE` | No | `false` | Run database migrations on startup |

> **Warning:** Set `AUTO_MIGRATE=false` in production. Run migrations explicitly via CI/CD.

## AWS Secrets Manager Integration

For production deployments, store sensitive values in AWS Secrets Manager:

### Recommended Secrets

| Secret Name | Contents |
|-------------|----------|
| `gatehouse/database-url` | Full PostgreSQL connection URL |
| `gatehouse/jwt-secret` | JWT signing key |
| `gatehouse/secrets-key` | Base64-encoded AES-256 key |
| `gatehouse/spicedb-key` | SpiceDB preshared key |
| `gatehouse/openrouter-api-key` | OpenRouter API key (if using env vars) |

### ECS Task Definition Example

```json
{
  "secrets": [
    {
      "name": "GATEHOUSE_DATABASE_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:gatehouse/database-url"
    },
    {
      "name": "GATEHOUSE_JWT_SECRET",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:gatehouse/jwt-secret"
    }
  ]
}
```

### IAM Policy for Secrets Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789:secret:gatehouse/*"
      ]
    }
  ]
}
```

## Database Configuration

### Connection Pooling

For production, configure connection pooling appropriately:

```bash
# Connection string with pool settings
GATEHOUSE_DATABASE_URL=postgresql://user:pass@host:5432/gatehouse?sslmode=require&pool_max_conns=25&pool_min_conns=5
```

**Recommended Pool Sizes:**

| Deployment Size | API Server | Worker |
|-----------------|------------|--------|
| Small | 10-25 | 5-10 |
| Medium | 25-50 | 10-25 |
| Large | 50-100 | 25-50 |

### pgvector Setup

GateHouse uses pgvector for embeddings. Enable it after database creation:

```sql
-- Connect as superuser
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify installation
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**RDS Parameter Group:**

For RDS, pgvector is available on PostgreSQL 15+. No parameter group changes needed; just run the `CREATE EXTENSION` command.

### Migration Strategy

1. **Development:** Use `AUTO_MIGRATE=true`
2. **Production:** Run migrations explicitly

**Manual Migration:**
```bash
# Build migration binary
go build -o migrate cmd/migrate/main.go

# Run migrations
./migrate -database "$GATEHOUSE_DATABASE_URL" up
```

**CI/CD Migration (GitHub Actions):**
```yaml
- name: Run Migrations
  run: |
    aws ecs run-task \
      --cluster ${{ env.ECS_CLUSTER }} \
      --task-definition gatehouse-migrate \
      --launch-type FARGATE \
      --network-configuration '...'
```

## Special Considerations

### Sandbox Execution on Fargate

Fargate doesn't support Docker socket access. Options:

1. **Disable Sandbox:**
   ```bash
   GATEHOUSE_SANDBOX_ENABLED=false
   ```

2. **Use EC2 Launch Type:**
   Configure ECS with EC2 instances that have Docker installed.

3. **AWS Lambda (Future):**
   Modify sandbox to use Lambda for code execution.

### Multi-Region Deployment

For multi-region deployments:

```bash
# Region-specific configuration
GATEHOUSE_STORAGE_ENDPOINT=s3.eu-west-1.amazonaws.com
CODEGEN_BEDROCK_REGION=eu-west-1
```

### High Availability

For HA deployments:

- Use Multi-AZ RDS
- Deploy 2+ API server replicas
- Deploy 2+ worker replicas
- Use Temporal Cloud or multi-node Temporal

## Environment File Templates

### Development (.env)

```bash
# Database
GATEHOUSE_DATABASE_URL=postgresql://gatehouse:gatehouse@localhost:5432/gatehouse

# Server
GATEHOUSE_PORT=8080
GATEHOUSE_LOG_LEVEL=debug
GATEHOUSE_LOG_FORMAT=text

# Authentication
GATEHOUSE_JWT_SECRET=dev-secret-change-in-production
GATEHOUSE_JWT_EXPIRY_HOURS=24

# Temporal
TEMPORAL_ADDRESS=localhost:7233
TEMPORAL_NAMESPACE=default

# Storage (MinIO)
GATEHOUSE_STORAGE_ENDPOINT=localhost:9000
GATEHOUSE_STORAGE_ACCESS_KEY=gatehouse
GATEHOUSE_STORAGE_SECRET_KEY=gatehouse123
GATEHOUSE_STORAGE_USE_SSL=false

# SpiceDB
SPICEDB_ENDPOINT=localhost:50051
SPICEDB_PRESHARED_KEY=dev-key
SPICEDB_INSECURE=true

# Development
AUTO_MIGRATE=true
```

### Production

See `.env.production.example` for a complete production template.

## Validation

GateHouse validates configuration on startup and fails fast if required variables are missing:

```
Error: missing required configuration: GATEHOUSE_DATABASE_URL, GATEHOUSE_JWT_SECRET
```

Check configuration with:

```bash
# Dry run (validates config without starting server)
./gatehouse validate-config
```
