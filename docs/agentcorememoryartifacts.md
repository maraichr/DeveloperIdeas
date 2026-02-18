# Context Management Framework
## Orchestrator-Child Agent Pattern — Ledger + STM + AgentCore LTM Design

---

## Overview

This document defines the combined context management system for a product manager orchestrator agent operating in an A2A (agent-to-agent) pattern on AWS Strands with AgentCore. The system has three complementary layers:

- **Session Ledger** — durable, append-only Postgres table. Write-heavy, read-rare. Agent-only infrastructure, never exposed to users.
- **STM Working Context** — compressed, in-context summary the orchestrator carries turn-to-turn. Zero retrieval cost. Rebuilt from the ledger on cold start.
- **AgentCore LTM** — cross-session persistent memory. Curated records promoted explicitly from the ledger via `batch_create_memory_records`. Retrieved on session start via `retrieve_memory_records`.

The principle: **write everything to the ledger, carry only conclusions in STM, promote durable facts to AgentCore LTM explicitly.**

---

## 1. Postgres Ledger Schema

### sessions

```sql
CREATE TABLE sessions (
    session_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id           TEXT NOT NULL,
    title             TEXT,
    status            TEXT NOT NULL DEFAULT 'active'
                      CHECK (status IN ('active', 'paused', 'completed', 'abandoned')),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    stm_snapshot      JSONB  -- last known STM state for session resume
);

CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_status ON sessions(status);
```

### session_ledger

```sql
CREATE TABLE session_ledger (
    entry_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id        UUID NOT NULL REFERENCES sessions(session_id),
    sequence_num      SERIAL,  -- monotonic ordering within session
    timestamp         TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Classification
    entry_type        TEXT NOT NULL
                      CHECK (entry_type IN (
                          'decision',           -- PM chose an option
                          'constraint',          -- scope/boundary established
                          'artifact_created',    -- confluence doc, markdown, etc.
                          'artifact_updated',    -- modification to existing artifact
                          'research_finding',    -- child research agent result
                          'child_agent_result',  -- any child agent completion
                          'plan_created',        -- initial plan decomposition
                          'plan_updated',        -- plan step completed/changed
                          'user_clarification',  -- PM corrected or elaborated
                          'context_injection',   -- LTM fact pulled into session
                          'checkpoint'           -- explicit state snapshot
                      )),

    -- Scoping for filtered retrieval
    scope             TEXT NOT NULL,  -- e.g. 'technical_requirements', 'user_stories',
                                     -- 'architecture', 'timeline', 'stakeholders', 'general'

    -- Content
    summary           TEXT NOT NULL,  -- compressed, agent-readable conclusion
                                     -- e.g. "Mobile-first approach, no email in v1"
    reasoning         TEXT,           -- optional: why this decision was made
                                     -- only populated for decisions/constraints
    raw_options       JSONB,          -- for decisions: what options were presented
                                     -- { "presented": ["A", "B", "C"], "selected": "B" }

    -- References
    artifact_ids      TEXT[],         -- linked artifact IDs (confluence URLs, markdown doc IDs)
    jira_keys         TEXT[],         -- linked Jira issue keys
    related_entries   UUID[],         -- links to other ledger entries
    child_agent       TEXT,           -- which child agent produced this, if applicable
                                     -- e.g. 'research', 'confluence', 'jira'

    -- LTM promotion tracking
    promoted_to_ltm   BOOLEAN DEFAULT FALSE,  -- has this been pushed to AgentCore LTM?
    ltm_record_id     TEXT,                    -- AgentCore memory record ID if promoted

    -- Source
    source            TEXT NOT NULL DEFAULT 'user'
                      CHECK (source IN ('user', 'child_agent', 'orchestrator'))
);

-- Primary access pattern: rebuild session context
CREATE INDEX idx_ledger_session_seq ON session_ledger(session_id, sequence_num);

-- Scoped retrieval: "all technical decisions for this session"
CREATE INDEX idx_ledger_session_scope ON session_ledger(session_id, scope);

-- Type-filtered retrieval: "all decisions for this session"
CREATE INDEX idx_ledger_session_type ON session_ledger(session_id, entry_type);

-- LTM promotion: find entries not yet promoted
CREATE INDEX idx_ledger_promotion ON session_ledger(session_id, promoted_to_ltm)
    WHERE promoted_to_ltm = FALSE;
```

### artifacts

```sql
CREATE TABLE artifacts (
    artifact_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id        UUID NOT NULL REFERENCES sessions(session_id),
    version           INTEGER NOT NULL DEFAULT 1,

    -- Classification
    artifact_type     TEXT NOT NULL
                      CHECK (artifact_type IN (
                          'prd',                -- product requirements document
                          'technical_spec',     -- technical specification
                          'user_stories',       -- set of user stories
                          'research_summary',   -- research findings
                          'epic',               -- Jira epic definition
                          'general'             -- catch-all
                      )),

    -- Where it will be published
    publish_target    TEXT NOT NULL
                      CHECK (publish_target IN ('confluence', 'jira', 'none')),

    -- Content (always markdown in Postgres — source of truth until published)
    title             TEXT NOT NULL,
    content           TEXT NOT NULL,        -- markdown content

    -- Lifecycle state
    status            TEXT NOT NULL DEFAULT 'draft'
                      CHECK (status IN (
                          'draft',              -- agent-generated, not yet reviewed
                          'review',             -- PM is reviewing
                          'revision_requested', -- PM requested changes (feedback attached)
                          'revising',           -- agent is making changes
                          'approved',           -- PM approved, ready to publish
                          'publishing',         -- publish in progress (child agent working)
                          'published',          -- successfully pushed to target
                          'publish_failed'      -- publish attempt failed
                      )),

    -- External references (populated after publish)
    external_url      TEXT,              -- confluence URL or Jira link
    external_id       TEXT,              -- confluence page ID or Jira issue key

    -- Tracking
    created_by        TEXT NOT NULL,        -- 'orchestrator', 'confluence_agent', etc.
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at      TIMESTAMPTZ,
    published_by      TEXT                  -- which child agent published
);

CREATE INDEX idx_artifacts_session ON artifacts(session_id);
CREATE INDEX idx_artifacts_status ON artifacts(status);
CREATE INDEX idx_artifacts_session_status ON artifacts(session_id, status);
```

