# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Developer Onboarding Assistant · Created: 2026-05-11

## Philosophy

This model splits the data into two tiers: a **property graph layer** for the code knowledge graph (entities, relationships, traversals) and a **relational layer** for operational CRUD data (users, organizations, journeys, Q&A sessions). The graph layer uses PostgreSQL's `ltree` extension for hierarchical path queries and a generic node/edge table pair that models the code knowledge graph as a first-class property graph. The relational layer handles structured operational data with standard tables.

This dual-tier approach is inspired by how production graph-based code analysis systems work. FalkorDB's CodeGraph and Memgraph's GraphRAG for Devs both model codebases as property graphs with typed nodes and edges, then expose graph query languages for traversal. The Sourcegraph SCIP indexer produces graph data (symbols, references, definitions) that is stored in a graph-queryable format. Apache AGE (A Graph Extension for PostgreSQL) brings Cypher query support directly to PostgreSQL, enabling graph traversals without leaving the relational database.

For the Developer Onboarding Assistant, the graph-relational approach is the strongest architectural fit because the core product capability — multi-hop architectural reasoning ("why does payment call inventory before order-service?") — is fundamentally a graph traversal problem. Storing the code knowledge graph as explicit nodes and edges with typed properties enables shortest-path queries, community detection, impact analysis, and PageRank-style importance scoring that are either impossible or extremely awkward with relational JOINs alone.

**Best for:** Products where relationship traversal is the primary value proposition — understanding call chains, dependency graphs, impact analysis, ownership chains, and architectural reasoning across service boundaries.

**Trade-offs:**
- (+) Graph traversals (multi-hop, shortest path, impact analysis) are natural and efficient
- (+) Adding new node types or edge types requires no schema migration — just new type values
- (+) Knowledge graph structure enables AI-powered reasoning over code architecture
- (+) Graph algorithms (PageRank, community detection, centrality) can identify key code entities
- (+) Compatible with Apache AGE for Cypher queries within PostgreSQL
- (-) Generic node/edge tables lose some type safety compared to dedicated entity tables
- (-) JOINs between graph layer and relational layer require extra mapping
- (-) Property graph pattern is less familiar to developers accustomed to traditional ORM patterns
- (-) GIN indexes on JSONB properties are less efficient than typed column indexes for high-cardinality fields
- (-) Requires careful node/edge lifecycle management to avoid orphaned graph data

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIP (Code Intelligence Protocol) | SCIP symbol IDs stored as node properties — the graph node for a code entity carries its SCIP symbol as a canonical external identifier |
| Property Graph Model (ISO/IEC GQL) | Node/edge schema follows the emerging ISO GQL (Graph Query Language) property graph model: nodes have labels, edges have types, both carry properties |
| Tree-sitter Grammar Specs | AST node types map to graph node labels; Tree-sitter-derived relationships (contains, calls, imports) map to edge types |
| Apache AGE (Graph Extension) | Schema designed for compatibility with Apache AGE, enabling Cypher queries over the same tables |
| JSON-LD 1.1 | Graph data can be exported as JSON-LD for interoperability with linked data systems and external knowledge graphs |
| OpenAPI 3.x | API endpoint nodes include OpenAPI operation metadata in their properties |
| `ltree` (PostgreSQL) | Hierarchical containment paths (module > class > method) modeled using `ltree` for efficient ancestor/descendant queries |

---

## Graph Layer: Code Knowledge Graph

