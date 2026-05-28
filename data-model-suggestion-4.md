# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Headless CMS · Created: 2026-05-20

## Philosophy

This approach layers a property graph on top of a relational foundation. Content entries, media, users, and content types are all nodes in a graph, connected by typed, weighted edges that represent relationships like "references", "authored_by", "tagged_with", "translated_from", and "similar_to". The relational tables store the operational CRUD data (entry fields, user profiles, media metadata), while a graph layer (`graph_nodes` / `graph_edges` tables in PostgreSQL, or optionally Neo4j) stores and queries the relationship network.

The key insight is that a headless CMS is fundamentally a graph of interconnected content. Blog posts reference categories, categories contain sub-categories, authors write posts, posts link to related posts, translations reference their source entry, and AI-generated semantic links connect topically related content across content types. Traditional relational joins work for known, fixed relationships, but a graph model excels when relationships are dynamic, multi-hop, and the queries are exploratory ("find all content within 3 hops of this entry", "detect circular references", "suggest related content by graph proximity").

This architecture is particularly powerful for the AI-native features in the project brief: the semantic content graph, internal linking suggestions, related content widgets, and content gap detection all benefit from a native graph representation. The graph also enables conflict-of-interest detection in editorial workflows (e.g., "this reviewer authored a competing article") and content dependency analysis for safe deletion.

**Best for:** Platforms with rich content relationships, semantic linking, AI-powered content recommendations, editorial dependency analysis, and multi-hop traversal queries.

**Trade-offs:**
- Pro: Multi-hop relationship queries are fast and natural (graph traversal vs. recursive CTEs)
- Pro: Dynamic relationship types -- new edge types require no schema migration
- Pro: AI content graph features (semantic linking, related content, gap detection) are first-class
- Pro: Content dependency analysis: "what breaks if I delete this entry?"
- Pro: Visualisable: the content graph can be rendered in the admin UI
- Con: Two data storage layers (relational + graph) increase complexity
- Con: Data consistency between relational and graph layers must be maintained
- Con: Graph queries require different skills than SQL
- Con: PostgreSQL-based graph (ltree/adjacency list) is less performant than Neo4j for deep traversals
- Con: More storage overhead due to edge table

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GraphQL June 2018 | Natural API layer for graph-structured content; types map to node types, connections map to edges |
| OpenAPI 3.1 | REST API for CRUD operations on the relational layer |
| JSON Schema | Content type definitions and validation |
| JSON-LD / Schema.org | Semantic edge types align with Schema.org vocabulary (e.g., `schema:author`, `schema:about`) |
| RFC 6749 (OAuth 2.0) | API authentication |
| RFC 7519 (JWT) | Preview tokens |
| CloudEvents | Event payloads for graph change notifications |
| ISO/IEC 27001 | Audit trail on graph mutations |
| BCP 47 | Locale codes for translation edges |
| CommonMark / Portable Text | Rich text within entry data |

---

## Graph Layer Tables

```sql
-- ============================================================
-- GRAPH: Node and Edge Tables
-- ============================================================

-- Every significant entity in the CMS is a node in the graph.
-- This table stores the graph identity and type; the actual data lives
-- in the domain-specific relational tables (entries, media, users, etc.).

CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY,                     -- same ID as the domain entity
    node_type VARCHAR(50) NOT NULL,          -- 'entry', 'media', 'user', 'content_type', 'tag', 'category'
    content_type_slug VARCHAR(128),          -- for entry nodes: which content type
    label VARCHAR(512),                      -- display label (denormalized for graph queries)
    locale_code VARCHAR(10),
    status VARCHAR(20),                      -- for entry nodes: current status
    properties JSONB DEFAULT '{}',           -- lightweight properties for graph queries
    -- Example properties for an entry node:
    -- {"slug": "my-post", "word_count": 1200, "reading_time_minutes": 5}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_content_type ON graph_nodes(content_type_slug)
    WHERE content_type_slug IS NOT NULL;
CREATE INDEX idx_graph_nodes_status ON graph_nodes(status)
    WHERE node_type = 'entry';
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN (properties);

-- Edges represent typed, directed relationships between nodes.

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type VARCHAR(100) NOT NULL,         -- see edge type taxonomy below
    weight FLOAT DEFAULT 1.0,               -- strength of relationship (for AI/ranking)
    properties JSONB DEFAULT '{}',           -- edge-specific metadata
    -- Example properties for a 'references' edge:
    -- {"field": "related_posts", "position": 0}
    -- Example properties for a 'semantically_similar' edge:
    -- {"similarity_score": 0.87, "model": "text-embedding-3-large", "computed_at": "2026-05-20T10:00:00Z"}
    created_by UUID,                         -- NULL for AI-generated edges
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core graph traversal indexes
CREATE INDEX idx_graph_edges_source ON graph_edges(source_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_id, edge_type);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_ai ON graph_edges(is_ai_generated, edge_type)
    WHERE is_ai_generated = true;
CREATE INDEX idx_graph_edges_weight ON graph_edges(edge_type, weight DESC);

-- Prevent duplicate edges of the same type between the same nodes
CREATE UNIQUE INDEX idx_graph_edges_unique
    ON graph_edges(source_id, target_id, edge_type)
    WHERE properties->>'field' IS NULL;
```

