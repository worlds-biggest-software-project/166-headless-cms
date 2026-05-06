# Headless CMS — Feature & Functionality Survey

> Candidate #166 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Contentful | Commercial SaaS | Free tier; paid from ~$300/mo; Enterprise custom | https://contentful.com |
| Sanity | Commercial SaaS | Free tier; Growth $15/user/mo; Enterprise custom | https://sanity.io |
| Strapi | Open Source / SaaS | Free self-host (MIT); Cloud from $29/mo | https://strapi.io |
| Storyblok | Commercial SaaS | Free; Grow from $99/mo; Enterprise custom | https://storyblok.com |
| Hygraph | Commercial SaaS | Free; Scale $299/mo; Enterprise custom | https://hygraph.com |
| Prismic | Commercial SaaS | Free tier; Starter from $0; Scale from $100/mo | https://prismic.io |
| Payload CMS | Open Source / SaaS | Free self-host (MIT); Cloud from $0 | https://payloadcms.com |
| Kontent.ai | Commercial SaaS | Custom enterprise pricing | https://kontent.ai |

## Feature Analysis by Solution

### Contentful

**Core features**
- API-first, structured content modeling with GraphQL and REST delivery
- Content federation: pull data from external systems into unified content graph
- Real-time collaboration and role-based access controls (RBAC)
- AI Actions: automated content workflows for translation, SEO metadata, alt-text
- Composable architecture: integrate with any frontend framework
- Multi-locale support and content versioning
- App marketplace with 300+ integrations
- Webhook support for downstream content pipelines

**Differentiating features**
- Market leader: 300K+ developer ecosystem and proven enterprise reliability
- Compose and Launch products: vertical-specific solutions for commerce and marketing
- AI Actions: LLM-powered content workflows integrated directly in editor
- Content federation: ingest data from external APIs at query time
- Enterprise-grade SLAs and compliance (SOC 2, ISO 27001)

**UX patterns**
- Developer-first: API-native with GraphQL, REST, SDKs across every major language
- Enterprise-oriented: strong governance, audit trails, complex workflows
- Composable: designed to sit at the center of a larger ecosystem
- AI-augmented: automation reduces manual content creation and maintenance

**Integration points**
- 300+ marketplace apps and plugins
- Webhooks for trigger-based workflows
- REST and GraphQL APIs for client applications
- SDKs for JavaScript, Python, Java, Go, Ruby, PHP
- Slack, email, and Zapier integrations

**Known gaps**
- GraphQL API is read-only (mutations via REST only)
- Expensive at scale ($300+/mo base, plus usage)
- Steep learning curve for non-developers
- Content federation can introduce latency (external API calls)

**Licence / IP notes**
- Proprietary commercial SaaS; no licensing concerns

---

### Sanity

**Core features**
- GROQ query language: designed specifically for structured content queries
- Real-time multiplayer editing with character-level sync
- Customizable Studio: React-based, fully programmable editing interface
- Inline comments and field-level collaboration
- Schema-as-code: define content models programmatically
- REST and auto-generated GraphQL APIs
- Real-time listeners for instant application responses
- AI-assisted content workflows (emerging feature)

**Differentiating features**
- GROQ language: purpose-built for nested content; more intuitive than GraphQL for structured data
- Real-time multiplayer: character-level sync with no locking or conflicts
- Customizable Studio: build unique editing experiences for your team
- Content lake: unstructured data stored as queries on a graph
- Strong G2 rankings and developer mindshare

**UX patterns**
- Developer-friendly: schema-as-code and flexible query language
- Collaboration-native: real-time editing with inline comments
- Customizable: full control over editing experience
- Modern: Jamstack-oriented with edge hosting support

**Integration points**
- REST and GROQ APIs for applications
- Auto-generated GraphQL for compatibility
- Webhooks for event-driven workflows
- Integrations with Next.js, Nuxt, Gatsby, and others
- Slack and email integrations

**Known gaps**
- Requires developer investment to configure (less out-of-the-box than Storyblok)
- Smaller ecosystem than Contentful
- GROQ has smaller community than GraphQL
- Limited pre-built integrations

**Licence / IP notes**
- Proprietary commercial SaaS; no licensing concerns

---

### Strapi

**Core features**
- Open-source Node.js headless CMS with MIT license
- Auto-generated REST and GraphQL APIs for every content type
- React-based admin panel connecting to same API as frontends
- Self-hostable: full data ownership, no vendor lock-in
- Plugin system: extend with official plugins or build custom
- Role-based access controls (RBAC)
- Flexible content type builder with relationships
- Supports PostgreSQL, MySQL, SQLite backends

