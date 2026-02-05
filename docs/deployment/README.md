# GateHouse Deployment Guide

This guide covers deploying GateHouse to production environments, with a focus on AWS ECS Fargate.

## System Architecture

```
                                    ┌─────────────────────────────────────────────────────────────┐
                                    │                         AWS Cloud                            │
                                    │  ┌─────────────────────────────────────────────────────────┐│
                                    │  │                     Public Subnet                        ││
     ┌──────────┐                   │  │  ┌─────────────┐                                        ││
     │  Users   │◄──────HTTPS───────┼──┼──┤     ALB     │                                        ││
     └──────────┘                   │  │  └──────┬──────┘                                        ││
                                    │  │         │                                               ││
     ┌──────────┐                   │  │  ┌──────┴──────┐                                        ││
     │ Browser  │◄──────HTTPS───────┼──┼──┤ CloudFront  │◄────┐                                  ││
     └──────────┘                   │  │  └─────────────┘     │                                  ││
                                    │  └──────────────────────┼──────────────────────────────────┘│
                                    │                         │                                   │
                                    │  ┌──────────────────────┼──────────────────────────────────┐│
                                    │  │                Private Subnet                           ││
                                    │  │         ┌────────────┴───────────┐                      ││
                                    │  │         │                        │                      ││
                                    │  │    ┌────▼────┐             ┌─────▼─────┐                ││
                                    │  │    │   API   │             │  Frontend │                ││
                                    │  │    │ Server  │             │   (S3)    │                ││
                                    │  │    │  (ECS)  │             └───────────┘                ││
                                    │  │    └────┬────┘                                          ││
                                    │  │         │                                               ││
                                    │  │    ┌────▼────┐         ┌───────────┐                    ││
                                    │  │    │ Worker  │◄───────►│  Temporal │                    ││
                                    │  │    │  (ECS)  │         │   (ECS)   │                    ││
                                    │  │    └────┬────┘         └─────┬─────┘                    ││
                                    │  │         │                    │                          ││
                                    │  │    ┌────▼────────────────────▼────┐                     ││
                                    │  │    │         PostgreSQL           │                     ││
                                    │  │    │      (RDS + pgvector)        │                     ││
                                    │  │    └─────────────────────────────┬┘                     ││
                                    │  │                                   │                      ││
                                    │  │    ┌──────────────┐    ┌─────────▼───┐                  ││
                                    │  │    │  TSX Builder │    │   SpiceDB   │                  ││
                                    │  │    │    (ECS)     │    │   (ECS)     │                  ││
                                    │  │    └──────────────┘    └─────────────┘                  ││
                                    │  │                                                         ││
                                    │  │    ┌──────────────┐    ┌─────────────┐                  ││
                                    │  │    │      S3      │    │   Secrets   │                  ││
                                    │  │    │   Buckets    │    │   Manager   │                  ││
                                    │  │    └──────────────┘    └─────────────┘                  ││
                                    │  └─────────────────────────────────────────────────────────┘│
                                    └─────────────────────────────────────────────────────────────┘
```

## Service Topology

| Service | Description | Dependencies |
|---------|-------------|--------------|
| **Frontend** | React SPA (Vite) | API Server |
| **API Server** | Go HTTP API | PostgreSQL, S3, Temporal, SpiceDB, TSX Builder |
| **Worker** | Temporal workflow worker | PostgreSQL, S3, Temporal, LLM Providers |
| **Temporal** | Workflow orchestration | PostgreSQL |
| **SpiceDB** | Authorization (Zanzibar) | PostgreSQL |
| **TSX Builder** | React component bundler | None |
| **PostgreSQL** | Primary database + pgvector | - |
| **S3** | Object storage | - |

## Quick Start (Local Docker Compose)

For testing the deployment configuration locally before AWS:

```bash
# Start all services
docker compose up -d

# Check service health
docker compose ps

# View logs
docker compose logs -f api worker

# Access services
# - Frontend: http://localhost:5173
# - API: http://localhost:8080
# - Temporal UI: http://localhost:8088
```

## Prerequisites

### AWS Resources

- AWS Account with administrative access
- Route 53 hosted zone (for DNS)
- ACM certificate (for HTTPS)

### Tools Required

```bash
# Install required CLI tools
brew install terraform awscli

# Configure AWS credentials
aws configure

# Verify access
aws sts get-caller-identity
```