## Edge Type Taxonomy

```sql
-- ============================================================
-- REFERENCE: Edge Types
-- ============================================================

-- Content Relationships (explicit, author-defined)
-- 'references'           - entry references another entry (via relationship field)
-- 'belongs_to'           - entry belongs to a category/collection
-- 'has_media'            - entry uses a media asset
-- 'translated_from'      - locale variant translated from source entry

-- Authorship and Ownership
-- 'authored_by'          - entry was created by user
-- 'edited_by'            - entry was last edited by user
-- 'reviewed_by'          - entry was reviewed by user
-- 'assigned_to'          - entry is assigned to user for workflow

-- Taxonomy and Classification
-- 'tagged_with'          - entry is tagged with a tag node
-- 'categorised_as'       - entry belongs to a category
-- 'child_of'             - hierarchical parent-child (categories, folders)

-- AI-Generated Relationships
-- 'semantically_similar' - AI computed content similarity (weight = similarity score)
-- 'suggested_link'       - AI suggests this internal link
-- 'content_gap'          - AI identifies this as a gap relative to another entry
-- 'topic_cluster'        - AI groups entries into topic clusters
-- 'supersedes'           - newer content supersedes older content

-- System Relationships
-- 'depends_on'           - entry depends on another (e.g. component references)
-- 'embedded_in'          - content block embedded in a page
-- 'variant_of'           - A/B test variant of another entry
```

## Content Storage Tables (Relational)

```sql
-- ============================================================
-- RELATIONAL: Content Types and Entries
-- ============================================================

CREATE TABLE content_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(128) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    kind VARCHAR(20) NOT NULL DEFAULT 'collection'
        CHECK (kind IN ('collection', 'single')),
    schema JSONB NOT NULL,                   -- JSON Schema for field definitions
    draft_enabled BOOLEAN NOT NULL DEFAULT true,
    versioning_enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type_id UUID NOT NULL REFERENCES content_types(id),
    content_type_slug VARCHAR(128) NOT NULL,
    document_id UUID,                        -- groups locale variants
    locale_code VARCHAR(10) NOT NULL DEFAULT 'en',
    status VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'review', 'approved', 'published', 'archived')),
    data JSONB NOT NULL DEFAULT '{}',        -- entry field values
    slug VARCHAR(255),
    version INT NOT NULL DEFAULT 1,
    published_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    updated_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entries_type_status ON entries(content_type_slug, status);
CREATE INDEX idx_entries_slug ON entries(slug, locale_code);
CREATE INDEX idx_entries_document ON entries(document_id);
CREATE INDEX idx_entries_data ON entries USING GIN (data jsonb_path_ops);

CREATE TABLE entry_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    data JSONB NOT NULL,
    status VARCHAR(20) NOT NULL,
    changed_by UUID REFERENCES users(id),
    change_summary TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, version_number)
);
```

## Users, Roles, Media, Locales, Webhooks

