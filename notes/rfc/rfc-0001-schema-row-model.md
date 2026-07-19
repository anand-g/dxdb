# RFC-0001: Schema & Record Model

| Field       | Value                          |
|-------------|--------------------------------|
| Status      | **Draft**                      |
| Parent      | [RFC-0000](RFC-0000-dxdb-root.md) |
| Authors     | anandg                         |
| Created     | 2026-07-03                     |
| Last Update | 2026-07-18                     |

---

## Summary

This RFC defines the **entity** (row) model: its physical structure within an entity type table, the schema version registry that gives each proto column its type, and the mutation semantics of a mutable document store with optimistic concurrency control and TTL.

---

## Entity Structure

An entity is an instance of a registered entity type. Its physical layout in the SQLite table is:

```
[root_key] [entity_local_key]          -- composite primary key
[<col>_data] [<col>_schema]            -- one pair per proto column
...
[ttl]                                   -- unix epoch millis; NULL = no expiry
[version]                               -- OCC counter, starts at 1
[created_at] [updated_at]              -- unix epoch millis
[<col>__<field>] ...                   -- materialized index columns (zero or more)
```

### Example: `folders` entity type (non-root)

Entity type `folders` has two proto columns: `body` (type `Folder`) and `meta` (type `EntityMeta`). Two indexes declared: `body.name` and `meta.tenant_id`.

```sql
-- Physical row in et_folders (two-component PK):
root_key:         "workspace-108"   -- Workspace's entity_local_key; never an intermediate key
entity_local_key: "1"
body_data:        <proto bytes for Folder>
body_schema:      7                          -- schema_id in schema_versions
meta_data:        <proto bytes for EntityMeta>
meta_schema:      3
ttl:              NULL                       -- no expiry
version:          4                          -- updated 4 times
created_at:       1721299200000
updated_at:       1721385600000
body__name:        "ABC-123"                  -- materialized index column
meta__tenant_id:  "acme"                     -- materialized index column
```

---

## Schema Version Registry

The `schema_versions` system table is the server-local authority for all proto descriptors. No external schema registry is required.

### Registration

The caller provides a `FileDescriptorSet` (the transitive closure of all `.proto` files required to decode the target message type) plus the fully-qualified message name.

**Important:** A `FileDescriptorSet` (not bare `FileDescriptorProto`) is required because the target message may import other `.proto` files (e.g. `google/protobuf/timestamp.proto`, shared common types). All imports must be included in the transitive set.

The server:
1. Parses the `FileDescriptorSet`, locates the named message descriptor.
2. Assigns a `schema_id` (auto-increment integer).
3. Stores `(schema_id, type_url, descriptor_set_bytes)` in `schema_versions`.
4. Returns `schema_id` and `type_url`.

### Schema Identity

- `schema_id` — compact `uint64`, stored in each entity row as `<col>_schema`.
- `type_url` — `"type.googleapis.com/{fully.qualified.MessageName}"`.

### Schema Versioning

Each `RegisterSchema` call creates a new `schema_id` even if the `type_url` already exists. `Folder/v1` and `Folder/v2` coexist. A proto column's `<col>_schema` may differ between rows if some rows were written before a schema upgrade and others after.

