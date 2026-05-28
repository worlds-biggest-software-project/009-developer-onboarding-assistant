# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Developer Onboarding Assistant · Created: 2026-05-11

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every concept in the domain — repositories, code entities, knowledge articles, onboarding journeys, developers, questions — gets its own table with explicit foreign key relationships. Junction tables handle many-to-many associations. Reference data (programming languages, entity types, relationship types) lives in dedicated lookup tables.

The normalized approach prioritizes data integrity and query flexibility. It is the pattern used by mature platforms like Sourcegraph's internal PostgreSQL schema (where code intelligence data, repositories, and users are stored in separate, heavily-indexed tables) and by enterprise documentation platforms that must support complex cross-entity reporting. Every relationship is explicit and enforceable at the database level.

This approach is best when the team values strong consistency guarantees, needs complex ad-hoc queries across many entity types (e.g., "which knowledge articles cover services that developer X's assigned issues depend on?"), and expects the schema to stabilize relatively early in the product lifecycle.

**Best for:** Teams that prioritize data integrity, need complex cross-entity reporting, and have a well-understood domain model that will not change shape frequently.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Complex ad-hoc queries are straightforward with JOINs
- (+) Well-understood by any engineer with SQL experience
- (+) Migration tooling (Flyway, Alembic, Prisma) works naturally
- (-) High table count (~35-45 tables) increases schema maintenance burden
- (-) Adding a new entity type requires a migration and potentially new junction tables
- (-) Junction tables for many-to-many relationships add query complexity
- (-) Schema rigidity makes rapid prototyping slower than JSONB-based approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIP (Code Intelligence Protocol) | Symbol ID format (`scip-java`, `scip-typescript`) used as the canonical identifier for code entities — stored as `scip_symbol` column in `code_entities` |
| Tree-sitter Grammar Specs | Entity type taxonomy (function, class, module, interface, etc.) derived from Tree-sitter node types; stored in `entity_types` lookup table |
| OpenAPI 3.x | API endpoint entities parsed from OAS definitions stored in `api_endpoints` table with method, path, and schema references |
| Conventional Commits v1.0.0 | Commit type classification (feat, fix, refactor, etc.) stored in `commits.conventional_type` column for change-history analysis |
| ISO 8601 | All timestamps use `TIMESTAMPTZ` in ISO 8601 format |
| OAuth 2.0 / OIDC | User authentication tokens stored in `user_auth_tokens` with standard OAuth fields (access_token, refresh_token, expires_at) |
| JSON-LD 1.1 | Knowledge graph export format — not stored in DB but generated from normalized tables for interoperability |

---

## Multi-Tenancy & Authentication

```sql
-- ============================================================
-- MULTI-TENANCY & USER MANAGEMENT
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,           -- URL-safe identifier
    plan            TEXT NOT NULL DEFAULT 'free',   -- free, team, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',    -- org-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    github_username TEXT,
    gitlab_username TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member', -- owner, admin, member, viewer
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);

CREATE TABLE user_auth_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,                  -- github, gitlab, bitbucket
    access_token    TEXT NOT NULL,
    refresh_token   TEXT,
    token_type      TEXT NOT NULL DEFAULT 'Bearer',
    scope           TEXT,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_auth_tokens_user ON user_auth_tokens(user_id);
```

## Repository & Code Graph

