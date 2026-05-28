# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Headless CMS · Created: 2026-05-20

## Philosophy

This approach uses PostgreSQL's JSONB capabilities to store content entry data as structured JSON documents within a small number of generic tables, while keeping system metadata (users, roles, locales, media) in conventional relational tables. Instead of generating a new database table for every content type, all entries live in a single `entries` table with their field data stored in a JSONB column. The content type schema defines what fields are expected, but the database itself is schema-flexible.

This is the pattern used by many document-oriented CMS platforms adapted to PostgreSQL. It trades the strict typing of Model 1 for deployment simplicity and instant schema changes. Adding a field to a content type means updating the schema definition -- no ALTER TABLE, no migration, no deployment. Content editors see the new field immediately.

PostgreSQL's GIN indexes on JSONB columns provide fast containment queries (`@>`), and computed/generated columns can extract frequently-queried JSONB paths into indexed virtual columns. This hybrid approach gives you 80% of the flexibility of a document database with 100% of the transactional guarantees of PostgreSQL.

**Best for:** Rapid MVP development, multi-tenant SaaS deployments, and projects where content types change frequently or vary by tenant/jurisdiction.

**Trade-offs:**
- Pro: No migrations needed when content types change -- instant schema updates
- Pro: Far fewer tables (tens instead of hundreds)
- Pro: Multi-tenant deployments can have different content types per tenant without schema changes
- Pro: Content import/export is trivial (entries are already JSON)
- Pro: Simpler backup and replication (fewer tables, less schema drift)
- Con: No database-level type enforcement on content fields (validation is application-level)
- Con: JSONB queries are slower than typed column queries for complex filtering
- Con: Foreign key integrity between content entries must be enforced at the application level
- Con: Reporting and analytics tools may struggle with JSONB data
- Con: GIN indexes consume more storage than B-tree indexes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JSON Schema | Content type definitions are stored as JSON Schema documents; used for both API validation and editor rendering |
| OpenAPI 3.1 | API specs generated from content type JSON Schemas |
| GraphQL June 2018 | Types generated from JSON Schema definitions at startup |
| RFC 6749 (OAuth 2.0) | Authentication for management API |
| RFC 7519 (JWT) | Preview tokens stored with expiry metadata |
| CloudEvents | Webhook payloads use CloudEvents envelope |
| CommonMark / Portable Text | Rich text stored as JSONB within the data column |
| BCP 47 | Locale codes for localized content variants |
| ISO/IEC 27001 | Audit log for security compliance |

---

## Content Type Schema Tables

```sql
-- ============================================================
-- SCHEMA: Content Type Definitions
-- ============================================================

CREATE TABLE content_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(128) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    kind VARCHAR(20) NOT NULL DEFAULT 'collection'
        CHECK (kind IN ('collection', 'single')),
    api_id VARCHAR(128) NOT NULL UNIQUE,

    -- The content type schema as JSON Schema
    -- Example:
    -- {
    --   "type": "object",
    --   "properties": {
    --     "title": {"type": "string", "maxLength": 512, "localizable": true},
    --     "slug": {"type": "string", "pattern": "^[a-z0-9-]+$"},
    --     "body": {"type": "string", "format": "richtext", "localizable": true},
    --     "category": {"type": "string", "format": "uuid", "x-relationship": {"target": "category", "type": "many_to_one"}},
    --     "tags": {"type": "array", "items": {"type": "string"}},
    --     "metadata": {"type": "object", "properties": {"seo_title": {"type": "string"}, "seo_description": {"type": "string"}}}
    --   },
    --   "required": ["title", "slug"]
    -- }
    schema JSONB NOT NULL,

    draft_enabled BOOLEAN NOT NULL DEFAULT true,
    versioning_enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_types_slug ON content_types(slug);
```

## Core Content Tables

