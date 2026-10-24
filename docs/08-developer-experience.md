# 08 — Developer Experience

Requirements covered: R2 (editor features), R3 (zero infra), R7 (no prod connections from dev machines), R8 (offline), R12 (broadcast).

## 1. Editor features

### 1.1 Syntax highlighting

- **Mechanism:** a VS Code extension contributes a TextMate grammar **injection** into `source.ts`/`source.tsx`/`source.js` that matches tagged templates with known tags (`sql`, `db.sql`, configurable) and `sql("...")` calls, and embeds `source.sql` inside them.
- Interpolations `${...}` inside the SQL must switch back to TS highlighting. A known fiddly part of injection grammars, but solved in existing extensions (several marketplace extensions highlight SQL in template literals; ⚠ survey them and reuse or contribute rather than starting from scratch).
- **Semantic highlighting** (via the language server's semantic tokens) can colour SQL identifiers by kind: table, column, function, CTE name, unknown (red). Grammars can't do this.
- Other editors: Neovim/Helix via tree-sitter injections (`injections.scm`) for `sql` tagged templates; JetBrains has built-in "language injection" (configurable to treat `sql` tags as SQL).

### 1.2 Red squiggles (diagnostics)

Sources of diagnostics, all mapped back from SQL offsets to TS source spans (via the frontend's source map, accounting for `${}` substitutions):

| Source | Example |
|--------|---------|
| Parser (libpg_query) | `syntax error at or near "FORM"` |
| Describe on the local engine | `column "eml" does not exist` (with a hint "did you mean email?" 💡, from edit distance over the snapshot) |
| Param type check | `Argument of type 'string' is not assignable to parameter $1 (int4 → number)` |
| Annotation mismatch | `Declared row type differs from inferred: phone is string \| null` (autofix) |
| Environment warnings | `users.plan exists in local but not in prod (needs claim)` (warning) |
| Lint rules | `SELECT * depends on every column of users; list the columns you need` (warning) · `${status} may be null, and "status = NULL" is never true; use IS NOT DISTINCT FROM` (warning) · `UPDATE without a WHERE clause changes every row in users` (error) |

### 1.3 Autocomplete

The language server knows the cursor position *within the SQL* and has the snapshot:

- After `FROM` / `JOIN` → tables, views, CTE names in scope, table-returning functions.
- After `alias.` → columns of the aliased relation.
- In the `SELECT` list → columns of relations in scope (needs a partial parse of the `FROM` clause even when the statement is incomplete; libpg_query fails on incomplete SQL, so completion needs a **tolerant parser**, e.g. tree-sitter-sql or a hand-rolled recovery layer. ⚠ Design spike).
- Function names with signatures, enum labels inside string literals compared to enum columns 💡, `INSERT INTO t (` → column list.
- Hover: column type, nullability, comment, which env it exists in, who else uses it (from the consumer registry) 💡.
- Go-to-definition: jump from a table name to the migration that created it / last altered it (from lineage), or to its snapshot definition.

### 1.4 Code actions

- Fix annotation to match the inferred type.
- Add claim for an unmet requirement (generates a claim file or links the in-PR migration).
- Create migration: "column `plan` doesn't exist → *Generate `ALTER TABLE users ADD COLUMN plan ...`*" (owner/co-owner mode only).
- Extract the query to a `.sql` file.

## 2. Performance budgets

| Operation | Target | Notes |
|-----------|--------|-------|
| Completion from cached snapshot | < 30 ms | Pure in-memory catalog lookup |
| Diagnostics for an edited query (cache hit) | < 10 ms | |
| Diagnostics for an edited query (cache miss, Describe on PGlite) | < 100 ms | ⚠ Measure PGlite Describe latency |
| Cold start: daemon + PGlite + rebuild from a 200-table snapshot | < 3 s | ⚠ Measure; cache a pre-built PGlite data dir per fingerprint to make warm starts near-instant 💡 |
| Env switch (cached engine) | < 200 ms | Engine pool keyed by fingerprint |
| Re-analysis after a schema change | proportional to *affected* queries | Object-level cache keys ([05 §3.5](05-state-model-and-versioning.md#35-object-level-fingerprints-and-partial-states)) |

## 3. The local engine: "just under the hood"

The user's requirement: *"i would hate consumers to have to worry about that, run in docker, etc. i'd love it if was just under the hood."*

- The **`dst` daemon** starts automatically (spawned by the VS Code extension, the ESLint plugin, or the CLI) and owns the engines. One per workspace; a lockfile/socket in `.dst/` prevents duplicates.
- Engines are **PGlite instances** built from snapshots (or from local migrations for the `local` env). Nobody configures them.
- If the snapshot needs something PGlite can't provide (an unavailable extension, a different PG major), the rebuilder **stubs** it from the catalog signatures (types and functions with the right signatures and dummy bodies), which is enough for Describe. It only escalates to a real Postgres (opt-in: a managed binary download or Docker) when stubbing isn't enough. That escalation must be **explicit and visible**, never silent.
- Fidelity disclosure: `dst status` says "local engine: PGlite (PG 17.x) — target prod: PG 16.4. Version-specific syntax differences won't be detected locally; CI checks against PG 16." ⚠ Verify which PG major the current PGlite tracks.

## 4. Offline and reconnect

| Situation | Behaviour |
|-----------|-----------|
| Online, State Service reachable | Live push of env heads; daemon prefetches snapshots; warnings when the lockfile is behind |
| Offline | Everything works from the cache + lockfile. Status bar: "offline — prod state as of 2026-10-23 17:40 (21h old)" |
| Reconnect | Daemon fetches heads, diffs cached vs live states, re-analyzes affected queries. New conflicts appear as diagnostics, plus a summary notification: "prod changed while you were offline: `users.legacy_ref` dropped — affects 2 queries in this workspace" |
| Never connected (new clone, no access) | Uses committed lockfile + committed snapshots if present; otherwise the `local` env from migrations; otherwise prompts for `dst login` |

The "offline claim" idea from the conversation reduces to: whatever you wrote offline is checked on reconnect, and any unmet requirement gets the normal claim treatment. No special offline-claim concept needed.

## 5. Configuration

Zero-config for the simplest case: if a `migrations/` folder exists, `dst` uses it as the `local` env with no config file.

Full config sketch:

```ts
// dst.config.ts
import { defineConfig } from "@dst/config";

export default defineConfig({
  databases: {
    main: {
      dialect: "postgres",
      environments: {
        local:   { source: { migrations: "./db/migrations", tool: "builtin" } },
        staging: { source: { service: "https://dst.internal.example", env: "main/staging" } },
        prod:    { source: { service: "https://dst.internal.example", env: "main/prod" } },
      },
      ownership: { mode: "co-owner", owns: ["schema:billing"] },   // see 09
    },
  },
  frontends: {
    ts: { tags: ["sql"], clients: ["db", "tx"], typeDelivery: "inline-annotation" },
    sqlFiles: { glob: "src/**/*.sql", out: "colocated" },
  },
  typeMapping: { int8: "bigint", date: "string", jsonb: "unknown" },
  editor: { primary: "local", alsoWarnAgainst: ["prod"] },
  ci: { targets: ["staging", "prod"], requireClaimsFor: ["prod"] },
  runtime: { startupCheck: "fail" },
});
```

## 6. CLI walkthroughs

```console
$ dst init
✔ Detected: TypeScript, postgres.js, ./migrations (12 files)
✔ Created dst.config.ts (local env from ./migrations)
✔ Built local engine (PGlite) at sha256:e5d2… in 0.8s
→ Install the VS Code extension for highlighting & completions: code --install-extension dst.dst

$ dst check
Analyzed 143 queries in 61 files against [local] (sha256:e5d2…)
✖ src/billing/invoices.ts:42:18  column "amount_cents" does not exist (did you mean "amount"?)
⚠ src/users/repo.ts:17:9  users.plan: present in local, missing in prod — no claim
1 error, 1 warning

$ dst status
database main
  local    sha256:e5d2…  (from ./migrations, 14 files)       +1 vs staging
  staging  sha256:7b10…  observed 3m ago                     +1 vs prod (teamB/0043)
  prod     sha256:9f3a…  observed 40s ago   lockfile: ✔ current
claims: 1 pending (plan-column → prod, accepted, unfulfilled, 2d old)

$ dst impact db/migrations/0044_drop_legacy_ref.sql --env prod
✖ Breaks 2 deployed consumers:
  reporting-service@v3.2.1 (prod)   src/reports/orders.sql:12   SELECT legacy_ref ...
  billing-worker@a91f0c   (prod)    jobs/reconcile.ts:88        WHERE legacy_ref = $1
```

## 7. Persona walkthroughs

1. **Solo dev, app owns the DB:** `npm i @dst/client`, writes migrations and queries, and gets types and squiggles. Never sees "state service" or "claims" unless they add a prod env. Single-player mode must stand on its own.
2. **Service team on a shared DB (observer):** no migrations folder. `dst login` → pulls `prod`/`staging` snapshots → types match reality. Gets notified when the platform team's changes affect them.
3. **Platform/DBA team:** runs the State Service; uses `dst impact` on every migration PR (as a required check); sees a dashboard of consumers per table/column; can finally drop columns safely.
4. **Polyglot consumer (Python analytics):** uses `dst` snapshots to drive its own tooling (stubs or sqlc-like), and registers its footprint so the platform team sees it.
5. **Airplane dev:** works offline from the lockfile; reconnect shows anything that changed.