**Differentiating features**
- Fully open-source with no usage limits or seat restrictions
- Self-hosted deployment: complete data ownership
- No API call caps or quotas
- MIT license: permissive for commercial use
- Large community for plugins and integrations
- Developer-friendly: code-first approach

**UX patterns**
- Developer-centric: designed for teams comfortable with infrastructure management
- API-native: auto-generated APIs encourage frontend flexibility
- Open: source code available for audit and customization
- Cost-conscious: free and self-hosted eliminates recurring fees

**Integration points**
- REST and GraphQL APIs auto-generated
- Webhook support for event-driven workflows
- Plugin system for extensibility
- Integrations with popular frontend frameworks
- Community-contributed plugins for common integrations

**Known gaps**
- Infrastructure management burden for self-hosted deployments
- Smaller ecosystem of commercial features vs SaaS competitors
- Community-driven development (less predictable roadmap)
- Limited enterprise support and SLAs

**Licence / IP notes**
- Open Source (MIT License); free to use, modify, and distribute
- No licensing restrictions for commercial use or derivative works

---

### Storyblok

**Core features**
- Visual WYSIWYG editor with drag-and-drop "Bloks" interface
- Component-based architecture: reusable content blocks
- Real-time visual preview of changes
- REST and GraphQL APIs
- Multi-space management for multiple projects
- Role-based access and workflow management
- CDN-backed fast content delivery
- SDKs for Next.js, Nuxt, Vue, React, and others

**Differentiating features**
- Visual editor: only major headless CMS with true WYSIWYG drag-and-drop
- Best-in-class for non-developers: marketers can build pages independently
- Component library: developers define once, marketers use everywhere
- Fast implementation: low time-to-value for marketing teams

**UX patterns**
- Marketer-friendly: visual editor removes need for code
- Component-first: reusable building blocks ensure consistency
- Developer-empowering: developers define guardrails, marketers have freedom
- Real-time: see changes as they happen

**Integration points**
- REST and GraphQL APIs
- Webhooks for trigger-based workflows
- SDKs for major frontend frameworks
- Integrations with marketing tools
- Slack and email notifications

**Known gaps**
- Pricing jumps sharply at enterprise tier
- Less suitable for complex data models
- Smaller ecosystem than Contentful
- GraphQL is secondary to REST API

**Licence / IP notes**
- Proprietary commercial SaaS; no licensing concerns

---

### Hygraph

**Core features**
- GraphQL-native, API-first architecture
- Content federation: query external APIs, databases, and CMS entries in single GraphQL query
- Federated content mesh: combine data from multiple sources at runtime
- Real-time GraphQL subscriptions
- Multi-locale support
- Role-based access controls
- Webhook support for event-driven workflows
- Fast global CDN delivery

**Differentiating features**
- Content federation: unique capability to federate from external sources
- GraphQL-native: pure GraphQL-first approach (no REST fallback required)
- Content mesh: combine CMS, PIM, DAM, commerce, and external APIs
- Lower pricing than Contentful at similar tiers

**UX patterns**
- Developer-first: GraphQL-native interface
- Composable: designed to federate content from multiple sources
- Modern: cloud-native architecture with edge computing
- Data-centric: focus on unified data layer

**Integration points**
- GraphQL subscriptions for real-time updates
- Webhooks for trigger-based workflows
- REST API for compatibility
- Integrations with major frontend frameworks
- Content federation: ingest from any API

**Known gaps**
- Smaller ecosystem than Contentful
- GraphQL expertise required
- Less visual/intuitive than Storyblok for non-developers
- Federation can introduce latency

**Licence / IP notes**
- Proprietary commercial SaaS; no licensing concerns

---

### Prismic

**Core features**
- Slice Machine: developer tool for creating reusable content sections
- Visual page builder for drag-and-drop page construction
- Native Next.js SDK and integration
- REST and GraphQL APIs
- Live editing with real-time preview
- Role-based access and collaborative workflows
- Scheduled publishing and versioning
- CDN delivery across multiple regions

**Differentiating features**
- Slice-based architecture: unique approach where developers define sections, marketers compose pages
- Best Next.js CMS: tight integration with Vercel ecosystem
- Page builder for marketers: slide-deck-like interface for non-technical content teams
- Developer-friendly slices: preview and test locally before shipping

**UX patterns**
- Hybrid: developers define structure, marketers build pages
- Live editing: real-time preview as you build
- Modular: reusable slices reduce configuration overhead
- Next.js-native: first-class support for modern React tooling

**Integration points**
- Native Next.js SDK
- REST and GraphQL APIs
- Webhooks for trigger-based workflows
- Vercel integration for edge rendering
- Integrations with popular services