```sql
-- ============================================================
-- REPOSITORY MANAGEMENT
-- ============================================================

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    full_name       TEXT NOT NULL,                  -- e.g. "org/repo-name"
    clone_url       TEXT NOT NULL,
    default_branch  TEXT NOT NULL DEFAULT 'main',
    primary_language TEXT,
    description     TEXT,
    last_indexed_at TIMESTAMPTZ,
    last_commit_sha TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, full_name)
);

CREATE INDEX idx_repos_org ON repositories(organization_id);

CREATE TABLE repository_index_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    commit_sha      TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, running, completed, failed
    entity_count    INTEGER,
    relationship_count INTEGER,
    duration_ms     INTEGER,
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_index_runs_repo ON repository_index_runs(repository_id);
CREATE INDEX idx_index_runs_status ON repository_index_runs(status);

-- ============================================================
-- CODE ENTITY GRAPH (NORMALIZED)
-- ============================================================

-- Lookup table for entity types derived from Tree-sitter node types
CREATE TABLE entity_types (
    id              SMALLINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name            TEXT NOT NULL UNIQUE,           -- function, class, module, interface, enum, variable, type_alias
    tree_sitter_type TEXT                            -- mapping to Tree-sitter grammar node type
);

INSERT INTO entity_types (name, tree_sitter_type) VALUES
    ('file', NULL),
    ('module', 'module'),
    ('class', 'class_declaration'),
    ('interface', 'interface_declaration'),
    ('function', 'function_declaration'),
    ('method', 'method_definition'),
    ('enum', 'enum_declaration'),
    ('variable', 'variable_declaration'),
    ('type_alias', 'type_alias_declaration'),
    ('api_endpoint', NULL);

-- Lookup table for relationship types
CREATE TABLE relationship_types (
    id              SMALLINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name            TEXT NOT NULL UNIQUE,           -- calls, imports, extends, implements, contains, depends_on, tests
    is_directional  BOOLEAN NOT NULL DEFAULT true
);

INSERT INTO relationship_types (name, is_directional) VALUES
    ('contains', true),        -- file contains class, class contains method
    ('calls', true),           -- function A calls function B
    ('imports', true),         -- module A imports module B
    ('extends', true),         -- class A extends class B
    ('implements', true),      -- class A implements interface B
    ('depends_on', true),      -- service A depends on service B
    ('tests', true),           -- test function tests target function
    ('overrides', true),       -- method overrides parent method
    ('references', true);      -- general reference

CREATE TABLE code_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    entity_type_id  SMALLINT NOT NULL REFERENCES entity_types(id),
    scip_symbol     TEXT,                           -- SCIP symbol ID (e.g. "scip-typescript npm @org/pkg 1.0.0 src/`auth.ts`/AuthService#")
    name            TEXT NOT NULL,                  -- short name (e.g. "AuthService")
    qualified_name  TEXT NOT NULL,                  -- fully qualified (e.g. "src/services/auth.AuthService")
    file_path       TEXT NOT NULL,                  -- relative path within repo
    start_line      INTEGER,
    end_line         INTEGER,
    language        TEXT NOT NULL,                  -- python, typescript, java, go, etc.
    signature       TEXT,                           -- function signature or class declaration line
    docstring       TEXT,                           -- extracted documentation comment
    complexity_score REAL,                          -- cyclomatic complexity if computed
    last_modified_commit TEXT,                      -- SHA of last commit that touched this entity
    last_modified_at TIMESTAMPTZ,
    index_run_id    UUID REFERENCES repository_index_runs(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_code_entities_repo ON code_entities(repository_id);
CREATE INDEX idx_code_entities_type ON code_entities(entity_type_id);
CREATE INDEX idx_code_entities_file ON code_entities(repository_id, file_path);
CREATE INDEX idx_code_entities_name ON code_entities(name);
CREATE INDEX idx_code_entities_scip ON code_entities(scip_symbol) WHERE scip_symbol IS NOT NULL;

CREATE TABLE code_entity_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_entity_id UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    target_entity_id UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    relationship_type_id SMALLINT NOT NULL REFERENCES relationship_types(id),
    weight          REAL DEFAULT 1.0,              -- strength/frequency of relationship
    metadata        TEXT,                           -- e.g. call count, import alias
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_entity_id, target_entity_id, relationship_type_id)
);

CREATE INDEX idx_entity_rels_source ON code_entity_relationships(source_entity_id);
CREATE INDEX idx_entity_rels_target ON code_entity_relationships(target_entity_id);
CREATE INDEX idx_entity_rels_type ON code_entity_relationships(relationship_type_id);
```

## Embeddings & Vector Search

```sql
-- ============================================================
-- VECTOR EMBEDDINGS (pgvector)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE code_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_entity_id  UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    embedding_model TEXT NOT NULL,                  -- e.g. "text-embedding-3-small", "nomic-embed-text"
    embedding       vector(1536) NOT NULL,          -- dimension matches model
    chunk_text      TEXT NOT NULL,                  -- the text that was embedded
    chunk_index     SMALLINT NOT NULL DEFAULT 0,    -- for entities split into multiple chunks
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_code_embeddings_entity ON code_embeddings(code_entity_id);
CREATE INDEX idx_code_embeddings_hnsw ON code_embeddings
    USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200);

CREATE TABLE knowledge_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    knowledge_article_id UUID NOT NULL REFERENCES knowledge_articles(id) ON DELETE CASCADE,
    embedding_model TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    chunk_text      TEXT NOT NULL,
    chunk_index     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_knowledge_embeddings_article ON knowledge_embeddings(knowledge_article_id);
CREATE INDEX idx_knowledge_embeddings_hnsw ON knowledge_embeddings
    USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200);
```