### artifact_versions

```sql
-- Immutable version history — every save creates a new version row
CREATE TABLE artifact_versions (
    version_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id       UUID NOT NULL REFERENCES artifacts(artifact_id),
    version           INTEGER NOT NULL,
    content           TEXT NOT NULL,          -- markdown snapshot at this version
    change_summary    TEXT,                   -- what changed (agent-generated)
    created_by        TEXT NOT NULL,          -- 'orchestrator', 'confluence_agent', 'pm_edit'
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(artifact_id, version)
);

CREATE INDEX idx_versions_artifact ON artifact_versions(artifact_id, version DESC);
```

### artifact_reviews

```sql
-- HIL feedback from PM — links a review action to an artifact
CREATE TABLE artifact_reviews (
    review_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id       UUID NOT NULL REFERENCES artifacts(artifact_id),
    session_id        UUID NOT NULL REFERENCES sessions(session_id),
    version_reviewed  INTEGER NOT NULL,        -- which version the PM reviewed

    -- Action taken
    action            TEXT NOT NULL
                      CHECK (action IN (
                          'approve',             -- PM approved → status becomes 'approved'
                          'request_revision',    -- PM wants changes → status becomes 'revision_requested'
                          'publish',             -- PM clicked publish → triggers publish flow
                          'reject'               -- PM rejected entirely
                      )),

    -- Feedback (required for request_revision, optional for others)
    feedback          TEXT,                     -- PM's comments / change requests
    inline_comments   JSONB,                   -- optional: line-level comments
                                               -- [{ "line": 42, "comment": "..." }, ...]

    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reviews_artifact ON artifact_reviews(artifact_id);
CREATE INDEX idx_reviews_session ON artifact_reviews(session_id);
```

---

## 2. STM Working Context Structure

This is a JSON block that the orchestrator maintains **in its context window** on every turn. It is the primary driver of turn-to-turn behavior. It gets written to `sessions.stm_snapshot` at checkpoints and on session pause for cold-start recovery.

```json
{
  "session_id": "uuid",
  "user_id": "user-123",
  "intent": "Create a PRD for the notifications feature targeting enterprise mobile users",

  "plan": {
    "status": "in_progress",
    "current_step": "draft_prd",
    "steps": [
      {
        "id": "research",
        "description": "Research notification patterns and enterprise requirements",
        "status": "completed",
        "outcome_summary": "Push notifications preferred 3:1 over email in enterprise mobile. APNs + FCM standard. Rate limiting critical for enterprise."
      },
      {
        "id": "draft_prd",
        "description": "Create PRD in Confluence",
        "status": "in_progress",
        "outcome_summary": null
      },
      {
        "id": "create_stories",
        "description": "Break down into Jira user stories",
        "status": "pending",
        "outcome_summary": null
      }
    ]
  },

  "active_decisions": [
    "Mobile-first approach, no email in v1",
    "Enterprise tier only for initial release",
    "Push notifications via APNs + FCM",
    "Rate limiting: max 50 notifications/user/day",
    "PRD target: Confluence space NOTIFY"
  ],

  "active_constraints": [
    "Timeline: must ship within Q2",
    "Team: 4 engineers, no dedicated mobile dev",
    "No breaking changes to existing notification preferences API"
  ],

  "artifact_refs": [
    {
      "type": "prd",
      "title": "Notifications PRD v1",
      "artifact_id": "uuid-123",
      "version": 2,
      "status": "review",
      "publish_target": "confluence",
      "target_config": { "space": "NOTIFY", "parent_page_id": "pg-789" },
      "external_url": null
    },
    {
      "type": "research_summary",
      "title": "Notification Research Summary",
      "artifact_id": "uuid-456",
      "version": 1,
      "status": "published",
      "publish_target": "none",
      "external_url": null
    }
  ],

  "open_questions": [
    "Should notification preferences be per-channel or global?",
    "Does enterprise SSO affect push token registration?"
  ],

  "turn_count": 14,
  "last_checkpoint_seq": 22
}
```

### STM Update Rules

The orchestrator follows these rules on every turn:

| Event | STM Action |
|---|---|
| PM selects guided option | Add to `active_decisions` |
| PM states a constraint | Add to `active_constraints` |
| Plan is created/updated | Replace `plan` block |
| Child agent returns | Update relevant `plan.steps[].outcome_summary` |
| Artifact created | Add to `artifact_refs` with status `draft` |
| Artifact status changes | Update `artifact_refs[].status` |
| Artifact published | Update `artifact_refs[].status` + set `external_url` |
| PM asks a new question | Add to `open_questions` |
| Open question resolved | Remove from `open_questions` |
| Step completed | Update `plan.steps[].status`, advance `current_step` |

**Critical rule**: `active_decisions` and `active_constraints` contain **conclusions only** — never the deliberation. Reasoning lives in the ledger.

---

## 3. Tool Definitions (Postgres Ledger)

These are Strands `@tool` functions the orchestrator calls to interact with the Postgres ledger. They are independent of AgentCore Memory — they operate against your own Postgres instance.

### write_ledger

Called on every structural event (decision, artifact creation, child agent return, etc.)

```python
@tool
def write_ledger(
    session_id: str,
    entry_type: str,       # 'decision' | 'constraint' | 'artifact_created' | etc.
    scope: str,            # 'technical_requirements' | 'architecture' | 'general' | etc.
    summary: str,          # compressed conclusion
    reasoning: str = None, # optional: why
    raw_options: dict = None,  # for decisions: {"presented": [...], "selected": "..."}
    artifact_ids: list[str] = None,
    jira_keys: list[str] = None,
    child_agent: str = None,
    source: str = "user"
) -> dict:
    """
    Persist a structured entry to the session ledger in Postgres.
    Called on every structural event — decisions, artifact operations,
    child agent results, plan updates, and user clarifications.

    Returns: { entry_id, sequence_num }
    """
```

**Trigger rules** (deterministic, not model-judged):
- PM selects a guided option → `write_ledger(entry_type='decision', ...)`
- Child agent completes → `write_ledger(entry_type='child_agent_result', ...)`
- Artifact created/updated → `write_ledger(entry_type='artifact_created|artifact_updated', ...)`
- PM provides clarification → `write_ledger(entry_type='user_clarification', ...)`
- Plan created/modified → `write_ledger(entry_type='plan_created|plan_updated', ...)`

