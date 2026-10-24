# 00 — Origin & Conversation Summary

The concept came out of one long brainstorming conversation with an AI assistant (Google Gemini), exported as [`Alternatives to Modern Node ORMs.pdf`](../Alternatives%20to%20Modern%20Node%20ORMs.pdf) in the repo root. This document traces how the idea evolved, because **the path matters**: the final concept is very different from where the conversation started, and several intermediate designs were tried and dropped for good reasons.

> The assistant in the source conversation was quite enthusiastic and sometimes stated things as fact that aren't accurate (see [13-vetting.md](13-vetting.md)). The *user's* prompts carry the real design intent. The summaries below keep the two apart.

---

## Stage 1: "What's better than Kysely / Drizzle / Prisma / pgTyped?"

**User asked:** for options better than the mainstream TypeScript SQL tools.

**Assistant answered:** with a categorized list of alternatives: MikroORM (data mapper / unit of work / DDD), TypeORM (decorators / active record), SafeQL + native driver and Rust's `sqlx` (raw SQL with compile-time checks), Sequelize v7 (classic multi-dialect).

**Takeaway:** none of these is "better" across the board. Each just picks a different trade-off.

## Stage 2: "Integrating SQL into applications just kind of sucks"

**User observed:** whatever the language or tool, some options lack support for things and others make for ugly syntax.

**Assistant framed** three camps, each with its own compromise:

| Camp | Gives you | Costs you |
|------|-----------|-----------|
| Raw SQL | Full power, copy-paste into a terminal | Stringly-typed, typos crash at runtime |
| Query builders (Kysely, Drizzle) | Type safety, close to the metal | Verbose method chains and helper soup |
| Schema-driven ORMs (Prisma, TypeORM) | Great autocomplete & relations | Abstraction breaks on CTEs, window functions, DB-specific features |

Root cause named: the **impedance mismatch** between relational algebra and imperative/OO code.

## Stage 3: "We should write a new tool that fixes all of it"

First design sketch (assistant):

1. **No method chaining**: write real SQL in tagged template literals (`` db.sql`SELECT ...` ``).
2. **No black-box ORM**: SQL is first-class.
3. **No mandatory codegen step**: types come from compiler hooks / an LSP plus live introspection.
4. **Type inference**: an LSP/ESLint plugin parses the SQL, asks a dev database to describe it, and maps column types and nullability back to TS.
5. **Relations via batching primitives** (`db.batch({...})`), not object-graph magic, to avoid N+1 and cartesian blowups.
6. **MVP**: build a meta-tool on an existing driver (`postgres` by porsager, or `pg`), an ESLint/LSP plugin in the style of SafeQL, introspect on save.

## Stage 4: Schema definition & migrations

**User asked:** how schema definition and migrations would work.

**Assistant proposed:** *pure SQL DDL is the schema language*. No Prisma DSL, no Drizzle `pgTable`. Versioned migration files (`V1__init.sql`), dry-run validation with an AST parser, generated or companion rollback files, and a **live introspection loop**: an ephemeral dev DB, migrations auto-applied on save, the LSP reading the live catalog, so renaming a column turns dependent queries red right away.

## Stage 5: Existing building blocks

**User asked:** what already exists to do the under-the-hood parts?

**Assistant listed:** SQLGlot and `sql-parser-cst` / `libpg-query` (parsing), SafeQL (TS type checking of raw SQL), Rust `sqlx` (the blueprint for compile-time SQL, including an offline metadata cache for CI), Atlas (declarative SQL-DDL schema diffing). Proposed stack: `postgres` driver + `libpg-query` + SafeQL-style plugin + Atlas.

## Stage 6: Type safety in and out

**Assistant explained:** input safety (interpolations map to `$1..$n`, and parameter types come from the DB) and output safety (the extended-protocol *Describe* returns column types; nullability comes from constraints). It drew an end-to-end loop diagram. Some technical details here are wrong or simplified; see [07-type-safety.md](07-type-safety.md) and [13-vetting.md](13-vetting.md).

