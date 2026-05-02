# Developer Onboarding Assistant

An AI-native codebase understanding platform that accelerates developer ramp-up by extracting tribal knowledge, generating personalized learning paths, and maintaining living architecture documentation.

## The Problem

Developer onboarding remains one of the highest-friction, highest-cost activities in software delivery:

- **Tribal knowledge loss**: The "why" behind architectural decisions, workarounds, and naming conventions exists only in senior engineers' heads—captured nowhere systematically
- **Generic documentation**: Existing doc tools treat all developers as identical; a mobile developer needs different context than a backend integrator
- **Architectural amnesia**: Teams cannot answer multi-hop questions requiring reasoning across the dependency graph ("why does payment service call inventory before order-service, and what happens if it fails?")
- **Staleness inevitable**: Documentation goes out-of-date immediately after writing; no tool auto-detects architectural changes and updates docs
- **Expensive commercial tools**: All codebase-aware AI assistants (Cursor $20/month, Copilot Enterprise $39/month, Cody Enterprise $59/month) are commercial; no production-grade open-source alternative exists

True cost of onboarding a developer: $25,000–$85,000 per hire. 22% of developers leave within 90 days without structured onboarding. Remote onboarding costs 10–20% more. Yet average ramp-up (time-to-10th-PR) remains 33 days as of Q1 2026.

## The Opportunity

Build an AI-native assistant that:

1. **Tribal knowledge extraction and structured capture**: Analyze git blame history, PR descriptions, Slack threads (via integrations), and code comments to automatically surface and synthesize implicit knowledge into structured, searchable onboarding content. No current open-source tool addresses this at the codebase-graph level.

2. **Interactive architecture explanation with multi-hop reasoning**: Existing tools answer questions about individual files/functions but struggle with architectural questions requiring reasoning across the dependency graph. A GraphRAG-based layer over the codebase's call graph, data flow, and deployment topology could answer cross-service questions coherently—something vector-only RAG cannot.

3. **Personalized onboarding journeys generated from first-PR analysis**: No current tool generates dynamic onboarding paths tailored to what a specific developer will actually work on. AI could analyze assigned issues, team's service ownership, and incoming developer's prior experience (inferred from GitHub history or CV) to generate a prioritized "codebase tour" with targeted exercises.

4. **Living architecture documentation that stays current**: The fundamental failure of all doc tools is staleness. AI could run as a CI hook, detect when merged PRs change architectural boundaries (new service-to-service calls, new DB tables, changed API contracts), automatically update affected architecture diagrams/explanations, and open a PR for human review.

5. **Accessible codebase Q&A without commercial subscription**: All current codebase-aware AI tools are commercial SaaS. An open-source AI-native tool that developers can self-host against their own LLM endpoint (Ollama, local models, BYO API key) addresses open-source projects, cost-sensitive startups, and enterprises with data-sovereignty requirements.

## Market Context

- **Market size**: AI code assistants $5.42–$8.5B (2026) → $6.5–20B+ (2035); Developer experience/tooling segment rapidly growing
- **Adoption**: 92.6% of developers use an AI coding assistant monthly (Stack Overflow 2025); 78% of Fortune 500 companies have AI-assisted dev in production (2026, up from 42% in 2024)
- **Impact**: Average 3.6 hours/week time saved per developer using AI tools; ramp-up time (time-to-10th-PR) dropped from 39 days (Q4 2025) to 33 days (Q1 2026)
- **Buyer personas**: Engineering managers at 50–500 engineer scaling companies, staff/principal engineers bearing institutional knowledge burden, remote-first/distributed teams, platform/DevEx teams focused on DORA metrics
- **Recent moves**: CodeSee (visual codebase maps) shut down 2024 post-Atlassian acquisition; Sourcegraph raised $125M Series D; GitHub Copilot Enterprise $39/user/month (2024)

## Key Features

**MVP**
- Whole-repository code graph indexing (files, functions, classes, dependencies, call graph)
- Natural-language Q&A over indexed codebase: answer architectural and implementation questions
- Multi-hop reasoning: questions traversing the dependency graph across files and services
- Self-hostable with bring-your-own-LLM (Ollama, OpenAI, Anthropic)—no data egress requirement
- VS Code extension for in-editor codebase Q&A

**v1.1 Enhancements**
- Tribal knowledge extraction: synthesize implicit context from git history, PR descriptions, code comments into structured documentation
- Code-coupled documentation with staleness detection: docs linked to code identifiers, CI check when they drift
- Personalized onboarding journey generation: role-specific codebase tour based on assigned work area
- Architecture diagram generation from indexed code graph (Mermaid or PlantUML output)

**Vision (Backlog)**
- Living documentation CI hook: automatically open documentation update PRs when architectural boundaries change
- Multi-repository context for microservice-per-repo organizations
- Onboarding analytics: time-to-first-PR, documentation views, Q&A patterns per new hire
- Integration with collaboration tools (Slack, Confluence) to capture tribal knowledge from existing conversations

## Research & References

- **Understanding Codebase like a Professional (2025)**: "Human–AI Collaboration for Code Comprehension" — arxiv:2504.04553
- **Knowledge Graph Based Repository-Level Code Generation (2025)**: arxiv:2505.14394
- **Peng et al. (2024)**: "Graph Retrieval-Augmented Generation: A Survey" — ACM Transactions on Information Systems
- **Meta Engineering (2026)**: "How Meta Used AI to Map Tribal Knowledge in Large-Scale Data Pipelines"
- **DX Research (2026)**: "Developer Ramp-Up Time Continues to Accelerate with AI"
- **McKinsey (2025)**: "AI Coding Tools Reduce Routine Coding Tasks by 46%" — survey of 4,500+ developers across 150 enterprises

## Technology Stack Considerations

- **Code graph construction**: Tree-sitter for AST parsing (100+ languages) + call graph analysis
- **Knowledge graph**: GraphRAG approach with semantic embedding + entity/relationship extraction (LLM-powered)
- **Multi-hop reasoning**: Graph-based retrieval augmented generation (GraphRAG) for dependency-aware Q&A
- **Tribal knowledge extraction**: NLP + LLM for git blame analysis, PR description mining, code comment synthesis
- **Documentation generation**: LLM-based architecture diagram generation (Mermaid/PlantUML) + narrative explanation
- **Model flexibility**: Support for Ollama, OpenAI, Anthropic, or self-hosted LLM for data sovereignty

## Why Now?

- **92.6% developer AI adoption**: momentum is unstoppable; open-source alternative fills data-sovereignty/cost gap
- **Ramp-up time still 33 days**: despite AI tooling, no product specifically addresses onboarding workflow
- **CodeSee shutdown**: indicated consolidation into larger platforms; room for purpose-built open-source alternative
- **Meta + McKinsey validation**: 2025–2026 peer-reviewed and industry research proving value of tribal knowledge capture
- **Regulatory/compliance tailwind**: enterprises with IP sensitivity cannot use commercial SaaS tools; self-hosted alternative is prerequisite

---

**Status**: Research complete (April 2026) | **Research Files**: [research.md](./research.md), [features.md](./features.md)
