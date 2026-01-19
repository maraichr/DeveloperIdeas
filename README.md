# The GateHouse Methodology: AI-Augmented Microservices Development

## A Schema-Driven Approach to Vibe Coding at Scale

> **Philosophy**: Define once in YAML, generate everywhere. Let humans focus on business logic while AI handles the boilerplate orchestration.

---

## Executive Summary

This guide presents a **declarative, schema-first approach** to building production-grade microservices. By capturing business requirements in a structured YAML specification and leveraging intelligent code generation, we achieve:

- **90% reduction** in boilerplate code
- **Consistent patterns** across all services
- **Type-safe SDK generation** for rapid frontend development
- **Self-documenting architecture** that stays in sync with implementation

```mermaid
graph LR
    subgraph "Human Domain"
        A[Business Requirements] --> B[Data Model]
        B --> C[App YAML Spec]
    end
    
    subgraph "GateHouse Orchestration"
        C --> D{YAML Parser<br/>+ Validator}
        D -->|Errors| E[Agent Fixes]
        E -->|Retry ≤10x| D
        D -->|Valid| F[Code Generator]
    end
    
    subgraph "Generated Artifacts"
        F --> G[Go Backend]
        F --> H[TypeScript SDK]
        F --> I[Docker Compose]
        F --> J[React Scaffolding]
    end
    
    style A fill:#e1f5fe
    style C fill:#fff9c4
    style D fill:#f3e5f5
    style F fill:#c8e6c9
```

---

## Table of Contents

