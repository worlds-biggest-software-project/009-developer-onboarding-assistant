# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Developer Onboarding Assistant · Created: 2026-05-11

## Philosophy

This model uses a pragmatic hybrid approach: core structural columns are fully relational (typed, indexed, constrained), while variable, domain-specific, and extensible fields are stored in JSONB columns. This leverages PostgreSQL's JSONB capabilities — GIN indexing, containment operators, path queries — to provide schema-on-write for the stable core and schema-on-read for the variable periphery.

The hybrid pattern is the dominant approach in modern SaaS platforms. Sourcegraph stores code intelligence data with relational structure for entities and JSONB for language-specific metadata. GraphRAG-on-Postgres implementations (documented in builder guides from 2025-2026) use relational tables for graph nodes and edges with JSONB `properties` columns for variable node/edge attributes. This avoids the table explosion of a fully normalized model while maintaining stronger guarantees than a pure document store.

For the Developer Onboarding Assistant, the hybrid approach is particularly attractive because code entities vary dramatically by language (a Python function has decorators, a Java method has annotations and generics, a Go function has receiver types) and because knowledge articles, onboarding journeys, and Q&A sessions all have variable metadata that differs by context. Rather than creating columns for every possible language-specific attribute or knowledge source type, JSONB columns absorb this variability while the relational core ensures efficient joins and foreign key integrity.

**Best for:** Rapid MVP development where the schema needs to evolve quickly, multi-language codebases with variable entity metadata, and teams that want the flexibility of document stores with the query power of relational databases.

**Trade-offs:**
- (+) Fewer tables than normalized: ~18 vs ~27, reducing schema maintenance burden
- (+) Variable metadata (language-specific, integration-specific) fits naturally in JSONB
- (+) No schema migrations required when adding new metadata fields
- (+) JSONB GIN indexes enable efficient queries on variable attributes
- (+) Faster development velocity: new fields are just new JSON keys
- (-) JSONB fields lack foreign key constraints — referential integrity relies on application logic
- (-) JSONB queries can be slower than typed column queries for high-cardinality fields
- (-) Schema drift risk: without validation, JSONB fields can accumulate inconsistent shapes
- (-) Harder to enforce NOT NULL or CHECK constraints on fields inside JSONB
- (-) ORM mapping for JSONB requires more explicit configuration

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIP (Code Intelligence Protocol) | Symbol IDs stored as a typed relational column for indexing; language-specific SCIP metadata stored in `properties` JSONB |
| Tree-sitter Grammar Specs | Entity type taxonomy in `entity_type` column; grammar-specific AST metadata in `properties` JSONB |
| OpenAPI 3.x | API endpoint definitions stored as structured JSONB in `code_entities.properties` when `entity_type = 'api_endpoint'` |
| Conventional Commits v1.0.0 | Commit analysis results stored in `commits.properties` JSONB: `{"conventional": {"type": "feat", "scope": "auth", "breaking": false}}` |
| JSON Schema | JSONB validation rules defined per entity type; stored in `entity_type_schemas` for application-level validation |
| ISO 8601 | All timestamp columns use `TIMESTAMPTZ`; JSONB date fields use ISO 8601 strings |
| MCP (Model Context Protocol) | Integration metadata for MCP tool connections stored in `integrations.config` JSONB |

---

## Multi-Tenancy & Users

```sql
-- ============================================================
-- MULTI-TENANCY & USERS (minimal relational core)
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    -- Variable settings: LLM provider config, feature flags, branding, etc.
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "llm_provider": "ollama",
    --   "llm_model": "llama-3-70b",
    --   "llm_endpoint": "http://localhost:11434",
    --   "features": {"living_docs": true, "journey_generation": true},
    --   "branding": {"logo_url": "...", "primary_color": "#3B82F6"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    -- Variable profile data: auth tokens, developer experience, preferences
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "github_username": "janesmith",
    --   "gitlab_username": "jane.smith",
    --   "languages_known": ["typescript", "python", "go"],
    --   "frameworks_known": ["react", "fastapi", "gin"],
    --   "years_experience": 5,
    --   "auth_tokens": {
    --     "github": {"access_token": "...", "expires_at": "2026-06-01T00:00:00Z"},
    --     "gitlab": {"access_token": "...", "refresh_token": "..."}
    --   },
    --   "preferences": {"theme": "dark", "default_model": "claude-3.5-sonnet"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Example permissions:
    -- {
    --   "can_manage_repos": true,
    --   "can_publish_articles": true,
    --   "can_view_analytics": false,
    --   "repo_access": ["repo-uuid-1", "repo-uuid-2"]
    -- }
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);
```

