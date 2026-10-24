# 07 — Type Safety in Depth

Requirement R1: type safety "both in and out, variables, attributes, etc." This document covers how types are *derived* (from the database), how they're *mapped* (to the host language), and how they're *delivered* to the type checker. Delivery is where the source conversation was most hand-wavy, and it's the biggest technical risk.

## 1. Where type information comes from

### 1.1 Postgres extended query protocol

Postgres can describe a statement without executing it:

1. **Parse** (`P` message): send SQL with `$1..$n` placeholders (parameter types optional; unspecified ones are inferred).
2. **Describe statement** (`D`, `S`): the server replies with
   - **ParameterDescription**: the inferred type OID of every parameter.
   - **RowDescription**: per result column, its name, **type OID**, **type modifier**, and, if the column comes straight from a table column, the **table OID and attribute number**. Otherwise those are zero.
   - or **NoData** for statements without results.
3. Never send **Execute**. Nothing runs, nothing is read or written.

This is how sqlx, SafeQL, pgTyped and postgres.js's `describe()` get their information. ⚠ Verify that PGlite exposes Describe through its JS API (or reach it via its wire-protocol mode).

❌ *Correction:* the source conversation repeatedly said type metadata comes from `EXPLAIN`. It doesn't. `EXPLAIN` shows a plan. It doesn't return result-column types, though `EXPLAIN (VERBOSE)` lists output expressions and can *help* with nullability (§3). Types come from Describe.

### 1.2 What Describe does *not* tell you

- **Nullability.** RowDescription has no nullable flag. This is the hardest output problem (§3).
- **Parameter nullability.** Whether `$1` may be `NULL` isn't described.
- **Typmod for parameters.** ParameterDescription only has type OIDs, so `vector(1536)` dimensions and `varchar(50)` lengths aren't enforced on inputs. ❌ *Correction:* the conversation claimed the tool would validate "that your input array matches the expected vector dimensionality". Describe can't do that. It's possible only with extra analysis (match the parameter to the column it's compared/assigned to, read that column's typmod, then require a TS tuple type or a runtime check). Even then, a TS tuple of 1536 numbers isn't practical; runtime validation is the realistic answer.
- **JSON shapes.** `json`/`jsonb` is opaque; it maps to `unknown` (or `JsonValue`) unless the author declares a shape (column comment annotation or config).
- **Exact numeric ranges / check constraints.** `CHECK (status IN (...))` doesn't turn a `text` column into a union. 💡 The analyzer *can* parse simple `IN`-list check constraints into unions. Enums are better and come for free from `pg_enum`.

## 2. Input safety (parameters)

### 2.1 Mechanics

- A frontend turns interpolations into placeholders: `` sql`... WHERE age > ${minAge}` `` → `... WHERE age > $1`, with `$1 ↔ minAge`.
- Describe gives `$1: int4`.
- Mapping (`int4 → number`) gives the **expected TS type** of each interpolation.
- The check: *TS type of the interpolated expression* is assignable to *the expected type*. That needs the TS type checker, which means a type-aware lint rule (typescript-eslint gives rules access to the checker) or a tsserver plugin, or the generated signature does it automatically (§5).

### 2.2 Subtleties

| Situation | Issue | Handling |
|-----------|-------|----------|
| `WHERE id = ANY(${ids})` | Param type is `uuid[]` | Expect `string[]` |
| `IN (${a}, ${b})` vs list expansion | Drivers differ: postgres.js expands arrays in some contexts | Frontend must model the driver's expansion rules |
| Unknown-typed params (`SELECT ${x}`) | Postgres infers `text` (or fails with "could not determine data type") | Report "add an explicit cast `${x}::int`" |
| Overloaded functions | Param type depends on resolution | Describe resolves it; trust it |
| Nullable params | `WHERE col = ${maybeNull}` is legal SQL but `= NULL` is never true | 💡 Lint: warn when a nullable TS value is compared with `=`; suggest `IS NOT DISTINCT FROM` |
| `INSERT` into NOT NULL column | Param must be non-null | Combine Describe with the target column's `attnotnull` → reject `T \| null` |
| Fragments / nested `sql` | Interpolated value is another SQL fragment, not a param | Frontend inlines fragment SQL before analysis (§6) |
| Identifiers | `${tableName}` can't be a param | Must use `sql.identifier()`; typed as a union of allowed names (§6) |

## 3. Output safety: nullability

The rules, roughly in order of reliability:

1. **Direct table column reference** (RowDescription has table OID + attnum): look up `attnotnull` in the snapshot. Non-null *unless* (2) applies.
2. **Outer joins:** columns from the nullable side of a `LEFT/RIGHT/FULL JOIN` become nullable. The AST knows which relation is on which side. Nested joins, `LATERAL`, and `USING` need careful handling.
3. **Expressions:** a small abstract interpreter over the AST:
   - literals: non-null (except `NULL`)
   - `COALESCE(a, b, nonNullLiteral)`: non-null if any argument is non-null
   - `count(*)`, `count(x)`: non-null. Other aggregates (`sum`, `max`, `avg`, ...): nullable (empty groups), *unless* in a grouped query where the group can't be empty... in practice, nullable.
   - operators/functions marked `STRICT`: null iff any input is null → non-null if all inputs are non-null
   - `CASE`: nullable if any branch is, or there's no `ELSE`
   - `EXISTS(...)`, `NOT x`, comparisons of non-null values: non-null
   - casts: preserve nullability
   - window functions: depends (`row_number()` non-null, `lag()` nullable)
4. **`WHERE x IS NOT NULL` refinements:** 💡 a filter on a column makes it non-null in the output (flow typing over predicates). Nice-to-have.
5. **Views:** ❌ *Correction:* the conversation said views are "handled identically to tables". For *types*, yes. For *nullability*, no: Postgres generally reports view columns as nullable (`attnotnull = false`) even when the underlying column is NOT NULL. Fix: resolve the view definition (`pg_get_viewdef` from the snapshot) and run the same inference *through* it, recursively. Expensive but cacheable per view fingerprint.
6. **Functions returning `TABLE(...)` / `SETOF composite`:** columns are nullable unless the analyzer inspects the function body (only feasible for `LANGUAGE sql`).
7. **`INSERT/UPDATE ... RETURNING`:** like a select over the target table; generated defaults are non-null if NOT NULL.
8. **`EXPLAIN (VERBOSE)` assist:** sqlx (Rust) is reported to use `EXPLAIN (VERBOSE, FORMAT JSON)` output to work out outer-join nullability in Postgres. ⚠ Verify sqlx's exact approach. Worth studying before reinventing.

When inference is uncertain, **default to nullable** (sound) and give the author an override:

```ts
// pgTyped-style suffix overrides inside SQL aliases (prior art: pgTyped `!` / `?`)
sql`SELECT max(created_at) AS "latest!" FROM posts WHERE user_id = ${id}`;
// or SafeQL/sqlx-style column aliases:  AS "latest: Date"   (to decide)
```

Overrides become requirements too. An override asserting non-null is a **claim about data** the tool can't verify. Mark it in the footprint as `asserted` and optionally check it at runtime in dev/test.

## 4. Type mapping (DB → host language)

Mapping isn't universal. **It's a contract between the analyzer and the runtime driver's parsers.** Sound typing requires that `dst` controls (or knows) the driver configuration.

### 4.1 Default TS mapping (proposed)

| Postgres type | `pg` default parse | Proposed `dst` default | Notes |
|---------------|-------------------|------------------------|-------|
| `int2`, `int4`, `float4`, `float8`, `oid` | `number` | `number` | |
| `int8` | `string` | **`bigint`** (configurable: `string` / `number`-unsafe) | ❌ *Correction:* the conversation's `get_user_post_count` returned `INTEGER` but its body returns `COUNT(*)`, which is `bigint`. In general `count(*)` → `int8` → `string` with `pg` defaults. A classic footgun |
| `numeric` | `string` | `string` (or a decimal lib) | Never `number` by default |
| `bool` | `boolean` | `boolean` | |
| `text`, `varchar`, `char`, `citext`, `name` | `string` | `string` | |
| `uuid` | `string` | `string` (optionally branded `Uuid`) | 💡 Branded types for ids: `UserId` from FK-aware naming |
| `timestamptz`, `timestamp` | `Date` | `Date` (configurable; `timestamp` without tz is a footgun) | Precision loss (μs → ms) |
| `date` | `Date` | **`string`** (`YYYY-MM-DD`) | Avoids timezone shifting bugs |
| `time`, `interval` | `string` | `string` / structured | |
| `json`, `jsonb` | parsed | `unknown` or declared shape | |
| `bytea` | `Buffer` | `Uint8Array` | |
| arrays | parsed arrays | `T[]` (elements nullable unless proven) | Postgres array elements are always nullable in principle |
| enums | `string` | union of labels | From `pg_enum` |
| domains | base type | base type (+ branded optional) | Domain `NOT NULL` affects nullability |
| composite | `string` (unparsed!) | parsed object or `string` | Needs parser registration |
| `vector` (pgvector) | `string` (`"[1,2,3]"`) | `number[]` **if** the runtime registers the pgvector parser | ❌ The conversation implied pgvector "just works". Typing it does; *decoding* needs driver setup |
| ranges | `string` | structured `Range<T>` | |
| PostGIS `geometry` | `string` (hex WKB) | `string` / GeoJSON with parser | |

