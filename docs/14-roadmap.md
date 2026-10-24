# 14 — Roadmap & Validation Spikes

Principle: **validate the riskiest assumptions first, and ship single-player value before org value.** Every phase must be useful on its own.

## Phase 0: Validation spikes (time-boxed)

Each spike has a question, a method, a success threshold and an output (a short write-up in `docs/spikes/`).

| ID | Question | Method | Pass threshold | Assumptions |
|----|----------|--------|----------------|-------------|
| **S0** | Does an existing product already do env-aware SQL schema registry + consumer impact + deploy gating? | Hands-on teardown of Atlas (OSS + Cloud), Bytebase, Prisma's current offerings, Supabase tooling, Postgres Language Server, SafeQL, sqlx; plus a GitHub/web search for "schema registry postgres consumers" | A clear gap remains in ≥2 of: multi-env typing, consumer registry, claims/gates | A8, K1 |
| **S1** | How accurate can nullability inference be? | Collect 500–1000 real queries (OSS apps: e.g. Postgres-backed projects on GitHub) + schemas; implement rules from [07 §3](07-type-safety.md#3-output-safety-nullability); run each query on seeded/fuzzed data; compare | ≥98% column agreement, 0 false non-null on the corpus after rule iteration | A3, K2 |
| **S2** | Can we snapshot → rebuild → describe faithfully, fast, in PGlite? | Implement the extractor + rebuilder for PG 16/17; compare Describe output on the source DB vs PGlite for the S1 corpus; measure latency, cold start, memory | ≥95% identical describes (rest explained by extensions/version); Describe p95 < 100 ms; cold start < 3 s for 200 tables | A1, A2, A10 |
| **S3** | Which TS type-delivery option do developers accept? | Prototype options A (inline + autofix), B′ (literal-keyed registry), C (`.sql` files) on one small app; 5–10 developer sessions | One option is rated ≥ "as good as Kysely/Drizzle" by most participants | A6, K3 |
| **S4** | Can extension stubbing replace real extensions for Describe? | Stub pgvector + PostGIS from catalog signatures in PGlite (where unavailable) or vanilla PG; compare describes | Matching describes for operator/function-heavy queries | A2 |
| **S5** | Observer viability on managed providers | On RDS, Cloud SQL, Neon, Supabase, Azure: catalog-only role extraction completeness; event trigger installability; LISTEN via the direct endpoint | Complete snapshots on all; event triggers on ≥3 or polling deemed adequate | A4, A5 |
| **S6** | Can `pg_stat_statements` reveal unregistered consumers? | Match normalized statements to footprints in a test org/DB | Useful attribution on a realistic workload | A9 |
| **S7** | Would platform teams deploy this? | 5+ interviews with platform/DBA engineers: their current "can I drop this column?" process, willingness to run an Observer | ≥3 say they'd pilot | A7, K4 |

**Gate to Phase 1:** S0, S1, S2, S3 pass (or a pivot is chosen per [13 §4](13-vetting.md#4-kill--pivot-criteria)).

## Phase 1: Single-player typed SQL (local only)

**Goal:** the best zero-setup typed Postgres experience in TypeScript.

- Analyzer core (libpg_query + PGlite Describe + nullability + mapping + footprint).
- `local` env from a migrations folder (built-in runner, or read Prisma/Drizzle/Atlas migration dirs).
- Frontends: TS tagged templates (inline-annotation delivery) + `.sql` files.
- ESLint plugin (diagnostics + autofix) + VS Code extension (highlighting, completions via a standalone LSP talking to the daemon).
- Runtime client wrapping `postgres` (porsager) with matching type parsers.
- `dst init`, `dst check`, `dst gen`.
- **Exit criteria:** used on 2–3 real projects; nullability bug rate acceptable; cold start and latency budgets met.

## Phase 2: Snapshots, environments, lockfile (no server)

**Goal:** check code against real environments without connecting dev machines to them.

- Snapshot format v1 + canonicalization + fingerprints + signing.
- `dst snapshot --url` (run in CI against staging/prod) → publish to an artifact store (S3/OCI/GitHub artifacts).
- `dst pull`, lockfile, multi-env checks in CLI/CI, env-aware editor warnings, offline mode.
- Feeding other tools from snapshots: kysely-codegen-compatible types, Prisma `schema.prisma`, sqlc schema 💡 (cheap polyglot value).
- **Exit criteria:** a team runs CI checks against prod snapshots; developers have no prod DB access.

## Phase 3: State Service (sidecar)

**Goal:** live state, push, history, drift.

- Observer (polling + event triggers), State Store (lineage), Broadcaster (SSE), API, auth (OIDC device flow, CI workload identity), per-DB ACLs, redaction.
- Drift detection + attribution (wrapping migration tools via `dst apply --via ...` for attribution).
- **Exit criteria:** deployed in one org; drift alerts useful; snapshots served to editors.

## Phase 4: Consumer registry & impact analysis

**Goal:** "who breaks if I change this?". The flagship org feature.

- Footprint registration from CI; deployment registration from CD.
- `dst impact` on migration PRs (required check), column/table usage views in a minimal web UI.
- `pg_stat_statements` passive discovery.
- **Exit criteria:** a platform team uses impact checks to safely drop or rename something they were previously afraid to touch.

## Phase 5: Claims, deploy gates, startup checks

**Goal:** enforce schema-before-code ordering across repos.

- Claim formats + ledger, auto-derived in-PR claims, cross-repo claims, shape/manual claims, expiry.
- `dst gate` for CD; runtime startup check; notifications on fulfilment; blocked-deploy re-trigger.
- Rolling-deploy compatibility checks (old code vs new state, both-states checks).
- **Exit criteria:** cross-team schema changes ship with zero "migration not applied yet" incidents in a pilot.

## Phase 6: Breadth

- Fragments/dynamic query composition (if not already done in Phase 1).
- Table families, block-local DDL overlays.
- Python adapter; Go via sqlc integration; Rust via sqlx offline data.
- MySQL and SQL Server adapters.
- SDK/ORM generation from state (R14).
- pgroll/Reshape integration for multi-version schemas.

## Success metrics

| Metric | Phase |
|--------|-------|
| Time from `npm i` to first typed query | 1 (target < 5 min, no Docker) |
| Nullability false-non-null rate | 1+ (target ≈ 0 on corpus; tracked bug reports) |
| Editor latency p95 | 1+ (see [08 §2](08-developer-experience.md#2-performance-budgets)) |
| % of queries analyzable without escape hatch | 1+ (target > 90% in pilot codebases) |
| Prod schema incidents caused by code/schema mismatch | 4–5 (pilot org; target trending to 0) |
| Number of registered consumers per DB / discovered unregistered | 4 |
| Median age of pending claims | 5 |

## Immediate next actions

1. Write `docs/spikes/S0-competitive-teardown.md` (start with Atlas Cloud and Bytebase).
2. Assemble the query corpus for S1/S2 (pick 5–10 OSS Postgres apps).
3. Hello-world: PGlite + Describe through its API, measure latency (S2 seed).
4. Draft interview script for S3/S7.
5. Decide naming (the repo name says "sync", which suggests data sync; see [15](15-open-questions-and-decisions.md)).