```sql
-- ============================================================
-- GRAPH LAYER: PROPERTY GRAPH FOR CODE KNOWLEDGE
-- ============================================================

CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS vector;

-- ============================================================
-- GRAPH NODES (all code entities, knowledge nodes, architecture concepts)
-- ============================================================

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL,                  -- FK enforced at app level (graph queries avoid JOINs)
    label           TEXT NOT NULL,                  -- node type: file, module, class, function, method,
                                                    -- interface, enum, api_endpoint, service, package,
                                                    -- knowledge_topic, architecture_pattern
    name            TEXT NOT NULL,                  -- human-readable short name
    qualified_name  TEXT NOT NULL,                  -- fully qualified path
    hierarchy_path  ltree,                          -- containment hierarchy path
                                                    -- e.g. "src.services.auth.AuthService.validateToken"
    -- Node properties (vary by label)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for label="class":
    -- {
    --   "scip_symbol": "scip-typescript npm @org/backend 1.0.0 src/services/`auth.ts`/AuthService#",
    --   "language": "typescript",
    --   "file_path": "src/services/auth.ts",
    --   "start_line": 15,
    --   "end_line": 142,
    --   "signature": "export class AuthService implements IAuthProvider",
    --   "docstring": "Handles JWT-based authentication",
    --   "is_exported": true,
    --   "complexity_score": 12.5,
    --   "last_modified_commit": "a1b2c3d",
    --   "last_modified_at": "2026-05-09T10:00:00Z"
    -- }
    --
    -- Example for label="service":
    -- {
    --   "deployment_url": "https://auth.internal.example.com",
    --   "team_owner": "platform-team",
    --   "health_check": "/healthz",
    --   "dependencies": ["user-service", "session-store"]
    -- }
    --
    -- Example for label="knowledge_topic":
    -- {
    --   "description": "Authentication and authorization patterns",
    --   "related_articles": ["article-uuid-1", "article-uuid-2"],
    --   "importance_score": 0.92
    -- }

    -- Computed graph metrics (updated by background jobs)
    in_degree       INTEGER NOT NULL DEFAULT 0,     -- number of incoming edges
    out_degree      INTEGER NOT NULL DEFAULT 0,     -- number of outgoing edges
    pagerank        REAL,                           -- PageRank score within repo graph
    betweenness     REAL,                           -- betweenness centrality
    community_id    INTEGER,                        -- community detection cluster ID

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nodes_repo ON graph_nodes(repository_id);
CREATE INDEX idx_nodes_label ON graph_nodes(label);
CREATE INDEX idx_nodes_name ON graph_nodes(name);
CREATE INDEX idx_nodes_qualified ON graph_nodes(qualified_name);
CREATE INDEX idx_nodes_hierarchy ON graph_nodes USING gist(hierarchy_path);
CREATE INDEX idx_nodes_props_gin ON graph_nodes USING gin(properties);
CREATE INDEX idx_nodes_community ON graph_nodes(repository_id, community_id) WHERE community_id IS NOT NULL;
CREATE INDEX idx_nodes_pagerank ON graph_nodes(repository_id, pagerank DESC NULLS LAST);

-- ============================================================
-- GRAPH EDGES (all relationships between nodes)
-- ============================================================

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,                  -- calls, imports, extends, implements, contains,
                                                    -- depends_on, tests, overrides, references,
                                                    -- documents, explains, belongs_to
    weight          REAL NOT NULL DEFAULT 1.0,
    -- Edge properties (vary by edge_type)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for edge_type="calls":
    -- {"call_count": 5, "is_conditional": true, "in_error_handler": false, "call_sites": [42, 67, 89]}
    -- Example for edge_type="documents":
    -- {"article_id": "article-uuid", "relevance": "primary", "confidence": 0.92}
    -- Example for edge_type="depends_on":
    -- {"dependency_type": "runtime", "version_constraint": "^2.0.0"}

    classification  TEXT NOT NULL DEFAULT 'extracted', -- extracted, inferred, user_defined
                                                       -- (per 2026 research: classify relationship provenance)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Enforce uniqueness per edge type (allow multiple edge types between same nodes)
CREATE UNIQUE INDEX idx_edges_unique ON graph_edges(source_id, target_id, edge_type);
CREATE INDEX idx_edges_source ON graph_edges(source_id);
CREATE INDEX idx_edges_target ON graph_edges(target_id);
CREATE INDEX idx_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_edges_classification ON graph_edges(classification);
CREATE INDEX idx_edges_props_gin ON graph_edges USING gin(properties);

-- ============================================================
-- GRAPH NODE EMBEDDINGS (for vector search over graph entities)
-- ============================================================

CREATE TABLE graph_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    embedding_model TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    chunk_text      TEXT NOT NULL,
    chunk_index     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_embeddings_node ON graph_embeddings(node_id);
CREATE INDEX idx_graph_embeddings_hnsw ON graph_embeddings
    USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200);
```