```sql
-- ============================================================
-- CONTENT: Unified Entry Storage
-- ============================================================

CREATE TABLE entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type_id UUID NOT NULL REFERENCES content_types(id),
    content_type_slug VARCHAR(128) NOT NULL,  -- denormalized for fast filtering

    -- The full entry data as JSONB
    -- Structure matches the content type's JSON Schema
    -- Example for a blog_post:
    -- {
    --   "title": "My First Post",
    --   "slug": "my-first-post",
    --   "body": "# Hello\n\nThis is **my first post**.",
    --   "category": "550e8400-e29b-41d4-a716-446655440000",
    --   "tags": ["javascript", "cms"],
    --   "metadata": {"seo_title": "My First Post | Blog", "seo_description": "..."}
    -- }
    data JSONB NOT NULL DEFAULT '{}',

    status VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'review', 'approved', 'published', 'archived')),
    locale_code VARCHAR(10) NOT NULL DEFAULT 'en',
    published_at TIMESTAMPTZ,
    scheduled_publish_at TIMESTAMPTZ,
    version INT NOT NULL DEFAULT 1,

    -- Common extracted fields for fast filtering (generated columns)
    slug VARCHAR(255) GENERATED ALWAYS AS (data->>'slug') STORED,

    created_by UUID REFERENCES users(id),
    updated_by UUID REFERENCES users(id),
    published_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Primary query indexes
CREATE INDEX idx_entries_content_type ON entries(content_type_slug);
CREATE INDEX idx_entries_status ON entries(status);
CREATE INDEX idx_entries_locale ON entries(locale_code);
CREATE INDEX idx_entries_slug ON entries(slug);
CREATE INDEX idx_entries_published ON entries(status, published_at)
    WHERE status = 'published';

-- GIN index for JSONB containment queries
-- Enables: SELECT * FROM entries WHERE data @> '{"tags": ["javascript"]}'
CREATE INDEX idx_entries_data ON entries USING GIN (data jsonb_path_ops);

-- Composite index for API queries
CREATE INDEX idx_entries_api_lookup ON entries(content_type_slug, status, locale_code);

-- ============================================================
-- CONTENT: Localized Entry Variants
-- ============================================================

-- When a content type has localizable fields, each locale gets its own entry row.
-- The locale_code + a shared document_id group them together.

CREATE TABLE entry_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type_id UUID NOT NULL REFERENCES content_types(id),
    content_type_slug VARCHAR(128) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- entries.document_id links locale variants together
ALTER TABLE entries ADD COLUMN document_id UUID REFERENCES entry_documents(id);
CREATE INDEX idx_entries_document ON entries(document_id);
CREATE UNIQUE INDEX idx_entries_document_locale ON entries(document_id, locale_code);
```

## Content Relationships

```sql
-- ============================================================
-- CONTENT: Relationships Between Entries
-- ============================================================

-- Since JSONB stores relationship UUIDs as strings, we maintain a separate
-- table for queryable cross-references (enables reverse lookups and cascades).

CREATE TABLE entry_relationships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_entry_id UUID NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    source_field VARCHAR(128) NOT NULL,      -- which field holds this relationship
    target_entry_id UUID NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    target_content_type_slug VARCHAR(128) NOT NULL,
    position INT NOT NULL DEFAULT 0,         -- ordering for array relationships
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entry_rels_source ON entry_relationships(source_entry_id);
CREATE INDEX idx_entry_rels_target ON entry_relationships(target_entry_id);
CREATE INDEX idx_entry_rels_target_type ON entry_relationships(target_content_type_slug, target_entry_id);
```

## Versioning

```sql
-- ============================================================
-- CONTENT: Versioning
-- ============================================================

CREATE TABLE entry_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    data JSONB NOT NULL,                     -- snapshot of entry data at this version
    status VARCHAR(20) NOT NULL,
    changed_by UUID REFERENCES users(id),
    change_summary TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, version_number)
);

CREATE INDEX idx_entry_versions_entry ON entry_versions(entry_id);
```

## Users, Roles, and Authentication

```sql
-- ============================================================
-- SYSTEM: Users and RBAC
-- ============================================================

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(128) NOT NULL UNIQUE,
    description TEXT,
    is_system BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    content_type_slug VARCHAR(128),          -- NULL = global permission
    action VARCHAR(20) NOT NULL
        CHECK (action IN ('create', 'read', 'update', 'delete', 'publish', 'unpublish')),
    conditions JSONB,                        -- field-level conditions
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(role_id, content_type_slug, action)
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(128),
    last_name VARCHAR(128),
    avatar_url TEXT,
    role_id UUID NOT NULL REFERENCES roles(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    preferences JSONB DEFAULT '{}',          -- UI preferences, notification settings
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);

CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    token_type VARCHAR(20) NOT NULL DEFAULT 'personal'
        CHECK (token_type IN ('personal', 'api', 'preview')),
    scopes JSONB,
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Media, Locales, Webhooks

```sql
-- ============================================================
-- SYSTEM: Locales
-- ============================================================