```sql
-- ============================================================
-- RELATIONAL: Users and RBAC
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

-- ============================================================
-- RELATIONAL: Locales
-- ============================================================

CREATE TABLE locales (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(128) NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT false,
    is_active BOOLEAN NOT NULL DEFAULT true,
    fallback_locale_code VARCHAR(10),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RELATIONAL: Media
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
    metadata JSONB,
    folder_path TEXT DEFAULT '/',
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RELATIONAL: Webhooks
-- ============================================================

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

-- ============================================================
-- RELATIONAL: Audit Log
-- ============================================================

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(50) NOT NULL,
    resource_type VARCHAR(128) NOT NULL,
    resource_id UUID,
    content_type_slug VARCHAR(128),
    diff JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## AI Content Graph Tables

```sql
-- ============================================================
-- AI: Embedding Storage for Semantic Graph
-- ============================================================

-- Content embeddings power the AI-generated edges (semantic similarity,
-- suggested links, topic clusters, content gap detection).

CREATE TABLE content_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    locale_code VARCHAR(10) NOT NULL,
    model VARCHAR(128) NOT NULL,             -- 'text-embedding-3-large', 'voyage-3', etc.
    embedding vector(3072),                  -- pgvector extension; dimension depends on model
    content_hash VARCHAR(64) NOT NULL,       -- SHA-256 of the source text (to detect stale embeddings)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, locale_code, model)
);

-- pgvector index for approximate nearest neighbor search
CREATE INDEX idx_embeddings_vector ON content_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_embeddings_entry ON content_embeddings(entry_id);

-- ============================================================
-- AI: Topic Clusters
-- ============================================================

CREATE TABLE topic_clusters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    centroid vector(3072),                   -- cluster center embedding
    entry_count INT NOT NULL DEFAULT 0,
    model VARCHAR(128) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

