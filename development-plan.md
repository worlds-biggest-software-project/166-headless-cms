# Headless CMS — Phased Development Plan

> Project: 166-headless-cms · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | TypeScript-native aligns with Payload CMS and Strapi patterns; strong typing for content schema generation; largest ecosystem for CMS tooling; first-class GraphQL and REST support |
| API framework | Fastify 5 | Fastest Node.js HTTP framework; native OpenAPI 3.1 schema generation via @fastify/swagger; plugin architecture mirrors CMS extensibility needs; JSON Schema validation built-in |
| GraphQL layer | Mercurius (Fastify GraphQL) | Tightly integrated with Fastify; supports subscriptions, federation, and JIT compilation; avoids Apollo licensing concerns |
| Database | PostgreSQL 16 + pgvector | Hybrid JSONB model (Data Model Suggestion 2) for flexible content storage; pgvector for AI embeddings (Data Model Suggestion 4); JSONB GIN indexes for fast content queries; industry standard for CMS platforms |
| ORM / query builder | Drizzle ORM | TypeScript-first with zero-overhead SQL generation; supports JSONB operators and pgvector; schema-as-code aligns with content type schema generation; better raw SQL escape hatch than Prisma |
| Task queue | BullMQ (Redis 7) | Mature Node.js queue for webhook delivery, AI enrichment pipelines, scheduled publishing, and content federation; supports priorities, retries, rate limiting, and cron jobs |
| Search engine | Meilisearch | Open-source, self-hostable full-text search; sub-50ms queries; typo tolerance; faceted search for content filtering; simpler ops than Elasticsearch |
| Object storage | S3-compatible (MinIO for self-hosted) | Industry standard for media storage; MinIO provides self-hosted S3 API; supports pre-signed URLs for direct upload |
| Frontend (admin) | React 19 + Vite 6 | React is the standard for CMS admin panels (Strapi, Sanity, Payload all use React); Vite for fast HMR during admin development |
| UI component library | shadcn/ui + Tailwind CSS 4 | Unstyled primitives allow full theming; accessible by default (WCAG 2.2 AA); copy-paste model avoids dependency bloat |
| Rich text editor | Lexical | Meta's extensible rich text framework; used by Payload CMS; supports Portable Text serialisation; plugin architecture for custom blocks |
| Authentication | Lucia Auth + oslo | Lightweight, framework-agnostic auth library; supports session-based auth, OAuth 2.0, SAML 2.0; no vendor lock-in |
| AI/LLM integration | Anthropic SDK + OpenAI SDK (pluggable) | Provider-agnostic AI layer; Anthropic for primary content enrichment; OpenAI for embeddings (text-embedding-3-large) |
| Containerisation | Docker + Docker Compose | Standard for self-hosted deployment; single docker-compose.yml for PostgreSQL, Redis, Meilisearch, MinIO, and CMS |
| Testing | Vitest + Supertest + Playwright | Vitest for unit/integration (fast, native ESM); Supertest for API testing; Playwright for admin UI e2e |
| Code quality | Biome (lint + format) + tsc --noEmit | Biome replaces ESLint + Prettier with 100x faster performance; tsc for type checking |
| Package manager | pnpm 9 | Strict dependency resolution; workspace support for monorepo; faster than npm |
| Monorepo | pnpm workspaces + Turborepo | Packages: @headless-cms/core, @headless-cms/admin, @headless-cms/cli, @headless-cms/sdk |

### Project Structure

```
headless-cms/
├── package.json                          # Root workspace config
├── pnpm-workspace.yaml
├── turbo.json                            # Turborepo pipeline
├── docker-compose.yml                    # PostgreSQL, Redis, Meilisearch, MinIO
├── Dockerfile                            # Production multi-stage build
├── .env.example
├── packages/
│   ├── core/                             # Backend API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── drizzle.config.ts
│   │   ├── src/
│   │   │   ├── index.ts                  # Fastify server entrypoint
│   │   │   ├── config.ts                 # Environment config with validation
│   │   │   ├── db/
│   │   │   │   ├── schema.ts             # Drizzle schema (system tables)
│   │   │   │   ├── migrations/           # Drizzle migrations
│   │   │   │   └── seed.ts               # Default roles, locales, admin user
│   │   │   ├── modules/
│   │   │   │   ├── content-type/         # Content type CRUD + schema registry
│   │   │   │   │   ├── content-type.schema.ts
│   │   │   │   │   ├── content-type.service.ts
│   │   │   │   │   ├── content-type.routes.ts
│   │   │   │   │   └── content-type.test.ts
│   │   │   │   ├── entry/                # Entry CRUD + publishing
│   │   │   │   │   ├── entry.schema.ts
│   │   │   │   │   ├── entry.service.ts
│   │   │   │   │   ├── entry.routes.ts
│   │   │   │   │   └── entry.test.ts
│   │   │   │   ├── auth/                 # Users, roles, tokens, sessions
│   │   │   │   ├── media/                # Upload, transform, serve
│   │   │   │   ├── locale/               # Multi-locale management
│   │   │   │   ├── webhook/              # Webhook CRUD + delivery
│   │   │   │   ├── workflow/             # Publishing workflows
│   │   │   │   ├── version/              # Entry versioning
│   │   │   │   ├── search/               # Meilisearch integration
│   │   │   │   ├── federation/           # Content federation
│   │   │   │   └── ai/                   # AI enrichment pipeline
│   │   │   │       ├── enrichment.service.ts
│   │   │   │       ├── embedding.service.ts
│   │   │   │       ├── translation.service.ts
│   │   │   │       └── writing-assistant.service.ts
│   │   │   ├── graphql/
│   │   │   │   ├── schema.ts             # Dynamic GraphQL schema from content types
│   │   │   │   ├── resolvers/
│   │   │   │   └── directives/
│   │   │   ├── queue/
│   │   │   │   ├── workers/              # BullMQ workers
│   │   │   │   └── jobs.ts               # Job type definitions
│   │   │   ├── plugins/                  # Plugin system
│   │   │   │   ├── plugin.interface.ts
│   │   │   │   └── plugin.loader.ts
│   │   │   └── shared/
│   │   │       ├── errors.ts
│   │   │       ├── pagination.ts
│   │   │       └── middleware/
│   │   └── tests/
│   │       ├── fixtures/                 # Test content types, entries, media
│   │       ├── helpers/                  # Test utilities, DB setup/teardown
│   │       └── e2e/                      # End-to-end API tests
│   ├── admin/                            # React admin panel
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts
│   │   ├── index.html
│   │   └── src/
│   │       ├── main.tsx
│   │       ├── App.tsx
│   │       ├── components/               # Shared UI components
│   │       ├── features/                 # Feature-sliced modules
│   │       │   ├── content-types/
│   │       │   ├── entries/
│   │       │   ├── media/
│   │       │   ├── users/
│   │       │   ├── workflows/
│   │       │   ├── webhooks/
│   │       │   └── ai/
│   │       ├── hooks/
│   │       ├── lib/                      # API client, auth, utils
│   │       └── styles/
│   ├── sdk/                              # JavaScript/TypeScript client SDK
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── client.ts
│   │   │   ├── types.ts
│   │   │   └── content-delivery.ts
│   │   └── tests/
│   └── cli/                              # CLI for project scaffolding
│       ├── package.json
│       └── src/
│           └── index.ts
└── docs/
    ├── api/                              # Auto-generated OpenAPI + GraphQL docs
    └── guides/
```

---

## Phase 1: Foundation — Project Setup and Database Schema

### Purpose
Establish the monorepo structure, development tooling, Docker services, and core database schema. After this phase, a developer can clone the repo, run `docker compose up && pnpm dev`, and have a running Fastify server connected to PostgreSQL with all system tables created and seeded.

### Tasks

#### 1.1 — Monorepo Scaffolding and Tooling

**What**: Create the pnpm workspace with Turborepo, configure TypeScript, Biome, and Vitest across all packages.

**Design**:

Root `package.json`:
```json
{
  "name": "headless-cms",
  "private": true,
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "biome check .",
    "format": "biome format --write .",
    "typecheck": "turbo typecheck"
  },
  "devDependencies": {
    "@biomejs/biome": "^2.0.0",
    "turbo": "^2.5.0",
    "typescript": "^5.8.0",
    "vitest": "^3.2.0"
  }
}
```

Root `tsconfig.json` (base config):
```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
```

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "typecheck": { "dependsOn": ["^build"] },
    "lint": {}
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: turbo build succeeds for all packages`
- `Unit: biome check . reports no issues on scaffolded code`
- `Unit: tsc --noEmit passes for all packages`
- `Unit: vitest runs with zero tests and exits cleanly`

---

#### 1.2 — Docker Compose Development Environment

**What**: Create docker-compose.yml with PostgreSQL 16 + pgvector, Redis 7, Meilisearch, and MinIO containers.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: headless_cms
      POSTGRES_USER: cms
      POSTGRES_PASSWORD: cms_dev_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U cms"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

  meilisearch:
    image: getmeili/meilisearch:v1.12
    ports: ["7700:7700"]
    environment:
      MEILI_MASTER_KEY: cms_dev_meili_key
      MEILI_ENV: development
    volumes:
      - meilidata:/meili_data

  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: cms_dev_minio
      MINIO_ROOT_PASSWORD: cms_dev_minio_password
    command: server /data --console-address ":9001"
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  meilidata:
  miniodata:
```

`.env.example`:
```env
# Database
DATABASE_URL=postgresql://cms:cms_dev_password@localhost:5432/headless_cms

# Redis
REDIS_URL=redis://localhost:6379

# Meilisearch
MEILI_URL=http://localhost:7700
MEILI_MASTER_KEY=cms_dev_meili_key

# MinIO (S3-compatible)
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=cms_dev_minio
S3_SECRET_KEY=cms_dev_minio_password
S3_BUCKET=cms-media
S3_REGION=us-east-1

# Auth
JWT_SECRET=dev-jwt-secret-change-in-production
SESSION_SECRET=dev-session-secret-change-in-production

# AI (optional — features degrade gracefully if absent)
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# Server
PORT=3000
HOST=0.0.0.0
NODE_ENV=development
LOG_LEVEL=debug
```

**Testing**:
- `Integration: docker compose up -d starts all services healthy within 30 seconds`
- `Integration: PostgreSQL accepts connections and has pgvector extension available`
- `Integration: Redis responds to PING`
- `Integration: Meilisearch health endpoint returns 200`
- `Integration: MinIO console accessible on port 9001`

---

#### 1.3 — Fastify Server Bootstrap and Configuration

**What**: Create the core Fastify server with config validation, CORS, request logging, and health check endpoint.

**Design**:

