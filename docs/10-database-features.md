# 10 — Database Features & Engine Support

What native features make a state service ("sidecar") possible, what managed providers allow, and how other engines compare. Postgres is the primary target.

## 1. PostgreSQL primitives

### 1.1 System catalogs (`pg_catalog`): the snapshot source

- `pg_namespace`, `pg_class`, `pg_attribute`, `pg_type`, `pg_enum`, `pg_constraint`, `pg_index`, `pg_proc`, `pg_trigger`, `pg_event_trigger`, `pg_extension`, `pg_policy`, `pg_description`, `pg_depend`, `pg_inherits`, `pg_partitioned_table`, `pg_attrdef`, `pg_collation`, `pg_range`, `pg_operator`, `pg_cast`.
- Helper functions: `pg_get_viewdef`, `pg_get_functiondef`, `pg_get_constraintdef`, `pg_get_indexdef`, `pg_get_triggerdef`, `pg_get_expr`, `format_type`.
- **Visibility:** most of `pg_catalog` is readable by `PUBLIC` by default, so a role with only `CONNECT` can see the structure of objects it can't read. ⚠ Verify per catalog (some columns, like `pg_authid` passwords and `pg_statistic`, are restricted; `pg_proc.prosrc` is visible, which matters for the security of function bodies).
- ❌ *Correction:* the conversation suggested using `information_schema.columns`. `information_schema` views **only show objects the current role has privileges on**, so a least-privilege observer would see an incomplete schema. Use `pg_catalog`.

### 1.2 Extended query protocol: Describe

