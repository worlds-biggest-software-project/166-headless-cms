# Standards & API Reference

> Project: Headless CMS · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access controls, audit logging, and encryption for headless CMS platforms handling website content, marketing assets, and editorial workflows; Annex A A.8.3 (Information access restriction) and A.5.14 (Information transfer). URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of author personal data, contributor information, and user-generated content in cloud-hosted headless CMS platforms. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **RFC 6749 — OAuth 2.0** — Authorization framework used by all major headless CMS platforms (Contentful, Sanity, Storyblok) for third-party app authorization, personal access token generation, and webhook delivery authentication. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Used for API authentication tokens and content preview tokens providing time-limited access to unpublished draft content in headless CMS preview environments. URL: https://datatracker.ietf.org/doc/html/rfc7519

- **RFC 5988 / RFC 8288 — Web Linking** — Defines HTTP Link header for pagination in REST API responses; used by headless CMS REST APIs for cursor-based and page-based content listing pagination. URL: https://datatracker.ietf.org/doc/html/rfc8288

- **RFC 7807 — Problem Details for HTTP APIs** — Standard JSON error response format; used by modern headless CMS REST APIs (Strapi, Payload) for structured API error responses. URL: https://datatracker.ietf.org/doc/html/rfc7807

- **RFC 9110 — HTTP Semantics** — Governs the REST API design of headless CMS Content Delivery and Management APIs; cache headers (ETag, Cache-Control) are critical for CDN-cached content delivery. URL: https://datatracker.ietf.org/doc/html/rfc9110

- **W3C WCAG 2.2 — Web Content Accessibility Guidelines** — Governs the web editor UIs of headless CMS admin interfaces; draft editing panels, media upload interfaces, and content preview tools must meet WCAG 2.2 AA accessibility standards. URL: https://www.w3.org/TR/WCAG22/

### Data Model & API Specifications

- **GraphQL (June 2018 Specification)** — The dominant query language for headless CMS content delivery; used natively by Contentful (GraphQL Content API), Sanity (secondary to GROQ), Hygraph (GraphQL-first), and Payload CMS; enables clients to fetch exactly the fields needed, avoiding over-fetching in complex content models. URL: https://spec.graphql.org/June2018/

- **GROQ — Graph-Relational Object Queries** — Sanity's proprietary query language purpose-built for filtering and projecting JSON documents; not GraphQL but offers more concise syntax for complex content queries; GROQ is Sanity's primary query language with GraphQL as a secondary option. URL: https://www.sanity.io/docs/groq

- **OpenAPI 3.1** — Used by Strapi, Payload, and Contentful (Management API) to describe their REST APIs; enables SDK code generation and Postman collection creation; Strapi auto-generates an OpenAPI spec from the content type schema. URL: https://spec.openapis.org/oas/latest.html

- **JSON Schema** — Standard for defining content type schemas in headless CMS platforms; Payload CMS uses JSON Schema extensively for content model validation; widely used for content import/export format definition. URL: https://json-schema.org/

- **CloudEvents (CNCF)** — Emerging standard for structured webhook event delivery; headless CMS platforms use webhooks for triggering static site rebuilds (Vercel, Netlify), incremental static regeneration (Next.js ISR), and content pipeline automation. URL: https://cloudevents.io/

- **Portable Text** — Sanity's open-source format for rich text content that is a JSON-based, block-level document format; provides a standardised way to serialise rich text to HTML, Markdown, or any format; more powerful than markdown, more portable than HTML. URL: https://portabletext.org/

- **Markdown / CommonMark** — The universal plain-text rich text format; used by Strapi, Payload, and other headless CMS platforms as one of their rich text field formats; CommonMark (spec.commonmark.org) is the canonical Markdown specification. URL: https://spec.commonmark.org/

### Security & Authentication Standards

- **GDPR Article 25 — Data Protection by Design and Default** — Headless CMS platforms handling user-generated content, author profiles, or personalised content must implement data minimisation and privacy-by-design in content model architecture. URL: https://gdpr-info.eu/art-25-gdpr/

- **GDPR Article 32 — Security of Processing** — Content management platforms storing editorial drafts, media assets, and content history must implement encryption at rest and in transit, access controls, and data retention policies. URL: https://gdpr-info.eu/art-32-gdpr/

- **SOC 2 Type II** — Required enterprise compliance certification for SaaS headless CMS platforms; Contentful, Sanity, Storyblok, and Hygraph maintain SOC 2 Type II reports for enterprise procurement. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **SCIM 2.0 (RFC 7643/7644)** — Used for automated user provisioning and deprovisioning in enterprise headless CMS deployments with identity providers (Okta, Azure AD) for managing author and editor accounts. URL: https://datatracker.ietf.org/doc/html/rfc7643

