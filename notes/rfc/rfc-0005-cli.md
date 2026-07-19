# RFC-0005: CLI & Single Binary

| Field       | Value                          |
|-------------|--------------------------------|
| Status      | **Draft**                      |
| Parent      | [RFC-0000](RFC-0000-dxdb-root.md) |
| Authors     | anandg                         |
| Created     | 2026-07-03                     |
| Last Update | 2026-07-18                     |

---

## Summary

The `dxdb` single Rust binary operates as both the gRPC server and the CLI client. CLI commands are thin wrappers over the four gRPC services: `SchemaService`, `EntityTypeService`, `EntityService`, and `IndexService`.

---

## Resolved: CLI Architecture

**Decision (closed):** The CLI always dials a running `dxdb serve` process. No direct file access from the CLI. Concurrent file-access hazards are eliminated.

---

## Rust Crates

| Concern | Crate |
|---------|-------|
| CLI argument parsing | [`clap`](https://docs.rs/clap) v4 (derive macros) |
| gRPC server + client | [`tonic`](https://docs.rs/tonic) |
| Proto codegen | [`prost`](https://docs.rs/prost) + [`tonic-build`](https://docs.rs/tonic-build) |
| Dynamic proto reflection + JSON transcoding | [`prost-reflect`](https://docs.rs/prost-reflect) |
| SQLite | [`rusqlite`](https://docs.rs/rusqlite) (bundled feature) |
| Async runtime | [`tokio`](https://docs.rs/tokio) |
| Structured logging | [`tracing`](https://docs.rs/tracing) + `tracing-subscriber` |

---

## Binary Entry Point

```
dxdb [global-flags] <subcommand> [subcommand-flags] [args]

Global flags:
  --host <addr>   gRPC server address (default: http://localhost:7979; env: DXDB_HOST)
  --timeout <ms>  Per-RPC timeout ms (default: 30000)
```

Mode selection by subcommand:

```rust
match cli.command {
    Command::Serve { .. } => run_server(config).await,
    _                     => run_client(cli).await,
}
```

---

## Command Structure

### `serve`

```
dxdb serve [--db <path>] [--addr <host:port>] [--tls-cert <f>] [--tls-key <f>]

  --db    SQLite file path (default: ./dxdb.db; env: DXDB_DB_PATH)
  --addr  Listen address  (default: 0.0.0.0:7979; env: DXDB_ADDR)
```

Graceful shutdown on `SIGTERM` / `SIGINT`.

---

### `schema` — SchemaService

```
dxdb schema register --file <descriptor_set.pb> --message <FQN>
    Register a schema from a compiled FileDescriptorSet.
    Output: schema_id, type_url

dxdb schema get (--id <schema_id> | --type <type_url>)
    Print schema_id, type_url, decoded field list.

dxdb schema list [--prefix <type_url_prefix>]
```

---

### `entity-type` — EntityTypeService (DDL)

```
dxdb entity-type create --name <name> [--parent <parent_type>]
    --column body:<type_url> [--column meta:<type_url>] [--column ...]
    Register a new entity type. Creates et_<name> SQLite table.
    May repeat --column for multiple proto columns.
    Add --required to a column: --column body:<type_url>:required

dxdb entity-type get --name <name>
    Show entity type definition: parent, columns, created_at.

dxdb entity-type list [--parent <parent_type>]
```

---

### `entity` — EntityService

```
dxdb entity insert --type <entity_type>
                    --root <root_key> --local <local_key>
                    --column body:<schema_id>:(--json '<json>' | --file <path>)
                    [--column meta:<schema_id>:(--json '<json>' | --file <path>)]
                    [--ttl <unix_ms>]

dxdb entity upsert   [same flags as insert]

dxdb entity get --type <entity_type> --root <root_key> --local <local_key>
                 [--columns body,meta]
    Output: protojson decoded. --raw for hex binary.

dxdb entity update --type <entity_type> --root <root_key> --local <local_key>
                    --column body:<schema_id>:(--json '<json>' | --file <path>)
                    [--expected-version <n>]   # 0 = skip OCC (default)
                    [--ttl <unix_ms>]

dxdb entity delete --type <entity_type> --root <root_key> --local <local_key>
                    [--expected-version <n>]

dxdb entity scan --type <entity_type>
                  [--column-filter <col>=<type_url>]
                  [--index <index_name>=<value>]
                  [--page-size <n>]
    Streams ndjson. Pipe to jq.

dxdb entity scan-root --root <root_key> [--type <entity_type>]
                       [--page-size <n>]
    Fetch all entities under root_key.
    Example: dxdb entity scan-root --root workspace-108 | jq '.columns.body'
```

---

### `index` — IndexService

```
dxdb index add --name <index_name> --type <entity_type>
                --column <col> --field <field_path>
                [--null-policy skip|null]
    Returns immediately. Status = BUILDING.

dxdb index drop --name <index_name>

dxdb index list --type <entity_type> [--column <col>]

dxdb index status --name <index_name>
    Shows READY|BUILDING|FAILED + rows_indexed/rows_total.
```

---

### `db` — Utility

```
dxdb db info
    Row count per entity type, schema count, index count, SQLite file size.

dxdb db backup --out <dest_file>
    Hot backup via SQLite .backup API.
```

---

## JSON Input: Client-Side Transcoding

`--json '<json>'` in entity commands enables human-friendly proto input. The flow:

```
1. CLI calls SchemaService.GetSchema(schema_id) → FileDescriptorSet
2. prost-reflect parses FileDescriptorSet → MessageDescriptor
3. prost-reflect::DeserializeMessage(json) → proto bytes
4. CLI sends ColumnData { schema_id, data: proto_bytes } to EntityService
```

No server-side transcoding RPC needed — `prost-reflect` handles it client-side. Schema resolution is one extra gRPC call, cached for the session.

**Example:**

```bash
dxdb entity insert \
  --type folders \
  --root workspace-108 --local 1 \
  --column body:7:--json '{"name":"ABC-123","quantity":5}' \
  --column meta:3:--json '{"tenant_id":"acme"}'
```

---

## Output Format

Default: protojson (decoded using `SchemaService.GetSchema` + `prost-reflect::SerializeMessage`).

`--raw`: hex-encoded proto bytes.

`--format json|text`: output mode; `json` is ndjson for scan commands (pipe to `jq`).

---

## Open Questions

### OQ-5A: Client-side JSON transcoding — confirmed

Confirmed: client-side via `prost-reflect`. No server `TranscodeJson` RPC needed.

### OQ-5B: `dxdb serve` daemon management

Should `dxdb` include `server start --daemonize` / `server stop` / `server status` for background process management? Or leave to OS (systemd, launchd, Docker)?

### OQ-5C: Shell completions

`clap` can generate completions (`bash`, `zsh`, `fish`). `dxdb completions <shell>` as a built-in?

### OQ-5D: Config file

Support `~/.dxdbrc` or `.dxdb.toml` for `--host`, `--timeout`? Or env vars only?

### OQ-5E: `entity-type create` — schema_id vs. type_url in column definition

`--column body:<type_url>` in the CLI flags the initial type_url. The server resolves this to a `schema_id` by looking up the most recent registration for that type_url. Should the CLI accept `schema_id` directly, or always resolve via `type_url`? Using `type_url` is more ergonomic; `schema_id` is more precise if multiple versions exist.

---

## Consequences

- `entity-type create` is a DDL operation — like `CREATE TABLE` in SQL. It should be treated with the same care (idempotent via `--if-not-exists` flag? — see OQ-5E).
- ndjson output for `entity scan` / `entity scan-root` is compatible with `jq` and standard UNIX pipeline tools.
- Client-side JSON transcoding adds `prost-reflect` as a binary dependency, but this is already required by the server for index field extraction.