- Parse + Describe without Execute → parameter and result types ([07 §1.1](07-type-safety.md#11-postgres-extended-query-protocol)).
- **Privileges:** permission checks on tables generally happen at execution time, so describing a statement on an object without `SELECT` privilege *may* succeed. ⚠ Verify. It matters if we ever describe against a real DB with the observer role (recommendation: don't, describe on a local engine rebuilt from the snapshot).

### 1.3 Event triggers + `LISTEN`/`NOTIFY`: push change detection

```sql
CREATE FUNCTION dst.notify_ddl() RETURNS event_trigger LANGUAGE plpgsql AS $$
BEGIN
  PERFORM pg_notify('dst_schema_changed',
    json_build_object('tag', tg_tag, 'txid', txid_current())::text);
END $$;

CREATE EVENT TRIGGER dst_ddl_end ON ddl_command_end EXECUTE FUNCTION dst.notify_ddl();
CREATE EVENT TRIGGER dst_drop    ON sql_drop        EXECUTE FUNCTION dst.notify_ddl();
```

- `pg_event_trigger_ddl_commands()` (in `ddl_command_end`) and `pg_event_trigger_dropped_objects()` (in `sql_drop`) give object-level detail. The Observer could do incremental snapshots instead of full ones.
- **Caveats:**
  - Creating event triggers traditionally requires **superuser**. Managed providers vary (some expose it through a special admin role). ⚠ Verify RDS/Aurora, Cloud SQL, Azure, Neon and Supabase individually.
  - Event triggers don't fire for every structural change (e.g. shared objects such as databases, roles and tablespaces, and some other commands). ⚠ Verify the exact exclusions for the target PG versions.
  - `NOTIFY` is delivered at **commit**, so a DDL in a rolled-back transaction produces no notification. Good.
  - `NOTIFY` payloads are limited (8000 bytes by default). Send a small signal, not the snapshot.
  - `LISTEN` doesn't work through **transaction-mode connection poolers** (PgBouncer transaction mode, some serverless proxies). The Observer needs a direct or session-mode connection.
  - Event-trigger bugs can block DDL, so keep the function trivial and fail-safe.
- **Fallback:** polling with a cheap fingerprint query. Full snapshot extraction is a handful of catalog queries; even with thousands of tables that's milliseconds to low seconds. Every 10–60 s is fine.

### 1.4 Logical replication / CDC: mostly not useful for DDL

❌ *Correction:* the conversation suggested CDC "emitted by the write-ahead log" gives DDL changes. Native Postgres logical decoding (`pgoutput`, used by logical replication and Debezium) **does not replicate DDL**. It streams row changes; relation metadata messages appear when a changed relation's shape is first used, but that's not a DDL feed. ⚠ Verify current PG release notes; DDL replication has been proposed for years. Third-party approaches (pglogical's DDL helpers, `pgl_ddl_deploy`, capturing DDL via event triggers into a table that *is* replicated) exist. **Conclusion:** use event triggers or polling for structure. CDC is irrelevant unless `dst` later grows data-aware features.

### 1.5 `pg_stat_statements`: discovering real consumers

- Normalized text of queries actually executed, with call counts, per user/database.
- 💡 The State Service can periodically read it (needs `pg_read_all_stats` or similar ⚠) and match queries against known registered footprints → surface **unregistered consumers** ("someone runs `SELECT legacy_ref FROM orders` 3k times/day, not from any registered service; likely a BI tool"). That's what makes impact analysis honest about consumers who never adopted `dst`.
- Privacy: query text can contain literals if not normalized. `pg_stat_statements` normalizes constants, but treat it as sensitive.

### 1.6 Comments / security labels: annotations

- `COMMENT ON COLUMN users.settings IS '@dst type=UserSettings'` → declared jsonb shapes, ownership, ephemeral markers, nullability hints. Prior art: PostGraphile "smart comments". Comments are part of the snapshot.

### 1.7 Other useful bits

- `server_version_num` for version-aware parsing and linting.
- `pg_database.datcollversion` / collation details if sorting semantics matter.
- `pg_settings` for `search_path`, etc.
- `pg_stat_user_tables.n_live_tup` / `pg_class.reltuples` for **size hints** in migration lint (not fingerprinted).

## 2. Managed Postgres provider constraints

| Concern | Why it matters | Notes (⚠ verify each) |
|---------|----------------|-----------------------|
| Event triggers | Push detection | Need superuser-equivalent; availability varies by provider |
| Custom extensions | "DB plugin" deployment shape | Generally only allow-listed extensions; a custom `dst` extension is unlikely to be allowed, which is why the **sidecar** shape is the default |
| Direct connections | `LISTEN` | Some serverless providers default to pooled endpoints; need the direct endpoint |
| `pg_stat_statements` | Consumer discovery | Usually available; may need enabling |
| Branching (Neon, Supabase, others) | **CI and preview envs**: create a copy-on-write branch of prod (schema + optionally data) per PR, apply in-PR change sets, run real checks | Excellent fit for high-fidelity CI checks when PGlite fidelity isn't enough |
| Private networking | Security | PrivateLink / VPC peering / private service connect; see [11](11-security.md) |

## 3. Local engine options

| Engine | Fidelity | Setup | Notes |
|--------|----------|-------|-------|
| **PGlite** (ElectricSQL) | Real Postgres code compiled to WASM; single-user | `npm i`, in-process | Default. ⚠ Verify: tracked PG major version, extension list (pgvector reportedly supported), Describe access, cold-start time, memory per instance |
| Real Postgres binaries (embedded-postgres style packages) | Exact version match | Downloads binaries per platform | Opt-in fidelity mode |
| Docker / Testcontainers | Exact version + any extension | Needs Docker | Opt-in; CI-friendly |
| Cloud branch | Exact prod engine + extensions | Needs provider API | CI / high-fidelity checks |
| pg-mem (JS emulation) | **Not real Postgres** | In-process | ❌ Not suitable: the type system would be emulated, not real |

**Stubbing strategy for missing extensions** 💡: from the snapshot, create `CREATE TYPE vector;`-like shells (with I/O functions borrowed from `text`) and functions/operators with matching signatures and dummy bodies. Describe only needs signatures. ⚠ Validate on pgvector and PostGIS with a spike: operator resolution, type categories and implicit casts may need more than shells.

## 4. Other engines

| Engine | Catalog | DDL change hooks | Describe-equivalent | Nullability info | Fit |
|--------|---------|------------------|---------------------|------------------|-----|
| **MySQL / MariaDB** | `information_schema` (also privilege-filtered) | No DDL triggers; **binlog carries DDL statements** (statement events), so CDC *is* an option here | `COM_STMT_PREPARE` returns param count and column definitions | Column flags include `NOT_NULL` for direct columns ⚠ | Good second target |
| **SQLite** | `sqlite_schema`, `PRAGMA table_info` / `table_xinfo` | None (file-level; poll `PRAGMA schema_version`) | `sqlite3_prepare` + `sqlite3_column_decltype` / `column_table_name` | `PRAGMA table_info.notnull` for direct columns | Simple; weak typing limits guarantees; great for local-first apps; no server (state service = file watcher) |
| **SQL Server** | `sys.*` catalog | **DDL triggers** (database-level) | `sp_describe_first_result_set`, `sp_describe_undeclared_parameters` | **Yes**: `is_nullable` per result column (best-in-class) | Strong fit; enterprise demand |
| **ClickHouse** | `system.tables`, `system.columns` | None; poll | `DESCRIBE (SELECT ...)` returns result types | Types carry `Nullable(T)` explicitly | Good fit for analytics consumers |
| **Oracle** | `ALL_*`/`DBA_*` views | DDL triggers | `DBMS_SQL.DESCRIBE_COLUMNS` | `null_ok` flag | Possible, low priority |
| **CockroachDB / YugabyteDB / AlloyDB / Aurora** | PG-compatible catalogs (with gaps) | Varies | PG wire Describe | As PG | PG adapter + compatibility profile |
| **DuckDB** | `duckdb_columns()` etc. | None | Prepared statement result types | Partial | Embedded analytics |

Per-engine adapters implement: `extractSnapshot`, `detectChanges`, `describe`, `inferNullability`, `parse`, `rebuildLocal`, `lintRules`.

## 5. Spikes this document implies

1. Observer extraction for PG 14–18: correctness and timing on a 1k-table schema.
2. Event-trigger availability matrix on 5 managed providers.
3. Describe without `SELECT` privilege (for the record, even if we don't depend on it).
4. PGlite: Describe API, extension list, cold start, memory, version.
5. Extension stubbing for pgvector and PostGIS.
6. `pg_stat_statements` → footprint matching accuracy.
