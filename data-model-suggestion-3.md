# Data Model Suggestion 3: Event-Sourced / Audit-First (CQRS)

> Project: Headless CMS · Created: 2026-05-20

## Philosophy

This approach treats every content change as an immutable event. Instead of updating rows in place, the system appends a new event record for every create, update, publish, unpublish, translate, and delete operation. The current state of any entry is reconstructed by replaying its event stream, or more practically, by querying materialised read models (projections) that are kept up to date asynchronously.

This pattern is inspired by the New York Times CMS architecture, which stores every edit, article, and byline since 1851 as events in a Kafka topic. It naturally provides a complete audit trail, enables temporal queries ("what was this entry on March 15th?"), supports undo/redo at any granularity, and produces a rich analytics dataset of authoring patterns. The CQRS (Command Query Responsibility Segregation) split means the write path is optimised for append-only durability while read paths are optimised for the specific query patterns of content delivery APIs, admin UIs, and analytics dashboards.

The trade-off is complexity: the system has two data paths (write events, read projections), and developers must understand eventual consistency between them. However, for a headless CMS where content changes are relatively low-frequency (compared to, say, financial transactions) and the audit trail is genuinely valuable, event sourcing is a strong architectural fit.

**Best for:** Platforms requiring full audit trails, temporal content queries, regulatory compliance, AI-powered content analytics, and sophisticated undo/redo capabilities.

**Trade-offs:**
- Pro: Complete, immutable audit trail -- every change is preserved forever
- Pro: Temporal queries: reconstruct content state at any point in time
- Pro: Natural undo/redo by replaying events up to a specific point
- Pro: Rich analytics: event streams feed content performance dashboards and AI training
- Pro: Projections can be rebuilt from scratch if read models need to change
- Con: Higher storage usage (events accumulate, snapshots help but add complexity)
- Con: Eventual consistency between write and read paths (typically milliseconds)
- Con: More complex codebase: event handlers, projections, snapshot management
- Con: Debugging requires understanding event replay rather than simple row inspection
- Con: Schema evolution of events requires careful versioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents (CNCF) | All events in the event store follow CloudEvents envelope structure |
| OpenAPI 3.1 | Read API generated from projection schemas |
| GraphQL June 2018 | GraphQL resolvers read from projections, mutations emit events |
| JSON Schema | Event payload schemas and content type definitions |
| RFC 6749 (OAuth 2.0) | Command authentication |
| RFC 7519 (JWT) | Preview tokens for draft content projections |
| ISO/IEC 27001 | Event store IS the audit log -- compliance built into the architecture |
| CommonMark / Portable Text | Rich text content stored within event payloads |
| BCP 47 | Locale codes in translation events |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- WRITE SIDE: Event Store
-- ============================================================

-- The event store is the single source of truth.
-- All events are immutable and append-only. No UPDATE or DELETE.

CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,                 -- groups events for one aggregate (entry, media, user)
    stream_type VARCHAR(50) NOT NULL,        -- 'entry', 'media', 'user', 'content_type', 'webhook'
    sequence_number BIGINT NOT NULL,         -- monotonic within a stream
    event_type VARCHAR(128) NOT NULL,        -- see event type taxonomy below
    event_version INT NOT NULL DEFAULT 1,    -- schema version of this event type

    -- CloudEvents-aligned envelope
    source VARCHAR(255) NOT NULL DEFAULT 'cms',
    subject VARCHAR(512),                    -- e.g. 'entry/blog_post/550e8400...'

    -- Event payload: the data that changed
    -- Example for 'entry.field_updated':
    -- {
    --   "content_type_slug": "blog_post",
    --   "locale": "en",
    --   "field": "title",
    --   "old_value": "Draft Title",
    --   "new_value": "My First Blog Post"
    -- }
    payload JSONB NOT NULL,

    -- Metadata
    user_id UUID,                            -- who triggered this event
    ip_address INET,
    correlation_id UUID,                     -- links related events (e.g. bulk publish)
    causation_id UUID,                       -- which event caused this one
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(stream_id, sequence_number)
);