```typescript
// packages/core/src/config.ts
import { z } from "zod";

export const configSchema = z.object({
  database: z.object({
    url: z.string().url(),
  }),
  redis: z.object({
    url: z.string().url(),
  }),
  meili: z.object({
    url: z.string().url(),
    masterKey: z.string().min(1),
  }),
  s3: z.object({
    endpoint: z.string().url(),
    accessKey: z.string().min(1),
    secretKey: z.string().min(1),
    bucket: z.string().default("cms-media"),
    region: z.string().default("us-east-1"),
  }),
  auth: z.object({
    jwtSecret: z.string().min(32),
    sessionSecret: z.string().min(32),
  }),
  ai: z.object({
    anthropicApiKey: z.string().optional(),
    openaiApiKey: z.string().optional(),
  }),
  server: z.object({
    port: z.coerce.number().default(3000),
    host: z.string().default("0.0.0.0"),
    nodeEnv: z.enum(["development", "production", "test"]).default("development"),
    logLevel: z.enum(["debug", "info", "warn", "error"]).default("info"),
  }),
});

export type Config = z.infer<typeof configSchema>;

export function loadConfig(): Config {
  return configSchema.parse({
    database: { url: process.env.DATABASE_URL },
    redis: { url: process.env.REDIS_URL },
    meili: { url: process.env.MEILI_URL, masterKey: process.env.MEILI_MASTER_KEY },
    s3: {
      endpoint: process.env.S3_ENDPOINT,
      accessKey: process.env.S3_ACCESS_KEY,
      secretKey: process.env.S3_SECRET_KEY,
      bucket: process.env.S3_BUCKET,
      region: process.env.S3_REGION,
    },
    auth: {
      jwtSecret: process.env.JWT_SECRET,
      sessionSecret: process.env.SESSION_SECRET,
    },
    ai: {
      anthropicApiKey: process.env.ANTHROPIC_API_KEY,
      openaiApiKey: process.env.OPENAI_API_KEY,
    },
    server: {
      port: process.env.PORT,
      host: process.env.HOST,
      nodeEnv: process.env.NODE_ENV,
      logLevel: process.env.LOG_LEVEL,
    },
  });
}
```

```typescript
// packages/core/src/index.ts
import Fastify from "fastify";
import cors from "@fastify/cors";
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";
import { loadConfig } from "./config.js";

export async function buildServer() {
  const config = loadConfig();

  const server = Fastify({
    logger: {
      level: config.server.logLevel,
      transport:
        config.server.nodeEnv === "development"
          ? { target: "pino-pretty" }
          : undefined,
    },
  });

  await server.register(cors, { origin: true, credentials: true });

  await server.register(swagger, {
    openapi: {
      info: {
        title: "Headless CMS API",
        version: "1.0.0",
        description: "API-first content management with AI enrichment",
      },
      servers: [{ url: `http://${config.server.host}:${config.server.port}` }],
    },
  });

  await server.register(swaggerUi, { routePrefix: "/docs" });

  server.get("/health", {
    schema: {
      response: {
        200: {
          type: "object",
          properties: {
            status: { type: "string", enum: ["ok"] },
            timestamp: { type: "string", format: "date-time" },
            version: { type: "string" },
          },
        },
      },
    },
    handler: async () => ({
      status: "ok" as const,
      timestamp: new Date().toISOString(),
      version: "1.0.0",
    }),
  });

  return server;
}
```

**Testing**:
- `Unit: loadConfig() with valid env vars returns typed Config object`
- `Unit: loadConfig() with missing DATABASE_URL throws ZodError with field name`
- `Unit: loadConfig() with invalid PORT (non-numeric) throws ZodError`
- `Integration: GET /health returns 200 with {status: "ok", timestamp, version}`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: GET /docs/json returns OpenAPI 3.1 spec`
- `Unit: CORS headers present on cross-origin request`

---

#### 1.4 — Database Schema and Migrations (System Tables)

**What**: Define Drizzle ORM schema for all system tables from Data Model Suggestion 2 (hybrid relational + JSONB) and create initial migration.

**Design**:

The schema follows Data Model Suggestion 2 (hybrid JSONB) for content storage, augmented with graph tables from Data Model Suggestion 4 for the AI content graph (introduced in Phase 9). System tables are conventional relational.

```typescript
// packages/core/src/db/schema.ts
import {
  pgTable, uuid, varchar, text, boolean, integer, bigint,
  timestamp, jsonb, inet, unique, index, pgEnum,
} from "drizzle-orm/pg-core";

// ── Enums ──────────────────────────────────────────────
export const contentTypeKind = pgEnum("content_type_kind", ["collection", "single"]);
export const entryStatus = pgEnum("entry_status", [
  "draft", "review", "approved", "published", "archived",
]);
export const tokenType = pgEnum("token_type", ["personal", "api", "preview"]);
export const permissionAction = pgEnum("permission_action", [
  "create", "read", "update", "delete", "publish", "unpublish",
]);

// ── Roles ──────────────────────────────────────────────
export const roles = pgTable("roles", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 128 }).notNull().unique(),
  description: text("description"),
  isSystem: boolean("is_system").notNull().default(false),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Users ──────────────────────────────────────────────
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  passwordHash: varchar("password_hash", { length: 255 }).notNull(),
  firstName: varchar("first_name", { length: 128 }),
  lastName: varchar("last_name", { length: 128 }),
  avatarUrl: text("avatar_url"),
  roleId: uuid("role_id").notNull().references(() => roles.id),
  isActive: boolean("is_active").notNull().default(true),
  preferences: jsonb("preferences").default({}),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Permissions ────────────────────────────────────────
export const permissions = pgTable("permissions", {
  id: uuid("id").primaryKey().defaultRandom(),
  roleId: uuid("role_id").notNull().references(() => roles.id, { onDelete: "cascade" }),
  contentTypeSlug: varchar("content_type_slug", { length: 128 }),
  action: permissionAction("action").notNull(),
  conditions: jsonb("conditions"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  unique().on(table.roleId, table.contentTypeSlug, table.action),
]);

// ── API Tokens ─────────────────────────────────────────
export const apiTokens = pgTable("api_tokens", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  tokenHash: varchar("token_hash", { length: 255 }).notNull().unique(),
  userId: uuid("user_id").references(() => users.id, { onDelete: "set null" }),
  tokenType: tokenType("token_type").notNull().default("personal"),
  scopes: jsonb("scopes"),
  expiresAt: timestamp("expires_at", { withTimezone: true }),
  lastUsedAt: timestamp("last_used_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Locales ────────────────────────────────────────────
export const locales = pgTable("locales", {
  id: uuid("id").primaryKey().defaultRandom(),
  code: varchar("code", { length: 10 }).notNull().unique(),
  name: varchar("name", { length: 128 }).notNull(),
  isDefault: boolean("is_default").notNull().default(false),
  isActive: boolean("is_active").notNull().default(true),
  fallbackLocaleCode: varchar("fallback_locale_code", { length: 10 }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Content Types ──────────────────────────────────────
export const contentTypes = pgTable("content_types", {
  id: uuid("id").primaryKey().defaultRandom(),
  slug: varchar("slug", { length: 128 }).notNull().unique(),
  displayName: varchar("display_name", { length: 255 }).notNull(),
  description: text("description"),
  kind: contentTypeKind("kind").notNull().default("collection"),
  apiId: varchar("api_id", { length: 128 }).notNull().unique(),
  schema: jsonb("schema").notNull(),  // JSON Schema defining fields
  draftEnabled: boolean("draft_enabled").notNull().default(true),
  versioningEnabled: boolean("versioning_enabled").notNull().default(true),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Entry Documents (locale grouping) ──────────────────
export const entryDocuments = pgTable("entry_documents", {
  id: uuid("id").primaryKey().defaultRandom(),
  contentTypeId: uuid("content_type_id").notNull().references(() => contentTypes.id),
  contentTypeSlug: varchar("content_type_slug", { length: 128 }).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Entries ────────────────────────────────────────────
export const entries = pgTable("entries", {
  id: uuid("id").primaryKey().defaultRandom(),
  contentTypeId: uuid("content_type_id").notNull().references(() => contentTypes.id),
  contentTypeSlug: varchar("content_type_slug", { length: 128 }).notNull(),
  documentId: uuid("document_id").references(() => entryDocuments.id),
  localeCode: varchar("locale_code", { length: 10 }).notNull().default("en"),
  status: entryStatus("status").notNull().default("draft"),
  data: jsonb("data").notNull().default({}),
  version: integer("version").notNull().default(1),
  publishedAt: timestamp("published_at", { withTimezone: true }),
  scheduledPublishAt: timestamp("scheduled_publish_at", { withTimezone: true }),
  createdBy: uuid("created_by").references(() => users.id),
  updatedBy: uuid("updated_by").references(() => users.id),
  publishedBy: uuid("published_by").references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_entries_content_type").on(table.contentTypeSlug),
  index("idx_entries_status").on(table.status),
  index("idx_entries_locale").on(table.localeCode),
  index("idx_entries_api_lookup").on(table.contentTypeSlug, table.status, table.localeCode),
  index("idx_entries_document").on(table.documentId),
  unique("idx_entries_document_locale").on(table.documentId, table.localeCode),
]);

// ── Entry Relationships ────────────────────────────────
export const entryRelationships = pgTable("entry_relationships", {
  id: uuid("id").primaryKey().defaultRandom(),
  sourceEntryId: uuid("source_entry_id").notNull().references(() => entries.id, { onDelete: "cascade" }),
  sourceField: varchar("source_field", { length: 128 }).notNull(),
  targetEntryId: uuid("target_entry_id").notNull().references(() => entries.id, { onDelete: "cascade" }),
  targetContentTypeSlug: varchar("target_content_type_slug", { length: 128 }).notNull(),
  position: integer("position").notNull().default(0),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_entry_rels_source").on(table.sourceEntryId),
  index("idx_entry_rels_target").on(table.targetEntryId),
]);

// ── Entry Versions ─────────────────────────────────────
export const entryVersions = pgTable("entry_versions", {
  id: uuid("id").primaryKey().defaultRandom(),
  entryId: uuid("entry_id").notNull().references(() => entries.id, { onDelete: "cascade" }),
  versionNumber: integer("version_number").notNull(),
  data: jsonb("data").notNull(),
  status: entryStatus("status").notNull(),
  changedBy: uuid("changed_by").references(() => users.id),
  changeSummary: text("change_summary"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  unique().on(table.entryId, table.versionNumber),
  index("idx_entry_versions_entry").on(table.entryId),
]);

// ── Media ──────────────────────────────────────────────
export const media = pgTable("media", {
  id: uuid("id").primaryKey().defaultRandom(),
  filename: varchar("filename", { length: 512 }).notNull(),
  originalFilename: varchar("original_filename", { length: 512 }).notNull(),
  mimeType: varchar("mime_type", { length: 128 }).notNull(),
  fileSize: bigint("file_size", { mode: "number" }).notNull(),
  width: integer("width"),
  height: integer("height"),
  altText: text("alt_text"),
  caption: text("caption"),
  storagePath: text("storage_path").notNull(),
  storageProvider: varchar("storage_provider", { length: 50 }).notNull().default("local"),
  focalPoint: jsonb("focal_point"),
  metadata: jsonb("metadata"),
  folderPath: text("folder_path").default("/"),
  uploadedBy: uuid("uploaded_by").references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_media_mime").on(table.mimeType),
  index("idx_media_folder").on(table.folderPath),
]);

// ── Webhooks ───────────────────────────────────────────
export const webhooks = pgTable("webhooks", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  url: text("url").notNull(),
  events: text("events").array().notNull(),
  contentTypeSlugs: text("content_type_slugs").array(),
  headers: jsonb("headers"),
  isActive: boolean("is_active").notNull().default(true),
  secret: varchar("secret", { length: 255 }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const webhookDeliveries = pgTable("webhook_deliveries", {
  id: uuid("id").primaryKey().defaultRandom(),
  webhookId: uuid("webhook_id").notNull().references(() => webhooks.id, { onDelete: "cascade" }),
  eventType: varchar("event_type", { length: 128 }).notNull(),
  payload: jsonb("payload").notNull(),
  responseStatus: integer("response_status"),
  responseBody: text("response_body"),
  attempt: integer("attempt").notNull().default(1),
  deliveredAt: timestamp("delivered_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_webhook_deliveries_webhook").on(table.webhookId),
]);

// ── Workflow Definitions ───────────────────────────────
export const workflowDefinitions = pgTable("workflow_definitions", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  contentTypeSlug: varchar("content_type_slug", { length: 128 }),
  stages: jsonb("stages").notNull(),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// ── Audit Log ──────────────────────────────────────────
export const auditLog = pgTable("audit_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").references(() => users.id),
  action: varchar("action", { length: 50 }).notNull(),
  resourceType: varchar("resource_type", { length: 128 }).notNull(),
  resourceId: uuid("resource_id"),
  contentTypeSlug: varchar("content_type_slug", { length: 128 }),
  diff: jsonb("diff"),
  ipAddress: inet("ip_address"),
  userAgent: text("user_agent"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_audit_user").on(table.userId),
  index("idx_audit_resource").on(table.resourceType, table.resourceId),
  index("idx_audit_created").on(table.createdAt),
]);
```