## Repositories & Code Graph

```sql
-- ============================================================
-- REPOSITORIES
-- ============================================================

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    clone_url       TEXT NOT NULL,
    default_branch  TEXT NOT NULL DEFAULT 'main',
    primary_language TEXT,
    -- Variable repo metadata: index stats, CI config, team ownership, etc.
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "description": "Core authentication service",
    --   "topics": ["auth", "jwt", "microservice"],
    --   "team_owners": ["platform-team"],
    --   "codeowners_path": ".github/CODEOWNERS",
    --   "index_stats": {
    --     "last_indexed_at": "2026-05-10T14:30:00Z",
    --     "last_commit_sha": "a1b2c3d",
    --     "entity_count": 347,
    --     "relationship_count": 1204
    --   },
    --   "ci_config": {"provider": "github_actions", "has_doc_check": true}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, full_name)
);

CREATE INDEX idx_repos_org ON repositories(organization_id);
CREATE INDEX idx_repos_metadata_gin ON repositories USING gin(metadata);

-- ============================================================
-- CODE ENTITIES (hybrid: relational core + JSONB properties)
-- ============================================================

CREATE TABLE code_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,

    -- Relational core: always present, always queryable, always indexed
    entity_type     TEXT NOT NULL,                  -- function, class, module, interface, enum, method, api_endpoint
    name            TEXT NOT NULL,
    qualified_name  TEXT NOT NULL,
    file_path       TEXT NOT NULL,
    language        TEXT NOT NULL,
    scip_symbol     TEXT,

    -- JSONB properties: language-specific, entity-type-specific, variable
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for a TypeScript class:
    -- {
    --   "start_line": 15, "end_line": 142,
    --   "signature": "export class AuthService implements IAuthProvider",
    --   "docstring": "Handles JWT-based authentication",
    --   "decorators": [],
    --   "generic_params": [],
    --   "is_exported": true,
    --   "is_abstract": false,
    --   "complexity_score": 12.5,
    --   "last_modified_commit": "a1b2c3d",
    --   "last_modified_at": "2026-05-09T10:00:00Z"
    -- }
    --
    -- Example for a Python function:
    -- {
    --   "start_line": 45, "end_line": 78,
    --   "signature": "def validate_token(token: str, *, strict: bool = True) -> TokenPayload",
    --   "docstring": "Validates a JWT token and returns the decoded payload.",
    --   "decorators": ["@require_auth", "@rate_limit(100)"],
    --   "return_type": "TokenPayload",
    --   "params": [
    --     {"name": "token", "type": "str", "default": null},
    --     {"name": "strict", "type": "bool", "default": "True", "keyword_only": true}
    --   ],
    --   "is_async": false,
    --   "complexity_score": 8.2
    -- }
    --
    -- Example for an API endpoint:
    -- {
    --   "method": "POST",
    --   "path": "/api/v1/auth/login",
    --   "request_body_schema": {"type": "object", "properties": {"email": {"type": "string"}, "password": {"type": "string"}}},
    --   "response_schema": {"type": "object", "properties": {"token": {"type": "string"}}},
    --   "auth_required": true,
    --   "rate_limited": true,
    --   "openapi_operation_id": "loginUser"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entities_repo ON code_entities(repository_id);
CREATE INDEX idx_entities_type ON code_entities(entity_type);
CREATE INDEX idx_entities_file ON code_entities(repository_id, file_path);
CREATE INDEX idx_entities_name ON code_entities(name);
CREATE INDEX idx_entities_lang ON code_entities(language);
CREATE INDEX idx_entities_scip ON code_entities(scip_symbol) WHERE scip_symbol IS NOT NULL;
CREATE INDEX idx_entities_props_gin ON code_entities USING gin(properties);

-- ============================================================
-- CODE RELATIONSHIPS (hybrid: relational edges + JSONB metadata)
-- ============================================================

CREATE TABLE code_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    rel_type        TEXT NOT NULL,                  -- calls, imports, extends, implements, contains, depends_on, tests
    weight          REAL NOT NULL DEFAULT 1.0,
    -- Variable metadata per relationship type
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for "calls":
    -- {"call_count": 5, "is_conditional": true, "in_error_handler": false}
    -- Example for "imports":
    -- {"alias": "auth", "is_default_import": false, "is_type_only": true}
    -- Example for "extends":
    -- {"generic_args": ["User", "string"]}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, target_id, rel_type)
);

CREATE INDEX idx_rels_source ON code_relationships(source_id);
CREATE INDEX idx_rels_target ON code_relationships(target_id);
CREATE INDEX idx_rels_type ON code_relationships(rel_type);
```

