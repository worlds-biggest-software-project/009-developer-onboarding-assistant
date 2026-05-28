# Developer Onboarding Assistant — Phased Development Plan

> Project: 009-developer-onboarding-assistant · Created: 2026-05-11
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | Dominant language for VS Code extension development; shared language between backend API, CLI tooling, and VS Code extension eliminates cross-language friction. Strong type safety with the hybrid JSONB data model. |
| Runtime | Node.js 22 + Bun for scripts | Node.js 22 LTS for production API server (ecosystem maturity, VS Code extension compatibility); Bun for indexing scripts and CLI tools (faster startup, native TypeScript execution). |
| Web Framework | Fastify 5 | Lower overhead than Express; native JSON Schema validation aligns with JSONB payload validation; built-in OpenAPI 3.x generation for self-documenting API. |
| Database | PostgreSQL 17 + pgvector + ltree | PostgreSQL is the only database that supports relational tables, JSONB, vector search (pgvector), and hierarchical queries (ltree) in a single engine. Eliminates the need for a separate vector database. |
| ORM / Query Builder | Drizzle ORM | Type-safe SQL queries with full JSONB operator support; lighter than Prisma; generates migrations from TypeScript schema definitions. |
| AST Parsing | Tree-sitter (via node-tree-sitter) | MIT-licensed; grammars for 100+ languages; produces structured ASTs for call graph and dependency analysis. Industry standard used by Code-Graph-RAG, Sourcegraph, and Cursor. |
| Embedding Model | Default: nomic-embed-text (Ollama); configurable | 768-dimension embeddings via Ollama for self-hosted; falls back to OpenAI text-embedding-3-small (1536-dim) or Anthropic when configured. Keeps data local by default. |
| Vector Dimensions | 768 (default) / 1536 (OpenAI) | Store as vector(1536) with zero-padding for 768-dim models; avoids schema changes when switching embedding providers. |
| LLM Integration | LiteLLM proxy pattern | Unified interface to Ollama, OpenAI, Anthropic, Azure OpenAI. Single adapter layer; model selection at runtime via configuration. |
| Graph Algorithms | graphology (npm) | In-memory graph library for PageRank, betweenness centrality, community detection. Runs as background job after indexing; results written back to graph_nodes table. |
| Architecture Diagrams | Mermaid.js | Text-based diagram format renderable in VS Code, GitHub, and web UI. Generates from code graph data without external dependencies. |
| VS Code Extension | VS Code Extension API + webview | Native sidebar panel for Q&A; TreeView for code graph navigation; CodeLens for inline knowledge annotations. |
| Authentication | OAuth 2.0 (GitHub, GitLab) + local JWT | OAuth for repository access; JWT for API session management. Passport.js for OAuth flows. |
| Testing | Vitest + Playwright | Vitest for unit/integration (fast, native TypeScript); Playwright for VS Code extension e2e tests. |
| CI/CD | GitHub Actions | Standard for open-source TypeScript projects; can run PostgreSQL service containers for integration tests. |
| Containerization | Docker Compose (dev) + Dockerfile (prod) | Single docker-compose.yml for PostgreSQL + API + worker. Production Dockerfile for self-hosted deployment. |
| Package Manager | pnpm workspaces | Monorepo with workspace packages for api, cli, vscode-extension, core, and worker. |

### Project Structure

```
developer-onboarding-assistant/
├── packages/
│   ├── core/                          # Shared types, utilities, DB schema
│   │   ├── src/
│   │   │   ├── db/
│   │   │   │   ├── schema.ts          # Drizzle schema (graph + relational)
│   │   │   │   ├── migrations/        # SQL migrations
│   │   │   │   └── client.ts          # DB connection pool
│   │   │   ├── types/
│   │   │   │   ├── graph.ts           # GraphNode, GraphEdge, etc.
│   │   │   │   ├── knowledge.ts       # KnowledgeArticle, Source, Anchor
│   │   │   │   ├── journey.ts         # OnboardingJourney, JourneyStep
│   │   │   │   ├── qa.ts              # QASession, QAMessage
│   │   │   │   └── config.ts          # LLM config, org settings
│   │   │   ├── llm/
│   │   │   │   ├── provider.ts        # LLM provider interface
│   │   │   │   ├── ollama.ts          # Ollama adapter
│   │   │   │   ├── openai.ts          # OpenAI adapter
│   │   │   │   ├── anthropic.ts       # Anthropic adapter
│   │   │   │   └── embeddings.ts      # Embedding generation
│   │   │   └── utils/
│   │   │       ├── logger.ts
│   │   │       └── errors.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── indexer/                        # Code graph indexing engine
│   │   ├── src/
│   │   │   ├── parsers/
│   │   │   │   ├── tree-sitter.ts     # AST parsing via Tree-sitter
│   │   │   │   ├── typescript.ts      # TS-specific entity extraction
│   │   │   │   ├── python.ts          # Python-specific entity extraction
│   │   │   │   ├── java.ts            # Java-specific entity extraction
│   │   │   │   ├── go.ts              # Go-specific entity extraction
│   │   │   │   └── registry.ts        # Language parser registry
│   │   │   ├── graph/
│   │   │   │   ├── builder.ts         # Graph construction from AST
│   │   │   │   ├── relationships.ts   # Call graph, imports, inheritance
│   │   │   │   ├── hierarchy.ts       # ltree path generation
│   │   │   │   └── algorithms.ts      # PageRank, community detection
│   │   │   ├── git/
│   │   │   │   ├── clone.ts           # Repository cloning
│   │   │   │   ├── history.ts         # Commit/PR history extraction
│   │   │   │   └── blame.ts           # Git blame analysis
│   │   │   ├── embeddings/
│   │   │   │   └── indexer.ts         # Batch embedding generation
│   │   │   └── index.ts               # Main indexing pipeline
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── api/                            # HTTP API server
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   │   ├── repositories.ts
│   │   │   │   ├── graph.ts
│   │   │   │   ├── qa.ts
│   │   │   │   ├── knowledge.ts
│   │   │   │   ├── journeys.ts
│   │   │   │   ├── auth.ts
│   │   │   │   └── analytics.ts
│   │   │   ├── services/
│   │   │   │   ├── retrieval.ts       # GraphRAG retrieval pipeline
│   │   │   │   ├── qa.ts              # Q&A orchestration
│   │   │   │   ├── knowledge.ts       # Knowledge extraction
│   │   │   │   ├── journey.ts         # Journey generation
│   │   │   │   └── diagrams.ts        # Architecture diagram generation
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   └── org-context.ts
│   │   │   └── server.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── worker/                         # Background job processor
│   │   ├── src/
│   │   │   ├── jobs/
│   │   │   │   ├── index-repository.ts
│   │   │   │   ├── compute-graph-metrics.ts
│   │   │   │   ├── generate-embeddings.ts
│   │   │   │   ├── extract-knowledge.ts
│   │   │   │   ├── check-staleness.ts
│   │   │   │   └── generate-diagrams.ts
│   │   │   └── queue.ts               # Job queue (BullMQ + Redis)
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── cli/                            # CLI tool
│   │   ├── src/
│   │   │   ├── commands/
│   │   │   │   ├── index.ts           # doa index <repo-path>
│   │   │   │   ├── ask.ts             # doa ask "question"
│   │   │   │   ├── graph.ts           # doa graph <entity>
│   │   │   │   └── config.ts          # doa config set/get
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── vscode/                         # VS Code extension
│       ├── src/
│       │   ├── extension.ts
│       │   ├── sidebar/
│       │   │   ├── qa-panel.ts        # Q&A webview panel
│       │   │   └── graph-tree.ts      # Code graph TreeView
│       │   ├── codelens/
│       │   │   └── knowledge-lens.ts  # Inline knowledge annotations
│       │   └── commands/
│       │       ├── ask.ts
│       │       └── index-repo.ts
│       ├── package.json
│       └── tsconfig.json
├── docker-compose.yml                  # PostgreSQL + Redis + API + Worker
├── Dockerfile
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
└── vitest.config.ts
```

---

## Phase 1: Foundation — Database, Core Types, and Project Scaffold

### Purpose

Establish the monorepo structure, database schema, connection pooling, core TypeScript types, and LLM provider abstraction layer. This phase produces no user-facing functionality but creates the foundation that every subsequent phase builds on. All database tables, the graph-relational hybrid schema, and the LLM adapter interface are implemented and tested here.

### Tasks

#### 1.1 — Monorepo Scaffold and Build Configuration

**What**: Initialize the pnpm workspace monorepo with all six packages and shared TypeScript configuration.

**Design**:

```typescript
// pnpm-workspace.yaml
// packages: ["packages/*"]

// tsconfig.base.json — shared compiler options
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "esModuleInterop": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src",
    "resolveJsonModule": true,
    "skipLibCheck": true
  }
}

// Each package extends tsconfig.base.json and declares workspace dependencies:
// packages/core/package.json
{
  "name": "@doa/core",
  "version": "0.1.0",
  "type": "module",
  "exports": { ".": "./dist/index.js" },
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "db:migrate": "drizzle-kit migrate",
    "db:generate": "drizzle-kit generate"
  }
}
```

**Testing**:
- **scaffold-builds**: Run `pnpm build` from root; all six packages compile without errors.
- **workspace-deps**: `@doa/api` can import from `@doa/core`; TypeScript resolves types correctly.
- **vitest-runs**: `pnpm test` from root discovers and runs placeholder tests in all packages.

---

#### 1.2 — Database Schema (Graph-Relational Hybrid)

**What**: Implement the complete PostgreSQL schema using Drizzle ORM, incorporating the graph-relational hybrid model (Data Model Suggestion 4) with the JSONB flexibility from Suggestion 3.

**Design**:

```typescript
// packages/core/src/db/schema.ts

import { pgTable, uuid, text, timestamp, boolean, integer, real, jsonb, uniqueIndex, index } from "drizzle-orm/pg-core";

// ── Multi-Tenancy ──────────────────────────────────────────

export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: text("name").notNull(),
  slug: text("slug").notNull().unique(),
  plan: text("plan").notNull().default("free"),         // free | team | enterprise
  settings: jsonb("settings").notNull().default({}),    // LLM config, feature flags
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  displayName: text("display_name").notNull(),
  avatarUrl: text("avatar_url"),
  profile: jsonb("profile").notNull().default({}),      // auth tokens, languages, frameworks, experience
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const organizationMembers = pgTable("organization_members", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  role: text("role").notNull().default("member"),       // owner | admin | member | viewer
  joinedAt: timestamp("joined_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex("idx_org_members_unique").on(t.organizationId, t.userId),
]);

// ── Repositories ───────────────────────────────────────────

export const repositories = pgTable("repositories", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  fullName: text("full_name").notNull(),                // "org/repo-name"
  cloneUrl: text("clone_url").notNull(),
  defaultBranch: text("default_branch").notNull().default("main"),
  primaryLanguage: text("primary_language"),
  metadata: jsonb("metadata").notNull().default({}),    // index stats, team owners, CI config
  isActive: boolean("is_active").notNull().default(true),
  lastIndexedAt: timestamp("last_indexed_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex("idx_repos_unique").on(t.organizationId, t.fullName),
]);

// ── Graph Layer ────────────────────────────────────────────

export const graphNodes = pgTable("graph_nodes", {
  id: uuid("id").primaryKey().defaultRandom(),
  repositoryId: uuid("repository_id").notNull(),        // FK enforced at app level for graph query perf
  label: text("label").notNull(),                       // file | module | class | function | method |
                                                         // interface | enum | api_endpoint | service |
                                                         // knowledge_topic
  name: text("name").notNull(),
  qualifiedName: text("qualified_name").notNull(),
  hierarchyPath: text("hierarchy_path"),                // ltree path: "src.services.auth.AuthService"
  properties: jsonb("properties").notNull().default({}),
  inDegree: integer("in_degree").notNull().default(0),
  outDegree: integer("out_degree").notNull().default(0),
  pagerank: real("pagerank"),
  betweenness: real("betweenness"),
  communityId: integer("community_id"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index("idx_nodes_repo").on(t.repositoryId),
  index("idx_nodes_label").on(t.label),
  index("idx_nodes_name").on(t.name),
]);

export const graphEdges = pgTable("graph_edges", {
  id: uuid("id").primaryKey().defaultRandom(),
  sourceId: uuid("source_id").notNull().references(() => graphNodes.id, { onDelete: "cascade" }),
  targetId: uuid("target_id").notNull().references(() => graphNodes.id, { onDelete: "cascade" }),
  edgeType: text("edge_type").notNull(),                // calls | imports | extends | implements |
                                                         // contains | depends_on | tests | documents
  weight: real("weight").notNull().default(1.0),
  properties: jsonb("properties").notNull().default({}),
  classification: text("classification").notNull().default("extracted"),  // extracted | inferred | user_defined
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex("idx_edges_unique").on(t.sourceId, t.targetId, t.edgeType),
  index("idx_edges_source").on(t.sourceId),
  index("idx_edges_target").on(t.targetId),
  index("idx_edges_type").on(t.edgeType),
]);

// ── Knowledge Articles ─────────────────────────────────────

export const knowledgeArticles = pgTable("knowledge_articles", {
  id: uuid("id").primaryKey().defaultRandom(),
  repositoryId: uuid("repository_id").notNull().references(() => repositories.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  content: text("content").notNull(),
  articleType: text("article_type").notNull(),           // architectural_decision | pattern_explanation |
                                                          // workaround | naming_convention | api_overview
  sourceType: text("source_type").notNull(),             // extracted | authored | ai_generated | imported
  status: text("status").notNull().default("draft"),     // draft | review | published | archived
  confidenceScore: real("confidence_score"),
  authorId: uuid("author_id").references(() => users.id),
  metadata: jsonb("metadata").notNull().default({}),     // sources, anchors, tags, related_entity_ids
  graphNodeId: uuid("graph_node_id").references(() => graphNodes.id, { onDelete: "set null" }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Onboarding Journeys ────────────────────────────────────

export const onboardingJourneys = pgTable("onboarding_journeys", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  developerId: uuid("developer_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  status: text("status").notNull().default("active"),    // active | completed | paused | abandoned
  generationMethod: text("generation_method").notNull(), // ai_generated | manual | template
  steps: jsonb("steps").notNull().default([]),            // JourneyStep[]
  completedSteps: integer("completed_steps").notNull().default(0),
  totalTimeMinutes: integer("total_time_minutes").notNull().default(0),
  seedNodeIds: jsonb("seed_node_ids").notNull().default([]),  // UUID[] of starting graph nodes
  graphTraversalDepth: integer("graph_traversal_depth").default(3),
  startedAt: timestamp("started_at", { withTimezone: true }).notNull().defaultNow(),
  completedAt: timestamp("completed_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Q&A Sessions ───────────────────────────────────────────

export const qaSessions = pgTable("qa_sessions", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  repositoryId: uuid("repository_id").notNull().references(() => repositories.id, { onDelete: "cascade" }),
  title: text("title"),
  messages: jsonb("messages").notNull().default([]),     // QAMessage[]
  messageCount: integer("message_count").notNull().default(0),
  totalTokens: integer("total_tokens").notNull().default(0),
  retrievalStrategy: text("retrieval_strategy").default("hybrid"), // graph_only | vector_only | hybrid
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Git History ────────────────────────────────────────────

export const commits = pgTable("commits", {
  id: uuid("id").primaryKey().defaultRandom(),
  repositoryId: uuid("repository_id").notNull().references(() => repositories.id, { onDelete: "cascade" }),
  sha: text("sha").notNull(),
  authorName: text("author_name").notNull(),
  authorEmail: text("author_email").notNull(),
  message: text("message").notNull(),
  committedAt: timestamp("committed_at", { withTimezone: true }).notNull(),
  analysis: jsonb("analysis").notNull().default({}),     // conventional commit fields, files changed, etc.
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex("idx_commits_unique").on(t.repositoryId, t.sha),
]);

// ── Embeddings ─────────────────────────────────────────────

export const graphEmbeddings = pgTable("graph_embeddings", {
  id: uuid("id").primaryKey().defaultRandom(),
  nodeId: uuid("node_id").notNull().references(() => graphNodes.id, { onDelete: "cascade" }),
  embeddingModel: text("embedding_model").notNull(),
  embedding: text("embedding").notNull(),               // Stored as vector(1536) via raw SQL; text placeholder for Drizzle
  chunkText: text("chunk_text").notNull(),
  chunkIndex: integer("chunk_index").notNull().default(0),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Analytics ──────────────────────────────────────────────

export const analyticsEvents = pgTable("analytics_events", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  userId: uuid("user_id").references(() => users.id),
  eventType: text("event_type").notNull(),
  properties: jsonb("properties").notNull().default({}),
  occurredAt: timestamp("occurred_at", { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- **migration-applies**: Run `drizzle-kit migrate` against a fresh PostgreSQL 17 instance; all tables created without errors.
- **extension-enabled**: Verify `CREATE EXTENSION IF NOT EXISTS vector` and `CREATE EXTENSION IF NOT EXISTS ltree` succeed.
- **schema-roundtrip**: Insert a sample organization, user, repository, graph_node, graph_edge, and verify reads return correct types.
- **unique-constraints**: Attempt duplicate `(organization_id, full_name)` on repositories; verify unique violation error.
- **cascade-delete**: Delete an organization; verify all child repositories, graph_nodes, and knowledge_articles are removed.

---

#### 1.3 — Core TypeScript Types

**What**: Define the TypeScript interfaces and Zod validation schemas for all domain objects, including JSONB payload shapes.

**Design**:

```typescript
// packages/core/src/types/graph.ts

import { z } from "zod";

export const GraphNodeLabel = z.enum([
  "file", "module", "class", "function", "method",
  "interface", "enum", "api_endpoint", "service",
  "package", "knowledge_topic",
]);
export type GraphNodeLabel = z.infer<typeof GraphNodeLabel>;

export const EdgeType = z.enum([
  "calls", "imports", "extends", "implements", "contains",
  "depends_on", "tests", "overrides", "references",
  "documents", "explains",
]);
export type EdgeType = z.infer<typeof EdgeType>;

export const EdgeClassification = z.enum(["extracted", "inferred", "user_defined"]);
export type EdgeClassification = z.infer<typeof EdgeClassification>;

export const CodeEntityProperties = z.object({
  scipSymbol: z.string().optional(),
  language: z.string(),
  filePath: z.string(),
  startLine: z.number().int().optional(),
  endLine: z.number().int().optional(),
  signature: z.string().optional(),
  docstring: z.string().optional(),
  isExported: z.boolean().optional(),
  isAsync: z.boolean().optional(),
  complexityScore: z.number().optional(),
  decorators: z.array(z.string()).optional(),
  params: z.array(z.object({
    name: z.string(),
    type: z.string().optional(),
    default: z.string().nullable().optional(),
  })).optional(),
  returnType: z.string().optional(),
  lastModifiedCommit: z.string().optional(),
  lastModifiedAt: z.string().datetime().optional(),
}).passthrough();
export type CodeEntityProperties = z.infer<typeof CodeEntityProperties>;

export interface GraphNode {
  id: string;
  repositoryId: string;
  label: GraphNodeLabel;
  name: string;
  qualifiedName: string;
  hierarchyPath: string | null;
  properties: CodeEntityProperties;
  inDegree: number;
  outDegree: number;
  pagerank: number | null;
  betweenness: number | null;
  communityId: number | null;
}

export interface GraphEdge {
  id: string;
  sourceId: string;
  targetId: string;
  edgeType: EdgeType;
  weight: number;
  properties: Record<string, unknown>;
  classification: EdgeClassification;
}

// packages/core/src/types/knowledge.ts

export const KnowledgeSource = z.object({
  type: z.enum(["commit", "pull_request", "code_comment", "issue", "slack_message"]),
  ref: z.string(),
  author: z.string().optional(),
  date: z.string().datetime().optional(),
  excerpt: z.string().optional(),
});
export type KnowledgeSource = z.infer<typeof KnowledgeSource>;

export const DocumentationAnchor = z.object({
  filePath: z.string(),
  line: z.number().int(),
  text: z.string(),
  hash: z.string(),
  isStale: z.boolean().default(false),
});
export type DocumentationAnchor = z.infer<typeof DocumentationAnchor>;

export const KnowledgeArticleMetadata = z.object({
  sources: z.array(KnowledgeSource).default([]),
  anchors: z.array(DocumentationAnchor).default([]),
  relatedEntityIds: z.array(z.string().uuid()).default([]),
  tags: z.array(z.string()).default([]),
  reviewedBy: z.string().uuid().optional(),
  publishedAt: z.string().datetime().optional(),
});
export type KnowledgeArticleMetadata = z.infer<typeof KnowledgeArticleMetadata>;

// packages/core/src/types/journey.ts

export const JourneyStepType = z.enum([
  "read_article", "explore_code", "complete_exercise",
  "review_architecture", "ask_question",
]);

export const JourneyStep = z.object({
  order: z.number().int(),
  title: z.string(),
  description: z.string(),
  type: JourneyStepType,
  targetEntityId: z.string().uuid().optional(),
  targetArticleId: z.string().uuid().optional(),
  estimatedMinutes: z.number().int().optional(),
  status: z.enum(["pending", "in_progress", "completed", "skipped"]).default("pending"),
  completedAt: z.string().datetime().nullable().optional(),
  timeSpentMinutes: z.number().int().optional(),
  notes: z.string().optional(),
});
export type JourneyStep = z.infer<typeof JourneyStep>;

// packages/core/src/types/qa.ts

export const QAMessage = z.object({
  role: z.enum(["user", "assistant"]),
  content: z.string(),
  timestamp: z.string().datetime(),
  model: z.string().optional(),
  tokensUsed: z.number().int().optional(),
  contextEntities: z.array(z.object({
    id: z.string().uuid(),
    name: z.string(),
    method: z.enum(["graph_traversal", "vector_search", "keyword_match"]),
    relevanceScore: z.number().optional(),
  })).optional(),
});
export type QAMessage = z.infer<typeof QAMessage>;
```

**Testing**:
- **valid-properties**: Parse a valid `CodeEntityProperties` object through Zod; verify success.
- **invalid-properties**: Parse an object with `startLine: "not-a-number"`; verify Zod rejection with field path.
- **passthrough-preserves**: Parse properties with extra language-specific fields; verify they pass through.
- **journey-step-defaults**: Parse a minimal `JourneyStep`; verify `status` defaults to `"pending"`.

---

#### 1.4 — LLM Provider Abstraction

**What**: Implement a unified LLM provider interface supporting Ollama, OpenAI, and Anthropic for both chat completions and embedding generation.

**Design**:

```typescript
// packages/core/src/llm/provider.ts

export interface LLMMessage {
  role: "system" | "user" | "assistant";
  content: string;
}

export interface LLMCompletionOptions {
  model: string;
  messages: LLMMessage[];
  temperature?: number;       // default 0.1
  maxTokens?: number;         // default 4096
  stream?: boolean;           // default false
}

export interface LLMCompletionResult {
  content: string;
  model: string;
  tokensUsed: { prompt: number; completion: number; total: number };
  finishReason: "stop" | "length" | "error";
}

export interface EmbeddingOptions {
  model: string;
  texts: string[];            // batch of texts to embed
}

export interface EmbeddingResult {
  embeddings: number[][];     // one vector per input text
  model: string;
  dimensions: number;
  tokensUsed: number;
}

export interface LLMProvider {
  readonly name: string;
  complete(options: LLMCompletionOptions): Promise<LLMCompletionResult>;
  embed(options: EmbeddingOptions): Promise<EmbeddingResult>;
  listModels(): Promise<string[]>;
  healthCheck(): Promise<boolean>;
}

// packages/core/src/llm/ollama.ts

export class OllamaProvider implements LLMProvider {
  readonly name = "ollama";
  constructor(private baseUrl: string = "http://localhost:11434") {}