- **SAML 2.0 / OIDC** — Required for enterprise SSO integration in headless CMS platforms; Contentful, Sanity, Storyblok, and Strapi Enterprise support SAML 2.0 for corporate identity provider integration. URL: https://docs.oasis-open.org/security/saml/v2.0/

- **OWASP API Security Top 10 (2023)** — Governs REST and GraphQL API security for headless CMS; API1 (Broken Object Level Authorization) is critical for multi-space/multi-organisation content isolation; API3 (Broken Object Property Level Authorization) governs draft vs. published content access. URL: https://owasp.org/API-Security/

### MCP Server Specifications

Headless CMS platforms are actively building MCP integrations for AI-native content workflows:

- **Sanity MCP Server (Official)** — Sanity's official MCP server hosted at `mcp.sanity.io`; enables AI agents to execute GROQ queries, manage content releases, patch documents, and perform content management operations with full schema awareness; follows Anthropic's official MCP specification; works with any MCP-compatible client. URL: https://www.sanity.io/docs/ai/mcp-server

- **Contentful MCP Integration** — Contentful's Content Management API and GraphQL API are well-documented for integration into AI workflows; community MCP servers exist for querying Contentful spaces and managing content entries via AI agents.

- **Strapi MCP Pattern** — Strapi's auto-generated OpenAPI spec and REST API make it straightforward to build MCP server adapters; community MCP server implementations for Strapi exist for content CRUD operations via AI agents.

---

## Similar Products — Developer Documentation & APIs

### Contentful

- **Description:** Leading enterprise headless CMS (SaaS); Content Delivery API (CDA), Content Management API (CMA), Content Preview API, and GraphQL Content API; removed free community tier in Q2 2025; minimum $3,600/year for startups; strong enterprise compliance.
- **API Documentation:** https://www.contentful.com/developers/docs/references/
- **GraphQL Content API:** https://www.contentful.com/developers/docs/references/graphql/
- **SDKs/Libraries:** contentful (JavaScript/TypeScript delivery SDK); contentful-management (Node.js management SDK); contentful-python; contentful-java; contentful-swift; contentful-android
- **Developer Guide:** https://www.contentful.com/developers/docs/
- **Standards:** REST/JSON, GraphQL, OpenAPI, OAuth 2.0, Webhooks, SAML 2.0 (Enterprise)
- **Authentication:** API token (CDN-cached delivery); OAuth 2.0 (management); personal access tokens

### Sanity

- **Description:** Developer-experience-first headless CMS; GROQ query language as primary API; real-time collaborative editing; Portable Text for rich text; Sanity Studio (customisable React admin); #1 on G2 for 4 years; official MCP server at mcp.sanity.io.
- **API Documentation:** https://www.sanity.io/docs/api
- **GROQ Reference:** https://www.sanity.io/docs/groq
- **MCP Server:** https://www.sanity.io/docs/ai/mcp-server
- **SDKs/Libraries:** @sanity/client (JavaScript/TypeScript); sanity (npm, Studio); next-sanity; groq (npm); sanity-python (community)
- **Developer Guide:** https://www.sanity.io/docs
- **Standards:** GROQ (proprietary query language), GraphQL (secondary), REST/JSON, OpenAPI, Portable Text, Webhooks, OAuth 2.0
- **Authentication:** API token (per project); CORS allowlisting; SAML 2.0 (Enterprise); personal tokens

### Strapi (Open Source)

- **Description:** Open-source (Elastic License 2.0 for Enterprise, Community Edition MIT) headless CMS and API framework; auto-generates REST + GraphQL APIs from content types; admin panel; Strapi Cloud (managed) or self-hosted; most mature open-source headless CMS.
- **API Documentation:** https://docs.strapi.io/dev-docs/api
- **REST API Reference:** https://docs.strapi.io/dev-docs/api/rest
- **GraphQL:** https://docs.strapi.io/dev-docs/api/graphql
- **SDKs/Libraries:** @strapi/sdk-js (official JavaScript SDK, v5); strapi-plugin ecosystem (100+ plugins); auto-generated OpenAPI spec per content type
- **Developer Guide:** https://docs.strapi.io/
- **Standards:** REST/JSON (OpenAPI 3.0 auto-generated), GraphQL, Webhooks, OAuth 2.0, SAML 2.0 (Enterprise), LDAP (Enterprise)
- **Authentication:** API token; JWT (user session); OAuth 2.0; SAML 2.0 (Enterprise)

### Payload CMS (Open Source)

- **Description:** Open-source (MIT licence) TypeScript-native headless CMS and application framework; runs inside Next.js as a package; REST and GraphQL APIs; Lexical rich text editor; no separate server required; strong type safety; growing rapidly in the Next.js ecosystem.
- **API Documentation:** https://payloadcms.com/docs/api
- **SDKs/Libraries:** payload (npm); @payloadcms/next (Next.js integration); @payloadcms/richtext-lexical; community SDKs
- **Developer Guide:** https://payloadcms.com/docs
- **Standards:** REST/JSON (OpenAPI), GraphQL, Webhooks, JWT, TypeScript-native, MIT licence
- **Authentication:** JWT (session cookies); API keys for server-to-server; OAuth 2.0 plugins available