### 4.2 Consequence: the runtime client matters

If the runtime used `pg` defaults while the analyzer assumed `int8 → bigint`, the types would lie. So:

- `dst` ships (or wraps) the runtime client and **registers the parsers the mapping assumes**, or
- the user declares the driver and its config, and `dst` derives the mapping from it.

Either way, the mapping config is part of the analysis cache key and is embedded in the generated output.

## 5. Delivering types to TypeScript (the crux)

❌ *Correction (major):* the conversation said an LSP plugin "dynamically injects TypeScript type definitions into your editor context inline" with "zero codegen", and that errors would appear "before you even save the file, without ever running a manual generation command." Several things get in the way:

1. **A tsserver (language-service) plugin can add diagnostics and completions, but it can't change the types `tsc` computes.** Plugins only run in the editor. `tsc` on the command line (and therefore CI) ignores them. A plugin can't make `const users = await db.sql\`...\`` *have* a computed type.
2. **Tagged template literals don't give TypeScript the literal text of the template.** The tag receives a `TemplateStringsArray`, not a string literal type, so type-level computation over the SQL text is impossible with tagged templates. ⚠ Verify against current TypeScript; this has been a long-standing limitation (there are open issues requesting literal-typed `TemplateStringsArray`).
3. **Language-service plugin APIs are synchronous**, and talking to a database (even an in-process WASM one) is asynchronous. SafeQL gets around this in ESLint by running async work in a worker and blocking on it synchronously (e.g. via `synckit`). ⚠ Verify. A tsserver plugin would need the same trick or a precomputed cache.
4. **TypeScript's native (Go) port** (TS 7 / "Corsa") changes the plugin and API story. ⚠ Verify the current status of language-service plugin support in the native compiler; this could break a tsserver-plugin-based design.

So, honestly, **some form of generated artifact or source annotation is unavoidable** for types to exist in `tsc`. The design goal becomes "**generation you never have to think about**", not "no generation".

### 5.1 Option A: inline annotations, lint-verified (SafeQL model)

```ts
const users = await db.sql<{ id: string; email: string; phone: string | null }>`
  SELECT id, email, phone FROM users WHERE status = 'active'
`;
```

- An ESLint rule (with typescript-eslint type info) analyzes the SQL, compares it with the annotation, reports a mismatch and **autofixes** it. Save → the annotation updates (format-on-save / fix-on-save).
- ✅ Plain `tsc` works, with no hidden files. Types are reviewable in diffs (a schema change shows up as a visible code change). Works offline once written. Tagged templates kept.
- ❌ Visual noise. Every query carries its type. Autofix churn on schema changes. Two sources of truth that lint keeps in sync.
- Param types: the tag signature can take a second generic for params, or the lint rule checks interpolations directly.

### 5.2 Option B: keyed generated declarations

```ts
const users = await db.query("users/active", sql`SELECT id, email, phone FROM users WHERE status = 'active'`);
// or: db.sql.users_active`...`  ← name as the key
```

The `dst` daemon writes `.dst/generated/queries.d.ts`:

```ts
declare module "@dst/client" {
  interface QueryRegistry {
    "users/active": { params: []; row: { id: string; email: string; phone: string | null } };
  }
}
```

- ✅ Clean call sites. `tsc` works if the generated file exists (generate in CI, or commit it).
- ❌ You have to name queries. The generated file goes stale if the daemon isn't running (CI must regenerate and diff). Key/SQL mismatch bugs need a lint rule.
- Variant **B′: key by SQL string literal** using `db.sql("SELECT ...", [params])` with a plain string literal. TS *does* see the literal type, so the registry can be keyed by the exact SQL text. No names needed, but you give up tagged-template interpolation ergonomics (positional params).

### 5.3 Option C: `.sql` files → generated modules (pgTyped / sqlc model)

```sql
-- src/queries/users.sql
-- name: activeUsers
SELECT id, email, phone FROM users WHERE status = 'active';
```
→ `src/queries/users.sql.ts` exporting `activeUsers(db): Promise<{...}[]>`.

- ✅ Best for big, complex queries; SQL gets full editor support as a real `.sql` file.
- ❌ Queries live away from the code that uses them. Generated files.

### 5.4 Option D: type-level SQL parsing

Generate only a **schema type** (`interface DB { users: { id: string; ... } }`) from the snapshot, and parse SQL *in the type system* from string literals (template literal types). Some experimental libraries do this. ⚠ Verify current examples.