  async complete(options: LLMCompletionOptions): Promise<LLMCompletionResult> {
    const response = await fetch(`${this.baseUrl}/api/chat`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: options.model,
        messages: options.messages,
        stream: false,
        options: { temperature: options.temperature ?? 0.1, num_predict: options.maxTokens ?? 4096 },
      }),
    });
    const data = await response.json();
    return {
      content: data.message.content,
      model: data.model,
      tokensUsed: {
        prompt: data.prompt_eval_count ?? 0,
        completion: data.eval_count ?? 0,
        total: (data.prompt_eval_count ?? 0) + (data.eval_count ?? 0),
      },
      finishReason: data.done ? "stop" : "error",
    };
  }

  async embed(options: EmbeddingOptions): Promise<EmbeddingResult> {
    const response = await fetch(`${this.baseUrl}/api/embed`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ model: options.model, input: options.texts }),
    });
    const data = await response.json();
    return {
      embeddings: data.embeddings,
      model: data.model,
      dimensions: data.embeddings[0]?.length ?? 0,
      tokensUsed: data.total_duration ?? 0,
    };
  }

  async listModels(): Promise<string[]> {
    const response = await fetch(`${this.baseUrl}/api/tags`);
    const data = await response.json();
    return data.models.map((m: { name: string }) => m.name);
  }

  async healthCheck(): Promise<boolean> {
    try {
      const response = await fetch(`${this.baseUrl}/api/tags`);
      return response.ok;
    } catch { return false; }
  }
}

// packages/core/src/llm/factory.ts

export interface LLMConfig {
  provider: "ollama" | "openai" | "anthropic";
  baseUrl?: string;
  apiKey?: string;
  chatModel: string;
  embeddingModel: string;
}

export function createLLMProvider(config: LLMConfig): LLMProvider {
  switch (config.provider) {
    case "ollama": return new OllamaProvider(config.baseUrl);
    case "openai": return new OpenAIProvider(config.apiKey!, config.baseUrl);
    case "anthropic": return new AnthropicProvider(config.apiKey!);
    default: throw new Error(`Unknown LLM provider: ${config.provider}`);
  }
}
```

**Testing**:
- **ollama-complete** (integration, requires Ollama running): Send a simple prompt to Ollama; verify response structure matches `LLMCompletionResult`.
- **ollama-embed** (integration): Embed a single text; verify vector has expected dimensions (768 for nomic-embed-text).
- **ollama-health**: Call `healthCheck()` against a running Ollama instance; verify `true`.
- **ollama-health-down**: Call `healthCheck()` against an unreachable URL; verify `false` without throwing.
- **factory-creates-ollama**: Call `createLLMProvider({ provider: "ollama", chatModel: "llama3", embeddingModel: "nomic-embed-text" })`; verify instance is `OllamaProvider`.
- **factory-unknown-throws**: Call `createLLMProvider({ provider: "unknown" as any, ... })`; verify error thrown.

---

#### 1.5 — Docker Compose Development Environment

**What**: Create a Docker Compose configuration providing PostgreSQL 17 (with pgvector and ltree), Redis (for BullMQ job queue), and optional Ollama service.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg17
    environment:
      POSTGRES_DB: doa
      POSTGRES_USER: doa
      POSTGRES_PASSWORD: doa_dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./packages/core/src/db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U doa"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

```sql
-- packages/core/src/db/init.sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

**Testing**:
- **compose-up**: Run `docker compose up -d`; verify PostgreSQL and Redis are healthy within 30 seconds.
- **extensions-available**: Connect to PostgreSQL; run `SELECT * FROM pg_extension WHERE extname IN ('vector', 'ltree', 'pg_trgm')`; verify all three rows returned.
- **migrations-run**: Run Drizzle migrations against the Docker PostgreSQL; verify all tables exist.

---

## Phase 2: Code Indexing Engine — Tree-sitter Parsing and Graph Construction

### Purpose

Build the code indexing pipeline that parses repositories using Tree-sitter, extracts code entities (files, classes, functions, methods), identifies relationships (calls, imports, inheritance), constructs the property graph in PostgreSQL, and computes hierarchy paths. This phase delivers the core data that powers all downstream features: Q&A, knowledge extraction, and journey generation.

### Tasks

#### 2.1 — Tree-sitter Parser Infrastructure

**What**: Implement a language-agnostic Tree-sitter parsing layer with pluggable language-specific entity extractors for TypeScript, Python, Java, and Go.

**Design**:

```typescript
// packages/indexer/src/parsers/tree-sitter.ts

import Parser from "tree-sitter";

export interface ParsedEntity {
  name: string;
  qualifiedName: string;
  entityType: GraphNodeLabel;
  filePath: string;
  startLine: number;
  endLine: number;
  language: string;
  signature: string | null;
  docstring: string | null;
  properties: Record<string, unknown>;
  children: ParsedEntity[];          // nested entities (class > methods)
}

export interface ParsedRelationship {
  sourceQualifiedName: string;
  targetQualifiedName: string;
  edgeType: EdgeType;
  properties: Record<string, unknown>;
}

export interface LanguageParser {
  readonly language: string;
  readonly extensions: string[];
  parse(filePath: string, sourceCode: string): {
    entities: ParsedEntity[];
    relationships: ParsedRelationship[];
  };
}

// packages/indexer/src/parsers/registry.ts

export class ParserRegistry {
  private parsers = new Map<string, LanguageParser>();

  register(parser: LanguageParser): void {
    for (const ext of parser.extensions) {
      this.parsers.set(ext, parser);
    }
  }

  getParser(filePath: string): LanguageParser | null {
    const ext = filePath.split(".").pop() ?? "";
    return this.parsers.get(ext) ?? null;
  }

  supportedExtensions(): string[] {
    return Array.from(this.parsers.keys());
  }
}

// packages/indexer/src/parsers/typescript.ts

export class TypeScriptParser implements LanguageParser {
  readonly language = "typescript";
  readonly extensions = ["ts", "tsx"];
  private parser: Parser;
  private treeSitterLang: Parser.Language;

  constructor() {
    this.parser = new Parser();
    this.treeSitterLang = require("tree-sitter-typescript").typescript;
    this.parser.setLanguage(this.treeSitterLang);
  }

  parse(filePath: string, sourceCode: string): {
    entities: ParsedEntity[];
    relationships: ParsedRelationship[];
  } {
    const tree = this.parser.parse(sourceCode);
    const entities: ParsedEntity[] = [];
    const relationships: ParsedRelationship[] = [];

    // Walk AST, extract class_declaration, function_declaration,
    // method_definition, interface_declaration, enum_declaration,
    // import_statement, export_statement nodes.
    this.walkNode(tree.rootNode, filePath, sourceCode, entities, relationships);

    return { entities, relationships };
  }

  private walkNode(
    node: Parser.SyntaxNode,
    filePath: string,
    source: string,
    entities: ParsedEntity[],
    relationships: ParsedRelationship[],
    parentQualifiedName?: string,
  ): void {
    // Implementation: switch on node.type, extract entity info,
    // recurse into children. For import_statement nodes, create
    // ParsedRelationship entries.
  }
}
```

**Testing**:
- **parse-ts-class**: Parse a TypeScript file containing `export class AuthService implements IAuthProvider { validateToken(token: string): boolean {} }`; verify one class entity with one child method.
- **parse-ts-imports**: Parse a file with `import { UserRepo } from "./user-repo"`; verify an `imports` relationship from the file to `UserRepo`.
- **parse-py-function**: Parse `def process_payment(amount: float, currency: str = "USD") -> bool:`; verify function entity with params in properties.
- **parse-go-struct**: Parse a Go struct with methods; verify struct entity with method children.
- **registry-resolution**: Register TypeScript and Python parsers; verify `.ts` resolves to TypeScript, `.py` to Python, `.rs` returns null.

---

#### 2.2 — Repository Cloning and File Enumeration

**What**: Clone or fetch a Git repository to a local working directory, enumerate parseable files, and provide file content to the parser pipeline.

**Design**:

```typescript
// packages/indexer/src/git/clone.ts

import { simpleGit, SimpleGit } from "simple-git";

export interface CloneResult {
  localPath: string;
  commitSha: string;
  branch: string;
  fileCount: number;
}

export interface IndexableFile {
  path: string;           // relative to repo root
  absolutePath: string;
  extension: string;
  sizeBytes: number;
}

export async function cloneOrFetch(
  cloneUrl: string,
  targetDir: string,
  branch: string = "main",
): Promise<CloneResult> {
  const git: SimpleGit = simpleGit();
  const repoDir = path.join(targetDir, hashRepoUrl(cloneUrl));

  if (await fs.pathExists(repoDir)) {
    await simpleGit(repoDir).fetch("origin", branch);
    await simpleGit(repoDir).checkout(`origin/${branch}`);
  } else {
    await git.clone(cloneUrl, repoDir, ["--depth", "1", "--branch", branch]);
  }

  const log = await simpleGit(repoDir).log({ maxCount: 1 });
  const commitSha = log.latest!.hash;

  const files = await enumerateFiles(repoDir);
  return { localPath: repoDir, commitSha, branch, fileCount: files.length };
}

export async function enumerateFiles(
  repoDir: string,
  supportedExtensions: string[] = ["ts", "tsx", "py", "java", "go"],
): Promise<IndexableFile[]> {
  // Walk directory tree, skip node_modules, .git, vendor, __pycache__,
  // dist, build. Filter by supported extensions. Skip files > 1MB.
  // Return sorted list.
}
```

**Testing**:
- **clone-public-repo** (integration): Clone a small public GitHub repo; verify `localPath` exists and contains files.
- **enumerate-filters**: Create a temp directory with `.ts`, `.py`, `.md`, `node_modules/foo.ts`; verify enumeration returns only `.ts` and `.py`, excludes `node_modules`.
- **skip-large-files**: Create a temp file > 1MB; verify it is excluded from enumeration.
- **fetch-existing**: Clone then fetch the same repo; verify no error and commitSha is returned.

---

#### 2.3 — Graph Builder Pipeline

**What**: Orchestrate the parsing of all files, merge entities and relationships into graph_nodes and graph_edges tables, and compute ltree hierarchy paths.

**Design**:

```typescript
// packages/indexer/src/graph/builder.ts

export interface IndexResult {
  repositoryId: string;
  commitSha: string;
  nodesCreated: number;
  edgesCreated: number;
  filesProcessed: number;
  durationMs: number;
  errors: Array<{ filePath: string; error: string }>;
}

export class GraphBuilder {
  constructor(
    private db: DrizzleClient,
    private parserRegistry: ParserRegistry,
  ) {}

  async indexRepository(
    repositoryId: string,
    repoDir: string,
    commitSha: string,
  ): Promise<IndexResult> {
    const startTime = Date.now();
    const files = await enumerateFiles(repoDir, this.parserRegistry.supportedExtensions());

    // Phase 1: Parse all files, collect entities and relationships
    const allEntities: Array<ParsedEntity & { filePath: string }> = [];
    const allRelationships: ParsedRelationship[] = [];
    const errors: Array<{ filePath: string; error: string }> = [];

    for (const file of files) {
      const parser = this.parserRegistry.getParser(file.path);
      if (!parser) continue;
      try {
        const source = await fs.readFile(file.absolutePath, "utf-8");
        const { entities, relationships } = parser.parse(file.path, source);
        allEntities.push(...entities);
        allRelationships.push(...relationships);
      } catch (err) {
        errors.push({ filePath: file.path, error: String(err) });
      }
    }

    // Phase 2: Upsert graph nodes in a transaction
    const nodeIdMap = new Map<string, string>(); // qualifiedName -> node UUID
    await this.db.transaction(async (tx) => {
      // Delete existing nodes for this repo (full re-index)
      await tx.delete(graphNodes).where(eq(graphNodes.repositoryId, repositoryId));

      for (const entity of allEntities) {
        const hierarchyPath = entity.qualifiedName.replace(/[\/\\]/g, ".").replace(/-/g, "_");
        const [inserted] = await tx.insert(graphNodes).values({
          repositoryId,
          label: entity.entityType,
          name: entity.name,
          qualifiedName: entity.qualifiedName,
          hierarchyPath,
          properties: entity.properties,
        }).returning({ id: graphNodes.id });
        nodeIdMap.set(entity.qualifiedName, inserted.id);
      }

      // Phase 3: Insert edges
      for (const rel of allRelationships) {
        const sourceId = nodeIdMap.get(rel.sourceQualifiedName);
        const targetId = nodeIdMap.get(rel.targetQualifiedName);
        if (!sourceId || !targetId) continue;
        await tx.insert(graphEdges).values({
          sourceId,
          targetId,
          edgeType: rel.edgeType,
          properties: rel.properties,
          classification: "extracted",
        }).onConflictDoNothing();
      }
    });

    return {
      repositoryId,
      commitSha,
      nodesCreated: nodeIdMap.size,
      edgesCreated: allRelationships.length,
      filesProcessed: files.length,
      durationMs: Date.now() - startTime,
      errors,
    };
  }
}
```

