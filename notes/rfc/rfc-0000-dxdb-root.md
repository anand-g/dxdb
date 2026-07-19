# RFC-0000: dxdb — Root Design RFC

| Field       | Value                          |
|-------------|--------------------------------|
| Status      | **Draft**                      |
| Authors     | anandg                         |
| Created     | 2026-07-03                     |
| Last Update | 2026-07-18                     |

---

## Summary

`dxdb` is a schema-driven mutable document store with a **hierarchical entity model** (initially supporting Protocol Buffers as the first of potential formats, such as JSON or FlatBuffers, to be supported in the future). Each entity is identified by a 3-part composite key. Entity types are registered as named schemas (analogous to `CREATE TABLE`) that define a fixed set of **proto columns** — each column storing a serialized Protocol Buffer message of a registered type.

The system exposes a gRPC endpoint and ships as a zero dependency single binary.

This root RFC establishes vocabulary and cross-cutting concerns. Each subsystem is detailed in a dedicated child RFC.

### Child RFCs

| # | Subsystem | Status |
|---|-----------|--------|
| [RFC-0001](RFC-0001-schema-row-model.md) | Schema & Record Model |
| [RFC-0002](RFC-0002-storage-engine.md) | Storage Engine |
| [RFC-0003](RFC-0003-index-ops.md) | Indexing & Reindex |
| [RFC-0004](RFC-0004-api-grpc.md) | gRPC API |
| [RFC-0005](RFC-0005-cli.md) | CLI & Single Binary |
| [RFC-0006](RFC-0006-entity-registry.md) | Entity Registry |

---

## Motivation

Protocol Buffers are widely used for typed, evolvable serialization, yet existing databases treat them as opaque blobs or force a single flat schema across all rows. As the first of potential schema/serialization formats supported by the system, `dxdb` treats the protobuf descriptor as a first-class citizen: it drives per-column schema identity, field indexing, and query planning.

The driving use case is a **general-purpose mutable document store** where:
- Entities have a natural parent-child hierarchy (e.g. workspace → workspace lines)
- Each entity type has its own fixed set of typed proto columns (like a schema-per-table)
- Multiple proto columns per entity type are supported from day one (e.g. `body`, `meta`)

### Clarification: "Each row may have a different schema version"

The overview states *"each row may have a different schema **version**."* This is satisfied directly by the `<col>_schema` column (a `schema_id`) stored per proto column per entity row: rows written before a schema upgrade carry the old `schema_id`; rows written after carry the new one. Different rows of the same entity type may therefore reference different schema versions of the same proto message type — this is a first-class design property, not an edge case.

Within a single entity type, all entities share the same **set** of proto columns (defined at `CreateEntityType` time). Schema version heterogeneity is per-column, per-row — not per entity type.

---

## Resolved Design Decisions

These are closed. A new RFC is required to reopen any of them.

| Decision | Answer |
|----------|--------|
| **Use case** | General-purpose mutable document store with hierarchical entity model |
| **Language** | Rust — `tonic`, `prost`, `prost-reflect`, `rusqlite` (bundled) |
| **Schema identity** | Server-local `schema_versions` system table (Spanner-style); no external dependency |
| **CLI architecture** | gRPC client only — always connects to a running `dxdb serve` |
| **Multiple proto columns per entity** | Yes — named proto columns defined per entity type (see RFC-0006) |
| **Index storage** | Materialized columns in the entity table — `ALTER TABLE ADD/DROP COLUMN` |
| **Physical layout** | One SQLite table per entity type |
| **Optimistic concurrency** | Yes — every entity has a `version` INTEGER; updates supply `expected_version` |
| **TTL** | First-class `ttl` column in every entity table |
| **Parent/child referential integrity** | None on insert; application manages cascade cleanup (TTL or explicit delete) |
| **Async reindex** | Yes — `AddIndex` (TBD `Reindex`)returns immediately with status `BUILDING`; caller polls |