### Permissions

The deployment requires IAM permissions for:
- ECS (clusters, services, task definitions)
- RDS (instances, parameter groups, subnet groups)
- S3 (buckets, policies)
- VPC (subnets, security groups, NAT gateways)
- Secrets Manager (secrets)
- CloudWatch (log groups, alarms)
- ACM (certificates)
- Route 53 (records)
- ALB (load balancers, target groups)

## Deployment Guides

| Guide | Description |
|-------|-------------|
| [AWS ECS Deployment](./aws-ecs.md) | Complete AWS ECS Fargate deployment guide |
| [Configuration Reference](./configuration.md) | All environment variables and settings |
| [OpenRouter Setup](./openrouter.md) | LLM provider configuration |
| [Monitoring & Operations](./monitoring.md) | CloudWatch, alerts, troubleshooting |
| [Security Hardening](./security.md) | IAM policies, encryption, best practices |

## Deployment Order

```
1. Infrastructure (Terraform)
   ├── VPC, Subnets, NAT Gateway
   ├── Security Groups
   └── IAM Roles

2. Data Layer
   ├── RDS PostgreSQL (enable pgvector)
   ├── S3 Buckets
   └── Secrets Manager

3. Platform Services
   ├── ECS Cluster
   ├── Temporal Server
   └── SpiceDB

4. Application Services
   ├── TSX Builder
   ├── API Server (run migrations)
   └── Worker

5. Frontend
   ├── Build static assets
   ├── Deploy to S3
   └── Configure CloudFront
```

## AWS Service Mapping

| GateHouse Component | AWS Service | Notes |
|--------------------|-------------|-------|
| PostgreSQL + pgvector | RDS PostgreSQL 17 | Enable pgvector extension |
| Object Storage | S3 | Replaces MinIO |
| API Server | ECS Fargate | Behind ALB |
| Worker | ECS Fargate | Internal only |
| Frontend | S3 + CloudFront | Static hosting |
| Temporal | ECS Fargate or Temporal Cloud | Self-hosted or managed |
| SpiceDB | ECS Fargate | Internal only |
| TSX Builder | ECS Fargate | Internal only |
| Secrets | Secrets Manager | JWT, API keys |
| Logs | CloudWatch Logs | Container logs |
| DNS | Route 53 | Domain management |
| TLS | ACM | Free certificates |
| Load Balancer | ALB | HTTPS termination |

## Cost Estimation

Approximate monthly costs for a small production deployment:

| Service | Configuration | Est. Cost |
|---------|--------------|-----------|
| ECS Fargate | 4 services, 0.5 vCPU, 1GB each | ~$60 |
| RDS PostgreSQL | db.t3.medium, Multi-AZ | ~$70 |
| ALB | 1 load balancer | ~$20 |
| NAT Gateway | 1 gateway | ~$35 |
| S3 | 50GB storage | ~$2 |
| CloudFront | 100GB transfer | ~$10 |
| Secrets Manager | 5 secrets | ~$2 |
| CloudWatch | Logs + alarms | ~$10 |
| **Total** | | **~$210/month** |

Costs scale with usage. Production deployments with higher availability requirements will cost more.

## Verification Checklist

After deployment, verify:

- [ ] RDS accessible from ECS tasks
- [ ] API health check passes (`/health`)
- [ ] Frontend loads and connects to API
- [ ] Temporal UI accessible (if deployed)
- [ ] Worker starts and registers with Temporal
- [ ] OpenRouter API key works (test chat)
- [ ] S3 uploads/downloads work
- [ ] CloudWatch logs flowing
- [ ] Alarms configured and tested
- [ ] SSL certificate valid

## Troubleshooting

### Common Issues

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| API won't start | Database unreachable | Check security groups, RDS status |
| Worker crashes | Temporal unreachable | Verify Temporal service health |
| Frontend 502 | API not healthy | Check API logs, health endpoint |
| S3 access denied | IAM policy missing | Review task role permissions |
| Slow responses | Resource constraints | Increase Fargate CPU/memory |

See [Monitoring & Operations](./monitoring.md) for detailed troubleshooting.

## Support

- [GitHub Issues](https://github.com/maraichr/gatehouse/issues)
- [Architecture Decision Records](../ADR/)