## Stage 7: Telling migration files from app code, and dynamic tables

**User asked:** how the tool tells migration files from application files, how it ensures ordering, and what happens with on-the-fly tables/views.

**Assistant proposed:** directory conventions (`migrations/` vs `src/`), lexicographic prefixes, a history table with checksums. Dynamic structures: CTEs and temp tables are in-query (the parser handles them); runtime DDL (per-tenant tables) goes through an `unsafe` escape hatch with user-supplied types.

## Stage 8: pgvector, views, functions, triggers

**Assistant argued:** since everything is SQL plus introspection, all of these are first-class. Views query like tables, function signatures are introspectable, triggers are invisible to app code. "If Postgres supports it, the tool supports it the day it's released." Mostly true, with caveats (view-column nullability, custom type parsing in drivers; see [07](07-type-safety.md)).

## Stage 9: Editor experience

**User asked:** for autocomplete, red squiggles and syntax highlighting for SQL inside `.ts` files.

**Assistant described:** a TextMate grammar injection (highlighting), a TS language-service plugin (diagnostics), a completion provider (cursor context in the SQL AST plus a schema cache). Packages: `@tool/core`, `@tool/lsp`, a VS Code extension.

## Stage 10: "I'd hate consumers to have to worry about the test database"

**User's requirement:** no Docker, no `.env`, no setup. It should just be under the hood.

**Assistant proposed:** run an **in-process embedded database**. PGlite (Postgres compiled to WASM) runs inside the language server, auto-applies migrations at boot and serves introspection. Production uses `DATABASE_URL`; dev defaults to `memory://`.

## Stage 11: Migration maintenance & versioning

**Assistant:** yes, it's needed. Immutable applied migrations, a checksum ledger, drift halts, squash/baseline, AST-based safety lint, generated rollbacks from snapshots.

## Stage 12: Schema defined outside the project

**User asked:** what if the schema is managed elsewhere, by this or another tool?

**Assistant proposed:** "Observer mode". `schemaSource: "external"` plus an introspection target, coexisting with Atlas/Flyway/Prisma Migrate, and a CI `dst check` that validates every query against the real schema.

## Stage 13: "We can't let local LSPs connect to the cloud database"

**User's constraint:** security and latency. **Assistant:** a two-tier, "air-gapped" design. The local sandbox (PGlite) only for dev; production touched only by CI/CD and runtime; CI validates against a production-equivalent container.

## Stage 14: "Another tool can always export SQL"

**User's insight:** even if another team uses Prisma, it outputs SQL, which can be dropped into migration files. **SQL is the universal interchange format.**

## Stage 15: "So every service has to copy the whole schema?"

**Assistant:** no. Use a shared migrations package in a monorepo, observer services that consume generated types or introspect a shared dev DB, or views per consuming service as an API contract.

## Stage 16: "Requiring migration files seems odd": schema from application structure

**User proposed:** derive ordering from the app's own control flow. Run typed `CREATE TABLE` at boot, and nested code sees the tables. A switch decides whether the app *owns* the DB (actually run the DDL) or *assumes* it exists (DDL is type-only).

**Assistant:** called it a "Declarative Local Topology Engine": control-flow-graph analysis, `CREATE TABLE IF NOT EXISTS` at boot, `mode: "assume-existing"`.

## Stage 17: "Then it's not just a SQL dump and commit anymore"

**User's pushback on their own idea:** schema-in-code means hand-copying DDL into JS files and restructuring apps around them. That makes them reconsider a schema DSL.

**Assistant:** this is the classic paradox. SQL-first means power plus sync pain; DSL-first means smooth DX plus abstraction walls.

## Stage 18: The reframe: **it's a state management problem** ← the key moment

**User:**
> "really i think it boils down to a state management problem, database have lots of use cases, a common one is to declare schema, apply it, then build applications around it, but sometimes that schema needs to change on the fly or be dynamic, eg. creating a dynamic view and trashing it when done, and regardless of that even applications interfacing with the database might not be the thing declaring its schema or making migrations, separate repos, etc. there is always staleness, and we want to build a tool that is all-flexible and ensures type safety"