### query_ledger

Called on cold start, context reconstruction, or explicit recall.

```python
@tool
def query_ledger(
    session_id: str,
    entry_types: list[str] = None,  # filter by type
    scopes: list[str] = None,       # filter by scope
    since_sequence: int = None,     # entries after this sequence number
    limit: int = 50
) -> list[dict]:
    """
    Retrieve ledger entries for a session with optional filters.
    Used for:
      - Session resume (all entries, rebuild STM)
      - Scoped retrieval (e.g. all technical decisions for child agent dispatch)
      - Explicit recall (PM asks "why did we decide X?")

    Returns: list of ledger entries ordered by sequence_num
    """
```

### save_stm_snapshot

Called at checkpoints and session pause.

```python
@tool
def save_stm_snapshot(
    session_id: str,
    stm_state: dict  # the full STM working context JSON
) -> dict:
    """
    Persist the current STM working context to sessions.stm_snapshot.
    Called at:
      - Explicit checkpoints (after major milestones)
      - Session pause/end
      - Every N turns as a safety net (e.g. every 5 turns)

    Returns: { session_id, updated_at }
    """
```

### load_stm_snapshot

Called on session resume.

```python
@tool
def load_stm_snapshot(
    session_id: str
) -> dict:
    """
    Load the last saved STM snapshot for a session.
    On session resume, this is loaded first, then any ledger entries
    written after last_checkpoint_seq are replayed to bring STM current.

    Returns: STM state JSON or null if no snapshot exists
    """
```

---

## 4. AgentCore Memory Configuration

AgentCore Memory handles two responsibilities in this architecture:

1. **STM event storage** — conversation events are written automatically via the Strands `AgentCoreMemorySessionManager`.
2. **LTM record storage** — curated records are promoted explicitly from the Postgres ledger via `batch_create_memory_records`.

### Memory Resource Setup

Create the AgentCore Memory resource with a **self-managed strategy** for project decisions (giving full control over what gets promoted) and a **built-in with overrides** user preference strategy (letting AgentCore automatically capture PM preferences from conversation).

```python
from bedrock_agentcore.memory import MemoryClient

client = MemoryClient(region_name="us-east-1")

memory = client.create_memory_and_wait(
    name="PMOrchestratorMemory",
    description="Memory for PM orchestrator agent with explicit LTM promotion",
    strategies=[
        # Auto-extract PM preferences (formatting, tools, methodology)
        # Uses built-in with overrides so we can steer extraction
        {
            "userPreferenceMemoryStrategy": {
                "name": "PMPreferenceLearner",
                "namespaces": ["/preferences/{actorId}"],
                "configuration": {
                    "type": "USER_PREFERENCE_OVERRIDE",
                    "extraction": {
                        "appendToPrompt": (
                            "Focus on extracting product management workflow preferences: "
                            "preferred document formats, user story formats (Given/When/Then vs As a/I want), "
                            "sprint methodology, estimation approach, meeting cadence preferences, "
                            "communication style, and tool preferences. "
                            "Ignore transient project-specific details — those are handled separately."
                        ),
                        "modelId": "anthropic.claude-3-5-haiku-20241022-v1:0"
                    },
                    "consolidation": {
                        "appendToPrompt": (
                            "When consolidating preferences, newer preferences take priority. "
                            "If a preference contradicts an existing one, replace it. "
                            "Maintain a clean, non-duplicated preference set."
                        ),
                        "modelId": "anthropic.claude-3-5-haiku-20241022-v1:0"
                    }
                }
            }
        },
        # Self-managed strategy for project decisions
        # We control extraction entirely — records are batch-ingested from the ledger
        {
            "selfManagedMemoryStrategy": {
                "name": "ProjectDecisions",
                "namespaces": ["/projects/{actorId}"]
            }
        }
    ]
)

MEMORY_ID = memory.get('id')
```

> **Why this split?** PM preferences (how they like user stories formatted, preferred estimation method) are low-stakes and emerge naturally in conversation — automatic extraction with a guided prompt works well here. Project decisions (no email in v1, chose Keycloak over Auth0) are high-stakes and must be deterministic — we push them explicitly from the ledger.

### Strands Session Manager Integration

The orchestrator agent uses `AgentCoreMemorySessionManager` for STM (conversation event storage) and for LTM retrieval at session start. This is the standard Strands integration.

```python
from strands import Agent
from bedrock_agentcore.memory.integrations.strands.config import (
    AgentCoreMemoryConfig,
    RetrievalConfig
)
from bedrock_agentcore.memory.integrations.strands.session_manager import (
    AgentCoreMemorySessionManager
)

config = AgentCoreMemoryConfig(
    memory_id=MEMORY_ID,
    session_id=session_id,       # from your React frontend
    actor_id=user_id,            # the PM's user ID
    retrieval_config={
        # Retrieve PM preferences on session start
        "/preferences/{actorId}": RetrievalConfig(
            top_k=10,
            relevance_score=0.6
        ),
        # Retrieve project decisions relevant to this session
        "/projects/{actorId}": RetrievalConfig(
            top_k=15,
            relevance_score=0.5
        )
    }
)

with AgentCoreMemorySessionManager(config, region_name="us-east-1") as session_manager:
    orchestrator = Agent(
        system_prompt="You are a PM orchestrator agent...",
        session_manager=session_manager,
        tools=[write_ledger, query_ledger, save_stm_snapshot, load_stm_snapshot,
               promote_to_ltm]  # custom tools defined in Section 3 + 5
    )
```

The `AgentCoreMemorySessionManager` handles:
- **On each turn**: Writing conversation events to STM (automatic via `create_event`).
- **On session start**: Retrieving relevant LTM records based on `retrieval_config` and injecting them into the agent's context.
- **Batching**: Configure `batch_size` if you want to buffer events for throughput (default is 1, send each immediately).

> **Important**: The STM managed by AgentCore is the raw conversation event stream. This is *separate from* the STM Working Context JSON (Section 2) which is the orchestrator's compressed in-context state. Both exist — AgentCore STM is the raw log, our STM Working Context is the agent's working memory.

---

## 5. LTM Promotion from Ledger (Option B — Explicit Push)

