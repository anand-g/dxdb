# RFC-0003: Indexing & Reindex

| Field       | Value                          |
|-------------|--------------------------------|
| Status      | **Draft**                      |
| Parent      | [RFC-0000](RFC-0000-dxdb-root.md) |
| Authors     | anandg                         |
| Created     | 2026-07-03                     |
| Last Update | 2026-07-18                     |

---

## Summary

This RFC defines how indexes are declared on proto fields within proto columns, how the system materializes index field values as extra columns in the entity table, how the server maintains those columns on every write, and how `AddIndex` backfills existing rows.

---

## Index Storage Model

Indexes are **materialized columns** in the entity table — not separate tables. When you declare an index on `body.name` in entity type `folders`, the effect is:

```
et_folders gains a new column: body__name TEXT (or INTEGER / REAL / BLOB)
A SQLite B-tree index is created:  et_folders__body__name ON et_folders(body__name)
```

A query `WHERE body__name = 'ABC'` is a native SQLite indexed lookup — no joins, no subqueries. The SQLite query planner uses `et_folders__body__name` directly.

See [RFC-0002 §AddIndex DDL](RFC-0002-storage-engine.md#addindex-ddl) for the exact DDL.

---

## Index Identity

Each index is identified by a unique `index_name` and targets a specific `(entity_type, column_name, field_path)`:

```
index_name:   string       -- unique, e.g. "folders_body_name"
entity_type:  string       -- e.g. "folders"
column_name:  string       -- proto column name, e.g. "body"
field_path:   string       -- dot-separated path, e.g. "name" or "address.city"
field_number: int32        -- proto field number (for efficient prost-reflect decode)
value_type:   SqliteType   -- TEXT | INTEGER | REAL | BLOB
null_policy:  NullPolicy   -- SKIP | INDEX_NULL
```

The `index_registry` DDL is in [RFC-0002 §System Tables](RFC-0002-storage-engine.md#system-tables).

### Materialized column naming

```
<column_name>__<field_path_with_dots_as_underscores>
```

Examples:
| Proto column | Field path | Materialized column |
|-------------|-----------|---------------------|
| `body` | `name` | `body__name` |
| `body` | `address.city` | `body__address_city` |
| `meta` | `tenant_id` | `meta__tenant_id` |

---

## AddIndex Operation

### Preconditions

1. `entity_type` must be registered in `entity_type_registry`.
2. `column_name` must be a declared proto column for that entity type.
3. `field_path` must resolve to a **scalar leaf field** in the column's type_url descriptor (no repeated or message fields in v1 — see OQ-3D).
4. `value_type` must match the proto scalar type of the target field.
5. `index_name` must be unique in `index_registry`.

### Steps

```
Step 1 — DDL (fast, O(1)):
  BEGIN TRANSACTION
    INSERT INTO index_registry (..., status='BUILDING', rows_total=COUNT(*) FROM et_<type>)
    ALTER TABLE et_<entity_type> ADD COLUMN <col>__<field> <value_type>
    CREATE INDEX et_<entity_type>__<col>__<field> ON et_<entity_type>(<col>__<field>)
    CREATE INDEX et_<entity_type>__root_<col>__<field>
        ON et_<entity_type>(root_key, <col>__<field>)   -- for root-scoped index queries
  COMMIT
  -- Return AddIndexResponse { status: "BUILDING" } to caller immediately

Step 2 — Backfill (async, O(n)):
  FOR EACH row IN et_<entity_type> (batched, N rows per transaction):
    schema_id = row.<col>_schema
    descriptor = schema_versions WHERE schema_id = ?
    value = prost_reflect::decode(row.<col>_data, descriptor, field_path)
    BEGIN TRANSACTION
      UPDATE et_<entity_type>
      SET <col>__<field> = value   -- NULL if field absent and null_policy = SKIP
      WHERE root_key = row.root_key AND entity_local_key = row.entity_local_key
      UPDATE index_registry SET rows_indexed = rows_indexed + batch_size WHERE index_name = ?
    COMMIT

Step 3 — Mark ready:
  BEGIN TRANSACTION
    UPDATE index_registry SET status = 'READY' WHERE index_name = ?
  COMMIT
```

**Batching:** the backfill commits N rows per transaction (e.g. N=500). This keeps the write lock release interval short during reindex, at the cost of the partially-built index being visible to concurrent reads while `status = BUILDING`.

**Concurrency during backfill:** Rows inserted/updated during the backfill are written with the correct `<col>__<field>` value (write path always extracts — see §Write Path below). The backfill only needs to update rows that existed before the `AddIndex` call.

---

## DropIndex Operation

```
Step 1:
  BEGIN TRANSACTION
    DELETE FROM index_registry WHERE index_name = ?
    DROP INDEX IF EXISTS et_<entity_type>__root_<col>__<field>
    DROP INDEX IF EXISTS et_<entity_type>__<col>__<field>
    ALTER TABLE et_<entity_type> DROP COLUMN <col>__<field>
  COMMIT
```

Note: SQLite requires `DROP INDEX` before `DROP COLUMN` when a B-tree index exists on the column.

---

## Write Path: Index Maintenance

On every `Insert`, `Upsert`, or `Update`, the server extracts materialized column values **synchronously** as part of the write, within the same transaction:

```rust
fn write_entity(conn, entity_type, key, columns, expected_version) {
    // 1. Look up all READY indexes for this entity type
    let indexes = index_registry.get_ready(entity_type);

    // 2. For each proto column being written, extract all indexed fields
    let mut materialized: HashMap<String, SqlValue> = HashMap::new();
    for (col_name, col_data) in &columns {
        let schema_id = col_data.schema_id;
        let descriptor = schema_versions.get(schema_id);
        for idx in indexes.filter(|i| i.column_name == col_name) {
            let value = prost_reflect::extract_field(
                &col_data.data, &descriptor, idx.field_number
            );
            materialized.insert(idx.materialized_col_name(), match value {
                Some(v) => SqlValue::from(v),
                None if idx.null_policy == NullPolicy::Skip => SqlValue::Null,
                None => SqlValue::Null,  // INDEX_NULL also stores NULL; same wire value
            });
        }
    }

    // 3. Single INSERT/UPDATE with proto columns + materialized index columns
    conn.execute(
        "INSERT INTO et_<type> (root_key, local_key, <col>_data, <col>_schema, ...,
                                ttl, version, created_at, updated_at, <idx_cols>...)
         VALUES (?, ?, ?, ?, ..., ?, 1, ?, ?, ?...)
         ON CONFLICT DO UPDATE SET ... version = version + 1",
        params![...]
    );
}
```

Key property: **one row write, one transaction** — no separate index table writes needed.

---

## Sparse Index Semantics

An entity row's proto column may not contain the indexed field (field absent in the proto encoding, or the column has a different schema version that pre-dates the field).

| `null_policy` | Behavior |
|---------------|---------|
| `SKIP` (default) | Materialized column = `NULL`. SQLite B-tree index skips NULLs by default in uniqueness checks; range scans include NULLs. For equality queries `WHERE body__name = 'X'`, NULL rows are naturally excluded. |
| `INDEX_NULL` | Materialized column = `NULL` as well (same wire value). Semantically flagged as "intentionally absent" for `WHERE body__name IS NULL` queries. |

In practice, `SKIP` and `INDEX_NULL` produce identical materialized column values (`NULL`). The distinction is semantic — `INDEX_NULL` signals that the index should be used for `IS NULL` queries, while `SKIP` signals the field is simply not applicable to rows that don't have it.

> **OQ-3A:** Should the two policies be distinguished at the SQLite level? Options: a separate `<col>__<field>__null_reason` column, a sentinel value (e.g. `0x00` bytes for `INDEX_NULL` vs actual NULL for `SKIP`). Or is the semantic distinction sufficient in the index_registry metadata only?

---

## Indexes During BUILDING

When `status = BUILDING`:
- Reads (`Scan`, `ScanByRoot`) that use the index: the server **must not** use it as a filter — `body__name` may be NULL for rows that haven't been backfilled yet. Server either: (a) rejects index-based queries on BUILDING indexes with `FAILED_PRECONDITION`, or (b) falls back to a full table scan.
- Writes: always write the correct `body__name` value for new/updated rows — the index is correct for post-AddIndex writes from the start.

> **OQ-3E:** Query behavior against a BUILDING index: reject with error, or fall back to full scan with a warning header in the gRPC response metadata?

---

## Proto Field Extraction

The server uses `prost-reflect` for dynamic proto decoding at index-build and write time:

```rust
use prost_reflect::{DynamicMessage, DescriptorPool, MessageDescriptor, Value};

fn extract_field(data: &[u8], descriptor: &MessageDescriptor, field_number: i32)
    -> Option<Value>
{
    let msg = DynamicMessage::decode(descriptor.clone(), data).ok()?;
    let field_desc = descriptor.get_field(field_number as u32)?;
    Some(msg.get_field(&field_desc).into_owned())
}
```

The `field_number` (stored in `index_registry`) is used for lookup rather than `field_path` (string) to avoid repeated proto reflection at hot-path write time.

---

## Open Questions

### OQ-3A: SKIP vs. INDEX_NULL wire distinction

See §Sparse Index Semantics above.

### OQ-3B: Proto schema evolution and materialized columns

When schema_id progresses from v1 → v2 (new optional field added):
- Existing rows have `body__new_field = NULL` (field didn't exist when they were written)
- New rows have `body__new_field = <extracted value>`
- This is correct behavior — handled by `null_policy = SKIP` on any index on the new field

The only problematic evolution is renaming a field while keeping its number (semantically fine by proto rules; the materialized value is the same) or changing a field's type at the same number (forbidden by proto; dxdb assumes callers follow proto evolution rules).

**No additional mechanism needed.** This OQ is resolved: proto field number immutability is a documented caller contract.

### OQ-3C: Composite indexes

Can a single index cover `(body.name, meta.tenant_id)` — a multi-field composite? In SQLite, this would be:

```sql
CREATE INDEX et_folders__body__name__meta__tenant_id
    ON et_folders (body__name, meta__tenant_id);
```

Both materialized columns must already exist. The index is a pure SQLite B-tree index over two existing columns. Is composite index declaration in scope for v1?

### OQ-3D: Repeated and nested field indexing

**Repeated scalar fields** (`tags: repeated string`): index all values? Would require a separate explosion table (one row per tag value per entity). Out of scope for v1.

**Nested message fields** (`address.city`): the dot-path supports nested fields; `prost-reflect` can navigate nested descriptors. Is this in scope for v1, or only top-level scalar fields?

### OQ-3E: Query behavior against BUILDING indexes

See §Indexes During BUILDING above.

### OQ-3F: Backfill batch size and progress reporting

What batch size for backfill commits? Trade-off: larger batches → less lock contention overhead, longer per-commit write lock hold. Suggested default: 500 rows/commit with tunable `--reindex-batch-size` server flag.

---

## Consequences

- Write path prost-reflect decode is O(indexed_fields_per_column) per write — bounded by declared indexes, not by proto message size.
- `DropIndex` is fast (O(1) DDL + lazy space reclaim) — no rows to update; the column is dropped.
- Range queries on index columns (e.g. `body__quantity BETWEEN 1 AND 10`) are supported natively via SQLite B-tree index without any API changes — the server generates the appropriate `WHERE` clause.