**Testing**:
- **index-small-repo** (integration): Index a test fixture directory with 5 TypeScript files; verify graph_nodes and graph_edges contain expected counts.
- **hierarchy-paths**: Index a file at `src/services/auth.ts` containing `AuthService.validateToken`; verify the node's `hierarchy_path` is `src.services.auth.AuthService.validateToken`.
- **relationship-resolution**: Create two files, one importing from the other; verify an `imports` edge exists between the correct nodes.
- **re-index-replaces**: Index a repo, then re-index; verify node count stays the same (no duplicates).
- **parse-errors-captured**: Include a file with syntax errors; verify it appears in `errors` array but other files are still processed.

---

#### 2.4 — Graph Degree and Containment Edge Computation

**What**: After initial indexing, compute `contains` edges for file-to-entity and class-to-method containment, and update in_degree/out_degree counters on all nodes.

**Design**:

```typescript
// packages/indexer/src/graph/relationships.ts

export async function computeContainmentEdges(
  db: DrizzleClient,
  repositoryId: string,
): Promise<number> {
  // For each node with a hierarchyPath, find its parent node
  // (hierarchyPath one level up) and create a "contains" edge.
  // Example: "src.services.auth.AuthService.validateToken" parent is
  //          "src.services.auth.AuthService"
  //
  // SQL: INSERT INTO graph_edges (source_id, target_id, edge_type)
  //      SELECT parent.id, child.id, 'contains'
  //      FROM graph_nodes child
  //      JOIN graph_nodes parent ON parent.hierarchy_path = subpath(child.hierarchy_path, 0, nlevel(child.hierarchy_path) - 1)
  //      WHERE child.repository_id = $1 AND parent.repository_id = $1
  //      ON CONFLICT DO NOTHING;
}

export async function updateDegreeCounters(
  db: DrizzleClient,
  repositoryId: string,
): Promise<void> {
  // UPDATE graph_nodes SET
  //   in_degree = (SELECT COUNT(*) FROM graph_edges WHERE target_id = graph_nodes.id),
  //   out_degree = (SELECT COUNT(*) FROM graph_edges WHERE source_id = graph_nodes.id)
  // WHERE repository_id = $1;
}
```

**Testing**:
- **containment-edges-created**: Index a class with 3 methods; run `computeContainmentEdges`; verify 3 `contains` edges from class to methods.
- **degree-counters**: Create a node with 2 incoming and 3 outgoing edges; run `updateDegreeCounters`; verify `in_degree = 2`, `out_degree = 3`.

---

## Phase 3: Embedding Generation and Vector Search

### Purpose

Generate vector embeddings for all code entities and store them in pgvector. Implement the vector search component of the retrieval pipeline, enabling semantic similarity queries over the codebase. This phase provides the "find relevant code" capability that the Q&A system uses in Phase 4.

### Tasks

#### 3.1 — Batch Embedding Pipeline

**What**: Read graph nodes, chunk their text content (signature + docstring + file context), generate embeddings via the configured LLM provider, and store them in the graph_embeddings table.

**Design**:

```typescript
// packages/indexer/src/embeddings/indexer.ts

export interface EmbeddingConfig {
  batchSize: number;         // default 32
  maxChunkTokens: number;    // default 512
  overlapTokens: number;     // default 64
}

export class EmbeddingIndexer {
  constructor(
    private db: DrizzleClient,
    private llmProvider: LLMProvider,
    private config: EmbeddingConfig = { batchSize: 32, maxChunkTokens: 512, overlapTokens: 64 },
  ) {}

  async indexRepository(repositoryId: string, embeddingModel: string): Promise<{
    nodesProcessed: number;
    embeddingsCreated: number;
    durationMs: number;
  }> {
    const startTime = Date.now();

    // Delete existing embeddings for this repo's nodes
    await this.db.execute(sql`
      DELETE FROM graph_embeddings
      WHERE node_id IN (SELECT id FROM graph_nodes WHERE repository_id = ${repositoryId})
    `);

    // Fetch all nodes for this repo
    const nodes = await this.db.select().from(graphNodes).where(eq(graphNodes.repositoryId, repositoryId));

    let embeddingsCreated = 0;

    // Process in batches
    for (let i = 0; i < nodes.length; i += this.config.batchSize) {
      const batch = nodes.slice(i, i + this.config.batchSize);
      const texts = batch.map((node) => this.buildChunkText(node));
      const result = await this.llmProvider.embed({ model: embeddingModel, texts });

      for (let j = 0; j < batch.length; j++) {
        await this.db.insert(graphEmbeddings).values({
          nodeId: batch[j].id,
          embeddingModel,
          embedding: pgVector(result.embeddings[j]),  // convert to pgvector format
          chunkText: texts[j],
          chunkIndex: 0,
        });
        embeddingsCreated++;
      }
    }

    return { nodesProcessed: nodes.length, embeddingsCreated, durationMs: Date.now() - startTime };
  }

  private buildChunkText(node: typeof graphNodes.$inferSelect): string {
    const props = node.properties as CodeEntityProperties;
    const parts: string[] = [
      `${node.label}: ${node.qualifiedName}`,
      props.signature ? `Signature: ${props.signature}` : "",
      props.docstring ? `Documentation: ${props.docstring}` : "",
      `File: ${props.filePath ?? "unknown"}`,
    ];
    return parts.filter(Boolean).join("\n");
  }
}
```

**Testing**:
- **embed-single-node** (integration): Create a graph node, run embedding; verify a row in graph_embeddings with correct nodeId and non-empty embedding vector.
- **embed-batch**: Create 50 nodes; run embedding with batchSize=10; verify 50 embedding rows created and 5 API calls made to the LLM provider.
- **chunk-text-format**: Call `buildChunkText` on a class node with signature and docstring; verify output contains label, qualifiedName, signature, and documentation.
- **re-embed-replaces**: Embed a repo, then re-embed; verify embedding count stays the same.

---

#### 3.2 — Vector Search Query

**What**: Implement a vector search function that takes a natural-language query, embeds it, and returns the top-k most similar graph nodes using pgvector's HNSW index.

**Design**:

```typescript
// packages/api/src/services/retrieval.ts

export interface VectorSearchResult {
  nodeId: string;
  name: string;
  label: GraphNodeLabel;
  qualifiedName: string;
  properties: Record<string, unknown>;
  similarity: number;          // cosine similarity (0-1)
  chunkText: string;
}

export async function vectorSearch(
  db: DrizzleClient,
  llmProvider: LLMProvider,
  repositoryId: string,
  query: string,
  embeddingModel: string,
  topK: number = 10,
): Promise<VectorSearchResult[]> {
  // 1. Embed the query
  const { embeddings } = await llmProvider.embed({ model: embeddingModel, texts: [query] });
  const queryVector = embeddings[0];

  // 2. Query pgvector with cosine similarity
  const results = await db.execute(sql`
    SELECT
      gn.id AS node_id,
      gn.name,
      gn.label,
      gn.qualified_name,
      gn.properties,
      ge.chunk_text,
      1 - (ge.embedding <=> ${pgVector(queryVector)}::vector) AS similarity
    FROM graph_embeddings ge
    JOIN graph_nodes gn ON gn.id = ge.node_id
    WHERE gn.repository_id = ${repositoryId}
    ORDER BY ge.embedding <=> ${pgVector(queryVector)}::vector
    LIMIT ${topK}
  `);

  return results.rows.map((row) => ({
    nodeId: row.node_id,
    name: row.name,
    label: row.label as GraphNodeLabel,
    qualifiedName: row.qualified_name,
    properties: row.properties,
    similarity: row.similarity,
    chunkText: row.chunk_text,
  }));
}
```

**Testing**:
- **search-returns-results** (integration): Index a test repo with embeddings; search for "authentication"; verify results contain `AuthService` or auth-related entities.
- **search-top-k**: Search with topK=3; verify exactly 3 results returned.
- **search-similarity-ordered**: Verify results are ordered by descending similarity.
- **search-empty-repo**: Search against a repo with no embeddings; verify empty array returned.

---

## Phase 4: Q&A Engine — GraphRAG Retrieval and Conversational Interface

### Purpose

Build the question-answering engine that combines vector search (Phase 3) with graph traversal to implement GraphRAG — the core value proposition of the product. Implement the API endpoint for asking questions about a codebase and receiving architecture-aware answers with source references.

### Tasks

#### 4.1 — GraphRAG Retrieval Pipeline

**What**: Implement the two-phase GraphRAG retrieval: (1) vector search finds semantically similar entities, (2) graph traversal expands the neighborhood of those entities to gather structural context.

**Design**:

```typescript
// packages/api/src/services/retrieval.ts

export interface RetrievalContext {
  vectorHits: VectorSearchResult[];
  graphNeighbors: Array<{
    nodeId: string;
    name: string;
    label: GraphNodeLabel;
    qualifiedName: string;
    properties: Record<string, unknown>;
    relationship: string;       // how it relates to a vector hit
    hops: number;               // distance from nearest vector hit
  }>;
  totalContextTokens: number;
}

export async function graphRAGRetrieve(
  db: DrizzleClient,
  llmProvider: LLMProvider,
  repositoryId: string,
  query: string,
  embeddingModel: string,
  options: {
    vectorTopK?: number;        // default 10
    graphDepth?: number;        // default 2
    maxContextNodes?: number;   // default 30
  } = {},
): Promise<RetrievalContext> {
  const { vectorTopK = 10, graphDepth = 2, maxContextNodes = 30 } = options;

  // Phase 1: Vector search
  const vectorHits = await vectorSearch(db, llmProvider, repositoryId, query, embeddingModel, vectorTopK);
  const hitNodeIds = vectorHits.map((h) => h.nodeId);

  // Phase 2: Graph expansion — BFS from vector hits
  const graphNeighbors = await db.execute(sql`
    WITH RECURSIVE expansion AS (
      -- Seed: direct neighbors of vector hits
      SELECT
        CASE WHEN e.source_id = ANY(${hitNodeIds}::uuid[]) THEN e.target_id ELSE e.source_id END AS node_id,
        e.edge_type AS relationship,
        1 AS hops,
        ARRAY[CASE WHEN e.source_id = ANY(${hitNodeIds}::uuid[]) THEN e.source_id ELSE e.target_id END] AS visited
      FROM graph_edges e
      WHERE e.source_id = ANY(${hitNodeIds}::uuid[])
         OR e.target_id = ANY(${hitNodeIds}::uuid[])

      UNION ALL

      -- Expand one more hop
      SELECT
        CASE WHEN e.source_id = exp.node_id THEN e.target_id ELSE e.source_id END,
        e.edge_type,
        exp.hops + 1,
        exp.visited || exp.node_id
      FROM expansion exp
      JOIN graph_edges e ON e.source_id = exp.node_id OR e.target_id = exp.node_id
      WHERE exp.hops < ${graphDepth}
        AND NOT (CASE WHEN e.source_id = exp.node_id THEN e.target_id ELSE e.source_id END) = ANY(exp.visited)
    )
    SELECT DISTINCT ON (n.id)
      n.id AS node_id, n.name, n.label, n.qualified_name, n.properties,
      exp.relationship, exp.hops
    FROM expansion exp
    JOIN graph_nodes n ON n.id = exp.node_id
    WHERE n.id != ALL(${hitNodeIds}::uuid[])
    ORDER BY n.id, exp.hops
    LIMIT ${maxContextNodes - vectorTopK}
  `);

  return {
    vectorHits,
    graphNeighbors: graphNeighbors.rows,
    totalContextTokens: estimateTokens(vectorHits, graphNeighbors.rows),
  };
}
```

**Testing**:
- **graphrag-expands-neighbors** (integration): Create a chain A->B->C. Vector search returns A. Verify B appears in graphNeighbors at hops=1 and C at hops=2.
- **graphrag-deduplicates**: Create a diamond graph A->B, A->C, B->D, C->D. Verify D appears only once in results.
- **graphrag-respects-depth**: Set graphDepth=1; verify only direct neighbors returned, not 2-hop.
- **graphrag-respects-max-nodes**: Set maxContextNodes=5; verify at most 5 total context items.

---

#### 4.2 — Q&A Orchestration Service

**What**: Implement the Q&A service that takes a user question, retrieves context via GraphRAG, constructs a prompt with retrieved code context, sends it to the LLM, and returns the answer with source references.

**Design**:

```typescript
// packages/api/src/services/qa.ts

export interface AskResult {
  answer: string;
  model: string;
  tokensUsed: { prompt: number; completion: number; total: number };
  contextEntities: Array<{
    id: string;
    name: string;
    qualifiedName: string;
    method: "vector_search" | "graph_traversal";
    relevanceScore?: number;
  }>;
}

export class QAService {
  constructor(
    private db: DrizzleClient,
    private llmProvider: LLMProvider,
    private config: { chatModel: string; embeddingModel: string },
  ) {}

  async ask(
    userId: string,
    repositoryId: string,
    sessionId: string | null,
    question: string,
  ): Promise<{ sessionId: string; result: AskResult }> {
    // 1. Retrieve context via GraphRAG
    const context = await graphRAGRetrieve(
      this.db, this.llmProvider, repositoryId, question, this.config.embeddingModel,
    );

    // 2. Build system prompt with retrieved context
    const systemPrompt = this.buildSystemPrompt(context);

    // 3. Load or create session
    let session = sessionId
      ? await this.db.select().from(qaSessions).where(eq(qaSessions.id, sessionId)).then(r => r[0])
      : null;
    if (!session) {
      [session] = await this.db.insert(qaSessions).values({
        userId, repositoryId, title: question.slice(0, 100),
        messages: [], messageCount: 0, totalTokens: 0,
      }).returning();
    }

    // 4. Build message history
    const existingMessages = (session.messages as QAMessage[]).map(m => ({
      role: m.role as "user" | "assistant",
      content: m.content,
    }));

    const messages: LLMMessage[] = [
      { role: "system", content: systemPrompt },
      ...existingMessages,
      { role: "user", content: question },
    ];

    // 5. Call LLM
    const completion = await this.llmProvider.complete({
      model: this.config.chatModel,
      messages,
      temperature: 0.1,
      maxTokens: 4096,
    });

    // 6. Build context entities list
    const contextEntities = [
      ...context.vectorHits.map(h => ({
        id: h.nodeId, name: h.name, qualifiedName: h.qualifiedName,
        method: "vector_search" as const, relevanceScore: h.similarity,
      })),
      ...context.graphNeighbors.map(n => ({
        id: n.nodeId, name: n.name, qualifiedName: n.qualifiedName,
        method: "graph_traversal" as const,
      })),
    ];

    // 7. Persist messages to session
    const userMsg: QAMessage = { role: "user", content: question, timestamp: new Date().toISOString() };
    const assistantMsg: QAMessage = {
      role: "assistant", content: completion.content, timestamp: new Date().toISOString(),
      model: completion.model, tokensUsed: completion.tokensUsed.total,
      contextEntities: contextEntities.slice(0, 10),
    };
    const updatedMessages = [...(session.messages as QAMessage[]), userMsg, assistantMsg];
    await this.db.update(qaSessions).set({
      messages: updatedMessages,
      messageCount: updatedMessages.length,
      totalTokens: (session.totalTokens ?? 0) + completion.tokensUsed.total,
      updatedAt: new Date(),
    }).where(eq(qaSessions.id, session.id));

    return {
      sessionId: session.id,
      result: { answer: completion.content, model: completion.model, tokensUsed: completion.tokensUsed, contextEntities },
    };
  }

  private buildSystemPrompt(context: RetrievalContext): string {
    const entityDescriptions = [
      ...context.vectorHits.map(h =>
        `[${h.label}] ${h.qualifiedName}\n${h.chunkText}`
      ),
      ...context.graphNeighbors.map(n =>
        `[${n.label}] ${n.qualifiedName} (${n.relationship}, ${n.hops} hop${n.hops > 1 ? "s" : ""} away)`
      ),
    ].join("\n\n");

    return `You are an expert software architect answering questions about a codebase.

Use the following code context to answer the question. Reference specific files, classes, and functions by name.
If the context does not contain enough information, say so clearly.

## Code Context

${entityDescriptions}

## Instructions
- Answer in clear, technical prose
- Reference specific code entities by their qualified name
- Explain architectural relationships when relevant
- If the question requires information not in the context, acknowledge the limitation`;
  }
}
```

**Testing**:
- **ask-creates-session** (integration): Ask a question with no sessionId; verify a new session is created in the database.
- **ask-continues-session**: Ask two questions with the same sessionId; verify the session's messages array contains 4 entries (2 user + 2 assistant).
- **ask-returns-context-entities**: Ask a question; verify `contextEntities` array is non-empty and contains ids matching graph nodes.
- **ask-stores-tokens**: Ask a question; verify session's `totalTokens` is updated.

---

#### 4.3 — Q&A API Endpoints

**What**: Expose REST API endpoints for creating Q&A sessions and asking questions.

**Design**:

```typescript
// packages/api/src/routes/qa.ts

// POST /api/v1/repositories/:repoId/qa
// Body: { question: string, sessionId?: string }
// Response: { sessionId: string, answer: string, model: string, contextEntities: [...] }

// GET /api/v1/qa/sessions/:sessionId
// Response: { id, repositoryId, messages: [...], messageCount, totalTokens, createdAt }

// GET /api/v1/repositories/:repoId/qa/sessions
// Query: ?limit=20&offset=0
// Response: { sessions: [...], total: number }

export async function qaRoutes(app: FastifyInstance): Promise<void> {
  app.post<{
    Params: { repoId: string };
    Body: { question: string; sessionId?: string };
  }>("/api/v1/repositories/:repoId/qa", {
    schema: {
      params: { type: "object", properties: { repoId: { type: "string", format: "uuid" } }, required: ["repoId"] },
      body: {
        type: "object",
        properties: {
          question: { type: "string", minLength: 1, maxLength: 10000 },
          sessionId: { type: "string", format: "uuid" },
        },
        required: ["question"],
      },
    },
  }, async (request, reply) => {
    const { repoId } = request.params;
    const { question, sessionId } = request.body;
    const userId = request.user.id;  // from auth middleware

    const qaService = new QAService(db, llmProvider, llmConfig);
    const { sessionId: sid, result } = await qaService.ask(userId, repoId, sessionId ?? null, question);

    return reply.status(200).send({
      sessionId: sid,
      answer: result.answer,
      model: result.model,
      contextEntities: result.contextEntities,
    });
  });
}
```

**Testing**:
- **api-ask-200** (e2e): POST a question; verify 200 response with `sessionId`, `answer`, and `contextEntities`.
- **api-ask-400-no-question**: POST with empty body; verify 400 validation error.
- **api-ask-404-bad-repo**: POST to a non-existent repoId; verify 404.
- **api-get-session**: Create a session, then GET it; verify messages match.
- **api-list-sessions**: Create 3 sessions; GET list with limit=2; verify 2 sessions returned with `total: 3`.

---

## Phase 5: CLI Tool and Basic Repository Management API

### Purpose

Build the CLI tool (`doa`) for indexing repositories and asking questions from the terminal, and the repository management API endpoints for adding, listing, and triggering indexing of repositories. This phase makes the product usable as a self-hosted command-line tool.

### Tasks

#### 5.1 — CLI Framework and Configuration

**What**: Implement the `doa` CLI tool with subcommands for config, index, ask, and graph inspection.

**Design**:

```typescript
// packages/cli/src/index.ts

import { Command } from "commander";

const program = new Command()
  .name("doa")
  .description("Developer Onboarding Assistant — AI-powered codebase understanding")
  .version("0.1.0");

// doa config set llm.provider ollama
// doa config set llm.chatModel llama3
// doa config get llm.provider
program.command("config")
  .description("Manage configuration")
  .command("set <key> <value>")
  .action(async (key: string, value: string) => { /* write to ~/.doa/config.json */ });

// doa index <path-or-url> [--branch main]
program.command("index <pathOrUrl>")
  .description("Index a repository")
  .option("--branch <branch>", "Branch to index", "main")
  .action(async (pathOrUrl: string, options: { branch: string }) => {
    // Clone if URL, otherwise use local path.
    // Run GraphBuilder.indexRepository + EmbeddingIndexer.indexRepository.
    // Print progress and results.
  });

// doa ask "why does the payment service call inventory?"
program.command("ask <question>")
  .description("Ask a question about the indexed codebase")
  .option("--repo <repoId>", "Repository UUID")
  .action(async (question: string, options: { repo?: string }) => {
    // Call QAService.ask, print answer with source references.
  });

// doa graph <entity-name> [--depth 2]
program.command("graph <entityName>")
  .description("Show dependency graph for an entity")
  .option("--depth <n>", "Traversal depth", "2")
  .action(async (entityName: string, options: { depth: string }) => {
    // Run recursive CTE, print as ASCII tree.
  });
```

**Testing**:
- **cli-config-set-get** (integration): Run `doa config set llm.provider ollama`, then `doa config get llm.provider`; verify output is `ollama`.
- **cli-index-local** (integration): Run `doa index ./test-fixtures/sample-repo`; verify output shows nodes and edges created.
- **cli-ask** (integration): After indexing, run `doa ask "what is AuthService?"`; verify answer printed to stdout.
- **cli-graph** (integration): After indexing, run `doa graph AuthService --depth 1`; verify ASCII tree output.
- **cli-help**: Run `doa --help`; verify all subcommands listed.

---

#### 5.2 — Repository Management API

**What**: Implement API endpoints for adding repositories, listing them, triggering indexing, and viewing index status.

**Design**:

```typescript
// packages/api/src/routes/repositories.ts

// POST /api/v1/repositories
// Body: { cloneUrl: string, name?: string, branch?: string }
// Response: 201 { id, name, fullName, cloneUrl, status: "pending" }

// GET /api/v1/repositories
// Response: { repositories: [...], total: number }

// POST /api/v1/repositories/:repoId/index
// Response: 202 { jobId: string, status: "queued" }

// GET /api/v1/repositories/:repoId
// Response: { id, name, fullName, metadata: { index_stats: { entity_count, relationship_count, ... } } }
```

**Testing**:
- **api-add-repo**: POST a clone URL; verify 201 with repository record.
- **api-list-repos**: Add 3 repos; GET list; verify 3 returned.
- **api-trigger-index**: POST to trigger indexing; verify 202 with jobId.
- **api-repo-details**: GET a repo after indexing; verify metadata contains index_stats.

---

#### 5.3 — Background Job Queue (BullMQ)

**What**: Implement the worker process using BullMQ and Redis for asynchronous repository indexing and embedding generation.

**Design**:

```typescript
// packages/worker/src/queue.ts

import { Queue, Worker, Job } from "bullmq";

export const indexQueue = new Queue("repository-index", { connection: redisConfig });

export interface IndexJobData {
  repositoryId: string;
  cloneUrl: string;
  branch: string;
  triggeredBy: string;
}

// packages/worker/src/jobs/index-repository.ts

export async function processIndexJob(job: Job<IndexJobData>): Promise<void> {
  const { repositoryId, cloneUrl, branch } = job.data;

  await job.updateProgress(10);
  // 1. Clone/fetch repository
  const cloneResult = await cloneOrFetch(cloneUrl, REPOS_DIR, branch);

  await job.updateProgress(30);
  // 2. Build code graph
  const graphResult = await graphBuilder.indexRepository(repositoryId, cloneResult.localPath, cloneResult.commitSha);

  await job.updateProgress(60);
  // 3. Compute containment edges and degree counters
  await computeContainmentEdges(db, repositoryId);
  await updateDegreeCounters(db, repositoryId);

  await job.updateProgress(80);
  // 4. Generate embeddings
  const embedResult = await embeddingIndexer.indexRepository(repositoryId, embeddingModel);

  await job.updateProgress(100);
  // 5. Update repository metadata
  await db.update(repositories).set({
    lastIndexedAt: new Date(),
    metadata: {
      index_stats: {
        last_indexed_at: new Date().toISOString(),
        last_commit_sha: cloneResult.commitSha,
        entity_count: graphResult.nodesCreated,
        relationship_count: graphResult.edgesCreated,
        embedding_count: embedResult.embeddingsCreated,
      },
    },
    updatedAt: new Date(),
  }).where(eq(repositories.id, repositoryId));
}
```

**Testing**:
- **job-queued**: Add a job to the queue; verify it appears in Redis.
- **job-processes** (integration): Add a job and start a worker; verify repository is indexed and metadata updated.
- **job-progress**: Monitor job progress events; verify they go from 10 to 100.
- **job-failure**: Submit a job with an invalid clone URL; verify the job enters failed state with error message.

---

## Phase 6: VS Code Extension — In-Editor Q&A

