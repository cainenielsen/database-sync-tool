# 12 — Prior Art & Building Blocks

The landscape, what to reuse, and where the actual gap is. Facts about third-party tools are from memory and **must be re-verified** (⚠) before decisions rely on them. Tools change quickly.

## 1. Typed raw SQL (closest to `dst`'s editor/type layer)

| Tool | Ecosystem | How it works | What it lacks relative to `dst` |
|------|-----------|--------------|--------------------------------|
| **sqlx** | Rust | `query!` macros connect to a DB at compile time (or use committed offline metadata in `.sqlx/`), describe queries, generate types. Nullability inference for Postgres reportedly uses `EXPLAIN` output | Single DB/env at a time; no environments, footprints, claims or registry. **The design benchmark for compile-time SQL** |
| **SafeQL** (`@ts-safeql/eslint-plugin`) | TS | ESLint rule; connects to a DB (or migrations → shadow DB), describes tagged-template queries, compares/autofixes the type annotation | Needs a running DB (Docker or URL); ESLint-only; no environments/registry/claims |
| **pgTyped** | TS | `.sql` files or `sql` tags → generated TS via a running DB; `!`/`?` nullability overrides | Codegen step + running DB |
| **sqlc** | Go (+ Kotlin, Python, TS via plugins) | Parses schema files + query files; generates typed code. Has its own static analyzer; can also use DB-backed analysis ⚠ | Schema must be in files; no live state |
| **Prisma TypedSQL** | TS | `.sql` files in `prisma/sql/` → typed functions via `prisma generate` (needs a DB connection) ⚠ | Prisma-centric |
| **Squirrel** | Gleam | `.sql` files → typed Gleam via the DB | Single ecosystem |
| **Zapatos**, **Kanel**, **kysely-codegen**, **Supabase `gen types`**, **pg-to-ts** | TS | Generate table types from a live DB | Tables only; queries not checked |
| **Slonik** | TS | Runtime validation of results with Zod | Runtime only |
| **jOOQ** | Java/Kotlin | Codegen from a live DB, typed DSL | DSL, not raw SQL; single env |
| **postgres.js** | JS | Driver with a `describe()` capability, tagged-template API | Primitive, not a typing system |
| **Postgres Language Server** (Supabase community) | Any editor | LSP for `.sql` files: completion, diagnostics, type checking via a DB connection ⚠ verify features | `.sql` files, not embedded SQL; no environments |

**Lesson:** the *editor/type* layer is a well-trodden path with good open-source components. `dst` should **reuse** (libpg_query, PGlite, maybe SafeQL's or pgTyped's nullability logic if the licenses allow) and **differentiate on the state layer**.

## 2. Schema management & migrations

