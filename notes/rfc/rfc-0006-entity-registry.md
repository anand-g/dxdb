# RFC-0006: Entity Type Registry (DDL)

| Field       | Value                          |
|-------------|--------------------------------|
| Status      | **Draft**                      |
| Parent      | [RFC-0000](RFC-0000-dxdb-root.md) |
| Authors     | anandg                         |
| Created     | 2026-07-03                     |
| Last Update | 2026-07-18                     |

> **Note:** This RFC supersedes the earlier exploratory RFC-0006-multi-message-slots.md, which described named proto slots. The design has been refined: slots are now called **proto columns**, and they are declared per entity type at registration time (DDL), not as a flexible per-row map.

---

## Summary

This RFC defines the entity type registry — the DDL layer of dxdb. Registering an entity type is the analog of `CREATE TABLE`: it defines the proto columns for that entity type and creates the physical SQLite table. Dxdb has a schema, not just a key-value namespace.

---

## Motivation

Without a type registry:
- Proto columns would be a flexible per-row `map<string, ColumnData>` — any row could have any columns
- Materialized index columns (RFC-0003) would be impossible: you can't `ALTER TABLE ADD COLUMN body__name` if you don't know which tables have a `body` column
- One-table-per-entity-type (RFC-0002) requires knowing the entity type at table-creation time

The registry ties everything together: entity type definition → physical table DDL → index column management.

---

## Entity Type Definition

An entity type has:

```
EntityType {
  name:        string          // unique identifier, e.g. "folders"
  parent_type: string?         // e.g. "workspaces"; null for root entity types
  columns:     ProtoColumnDef[] // workspaceed list of proto columns; ≥ 1
  created_at:  Timestamp
}

ProtoColumnDef {
  column_name: string          // e.g. "body", "meta"
  type_url:    string          // initial type_url; schema may evolve via schema_versions
  required:    bool            // true → NOT NULL in SQLite; NULL data rejected on write
  ordinal:     int32           // column position (server assigns if 0)
}
```

---

## Hierarchy: Parent-Child Relationships

Entity types form a tree via `parent_type`. This mirrors Spanner's interleaved table hierarchy:

```
workspaces          (root entity type; parent_type = null)
├── folders  (parent_type = "workspaces")
├── folders   (parent_type = "workspaces")
│   └── documents (parent_type = "folders")
└── members    (parent_type = "workspaces")
```

**Semantics:**
- The `root_key` is always the identity of the **root** entity type instance (e.g. `"workspace-108"` for the `workspaces` entity type).
- Child entities reference their root via `root_key`. No referential integrity is enforced on write.
- The `entity_type` field in `EntityKey` routes the operation to the correct physical table.
- Deeper hierarchies (e.g. `folders → documents`) encode the intermediate key in `entity_local_key` by convention (e.g. `entity_local_key = "folder-5/document-1"`), or use `root_key` as a compound key. See OQ-6C.

**Root entity representation:** A root entity (e.g. an workspace) is an entity of its root type (`workspaces`) with `entity_local_key = root_key`:

```
EntityKey { entity_type="workspaces", root_key="workspace-108", entity_local_key="workspace-108" }
```

This gives the root entity its own proto columns (e.g. workspace status, customer ID) while sharing the `root_key` with all children.

---

## Registration Effect: Physical DDL

`CreateEntityType` is idempotent on identical inputs (`entity_type` name + same column definitions). It executes:

### For a root entity type (`workspaces`, columns: `body` required)

Root entity types have `parent_type = NULL`. Their physical table has a **single-component PK** — no `root_key` column.