## Knowledge & Documentation

```sql
-- ============================================================
-- KNOWLEDGE ARTICLES (hybrid: relational core + JSONB for sources and anchors)
-- ============================================================

CREATE TABLE knowledge_articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,
    article_type    TEXT NOT NULL,                  -- architectural_decision, pattern_explanation,
                                                    -- workaround, naming_convention, api_overview
    source_type     TEXT NOT NULL,                  -- extracted, authored, ai_generated, imported
    status          TEXT NOT NULL DEFAULT 'draft',
    confidence_score REAL,
    author_id       UUID REFERENCES users(id),

    -- Variable metadata: sources, anchors, related entities — all in JSONB
    -- This eliminates 3 junction/detail tables from the normalized model
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "sources": [
    --     {"type": "pull_request", "ref": "#247", "author": "jane.smith", "date": "2025-09-15", "excerpt": "Added token validation step..."},
    --     {"type": "commit", "ref": "f4e5d6c", "author": "jane.smith", "date": "2025-09-15"},
    --     {"type": "code_comment", "ref": "src/auth.ts:47", "excerpt": "// IMPORTANT: validate before permission check"}
    --   ],
    --   "related_entity_ids": ["entity-uuid-1", "entity-uuid-2"],
    --   "anchors": [
    --     {"file_path": "src/services/auth.ts", "line": 47, "text": "validateToken(token)", "hash": "abc123", "is_stale": false},
    --     {"file_path": "src/services/auth.ts", "line": 52, "text": "checkPermissions(user)", "hash": "def456", "is_stale": true}
    --   ],
    --   "tags": ["authentication", "security", "jwt"],
    --   "reviewed_by": "user-uuid",
    --   "published_at": "2026-05-10T14:00:00Z"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_articles_repo ON knowledge_articles(repository_id);
CREATE INDEX idx_articles_type ON knowledge_articles(article_type);
CREATE INDEX idx_articles_status ON knowledge_articles(status);
CREATE INDEX idx_articles_metadata_gin ON knowledge_articles USING gin(metadata);

-- Find articles with stale anchors:
-- SELECT * FROM knowledge_articles
-- WHERE metadata @> '{"anchors": [{"is_stale": true}]}';

-- Find articles tagged with "authentication":
-- SELECT * FROM knowledge_articles
-- WHERE metadata @> '{"tags": ["authentication"]}';
```

## Onboarding Journeys

```sql
-- ============================================================
-- ONBOARDING JOURNEYS (hybrid: relational structure + JSONB steps)
-- ============================================================

CREATE TABLE onboarding_journeys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    developer_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    generation_method TEXT NOT NULL,                -- ai_generated, manual, template

    -- Journey steps stored as JSONB array — eliminates the journey_steps table
    steps           JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "order": 1,
    --     "title": "Understand the AuthService",
    --     "description": "Read the AuthService class and understand JWT validation flow",
    --     "type": "explore_code",
    --     "target_entity_id": "entity-uuid",
    --     "estimated_minutes": 15,
    --     "status": "completed",
    --     "completed_at": "2026-05-10T10:30:00Z",
    --     "time_spent_minutes": 12,
    --     "notes": "Asked 2 questions during this step"
    --   },
    --   {
    --     "order": 2,
    --     "title": "Read: Why we chose JWT over session tokens",
    --     "description": "Architectural decision record explaining the auth strategy",
    --     "type": "read_article",
    --     "target_article_id": "article-uuid",
    --     "estimated_minutes": 5,
    --     "status": "in_progress",
    --     "completed_at": null
    --   }
    -- ]

    -- Denormalized progress counters (updated when steps JSONB changes)
    total_steps     INTEGER GENERATED ALWAYS AS (jsonb_array_length(steps)) STORED,
    completed_steps INTEGER NOT NULL DEFAULT 0,
    total_time_minutes INTEGER NOT NULL DEFAULT 0,

    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journeys_org ON onboarding_journeys(organization_id);
CREATE INDEX idx_journeys_dev ON onboarding_journeys(developer_id);
CREATE INDEX idx_journeys_status ON onboarding_journeys(status);
```