**Assistant:** treat schema as a *live, polymorphic contract*. Pluggable schema-source providers (local migrations / remote DB / registry), continuous background sync, typed escape hatches for dynamic structures.

## Stage 19: The concept crystallizes: **a database state server**

**User (the core concept, close to verbatim):**
> Maybe the best case is some sort of **database state server** served alongside your database, maybe a plugin. It emits the database's details and state, version, plugins, etc. We build tools around it that let application authors **manage database state** (define tables, functions, plugins) and those tools automatically ensure operations are valid, or whether they already succeeded. We'd have to **explore how versioning would work**. Separately or in the same repo, authors **make requests against the database that are also state- and type-safe**. Both processes (structure *and* data) can be written **however you want**: custom schema DSL, schema in language, plain `.sql`, inline SQL, query builder, abstract fluent syntax. All of them are **aware of the database's state** and give syntax highlighting, red squiggles, and **prevent the application from compiling/building** unless the author declares an **authority**, e.g. declaring the related state change in the same PR and **claiming it WILL run before the application code**, or making the change some other way (manually, another tool, another repo, another language, who cares). The database state **syncs periodically and is broadcast to authors**, and the next build succeeds. Support syncing with local and cross-environment databases. Tools can use this to **generate SDKs / language-level methods** (ORM-like, or not, up to the author).

**Assistant:** compared it to a **Kubernetes-style control plane** (desired state, reconciliation, broadcasting): a state server as source of truth, an intent/claim mechanism for the chicken-and-egg problem, polyglot syntax ingestion, drift detection, optional SDK generation.

## Stage 20: Database features & securing the sidecar

**User asked:** what DB features support a sidecar, and how to restrict it to employees, private networks and in-org CI/CD.

**Assistant listed:** system catalogs, event triggers + `LISTEN/NOTIFY`, logical replication/CDC. For security: private VPC + zero-trust access (Cloudflare Zero Trust, Tailscale, client VPN) with SSO/MFA, mTLS with an internal CA, short-lived scoped JWTs, a least-privilege observer DB role. Several details here are wrong; see [10](10-database-features.md), [11](11-security.md), [13](13-vetting.md).

## Stage 21: Local / offline development

**User asked:** we can't always be connected; maybe a cache?

**Assistant proposed:** a local **state snapshot / lockfile** (`.dreamdb/schema.lock`) synced when online and used offline, PGlite for actually running queries offline, and **reconciliation on reconnect** (offline claims, schema diff check, drift warning, CI blocks until resolved).

---

## What survived, what was dropped

| Idea | Status | Why |
|------|--------|-----|
| Real SQL in template tags / `.sql` files as one first-class syntax | **Kept**, as *one of several* frontends | The user later widened it to "write it however you want" |
| Pure SQL DDL as the *only* schema language | **Relaxed** | SQL stays the interchange format; authoring style is pluggable |
| Migration files as the required source of truth | **Dropped as a requirement** | The *database state* is the source of truth; migrations are one way to change it |
| Schema derived from app control flow | **Dropped as a general mechanism** | Hand-copying DDL, restructuring apps, undecidable flow analysis. A narrow form survives (see [09](09-schema-authoring-and-migrations.md)) |
| Ephemeral embedded DB (PGlite) for dev | **Kept, repurposed** | Now it's built *from a state snapshot* to answer type questions offline, not the source of truth |
| Local LSP connecting directly to cloud DB | **Rejected** | Security and latency. The state service and snapshots replace it |
| State server alongside the DB, broadcasting state | **Core of the concept** | |
| Claims / authority to gate builds | **Core of the concept** | Needs formalizing; see [06](06-claims-and-build-gates.md) |
| Syntax-agnostic frontends | **Core of the concept** | |
| Optional SDK/ORM generation from state | **Kept, as a downstream consumer** | |
| Local cache / lockfile for offline | **Core** | |
