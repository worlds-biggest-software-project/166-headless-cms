# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Headless CMS · Created: 2026-05-20

## Philosophy

This approach follows the pattern established by Directus: every content type maps to its own dedicated database table, and the CMS maintains a parallel set of system metadata tables that describe the schema, field definitions, relationships, and access controls. The database schema is the source of truth. Content types are real PostgreSQL tables with real columns, foreign keys, and constraints -- not rows in a generic "entries" table.

This is the most traditional and well-understood approach. It mirrors how Payload CMS and Strapi work internally: each collection/content-type gets its own table, with array fields and block fields spawning additional child tables. The schema is fully introspectable by standard database tools, and queries are plain SQL with typed columns and proper indexes.

The trade-off is rigidity: adding a new field to a content type requires a database migration (ALTER TABLE). This is manageable with migration tooling (Drizzle, Knex, or raw SQL migrations), but it means schema changes flow through a deployment pipeline rather than being instant in the admin UI.

**Best for:** Teams that want maximum query performance, full SQL tooling compatibility, and strict data integrity guarantees.

**Trade-offs:**
- Pro: Standard SQL queries, no JSONB parsing overhead, full foreign key integrity
- Pro: Database-level constraints prevent invalid data
- Pro: Any SQL tool (pgAdmin, DBeaver, Metabase) can query content directly
- Pro: Drizzle/Knex ORM integration is straightforward
- Con: Schema changes require migrations and deployments
- Con: Many content types means many tables (can reach 100+ tables quickly)
- Con: Array and block fields create child tables, increasing join complexity
- Con: Multi-locale support doubles table count (one _locales table per content table)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1 | Auto-generated from table/column metadata in system tables |
| GraphQL June 2018 | Schema generated from field definitions; each content type becomes a GraphQL type |
| JSON Schema | Content type definitions exportable as JSON Schema for validation |
| RFC 6749 (OAuth 2.0) | Authentication tokens stored in `api_tokens` table |
| RFC 7519 (JWT) | Preview tokens for draft content access |
| ISO/IEC 27001 | Audit log table provides security event trail |
| WCAG 2.2 | Media table includes alt_text field for accessibility |
| CloudEvents | Webhook event payloads follow CloudEvents envelope structure |
| CommonMark | Rich text fields store CommonMark; Portable Text stored as JSONB |

---

## System Metadata Tables

These tables describe the CMS schema itself -- what content types exist, what fields they have, how they relate to each other.

```sql
-- ============================================================
-- SYSTEM: Schema Registry
-- ============================================================

CREATE TABLE content_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(128) NOT NULL UNIQUE,       -- e.g. 'blog_post', 'product'
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    kind VARCHAR(20) NOT NULL DEFAULT 'collection'
        CHECK (kind IN ('collection', 'single')),
    api_id VARCHAR(128) NOT NULL UNIQUE,     -- used in REST/GraphQL endpoints
    draft_enabled BOOLEAN NOT NULL DEFAULT true,
    versioning_enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_types_slug ON content_types(slug);

CREATE TABLE field_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type_id UUID NOT NULL REFERENCES content_types(id) ON DELETE CASCADE,
    name VARCHAR(128) NOT NULL,              -- column name in the content table
    display_name VARCHAR(255) NOT NULL,
    field_type VARCHAR(50) NOT NULL,         -- text, richtext, number, boolean, date,
                                             -- media, relationship, array, blocks, json, select, email, slug
    position INT NOT NULL DEFAULT 0,
    required BOOLEAN NOT NULL DEFAULT false,
    unique_field BOOLEAN NOT NULL DEFAULT false,
    localizable BOOLEAN NOT NULL DEFAULT false,
    default_value TEXT,
    validation_rules JSONB,                  -- {"min": 0, "max": 1000, "regex": "..."}
    field_options JSONB,                     -- enum values, relationship target, etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(content_type_id, name)
);

CREATE INDEX idx_field_definitions_content_type ON field_definitions(content_type_id);

CREATE TABLE content_type_relations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_content_type_id UUID NOT NULL REFERENCES content_types(id) ON DELETE CASCADE,
    source_field_name VARCHAR(128) NOT NULL,
    target_content_type_id UUID NOT NULL REFERENCES content_types(id) ON DELETE CASCADE,
    relation_type VARCHAR(20) NOT NULL
        CHECK (relation_type IN ('one_to_one', 'one_to_many', 'many_to_one', 'many_to_many')),
    junction_table VARCHAR(255),             -- for many-to-many relations
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_relations_source ON content_type_relations(source_content_type_id);
CREATE INDEX idx_relations_target ON content_type_relations(target_content_type_id);
```

## User and Access Control Tables