Database seed script:
```typescript
// packages/core/src/db/seed.ts
export async function seed(db: Database) {
  // Default roles
  await db.insert(roles).values([
    { name: "admin", description: "Full system access", isSystem: true },
    { name: "editor", description: "Create, edit, and publish content", isSystem: true },
    { name: "author", description: "Create and edit own content", isSystem: true },
    { name: "viewer", description: "Read-only access", isSystem: true },
  ]).onConflictDoNothing();

  // Default locale
  await db.insert(locales).values([
    { code: "en", name: "English", isDefault: true, isActive: true },
  ]).onConflictDoNothing();

  // Default admin user (password: "admin" — hashed)
  // In production, this is overridden by env vars or first-run setup
  const adminRole = await db.select().from(roles).where(eq(roles.name, "admin")).limit(1);
  await db.insert(users).values({
    email: "admin@headless-cms.local",
    passwordHash: await hashPassword("admin"),
    firstName: "Admin",
    lastName: "User",
    roleId: adminRole[0].id,
  }).onConflictDoNothing();

  // Default workflow
  await db.insert(workflowDefinitions).values({
    name: "Default Publishing Workflow",
    stages: [
      { name: "draft", color: "#6B7280", label: "Draft" },
      { name: "review", color: "#F59E0B", label: "In Review" },
      { name: "approved", color: "#10B981", label: "Approved" },
      { name: "published", color: "#3B82F6", label: "Published" },
    ],
    isActive: true,
  }).onConflictDoNothing();
}
```

**Testing**:
- `Unit: Drizzle schema compiles without type errors`
- `Integration: drizzle-kit generate produces valid SQL migration`
- `Integration: drizzle-kit migrate applies migration to test database without errors`
- `Integration: seed() creates 4 default roles, 1 default locale, 1 admin user, 1 workflow`
- `Integration: seed() is idempotent — running twice does not create duplicates`
- `Unit: all foreign key constraints reference valid tables`
- `Integration: inserting entry with invalid content_type_id raises foreign key violation`

---

## Phase 2: Authentication and Access Control

### Purpose
Implement user authentication (login, registration, sessions), API token management, and role-based access control (RBAC). After this phase, the API is secured — every endpoint requires authentication and checks permissions before allowing access.

### Tasks

#### 2.1 — User Authentication (Session-Based + JWT)

**What**: Implement login, logout, session management, and JWT token generation for API access using Lucia Auth.

**Design**:

```typescript
// packages/core/src/modules/auth/auth.types.ts
export interface LoginRequest {
  email: string;
  password: string;
}

export interface LoginResponse {
  user: {
    id: string;
    email: string;
    firstName: string | null;
    lastName: string | null;
    role: { id: string; name: string };
  };
  sessionToken: string;
  expiresAt: string; // ISO 8601
}

export interface RegisterRequest {
  email: string;
  password: string;
  firstName?: string;
  lastName?: string;
}
```

API endpoints:
```
POST   /api/auth/login          — Authenticate, return session token
POST   /api/auth/logout         — Invalidate session
POST   /api/auth/register       — Create new user (admin-only or first-run)
GET    /api/auth/me             — Return current user profile
PATCH  /api/auth/me             — Update current user profile
POST   /api/auth/change-password — Change password for current user
```

Password hashing: Argon2id via `@node-rs/argon2` (RFC 9106 compliant).

Session storage: Redis-backed sessions with 30-day expiry. Session token is a cryptographically random 32-byte hex string.

```typescript
// packages/core/src/modules/auth/auth.service.ts
export class AuthService {
  async login(email: string, password: string): Promise<LoginResponse>;
  async logout(sessionToken: string): Promise<void>;
  async register(data: RegisterRequest, createdBy?: string): Promise<User>;
  async validateSession(token: string): Promise<Session | null>;
  async changePassword(userId: string, oldPassword: string, newPassword: string): Promise<void>;
  async getProfile(userId: string): Promise<UserProfile>;
  async updateProfile(userId: string, data: Partial<UserProfile>): Promise<UserProfile>;
}
```

**Testing**:
- `Unit: valid email + correct password → session token returned, user object included`
- `Unit: valid email + wrong password → 401 Unauthorized`
- `Unit: non-existent email → 401 Unauthorized (no information leakage)`
- `Unit: session token validates within expiry → returns user`
- `Unit: expired session token → returns null`
- `Unit: logout invalidates session immediately`
- `Unit: password hashed with Argon2id, never stored in plaintext`
- `Unit: change password with wrong old password → 400 error`
- `Integration: login → GET /api/auth/me with session token → returns user profile`
- `Integration: register with duplicate email → 409 Conflict`

---

#### 2.2 — API Token Management

**What**: CRUD for API tokens (personal, API, preview) with scoped permissions and expiry.

**Design**:

```typescript
// packages/core/src/modules/auth/token.types.ts
export interface CreateTokenRequest {
  name: string;
  tokenType: "personal" | "api" | "preview";
  scopes?: TokenScope[];
  expiresAt?: string; // ISO 8601
}

export interface TokenScope {
  contentTypeSlug?: string; // null = all content types
  actions: ("read" | "create" | "update" | "delete" | "publish")[];
}

export interface TokenResponse {
  id: string;
  name: string;
  tokenType: string;
  token: string; // Only returned on creation
  scopes: TokenScope[] | null;
  expiresAt: string | null;
  lastUsedAt: string | null;
  createdAt: string;
}
```

API endpoints:
```
GET    /api/tokens              — List tokens for current user
POST   /api/tokens              — Create new token (returns plaintext token once)
DELETE /api/tokens/:id          — Revoke a token
```

Token format: `cms_<type>_<32 random bytes in hex>` (e.g., `cms_api_a1b2c3d4...`). Stored as SHA-256 hash. Plaintext returned only on creation.

Authentication header: `Authorization: Bearer cms_api_...` or `X-API-Key: cms_api_...`.

**Testing**:
- `Unit: create token → returns plaintext token and stores SHA-256 hash`
- `Unit: token plaintext not retrievable after creation`
- `Unit: API request with valid token → authenticated`
- `Unit: API request with expired token → 401`
- `Unit: API request with revoked token → 401`
- `Unit: preview token can only read draft content`
- `Unit: API token scoped to "blog_post" cannot read "page" entries`
- `Integration: list tokens returns only tokens for current user`

---

#### 2.3 — RBAC Middleware and Permission Checks

**What**: Fastify preHandler middleware that checks role permissions on every route, supporting content-type-level and action-level granularity.

**Design**:

```typescript
// packages/core/src/modules/auth/rbac.middleware.ts
export interface PermissionCheck {
  action: "create" | "read" | "update" | "delete" | "publish" | "unpublish";
  contentTypeSlug?: string;
}

/**
 * Fastify preHandler that validates the current user's role
 * has the required permission for the requested action
 * on the specified content type.
 */
export function requirePermission(check: PermissionCheck): FastifyPreHandlerAsync {
  return async (request, reply) => {
    const user = request.user; // set by auth middleware
    if (!user) return reply.status(401).send({ error: "Authentication required" });

    const hasPermission = await checkPermission(user.roleId, check);
    if (!hasPermission) {
      return reply.status(403).send({
        error: "Forbidden",
        message: `Missing permission: ${check.action} on ${check.contentTypeSlug ?? "global"}`,
      });
    }
  };
}

// Usage in route:
// server.get("/api/entries/:contentType", {
//   preHandler: [authenticate, requirePermission({ action: "read" })],
//   handler: listEntries,
// });
```

Error responses follow RFC 7807 (Problem Details for HTTP APIs):
```typescript
export interface ProblemDetail {
  type: string;    // URI reference identifying the problem type
  title: string;   // Human-readable summary
  status: number;  // HTTP status code
  detail?: string; // Human-readable explanation
  instance?: string; // URI of the specific occurrence
}
```

**Testing**:
- `Unit: admin role has all permissions on all content types`
- `Unit: editor role can create, read, update, publish but not delete`
- `Unit: author role can only modify own entries`
- `Unit: viewer role can only read published content`
- `Unit: custom role with scoped permission on "blog_post" cannot access "page"`
- `Integration: unauthenticated request → 401 with RFC 7807 body`
- `Integration: authenticated request without permission → 403 with RFC 7807 body`
- `Integration: admin accessing any endpoint → 200`
- `Unit: permission check caches role permissions for duration of request`

---

## Phase 3: Content Type Management and Schema Registry

### Purpose
Implement the content type system — the ability to define, validate, and manage content type schemas (field definitions as JSON Schema). After this phase, administrators can create content types through the API and the system validates entry data against the defined schema.

### Tasks

#### 3.1 — Content Type CRUD API

**What**: REST endpoints for creating, reading, updating, and deleting content type definitions, including JSON Schema validation of the schema definition itself.

**Design**:

```typescript
// packages/core/src/modules/content-type/content-type.types.ts

/** JSON Schema extension for CMS-specific field metadata */
export interface FieldDefinition {
  type: "string" | "number" | "integer" | "boolean" | "array" | "object";
  format?: "richtext" | "markdown" | "slug" | "email" | "url" | "date" | "datetime" | "uuid" | "media";
  localizable?: boolean;
  description?: string;
  default?: unknown;
  // String constraints
  minLength?: number;
  maxLength?: number;
  pattern?: string;
  // Number constraints
  minimum?: number;
  maximum?: number;
  // Array constraints
  items?: FieldDefinition;
  minItems?: number;
  maxItems?: number;
  // Relationship metadata
  "x-relationship"?: {
    target: string; // content type slug
    type: "one_to_one" | "one_to_many" | "many_to_one" | "many_to_many";
  };
  // Select/enum
  enum?: string[];
  // Nested object
  properties?: Record<string, FieldDefinition>;
  required?: string[];
}

export interface ContentTypeSchema {
  type: "object";
  properties: Record<string, FieldDefinition>;
  required?: string[];
}

export interface CreateContentTypeRequest {
  slug: string;         // URL-safe identifier: /^[a-z][a-z0-9_]{1,126}$/
  displayName: string;
  description?: string;
  kind: "collection" | "single";
  schema: ContentTypeSchema;
  draftEnabled?: boolean;
  versioningEnabled?: boolean;
}

export interface ContentTypeResponse {
  id: string;
  slug: string;
  displayName: string;
  description: string | null;
  kind: "collection" | "single";
  apiId: string;
  schema: ContentTypeSchema;
  draftEnabled: boolean;
  versioningEnabled: boolean;
  createdAt: string;
  updatedAt: string;
}
```

API endpoints:
```
GET    /api/content-types            — List all content types
POST   /api/content-types            — Create content type
GET    /api/content-types/:slug      — Get content type by slug
PATCH  /api/content-types/:slug      — Update content type schema
DELETE /api/content-types/:slug      — Delete content type (fails if entries exist)
```

Schema meta-validation: The `schema` field in CreateContentTypeRequest is itself validated against a JSON Schema meta-schema that enforces CMS-specific constraints (valid field types, valid relationship targets, etc.).

**Testing**:
- `Unit: create content type with valid schema → 201 with generated apiId`
- `Unit: create content type with duplicate slug → 409 Conflict`
- `Unit: create content type with invalid slug (starts with number) → 400`
- `Unit: create content type with invalid field type → 400 with field path`
- `Unit: create content type with relationship to non-existent target → 400`
- `Unit: update content type schema adds new field → 200, existing entries unaffected`
- `Unit: delete content type with existing entries → 409 Conflict`
- `Unit: delete content type with no entries → 204`
- `Integration: create "blog_post" → GET /api/content-types/blog_post returns definition`
- `Unit: apiId auto-generated from slug as camelCase (blog_post → blogPost)`
- `Fixture: test with 5 pre-defined content types (blog_post, page, category, author, tag)`

---

#### 3.2 — Entry Data Validation Engine

**What**: A runtime validator that checks entry JSONB data against the content type's JSON Schema definition, supporting CMS-specific formats and relationship validation.

**Design**:

```typescript
// packages/core/src/modules/content-type/schema-validator.ts
import Ajv from "ajv";
import addFormats from "ajv-formats";

export class SchemaValidator {
  private ajv: Ajv;

  constructor() {
    this.ajv = new Ajv({
      allErrors: true,
      coerceTypes: false,
      useDefaults: true,
    });
    addFormats(this.ajv);
    this.registerCmsFormats();
  }

  /**
   * Validate entry data against a content type schema.
   * Returns { valid: true } or { valid: false, errors: ValidationError[] }
   */
  validate(
    schema: ContentTypeSchema,
    data: Record<string, unknown>,
    options?: { partial?: boolean } // partial = true for PATCH updates
  ): ValidationResult;

  /**
   * Extract localizable field paths from a schema.
   * Used to split entry data into locale-specific and locale-independent fields.
   */
  getLocalizableFields(schema: ContentTypeSchema): string[];

  /**
   * Extract relationship fields from a schema.
   * Used to maintain the entry_relationships table.
   */
  getRelationshipFields(schema: ContentTypeSchema): RelationshipFieldInfo[];

  private registerCmsFormats(): void {
    // Register CMS-specific formats:
    // "richtext" — validates Portable Text or CommonMark
    // "slug" — validates /^[a-z0-9]+(?:-[a-z0-9]+)*$/
    // "media" — validates UUID referencing media table
  }
}

export interface ValidationResult {
  valid: boolean;
  errors?: ValidationError[];
}

export interface ValidationError {
  field: string;       // JSON path e.g. "/title" or "/metadata/seo_title"
  message: string;
  keyword: string;     // AJV keyword e.g. "required", "type", "maxLength"
}
```

**Testing**:
- `Unit: valid data against schema → { valid: true }`
- `Unit: missing required field → error with field path and "required" keyword`
- `Unit: wrong type (number instead of string) → error with "type" keyword`
- `Unit: string exceeding maxLength → error with "maxLength" keyword`
- `Unit: invalid slug format ("Hello World") → error with "format" keyword`
- `Unit: valid richtext (CommonMark string) → passes`
- `Unit: partial validation (PATCH) allows missing required fields`
- `Unit: nested object validation works for metadata.seo_title`
- `Unit: array field with minItems/maxItems enforced`
- `Unit: getLocalizableFields returns only fields with localizable: true`
- `Unit: getRelationshipFields returns target slug and relation type`

---

## Phase 4: Content Entry CRUD and Publishing

### Purpose
Implement the core content management operations — creating, reading, updating, deleting, and publishing content entries. This is the heart of the CMS. After this phase, users can manage content through the REST API with full validation, versioning, and status lifecycle.

### Tasks

#### 4.1 — Entry CRUD Operations

**What**: REST endpoints for creating, listing, reading, updating, and deleting content entries with JSONB storage, pagination, filtering, and sorting.

**Design**:

```typescript
// packages/core/src/modules/entry/entry.types.ts
export interface CreateEntryRequest {
  data: Record<string, unknown>;
  localeCode?: string;   // defaults to default locale
  status?: "draft";      // new entries always start as draft
}

export interface UpdateEntryRequest {
  data: Record<string, unknown>; // partial update — merged with existing
}

export interface EntryResponse {
  id: string;
  contentType: string;   // slug
  documentId: string;
  localeCode: string;
  status: "draft" | "review" | "approved" | "published" | "archived";
  data: Record<string, unknown>;
  version: number;
  publishedAt: string | null;
  createdBy: string;
  updatedBy: string;
  createdAt: string;
  updatedAt: string;
}

export interface EntryListResponse {
  data: EntryResponse[];
  meta: {
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
  };
}

export interface EntryListQuery {
  page?: number;         // default 1
  pageSize?: number;     // default 25, max 100
  sort?: string;         // field name, prefix with - for desc (e.g. "-createdAt")
  status?: string;       // filter by status
  locale?: string;       // filter by locale
  filters?: Record<string, unknown>; // JSONB containment filters on data
}
```

API endpoints:
```
GET    /api/content/:contentType/entries         — List entries (paginated, filtered)
POST   /api/content/:contentType/entries         — Create entry
GET    /api/content/:contentType/entries/:id      — Get entry by ID
PATCH  /api/content/:contentType/entries/:id      — Update entry (partial)
DELETE /api/content/:contentType/entries/:id      — Delete entry (soft or hard)
```

Entry service:
```typescript
// packages/core/src/modules/entry/entry.service.ts
export class EntryService {
  async create(contentTypeSlug: string, data: CreateEntryRequest, userId: string): Promise<EntryResponse>;
  async findById(contentTypeSlug: string, id: string, locale?: string): Promise<EntryResponse | null>;
  async list(contentTypeSlug: string, query: EntryListQuery): Promise<EntryListResponse>;
  async update(contentTypeSlug: string, id: string, data: UpdateEntryRequest, userId: string): Promise<EntryResponse>;
  async delete(contentTypeSlug: string, id: string, userId: string): Promise<void>;
}
```

On create: validate data against content type schema, create entry_document for locale grouping, extract and store relationships in entry_relationships table.

On update: validate partial data, merge with existing data, increment version, create version snapshot, update relationships.

**Testing**:
- `Unit: create entry with valid data → 201 with entry response`
- `Unit: create entry with invalid data → 400 with validation errors`
- `Unit: create entry for non-existent content type → 404`
- `Unit: list entries with pagination → correct total, page, pageSize`
- `Unit: list entries with status filter → only matching entries returned`
- `Unit: list entries with JSONB filter (tags contains "javascript") → correct results`
- `Unit: list entries sorted by createdAt desc → correct ordering`
- `Unit: update entry merges data (not replaces)`
- `Unit: update entry increments version number`
- `Unit: delete entry removes from database`
- `Unit: get entry by ID returns full data with relationships resolved`
- `Integration: create 100 entries → list with pageSize=25 returns 4 pages`
- `Fixture: pre-created blog_post content type with 10 test entries`

---

#### 4.2 — Publishing Lifecycle and Status Transitions

**What**: Implement the Draft → Review → Approved → Published → Archived status lifecycle with validation of legal transitions, publish/unpublish operations, and scheduled publishing.

**Design**:

State machine for entry status:
```
         ┌─────────────────────────────────────┐
         │                                     ▼
  draft ──→ review ──→ approved ──→ published ──→ archived
    ▲         │            │           │
    │         │            │           │
    └─────────┘            └───────────┘
   (reject)              (unpublish → draft)
```

Valid transitions:
```typescript
// packages/core/src/modules/entry/status-machine.ts
export const VALID_TRANSITIONS: Record<EntryStatus, EntryStatus[]> = {
  draft: ["review"],
  review: ["draft", "approved"],
  approved: ["draft", "published"],
  published: ["draft", "archived"],
  archived: ["draft"],
};

export function canTransition(from: EntryStatus, to: EntryStatus): boolean {
  return VALID_TRANSITIONS[from]?.includes(to) ?? false;
}
```

API endpoints:
```
POST   /api/content/:contentType/entries/:id/publish     — Publish entry (approved → published)
POST   /api/content/:contentType/entries/:id/unpublish   — Unpublish (published → draft)
PATCH  /api/content/:contentType/entries/:id/status       — Change status { status: "review" }
POST   /api/content/:contentType/entries/:id/schedule     — Schedule future publish { publishAt: "ISO8601" }
```

Scheduled publishing: BullMQ delayed job that fires at the scheduled time and transitions the entry to "published". Uses Redis for reliable scheduling.

**Testing**:
- `Unit: draft → review → approved → published is valid path`
- `Unit: draft → published directly is invalid → 400`
- `Unit: published → archived is valid`
- `Unit: archived → draft is valid (re-open for editing)`
- `Unit: publish sets publishedAt timestamp and publishedBy user`
- `Unit: unpublish clears publishedAt and resets to draft`
- `Unit: schedule future publish creates BullMQ delayed job`
- `Unit: scheduled publish job transitions entry at correct time`
- `Unit: cancelling scheduled publish removes delayed job`
- `Integration: full lifecycle: create → review → approve → publish → verify published`
- `Integration: publish requires "publish" permission on role`

---

#### 4.3 — Content Versioning

**What**: Automatic version snapshots on every update, with version history listing and version restore capability.

**Design**:

```typescript
// packages/core/src/modules/version/version.types.ts
export interface VersionResponse {
  id: string;
  entryId: string;
  versionNumber: number;
  data: Record<string, unknown>;
  status: string;
  changedBy: { id: string; email: string; firstName: string | null };
  changeSummary: string | null;
  createdAt: string;
}

export interface VersionDiff {
  field: string;
  oldValue: unknown;
  newValue: unknown;
}
```

API endpoints:
```
GET    /api/content/:contentType/entries/:id/versions       — List versions
GET    /api/content/:contentType/entries/:id/versions/:num  — Get specific version
POST   /api/content/:contentType/entries/:id/versions/:num/restore — Restore version
GET    /api/content/:contentType/entries/:id/versions/:a/diff/:b  — Diff two versions
```

On every entry update, the pre-update state is snapshotted into entry_versions with an incremented version_number. Restore creates a new version with the restored data (does not overwrite history).

**Testing**:
- `Unit: create entry → version 1 snapshot created`
- `Unit: update entry → version 2 snapshot created, entry.version = 2`
- `Unit: 5 updates → 5 versions in history, ordered by version_number`
- `Unit: restore version 2 on entry at version 5 → entry now at version 6 with version 2 data`
- `Unit: diff version 1 and 3 → list of changed fields with old/new values`
- `Unit: version snapshot contains full entry data (not just diff)`
- `Integration: list versions returns versions newest-first with user info`

---

## Phase 5: Media Management

### Purpose
Implement file upload, storage (S3-compatible), image transformation metadata, and the media library API. After this phase, users can upload images and files, organize them in folders, and reference them from content entries.

### Tasks

#### 5.1 — File Upload and S3 Storage

**What**: Multi-part file upload endpoint with S3 storage, MIME type validation, file size limits, and pre-signed URL generation for direct uploads.

**Design**:

```typescript
// packages/core/src/modules/media/media.types.ts
export interface UploadMediaRequest {
  file: MultipartFile;
  altText?: string;
  caption?: string;
  folderPath?: string;  // default "/"
}

export interface MediaResponse {
  id: string;
  filename: string;
  originalFilename: string;
  mimeType: string;
  fileSize: number;
  width: number | null;
  height: number | null;
  altText: string | null;
  caption: string | null;
  url: string;          // CDN or S3 URL for the file
  folderPath: string;
  metadata: Record<string, unknown> | null;
  createdAt: string;
  updatedAt: string;
}

export interface PresignedUploadResponse {
  uploadUrl: string;    // Pre-signed S3 PUT URL
  mediaId: string;      // ID to use for confirming upload
  expiresAt: string;    // When the pre-signed URL expires
}
```

API endpoints:
```
POST   /api/media/upload          — Upload file (multipart/form-data)
POST   /api/media/presign         — Get pre-signed URL for direct upload
POST   /api/media/:id/confirm     — Confirm direct upload completed
GET    /api/media                 — List media (paginated, filterable by MIME, folder)
GET    /api/media/:id             — Get media by ID
PATCH  /api/media/:id             — Update alt text, caption, folder
DELETE /api/media/:id             — Delete media file and S3 object
GET    /api/media/folders         — List folder tree
POST   /api/media/folders         — Create folder
```

Configuration:
```typescript
const UPLOAD_LIMITS = {
  maxFileSize: 50 * 1024 * 1024, // 50 MB
  allowedMimeTypes: [
    "image/jpeg", "image/png", "image/gif", "image/webp", "image/svg+xml", "image/avif",
    "video/mp4", "video/webm",
    "application/pdf",
    "audio/mpeg", "audio/ogg", "audio/wav",
    "text/plain", "text/csv",
  ],
};
```

Storage path format: `/{year}/{month}/{uuid}.{ext}` (e.g., `/2026/05/a1b2c3d4.jpg`).

For images: extract width/height using `sharp` library. Store EXIF metadata in the `metadata` JSONB column.

**Testing**:
- `Unit: upload JPEG → 201 with media response including width/height`
- `Unit: upload PNG → stored in S3 with correct path format`
- `Unit: upload exceeding 50 MB → 413 Payload Too Large`
- `Unit: upload disallowed MIME type (.exe) → 415 Unsupported Media Type`
- `Unit: presign → returns valid pre-signed PUT URL`
- `Unit: delete media → removes from S3 and database`
- `Unit: list media with MIME filter → only images returned`
- `Unit: list media with folder filter → only matching folder`
- `Unit: update alt text → 200, alt_text column updated`
- `Integration: upload file → GET /api/media/:id returns file metadata with accessible URL`
- `Integration: create folder → folder appears in folder tree`
- `Unit: image dimensions extracted correctly for JPEG, PNG, WebP`

---

## Phase 6: Multi-Locale Content and Webhooks

### Purpose
Implement multi-locale content management (creating locale variants of entries, locale fallback chains) and webhook delivery (firing HTTP callbacks on content events). These are table-stakes features required by all headless CMS platforms.

### Tasks

#### 6.1 — Multi-Locale Entry Management

**What**: Support creating locale variants of entries grouped by document_id, locale-aware content delivery, and locale fallback chains.

**Design**:

```typescript
// packages/core/src/modules/locale/locale.types.ts
export interface CreateLocaleRequest {
  code: string;        // BCP 47: "en", "en-US", "fr-FR"
  name: string;
  isDefault?: boolean;
  fallbackLocaleCode?: string;
}

// packages/core/src/modules/entry/entry-locale.types.ts
export interface CreateLocaleVariantRequest {
  localeCode: string;
  data: Record<string, unknown>; // Only localizable fields
  copyFromLocale?: string;       // Copy non-localizable fields from source locale
}
```

API endpoints:
```
GET    /api/locales                                              — List active locales
POST   /api/locales                                              — Create locale
PATCH  /api/locales/:code                                        — Update locale
DELETE /api/locales/:code                                        — Delete locale (fails if entries exist)
POST   /api/content/:type/entries/:id/locales                    — Create locale variant
GET    /api/content/:type/entries/:id/locales                    — List all locale variants
GET    /api/content/:type/entries/:id/locales/:code              — Get specific locale variant
DELETE /api/content/:type/entries/:id/locales/:code              — Delete locale variant
```

Locale fallback: When requesting content in locale "fr-CA" and no entry exists, fall back to "fr", then to the default locale "en". The fallback chain is defined by the `fallback_locale_code` column.

When creating a locale variant, only localizable fields (marked in content type schema) are stored in the new entry row. Non-localizable fields are shared across all locale variants via the entry_documents grouping.

**Testing**:
- `Unit: create locale "fr" → 201 with locale response`
- `Unit: create locale variant "fr" for blog post → new entry row with same document_id`
- `Unit: list locale variants → returns all locales for entry with their statuses`
- `Unit: locale fallback "fr-CA" → "fr" → "en" when fr-CA doesn't exist`
- `Unit: delete locale with existing entries → 409`
- `Unit: only localizable fields differ between locale variants`
- `Unit: updating non-localizable field on one locale updates all variants`
- `Integration: create entry in "en" → create "fr" variant → GET with locale=fr returns French data`
- `Integration: GET with locale=fr-CA falls back to "fr" if fr-CA doesn't exist`

---

#### 6.2 — Webhook CRUD and Delivery

**What**: CRUD for webhook subscriptions and reliable HTTP delivery with retry logic, HMAC signing, and delivery logging.

**Design**:

```typescript
// packages/core/src/modules/webhook/webhook.types.ts
export interface CreateWebhookRequest {
  name: string;
  url: string;
  events: WebhookEvent[];
  contentTypeSlugs?: string[];  // null = all content types
  headers?: Record<string, string>;
  secret?: string;             // HMAC signing secret
}

export type WebhookEvent =
  | "entry.created" | "entry.updated" | "entry.published"
  | "entry.unpublished" | "entry.deleted"
  | "media.uploaded" | "media.deleted"
  | "content_type.created" | "content_type.updated" | "content_type.deleted";

export interface WebhookPayload {
  // CloudEvents envelope (CNCF standard)
  specversion: "1.0";
  id: string;           // unique event ID
  type: string;         // e.g. "cms.entry.published"
  source: string;       // CMS instance URL
  subject: string;      // e.g. "/content/blog_post/entries/abc123"
  time: string;         // ISO 8601
  datacontenttype: "application/json";
  data: {
    contentType: string;
    entryId: string;
    locale: string;
    status: string;
    entry: Record<string, unknown>; // full entry data
  };
}
```

API endpoints:
```
GET    /api/webhooks               — List webhooks
POST   /api/webhooks               — Create webhook
GET    /api/webhooks/:id           — Get webhook
PATCH  /api/webhooks/:id           — Update webhook
DELETE /api/webhooks/:id           — Delete webhook
GET    /api/webhooks/:id/deliveries — List recent deliveries for webhook
POST   /api/webhooks/:id/test       — Send test delivery
```

Delivery: BullMQ job with exponential backoff retry (3 attempts: immediate, 1 min, 5 min). HMAC-SHA256 signature in `X-CMS-Signature` header. Delivery results logged in webhook_deliveries table.

**Testing**:
- `Unit: create webhook → 201 with webhook response`
- `Unit: publish entry triggers "entry.published" webhook for matching content type`
- `Unit: webhook payload follows CloudEvents envelope structure`
- `Unit: HMAC signature matches expected SHA-256 hash`
- `Unit: failed delivery retries 3 times with exponential backoff`
- `Unit: webhook filtered to "blog_post" does not fire for "page" entries`
- `Unit: test delivery sends payload to webhook URL and logs result`
- `Integration (mocked HTTP): publish entry → webhook delivered to mock server`
- `Integration: delivery log contains response status and body`
- `Unit: disabled webhook does not fire`

---

## Phase 7: GraphQL API and Content Delivery API

### Purpose
Add the GraphQL API layer that dynamically generates a schema from content type definitions, and create the public-facing Content Delivery API (read-only, for frontend applications). After this phase, frontend developers can query published content via both REST and GraphQL.

### Tasks

#### 7.1 — Dynamic GraphQL Schema Generation

**What**: Generate a GraphQL schema at runtime from content type definitions, with types, queries, filters, and pagination matching the content type schemas.

**Design**:

```typescript
// packages/core/src/graphql/schema-generator.ts
export class GraphQLSchemaGenerator {
  /**
   * Generate a complete GraphQL schema from all content type definitions.
   * Called on server startup and when content types change.
   */
  generateSchema(contentTypes: ContentTypeDefinition[]): GraphQLSchema;

  /**
   * Generate GraphQL type for a single content type.
   * Maps JSON Schema field types to GraphQL scalar/object types.
   */
  private generateType(contentType: ContentTypeDefinition): GraphQLObjectType;

  /**
   * Generate query fields: list (with pagination/filters) and get-by-id.
   */
  private generateQueryFields(contentType: ContentTypeDefinition): GraphQLFieldConfigMap;
}
```

For a content type "blog_post" with fields title (string), body (richtext), publishedAt (datetime), and category (relationship to category), the generated schema is:

```graphql
type BlogPost {
  id: ID!
  title: String!
  body: String
  publishedAt: DateTime
  category: Category
  locale: String!
  status: EntryStatus!
  version: Int!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type BlogPostConnection {
  data: [BlogPost!]!
  meta: PaginationMeta!
}

type Query {
  blogPost(id: ID!, locale: String): BlogPost
  blogPosts(
    page: Int = 1
    pageSize: Int = 25
    sort: String
    status: EntryStatus
    locale: String
    where: BlogPostWhereInput
  ): BlogPostConnection!
}

input BlogPostWhereInput {
  title_contains: String
  title_eq: String
  category_eq: ID
  publishedAt_gt: DateTime
  publishedAt_lt: DateTime
}
```

**Testing**:
- `Unit: content type with 3 fields generates GraphQL type with 3 fields + system fields`
- `Unit: string field → GraphQL String type`
- `Unit: number field → GraphQL Float type`
- `Unit: boolean field → GraphQL Boolean type`
- `Unit: relationship field → GraphQL reference type with resolver`
- `Unit: array field → GraphQL list type`
- `Unit: query with pagination returns BlogPostConnection with meta`
- `Unit: query with where filter generates correct SQL conditions`
- `Integration: create content type → GraphQL schema includes new type`
- `Integration: delete content type → GraphQL schema removes type`
- `Integration: query blogPosts → returns published entries`
- `Unit: schema regenerates when content type updated`

---

#### 7.2 — Content Delivery API (Public Read-Only REST)

**What**: A separate, optimized read-only REST API for frontend applications to fetch published content, with CDN cache headers and preview token support.

**Design**:

```typescript
// Public delivery API — separate from management API
// Base path: /api/delivery/v1

// All responses include cache headers:
// Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=86400
// ETag: based on content hash
// Vary: Accept-Language
```

API endpoints:
```
GET /api/delivery/v1/:contentType               — List published entries
GET /api/delivery/v1/:contentType/:slug          — Get published entry by slug
GET /api/delivery/v1/:contentType/id/:id         — Get published entry by ID
```

Preview mode: When a valid preview token is provided via `?preview=true&token=cms_preview_...`, the delivery API returns draft content instead of only published content. Preview tokens are validated per-request and have configurable TTL.

Response format:
```typescript
export interface DeliveryResponse<T = Record<string, unknown>> {
  data: T;
  meta: {
    locale: string;
    publishedAt: string;
    version: number;
  };
  sys: {
    id: string;
    contentType: string;
    createdAt: string;
    updatedAt: string;
  };
}
```

**Testing**:
- `Unit: delivery API returns only published entries (no drafts)`
- `Unit: delivery API returns Cache-Control headers`
- `Unit: delivery API returns ETag header`
- `Unit: conditional request with matching ETag → 304 Not Modified`
- `Unit: ?locale=fr returns French content with en fallback`
- `Unit: ?preview=true with valid preview token returns draft content`
- `Unit: ?preview=true without token → 401`
- `Unit: GET by slug returns single entry`
- `Unit: delivery API does not require authentication (public)`
- `Integration: publish entry → immediately available via delivery API`
- `Integration: unpublish entry → no longer returned by delivery API`

---

## Phase 8: Admin Panel (React)

### Purpose
Build the React-based admin panel for content management — the web interface where editors create, edit, and publish content. After this phase, non-developer users can manage content through a browser UI.

### Tasks

#### 8.1 — Admin Shell and Authentication UI

**What**: React app scaffold with login screen, authenticated layout shell (sidebar navigation, header with user menu), and routing.

**Design**:

```typescript
// packages/admin/src/lib/api-client.ts
export class ApiClient {
  private baseUrl: string;
  private sessionToken: string | null;

  async login(email: string, password: string): Promise<LoginResponse>;
  async logout(): Promise<void>;
  async getProfile(): Promise<UserProfile>;
  async request<T>(method: string, path: string, body?: unknown): Promise<T>;
}

// packages/admin/src/App.tsx — Route structure
// /login                          — Login page
// /                               — Dashboard (redirect to /content)
// /content                        — Content type list
// /content/:contentType           — Entry list for content type
// /content/:contentType/new       — Create new entry
// /content/:contentType/:id       — Edit entry
// /media                          — Media library
// /users                          — User management (admin only)
// /webhooks                       — Webhook management (admin only)
// /settings                       — Content type management (admin only)
// /settings/content-types         — Content type list
// /settings/content-types/new     — Create content type
// /settings/locales               — Locale management
```

Sidebar navigation items derived from content types (fetched via API on mount). Protected routes redirect to /login if no valid session.

**Testing**:
- `E2E: load /login → login form visible`
- `E2E: login with valid credentials → redirect to /content`
- `E2E: login with invalid credentials → error message displayed`
- `E2E: accessing /content without login → redirect to /login`
- `E2E: sidebar shows all content types as navigation items`
- `E2E: logout clears session and redirects to /login`
- `Unit: ApiClient stores session token in memory (not localStorage)`
- `Unit: ApiClient attaches Authorization header on every request`

---

#### 8.2 — Dynamic Entry Editor

**What**: A form builder that generates editing forms from content type JSON Schema definitions, supporting all field types (text, richtext, media picker, relationship selector, etc.).

**Design**:

```typescript
// packages/admin/src/features/entries/components/FieldRenderer.tsx
// Renders the appropriate form widget based on field type:
//
// string (plain)     → <Input type="text" />
// string (richtext)  → <LexicalEditor /> (rich text with toolbar)
// string (markdown)  → <MarkdownEditor /> (CodeMirror-based)
// string (slug)      → <SlugInput /> (auto-generates from title, editable)
// string (email)     → <Input type="email" />
// string (url)       → <Input type="url" />
// string (date)      → <DatePicker />
// string (datetime)  → <DateTimePicker />
// string (select)    → <Select options={enum values} />
// number             → <Input type="number" />
// boolean            → <Switch />
// media (uuid)       → <MediaPicker /> (opens media library modal)
// relationship       → <RelationshipPicker /> (search + select entries)
// array              → <RepeatableField /> (add/remove/reorder items)
// object             → <FieldGroup /> (nested form fields)

export interface FieldRendererProps {
  fieldName: string;
  definition: FieldDefinition;
  value: unknown;
  onChange: (value: unknown) => void;
  errors?: ValidationError[];
  locale?: string;
}
```

The editor page layout:
- Left panel (70%): Form fields rendered from schema
- Right panel (30%): Entry metadata (status, locale selector, version history, publish button)
- Top bar: Save (draft), Submit for Review, Publish actions based on current status and user permissions
- Auto-save every 30 seconds as draft

**Testing**:
- `E2E: create blog_post content type → navigate to /content/blog_post/new → form shows title, body, slug fields`
- `E2E: fill in form and save → entry appears in entry list`
- `E2E: edit existing entry → form pre-filled with current data`
- `E2E: rich text editor renders toolbar and allows formatting`
- `E2E: media picker opens modal, shows uploaded images, selecting one sets the field`
- `E2E: relationship picker searches entries by title, selecting one sets the field`
- `E2E: validation errors displayed next to fields on save`
- `E2E: slug auto-generates from title field`
- `E2E: publish button available when entry is in "approved" status`
- `E2E: locale switcher shows available locales with translation status`

---

#### 8.3 — Media Library UI

**What**: Grid/list view of uploaded media with upload, search, folder navigation, and inline editing of alt text and metadata.

**Design**:

- Grid view with thumbnail previews (images) or file type icons (documents, videos)
- Drag-and-drop upload zone
- Folder tree sidebar for navigation
- Search bar filtering by filename and alt text
- Click to open detail panel: preview, metadata, alt text editor, usage list (which entries reference this media)
- Bulk actions: move to folder, delete selected

**Testing**:
- `E2E: upload image via drag-and-drop → appears in grid`
- `E2E: upload image via file picker → appears in grid`
- `E2E: click image → detail panel shows preview, metadata, alt text form`
- `E2E: edit alt text and save → alt text updated via API`
- `E2E: create folder → folder appears in tree`
- `E2E: move image to folder → image appears in folder, disappears from root`
- `E2E: search "logo" → only matching media shown`
- `E2E: delete media → removed from grid and S3`

---

## Phase 9: Search and Content Federation

### Purpose
Integrate full-text search via Meilisearch for fast content discovery, and implement content federation to query and combine data from external APIs at runtime. These features elevate the CMS from basic CRUD to a composable content platform.

### Tasks

#### 9.1 — Meilisearch Integration and Content Indexing

**What**: Automatic indexing of published content to Meilisearch on publish events, with a search API for the admin panel and delivery API.

**Design**:

```typescript
// packages/core/src/modules/search/search.service.ts
export class SearchService {
  /**
   * Index or update an entry in Meilisearch.
   * Called by webhook/event handler on entry.published events.
   */
  async indexEntry(entry: EntryResponse, contentType: ContentTypeDefinition): Promise<void>;

  /**
   * Remove an entry from the search index.
   * Called on entry.unpublished and entry.deleted events.
   */
  async deindexEntry(contentTypeSlug: string, entryId: string): Promise<void>;

  /**
   * Search across all content types or a specific content type.
   */
  async search(query: SearchQuery): Promise<SearchResponse>;

  /**
   * Rebuild the entire search index from database.
   */
  async rebuildIndex(contentTypeSlug?: string): Promise<void>;
}

export interface SearchQuery {
  q: string;                      // search query string
  contentType?: string;           // filter to specific content type
  locale?: string;                // filter by locale
  status?: string;                // filter by status (default: "published")
  page?: number;
  pageSize?: number;
  facets?: string[];              // facet fields
}

export interface SearchResponse {
  hits: SearchHit[];
  totalHits: number;
  page: number;
  pageSize: number;
  processingTimeMs: number;
  facetDistribution?: Record<string, Record<string, number>>;
}
```

Meilisearch index per content type (e.g., `cms_blog_post`, `cms_page`). Searchable fields determined from content type schema (string fields).  Filterable attributes: status, locale, content type, and any enum/select fields.

API endpoints:
```
GET /api/search?q=...&contentType=...&locale=...   — Search entries
POST /api/search/rebuild                            — Rebuild index (admin)
```

**Testing**:
- `Unit: publish entry → entry indexed in Meilisearch`
- `Unit: unpublish entry → entry removed from index`
- `Unit: search "hello" → returns entries with "hello" in any string field`
- `Unit: search with contentType filter → only matching content type`
- `Unit: search with locale filter → only matching locale`
- `Unit: rebuild index re-indexes all published entries`
- `Integration: create entry, publish, search → entry found`
- `Integration: typo-tolerant search ("helo" matches "hello")`
- `Unit: facet distribution returns count per content type`

---

#### 9.2 — Content Federation

**What**: Allow content types to define federated fields that resolve from external APIs at query time, enabling the CMS to stitch content from multiple sources.

**Design**:

```typescript
// packages/core/src/modules/federation/federation.types.ts
export interface FederatedSource {
  id: string;
  name: string;
  type: "rest" | "graphql";
  baseUrl: string;
  authType: "none" | "api_key" | "bearer" | "basic";
  authConfig: Record<string, string>;  // encrypted at rest
  cacheTtlSeconds: number;            // default 300 (5 min)
  healthCheckPath?: string;
}

export interface FederatedFieldConfig {
  sourceId: string;
  path: string;          // REST endpoint path or GraphQL query
  mapping: Record<string, string>; // map external fields to CMS fields
  joinOn: string;        // local field to match with external data
}
```