### Purpose

Build the VS Code extension that provides an in-editor Q&A sidebar, code graph navigation, and context-aware questions. This phase delivers the primary user interface for the product — developers can ask questions about their codebase without leaving their editor.

### Tasks

#### 6.1 — Extension Scaffold and Activation

**What**: Create the VS Code extension package with activation events, sidebar registration, and API client connection.

**Design**:

```typescript
// packages/vscode/src/extension.ts

import * as vscode from "vscode";
import { QAPanelProvider } from "./sidebar/qa-panel";

export function activate(context: vscode.ExtensionContext): void {
  const config = vscode.workspace.getConfiguration("doa");
  const apiBaseUrl = config.get<string>("apiUrl", "http://localhost:3000");

  // Register sidebar webview provider
  const qaProvider = new QAPanelProvider(context.extensionUri, apiBaseUrl);
  context.subscriptions.push(
    vscode.window.registerWebviewViewProvider("doa.qaPanel", qaProvider),
  );

  // Register commands
  context.subscriptions.push(
    vscode.commands.registerCommand("doa.ask", () => qaProvider.focusAndAsk()),
    vscode.commands.registerCommand("doa.askAboutSelection", () => {
      const editor = vscode.window.activeTextEditor;
      if (!editor) return;
      const selection = editor.document.getText(editor.selection);
      qaProvider.askAbout(selection, editor.document.fileName);
    }),
    vscode.commands.registerCommand("doa.indexWorkspace", async () => {
      // Trigger indexing of the current workspace folder
    }),
  );
}
```

**Testing**:
- **extension-activates** (e2e): Open VS Code with the extension; verify activation without errors.
- **sidebar-renders**: Open the DOA sidebar panel; verify the Q&A webview loads.
- **command-registered**: Verify `doa.ask`, `doa.askAboutSelection`, and `doa.indexWorkspace` commands are available in the command palette.

---

#### 6.2 — Q&A Webview Panel

**What**: Implement the sidebar webview with a chat interface for asking questions and displaying answers with code references.

**Design**:

```typescript
// packages/vscode/src/sidebar/qa-panel.ts

export class QAPanelProvider implements vscode.WebviewViewProvider {
  private _view?: vscode.WebviewView;

  constructor(
    private extensionUri: vscode.Uri,
    private apiBaseUrl: string,
  ) {}

  resolveWebviewView(webviewView: vscode.WebviewView): void {
    this._view = webviewView;
    webviewView.webview.options = { enableScripts: true };
    webviewView.webview.html = this.getHtml(webviewView.webview);

    webviewView.webview.onDidReceiveMessage(async (message) => {
      switch (message.type) {
        case "ask":
          await this.handleQuestion(message.question, message.sessionId);
          break;
        case "openFile":
          await this.openFileAtLine(message.filePath, message.line);
          break;
      }
    });
  }

  private async handleQuestion(question: string, sessionId?: string): Promise<void> {
    this._view?.webview.postMessage({ type: "thinking" });

    const response = await fetch(`${this.apiBaseUrl}/api/v1/repositories/${repoId}/qa`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ question, sessionId }),
    });
    const data = await response.json();

    this._view?.webview.postMessage({
      type: "answer",
      answer: data.answer,
      contextEntities: data.contextEntities,
      sessionId: data.sessionId,
    });
  }

  private async openFileAtLine(filePath: string, line: number): Promise<void> {
    const uri = vscode.Uri.file(path.join(vscode.workspace.rootPath!, filePath));
    const doc = await vscode.workspace.openTextDocument(uri);
    const editor = await vscode.window.showTextDocument(doc);
    const range = new vscode.Range(line - 1, 0, line - 1, 0);
    editor.revealRange(range, vscode.TextEditorRevealType.InCenter);
    editor.selection = new vscode.Selection(range.start, range.start);
  }
}
```

**Testing**:
- **webview-sends-question** (e2e): Type a question in the sidebar and submit; verify API call is made.
- **webview-displays-answer** (e2e): Mock the API response; verify the answer renders in the webview with markdown formatting.
- **webview-open-file**: Click a code reference in an answer; verify the file opens at the correct line.
- **webview-loading-state**: Submit a question; verify a loading indicator appears until the answer arrives.

---

#### 6.3 — Context-Aware Ask About Selection

**What**: Implement the "Ask about selection" command that uses the currently selected code as context for the question.

**Design**:

```typescript
// When the user selects code and runs doa.askAboutSelection:
// 1. Get the selected text and file path
// 2. Pre-fill the question with "Explain the following code from <file>:<line>"
// 3. Include the selected code as additional context in the API request

// Extension of the Q&A API:
// POST /api/v1/repositories/:repoId/qa
// Body: {
//   question: string,
//   sessionId?: string,
//   codeContext?: { filePath: string, startLine: number, endLine: number, code: string }
// }
```

**Testing**:
- **selection-populates-context**: Select code in editor, run command; verify the webview opens with pre-filled question including the selected code.
- **selection-empty**: Run command with no selection; verify a prompt asking user to select code first.

---

## Phase 7: Git History Analysis and Tribal Knowledge Extraction

### Purpose

Analyze git commit history, PR descriptions, and code comments to extract implicit tribal knowledge — the "why" behind code decisions. Generate structured knowledge articles from this analysis and link them to the code graph. This is the key differentiator: no existing open-source tool performs this extraction at the codebase-graph level.

### Tasks

#### 7.1 — Git History Ingestion

**What**: Parse git log and PR metadata from GitHub/GitLab APIs, store commit records with Conventional Commits analysis, and identify high-signal commits and PRs.

**Design**:

```typescript
// packages/indexer/src/git/history.ts

export interface CommitRecord {
  sha: string;
  authorName: string;
  authorEmail: string;
  message: string;
  committedAt: Date;
  analysis: {
    conventional?: { type: string; scope?: string; breaking: boolean };
    filesChanged: number;
    insertions: number;
    deletions: number;
    entitiesAffected: string[];    // qualified names of changed code entities
    architecturalImpact: "none" | "minor" | "major";
  };
}

export async function ingestGitHistory(
  repoDir: string,
  repositoryId: string,
  db: DrizzleClient,
  options: { maxCommits?: number; since?: Date } = {},
): Promise<{ commitsIngested: number; prsIngested: number }> {
  const git = simpleGit(repoDir);
  const log = await git.log({ maxCount: options.maxCommits ?? 5000, "--since": options.since?.toISOString() });

  for (const entry of log.all) {
    const conventional = parseConventionalCommit(entry.message);
    const diffStat = await git.diffSummary([`${entry.hash}~1`, entry.hash]).catch(() => null);

    await db.insert(commits).values({
      repositoryId,
      sha: entry.hash,
      authorName: entry.author_name,
      authorEmail: entry.author_email,
      message: entry.message,
      committedAt: new Date(entry.date),
      analysis: {
        conventional,
        filesChanged: diffStat?.changed ?? 0,
        insertions: diffStat?.insertions ?? 0,
        deletions: diffStat?.deletions ?? 0,
        entitiesAffected: [],  // populated later by cross-referencing with graph nodes
        architecturalImpact: classifyImpact(diffStat),
      },
    }).onConflictDoNothing();
  }
}

function parseConventionalCommit(message: string): { type: string; scope?: string; breaking: boolean } | undefined {
  const match = message.match(/^(\w+)(?:\(([^)]+)\))?(!?):\s/);
  if (!match) return undefined;
  return { type: match[1], scope: match[2], breaking: match[3] === "!" };
}
```

**Testing**:
- **parse-conventional-feat**: Parse `"feat(auth): add JWT validation"`; verify `{ type: "feat", scope: "auth", breaking: false }`.
- **parse-conventional-breaking**: Parse `"refactor(api)!: change response format"`; verify `breaking: true`.
- **parse-non-conventional**: Parse `"Fix the login bug"`; verify returns `undefined`.
- **ingest-commits** (integration): Ingest history from the test fixture repo; verify commits table populated with correct fields.

---

#### 7.2 — Tribal Knowledge Extraction via LLM

**What**: Identify high-signal commits and PRs (large refactors, architectural changes, important bug fixes), feed them to an LLM with surrounding code context, and generate structured knowledge articles.

**Design**:

```typescript
// packages/worker/src/jobs/extract-knowledge.ts

export interface ExtractionCandidate {
  type: "commit_cluster" | "pull_request" | "code_comment_thread";
  commits: CommitRecord[];
  relatedEntities: GraphNode[];
  context: string;               // combined text from commits, PR body, comments
}

export async function extractTribalKnowledge(
  db: DrizzleClient,
  llmProvider: LLMProvider,
  repositoryId: string,
  config: { chatModel: string; minConfidence: number },
): Promise<{ articlesCreated: number }> {
  // 1. Find extraction candidates:
  //    - Commits with architecturalImpact = "major"
  //    - Clusters of commits touching the same entities
  //    - Commits with long messages (>200 chars) explaining decisions
  const candidates = await findExtractionCandidates(db, repositoryId);

  let articlesCreated = 0;

  for (const candidate of candidates) {
    // 2. Build extraction prompt
    const prompt = buildExtractionPrompt(candidate);

    // 3. Call LLM to generate a knowledge article
    const result = await llmProvider.complete({
      model: config.chatModel,
      messages: [
        { role: "system", content: KNOWLEDGE_EXTRACTION_SYSTEM_PROMPT },
        { role: "user", content: prompt },
      ],
      temperature: 0.2,
    });

    // 4. Parse structured output
    const extracted = parseExtractionResult(result.content);
    if (!extracted || extracted.confidenceScore < config.minConfidence) continue;

    // 5. Store as knowledge article
    await db.insert(knowledgeArticles).values({
      repositoryId,
      title: extracted.title,
      content: extracted.content,
      articleType: extracted.articleType,
      sourceType: "extracted",
      status: "draft",
      confidenceScore: extracted.confidenceScore,
      metadata: {
        sources: candidate.commits.map(c => ({
          type: "commit", ref: c.sha, author: c.authorName, date: c.committedAt.toISOString(),
          excerpt: c.message.slice(0, 200),
        })),
        relatedEntityIds: candidate.relatedEntities.map(e => e.id),
        tags: extracted.tags,
      },
    });
    articlesCreated++;
  }

  return { articlesCreated };
}

const KNOWLEDGE_EXTRACTION_SYSTEM_PROMPT = `You are an expert at extracting architectural knowledge from code change history.
Given a set of commits and the code entities they modified, identify the implicit architectural decision or pattern being expressed.

Output a JSON object with:
- title: A clear, descriptive title for the knowledge article (e.g., "Why AuthService validates tokens before checking permissions")
- content: A 2-4 paragraph markdown explanation of the architectural decision, pattern, or workaround
- articleType: One of "architectural_decision", "pattern_explanation", "workaround", "naming_convention", "deployment_note"
- confidenceScore: 0.0-1.0 confidence that this represents genuine architectural knowledge (not routine changes)
- tags: Array of 2-5 relevant tags

Only extract knowledge when the commits reveal a genuine architectural decision, non-obvious pattern, or important context that would help a new developer understand the codebase. Do NOT extract knowledge from routine bug fixes or minor changes.`;
```

**Testing**:
- **find-candidates-major**: Insert commits with architecturalImpact="major"; verify they appear as extraction candidates.
- **find-candidates-clusters**: Insert 5 commits touching the same entity; verify they form a cluster candidate.
- **extraction-produces-article** (integration): Run extraction on a candidate with a clear architectural decision; verify a knowledge article is created with type "architectural_decision".
- **extraction-rejects-low-confidence**: Mock LLM response with confidenceScore=0.2 and minConfidence=0.5; verify no article created.
- **extraction-links-sources**: Verify created article's metadata.sources contains the commit SHAs.

---

#### 7.3 — Code Comment Analysis

**What**: Extract inline code comments that contain architectural rationale (TODO, HACK, IMPORTANT, NOTE, FIXME patterns) and generate knowledge articles from significant ones.

**Design**:

```typescript
// packages/indexer/src/git/comments.ts

export interface SignificantComment {
  filePath: string;
  line: number;
  text: string;
  tag: "TODO" | "HACK" | "IMPORTANT" | "NOTE" | "FIXME" | "WORKAROUND" | null;
  nearestEntityQualifiedName: string;
}

export async function extractSignificantComments(
  repoDir: string,
  repositoryId: string,
  db: DrizzleClient,
): Promise<SignificantComment[]> {
  // Scan all indexed files for comment patterns.
  // Filter to comments > 50 characters (likely to contain rationale).
  // Cross-reference with graph nodes to find the nearest containing entity.
}
```