## Q&A Sessions

```sql
-- ============================================================
-- Q&A SESSIONS (hybrid: relational session + JSONB messages)
-- ============================================================

CREATE TABLE qa_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    title           TEXT,                           -- auto-generated from first question

    -- Messages stored as JSONB array — avoids a separate messages table
    messages        JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "role": "user",
    --     "content": "Why does the payment service call inventory before order-service?",
    --     "timestamp": "2026-05-10T14:00:00Z"
    --   },
    --   {
    --     "role": "assistant",
    --     "content": "The payment service calls inventory first because...",
    --     "timestamp": "2026-05-10T14:00:03Z",
    --     "model": "claude-3.5-sonnet",
    --     "tokens_used": 4200,
    --     "context_entities": [
    --       {"id": "entity-uuid-1", "name": "PaymentService.processOrder", "method": "graph_traversal"},
    --       {"id": "entity-uuid-2", "name": "InventoryService.checkStock", "method": "vector_search"}
    --     ]
    --   }
    -- ]

    message_count   INTEGER NOT NULL DEFAULT 0,
    total_tokens    INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qa_sessions_user ON qa_sessions(user_id);
CREATE INDEX idx_qa_sessions_repo ON qa_sessions(repository_id);
CREATE INDEX idx_qa_sessions_created ON qa_sessions(created_at);
```

## Embeddings & Git History

