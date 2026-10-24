# database-sync-tool

> **Status:** concept / pre-vetting. No code yet. These docs are where we work out the idea before writing any code.
>
> **Working name:** `dst` (database-sync-tool). Placeholder. The source conversation used `dreamdb`; both mean the same thing.

## The idea in one paragraph

Hooking SQL databases into applications is painful in every language and with every tool. That isn't mainly a syntax problem; **it's a distributed state-synchronization problem.** A database is a live state machine whose schema changes over time, sometimes on the fly, and often from somewhere other than the application using it (another repo, another team, another tool, a DBA at a console). Application code is static and is checked at build time. `dst` proposes a **database state service** that runs alongside a database and publishes its *current, versioned* structural state (tables, columns, types, functions, views, extensions, server version, ...). Tooling built on that published state gives any author, whatever their syntax (raw `.sql`, inline SQL template tags, a query builder, a schema DSL, an ORM-like generated SDK), editor feedback (highlighting, completion, red squiggles) and **build-time guarantees** that their code matches the database it will run against. When code needs state that doesn't exist yet, the author has to supply an **authority**: a *claim* that some change (in this PR, another repo, or applied by hand) will land before the code runs. The pipeline verifies that claim before deploying.

## Why this repo exists

The goal is to **fully vet** the concept (find out whether it's sound, novel, buildable and worth building), **understand** it in depth, and **eventually build it**. These documents are the working record of that process.

## Document map

Read in order the first time. Later, each file stands on its own.

| # | Document | What it covers |
|---|----------|----------------|
| 00 | [Origin & conversation summary](docs/00-origin.md) | Where the idea came from, how it evolved step by step, and the source PDF |
| 01 | [Problem statement](docs/01-problem-statement.md) | The pains, why existing tools hit them, and the reframing as state management |
| 02 | [Concept overview](docs/02-concept-overview.md) | The proposed system at a high level, its principles, and what it is and isn't |
| 03 | [Glossary](docs/03-glossary.md) | Terms used across all documents |
| 04 | [Architecture](docs/04-architecture.md) | Components, data flow, deployment shapes, protocols |
| 05 | [Schema state model & versioning](docs/05-state-model-and-versioning.md) | What "database state" is, how it's snapshotted, fingerprinted, versioned, diffed and distributed |
| 06 | [Claims, footprints & build gates](docs/06-claims-and-build-gates.md) | The authority/claim mechanism, made formal, plus rolling deploys and enforcement points |
| 07 | [Type safety in depth](docs/07-type-safety.md) | Input/output typing, nullability, type mapping, how types reach TypeScript, dynamic queries |
| 08 | [Developer experience](docs/08-developer-experience.md) | Editor features, local/offline development, config, CLI, walkthroughs |
| 09 | [Schema authoring, ownership & migrations](docs/09-schema-authoring-and-migrations.md) | Ownership modes, authoring styles, migration ledger, linting, schema-from-code, runtime DDL |
| 10 | [Database features & engine support](docs/10-database-features.md) | Postgres primitives that make this possible, managed-provider limits, other engines |
| 11 | [Security](docs/11-security.md) | Threat model, network/identity/privilege design, snapshot confidentiality |
| 12 | [Prior art & building blocks](docs/12-prior-art.md) | The current landscape, what to reuse, where the gap really is |
| 13 | [Vetting: corrections, risks & hard problems](docs/13-vetting.md) | Critical review of the original conversation's claims, risk register, kill criteria |
| 14 | [Roadmap & validation spikes](docs/14-roadmap.md) | Experiments to run first, then phased MVP plan and success metrics |
| 15 | [Open questions & decision log](docs/15-open-questions-and-decisions.md) | Unresolved questions and decisions made (or pending) |

## Conventions used in these docs

- **⚠ Verify**: a factual claim (often about a third-party tool or a database behavior) that we believe but haven't confirmed first-hand. Each one should become a spike or a quick check before it's relied on.
- **❌ Correction**: a place where the original conversation (an AI chat) said something that is wrong or overstated. These are collected in [13-vetting.md](docs/13-vetting.md) and noted inline where relevant.
- **💡 Idea**: a design idea that came up while writing these docs and wasn't in the original conversation.
- Code samples are **illustrative API sketches**, not settled interfaces.
- PostgreSQL is the primary target throughout. Other engines are discussed in [10](docs/10-database-features.md).

## Source material

- [`Alternatives to Modern Node ORMs.pdf`](Alternatives%20to%20Modern%20Node%20ORMs.pdf): export of the original brainstorming conversation (Gemini, 2026-10-24). Summarized in [00-origin.md](docs/00-origin.md).