| Tool | Relevant idea |
|------|---------------|
| **Atlas** (Ariga) | Declarative (desired-state) and versioned workflows; schema-as-code in HCL or SQL; diff engine; migration linting; **Atlas Cloud** offers schema registry/drift monitoring features ⚠ verify the current feature set and licensing (some features commercial). **Biggest overlap risk; study closely** |
| **Flyway / Liquibase** | Ledger tables, checksums, out-of-order policies, repair, baselines |
| **Sqitch** | **Dependency-based** change ordering (`requires`) rather than linear: a natural match for claims |
| **graphile-migrate** | Fast iteration loop: an idempotent `current.sql` re-applied on save, committed into a numbered migration when done. The same "auto-apply on save" loop the conversation described |
| **dbmate / golang-migrate / node-pg-migrate** | Minimal runners |
| **pgroll** (Xata), **Reshape** | **Zero-downtime migrations that serve multiple schema versions at once** through views. Directly addresses the rolling-deploy window ([06 §4](06-claims-and-build-gates.md#4-rolling-deploys-and-the-compatibility-window)). Integration candidates |
| **pg-schema-diff** (Stripe), **migra** (⚠ maintenance status), **Skeema** (MySQL declarative) | Diff engines |
| **Squawk**, **strong_migrations** | Migration safety linting |
| **Bytebase** | Database change management / review workflows, GitOps, drift detection ⚠ verify. Overlaps with the org/workflow side |
| **Prisma Migrate / drizzle-kit** | DSL → SQL generation; shadow-DB technique for diffing |

## 3. Contracts & registries in other ecosystems (the strongest analogies)

| Tool | Analogy to `dst` |
|------|------------------|
| **Apollo GraphOS schema checks / operation registry** (GraphQL) | Clients register the operations they use; schema changes are checked against real client operations before deploy. **This is `dst`'s consumer registry + impact check, for GraphQL.** Strong evidence the model is valuable |
| **Buf Schema Registry** (Protobuf) | Breaking-change detection rules, versioned schema modules, generated SDKs from a central registry |
| **Confluent Schema Registry** (Kafka) | Compatibility modes (BACKWARD, FORWARD, FULL) enforced on schema evolution |
| **Pact** (consumer-driven contracts) | Consumers publish expectations; providers verify against them; "can-i-deploy" gate. **Very close to footprints + deploy gates** |
| **dbt** (model contracts, manifests) | Data models with declared contracts; lineage graph; state comparison (`state:modified`) |
| **Terraform / Kubernetes** | Desired vs observed state, plan/apply, drift, reconciliation. The conversation's "control plane" analogy |
| **OpenAPI + generators** | Contract → SDK generation |
| **Hasura / PostGraphile / PostgREST** | Generate APIs from live Postgres schema; PostGraphile's **smart comments** for annotations |

## 4. Parsing & analysis building blocks

| Component | Use |
|-----------|-----|
| **libpg_query** (C; the real Postgres parser) + bindings: `libpg-query` / `pgsql-parser` (npm), `pg_query.rs`, `pg_query_go`, `pglast` (Python) | Exact Postgres parsing, fingerprinting of queries (it has a query fingerprint function), deparsing |
| **tree-sitter-sql** variants | Tolerant parsing for completions on incomplete SQL; editor highlighting |
| **sql-parser-cst** | Multi-dialect CST with whitespace and comments (good for code mods) |
| **node-sql-parser** | Multi-dialect JS parser |
| **sqlparser-rs** | Rust multi-dialect parser (used by DataFusion and others) |
| **SQLGlot** | Python multi-dialect parser/transpiler/optimizer. ❌ The conversation mentioned a JS port named "sqlingo". **Unverified; treat as nonexistent until confirmed** |

## 5. Embedded / ephemeral databases

| Component | Use |
|-----------|-----|
| **PGlite** | In-process WASM Postgres. Default local engine |
| **embedded-postgres**-style packages | Real binaries for exact-version fidelity |
| **Testcontainers** | Docker-based fidelity in CI |
| **Neon / Supabase / other branching** | Copy-on-write DB branches for PR checks |
| **pg-mem** | ❌ Emulation, not suitable |

## 6. Editor plumbing

- VS Code TextMate grammar injection; existing "SQL in template literal" highlighting extensions (⚠ survey which are maintained).
- TypeScript language-service plugins (editor-only), typescript-eslint type-aware rules, `synckit`-style sync-over-async for ESLint.
- LSP libraries: `vscode-languageserver` (Node), `tower-lsp` (Rust).
- ⚠ TypeScript native port (Go): track language-service plugin support.

## 7. Where the gap actually is

| Capability | Exists today? |
|-----------|---------------|
| Typed raw SQL in the editor against a DB | ✅ (sqlx, SafeQL, pgTyped, PG LSP). Mature-ish |
| Zero-Docker typed SQL via embedded PG | 🟡 Partial: some tools can use PGlite ⚠; not mainstream |
| Types against **multiple named environments** with staleness reporting | ❌ Rare |
| Snapshots as a **signed, distributable contract** decoupled from DB access | 🟡 sqlx offline data is per-project, unsigned, query-level |
| **Consumer registry + impact analysis for SQL schema changes across repos/languages** | ❌ / 🟡 (Atlas Cloud / Bytebase may cover parts ⚠) |
| **Claims + deploy gates** for schema/code ordering | ❌ Not as a general mechanism (Pact's can-i-deploy is the nearest analog) |
| State-observed (tool-agnostic) **drift + attribution** | 🟡 Atlas/Bytebase drift detection ⚠ |
| Syntax-agnostic: feeding sqlc/sqlx/jOOQ/Prisma/Kysely from one authoritative state | ❌ Each tool introspects separately |

**Positioning hypothesis:** "*Apollo GraphOS / Pact for SQL databases*", with an excellent typed-SQL editor experience as the on-ramp. Vetting must confirm that Atlas Cloud and Bytebase don't already occupy this position ([13](13-vetting.md), [14](14-roadmap.md) spike S0).
