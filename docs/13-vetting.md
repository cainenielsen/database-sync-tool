# 13 — Vetting: Corrections, Risks & Hard Problems

This is the critical-review document. Its job is to try to **break** the concept before we invest in building it.

## 1. Corrections to the source conversation

The source conversation was with an AI assistant that was consistently enthusiastic ("you nailed it", "the ultimate", "mathematically impossible"). Its technical details need auditing. Each item notes where the corrected treatment lives.

| # | Claim in conversation | Reality | Severity | See |
|---|----------------------|---------|----------|-----|
| C1 | An LSP plugin can "dynamically inject" types into TS with zero codegen; errors appear "before you even save" | tsserver plugins can't change `tsc`'s types and don't run in CI; tagged templates don't give TS the literal SQL; plugin APIs are synchronous. Some generated artifact or source annotation is unavoidable | **High**: affects the core DX promise | [07 §5](07-type-safety.md#5-delivering-types-to-typescript-the-crux) |
| C2 | Result types come from `EXPLAIN` | Types come from the extended protocol's Describe; `EXPLAIN (VERBOSE)` can only help with nullability | Medium | [07 §1](07-type-safety.md#1-where-type-information-comes-from) |
| C3 | Nullability comes back from the database ("Nullable: true because no NOT NULL") | Describe has no nullability; it must be inferred (outer joins, expressions, views); heuristic, sometimes needs overrides | **High**: the hardest correctness problem | [07 §3](07-type-safety.md#3-output-safety-nullability) |
| C4 | Views are "handled identically" to tables | Same for types; **not** for nullability (view columns report nullable) | Medium | [07 §3](07-type-safety.md#3-output-safety-nullability) |
| C5 | pgvector input dimensionality is validated | Param descriptions don't carry typmod; only possible with extra analysis, realistically at runtime | Low | [07 §1.2](07-type-safety.md#12-what-describe-does-not-tell-you) |
| C6 | `get_user_post_count` result typed as `number` | `COUNT(*)` is `bigint`; with `pg` defaults it comes back as a **string**. Type mapping depends on the driver's configuration | Medium | [07 §4](07-type-safety.md#4-type-mapping-db--host-language) |
| C7 | pgvector "just works" with no drivers | Typing works; decoding `vector` values needs a parser registered in the driver | Low | [07 §4](07-type-safety.md#4-type-mapping-db--host-language) |
| C8 | `db.sql.as<T>` gives "type safety" for dynamic queries | It's an unchecked assertion; real safety needs runtime validation | Medium | [07 §6.1](07-type-safety.md#61-escape-hatches-honest-naming) |
| C9 | Logical replication/CDC streams DDL changes | Native PG logical decoding doesn't carry DDL | Medium | [10 §1.4](10-database-features.md#14-logical-replication--cdc-mostly-not-useful-for-ddl) |
| C10 | Use `information_schema` for introspection | It's privilege-filtered; a least-privilege observer sees an incomplete schema. Use `pg_catalog` | Medium | [10 §1.1](10-database-features.md#11-system-catalogs-pg_catalog-the-snapshot-source) |
| C11 | Observer role: `GRANT SELECT ON ALL TABLES` + `REVOKE ... DROP` → "cannot exfiltrate data" | `GRANT SELECT` *grants* data access; `DROP` isn't a privilege (ownership-based) | **High** (security) | [11 §4](11-security.md#4-database-roles-least-privilege-done-right) |
| C12 | "Staleness becomes mathematically impossible" | Staleness is unavoidable; it can be bounded, made visible, and caught at gates/startup | Medium (expectation-setting) | [05 §5.2](05-state-model-and-versioning.md#52-staleness-is-bounded-not-eliminated) |
| C13 | Adding a NOT NULL column without default "would lock the table" | It *fails* on non-empty tables; constant defaults are metadata-only since PG 11; the real hazards are rewrites, scans and non-concurrent indexes | Low | [09 §4](09-schema-authoring-and-migrations.md#4-migration-safety-linting) |
| C14 | Rollbacks can be auto-generated safely | Structural reverse diffs yes; data loss can't be undone | Medium | [09 §3.5](09-schema-authoring-and-migrations.md#35-rollback--down-migrations) |
| C15 | A unified intermediate AST across dialects for polyglot syntaxes | Unnecessary and likely intractable; SQL text in the target dialect is the IR | Low (simplifies) | [02 §3.5](02-concept-overview.md#35-syntax-is-a-frontend-sql-is-the-intermediate-representation) |
| C16 | SQLGlot "ported to JS/TS as sqlingo" | Unverified; possibly hallucinated | Low | [12 §4](12-prior-art.md#4-parsing--analysis-building-blocks) |
| C17 | SafeQL "dynamically injects TypeScript type definitions into your editor context" | SafeQL is an ESLint rule that checks/autofixes a written type annotation ⚠ verify | Low | [12 §1](12-prior-art.md#1-typed-raw-sql-closest-to-dsts-editortype-layer) |
| C18 | Control-flow analysis can determine which tables exist at which point in an app | Undecidable in general; only narrow forms are feasible | Medium | [09 §6](09-schema-authoring-and-migrations.md#6-schema-from-code-what-survives) |
| C19 | "If PostgreSQL supports it, your tool supports it the day it's released" | Mostly true for *passing SQL through*; parsers (libpg_query version), PGlite version, nullability rules and type mappings all lag new releases | Low | [10 §3](10-database-features.md#3-local-engine-options) |
| C20 | CI should validate queries "against the actual production database schema" by connecting to it | Better: validate against signed snapshots of prod / a branch; CI shouldn't need prod connectivity for checks (only the deploy gate observes prod, via the Observer) | Medium | [06](06-claims-and-build-gates.md), [11](11-security.md) |

## 2. Hard problems, ranked

### H1. Type delivery into TypeScript without awful DX
The "no codegen" dream is off the table (C1). Remaining options all have costs: inline annotations (noise), keyed registries (naming), `.sql` files (separation), type-level parsing (limited). **If users find every option worse than Kysely/Drizzle, the on-ramp fails.** → Spike S3 plus user interviews.

### H2. Nullability inference accuracy
Wrong nullability is the most common way typed-SQL tools lie. False non-null → runtime crashes the type system promised wouldn't happen. False nullable → `!` everywhere, and users stop trusting the types. → Spike S1, with a large query corpus and differential testing.

### H3. Dynamic query composition
Most real codebases build some SQL dynamically. If `dst` only handles static SQL, users will hit the escape hatch constantly, and footprints become incomplete (which weakens the impact-analysis value too). → Design spike on fragment analysis.

### H4. Local engine fidelity
PGlite ≠ prod Postgres (version, extensions). Stubbing extensions from signatures is promising but unproven. A wrong describe → wrong types. → Spikes S2 and S4.

### H5. Adoption chicken-and-egg (the business risk)
The novel org value (registry, impact, claims) requires many teams to adopt. Nobody adopts until there's value. **Mitigations:**
- Single-player mode must be excellent and stand alone.
- **Passive consumer discovery** via `pg_stat_statements` gives impact analysis *some* value before anyone registers.
- Feed existing tools (sqlc, sqlx, kysely-codegen, Prisma) from snapshots so non-TS teams benefit without switching.
- Platform teams (who feel the pain of "can I drop this column?") are the buyers, and they can deploy it unilaterally (Observer + `pg_stat_statements` + `dst impact` on their own migration PRs).

### H6. Scope: this is five products
Typed-SQL editor tooling, a migration tool, a schema registry, a deploy-gating system and SDK generators. Each is a company-sized category. **Mitigation:** strict phasing ([14](14-roadmap.md)); integrate rather than build wherever an established tool exists (Atlas, Squawk, pgroll, sqlc).

### H7. Competition / overlap
- **Atlas (Cloud)**: declarative schema, linting, drift, registry. If it already does env-aware schema registry + impact checks against code, the gap narrows sharply.
- **Bytebase**: change review and drift.
- **Prisma** (TypedSQL, Accelerate/Optimize, Postgres offering): could move into this space.
- **Supabase**: owns the Postgres LSP, type generation and branching; well placed to build parts of this.
→ Spike S0 (competitive teardown) **before** significant building.

### H8. Rolling-deploy semantics and multi-instance environments
Getting the compatibility-window logic right ([06 §4](06-claims-and-build-gates.md#4-rolling-deploys-and-the-compatibility-window)) across regions, shards and DB-per-tenant fleets is subtle. Errors here mean false confidence. → Model formally; test with simulated deploy timelines.

### H9. TypeScript platform shift
The native TS compiler (Go) may not support language-service plugins the same way, at least initially. ⚠ Verify current status. A design that relies on tsserver plugins could be stranded. → Prefer ESLint + generated files + a standalone LSP over tsserver plugins.

### H10. Performance in large monorepos
Thousands of queries, a dozen environments, frequent schema changes. Needs object-level cache keys and incremental analysis from day one.

## 3. Assumptions register

| ID | Assumption | Test | Status |
|----|-----------|------|--------|
| A1 | PGlite can describe statements with <100 ms latency | Spike S2 | Untested |
| A2 | A snapshot can be rebuilt into an engine that describes queries identically to the source DB, for ≥95% of a real corpus | Spike S2/S4 | Untested |
| A3 | Nullability inference can reach ≥98% agreement with runtime observation on a real corpus, with sound defaults | Spike S1 | Untested |
| A4 | Catalog-only observer role can extract a complete snapshot on managed providers | Spike S5 | Untested |
| A5 | Event triggers are available on major managed providers (or polling is good enough) | Spike S5 | Untested |
| A6 | Developers accept one of the type-delivery options (A/B′/C) | Interviews + prototype S3 | Untested |
| A7 | Platform teams would deploy an Observer + use impact checks even before app teams adopt | Interviews | Untested |
| A8 | No existing product already offers env-aware SQL consumer registry + impact + gating | Spike S0 | Untested |
| A9 | `pg_stat_statements` matching can find meaningful unregistered consumers | Spike S6 | Untested |
| A10 | Fingerprint canonicalization is stable across minor versions and instances | Spike S2 | Untested |

## 4. Kill / pivot criteria

Stop or pivot if:

- **K1:** S0 finds an existing product covering the registry + impact + gating space well → pivot to contributing integrations (e.g. a typed-SQL frontend on top of that product's registry) or narrow to the editor tool.
- **K2:** S1 nullability can't get past ~95% agreement without pervasive overrides → typed raw SQL is fundamentally a worse UX than builders; pivot `dst` to "state service that *feeds builders/ORMs*" (Kysely/Drizzle/Prisma/sqlc types from authoritative env state) and drop the raw-SQL analyzer as the centerpiece.
- **K3:** S3 shows every TS delivery option is disliked → same pivot as K2.
- **K4:** A7 is rejected by interviewed platform teams → org features have no buyer; consider shipping only the single-player tool (and question whether that's differentiated enough versus SafeQL/sqlx).

## 5. Steelman of "don't build this"

- Most teams have one service per database and one pipeline that migrates and deploys. For them, Drizzle/Prisma + "run migrations before deploy" is good enough, and the claims/registry machinery is overhead.
- Typed raw SQL has existed for years (sqlx, pgTyped, SafeQL) and stayed niche in TS land. Maybe developers actually prefer builders.
- Organizations with the multi-team problem often fix it socially (DB ownership, review boards, views as APIs, a "never drop columns" policy) and won't adopt another control-plane service.

**Response:** each point is real, which is why the plan validates the *org* value (A7, A8) with platform teams early and refuses to depend on raw-SQL adoption alone (K2 pivot path). The idea has a fallback: **authoritative, environment-aware schema state as a service that feeds every existing tool**. That's useful even if nobody writes raw SQL.

## 6. Overall assessment (initial)

- **Soundness:** the core model (observed state as truth, content-addressed versions, footprints, claims verified at deploy gates, consumer registry) is coherent and maps onto proven patterns from other ecosystems (Apollo, Pact, Confluent, Kubernetes).
- **Novelty:** low for the editor/type layer, **high for the cross-repo state/registry/claims layer** in the SQL world (pending S0).
- **Feasibility:** each component is buildable with existing parts; the hard parts are correctness (nullability, fidelity) and DX (type delivery), not raw engineering.
- **Value:** highest for multi-team orgs on shared Postgres; moderate for solo devs (crowded space).
- **Recommendation:** proceed to the Phase 0 spikes ([14](14-roadmap.md)). Don't start Phase 1 until S0, S1, S2 and S3 report.