---

## Terminology

| Term | Definition |
|------|-----------|
| **Entity** | The atomic unit of storage. Identified by an `EntityKey`. Has fixed proto columns as defined by its entity type registration. |
| **Entity Type** | A named, registered schema definition — analogous to a table. Defines the proto columns, parent type, and key structure. See RFC-0006. |
| **Entity Key** | The 3-tuple `(entity_type, root_key, entity_local_key)` uniquely identifying an entity. |
| **Root Key** | The aggregate-root partition identifier (e.g. `"workspace-108"`). All entities sharing a root key form a logical hierarchy and map to the same future shard. |
| **Entity Local Key** | The identity of this entity within `(entity_type, root_key)`. |
| **Proto Column** | A named, typed field within an entity type. Stores a serialized proto message of a registered schema. Each entity type has ≥1 proto columns. Analogous to a typed column in a relational table, but the column type is a proto message. |
| **Schema** | A protobuf `FileDescriptorSet` + fully-qualified message name, registered in the `schema_versions` system table. Identified by a `schema_id`. |
| **Schema Version** | A specific registration of a proto message type. The same logical type may have multiple schema versions (e.g. `Folder/v1` = schema_id 7, `Folder/v2` = schema_id 12). |
| **Materialized Index Column** | A SQLite column in the entity table storing the pre-extracted value of an indexed proto field. Named `<column_name>__<field_path>`. Added/removed via `ALTER TABLE`. |
| **OCC Version** | An integer counter on each entity row, incremented on every write. `UpdateRequest` supplies an `expected_version`; a mismatch returns `ABORTED`. |
| **TTL** | A `ttl INTEGER` column (unix epoch millis) on every entity table. `NULL` = no expiry. Enforcement mechanism is an open question (RFC-0001 §OQ-1E). |

---

## The Entity Key

```
EntityKey {
  entity_type:      string          // selects the physical table
  root_key:         string | absent // ABSENT for root entity types;
                                    // for all other depths: the root ancestor's
                                    // entity_local_key (never an intermediate parent's key)
  entity_local_key: string          // identity within (entity_type, [root_key])
}
```

### Root entities

Root entity types (`parent_type = NULL` in the registry) have **no `root_key`**:

```
EntityKey { entity_type: "workspaces", entity_local_key: "workspace-108" }
```

The physical table `et_workspaces` has a **single-component PK**: `(entity_local_key)`.

### Non-root entities — all depths

Every descendant at any depth carries the **same `root_key`**: the root ancestor's `entity_local_key`. It is never the immediate parent's key.

```
// Direct child (folder under workspace-108)
EntityKey { entity_type: "folders", root_key: "workspace-108", entity_local_key: "1" }

// Grandchild (document, descendant of an folder — NOT of the folder)
EntityKey { entity_type: "documents", root_key: "workspace-108", entity_local_key: "folder-5/doc-1" }
//                                    ^^^^^^^^^^^^^^ still the Workspace's key at any depth
```

The intermediate ancestor's identity is the application's responsibility to embed in `entity_local_key`. dxdb treats `entity_local_key` as an opaque string.

Non-root entity type tables have a **two-component PK**: `(root_key, entity_local_key)`.

**Root key = future shard key.** All entities with the same `root_key` must be processable on a single shard. The API must never require cross-root-key operations as a primary access pattern.

**No referential integrity on insert.** Writing `(folders, workspace-108, 1)` does not require the `workspaces` entity `workspace-108` to exist.

---

## Cross-Cutting Design Constraints

All child RFCs must satisfy:

1. **Fixed proto columns per entity type** — entity types are registered upfront (DDL); all entities of a type have the same columns.
2. **Materialized index columns** — indexed field values are stored as extra columns in the entity table; no separate index tables.
3. **One SQLite table per entity type** — column explosion across entity types is avoided by isolation.
4. **gRPC-only network interface** — no REST, no SQL wire protocol.
5. **Rust single binary** — same executable for `dxdb serve` and `dxdb <command>`.
6. **OCC** — every entity write is conditional on `expected_version` (0 = skip check).
7. **Write atomicity** — all index column updates happen within the same SQLite write transaction as the entity row.
8. **Root-key locality** — all multi-entity access patterns must use `root_key` as the primary filter; full-table scans without `root_key` are explicitly not guaranteed to be performant.
9. **Permitted SQL patterns** — the server's internal SQL must only use: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `BEGIN`/`COMMIT`, `CREATE TABLE`, `ALTER TABLE`, `CREATE INDEX`, `DROP INDEX`, `DROP TABLE`. The following constructs are **prohibited without explicit architectural debate and RFC update**: `JOIN`, `UNION` / `UNION ALL`, `IN` (subquery form), `EXISTS`, `WITH` (CTEs), correlated subqueries. Rationale: these constructs cross entity-type table boundaries or create query plans that cannot be trivially replaced by a future distributed storage engine. Simple range scans on a single table with `WHERE root_key = ?` are the canonical access pattern.

---

## Global Database Invariants

These are correctness invariants that the server must enforce at runtime and that tests must assert. Violations indicate a bug. New invariants must be added here first, then referenced in the relevant child RFC.

| # | Invariant | Enforced at | RFC |
|---|-----------|------------|-----|
| I-1 | Entity type names are globally unique within a database | `CreateEntityType` | RFC-0006 §OQ-6E |
| I-2 | A non-root entity type must reference an existing `parent_type` | `CreateEntityType` | RFC-0006 |
| I-3 | A root entity's `EntityKey` must have empty `root_key`; a non-root entity's must have non-empty `root_key` | Every write RPC | RFC-0001 |
| I-4 | Every entity write must update all READY index materialized columns in the same transaction | Insert / Upsert / Update / Delete | RFC-0003 |
| I-5 | A proto column's `<col>_schema` must reference an existing `schema_id` in `schema_versions` | Every write RPC | RFC-0001 |
| I-6 | An index's `field_path` must resolve to a scalar leaf field in the column's registered descriptor | `AddIndex` | RFC-0003 |
| I-7 | `root_key` in a non-root entity row always equals the `entity_local_key` of its root ancestor — never an intermediate parent's key | Write path (asserted by entity type hierarchy) | RFC-0001, RFC-0006 |
| I-8 | The SQL patterns `JOIN`, `UNION`, `IN` (subquery), `EXISTS`, `WITH` are not used in server-generated queries | Code review / static analysis | RFC-0000 constraint #9 |

> **Process:** when a new invariant is identified in discussion, add it here immediately, then reference it from the relevant child RFC. This section is the single source of truth for runtime correctness properties.

---

## Non-Goals (initial scope)

- SQL or SQL-compatible query language
- Distributed / sharded deployment (design is distribution-ready by API contract; not implemented)
- True physical storage interleaving across entity type tables (Spanner-style; not achievable in SQLite — see RFC-0002 §SQLite Interleave Limitation)
- Schema evolution enforcement (proto wire compatibility is the caller's responsibility)
- Referential integrity between parent and child entities
- Authentication / authorization
- Full-text search

---

## References

- [Protocol Buffers — FileDescriptorSet](https://protobuf.dev/reference/protobuf/descriptor.proto/)
- [tonic — Rust gRPC](https://github.com/hyperium/tonic)
- [prost-reflect — Rust dynamic proto reflection](https://github.com/andrewhickman/prost-reflect)
- [rusqlite — Rust SQLite bindings](https://github.com/rusqlite/rusqlite)
- [Cloud Spanner schema and data model](https://cloud.google.com/spanner/docs/schema-and-data-model)
- [SQLite ALTER TABLE](https://www.sqlite.org/lang_altertable.html)