**Known gaps**
- Less flexible for complex data models
- Smaller ecosystem than competitors
- Smaller user base limits community resources
- Best experience with Next.js; other frameworks less optimized

**Licence / IP notes**
- Proprietary commercial SaaS; no licensing concerns

---

### Payload CMS

**Core features**
- Open-source headless CMS and app framework (MIT license)
- TypeScript-native: full-stack TypeScript for backend and admin panel
- Installs directly into Next.js /app folder (same repo, same deployment)
- REST and GraphQL APIs auto-generated from TypeScript config
- React-based admin panel
- Extends Next.js: middleware, hooks, and file-based routing
- No separate backend infrastructure required
- Webhook and plugin support

**Differentiating features**
- TypeScript-first: developer ergonomics at the core
- Co-located: CMS lives in your Next.js app, same codebase
- Full ownership: free, open-source, no vendor lock-in
- Next.js-native: extends the framework rather than bolting on
- Emerging ecosystem: faster development velocity

**UX patterns**
- Developer-centric: assume TypeScript and Next.js expertise
- Integrated: minimize context switching between app and CMS
- Self-hosted: full control over deployment and data
- Modern: leverages latest Next.js features and patterns

**Integration points**
- REST and GraphQL APIs
- Webhook support for event-driven workflows
- Plugin system for extensibility
- File-based routing and middleware in Next.js
- Community plugins (emerging ecosystem)

**Known gaps**
- Emerging ecosystem: fewer pre-built integrations than mature competitors
- Requires Next.js and TypeScript expertise
- Self-hosted infrastructure management burden
- Smaller community and fewer resources

**Licence / IP notes**
- Open Source (MIT License); free to use, modify, and distribute
- No licensing restrictions for commercial use

---

### Kontent.ai

**Core features**
- Agentic CMS: always-on AI agents automate content workflows
- Content operations platform with governance and compliance
- Workflow automation: agents handle routine tasks without manual intervention
- AI-assisted content creation and translation
- Multi-locale and multi-market support
- Role-based access controls with approval workflows
- REST API for integration
- Mission Control: content monitoring and governance dashboard

**Differentiating features**
- Agentic approach: AI agents continuously execute workflows
- Enterprise-ready governance: compliance, audit trails, enforcement
- Large-scale automation: scale content operations without team growth
- Natural language configuration: define workflows with text prompts
- AI-driven localization: automated translation at scale

**UX patterns**
- Enterprise-focused: governance and compliance built in
- AI-first: automation reduces manual work
- Operations-centric: focus on process efficiency at scale
- Governance-aware: enforce brand voice and compliance automatically

**Integration points**
- REST API for integrations
- Webhook support for trigger-based workflows
- MCP server for AI-native workflows
- Integration with major enterprise systems
- Custom agent configuration

**Known gaps**
- Limited SMB accessibility: enterprise pricing only
- Smaller ecosystem of pre-built integrations
- Complex setup for sophisticated agentic workflows
- Requires change management for process automation

**Licence / IP notes**
- Proprietary commercial SaaS; no licensing concerns

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

- **REST and/or GraphQL API** — Standard query language for clients to fetch content
- **Content modeling** — Flexible schema definition for structured content types
- **Multi-locale support** — Deliver content in multiple languages and regions
- **Role-based access controls** — RBAC for teams with different responsibilities
- **Webhooks** — Trigger downstream workflows on content changes
- **Versioning and history** — Track content changes over time
- **Publishing workflows** — Draft, review, approve, publish content lifecycle
- **Media management** — Upload, organize, and deliver images and files
- **CDN delivery** — Fast, global content delivery
- **API documentation** — Clear, comprehensive API reference

### Differentiating Features

- **Visual page builder** — Drag-and-drop interface for non-developers (Storyblok, Prismic)
- **Real-time collaboration** — Multiple editors on same content simultaneously (Sanity)
- **Content federation** — Query and combine data from external sources (Hygraph, Contentful)
- **AI-assisted workflows** — LLM-powered content generation and optimization (Contentful, Kontent.ai)
- **Customizable editing interface** — Tailor Studio/admin panel to team needs (Sanity)
- **GraphQL-native** — Pure GraphQL-first architecture (Hygraph)
- **Component/slice library** — Reusable content blocks for consistency (Storyblok, Prismic, Payload)
- **Agentic automation** — Always-on AI agents handling routine tasks (Kontent.ai)
- **Open-source options** — Full data ownership and customization (Strapi, Payload)
- **Framework-specific SDKs** — Native integrations with Next.js, Nuxt, etc.

### Underserved Areas / Opportunities