## Relational Layer: Operational Data

```sql
-- ============================================================
-- RELATIONAL LAYER: OPERATIONAL CRUD
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    profile         JSONB NOT NULL DEFAULT '{}',    -- languages, frameworks, experience, auth tokens
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    clone_url       TEXT NOT NULL,
    default_branch  TEXT NOT NULL DEFAULT 'main',
    primary_language TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_indexed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, full_name)
);

CREATE INDEX idx_repos_org ON repositories(organization_id);

-- ============================================================
-- KNOWLEDGE ARTICLES (relational, linked to graph via edges)
-- ============================================================

CREATE TABLE knowledge_articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,
    article_type    TEXT NOT NULL,
    source_type     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    confidence_score REAL,
    author_id       UUID REFERENCES users(id),
    -- Sources and anchors in JSONB (same as hybrid model)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Graph node ID for this article (articles also exist as graph nodes
    -- with label="knowledge_topic" for graph traversal)
    graph_node_id   UUID REFERENCES graph_nodes(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_articles_repo ON knowledge_articles(repository_id);
CREATE INDEX idx_articles_type ON knowledge_articles(article_type);
CREATE INDEX idx_articles_graph_node ON knowledge_articles(graph_node_id) WHERE graph_node_id IS NOT NULL;

-- ============================================================
-- ONBOARDING JOURNEYS
-- ============================================================

CREATE TABLE onboarding_journeys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    developer_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    generation_method TEXT NOT NULL,
    -- Steps stored as JSONB (same as hybrid model)
    steps           JSONB NOT NULL DEFAULT '[]',
    total_steps     INTEGER GENERATED ALWAYS AS (jsonb_array_length(steps)) STORED,
    completed_steps INTEGER NOT NULL DEFAULT 0,
    -- Graph-powered: journey was generated by traversing the code graph
    -- starting from these seed nodes
    seed_node_ids   UUID[] NOT NULL DEFAULT '{}',
    graph_traversal_depth INTEGER DEFAULT 3,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_journeys_org ON onboarding_journeys(organization_id);
CREATE INDEX idx_journeys_dev ON onboarding_journeys(developer_id);

-- ============================================================
-- Q&A SESSIONS
-- ============================================================

CREATE TABLE qa_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    messages        JSONB NOT NULL DEFAULT '[]',
    message_count   INTEGER NOT NULL DEFAULT 0,
    total_tokens    INTEGER NOT NULL DEFAULT 0,
    -- Graph context: which graph traversal strategy was used for retrieval
    retrieval_strategy TEXT DEFAULT 'hybrid',       -- graph_only, vector_only, hybrid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qa_sessions_user ON qa_sessions(user_id);
CREATE INDEX idx_qa_sessions_repo ON qa_sessions(repository_id);

-- ============================================================
-- GIT HISTORY
-- ============================================================

CREATE TABLE commits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    sha             TEXT NOT NULL,
    author_name     TEXT NOT NULL,
    author_email    TEXT NOT NULL,
    message         TEXT NOT NULL,
    committed_at    TIMESTAMPTZ NOT NULL,
    analysis        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, sha)
);

CREATE INDEX idx_commits_repo ON commits(repository_id);
CREATE INDEX idx_commits_date ON commits(committed_at);

-- ============================================================
-- ANALYTICS
-- ============================================================

CREATE TABLE analytics_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    event_type      TEXT NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analytics_org ON analytics_events(organization_id);
CREATE INDEX idx_analytics_type ON analytics_events(event_type);
CREATE INDEX idx_analytics_time ON analytics_events(occurred_at);
```

## Graph Query Examples

