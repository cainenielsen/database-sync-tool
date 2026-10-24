# 15 — Open Questions & Decision Log

## 1. Decision log

Format: **ID — Decision** (status). Context → choice → consequences. Statuses: `proposed`, `accepted`, `rejected`, `superseded`.

### D1 — The database's observed state is the source of truth (proposed)
Context: migration files, ORMs and manual changes all modify the DB; any file-based source of truth is wrong whenever another writer exists. Choice: the authoritative state is what the Observer reads from the catalog; files are change sets. Consequences: tool-agnostic; requires an Observer or snapshots for non-local envs; drift becomes detectable.

### D2 — Content-addressed state identity + lineage history + transition labels (proposed)
See [05 §3](05-state-model-and-versioning.md#3-identity-and-versioning).

### D3 — SQL text (target dialect) is the IR; no cross-dialect AST (proposed)
See [02 §3.5](02-concept-overview.md#35-syntax-is-a-frontend-sql-is-the-intermediate-representation), correction C15.

### D4 — Developer machines never connect to shared/prod databases (accepted, user requirement R7)
Snapshots + local engines only.

### D5 — Describe happens on local engines rebuilt from snapshots, never on real shared DBs (proposed)
Security (no query probing of prod), no load on prod, works offline. Fidelity risk handled by stubbing / opt-in real engines / CI branches.

### D6 — PGlite is the default local engine (proposed, pending S2)

### D7 — Type delivery: inline annotations (lint-autofixed) + `.sql` files; literal-keyed registry optional (proposed, pending S3)
See [07 §5.6](07-type-safety.md#56-recommendation). Rejects "no generated artifacts at all" as infeasible (C1).

### D8 — Prefer ESLint + standalone LSP + daemon over tsserver plugins (proposed)
Reason: tsserver plugins don't affect `tsc`, are synchronous, and are at risk from the TS native port (H9).

### D9 — Claims are fulfilled only by observation (proposed)
No API can mark a claim fulfilled; only an observed state can. Integrity property for gates.

### D10 — Observer role is catalog-only; features needing row data are out of scope (proposed)
See [11 §4](11-security.md#4-database-roles-least-privilege-done-right).

### D11 — Integrate rather than build: migration engines (Atlas/Flyway/Sqitch/...), linters (Squawk), multi-version schemas (pgroll) (proposed)
A minimal built-in migration runner is still shipped for single-player mode.

### D12 — Phasing: single-player → snapshots/envs → service → registry → claims (proposed)
See [14](14-roadmap.md).

### D13 — Postgres first; MySQL and SQL Server next (proposed)

### D14 — First implementation language: TypeScript for analyzer/daemon/CLI (proposed)
Revisit Rust for the core if polyglot embedding or performance requires it.

## 2. Open questions

### Product & scope
- **Q1.** Name. "database-sync-tool" suggests row/data sync (ElectricSQL, PowerSync). Is "sync" the right word for *state/schema* sync? Alternatives to brainstorm: something around "contract", "state", "ledger", "registry".
- **Q2.** Who is the first customer: solo TS developers (crowded) or platform teams (novel, slower sales)? The roadmap builds for the former first but validates the latter early (S7).
- **Q3.** Open source vs open core vs hosted? Which pieces are OSS (analyzer, CLI, editor, snapshot format) vs commercial (hosted State Service, registry UI, gates)?
- **Q4.** Is ORM/SDK generation (R14) in scope for the core project or left to the ecosystem consuming snapshots?

### Model
- **Q5.** Exact requirement vocabulary for footprints: how fine-grained? (column type equality vs compatibility; nullability only when relied on; insertability.)
- **Q6.** How do feature-flagged code paths express conditional requirements?
- **Q7.** How to represent multi-instance environments (regions, shards, tenant fleets) in gates?
- **Q8.** Should privileges/grants be part of state (powerful "your role can't UPDATE this" checks) given their sensitivity?
- **Q9.** Canonical JSON scheme (RFC 8785?) and hash algorithm; behaviour across PG major upgrades.
- **Q10.** Do data-level facts ever matter (e.g. "this lookup table contains row X")? Probably out of scope; confirm.

### Type system
- **Q11.** Default for `int8` (`bigint` vs `string`) and `timestamp`/`date` mappings.
- **Q12.** Syntax for nullability overrides inside SQL (`"col!"`, `"col?"`, comments, or annotation-side).
- **Q13.** Fragment composition design. Which patterns are supported statically?
- **Q14.** Branded ID types from FK relationships: default on or off?

### Workflow
- **Q15.** Lockfile update policy: manual `dst pull` only, or a bot PR when an env moves?
- **Q16.** Editor primary env: `local` vs `dev`. Warnings against prod by default?
- **Q17.** Migration ordering default: linear timestamps vs dependency-based; out-of-order policy.
- **Q18.** Where does ownership metadata live: DB comments, config, or the State Service?

### Technical
- **Q19.** PGlite: which PG version and extensions now; Describe access path; per-fingerprint data-dir caching.
- **Q20.** Completion on incomplete SQL: tree-sitter vs custom recovery on libpg_query.
- **Q21.** Serverless snapshot distribution: OCI registry vs S3 vs GitHub artifacts as the first-class option.
- **Q22.** TS native compiler (Go) plugin story: current status and implications.

## 3. Parking lot (ideas worth keeping, not yet placed)

- Hover info showing "who else uses this column" from the registry.
- "Did you mean" suggestions from edit distance over the snapshot.
- Check-constraint `IN` lists → TS unions.
- `WHERE x IS NOT NULL` flow refinement for nullability.
- Approximate table sizes as lint hints (non-fingerprinted).
- Schema change audit trail as a compliance feature.
- Bot that removes fulfilled claim files automatically.
- Per-fingerprint prebuilt PGlite data directories for instant warm starts.
- OCI artifacts as the snapshot distribution format.
- DB-comment annotations (`@dst ...`) for jsonb shapes, ownership, ephemeral markers.
