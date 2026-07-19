# RFC-0002: Storage Engine

| Field       | Value                          |
|-------------|--------------------------------|
| Status      | **Draft**                      |
| Parent      | [RFC-0000](RFC-0000-dxdb-root.md) |
| Authors     | anandg                         |
| Created     | 2026-07-03                     |
| Last Update | 2026-07-18                     |

---

## Summary

This RFC defines the storage backend and the canonical SQL DDL for dxdb. The storage engine is **SQLite** via `rusqlite` (bundled feature). Each entity type gets its own SQLite table, dynamically created at entity type registration time and altered as indexes are added or removed.

---

## Storage Backend: SQLite via `rusqlite`

**Crate:** `rusqlite` with the `bundled` feature — SQLite is compiled into the binary; no system SQLite dependency. Version bundled: SQLite 3.45+ (supports `ALTER TABLE DROP COLUMN`, added in 3.35.0).

**Why SQLite:**
- Satisfies single-binary requirement without C++ build complexity
- ACID transactions cover entity + index column atomicity in one write
- Composite PK `(root_key, entity_local_key)` per entity type table gives B-tree-workspaceed locality: all entities with the same `root_key` are physically adjacent within a type's table
- `ALTER TABLE ADD COLUMN` / `DROP COLUMN` enables dynamic index management
- WAL mode: concurrent readers + one writer; suitable for v1

**SQLite configuration:**

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;  -- survives OS crash; not power-loss safe (see OQ-2B)
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;   -- ms; prevents immediate SQLITE_BUSY under write contention
```

---

## SQLite Interleave Limitation

Spanner physically co-locates parent and child rows in the same storage "split" across entity type tables. SQLite has no equivalent: `et_workspaces` and `et_folders` are separate B-trees. Within each table, entities sharing the same `root_key` are adjacent (due to PK workspaceing). Cross-table locality (workspaces and folders physically adjacent) requires a future storage engine that supports physical interleaving.

**This is documented as a known limitation, not a design flaw.** The API is designed to be distribution-ready (root_key = shard key) so a future engine can provide true interleaving without API changes.

---

## Canonical DDL

This section is the **single source of truth** for all table definitions. RFC-0001 and RFC-0003 reference this section — do not duplicate DDL elsewhere.

### System Tables

```sql
-- Proto descriptor registry (server-local; no external dependency)
CREATE TABLE schema_versions (
    schema_id    INTEGER PRIMARY KEY AUTOINCREMENT,
    type_url     TEXT    NOT NULL UNIQUE,
    -- Transitive closure of all .proto files needed to decode the target message.
    -- Stored as serialized google.protobuf.FileDescriptorSet bytes.
    descriptor   BLOB    NOT NULL,
    created_at   INTEGER NOT NULL  -- unix epoch millis
);

-- Entity type registry (one row per RegisterEntityType call)
CREATE TABLE entity_type_registry (
    entity_type   TEXT    PRIMARY KEY,
    parent_type   TEXT,             -- NULL for root entity types
    created_at    INTEGER NOT NULL
);

-- Proto column registry (one row per proto column per entity type)
CREATE TABLE proto_column_registry (
    entity_type   TEXT    NOT NULL REFERENCES entity_type_registry(entity_type),
    column_name   TEXT    NOT NULL,
    type_url      TEXT    NOT NULL,  -- initial type_url; may evolve via schema_versions
    required      INTEGER NOT NULL   -- 0 = optional, 1 = required (NOT NULL enforcement)
        CHECK (required IN (0, 1)),
    ordinal       INTEGER NOT NULL,  -- column workspace within the entity type
    created_at    INTEGER NOT NULL,
    PRIMARY KEY (entity_type, column_name)
);