**Testing**:
- **extract-todo**: File contains `// TODO(phil): refactor this once the billing API stabilizes`; verify extracted with tag "TODO".
- **extract-important**: File contains `// IMPORTANT: order matters here — validate before authorize`; verify extracted with tag "IMPORTANT".
- **skip-short-comments**: File contains `// increment counter`; verify NOT extracted (< 50 chars).
- **link-to-entity**: Comment is inside a function body; verify `nearestEntityQualifiedName` matches the function.

---

## Phase 8: Architecture Diagram Generation

### Purpose

Generate Mermaid architecture diagrams from the code graph — service dependency diagrams, class hierarchies, module dependency graphs, and call flow diagrams. Diagrams are stored in the database and can be regenerated when the codebase changes.

### Tasks

#### 8.1 — Diagram Generation Service

**What**: Implement a service that queries the code graph and produces Mermaid diagram source code for different diagram types.

**Design**:

```typescript
// packages/api/src/services/diagrams.ts

export type DiagramType = "service_dependency" | "class_hierarchy" | "module_dependency" | "call_flow";

export interface DiagramResult {
  title: string;
  diagramType: DiagramType;
  mermaidSource: string;
  nodeCount: number;
  edgeCount: number;
}

export class DiagramService {
  constructor(private db: DrizzleClient) {}

  async generateServiceDependencyDiagram(repositoryId: string): Promise<DiagramResult> {
    // Query graph_nodes with label IN ('service', 'module', 'package')
    // Query graph_edges with edge_type = 'depends_on' between those nodes
    // Generate: graph LR\n  ServiceA --> ServiceB\n  ServiceA --> ServiceC
    const nodes = await this.db.select().from(graphNodes)
      .where(and(eq(graphNodes.repositoryId, repositoryId), inArray(graphNodes.label, ["service", "module"])));
    const edges = await this.db.select().from(graphEdges)
      .where(and(eq(graphEdges.edgeType, "depends_on"),
        inArray(graphEdges.sourceId, nodes.map(n => n.id))));

    const lines = ["graph LR"];
    for (const edge of edges) {
      const source = nodes.find(n => n.id === edge.sourceId);
      const target = nodes.find(n => n.id === edge.targetId);
      if (source && target) {
        lines.push(`  ${sanitizeMermaidId(source.name)}["${source.name}"] --> ${sanitizeMermaidId(target.name)}["${target.name}"]`);
      }
    }
    return {
      title: "Service Dependency Diagram",
      diagramType: "service_dependency",
      mermaidSource: lines.join("\n"),
      nodeCount: nodes.length,
      edgeCount: edges.length,
    };
  }

  async generateClassHierarchyDiagram(repositoryId: string, rootClassName?: string): Promise<DiagramResult> {
    // Query class and interface nodes, extends/implements edges.
    // Generate: classDiagram\n  IAuthProvider <|-- AuthService\n  AuthService : +validateToken()
  }

  async generateCallFlowDiagram(repositoryId: string, entryPoint: string, maxDepth: number = 3): Promise<DiagramResult> {
    // Run recursive CTE from entry point, following 'calls' edges.
    // Generate: sequenceDiagram\n  AuthService->>UserRepo: findByEmail()\n  UserRepo->>DB: query()
  }
}
```

**Testing**:
- **service-dep-diagram**: Create 3 service nodes with depends_on edges; generate diagram; verify valid Mermaid syntax with all 3 nodes.
- **class-hierarchy**: Create class A extends B implements C; generate diagram; verify `classDiagram` output with inheritance arrows.
- **call-flow-diagram**: Create call chain A->B->C; generate from A with depth 2; verify sequence diagram contains both calls.
- **empty-diagram**: Generate for a repo with no services; verify empty but valid Mermaid output.

---

#### 8.2 — Diagram API Endpoints

**What**: Expose API endpoints for generating and retrieving architecture diagrams.

**Design**:

```typescript
// POST /api/v1/repositories/:repoId/diagrams/generate
// Body: { type: DiagramType, rootEntity?: string, depth?: number }
// Response: 200 { title, diagramType, mermaidSource, nodeCount, edgeCount }

// GET /api/v1/repositories/:repoId/diagrams
// Response: { diagrams: DiagramResult[] }
```

**Testing**:
- **api-generate-diagram**: POST to generate a service_dependency diagram; verify 200 with valid Mermaid source.
- **api-generate-call-flow**: POST with type=call_flow and rootEntity; verify sequence diagram generated.

---

## Phase 9: Documentation Staleness Detection and Living Docs

### Purpose

Implement the documentation anchoring system that links knowledge articles to specific code lines. When anchored code changes, the system detects staleness and can automatically open PRs with updated documentation. This addresses the "documentation rot" problem that every documentation tool fails to solve.

### Tasks

#### 9.1 — Documentation Anchor Management

**What**: Create and verify documentation anchors that link knowledge articles to specific code snippets, detecting when the anchored code drifts.

**Design**:

```typescript
// packages/api/src/services/staleness.ts

import { createHash } from "crypto";

export function computeAnchorHash(codeSnippet: string): string {
  return createHash("sha256").update(codeSnippet.trim()).digest("hex").slice(0, 16);
}

export async function checkStaleness(
  db: DrizzleClient,
  repositoryId: string,
  repoDir: string,
): Promise<{
  articlesChecked: number;
  anchorsChecked: number;
  staleAnchorsFound: number;
  staleArticleIds: string[];
}> {
  const articles = await db.select().from(knowledgeArticles)
    .where(eq(knowledgeArticles.repositoryId, repositoryId));

  let anchorsChecked = 0;
  let staleAnchorsFound = 0;
  const staleArticleIds: string[] = [];

  for (const article of articles) {
    const metadata = article.metadata as KnowledgeArticleMetadata;
    if (!metadata.anchors || metadata.anchors.length === 0) continue;

    let articleHasStaleAnchor = false;
    const updatedAnchors = [...metadata.anchors];

    for (let i = 0; i < updatedAnchors.length; i++) {
      const anchor = updatedAnchors[i];
      anchorsChecked++;

      const filePath = path.join(repoDir, anchor.filePath);
      if (!await fs.pathExists(filePath)) {
        updatedAnchors[i] = { ...anchor, isStale: true };
        staleAnchorsFound++;
        articleHasStaleAnchor = true;
        continue;
      }

      const lines = (await fs.readFile(filePath, "utf-8")).split("\n");
      const currentLine = lines[anchor.line - 1]?.trim() ?? "";
      const currentHash = computeAnchorHash(currentLine);

      if (currentHash !== anchor.hash) {
        updatedAnchors[i] = { ...anchor, isStale: true };
        staleAnchorsFound++;
        articleHasStaleAnchor = true;
      }
    }

    if (articleHasStaleAnchor) {
      staleArticleIds.push(article.id);
      await db.update(knowledgeArticles).set({
        metadata: { ...metadata, anchors: updatedAnchors },
        updatedAt: new Date(),
      }).where(eq(knowledgeArticles.id, article.id));
    }
  }

  return { articlesChecked: articles.length, anchorsChecked, staleAnchorsFound, staleArticleIds };
}
```

**Testing**:
- **anchor-hash-consistent**: Compute hash of the same string twice; verify identical results.
- **anchor-hash-differs**: Compute hash of "function a()" and "function b()"; verify different hashes.
- **detect-stale-anchor**: Create an anchor for line 10 of a file, then modify that line; run staleness check; verify anchor marked stale.
- **detect-fresh-anchor**: Create an anchor, do not modify the file; run staleness check; verify anchor NOT marked stale.
- **detect-missing-file**: Create an anchor for a file that does not exist; verify anchor marked stale.

---

#### 9.2 — CI Hook for Staleness Detection

**What**: Implement a CLI command and GitHub Action for running staleness checks as a CI step, failing the build when documentation is stale.

**Design**:

```typescript
// doa check-docs [--fail-on-stale] [--repo-path .]
// Exit code 0: all documentation fresh
// Exit code 1: stale documentation found (when --fail-on-stale)

// .github/workflows/doc-freshness.yml
// name: Documentation Freshness
// on: [pull_request]
// jobs:
//   check-docs:
//     runs-on: ubuntu-latest
//     steps:
//       - uses: actions/checkout@v4
//       - run: npx doa check-docs --fail-on-stale
```

**Testing**:
- **ci-passes-fresh**: All anchors fresh; verify exit code 0.
- **ci-fails-stale**: One anchor stale with `--fail-on-stale`; verify exit code 1.
- **ci-warns-stale**: One anchor stale without `--fail-on-stale`; verify exit code 0 with warning output.

---

## Phase 10: Personalized Onboarding Journey Generation

### Purpose

Generate personalized onboarding learning paths for new developers based on their assigned work area, prior experience, and the code graph structure. The journey guides new hires through the most relevant parts of the codebase with targeted reading, exploration, and exercise steps.

### Tasks

#### 10.1 — Developer Profile Analysis

**What**: Analyze a developer's prior experience from their GitHub profile and self-reported skills to build a developer profile used for journey personalization.

**Design**:

```typescript
// packages/api/src/services/journey.ts

export interface DeveloperContext {
  userId: string;
  languagesKnown: string[];
  frameworksKnown: string[];
  yearsExperience: number | null;
  assignedTeam: string | null;
  assignedServiceArea: string | null;    // which part of the codebase they'll work on
}

export async function analyzeDeveloperProfile(
  db: DrizzleClient,
  llmProvider: LLMProvider,
  userId: string,
  githubUsername?: string,
): Promise<DeveloperContext> {
  // 1. Check if profile already exists
  const user = await db.select().from(users).where(eq(users.id, userId)).then(r => r[0]);
  const profile = user?.profile as Record<string, unknown> ?? {};

  // 2. If GitHub username provided and not yet analyzed, fetch public repos
  if (githubUsername && !profile.githubAnalyzed) {
    const repos = await fetchGitHubRepos(githubUsername);
    const languages = extractLanguages(repos);
    const frameworks = await inferFrameworks(repos, llmProvider);
    // Update user profile
  }

  return {
    userId,
    languagesKnown: (profile.languagesKnown as string[]) ?? [],
    frameworksKnown: (profile.frameworksKnown as string[]) ?? [],
    yearsExperience: (profile.yearsExperience as number) ?? null,
    assignedTeam: (profile.assignedTeam as string) ?? null,
    assignedServiceArea: (profile.assignedServiceArea as string) ?? null,
  };
}
```

**Testing**:
- **profile-from-stored**: User has languagesKnown in profile JSONB; verify they are returned.
- **profile-defaults**: User has empty profile; verify empty arrays and nulls returned.

---

#### 10.2 — Journey Generation via Graph Traversal