## Tribal Knowledge & Documentation

```sql
-- ============================================================
-- TRIBAL KNOWLEDGE & DOCUMENTATION
-- ============================================================

CREATE TABLE knowledge_articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,                  -- markdown content
    article_type    TEXT NOT NULL,                  -- architectural_decision, pattern_explanation,
                                                    -- workaround, naming_convention, deployment_note,
                                                    -- onboarding_guide, api_overview
    source_type     TEXT NOT NULL,                  -- extracted (from git/PR), authored (human-written),
                                                    -- ai_generated, imported
    confidence_score REAL,                          -- AI confidence for extracted knowledge (0.0-1.0)
    status          TEXT NOT NULL DEFAULT 'draft',  -- draft, review, published, archived
    author_id       UUID REFERENCES users(id),
    reviewed_by_id  UUID REFERENCES users(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_knowledge_articles_repo ON knowledge_articles(repository_id);
CREATE INDEX idx_knowledge_articles_type ON knowledge_articles(article_type);
CREATE INDEX idx_knowledge_articles_status ON knowledge_articles(status);

-- Junction: which code entities does a knowledge article explain?
CREATE TABLE knowledge_article_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES knowledge_articles(id) ON DELETE CASCADE,
    code_entity_id  UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    relevance       TEXT NOT NULL DEFAULT 'primary', -- primary, secondary, mentioned
    UNIQUE (article_id, code_entity_id)
);

CREATE INDEX idx_ka_entities_article ON knowledge_article_entities(article_id);
CREATE INDEX idx_ka_entities_entity ON knowledge_article_entities(code_entity_id);

-- Knowledge extracted from specific git artifacts
CREATE TABLE knowledge_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES knowledge_articles(id) ON DELETE CASCADE,
    source_type     TEXT NOT NULL,                  -- commit, pull_request, code_comment, issue, slack_message
    source_ref      TEXT NOT NULL,                  -- SHA, PR number, file:line, issue URL, etc.
    source_author   TEXT,                           -- git author or PR author
    source_date     TIMESTAMPTZ,
    excerpt         TEXT,                           -- relevant excerpt from the source
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_knowledge_sources_article ON knowledge_sources(article_id);

-- Staleness detection: link documentation to code lines
CREATE TABLE documentation_anchors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES knowledge_articles(id) ON DELETE CASCADE,
    code_entity_id  UUID REFERENCES code_entities(id) ON DELETE SET NULL,
    file_path       TEXT NOT NULL,
    anchor_line     INTEGER NOT NULL,
    anchor_text     TEXT NOT NULL,                  -- the code snippet anchored to
    anchor_hash     TEXT NOT NULL,                  -- hash of anchor_text for drift detection
    is_stale        BOOLEAN NOT NULL DEFAULT false,
    last_verified_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_anchors_article ON documentation_anchors(article_id);
CREATE INDEX idx_doc_anchors_file ON documentation_anchors(file_path);
CREATE INDEX idx_doc_anchors_stale ON documentation_anchors(is_stale) WHERE is_stale = true;
```

## Onboarding Journeys

```sql
-- ============================================================
-- ONBOARDING JOURNEYS
-- ============================================================

CREATE TABLE onboarding_journeys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    developer_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    description     TEXT,
    team_context    TEXT,                           -- which team/service area
    status          TEXT NOT NULL DEFAULT 'active', -- active, completed, paused, abandoned
    generation_method TEXT NOT NULL,                -- ai_generated, manual, template
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journeys_org ON onboarding_journeys(organization_id);
CREATE INDEX idx_journeys_developer ON onboarding_journeys(developer_id);

CREATE TABLE journey_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id      UUID NOT NULL REFERENCES onboarding_journeys(id) ON DELETE CASCADE,
    step_order      INTEGER NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    step_type       TEXT NOT NULL,                  -- read_article, explore_code, complete_exercise,
                                                    -- review_architecture, ask_question
    target_entity_id UUID REFERENCES code_entities(id) ON DELETE SET NULL,
    target_article_id UUID REFERENCES knowledge_articles(id) ON DELETE SET NULL,
    estimated_minutes INTEGER,
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, in_progress, completed, skipped
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journey_steps_journey ON journey_steps(journey_id);
CREATE INDEX idx_journey_steps_status ON journey_steps(status);

-- Track developer's prior experience for personalization
CREATE TABLE developer_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE UNIQUE,
    languages_known TEXT[] NOT NULL DEFAULT '{}',   -- {python, typescript, java}
    frameworks_known TEXT[] NOT NULL DEFAULT '{}',  -- {react, django, spring}
    years_experience INTEGER,
    github_repos_analyzed BOOLEAN NOT NULL DEFAULT false,
    experience_summary TEXT,                        -- AI-generated summary of background
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dev_profiles_user ON developer_profiles(user_id);
```

