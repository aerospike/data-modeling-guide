# Aerospike data modeling guide

A data modeling resource for humans and AI coding agents. Use this guide to design, review, and revise Aerospike data models — record granularity, key design, bin structure, relationship patterns, and the indexing and server-side filtering that follow from them.

**This is not an API reference.** API surfaces — CDT operations, expressions, path expressions — are summarized here only where they drive modeling decisions: what the server can do in place, what an operation costs, how it orders and compares data, and which server version gates it. Those properties change what a good model looks like, so they belong here. For authoritative and current API details — signatures, per-language client syntax, parameter semantics — go to the [Aerospike documentation](https://aerospike.com/docs/) or an Aerospike documentation search tool. Where this guide and the official docs disagree, the official docs win; correct this guide when you find drift.

**Agents start here:** [AGENTS.md](AGENTS.md) — hard rules, routing table, and version gates. The full Step 0–8 modeling workflow is in [Using this research with an LLM](#using-this-research-with-an-llm) below.

**URL tracking:** Before processing a doc URL for research, check [urls-processed.md](urls-processed.md). If the URL is already listed, ask whether to reprocess before fetching again.

**Data modeling updates:** Validate any new data modeling information against the existing content in this repo. Do not add to the data modeling knowledge unless the information is new. If new material seems to conflict with what is already documented, ask for clarification before adding or changing items.

## What this guide produces

Working through this guide is not an exercise in reading — it produces two documents. Naming them up front matters, because "we did the data modeling" is otherwise unfalsifiable.

| Deliverable        | What it is                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Who reads it                                                                            |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Schema guide**   | The complete design document. Everything in [new-app-modeling-checklist.md](new-app-modeling-checklist.md) § 0: entity and relationship map, access pattern matrix, key schema, bin schema, a JSON example record per set, relationship and consolidation decisions, completed pattern-decision forms and sizing worksheets, index rationale, growth and hot-key plan, validation plan. Critically, it carries the **reasoning** — the assumptions log, the alternatives rejected, and the trigger that would reopen each decision. | Architects and reviewers, during design and whenever the model is revisited.            |
| **Schema summary** | The condensed operational reference derived from the schema guide: one table per set giving key format, bins, types, and a one-line purpose; the index list; the growth and overflow triggers. No rationale, no alternatives — just the contract.                                                                                                                                                                                                                                                                                   | Developers, while implementing. Reviewers, as the thing to diff when the model changes. |

The schema guide is the artifact the checklist's gates and stakeholder checkpoints operate on. The schema summary is derived from it, never authored independently — if the two disagree, the schema guide wins and the summary is regenerated.

Both are outputs of _your_ modeling work; this repo supplies the process and the patterns, not the documents themselves.

## Where this guide fits alongside `aerospike/agent-skills`

Aerospike's agent guidance is split across two internal repos, and the boundary is **when in the lifecycle**, not what subject:

|          | `aerospike/agent-skills`                                                  | this repo                                                       |
| -------- | ------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Moment   | **Implementation-time** — a model exists; write or review code against it | **Design-time** — no model exists; derive one from requirements |
| Audience | Developer                                                                 | Architect                                                       |
| Output   | Code                                                                      | Schema guide and schema summary                                 |
| Shape    | Compact skills, single-turn, code-shaped                                  | Long-form workflow, multi-session, with stakeholder gates       |

The `aerospike-development` skill covers data modeling at implementation depth — it ships `model-*` and `cdt-*` reference files for keys, sets, bins, denormalization, record size, and hot keys. That is deliberate and stays. This guide is not a replacement for it and does not compete with it; it covers the design pass that happens _before_ any of that is relevant.

### The gateway skill

`agent-skills` carries a skill named **`aerospike-data-modeling`**, scoped to greenfield and redesign work, which routes design-time tasks here. It is a _gateway_, not a redirect — a bare "go read this repo" would lose the trigger contest against `aerospike-development` and hand back a pointer with no routing. Instead it carries the durable material inline (the mental shift, the seven portable [failure modes](modeling-failure-modes.md), the clarify-first rule, the schema guide / schema summary contract), then escalates here for the full workflow with a task-to-file routing table. Its counterpart `aerospike-development` carries a handoff clause in its description and scope so the two do not compete.

**This makes filenames in this repo a contract.** The skill's routing table names these files directly:

```
new-app-modeling-checklist.md   concepts-and-patterns.md      one-to-many-relationships.md
follow-relationship-scale.md    cdt-api.md                    expressions.md
path-expressions.md             workload-archetypes.md        modeling-failure-modes.md
id-selection-guidance.md        timestamp-bin-naming-guidance.md
```

Renaming or removing any of them silently breaks the skill's escalation path — nothing in either repo will fail loudly. If a rename is necessary, update the skill's `SKILL.md` routing table and `references/ex-guide-escalation.md` in the same change.

**This repo cannot be linked by URL from the skill.** `agent-skills` runs `skill-validator` in CI, which live-checks every URL. Both repos are internal, so a markdown link to `github.com/aerospike/data-modeling-guide` returns a non-200 and fails the build permanently. The skill therefore refers to this repo by name plus a `gh repo clone` command, and points its `doc:` frontmatter at the public [Aerospike data modeling docs](https://aerospike.com/docs/develop/data-modeling/). Keep it that way.

### The rule that keeps the two in sync — split by rate of change, not by topic

The skill may duplicate the **slow** layer: the 64-byte index cost, no server-side joins, access-patterns-drive-the-model, and the failure modes. Those barely move, and duplicating them keeps the skill useful even when this repo is unreachable. Entries in [modeling-failure-modes.md](modeling-failure-modes.md) are tagged **Portable** or **Guide-only** precisely to mark what is safe to lift — the skill carries the seven Portable ones and omits the Guide-only entry, which is meaningless outside this workflow.

The skill must **never** duplicate the **fast** layer — version gates, complexity tables, API surfaces, `max-record-size` values. These changed twice in the last two months, and a stale copy is worse than a pointer because nothing signals it is wrong. The skill states these by name and directs the reader here to read the current value.

**Resolved — the record-size band.** `aerospike-development/references/model-record-size-hardware-efficiency.md` once gave the sweet spot as roughly 1–10 KiB against this guide's **1–128 KiB**. Both repos are now aligned on **1–128 KiB**, and both state it the same way: **the band is a distribution, not a target.**

That framing settles two opposite misreadings the wording went through. Treating "a few KiB" as a hard single-digit cap is wrong — a 40 KiB record is inside the band and is not an error. But so is reading the band as permission to build 100 KiB objects. Design so the **bulk of records sit in single-digit KiB**, with the upper end reserved for **outliers** and **slowly-changing consolidated structures** — 1:N and N:M relationship lists, where one record per edge would cost more.

**The deciding variable is update rate, not size.** Size only hurts once multiplied by write frequency: the same 100 KiB record is unremarkable rewritten once an hour and a device-saturation problem rewritten thousands of times a second. So where writes are infrequent relative to reads, records near the upper end are a legitimate design rather than a compromise; on a hot write path the same size is a defect. Above roughly **50 KiB**, the record needs an explicit justification naming an update rate shown to be low. See [concepts-and-patterns.md](concepts-and-patterns.md) § Record size limits, which quotes the write-amplification figures from [workload-archetypes.md](workload-archetypes.md) as evidence.

The band is a design target derived from index-to-data ratio, I/O size, and defragmentation cost — **not a measured hard boundary**. If benchmarking on specific hardware and workload produces different figures, replace it in both places with the verified numbers and record the test conditions.

## Files in this guide

| File                                                                 | Purpose                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [new-app-modeling-checklist.md](new-app-modeling-checklist.md)       | **Required first read for new applications.** Pre-implementation checklist: access pattern matrix, key/bin contracts, consolidation vs split criteria, version gates (transactions/expression index/path expressions), validation and failure-mode tests.                                                                                                                                                                                                                                                                                                                              |
| [id-selection-guidance.md](id-selection-guidance.md)                 | Standard decision framework for identifier formats (`UUID/ULID` vs deterministic short hash), including canonicalization/collision contract and per-entity ID checklist.                                                                                                                                                                                                                                                                                                                                                                                                               |
| [timestamp-bin-naming-guidance.md](timestamp-bin-naming-guidance.md) | Contract guidance for timestamp field naming (`*_at_ms`), unit/type consistency, mutability, serialization, and migration rules to prevent cross-client drift.                                                                                                                                                                                                                                                                                                                                                                                                                         |
| [concepts-and-patterns.md](concepts-and-patterns.md)                 | Data model concepts, primary/secondary indexes, distribution; Goldilocks Principle; **"few KiB"** = **1–128 KiB** sweet spot (terminology in Foundational concepts); **Storage compression** (logical vs physical record size, effect on the Goldilocks band); **Relationships at a glance** (1:1, 1:N, N:M patterns + multi-record consistency); **data modeling tips** (denormalization, namespace-wide SI, unique lookup table, sample population, **small independent entity consolidation**); applied patterns from IoT, profiles, relationships, leaderboards, time series, etc. |
| [one-to-many-relationships.md](one-to-many-relationships.md)         | One-to-many patterns: list on parent, consolidate (one record per parent), or child-held reference + secondary index. Choice by cardinality and who drives the read; record size (Goldilocks).                                                                                                                                                                                                                                                                                                                                                                                         |
| [follow-relationship-scale.md](follow-relationship-scale.md)         | Scaling follow relationships: why lists on the user record don’t scale; consolidated alternative (e.g. `user_followers` / `user_following` per user with list of handles).                                                                                                                                                                                                                                                                                                                                                                                                             |
| [cdt-api.md](cdt-api.md)                                             | List and Map APIs: three map subtypes (Unordered / K-ordered / KV-ordered), `persistIndex` for lists and maps, performance tables, nested context, ordering/comparison, secondary index on list elements and map keys/values, path expression context types (8.1.1+/8.1.2+). Covers the CDT behavior that drives modeling choices — cost, ordering, and what can be done in place; see the [official collections docs](https://aerospike.com/docs/develop/data-types/collections/) for the full operation set and syntax.                                                              |
| [expressions.md](expressions.md)                                     | Filter and operation expressions (WHERE clause, computed bins), secondary index expressions (expression indexes, 8.1+), path expressions inside the expression API (`CdtExp.selectByPath`/`modifyByPath`), query by index name. Research summary from Summit talk and official docs; use official [Expressions docs](https://aerospike.com/docs/develop/expressions/) for current API.                                                                                                                                                                                                 |
| [path-expressions.md](path-expressions.md)                           | Path expressions (**Aerospike DB 8.1.2+**): nested CDT query/index, selectByPath/modifyByPath, mapKeysIn/andFilter context types, expression index creation with path expressions. For document-style or nested-CDT models; see the [official path expression docs](https://aerospike.com/docs/develop/expressions/path/) for current syntax.                                                                                                                                                                                                                                          |
| [workload-archetypes.md](workload-archetypes.md)                     | Thirteen representative customer workload archetypes (blob store, counters, multi-bin scalar, document/CDT, segment map, leaderboard, association lists, time-series roll-up, consolidated hierarchy, event timeline, multi-bin entity), each described by record structure (key, bins, types, sizes) and write behavior (which bins change, which operations, bytes moved, write amplification). Use to match a new workload to a known shape and its sizing/amplification profile.                                                                                                   |
| [modeling-failure-modes.md](modeling-failure-modes.md)               | The eight ways Aerospike data models most often go wrong, each with a detection test you can run against a drafted schema and the corrective pattern. Serves both as priming before design and as a review rubric afterward. Entries are tagged Portable or Guide-only for reuse in downstream skills.                                                                                                                                                                                                                                                                                 |
| [urls-processed.md](urls-processed.md)                               | Research provenance: which source URLs have been processed, when, and which guide file each one fed. Check before re-fetching a doc page.                                                                                                                                                                                                                                                                                                                                                                                                                                              |

---

## Using this research with an LLM

This section explains how to use the research in this guide to produce good Aerospike data models with an LLM. It is written for the LLM, but a human can follow the same process.

### The core mental shift

Aerospike is not a relational database. The single most important thing to internalize before modeling:

- **Relational modeling** minimizes storage through normalization. Entities map to tables, rows are small, and joins assemble data at query time.
- **Document modeling** (e.g. MongoDB) favors embedding related data in a single document when it is accessed together, giving atomic single-document writes and one-read access. MongoDB's own guidance is nuanced — they recommend referencing (separate collections) for high-cardinality, unbounded, or independently accessed data, and warn against "bloated documents." But when a document database does split data across collections, the reunification tool is a server-side join (`$lookup`) or multiple round-trips. Aerospike supports document-style modeling — a record can hold nested lists and maps (CDTs). But it has no server-side joins; instead, the tool for multi-record access is **batch reads**, which scatter/gather efficiently across nodes. This architectural difference, combined with a per-namespace record size limit (`max-record-size`, 1 MiB by default) and defragmentation cost, means Aerospike does **not** favor packing everything into one giant record the way a document database might.

- **Aerospike modeling** minimizes latency and memory cost through denormalization and consolidation. The data model is shaped by **access patterns**, not by entity normalization. **Records are semi-structured**: a record is a collection of strongly typed bins, typed per bin per record rather than by a set-level schema, so two records in the same set can have entirely different bins and the server enforces nothing — which makes sparse and heterogeneous shapes cheap, and makes the data model an application-level contract every client must agree on. **Records are the unit of I/O**: record data is stored **contiguously**, so every read fetches the entire record from storage and every write rewrites it in full — there are no in-place updates, and requesting a subset of bins trims network transfer, not device I/O. A record in the tens of KiB spends tens of KiB of I/O on every access, however small the change, which makes record size an I/O budget rather than just a storage number. (All sizes are logical, uncompressed bytes: storage compression — an Enterprise feature — shrinks the physical record size on storage but never relaxes `max-record-size`; see [concepts-and-patterns.md](concepts-and-patterns.md) § Storage compression.) There are also **no joins**. Every record costs **64 bytes of primary index metadata** (usually in RAM), so many tiny records waste memory. The target record size is **1–128 KiB** (the Goldilocks Principle), and that band is a **distribution rather than a target**: design so the **bulk of records sit in single-digit KiB**, and treat the upper end as headroom for outliers and slowly-changing consolidated structures — 1:N and N:M relationship lists — rather than a size to aim for. Above roughly **50 KiB, justify the record explicitly**, because every update rewrites it in full — size only hurts once multiplied by write frequency. Where writes are infrequent relative to reads, records near the upper end are a legitimate design rather than a compromise; the same size on a hot write path is a defect. In this research, **"a few KiB"** means that same **1–128 KiB** band — it is not a hard single-digit cap, but neither does it make everything up to 128 KiB equally good. Consolidate enough to avoid tiny records (bad PI-to-data ratio), but not so much that a single record becomes a monolith. Aerospike's efficient **batch reads** make many medium-sized records cheap to fetch together, so spreading data across well-sized records is preferred over packing everything into one.

If the LLM's instinct is to create a table per entity and a row per sub-entity, or to embed everything in one giant document, it will produce a bad Aerospike model. The research in this guide provides the patterns and reasoning to do it correctly.

### Modeling failure modes

The eight ways Aerospike data models most often go wrong. These are not LLM-specific — they are relational and document-database habits applied to an architecture that rewards neither.

Full detail, including a detection test and the corrective pattern for each, is in **[modeling-failure-modes.md](modeling-failure-modes.md)**. Read it before designing, and re-read it as a review rubric against a drafted model.

| #   | Failure mode                                                                                                                                 | The rule                                                                                         |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| 1   | [Defaulting to one record per sub-entity](modeling-failure-modes.md#1-defaulting-to-one-record-per-sub-entity)                               | Decide record granularity from cardinality and who drives the read — never from the entity list. |
| 2   | [Using secondary indexes as the primary query mechanism](modeling-failure-modes.md#2-using-secondary-indexes-as-the-primary-query-mechanism) | The most frequent reads must resolve to a key lookup or a bounded batch read.                    |
| 3   | [Ignoring CDT capabilities](modeling-failure-modes.md#3-ignoring-cdt-capabilities)                                                           | Mutations that touch one element of a collection must happen server-side, in place.              |
| 4   | [Treating bins like columns](modeling-failure-modes.md#4-treating-bins-like-columns)                                                         | A bin is a container, not a field. Repeating or sparse data belongs in one CDT bin.              |
| 5   | [Normalizing instead of denormalizing](modeling-failure-modes.md#5-normalizing-instead-of-denormalizing)                                     | Duplicate data deliberately when two access patterns need it in two shapes.                      |
| 6   | [Unbounded collection growth](modeling-failure-modes.md#6-unbounded-collection-growth)                                                       | Every list or map bin needs a growth ceiling and a decided behavior at that ceiling.             |
| 7   | [Treating the checklist as a document template](modeling-failure-modes.md#7-treating-the-checklist-as-a-document-template)                   | Design one entity group at a time, and pass its gates before starting the next.                  |
| 8   | [Ignoring PI cost for small independent entities](modeling-failure-modes.md#8-ignoring-pi-cost-for-small-independent-entities)               | An entity with no relationships and a small payload still needs an explicit sizing decision.     |

### Workflow: from requirements to data model

Follow these steps in order. Each step references the files where the relevant patterns and concepts live.

**Mandatory pre-step for new apps:** Start with [new-app-modeling-checklist.md](new-app-modeling-checklist.md) and complete the required outputs before finalizing record layout. The checklist includes deterministic decision packs for recurring choices (for example 1:N and N:M pattern selection), ambiguity stop gates (`BLOCKED_MISSING_INPUT`), and required decision records.

**Mandatory clarification behavior for LLMs (new apps):** Before proposing a data model, the LLM must explicitly check for ambiguity and ask targeted clarifying questions when needed. Step 0 below makes this concrete — it requires a written Clarification Document as the first deliverable. At minimum, confirm these areas: (1) PRD scope and invariants, (2) entity definitions and ownership/lifecycle rules, (3) access patterns and latency/correctness expectations, and (4) object growth/cardinality/skew assumptions. Do not skip this and wait for the user to prompt.

**Clarification quality rules are mandatory for Step 0 (see below).**

**Step 0 — Produce the Clarification Document (mandatory first deliverable).**

**Question quality rules (mandatory for Step 0):**

- Ask only requirements-gap questions (behavior/usage, expected frequency, growth/skew, consistency tolerance). Avoid storage/mechanism preference prompts unless the product requirement explicitly depends on a mechanism choice.
- Infer from requirements and local research first. Ask only for irreducible missing inputs required by the decision packs.
- Phrase clarification in project language ("How often does X happen?", "What is p95 fan-out?", "Is eventual consistency acceptable?"), then map answers to Aerospike pattern decisions internally.
- If deterministic guidance already resolves a choice, do not ask "which pattern do you prefer?"
- When a default is justified, present: recommended default + rationale + explicit exception-check question ("Any constraint that invalidates this default?").

**Structural-change gate (mandatory before introducing new sets/records/indexes):**

- Default to the canonical/baseline shape when requirements are satisfied.
- Do not introduce a new structural pattern (new set, split record, extra index, materialized view) unless at least one is true:
  1. **Requirement-driven:** a requirement cannot be met with the baseline, or
  2. **Evidence-driven:** measured data shows baseline misses SLO/correctness/capacity.
- If neither is true, mark decision status as `BLOCKED_MISSING_INPUT` or `DEFERRED_OPTIMIZATION`.
- Required proof inputs for structural promotion:
  - cardinality distribution (p50/p95/p99),
  - read/write pressure and burst profile,
  - expected latency/cost impact versus baseline,
  - explicit migration/rollback trigger.
- For read/write tradeoff changes (for example, fan-in vs fan-out materialization), include a minimal comparative test plan before promotion.
- Resolve schema drift before locking contract changes (field names, value types, units, and required/optional semantics across artifacts/clients).
- Log rejected optimizations in an existing decision artifact (for example, savepoint, journal, or ADR note) with reconsider evidence/trigger.
- Any proposal without these inputs is a hypothesis, not a contract change.

Before any record layout, key design, or bin structure work, produce a written document that:

1. Lists every entity and major relationship from Step 1 (entities can be enumerated before access patterns are finalized).
2. For each relationship, lists the required inputs from the applicable decision pack (sections 5.1–5.8 in [new-app-modeling-checklist.md](new-app-modeling-checklist.md)).
3. For each required input, states either: (a) the confirmed value and its source (requirements document, sizing reference, stakeholder answer), or (b) `MISSING — question: [specific question]` where the question is requirements-gap focused (not pattern-preference focused).
4. Includes any PRD scope, lifecycle, or access pattern ambiguities from section 0.1 of the checklist.

This document IS the deliverable for this step. Present it to the stakeholder. **Do not proceed to Step 1 until all MISSING items are resolved or explicitly marked as approved assumptions.** A data model produced without a completed Clarification Document is not compliant with this checklist.

If any major operation has multi-record delete/update side effects and may rely on eventual cleanup, complete Decision Pack 5.8 (Deferred cleanup strategy) before finalizing lifecycle semantics.

If a prior clarification document exists (e.g., from an earlier conversation), verify its completeness against the decision pack inputs before treating it as sufficient.

**Order of Steps 1 and 2:** Understand **what exists in the domain** (entities and relationships) **immediately before** you nail down **how it is read and written** (access patterns). You need a shared vocabulary of entities before access patterns are meaningful; once both are captured, **access patterns drive** record layout, consolidation vs split records, and indexes—not an ER diagram by itself.

**Step 1 — Identify entities and relationships.** List the domain entities (e.g. user, post, comment) and their relationships (1:1, 1:N, N:M). For each relationship, note the cardinality (how large is the "many" side?) and who drives the read (parent→child, child→parent, or both). Research how big different entities **are** (typical size, min and max, and growth rate). Call out **skew**: in social-style products, most users have modest fan-out (e.g. followers) while a few have very large counts—designs often optimize for the common case while tolerating edge cases via consolidation with **paginated list reads**, dedicated consolidated records, or overflow patterns (see [follow-relationship-scale.md](follow-relationship-scale.md)). If entity ownership, lifecycle, or cardinality assumptions are unclear, stop and ask before continuing. For a quick map from relationship type to Aerospike pattern, see [concepts-and-patterns.md](concepts-and-patterns.md) § Relationships at a glance.

**Step 2 — Understand access patterns.** Building on Step 1, enumerate every read and write the application will perform: what data is fetched, by what key or criteria, how often, and what latency is acceptable. Access patterns drive every design decision after this point. If any access path, correctness requirement, or growth assumption is unclear, stop and ask.

**Step 2.5 — Entity-group plan (mandatory before Step 3).** Partition the entities and relationships from Steps 1–2 into groups that share relationships and will be modeled together. An entity in isolation is rarely meaningful — the interesting decisions (consolidation vs split, companion records, fan-out, cascade) live at relationship boundaries. Natural entity groups for a social application might be: "Post + comments + likes", "User + follows + feed", "User + blocks + notification cleanup". **Exception — small independent entities:** entities that have no relationship with other entities and are individually small (below the 1 KiB Goldilocks floor) still need a deliberate sizing decision. They form their own single-entity group and are routed to the "Small independent entities below Goldilocks" guidance in [concepts-and-patterns.md](concepts-and-patterns.md) (Data modeling tips) rather than through the relationship decision packs.

Produce a written plan listing:

1. **Groups and order.** Name each group and the order in which they will be modeled. If one group's output feeds another (e.g., content IDs from the Post group appear in the Feed group), model the dependency first.
2. **Packs per group.** Which decision packs (5.4–5.8) apply to each group.
3. **Input status per group.** For each group, which required inputs from the applicable packs are already confirmed (from Step 0 clarification or requirements) and which are still missing. Missing inputs become group-specific clarification questions in Step 3a.

This plan is the checkpointing artifact. Each group carries a status: `pending`, `in progress`, `review`, `done`, `blocked`, or `revision needed`. Update the status as you work through the groups.

The entity-group plan is a **routing and status** artifact, not a design artifact. It identifies which packs apply to each group and which inputs are confirmed vs missing. It does not record pattern selections, bin structures, or key formats — those are outputs of the per-group record design step (5b) and are not finalized until the group passes its stakeholder checkpoint (5d). An LLM that pre-fills pattern choices in the plan has skipped the per-group gates.

**Gate: Input completeness check.** Before entering the per-group loop, verify that every _broad_ input (PRD scope, consistency model, delete strategy, ID format fork) from Step 0 is confirmed. Group-specific sizing and cardinality questions can wait for the per-group pass, but cross-cutting invariants must be locked.

**Steps 3–5 — Per entity group (repeat for each group in plan order).**

Work through each entity group in the order from the Step 2.5 plan. For each group, complete substeps (a) through (e) before proceeding to the next group.

**(a) Group-specific clarification.** For this group's decision packs, ask any remaining clarifying questions not resolved in Step 0. Because the modeler is now focused on one group and its specific entities, this is where group-specific subtleties naturally surface — for example, which author handle enters the ID recipe for multi-author posts, what the notification payload structure needs for deep-link navigation, or how as-is reposts propagate to follower feeds. Follow the same question quality rules as Step 0 (requirements-gap questions, not implementation-preference questions). Do not proceed to record design for this group until its pack inputs are confirmed or marked as approved assumptions.

**(b) Record design.** For this group's entities and relationships:

- **Choose record granularity (Step 3).** Decide what constitutes one Aerospike record. The default goal is the **few-KiB band: 1–128 KiB** per record, read as a **distribution** — the bulk of records in single-digit KiB, with the upper end reserved for outliers and slowly-changing consolidated structures. Anything expected to exceed ~50 KiB at p99 needs an explicit justification naming its update rate (see [concepts-and-patterns.md](concepts-and-patterns.md) Foundational concepts — Record size limits). Complete the **sizing decision worksheet** and **sizing-to-pattern decision gate** in [new-app-modeling-checklist.md](new-app-modeling-checklist.md). Consult:
  - [concepts-and-patterns.md](concepts-and-patterns.md) — Foundational concepts § Record sizing and primary index cost; Summary § Foundation.
  - [one-to-many-relationships.md](one-to-many-relationships.md) — The three 1:N patterns (list on parent, consolidation, child-held reference + SI) and when to use each.
  - [follow-relationship-scale.md](follow-relationship-scale.md) — Example of how consolidation scales vs. one-record-per-relationship.
  - Also see the worked sizing example in [concepts-and-patterns.md](concepts-and-patterns.md) under "Worked example — comments on a post (sizing sample)".

- **Design record keys (Step 4).** The key determines partition placement and is the primary access path. Design keys so the most common reads map to a single `get` or a bounded `batch get`. Consult:
  - [concepts-and-patterns.md](concepts-and-patterns.md) — Foundational concepts § Key and digest, Data distribution; Data modeling tips § Object ID design for direct access; Applied examples (IoT key = sensor+day, time series key = account+bucket).

- **Design bin structure with CDTs (Step 5).** Choose what goes in each bin. Each bin must pass the bin-extraction checkpoint (4.1) and per-bin justification gate (4.2). Consult:
  - [cdt-api.md](cdt-api.md) — List and Map APIs, nested context, ordering and comparison rules.
  - [concepts-and-patterns.md](concepts-and-patterns.md) — Data modeling tips § Value-based access: prefer List tuple over Map, § Direct access by container type.
  - [path-expressions.md](path-expressions.md) — List-of-structs pattern. **Requires Aerospike DB 8.1.2+**; verify version, client support, and operational acceptance before production use.

- Bin naming convention: use descriptive names unless that exceeds the 15-character bin-name limit, then abbreviate minimally while preserving readability.

**(c) Developer walkthrough.** After completing record design for this group, pause and trace 2–3 concrete developer scenarios through the drafted schema: a create-and-read flow, a mutation that touches multiple records, and a cleanup/cascade. Verify every operation resolves to a specific record key and CDT path without ambiguity, and that the client has all context it needs from prior steps in the workflow — not from additional lookups. Distinguish between actual schema gaps (require revision) and normal engineering work (combining two well-defined patterns). See [new-app-modeling-checklist.md](new-app-modeling-checklist.md) section 5.9 for the full walkthrough protocol.

**(d) Stakeholder review checkpoint.** After the developer walkthrough passes, present a brief decision summary to the stakeholder before proceeding to the next group. Include: the group's schemas (sets, keys, bins), an assumptions log listing every judgment call where the modeler chose between alternatives not fully resolved by deterministic gates, the walkthrough results, and any open items. The stakeholder may approve, request revision, or ask follow-up questions. Do not proceed to the next entity group until the stakeholder approves or explicitly defers review. See [new-app-modeling-checklist.md](new-app-modeling-checklist.md) section 5.9.1 for the full checkpoint protocol.

**(e) Update plan.** Mark this group's status in the entity-group plan: `done`, `blocked`, or `revision needed`. If revision is needed, revise and re-run substeps (b)–(d) before proceeding.

**Step 6 — Consider indexes.** Secondary indexes, set indexes, and expression indexes are tools for specific query patterns — not defaults. Use them when:

- You need to query by a bin value across records (secondary index).
- You need "all records in set S" and the set is small relative to the namespace (set index).
- You need to index a computed value or a sparse subset of records (expression index with `cond(..., unknown())`).
- You need a unique lookup by an external ID (use a lookup-table record, not an SI — see Data modeling tips).

Do **not** create a secondary index for every query pattern. Many access patterns are better served by key design + batch reads, or by lists of related keys on the parent record. Consult:

- [concepts-and-patterns.md](concepts-and-patterns.md) — Foundational concepts § Secondary index, Set index; Data modeling tips § Namespace-wide SI, Unique lookup table, Expression index for sampling; Capacity planning § SI capacity.
- [expressions.md](expressions.md) — Secondary index expressions (expression indexes, 8.1+).

**Step 7 — Plan server-side filtering and operations.** Expressions act as the WHERE clause for scans, queries, and batch reads, and enable computed bins and atomic cross-bin updates. Design with expressions in mind: metadata-based filters (TTL, since_update_time) avoid storage reads; operation expressions replace UDFs for most computed-bin and conditional-write logic. Consult:

- [expressions.md](expressions.md) — Full expression reference (filter, operation, declare/control, CDT ops in expressions).
- [concepts-and-patterns.md](concepts-and-patterns.md) — Applied examples that use expressions (leaderboard rank window, time series combined range + device filter, player matching conditional write, advanced expression techniques).

**Step 8 — Validate record sizes and access patterns.** Walk through each access pattern against the proposed model. For each:

- How many records are read or written?
- What is the expected record size at **p50 and p99** — not just "in the band"? The bulk should sit in single-digit KiB, with the upper end of the **1–128 KiB** Goldilocks band carrying the outliers. Any class above ~50 KiB should have a stated update rate and a reason it is slowly-changing. (Some patterns intentionally exceed that band for skewed fan-out; **paginated reads** keep each response small — see follow-relationship-scale research.)
- Are there hot keys (one record getting disproportionate writes)? If so, consider sharding (see Data modeling tips § Shard-on-demand pattern).
- Does the model handle the "delete" or "cascade" case?

### Which files to consult by task

| Task                                                                                                                                             | Start with                                                                                                              | Then consult                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **New data model from scratch**                                                                                                                  | [new-app-modeling-checklist.md](new-app-modeling-checklist.md)                                                          | [concepts-and-patterns.md](concepts-and-patterns.md) (Foundational concepts + Summary), then other files as needed.             |
| **Modeling a 1:1, 1:N, or N:M relationship**                                                                                                     | [concepts-and-patterns.md](concepts-and-patterns.md) (§ Relationships at a glance)                                      | [one-to-many-relationships.md](one-to-many-relationships.md); [concepts-and-patterns.md](concepts-and-patterns.md) Sources 4, 5 |
| **Scaling a list that can grow very large**                                                                                                      | [follow-relationship-scale.md](follow-relationship-scale.md)                                                            | [one-to-many-relationships.md](one-to-many-relationships.md)                                                                    |
| **Choosing list vs map; ordering and persist-index** (three map subtypes: Unordered / K-ordered / KV-ordered; `persistIndex` for lists and maps) | [cdt-api.md](cdt-api.md)                                                                                                | [concepts-and-patterns.md](concepts-and-patterns.md) (Data modeling tips § value-based access)                                  |
| **Server-side filtering or computed bins**                                                                                                       | [expressions.md](expressions.md)                                                                                        | [concepts-and-patterns.md](concepts-and-patterns.md) (applied examples using expressions)                                       |
| **Nested CDT querying or list-of-structs**                                                                                                       | [path-expressions.md](path-expressions.md) (**Aerospike Database 8.1.2+**)                                              | [cdt-api.md](cdt-api.md) (nested context)                                                                                       |
| **Time series or event data**                                                                                                                    | [concepts-and-patterns.md](concepts-and-patterns.md) (Sources 1, 8, 9)                                                  | [expressions.md](expressions.md) (combined range + filter)                                                                      |
| **Leaderboard or ranked data**                                                                                                                   | [concepts-and-patterns.md](concepts-and-patterns.md) (Source 6)                                                         | [expressions.md](expressions.md) (MapExp for rank window)                                                                       |
| **Small independent entities below Goldilocks**                                                                                                  | [concepts-and-patterns.md](concepts-and-patterns.md) (Data modeling tips § Small independent entities below Goldilocks) | [cdt-api.md](cdt-api.md) (map operations for hash-bucket consolidation)                                                         |
| **Deployment uses storage compression**                                                                                                          | [concepts-and-patterns.md](concepts-and-patterns.md) (§ Storage compression)                                            | [concepts-and-patterns.md](concepts-and-patterns.md) (§ Capacity planning)                                                      |