-- Primary query: replay events for a specific stream
CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);

-- Query by event type (for projectors)
CREATE INDEX idx_events_type ON events(event_type, created_at);

-- Query by time range (for analytics and audit)
CREATE INDEX idx_events_created ON events(created_at);

-- Query by user (for user activity feed)
CREATE INDEX idx_events_user ON events(user_id, created_at);

-- Query by correlation (for tracing related operations)
CREATE INDEX idx_events_correlation ON events(correlation_id);

-- Partition by month for performance at scale
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

## Event Type Taxonomy

```sql
-- ============================================================
-- REFERENCE: Event Types
-- ============================================================

-- Content Type Events
-- content_type.created     - new content type defined
-- content_type.updated     - schema or settings changed
-- content_type.deleted     - content type removed

-- Entry Events
-- entry.created            - new entry initialised
-- entry.field_updated      - single field changed (granular)
-- entry.bulk_updated       - multiple fields changed at once
-- entry.status_changed     - draft -> review -> published etc.
-- entry.published          - entry goes live
-- entry.unpublished        - entry taken offline
-- entry.archived           - entry moved to archive
-- entry.deleted            - entry soft-deleted
-- entry.restored           - entry un-deleted
-- entry.locale_added       - new locale variant created
-- entry.locale_removed     - locale variant removed
-- entry.relationship_added - reference to another entry added
-- entry.relationship_removed - reference removed

-- Media Events
-- media.uploaded           - new file uploaded
-- media.metadata_updated   - alt text, caption, tags changed
-- media.replaced           - file replaced (new version)
-- media.deleted            - file removed

-- User Events
-- user.created             - new user account
-- user.updated             - profile or role changed
-- user.deactivated         - account disabled
-- user.logged_in           - login event
-- user.logged_out          - logout event

-- Webhook Events
-- webhook.created          - new webhook configured
-- webhook.triggered        - webhook fired
-- webhook.delivery_success - delivery confirmed
-- webhook.delivery_failed  - delivery failed

-- AI Events
-- ai.enrichment_completed  - AI generated metadata for an entry
-- ai.translation_completed - AI translated an entry
-- ai.suggestion_generated  - AI suggested content improvements
```

## Snapshots (Performance Optimisation)

```sql
-- ============================================================
-- WRITE SIDE: Snapshots
-- ============================================================

-- Snapshots cache the aggregate state at a point in time to avoid
-- replaying the full event stream on every read.
-- Snapshots are created periodically (e.g. every 50 events).

CREATE TABLE snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    sequence_number BIGINT NOT NULL,         -- snapshot is valid up to this event
    state JSONB NOT NULL,                    -- serialised aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, sequence_number)
);

CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, sequence_number DESC);
```

## Read Side: Projections

