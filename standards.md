# Standards & API Reference

> Project: Developer Onboarding Assistant (Candidate #009) · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

1. **ISO 8601 - Date and Time Format**
   - URL: https://www.iso.org/standard/40874.html
   - Description: Global standard for date and time representation in APIs. Ensures consistent, unambiguous datetime formatting across all API responses and documentation, critical for onboarding systems that track developer activity timelines.

2. **ISO/TS 23029:2020 - Web-Service-Based Application Programming Interface (WAPI) in Financial Services**
   - URL: https://www.iso.org/standard/74353.html
   - Description: Defines framework, function, and protocols for an API ecosystem with logical and technical layered approach for developing APIs. Relevant for data model standardization and API maturity levels applicable to developer onboarding platforms.

3. **ISO 27001 - Information Security Management System**
   - URL: https://www.iso.org/standard/27001
   - Description: International standard for information security. Essential for developer onboarding systems that access codebases, git history, and potentially sensitive architectural information.

### W3C & IETF Standards

4. **JSON-LD 1.1 - JSON for Linked Data**
   - URL: https://www.w3.org/TR/json-ld11/
   - Description: W3C specification for expressing linked data in JSON. Enables semantic enrichment of API documentation and knowledge graphs, allowing onboarding systems to link concepts (services, components, dependencies) across codebases.

5. **RFC 8288 - Web Linking**
   - URL: https://tools.ietf.org/html/rfc8288
   - Description: Defines model for relationships between web resources and link relation types. Critical for API discovery and hypermedia-driven navigation in onboarding documentation.

6. **RFC 9727 - api-catalog: A Well-Known URI and Link Relation to Help Discovery of APIs**
   - URL: https://datatracker.ietf.org/doc/rfc9727/
   - Description: Published June 2025; provides standardized API discovery mechanism. Directly applicable for onboarding systems to programmatically discover and document organization's APIs.

7. **RFC 9457 - Problem Details for HTTP APIs**
   - URL: https://www.rfc-editor.org/rfc/rfc9457.html
   - Description: Standard format for error responses in HTTP APIs. Ensures consistent error documentation and improves developer understanding of failure modes during onboarding.

8. **RFC 9205 - Building Protocols with HTTP**
   - URL: https://datatracker.ietf.org/doc/html/rfc9205
   - Description: Guidelines for building robust HTTP-based APIs and protocols. Foundational for documenting REST API best practices within onboarding systems.

### Data Model & API Specifications

9. **OpenAPI Specification (OAS 3.x)**
   - URL: https://spec.openapis.org/oas/v3.2.0.html
   - Description: Machine-readable standard for describing RESTful APIs. Central to onboarding systems: OAS documents can be automatically parsed to generate API endpoint explanations, code examples, and interactive documentation.

10. **GraphQL Specification**
    - URL: https://spec.graphql.org/
    - Description: Query language and runtime for APIs. Relevant for onboarding systems serving organizations with GraphQL APIs; enables rich, flexible querying of codebase structure and documentation.

11. **JSON Schema Specification**
    - URL: https://json-schema.org/
    - Description: Vocabulary for annotating and validating JSON documents. Used by OpenAPI for data model validation; enables onboarding systems to auto-generate typed documentation and validate code examples.

12. **Tree-sitter Grammar Specifications**
    - URL: https://tree-sitter.github.io/
    - Description: Open parsing library with grammars for 100+ programming languages. Critical for multi-language codebase understanding; used by Code-Graph-RAG and similar tools to build language-agnostic AST-level analysis of codebases.

### Security & Authentication Standards

13. **OAuth 2.0 Protocol & RFC 6749**
    - URL: https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html
    - Description: Industry-standard protocol for secure API authentication. Onboarding systems must integrate OAuth 2.0 to access Git repositories, CI/CD pipelines, and internal documentation safely.

14. **OpenID Connect (OIDC) 1.0**
    - URL: https://openid.net/connect/
    - Description: Identity layer on top of OAuth 2.0. Enables single sign-on (SSO) for onboarding systems, allowing seamless access across organizational tools and repositories.

15. **OWASP Application Security Verification Standard (ASVS) v5.0**
    - URL: https://github.com/OWASP/ASVS
    - Description: Comprehensive security testing framework. Provides guidance on securing onboarding systems that may expose sensitive architectural information and codebase dependencies.

16. **NIST Cybersecurity Framework (CSF) 2.0**
    - URL: https://www.nist.gov/cyberframework
    - Description: Voluntary framework for managing cybersecurity risk through Identify, Protect, Detect, Respond, Recover, and Govern functions. Applies to onboarding systems handling code, intellectual property, and organizational architecture.

### Domain-Specific Data Standards

17. **Conventional Commits Specification v1.0.0**
    - URL: https://www.conventionalcommits.org/en/v1.0.0/
    - Description: Standardized commit message format enabling structured analysis of change history. Onboarding systems can parse commit messages to automatically generate meaningful architecture evolution timelines and decision rationales for new developers.

---

## Similar Products — Developer Documentation & APIs

### 1. Swimm
- **Description:** AI-powered code documentation platform with code-coupled documentation that auto-updates as code changes. Provides IDE plugins and GitHub integration for inline documentation visibility.
- **API Documentation:** https://docs.swimm.io/
- **Developer Guide:** https://docs.swimm.io/getting-started-guide/creating-a-doc/
- **IDE Integration:** VS Code and JetBrains plugins
- **Standards:** Proprietary REST API, GitHub App for CI integration
- **Authentication:** OAuth 2.0 via GitHub

### 2. Sourcegraph Cody
- **Description:** AI-powered code assistant with whole-repository context understanding. Provides codebase Q&A, batch code changes, and cross-repository reasoning for large organizations.
- **API Documentation:** https://sourcegraph.com/docs/api
- **GraphQL API Reference:** https://docs.sourcegraph.com/api/graphql
- **REST API Reference:** https://sourcegraph.com/docs/api
- **Developer Guide:** https://docs.sourcegraph.com/api/graphql/examples
- **Standards:** Follows GraphQL specification for queries; supports REST API (recommended for new integrations in v7.0+)
- **Authentication:** OAuth 2.0, Personal Access Tokens, SAML/OIDC for Enterprise

### 3. GitHub Copilot Chat
- **Description:** Conversational AI assistant integrated into VS Code, JetBrains, and GitHub.com. Provides `@codebase` context for repository-aware Q&A and `@github` for PR/issue integration.
- **API Documentation:** https://docs.github.com/en/rest/copilot
- **Copilot User Management API:** https://docs.github.com/en/rest/copilot/copilot-user-management
- **Copilot Metrics API:** https://docs.github.com/en/rest/copilot/copilot-metrics
- **Developer Guide:** https://github.blog/ai-and-ml/github-copilot/github-for-beginners-building-a-rest-api-with-copilot/
- **Standards:** Follows OpenAPI 3.x specification; REST API with standard HTTP methods
- **Authentication:** OAuth app tokens, Personal Access Tokens with `manage_billing:copilot` scope

### 4. Cursor IDE
- **Description:** AI-first IDE with native whole-repository indexing. Offers Composer Agent mode for autonomous multi-file edits and supports bring-your-own-LLM for data sovereignty.
- **API Documentation:** https://cursor.com/docs/mcp
- **Model Context Protocol Reference:** https://docs.cursor.com/context/model-context-protocol
- **MCP Server Directory:** https://cursor.directory/plugins
- **Developer Guide:** https://cursor.com/docs/mcp
- **Standards:** Supports Model Context Protocol (MCP) servers; compatible with OpenAI, Anthropic, Azure OpenAI, and Ollama
- **Authentication:** API key-based for model backends; custom MCP servers configured via `~/.cursor/mcp.json`

### 5. Aider
- **Description:** Open-source CLI AI pair programmer maintaining a repository map of all files, functions, and classes. Supports multi-model backend and automatic git integration.
- **GitHub Repository:** https://github.com/Aider-AI/aider
- **Official Documentation:** https://aider.chat/docs/
- **Configuration Guide:** https://aider.chat/docs/config.html
- **Usage Documentation:** https://aider.chat/docs/usage.html
- **Standards:** Follows REST API patterns for LLM backends (OpenAI, Anthropic, Ollama); Apache-2.0 licensed
- **Authentication:** API key-based for OpenAI, Anthropic, Azure, AWS Bedrock; supports local Ollama without authentication

### 6. Mintlify
- **Description:** AI-powered documentation platform that auto-generates API documentation from OpenAPI specs and proposes documentation updates whenever code ships. Serves 10,000+ companies.
- **API Documentation:** https://www.mintlify.com/docs
- **API Reference Platform:** https://mint.mintlify.app/api-reference/introduction
- **GitHub Repository:** https://github.com/mintlify/docs
- **Developer Guide:** https://www.mintlify.com/library/api-documentation-guide
- **Standards:** Supports OpenAPI 3.x and AsyncAPI specifications; integrates with Speakeasy and Stainless for SDK generation
- **Authentication:** GitHub OAuth for documentation management; supports bearer, basic, and API key auth in API playgrounds

### 7. Continue IDE
- **Description:** Open-source IDE extension providing custom code RAG for codebase awareness. Supports building retrieval-augmented generation systems for fast, cost-efficient code search.
- **Documentation Site:** https://docs.continue.dev/
- **Custom Code RAG Guide:** https://docs.continue.dev/guides/custom-code-rag
- **Codebase Awareness Guide:** https://docs.continue.dev/guides/codebase-documentation-awareness
- **GitHub Repository:** https://github.com/continuedev/continue
- **Standards:** Supports MCP (Model Context Protocol) servers for context providers; compatible with OpenAI, Anthropic, and other LLM backends
- **Authentication:** API key-based for LLM backends; custom context providers via MCP

### 8. Tabnine
- **Description:** AI code assistant with autocomplete and chat interface. Offers enterprise deployment options (SaaS, single-tenant VPC, on-prem, offline) with full code privacy controls.
- **Documentation Site:** https://docs.tabnine.com/main
- **Quickstart Guide:** https://docs.tabnine.com/main/getting-started/quickstart
- **AI Models Documentation:** https://docs.tabnine.com/main/welcome/readme/ai-models
- **Developer Blog:** https://www.tabnine.com/blog/documenting-code-with-the-tabnine-documentation-agent/
- **Standards:** Proprietary API; supports enterprise deployment modes; REST API for integrations
- **Authentication:** IDE-based authentication; enterprise SSO support (SAML, OIDC)

### 9. ReadMe
- **Description:** API documentation platform for creating beautiful, interactive developer portals. Features Git-style workflows, interactive "Try It" API console, and built-in AI tools.
- **Documentation Platform:** https://readme.com/documentation
- **API Reference:** https://docs.readme.com/main/reference/intro-to-the-readme-api
- **Developer Guide:** https://docs.readme.com/main/docs/quickstart
- **API Documentation Overview:** https://docs.readme.com/main/docs/document-api-overview
- **Standards:** Follows OpenAPI 3.x specification; supports 60+ language syntax highlighting
- **Authentication:** OAuth for ReadMe integrations; API key-based authentication for programmatic access

### 10. Code-Graph-RAG (Multiple Implementations)
- **Description:** Graph-based RAG systems using Tree-sitter for multi-language codebase analysis. Available as MCP servers and CLI tools for semantic code navigation and knowledge graph construction.
- **er77 Implementation:** https://github.com/er77/code-graph-rag-mcp
- **MCP Server Registry:** https://mcpservers.org/servers/github-com-er77-code-graph-rag-mcp
- **RagCode MCP (Privacy-First):** https://github.com/doITmagic/rag-code-mcp
- **CodeGraphContext CLI+MCP:** https://github.com/CodeGraphContext/CodeGraphContext
- **Standards:** Implements Model Context Protocol (MCP) 2025-11-25 specification; supports Tree-sitter grammars for 100+ languages
- **Authentication:** Local-first (no authentication for Ollama backend); supports LLM API keys for cloud models

---

## Integration Patterns & Emerging Standards

### Model Context Protocol (MCP) for Tool Integration
The Model Context Protocol (v2025-11-25) is rapidly becoming the de facto standard for connecting AI systems to external tools and data sources. Several onboarding tools (Cursor, Continue, Code-Graph-RAG variants) now support MCP servers, enabling standardized integration patterns. This creates an opportunity for developer onboarding assistants to expose codebase understanding capabilities as standardized MCP servers, making them available to any MCP-compatible AI IDE.

### Semantic Web & Linked Data for Knowledge Graphs
JSON-LD 1.1 and W3C Linked Data standards enable rich semantic representation of codebase components (services, dependencies, ownership). Emerging onboarding systems can leverage these standards to create queryable, machine-readable knowledge graphs that integrate with other organizational knowledge systems.

### RAG & Retrieval Standards for Code Understanding
Retrieval-Augmented Generation (RAG) is not yet a formal standard but is emerging as the dominant pattern for grounding LLM queries in codebase context. Tree-sitter (MIT-licensed), vector databases (Qdrant, LanceDB), and reranking models are becoming commoditized building blocks for custom code RAG systems.

---

## Notes

### Key Gaps in Current Standards Landscape

1. **No formal standard for codebase knowledge representation**: While OpenAPI covers API contracts, no standard exists for representing codebase architecture, service dependencies, and call graphs. This is a primary opportunity for the Developer Onboarding Assistant to innovate.

2. **Tribal knowledge capture standards**: Standards exist for documentation (Diátaxis framework, but non-normative) and commit conventions (Conventional Commits), but no formal specification for extracting and structuring implicit knowledge from git blame, PR descriptions, and code comments.

3. **Onboarding journey personalization**: No standards exist for generating personalized developer onboarding paths based on role, prior experience, and assigned work area. This represents a key differentiator for AI-native solutions.

4. **Multi-hop architectural reasoning**: Standards like OpenAPI and GraphQL define point-to-point relationships, but no specification covers cross-service, multi-hop reasoning required to answer "why does service A call B before C?" questions.

### Recommended Adoption Path

For the Developer Onboarding Assistant:

1. **Phase 1 (MVP)**: Adopt OpenAPI 3.x for API contract representation; use Conventional Commits for change history analysis; implement OAuth 2.0 for secure codebase access.

2. **Phase 2 (v1.1)**: Add Tree-sitter grammar support for multi-language codebase analysis; implement MCP server pattern to expose onboarding capabilities to other tools; consider JSON-LD for semantic knowledge representation.

3. **Phase 3 (Future)**: Publish proprietary standards for tribal knowledge representation and onboarding journey specification; contribute to emerging standards bodies (IETF, W3C) if addressing generic problems.

### Standards Still Evolving

- **RAG standardization**: Vector database APIs, embedding model specifications, and reranking protocols are rapidly evolving; no stable standard yet (as of May 2026).
- **MCP ecosystem maturity**: While MCP 2025-11-25 is published, adoption is still growing; server discovery and capability negotiation patterns are still consolidating.
- **LLM API standards**: OpenAI, Anthropic, and other vendors maintain proprietary APIs with some convergence (e.g., message format), but no formal standard like OpenAPI exists yet.