This is the core of the architecture. At session end or at checkpoints, the orchestrator explicitly promotes curated ledger entries to AgentCore LTM using the `batch_create_memory_records` API.

### promote_to_ltm Tool

```python
import boto3
from datetime import datetime, timezone

agentcore_client = boto3.client('bedrock-agentcore', region_name='us-east-1')

@tool
def promote_to_ltm(
    session_id: str,
    memory_id: str,
    actor_id: str
) -> dict:
    """
    Promote eligible ledger entries to AgentCore LTM.
    Called at session completion or major checkpoints.

    Queries the Postgres ledger for unpromoted entries of promotable types.
    Formats them as AgentCore memory records and batch-ingests them
    via batch_create_memory_records.

    Returns: { promoted_count, failed_count, record_ids }
    """

    # 1. Query Postgres ledger for unpromoted, LTM-eligible entries
    #    (implementation filters on promoted_to_ltm = FALSE)
    eligible_entries = _query_promotable_entries(session_id)

    if not eligible_entries:
        return {"promoted_count": 0, "failed_count": 0, "record_ids": []}

    # 2. Format as AgentCore memory records
    #    batch_create_memory_records accepts up to 100 records per call
    #    Each record requires: requestIdentifier, namespaces, content, timestamp
    #    Optional: memoryStrategyId
    records = []
    for entry in eligible_entries:
        # Build the text content — include reasoning for decisions
        content_parts = [entry['summary']]
        if entry.get('reasoning'):
            content_parts.append(f"Reasoning: {entry['reasoning']}")
        if entry.get('scope'):
            content_parts.append(f"Scope: {entry['scope']}")

        # requestIdentifier: alphanumeric + hyphens/underscores, max 80 chars
        request_id = str(entry['entry_id']).replace('-', '')[:80]

        records.append({
            'requestIdentifier': request_id,
            'namespaces': [f"/projects/{actor_id}"],
            'content': {
                'text': ' | '.join(content_parts)
                # text max: 16,000 characters per record
            },
            'timestamp': entry['timestamp'],
            'memoryStrategyId': 'ProjectDecisions'
        })

    # 3. Batch ingest via AgentCore Data Plane API
    all_results = {"promoted_count": 0, "failed_count": 0, "record_ids": []}

    for i in range(0, len(records), 100):  # max 100 records per batch call
        batch = records[i:i + 100]
        response = agentcore_client.batch_create_memory_records(
            memoryId=memory_id,
            records=batch
        )

        # Response contains successfulRecords and failedRecords lists
        # Each has: memoryRecordId, status ('SUCCEEDED'|'FAILED'),
        #           requestIdentifier, errorCode, errorMessage
        for success in response.get('successfulRecords', []):
            if success['status'] == 'SUCCEEDED':
                all_results['promoted_count'] += 1
                all_results['record_ids'].append(success['memoryRecordId'])

        for failure in response.get('failedRecords', []):
            all_results['failed_count'] += 1
            # Log: failure['errorMessage'], failure['requestIdentifier']

    # 4. Mark entries as promoted in Postgres ledger
    _mark_entries_promoted(
        session_id,
        [e['entry_id'] for e in eligible_entries],
        all_results['record_ids']
    )

    return all_results


def _query_promotable_entries(session_id: str) -> list[dict]:
    """
    Query Postgres for ledger entries eligible for LTM promotion.
    Filters: promoted_to_ltm = FALSE AND entry_type IN promotable types.
    """
    # SQL: SELECT * FROM session_ledger
    #      WHERE session_id = %s
    #        AND promoted_to_ltm = FALSE
    #        AND entry_type IN ('decision', 'constraint', 'research_finding')
    #      ORDER BY sequence_num
    ...


def _mark_entries_promoted(session_id: str, entry_ids: list, record_ids: list):
    """
    Update Postgres: set promoted_to_ltm = TRUE and ltm_record_id
    for successfully promoted entries.
    """
    # SQL: UPDATE session_ledger
    #      SET promoted_to_ltm = TRUE, ltm_record_id = %s
    #      WHERE entry_id = %s
    ...
```

### What Gets Promoted vs. What Doesn't

| Entry Type | Promote to LTM? | Rationale |
|---|---|---|
| `decision` | **Yes** | Project decisions affect future sessions |
| `constraint` | **Yes** | Constraints persist across sessions |
| `research_finding` | **Selective** | Only if it establishes a durable fact |
| `artifact_created` | **No** | Artifact references are retrievable from Postgres |
| `artifact_updated` | **No** | Same — Postgres is the source of truth |
| `plan_created` | **No** | Plans are session-scoped |
| `plan_updated` | **No** | Plans are session-scoped |
| `user_clarification` | **No** | Preferences auto-captured by UserPreference strategy |
| `child_agent_result` | **No** | Results are incorporated into decisions/artifacts |
| `checkpoint` | **No** | Internal state management only |

### Retrieval on Session Start

When a new session begins, the orchestrator retrieves relevant LTM records before the first turn. This happens automatically via the `RetrievalConfig` on the `AgentCoreMemorySessionManager`, but can also be called explicitly for mid-session context enrichment:

```python
# Automatic: handled by AgentCoreMemorySessionManager on agent creation
# The retrieval_config in Section 4 pulls from /preferences/{actorId}
# and /projects/{actorId} namespaces automatically.

# Explicit (if needed for mid-session context enrichment):
response = agentcore_client.retrieve_memory_records(
    memoryId=MEMORY_ID,
    namespace=f"/projects/{actor_id}",
    searchCriteria={
        'searchQuery': 'notifications feature enterprise mobile push',
        'memoryStrategyId': 'ProjectDecisions',
        'topK': 10
    }
)

# Response structure:
# {
#   'memoryRecordSummaries': [
#     {
#       'memoryRecordId': 'mem-xxxx',
#       'content': {'text': 'Decision text here'},
#       'memoryStrategyId': 'ProjectDecisions',
#       'namespaces': ['/projects/user-123'],
#       'createdAt': '2025-01-15T10:00:00Z',
#       'score': 0.87  # cosine similarity relevance score
#     },
#     ...
#   ],
#   'nextToken': '...'  # for pagination, default max 100 per page
# }

for record in response.get('memoryRecordSummaries', []):
    print(f"Score: {record['score']} | {record['content']['text']}")
```