**What**: Generate a personalized onboarding journey by traversing the code graph starting from seed entities (the developer's assigned service area) and selecting the most important entities to learn about, ordered by dependency depth and PageRank.

**Design**:

```typescript
// packages/api/src/services/journey.ts

export async function generateJourney(
  db: DrizzleClient,
  llmProvider: LLMProvider,
  organizationId: string,
  developerId: string,
  repositoryId: string,
  developerContext: DeveloperContext,
  config: { chatModel: string; maxSteps?: number; traversalDepth?: number },
): Promise<string> {
  const { maxSteps = 15, traversalDepth = 3 } = config;

  // 1. Identify seed nodes from assigned service area
  const seedNodes = await db.select().from(graphNodes)
    .where(and(
      eq(graphNodes.repositoryId, repositoryId),
      or(
        sql`${graphNodes.properties}->>'team_owner' = ${developerContext.assignedTeam}`,
        sql`${graphNodes.qualifiedName} LIKE ${`%${developerContext.assignedServiceArea}%`}`,
      ),
    ));

  // 2. Traverse graph from seeds, collect important entities
  const seedIds = seedNodes.map(n => n.id);
  const reachableEntities = await db.execute(sql`
    WITH RECURSIVE reach AS (
      SELECT id, name, label, qualified_name, properties, pagerank, 0 AS depth
      FROM graph_nodes WHERE id = ANY(${seedIds}::uuid[])
      UNION ALL
      SELECT n.id, n.name, n.label, n.qualified_name, n.properties, n.pagerank, r.depth + 1
      FROM reach r
      JOIN graph_edges e ON e.source_id = r.id OR e.target_id = r.id
      JOIN graph_nodes n ON n.id = CASE WHEN e.source_id = r.id THEN e.target_id ELSE e.source_id END
      WHERE r.depth < ${traversalDepth}
    )
    SELECT DISTINCT ON (id) * FROM reach
    ORDER BY id, depth
  `);

  // 3. Rank entities by importance (PageRank + depth proximity to seeds)
  const rankedEntities = reachableEntities.rows
    .sort((a, b) => {
      const scoreA = (a.pagerank ?? 0) * 10 + (traversalDepth - a.depth);
      const scoreB = (b.pagerank ?? 0) * 10 + (traversalDepth - b.depth);
      return scoreB - scoreA;
    })
    .slice(0, maxSteps);

  // 4. Use LLM to generate step descriptions and determine step types
  const stepsPrompt = buildJourneyPrompt(rankedEntities, developerContext);
  const result = await llmProvider.complete({
    model: config.chatModel,
    messages: [
      { role: "system", content: JOURNEY_GENERATION_SYSTEM_PROMPT },
      { role: "user", content: stepsPrompt },
    ],
    temperature: 0.3,
  });

  const steps: JourneyStep[] = parseJourneySteps(result.content);

  // 5. Store journey
  const [journey] = await db.insert(onboardingJourneys).values({
    organizationId,
    developerId,
    title: `Onboarding Journey for ${developerContext.assignedServiceArea ?? "General"}`,
    status: "active",
    generationMethod: "ai_generated",
    steps,
    seedNodeIds: seedIds,
    graphTraversalDepth: traversalDepth,
  }).returning();

  return journey.id;
}
```

**Testing**:
- **generate-journey-creates-steps** (integration): Create a repo with 20 nodes; generate journey with maxSteps=10; verify journey with 10 steps created.
- **generate-journey-seeds-from-team**: Set assignedTeam="platform"; create nodes with matching team_owner; verify seed nodes are from platform team.
- **generate-journey-pagerank-priority**: Create nodes with varying PageRank; verify higher PageRank entities appear earlier in journey.
- **journey-step-types**: Verify steps include a mix of types (explore_code, read_article, etc.).

---

#### 10.3 — Journey Progress Tracking API

**What**: Implement API endpoints for viewing journey details, marking steps as completed, and tracking overall progress.

**Design**:

```typescript
// POST /api/v1/journeys/generate
// Body: { repositoryId, assignedTeam?, assignedServiceArea?, githubUsername? }
// Response: 201 { journeyId, title, totalSteps }

// GET /api/v1/journeys/:journeyId
// Response: { id, title, status, steps, completedSteps, totalSteps, totalTimeMinutes }

// PATCH /api/v1/journeys/:journeyId/steps/:stepOrder
// Body: { status: "completed" | "skipped", timeSpentMinutes?: number }
// Response: 200 { step, journeyProgress }

// GET /api/v1/users/:userId/journeys
// Response: { journeys: [...] }
```

**Testing**:
- **api-generate-journey**: POST to generate; verify 201 with journeyId and steps.
- **api-complete-step**: PATCH step 1 as completed; verify completedSteps increments and step status updated.
- **api-skip-step**: PATCH step as skipped; verify status is "skipped" and completedSteps does not increment.
- **api-journey-completion**: Complete all steps; verify journey status changes to "completed".

---

## Phase 11: Graph Algorithms and Advanced Analytics

### Purpose

Compute graph-level metrics (PageRank, betweenness centrality, community detection) to identify the most important code entities, architectural clusters, and bottleneck components. Implement onboarding analytics tracking (time-to-first-PR, questions asked, journey completion rates).

### Tasks

#### 11.1 — Graph Metric Computation

**What**: Run PageRank, betweenness centrality, and Louvain community detection on the code graph using the graphology library, then write results back to graph_nodes.

**Design**:

```typescript
// packages/indexer/src/graph/algorithms.ts

import Graph from "graphology";
import pagerank from "graphology-metrics/centrality/pagerank";
import betweennessCentrality from "graphology-metrics/centrality/betweenness";
import louvain from "graphology-communities-louvain";

export async function computeGraphMetrics(
  db: DrizzleClient,
  repositoryId: string,
): Promise<{
  nodesUpdated: number;
  communities: number;
  topEntities: Array<{ name: string; pagerank: number }>;
}> {
  // 1. Load graph into graphology
  const nodes = await db.select().from(graphNodes).where(eq(graphNodes.repositoryId, repositoryId));
  const edges = await db.select().from(graphEdges)
    .where(inArray(graphEdges.sourceId, nodes.map(n => n.id)));

  const graph = new Graph();
  for (const node of nodes) graph.addNode(node.id, { name: node.name });
  for (const edge of edges) {
    if (graph.hasNode(edge.sourceId) && graph.hasNode(edge.targetId)) {
      graph.addEdge(edge.sourceId, edge.targetId, { type: edge.edgeType });
    }
  }

  // 2. Compute metrics
  const pagerankScores = pagerank(graph);
  const betweennessScores = betweennessCentrality(graph);
  const communities = louvain(graph);

  // 3. Write back to database
  for (const node of nodes) {
    await db.update(graphNodes).set({
      pagerank: pagerankScores[node.id] ?? null,
      betweenness: betweennessScores[node.id] ?? null,
      communityId: communities[node.id] ?? null,
      updatedAt: new Date(),
    }).where(eq(graphNodes.id, node.id));
  }

  const topEntities = nodes
    .map(n => ({ name: n.name, pagerank: pagerankScores[n.id] ?? 0 }))
    .sort((a, b) => b.pagerank - a.pagerank)
    .slice(0, 10);

  const uniqueCommunities = new Set(Object.values(communities));

  return { nodesUpdated: nodes.length, communities: uniqueCommunities.size, topEntities };
}
```

**Testing**:
- **pagerank-hub**: Create a star graph with one hub connected to 10 leaves; verify hub has highest PageRank.
- **community-detection**: Create two clusters of 5 densely connected nodes with one bridge edge; verify 2 communities detected.
- **metrics-persisted**: Run computation; verify graph_nodes rows have non-null pagerank, betweenness, communityId values.

---

#### 11.2 — Onboarding Analytics Dashboard API

**What**: Implement analytics event tracking and reporting endpoints for onboarding metrics.

**Design**:

```typescript
// POST /api/v1/analytics/events
// Body: { eventType: string, properties: Record<string, unknown> }

// GET /api/v1/analytics/onboarding/:organizationId
// Query: ?period=30d
// Response: {
//   averageTimeToFirstPR: number (days),
//   averageJourneyCompletionPct: number,
//   totalQuestionsAsked: number,
//   totalArticlesViewed: number,
//   developerMetrics: [{ userId, displayName, timeToFirstPR, journeyCompletion, questionsAsked }]
// }
```

**Testing**:
- **track-event**: POST an analytics event; verify it is stored in analytics_events table.
- **aggregate-metrics**: Insert events for 3 developers over 30 days; GET analytics; verify correct averages.

---

## Phase 12: Multi-Repository Support and MCP Server

### Purpose

Extend the system to support indexing and querying across multiple repositories within an organization — critical for microservice architectures. Implement a Model Context Protocol (MCP) server to expose the onboarding assistant's capabilities to any MCP-compatible AI tool (Cursor, Continue, Claude Code).

### Tasks

#### 12.1 — Multi-Repository Indexing and Cross-Repo Queries

**What**: Allow an organization to index multiple repositories and resolve cross-repository dependencies (service A in repo-1 calls service B in repo-2).

**Design**:

```typescript
// Cross-repository edge resolution:
// 1. After indexing each repo, identify outbound "depends_on" edges that reference
//    entities not found in the current repo.
// 2. Match against graph_nodes from other repos in the same organization.
// 3. Create cross-repo edges with properties: { crossRepo: true, targetRepoId: "..." }

// Q&A retrieval is extended to search across all repos in the organization:
// vectorSearch and graphRAGRetrieve accept repositoryId: string | string[] (array for multi-repo)

export async function resolveCrossRepoEdges(
  db: DrizzleClient,
  organizationId: string,
): Promise<{ edgesCreated: number }> {
  // Find unresolved "depends_on" edges where the target qualified name
  // exists in another repo within the same org.
}
```

**Testing**:
- **cross-repo-resolution**: Repo A has a `depends_on` edge to "UserService"; Repo B has a node named "UserService"; verify cross-repo edge created.
- **multi-repo-search**: Search across 2 repos; verify results from both repos returned.

---

#### 12.2 — MCP Server Implementation

**What**: Expose the onboarding assistant as a Model Context Protocol server, enabling Cursor, Continue, and Claude Code to use it as a context provider.

**Design**:

```typescript
// packages/mcp/src/server.ts

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({ name: "developer-onboarding-assistant", version: "1.0.0" });

// Tool: ask_codebase
server.tool("ask_codebase", {
  description: "Ask an architectural question about the indexed codebase",
  inputSchema: {
    type: "object",
    properties: {
      question: { type: "string", description: "The question to ask" },
      repositoryId: { type: "string", description: "Repository UUID" },
    },
    required: ["question", "repositoryId"],
  },
}, async ({ question, repositoryId }) => {
  const result = await qaService.ask(systemUserId, repositoryId, null, question);
  return { content: [{ type: "text", text: result.result.answer }] };
});

// Tool: get_entity_context
server.tool("get_entity_context", {
  description: "Get code graph context for a specific entity",
  inputSchema: {
    type: "object",
    properties: {
      entityName: { type: "string" },
      repositoryId: { type: "string" },
      depth: { type: "number", default: 2 },
    },
    required: ["entityName", "repositoryId"],
  },
}, async ({ entityName, repositoryId, depth }) => {
  // Return entity details + graph neighbors
});

// Resource: architecture diagrams
server.resource("diagram://{repositoryId}/{diagramType}", async (uri) => {
  // Return Mermaid diagram source
});
```

**Testing**:
- **mcp-ask-tool**: Call `ask_codebase` via MCP protocol; verify answer returned.
- **mcp-entity-tool**: Call `get_entity_context` for a known entity; verify context returned with neighbors.
- **mcp-resource-diagram**: Request a diagram resource; verify Mermaid source returned.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (DB, Types, LLM)
  │
  ├── Phase 2: Code Indexing (Tree-sitter, Graph)
  │     │
  │     ├── Phase 3: Embeddings & Vector Search
  │     │     │
  │     │     └── Phase 4: Q&A Engine (GraphRAG)
  │     │           │
  │     │           ├── Phase 5: CLI + Repo API
  │     │           │
  │     │           └── Phase 6: VS Code Extension
  │     │
  │     ├── Phase 7: Git History & Tribal Knowledge
  │     │     │
  │     │     └── Phase 9: Staleness Detection & Living Docs
  │     │
  │     ├── Phase 8: Architecture Diagrams
  │     │
  │     └── Phase 11: Graph Algorithms & Analytics
  │
  ├── Phase 10: Onboarding Journeys
  │     (depends on Phase 4 + Phase 7)
  │
  └── Phase 12: Multi-Repo + MCP Server
        (depends on Phase 4 + Phase 8)
```

**Parallel tracks after Phase 4**:
- Track A: Phase 5 (CLI) + Phase 6 (VS Code) — user interfaces
- Track B: Phase 7 (Tribal Knowledge) → Phase 9 (Living Docs) — knowledge pipeline
- Track C: Phase 8 (Diagrams) + Phase 11 (Graph Algorithms) — visualization & analytics
- Phase 10 and Phase 12 require multiple earlier phases to complete.

---

## Definition of Done (per phase)

1. All tasks within the phase are implemented and code compiles without errors.
2. All unit tests pass with > 80% line coverage for new code in the phase.
3. All integration tests pass against a real PostgreSQL 17 + Redis instance (Docker Compose).
4. No lint errors (ESLint with strict TypeScript rules).
5. Database migrations are idempotent and can run on a fresh database or an existing one.
6. All new API endpoints have JSON Schema validation on request bodies and return proper error responses (RFC 9457 Problem Details).
7. All new API endpoints are documented in the auto-generated OpenAPI spec.
8. Environment variables and configuration options are documented in the package README.
9. The Docker Compose stack starts cleanly with `docker compose up` and all services are healthy.
10. Manual smoke test of the primary workflow for the phase completes successfully.