```sql
-- ============================================================
-- EMBEDDINGS (relational — high-volume, needs efficient indexing)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type     TEXT NOT NULL,                  -- code_entity, knowledge_article
    source_id       UUID NOT NULL,
    embedding_model TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    chunk_text      TEXT NOT NULL,
    chunk_index     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_source ON embeddings(source_type, source_id);
CREATE INDEX idx_embeddings_hnsw ON embeddings
    USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200);

-- ============================================================
-- GIT HISTORY (hybrid: relational core + JSONB for analysis results)
-- ============================================================

CREATE TABLE commits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    sha             TEXT NOT NULL,
    author_name     TEXT NOT NULL,
    author_email    TEXT NOT NULL,
    message         TEXT NOT NULL,
    committed_at    TIMESTAMPTZ NOT NULL,
    -- Analysis results stored in JSONB
    analysis        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "conventional": {"type": "feat", "scope": "auth", "breaking": false},
    --   "files_changed": 4,
    --   "insertions": 127,
    --   "deletions": 34,
    --   "entities_affected": ["entity-uuid-1", "entity-uuid-2"],
    --   "architectural_impact": "minor",
    --   "knowledge_extracted": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, sha)
);

CREATE INDEX idx_commits_repo ON commits(repository_id);
CREATE INDEX idx_commits_author ON commits(author_email);
CREATE INDEX idx_commits_date ON commits(committed_at);
CREATE INDEX idx_commits_analysis_gin ON commits USING gin(analysis);

-- Find all feat commits for auth scope:
-- SELECT * FROM commits WHERE analysis @> '{"conventional": {"type": "feat", "scope": "auth"}}';

-- ============================================================
-- ANALYTICS (single table with JSONB metrics)
-- ============================================================

CREATE TABLE analytics_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    event_type      TEXT NOT NULL,                  -- article_viewed, question_asked, journey_step_completed,
                                                    -- code_explored, first_pr_submitted
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "article_id": "article-uuid",
    --   "time_spent_seconds": 180,
    --   "repository_id": "repo-uuid"
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analytics_org ON analytics_events(organization_id);
CREATE INDEX idx_analytics_user ON analytics_events(user_id);
CREATE INDEX idx_analytics_type ON analytics_events(event_type);
CREATE INDEX idx_analytics_time ON analytics_events(occurred_at);
CREATE INDEX idx_analytics_props_gin ON analytics_events USING gin(properties);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Users | 3 | organizations, users, organization_members |
| Repositories | 1 | repositories (index stats in metadata JSONB) |
| Code Graph | 2 | code_entities, code_relationships |
| Knowledge | 1 | knowledge_articles (sources, anchors, tags all in metadata JSONB) |
| Onboarding | 1 | onboarding_journeys (steps in JSONB array) |
| Q&A | 1 | qa_sessions (messages in JSONB array) |
| Embeddings | 1 | embeddings (unified) |
| Git History | 1 | commits (analysis in JSONB) |
| Analytics | 1 | analytics_events (properties in JSONB) |
| **Total** | **12** | Reduced from 27 in normalized model by consolidating detail/junction tables into JSONB |

---

## Key Design Decisions

1. **JSONB absorbs junction tables and detail tables.** The normalized model's `knowledge_sources`, `documentation_anchors`, `knowledge_article_entities`, `journey_steps`, `qa_messages`, and `qa_context_entities` tables are all consolidated into JSONB columns on their parent tables. This halves the table count while keeping the data co-located with its parent — optimal for the common read pattern of "fetch entity with all its details."

2. **GIN indexes on all JSONB columns.** Every JSONB column has a GIN index, enabling efficient containment queries (`@>`) and existence checks (`?`). This makes queries like "find all articles with stale anchors" or "find all commits that affected auth scope" performant without relational joins.

3. **Relational columns for high-cardinality query fields.** Fields that appear in WHERE clauses, JOINs, or ORDER BY — `entity_type`, `name`, `file_path`, `language`, `repository_id` — are relational columns with B-tree indexes. JSONB is reserved for fields that are primarily read after the row is located.

4. **Language-specific metadata in `properties` JSONB.** A TypeScript class has `is_exported`, `generic_params`, and `decorators`. A Python function has `is_async`, `return_type`, and `decorators`. A Go function has `receiver_type`. Rather than null-heavy columns or entity-type subtables, all language-specific attributes live in `properties`. The `entity_type` column tells the application which JSON shape to expect.

5. **Journey steps as JSONB array with generated column for count.** `total_steps` is a `GENERATED ALWAYS AS (jsonb_array_length(steps)) STORED` column, keeping the progress counter synchronized without triggers. This pattern works because journey steps are always read and written as a batch (the whole journey), never queried individually across journeys.

6. **Q&A messages as JSONB array, not a separate table.** Chat sessions are typically short (5-20 messages) and always read as a complete conversation. Storing messages as a JSONB array eliminates the `qa_messages` and `qa_context_entities` tables. For sessions that grow very large, the application can archive older messages to a separate cold storage table.

7. **Analytics as a single event table with JSONB properties.** Rather than separate metrics tables or typed event tables, a single `analytics_events` table with an `event_type` discriminator and `properties` JSONB handles all analytics. This is the pattern used by Segment, Mixpanel, and PostHog — proven at scale for product analytics.

8. **Application-level JSON Schema validation.** Since JSONB columns lack database-level constraints on internal structure, the application validates JSONB payloads against JSON Schema definitions before writes. This provides the safety net that the database cannot enforce directly.

## Example Queries

### Find all code entities that are exported TypeScript classes
```sql
SELECT id, name, qualified_name, properties
FROM code_entities
WHERE repository_id = $1
  AND entity_type = 'class'
  AND language = 'typescript'
  AND properties @> '{"is_exported": true}';
```

### Find knowledge articles with stale documentation anchors
```sql
SELECT id, title, metadata->'anchors' AS anchors
FROM knowledge_articles
WHERE repository_id = $1
  AND metadata @> '{"anchors": [{"is_stale": true}]}';
```

### Onboarding journey progress with step details
```sql
SELECT
    j.title,
    j.total_steps,
    j.completed_steps,
    ROUND(j.completed_steps::numeric / NULLIF(j.total_steps, 0) * 100, 1) AS completion_pct,
    jsonb_path_query_array(j.steps, '$[*] ? (@.status == "in_progress")') AS current_steps
FROM onboarding_journeys j
WHERE j.developer_id = $1
  AND j.status = 'active';
```

### Multi-hop dependency traversal (same recursive CTE as normalized model)
```sql
WITH RECURSIVE deps AS (
    SELECT ce.id, ce.name, ce.qualified_name, 1 AS depth
    FROM code_entities ce
    WHERE ce.name = 'AuthService' AND ce.repository_id = $1

    UNION ALL

    SELECT target.id, target.name, target.qualified_name, deps.depth + 1
    FROM deps
    JOIN code_relationships rel ON rel.source_id = deps.id
        AND rel.rel_type IN ('calls', 'depends_on', 'imports')
    JOIN code_entities target ON target.id = rel.target_id
    WHERE deps.depth < 5
)
SELECT DISTINCT name, qualified_name, depth FROM deps ORDER BY depth;
```