> **Retrieval latency**: Semantic search via `retrieve_memory_records` returns results in approximately 200ms per the AgentCore documentation. This is acceptable for session-start context loading but is not something you want on every turn — which is why the STM Working Context exists.

### Namespace Strategy

```
/preferences/{actorId}                  ← PM workflow preferences (auto-extracted)
/projects/{actorId}                     ← Project decisions (explicitly promoted from ledger)
```

The `{actorId}` placeholder is automatically replaced by the `actor_id` from your `AgentCoreMemoryConfig`. This means each PM gets isolated memory — PM Alice's decisions don't bleed into PM Bob's context.

If you later need project-level sharing (multiple PMs on one project), you can add a project-scoped namespace like `/projects/shared/{projectId}` and promote relevant decisions there as well.

---

## 6. Lifecycle Flows

### Normal Turn (Hot Path — No Ledger Reads, No LTM Reads)

```
PM message arrives
    │
    ├─ Orchestrator reads STM Working Context (already in context window)
    ├─ Intent router determines action
    ├─ If structural event:
    │   ├─ Tool call: write_ledger(...)            ← Postgres INSERT, async-safe
    │   └─ Update STM Working Context in-context
    ├─ If child agent dispatch needed:
    │   ├─ Assemble child context from STM (not ledger, not LTM)
    │   └─ Dispatch via A2A
    ├─ Generate response / guided options
    └─ Return to PM
```

**Latency impact**: One Postgres INSERT (async-safe) + zero reads. Conversation events also flow to AgentCore STM via the session manager. Negligible user-facing latency.

### Session Start (New Session — LTM Retrieval)

```
PM starts new session from React UI
    │
    ├─ React frontend creates session in Postgres, gets session_id
    ├─ AgentCoreMemorySessionManager initializes with retrieval_config
    │   ├─ Retrieves from /preferences/{actorId}    ← PM preferences (~200ms)
    │   └─ Retrieves from /projects/{actorId}        ← Past project decisions (~200ms)
    ├─ Retrieved LTM records injected into agent context
    ├─ Orchestrator initializes empty STM Working Context
    └─ Ready for first PM message
```

### Session Resume (Cold Start — Ledger Replay + LTM)

```
PM reopens existing session
    │
    ├─ AgentCoreMemorySessionManager retrieves LTM (same as new session)
    ├─ Tool call: load_stm_snapshot(session_id)     ← Postgres read
    ├─ Tool call: query_ledger(session_id, since_sequence=last_checkpoint_seq)
    ├─ Replay new ledger entries onto STM snapshot
    ├─ STM Working Context is now current
    ├─ Tool call: save_stm_snapshot(session_id, updated_stm)
    └─ Greet PM with "Here's where we left off..." + current state summary
```

### Session End (LTM Promotion)

```
PM completes session or hits major milestone
    │
    ├─ Tool call: promote_to_ltm(session_id, memory_id, actor_id)
    │   ├─ Queries Postgres ledger for unpromoted decisions + constraints
    │   ├─ Formats as AgentCore memory records
    │   ├─ Calls batch_create_memory_records         ← AgentCore Data Plane API
    │   │   ├─ Returns successfulRecords[]            (memoryRecordId, status)
    │   │   └─ Returns failedRecords[]                (errorCode, errorMessage)
    │   └─ Marks entries as promoted in Postgres      (promoted_to_ltm = TRUE)
    ├─ Tool call: save_stm_snapshot(session_id, final_stm)
    ├─ Update session status to 'completed'
    └─ Confirm to PM: "Session saved. Decisions available in future sessions."
```

> **Note on async extraction**: The built-in UserPreference strategy with overrides processes conversation events asynchronously. Extraction and consolidation typically complete within 20-40 seconds after events are written. This means PM preferences may not be available immediately in LTM after a session — but they will be there by the time the PM starts their next session.

### Child Agent Dispatch (Scoped Ledger Read)

```
Orchestrator needs to send work to Confluence agent
    │
    ├─ STM Working Context has artifact_refs and active_decisions (usually sufficient)
    ├─ If child needs deeper context:
    │   └─ Tool call: query_ledger(session_id, scopes=['technical_requirements', 'architecture'])
    ├─ Assemble focused payload:
    │   {
    │     task: "Draft technical requirements section",
    │     decisions: [...filtered from STM/ledger...],
    │     constraints: [...],
    │     artifact_ref: { confluence_url, page_id }
    │   }
    └─ Dispatch via A2A
```

### Explicit Recall ("Why did we decide X?")

```
PM asks: "Why did we exclude email notifications?"
    │
    ├─ Orchestrator checks STM Working Context — finds decision but no reasoning
    ├─ Tool call: query_ledger(session_id, entry_types=['decision'], scopes=['general'])
    ├─ Finds entry with reasoning: "PM noted enterprise clients primarily use
    │   mobile app, email adds 3 weeks, revisit in v2"
    └─ Responds with reasoning to PM
```

---

## 7. Artifact Lifecycle & Human-in-the-Loop Publish

Agents produce drafts. Humans publish. This is non-negotiable — no agent should push directly to Confluence or create Jira stories without PM approval. The artifact lifecycle enforces this boundary.

### Lifecycle State Machine

```
                    ┌─────────────────────────────────┐
                    │                                 │
                    ▼                                 │
  ┌─────────┐   ┌────────┐   ┌─────────────────────┐ │  ┌──────────┐
  │  draft   │──▶│ review │──▶│ revision_requested  │─┘  │ rejected │
  └─────────┘   └────────┘   └─────────────────────┘     └──────────┘
                    │                    │                      ▲
                    │                    ▼                      │
                    │              ┌──────────┐                │
                    │              │ revising │                 │
                    │              └──────────┘                │
                    │                    │                      │
                    │                    │ (new version)        │
                    │                    ▼                      │
                    │              ┌────────┐                  │
                    │              │ review │──────────────────┘
                    │              └────────┘
                    │
                    ▼
              ┌──────────┐   ┌─────────────┐   ┌───────────┐
              │ approved │──▶│ publishing  │──▶│ published │
              └──────────┘   └─────────────┘   └───────────┘
                                    │
                                    ▼
                             ┌──────────────┐
                             │publish_failed│──▶ (retry or manual)
                             └──────────────┘
```