```sql
-- ============================================================
-- GRAPH QUERIES
-- ============================================================

-- Find all content directly related to a specific blog post
SELECT
    gn.id,
    gn.node_type,
    gn.label,
    ge.edge_type,
    ge.weight
FROM graph_edges ge
JOIN graph_nodes gn ON gn.id = ge.target_id
WHERE ge.source_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY ge.weight DESC;

-- Find content within 2 hops (related to related content)
WITH RECURSIVE content_graph AS (
    -- Start node
    SELECT
        ge.target_id AS node_id,
        ge.edge_type,
        ge.weight,
        1 AS depth,
        ARRAY[ge.source_id, ge.target_id] AS path
    FROM graph_edges ge
    WHERE ge.source_id = '550e8400-e29b-41d4-a716-446655440000'
      AND ge.edge_type IN ('references', 'semantically_similar', 'tagged_with')

    UNION ALL

    -- Traverse
    SELECT
        ge.target_id,
        ge.edge_type,
        ge.weight,
        cg.depth + 1,
        cg.path || ge.target_id
    FROM graph_edges ge
    JOIN content_graph cg ON ge.source_id = cg.node_id
    WHERE cg.depth < 2
      AND ge.target_id != ALL(cg.path)  -- prevent cycles
)
SELECT DISTINCT
    gn.id,
    gn.node_type,
    gn.label,
    cg.edge_type,
    cg.depth,
    cg.weight
FROM content_graph cg
JOIN graph_nodes gn ON gn.id = cg.node_id
WHERE gn.node_type = 'entry'
  AND gn.status = 'published'
ORDER BY cg.depth ASC, cg.weight DESC
LIMIT 20;

-- Dependency analysis: "What depends on this entry?"
-- (Safe deletion check)
SELECT
    gn.id,
    gn.node_type,
    gn.content_type_slug,
    gn.label,
    ge.edge_type
FROM graph_edges ge
JOIN graph_nodes gn ON gn.id = ge.source_id
WHERE ge.target_id = '550e8400-e29b-41d4-a716-446655440000'
  AND ge.edge_type IN ('references', 'depends_on', 'embedded_in');

-- AI: Find semantically similar content using embeddings
SELECT
    e.id,
    e.data->>'title' AS title,
    1 - (ce1.embedding <=> ce2.embedding) AS similarity
FROM content_embeddings ce1
JOIN content_embeddings ce2 ON ce2.model = ce1.model
    AND ce2.entry_id != ce1.entry_id
    AND ce2.locale_code = ce1.locale_code
JOIN entries e ON e.id = ce2.entry_id
WHERE ce1.entry_id = '550e8400-e29b-41d4-a716-446655440000'
  AND ce1.locale_code = 'en'
  AND e.status = 'published'
ORDER BY ce1.embedding <=> ce2.embedding ASC
LIMIT 10;

-- Content gap detection: entries with few outgoing edges
-- (possibly under-linked or isolated content)
SELECT
    gn.id,
    gn.label,
    gn.content_type_slug,
    COUNT(ge.id) AS outgoing_edges
FROM graph_nodes gn
LEFT JOIN graph_edges ge ON ge.source_id = gn.id
    AND ge.edge_type IN ('references', 'tagged_with', 'categorised_as')
WHERE gn.node_type = 'entry'
  AND gn.status = 'published'
GROUP BY gn.id, gn.label, gn.content_type_slug
HAVING COUNT(ge.id) < 2
ORDER BY COUNT(ge.id) ASC;

-- Editorial conflict check: has this reviewer authored related content?
SELECT
    ge_author.source_id AS reviewer_authored_entry,
    gn.label AS authored_entry_title,
    ge_similar.weight AS similarity_score
FROM graph_edges ge_review
JOIN graph_edges ge_similar ON ge_similar.source_id = ge_review.target_id
    AND ge_similar.edge_type = 'semantically_similar'
    AND ge_similar.weight > 0.8
JOIN graph_edges ge_author ON ge_author.target_id = ge_review.source_id
    AND ge_author.source_id = ge_similar.target_id
    AND ge_author.edge_type = 'authored_by'
JOIN graph_nodes gn ON gn.id = ge_author.source_id
WHERE ge_review.target_id = '660e8400-e29b-41d4-a716-446655440000'  -- reviewer user
  AND ge_review.edge_type = 'assigned_to';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Content Storage | 3 | content_types, entries, entry_versions |
| AI/Embeddings | 2 | content_embeddings, topic_clusters |
| Users & RBAC | 4 | users, roles, permissions, api_tokens |
| Localization | 1 | locales |
| Media | 1 | media |
| Webhooks | 2 | webhooks, webhook_deliveries |
| Audit | 1 | audit_log |
| **Total** | **16** | Fixed count; graph edges scale with relationships |

---

## Key Design Decisions

1. **Graph is implemented in PostgreSQL** using `graph_nodes` and `graph_edges` tables rather than requiring a separate Neo4j instance. This keeps the deployment stack simple (single database) while still supporting multi-hop traversal via recursive CTEs. For teams that need deeper graph performance, Neo4j can be added as a secondary read store.

2. **Every entity is a graph node** with the same UUID used in both the relational table and the graph_nodes table. This makes joins between the graph and relational layers trivial and avoids ID mapping complexity.

3. **Edges carry weight and properties**, enabling ranked traversal. AI-generated edges store their similarity score as the weight, so "find most related content" queries sort by weight naturally.

4. **AI-generated edges are flagged** with `is_ai_generated = true` and can be regenerated without affecting human-authored relationships. This separation ensures AI suggestions can be refreshed (e.g., when the embedding model changes) without disturbing editorial decisions.

5. **Content embeddings use pgvector** for approximate nearest-neighbor search directly in PostgreSQL. The `content_hash` column detects stale embeddings (when content changes, the hash changes, triggering re-embedding).

6. **The relational layer uses the Hybrid JSONB pattern** (Model 2) for entries, keeping the table count low while the graph layer handles relationships. This is a deliberate combination: JSONB for flexible content, graph for flexible relationships.

7. **Edge types follow a vocabulary** loosely aligned with Schema.org predicates (`authored_by`, `categorised_as`, `translated_from`). This makes the graph exportable as JSON-LD for structured data / SEO purposes.

8. **Dependency analysis is a first-class feature.** Before deleting any entry, the system queries `graph_edges` for incoming edges to identify dependent content, preventing broken references. The graph makes this query trivial compared to scanning JSONB fields across all entries.
