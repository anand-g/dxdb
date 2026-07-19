# RFC-0004: gRPC API

| Field       | Value                          |
|-------------|--------------------------------|
| Status      | **Draft**                      |
| Parent      | [RFC-0000](RFC-0000-dxdb-root.md) |
| Authors     | anandg                         |
| Created     | 2026-07-03                     |
| Last Update | 2026-07-18                     |

---

## Summary

This RFC defines the gRPC service definition for dxdb. Four services cover schema management, entity type registration (DDL), entity CRUD, and index management. Implemented in Rust using `tonic` / `tonic-build` / `prost`.

**Design directions (not yet finalized — see OQ-4J):**
- CRUD operations should be **declarative** (express desired state; `Upsert` not `Insert`+`Update`)
- DDL and DML should be expressible via a **simple text DSL** with named parameters, in addition to structured proto RPCs
- The query model should embrace **path-based hierarchy** (Firestore-style) with `INCLUDE ANCESTOR` result enrichment and `ANCESTOR(kind, id)` scoping clauses (Datastore GQL-style)

---

## Design Reference: Learnings from Peer Systems

### From HRANA v3 (libSQL) — proto value encoding

HRANA's `Value` type is the reference design for encoding SQLite scalar values in proto. dxdb should adopt the same pattern for its `ScalarValue` message (used in index filters and DSL parameters):

```proto
// HRANA pattern — adopt for dxdb ScalarValue
message Value {
  oneof value {
    Null   null    = 1;  // explicit Null wrapper: distinguishes "null" from proto default
    sint64 integer = 2;  // zigzag encoding — efficient for signed integers (not int64)
    double float   = 3;
    string text    = 4;
    bytes  blob    = 5;
  }
  message Null {}
}
```

**Key decisions adopted from HRANA:**
- `sint64` (zigzag) instead of `int64` for integer values — more compact for negative numbers (line items, quantities, timestamps as deltas)
- Explicit `Null` inner message — unambiguously encodes SQL NULL vs. proto field-not-set
- Named parameters (`NamedArg { name: string, value: Value }`) — preferred over positional in the DSL
- `want_rows: bool` hint — suppresses row payload for write-only or keys-only operations
- Statement caching by ID (`sql_id`) — server caches parsed DSL statements; clients reference by integer ID on repeated calls
- Conditional batch execution (`BatchCond`) — maps to OCC: execute a mutation only if `version = @expected`
- Cursor-based streaming (`OpenCursor` / `FetchCursor` / `CloseCursor`) — client pulls result batches on demand; better backpressure than gRPC server streaming

### From Google Cloud Datastore GQL — hierarchical query syntax

Datastore's GQL provides the reference for ancestor-scoped queries:

```sql
-- Scope a query to a specific root entity's subtree
SELECT body, meta FROM folders
WHERE __key__ HAS ANCESTOR KEY(workspaces, 'workspace-108')
AND body.name = @name

-- Kindless ancestor query: all entity types under a root
SELECT * WHERE __key__ HAS ANCESTOR KEY(workspaces, 'workspace-108')
```

**Learnings for dxdb DSL:**
- `ANCESTOR(entity_type, @root_id)` clause scopes the query to a root entity's subtree — equivalent to `WHERE root_key = @root_id` but semantically richer
- Kindless ancestor queries (no entity type specified) = `ScanByRoot` without a type filter
- Named bind variables `@name` — adopted
- `__key__` is a first-class queryable property — dxdb path is its equivalent

### From Firestore — subcollection search with ancestor inclusion

Firestore's path-based collection queries, and the user requirement to include ancestor entities in results:

```javascript
// Query a subcollection, optionally include ancestor in each result
collection(db, "workspaces", "workspace-108", "folders")
// → result rows can carry: { folder entity + parent workspace entity }
```