### Storyblok

- **Description:** Component-based headless CMS with a visual editor showing live previews; popular for marketing teams requiring WYSIWYG editing; Content Delivery API v2, Management API, and GraphQL API; strong in e-commerce and marketing sites.
- **API Documentation:** https://www.storyblok.com/docs/api/
- **SDKs/Libraries:** @storyblok/js (JavaScript); storyblok-nuxt; storyblok-react; storyblok-python; @storyblok/svelte
- **Developer Guide:** https://www.storyblok.com/docs/guide/introduction
- **Standards:** REST/JSON (v2 CDN API), GraphQL, Webhooks, OAuth 2.0, SAML 2.0 (Enterprise), OpenAPI
- **Authentication:** API token (public + preview tokens); OAuth 2.0 for management; SAML 2.0 (Enterprise)

### Hygraph (formerly GraphCMS)

- **Description:** GraphQL-native headless CMS; federation-first (content federation across multiple data sources); Content API is exclusively GraphQL; used in enterprise omnichannel content delivery; strong in combining headless CMS with API orchestration.
- **API Documentation:** https://hygraph.com/docs/api-reference/
- **SDKs/Libraries:** @hygraph/utils (JavaScript); graphql-request; Apollo Client; community SDKs
- **Developer Guide:** https://hygraph.com/docs/
- **Standards:** GraphQL (primary), REST/JSON (Management API), Webhooks, OAuth 2.0, SAML 2.0, OpenAPI
- **Authentication:** API token (public/permanent tokens); OAuth 2.0; PAT; SAML 2.0 (Enterprise)

### Directus (Open Source)

- **Description:** Open-source (BSL 1.1 / self-hosted free) data platform and headless CMS wrapping any SQL database; auto-generates REST and GraphQL APIs from the database schema; Directus Cloud (managed) or self-hosted; strong for data-centric content management.
- **API Documentation:** https://docs.directus.io/reference/introduction.html
- **SDKs/Libraries:** @directus/sdk (JavaScript/TypeScript); directus-python; @directus/extensions-sdk
- **Developer Guide:** https://docs.directus.io/
- **Standards:** REST/JSON (OpenAPI 3.0), GraphQL, Webhooks, OAuth 2.0, SAML 2.0 (Enterprise), LDAP, SCIM 2.0
- **Authentication:** Static tokens; Session tokens (JWT); OAuth 2.0; SAML 2.0 (Enterprise)

### Ghost (Open Source)

- **Description:** Open-source (MIT) publishing platform designed for content creators; headless mode via Content and Admin APIs; member subscriptions and newsletter integration; widely used for technical blogs and creator publications.
- **API Documentation:** https://ghost.org/docs/content-api/
- **SDKs/Libraries:** @tryghost/content-api (JavaScript); @tryghost/admin-api (JavaScript); Ghost.js community SDK; ghost-python
- **Developer Guide:** https://ghost.org/docs/
- **Standards:** REST/JSON (OpenAPI), Webhooks, OAuth 2.0, JWT, CommonMark (Markdown)
- **Authentication:** Content API key (public read); Admin API key (JWT-based management)

---

## Notes

- **Sanity MCP Server as the 2026 standard**: Sanity's official MCP server at `mcp.sanity.io` is the most mature headless CMS MCP integration; it enables AI agents to perform full CRUD content operations with schema awareness — representing the leading edge of AI-native CMS workflows.

- **Contentful free tier removal (Q2 2025)**: Contentful's removal of its free community tier in Q2 2025 ended its position as the default recommendation for new developer projects; Sanity (free up to $180/year) and Strapi (open source, self-hostable) are the primary beneficiaries.

- **GraphQL vs GROQ**: GraphQL provides familiar cross-platform queries; GROQ (Sanity-specific) provides more concise syntax for complex nested queries; new projects should evaluate whether GROQ lock-in is acceptable given Sanity's superior developer experience.

- **Payload CMS growth**: Payload's TypeScript-native Next.js integration (running CMS and API in the same Next.js application) is gaining significant traction in the Next.js ecosystem for developers who want to avoid a separate CMS infrastructure.

- **Portable Text as a portable rich-text standard**: Sanity's Portable Text format (portabletext.org) provides a more structured and portable alternative to HTML or Markdown for rich text content; serialisers exist for all major frameworks and output formats.

- **Open-source landscape**: Strapi (MIT Community Edition), Payload (MIT), Directus (BSL 1.1 self-hosted), and Ghost (MIT) represent the major open-source headless CMS options; Strapi has the largest ecosystem and most mature enterprise features; Payload is the fastest-growing option for TypeScript/Next.js developers.