-- Index registry (one row per AddIndex call)
CREATE TABLE index_registry (
    index_name    TEXT    PRIMARY KEY,
    entity_type   TEXT    NOT NULL REFERENCES entity_type_registry(entity_type),
    column_name   TEXT    NOT NULL,  -- which proto column the indexed field lives in
    field_path    TEXT    NOT NULL,  -- dot-separated, e.g. "name" or "address.city"
    field_number  INTEGER NOT NULL,  -- proto field number for efficient decode
    value_type    TEXT    NOT NULL   -- 'TEXT'|'INTEGER'|'REAL'|'BLOB'
        CHECK (value_type IN ('TEXT', 'INTEGER', 'REAL', 'BLOB')),
    null_policy   TEXT    NOT NULL DEFAULT 'SKIP'
        CHECK (null_policy IN ('SKIP', 'INDEX_NULL')),
    status        TEXT    NOT NULL DEFAULT 'BUILDING'
        CHECK (status IN ('READY', 'BUILDING', 'FAILED')),
    rows_indexed  INTEGER NOT NULL DEFAULT 0,
    rows_total    INTEGER NOT NULL DEFAULT 0,
    created_at    INTEGER NOT NULL,
    FOREIGN KEY (entity_type, column_name)
        REFERENCES proto_column_registry(entity_type, column_name)
);
```

---

### Entity Type Table Template

When `CreateEntityType` is called for entity type `"folders"` with proto columns `body` (required) and `meta` (optional), the server executes:

```sql
CREATE TABLE et_folders (
    -- Composite primary key
    root_key          TEXT    NOT NULL,
    entity_local_key  TEXT    NOT NULL,
    -- Proto column: body (required)
    body_data         BLOB    NOT NULL,   -- serialized proto bytes
    body_schema       INTEGER NOT NULL    -- schema_id from schema_versions
        REFERENCES schema_versions(schema_id),
    -- Proto column: meta (optional)
    meta_data         BLOB,               -- NULL if not provided
    meta_schema       INTEGER             -- NULL if meta_data is NULL
        REFERENCES schema_versions(schema_id),
    -- Entity metadata
    ttl               INTEGER,            -- unix epoch millis; NULL = no expiry
    version           INTEGER NOT NULL DEFAULT 1,  -- OCC counter
    created_at        INTEGER NOT NULL,
    updated_at        INTEGER NOT NULL,
    -- Materialized index columns (none at creation time; added via AddIndex)
    PRIMARY KEY (root_key, entity_local_key)
);

-- Standard secondary indexes for root-key access patterns
CREATE INDEX et_folders__root
    ON et_folders (root_key);

CREATE INDEX et_folders__root_type
    ON et_folders (root_key, entity_local_key);  -- covered by PK; may omit

CREATE INDEX et_folders__ttl
    ON et_folders (ttl) WHERE ttl IS NOT NULL;   -- for TTL sweeper efficiency
```

**Column naming conventions:**

| Column | Convention | Example |
|--------|-----------|---------|
| Proto bytes | `<col>_data` | `body_data`, `meta_data` |
| Proto schema version | `<col>_schema` | `body_schema`, `meta_schema` |
| Materialized index | `<col>__<field_path>` (double underscore; dots → underscores) | `body__name`, `body__address_city`, `meta__tenant_id` |

The double-underscore separator for index columns prevents ambiguity: `body__name` is unambiguously an index column on proto column `body`, field `name`.

---

### AddIndex DDL

When `AddIndex` is called to index `body.name` on `et_folders`:

```sql
-- Step 1: Add materialized column (O(1) — schema update only; existing rows get NULL)
ALTER TABLE et_folders ADD COLUMN body__name TEXT;

-- Step 2: Build SQLite B-tree index on the new column
CREATE INDEX et_folders__body__name
    ON et_folders (body__name);

-- Optional: composite index for root-scoped queries (most common access pattern)
CREATE INDEX et_folders__root_body__name
    ON et_folders (root_key, body__name);
```

After these DDL steps, the server runs the backfill loop (see RFC-0003 §Reindex).

---

### DropIndex DDL

```sql
-- Must drop SQLite indexes before dropping the column
DROP INDEX IF EXISTS et_folders__root_body__name;
DROP INDEX IF EXISTS et_folders__body__name;