**Proto field number immutability contract:** dxdb assumes callers follow protobuf evolution rules — field numbers are never reused or renumbered. This contract guarantees that materialized index column values remain valid across schema versions: `body__name` extracted with schema_id=7 is identical to `body__name` extracted with schema_id=12 for the same encoded field (field numbers don't change). New fields added in v2 will be absent (NULL) in rows encoded with v1 — handled by `null_policy` on the index. See RFC-0003 §Sparse Index Semantics.

---

## Mutation Semantics

dxdb is a **mutable document store** with last-write-wins semantics, guarded by optional optimistic concurrency control.

| Operation | Behavior |
|-----------|---------|
| `Insert` | Creates a new entity. Fails with `ALREADY_EXISTS` if the key exists. Sets `version = 1`. |
| `Upsert` | Insert-or-update. Creates with `version = 1`; on existing key, replaces all proto columns and increments `version`. |
| `Update` | Replaces proto columns on an existing entity. Fails with `NOT_FOUND` if absent. Increments `version`. Optionally validates `expected_version`. |
| `Delete` | Removes the entity row. Hard delete. `version` check optional. |

### Optimistic Concurrency Control (OCC)

Every entity row has a `version INTEGER NOT NULL DEFAULT 1`. On `Update` or `Delete`:

- If `expected_version = 0`: no OCC check — write proceeds unconditionally (last-write-wins).
- If `expected_version > 0`: server checks `actual_version == expected_version`. Mismatch → gRPC `ABORTED` status. Caller retries with a fresh `Get` to obtain the current version.

Reads always return the current `version` in `EntityRecord`. A typical OCC flow:

```
Get(key) → EntityRecord { version: 4, ... }
Update(key, expected_version: 4, new_data) → OK (version becomes 5)
                                           → ABORTED if another writer changed it first
```

### Partial column update

`Update` replaces **all** proto columns (full replace). Partial column update (patch one column, leave others) is an open question — see OQ-1D.

---

## TTL

Every entity table has a `ttl INTEGER` column (unix epoch millis; `NULL` = no expiry). Entities with `ttl < now()` are logically expired. How expired entities are physically removed is an open question — see OQ-1E.

TTL is set per-entity at insert/upsert/update time. It is not an entity-type-wide setting.

---

## Hierarchical Model

The 3-part key encodes a 2-level hierarchy by convention:

```
entity_type     → selects the physical table (e.g. et_folders)
root_key        → aggregate root / partition (e.g. "workspace-108")
entity_local_key → identity within (entity_type, root_key) (e.g. "1")
```

**Root entities** (type registered with `parent_type = NULL`) have no `root_key` — their table PK is `(entity_local_key)` alone. **All non-root descendants** at any depth carry `root_key` = the root ancestor's `entity_local_key`. It is never the immediate parent's key. No referential integrity is enforced on write.

Deeper hierarchies are encoded by the application in the key strings (e.g. `entity_local_key = "folder-5/document-1"`). dxdb does not interpret the content of key components.

---

## Open Questions

### OQ-1A: Entity local key type

Currently `TEXT`. Should `entity_local_key` support integer workspaceing (e.g. sort folders by line number numerically)? Options:

- Keep `TEXT` — caller zero-pads integers (`"001"`, `"002"`) for lexicographic workspaceing
- Add `INTEGER` variant via `oneof` in the API, stored as `INTEGER` in SQLite — enables native numeric range scans

### OQ-1B: Root entity convention

When a root entity `(workspaces, workspace-108, workspace-108)` is deleted, children are NOT automatically deleted (application manages cleanup). Should `ScanByRoot` warn or error if no root entity exists? Or is root-entity-existence entirely untracked?

### OQ-1C: Required vs. optional proto columns

Can a required proto column's `_data` be NULL? Options:
- All proto columns are nullable (a row may omit any column's data)
- Required columns enforced at write time: `NOT NULL` in SQLite schema
- Configurable per column at entity type registration time

### OQ-1D: Partial column update

`Update` replaces all proto columns. Should there be a `PatchColumn(key, col_name, schema_id, data)` RPC for updating a single column without touching others?

Requires: read the current `version`, update only the named column's `_data` + `_schema` + its materialized index columns, increment `version`. No other columns are read or written.

### OQ-1E: TTL enforcement mechanism

Two options:
- **Background sweeper**: a server background task periodically scans all entity tables for `ttl < now()` and deletes expired rows. Simple; expired rows may linger briefly.
- **Lazy-on-read**: reads check `ttl < now()` and return `NOT_FOUND` for expired entities (without deleting). Physical cleanup happens out-of-band (VACUUM or explicit sweep). Reads are slightly slower; storage is not immediately reclaimed.

---

## Consequences

- The fixed-schema-per-entity-type model (vs. flexible per-row schema) simplifies index management: the set of materialized index columns for `et_folders` is fixed by declared indexes on `folders` entity type. No cross-type column pollution.
- The `<col>_schema` column per proto column enables per-row schema version tracking, supporting mixed-version reads after a schema upgrade without rewriting old rows.
- RFC-0002 (DDL) is the canonical source for physical table creation and column naming conventions.