```sql
-- ============================================================
-- SYSTEM: Users, Roles, Permissions (RBAC)
-- ============================================================

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(128) NOT NULL UNIQUE,       -- 'admin', 'editor', 'author', 'viewer'
    description TEXT,
    is_system BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    content_type_id UUID REFERENCES content_types(id) ON DELETE CASCADE,  -- NULL = global
    action VARCHAR(20) NOT NULL
        CHECK (action IN ('create', 'read', 'update', 'delete', 'publish', 'unpublish')),
    conditions JSONB,                        -- field-level or filter conditions
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(role_id, content_type_id, action)
);

CREATE INDEX idx_permissions_role ON permissions(role_id);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(128),
    last_name VARCHAR(128),
    avatar_url TEXT,
    role_id UUID NOT NULL REFERENCES roles(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role_id);

CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    token_type VARCHAR(20) NOT NULL DEFAULT 'personal'
        CHECK (token_type IN ('personal', 'api', 'preview')),
    permissions JSONB,                       -- scoped permissions for this token
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Localization Tables

```sql
-- ============================================================
-- SYSTEM: Localization
-- ============================================================

CREATE TABLE locales (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(10) NOT NULL UNIQUE,        -- 'en', 'en-US', 'fr', 'de'  (BCP 47)
    name VARCHAR(128) NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT false,
    is_active BOOLEAN NOT NULL DEFAULT true,
    fallback_locale_id UUID REFERENCES locales(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Media Management Tables

```sql
-- ============================================================
-- SYSTEM: Media / Assets
-- ============================================================

CREATE TABLE media (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename VARCHAR(512) NOT NULL,
    original_filename VARCHAR(512) NOT NULL,
    mime_type VARCHAR(128) NOT NULL,
    file_size BIGINT NOT NULL,               -- bytes
    width INT,                               -- for images
    height INT,                              -- for images
    alt_text TEXT,                            -- WCAG 2.2 accessibility
    caption TEXT,
    storage_path TEXT NOT NULL,               -- path in object storage
    storage_provider VARCHAR(50) NOT NULL DEFAULT 'local',
    focal_point JSONB,                       -- {"x": 0.5, "y": 0.3}
    metadata JSONB,                          -- EXIF data, color palette, etc.
    folder_id UUID REFERENCES media_folders(id),
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_media_mime ON media(mime_type);
CREATE INDEX idx_media_folder ON media(folder_id);

CREATE TABLE media_folders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    parent_id UUID REFERENCES media_folders(id),
    path TEXT NOT NULL,                      -- materialised path: '/images/blog/'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_media_folders_path ON media_folders USING btree(path);
```

## Workflow and Publishing Tables

```sql
-- ============================================================
-- SYSTEM: Workflows and Publishing
-- ============================================================

CREATE TABLE workflow_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    content_type_id UUID REFERENCES content_types(id),  -- NULL = applies to all
    stages JSONB NOT NULL,                   -- [{"name":"draft","color":"gray"},{"name":"review","color":"yellow"},...]
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Per-content-type version/publishing tracking
-- This table tracks the status of each entry across the publish lifecycle.
-- The actual content lives in the content type's own table (see below).

CREATE TABLE entry_status (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type_slug VARCHAR(128) NOT NULL,
    entry_id UUID NOT NULL,                  -- FK to the content table (enforced at app level)
    locale_code VARCHAR(10) NOT NULL REFERENCES locales(code),
    status VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'review', 'approved', 'published', 'archived')),
    published_at TIMESTAMPTZ,
    published_by UUID REFERENCES users(id),
    scheduled_publish_at TIMESTAMPTZ,
    version INT NOT NULL DEFAULT 1,
    workflow_stage VARCHAR(128),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(content_type_slug, entry_id, locale_code)
);

CREATE INDEX idx_entry_status_lookup ON entry_status(content_type_slug, entry_id);
CREATE INDEX idx_entry_status_published ON entry_status(status, published_at);
```

## Versioning Tables

```sql
-- ============================================================
-- SYSTEM: Content Versioning
-- ============================================================

CREATE TABLE entry_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type_slug VARCHAR(128) NOT NULL,
    entry_id UUID NOT NULL,
    locale_code VARCHAR(10) NOT NULL,
    version_number INT NOT NULL,
    snapshot JSONB NOT NULL,                  -- full serialised entry at this version
    changed_fields TEXT[],                   -- which fields changed
    changed_by UUID REFERENCES users(id),
    change_summary TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(content_type_slug, entry_id, locale_code, version_number)
);

CREATE INDEX idx_versions_entry ON entry_versions(content_type_slug, entry_id, locale_code);
CREATE INDEX idx_versions_created ON entry_versions(created_at);
```

## Webhook and Integration Tables

```sql
-- ============================================================
-- SYSTEM: Webhooks and Events
-- ============================================================

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    events TEXT[] NOT NULL,                  -- {'entry.create', 'entry.publish', 'media.upload'}
    content_type_ids UUID[],                 -- filter to specific content types (NULL = all)
    headers JSONB,                           -- custom headers (e.g. auth tokens)
    is_active BOOLEAN NOT NULL DEFAULT true,
    secret VARCHAR(255),                     -- HMAC signing secret
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
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_webhook ON webhook_deliveries(webhook_id);
CREATE INDEX idx_webhook_deliveries_created ON webhook_deliveries(created_at);
```

## Audit Log

```sql
-- ============================================================
-- SYSTEM: Audit Log
-- ============================================================

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(50) NOT NULL,             -- 'entry.create', 'entry.update', 'entry.publish', 'user.login'
    resource_type VARCHAR(128) NOT NULL,     -- 'entry', 'media', 'user', 'role', 'webhook'
    resource_id UUID,
    content_type_slug VARCHAR(128),
    changes JSONB,                           -- {"field": {"old": ..., "new": ...}}
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
```

## Example Content Type Tables

When a user creates a content type "Blog Post" with fields title, slug, body, author, category, and featured_image, the system generates these tables:

```sql
-- ============================================================
-- CONTENT: Blog Post (auto-generated from content type definition)
-- ============================================================

CREATE TABLE ct_blog_post (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(255) NOT NULL UNIQUE,
    author_id UUID REFERENCES users(id),
    featured_image_id UUID REFERENCES media(id),
    created_by UUID REFERENCES users(id),
    updated_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Localizable fields go in a separate table (following Payload CMS pattern)
CREATE TABLE ct_blog_post_locales (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID NOT NULL REFERENCES ct_blog_post(id) ON DELETE CASCADE,
    locale_code VARCHAR(10) NOT NULL REFERENCES locales(code),
    title VARCHAR(512) NOT NULL,
    body TEXT,                               -- CommonMark or Portable Text (JSONB)
    excerpt TEXT,
    seo_title VARCHAR(255),
    seo_description VARCHAR(512),
    UNIQUE(parent_id, locale_code)
);

CREATE INDEX idx_blog_post_locales_parent ON ct_blog_post_locales(parent_id);

-- Array field: "tags" on blog_post spawns its own table
CREATE TABLE ct_blog_post_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID NOT NULL REFERENCES ct_blog_post(id) ON DELETE CASCADE,
    position INT NOT NULL DEFAULT 0,
    tag VARCHAR(128) NOT NULL
);

CREATE INDEX idx_blog_post_tags_parent ON ct_blog_post_tags(parent_id);

-- Many-to-many: blog_post <-> category (junction table)
CREATE TABLE ct_blog_post_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    blog_post_id UUID NOT NULL REFERENCES ct_blog_post(id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES ct_category(id) ON DELETE CASCADE,
    position INT NOT NULL DEFAULT 0,
    UNIQUE(blog_post_id, category_id)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Schema Registry | 3 | content_types, field_definitions, content_type_relations |
| Users & RBAC | 4 | users, roles, permissions, api_tokens |
| Localization | 1 | locales |
| Media | 2 | media, media_folders |
| Workflows | 2 | workflow_definitions, entry_status |
| Versioning | 1 | entry_versions |
| Webhooks | 2 | webhooks, webhook_deliveries |
| Audit | 1 | audit_log |
| **System Total** | **16** | Fixed system tables |
| Per Content Type | 2-5+ | Base table + _locales + child tables for arrays/blocks/junctions |

---

## Key Design Decisions

1. **Content types get real tables** (prefixed `ct_`) rather than storing all entries in a single generic table. This provides full SQL type safety, proper indexing, and foreign key integrity at the database level.

2. **Localizable fields live in `_locales` child tables** following the Payload CMS pattern. Non-localizable fields stay on the parent table, avoiding unnecessary joins when locale is not relevant.

3. **Array fields spawn child tables** rather than being stored as JSONB. This allows array elements to have their own relationships, be individually queryable, and maintain referential integrity.

4. **Schema metadata is stored in system tables** (content_types, field_definitions) that describe the dynamic content tables. The API layer reads these to generate REST/GraphQL endpoints dynamically.

5. **Versioning uses JSONB snapshots** rather than duplicating the full table structure. This is a pragmatic compromise -- the current state lives in typed columns for fast queries, while historical versions are serialized for storage efficiency.

6. **Entry status is decoupled from content tables** via the entry_status table. This allows the publishing lifecycle to be managed independently of the content data, and supports per-locale publishing states.

7. **Media folders use materialised paths** (`/images/blog/2026/`) for efficient tree queries without recursive CTEs. The path column is indexed for prefix matching.

8. **Audit log is append-only** with no foreign key constraints on resource_id (since the referenced row may be deleted). This ensures audit entries are never lost.