Content type schema extension for federated fields:
```json
{
  "type": "object",
  "properties": {
    "title": { "type": "string" },
    "productData": {
      "type": "object",
      "x-federated": {
        "sourceId": "shopify-api",
        "path": "/products/{slug}.json",
        "mapping": { "price": "variants[0].price", "inventory": "variants[0].inventory_quantity" },
        "joinOn": "slug"
      }
    }
  }
}
```

Federated data is resolved at query time (not stored) and cached in Redis with configurable TTL. GraphQL resolvers handle federated fields transparently.

API endpoints:
```
GET    /api/federation/sources      — List federated sources
POST   /api/federation/sources      — Create federated source
PATCH  /api/federation/sources/:id  — Update source
DELETE /api/federation/sources/:id  — Delete source
POST   /api/federation/sources/:id/test — Test connectivity
```

**Testing**:
- `Unit: federated field resolves data from external API`
- `Unit: federated data cached in Redis for TTL duration`
- `Unit: cache miss → fetch from external API → cache result`
- `Unit: external API failure → return null for federated field (graceful degradation)`
- `Unit: external API timeout (>5s) → return null, log warning`
- `Integration (mocked API): query entry with federated field → external data merged`
- `Unit: test connectivity endpoint returns success/failure`
- `Unit: federated source credentials encrypted at rest`

---

## Phase 10: AI Content Enrichment Pipeline

### Purpose
Implement the AI-native features that differentiate this CMS from incumbents: automatic content enrichment (SEO metadata, alt text, summaries), writing assistant, and intelligent translation. These features run as background jobs triggered by content events.

### Tasks

#### 10.1 — AI Enrichment Pipeline (SEO, Alt Text, Summaries)

**What**: Background pipeline that automatically generates SEO metadata, alt text for images, and content summaries when entries are published or updated.

**Design**:

```typescript
// packages/core/src/modules/ai/enrichment.service.ts
export class EnrichmentService {
  /**
   * Enrich an entry with AI-generated metadata.
   * Triggered by entry.published event via BullMQ job.
   */
  async enrichEntry(entryId: string, locale: string): Promise<EnrichmentResult>;

  /**
   * Generate alt text for a media asset.
   * Triggered by media.uploaded event for images.
   */
  async generateAltText(mediaId: string): Promise<string>;

  /**
   * Generate SEO metadata (title, description, keywords).
   */
  async generateSeoMetadata(content: string, contentType: string): Promise<SeoMetadata>;

  /**
   * Generate a summary of the content.
   */
  async generateSummary(content: string, maxLength: number): Promise<string>;
}

export interface EnrichmentResult {
  seoTitle: string | null;
  seoDescription: string | null;
  summary: string | null;
  suggestedTags: string[];
  structuredData: Record<string, unknown> | null; // JSON-LD / Schema.org
}

export interface SeoMetadata {
  title: string;          // max 60 chars
  description: string;    // max 160 chars
  keywords: string[];
}
```

AI provider abstraction:
```typescript
// packages/core/src/modules/ai/ai-provider.ts
export interface AiProvider {
  generateText(prompt: string, options?: GenerateOptions): Promise<string>;
  generateJson<T>(prompt: string, schema: z.ZodType<T>): Promise<T>;
  generateEmbedding(text: string): Promise<number[]>;
}

export class AnthropicProvider implements AiProvider { /* ... */ }
export class OpenAiProvider implements AiProvider { /* ... */ }

// Factory selects provider based on config
export function createAiProvider(config: Config): AiProvider;
```

Enrichment is opt-in per content type via a schema flag. Results stored in a dedicated `ai_enrichments` table (not modifying the entry data directly) until an editor approves them.

```typescript
// packages/core/src/modules/ai/enrichment.schema.ts (Drizzle)
export const aiEnrichments = pgTable("ai_enrichments", {
  id: uuid("id").primaryKey().defaultRandom(),
  entryId: uuid("entry_id").notNull().references(() => entries.id, { onDelete: "cascade" }),
  localeCode: varchar("locale_code", { length: 10 }).notNull(),
  enrichmentType: varchar("enrichment_type", { length: 50 }).notNull(), // "seo", "summary", "tags", "alt_text"
  result: jsonb("result").notNull(),
  status: varchar("status", { length: 20 }).notNull().default("pending"), // "pending", "approved", "rejected"
  approvedBy: uuid("approved_by").references(() => users.id),
  model: varchar("model", { length: 128 }).notNull(), // e.g. "claude-sonnet-4-20250514"
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Unit: publish entry with AI enabled → BullMQ job enqueued`
- `Unit: enrichment job generates SEO title under 60 chars`
- `Unit: enrichment job generates SEO description under 160 chars`
- `Unit: enrichment job generates summary`
- `Unit: enrichment results stored in ai_enrichments table with "pending" status`
- `Unit: approve enrichment → result merged into entry data`
- `Unit: reject enrichment → result discarded`
- `Unit: missing AI API key → enrichment skipped gracefully, warning logged`
- `Unit: AI provider timeout → job retried, entry unaffected`
- `Integration (mocked AI): publish blog post → SEO metadata generated and stored`
- `Unit: generate alt text for uploaded image → descriptive text returned`

---

#### 10.2 — AI Writing Assistant

**What**: Real-time API endpoint for inline writing suggestions — tone, clarity, SEO optimization, and brand voice checks.

**Design**:

```typescript
// packages/core/src/modules/ai/writing-assistant.service.ts
export class WritingAssistantService {
  /**
   * Analyze content and return suggestions.
   */
  async analyze(request: AnalyzeRequest): Promise<WritingSuggestion[]>;

  /**
   * Rewrite a selection for improved clarity or tone.
   */
  async rewrite(request: RewriteRequest): Promise<string>;
}

export interface AnalyzeRequest {
  content: string;
  contentType: string;    // context for appropriate suggestions
  locale: string;
  checks: ("tone" | "clarity" | "seo" | "brand_voice" | "accessibility" | "readability")[];
}

export interface WritingSuggestion {
  type: "tone" | "clarity" | "seo" | "brand_voice" | "accessibility" | "readability";
  severity: "info" | "warning" | "error";
  range: { start: number; end: number };  // character offsets
  message: string;
  suggestion?: string;   // suggested replacement text
}

export interface RewriteRequest {
  content: string;
  selection: { start: number; end: number };
  instruction: string;   // e.g. "make more concise", "formal tone"
}
```

API endpoints:
```
POST /api/ai/analyze    — Analyze content for suggestions
POST /api/ai/rewrite    — Rewrite selected text
```

Rate-limited to 10 requests/minute per user to control AI costs.

**Testing**:
- `Unit: analyze content with clarity check → returns readability suggestions`
- `Unit: analyze content with SEO check → returns keyword density, heading structure suggestions`
- `Unit: rewrite selection with "make concise" → shorter text returned`
- `Unit: rate limit exceeded → 429 Too Many Requests`
- `Unit: empty content → empty suggestions array`
- `Unit: suggestions include character offset ranges for highlighting`
- `Integration (mocked AI): full analyze request → suggestions returned with correct types`

---

#### 10.3 — Intelligent Translation

**What**: AI-powered content translation with human review queue, supporting batch translation of entries across locales.

**Design**:

```typescript
// packages/core/src/modules/ai/translation.service.ts
export class TranslationService {
  /**
   * Translate an entry to a target locale using AI.
   * Creates a new locale variant with "review" status.
   */
  async translateEntry(
    entryId: string,
    sourceLocale: string,
    targetLocale: string,
    options?: { glossary?: Record<string, string> }
  ): Promise<EntryResponse>;

  /**
   * Batch translate multiple entries to a target locale.
   * Enqueues as BullMQ jobs.
   */
  async batchTranslate(
    entryIds: string[],
    sourceLocale: string,
    targetLocale: string
  ): Promise<{ jobId: string; count: number }>;
}
```

API endpoints:
```
POST /api/ai/translate                    — Translate single entry
POST /api/ai/translate/batch              — Batch translate entries
GET  /api/ai/translate/jobs/:jobId        — Check batch translation status
```

Translated entries are created as new locale variants with status "review" so editors can verify before publishing. A glossary (term → translation mapping) can be provided for domain-specific terminology.

**Testing**:
- `Unit: translate entry en → fr → new locale variant created with "review" status`
- `Unit: translated entry has same document_id as source`
- `Unit: only localizable fields are translated`
- `Unit: glossary terms applied correctly in translation`
- `Unit: batch translate 10 entries → 10 BullMQ jobs enqueued`
- `Unit: batch status endpoint returns progress (completed/total)`
- `Integration (mocked AI): translate blog post → French locale variant with translated title and body`

---

## Phase 11: Semantic Content Graph and AI Features

### Purpose
Implement the semantic content graph (from Data Model Suggestion 4) — content embeddings, semantic similarity, internal linking suggestions, and topic clustering. These features power the AI-native content discovery and optimization capabilities.

### Tasks

#### 11.1 — Content Embeddings and Graph Tables

**What**: Add pgvector-backed embedding storage and graph tables (graph_nodes, graph_edges) to enable semantic content queries.

**Design**:

```typescript
// packages/core/src/db/schema-graph.ts (additional Drizzle schema)
export const graphNodes = pgTable("graph_nodes", {
  id: uuid("id").primaryKey(), // same as domain entity ID
  nodeType: varchar("node_type", { length: 50 }).notNull(),
  contentTypeSlug: varchar("content_type_slug", { length: 128 }),
  label: varchar("label", { length: 512 }),
  localeCode: varchar("locale_code", { length: 10 }),
  status: varchar("status", { length: 20 }),
  properties: jsonb("properties").default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const graphEdges = pgTable("graph_edges", {
  id: uuid("id").primaryKey().defaultRandom(),
  sourceId: uuid("source_id").notNull().references(() => graphNodes.id, { onDelete: "cascade" }),
  targetId: uuid("target_id").notNull().references(() => graphNodes.id, { onDelete: "cascade" }),
  edgeType: varchar("edge_type", { length: 100 }).notNull(),
  weight: real("weight").default(1.0),
  properties: jsonb("properties").default({}),
  createdBy: uuid("created_by"),
  isAiGenerated: boolean("is_ai_generated").notNull().default(false),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const contentEmbeddings = pgTable("content_embeddings", {
  id: uuid("id").primaryKey().defaultRandom(),
  entryId: uuid("entry_id").notNull().references(() => entries.id, { onDelete: "cascade" }),
  localeCode: varchar("locale_code", { length: 10 }).notNull(),
  model: varchar("model", { length: 128 }).notNull(),
  embedding: vector("embedding", { dimensions: 3072 }), // pgvector
  contentHash: varchar("content_hash", { length: 64 }).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  unique().on(table.entryId, table.localeCode, table.model),
]);
```