1. [Philosophy: Why Schema-Driven Development](#1-philosophy-why-schema-driven-development)
2. [Enterprise Architecture Foundation](#2-enterprise-architecture-foundation)
3. [Business Requirements Document](#3-business-requirements-document)
4. [Data Model Reference](#4-data-model-reference)
5. [The App YAML Specification](#5-the-app-yaml-specification)
6. [The Validation Loop](#6-the-validation-loop)
7. [Code Generation Pipeline](#7-code-generation-pipeline)
8. [Frontend Development Acceleration](#8-frontend-development-acceleration)
9. [GateHouse Orchestration Platform](#9-gatehouse-orchestration-platform)
10. [Putting It All Together](#10-putting-it-all-together)

---

## 1. Philosophy: Why Schema-Driven Development

### The Vibe Coding Manifesto

Traditional development treats code as the source of truth. This approach inverts that relationship: **the specification is the source of truth**, and code is a derived artifact.

```mermaid
graph TB
    subgraph "Traditional Approach"
        T1[Write Code] --> T2[Document After]
        T2 --> T3[Docs Drift]
        T3 --> T4[Confusion]
    end
    
    subgraph "Schema-Driven Approach"
        S1[Define Spec] --> S2[Generate Code]
        S1 --> S3[Generate Docs]
        S2 --> S4[Always in Sync]
        S3 --> S4
    end
    
    style T4 fill:#ffcdd2
    style S4 fill:#c8e6c9
```

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Declarative Over Imperative** | Describe *what* you want, not *how* to build it |
| **Single Source of Truth** | The YAML spec drives everything else |
| **Fail Fast, Fix Fast** | Validation loop catches errors before generation |
| **Human-AI Collaboration** | Humans define intent; AI handles implementation |
| **Composable Patterns** | Reusable templates for common patterns |

### The YAML Advantage

YAML serves as the **universal interface** between human intent and machine execution:

```yaml
# Humans can read and write this
entity: Subcontractor
fields:
  - name: company_name
    type: string
    required: true
    validation: "len >= 2 && len <= 200"
  - name: status
    type: enum
    values: [pending, approved, suspended]
    default: pending

# Machines can parse and generate from this
state_machine:
  initial: pending
  transitions:
    - from: pending
      to: approved
      action: approve
      requires: [has_insurance, has_license]
```

---

## 2. Enterprise Architecture Foundation

Before diving into individual services, establish the architectural context that enables horizontal scaling.

### Multi-Environment Topology

```mermaid
graph TB
    subgraph "Development"
        DEV_GH[GateHouse<br/>Orchestrator]
        DEV_API[API Services<br/>Docker Compose]
        DEV_DB[(PostgreSQL<br/>Per-Tenant)]
        DEV_GH --> DEV_API
        DEV_API --> DEV_DB
    end
    
    subgraph "Staging"
        STG_GH[GateHouse<br/>Orchestrator]
        STG_K8S[Kubernetes<br/>Cluster]
        STG_RDS[(RDS PostgreSQL<br/>Multi-AZ)]
        STG_GH --> STG_K8S
        STG_K8S --> STG_RDS
    end
    
    subgraph "Production"
        PROD_GH[GateHouse<br/>Orchestrator]
        PROD_K8S[Kubernetes<br/>Multi-Region]
        PROD_RDS[(RDS Aurora<br/>Global)]
        PROD_CDN[CloudFront<br/>CDN]
        PROD_GH --> PROD_K8S
        PROD_K8S --> PROD_RDS
        PROD_CDN --> PROD_K8S
    end
    
    DEV_GH -.->|Promote| STG_GH
    STG_GH -.->|Promote| PROD_GH
    
    style DEV_GH fill:#e3f2fd
    style STG_GH fill:#fff3e0
    style PROD_GH fill:#e8f5e9
```

### Service Mesh Architecture

Each microservice follows a consistent internal structure while participating in a larger mesh:

```mermaid
graph TB
    subgraph "API Gateway Layer"
        GW[Kong / Traefik<br/>Rate Limiting, Auth]
    end
    
    subgraph "Service Mesh"
        subgraph "Service A"
            A_API[REST API]
            A_SM[State Machine]
            A_DB[(Tenant DB)]
            A_API --> A_SM
            A_SM --> A_DB
        end
        
        subgraph "Service B"
            B_API[REST API]
            B_SM[State Machine]
            B_DB[(Tenant DB)]
            B_API --> B_SM
            B_SM --> B_DB
        end
        
        subgraph "Service C"
            C_API[REST API]
            C_SM[State Machine]
            C_DB[(Tenant DB)]
            C_API --> C_SM
            C_SM --> C_DB
        end
    end
    
    subgraph "Shared Infrastructure"
        VAULT[Vault<br/>Secrets]
        REDIS[Redis<br/>Cache]
        QUEUE[Message Queue<br/>Events]
    end
    
    GW --> A_API
    GW --> B_API
    GW --> C_API
    
    A_API --> VAULT
    A_API --> REDIS
    A_API --> QUEUE
    
    B_API -.->|Events| QUEUE
    C_API -.->|Events| QUEUE
    
    style GW fill:#ffecb3
    style VAULT fill:#fff9c4
    style QUEUE fill:#e1bee7
```

### Database-Per-Tenant Model

For SaaS applications requiring strict data isolation:

```mermaid
graph TB
    subgraph "Request Flow"
        REQ[Incoming Request<br/>JWT: tenant_id=abc123]
    end
    
    subgraph "Tenant Router"
        META[(Metadata DB<br/>Tenant Registry)]
        ROUTER[Connection Router]
        REQ --> ROUTER
        ROUTER --> META
    end
    
    subgraph "Tenant Databases"
        DB1[(tenant_abc123<br/>Acme Corp)]
        DB2[(tenant_def456<br/>Global Ltd)]
        DB3[(tenant_ghi789<br/>Tech Inc)]
    end
    
    subgraph "Secrets"
        VAULT[Vault<br/>DB Credentials]
    end
    
    META -->|Lookup| ROUTER
    VAULT -->|Credentials| ROUTER
    ROUTER -->|Route| DB1
    
    style REQ fill:#e3f2fd
    style ROUTER fill:#fff3e0
    style DB1 fill:#c8e6c9
```

### Infrastructure as Code Alignment

The generated Docker Compose and Kubernetes manifests align with your enterprise standards:

```yaml
# Generated docker-compose.yml follows enterprise patterns
services:
  subcontractor-service:
    image: ${REGISTRY}/subcontractor-service:${VERSION}
    environment:
      - VAULT_ADDR=${VAULT_ADDR}
      - DATABASE_HOST=${DB_HOST}
      - TENANT_MODE=per-database
    depends_on:
      postgres-metadata:
        condition: service_healthy
      vault:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3
```

---

## 3. Business Requirements Document

### The BRD as Foundation

The Business Requirements Document bridges the gap between stakeholder intent and technical implementation. In AI-augmented development, the BRD becomes **machine-readable context**.

```mermaid
graph LR
    subgraph "Input Sources"
        STAKE[Stakeholder<br/>Interviews]
        LEGACY[Legacy<br/>Codebase]
        DOCS[Existing<br/>Documentation]
    end
    
    subgraph "AI Processing"
        INDEX[Semantic<br/>Indexing]
        EXTRACT[Requirement<br/>Extraction]
        STRUCT[Structure<br/>Generation]
    end
    
    subgraph "Output"
        BRD[Business Requirements<br/>Document]
    end
    
    STAKE --> EXTRACT
    LEGACY --> INDEX
    INDEX --> EXTRACT
    DOCS --> EXTRACT
    EXTRACT --> STRUCT
    STRUCT --> BRD
    
    style LEGACY fill:#ffecb3
    style INDEX fill:#e1bee7
    style BRD fill:#c8e6c9
```

### Legacy Migration: Treating Old as New

When migrating from legacy systems, **semantic indexing** allows us to treat the migration like a greenfield project:

```mermaid
sequenceDiagram
    participant LC as Legacy Codebase
    participant SI as Semantic Index
    participant AI as AI Agent
    participant BRD as BRD Generator
    
    LC->>SI: Index source files
    Note over SI: Chunk by function,<br/>class, module
    LC->>SI: Index database schemas
    LC->>SI: Index API endpoints
    
    AI->>SI: Query: "What entities exist?"
    SI-->>AI: Entities + relationships
    
    AI->>SI: Query: "What business rules apply to X?"
    SI-->>AI: Validation logic, state transitions
    
    AI->>BRD: Generate structured requirements
    BRD-->>AI: Draft BRD
    
    Note over AI,BRD: Human review<br/>and refinement
```

### BRD Structure for Code Generation

A BRD optimized for downstream code generation includes:

```markdown
## 1. Domain Overview
- Business context and objectives
- Key stakeholders and their needs
- Success metrics

## 2. Entity Definitions
For each entity:
- Name and description
- Attributes with types and constraints
- Relationships to other entities
- Lifecycle states

## 3. Business Rules
- Validation rules (field-level, entity-level)
- State transition rules
- Authorization rules
- Calculation rules

## 4. Integration Requirements
- External system interfaces
- Event triggers and handlers
- Batch processing needs

## 5. Non-Functional Requirements
- Performance expectations
- Audit and compliance needs
- Multi-tenancy requirements
```

### Example: Subcontractor Management BRD Extract

```markdown
### Entity: Subcontractor

**Description**: External contractors who perform work on behalf of the organization.

**Attributes**:
| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| company_name | string | yes | 2-200 chars |
| tax_id | string | yes | Valid EIN format |
| status | enum | yes | pending, approved, suspended, terminated |
| insurance_expiry | date | no | Must be future date when set |
| trade_codes | array[string] | yes | Min 1, from trades lookup |

**Lifecycle States**:
- pending → approved (requires: insurance_verified, license_verified)
- approved → suspended (trigger: insurance_expired OR compliance_violation)
- suspended → approved (requires: issue_resolved)
- any → terminated (requires: admin_approval)

**Business Rules**:
1. Cannot create work orders for suspended subcontractors
2. Insurance expiry triggers automatic suspension 30 days prior
3. Trade codes must match project requirements for assignment
```

---

## 4. Data Model Reference

### From BRD to Schema

The data model translates business requirements into a precise database schema, serving as the structural foundation for code generation.

```mermaid
graph TB
    subgraph "BRD Entities"
        E1[Subcontractor]
        E2[Document]
        E3[WorkOrder]
        E1 --> E3
        E2 --> E1
    end
    
    subgraph "Data Model"
        T1[subcontractors<br/>table]
        T2[documents<br/>table]
        T3[work_orders<br/>table]
        T4[subcontractor_trades<br/>junction]
        T5[trades<br/>lookup]
        
        T1 --> T4
        T5 --> T4
        T2 --> T1
        T3 --> T1
    end
    
    E1 -.->|Maps to| T1
    E2 -.->|Maps to| T2
    E3 -.->|Maps to| T3
    
    style E1 fill:#e3f2fd
    style T1 fill:#c8e6c9
```

### Schema Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Audit Everything** | `created_at`, `updated_at`, `created_by`, `updated_by` on all tables |
| **Soft Deletes** | `deleted_at` timestamp instead of physical deletes |
| **UUID Primary Keys** | Prevents enumeration attacks, enables distributed generation |
| **Enum as Tables** | Reference tables for type-safe enumeration with descriptions |
| **Temporal Data** | Effective dates for time-sensitive business data |

### Data Model Reference Document Structure

```yaml
# data_model_reference.yaml

metadata:
  version: "1.0.0"
  database: postgresql
  schema: tenant

tables:
  subcontractors:
    description: "External contractors who perform work"
    columns:
      - name: id
        type: uuid
        primary_key: true
        default: gen_random_uuid()
      - name: company_name
        type: varchar(200)
        nullable: false
        index: true
      - name: tax_id
        type: varchar(20)
        nullable: false
        unique: true
        encrypted: true  # Signals PII handling
      - name: status
        type: varchar(20)
        nullable: false
        default: "'pending'"
        check: "status IN ('pending', 'approved', 'suspended', 'terminated')"
      - name: insurance_expiry_date
        type: date
        nullable: true
      # Standard audit columns
      - name: created_at
        type: timestamptz
        nullable: false
        default: now()
      - name: updated_at
        type: timestamptz
        nullable: false
        default: now()
      - name: deleted_at
        type: timestamptz
        nullable: true
    
    indexes:
      - name: idx_subcontractors_status
        columns: [status]
        where: "deleted_at IS NULL"
      - name: idx_subcontractors_insurance_expiry
        columns: [insurance_expiry_date]
        where: "deleted_at IS NULL AND status = 'approved'"
    
    triggers:
      - name: set_updated_at
        timing: BEFORE UPDATE
        function: update_timestamp()

relationships:
  subcontractor_documents:
    type: one_to_many
    from: subcontractors.id
    to: documents.subcontractor_id
    on_delete: CASCADE
    
  subcontractor_trades:
    type: many_to_many
    through: subcontractor_trade_assignments
    from: subcontractors.id
    to: trades.id
```

### Migration Generation

The data model reference enables automatic migration generation:

```mermaid
graph LR
    DMR[Data Model<br/>Reference v1.0] --> DIFF{Diff Engine}
    DMR2[Data Model<br/>Reference v1.1] --> DIFF
    DIFF --> MIG[Migration<br/>000023_add_insurance_fields.sql]
    
    style DMR fill:#e3f2fd
    style DMR2 fill:#fff3e0
    style MIG fill:#c8e6c9
```

---

## 5. The App YAML Specification

### The Heart of Generation

The App YAML file is the **central artifact** that drives all code generation. It synthesizes the BRD and Data Model into a machine-parseable specification.

```mermaid
graph TB
    subgraph "Inputs"
        BRD[Business Requirements<br/>Document]
        DMR[Data Model<br/>Reference]
        TMPL[YAML Template<br/>Schema]
    end
    
    subgraph "Agent Processing"
        AGENT[GateHouse Agent<br/>YAML Generator]
    end
    
    subgraph "Output"
        YAML[App YAML<br/>Specification]
    end
    
    BRD --> AGENT
    DMR --> AGENT
    TMPL --> AGENT
    AGENT --> YAML
    
    style AGENT fill:#e1bee7
    style YAML fill:#c8e6c9
```

### YAML Schema Structure

```yaml
# app.yaml - The complete service specification

meta:
  name: subcontractor-service
  version: "1.0.0"
  description: "Manages subcontractor lifecycle and compliance"
  
config:
  database:
    type: postgresql
    multi_tenant: true
    tenant_isolation: database  # database | schema | row
  auth:
    provider: keycloak
    jwt_claims: [tenant_id, user_id, roles]
  cache:
    provider: redis
    default_ttl: 300

entities:
  - name: Subcontractor
    table: subcontractors
    description: "External contractor organization"
    
    fields:
      - name: id
        type: uuid
        primary_key: true
        generated: true
        
      - name: company_name
        type: string
        required: true
        min_length: 2
        max_length: 200
        searchable: true
        
      - name: tax_id
        type: string
        required: true
        unique: true
        sensitive: true  # Triggers encryption/masking
        pattern: "^\\d{2}-\\d{7}$"
        
      - name: status
        type: enum
        required: true
        values:
          - value: pending
            description: "Awaiting initial approval"
          - value: approved
            description: "Cleared to receive work orders"
          - value: suspended
            description: "Temporarily blocked"
          - value: terminated
            description: "Permanently deactivated"
        default: pending
        
      - name: insurance_expiry_date
        type: date
        required: false
        future_only: true
        triggers_event: insurance_expiring
        event_threshold_days: 30
        
      - name: rating
        type: decimal
        precision: 3
        scale: 2
        min: 0.00
        max: 5.00
        computed: true
        computation: "AVG(work_order_ratings.rating)"
    
    state_machine:
      field: status
      initial: pending
      
      transitions:
        - name: approve
          from: [pending, suspended]
          to: approved
          guards:
            - insurance_not_expired
            - has_required_documents
          actions:
            - send_approval_notification
          permissions: [admin, compliance_officer]
          
        - name: suspend
          from: [approved]
          to: suspended
          guards: []  # Can always suspend
          actions:
            - cancel_pending_work_orders
            - send_suspension_notification
          permissions: [admin, compliance_officer]
          
        - name: terminate
          from: [pending, approved, suspended]
          to: terminated
          guards:
            - no_active_work_orders
          actions:
            - archive_documents
            - send_termination_notification
          permissions: [admin]
    
    validations:
      - name: insurance_not_expired
        condition: "insurance_expiry_date IS NULL OR insurance_expiry_date > CURRENT_DATE"
        message: "Insurance must not be expired"
        
      - name: has_required_documents
        condition: "EXISTS(SELECT 1 FROM documents WHERE subcontractor_id = $id AND type IN ('w9', 'insurance_cert') AND status = 'verified')"
        message: "Required documents (W9, Insurance Certificate) must be verified"
    
    hooks:
      before_create:
        - validate_tax_id_format
        - check_duplicate_tax_id
      after_create:
        - create_audit_log
        - trigger_compliance_check
      before_update:
        - validate_state_transition
      after_update:
        - create_audit_log
        - sync_to_external_system
    
    api:
      endpoints:
        list:
          enabled: true
          filterable: [status, trade_codes, created_at]
          sortable: [company_name, rating, created_at]
          searchable: [company_name, tax_id]
          pagination: cursor
          
        get:
          enabled: true
          includes: [documents, trades, work_orders]
          
        create:
          enabled: true
          required_fields: [company_name, tax_id, trade_codes]
          
        update:
          enabled: true
          immutable_fields: [tax_id]
          
        delete:
          enabled: true
          soft_delete: true
          
        # Custom endpoints
        custom:
          - name: approve
            method: POST
            path: "/:id/approve"
            action: state_transition
            transition: approve
            
          - name: bulk_import
            method: POST
            path: "/import"
            action: bulk_create
            max_batch_size: 100

  - name: Document
    table: documents
    # ... document entity definition
    
  - name: Trade
    table: trades
    # ... trade entity definition

relationships:
  - name: subcontractor_documents
    type: has_many
    from: Subcontractor
    to: Document
    foreign_key: subcontractor_id
    cascade_delete: true
    
  - name: subcontractor_trades
    type: many_to_many
    from: Subcontractor
    to: Trade
    through: subcontractor_trade_assignments
    
events:
  - name: subcontractor.created
    payload: [id, company_name, status]
    
  - name: subcontractor.status_changed
    payload: [id, old_status, new_status, changed_by]
    
  - name: insurance_expiring
    schedule: "0 9 * * *"  # Daily at 9 AM
    query: |
      SELECT id, company_name, insurance_expiry_date
      FROM subcontractors
      WHERE status = 'approved'
        AND insurance_expiry_date BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '30 days'
```

### YAML Template Schema

The agent follows a strict template schema that defines all valid constructs:

```yaml
# template_schema.yaml - What the agent must follow

required_sections:
  - meta
  - config
  - entities

meta:
  required: [name, version]
  optional: [description, team, repository]

config:
  database:
    type: enum[postgresql, mysql]
    multi_tenant: boolean
    tenant_isolation: enum[database, schema, row]
  auth:
    provider: enum[keycloak, auth0, cognito, custom]
    jwt_claims: array[string]

entities:
  type: array
  items:
    required: [name, table, fields]
    optional: [state_machine, validations, hooks, api]
    
    fields:
      type: array
      items:
        required: [name, type]
        type_specific:
          string:
            optional: [min_length, max_length, pattern, searchable]
          enum:
            required: [values]
            values_structure: [value, description]
          decimal:
            optional: [precision, scale, min, max]
```

---

## 6. The Validation Loop

### Fail Fast, Fix Fast

The validation loop ensures the YAML specification is correct before any code generation begins. This prevents cascading errors and wasted generation cycles.

```mermaid
graph TB
    subgraph "Agent Generation"
        AGENT[GateHouse Agent] --> YAML[App YAML Draft]
    end
    
    subgraph "Validation Pipeline"
        YAML --> V1{Schema<br/>Validation}
        V1 -->|Pass| V2{Semantic<br/>Validation}
        V1 -->|Fail| E1[Schema Errors]
        V2 -->|Pass| V3{Cross-Reference<br/>Validation}
        V2 -->|Fail| E2[Semantic Errors]
        V3 -->|Pass| SUCCESS[Valid YAML]
        V3 -->|Fail| E3[Reference Errors]
    end
    
    subgraph "Error Feedback Loop"
        E1 --> FEEDBACK[Detailed Error Report]
        E2 --> FEEDBACK
        E3 --> FEEDBACK
        FEEDBACK --> COUNTER{Attempts < 10?}
        COUNTER -->|Yes| AGENT
        COUNTER -->|No| HUMAN[Human Intervention<br/>Required]
    end
    
    style SUCCESS fill:#c8e6c9
    style HUMAN fill:#ffcdd2
    style FEEDBACK fill:#fff3e0
```

### Validation Layers

#### Layer 1: Schema Validation

Ensures structural correctness against the template schema:

```yaml
# Example schema errors
errors:
  - path: entities[0].fields[3]
    error: "Missing required property 'type'"
    suggestion: "Add 'type: string' or appropriate type"
    
  - path: entities[0].state_machine.transitions[0].from
    error: "Value 'active' not in enum values [pending, approved, suspended, terminated]"
    suggestion: "Use one of the defined status values"
```

#### Layer 2: Semantic Validation

Validates business logic coherence:

```yaml
# Example semantic errors
errors:
  - path: entities[0].fields[5].computation
    error: "Computed field references non-existent table 'work_order_ratings'"
    suggestion: "Define the 'work_order_ratings' relationship or correct the table name"
    
  - path: entities[0].state_machine.transitions[1].guards[0]
    error: "Guard 'insurance_not_expired' references field 'insurance_expiry_date' but field allows NULL without default"
    suggestion: "Add null handling to guard condition or make field required"
```

#### Layer 3: Cross-Reference Validation

Validates consistency with Data Model Reference:

```yaml
# Example cross-reference errors
errors:
  - path: entities[0].table
    error: "Table 'subcontractors' exists in data model but column 'rating' is not defined"
    suggestion: "Add column to data model or mark field as 'virtual: true'"
    
  - path: relationships[0].foreign_key
    error: "Foreign key 'subcontractor_id' not found in 'documents' table schema"
    suggestion: "Add foreign key to data model reference"
```

### Error Report Format

The error report is structured for optimal agent comprehension:

```yaml
# validation_report.yaml

summary:
  status: failed
  attempt: 3
  total_errors: 5
  by_severity:
    critical: 2
    warning: 2
    info: 1

errors:
  - id: ERR001
    severity: critical
    category: schema
    path: entities[0].fields[3].type
    message: "Invalid field type 'string[]'. Use 'array' with 'items: string'"
    current_value: "string[]"
    expected: "type: array\nitems:\n  type: string"
    documentation_ref: "https://docs.gatehouse.io/yaml-schema#array-fields"
    auto_fixable: true
    suggested_fix:
      old: |
        - name: trade_codes
          type: string[]
      new: |
        - name: trade_codes
          type: array
          items:
            type: string
            
  - id: ERR002
    severity: critical
    category: semantic
    path: entities[0].state_machine.transitions[0]
    message: "Transition 'approve' has guard 'has_required_documents' but no validation with this name is defined"
    context:
      defined_validations: [insurance_not_expired]
      referenced_guards: [insurance_not_expired, has_required_documents]
    suggested_fix: "Add validation definition for 'has_required_documents'"
```

### The Temporal Workflow

GateHouse uses Temporal to orchestrate the validation loop:

```mermaid
sequenceDiagram
    participant W as Temporal Workflow
    participant A as Agent
    participant V as Validator
    participant H as Human (if needed)
    
    W->>A: Generate YAML from inputs
    A-->>W: YAML Draft
    
    loop Max 10 attempts
        W->>V: Validate YAML
        V-->>W: Validation Result
        
        alt Valid
            W->>W: Proceed to Generation
        else Invalid & attempts < 10
            W->>A: Fix errors (with report)
            A-->>W: Updated YAML
        else Invalid & attempts >= 10
            W->>H: Request human intervention
            H-->>W: Fixed YAML
        end
    end
```

---

## 7. Code Generation Pipeline

### Template-Driven Generation

Once validation passes, the YAML specification feeds into a template engine that generates production-ready code.

```mermaid
graph TB
    subgraph "Inputs"
        YAML[Valid App YAML]
        TPL_GO[Go Templates]
        TPL_TS[TypeScript Templates]
        TPL_DOCKER[Docker Templates]
    end
    
    subgraph "Generator Engine"
        PARSE[YAML Parser]
        ENRICH[Context Enricher]
        GEN[Template Engine]
        
        YAML --> PARSE
        PARSE --> ENRICH
        ENRICH --> GEN
        TPL_GO --> GEN
        TPL_TS --> GEN
        TPL_DOCKER --> GEN
    end
    
    subgraph "Generated Artifacts"
        GO_API[Go API Server]
        GO_SM[State Machine]
        GO_REPO[Repository Layer]
        TS_SDK[TypeScript SDK]
        TS_TYPES[Type Definitions]
        DOCKER[Docker Compose]
        OAS[OpenAPI Spec]
    end
    
    GEN --> GO_API
    GEN --> GO_SM
    GEN --> GO_REPO
    GEN --> TS_SDK
    GEN --> TS_TYPES
    GEN --> DOCKER
    GEN --> OAS
    
    style YAML fill:#fff9c4
    style GEN fill:#e1bee7
    style GO_API fill:#c8e6c9
    style TS_SDK fill:#bbdefb
```

### Generated Go Structure

```
subcontractor-service/
├── cmd/
│   └── server/
│       └── main.go                 # Entry point with DI setup
├── internal/
│   ├── api/
│   │   ├── handlers/
│   │   │   ├── subcontractor.go    # HTTP handlers
│   │   │   └── document.go
│   │   ├── middleware/
│   │   │   ├── auth.go             # JWT validation
│   │   │   ├── tenant.go           # Tenant context
│   │   │   └── logging.go
│   │   └── router.go               # Chi router setup
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── subcontractor.go    # Entity structs
│   │   │   └── document.go
│   │   ├── events/
│   │   │   └── subcontractor.go    # Domain events
│   │   └── statemachine/
│   │       └── subcontractor.go    # State transitions
│   ├── repository/
│   │   ├── subcontractor.go        # DB operations
│   │   └── document.go
│   └── service/
│       ├── subcontractor.go        # Business logic
│       └── document.go
├── pkg/
│   └── sdk/
│       └── typescript/             # Generated TS SDK
│           ├── src/
│           │   ├── client.ts
│           │   ├── types.ts
│           │   └── subcontractor.ts
│           └── package.json
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── api/
│   └── openapi.yaml                # Generated OpenAPI spec
└── go.mod
```

### Example Generated Code

#### Go Entity with State Machine

```go
// internal/domain/entities/subcontractor.go
// Generated by GateHouse - DO NOT EDIT

package entities

import (
    "time"
    "github.com/google/uuid"
)

// SubcontractorStatus represents the lifecycle states
type SubcontractorStatus string

const (
    SubcontractorStatusPending    SubcontractorStatus = "pending"
    SubcontractorStatusApproved   SubcontractorStatus = "approved"
    SubcontractorStatusSuspended  SubcontractorStatus = "suspended"
    SubcontractorStatusTerminated SubcontractorStatus = "terminated"
)

// Subcontractor represents an external contractor organization
type Subcontractor struct {
    ID                  uuid.UUID           `json:"id" db:"id"`
    CompanyName         string              `json:"company_name" db:"company_name"`
    TaxID               string              `json:"tax_id" db:"tax_id"`
    Status              SubcontractorStatus `json:"status" db:"status"`
    InsuranceExpiryDate *time.Time          `json:"insurance_expiry_date,omitempty" db:"insurance_expiry_date"`
    Rating              *float64            `json:"rating,omitempty" db:"rating"`
    CreatedAt           time.Time           `json:"created_at" db:"created_at"`
    UpdatedAt           time.Time           `json:"updated_at" db:"updated_at"`
    DeletedAt           *time.Time          `json:"deleted_at,omitempty" db:"deleted_at"`
}

// internal/domain/statemachine/subcontractor.go
// Generated by GateHouse - DO NOT EDIT

package statemachine

import (
    "context"
    "fmt"
    "myapp/internal/domain/entities"
)

// SubcontractorStateMachine manages status transitions
type SubcontractorStateMachine struct {
    guardCheckers map[string]GuardChecker
    actionRunners map[string]ActionRunner
}

// Transition defines a valid state transition
type Transition struct {
    Name        string
    From        []entities.SubcontractorStatus
    To          entities.SubcontractorStatus
    Guards      []string
    Actions     []string
    Permissions []string
}

var subcontractorTransitions = []Transition{
    {
        Name:        "approve",
        From:        []entities.SubcontractorStatus{entities.SubcontractorStatusPending, entities.SubcontractorStatusSuspended},
        To:          entities.SubcontractorStatusApproved,
        Guards:      []string{"insurance_not_expired", "has_required_documents"},
        Actions:     []string{"send_approval_notification"},
        Permissions: []string{"admin", "compliance_officer"},
    },
    {
        Name:        "suspend",
        From:        []entities.SubcontractorStatus{entities.SubcontractorStatusApproved},
        To:          entities.SubcontractorStatusSuspended,
        Guards:      []string{},
        Actions:     []string{"cancel_pending_work_orders", "send_suspension_notification"},
        Permissions: []string{"admin", "compliance_officer"},
    },
    {
        Name:        "terminate",
        From:        []entities.SubcontractorStatus{entities.SubcontractorStatusPending, entities.SubcontractorStatusApproved, entities.SubcontractorStatusSuspended},
        To:          entities.SubcontractorStatusTerminated,
        Guards:      []string{"no_active_work_orders"},
        Actions:     []string{"archive_documents", "send_termination_notification"},
        Permissions: []string{"admin"},
    },
}

// CanTransition checks if a transition is valid
func (sm *SubcontractorStateMachine) CanTransition(ctx context.Context, entity *entities.Subcontractor, transitionName string) (bool, error) {
    transition, err := sm.findTransition(transitionName)
    if err != nil {
        return false, err
    }
    
    // Check current state is valid source
    if !sm.isValidSource(entity.Status, transition.From) {
        return false, fmt.Errorf("cannot %s from status %s", transitionName, entity.Status)
    }
    
    // Check all guards
    for _, guardName := range transition.Guards {
        checker, ok := sm.guardCheckers[guardName]
        if !ok {
            return false, fmt.Errorf("guard %s not registered", guardName)
        }
        if allowed, err := checker.Check(ctx, entity); !allowed || err != nil {
            return false, err
        }
    }
    
    return true, nil
}

// Execute performs the transition with all actions
func (sm *SubcontractorStateMachine) Execute(ctx context.Context, entity *entities.Subcontractor, transitionName string) error {
    // ... implementation
}
```

#### TypeScript SDK

```typescript
// pkg/sdk/typescript/src/client.ts
// Generated by GateHouse - DO NOT EDIT

import { Subcontractor, SubcontractorCreate, SubcontractorUpdate, SubcontractorStatus } from './types';
import { PaginatedResponse, QueryParams } from './pagination';

export interface SubcontractorServiceConfig {
  baseUrl: string;
  getToken: () => Promise<string>;
}

export class SubcontractorClient {
  private config: SubcontractorServiceConfig;
  
  constructor(config: SubcontractorServiceConfig) {
    this.config = config;
  }
  
  /**
   * List subcontractors with filtering and pagination
   * @param params - Query parameters for filtering, sorting, and pagination
   * @returns Paginated list of subcontractors
   */
  async list(params?: QueryParams<Subcontractor>): Promise<PaginatedResponse<Subcontractor>> {
    const response = await this.fetch('/api/v1/subcontractors', {
      method: 'GET',
      params,
    });
    return response.json();
  }
  
  /**
   * Get a single subcontractor by ID
   * @param id - Subcontractor UUID
   * @param includes - Related entities to include
   * @returns Subcontractor with optional related data
   */
  async get(id: string, includes?: ('documents' | 'trades' | 'work_orders')[]): Promise<Subcontractor> {
    const params = includes ? { include: includes.join(',') } : undefined;
    const response = await this.fetch(`/api/v1/subcontractors/${id}`, {
      method: 'GET',
      params,
    });
    return response.json();
  }
  
  /**
   * Create a new subcontractor
   * @param data - Subcontractor creation data
   * @returns Created subcontractor
   */
  async create(data: SubcontractorCreate): Promise<Subcontractor> {
    const response = await this.fetch('/api/v1/subcontractors', {
      method: 'POST',
      body: JSON.stringify(data),
    });
    return response.json();
  }
  
  /**
   * Update an existing subcontractor
   * @param id - Subcontractor UUID
   * @param data - Fields to update
   * @returns Updated subcontractor
   */
  async update(id: string, data: SubcontractorUpdate): Promise<Subcontractor> {
    const response = await this.fetch(`/api/v1/subcontractors/${id}`, {
      method: 'PATCH',
      body: JSON.stringify(data),
    });
    return response.json();
  }
  
  /**
   * Approve a subcontractor (state transition)
   * @param id - Subcontractor UUID
   * @returns Updated subcontractor with 'approved' status
   * @throws Error if transition guards fail
   */
  async approve(id: string): Promise<Subcontractor> {
    const response = await this.fetch(`/api/v1/subcontractors/${id}/approve`, {
      method: 'POST',
    });
    return response.json();
  }
  
  /**
   * Suspend a subcontractor (state transition)
   * @param id - Subcontractor UUID
   * @returns Updated subcontractor with 'suspended' status
   */
  async suspend(id: string): Promise<Subcontractor> {
    const response = await this.fetch(`/api/v1/subcontractors/${id}/suspend`, {
      method: 'POST',
    });
    return response.json();
  }
  
  // ... private helper methods
}

// pkg/sdk/typescript/src/types.ts
// Generated by GateHouse - DO NOT EDIT

export type SubcontractorStatus = 'pending' | 'approved' | 'suspended' | 'terminated';

export interface Subcontractor {
  id: string;
  company_name: string;
  tax_id: string;
  status: SubcontractorStatus;
  insurance_expiry_date?: string;
  rating?: number;
  created_at: string;
  updated_at: string;
  
  // Relationships (when included)
  documents?: Document[];
  trades?: Trade[];
  work_orders?: WorkOrder[];
}

export interface SubcontractorCreate {
  company_name: string;
  tax_id: string;
  trade_codes: string[];
  insurance_expiry_date?: string;
}

export interface SubcontractorUpdate {
  company_name?: string;
  insurance_expiry_date?: string;
  // Note: tax_id is immutable and cannot be updated
}
```

### Docker Generation

```yaml
# docker/docker-compose.yml
# Generated by GateHouse - DO NOT EDIT

version: '3.8'

services:
  subcontractor-service:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    ports:
      - "8081:8080"
    environment:
      - DATABASE_URL=postgres://buildflow:dev_password@postgres:5432/subcontractor_dev?sslmode=disable
      - VAULT_ADDR=http://vault:8200
      - VAULT_TOKEN=dev-root-token
      - REDIS_URL=redis://redis:6379
      - AUTH_ISSUER=http://keycloak:8180/realms/buildflow
      - LOG_LEVEL=debug
    depends_on:
      postgres:
        condition: service_healthy
      vault:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: buildflow
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: subcontractor_dev
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U buildflow"]
      interval: 5s
      timeout: 3s
      retries: 5

  vault:
    image: hashicorp/vault:1.15
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: dev-root-token
      VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
    healthcheck:
      test: ["CMD", "vault", "status"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

---

## 8. Frontend Development Acceleration

### From SDK to UI

The generated TypeScript SDK enables rapid frontend development with full type safety.

```mermaid
graph TB
    subgraph "Generated"
        SDK[TypeScript SDK]
        TYPES[Type Definitions]
        OAS[OpenAPI Spec]
    end
    
    subgraph "React Scaffolding"
        HOOKS[Custom Hooks]
        COMP[UI Components]
        PAGES[Page Templates]
        FORMS[Form Generators]
    end
    
    subgraph "Developer Experience"
        AUTO[Autocomplete]
        VALID[Type Validation]
        DOC[Inline Docs]
    end
    
    SDK --> HOOKS
    TYPES --> COMP
    TYPES --> FORMS
    OAS --> DOC
    
    HOOKS --> AUTO
    COMP --> VALID
    FORMS --> VALID
    
    style SDK fill:#bbdefb
    style HOOKS fill:#c8e6c9
    style AUTO fill:#fff9c4
```

### React Hook Generation

```typescript
// hooks/useSubcontractors.ts
// Can be generated or hand-written using the SDK

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { SubcontractorClient, Subcontractor, SubcontractorCreate } from '@myapp/sdk';

const client = new SubcontractorClient({
  baseUrl: import.meta.env.VITE_API_URL,
  getToken: () => getAuthToken(),
});

export function useSubcontractors(params?: QueryParams<Subcontractor>) {
  return useQuery({
    queryKey: ['subcontractors', params],
    queryFn: () => client.list(params),
  });
}

export function useSubcontractor(id: string) {
  return useQuery({
    queryKey: ['subcontractors', id],
    queryFn: () => client.get(id, ['documents', 'trades']),
    enabled: !!id,
  });
}

export function useCreateSubcontractor() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (data: SubcontractorCreate) => client.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['subcontractors'] });
    },
  });
}

export function useApproveSubcontractor() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (id: string) => client.approve(id),
    onSuccess: (_, id) => {
      queryClient.invalidateQueries({ queryKey: ['subcontractors', id] });
      queryClient.invalidateQueries({ queryKey: ['subcontractors'] });
    },
  });
}
```

### Component Scaffolding

The generator can produce starter components that leverage the SDK:

```typescript
// components/SubcontractorList.tsx
// Generated scaffold - customize as needed

import { useSubcontractors } from '../hooks/useSubcontractors';
import { SubcontractorStatus } from '@myapp/sdk';

export function SubcontractorList() {
  const [filters, setFilters] = useState<{
    status?: SubcontractorStatus;
    search?: string;
  }>({});
  
  const { data, isLoading, error } = useSubcontractors({
    filter: filters,
    sort: { field: 'created_at', order: 'desc' },
  });
  
  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;
  
  return (
    <div className="space-y-4">
      <SubcontractorFilters filters={filters} onChange={setFilters} />
      
      <DataTable
        data={data.items}
        columns={[
          { key: 'company_name', header: 'Company', sortable: true },
          { key: 'status', header: 'Status', render: StatusBadge },
          { key: 'rating', header: 'Rating', render: RatingStars },
          { key: 'actions', header: '', render: ActionMenu },
        ]}
      />
      
      <Pagination
        total={data.total}
        pageSize={data.page_size}
        cursor={data.next_cursor}
        onPageChange={handlePageChange}
      />
    </div>
  );
}
```

### LLM Instructions for Feature Development

Include instructions for AI assistants building features:

```markdown
# Subcontractor Service - Frontend Development Guide

## SDK Usage

The TypeScript SDK at `@myapp/sdk` provides type-safe access to all API endpoints.

### Import Pattern
\`\`\`typescript
import { SubcontractorClient, Subcontractor, SubcontractorStatus } from '@myapp/sdk';
\`\`\`

### Available Operations

| Operation | Method | Description |
|-----------|--------|-------------|
| list | `client.list(params)` | Paginated list with filtering |
| get | `client.get(id, includes)` | Single entity with relations |
| create | `client.create(data)` | Create new subcontractor |
| update | `client.update(id, data)` | Partial update |
| approve | `client.approve(id)` | State transition to approved |
| suspend | `client.suspend(id)` | State transition to suspended |

### State Transitions

The Subcontractor entity has a state machine. Valid transitions:

- `pending` → `approved` (requires: insurance verified, documents uploaded)
- `approved` → `suspended` (any admin can suspend)
- `suspended` → `approved` (reinstatement)
- `*` → `terminated` (admin only, no active work orders)

### Form Validation

Use the SDK types for form validation:

\`\`\`typescript
import { SubcontractorCreate } from '@myapp/sdk';
import { z } from 'zod';

const createSchema = z.object({
  company_name: z.string().min(2).max(200),
  tax_id: z.string().regex(/^\d{2}-\d{7}$/),
  trade_codes: z.array(z.string()).min(1),
});
\`\`\`

### Error Handling

API errors follow this structure:
\`\`\`typescript
interface APIError {
  code: string;
  message: string;
  details?: Record<string, string[]>;
}
\`\`\`

Common error codes:
- `VALIDATION_ERROR` - Field validation failed
- `TRANSITION_BLOCKED` - State machine guard failed
- `NOT_FOUND` - Entity doesn't exist
- `FORBIDDEN` - Insufficient permissions
```

---

## 9. GateHouse Orchestration Platform

### Platform Overview

GateHouse serves as the central nervous system for the entire development workflow.

```mermaid
graph TB
    subgraph "GateHouse Platform"
        subgraph "Project Management"
            PROJ[Projects]
            ASSETS[Asset Registry]
            VER[Version Control]
        end
        
        subgraph "Intelligence Layer"
            SKILLS[Skill System]
            TOOLS[Tool Registry]
            AGENTS[Agent Framework]
        end
        
        subgraph "Automation Engine"
            TEMP[Temporal Workflows]
            SCHED[Scheduler]
            EVENTS[Event Bus]
        end
        
        subgraph "Generation Pipeline"
            VALID[Validator]
            GEN[Generator]
            DEPLOY[Deployer]
        end
    end
    
    PROJ --> AGENTS
    SKILLS --> AGENTS
    TOOLS --> AGENTS
    AGENTS --> TEMP
    TEMP --> VALID
    VALID --> GEN
    GEN --> DEPLOY
    SCHED --> TEMP
    EVENTS --> TEMP
    
    style AGENTS fill:#e1bee7
    style TEMP fill:#fff3e0
    style GEN fill:#c8e6c9
```

### Skill System

Skills are reusable capabilities that agents can invoke:

```yaml
# skills/yaml-generator.yaml

name: yaml-generator
version: "1.0.0"
description: "Generates App YAML specifications from BRD and Data Model"

inputs:
  - name: brd
    type: document
    required: true
    description: "Business Requirements Document"
  - name: data_model
    type: document
    required: true
    description: "Data Model Reference"
  - name: template_schema
    type: document
    required: true
    description: "YAML Template Schema"

outputs:
  - name: app_yaml
    type: yaml
    description: "Generated App YAML specification"

prompts:
  system: |
    You are an expert software architect specializing in schema-driven development.
    Your task is to synthesize business requirements and data models into a 
    comprehensive App YAML specification.
    
    Follow these principles:
    1. Every field must have explicit types and constraints
    2. State machines must cover all valid transitions
    3. Business rules become validation conditions or guards
    4. API endpoints follow RESTful conventions
    
  generation: |
    Given the following inputs:
    
    ## Business Requirements Document
    {{brd}}
    
    ## Data Model Reference  
    {{data_model}}
    
    ## Template Schema
    {{template_schema}}
    
    Generate a complete App YAML specification that:
    1. Defines all entities from the BRD
    2. Maps fields to data model columns
    3. Implements state machines for lifecycle management
    4. Configures API endpoints for CRUD and custom operations
    5. Defines events for integration points
```

### Tool Registry

Tools are atomic operations agents can perform:

```yaml
# tools/validate-yaml.yaml

name: validate-yaml
version: "1.0.0"
description: "Validates App YAML against schema and data model"

execution:
  type: container
  image: gatehouse/yaml-validator:latest
  command: ["validate", "--input", "{{input_path}}", "--schema", "{{schema_path}}"]

inputs:
  - name: yaml_content
    type: string
    required: true
  - name: schema_path
    type: path
    default: "/schemas/app-yaml-v1.json"
  - name: data_model_path
    type: path
    required: false

outputs:
  - name: valid
    type: boolean
  - name: errors
    type: array
    items:
      type: object
      properties:
        path: string
        message: string
        severity: enum[critical, warning, info]
        suggestion: string
```

### Agent Framework

Provider-agnostic agents that can use any LLM:

```yaml
# agents/yaml-author.yaml

name: yaml-author
version: "1.0.0"
description: "AI agent that creates and refines App YAML specifications"

provider:
  type: configurable  # anthropic | openai | azure | local
  model: claude-sonnet-4-20250514
  fallback:
    - provider: openai
      model: gpt-4-turbo

skills:
  - yaml-generator
  - yaml-refiner
  - error-interpreter

tools:
  - validate-yaml
  - query-data-model
  - search-documentation

behavior:
  max_iterations: 10
  on_validation_failure: refine_and_retry
  on_max_iterations: escalate_to_human
  
memory:
  type: conversation
  max_tokens: 100000
  
guardrails:
  - no_code_execution
  - yaml_only_output
  - schema_compliance
```

### Temporal Workflows

Orchestration of the complete pipeline:

```go
// workflows/generation_workflow.go

package workflows

import (
    "go.temporal.io/sdk/workflow"
    "time"
)

type GenerationInput struct {
    ProjectID    string
    BRDPath      string
    DataModelPath string
}

type GenerationOutput struct {
    Success       bool
    ArtifactPaths map[string]string
    Errors        []string
}

func GenerationWorkflow(ctx workflow.Context, input GenerationInput) (*GenerationOutput, error) {
    logger := workflow.GetLogger(ctx)
    
    // Step 1: Generate YAML with agent
    var yamlResult YAMLGenerationResult
    err := workflow.ExecuteActivity(ctx, GenerateYAMLActivity, input).Get(ctx, &yamlResult)
    if err != nil {
        return nil, err
    }
    
    // Step 2: Validation loop (max 10 iterations)
    var validYAML string
    for attempt := 1; attempt <= 10; attempt++ {
        var validationResult ValidationResult
        err := workflow.ExecuteActivity(ctx, ValidateYAMLActivity, yamlResult.YAML).Get(ctx, &validationResult)
        if err != nil {
            return nil, err
        }
        
        if validationResult.Valid {
            validYAML = yamlResult.YAML
            break
        }
        
        // Refine with error feedback
        var refineResult YAMLGenerationResult
        refineInput := RefineInput{
            CurrentYAML: yamlResult.YAML,
            Errors:      validationResult.Errors,
        }
        err = workflow.ExecuteActivity(ctx, RefineYAMLActivity, refineInput).Get(ctx, &refineResult)
        if err != nil {
            return nil, err
        }
        yamlResult = refineResult
    }
    
    if validYAML == "" {
        // Escalate to human
        var humanResult HumanInterventionResult
        err := workflow.ExecuteActivity(ctx, RequestHumanInterventionActivity, yamlResult).Get(ctx, &humanResult)
        if err != nil {
            return nil, err
        }
        validYAML = humanResult.FixedYAML
    }
    
    // Step 3: Generate all artifacts
    var genResult GenerationResult
    err = workflow.ExecuteActivity(ctx, GenerateArtifactsActivity, validYAML).Get(ctx, &genResult)
    if err != nil {
        return nil, err
    }
    
    // Step 4: Build and test
    var buildResult BuildResult
    err = workflow.ExecuteActivity(ctx, BuildAndTestActivity, genResult.ArtifactPaths).Get(ctx, &buildResult)
    if err != nil {
        return nil, err
    }
    
    return &GenerationOutput{
        Success:       buildResult.Success,
        ArtifactPaths: genResult.ArtifactPaths,
    }, nil
}
```

---

## 10. Putting It All Together

### Complete Workflow Visualization

```mermaid
graph TB
    subgraph "Phase 1: Requirements"
        START((Start)) --> BRD_IN{BRD Source?}
        BRD_IN -->|New Project| STAKE[Stakeholder Input]
        BRD_IN -->|Legacy Migration| LEGACY[Legacy Codebase]
        
        STAKE --> BRD_GEN[Generate BRD]
        LEGACY --> INDEX[Semantic Indexing]
        INDEX --> BRD_GEN
        
        BRD_GEN --> BRD[Business Requirements<br/>Document]
    end
    
    subgraph "Phase 2: Data Architecture"
        BRD --> DM_GEN[Generate Data Model]
        DM_GEN --> DMR[Data Model<br/>Reference]
    end
    
    subgraph "Phase 3: Specification"
        BRD --> YAML_AGENT[YAML Generator<br/>Agent]
        DMR --> YAML_AGENT
        YAML_AGENT --> YAML_DRAFT[App YAML<br/>Draft]
    end
    
    subgraph "Phase 4: Validation Loop"
        YAML_DRAFT --> VALIDATE{Validate}
        VALIDATE -->|Pass| VALID_YAML[Valid YAML]
        VALIDATE -->|Fail| ERROR[Error Report]
        ERROR --> COUNTER{Attempt < 10?}
        COUNTER -->|Yes| REFINE[Refine Agent]
        REFINE --> YAML_DRAFT
        COUNTER -->|No| HUMAN[Human<br/>Intervention]
        HUMAN --> VALID_YAML
    end
    
    subgraph "Phase 5: Generation"
        VALID_YAML --> GEN_ENGINE[Generator Engine]
        GEN_ENGINE --> GO[Go Backend]
        GEN_ENGINE --> SDK[TypeScript SDK]
        GEN_ENGINE --> DOCKER[Docker Config]
        GEN_ENGINE --> OAS[OpenAPI Spec]
    end
    
    subgraph "Phase 6: Build & Deploy"
        GO --> BUILD[Build & Test]
        SDK --> BUILD
        DOCKER --> BUILD
        BUILD --> DEPLOY[Deploy to<br/>Environment]
        DEPLOY --> DONE((Done))
    end
    
    style BRD fill:#e3f2fd
    style DMR fill:#fff3e0
    style VALID_YAML fill:#fff9c4
    style GO fill:#c8e6c9
    style SDK fill:#bbdefb
```

### Timeline Estimation

| Phase | Traditional | AI-Augmented | Speedup |
|-------|-------------|--------------|---------|
| Requirements Gathering | 2-4 weeks | 1-2 days | 10x |
| Data Modeling | 1-2 weeks | 2-4 hours | 20x |
| YAML Specification | N/A | 4-8 hours | N/A |
| Backend Development | 4-8 weeks | 1-2 hours (generation) | 100x |
| SDK Generation | 1-2 weeks | Instant | ∞ |
| Frontend Scaffolding | 1-2 weeks | 2-4 hours | 20x |
| **Total** | **10-18 weeks** | **1-2 weeks** | **10x** |

### Success Criteria

```yaml
quality_gates:
  yaml_specification:
    - schema_valid: true
    - all_entities_mapped: true
    - state_machines_complete: true
    - no_orphan_references: true
    
  generated_backend:
    - compiles_successfully: true
    - all_tests_pass: true
    - api_matches_openapi: true
    - no_security_vulnerabilities: true
    
  generated_sdk:
    - types_match_backend: true
    - all_endpoints_covered: true
    - documentation_complete: true
    
  integration:
    - end_to_end_tests_pass: true
    - performance_benchmarks_met: true
    - security_scan_clean: true
```

---

## Appendix A: Quick Start Checklist

### Project Setup

- [ ] Define problem statement and success metrics
- [ ] Create or import Business Requirements Document
- [ ] Design Data Model Reference
- [ ] Configure GateHouse project settings

### YAML Specification

- [ ] Review template schema documentation
- [ ] Run YAML generator agent
- [ ] Review validation results
- [ ] Iterate until valid

### Generation

- [ ] Run code generator
- [ ] Review generated Go structure
- [ ] Verify TypeScript SDK types
- [ ] Test Docker Compose locally

### Frontend Development

- [ ] Install generated SDK package
- [ ] Create React Query hooks
- [ ] Build UI components
- [ ] Implement forms with validation

### Deployment

- [ ] Run integration tests
- [ ] Deploy to staging
- [ ] Perform acceptance testing
- [ ] Promote to production

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **App YAML** | The central specification file that drives all code generation |
| **BRD** | Business Requirements Document - captures stakeholder needs |
| **DMR** | Data Model Reference - database schema specification |
| **Guard** | A condition that must be true for a state transition |
| **GateHouse** | The orchestration platform for AI-augmented development |
| **Semantic Indexing** | Chunking and embedding code for AI retrieval |
| **State Machine** | Formal model of entity lifecycle transitions |
| **Temporal** | Workflow engine for durable, distributed execution |
| **Vibe Coding** | AI-assisted development with human intent guidance |

---

*Document Version 1.0 | GateHouse Methodology*