**Learnings for dxdb:**
- `INCLUDE ANCESTOR [entity_type]` in a query result spec — when querying descendants, return the ancestor entity alongside each result row (avoids a second round-trip)
- `FROM *` (no entity type) + `ANCESTOR(workspaces, @id)` = subtree scan across all descendant entity types
- Collection group queries (`FROM folders WHERE root_key IN (...)`) — query the same entity type across multiple root keys; forbidden in v1 (cross-root, violates constraint #9) but worth noting as a future concern

---

## Service Layout

| Service | Responsibility |
|---------|---------------|
| `SchemaService` | Register and query proto descriptors |
| `EntityTypeService` | DDL — register entity types and proto columns |
| `EntityService` | CRUD + scan/query for entities |
| `IndexService` | Add/drop/query indexes |

---

## Proto Definition

```protobuf
syntax = "proto3";

document dxdb.v1;

import "google/protobuf/descriptor.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";


// ═══════════════════════════════════════════════════════════════
// Common types
// ═══════════════════════════════════════════════════════════════

// Unique identity of every entity in dxdb.
//
// root_key rules:
//   - Root entity types (parent_type = NULL in the registry): root_key MUST be empty.
//     The table PK is (entity_local_key) only.
//   - All non-root entity types at any depth: root_key MUST be set to the root
//     ancestor's entity_local_key. It is never an intermediate parent's key.
//     The table PK is (root_key, entity_local_key).
//
// Example hierarchy:  workspaces → folders → documents
//   Workspace:     { entity_type:"workspaces",    root_key:"",           entity_local_key:"workspace-108" }
//   Folder: { entity_type:"folders",root_key:"workspace-108",  entity_local_key:"1"         }
//   Document:  { entity_type:"documents",  root_key:"workspace-108",  entity_local_key:"folder-5/doc-1"}
//                                          ^^^ still the Workspace's key, not the Folder's
message EntityKey {
  string entity_type       = 1;
  string root_key          = 2;  // empty for root entity types; root ancestor's key otherwise
  string entity_local_key  = 3;
}

// One proto column value for an Insert/Update/Upsert request.
message ColumnData {
  uint64 schema_id = 1;  // must be pre-registered via SchemaService.RegisterSchema
  bytes  data      = 2;  // raw proto-encoded bytes matching schema_id
}

// A full entity record as returned from reads.
message EntityRecord {
  EntityKey                    key        = 1;
  map<string, ColumnData>      columns    = 2;  // column_name → ColumnData
  int64                        version    = 3;  // OCC version counter
  google.protobuf.Timestamp    ttl        = 4;  // null if no expiry
  google.protobuf.Timestamp    created_at = 5;
  google.protobuf.Timestamp    updated_at = 6;
}


// ═══════════════════════════════════════════════════════════════
// SchemaService — proto descriptor registry
// ═══════════════════════════════════════════════════════════════

service SchemaService {
  // Register a proto message type. Returns a stable schema_id.
  // Idempotent on identical (type_url + descriptor_set bytes) — returns existing schema_id.
  // A changed descriptor for the same type_url creates a NEW schema_id (schema versioning).
  rpc RegisterSchema(RegisterSchemaRequest) returns (RegisterSchemaResponse);

  rpc GetSchema(GetSchemaRequest) returns (GetSchemaResponse);
  rpc ListSchemas(ListSchemasRequest) returns (stream SchemaInfo);
}

message RegisterSchemaRequest {
  // Full transitive closure of all .proto files needed to decode the target message.
  // Obtain via: protoc --descriptor_set_out=out.pb --include_imports ...
  // Or via prost-reflect / protoreflect APIs at build time.
  google.protobuf.FileDescriptorSet descriptor_set = 1;

  // Fully-qualified message name within descriptor_set, e.g. "my.document.Folder"
  string message_name = 2;
}

message RegisterSchemaResponse {
  uint64 schema_id = 1;
  string type_url  = 2;  // "type.googleapis.com/my.document.Folder"
}

message GetSchemaRequest {
  oneof id {
    uint64 schema_id = 1;
    string type_url  = 2;
  }
}

message GetSchemaResponse {
  uint64                            schema_id      = 1;
  string                            type_url       = 2;
  google.protobuf.FileDescriptorSet descriptor_set = 3;
}

message ListSchemasRequest {
  string type_url_prefix = 1;  // optional filter
}

message SchemaInfo {
  uint64                    schema_id  = 1;
  string                    type_url   = 2;
  google.protobuf.Timestamp created_at = 3;
}


// ═══════════════════════════════════════════════════════════════
// EntityTypeService — DDL (entity type registration)
// ═══════════════════════════════════════════════════════════════

service EntityTypeService {
  // Register a new entity type (analogous to CREATE TABLE).
  // Creates the physical SQLite table et_<name> with the declared proto columns.
  rpc CreateEntityType(CreateEntityTypeRequest) returns (CreateEntityTypeResponse);

  rpc GetEntityType(GetEntityTypeRequest) returns (GetEntityTypeResponse);
  rpc ListEntityTypes(ListEntityTypesRequest) returns (stream EntityTypeInfo);
}

// Definition of a proto column within an entity type.
message ProtoColumnDef {
  string column_name = 1;  // e.g. "body", "meta"
  string type_url    = 2;  // initial type_url; schema may evolve via schema_versions
  bool   required    = 3;  // if true, column_data may not be NULL on insert
  int32  ordinal     = 4;  // display / storage workspaceing (server assigns if 0)
}

message CreateEntityTypeRequest {
  string               entity_type   = 1;  // name, e.g. "folders"
  string               parent_type   = 2;  // e.g. "workspaces"; empty for root entity types
  repeated ProtoColumnDef columns    = 3;  // must have ≥ 1
}

message CreateEntityTypeResponse {
  string entity_type = 1;
}

message GetEntityTypeRequest {
  string entity_type = 1;
}

message GetEntityTypeResponse {
  string                  entity_type  = 1;
  string                  parent_type  = 2;
  repeated ProtoColumnDef columns      = 3;
  google.protobuf.Timestamp created_at = 4;
}

message ListEntityTypesRequest {
  string parent_type = 1;  // optional: list only children of a given parent type
}

message EntityTypeInfo {
  string entity_type  = 1;
  string parent_type  = 2;
  int32  column_count = 3;
}


// ═══════════════════════════════════════════════════════════════
// EntityService — CRUD + scan
// ═══════════════════════════════════════════════════════════════

service EntityService {
  // Insert a new entity. Fails with ALREADY_EXISTS if key exists.
  rpc Insert(InsertRequest) returns (InsertResponse);

  // Insert or replace. Creates with version=1; on existing key, increments version.
  rpc Upsert(UpsertRequest) returns (UpsertResponse);

  // Read a single entity by key.
  rpc Get(GetEntityRequest) returns (GetEntityResponse);

  // Replace all proto columns of an existing entity.
  // expected_version: 0 = skip OCC check; >0 = fail with ABORTED if mismatch.
  rpc Update(UpdateRequest) returns (UpdateResponse);

  // Delete an entity. Hard delete. Idempotent.
  rpc Delete(DeleteRequest) returns (google.protobuf.Empty);

  // Scan entities with optional index filter. Server-streaming.
  rpc Scan(ScanRequest) returns (stream EntityRecord);

  // Fetch all entities under a root_key, optionally filtered by entity_type.
  // This is the primary hierarchy access pattern.
  rpc ScanByRoot(ScanByRootRequest) returns (stream EntityRecord);
}

message InsertRequest {
  EntityKey                 key         = 1;
  map<string, ColumnData>   columns     = 2;  // column_name → ColumnData
  google.protobuf.Timestamp ttl         = 3;  // optional; null = no expiry
}

message InsertResponse {
  EntityKey                 key        = 1;
  int64                     version    = 2;  // always 1 for a new insert
  google.protobuf.Timestamp created_at = 3;
}

message UpsertRequest {
  EntityKey                 key         = 1;
  map<string, ColumnData>   columns     = 2;
  google.protobuf.Timestamp ttl         = 3;
}

message UpsertResponse {
  EntityKey                 key        = 1;
  bool                      created    = 2;  // true = inserted; false = updated
  int64                     version    = 3;
  google.protobuf.Timestamp updated_at = 4;
}

message GetEntityRequest {
  EntityKey        key     = 1;
  repeated string  columns = 2;  // optional projection; omit to return all columns
}

message GetEntityResponse {
  EntityRecord entity = 1;  // absent (default) if not found — gRPC NOT_FOUND status
}

message UpdateRequest {
  EntityKey                 key              = 1;
  map<string, ColumnData>   columns          = 2;  // full replace of all proto columns
  int64                     expected_version = 3;  // 0 = skip OCC; >0 = conditional
  google.protobuf.Timestamp ttl              = 4;  // optional; omit to keep current TTL
}

message UpdateResponse {
  EntityKey                 key        = 1;
  int64                     version    = 2;  // new version after successful update
  google.protobuf.Timestamp updated_at = 3;
}

message DeleteRequest {
  EntityKey key              = 1;
  int64     expected_version = 2;  // 0 = unconditional; >0 = conditional delete
}

message ScanRequest {
  string entity_type   = 1;  // required

  // Optional: filter by schema (only return entities whose named column has this schema)
  string column_name   = 2;
  string type_url      = 3;

  // Optional: index-based filter (requires a READY index)
  IndexFilter index_filter = 4;

  // Pagination
  bytes  page_token = 10;
  uint32 page_size  = 11;

  // Optional column projection
  repeated string columns = 12;
}

message IndexFilter {
  string index_name = 1;
  oneof predicate {
    ScalarValue eq         = 2;  // equality: field = value
    RangeFilter range      = 3;  // range: lower ≤ field ≤ upper (see OQ-4D)
  }
}

// Scalar value for index filters and DSL parameters.
// Encoding follows the HRANA v3 pattern (libSQL):
//   - sint64 for integers (zigzag encoding, efficient for negatives)
//   - explicit Null inner message (distinguishes SQL NULL from proto field-not-set)
//   - named parameters use NamedParam { name, value } alongside this type
message ScalarValue {
  oneof value {
    Null   null_value  = 1;  // SQL NULL — field absent or explicitly null
    sint64 int_value   = 2;  // zigzag sint64 (not int64) for signed integers
    double float_value = 3;
    string text_value  = 4;
    bytes  blob_value  = 5;
    bool   bool_value  = 6;
  }
  message Null {}
}

// Named parameter for DSL statements and structured queries.
message NamedParam {
  string      name  = 1;  // e.g. "name", "root_id"
  ScalarValue value = 2;
}

message RangeFilter {
  ScalarValue lower_bound     = 1;  // optional; omit for open lower bound
  ScalarValue upper_bound     = 2;  // optional; omit for open upper bound
  bool        lower_inclusive = 3;  // default true
  bool        upper_inclusive = 4;  // default true
}

// Fetch all entities belonging to a root aggregate.
// Executes one range scan per entity type table (separate queries, one read transaction).
// For the root entity type itself: queries et_<root_type> WHERE entity_local_key = root_key_value.
// For all descendant types:        queries et_<type>      WHERE root_key          = root_key_value.
message ScanByRootRequest {
  string          root_key_value = 1;  // the root entity's entity_local_key, e.g. "workspace-108"
  string          root_type      = 2;  // the root entity type, e.g. "workspaces" — required to
                                       // determine the table hierarchy to scan
  string          entity_type    = 3;  // optional: restrict to one descendant entity type
  bytes           page_token     = 10;
  uint32          page_size      = 11;
  repeated string columns        = 12; // optional column projection
}


// ═══════════════════════════════════════════════════════════════
// IndexService
// ═══════════════════════════════════════════════════════════════

service IndexService {
  // Declare an index on a proto field within a proto column.
  // Immediately creates DDL (ALTER TABLE ADD COLUMN + CREATE INDEX).
  // Backfill runs asynchronously; response has status = BUILDING.
  rpc AddIndex(AddIndexRequest) returns (AddIndexResponse);

  rpc DropIndex(DropIndexRequest) returns (google.protobuf.Empty);
  rpc ListIndexes(ListIndexesRequest) returns (stream IndexInfo);
  rpc IndexStatus(IndexStatusRequest) returns (IndexStatusResponse);
}

message AddIndexRequest {
  string index_name   = 1;   // unique name, e.g. "folders_body_name"
  string entity_type  = 2;
  string column_name  = 3;   // which proto column, e.g. "body"
  string field_path   = 4;   // e.g. "name" or "address.city"
  string null_policy  = 5;   // "SKIP" (default) | "INDEX_NULL"
}

message AddIndexResponse {
  string index_name = 1;
  string status     = 2;  // "BUILDING" — always; caller polls IndexStatus
}

message DropIndexRequest {
  string index_name = 1;
}

message ListIndexesRequest {
  string entity_type  = 1;  // required
  string column_name  = 2;  // optional filter
}

message IndexInfo {
  string                    index_name    = 1;
  string                    entity_type   = 2;
  string                    column_name   = 3;
  string                    field_path    = 4;
  string                    null_policy   = 5;
  string                    status        = 6;  // "READY"|"BUILDING"|"FAILED"
  google.protobuf.Timestamp created_at    = 7;
}

message IndexStatusRequest {
  string index_name = 1;
}

message IndexStatusResponse {
  string index_name    = 1;
  string status        = 2;
  uint64 rows_indexed  = 3;
  uint64 rows_total    = 4;
}
```

---

## Open Questions

### OQ-4A: Insert conflict behavior

`Insert` on duplicate key returns `ALREADY_EXISTS`. Alternative: merge Insert and Upsert into one RPC with a `conflict_policy` enum (`FAIL | REPLACE`). Keeping them separate (current proposal) is more explicit.

### OQ-4B: Partial column update

`Update` replaces all proto columns. Should there be a `PatchColumn` RPC? See RFC-0001 §OQ-1D.

### OQ-4C: `ScanByRoot` across entity types — **RESOLVED**

**Decision: separate query per entity type table, wrapped in a single `BEGIN DEFERRED` transaction.**

Implementation:
```rust
conn.execute("BEGIN DEFERRED")?;
for et in entity_type_registry.get_subtree(root_entity_type) {
    let stmt = conn.prepare_cached("SELECT * FROM et_{et} WHERE root_key = ? ORDER BY entity_local_key")?;
    for row in stmt.query([root_key])? { yield row_to_record(et, row); }
}
conn.execute("COMMIT")?;
```

**Why not SQL JOINs:** Column schemas differ per entity type table — JOIN aliases become unmanageable across N tables. Nested-loop JOINs offer no performance advantage over separate B-tree range scans for a single `root_key`.

**Why not UNION ALL:** Column count must match across all branches, requiring NULLs for non-applicable columns and dynamically-generated SQL text that cannot be a prepared statement. No streaming benefit (SQLite executes the full UNION ALL before returning rows).

**Why separate queries win (embedded SQLite):** Each is a pure PK range scan on `(root_key, entity_local_key)` — the fastest possible access path. No network round-trip cost (SQLite is embedded). `prepare_cached` amortizes parse cost after the first call. `BEGIN DEFERRED` provides a consistent read snapshot across all type tables.

**Workspaceing:** results are streamed type-by-type, workspaceed within each type by `(root_key, entity_local_key)`. Cross-type workspaceing is undefined — callers sort client-side if needed.

**Spanner note:** Spanner's physical interleaving collapses N separate B-tree scans into one sequential scan of physically adjacent rows. This is the specific advantage SQLite cannot replicate. See RFC-0002 §SQLite Interleave Limitation.

### OQ-4D: Range filter

`RangeFilter` is included in `IndexFilter.predicate` in the strawman. Is this in scope for v1, or equality-only first?

### OQ-4E: `GetEntityResponse` on missing key

Current proposal: return gRPC `NOT_FOUND` status. Alternative: return an `EntityRecord` with an empty key (caller checks). gRPC `NOT_FOUND` is idiomatic and preferred.

### OQ-4F: gRPC server reflection

Implement `grpc.reflection.v1.ServerReflection`? Enables `grpcurl`, `evans`, Postman. Low implementation cost, high developer ergonomics value.

### OQ-4G: Authentication / TLS

Unauthenticated plain TCP for v1 (local/dev use), or `--tls-cert` / `--tls-key` flags for v1?

### OQ-4J: Declarative + DSL API design ⚠️ DEFERRED — needs design iteration

The current API has `ScanByRoot` (flat, per-type) and `Scan` (single entity type). A more expressive model — inspired by Firebase's path + subcollection design — would expose a unified path-based query RPC.

Three requirements converge here:
1. **Declarative CRUD** — operations express desired state (`Upsert`, not `Insert`/`Update` separately)
2. **Simple DSL for DDL + DML** — text statements with named parameters; server parses
3. **Path-based subtree query with ANCESTOR scoping and INCLUDE ANCESTOR result enrichment**

#### Declarative CRUD principle

"Declarative" means the caller states *what* the entity state should be, not *which operation* to apply. Concretely:

- `Upsert` (not separate Insert + Update) — declare the desired state; server creates or replaces
- `Delete` is idempotent — declaring "this entity should not exist" is safe to call twice
- Conditional updates express the *expected* version, not "if exists, update, else fail"

The current service already leans this way (`Upsert` exists). The API should complete the shift: remove `Insert` as a separate RPC (or keep it only for strict create-or-fail semantics).

#### DSL surface area

Inspired by HRANA's `Stmt { sql, named_args }` pattern — a single message carries a DSL string with typed named parameters. The server parses, plans, and executes. Server caches parsed statements by ID (HRANA's `sql_id` pattern).