```typescript
// packages/core/src/modules/ai/embedding.service.ts
export class EmbeddingService {
  /**
   * Generate and store embedding for an entry.
   * Triggered on entry.published event.
   */
  async embedEntry(entryId: string, locale: string): Promise<void>;

  /**
   * Find entries semantically similar to the given entry.
   * Uses pgvector cosine similarity search.
   */
  async findSimilar(entryId: string, locale: string, limit?: number): Promise<SimilarEntry[]>;

  /**
   * Check if embedding is stale (content changed since last embedding).
   */
  async isStale(entryId: string, locale: string): Promise<boolean>;

  /**
   * Rebuild all embeddings for a content type.
   */
  async rebuildEmbeddings(contentTypeSlug?: string): Promise<void>;
}
```

**Testing**:
- `Unit: publish entry → embedding generated and stored in content_embeddings`
- `Unit: update and re-publish entry → embedding regenerated (content_hash changed)`
- `Unit: find similar returns top N entries sorted by cosine similarity`
- `Unit: stale check returns true when content changed since last embedding`
- `Unit: graph_nodes entry created when entry is created`
- `Unit: graph_edges "references" created when relationship field set`
- `Unit: graph_edges "authored_by" created linking entry to author`
- `Integration (mocked OpenAI): embed 5 entries → find similar for entry 1 → returns entries 2-5 ranked`
- `Unit: missing OpenAI API key → embedding skipped, warning logged`

---

#### 11.2 — Semantic Linking and Related Content Suggestions

**What**: Use embeddings and the content graph to suggest internal links and related content for editors.

**Design**:

```typescript
// packages/core/src/modules/ai/semantic-linking.service.ts
export class SemanticLinkingService {
  /**
   * Suggest internal links for an entry based on semantic similarity
   * and existing graph relationships.
   */
  async suggestLinks(entryId: string, locale: string): Promise<LinkSuggestion[]>;

  /**
   * Suggest related content for a "related articles" widget.
   */
  async suggestRelated(entryId: string, locale: string, limit?: number): Promise<RelatedContent[]>;

  /**
   * Detect content gaps — topics covered by competitors but missing
   * from the current content graph.
   */
  async detectGaps(contentTypeSlug: string): Promise<ContentGap[]>;

  /**
   * Dependency analysis — what content depends on a given entry.
   * Used for safe-delete checks.
   */
  async getDependents(entryId: string): Promise<DependentEntry[]>;
}

export interface LinkSuggestion {
  targetEntryId: string;
  targetTitle: string;
  targetSlug: string;
  relevanceScore: number;       // 0-1 from embedding similarity
  suggestedAnchorText: string;  // AI-generated anchor text
  reason: string;               // "semantically similar", "same category", "related topic"
}

export interface RelatedContent {
  entryId: string;
  title: string;
  contentType: string;
  similarity: number;
}
```

API endpoints:
```
GET /api/ai/links/:entryId              — Get link suggestions
GET /api/ai/related/:entryId            — Get related content
GET /api/ai/gaps/:contentType           — Detect content gaps
GET /api/ai/dependents/:entryId         — Get dependent entries
```

**Testing**:
- `Unit: suggest links returns entries sorted by relevance score`
- `Unit: suggest links excludes already-linked entries`
- `Unit: suggest related returns entries from same and different content types`
- `Unit: dependents returns all entries with references to target entry`
- `Unit: dependents used by delete confirmation UI`
- `Integration: create 10 blog posts with embeddings → link suggestions reference semantically related posts`
- `Unit: content gap detection identifies under-linked entries`

---

## Phase 12: Plugin System, CLI, and SDK

### Purpose
Implement the plugin/extension system for third-party integrations, the CLI tool for project scaffolding, and the JavaScript/TypeScript client SDK for frontend developers. After this phase, the platform is extensible and has a first-class developer experience for consuming content.

### Tasks

#### 12.1 — Plugin System

**What**: A plugin architecture that allows extending the CMS with custom field types, API endpoints, webhook handlers, and admin panel widgets.

**Design**:

```typescript
// packages/core/src/plugins/plugin.interface.ts
export interface CmsPlugin {
  name: string;
  version: string;
  description?: string;

  /** Called when the plugin is registered. Use to add routes, hooks, etc. */
  register(ctx: PluginContext): Promise<void>;

  /** Called when the plugin is unregistered. Use for cleanup. */
  unregister?(): Promise<void>;
}

export interface PluginContext {
  /** Register additional Fastify routes */
  addRoutes(prefix: string, routes: FastifyPluginCallback): void;

  /** Register a custom field type for the content type schema */
  addFieldType(type: CustomFieldType): void;

  /** Register a hook that fires on content events */
  addHook(event: string, handler: HookHandler): void;

  /** Access the database connection */
  db: Database;

  /** Access the BullMQ queue for background jobs */
  queue: Queue;

  /** Access the config */
  config: Config;
}

export interface CustomFieldType {
  name: string;         // e.g. "color-picker"
  jsonSchemaType: string; // underlying JSON Schema type
  validate(value: unknown): boolean;
  serialize(value: unknown): unknown;  // for API output
  deserialize(value: unknown): unknown; // from API input
}
```

Plugins loaded from `plugins/` directory or from npm packages listed in config.

**Testing**:
- `Unit: register plugin adds routes at specified prefix`
- `Unit: register plugin with custom field type → field type available in content type schemas`
- `Unit: plugin hook fires on entry.published event`
- `Unit: unregister plugin removes routes and hooks`
- `Unit: plugin with invalid interface → error with descriptive message`
- `Integration: create plugin that adds /api/plugins/hello → GET returns 200`

---

#### 12.2 — Client SDK (@headless-cms/sdk)

**What**: A TypeScript SDK for frontend developers to query the Content Delivery API with type safety, caching, and framework integrations.

**Design**:

```typescript
// packages/sdk/src/index.ts
export class HeadlessCmsClient {
  constructor(options: ClientOptions);

  /**
   * Fetch a single entry by slug or ID.
   */
  async getEntry<T = Record<string, unknown>>(
    contentType: string,
    slugOrId: string,
    options?: QueryOptions
  ): Promise<DeliveryResponse<T>>;

  /**
   * Fetch a list of entries with pagination and filtering.
   */
  async getEntries<T = Record<string, unknown>>(
    contentType: string,
    options?: ListOptions
  ): Promise<DeliveryListResponse<T>>;

  /**
   * Search across content types.
   */
  async search<T = Record<string, unknown>>(
    query: string,
    options?: SearchOptions
  ): Promise<SearchResponse<T>>;
}

export interface ClientOptions {
  baseUrl: string;
  apiKey?: string;          // API token for authenticated requests
  previewToken?: string;    // Preview token for draft content
  locale?: string;          // Default locale
  cache?: CacheOptions;     // Client-side response cache
}

export interface QueryOptions {
  locale?: string;
  preview?: boolean;
}

export interface ListOptions {
  page?: number;
  pageSize?: number;
  sort?: string;
  locale?: string;
  preview?: boolean;
  filters?: Record<string, unknown>;
}
```

Usage example:
```typescript
import { HeadlessCmsClient } from "@headless-cms/sdk";

const cms = new HeadlessCmsClient({
  baseUrl: "https://cms.example.com",
  apiKey: "cms_api_...",
  locale: "en",
});

// Fetch blog posts
const posts = await cms.getEntries("blog_post", {
  pageSize: 10,
  sort: "-publishedAt",
  filters: { "data.category": "technology" },
});

// Fetch single post by slug
const post = await cms.getEntry("blog_post", "my-first-post");
```

**Testing**:
- `Unit: getEntry with valid slug → returns typed entry`
- `Unit: getEntry with invalid slug → throws NotFoundError`
- `Unit: getEntries with pagination → returns paginated response`
- `Unit: getEntries with filters → correct query params sent`
- `Unit: preview mode sends preview token in request`
- `Unit: default locale applied when not specified per-request`
- `Unit: response cache returns cached result within TTL`
- `Unit: search returns hits with highlights`
- `Integration: SDK → real API server → returns blog posts`

---

#### 12.3 — CLI Tool (@headless-cms/cli)

**What**: A CLI for creating new CMS projects, generating content types from the command line, and managing development instances.

**Design**:

Commands:
```
headless-cms init [project-name]       — Scaffold new project with docker-compose
headless-cms dev                        — Start development server
headless-cms migrate                    — Run database migrations
headless-cms seed                       — Seed default data
headless-cms content-type:create        — Interactive content type creation
headless-cms content-type:list          — List content types
headless-cms export [--content-type]    — Export content as JSON
headless-cms import [file]              — Import content from JSON
```

Built with `commander` for argument parsing and `inquirer` for interactive prompts.

**Testing**:
- `Unit: init creates project directory with docker-compose.yml, .env.example, package.json`
- `Unit: content-type:create with --name blog_post --fields title:string,body:richtext → creates content type via API`
- `Unit: export generates JSON file with all entries for content type`
- `Unit: import reads JSON file and creates entries via API`
- `Unit: --help prints usage for all commands`

---

## Phase Summary & Dependencies

```
Phase 1:  Foundation                      ─── required by everything
    │
Phase 2:  Authentication & RBAC           ─── requires Phase 1
    │
Phase 3:  Content Type Management         ─── requires Phase 2
    │
Phase 4:  Entry CRUD & Publishing         ─── requires Phase 3
    │
    ├── Phase 5:  Media Management        ─── requires Phase 4, can parallel with Phase 6
    │
    ├── Phase 6:  Locales & Webhooks      ─── requires Phase 4, can parallel with Phase 5
    │
    └── Phase 7:  GraphQL & Delivery API  ─── requires Phase 4, can parallel with Phase 5 & 6
         │
         ├── Phase 8:  Admin Panel        ─── requires Phases 4, 5, 6, 7
         │
         └── Phase 9:  Search & Federation ─── requires Phase 7, can parallel with Phase 8
              │
              ├── Phase 10: AI Enrichment  ─── requires Phases 4, 5 (media), 9 (search indexing)
              │
              └── Phase 11: Semantic Graph ─── requires Phase 10 (embeddings), can parallel with Phase 12
                   │
                   └── Phase 12: Plugin, CLI, SDK ─── requires Phase 7 (delivery API)
```

Parallelism opportunities:
- Phases 5, 6, and 7 can be developed concurrently after Phase 4
- Phase 8 and Phase 9 can be developed concurrently (admin panel vs. search/federation)
- Phase 11 and Phase 12 can be developed concurrently after Phase 10

---

## Definition of Done (per phase)

1. All tasks in the phase implemented and code reviewed.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (requires Docker services running).
4. Type checking passes (`tsc --noEmit` reports zero errors).
5. Linting and formatting pass (`biome check .` reports zero errors).
6. Docker build succeeds (`docker build .` completes without errors).
7. Database migrations created and applied without errors.
8. API endpoints appear in auto-generated OpenAPI spec at `/docs`.
9. GraphQL schema includes new types/queries (where applicable).
10. New environment variables documented in `.env.example`.
11. All error responses follow RFC 7807 Problem Details format.
12. Audit log entries created for all state-changing operations.
13. Webhook events fire for all content lifecycle changes (where applicable).
14. Feature works end-to-end: create → read → update → delete cycle verified.