```sql
-- ============================================================
-- GRAPH QUERY PATTERNS
-- ============================================================

-- 1. Multi-hop dependency traversal: "What does AuthService depend on, 3 levels deep?"
WITH RECURSIVE deps AS (
    SELECT n.id, n.name, n.label, n.qualified_name,
           ARRAY[n.id] AS path, 1 AS depth
    FROM graph_nodes n
    WHERE n.name = 'AuthService'
      AND n.repository_id = $1
      AND n.label = 'class'

    UNION ALL

    SELECT target.id, target.name, target.label, target.qualified_name,
           deps.path || target.id, deps.depth + 1
    FROM deps
    JOIN graph_edges e ON e.source_id = deps.id
        AND e.edge_type IN ('calls', 'depends_on', 'imports')
    JOIN graph_nodes target ON target.id = e.target_id
    WHERE deps.depth < 3
      AND target.id != ALL(deps.path)  -- prevent cycles
)
SELECT DISTINCT name, label, qualified_name, depth
FROM deps
ORDER BY depth, name;

-- 2. Hierarchy query using ltree: "All methods inside AuthService"
SELECT name, qualified_name, properties
FROM graph_nodes
WHERE hierarchy_path <@ 'src.services.auth.AuthService'  -- descendants of AuthService
  AND label = 'method';

-- 3. Impact analysis: "What would break if I change UserRepository?"
WITH RECURSIVE impact AS (
    SELECT n.id, n.name, n.label, 0 AS distance
    FROM graph_nodes n
    WHERE n.name = 'UserRepository' AND n.repository_id = $1

    UNION ALL

    SELECT caller.id, caller.name, caller.label, impact.distance + 1
    FROM impact
    JOIN graph_edges e ON e.target_id = impact.id
        AND e.edge_type IN ('calls', 'depends_on', 'imports')
    JOIN graph_nodes caller ON caller.id = e.source_id
    WHERE impact.distance < 4
)
SELECT name, label, MIN(distance) AS min_distance, COUNT(*) AS path_count
FROM impact
WHERE distance > 0
GROUP BY name, label
ORDER BY min_distance, path_count DESC;

-- 4. Find most important nodes (by PageRank)
SELECT name, label, qualified_name, pagerank, in_degree, out_degree
FROM graph_nodes
WHERE repository_id = $1
  AND pagerank IS NOT NULL
ORDER BY pagerank DESC
LIMIT 20;

-- 5. Community detection: "Which code entities cluster together?"
SELECT community_id,
       COUNT(*) AS node_count,
       array_agg(name ORDER BY pagerank DESC NULLS LAST) AS top_entities
FROM graph_nodes
WHERE repository_id = $1
  AND community_id IS NOT NULL
GROUP BY community_id
ORDER BY node_count DESC;

-- 6. GraphRAG retrieval: combine graph traversal with vector search
-- Step 1: Find semantically similar nodes via vector search
WITH vector_hits AS (
    SELECT ge.node_id, 1 - (ge.embedding <=> $2::vector) AS similarity
    FROM graph_embeddings ge
    JOIN graph_nodes n ON n.id = ge.node_id AND n.repository_id = $1
    ORDER BY ge.embedding <=> $2::vector
    LIMIT 10
),
-- Step 2: Expand graph neighborhood of vector hits
graph_context AS (
    SELECT DISTINCT n.id, n.name, n.label, n.qualified_name, n.properties,
           vh.similarity, 'vector_hit' AS source
    FROM vector_hits vh
    JOIN graph_nodes n ON n.id = vh.node_id

    UNION

    SELECT DISTINCT neighbor.id, neighbor.name, neighbor.label,
           neighbor.qualified_name, neighbor.properties,
           vh.similarity * 0.8, 'graph_neighbor' AS source
    FROM vector_hits vh
    JOIN graph_edges e ON e.source_id = vh.node_id OR e.target_id = vh.node_id
    JOIN graph_nodes neighbor ON neighbor.id =
        CASE WHEN e.source_id = vh.node_id THEN e.target_id ELSE e.source_id END
)
SELECT * FROM graph_context ORDER BY similarity DESC;

-- 7. Knowledge article graph context: which articles document a code entity's neighborhood?
SELECT DISTINCT ka.id, ka.title, ka.article_type,
       e.edge_type, n.name AS code_entity_name
FROM knowledge_articles ka
JOIN graph_edges e ON e.source_id = ka.graph_node_id
    AND e.edge_type IN ('documents', 'explains')
JOIN graph_nodes n ON n.id = e.target_id
WHERE n.id IN (
    -- Get neighbors of the target entity
    SELECT CASE WHEN ge.source_id = $1 THEN ge.target_id ELSE ge.source_id END
    FROM graph_edges ge
    WHERE ge.source_id = $1 OR ge.target_id = $1
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 3 | graph_nodes, graph_edges, graph_embeddings |
| Multi-Tenancy & Users | 3 | organizations, users, organization_members |
| Repositories | 1 | repositories |
| Knowledge | 1 | knowledge_articles (linked to graph via graph_node_id) |
| Onboarding | 1 | onboarding_journeys (generated from graph traversal) |
| Q&A | 1 | qa_sessions |
| Git History | 1 | commits |
| Analytics | 1 | analytics_events |
| **Total** | **12** | Graph layer handles all code entity/relationship data |

---

## Key Design Decisions

1. **Generic node/edge tables instead of typed entity tables.** A single `graph_nodes` table with a `label` discriminator replaces the normalized model's separate tables for functions, classes, modules, interfaces, etc. This enables adding new entity types (services, packages, architecture patterns, knowledge topics) without schema migrations — just insert nodes with a new label value.

2. **`ltree` for hierarchical containment.** The `hierarchy_path` column uses PostgreSQL's `ltree` extension to represent file/module/class/method containment. Queries like "all methods inside AuthService" use the `<@` (descendant-of) operator, which is dramatically more efficient than recursive CTEs for pure hierarchy queries. The GiST index on `hierarchy_path` enables O(log n) lookups.

3. **Pre-computed graph metrics on nodes.** `in_degree`, `out_degree`, `pagerank`, `betweenness`, and `community_id` are stored directly on nodes. These are computed by background jobs after each index run using standard graph algorithms. Storing them on the node avoids recomputing expensive graph metrics at query time and enables instant "most important entities" queries.

4. **Edge classification: extracted vs. inferred vs. user-defined.** Following 2026 research on codebase knowledge graphs, relationships are classified by provenance. Extracted edges come from AST analysis (definite). Inferred edges come from LLM reasoning (probable). User-defined edges are manually added by developers. This classification allows the Q&A system to communicate confidence levels in its architectural explanations.

5. **Knowledge articles bridge graph and relational layers.** Each knowledge article has a `graph_node_id` that places it into the code knowledge graph. The article can then be connected to code entities via `graph_edges` with edge_type `documents` or `explains`. This means graph traversal naturally discovers relevant documentation alongside code entities — enabling answers like "here is the AuthService, and here is the architectural decision explaining why it validates tokens before checking permissions."

6. **GraphRAG retrieval as a two-phase query.** The example query pattern shows the core GraphRAG approach: Phase 1 uses pgvector to find semantically similar nodes, Phase 2 expands the graph neighborhood of those hits to gather structural context. This hybrid retrieval outperforms pure vector search for architectural questions because it captures relationships that semantic similarity misses.

7. **Onboarding journeys generated from graph traversal.** The `seed_node_ids` and `graph_traversal_depth` columns on `onboarding_journeys` record how the AI generated the journey: starting from seed entities (e.g., the services the new developer's team owns), traversing the code graph to discover related entities, and ordering them into a learning path. This makes journey generation reproducible and debuggable.

8. **Apache AGE compatibility.** The `graph_nodes`/`graph_edges` table structure is compatible with Apache AGE (A Graph Extension), which adds Cypher query support to PostgreSQL. If Cypher queries become needed for more complex graph patterns (variable-length paths, pattern matching), AGE can be layered on top of these tables without structural changes.