## Q&A Sessions & Analytics

```sql
-- ============================================================
-- Q&A SESSIONS
-- ============================================================

CREATE TABLE qa_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qa_sessions_user ON qa_sessions(user_id);
CREATE INDEX idx_qa_sessions_repo ON qa_sessions(repository_id);

CREATE TABLE qa_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES qa_sessions(id) ON DELETE CASCADE,
    role            TEXT NOT NULL,                  -- user, assistant
    content         TEXT NOT NULL,
    tokens_used     INTEGER,
    model_used      TEXT,                           -- gpt-4o, claude-3.5-sonnet, llama-3, etc.
    message_order   INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qa_messages_session ON qa_messages(session_id);

-- Which code entities were retrieved as context for each answer?
CREATE TABLE qa_context_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES qa_messages(id) ON DELETE CASCADE,
    code_entity_id  UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    retrieval_method TEXT NOT NULL,                 -- vector_search, graph_traversal, keyword_match
    relevance_score REAL,
    UNIQUE (message_id, code_entity_id)
);

CREATE INDEX idx_qa_context_message ON qa_context_entities(message_id);

-- ============================================================
-- ONBOARDING ANALYTICS
-- ============================================================

CREATE TABLE onboarding_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    developer_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    metric_type     TEXT NOT NULL,                  -- time_to_first_pr, time_to_10th_pr,
                                                    -- articles_viewed, questions_asked,
                                                    -- journey_completion_pct
    metric_value    REAL NOT NULL,
    measured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_onboarding_metrics_org ON onboarding_metrics(organization_id);
CREATE INDEX idx_onboarding_metrics_developer ON onboarding_metrics(developer_id);
CREATE INDEX idx_onboarding_metrics_type ON onboarding_metrics(metric_type);
```

## Commits & Change History

```sql
-- ============================================================
-- GIT HISTORY (for tribal knowledge extraction)
-- ============================================================

CREATE TABLE commits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    sha             TEXT NOT NULL,
    author_name     TEXT NOT NULL,
    author_email    TEXT NOT NULL,
    message         TEXT NOT NULL,
    conventional_type TEXT,                         -- feat, fix, refactor, docs, test, chore (per Conventional Commits)
    conventional_scope TEXT,                        -- scope from conventional commit (e.g. "auth", "api")
    committed_at    TIMESTAMPTZ NOT NULL,
    files_changed   INTEGER,
    insertions      INTEGER,
    deletions       INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, sha)
);

CREATE INDEX idx_commits_repo ON commits(repository_id);
CREATE INDEX idx_commits_author ON commits(author_email);
CREATE INDEX idx_commits_date ON commits(committed_at);
CREATE INDEX idx_commits_type ON commits(conventional_type) WHERE conventional_type IS NOT NULL;

CREATE TABLE pull_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    external_id     TEXT NOT NULL,                  -- PR number from GitHub/GitLab
    title           TEXT NOT NULL,
    body            TEXT,
    author          TEXT NOT NULL,
    state           TEXT NOT NULL,                  -- open, merged, closed
    merged_at       TIMESTAMPTZ,
    review_comments_count INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, external_id)
);

CREATE INDEX idx_prs_repo ON pull_requests(repository_id);
CREATE INDEX idx_prs_author ON pull_requests(author);

-- Which code entities were changed by which commits?
CREATE TABLE commit_entity_changes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commit_id       UUID NOT NULL REFERENCES commits(id) ON DELETE CASCADE,
    code_entity_id  UUID NOT NULL REFERENCES code_entities(id) ON DELETE CASCADE,
    change_type     TEXT NOT NULL,                  -- added, modified, deleted, renamed
    UNIQUE (commit_id, code_entity_id)
);

CREATE INDEX idx_commit_changes_commit ON commit_entity_changes(commit_id);
CREATE INDEX idx_commit_changes_entity ON commit_entity_changes(code_entity_id);
```

## Architecture Diagrams

