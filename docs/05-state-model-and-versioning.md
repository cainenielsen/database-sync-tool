# 05 — Schema State Model & Versioning

The user explicitly flagged versioning as unexplored: *"we would have to explore how versioning would work"*. This document does that exploration.

## 1. What counts as "state"

State is everything a query's validity or type could depend on, plus what schema-management tools need to decide whether an operation is valid or already applied (R17).

### 1.1 Postgres inventory (initial scope)

| Category | Objects / properties | Why it matters |
|----------|----------------------|----------------|
| Server | `server_version_num`, relevant GUCs (`search_path` default, `standard_conforming_strings`, `TimeZone`?, `DateStyle`?) | Syntax/semantics differ by version; `search_path` resolves unqualified names |
| Extensions | name, version, schema | Types/operators/functions they provide (`vector`, `pg_trgm`, `postgis`, ...) |
| Namespaces | schemas | Name resolution, ownership boundaries |
| Types | base, enum (labels, **order**), domain (base type, `NOT NULL`, checks), composite, range/multirange, array element types | Type mapping; enum → TS union; domain nullability |
| Tables | name, kind (regular, partitioned, foreign, unlogged), partition parent/bounds | Existence, queryability |
| Columns | name, ordinal, type + **typmod** (e.g. `varchar(50)`, `vector(1536)`, `numeric(10,2)`), `NOT NULL`, default, identity, generated (stored/virtual), collation | Core typing; insertability (generated/identity columns can't be inserted normally) |
| Constraints | PK, unique, FK (with actions), check, exclusion; deferrable | `ON CONFLICT` targets, nullability reasoning, join cardinality reasoning |
| Indexes | definition, uniqueness, partial predicate | `ON CONFLICT` inference needs unique indexes; migration linting |
| Views | columns (type, *reported* nullability), definition, `security_invoker`, updatability | Queryable shapes; view-definition dependencies |
| Materialized views | as views + indexes | |
| Functions / procedures | signature (arg types, modes, defaults, variadic), return type (scalar, `SETOF`, `TABLE(...)`), volatility, strictness, language, security definer | Call typing; `STRICT` implies null-in → null-out |
| Operators / casts | user-defined ones (and extension-provided) | Expression typing |
| Triggers | table, timing, events, function | Not needed for typing; needed for impact analysis & awareness |
| Event triggers | | Awareness |
| RLS | enabled/forced flags, policies | Awareness; future "this query returns rows only for role X" warnings |
| Sequences | | Defaults/identity |
| Privileges (optional) | grants per role | 💡 "Your service's DB role can't `UPDATE` this table" as a build error. Powerful, but optional because it's sensitive |
| Comments | `COMMENT ON ...` | 💡 Carries annotations: JSON shapes of `jsonb` columns, nullability overrides, ownership tags (prior art: PostGraphile "smart comments") |

### 1.2 Explicitly *not* state

- Row data, statistics, sizes, bloat. (Optional "hints" like approximate row counts could feed migration lint: "this table has 80M rows; `CREATE INDEX` without `CONCURRENTLY` will lock it". Treated as separate, non-fingerprinted metadata.)
- Session-local objects: temp tables/views in `pg_temp_*` schemas. Filtered out; see [§7](#7-dynamic-and-ephemeral-state).
- Roles/users and their secrets.

## 2. Snapshot format

### 2.1 Requirements

- **Canonical**: the same logical state always serializes to the same bytes (sorted keys, normalized whitespace in definitions, OIDs *removed*, since OIDs differ across instances).
- **Engine-neutral envelope, engine-specific body**: a common envelope plus a per-dialect body schema.
- **Diffable**: object-level identity by qualified name, so diffs are meaningful.
- **Versioned format** (`formatVersion`) with forward-compat rules.
- **Signable**: detached signature over the canonical bytes.

### 2.2 Sketch

```jsonc
{
  "formatVersion": 1,
  "dialect": "postgres",
  "server": { "versionNum": 170002, "settings": { "search_path": "\"$user\", public" } },
  "extensions": [{ "name": "vector", "version": "0.8.0", "schema": "public" }],
  "schemas": {
    "public": {
      "types": {
        "user_status": { "kind": "enum", "labels": ["active", "suspended", "pending"] }
      },
      "tables": {
        "users": {
          "kind": "table",
          "columns": [
            { "name": "id",         "type": "uuid",        "notNull": true,  "default": "gen_random_uuid()" },
            { "name": "email",      "type": "text",        "notNull": true },
            { "name": "status",     "type": "user_status", "notNull": false },
            { "name": "embedding",  "type": "vector", "typmod": 1536, "notNull": false },
            { "name": "created_at", "type": "timestamptz", "notNull": true,  "default": "now()" }
          ],
          "constraints": [
            { "name": "users_pkey", "kind": "primary", "columns": ["id"] },
            { "name": "users_email_key", "kind": "unique", "columns": ["email"] }
          ],
          "indexes": [],
          "triggers": [],
          "rls": { "enabled": false, "policies": [] }
        }
      },
      "views": {},
      "functions": {
        "get_user_post_count(uuid)": {
          "args": [{ "name": "target_user_id", "type": "uuid", "mode": "in" }],
          "returns": { "type": "int4", "set": false },
          "volatility": "stable", "strict": false, "language": "plpgsql"
        }
      }
    }
  }
}
```

Envelope (stored alongside, not part of the fingerprint):

```jsonc
{
  "fingerprint": "sha256:9f3a…",
  "objectFingerprints": { "public.users": "sha256:…", "public.get_user_post_count(uuid)": "sha256:…" },
  "source": { "env": "prod", "observer": "dst-observer@prod-eu-1", "capturedAt": "2026-10-24T14:03:11Z" },
  "transition": { "from": "sha256:c21e…", "id": "teamB/migrations/0042@sha256:…", "attributed": true },
  "signature": { "alg": "ed25519", "keyId": "dst-prod-2026", "sig": "…" }
}
```

### 2.3 Normalization pitfalls (to be tested)

- View and function definitions: use `pg_get_viewdef` / `pg_get_functiondef` output, which Postgres normalizes, but it can still differ across major versions. Fingerprints may change on a PG upgrade with no logical change. Acceptable: record it as an attributed "server upgrade" transition.
- Defaults: `pg_get_expr` output. Same caveat.
- Column order: semantically meaningful (`SELECT *`, `INSERT` without a column list), so keep it.
- Enum label order: meaningful (comparison), so keep it.
- Names: always schema-qualified internally.

## 3. Identity and versioning

There are three plausible versioning models. They aren't mutually exclusive.

### 3.1 Model A: linear version numbers (Flyway-style)

`V1, V2, V3...` per database, from the migration tool.

- ✅ Simple, familiar.
- ❌ Only meaningful if *one* tool owns *all* changes. Manual changes and multiple owners break it. Two branches both writing `V43` conflict.
- **Verdict:** keep as *metadata* (the transition ID from whichever tool applied it), but not as the identity of state.

### 3.2 Model B: content-addressed state (git-style) ← recommended identity

State identity = hash of the canonical snapshot.

- ✅ Tool-agnostic: the same state reached by different paths (Prisma vs hand-written SQL) gets the same identity.
- ✅ Equality across environments is trivial: "staging == prod?" is a fingerprint comparison.
- ✅ Natural dedupe, caching and signing.
- ❌ Fingerprints are opaque; humans need names → environments act as refs and transitions carry human-readable IDs.
- ❌ Sensitive to normalization bugs (two equal states hashing differently). Needs a strong canonicalizer and tests.

### 3.3 Model C: lineage graph of transitions (event-sourced)

A DAG: nodes are states (fingerprints) and edges are transitions (change sets, attributed or not), annotated with environment and time.

- ✅ Answers "how did prod get here?", "what is staging ahead of prod by?", "which change introduced this column?".
- ✅ Gives claims something to point to (a transition ID).
- ❌ Unattributed edges (drift) only have "before" and "after" plus a computed diff, no intent.

**Recommendation:** identity = **B**, history = **C**, human labels = **A** (as transition metadata).

### 3.4 Environments as refs

```
prod     → 9f3a…   (observed 2026-10-24T14:03Z)
staging  → 7b10…   (= 9f3a + teamB/0043)
dev      → 7b10…
pr-1234  → e5d2…   (= 7b10 + mine/0044)    # ephemeral
local    → e5d2…   # my machine, rebuilt from migrations
```

`dst status` shows how refs relate: `staging is ahead of prod by 1 transition (teamB/0043, applied 2h ago)`.

### 3.5 Object-level fingerprints and partial states

Whole-state fingerprints change on *any* change. For checking code, **object-level fingerprints** plus **footprints** are what matter:

- A footprint references objects; the check is "is every referenced object's relevant shape compatible?"
- Unrelated changes (a new table, a new index) don't invalidate cached analysis of queries that don't touch them.
- 💡 The analyzer cache key can be `hash(sql, fingerprints of the objects in this query's footprint, mapping config)` rather than the whole-state fingerprint. A big incremental-analysis win: most schema changes then re-analyze almost nothing.

## 4. Diffing

`diff(S1, S2)` → a list of object-level changes:

```
+ table public.invoices
~ table public.users
    + column plan text NULL
    ~ column email: notNull false → true
- function public.legacy_score(int4)
~ enum public.user_status: + label 'deleted' (appended)
```

Each change is classified by **compatibility** relative to consumers, much like Confluent Schema Registry's BACKWARD/FORWARD modes or Buf's breaking-change rules:

| Change | Safe for existing readers? | Safe for existing writers? |
|--------|---------------------------|----------------------------|
| Add nullable column | ✅ (except `SELECT *` consumers with positional assumptions) | ✅ |
| Add NOT NULL column with default | ✅ | ✅ |
| Add NOT NULL column without default | ✅ | ❌ (inserts without it fail) |
| Drop column | ❌ if referenced | ❌ if referenced |
| Rename column | ❌ (equivalent to drop + add for consumers) | ❌ |
| Widen type (`int4 → int8`) | ⚠ TS type may change (`number → bigint/string`) | ✅ |
| Narrow type | ✅ for readers (mostly) | ❌ possibly |
| nullable → NOT NULL | ✅ (readers get a stricter type) | ❌ writers passing null |
| NOT NULL → nullable | ❌ readers assuming non-null | ✅ |
| Add enum label | ⚠ exhaustive `switch` in code breaks | ✅ |
| Change function return type | ❌ if called | |

The **footprint check** gives a precise answer per consumer instead of these generic rules: the rules are the fallback when no footprints are registered.

## 5. Distribution and freshness

### 5.1 Getting state to authors

1. **Push (online):** subscribe to the env stream and get head changes in seconds.
2. **Pull (on demand):** `dst pull` or daemon polling every N minutes.
3. **Lockfile (committed):** pins what the code was checked against; works offline and in CI.
4. **Committed snapshots (hermetic):** CI needs no State Service access. Trade-off: schema metadata lives in the repo ([11](11-security.md)).

### 5.2 Staleness is bounded, not eliminated

❌ *Correction:* the source conversation said staleness becomes "mathematically impossible". It doesn't. Between a schema change and your build there is always a window. What `dst` provides:

- **Visibility:** every check reports which fingerprint it ran against and how old the observation is.
- **Bounding:** push keeps editors within seconds when online; deploy gates check the *actual* state at deploy time, which is the moment that matters.
- **A safety net:** the runtime startup check catches anything that slipped through (the state changed between gate and boot).

### 5.3 Which environment does code type-check against?

This is subtle and important. Code may be checked against:

- `local`: what I'm developing against (fast iteration, includes my unapplied changes).
- The **deployment target(s)**: where this code will run. Usually `prod` (plus intermediate stages).

**Proposed default:** the editor shows diagnostics against **`local` (or `dev`)** and shows **warnings** for requirements that `prod` doesn't satisfy yet ("needs claim"). CI is strict against every configured target env.

```ts
// dst.config.ts
export default defineConfig({
  environments: {
    local:   { source: { migrations: "./migrations" } },        // rebuilt locally
    staging: { source: { service: "https://dst.internal/", env: "staging" } },
    prod:    { source: { service: "https://dst.internal/", env: "prod" } },
  },
  editor: { primary: "local", alsoWarnAgainst: ["prod"] },
  ci:     { targets: ["staging", "prod"] },
});
```

## 6. Multi-database and multi-schema

- A project can target several databases (`main`, `analytics`). Each query is bound to one through its client instance (`mainDb.sql`, `analyticsDb.sql`), and each database has its own environments and lockfile entries.
- Postgres schemas inside one DB are just namespaces in the snapshot. Ownership can be assigned per schema ([09](09-schema-authoring-and-migrations.md)).

## 7. Dynamic and ephemeral state

The user stressed that schema "sometimes needs to change on the fly or be dynamic, e.g. creating a dynamic view and trashing it when done". There are three distinct kinds:

| Kind | Example | Model |
|------|---------|-------|
| **Query-local** | CTEs, subqueries, `VALUES` lists | Not state. Handled entirely by the analyzer (Describe already resolves them) |
| **Session/transaction-local** | `CREATE TEMP VIEW`/`TABLE` then use it, then `DROP` | **Local state overlay**: within a single function/transaction block, the analyzer applies preceding DDL to a *scratch copy* of the engine before describing the following queries. Flow-sensitive, but only within one lexical block (a feasible, narrow version of the user's "schema from control flow" idea; see [09 §6](09-schema-authoring-and-migrations.md)) |
| **Runtime-provisioned, persistent** | Per-tenant tables `tenant_123_events`, user-defined views | **Table families** 💡: declare a family pattern and a template shape; queries over the family are typed by the template. The Observer recognizes family members and checks they conform (and reports non-conforming members as drift) instead of listing thousands of tables |
| **Truly unknowable** | User-defined columns (no-code builders, EAV) | Escape hatch with runtime validation; or model as `jsonb` with declared shapes |

Table family sketch:

```ts
export const tenantEvents = family("tenant_events", {
  pattern: /^tenant_(?<tenantId>\d+)_events$/,
  template: "tenant_template_events",   // a real table (possibly empty) defining the shape
});

const rows = await db.sql`SELECT id, kind FROM ${tenantEvents.of(tenantId)} WHERE ts > ${since}`;
// analyzer substitutes the template table → fully typed
```

## 8. Open questions

- Hash algorithm and canonical JSON spec (RFC 8785 JCS?).
- How to fingerprint across PG major-version upgrades without false "changes".
- Retention of history; compaction.
- Whether to include privileges by default (sensitive vs useful).
- How to represent extension-provided objects: include them in full, or just "extension X@version" and resolve from a known catalog of extension shapes?