```sql
INSERT INTO entity_type_registry (entity_type, parent_type, created_at)
VALUES ('workspaces', NULL, <now>);

INSERT INTO proto_column_registry
    (entity_type, column_name, type_url, required, ordinal, created_at)
VALUES ('workspaces', 'body', 'type.googleapis.com/pkg.Workspace', 1, 1, <now>);

-- Single-component PK: entity_local_key only; no root_key column
CREATE TABLE et_workspaces (
    entity_local_key  TEXT    NOT NULL,   -- "workspace-108"; this IS the root identity
    body_data         BLOB    NOT NULL,
    body_schema       INTEGER NOT NULL
        REFERENCES schema_versions(schema_id),
    ttl               INTEGER,
    version           INTEGER NOT NULL DEFAULT 1,
    created_at        INTEGER NOT NULL,
    updated_at        INTEGER NOT NULL,
    PRIMARY KEY (entity_local_key)
);

CREATE INDEX et_workspaces__ttl ON et_workspaces (ttl) WHERE ttl IS NOT NULL;
-- No __root index: root entities are addressed directly by entity_local_key
```

### For a child entity type (`folders`, parent: `workspaces`, columns: `body` required, `meta` optional)

All non-root entity types have a **two-component PK**: `(root_key, entity_local_key)`.
`root_key` is always the **root ancestor's** `entity_local_key` — never the immediate parent's key.

```sql
INSERT INTO entity_type_registry VALUES ('folders', 'workspaces', <now>);

INSERT INTO proto_column_registry VALUES
    ('folders', 'body', 'type.googleapis.com/pkg.Folder', 1, 1, <now>),
    ('folders', 'meta', 'type.googleapis.com/pkg.EntityMeta', 0, 2, <now>);

CREATE TABLE et_folders (
    root_key          TEXT    NOT NULL,   -- always the Workspace's entity_local_key ("workspace-108")
    entity_local_key  TEXT    NOT NULL,   -- identity within this type under that root ("1")
    body_data         BLOB    NOT NULL,
    body_schema       INTEGER NOT NULL
        REFERENCES schema_versions(schema_id),
    meta_data         BLOB,
    meta_schema       INTEGER
        REFERENCES schema_versions(schema_id),
    ttl               INTEGER,
    version           INTEGER NOT NULL DEFAULT 1,
    created_at        INTEGER NOT NULL,
    updated_at        INTEGER NOT NULL,
    PRIMARY KEY (root_key, entity_local_key)
);

CREATE INDEX et_folders__root ON et_folders (root_key);
CREATE INDEX et_folders__ttl  ON et_folders (ttl) WHERE ttl IS NOT NULL;
```

No SQLite `FOREIGN KEY` from `et_folders.root_key` to `et_workspaces.entity_local_key` — no referential integrity enforcement on write.

### For a grandchild entity type (`documents`, parent: `folders`, grandparent: `workspaces`)

`root_key` is **still** `workspace-108` — the root's key, not the intermediate folder's key.
The folder identity is embedded in `entity_local_key` by the application.

```sql
INSERT INTO entity_type_registry VALUES ('documents', 'folders', <now>);

CREATE TABLE et_documents (
    root_key          TEXT    NOT NULL,   -- "workspace-108" — the Workspace's key, not the Folder's
    entity_local_key  TEXT    NOT NULL,   -- application encodes path, e.g. "folder-5/doc-1"
    body_data         BLOB    NOT NULL,
    body_schema       INTEGER NOT NULL
        REFERENCES schema_versions(schema_id),
    ttl               INTEGER,
    version           INTEGER NOT NULL DEFAULT 1,
    created_at        INTEGER NOT NULL,
    updated_at        INTEGER NOT NULL,
    PRIMARY KEY (root_key, entity_local_key)
);

CREATE INDEX et_documents__root ON et_documents (root_key);
CREATE INDEX et_documents__ttl  ON et_documents (ttl) WHERE ttl IS NOT NULL;
```

---

## Proto Column Naming Conventions (Canonical)

| What | Column name | Example |
|------|------------|---------|
| Proto bytes | `<col>_data` | `body_data`, `meta_data` |
| Schema ID | `<col>_schema` | `body_schema`, `meta_schema` |
| Materialized index (scalar) | `<col>__<field_path>` (dots → underscores) | `body__name`, `body__address_city`, `meta__tenant_id` |