- **Open-source with modern DX** — Strong open-source with commercial feature parity
- **Privacy-first, self-hosted** — For organizations avoiding SaaS and cloud storage
- **Lightweight, headless** — Minimal CMS without feature bloat
- **True content mesh** — Seamless federation across 10+ external data sources
- **AI coaching for writers** — Real-time suggestions on tone, SEO, readability during authoring
- **Semantic content graph** — Understand relationships between entries; suggest internal links
- **Automated content optimization** — ML improve performance by testing variants and surfacing winners
- **Accessibility-first** — Content audit for WCAG compliance, alt text, captions
- **Low-code workflow builder** — Visual workflow automation without code
- **Enterprise headless + visual** — Contentful's power with Storyblok's visual ease

### AI-Augmentation Candidates

- **AI content enrichment** — Current: manual metadata. Better: LLM auto-generate SEO, structured data, alt-text, summaries
- **AI writing assistant** — Current: none. Better: LLM suggest rewrites for tone, clarity, SEO; detect brand voice violations
- **Content gap detection** — Current: manual review. Better: ML identify missing content types, outdated entries, underlinked pages
- **Intelligent translation** — Current: human translation. Better: ML auto-translate with human review queues for high-stakes content
- **Semantic linking** — Current: manual internal links. Better: ML suggest related entries and optimal link placement
- **Agentic workflows** — Current: manual task execution. Better: Always-on agents handle content migrations, bulk updates, compliance audits
- **Predictive publishing** — Current: manual scheduling. Better: ML predict optimal publish time based on audience and topic
- **Content compliance** — Current: manual audit. Better: ML flag content for legal, brand voice, and accessibility issues pre-publication
- **Performance analysis** — Current: basic analytics. Better: ML analyze conversion impact of content changes; suggest optimizations

---

## Legal & IP Summary

**GraphQL and REST standards:** Both GraphQL and REST are open standards with no patent concerns. Organizations can implement either without IP issues.

**Content federation and data integration:** Techniques for federating and combining data from multiple sources are not inherently patented, but specific implementations (e.g., Hygraph's Content Federation) are proprietary. No blocking IP concerns for organizations implementing similar patterns independently.

**AI-assisted workflows:** Kontent.ai's agentic CMS and Contentful's AI Actions rely on LLM technology. While the LLM providers (OpenAI, Anthropic, etc.) may have IP positions, the workflows themselves use standard patterns (prompt engineering, fine-tuning) without novel patent risk.

**CRDT technology:** Real-time collaboration features (Sanity) may use Conflict-free Replicated Data Types (CRDTs) for concurrent editing. CRDT techniques have active patents but are broadly licensed and widely implemented. Independent implementation is feasible.

**No material was omitted due to copyright uncertainty.** All sources were publicly available product documentation, blogs, and comparison articles.

---

## Recommended Feature Scope

Based on the analysis, here's a prioritised feature scope for the project:

### Must-Have (MVP)

- **Content modeling** — Flexible schema definition with support for relationships and references
- **REST API** — Standard HTTP API for client applications to fetch content
- **RBAC and authentication** — Role-based access controls and user management
- **Multi-locale support** — Deliver content in multiple languages
- **Publishing workflows** — Draft → Review → Approve → Publish lifecycle
- **Webhooks** — Trigger downstream workflows on content changes
- **Media management** — Upload, organize, and serve images and files
- **Basic versioning** — Track content changes and revert to prior versions

### Should-Have (v1.1)

- **GraphQL API** — Query language for precise field selection
- **Real-time collaboration** — Multiple editors on same content simultaneously
- **Component/slice library** — Reusable content blocks for consistency
- **Content federation** — Query external APIs and combine results
- **Visual page builder** — Drag-and-drop interface for non-developers
- **CDN delivery** — Fast, global content distribution
- **Workflow automation** — Pre-built workflows for common scenarios
- **Search and indexing** — Full-text search across content

### Nice-to-Have (Backlog)

- **AI content enrichment** — Auto-generate SEO metadata, alt-text, summaries
- **AI writing assistant** — Real-time suggestions on tone, clarity, brand voice
- **Agentic workflows** — Always-on agents for bulk updates and migrations
- **Customizable editing interface** — Tailor admin panel to team needs
- **Semantic linking** — Auto-suggest internal links and related content
- **Content compliance audit** — Flag content for legal and accessibility issues
- **Performance analytics** — Understand content impact on conversion and engagement
- **Intelligent translation** — ML-powered with human review queues
- **Predictive publishing** — Suggest optimal publish times based on audience
- **Visual workflow builder** — No-code automation for complex processes