```sql
-- ============================================================
-- READ SIDE: Content Delivery Projection
-- ============================================================

-- This projection is optimised for the Content Delivery API (CDA).
-- It stores the current published state of each entry, flattened for fast reads.
-- It is rebuilt from events by the projector.

CREATE TABLE projection_entries (
    id UUID NOT NULL,                        -- same as stream_id
    content_type_slug VARCHAR(128) NOT NULL,
    locale_code VARCHAR(10) NOT NULL,
    document_id UUID,                        -- groups locale variants
    status VARCHAR(20) NOT NULL,
    data JSONB NOT NULL,                     -- current field values
    slug VARCHAR(255),
    published_at TIMESTAMPTZ,
    first_published_at TIMESTAMPTZ,
    version INT NOT NULL DEFAULT 1,
    last_event_sequence BIGINT NOT NULL,     -- tracks projection currency

    created_by UUID,
    updated_by UUID,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, locale_code)
);

-- Delivery API indexes
CREATE INDEX idx_proj_entries_type_status ON projection_entries(content_type_slug, status);
CREATE INDEX idx_proj_entries_slug ON projection_entries(slug, locale_code);
CREATE INDEX idx_proj_entries_published ON projection_entries(published_at DESC)
    WHERE status = 'published';
CREATE INDEX idx_proj_entries_data ON projection_entries USING GIN (data jsonb_path_ops);

-- ============================================================
-- READ SIDE: Content Management Projection
-- ============================================================

-- This projection is optimised for the admin UI: includes drafts,
-- workflow state, and editor metadata.

CREATE TABLE projection_entries_admin (
    id UUID NOT NULL,
    content_type_slug VARCHAR(128) NOT NULL,
    locale_code VARCHAR(10) NOT NULL,
    document_id UUID,
    status VARCHAR(20) NOT NULL,
    workflow_stage VARCHAR(128),
    data JSONB NOT NULL,
    slug VARCHAR(255),
    version INT NOT NULL DEFAULT 1,
    last_event_sequence BIGINT NOT NULL,

    -- Admin-specific fields
    assigned_to UUID,
    review_notes TEXT,
    has_unsaved_changes BOOLEAN DEFAULT false,

    created_by UUID,
    updated_by UUID,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, locale_code)
);

CREATE INDEX idx_proj_admin_status ON projection_entries_admin(content_type_slug, status);
CREATE INDEX idx_proj_admin_assigned ON projection_entries_admin(assigned_to)
    WHERE assigned_to IS NOT NULL;

-- ============================================================
-- READ SIDE: Content History Projection
-- ============================================================

-- This projection provides the version history UI and diff view.

CREATE TABLE projection_entry_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID NOT NULL,
    locale_code VARCHAR(10) NOT NULL,
    version_number INT NOT NULL,
    event_id UUID NOT NULL,                  -- reference to the source event
    event_type VARCHAR(128) NOT NULL,
    changed_fields TEXT[],
    summary TEXT,                            -- human-readable change description
    changed_by UUID,
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE(entry_id, locale_code, version_number)
);

CREATE INDEX idx_proj_history_entry ON projection_entry_history(entry_id, locale_code);

-- ============================================================
-- READ SIDE: Analytics Projection
-- ============================================================

-- Aggregated metrics derived from events for content performance dashboards.

CREATE TABLE projection_content_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type_slug VARCHAR(128) NOT NULL,
    entry_id UUID NOT NULL,
    locale_code VARCHAR(10) NOT NULL,
    total_edits INT NOT NULL DEFAULT 0,
    total_publishes INT NOT NULL DEFAULT 0,
    unique_editors INT NOT NULL DEFAULT 0,
    avg_time_to_publish INTERVAL,            -- draft to first publish
    last_published_at TIMESTAMPTZ,
    last_edited_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, locale_code)
);

CREATE INDEX idx_proj_metrics_type ON projection_content_metrics(content_type_slug);
```

## System Tables (Conventional Relational)

```sql
-- ============================================================
-- SYSTEM: Users, Roles, Media, Locales, Webhooks
-- (These use conventional relational tables, not event-sourced,
--  because they change infrequently and benefit from direct queries.)
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
    content_type_slug VARCHAR(128),
    action VARCHAR(20) NOT NULL
        CHECK (action IN ('create', 'read', 'update', 'delete', 'publish', 'unpublish')),
    conditions JSONB,
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
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    token_type VARCHAR(20) NOT NULL DEFAULT 'personal',
    scopes JSONB,
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE content_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(128) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    kind VARCHAR(20) NOT NULL DEFAULT 'collection',
    schema JSONB NOT NULL,
    draft_enabled BOOLEAN NOT NULL DEFAULT true,
    versioning_enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE locales (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(128) NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT false,
    is_active BOOLEAN NOT NULL DEFAULT true,
    fallback_locale_code VARCHAR(10),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    metadata JSONB,
    folder_path TEXT DEFAULT '/',
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    events TEXT[] NOT NULL,
    content_type_slugs TEXT[],
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

## Example: Event Replay and Temporal Query

```sql
-- Reconstruct entry state at a specific point in time
-- "What did this blog post look like on March 15, 2026?"