```proto
// Strawman — not final; needs dedicated RFC iteration
message DxDbStmt {
  oneof source {
    string  dsl    = 1;  // DSL text (first call; server caches)
    int32   stmt_id = 2; // reference a previously cached statement
  }
  repeated NamedParam params = 3;  // @name → ScalarValue bindings
}

service DxDbService {
  rpc Execute(ExecuteRequest) returns (ExecuteResponse);  // DDL + DML
  rpc Query(QueryRequest)     returns (stream QueryRow);  // DQL
}
```

**Confirmed DSL keywords (closed decisions):**

| Concern | Keyword | Rationale |
|---------|---------|-----------|
| Define entity type | `ENTITY` | Sole DDL keyword; root vs. child distinguished by `UNDER` |
| Hierarchy declaration | `UNDER <parent>` | Appended to `ENTITY`; absent = root entity type |
| Upsert / write entity | `WRITE` | Declarative; avoids HTTP `PUT` connotation |
| Delete entity | `REMOVE` | Avoids SQL `DELETE` association |
| Collection read | `FETCH` | Active; not confused with ANSI `SELECT` |
| Point read | `GET` | Single entity by full key |
| Column projection | `PICK` | Novel; no SQL or graph-DB precedent |
| Ancestor enrichment | `INCLUDE ANCESTOR` | Self-describing |

