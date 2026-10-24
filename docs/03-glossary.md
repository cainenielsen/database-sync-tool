# 03 — Glossary

Terms are listed in rough dependency order (later terms build on earlier ones), not alphabetically.

| Term | Definition |
|------|------------|
| **Database state** (or **schema state**) | The *structural* (not data) state of a database at a point in time: schemas, tables, columns, types, constraints, indexes, views, functions, triggers, extensions, RLS policies, relevant server settings and server version. See [05](05-state-model-and-versioning.md) §1. |
| **Snapshot** | A canonical, serialized capture of database state, plus metadata (source environment, capture time, server version, signer). |
| **State fingerprint** | Content hash of a snapshot's canonical form. Identifies a state regardless of how it was reached. |
| **Object fingerprint** | Content hash of a single object (e.g. one table) within a snapshot. Used for fine-grained diffs and footprint checks. |
| **Environment** | A named database instance or class of instances (`local`, `dev`, `staging`, `prod`, `pr-1234`). Points at its current state fingerprint, much like a git ref. |
| **Lineage** | The history of states an environment has moved through, with the transitions between them. A DAG across environments. |
| **Transition** / **change set** | A unit of change that moves one state to another: a migration file, a declarative plan, a manual hotfix (observed after the fact). Identified by content hash. |
| **State Service** | The component that observes databases, records snapshots and lineage, holds the consumer registry and answers check queries. Can run as a sidecar, a standalone service, or a "serverless" CI job that writes snapshots to an artifact store. |
| **Observer role** | A database role with only the privileges needed to read the catalog. It can't read table data or change anything. |
| **Frontend** / **syntax adapter** | A component that pulls SQL (plus parameter info) out of some authoring syntax: SQL template tags in TS, `.sql` files, Kysely queries, a DSL, etc. |
| **Analyzer** | The core that takes `(SQL, dialect, state)` and returns parameter types, result types, nullability, a footprint and diagnostics. |
| **Local engine** | A database engine running locally (PGlite in-process by default) rebuilt from a snapshot, used by the analyzer to *describe* queries without touching real databases. |
| **Footprint** | The set of **requirements** a piece of code places on database state: objects it references and the shape properties it relies on. Computed per query, aggregated per module/service. |
| **Requirement** | One atomic footprint entry, e.g. `column(public.users.email): type=text, nullable=false, exists`. |
| **Consumer** | A deployable unit (service, job, report) that registers its footprint with the State Service for a given environment. |
| **Consumer registry** | The State Service's record of which consumers, at which versions, are deployed to which environments, and their footprints. Lets migrations be checked against everything that would break. |
| **Claim** | A declared authority for an unmet requirement: "change set X (or some change producing shape Y) will be applied to environment E before this code runs there." See [06](06-claims-and-build-gates.md). |
| **Build gate** | CI step: code's footprint must be satisfied by (target state + verified-in-PR claims), and any remaining claims must be well-formed. |
| **Deploy gate** | Pre-rollout step: the footprint must be satisfied by the environment's *actual current* state. This is where external and manual claims are finally checked. |
| **Startup check** | Runtime step: on boot the app compares its embedded footprint/fingerprint with the live DB and refuses to start, warns or degrades on mismatch. |
| **Drift** | A difference between an environment's observed state and its expected state (e.g. an untracked manual change). |
| **Lockfile** | A file committed to the repo (`.dst/state.lock.json`) that pins, per environment, the snapshot fingerprint the code was checked against. Lets builds be reproducible and offline. |
| **Expand / contract** | The standard zero-downtime migration pattern: *expand* the schema compatibly (add), migrate code, then *contract* (remove old). `dst` checks each step against deployed consumers. |
| **Escape hatch** | An API for queries that can't be statically analyzed (dynamic identifiers, runtime DDL). The result type comes from the author, ideally with runtime validation. |
| **Table family** | 💡 A declared pattern of runtime-created tables that share a shape (e.g. `tenant_*_events` shaped like template table `tenant_template_events`). Lets dynamic table names stay typed. |
| **Ownership mode** | How a project relates to a database's state: **owner** (it applies changes), **co-owner** (owns some objects), **observer** (never changes structure). Can be per object. |