WITH ordered_events AS (
    SELECT
        e.event_type,
        e.payload,
        e.created_at,
        e.sequence_number
    FROM events e
    WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
      AND e.stream_type = 'entry'
      AND e.created_at <= '2026-03-15 23:59:59+00'
    ORDER BY e.sequence_number ASC
)
SELECT * FROM ordered_events;
-- Application code replays these events to reconstruct the state

-- Find all changes to a specific entry by a specific user
SELECT
    e.event_type,
    e.payload->>'field' AS changed_field,
    e.payload->>'new_value' AS new_value,
    e.created_at
FROM events e
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.user_id = '660e8400-e29b-41d4-a716-446655440000'
ORDER BY e.sequence_number ASC;

-- Content delivery API query (reads from projection, not events)
SELECT
    pe.id,
    pe.data->>'title' AS title,
    pe.data->>'slug' AS slug,
    pe.published_at
FROM projection_entries pe
WHERE pe.content_type_slug = 'blog_post'
  AND pe.status = 'published'
  AND pe.locale_code = 'en'
ORDER BY pe.published_at DESC
LIMIT 20;

-- Analytics: most-edited entries this month
SELECT
    cm.entry_id,
    pe.data->>'title' AS title,
    cm.total_edits,
    cm.unique_editors,
    cm.avg_time_to_publish
FROM projection_content_metrics cm
JOIN projection_entries pe ON pe.id = cm.entry_id AND pe.locale_code = cm.locale_code
WHERE cm.content_type_slug = 'blog_post'
ORDER BY cm.total_edits DESC
LIMIT 10;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month at scale) |
| Snapshots | 1 | snapshots |
| Delivery Projection | 1 | projection_entries |
| Admin Projection | 1 | projection_entries_admin |
| History Projection | 1 | projection_entry_history |
| Analytics Projection | 1 | projection_content_metrics |
| Content Types | 1 | content_types |
| Users & RBAC | 4 | users, roles, permissions, api_tokens |
| Localization | 1 | locales |
| Media | 1 | media |
| Webhooks | 2 | webhooks, webhook_deliveries |
| **Total** | **15** | Fixed count; projections can be added without changing write path |

---

## Key Design Decisions

1. **Events are the source of truth**, not projections. If a projection becomes corrupted or needs a schema change, it can be dropped and rebuilt by replaying the event store. This makes the system self-healing.

2. **Events are granular** (field-level changes, not full document snapshots). This enables precise diff views, fine-grained analytics ("which fields are edited most?"), and smaller event payloads.

3. **Multiple read projections serve different consumers**: the Content Delivery API reads from `projection_entries` (published content only), the admin UI reads from `projection_entries_admin` (all statuses), and the history view reads from `projection_entry_history`. Each is optimised for its query patterns.

4. **System tables (users, roles, media) are conventional relational**, not event-sourced. Event sourcing adds complexity, and these entities change infrequently with simple access patterns. Only content entries -- the core domain -- are event-sourced.

5. **CloudEvents envelope structure** aligns events with the CNCF standard, making them publishable to external event brokers (Kafka, NATS, RabbitMQ) for downstream consumers like search indexers, AI enrichment pipelines, and analytics platforms.

6. **Snapshots prevent unbounded replay cost**. After every N events (e.g., 50), a snapshot of the current aggregate state is stored. Reconstruction loads the latest snapshot and replays only subsequent events.

7. **Correlation and causation IDs** enable distributed tracing. A bulk publish operation creates a single correlation_id shared by all entry.published events, making it easy to audit or roll back the entire operation.

8. **The event store can be partitioned by time** for scalability. Monthly partitions allow old events to be moved to cold storage while keeping recent events on fast SSDs. The projections remain on hot storage regardless of event store partitioning.
