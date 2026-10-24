# 01 — Problem Statement

## 1. The surface complaint

Using a SQL database from application code is annoying whatever the language or tool:

- **Raw SQL** is expressive and portable (paste it into `psql` and it runs) but *stringly typed*. A typo in a column name, a wrong parameter type, or a forgotten `NULL` case shows up at runtime, often in production.
- **Query builders** (Kysely, Drizzle, jOOQ, ...) give type safety but make you rewrite SQL as method chains. Simple queries look fine. Window functions, recursive CTEs, `LATERAL` joins, `DISTINCT ON`, `FILTER (WHERE ...)`, vendor operators (`<->`, `@>`, `&&`) and so on turn into helper soup or drop back to an untyped `sql` escape hatch.
- **Schema-driven ORMs** (Prisma, TypeORM, Sequelize, MikroORM, ...) give the best autocomplete and relation handling, but they own a model of the database that is *smaller* than the database. Anything outside that model (extensions, custom types, advanced constraints, functions, triggers, RLS policies) is unsupported, partly supported, or needs raw queries that lose type safety.
- **Codegen tools** (pgTyped, sqlc, kysely-codegen, Prisma generate, ...) add a build step that is easy to forget and produces artifacts that go stale.
- **Schema definition** is split across proprietary DSLs (`schema.prisma`), TS object builders (`pgTable(...)`), or raw migration folders, and each tool assumes *it* is the owner.

## 2. Why every tool hits the same wall

The usual diagnosis is the **object-relational impedance mismatch**: sets and relations vs. objects and control flow. That's real, but it only explains the *syntax* complaints. The deeper issue, which surfaced in the conversation (see [00-origin.md, stage 18](00-origin.md#stage-18-the-reframe-its-a-state-management-problem--the-key-moment)), is this:

> **Type-checking database code means checking static code against a dynamic, externally owned, versioned state that the code doesn't control and may not be able to see.**

Every existing tool quietly assumes one of these:

| Assumption | Tools that make it | Where it breaks |
|------------|--------------------|-----------------|
| *"The app owns the schema"* | Prisma, Drizzle, TypeORM, most ORMs | Shared DBs, DBA-managed schemas, multiple services, legacy DBs |
| *"The schema is in these files in this repo"* | pgTyped, sqlc, migration-file-based tools | Schema managed in another repo, by another tool, or by hand |
| *"There's a reachable DB to check against at build/edit time"* | sqlx (online mode), SafeQL | Offline dev, CI without DB access, security policy against dev → prod-like connections |
| *"The schema is fixed while the code runs"* | All of them | Rolling deploys, migrations running during deploy, temp views, per-tenant tables, runtime DDL |
| *"There's one schema"* | All of them | dev / staging / prod / per-PR preview DBs differ at any given moment |

When an assumption breaks, the tool either gets in the way (false errors) or silently lets errors through (false confidence). Users then fall back to untyped escape hatches, which gets them back to stringly-typed code.

## 3. The reframe: database integration is a state-synchronization problem

Treat it as distributed state:

- The **database schema is state**, held by the database, changed by many writers (migration tools in many repos, ORMs, DBAs at consoles, the app itself at runtime).
- **Application code is a consumer** of that state, compiled at time *t₀* and running at time *t₁* against whatever state exists then. Possibly several versions of the code at once, against several environments.
- **Staleness is unavoidable.** Any copy of the state (generated types, a lockfile, an editor's cache) is stale as soon as someone changes the DB. The question is not "how do we eliminate staleness" but "how do we **bound, detect and resolve** it, and how do we stop stale assumptions from reaching production?"
- **Ordering is the hard part.** Code that needs a new column is only safe once the column exists in the environment it runs in. The relationship between "schema change applied" and "code deployed" is an *ordering constraint* across systems that today is carried in people's heads, in PR descriptions and in deploy runbooks.

So the problem is how to give every author, in any repo, language or syntax, **an accurate, versioned, timely view of the database state they'll run against**, plus a **mechanism to declare and verify ordering dependencies** between state changes and the code that depends on them.

## 4. Who has this problem

| Persona | Pain today |
|---------|------------|
| **Solo / small-team app developer** | Wants raw-SQL power with ORM-level autocomplete, without Docker or codegen chores. *Single-player value matters; see [14-roadmap.md](14-roadmap.md).* |
| **Service team on a shared DB** | Doesn't own the schema; types drift from reality; finds out at runtime |
| **Platform / DBA team** | Can't tell which services use which columns, so every change is scary; no safe way to drop or rename anything |
| **Polyglot org** | TS, Python and Go services hit the same DB with different tools; no shared contract |
| **Data / analytics consumers** | Views and reporting queries break silently on upstream changes |
| **Security / compliance** | Developers want prod-like schemas; giving them DB credentials is unacceptable |

## 5. Requirements distilled from the conversation

The user's own statements, written as requirements (IDs used throughout the docs):

| ID | Requirement | Source |
|----|-------------|--------|
| R1 | Type safety on **inputs** (parameters/variables) and **outputs** (columns, nullability) | Stage 6 |
| R2 | Editor support: syntax highlighting, autocomplete, red squiggles for SQL in host-language files | Stage 9 |
| R3 | **Zero-infrastructure dev experience**: no Docker or env setup required just to get types | Stage 10 |
| R4 | Support **advanced DB features** natively (extensions like pgvector, views, functions, triggers) | Stage 8 |
| R5 | Support **dynamic / on-the-fly** structures (temp views, runtime DDL) without losing safety where it's achievable | Stages 7, 18 |
| R6 | Work when the **schema is owned elsewhere** (another repo, team, tool, or by hand) | Stages 12, 14, 18, 19 |
| R7 | **Developer machines must not connect to cloud/production databases** | Stage 13 |
| R8 | Work **offline**, with reconciliation when back online | Stage 21 |
| R9 | **Syntax agnostic**: raw SQL, inline SQL, DSLs, schema-in-language, query builders, fluent APIs | Stage 19 |
| R10 | **Build gating**: code that depends on non-existent state must not build unless an *authority/claim* is declared | Stage 19 |
| R11 | Claims can be satisfied by **any** mechanism: same PR, other repo, other tool, manual | Stage 19 |
| R12 | State **syncs periodically and is broadcast** to authors | Stage 19 |
| R13 | Support **local and cross-environment** databases | Stage 19 |
| R14 | Enable **optional SDK/ORM generation** from state | Stage 19 |
| R15 | Migration **versioning and maintenance** with integrity (checksums, drift detection) when the tool manages migrations | Stage 11 |
| R16 | Access limited to **employees, private networks and in-org CI/CD** | Stage 20 |
| R17 | Tools that manage state should know **whether an operation is valid or already applied** | Stage 19 |

## 6. Non-goals (proposed; see [15](15-open-questions-and-decisions.md))

- **Not a new database or driver.** Use existing drivers (`pg`, `postgres`, `psycopg`, `pgx`, ...).
- **Not a new query language.** SQL (in whatever dialect the database speaks) is the interchange format.
- **Not data synchronization / replication.** Despite the repo name, "sync" means *schema/state metadata* sync, not row-level sync like ElectricSQL, PowerSync or Zero. ⟵ worth confirming; the name could mislead.
- **Not a migration engine competing with Atlas/Flyway.** It *integrates* migration engines and may ship a simple one, but it shouldn't need one.
- **Not an ORM.** ORM-like SDKs can be *generated* by downstream consumers of the state.
