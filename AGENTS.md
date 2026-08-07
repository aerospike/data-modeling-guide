# Agent instructions

This repository is a data modeling resource for Aerospike, for humans and AI coding agents alike. If you are an agent helping someone design or review an Aerospike data model, read this file first, then follow the workflow it points you to.

## Scope: modeling, not API reference

This guide exists to produce good **data models** — record granularity, key design, bin structure, relationship patterns, and the indexing and server-side filtering that follow from them.

It is **not** an API reference, and you should not treat it as one. API surfaces are summarized here only to the depth that modeling decisions require:

- **What the server can do in place** — whether an update is a CDT operation or a read-modify-write round trip changes the model.
- **What an operation costs** — complexity and write amplification decide consolidation vs. split.
- **How data is ordered and compared** — list vs. map, and which map subtype, follow from this.
- **Which server version gates a capability** — a pattern you cannot deploy is not a pattern.

For anything beyond that — exact signatures, per-language client syntax, parameter semantics, the current full operation set — **use the [Aerospike documentation](https://aerospike.com/docs/) or an Aerospike documentation search tool.** Do not infer an API contract from this repo's summaries, and do not answer a pure API question out of it.

Where this guide and the official documentation disagree, **the official documentation wins.** Flag the drift so this guide can be corrected.

## What you are producing

Your output is two documents, not a chat answer and not a schema pasted into a reply. Know which one you are writing at any moment.

**Schema guide** — the complete design document, and the artifact every gate and checkpoint in this workflow operates on. It contains all of [new-app-modeling-checklist.md](new-app-modeling-checklist.md) § 0: entity and relationship map, access pattern matrix, key schema, bin schema, one JSON example record per set, relationship and consolidation decisions, completed pattern-decision forms and sizing worksheets, index rationale, growth and hot-key plan, validation plan. It must also carry the **reasoning** — an assumptions log naming every judgment call you made, the alternatives you rejected, and the evidence that would reopen each decision. A schema without that reasoning is not a schema guide; it is a guess someone will have to re-derive.

**Schema summary** — the condensed operational reference **derived from** the schema guide: one table per set with key format, bins, types, and a one-line purpose; the index list; growth and overflow triggers. No rationale, no alternatives — just the contract a developer keeps open while implementing.

Rules for both:

- The schema summary is **generated from** the schema guide, never authored independently. If they disagree, the schema guide is authoritative and the summary is regenerated.
- Do not produce either before Step 0's Clarification Document is resolved (see Hard rules below).
- Build the schema guide **incrementally, one entity group at a time**, passing each group's walkthrough and stakeholder checkpoint before starting the next. Produce the schema summary only once every group is done.
- Write both to files. These are review artifacts with a life beyond the session.

## Hard rules

1. **Do not propose a data model before completing Step 0.** The workflow's first deliverable is a written Clarification Document, not a schema. A model produced without one is not compliant with this guide.
2. **The checklist is an interactive process, not a document template.** [new-app-modeling-checklist.md](new-app-modeling-checklist.md) defines mandatory stop points — group-specific clarification, blocker gates, and stakeholder review checkpoints. Extracting its headings into a single-pass spec produces a model that looks compliant but was never validated at any stage.
3. **Ask requirements-gap questions, not mechanism-preference questions.** Ask "what is the p95 fan-out?", never "which pattern do you prefer?". If deterministic guidance in this repo already resolves a choice, apply it instead of asking.
4. **Default to the baseline shape.** Do not introduce a new set, split record, extra index, or materialized view unless a requirement cannot be met with the baseline, or measured data shows the baseline misses an SLO. Otherwise mark the decision `BLOCKED_MISSING_INPUT` or `DEFERRED_OPTIMIZATION`.
5. **Stop and ask when entity ownership, lifecycle, cardinality, or an access path is unclear.** Do not fill gaps with assumptions and continue.

## The core mental shift

Aerospike is not a relational database, and it is not a document database.

- Records are the unit of I/O. **There are no joins** — the multi-record tool is the **batch read**, which scatters and gathers efficiently across nodes.
- Every record costs **64 bytes of primary index metadata**, usually in RAM. Many tiny records waste memory.
- The target record size is **1–128 KiB** — the Goldilocks band. Throughout this repo, **"a few KiB" means that same 1–128 KiB range**, not only very small records.
- **Access patterns drive the model**, not entity normalization.

Consolidate enough to avoid tiny records, but not so much that one record becomes a monolith. Efficient batch reads make many medium records cheap to fetch together, so spreading data across well-sized records beats packing everything into one.

If your instinct is a table per entity and a row per sub-entity, or one giant embedded document, you will produce a bad Aerospike model.

## Where to start

| Situation | Read |
| --- | --- |
| **New application, from scratch** | [new-app-modeling-checklist.md](new-app-modeling-checklist.md) — required first read — then the full workflow in [README.md](README.md#using-this-research-with-an-llm) |
| Reviewing or extending an existing model | [README.md](README.md#using-this-research-with-an-llm) workflow, entering at the relevant step |
| Deciding between modeling options (list vs. map, consolidate vs. split, index or not) | The routing table below |
| A pure API question — signature, syntax, parameter meaning | Not this repo. [Aerospike documentation](https://aerospike.com/docs/) or a docs search tool |

The complete Step 0–8 workflow, the question-quality rules, and the full list of common modeling mistakes live in [README.md](README.md#using-this-research-with-an-llm). Follow it there rather than improvising an order.

## Failure modes

The seven portable ways Aerospike models go wrong. Check these while drafting, not after. Each links to a detection test and the corrective pattern in [modeling-failure-modes.md](modeling-failure-modes.md); an eighth entry there covers misuse of the checklist itself.

1. **Record granularity comes from cardinality and who drives the read** — never from the entity list. One set per domain noun means the model came from an ER diagram.
2. **The most frequent reads must be key lookups or bounded batch reads.** If more than one or two access patterns resolve via secondary-index query, fix the keys, not the indexes.
3. **Single-element mutations happen server-side, in place.** Any read-modify-write of a whole bin should have been a CDT operation.
4. **A bin is a container, not a field.** Bin counts that scale with data rather than schema belong in one CDT bin. Bin names cap at 15 characters.
5. **Duplicate data deliberately** when two access patterns need it in two shapes. A second round trip purely to assemble a response is a normalization you should have collapsed.
6. **Every collection bin needs a growth ceiling and a decided behavior at it.** If element count is driven by user behavior rather than a design decision, it is unbounded — and `max-record-size` is configurable with a default well below its maximum, so never size against the ceiling.
7. **Small independent entities still need an explicit sizing decision.** 64 bytes of index against a few hundred bytes of payload is double-digit overhead; consolidating all of them into one record is the opposite error.

## Routing table

| Task | Start with | Then consult |
| --- | --- | --- |
| New data model from scratch | [new-app-modeling-checklist.md](new-app-modeling-checklist.md) | [concepts-and-patterns.md](concepts-and-patterns.md) |
| 1:1, 1:N, or N:M relationship | [concepts-and-patterns.md](concepts-and-patterns.md) § Relationships at a glance | [one-to-many-relationships.md](one-to-many-relationships.md) |
| A list that can grow very large | [follow-relationship-scale.md](follow-relationship-scale.md) | [one-to-many-relationships.md](one-to-many-relationships.md) |
| List vs map; ordering; persist-index | [cdt-api.md](cdt-api.md) | [concepts-and-patterns.md](concepts-and-patterns.md) § value-based access |
| Server-side filtering or computed bins | [expressions.md](expressions.md) | [concepts-and-patterns.md](concepts-and-patterns.md) applied examples |
| Nested CDT querying or list-of-structs | [path-expressions.md](path-expressions.md) (**DB 8.1.2+**) | [cdt-api.md](cdt-api.md) nested context |
| Matching a workload to a known shape | [workload-archetypes.md](workload-archetypes.md) | [concepts-and-patterns.md](concepts-and-patterns.md) |
| Reviewing a drafted model for defects | [modeling-failure-modes.md](modeling-failure-modes.md) (use the Detect lines as tests) | the file each entry points to |
| Identifier format | [id-selection-guidance.md](id-selection-guidance.md) | — |
| Timestamp field naming | [timestamp-bin-naming-guidance.md](timestamp-bin-naming-guidance.md) | — |
| Sizing, amplification, benchmark shape | [workload-archetypes.md](workload-archetypes.md) | [concepts-and-patterns.md](concepts-and-patterns.md) § Capacity planning |

## Version gates

Several patterns depend on server version. Verify the target version, client support, and operational acceptance before recommending them for production.

- **Path expressions** (`selectByPath` / `modifyByPath`, `mapKeysIn`, `andFilter`): Aerospike Database **8.1.2+**.
- **Expression indexes**: **8.1+**.
- Check [path-expressions.md](path-expressions.md) and the version gate table in the checklist before committing to either.

## Relationship to `aerospike/agent-skills`

Aerospike's agent guidance is split by **when in the lifecycle**, not by subject:

- **`aerospike/agent-skills`** — implementation-time. A model already exists; the `aerospike-development` skill covers client code, CDT and expression usage, policies, and ships `model-*` reference files for keys, bins, record size, and hot keys. That overlap is deliberate.
- **This repo** — design-time. No model exists yet; you are deriving one from requirements and producing a schema guide and schema summary.

If the task is "write or review Aerospike client code against an existing model," this repo is the wrong tool — `aerospike-development` is closer. If it is "there are requirements and no schema," you are in the right place.

A gateway skill in `agent-skills` scoped to greenfield design is planned, to route that second case here. See [README.md](README.md#where-this-guide-fits-alongside-aerospikeagent-skills) for the full division and the maintenance rule that governs it: content may be duplicated into a skill only if it is **slow-moving** (the 64-byte index cost, the 1–128 KiB band, the failure modes). Version gates, complexity tables, and `max-record-size` values must never be copied out of this repo — they drift, and a stale copy is worse than a pointer.

## Maintaining this repo

- **Before fetching a documentation URL for research,** check [urls-processed.md](urls-processed.md). If it is already listed, ask whether to reprocess before fetching again. Record new URLs there with the date and which file they fed.
- **Validate new information against existing content before adding it.** Do not add material that is already covered. If new material appears to conflict with what is documented, ask for clarification rather than overwriting.
- **Keep customer material anonymized.** Worked examples drawn from customer engagements use generic descriptors ("telco session-data platform", "web portal session store"). Do not reintroduce company names, customer set or namespace names, support case numbers, or links to internal Confluence or Jira.