**Avoided vocabulary:** `SELECT`, `KIND`, `KINDLESS`, `PUT`, `UPSERT`, `DELETE`, `CREATE`, `DEFINE`, `ROOT` (as a keyword).

**DDL DSL sketch:**
```
-- Root entity type: no UNDER clause = no parent = root
ENTITY workspaces {
  body  pkg.Workspace      REQUIRED
  meta  pkg.EntityMeta
}

-- Child entity type: UNDER <parent> declares the hierarchy
ENTITY folders UNDER workspaces {
  body  pkg.Folder  REQUIRED
  meta  pkg.EntityMeta
}

-- Grandchild: UNDER refers to immediate parent
ENTITY documents UNDER folders {
  body  pkg.Document   REQUIRED
}

-- Indexes
ADD INDEX folders_name    ON folders.body.name
ADD INDEX folders_tenant ON folders.meta.tenant_id  NULL-POLICY INDEX_NULL
DROP INDEX folders_name

-- Introspection
DESCRIBE ENTITY folders        -- columns, indexes, parent type
DESCRIBE HIERARCHY workspaces         -- full subtree tree view
SHOW ENTITIES                     -- all entity types, flat list
SHOW ENTITIES UNDER workspaces        -- direct children only
```

`ENTITY` is the sole DDL keyword. Root vs. child is distinguished entirely by presence/absence of `UNDER` — no `ROOT`, `DEFINE`, or `CREATE` prefix needed. Parallel to proto's `message` keyword.