```sql
-- ============================================================
-- ARCHITECTURE DIAGRAMS
-- ============================================================

CREATE TABLE architecture_diagrams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    diagram_type    TEXT NOT NULL,                  -- service_dependency, class_hierarchy, data_flow,
                                                    -- deployment, module_dependency
    format          TEXT NOT NULL DEFAULT 'mermaid', -- mermaid, plantuml, d2
    content         TEXT NOT NULL,                  -- diagram source code
    generated_from_commit TEXT,                     -- SHA at generation time
    is_stale        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_diagrams_repo ON architecture_diagrams(repository_id);
CREATE INDEX idx_diagrams_type ON architecture_diagrams(diagram_type);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Users | 4 | organizations, users, organization_members, user_auth_tokens |
| Repositories & Indexing | 2 | repositories, repository_index_runs |
| Code Graph | 4 | entity_types, relationship_types, code_entities, code_entity_relationships |
| Embeddings | 2 | code_embeddings, knowledge_embeddings |
| Knowledge & Documentation | 4 | knowledge_articles, knowledge_article_entities, knowledge_sources, documentation_anchors |
| Onboarding Journeys | 3 | onboarding_journeys, journey_steps, developer_profiles |
| Q&A & Analytics | 4 | qa_sessions, qa_messages, qa_context_entities, onboarding_metrics |
| Git History | 3 | commits, pull_requests, commit_entity_changes |
| Architecture Diagrams | 1 | architecture_diagrams |
| **Total** | **27** | |

---

## Key Design Decisions

1. **SCIP symbol IDs as canonical code entity identifiers.** The `scip_symbol` column on `code_entities` stores the SCIP Code Intelligence Protocol symbol string, providing a language-agnostic, human-readable identifier that is compatible with Sourcegraph and other SCIP-aware tools. This enables future interoperability without custom ID mapping.

2. **Separate lookup tables for entity types and relationship types.** Using `SMALLINT` foreign keys to `entity_types` and `relationship_types` instead of `TEXT` enum columns keeps the graph edge table compact and enables adding new types without schema migrations.

3. **Explicit junction tables for all many-to-many relationships.** `knowledge_article_entities`, `qa_context_entities`, and `commit_entity_changes` are dedicated junction tables rather than array columns or JSONB. This enables efficient joins and foreign key integrity at the cost of more tables.

4. **pgvector with HNSW indexes for embedding search.** Code and knowledge embeddings are stored in separate tables with HNSW indexes (cosine similarity). The `embedding_model` column supports model migration — when upgrading from `text-embedding-3-small` to a new model, old and new embeddings coexist until migration is complete.

5. **Documentation anchors for staleness detection.** The `documentation_anchors` table implements Swimm-style code-coupled documentation: each anchor stores a hash of the anchored code snippet. A CI hook can compare the current code to `anchor_hash` and set `is_stale = true` when drift is detected.

6. **Conventional Commits fields on the `commits` table.** Parsing commit messages according to the Conventional Commits spec and storing `conventional_type` and `conventional_scope` enables the tribal knowledge extraction pipeline to categorize changes and generate architectural evolution timelines.

7. **Developer profiles with array columns for language/framework experience.** PostgreSQL `TEXT[]` arrays for `languages_known` and `frameworks_known` support efficient containment queries (`WHERE 'typescript' = ANY(languages_known)`) for onboarding journey personalization without requiring additional junction tables.

8. **Organization-scoped multi-tenancy with `organization_id` foreign keys.** All tenant-scoped data flows through `organization_id`. Row-Level Security (RLS) policies can be layered on top of this design to enforce tenant isolation at the database level.

## Example Queries

### Multi-hop architectural query: "What does AuthService depend on?"
```sql
WITH RECURSIVE deps AS (
    SELECT ce.id, ce.name, ce.qualified_name, 1 AS depth
    FROM code_entities ce
    WHERE ce.name = 'AuthService' AND ce.repository_id = $1

    UNION ALL

    SELECT target.id, target.name, target.qualified_name, deps.depth + 1
    FROM deps
    JOIN code_entity_relationships rel ON rel.source_entity_id = deps.id
    JOIN relationship_types rt ON rt.id = rel.relationship_type_id AND rt.name IN ('calls', 'depends_on', 'imports')
    JOIN code_entities target ON target.id = rel.target_entity_id
    WHERE deps.depth < 5
)
SELECT DISTINCT name, qualified_name, depth FROM deps ORDER BY depth;
```

### Find stale documentation
```sql
SELECT ka.title, da.file_path, da.anchor_line
FROM documentation_anchors da
JOIN knowledge_articles ka ON ka.id = da.article_id
WHERE da.is_stale = true
  AND ka.repository_id = $1
ORDER BY ka.title;
```