### State Transitions & Who Triggers Them

| From | To | Triggered By | How |
|---|---|---|---|
| `draft` | `review` | Orchestrator | Agent finishes generating content |
| `review` | `approved` | PM | Clicks **Approve** button in React UI |
| `review` | `revision_requested` | PM | Clicks **Request Changes** + provides feedback |
| `review` | `rejected` | PM | Clicks **Reject** (terminal state) |
| `revision_requested` | `revising` | Orchestrator | Dispatches revision to child agent |
| `revising` | `review` | Orchestrator | Child agent returns revised content (new version created) |
| `approved` | `publishing` | PM | Clicks **Publish** button in React UI |
| `publishing` | `published` | Child Agent | Confluence/Jira agent confirms successful creation |
| `publishing` | `publish_failed` | Child Agent | External API call failed |
| `publish_failed` | `publishing` | PM | Clicks **Retry Publish** |

### React Frontend — Artifact View

The PM sees artifacts rendered as markdown in the React UI, retrieved from Postgres by session ID. The UI renders contextual action buttons based on the artifact's current `status`:

```
┌─────────────────────────────────────────────────────────┐
│  Notifications PRD v1                        v2 (draft) │
│  ─────────────────────────────────────────────────────── │
│                                                         │
│  ## Overview                                            │
│  This PRD defines the push notification system for      │
│  enterprise mobile users...                             │
│                                                         │
│  ## Technical Requirements                              │
│  - APNs + FCM dual-channel delivery                     │
│  - Rate limiting: 50 notifications/user/day             │
│  ...                                                    │
│                                                         │
│  ─────────────────────────────────────────────────────── │
│                                                         │
│  ┌──────────┐  ┌───────────────────┐  ┌──────────────┐  │
│  │ Approve  │  │ Request Changes ✎ │  │   Reject ✗   │  │
│  └──────────┘  └───────────────────┘  └──────────────┘  │
│                                                         │
│  status: review │ target: confluence │ version: 2       │
└─────────────────────────────────────────────────────────┘
```

After approval:

```
┌─────────────────────────────────────────────────────────┐
│  Notifications PRD v1                    v2 (approved)  │
│  ─────────────────────────────────────────────────────── │
│  ...content...                                          │
│  ─────────────────────────────────────────────────────── │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │  📄 Publish to Confluence                           ││
│  │  Space: NOTIFY  │  Parent: Product Requirements     ││
│  │                                                     ││
│  │  ┌──────────────────┐                               ││
│  │  │  Publish Now  ▶  │                               ││
│  │  └──────────────────┘                               ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

The publish target metadata (Confluence space, parent page, Jira project/board) can either be:
- Set during the guided conversation (orchestrator asks PM where to publish and stores it)
- Defaulted from LTM (PM always publishes PRDs to the NOTIFY space)
- Configured at publish time via the UI

### The Publish Flow (HIL → Orchestrator → Child Agent)

When the PM clicks **Publish**, the React frontend does not call Confluence/Jira directly. It sends a publish request back to the orchestrator, which dispatches to the appropriate child agent. This keeps all external system interaction going through the agent layer.

```
PM clicks "Publish to Confluence" in React UI
    │
    ├─ Frontend POST /api/artifacts/{artifact_id}/publish
    │   Body: { action: "publish", target_config: { space: "NOTIFY", parent_page_id: "..." } }
    │
    ├─ Backend updates artifact status → 'publishing'
    ├─ Backend writes artifact_reviews row (action: 'publish')
    ├─ Backend sends message to orchestrator session
    │
    ├─ Orchestrator receives publish instruction
    │   ├─ Reads artifact content from Postgres (or from STM artifact_ref)
    │   ├─ Writes ledger entry: entry_type='artifact_updated', summary='Publishing PRD to Confluence'
    │   ├─ Assembles child agent payload:
    │   │   {
    │   │     task: "publish_to_confluence",
    │   │     artifact_id: "uuid",
    │   │     content: "...markdown...",
    │   │     title: "Notifications PRD v1",
    │   │     target: { space: "NOTIFY", parent_page_id: "..." }
    │   │   }
    │   └─ Dispatches to Confluence child agent via A2A
    │
    ├─ Confluence agent:
    │   ├─ Converts markdown → Confluence storage format (XHTML)
    │   ├─ Calls Confluence REST API to create/update page
    │   ├─ Returns: { success: true, page_id: "12345", url: "https://..." }
    │   └─ (or returns failure with error details)
    │
    ├─ Orchestrator receives child result
    │   ├─ On success:
    │   │   ├─ Updates artifact in Postgres:
    │   │   │   status → 'published'
    │   │   │   external_url → confluence URL
    │   │   │   external_id → page ID
    │   │   │   published_at → now()
    │   │   ├─ Writes ledger entry: 'artifact_updated', 'Published to Confluence: <url>'
    │   │   ├─ Updates STM artifact_ref with external URL
    │   │   └─ Notifies PM in chat: "PRD published to Confluence: <link>"
    │   │
    │   └─ On failure:
    │       ├─ Updates artifact status → 'publish_failed'
    │       ├─ Writes ledger entry with error details
    │       └─ Notifies PM: "Publish failed: <reason>. You can retry from the artifact view."
    │
    └─ Frontend polls or receives websocket update → re-renders artifact with new status
