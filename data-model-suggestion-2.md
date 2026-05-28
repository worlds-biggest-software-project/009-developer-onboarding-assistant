# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Developer Onboarding Assistant · Created: 2026-05-11

## Philosophy

This model treats the immutable event log as the single source of truth for all system state. Every meaningful action — a repository being indexed, a code entity being discovered, a developer asking a question, a knowledge article being extracted, a documentation anchor going stale — is captured as an append-only event in a central event store. Current state is derived by replaying or materializing these events into read-optimized projections (CQRS pattern).

The event-sourced approach is used in production by systems like EventStoreDB, the Marten library for .NET, and financial trading platforms where complete audit trails and temporal queries ("what did the code graph look like on March 15th?") are non-negotiable. Meta's tribal knowledge mapping system (2026) used event-driven pipelines to track how institutional knowledge evolved over time — a pattern directly relevant to this project's mission of capturing the "why" behind code decisions.

For a Developer Onboarding Assistant, event sourcing is uniquely powerful because the core value proposition is understanding *how* a codebase evolved and *why* decisions were made. An event-sourced system naturally preserves this temporal dimension: you can replay the history of any code entity, knowledge article, or onboarding journey to understand its full lifecycle. The audit trail is not a bolt-on feature — it IS the data model.

**Best for:** Organizations that require full audit trails of knowledge evolution, need temporal queries ("what was true about this service on date X?"), and want AI analytics on change patterns over time.