- ✅ Only one tiny generated file (the schema); queries are typed by TS itself, instantly, with no daemon.
- ❌ Can't use tagged templates (see §5 point 2). Type-checker performance and recursion limits. Realistically only a SQL subset. Reimplements Postgres's type resolution in TS types. Not viable as the main path, but interesting for simple queries.

### 5.5 Option E: compiler patching / transformers

`ts-patch`-style hooks that let plugins affect `tsc`. Fragile and unofficial, and likely broken by the native port. **Not recommended.**

### 5.6 Recommendation

Support **A (inline annotations)** as the default for tagged templates, and **C (`.sql` files)** for large queries. Both work with plain `tsc` and are honest about what's generated. Keep **B′** as an option for teams that prefer clean call sites. All three share the same analyzer. The daemon + lint autofix + CI `dst check --verify-annotations` keep them in sync.

This is a **decision to validate with users** ([15](15-open-questions-and-decisions.md)). The inline-annotation noise may be a dealbreaker for some; if so, B′ or a VS Code "fold annotations" feature might help.

## 6. Dynamic queries

Static analysis needs the SQL text at analysis time. Real apps build SQL dynamically.

| Pattern | Example | Strategy |
|---------|---------|----------|
| **Optional filters** | add `AND status = $x` if a filter is set | (a) write it statically: `WHERE (${status}::text IS NULL OR status = ${status})`, fully analyzable; (b) **fragment composition** where each fragment is analyzed in the context of its parent (the parent query is analyzed with every fragment slot filled by its declared alternatives) |
| **Dynamic ORDER BY / column choice** | sort by a user-chosen column | `sql.identifier(col)` with `col` typed as a union of allowed column names; the analyzer checks **each alternative** (finite union) |
| **Bulk insert / VALUES lists** | insert N rows | `sql.values(rows)` helper with a known per-row shape → analyzed as a 1-row template |
| **IN lists** | `IN (...)` | Use `= ANY($1::type[])` (one param, analyzable) |
| **Dynamic table name** | tenant tables | **Table families** ([05 §7](05-state-model-and-versioning.md#7-dynamic-and-ephemeral-state)) |
| **Temp view then query** | `CREATE TEMP VIEW v AS ...; SELECT * FROM v` | **Local state overlay** within a transaction block ([05 §7](05-state-model-and-versioning.md#7-dynamic-and-ephemeral-state)) |
| **Truly dynamic** (query builders at runtime, user-authored SQL) | admin SQL consoles, report builders | **Escape hatch** with explicit type + runtime validation |

Composition explosion: a query with k independent optional fragments has 2ᵏ variants. Strategies: analyze each fragment independently against the parent's scope (enough for type and footprint correctness in most cases), and only analyze the "maximal" combined query for result shape. ⚠ Needs a design spike; this is where SafeQL-style tools usually give up.

### 6.1 Escape hatches: honest naming

❌ *Correction:* the conversation offered `db.sql.as<T>` as "explicit generic type safety". It's a **type assertion**, unchecked. Proposed API:

```ts
// unchecked: explicit and greppable
const rows = await db.unsafe.sql<WorkspaceUser>`SELECT * FROM ${sql.raw(table)}`;

// checked at runtime with any Standard Schema validator (zod, valibot, arktype...)
const rows = await db.sql.validate(WorkspaceUserSchema)`SELECT * FROM ${sql.raw(table)}`;
```

Escape-hatch queries still contribute a **partial footprint** where determinable (e.g. "touches some table in family X"), and are listed in `dst check` reports so they stay visible.

## 7. Other host languages (later)

| Language | Delivery mechanism |
|----------|--------------------|
| Python | Generated `.pyi` stubs / `TypedDict`s; a mypy/pyright plugin is possible for checking; `sql("...")` literal-keyed overloads (Python type checkers see literal strings, like option B′) |
| Go | Generate code from `.sql` files (sqlc-style), or feed sqlc a schema from the snapshot |
| Rust | sqlx already does this; `dst` can supply sqlx's offline data or a DB rebuilt from a snapshot |
| Java/Kotlin | jOOQ codegen from the rebuilt engine |

So for non-TS languages `dst` often doesn't need its own frontend at all. **It supplies authoritative state to existing tools** (sqlc, sqlx, jOOQ, kysely-codegen, Prisma). That's a cheap way to be polyglot early.

## 8. Testing the type system

- A **corpus** of real-world queries (from OSS projects) with expected types, used as a regression suite against several PG versions.
- **Differential testing:** for each query, execute it on seeded data and check that observed runtime values conform to the inferred types (catches nullability misinference).
- **Property tests** on the nullability interpreter (randomly generated joins/expressions vs actual execution on random data).