CREATE TABLE locales (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(10) NOT NULL UNIQUE,        -- BCP 47: 'en', 'en-US', 'fr-FR'
    name VARCHAR(128) NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT false,
    is_active BOOLEAN NOT NULL DEFAULT true,
    fallback_locale_code VARCHAR(10) REFERENCES locales(code),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SYSTEM: Media
-- ============================================================

CREATE TABLE media (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename VARCHAR(512) NOT NULL,
    original_filename VARCHAR(512) NOT NULL,
    mime_type VARCHAR(128) NOT NULL,
    file_size BIGINT NOT NULL,
    width INT,
    height INT,
    alt_text TEXT,
    caption TEXT,
    storage_path TEXT NOT NULL,
    storage_provider VARCHAR(50) NOT NULL DEFAULT 'local',
    focal_point JSONB,
    metadata JSONB,                          -- EXIF, color palette, AI-generated tags
    folder_path TEXT DEFAULT '/',
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_media_mime ON media(mime_type);
CREATE INDEX idx_media_folder ON media(folder_path);

-- ============================================================
-- SYSTEM: Webhooks
-- ============================================================

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    events TEXT[] NOT NULL,
    content_type_slugs TEXT[],               -- NULL = all content types
    headers JSONB,
    is_active BOOLEAN NOT NULL DEFAULT true,
    secret VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
    event_type VARCHAR(128) NOT NULL,
    payload JSONB NOT NULL,
    response_status INT,
    response_body TEXT,
    attempt INT NOT NULL DEFAULT 1,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Audit Log

```sql
-- ============================================================
-- SYSTEM: Audit Log
-- ============================================================

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(50) NOT NULL,
    resource_type VARCHAR(128) NOT NULL,
    resource_id UUID,
    content_type_slug VARCHAR(128),
    diff JSONB,                              -- {"field": {"old": ..., "new": ...}}
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Example Queries

```sql
-- Fetch all published blog posts in English
SELECT e.id, e.data->>'title' AS title, e.data->>'slug' AS slug, e.published_at
FROM entries e
WHERE e.content_type_slug = 'blog_post'
  AND e.status = 'published'
  AND e.locale_code = 'en'
ORDER BY e.published_at DESC
LIMIT 20;

-- Fetch entries tagged with "javascript" using GIN index
SELECT e.id, e.data->>'title' AS title
FROM entries e
WHERE e.content_type_slug = 'blog_post'
  AND e.data @> '{"tags": ["javascript"]}'
  AND e.status = 'published';

-- Fetch entry with all its locale variants
SELECT e.locale_code, e.data->>'title' AS title, e.status
FROM entries e
JOIN entry_documents d ON e.document_id = d.id
WHERE d.id = '550e8400-e29b-41d4-a716-446655440000';

-- Full-text search across all content types
SELECT e.id, e.content_type_slug, e.data->>'title' AS title
FROM entries e
WHERE to_tsvector('english', e.data::text) @@ to_tsquery('english', 'headless & cms')
  AND e.status = 'published';

-- Find all entries that reference a specific category
SELECT e.id, e.data->>'title' AS title
FROM entries e
JOIN entry_relationships r ON r.source_entry_id = e.id
WHERE r.target_entry_id = '550e8400-e29b-41d4-a716-446655440000'
  AND r.source_field = 'category';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Schema Registry | 1 | content_types (schema stored as JSONB) |
| Content Storage | 3 | entries, entry_documents, entry_relationships |
| Versioning | 1 | entry_versions |
| Users & RBAC | 4 | users, roles, permissions, api_tokens |
| Localization | 1 | locales |
| Media | 1 | media (folders as path strings, not separate table) |
| Webhooks | 2 | webhooks, webhook_deliveries |
| Audit | 1 | audit_log |
| **Total** | **14** | Fixed count regardless of content types |

---

## Key Design Decisions

1. **All content entries live in a single `entries` table** with a JSONB `data` column. This dramatically reduces table count and eliminates the need for migrations when content types change. The content type schema (stored as JSON Schema in `content_types`) drives validation at the application level.

2. **Generated columns extract hot paths** from JSONB. The `slug` column is a generated column that extracts `data->>'slug'` for efficient B-tree indexing. Additional generated columns can be added for any frequently-filtered field.

3. **GIN indexes with `jsonb_path_ops`** enable fast containment queries on the data column. This supports queries like "find all entries where tags contain 'javascript'" without scanning every row.

4. **Relationships are dual-stored**: as UUID strings within the JSONB data (for API serialization) and as rows in `entry_relationships` (for reverse lookups and cascade deletes). The relationship table is maintained by application-level triggers on entry create/update.

5. **Localization uses separate entry rows** grouped by `document_id` rather than nested JSONB. This allows each locale to have its own publishing status, version history, and workflow stage.

6. **Content type schema is JSON Schema**, which serves triple duty: database validation rules, API request/response schema generation (OpenAPI), and admin UI form rendering. One schema definition drives all three concerns.

7. **Media uses path strings** instead of a separate folders table. The `folder_path` column (e.g., `/images/blog/2026/`) is simpler to manage and query with LIKE/prefix matching than a recursive folder hierarchy.

8. **Table count is fixed** regardless of how many content types exist. This makes the database predictable for operations teams and simplifies backup, replication, and monitoring.