**DML DSL sketch (declarative — state-based):**
```
-- Upsert: declare desired state; server creates or replaces
WRITE /workspaces/@root_id/folders/@local_id {
  body: @body,          -- NamedParam: ColumnData { schema_id, bytes } (Option 3)
  meta: @meta
} VERSION @expected_ver -- OCC: 0 = unconditional

-- Delete
REMOVE /workspaces/@root_id/folders/@local_id VERSION @expected_ver
```

**DQL DSL illustration:**
```
-- Point read (single entity by full key)
GET /workspaces/@root_id/folders/@local_id
PICK body

-- Collection fetch: filter + projection + workspaceing
FETCH /workspaces/@root_id/folders
WHERE body.name = @name AND body.size > @min_size
PICK body, meta
ORDER BY body.size DESC
LIMIT @n

-- Subtree fetch: all entity types under a root, per-type column projection
FETCH /workspaces/@root_id/*
PICK { folders: [body], folders: [body, meta] }
INCLUDE ANCESTOR workspaces            -- attach the Workspace entity to each result row
LIMIT @n

-- Subtree fetch, all types, all columns
FETCH /workspaces/@root_id/*
```

**What to decide in the next iteration:**
- Does the DSL replace or complement the structured proto RPCs (SchemaService, EntityTypeService, etc.)?
- `INCLUDE ANCESTOR` depth — direct parent only, or the full ancestor chain?
- Statement caching lifecycle — server-side TTL, or explicit `CLOSE STMT @id`?
- Cursor-based pull streaming vs. gRPC server-side streaming for `FETCH` results.
- Cross-root queries (same entity type across multiple root keys) — out of scope for v1 (violates constraint #9); flag for v2.

### OQ-4H: `ScanByRoot` workspaceing guarantee

When `entity_type` is specified, results are workspaceed by `(root_key, entity_local_key)` (PK workspace from SQLite). When no `entity_type` is specified (multi-table query), workspaceing is undefined. Document this explicitly.

---

## Consequences

- `EntityTypeService.CreateEntityType` is a DDL operation — it executes `CREATE TABLE et_<name>`. Schema changes have the same operational weight as database DDL in any SQL system.
- `OCC version` field in `EntityRecord` requires that `Get` returns it and `Update` / `Delete` accept it. Zero values disable the check — ergonomic default for non-concurrent use cases.
- `ScanByRoot` is a first-class RPC (not just a filter on `Scan`) because `root_key` is the partition boundary and this operation must be implementable as a single B-tree range scan per entity type table.