-- Remove the materialized column (O(1) schema update; space reclaimed on VACUUM)
ALTER TABLE et_folders DROP COLUMN body__name;
```

---

## Write Atomicity

Every entity write (Insert, Upsert, Update, Delete) wraps all of the following in a single SQLite transaction:

1. Write the entity row (including all materialized index columns)
2. Update `entity_type_registry` / `index_registry` if needed (admin operations only)

There are no separate index table writes — all data is in the entity table. This makes write atomicity trivially correct: one row, one transaction.

---

## Open Questions

### OQ-2F: libSQL as an alternative to SQLite / rusqlite

**libSQL** (by [Turso](https://turso.tech)) is an open-source Apache-licensed fork of SQLite. Assessment for dxdb:

| Property | rusqlite (bundled) | libsql crate |
|----------|--------------------|-------------|
| SQLite version | 3.45+ (compiled in) | libSQL fork of SQLite (3.44 base + patches) |
| Rust API style | Synchronous (blocking); use `spawn_blocking` with tokio | **Async-first** (tokio-native) |
| Single-binary | ✓ (`bundled` feature compiles SQLite in) | ✓ (similar bundled mode) |
| Maturity | 25+ years of SQLite; rusqlite is stable | libSQL fork is ~2 years old; production-used by Turso |
| C FFI surface | Yes (SQLite C library) | Yes (same) |
| Unique additions | — | HTTP/WebSocket server (`sqld`), embedded replicas, WASM UDFs, vector search |
| Build complexity | Low | Slightly higher (fork diverges from upstream SQLite) |

**Does dxdb need libSQL's unique additions?**

- **HTTP/WebSocket server (`sqld`)**: No — dxdb provides its own gRPC server. libSQL's wire protocol would be a parallel, unused endpoint.
- **Embedded replicas** (local libSQL file + remote `sqld` primary): Interesting for a "read replica" story — local reads, writes routed to a central primary. But this is Turso's specific architecture, not Spanner-style sharding by `root_key`. It doesn't advance the declared distribution goal.
- **WASM UDFs**: Not needed — field extraction is handled by prost-reflect in the Rust server layer, not pushed into SQLite.
- **Vector search**: Out of scope for dxdb.

**Async API advantage:** `rusqlite` is synchronous. In a tokio server, SQLite calls must go through `spawn_blocking`. This is standard practice and well-understood; `tokio-rusqlite` wraps this cleanly. `libsql` is natively async, which is ergonomically cleaner in tonic handlers. This is the only substantive technical differentiator.

**Verdict:** Use `rusqlite` (bundled) for v1. The async ergonomics difference is real but manageable via `tokio-rusqlite`. libSQL's distributed features don't map to dxdb's distribution model. Revisit if the async overhead in `spawn_blocking` becomes a measured bottleneck or if the embedded-replica model becomes attractive for v2.

### OQ-2A: `redb` as an alternative to SQLite

`redb` (pure Rust, no C FFI, ACID, embedded B-tree) eliminates the C FFI surface. Trade-offs:
- No SQL — dynamic schema (ADD/DROP COLUMN) must be hand-implemented as typed B-tree tables
- Less mature than SQLite
- No `ALTER TABLE` equivalent — adding a "column" means restructuring the B-tree layout

**Is eliminating C FFI a hard requirement?** If yes, `redb` needs a separate design for materialized column management.

### OQ-2B: Durability guarantee

- `synchronous = NORMAL` (proposed): survives OS crash; not power-loss safe
- `synchronous = FULL`: fsync per WAL commit; 2–5× slower
- `synchronous = OFF`: fastest; data loss on any crash

**What durability SLA does dxdb commit to for v1?**

### OQ-2C: `AddEntityType` with existing data migration

If an entity type is modified after creation (adding a new proto column), the entity table must be `ALTER TABLE`'d to add `<col>_data` and `<col>_schema` columns. Existing rows get NULL for the new column. Is this in scope for v1?

### OQ-2D: Backup / restore

Mechanism options:
- SQLite `.backup` API via `rusqlite` — hot backup, streams to a file
- `VACUUM INTO '<dest>'` — consistent snapshot without WAL
- gRPC streaming export — schema + entity data as a proto stream

### OQ-2E: Maximum practical entity table width

With many indexes per entity type, the entity table grows wide (one materialized column per index). SQLite supports up to 32767 columns per table, but practical performance degrades before that. Is there a per-entity-type index count limit to document?

---

## Consequences

- One-table-per-entity-type cleanly isolates materialized index columns: adding an index on `folders` only widens `et_folders`, not all entity tables.
- Dynamic DDL (`ALTER TABLE`) is the index management mechanism — no separate index tables, no cross-table joins needed for index-based queries.
- The B-tree workspaceing `(root_key, entity_local_key)` gives within-table locality for `ScanByRoot` — a range scan on `root_key = ?` is O(children_count), not O(total_rows).