```

### The Revision Flow (HIL → Orchestrator → Child Agent → New Version)

When the PM clicks **Request Changes**, they provide feedback (free-text or inline comments). This routes back through the orchestrator to the child agent that originally created the artifact.

```
PM clicks "Request Changes" in React UI
    │
    ├─ Frontend POST /api/artifacts/{artifact_id}/review
    │   Body: {
    │     action: "request_revision",
    │     feedback: "Add a section on rate limiting edge cases. The technical reqs
    │                need to address what happens when a user exceeds the daily limit.",
    │     inline_comments: [
    │       { "line": 42, "comment": "This should reference the existing preferences API" }
    │     ]
    │   }
    │
    ├─ Backend updates artifact status → 'revision_requested'
    ├─ Backend writes artifact_reviews row with feedback
    ├─ Backend sends message to orchestrator session
    │
    ├─ Orchestrator receives revision request
    │   ├─ Writes ledger entry: entry_type='user_clarification',
    │   │   summary='PM requested revisions to PRD: rate limiting edge cases, preferences API reference'
    │   ├─ Updates STM: adds revision context to relevant plan step
    │   ├─ Updates artifact status → 'revising'
    │   ├─ Assembles child agent payload:
    │   │   {
    │   │     task: "revise_artifact",
    │   │     artifact_id: "uuid",
    │   │     current_content: "...current markdown...",
    │   │     feedback: "...PM feedback...",
    │   │     inline_comments: [...],
    │   │     decisions: [...relevant decisions from STM...],
    │   │     constraints: [...relevant constraints...]
    │   │   }
    │   └─ Dispatches to Confluence child agent (or whichever agent authored it) via A2A
    │
    ├─ Child agent returns revised markdown
    │
    ├─ Orchestrator:
    │   ├─ Increments version on artifacts table
    │   ├─ Inserts new row in artifact_versions (snapshot of new content)
    │   ├─ Updates artifacts.content with new markdown
    │   ├─ Updates artifact status → 'review'
    │   ├─ Writes ledger entry: 'artifact_updated', 'PRD revised (v3): added rate limiting edge cases'
    │   └─ Notifies PM in chat: "PRD has been revised. Please review v3."
    │
    └─ Frontend re-renders artifact with new content + Review/Approve/Reject buttons