This convention is defined here and is the canonical reference for RFC-0002 and RFC-0003.

---

## Schema Evolution at the Column Level

Each proto column tracks its current encoding via `<col>_schema` (a `schema_id`). When `Folder/v2` is registered (new `schema_id = 12`), writes using v2 store `body_schema = 12`. Old rows retain `body_schema = 7`. The server uses `<col>_schema` per-row to select the correct `FileDescriptorSet` for decoding (e.g. when serving a `Get` with `protojson` output, or when extracting index fields during reindex).

This means the column's physical type does not change across schema versions — only the interpretation of `body_data` changes, guided by `body_schema`.

---

## Open Questions

### OQ-6A: Explicit `CreateEntityType` vs. implicit-on-first-insert

**Option A — Explicit DDL (current proposal):** `CreateEntityType` must be called before any `Insert`. This is analogous to `CREATE TABLE` in SQL — the schema must exist before data is written.

**Option B — Implicit on first insert:** If `entity_type` does not exist in the registry, `Insert` auto-registers it based on the provided column names and schema IDs. Lower friction for getting started; harder to enforce column consistency (two callers may insert with different column sets for the same entity type).

Recommendation: explicit DDL (Option A) for v1. Implicit registration can be added as a convenience flag (`--create-if-not-exists`) in a future version.

### OQ-6B: Adding a proto column to an existing entity type

What happens when a user wants to add a new proto column `ext` to `folders` after the type is registered?

```
ALTER TABLE et_folders ADD COLUMN ext_data BLOB;
ALTER TABLE et_folders ADD COLUMN ext_schema INTEGER REFERENCES schema_versions(schema_id);
INSERT INTO proto_column_registry VALUES ('folders', 'ext', ..., 0, 3, <now>);
```

Existing rows have `ext_data = NULL`, `ext_schema = NULL` (column is optional by default). Is `AlterEntityType` (add column) in scope for v1?

### OQ-6C: Deeper hierarchy key encoding

The current model supports 2 levels (root type → child types). Deeper hierarchies (root → child → grandchild) require the application to encode the intermediate key in `entity_local_key`. Should dxdb formalize multi-level hierarchy with additional key components, or leave it to application convention?

Example (application convention):
```
EntityKey {
  entity_type:      "documents"
  root_key:         "workspace-108"          // the root
  entity_local_key: "folder-5/doc-1"   // encodes the intermediate level
}
```

Formalizing adds a `parent_local_key` component to `EntityKey` but enables server-side grandchild queries.

### OQ-6D: Entity type deletion

Should `DropEntityType` (analogous to `DROP TABLE`) be in scope for v1? This would:
1. Validate no entities exist (or force-drop with `--cascade`)
2. Drop all associated indexes
3. `DROP TABLE et_<name>`
4. Remove from `entity_type_registry` and `proto_column_registry`

### OQ-6E: Entity type naming constraints

**Global uniqueness invariant (closed):** Entity type names are globally unique within a database — `entity_type` is the primary key of `entity_type_registry`. Two independent entity type hierarchies that both need a child type named `"line_items"` must use distinct names (e.g. `"workspace_line_items"` and `"invoice_line_items"`). This is a naming constraint, not a structural one.

**Character set (open):** Should `entity_type` names be restricted (e.g. `[a-z][a-z0-9_]*`, max 64 chars)? The name is used directly in SQLite table names (`et_<name>`), so characters that are invalid in SQLite identifiers must be rejected or escaped at `CreateEntityType` time.

---

## Consequences

- `CreateEntityType` is a one-time DDL operation with permanent effects (physical SQLite table creation). Operators should treat it with the same care as `CREATE TABLE` in production SQL databases.
- RFC-0002's DDL section is the canonical source for the exact `CREATE TABLE` SQL. This RFC defines the logical model; RFC-0002 defines the physical implementation.
- The `parent_type` field in the entity type registry is metadata only in v1 — it documents the intended hierarchy and enables `ListEntityTypes(parent_type=?)` queries, but does not enforce FK constraints.