**Trade-offs:**
- (+) Complete, immutable audit trail of every system change — ideal for tribal knowledge evolution
- (+) Temporal queries are first-class: replay events to reconstruct state at any point in time
- (+) Event replay enables AI analysis of how codebases and knowledge evolve
- (+) Natural fit for CQRS: optimize read models independently from write path
- (+) Events can feed real-time notification streams and analytics pipelines
- (-) Higher write amplification: every state change requires an event + projection update
- (-) Read model rebuilds can be slow for large event stores
- (-) More complex to implement and debug than direct CRUD
- (-) Eventual consistency between event store and projections requires careful handling
- (-) Schema evolution for events requires versioning strategy (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIP (Code Intelligence Protocol) | Symbol IDs stored as event payload fields for code entity events — enables replaying entity history by SCIP ID |
| Conventional Commits v1.0.0 | `CommitAnalyzed` events include parsed conventional commit fields (type, scope) as structured payload data |
| CloudEvents v1.0.2 | Event envelope format follows CloudEvents specification (type, source, subject, time, data) for interoperability |
| OCSF (Open Cybersecurity Schema Framework) | Activity event categorization borrows from OCSF's activity_id and category_uid taxonomy for structured event types |
| ISO 8601 | All event timestamps use `TIMESTAMPTZ` in ISO 8601 format; temporal queries use ISO 8601 intervals |
| JSON Schema | Event payloads validated against versioned JSON Schema definitions; schema registry tracks event versions |

---

## Event Store (Core)

```sql
-- ============================================================
-- EVENT STORE — SINGLE SOURCE OF TRUTH
-- ============================================================

-- Central event store: append-only, immutable
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                 -- aggregate/entity this event belongs to
    stream_type     TEXT NOT NULL,                  -- repository, code_entity, knowledge_article,
                                                    -- onboarding_journey, qa_session, user
    event_type      TEXT NOT NULL,                  -- e.g. RepositoryIndexed, CodeEntityDiscovered,
                                                    -- KnowledgeExtracted, QuestionAsked, JourneyStepCompleted
    event_version   INTEGER NOT NULL,               -- monotonically increasing per stream
    data            JSONB NOT NULL,                 -- event payload (see examples below)
    metadata        JSONB NOT NULL DEFAULT '{}',    -- correlation_id, causation_id, user_id, org_id
    schema_version  SMALLINT NOT NULL DEFAULT 1,    -- for event upcasting during replay
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Optimistic concurrency: no two events can have the same version in a stream
    UNIQUE (stream_id, event_version)
);

-- Partition by month for efficient temporal queries and retention management
-- (In production, use declarative partitioning on occurred_at)

-- Primary access patterns
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, occurred_at);
CREATE INDEX idx_events_stream_type ON events(stream_type, occurred_at);
CREATE INDEX idx_events_occurred ON events(occurred_at);
CREATE INDEX idx_events_metadata_org ON events((metadata->>'org_id'), occurred_at);
CREATE INDEX idx_events_metadata_user ON events((metadata->>'user_id'), occurred_at);

-- Event type registry: defines known event types and their schemas
CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    current_schema_version SMALLINT NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,                -- JSON Schema definition for the data payload
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Payload Examples

```jsonc
// RepositoryIndexStarted
{
  "event_type": "RepositoryIndexStarted",
  "stream_type": "repository",
  "data": {
    "repository_name": "org/backend-api",
    "clone_url": "https://github.com/org/backend-api.git",
    "commit_sha": "a1b2c3d",
    "branch": "main",
    "initiated_by": "user-uuid-here"
  }
}

// CodeEntityDiscovered
{
  "event_type": "CodeEntityDiscovered",
  "stream_type": "code_entity",
  "data": {
    "repository_id": "repo-uuid",
    "entity_type": "class",
    "name": "AuthService",
    "qualified_name": "src/services/auth.AuthService",
    "scip_symbol": "scip-typescript npm @org/backend 1.0.0 src/services/`auth.ts`/AuthService#",
    "file_path": "src/services/auth.ts",
    "start_line": 15,
    "end_line": 142,
    "language": "typescript",
    "signature": "export class AuthService implements IAuthProvider",
    "docstring": "Handles JWT-based authentication and session management"
  }
}

// CodeRelationshipDiscovered
{
  "event_type": "CodeRelationshipDiscovered",
  "stream_type": "code_entity",
  "data": {
    "source_entity_id": "entity-uuid-1",
    "target_entity_id": "entity-uuid-2",
    "relationship_type": "calls",
    "source_name": "AuthService.validateToken",
    "target_name": "UserRepository.findByEmail",
    "call_count": 3
  }
}

// KnowledgeExtracted
{
  "event_type": "KnowledgeExtracted",
  "stream_type": "knowledge_article",
  "data": {
    "title": "Why AuthService validates tokens before checking permissions",
    "content": "The token validation step was added in PR #247 after...",
    "article_type": "architectural_decision",
    "source_type": "extracted",
    "sources": [
      {"type": "pull_request", "ref": "#247", "author": "jane.smith", "date": "2025-09-15"},
      {"type": "commit", "ref": "f4e5d6c", "author": "jane.smith", "date": "2025-09-15"}
    ],
    "related_entities": ["entity-uuid-1", "entity-uuid-2"],
    "confidence_score": 0.87
  }
}

// QuestionAsked
{
  "event_type": "QuestionAsked",
  "stream_type": "qa_session",
  "data": {
    "question": "Why does the payment service call inventory before order-service?",
    "repository_id": "repo-uuid",
    "context_entities_retrieved": ["entity-uuid-3", "entity-uuid-4", "entity-uuid-5"],
    "retrieval_methods": ["graph_traversal", "vector_search"],
    "model_used": "claude-3.5-sonnet",
    "tokens_used": 4200
  }
}

// JourneyStepCompleted
{
  "event_type": "JourneyStepCompleted",
  "stream_type": "onboarding_journey",
  "data": {
    "journey_id": "journey-uuid",
    "step_order": 3,
    "step_type": "explore_code",
    "title": "Understand the AuthService",
    "time_spent_minutes": 12,
    "questions_asked_during": 2
  }
}

// DocumentationAnchorDrifted
{
  "event_type": "DocumentationAnchorDrifted",
  "stream_type": "knowledge_article",
  "data": {
    "article_id": "article-uuid",
    "file_path": "src/services/auth.ts",
    "anchor_line": 47,
    "original_hash": "abc123",
    "current_hash": "def456",
    "change_commit": "g7h8i9j",
    "drift_severity": "major"
  }
}
```

## Read Model Projections (Materialized Views)

```sql
-- ============================================================
-- PROJECTED READ MODELS (rebuilt from events)
-- ============================================================

-- Current state of organizations (projected from UserRegistered, OrgCreated, MemberAdded events)
CREATE TABLE proj_organizations (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    member_count    INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,                 -- watermark: last event projected
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proj_users (
    id              UUID PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    github_username TEXT,
    organizations   UUID[] NOT NULL DEFAULT '{}',   -- denormalized list of org IDs
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Current state of repositories
CREATE TABLE proj_repositories (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    clone_url       TEXT NOT NULL,
    default_branch  TEXT NOT NULL DEFAULT 'main',
    primary_language TEXT,
    entity_count    INTEGER NOT NULL DEFAULT 0,     -- denormalized from indexing events
    relationship_count INTEGER NOT NULL DEFAULT 0,
    article_count   INTEGER NOT NULL DEFAULT 0,
    last_indexed_at TIMESTAMPTZ,
    last_commit_sha TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_repos_org ON proj_repositories(organization_id);

-- Current state of code entities (projected from CodeEntityDiscovered, CodeEntityModified, etc.)
CREATE TABLE proj_code_entities (
    id              UUID PRIMARY KEY,
    repository_id   UUID NOT NULL,
    entity_type     TEXT NOT NULL,
    scip_symbol     TEXT,
    name            TEXT NOT NULL,
    qualified_name  TEXT NOT NULL,
    file_path       TEXT NOT NULL,
    start_line      INTEGER,
    end_line        INTEGER,
    language        TEXT NOT NULL,
    signature       TEXT,
    docstring       TEXT,
    complexity_score REAL,
    -- Denormalized relationship counts for fast display
    incoming_calls  INTEGER NOT NULL DEFAULT 0,
    outgoing_calls  INTEGER NOT NULL DEFAULT 0,
    total_changes   INTEGER NOT NULL DEFAULT 0,     -- times modified across all commits
    last_modified_at TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_entities_repo ON proj_code_entities(repository_id);
CREATE INDEX idx_proj_entities_name ON proj_code_entities(name);
CREATE INDEX idx_proj_entities_file ON proj_code_entities(repository_id, file_path);
CREATE INDEX idx_proj_entities_scip ON proj_code_entities(scip_symbol) WHERE scip_symbol IS NOT NULL;

-- Current relationships between code entities
CREATE TABLE proj_code_relationships (
    id              UUID PRIMARY KEY,
    source_entity_id UUID NOT NULL,
    target_entity_id UUID NOT NULL,
    relationship_type TEXT NOT NULL,
    weight          REAL DEFAULT 1.0,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_rels_source ON proj_code_relationships(source_entity_id);
CREATE INDEX idx_proj_rels_target ON proj_code_relationships(target_entity_id);

-- Current state of knowledge articles
CREATE TABLE proj_knowledge_articles (
    id              UUID PRIMARY KEY,
    repository_id   UUID NOT NULL,
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,
    article_type    TEXT NOT NULL,
    source_type     TEXT NOT NULL,
    confidence_score REAL,
    status          TEXT NOT NULL DEFAULT 'draft',
    author_id       UUID,
    is_stale        BOOLEAN NOT NULL DEFAULT false,
    stale_anchor_count INTEGER NOT NULL DEFAULT 0,
    related_entity_ids UUID[] NOT NULL DEFAULT '{}', -- denormalized
    version         INTEGER NOT NULL DEFAULT 1,     -- incremented on each update
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_articles_repo ON proj_knowledge_articles(repository_id);
CREATE INDEX idx_proj_articles_type ON proj_knowledge_articles(article_type);
CREATE INDEX idx_proj_articles_stale ON proj_knowledge_articles(is_stale) WHERE is_stale = true;

-- Current state of onboarding journeys
CREATE TABLE proj_onboarding_journeys (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    developer_id    UUID NOT NULL,
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    total_steps     INTEGER NOT NULL DEFAULT 0,
    completed_steps INTEGER NOT NULL DEFAULT 0,
    completion_pct  REAL NOT NULL DEFAULT 0.0,
    total_time_minutes INTEGER NOT NULL DEFAULT 0,
    questions_asked INTEGER NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_journeys_org ON proj_onboarding_journeys(organization_id);
CREATE INDEX idx_proj_journeys_dev ON proj_onboarding_journeys(developer_id);
```

## Vector Embeddings

```sql
-- ============================================================
-- EMBEDDINGS (not event-sourced — these are derived/computed data)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type     TEXT NOT NULL,                  -- code_entity, knowledge_article, qa_question
    source_id       UUID NOT NULL,                  -- ID of the source entity
    embedding_model TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    chunk_text      TEXT NOT NULL,
    chunk_index     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_source ON embeddings(source_type, source_id);
CREATE INDEX idx_embeddings_hnsw ON embeddings
    USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200);
```

## Snapshots (for Performance)

```sql
-- ============================================================
-- SNAPSHOTS — periodic state captures for fast replay
-- ============================================================

CREATE TABLE stream_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,              -- event_version at snapshot time
    state           JSONB NOT NULL,                 -- full aggregate state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON stream_snapshots(stream_id, snapshot_version DESC);

-- Projection checkpoints: track which event each projection has processed
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,               -- e.g. "proj_code_entities", "proj_knowledge_articles"
    last_event_id   UUID NOT NULL,
    last_occurred_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events (append-only), event_type_registry |
| Projected Read Models | 7 | proj_organizations, proj_users, proj_repositories, proj_code_entities, proj_code_relationships, proj_knowledge_articles, proj_onboarding_journeys |
| Embeddings | 1 | embeddings (unified table, not event-sourced) |
| Snapshots & Checkpoints | 2 | stream_snapshots, projection_checkpoints |
| **Total** | **12** | Plus any additional projections added for specific query patterns |

---

## Key Design Decisions

1. **Single `events` table with JSONB payloads.** Rather than separate event tables per stream type, all events go into one append-only table. The `stream_type` and `event_type` columns enable efficient filtering. JSONB payloads accommodate the diverse shapes of different event types while maintaining queryability. This is the pattern used by PostgreSQL event sourcing implementations like Marten and the `postgresql-event-sourcing` reference implementation.

2. **CloudEvents-inspired envelope format.** Events include `event_type`, `stream_id` (subject), `occurred_at` (time), and `data` (payload) — aligned with the CloudEvents v1.0.2 specification. This enables future integration with event streaming platforms (Kafka, CloudEvents-compatible brokers) without reformatting.

3. **Optimistic concurrency via `(stream_id, event_version)` uniqueness.** Each event within a stream has a monotonically increasing version number. The unique constraint prevents concurrent writes to the same aggregate, providing consistency without pessimistic locking.

4. **Projections are disposable and rebuildable.** All `proj_*` tables can be dropped and rebuilt by replaying events from the event store. The `projection_checkpoints` table tracks the replay watermark for each projection, enabling incremental catch-up after downtime.

5. **Snapshots for performance.** For aggregates with many events (e.g., a repository with thousands of CodeEntityDiscovered events), periodic snapshots capture the full state at a point in time. Replay starts from the snapshot rather than event zero.

6. **Embeddings are NOT event-sourced.** Vector embeddings are derived data (computed from code entity text), not business events. They are stored in a standard table and regenerated when the source entity changes, avoiding the overhead of event sourcing for purely computational artifacts.

7. **Event schema versioning with `schema_version` field.** When event payload structures change, the `schema_version` is incremented. Projection rebuilders use upcasting functions to transform old event versions to the current schema during replay. The `event_type_registry` stores JSON Schema definitions for validation.

8. **Temporal queries are first-class.** To answer "what did the code graph look like on March 15th?", replay events up to that date and materialize a point-in-time view. This is the killer feature for understanding how architecture and tribal knowledge evolved — the exact problem this product solves.

## Example Queries

### Replay: what knowledge articles existed for a repository on a specific date?
```sql
SELECT
    e.stream_id AS article_id,
    e.data->>'title' AS title,
    e.data->>'article_type' AS article_type,
    e.data->>'content' AS content,
    e.occurred_at
FROM events e
WHERE e.stream_type = 'knowledge_article'
  AND e.event_type IN ('KnowledgeExtracted', 'KnowledgeUpdated')
  AND e.metadata->>'org_id' = $1
  AND e.occurred_at <= '2026-03-15T00:00:00Z'
  AND e.stream_id NOT IN (
      SELECT stream_id FROM events
      WHERE event_type = 'KnowledgeArchived'
        AND occurred_at <= '2026-03-15T00:00:00Z'
  )
ORDER BY e.occurred_at DESC;
```

### Analytics: how many code entities were discovered per week?
```sql
SELECT
    date_trunc('week', occurred_at) AS week,
    COUNT(*) AS entities_discovered
FROM events
WHERE event_type = 'CodeEntityDiscovered'
  AND metadata->>'org_id' = $1
  AND occurred_at >= now() - INTERVAL '6 months'
GROUP BY date_trunc('week', occurred_at)
ORDER BY week;
```

### Tribal knowledge evolution: all changes to a specific knowledge article
```sql
SELECT
    event_type,
    event_version,
    data->>'title' AS title,
    data->>'content' AS content,
    metadata->>'user_id' AS changed_by,
    occurred_at
FROM events
WHERE stream_id = $1  -- article UUID
  AND stream_type = 'knowledge_article'
ORDER BY event_version;
```