```

### Jira Publish — Special Case

Jira artifacts (epics, stories) are structurally different from documents. A "user stories" artifact in Postgres contains markdown with multiple stories defined. Publishing to Jira means creating multiple issues, not a single page.

The publish flow is the same pattern (PM clicks Publish → orchestrator → Jira child agent), but the child agent:
1. Parses the markdown to extract individual stories
2. Creates each as a Jira issue
3. Returns a mapping: `{ stories_created: [{ title, jira_key, url }, ...] }`
4. The orchestrator updates the artifact with all Jira keys and marks it published

The artifact's `external_id` becomes a comma-separated list of Jira keys, or you can store the full mapping in a JSONB column. The `external_url` can point to the Jira board filtered to those issues.

### Frontend API Endpoints

Your React frontend needs these endpoints to drive the artifact lifecycle:

```
GET    /api/sessions/{session_id}/artifacts          ← list artifacts for session
GET    /api/artifacts/{artifact_id}                   ← get artifact with current content
GET    /api/artifacts/{artifact_id}/versions          ← version history
GET    /api/artifacts/{artifact_id}/versions/{version} ← specific version content
POST   /api/artifacts/{artifact_id}/review            ← approve, request revision, reject
POST   /api/artifacts/{artifact_id}/publish           ← trigger publish flow
POST   /api/artifacts/{artifact_id}/retry-publish     ← retry failed publish
```

The `/review` and `/publish` endpoints write to `artifact_reviews`, update `artifacts.status`, and send a message to the orchestrator session to trigger the agent-side flow.

### STM Working Context Update

The `artifact_refs` in the STM Working Context should reflect lifecycle state:

```json
"artifact_refs": [
  {
    "type": "prd",
    "title": "Notifications PRD v1",
    "artifact_id": "uuid-123",
    "version": 2,
    "status": "review",
    "publish_target": "confluence",
    "target_config": { "space": "NOTIFY", "parent_page_id": "..." },
    "external_url": null
  },
  {
    "type": "user_stories",
    "title": "Notifications User Stories",
    "artifact_id": "uuid-456",
    "version": 1,
    "status": "draft",
    "publish_target": "jira",
    "target_config": { "project": "NOTIFY", "board": "Sprint Board" },
    "external_url": null
  }
]
```

After publish, `external_url` is populated and `status` becomes `published`. The orchestrator tracks this in STM so it knows which artifacts still need attention without querying Postgres.

---

## 8. Key Design Principles

1. **Write to ledger on structural events, not vibes.** The triggers are deterministic — guided option selected, child agent returned, artifact created. The orchestrator authors the content but doesn't decide whether to write.

2. **STM Working Context carries conclusions. Ledger carries reasoning.** The working context stays lean. "No email in v1" is in STM. "Because enterprise clients use mobile and email adds 3 weeks" is in the ledger.

3. **Lazy-load from ledger, don't poll.** Normal turns never read the ledger. Cold starts and explicit recall do. This keeps the hot path fast.

4. **The ledger is agent infrastructure, not a user feature.** PMs see the chat, the guided options, and the artifacts. If you want a "decision history" view later, build a separate read model.

5. **STM snapshots are your crash recovery.** If the orchestrator dies mid-session, the snapshot + ledger replay gets you back to a consistent state without replaying the full conversation.

6. **LTM promotion is explicit, not automatic, for project decisions.** The `batch_create_memory_records` API gives you deterministic control over what persists across sessions. The ledger is the staging area — the `promoted_to_ltm` flag prevents duplicates.

7. **Let AgentCore handle what it's good at.** User preferences are low-stakes and conversational — the built-in UserPreference strategy with a custom `append_to_prompt` handles these well. Don't over-engineer what the managed pipeline already does.

8. **Two STM layers exist — understand the distinction.** AgentCore STM (raw conversation events via `create_event`) is the platform's log. Your STM Working Context JSON is the orchestrator's compressed working memory. Both serve different purposes and both are needed.

9. **Agents draft, humans publish.** No agent pushes directly to Confluence or Jira. Everything is a draft in Postgres until the PM explicitly approves and publishes. The publish action routes through the orchestrator to maintain the agent coordination pattern.

10. **Artifact versions are immutable snapshots.** Every revision creates a new version row. The PM can always see what changed and roll back. The `artifacts` table holds the current state; `artifact_versions` holds the history.

---

## 9. API Reference Summary

| Operation | Source | Used For |
|---|---|---|
| `write_ledger` | Custom Postgres tool | Recording structural events during session |
| `query_ledger` | Custom Postgres tool | Cold start replay, scoped retrieval, explicit recall |
| `save_stm_snapshot` | Custom Postgres tool | Checkpoint persistence |
| `load_stm_snapshot` | Custom Postgres tool | Session resume |
| `AgentCoreMemorySessionManager` | `bedrock_agentcore.memory.integrations.strands` | STM event storage + LTM retrieval on session start |
| `AgentCoreMemoryConfig` | `bedrock_agentcore.memory.integrations.strands.config` | Memory + retrieval configuration |
| `RetrievalConfig` | `bedrock_agentcore.memory.integrations.strands.config` | Per-namespace retrieval settings (top_k, relevance_score) |
| `batch_create_memory_records` | `boto3 bedrock-agentcore` Data Plane | Explicit LTM promotion from ledger (max 100/batch) |
| `retrieve_memory_records` | `boto3 bedrock-agentcore` Data Plane | On-demand LTM semantic search (~200ms) |
| `create_event` | `boto3 bedrock-agentcore` Data Plane (via session manager) | Raw conversation event storage to AgentCore STM |
| `MemoryClient.create_memory_and_wait` | `bedrock_agentcore.memory` | One-time memory resource setup |

---

## 10. Citations & Evidence: AgentCore LTM Control Points

A common misconception is that AgentCore LTM is fully automatic with no developer control. This is incorrect. The documentation explicitly provides multiple levels of control over what gets extracted, how it's consolidated, what model runs the extraction, and the ability to bypass the extraction pipeline entirely and batch-ingest your own records.

### You can override extraction and consolidation prompts

> *"Built-in with overrides strategies let you override the default behavior of the built-in strategies by providing your own instructions and selecting a specific foundation model for the extraction and consolidation steps."*

**Source**: [Customize a built-in strategy or create your own strategy — Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-custom-strategy.html)

The `appendToPrompt` field lets you steer what the extraction LLM looks for. You can constrain it to only extract specific types of information (e.g., "only capture dietary preferences, ignore clothing") or adjust granularity (e.g., "only save high-level key points rather than a detailed narrative").

### You can choose the model that runs extraction and consolidation

> *"AgentCore Memory also supports custom model selection for memory extraction and consolidation. This flexibility helps developers balance accuracy and latency based on their specific needs."*

**Source**: [Building smarter AI agents: AgentCore long-term memory deep dive — AWS Blog](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/)

The `modelId` field in both `extraction` and `consolidation` config accepts any Bedrock model ID. Use Haiku for fast/cheap extraction, Sonnet for higher-accuracy consolidation.

### You can completely own the extraction pipeline (self-managed strategies)

> *"Complete ownership of memory processing pipeline. Custom extraction and consolidation algorithms using any model, prompts, etc. Full control over memory record schemas, namespaces etc."*

**Source**: [Memory strategies — Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-strategies.html)

Self-managed strategies let you implement your own extraction logic and use AgentCore purely for storage and retrieval.

### You can directly batch-ingest records into LTM, bypassing extraction entirely

> *"Using the Batch APIs, you can directly ingest extracted records into AgentCore Memory while maintaining full ownership of the processing logic."*

**Source**: [Building smarter AI agents: AgentCore long-term memory deep dive — AWS Blog](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/)

The `batch_create_memory_records` API is the verified mechanism. Full API reference:

**Source**: [batch-create-memory-records — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/bedrock-agentcore/batch-create-memory-records.html)

Each record takes: `requestIdentifier` (string, max 80 chars), `namespaces` (list, max 1), `content.text` (string, max 16,000 chars), `timestamp`, and optional `memoryStrategyId`. Max 100 records per batch call. Response returns `successfulRecords` and `failedRecords` with individual `memoryRecordId` and status.

### You can retrieve LTM records with semantic search and filtering

> *"Searches for and retrieves memory records from an AgentCore Memory resource based on specified search criteria."*

**Source**: [retrieve_memory_records — Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agentcore/client/retrieve_memory_records.html)

Supports `searchQuery` (semantic), `memoryStrategyId` (filter by strategy), `topK`, and `metadataFilters`. Results include a relevance `score` based on cosine similarity. Retrieval latency is approximately 200ms.

### You control memory organization via namespaces with IAM-level access control

> *"When you create an AgentCore Memory, use a namespace to specify where the long-term memories for a memory strategy are logically grouped. [...] You should use a hierarchical format separated by forward slashes."*

**Source**: [Specify long-term memory organization with namespaces — Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/specify-long-term-memory-organization.html)

Namespaces support variables (`{actorId}`, `{sessionId}`, `{memoryStrategyId}`) and can be locked down with IAM policies to restrict which agents or users can read/write to specific namespaces.

### You can also delete and update LTM records

The AgentCore Data Plane API includes `batch_delete_memory_records`, `batch_update_memory_records`, `delete_memory_record`, and `get_memory_record` operations, giving full CRUD control over what's stored.

**Source**: [bedrock-agentcore CLI command index — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/bedrock-agentcore/)

### The extraction prompts are documented and fully editable

> *"These prompts are fully editable or can be used as is. These prompts are appended to a non-editable system prompt which is required to return the extracted memory in an expected output format."*

**Source**: [Extraction prompts — Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/custom-prompts-extraction.html)

The default extraction prompts for semantic, user preference, and summary strategies are published in the docs. You can read them, understand what they do, and override them with your own instructions.

---

### Summary of LTM Control Points

| Control | Mechanism | Documentation |
|---|---|---|
| What gets extracted | `appendToPrompt` on extraction config | [Custom strategy docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-custom-strategy.html) |
| How records are consolidated | `appendToPrompt` on consolidation config | [Custom strategy docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-custom-strategy.html) |
| Which model runs extraction | `modelId` on extraction/consolidation config | [Custom strategy config](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/long-term-configuring-custom-strategies.html) |
| Full pipeline ownership | Self-managed strategies | [Memory strategies overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-strategies.html) |
| Direct record ingestion | `batch_create_memory_records` API | [CLI reference](https://docs.aws.amazon.com/cli/latest/reference/bedrock-agentcore/batch-create-memory-records.html) |
| Semantic search + filtering | `retrieve_memory_records` API | [Boto3 reference](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agentcore/client/retrieve_memory_records.html) |
| Record CRUD | `batch_update`, `batch_delete`, `get`, `delete` | [CLI command index](https://docs.aws.amazon.com/cli/latest/reference/bedrock-agentcore/) |
| Memory organization | Hierarchical namespaces with IAM policies | [Namespace docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/specify-long-term-memory-organization.html) |
| Default prompt visibility | Published extraction/consolidation prompts | [Extraction prompts](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/custom-prompts-extraction.html) |
